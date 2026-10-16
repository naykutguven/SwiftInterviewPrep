---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - detached-tasks
  - task-inheritance
  - main-actor
---

# D11 — Detached Tasks, Task Inheritance, and Isolation Loss

**Goal:** Give a throwing unstructured task an explicit error and cancellation owner.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Task {} and Task.detached {} both create unstructured work. Task {} inherits lexical actor context plus current task metadata where available. Detached work deliberately starts without that inheritance. Neither initializer automatically propagates creator cancellation like a structured child; each needs an owner for result, error, cancellation, and completion. [Unstructured tasks](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#unstructured-tasks)

## Rules and pitfalls

### Prefer an existing async call when lifetime is lexical

Use direct await or structured children when work belongs to the current operation. Retain an unstructured handle when a synchronous entry point or separately owned lifecycle requires one, and define how the owner joins it.

**Common pitfall:** Discarding a throwing handle leaves its error unobserved even if the compiler warning is silenced. Swift 6.4's warning is a prompt to define ownership, not an error-handling mechanism.

### Do not confuse lexical and dynamic context

A Task closure formed in a synchronous nonisolated helper does not inherit the MainActor merely because MainActor called that helper. A lexically nonisolated caller-isolated async helper likewise does not make every nested task actor-inheriting.

**Common pitfall:** Detached work is not a dedicated background thread and is rarely needed merely to offload CPU work. Prefer @concurrent for a current-task execution boundary, with explicit transferable inputs and cancellation checks.

## Interview questions

### Does explicitly discarding a throwing Task handle handle its error?

No. Explicit discard acknowledges that the handle is unused, but it neither awaits its error nor reports failure to a caller. Use an awaited handle, an owner that records the result, or a narrowly defined fire-and-forget policy that catches and handles errors inside the task.

### Why is `Task.detached` often a footgun?

It drops actor, task-local, and priority inheritance while keeping an independent lifetime that can outlive its creator. Captures need a valid transfer/sharing contract and errors still need an owner. Using it to silence isolation errors often exposes the same state without its protection.

### What does `Task {}` inherit that a detached task does not?

It inherits the lexical actor context and current priority/task-local context where the initializer's rules apply. Detached does not inherit those automatically. Effective priority can later escalate, so an observed priority is not proof of creation-time inheritance or execution ordering.

### Does Task {} in a synchronous nonisolated helper inherit the MainActor of its caller? What if its creator is already cancelled?

No: task actor inheritance is lexical, not determined by a synchronous caller's dynamic actor. Creator cancellation also does not automatically cancel either unstructured initializer's new task, including when the creator is already cancelled. An owner must explicitly forward cancellation when policy requires it.

## Focused exercise

**Task:** Replace an unowned throwing task with an awaited handle or explicit task owner. Check error delivery and state how cancellation reaches the operation.

**Acceptance checks:**

- The owner observes the deliberately thrown error.
- Explicit cancellation occurs before the gated cancellation check.
- Both tasks finish and are joined; no creator-cancellation inheritance is assumed.

**Solution:**

Reuse Gate from [[D2 — Task Hierarchy, Cancellation, and Priorities#Focused exercise]]. This executable owns both handles and joins their outcomes:

```swift
enum Failure: Error { case expected }
@main struct Probe {
    static func main() async {
        let failing = Task<Int, Error> { throw Failure.expected }
        do { _ = try await failing.value; preconditionFailure("expected error") }
        catch Failure.expected { }
        catch { preconditionFailure("unexpected error") }

        let entry = Gate()
        let cancellable = Task {
            await entry.wait()
            try Task.checkCancellation()
            return 7
        }
        cancellable.cancel()
        await entry.open()
        do { _ = try await cancellable.value; preconditionFailure("expected cancellation") }
        catch is CancellationError { }
        catch { preconditionFailure("unexpected error") }
    }
}
```

**Reasoning:** The gate establishes cancellation ordering without a scheduling race. Awaiting value gives error ownership, while cancel separately forwards a request to the chosen task. A longer-lived UI owner also needs request identity checks on success, error, and cleanup.

## In practice

**Choose it when:** Use an unstructured boundary only for a lifecycle that needs an explicit owner.

**Investigate by:** Trace lexical isolation, task-local inheritance, cancellation forwarding, and terminal result observation.

**Strong evidence:** Explain why the boundary is needed and verify late publication and teardown instead of equating a handle with complete lifecycle management.

### Throwing-task diagnostics and inheritance — optional

#### Swift 6.4 warns about discarded throwing tasks

```swift
enum Failure: Error { case work }
func work() async throws { throw Failure.work }
func start() {
    Task { try await work() } // expected NoUseUnstructuredThrowingTask warning
}
```

The compiler warns because the task's failure may be ignored. This diagnostic also applies to throwing detached and immediate task APIs; their individual deployment availability still matters, and immediate execution does not create structured ownership. Discarding the handle still does not stop the task or propagate its error to `start()`. Assigning the handle to `_` merely acknowledges the discard; it does not implement an error policy. [SE-0520](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0520-discardableresult-task-initializers.md)

```swift
enum Failure: Error { case work }
func work() async throws { throw Failure.work }
func run() async throws {
    let handle = Task { try await work() }
    try await handle.value
}
```

The second independent snippet compiles without that warning. If there is no need for a separately owned task, simply `try await work()` instead. At a synchronous UI boundary, retain the task and/or handle failure inside its body with a deliberate cancellation policy.

For scope-bound asynchronous teardown, use Swift 6.4 awaited `defer` (A6). Starting a detached cleanup task changes lifetime and context, so it is not a replacement for directly awaited cleanup. A cancellation shield is a separate choice with a newer-runtime requirement (D2).

#### Inheritance is lexical; cancellation still needs an owner

```swift
func startOutsideActor() -> Task<(any Actor)?, Never> {
    Task { #isolation }
}

@MainActor
func compareTaskIsolation() async -> [Bool] {
    let inherited = Task { #isolation }
    let detached = Task.detached { #isolation }
    let helper = startOutsideActor()
    let first = await inherited.value
    let second = await detached.value
    let third = await helper.value
    return [first === MainActor.shared, second == nil, third == nil]
}
```

This returns [true, true, true] in the checked settings: the closure directly inside the MainActor method inherits it; detached work and the explicitly nonisolated synchronous helper do not. An explicit actor annotation on a detached closure can choose an actor, but that is not inheritance and does not restore task-local context or structured lifetime.

Neither Task initializer automatically propagates cancellation from its creator, including creation after the creator is already cancelled. Awaiting a handle also does not create that relationship. A handle owner can forward cancellation explicitly or prefer structured child work. D2 covers cancellation shields; shielding observation is separate from ownership.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D13 — Swift 6 Strict Concurrency Migration]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [Unstructured tasks](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#unstructured-tasks)
- [SE-0520](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0520-discardableresult-task-initializers.md)
- [GitHub. "Structured Concurrency." Swift Evolution SE-0304](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Apple Developer. "Explore Structured Concurrency in Swift." WWDC21](https://developer.apple.com/videos/play/wwdc2021/10134/)
- [GitHub. "Global Actors." Swift Evolution SE-0316](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)
- [SE-0461 — Async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)
