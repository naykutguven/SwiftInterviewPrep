---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actors
  - actor-isolation
---

# D4 — Actors and Actor Isolation

**Learning goals:**

- Put shared mutable state behind an actor and keep a complete state transition within one uninterrupted operation.
- Distinguish data-race protection from correctness across suspension, invalidation, and shared work.
- Define who owns a shared producer and which operation may publish or clear its result.

**Prerequisites:** [[D1 — Structured Concurrency Fundamentals]] for tasks and joining; [[D2 — Task Hierarchy, Cancellation, and Priorities]] for cancellation and request identity.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An actor is a reference type that owns an **isolation domain**: the state that only operations running with that actor's permission may access. Imagine a counter used by several requests. Each caller asks the actor to increment; it does not take a reference to the actor's mutable integer and change it elsewhere.

A synchronous actor method can read, check, and update its state without another actor operation executing in the middle. At an actual suspension, another operation may run before the first resumes. The protection is against simultaneous unsafe memory access, not against every incorrect ordering of business events. An actor is not a dedicated thread or a promise of first-in, first-out ordering among independent callers. [SE-0306 — actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Shared setup — counter

This actor has one mutable integer. `increment()` owns the whole read-modify-write. Prepend **counter** only where named.

```swift
actor Counter {
    private var count = 0
    func increment() { count += 1 }
    func snapshot() -> Int { count }
}
```

### Cross the boundary to access live state

**Setup:** Prepend **counter**. Runnable.

```swift
let counter = Counter()
await counter.increment()
let value = await counter.snapshot()
precondition(value == 1)
```

**Walkthrough:** Each await authorizes an actor call. The increment finishes before the snapshot begins, so the value is one. The integer itself never escapes as mutable shared storage. A snapshot is an independent value; it describes one moment, not a continuously updated view.

## Rules and pitfalls

### Keep the whole invariant in one operation

**Common pitfall:** Separately protected getters and setters are assumed to make a compound update atomic.

**Mistake:** Standalone **compile-only** declaration. Two callers can read the same old value.

```swift
actor SplitCounter {
    private var count = 0
    func read() -> Int { count }
    func write(_ value: Int) { count = value }
}
func increment(_ counter: SplitCounter) async {
    let old = await counter.read()
    await counter.write(old + 1)
}
```

**Consequence:** Both callers may read zero and later write one, losing an increment even though each access is isolated.

**Why:** The actor call boundary separates reading from writing. Actor isolation does not combine two requests into one operation.

**Correction:** Prepend **counter**, whose `increment()` contains both accesses. Runnable.

```swift
let counter = Counter()
async let first: Void = counter.increment()
async let second: Void = counter.increment()
_ = await (first, second)
let value = await counter.snapshot()
precondition(value == 2)
```

**What changes:** No suspension separates the read and write within an increment. Both updates contribute, without requiring a particular caller order.

### Old cleanup must not clear a replacement

**Common pitfall:** An awaited operation is followed by cleanup that assumes it still owns the current request slot.

**Mistake:** Standalone **compile-only** actor. The injected operation may suspend.

```swift
import Foundation
actor RefreshSlot {
    var current: UUID?
    func refresh(using operation: @Sendable () async -> Void) async {
        current = UUID()
        defer { current = nil }
        await operation()
    }
}
```

**Consequence:** A finishes after B has started and clears B's slot. B's state is lost even though neither operation accessed memory simultaneously.

**Why:** The `defer` belongs to A, but the mutable property may belong to a newer request after suspension.

**Correction:** Standalone **compile-only** version. Compare the invocation's token before cleanup.

```swift
import Foundation
actor RefreshSlot {
    var current: UUID?
    func refresh(using operation: @Sendable () async -> Void) async {
        let request = UUID()
        current = request
        defer {
            if current == request { current = nil }
        }
        await operation()
    }
}
```

**What changes:** A can finish its own method without clearing B's identity. If cleanup itself awaits, compare again after that await; D7 demonstrates that extra boundary.

## Worked example

### Share one token refresh and reject invalidated results

**Shared setup — token:** Prepend this value to **TokenStore** below. Validity uses wall time because it represents a server-issued expiry date, not a duration budget.

```swift
import Foundation
struct Token: Sendable {
    let rawValue: String
    let expiresAt: Date
    var isValid: Bool { expiresAt > .now }
}
```

**Setup:** Prepend **token**. **Compile-only** actor implementation; the driver below exercises it. A retained task is a shared producer; each caller is a waiter.

**Predict:** Can an old refresh repopulate the cache after logout has invalidated it?

```swift
enum RefreshError: Error { case superseded }

actor TokenStore {
    private enum State {
        case empty
        case valid(id: UUID, token: Token)
        case refreshing(id: UUID, task: Task<Token, any Error>)
    }

    private var state: State = .empty
    private let refresh: @Sendable () async throws -> Token

    init(refresh: @escaping @Sendable () async throws -> Token) {
        self.refresh = refresh
    }

    func validToken() async throws -> Token {
        try Task.checkCancellation()
        let id: UUID
        let task: Task<Token, any Error>
        switch state {
        case .valid(_, let token) where token.isValid:
            return token
        case .refreshing(let currentID, let currentTask):
            id = currentID
            task = currentTask
        case .empty, .valid:
            id = UUID()
            task = Task { [refresh] in try await refresh() }
            state = .refreshing(id: id, task: task)
        }

        let refreshed: Token
        do {
            refreshed = try await task.value
        } catch {
            if case .refreshing(let currentID, _) = state, currentID == id {
                state = .empty
            }
            throw error
        }

        switch state {
        case .refreshing(let currentID, _) where currentID == id:
            state = .valid(id: id, token: refreshed)
        case .valid(let currentID, _) where currentID == id:
            break // another waiter already committed this producer's result
        default:
            throw RefreshError.superseded
        }
        // A cancelled waiter does not invalidate a successful shared result.
        try Task.checkCancellation()
        return refreshed
    }

    func invalidate() {
        if case .refreshing(_, let task) = state { task.cancel() }
        state = .empty
    }
}
```

**Outcome:** Invalidation clears local state and asks the producer to cancel. If that producer ignores cancellation and later succeeds, its ID no longer matches and the old waiter receives `superseded`. A late failure clears the slot only if that producer still owns it.

**Why:** The actor stores three states: no token, a refresh with its ID/task, or a valid token with the ID that produced it. Waiters reuse the same task. After suspension, a waiter either commits that producer's result, recognizes that another waiter already committed it, or discovers replacement/invalidation. The ID check applies to successful publication and failure cleanup.

The successful result can be cached even when the original waiter is cancelled, because other waiters may still need it. Waiter cancellation is checked before entry and after completion; it does not promptly interrupt `task.value` or cancel shared work. Invalidation does not revoke returned tokens, undo server rotation, or prove the network request has stopped. Expiry validation of new responses, persistence, retry/backoff, and prompt per-waiter cancellation need domain policy.

**Shared setup — gate:** A test-only one-way door; it deliberately ignores cancellation. Prepend it to the next driver.

```swift
actor Gate {
    private var opened = false
    private var waiters: [CheckedContinuation<Void, Never>] = []
    func wait() async {
        if opened { return }
        await withCheckedContinuation { waiters.append($0) }
    }
    func open() {
        opened = true
        let pending = waiters
        waiters.removeAll()
        for waiter in pending { waiter.resume() }
    }
}
```

**Driver:** Prepend **token**, **TokenStore**, and **gate**. Runnable. The old service completes successfully despite invalidation; no delay guesses establish ordering.

```swift
actor Provider {
    let started: Gate
    let release: Gate
    private var calls = 0
    init(started: Gate, release: Gate) { self.started = started; self.release = release }
    func refresh() async -> Token {
        calls += 1
        let call = calls
        if call == 1 { await started.open(); await release.wait() }
        return Token(rawValue: "token-\(call)", expiresAt: .distantFuture)
    }
}
let started = Gate(), release = Gate()
let provider = Provider(started: started, release: release)
let store = TokenStore { await provider.refresh() }
let old = Task { try await store.validToken() }
await started.wait()
await store.invalidate()
let current = try await store.validToken()
precondition(current.rawValue == "token-2")
await release.open()
do { _ = try await old.value; preconditionFailure("expected superseded result") }
catch RefreshError.superseded { }
let cached = try await store.validToken()
precondition(cached.rawValue == "token-2")
```

## Interview questions

### Which state is isolated, and which invariants can become stale at an await or async defer?

**Answer:** An actor's instance state belongs to that actor. An invariant based on mutable state can become stale when an operation suspends, including during awaited cleanup. Independent stored snapshots remain values, but their relevance can change.

**Example:** Standalone **compile-only** actor. This version rechecks availability after awaiting approval.

```swift
actor Inventory {
    private var available = 1
    func reserve(approval: @Sendable () async -> Bool) async -> Bool {
        guard available > 0 else { return false }
        let approved = await approval()
        guard approved, available > 0 else { return false }
        available -= 1
        return true
    }
}
```

**Walkthrough:** Another reservation may use the last item while approval is pending. The second check and decrement occur together after resumption, so this method cannot make availability negative.

**Boundary:** Holding a local `available` value across the await would not reserve stock. A different domain might reserve before awaiting and compensate on failure. A `Sendable` value can also have actor-isolated computed members; sendability does not authorize calling those members from any context. See D5 and D14.

### Can an older refresh repopulate the cache after invalidation, or clear a newer in-flight operation on failure?

**Answer:** Yes, unless publication and cleanup compare the producer's identity with the current state. Actor serialization alone does not decide which refresh remains authoritative.

**Example:** Standalone runnable model of that decision.

```swift
import Foundation
actor RefreshOwner {
    private(set) var current: UUID?
    private(set) var cached: String?
    func begin() -> UUID { let id = UUID(); current = id; return id }
    func invalidate() { current = nil; cached = nil }
    func commit(_ value: String, from id: UUID) {
        guard current == id else { return }
        cached = value
    }
    func failed(_ id: UUID) { if current == id { current = nil } }
}
let owner = RefreshOwner()
let old = await owner.begin()
await owner.invalidate()
let latest = await owner.begin()
await owner.commit("old token", from: old)
await owner.failed(old)
let active = await owner.current
let cached = await owner.cached
precondition(active == latest && cached == nil)
```

**Walkthrough:** Invalidation removes A's authority. B gets a new identity. A's late success cannot cache a value, and A's late failure cannot clear B. The full TokenStore also owns the shared producer handle and propagates outcomes to waiters.

**Boundary:** One waiter's cancellation must not automatically cancel a producer shared by other waiters. Identity, producer cancellation, and prompt cancellation of a waiter are separate policies.

## Focused exercise

**Task:** Put a shared counter behind an actor, make 100 concurrent increments, and explain the protected invariant.

**Acceptance checks:**

- After the group joins, the snapshot is exactly 100.
- The actor's read-modify-write contains no suspension.
- No result depends on caller order or a dedicated actor thread.

**Solution:** Prepend **counter**. Runnable.

```swift
let counter = Counter()
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 { group.addTask { await counter.increment() } }
}
let result = await counter.snapshot()
precondition(result == 100)
```

**Reasoning:** The group is the completion boundary; the actor is the state owner. Each synchronous increment contributes once. This bounded fixture cannot overflow; production counters need their own range policy.

## In practice

**Choose it when:** A single owner can express the invariant and callers already use asynchronous boundaries.

**Investigate by:** Mark state reads, suspension points, publication, and cleanup separately. Test late success and late failure, not just simultaneous increments.

**Strong evidence:** Explain shared-producer ownership and invalidation without confusing memory safety with workflow correctness.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[D13 — Swift 6 Strict Concurrency Migration]]

### Official sources

- [Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)
- [Swift.org. "Concurrency." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [Swift.org. "Calling an Actor-Isolated Method from a Synchronous Nonisolated Context." Swift Compiler Diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/)
- [SE-0493 — Async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)
- [SE-0466 — Control default actor isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
