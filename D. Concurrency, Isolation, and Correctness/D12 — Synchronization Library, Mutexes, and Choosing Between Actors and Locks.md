---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actors
  - mutex
  - synchronization
  - swift6
---
## 0. Rubric snapshot

**Rubric expectation**

Know that actors are not the only concurrency primitive; low-level synchronization still matters in some contexts. The rubric specifically asks you to compare actors, mutexes, locks, and immutability for shared mutable state.

**Caveats**

Locking can be the correct tool for tightly scoped shared mutable state, but only with disciplined invariants. The rubric puts D12 in “Strong Senior / Emerging Staff Depth,” which is accurate: this is less about syntax and more about choosing the right ownership model.

**You should be able to answer**

- When is an actor a better fit than a mutex?
- When is a mutex still appropriate?
- Why can wrapping everything in a lock be worse than modeling ownership explicitly?

**You should be able to do**

- Choose between a lock, an actor, and immutability for a shared cache used from UI and background work.

---

## 1. Core mental model

Swift gives you several tools for avoiding data races. They do not solve the same problem.

Actors protect mutable state by isolating it into a concurrency domain. Access from outside the actor is asynchronous and goes through actor isolation. Swift’s concurrency model guarantees that only code running on an actor can access that actor’s isolated state directly. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/?utm_source=chatgpt.com "Concurrency - Documentation | Swift.org"))

A mutex protects mutable state by mutual exclusion. One execution context enters a critical section at a time. Swift 6 introduced the `Synchronization` library with low-level concurrency APIs, including atomics and a mutex API. ([Swift.org](https://swift.org/blog/announcing-swift-6/ "Announcing Swift 6 | Swift.org"))

The key distinction:

```text
Actor = isolated async owner of state.
Mutex = synchronous critical section around state.
Immutability = no synchronization needed because nothing mutates.
```

A good default in modern Swift is:

```text
Prefer no shared mutable state.
If state must be shared:
  use value snapshots when possible,
  use actors for async/domain ownership,
  use Mutex for tiny synchronous critical sections.
```

The mistake is treating “make it thread-safe” as “put a lock around it.” Thread safety is not just mutual exclusion. It also includes ownership, lifecycle, cancellation, reentrancy, invariants, API shape, and whether callers can observe half-updated state.

---

## 2. Essential mechanics

### 2.1 `Synchronization.Mutex` protects state, not arbitrary code

Swift’s `Mutex` is a synchronization primitive that protects shared mutable state through mutual exclusion. SE-0433 introduced it as a standard-library synchronization primitive in the `Synchronization` module. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md "swift-evolution/proposals/0433-mutex.md at main · swiftlang/swift-evolution · GitHub"))

```swift
import Synchronization

final class Counter: Sendable {
    private let value = Mutex(0)

    func increment() -> Int {
        value.withLock { value in
            value += 1
            return value
        }
    }
}

let counter = Counter()
print(counter.increment())
print(counter.increment())
```

Output:

```text
1
2
```

The important design detail is `withLock`. The lock is acquired before the closure and released after the closure. Apple documents `withLock(_:)` as calling a closure after acquiring the lock and releasing ownership afterward. ([Apple Developer](https://developer.apple.com/documentation/synchronization/mutex/withlock%28_%3A%29?utm_source=chatgpt.com "withLock(_:) | Apple Developer Documentation"))

This is safer than manually calling `lock()` / `unlock()` because `defer`-style release is built into the API shape.

---

### 2.2 A mutex critical section must stay synchronous and small

Do not suspend inside a lock. `Mutex.withLock` expects a synchronous closure. This is deliberate.

Bad:

```swift
import Synchronization

let cache = Mutex([String: Int]())

func fetchValue() async -> Int {
    42
}

func bad() async {
    await cache.withLock { state in
        state["answer"] = await fetchValue()
    }
}
```

Compiler error with Swift 6.2:

```text
/tmp/mutex_await.swift:8:26: error: cannot pass function of type '(inout sending [String : Int]) async -> sending ()' to parameter expecting synchronous function type
 6 | 
 7 | func bad() async {
 8 |     await mutex.withLock { state in
   |                          `- error: cannot pass function of type '(inout sending [String : Int]) async -> sending ()' to parameter expecting synchronous function type
 9 |         state["a"] = await load()
   |                      `- note: 'async' inferred from asynchronous operation used here
10 |     }
11 | }
```

SE-0433 explicitly calls out that primitive `lock()` / `unlock()` operations are dangerous with `async` / `await`, because after a suspension, execution may resume on a different thread. The closure-based API keeps the critical section synchronous. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md "swift-evolution/proposals/0433-mutex.md at main · swiftlang/swift-evolution · GitHub"))

Correct shape:

```swift
func good() async {
    let value = await fetchValue()

    cache.withLock { state in
        state["answer"] = value
    }
}
```

The async work happens outside the lock. The mutation happens inside a short synchronous critical section.

---

### 2.3 Actors are better when the state has async behavior or domain ownership

Use an actor when the protected state belongs to a logical subsystem:

```swift
actor TokenStore {
    private var token: String?
    private var refreshTask: Task<String, Error>?

    func validToken() async throws -> String {
        if let token {
            return token
        }

        if let refreshTask {
            return try await refreshTask.value
        }

        let task = Task {
            try await refreshTokenFromServer()
        }

        refreshTask = task

        do {
            let newToken = try await task.value
            token = newToken
            refreshTask = nil
            return newToken
        } catch {
            refreshTask = nil
            throw error
        }
    }
}
```

This is actor-shaped because:

```text
The state is not just a dictionary.
The actor owns a workflow:
  cache hit,
  in-flight request coalescing,
  async refresh,
  error handling,
  state update.
```

A mutex would protect individual reads and writes, but it would not automatically model the async workflow.

---

### 2.4 Immutability is the best synchronization primitive

If data can be represented as immutable values, do that first.

```swift
struct FeatureFlags: Sendable {
    let isNewCheckoutEnabled: Bool
    let maxRetryCount: Int
    let cacheTTL: Duration
}
```

No actor. No mutex. No queue. No lock ordering. No deadlock. No stale partial mutation.

For UI, this is often the cleanest model:

```swift
@MainActor
final class ProductViewModel: ObservableObject {
    @Published private(set) var state: State = .idle

    enum State: Sendable {
        case idle
        case loading
        case loaded(ProductSnapshot)
        case failed(String)
    }
}

struct ProductSnapshot: Sendable {
    let title: String
    let priceText: String
    let imageURL: URL?
}
```

The UI consumes immutable snapshots. Background systems can build snapshots and send them across isolation boundaries safely.

---

## 3. Common traps and misconceptions

### Trap 1: “Actors replaced locks”

They did not. Actors are excellent for isolated mutable state with async access. Mutexes are still useful for synchronous shared state, low-level data structures, callback bridging, and legacy synchronous APIs.

Bad:

```swift
actor FastCounter {
    private var value = 0

    func increment() -> Int {
        value += 1
        return value
    }
}
```

This forces callers to use `await` even though the operation is tiny and synchronous:

```swift
let value = await counter.increment()
```

Better when you truly need a synchronous counter:

```swift
import Synchronization

final class FastCounter: Sendable {
    private let value = Mutex(0)

    func increment() -> Int {
        value.withLock {
            $0 += 1
            return $0
        }
    }
}
```

Use the actor if the counter belongs to a larger async domain. Use the mutex if the operation is a tiny synchronous critical section.

---

### Trap 2: Holding a lock across slow work

Bad:

```swift
cache.withLock { state in
    let image = expensiveDecode(state.rawData)
    state.decodedImage = image
}
```

Better:

```swift
let rawData = cache.withLock { state in
    state.rawData
}

let image = expensiveDecode(rawData)

cache.withLock { state in
    state.decodedImage = image
}
```

The critical section should protect the shared state, not become a random serialized execution zone.

---

### Trap 3: Protecting data but not protecting invariants

This is thread-safe at the individual operation level:

```swift
final class BadInventory: Sendable {
    private let stock = Mutex(["book": 1])

    func count(for item: String) -> Int {
        stock.withLock { $0[item, default: 0] }
    }

    func removeOne(_ item: String) {
        stock.withLock { $0[item, default: 0] -= 1 }
    }
}
```

But this caller can still create a logic race:

```swift
if inventory.count(for: "book") > 0 {
    inventory.removeOne("book")
}
```

Two callers can both observe `count > 0` and then both remove.

Better:

```swift
final class Inventory: Sendable {
    private let stock = Mutex(["book": 1])

    func reserveOne(_ item: String) -> Bool {
        stock.withLock { stock in
            guard stock[item, default: 0] > 0 else {
                return false
            }

            stock[item, default: 0] -= 1
            return true
        }
    }
}
```

The invariant must live inside the critical section:

```text
check + mutate must be one operation
```

---

### Trap 4: Using a lock to hide bad ownership

Bad:

```swift
final class AppGlobals {
    static let shared = AppGlobals()

    let user = Mutex<User?>(nil)
    let token = Mutex<String?>(nil)
    let cart = Mutex<Cart>(Cart())
    let settings = Mutex<Settings>(Settings())
}
```

This is not good design just because it is synchronized. It creates global shared mutable state with unclear ownership.

Better:

```text
SessionActor owns session/token state.
CartStore owns cart state.
Settings are immutable snapshots.
UI receives state through @MainActor view models.
```

A lock protects memory access. It does not create a sane architecture.

---

## 4. Direct answers to rubric questions

### Q1. When is an actor a better fit than a mutex, and when is a mutex still appropriate?

An actor is better when the state belongs to a logical async domain: token refresh, image loading, database access, websocket state, upload coordination, cache miss handling, in-flight request deduplication, or anything where operations naturally contain `await`.

A mutex is still appropriate when the state is small, synchronous, tightly scoped, and the critical section is short: counters, simple memory caches, callback continuation storage, LRU bookkeeping, metrics aggregation, or bridging code that must be callable from non-async contexts.

Interview version:

> I use actors when I want isolated ownership of mutable state and the operations are naturally async or domain-level. I use a mutex when I need a small synchronous critical section around shared mutable state. The key is that actors model ownership and isolation, while mutexes model mutual exclusion. If the operation may suspend, it should not be inside a lock.

---

### Q2. Why can wrapping everything in a lock be worse than modeling ownership explicitly?

Because locks serialize memory access, not architecture. They do not clarify who owns the state, whether operations are atomic at the business-logic level, how cancellation works, whether state can be observed half-updated, or whether lock ordering can deadlock.

A lock-heavy design often produces:

```text
global mutable state
unclear ownership
wide critical sections
deadlock risk
priority inversion risk
check-then-act races
hard testing
hidden performance bottlenecks
```

Modeling ownership explicitly often removes the need for locks:

```swift
struct UserSessionSnapshot: Sendable {
    let userID: User.ID
    let accessToken: String
    let expirationDate: Date
}
```

Or concentrates mutation behind one owner:

```swift
actor SessionStore {
    private var session: UserSessionSnapshot?

    func update(_ session: UserSessionSnapshot?) {
        self.session = session
    }

    func currentSession() -> UserSessionSnapshot? {
        session
    }
}
```

Interview version:

> A lock is a low-level safety tool, not an ownership model. If every object has a lock, the code may be data-race-safe but still logically unsafe. Staff-level Swift code tries to make ownership explicit: immutable values where possible, actors for isolated async domains, and locks only for small synchronous state with well-defined invariants.

---

## 5. Code probe

The rubric has no D12 code probe. Use these instead.

### 5.1 Minimal correct mutex example

```swift
import Synchronization

final class MemoryCounter: Sendable {
    private let state = Mutex(0)

    func increment() -> Int {
        state.withLock { value in
            value += 1
            return value
        }
    }
}

let counter = MemoryCounter()
print(counter.increment())
print(counter.increment())
```

Output:

```text
1
2
```

Why:

```text
Mutex owns the protected Int.
withLock gives exclusive inout access to that Int.
Only one execution context can run the closure at a time.
The mutation and return happen inside the critical section.
```

---

### 5.2 Counterexample: async work inside a lock

```swift
import Synchronization

let mutex = Mutex([String: Int]())

func load() async -> Int {
    1
}

func bad() async {
    await mutex.withLock { state in
        state["a"] = await load()
    }
}
```

Compiler error with Swift 6.2:

```text
/tmp/mutex_await.swift:8:26: error: cannot pass function of type '(inout sending [String : Int]) async -> sending ()' to parameter expecting synchronous function type
 6 | 
 7 | func bad() async {
 8 |     await mutex.withLock { state in
   |                          `- error: cannot pass function of type '(inout sending [String : Int]) async -> sending ()' to parameter expecting synchronous function type
 9 |         state["a"] = await load()
   |                      `- note: 'async' inferred from asynchronous operation used here
10 |     }
11 | }
```

Why:

```text
withLock expects a synchronous critical section.
await would introduce a suspension point.
Suspending while holding a lock is dangerous and not allowed by this API shape.
```

---

### 5.3 Production-style mutex cache

```swift
import Synchronization

public final class MemoryCache<Key: Hashable & Sendable, Value: Sendable>: Sendable {
    private struct State {
        var values: [Key: Value] = [:]
    }

    private let state = Mutex(State())

    public init() {}

    public func value(for key: Key) -> Value? {
        state.withLock { state in
            state.values[key]
        }
    }

    public func insert(_ value: Value, for key: Key) {
        state.withLock { state in
            state.values[key] = value
        }
    }

    public func removeAll() {
        state.withLock { state in
            state.values.removeAll()
        }
    }
}
```

This is reasonable because:

```text
The cache operations are synchronous.
The critical sections are small.
The protected state is private.
The public API does not expose the mutable dictionary.
The invariants are contained in the type.
```

---

## 6. Exercise

### Problem

Choose between a lock, an actor, and immutability for a shared cache used from UI and background work.

---

### Bad / naive version

```swift
final class ImageCache {
    static let shared = ImageCache()

    var storage: [URL: Data] = [:]

    func data(for url: URL) -> Data? {
        storage[url]
    }

    func insert(_ data: Data, for url: URL) {
        storage[url] = data
    }
}
```

### What is wrong?

```text
storage is mutable shared state.
UI and background tasks can access it concurrently.
The type is not Sendable-safe.
There is no ownership model.
Cache hit/miss/fetch logic can race.
The global singleton makes lifecycle and tests worse.
```

---

### Improved version: split the responsibilities

Use **immutability** for UI state:

```swift
struct ImageSnapshot: Sendable {
    let url: URL
    let data: Data
}
```

Use a **mutex** for tiny synchronous memory-cache operations:

```swift
import Synchronization

final class ImageMemoryCache: Sendable {
    private let state = Mutex([URL: Data]())

    func data(for url: URL) -> Data? {
        state.withLock { cache in
            cache[url]
        }
    }

    func insert(_ data: Data, for url: URL) {
        state.withLock { cache in
            cache[url] = data
        }
    }
}
```

Use an **actor** for async cache-miss behavior and in-flight request deduplication:

```swift
actor ImageRepository {
    private let memoryCache: ImageMemoryCache
    private var inFlight: [URL: Task<Data, Error>] = [:]

    init(memoryCache: ImageMemoryCache = ImageMemoryCache()) {
        self.memoryCache = memoryCache
    }

    func data(for url: URL) async throws -> Data {
        if let cached = memoryCache.data(for: url) {
            return cached
        }

        if let existingTask = inFlight[url] {
            return try await existingTask.value
        }

        let task = Task {
            try await fetchData(from: url)
        }

        inFlight[url] = task

        do {
            let data = try await task.value
            memoryCache.insert(data, for: url)
            inFlight[url] = nil
            return data
        } catch {
            inFlight[url] = nil
            throw error
        }
    }
}
```

Then isolate UI mutation to the main actor:

```swift
@MainActor
final class ImageViewModel: ObservableObject {
    enum State: Sendable {
        case idle
        case loading
        case loaded(ImageSnapshot)
        case failed(String)
    }

    @Published private(set) var state: State = .idle

    private let repository: ImageRepository

    init(repository: ImageRepository) {
        self.repository = repository
    }

    func load(url: URL) {
        state = .loading

        Task {
            do {
                let data = try await repository.data(for: url)
                state = .loaded(ImageSnapshot(url: url, data: data))
            } catch {
                state = .failed(error.localizedDescription)
            }
        }
    }
}
```

### Why this is better

```text
Immutable snapshot:
  safe to pass into UI state.

Mutex cache:
  fast synchronous access for simple memory storage.

Actor repository:
  owns async workflow and in-flight request coordination.

@MainActor view model:
  owns UI state mutation.
```

This is the staff-level answer: do not pick one primitive globally. Pick the smallest correct primitive for each layer.

---

## 7. Production guidance

Use **immutability** when:

```text
Data can be represented as a snapshot.
State changes can be modeled by replacing a value.
You want easy Sendable conformance.
You want simple testing and predictable UI updates.
```

Use an **actor** when:

```text
The state has a logical owner.
Operations are async.
You need cancellation-aware workflows.
You coordinate in-flight tasks.
You protect domain invariants across multiple fields.
You want compiler-enforced isolation.
```

Use a **mutex** when:

```text
The operation must be synchronous.
The protected state is small and private.
Critical sections are short.
You are implementing low-level infrastructure.
You are bridging callback/delegate state into Swift concurrency.
An actor would infect a large synchronous API with await for no real benefit.
```

Be careful when:

```text
The critical section does expensive work.
The lock protects multiple independent concerns.
Callers must perform check-then-act sequences.
The lock is exposed publicly.
You need cancellation, fairness, or priority-aware behavior.
You are tempted to call async APIs while holding the lock.
```

Avoid when:

```text
A value snapshot would remove sharing.
An actor would model ownership more clearly.
You cannot describe the invariant protected by the lock.
You need lock ordering across several locks.
You are using a lock to make a global singleton look safe.
```

Debugging checklist:

```text
What state is actually shared?
Who owns it?
Can it be immutable?
Is the invariant protected by one operation or split across calls?
Can any code suspend while holding synchronization?
Are critical sections small?
Are there multiple locks? If yes, what is the lock ordering?
Does the API expose protected mutable state?
Would an actor make the domain clearer?
Would a value snapshot eliminate synchronization?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Actors are safer than locks, so I would use actors.

This is incomplete. It ignores synchronous APIs, legacy code, low-level infrastructure, and critical-section costs.

### Senior answer

> Actors are good for async state ownership. Mutexes are good for small synchronous critical sections. I would avoid holding locks across expensive work and keep protected state private.

Good. This shows practical correctness.

### Staff-level answer

> I first try to remove shared mutable state with immutable snapshots. If mutation needs a domain owner or async workflow, I use an actor. If I only need a small synchronous critical section, I use `Synchronization.Mutex`. I do not lock “everything”; I design the API so invariants are protected by single operations, mutable state is not exposed, and async work does not happen inside the critical section.

That is the target answer.

Staff-level questions to ask:

```text
Can this state be a value snapshot instead of shared mutable state?
Who owns the mutation?
Is this operation naturally synchronous or async?
What invariant must be protected?
Can callers accidentally split check and mutation across calls?
What is the expected contention?
Can this code run on the main actor?
Can cancellation happen while work is in progress?
Is the synchronization primitive leaking into the public API?
How will this be tested deterministically?
```

---

## 9. Interview-ready summary

Actors and mutexes solve related but different problems. Actors isolate mutable state in an async concurrency domain and are best when the state has domain ownership or operations naturally suspend. `Synchronization.Mutex` protects shared mutable state through synchronous mutual exclusion and is appropriate for small, private, fast critical sections. The strongest design is often neither: immutable value snapshots remove synchronization entirely. The staff-level judgment is to model ownership first, then use actors or locks only where they match the shape of the problem.

---

## 10. Flashcards

Q: What is the main difference between an actor and a mutex?  
A: An actor provides async isolation and ownership of mutable state; a mutex provides synchronous mutual exclusion for a critical section.

Q: When is `Synchronization.Mutex` appropriate?  
A: When shared mutable state is small, private, synchronous, and protected by short critical sections.

Q: Why should you avoid `await` inside a lock?  
A: `await` introduces a suspension point. Holding a lock across suspension can cause deadlocks, thread-affinity bugs, priority issues, and invalid assumptions. Swift’s `Mutex.withLock` uses a synchronous closure to prevent this shape.

Q: What is the best synchronization primitive?  
A: Immutability, when possible. Immutable values do not need synchronization.

Q: What does a lock not solve?  
A: Ownership, lifecycle, cancellation, public API design, business invariants, lock ordering, or stale-state logic bugs.

Q: What is a check-then-act race?  
A: A bug where checking state and mutating state happen in separate synchronized operations, allowing another task/thread to change the state in between.

Q: Why can an actor still have logic races?  
A: Actor methods can suspend at `await`, allowing other actor-isolated work to interleave before the original method resumes.

Q: Why is exposing a locked dictionary still bad API design?  
A: It leaks mutable state and lets callers bypass or split invariants. The type should expose operations, not protected storage.

---

## 11. Related sections

- [[D4 — Actors and actor isolation]]
- [[D5 — Global actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and assumeIsolatedUntitled]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-aware design]]
- [[D13 — Swift 6 strict concurrency migration]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[F1 — Testing async and concurrent code]]

---

## 12. Sources

- Swift Senior/Staff Rubric, D12 section and study tier placement.
- Swift.org — “Announcing Swift 6”: Swift 6 introduced the `Synchronization` library with low-level concurrency APIs, including atomics and mutexes. ([Swift.org](https://swift.org/blog/announcing-swift-6/ "Announcing Swift 6 | Swift.org"))
- Swift Evolution SE-0433 — Synchronous Mutual Exclusion Lock: motivation, actor comparison, `Mutex` design, critical-section semantics, non-recursive warning, and async/await caveats. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md "swift-evolution/proposals/0433-mutex.md at main · swiftlang/swift-evolution · GitHub"))
- Apple Developer Documentation — `Synchronization.Mutex` and `withLock(_:)`. ([Apple Developer](https://developer.apple.com/documentation/Synchronization/Mutex?utm_source=chatgpt.com "Mutex | Apple Developer Documentation"))
- The Swift Programming Language — Concurrency chapter: actor isolation guarantee. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/?utm_source=chatgpt.com "Concurrency - Documentation | Swift.org"))