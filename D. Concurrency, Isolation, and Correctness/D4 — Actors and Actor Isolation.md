---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actors
  - actor-isolation
---
## 0. Rubric snapshot

**Rubric expectation**

Understand actor-isolated state, message sends across actor boundaries, and why actors prevent data races but not all correctness issues.

**Caveats**

Actors serialize access to their isolated state, not the entire logical workflow of a feature.

**You should be able to answer**

- What kind of problem does an actor solve, and what kind of problem does it not solve?
- Why can actor-based code still have stale-read or check-then-act logic bugs?

**You should be able to do**

- Review a token-refresh actor and identify a logic race that can still happen despite actor isolation.

---

## 1. Core mental model

An `actor` is a reference type whose mutable instance state is protected by **actor isolation**. Code inside the actor can synchronously read and mutate actor-isolated state. Code outside the actor must cross the actor boundary, usually with `await`. Swift’s language model guarantees that only code running in the actor’s isolation domain can directly access that actor’s local state. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/?utm_source=chatgpt.com "Concurrency - Documentation | Swift.org"))

The actor is not “a magic thread.” It is closer to a protected concurrency domain. Calls from outside the actor become work that the actor executes when it can safely do so. SE-0306 describes actor calls as messages placed in the actor’s mailbox; the actor processes actor-isolated work one-at-a-time, preventing simultaneous mutation of isolated state. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

The key distinction:

```text
Actor isolation prevents data races.
It does not automatically make multi-step business logic atomic.
```

That means this is safe from raw memory/data-race corruption:

```swift
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }
}
```

But this can still be logically wrong:

```swift
actor TokenStore {
    private var token: Token?

    func validToken() async throws -> Token {
        if let token, token.isValid {
            return token
        }

        let newToken = try await refreshFromServer()
        token = newToken
        return newToken
    }
}
```

The actor protects `token` from being mutated by two pieces of actor-isolated code at the exact same time. But the `await refreshFromServer()` is a suspension point. While this method is suspended, the actor can run other work. Another caller can enter `validToken()`, also observe no valid token, and start another refresh. That is not a data race. It is a **logic race**.

---

## 2. Essential mechanics

### 2.1 Actor-isolated state is protected by default

Instance properties and instance methods of an actor are actor-isolated by default. Inside the actor, you can access isolated state synchronously.

```swift
actor ImageCache {
    private var images: [URL: Data] = [:]

    func insert(_ data: Data, for url: URL) {
        images[url] = data
    }

    func data(for url: URL) -> Data? {
        images[url]
    }
}
```

From outside:

```swift
let cache = ImageCache()

await cache.insert(data, for: url)
let data = await cache.data(for: url)
```

Even though `insert` and `data(for:)` are synchronous methods inside the actor, crossing the actor boundary from outside requires `await`. Swift’s compiler diagnostics explicitly enforce that actor-isolated calls from outside must be asynchronous, because synchronous access could race with actor state. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/?utm_source=chatgpt.com "Calling an actor-isolated method from a synchronous ..."))

---

### 2.2 Actors serialize isolated execution, not every operation in the universe

An actor serializes access to its own isolated state. It does not serialize:

- network requests
- database transactions outside the actor
- file system side effects
- Keychain writes unless they are modeled behind the actor
- state owned by another actor
- state returned from the actor and mutated elsewhere
- a whole user workflow across multiple suspension points

```swift
actor Cart {
    private var items: [Item] = []

    func checkout(paymentService: PaymentService) async throws {
        guard !items.isEmpty else { return }

        let snapshot = items

        // Suspension point.
        try await paymentService.charge(for: snapshot)

        // Another task may have modified `items` while this method was suspended.
        items.removeAll()
    }
}
```

The actor prevents simultaneous unsynchronized mutation of `items`. It does not guarantee that `items` is unchanged after `await`.

---

### 2.3 Cross-actor calls are message sends

When you call an actor from outside, you are not “just calling a method.” You are asking the actor to run that operation in its isolation domain.

```swift
actor Account {
    private var balance: Decimal = 0

    func deposit(_ amount: Decimal) {
        balance += amount
    }

    func currentBalance() -> Decimal {
        balance
    }
}

let account = Account()

await account.deposit(100)
let balance = await account.currentBalance()
```

SE-0306 describes this as placing a message in the actor’s mailbox; the actor processes actor-isolated work one-at-a-time. The proposal also notes that actor execution is not strictly FIFO like a serial `DispatchQueue`; the runtime may consider task priority. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

Production implication: do not design actor APIs that depend on exact call ordering unless you encode that ordering explicitly in the actor’s state machine.

---

### 2.4 `await` is where your actor invariants can become stale

Between two non-suspending lines inside an actor method, actor-isolated state is protected.

```swift
actor BankAccount {
    private var balance: Int = 100

    func withdraw(_ amount: Int) throws {
        guard balance >= amount else {
            throw BankError.insufficientFunds
        }

        balance -= amount
    }
}
```

This check-then-act is safe because there is no suspension point between the check and mutation.

This version is different:

```swift
actor BankAccount {
    private var balance: Int = 100

    func withdrawAfterApproval(_ amount: Int, approver: Approver) async throws {
        guard balance >= amount else {
            throw BankError.insufficientFunds
        }

        try await approver.approveWithdrawal(amount)

        balance -= amount
    }
}
```

The `guard` result can become stale while the actor method is suspended. Another task could withdraw money before this method resumes. The actor prevents data races, but not stale business assumptions.

---

## 3. Common traps and misconceptions

### Trap 1: “Actors make this whole workflow atomic”

Bad:

```swift
actor SessionManager {
    private var token: Token?

    func tokenForRequest() async throws -> Token {
        if let token, token.isValid {
            return token
        }

        let refreshed = try await refreshToken()
        token = refreshed
        return refreshed
    }
}
```

Problem:

```text
Two callers can both observe an expired token, both suspend during refresh, and both perform refresh work.
```

Better:

```swift
actor SessionManager {
    private var token: Token?
    private var refreshTask: Task<Token, Error>?

    private let refresh: @Sendable () async throws -> Token

    init(refresh: @escaping @Sendable () async throws -> Token) {
        self.refresh = refresh
    }

    func tokenForRequest() async throws -> Token {
        if let token, token.isValid {
            return token
        }

        if let refreshTask {
            return try await refreshTask.value
        }

        let task = Task<Token, Error> {
            try await refresh()
        }

        refreshTask = task

        do {
            let refreshed = try await task.value
            token = refreshed
            refreshTask = nil
            return refreshed
        } catch {
            refreshTask = nil
            throw error
        }
    }
}
```

This turns refresh into a **single-flight operation**: one refresh task is shared by concurrent callers.

---

### Trap 2: “A serial actor is the same as a serial dispatch queue”

Not exactly.

A serial queue is usually FIFO. Swift actors process isolated work one-at-a-time, but actor scheduling is not specified as strict FIFO; the runtime can account for priorities. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

Do not depend on this:

```swift
await store.setMode(.loading)
await store.setMode(.loaded)
```

as a global ordering guarantee across multiple tasks. If ordering matters, encode it:

```swift
actor StateMachine {
    private var generation = 0
    private var state: State = .idle

    func beginLoad() -> Int {
        generation += 1
        state = .loading
        return generation
    }

    func finishLoad(generation expectedGeneration: Int, value: Value) {
        guard expectedGeneration == generation else {
            return
        }

        state = .loaded(value)
    }
}
```

---

### Trap 3: “Returning a value from an actor is always safe”

Returning value-semantic `Sendable` data is usually fine.

```swift
actor ProfileStore {
    private var profile: Profile

    func snapshot() -> Profile {
        profile
    }
}
```

But returning mutable reference state is dangerous unless the type is safely shareable.

```swift
final class MutableProfile {
    var name: String = ""
}

actor ProfileStore {
    private var profile = MutableProfile()

    func currentProfile() -> MutableProfile {
        profile
    }
}
```

This leaks mutable state out of the actor. In Swift 6 strict concurrency, non-`Sendable` mutable reference types crossing isolation boundaries are exactly the kind of design the compiler tries to reject or warn about. SE-0306 states that cross-actor references necessarily traffic in values shared across concurrently executing code, which is why `Sendable` matters. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

Better:

```swift
struct ProfileSnapshot: Sendable {
    let name: String
}

actor ProfileStore {
    private var name = ""

    func snapshot() -> ProfileSnapshot {
        ProfileSnapshot(name: name)
    }
}
```

---

## 4. Direct answers to rubric questions

### Q1. What kind of problem does an actor solve, and what kind of problem does it not solve?

An actor solves **safe access to shared mutable state**. It gives a reference type an isolation domain so that its mutable state cannot be directly accessed concurrently from arbitrary code.

It does **not** solve all ordering, freshness, transactionality, cancellation, or distributed side-effect problems.

Actors solve:

```text
"Can two tasks mutate this dictionary at the same time?"
```

Actors do not automatically solve:

```text
"Did the token I checked before await remain valid after await?"
"Did two callers start duplicate refresh work?"
"Did this sequence of calls happen in the business order I intended?"
"Did my external database transaction remain consistent?"
```

Interview version:

> An actor is the right tool when I have shared mutable state that needs a clear isolation boundary. Swift enforces that only actor-isolated code can touch that state directly, so it prevents data races. But an actor is not a transaction system. Once an actor method suspends, other work can interleave, and any assumptions made before the suspension may be stale when the method resumes.

---

### Q2. Why can actor-based code still have stale-read or check-then-act logic bugs?

Because `await` is a suspension point. Actor isolation protects state while actor-isolated code is actively executing. But if an actor method checks state, then awaits external work, the actor can process other messages before the original method resumes.

Bad pattern:

```swift
actor Inventory {
    private var stock = 1

    func reserveOne(paymentService: PaymentService) async throws {
        guard stock > 0 else {
            throw InventoryError.outOfStock
        }

        try await paymentService.authorize()

        stock -= 1
    }
}
```

Possible interleaving:

```text
Task A enters reserveOne()
Task A sees stock == 1
Task A awaits payment authorization

Task B enters reserveOne()
Task B sees stock == 1
Task B awaits payment authorization

Task A resumes, stock becomes 0
Task B resumes, stock becomes -1
```

There is no data race. Every access to `stock` is isolated. But the logic is wrong because the check and mutation were separated by a suspension point.

Better:

```swift
actor Inventory {
    private var stock = 1

    func reserveOne() throws -> Reservation {
        guard stock > 0 else {
            throw InventoryError.outOfStock
        }

        stock -= 1
        return Reservation()
    }

    func release(_ reservation: Reservation) {
        stock += 1
    }
}
```

Then perform payment outside the critical actor mutation:

```swift
let reservation = try await inventory.reserveOne()

do {
    try await paymentService.authorize()
} catch {
    await inventory.release(reservation)
    throw error
}
```

Interview version:

> Actor isolation makes individual access to actor state race-free. It does not freeze the actor across suspension points. If I check a condition, then `await`, then act based on the earlier condition, that condition may no longer be true. The fix is usually to keep invariant-sensitive mutations in synchronous actor methods, store an in-flight operation, or model the workflow as an explicit state machine.

---

## 5. Code probe

The D4 rubric has no explicit code probe, so use these probes to test the core mechanics.

### 5.1 Minimal valid probe

Given:

```swift
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }

    func snapshot() -> Int {
        value
    }
}

let counter = Counter()

await counter.increment()
print(await counter.snapshot())
```

### What happens?

```text
1
```

### Why?

`value` is actor-isolated. It is mutated only from inside the actor. From outside, both calls cross the actor boundary, so they require `await`.

```text
Outside task
   |
   | await counter.increment()
   v
Counter actor isolation domain
   value += 1

Outside task
   |
   | await counter.snapshot()
   v
Counter actor isolation domain
   return value
```

---

### 5.2 Counterexample probe

Given:

```swift
actor Counter {
    var value = 0
}

func read(_ counter: Counter) -> Int {
    counter.value
}
```

### What happens?

In Swift 6.x, this is rejected with a diagnostic equivalent to:

```text
error: actor-isolated property 'value' can not be referenced from a nonisolated context
```

### Why?

`read(_:)` is not isolated to `counter`. It is ordinary synchronous code. Reading `counter.value` directly would bypass the actor’s isolation boundary. The correct fix is to expose an actor method and await it:

```swift
actor Counter {
    private var value = 0

    func snapshot() -> Int {
        value
    }
}

func read(_ counter: Counter) async -> Int {
    await counter.snapshot()
}
```

---

### 5.3 Production-style probe: stale check

Given:

```swift
actor DownloadGate {
    private var isDownloading = false

    func startIfNeeded() async {
        guard !isDownloading else {
            return
        }

        try? await Task.sleep(for: .milliseconds(100))

        isDownloading = true
        print("started")
    }
}

let gate = DownloadGate()

await withTaskGroup(of: Void.self) { group in
    for _ in 0..<2 {
        group.addTask {
            await gate.startIfNeeded()
        }
    }
}
```

### What can happen?

```text
started
started
```

### Why?

Both tasks can observe `isDownloading == false`, then both suspend. When they resume, each sets `isDownloading = true`.

The actor prevented simultaneous memory access. It did not make the check and later mutation atomic because the check and mutation were separated by `await`.

### Fix

```swift
actor DownloadGate {
    private var isDownloading = false

    func claimDownloadSlot() -> Bool {
        guard !isDownloading else {
            return false
        }

        isDownloading = true
        return true
    }

    func finishDownload() {
        isDownloading = false
    }
}

let gate = DownloadGate()

await withTaskGroup(of: Void.self) { group in
    for _ in 0..<2 {
        group.addTask {
            guard await gate.claimDownloadSlot() else {
                return
            }

            defer {
                Task {
                    await gate.finishDownload()
                }
            }

            try? await Task.sleep(for: .milliseconds(100))
            print("started")
        }
    }
}
```

Better production shape:

```swift
actor DownloadGate {
    private var task: Task<Void, Never>?

    func startIfNeeded(
        operation: @escaping @Sendable () async -> Void
    ) {
        guard task == nil else {
            return
        }

        task = Task {
            await operation()
            await clearTask()
        }
    }

    private func clearTask() {
        task = nil
    }
}
```

The stronger version stores the in-flight work itself, not just a Boolean.

---

## 6. Exercise

### Problem

Review a token-refresh actor and identify a logic race that can still happen despite actor isolation.

### Bad / naive version

```swift
struct Token: Sendable {
    let rawValue: String
    let expiresAt: Date

    var isValid: Bool {
        expiresAt > .now
    }
}

protocol AuthClient: Sendable {
    func refreshToken() async throws -> Token
}

actor TokenStore {
    private var token: Token?
    private let authClient: AuthClient

    init(authClient: AuthClient) {
        self.authClient = authClient
    }

    func validToken() async throws -> Token {
        if let token, token.isValid {
            return token
        }

        let refreshed = try await authClient.refreshToken()
        token = refreshed
        return refreshed
    }
}
```

### What is wrong?

```text
This actor is data-race safe, but it is not logic-race safe.
```

Possible interleaving:

```text
Task A calls validToken()
Task A sees token is nil/expired
Task A starts refreshToken() and suspends

Task B calls validToken()
Task B also sees token is nil/expired
Task B starts another refreshToken() and suspends

Task A receives token A and stores it
Task B receives token B and stores it
```

Problems this can cause:

- duplicate refresh requests
- wasted network work
- token rotation bugs
- refresh-token invalidation
- last-write-wins with an older or undesired token
- inconsistent request retry behavior

No actor data race exists. The bug is in the workflow.

---

### Improved version: single-flight token refresh

```swift
struct Token: Sendable {
    let rawValue: String
    let expiresAt: Date

    var isValid: Bool {
        expiresAt > .now
    }
}

actor TokenStore {
    private var token: Token?
    private var refreshTask: Task<Token, Error>?

    private let refresh: @Sendable () async throws -> Token

    init(refresh: @escaping @Sendable () async throws -> Token) {
        self.refresh = refresh
    }

    func validToken() async throws -> Token {
        if let token, token.isValid {
            return token
        }

        if let refreshTask {
            return try await refreshTask.value
        }

        let task = Task<Token, Error> {
            try await refresh()
        }

        refreshTask = task

        do {
            let refreshed = try await task.value
            token = refreshed
            refreshTask = nil
            return refreshed
        } catch {
            refreshTask = nil
            throw error
        }
    }

    func invalidate() {
        token = nil
    }
}
```

### Why this is better

The actor now models both:

```text
cached token
in-flight refresh
```

That means the state machine has three useful states:

```swift
enum TokenState {
    case empty
    case valid(Token)
    case refreshing(Task<Token, Error>)
}
```

The important fix is not “use an actor.” The fix is “make the in-flight operation part of the actor’s isolated state.”

With this shape:

```text
Task A starts refresh and stores refreshTask
Task B sees refreshTask and awaits the same task
Task C sees refreshTask and awaits the same task
Only one refresh request is active
```

---

### Even better: explicit state machine

```swift
actor TokenStore {
    private enum State {
        case empty
        case valid(Token)
        case refreshing(Task<Token, Error>)
    }

    private var state: State = .empty
    private let refresh: @Sendable () async throws -> Token

    init(refresh: @escaping @Sendable () async throws -> Token) {
        self.refresh = refresh
    }

    func validToken() async throws -> Token {
        switch state {
        case .valid(let token) where token.isValid:
            return token

        case .refreshing(let task):
            return try await task.value

        case .empty, .valid:
            let task = Task<Token, Error> {
                try await refresh()
            }

            state = .refreshing(task)

            do {
                let token = try await task.value
                state = .valid(token)
                return token
            } catch {
                state = .empty
                throw error
            }
        }
    }

    func invalidate() {
        state = .empty
    }
}
```

This is usually the staff-level answer: use the actor to guard a real domain state machine, not a loose bag of properties.

---

## 7. Production guidance

Use actors in production when:

```text
You have shared mutable state.
The state has a clear owner.
You want the compiler to enforce isolation.
The API can tolerate async boundary crossings.
The actor can expose domain operations, not raw mutable storage.
```

Good iOS examples:

```text
Token/session store
Image cache with in-flight request coalescing
Analytics event buffer
Feature flag store
Local database write coordinator
Upload/download coordinator
WebSocket connection state
```

Be careful when:

```text
The actor method awaits in the middle of an invariant-sensitive workflow.
You coordinate multiple actors.
You expose mutable reference types from the actor.
You assume actor calls are FIFO.
You use an actor for tiny hot-path synchronous state where a lock would be simpler.
You mark everything actor-isolated instead of designing ownership clearly.
```

Avoid actors when:

```text
The data is immutable.
The type is a simple value model.
The state is UI-only and should be @MainActor instead.
You need tight synchronous performance with very small critical sections.
You are using the actor as a dumping ground for unrelated global state.
```

Debugging checklist:

```text
Where is the isolated state owned?
Which methods cross the actor boundary?
Where are the suspension points?
What assumptions are made before await?
Can another call invalidate those assumptions before resume?
Does the actor return mutable reference state?
Should in-flight work be stored as state?
Is this a data race, a logic race, or an ordering bug?
Is the actor API exposing operations or exposing storage?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Actors make code thread-safe.

This is incomplete. It misses isolation boundaries, `await`, reentrancy, stale state, and business invariants.

### Senior answer

> Actors isolate mutable state and force cross-actor access through async boundaries. They prevent data races on actor-isolated state, but I still need to reason carefully around suspension points because state checked before `await` can become stale.

### Staff-level answer

> I use actors to define ownership boundaries around mutable state, but I do not treat them as transactions. The actor API should expose domain operations that preserve invariants, not raw getters and setters. If an operation has an in-flight phase, such as token refresh or image loading, I model that in-flight work as actor state. I also review Sendable boundaries, cancellation behavior, priority behavior, and whether a lock or value model would be simpler.

Staff-level questions to ask:

```text
What state does this actor actually own?
What invariant must never be broken?
Can that invariant be preserved without suspension?
If not, should the in-flight operation be stored?
What crosses the actor boundary, and is it Sendable?
Does cancellation of one caller cancel shared work?
Are multiple actors involved, and can that create ordering bugs?
Would @MainActor, a Mutex, or immutability be a better fit?
```

---

## 9. Interview-ready summary

Actors are Swift’s language-level tool for isolating shared mutable state. An actor gives its instance state a concurrency domain, and outside code must cross that boundary asynchronously, usually with `await`. This prevents data races because actor-isolated state is not directly accessed by multiple concurrent tasks. But actors do not make an entire workflow atomic. Any `await` inside an actor method is a suspension point where other actor work may interleave, so check-then-act logic can still be wrong. In production, I design actor APIs around domain invariants, keep critical state transitions synchronous when possible, and explicitly model in-flight work such as token refreshes.

---

## 10. Flashcards

Q: What is actor isolation?  
A: Actor isolation is Swift’s rule that actor-isolated instance state can be directly accessed only by code running in that actor’s isolation domain.

Q: Are actors value types or reference types?  
A: Actors are reference types, but unlike ordinary classes, they protect mutable instance state through actor isolation. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

Q: Why does calling an actor method from outside usually require `await`?  
A: Because the call crosses the actor boundary and may need to wait until the actor can run that operation safely.

Q: Do actors prevent data races?  
A: Yes, for actor-isolated state, when you stay within Swift’s isolation and `Sendable` rules.

Q: Do actors prevent all race conditions?  
A: No. They prevent data races, but logic races, stale reads, duplicate work, and incorrect ordering can still happen.

Q: Why is `await` inside an actor method dangerous?  
A: It can suspend the method, allowing other work to run on the same actor before the original method resumes.

Q: What is a stale-read bug in an actor?  
A: A bug where code reads actor state, awaits, then acts on the old assumption after another task has changed the state.

Q: How do you fix duplicate token refresh in an actor?  
A: Store the in-flight refresh task inside the actor and make concurrent callers await the same task.

Q: Should actor APIs expose raw mutable storage?  
A: Usually no. They should expose operations that preserve domain invariants.

Q: When might a lock be better than an actor?  
A: For tiny synchronous critical sections where async boundaries are unnecessary and a disciplined lock is simpler.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D5 — Global actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D8 — Isolation control: `nonisolated`, isolated parameters, and `assumeIsolated`]]
- [[D12 — Synchronization library, mutexes, and choosing between actors and locks]]
- [[D13 — Swift 6 strict concurrency migration]]

---

## 12. Sources

- Swift Senior/Staff Rubric — D4 Actors and actor isolation.
- The Swift Programming Language — Concurrency / actor isolation. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/?utm_source=chatgpt.com "Concurrency - Documentation | Swift.org"))
- Swift compiler diagnostics — actor-isolated calls from synchronous nonisolated contexts. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/?utm_source=chatgpt.com "Calling an actor-isolated method from a synchronous ..."))
- Swift Evolution SE-0306 — Actors, actor isolation, cross-actor references, actor mailbox, and `Sendable` interaction. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))