---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actors
  - actor-reentrancy
  - logic-races
---

# D7 — Actor Reentrancy and Logic Races

**Learning goals:**

- Trace the synchronous segments of an actor method around actual suspension.
- Protect publication and cleanup against stale assumptions, including repeated inputs.
- Separate freshness, shared producers, invalidation, and cancellation policies.

**Prerequisites:** [[D4 — Actors and Actor Isolation]] for actor ownership and [[D2 — Task Hierarchy, Cancellation, and Priorities]] for identity/cancellation. The gates below are test support, not production cancellable waits.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A data race is unsynchronized access to memory. A **logic race** is a wrong result caused by the order of otherwise safe operations. An actor can prevent the first while permitting the second.

**Reentrancy** means another actor operation may run while an async method is suspended. For example, request A reads the current selection, awaits a service, and later publishes. Request B can change the selection during that wait. A's old local value has not changed, but its assumption about which selection is current is no longer valid. `await` marks possible suspension; it does not require every call to suspend. [SE-0306 — reentrancy](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md#actor-reentrancy)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Read, await, and commit are different stages

**Setup:** Standalone **compile-only** actor. The second availability check is essential.

```swift
actor Seat {
    private var available = true
    func reserve(approved: @Sendable () async -> Bool) async -> Bool {
        guard available else { return false }
        let allowed = await approved()
        guard allowed, available else { return false }
        available = false
        return true
    }
}
```

**Walkthrough:** Two callers may both reach approval after observing availability. Only one can pass the post-await check and reserve the seat. Checking and setting happen in one synchronous segment. Reserving before approval with explicit rollback would be a different domain policy.

### Shared setup — gate

A gate acknowledges arrival or controls release. Opening before waiting is safe. It intentionally ignores cancellation so the test driver must release every wait it enters.

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

### Freshness does not deduplicate work

**Setup:** Standalone **compile-only** teaching cache without an invalidation API.

```swift
actor SharedValue {
    private var cached: String?
    private var inFlight: Task<String, Error>?
    func value(load: @escaping @Sendable () async throws -> String) async throws -> String {
        if let cached { return cached }
        if let inFlight { return try await inFlight.value }
        let producer = Task { try await load() }
        inFlight = producer
        defer { inFlight = nil }
        let value = try await producer.value
        cached = value
        return value
    }
}
```

**Walkthrough:** Retaining the task before awaiting lets later callers join the same producer. Merely allowing only the first completion to fill a cache would not stop duplicate requests. This small version has no invalidation, and a cancelled waiter waits for shared completion. D4's TokenStore adds producer identity and invalidation; adding a reset method here without those checks would create a new race.

## Rules and pitfalls

### Track the invocation, not just its input

**Common pitfall:** Comparing query strings is expected to distinguish overlapping requests when the sequence is A → B → A.

**Mistake:** Standalone **compile-only** actor.

```swift
actor Results {
    var query = ""
    var value: String?
    func load(_ query: String, using service: @Sendable () async -> String) async {
        self.query = query
        let result = await service()
        guard self.query == query else { return }
        value = result
    }
}
```

**Consequence:** The first A may overwrite the latest A, because both have equal query text.

**Why:** The resource name is not the operation's identity. The actor can safely execute a stale decision.

**Correction:** **Shared setup — latest results.** Standalone **compile-only** actor; prepend this named setup where used below. It also protects error publication and cleanup.

```swift
import Foundation
actor Latest {
    private var current: UUID?
    private(set) var value: String?
    private(set) var error: String?
    func load(_ query: String,
              using service: @Sendable (String) async throws -> String) async {
        let id = UUID(); current = id
        defer { if current == id { current = nil } }
        do {
            let loaded = try await service(query)
            guard !Task.isCancelled, current == id else { return }
            value = loaded; error = nil
        } catch {
            guard !Task.isCancelled, current == id else { return }
            self.error = "Failed"
        }
    }
}
```

**What changes:** Each invocation has a fresh token. It must still own the slot when it publishes. Cancellation suppresses a current result even without replacement, while stale cleanup cannot erase a newer token.

### Cleanup must revalidate after its own suspension

**Common pitfall:** A check before entering async deferred cleanup is assumed to remain true after teardown awaits.

**Mistake:** Standalone **compile-only** Swift 6.4 actor.

```swift
actor Owner {
    var current: Int?
    func finish(_ id: Int, teardown: @Sendable () async -> Void) async {
        guard current == id else { return }
        defer {
            await teardown()
            current = nil
        }
        return
    }
}
```

**Consequence:** While teardown is suspended, a new operation can replace current. The old defer then clears that new state.

**Why:** A defer describes when cleanup runs; it does not make its async body indivisible.

**Correction:** Standalone **compile-only** actor.

```swift
actor Owner {
    var current: Int?
    func finish(_ id: Int, teardown: @Sendable () async -> Void) async {
        guard current == id else { return }
        defer {
            await teardown()
            if current == id { current = nil }
        }
        return
    }
}
```

**What changes:** The post-teardown check protects the actual write. Both versions still wait for deferred cleanup before returning.

## Worked example

### Force the newer request to finish first

**Setup:** Prepend **gate** and **latest results**. Runnable.

**Predict:** Which value remains after the old request for the same query finishes last?

```swift
let model = Latest()
let oldStarted = Gate(), oldRelease = Gate()
let old = Task {
    await model.load("A") { _ in
        await oldStarted.open(); await oldRelease.wait(); return "old A"
    }
}
await oldStarted.wait()
await model.load("A") { _ in "new A" }
await oldRelease.open()
await old.value
let result = await model.value
precondition(result == "new A")
```

**Outcome:** `new A` remains selected.

**Why:** The acknowledgment proves the old operation entered before the new one. The new call completes before the driver releases the old service. The old token cannot match the current slot, regardless of equal query text. No assertion depends on a scheduler delay.

## Interview questions

### What does actor reentrancy mean in practice?

**Answer:** An async actor method runs in segments. Other operations can access the actor between segments when the first method suspends, so the entire method is not one transaction.

**Example:** Standalone **compile-only** actor.

```swift
actor Revision {
    private var revision = 0
    func bump() { revision += 1 }
    func unchanged(after pause: @Sendable () async -> Void) async -> Bool {
        let before = revision
        await pause()
        return revision == before
    }
}
```

**Walkthrough:** `bump()` may run while `pause()` is unfinished. On resumption the comparison can be false, even though each access to revision was protected.

**Boundary:** Do not infer an interleaving merely because await appears in source; use a gate to force it in a regression test.

### How can an await in the middle of an actor method invalidate assumptions made earlier in the method?

**Answer:** Another operation can change the state on which the pending operation relied. Re-read, compare a version, or reserve state before suspension according to the domain.

**Example:** Standalone **compile-only** version-checked commit.

```swift
actor Draft {
    private var revision = 0
    private(set) var text = ""
    func edit(_ text: String) { self.text = text; revision += 1 }
    func suggest(using service: @Sendable (String) async -> String) async {
        let observed = revision
        let suggestion = await service(text)
        guard revision == observed else { return }
        text = suggestion
        revision += 1
    }
}
```

**Walkthrough:** A user edit during service execution increments the revision. The old suggestion then fails the comparison and cannot overwrite the edit. The check and update contain no suspension.

**Boundary:** A version check protects freshness, not resource use. It does not cancel old service calls, deduplicate them, or define persistence behavior.

### Extension — Can old deferred cleanup overwrite newer actor state while an async function is exiting? Why must cleanup finish before return?

**Answer:** Yes: awaited cleanup can suspend and later resume against newer state. Swift 6.4 still waits for that defer before the enclosing function returns.

**Example:** Prepend **gate**. Runnable controlled cleanup.

```swift
actor OperationOwner {
    private(set) var current: Int?
    func start(_ id: Int) { current = id }
    func finish(_ id: Int, entered: Gate, release: Gate) async {
        guard current == id else { return }
        defer {
            await entered.open()
            await release.wait()
            if current == id { current = nil }
        }
        return
    }
}
let owner = OperationOwner()
let entered = Gate(), release = Gate()
await owner.start(1)
let cleanup = Task { await owner.finish(1, entered: entered, release: release) }
await entered.wait()
await owner.start(2)
await release.open()
await cleanup.value
let current = await owner.current
precondition(current == 2)
```

**Walkthrough:** Operation 1 enters its defer and pauses. Operation 2 becomes current. Resuming cleanup compares identity again and retains 2. Joining `cleanup.value` proves deferred work finished before the caller proceeds. Removing that final comparison erases operation 2. [SE-0493](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)

**Boundary:** This test helper only establishes ordering. Real teardown still needs error, cancellation, and termination policy; an await does not install a cancellation shield.

## Focused exercise

**Task:** Force old actor work to finish after a newer selected request, for both late success and ordinary failure.

**Acceptance checks:**

- Old work is known to have started before replacement.
- Replacement completes before old work is released.
- Neither late success nor error replaces current state; both task paths finish.

**Solution:** Prepend **gate** and **latest results**. Runnable.

```swift
enum ServiceError: Error { case offline }
for fails in [false, true] {
    let model = Latest()
    let entered = Gate(), release = Gate()
    let old = Task {
        await model.load("A") { _ in
            await entered.open(); await release.wait()
            if fails { throw ServiceError.offline }
            return "old A"
        }
    }
    await entered.wait()
    await model.load("A") { _ in "new A" }
    await release.open()
    await old.value
    let value = await model.value
    let error = await model.error
    precondition(value == "new A" && error == nil)
}
```

**Reasoning:** The driver selects the interleaving through acknowledgments. Equal input text makes a string-only guard insufficient. The identity checks protect both outcomes; old work is joined rather than abandoned.

## In practice

**Choose it when:** An actor reads a mutable assumption before awaiting external work or cleanup.

**Investigate by:** Mark every commit after suspension and test stale success, error, invalidation, and cleanup independently.

**Strong evidence:** Describe freshness, producer sharing, cancellation, and invalidation as separate policies and prove the required interleavings.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]]
- [[D4 — Actors and Actor Isolation]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [Actor reentrancy](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md#actor-reentrancy)
- [Apple Developer. "Protect Mutable State with Swift Actors." WWDC21](https://developer.apple.com/videos/play/wwdc2021/10133/)
- [Swift.org. "Concurrency." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [SE-0493 — Async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)
