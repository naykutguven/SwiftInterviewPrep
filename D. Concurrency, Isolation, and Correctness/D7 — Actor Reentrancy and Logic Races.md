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

**Goal:** Force out-of-order actor completions and prevent an older request from replacing current state.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Actor isolation serializes synchronous access, not an entire async method. At a suspension, another actor operation can change the state on which the first relied. When it resumes, its earlier assumptions may be stale. This logic race can occur in a program with no data race. Await marks potential suspension; a particular call need not actually pause. [Actor reentrancy](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md#actor-reentrancy)

## Rules and pitfalls

### Revalidate the invariant at publication

Assign a unique operation identity before suspension, then check it before publishing success, error, or cleanup. Alternatively reserve/commit state before suspension when that matches the domain. Moving work outside the actor does not remove the need to revalidate.

**Common pitfall:** Checking only the input string cannot distinguish repeated requests for the same input. Track the operation, not merely its parameters.

### Separate freshness, deduplication, and cancellation

First-result caching can still perform duplicate work. A retained shared producer can deduplicate but needs consumer-cancellation and invalidation policy. An older producer must not clear or repopulate a replacement's slot.

**Common pitfall:** Async deferred cleanup can itself suspend and then overwrite newer state. Guard cleanup ownership just as carefully as successful publication; see [[D4 — Actors and Actor Isolation]].

## Interview questions

### Can old deferred cleanup overwrite newer actor state while an async function is exiting? Why must that cleanup finish before the function returns?

Yes. An awaited defer can interleave with newer actor work before it resumes and writes. The enclosing function must still await that cleanup before returning under Swift 6.4. Tag cleanup with its operation identity and revalidate after its own suspension before touching shared state.

### What does actor reentrancy mean in practice?

One method can run a synchronous segment, suspend, allow another operation to run, and later resume. Isolated state remains protected against simultaneous access, but a multi-segment business operation is not indivisible. Design the state transition to tolerate that interleaving.

### How can an `await` in the middle of an actor method invalidate assumptions made earlier in the method?

Another operation can invalidate a cache, replace a request, change a balance, or update a revision while the first is suspended. Re-read or compare the relevant identity/version before committing; simply retaining a pre-await local does not keep its underlying assumption current.

## Focused exercise

**Task:** Gate two actor operations so they complete out of order. Check that an older completion cannot replace the selected current result.

**Acceptance checks:**

- The first request is known to have entered before the second starts.
- The second completes before the first is released.
- The late first result cannot replace the selected current result; both tasks are joined.

**Solution:**

Reuse the Gate actor from [[D2 — Task Hierarchy, Cancellation, and Priorities#Focused exercise]]. The rest is a standalone executable with that dependency; identities 1 and 2 are distinct operations in this fixture.

```swift
actor Latest {
    private var current = 0
    private var value: Int?
    func load(id: Int, started: Gate, release: Gate) async {
        current = id
        await started.open()
        await release.wait()
        guard current == id else { return }
        value = id
    }
    func snapshot() -> Int? { value }
}
@main struct Probe {
    static func main() async {
        let model = Latest()
        let firstStarted = Gate(), firstRelease = Gate()
        let secondStarted = Gate(), secondRelease = Gate()
        let first = Task { await model.load(id: 1, started: firstStarted, release: firstRelease) }
        await firstStarted.wait()
        let second = Task { await model.load(id: 2, started: secondStarted, release: secondRelease) }
        await secondStarted.wait()
        await secondRelease.open()
        await second.value
        await firstRelease.open()
        await first.value
        let result = await model.snapshot()
        precondition(result == 2)
    }
}
```

**Reasoning:** The gates establish the interleaving directly, with no scheduler-speed assumption. The guard restores the freshness invariant after suspension. This does not deduplicate requests or stop old work; those are separate resource policies.

## In practice

**Choose it when:** An async workflow reads actor state before awaiting external work.

**Investigate by:** Control suspension points and test late success, error, cancellation, and cleanup.

**Strong evidence:** State and validate independent freshness, sharing, invalidation, and task-ownership policies.

### Awaited cleanup interleaving — optional

#### Checked Swift 6.4 cleanup interleaving

This standalone -parse-as-library probe coordinates the exact interleaving with a one-use gate; it does not depend on a sleep. The gate is test support and has one waiter for each operation.

```swift
actor CleanupGate {
    private var paused = false
    private var arrival: CheckedContinuation<Void, Never>?
    private var release: CheckedContinuation<Void, Never>?

    func pause() async {
        await withCheckedContinuation { continuation in
            release = continuation
            paused = true
            arrival?.resume()
            arrival = nil
        }
    }
    func waitUntilPaused() async {
        if paused { return }
        await withCheckedContinuation { arrival = $0 }
    }
    func open() { release?.resume(); release = nil }
}

actor OperationOwner {
    private(set) var current: Int?
    func start(_ id: Int) { current = id }

    func finish(_ id: Int, gate: CleanupGate) async {
        guard current == id else { return }
        defer {
            await gate.pause()
            if current == id { current = nil }
        }
        print("finishing", id)
    }
}

@main struct Main {
    static func main() async {
        let owner = OperationOwner()
        let gate = CleanupGate()
        await owner.start(1)
        let cleanup = Task { await owner.finish(1, gate: gate) }
        await gate.waitUntilPaused()
        await owner.start(2)
        await gate.open()
        await cleanup.value
        print(await owner.current as Any)
    }
}
```

Output is finishing 1, then Optional(2). Removing the identity check after awaited teardown produces nil instead: cleanup from operation 1 erases operation 2. The gate only demonstrates ordering; real cleanup still needs error, cancellation, and termination policies. The plain async-defer syntax compiles for the older targets recorded in the validation record; it does not require a cancellation shield.

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
- [GitHub. "Actors." Swift Evolution SE-0306](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)
- [Apple Developer. "Protect Mutable State with Swift Actors." WWDC21](https://developer.apple.com/videos/play/wwdc2021/10133/)
- [Swift.org. "Concurrency." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [SE-0493 — Async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)
