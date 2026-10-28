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

**Learning goals:**

- Distinguish ordinary and detached unstructured tasks from structured children.
- Predict lexical actor and task-local inheritance without assuming cancellation propagation.
- Assign error, cancellation, joining, and publication ownership at every unstructured boundary.

**Prerequisites:** [[D1 — Structured Concurrency Fundamentals]]–[[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]] for structured lifetime, [[D5 — Global Actors and `@MainActor`]] for MainActor, and [[D6 — `Sendable`, `@Sendable`, and `sending`]] for transfer/capture contracts.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Both `Task {}` and `Task.detached {}` create **unstructured** work: the creating function does not automatically join it when returning. A handle provides a way to observe its result, request cancellation, and wait for completion. Keeping a handle is useful only when someone actually follows that ownership policy.

An ordinary Task inherits actor context from where its closure is formed, plus current task-local metadata and priority under its initializer's rules. Detached work deliberately starts without those inherited properties. Neither initializer automatically propagates creator cancellation, including creation after its creator was cancelled. [SE-0304 — unstructured tasks](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#unstructured-tasks)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Observe the handle's result

**Setup:** Standalone runnable failure path.

```swift
enum Failure: Error { case offline }
let request = Task<String, Error> { throw Failure.offline }
do { _ = try await request.value; preconditionFailure("expected failure") }
catch Failure.offline { }
```

**Walkthrough:** Awaiting the value joins the task and observes its error. Creating it or discarding its handle would not make that error reach the original function's caller.

### Task-local inheritance is separate from lifetime

**Setup:** Standalone runnable comparison.

```swift
enum Trace { @TaskLocal static var id = "unbound" }
await Trace.$id.withValue("profile") {
    let ordinary = Task { Trace.id }
    let detached = Task.detached { Trace.id }
    let first = await ordinary.value
    let second = await detached.value
    precondition(first == "profile" && second == "unbound")
}
```

**Walkthrough:** Ordinary creation inherits the binding; detached creation does not. Both handles still need explicit joining. Effective priorities can later escalate, so numeric observations cannot prove creation-time inheritance or ordering.

### Shared setup — gate

This test door ignores cancellation so the driver can control exactly when a cancellation check runs. Prepend **gate** where named.

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

## Rules and pitfalls

### Discarding a throwing handle does not handle failure

**Common pitfall:** Ignoring the returned handle is expected to make task failure part of the caller's error flow.

**Mistake:** Standalone **compile-only, expected warning** under Swift 6.4.

```swift
enum Failure: Error { case offline }
func work() async throws { throw Failure.offline }
func start() { Task { try await work() } }
```

**Consequence:** The synchronous caller can finish without observing the task's error. Swift 6.4's unused throwing-task warning points out this missing policy.

**Why:** An unstructured task stores its result in its own handle. It does not throw into the function that created it.

**Correction:** Standalone runnable joined handle.

```swift
enum Failure: Error { case offline }
func work() async throws { throw Failure.offline }
func run() async throws {
    let handle = Task { try await work() }
    try await handle.value
}
do { try await run(); preconditionFailure("expected failure") }
catch Failure.offline { }
```

**What changes:** The caller observes the failure through `run`. If no separate task lifetime is required, `try await work()` is simpler. A synchronous UI boundary can instead retain an owned task and explicitly handle outcomes and shutdown.

### Detached does not authorize UI access

**Common pitfall:** Detached work is used to escape an isolation diagnostic while continuing to mutate the same UI model.

**Mistake:** Standalone **intentionally rejected** rubric probe.

```swift
@MainActor final class ViewModel {
    var title = ""
    func load() {
        _ = Task.detached { self.title = "Done" }
    }
}
```

**Consequence:** The detached closure lacks permission to mutate the MainActor property.

**Why:** Detachment removes inherited context; it does not make the referenced state safe outside that context.

**Correction:** Standalone runnable actor-inheriting boundary, with a handle returned to its owner.

```swift
@MainActor final class ViewModel {
    var title = ""
    func load() -> Task<Void, Never> { Task { title = "Done" } }
}
let model = ViewModel()
let handle = model.load()
await handle.value
precondition(model.title == "Done")
```

**What changes:** The closure is formed in a MainActor method, and the caller joins the handle. For expensive independent CPU work, prefer an explicit `@concurrent` helper returning transferable data to the UI owner; a detached task is not a dedicated background thread.

## Worked example

### Waiting does not create cancellation inheritance

**Setup:** Prepend **gate**. Runnable.

**Predict:** Does cancelling a task awaiting another unstructured task cancel that producer?

```swift
let release = Gate(), waiting = Gate()
let producer = Task { await release.wait(); return 7 }
let consumer = Task { await waiting.open(); return await producer.value }
await waiting.wait()
consumer.cancel()
precondition(!producer.isCancelled)
await release.open()
let value = await consumer.value
precondition(value == 7)
```

**Outcome:** The producer remains uncancelled and its value can still reach the consumer.

**Why:** Awaiting a handle creates a dependency for completion, not a structured cancellation edge. A shared producer's owner may intentionally let it continue; a dedicated child-like producer may instead need explicit forwarding.

## Interview questions

### Why is Task.detached often a footgun?

**Answer:** It drops inherited actor/task-local/priority context while retaining an independent lifetime. Captures and errors still need valid ownership, and work can outlive the initiating operation.

**Example:** Standalone runnable current-task alternative for CPU preparation.

```swift
@concurrent func prepare(_ values: [Int]) async throws -> [Int] {
    try Task.checkCancellation()
    let prepared = values.sorted()
    try Task.checkCancellation()
    return prepared
}
@MainActor func load() async throws -> [Int] { try await prepare([3, 1, 2]) }
let values = try await load()
precondition(values == [1, 2, 3])
```

**Walkthrough:** Execution leaves MainActor but remains in the same task, with cancellation and task-local context still part of that task. The result returns through an awaited call.

**Boundary:** Sorting does not poll cancellation internally. A truly independent long-lived task can be valid, but its result, cancellation, and completion still need a separate owner.

### What does Task {} inherit that a detached task does not?

**Answer:** It inherits the lexical actor context and current task-local/priority metadata under its rules. Detached does not automatically inherit those properties.

**Example:** Standalone runnable actor comparison.

```swift
@MainActor func compare() async -> (Bool, Bool) {
    let ordinary = Task { let actor: (any Actor)? = #isolation; return actor === MainActor.shared }
    let detached = Task.detached { let actor: (any Actor)? = #isolation; return actor == nil }
    return await (ordinary.value, detached.value)
}
let result = await compare()
precondition(result.0 && result.1)
```

**Walkthrough:** The directly formed ordinary closure inherits MainActor. The detached closure has no actor. These are isolation observations, not physical thread checks.

**Boundary:** Explicitly annotating a detached closure with MainActor chooses an actor but does not restore inherited task-local metadata or structured lifetime.

### Does Task {} in a synchronous nonisolated helper inherit MainActor from its caller? What if the creator is already cancelled?

**Answer:** Actor inheritance is lexical, so the synchronous helper's Task does not inherit its caller's dynamic MainActor. Creator cancellation also does not automatically cancel either unstructured initializer's new task.

**Example:** Standalone runnable helper comparison.

```swift
func startOutside() -> Task<Bool, Never> {
    Task { let actor: (any Actor)? = #isolation; return actor == nil }
}
@MainActor func check() async -> Bool { await startOutside().value }
let outside = await check()
precondition(outside)
let creator = Task {
    withUnsafeCurrentTask { $0?.cancel() }
    let ordinary = Task { Task.isCancelled }
    let detached = Task.detached { Task.isCancelled }
    return await (ordinary.value, detached.value)
}
let cancelled = await creator.value
precondition(!cancelled.0 && !cancelled.1)
```

**Walkthrough:** The helper's closure was formed outside an actor declaration. The creator then cancels itself and creates two independent tasks; both start with uncancelled state.

**Boundary:** A lexically nonisolated caller-isolated async helper also does not automatically give all nested tasks the actor of its dynamic caller. Use an explicit actor-bound API or structured work when required.

### Extension — Does explicitly discarding a throwing Task handle handle its error?

**Answer:** No. It acknowledges unused ownership and can silence the diagnostic; it neither reports the error nor stops the task.

**Example:** Standalone **compile-only** explicit discard. It is accepted, with the failure still unobserved.

```swift
enum Failure: Error { case offline }
func start() { _ = Task<Int, Error> { throw Failure.offline } }
```

**Walkthrough:** No code awaits the stored result or catches the error inside the task. Compiler silence does not supply an error policy. [SE-0520](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0520-discardableresult-task-initializers.md)

**Boundary:** The diagnostic also covers other throwing task initializers under their availability. Awaited defer is the scope-bound teardown option; spawning detached cleanup changes ownership and context.

## Focused exercise

**Task:** Own a failing task and a cancellable task, observe both terminal results, and join both handles.

**Acceptance checks:**

- A deliberate service error reaches the owner.
- Cancellation precedes the gated check.
- Both operations finish; no creator-cancellation inheritance is assumed.

**Solution:** Prepend **gate**. Runnable.

```swift
enum Failure: Error { case expected }
let failing = Task<Int, Error> { throw Failure.expected }
do { _ = try await failing.value; preconditionFailure("expected error") }
catch Failure.expected { }
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
```

**Reasoning:** The gate makes cancellation ordering explicit. Awaiting observes the result; cancelling is a separate action. A longer-lived UI owner additionally needs freshness checks on success, error, and cleanup.

## In practice

**Choose it when:** A synchronous entry or genuinely separate lifecycle needs an explicit task owner.

**Investigate by:** Trace lexical isolation, metadata, cancellation forwarding, and error observation separately.

**Strong evidence:** Explain why the boundary exists and how the owner completes teardown rather than equating a stored handle with a complete lifecycle policy.

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
- [Apple Developer. "Explore Structured Concurrency in Swift." WWDC21](https://developer.apple.com/videos/play/wwdc2021/10134/)
- [GitHub. "Global Actors." Swift Evolution SE-0316](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)
- [SE-0461 — Async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)
