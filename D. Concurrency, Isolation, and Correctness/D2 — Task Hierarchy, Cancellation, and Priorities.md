---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - structured-concurrency
  - task-hierarchy
  - priorities
---

# D2 — Task Hierarchy, Cancellation, and Priorities

**Goal:** Observe cancellation before and during gated work, then join its cleanup.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Cancellation marks task state; it does not preempt arbitrary instructions or roll back effects. Code cooperates by checking cancellation or calling a cancellation-aware operation. Structured children inherit the task tree's cancellation and task-local context, with priority rules that may escalate effective priority. Completion and cancellation request are different events. [Task cancellation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)

## Rules and pitfalls

### Define the owner and publication policy

An owner must cancel and join unstructured tasks. Awaiting a shared task's value does not automatically cancel that producer or stop waiting when one consumer is cancelled. Repeated UI requests need unique identity, including repeated requests for the same resource.

**Common pitfall:** Guarding only successful publication leaves stale errors and cleanup able to overwrite the current request. Check cancellation and request identity at every publication path.

### Specify time and bounded cancellation

Use an elapsed-time clock for duration budgets, not wall-clock dates that can change. ContinuousClock includes system sleep; SuspendingClock excludes time while the system sleeps. A timeout policy must define cancellation and completion of losing work.

**Common pitfall:** Choosing a timeout winner does not let structured children outlive the race scope. Cancelling the loser still requires joining cooperative completion, so return can exceed the nominal budget.

## Interview questions

### Does a shield uncancel the task, or change how the current context observes cancellation?

A shield changes how the current context observes cancellation and can block automatic cancellation propagation to children inside its scope. It does not clear the task's cancelled state permanently. After leaving, cancellation remains observable. Shield only bounded required cleanup and check runtime availability.

### Why does repeated cell configuration need a unique request identity, and why must stale error handling check cancellation too?

Query text or resource ID can repeat while an older operation is still finishing. A unique operation token distinguishes those requests. Error handling must check cancellation and token freshness too, because cancellation-insensitive services can throw ordinary errors after replacement.

### Why is cancellation in Swift cooperative rather than preemptive?

Preempting arbitrary code could interrupt invariant updates, resource ownership, or external effects at unsafe points. Cooperative checking lets an operation leave through a valid cleanup path. A noncooperating operation may therefore continue after cancel is called.

### What does a child task inherit from its parent?

Structured children participate in cancellation, priority, and task-local propagation. Actor execution follows the specific child closure/callee isolation rules rather than an unconditional same-actor promise. Task {} inherits contextual properties but is unstructured; detached work does not inherit the same context.

### When does a policy need elapsed time instead of a wall-clock date? How do ContinuousClock and SuspendingClock differ during system sleep?

Use elapsed time for debounce, retry delay, and timeout budgets. Wall time fits calendar deadlines but can jump. ContinuousClock continues through system sleep; SuspendingClock pauses for sleep. Choose which interval the product means and inject the time dependency in tests.

### If a timeout wins a structured race, who cancels and joins the losing work? Why can the function still return after its time budget?

The race's owner requests cancellation and the structured scope joins the loser. A blocking or cancellation-insensitive loser can delay return after the time budget. State this completion policy instead of promising a hard deadline that the structure cannot enforce.

## Focused exercise

**Task:** Cancel a gated operation before and during execution. Check its cancellation observation, completion path, and cleanup without elapsed-time sleeps.

**Acceptance checks:**

- Cancellation before work entry and after its started acknowledgment both reach CancellationError.
- The test releases every wait it enters and joins the owned task.
- Awaited cleanup finishes once before task completion is observed; no elapsed-time sleep orders events.

**Solution:**

Standalone deterministic harness. Gate intentionally ignores cancellation so the test controls when the operation reaches its next check; it is not a production cancellable primitive.

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
actor Cleanup {
    private(set) var count = 0
    func record() { count += 1 }
}
func work(started: Gate, release: Gate, cleanup: Cleanup) async throws {
    defer { await cleanup.record() }
    try Task.checkCancellation()
    await started.open()
    await release.wait()
    try Task.checkCancellation()
}
@main struct Probe {
    static func main() async {
        for cancelBeforeEntry in [true, false] {
            let entry = Gate(), started = Gate(), release = Gate()
            let cleanup = Cleanup()
            let task = Task {
                await entry.wait()
                try await work(started: started, release: release, cleanup: cleanup)
            }
            if cancelBeforeEntry { task.cancel() }
            await entry.open()
            if !cancelBeforeEntry {
                await started.wait()
                task.cancel()
                await release.open()
            }
            do { try await task.value; preconditionFailure("expected cancellation") }
            catch is CancellationError { }
            catch { preconditionFailure("unexpected error") }
            let count = await cleanup.count
            precondition(count == 1)
        }
    }
}
```

**Reasoning:** Each gate establishes a causal boundary. The before-entry case never needs the release gate because its first cancellation check throws. The during-work case controls resumption, then observes cancellation and joins cleanup. This proves those paths, not prompt cancellation of arbitrary services.

## In practice

**Choose it when:** Cancellation and timeout behavior are part of an operation's public contract.

**Investigate by:** Control entry, suspension, completion, and cleanup separately.

**Strong evidence:** Combine task ownership, request freshness, chosen clock semantics, and bounded teardown under ordinary failure and cancellation.

### Cancellation shielding contract — optional

#### Cancellation shields in Swift 6.4

`withTaskCancellationShield` temporarily masks current-context cancellation observation. It does not clear the task's cancelled state. The installed SDK requires `anyAppleOS 27.0` for both synchronous and asynchronous forms. Keep shielded teardown small: it can still fail or hang for reasons unrelated to cancellation.

| Observation | Inside shield | After shield |
| --- | --- | --- |
| `Task.isCancelled` after cancellation | `false` | `true` |
| `Task.checkCancellation()` | Does not throw because of that cancellation | Throws |
| A task handle's `isCancelled` | Actual state, still `true` | Still `true` |

The proposal protects handlers installed inside a shield. Do not promise that an outer handler will fire while an inner shield is active: in the recorded build, a controlled nested-handler probe suppressed both, for both self-cancellation and cancellation through an external Task handle. The unshielded control invoked both handlers. Treat this as an observed boundary requiring validation on the shipping runtime, not a universal contract for every nesting pattern. Automatic parent cancellation into enclosed child work is shielded, but `group.cancelAll()` and cancellation of unawaited `async let` children at scope exit retain their behavior. Merely shielding `group.addTask` does not shield the child body's lifetime. [SE-0504](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md)

**Migration:** the underlying cancellation remains permanent. Older descriptions saying every later *static* `Task.isCancelled` read must remain true now need this scoped exception. Awaited defer is useful for teardown; it does not provide shielding by itself.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]]
- [[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

### Official sources

- [Task cancellation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [SE-0504](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md)
- [Apple Developer. "TaskGroup." Apple Developer Documentation](https://developer.apple.com/documentation/swift/taskgroup)
- [Apple Developer. "detached(name:priority:operation:)." Apple Developer Documentation](https://developer.apple.com/documentation/swift/task/detached%28name%3apriority%3aoperation%3a%29-795w1)
- [Apple Developer. "Beyond the Basics of Structured Concurrency." WWDC23](https://developer.apple.com/videos/play/wwdc2023/10170/)
- [GitHub. "Structured Concurrency." Swift Evolution SE-0304](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [GitHub. "Task-Local Values." Swift Evolution SE-0311](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0311-task-locals.md)
