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

**Learning goals:**

- Distinguish requesting cancellation, observing it, finishing cleanup, and joining a task.
- Explain child-task cancellation, task-local propagation, priority inheritance, and priority escalation.
- Prevent old UI requests from publishing stale results, errors, or cleanup after replacement.
- Choose a clock for an elapsed-time budget and explain why a timeout can still wait for losing work.
- As an extension, explain cancellation shields without treating them as an undo operation.

**Prerequisites:** [[D1 — Structured Concurrency Fundamentals]] introduces tasks, suspension, children, and joining. An actor serializes access to its protected state; the test helpers below use actors for gates and counters, explained before their use.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A user starts a search and then changes the query. The first result is no longer useful. Calling `cancel()` tells the old task that its result is no longer wanted; it does not erase completed effects, forcibly interrupt instructions, or guarantee that the old service stops immediately.

Cancellation is **cooperative**: code checks the task's cancellation state or awaits an operation that responds to cancellation. A task can then throw, return a partial result under an explicit policy, or finish required cleanup. **Joining** means waiting for its completion. A cancellation request and a completed join are different events. [SE-0304 — cancellation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#cancellation)

The task hierarchy carries context. Structured children normally receive parent cancellation, inherit priority by default, and see task-local values such as a request's trace ID. A **priority** expresses urgency to the runtime; it does not establish execution order. **Task-local values** are scoped contextual metadata associated with tasks, rather than with a physical thread. A request can retain its trace ID after suspension moves its execution elsewhere.

UI freshness is another responsibility. **Publishing** means changing state visible to the caller or UI. A cancelled request may still return a value or throw an ordinary error. The state owner must decide whether that particular request is still allowed to publish.

## Essential mechanics

### Example setup

Examples are separate Swift scripts using the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]], including top-level `await`. Prepend a shared setup only where named. Blocks marked **compile-only** are checked with `-typecheck`; they need a caller or external service to run. The selected runtime is macOS 27. Clock APIs require macOS 13/iOS 16 or later; the shielding extension separately requires OS 27.

### A cancellation request does not force a task to throw

**Setup:** Standalone runnable script.

```swift
let worker = Task {
    return "completed without checking cancellation"
}
worker.cancel()
let result = await worker.value
print(result)
```

**Walkthrough:** The task's body contains no cancellation check or cancellation-aware operation. Asking it to cancel therefore does not change its ordinary return value. Awaiting `value` joins it and retrieves that value. This example does not use completion timing to prove when work began; the focused exercise controls that explicitly.

To leave by throwing, use `Task.checkCancellation()` at a safe boundary between pieces of work. To inspect the state without throwing, use `Task.isCancelled` and define what an early return means.

**Setup:** Standalone **compile-only** function for a throwing, all-or-nothing index build.

```swift
func buildIndex(_ terms: [String]) async throws -> Set<String> {
    var index: Set<String> = []
    for term in terms {
        try Task.checkCancellation()
        index.insert(term.lowercased())
    }
    try Task.checkCancellation()
    return index
}
```

**Walkthrough:** Each check can reject further work before another term is indexed. Throwing prevents the function from returning an unfinished index as a successful complete result. The last check covers cancellation after the final iteration. Checks cannot undo an external write already performed, and they do not guarantee the caller cannot cancel just after the last check.

### Task-local values follow task context

**Setup:** Standalone runnable script. `withValue` binds the trace ID for its operation.

```swift
enum Trace {
    @TaskLocal static var requestID = "unbound"
}

await Trace.$requestID.withValue("profile-42") {
    let ordinary = Task { Trace.requestID }
    let detached = Task.detached { Trace.requestID }
    let inherited = await ordinary.value
    let independent = await detached.value
    precondition(inherited == "profile-42")
    precondition(independent == "unbound")
    print(inherited, independent)
}
```

**Walkthrough:** The ordinary task inherits the task-local binding; the detached task does not. Both handles are explicitly awaited because both tasks are unstructured. Their context behavior does not give them structured lifetime. The output is `profile-42 unbound`. The inheritance question below shows the corresponding structured-child case. [SE-0311 — task-local values](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0311-task-locals.md)

### Priorities express urgency and can be escalated

**Setup:** Standalone runnable script; its numeric output is intentionally unspecified.

```swift
let prefetch = Task(priority: .utility) {
    Task.currentPriority
}
let visibleRequest = Task(priority: .userInitiated) {
    await prefetch.value
}
let observedPriority = await visibleRequest.value
print(observedPriority.rawValue)
```

**Walkthrough:** The prefetch is requested at utility priority. When higher-priority work waits for a lower-priority task's value, the producer's effective priority can be raised. This is **priority escalation**, used to reduce priority inversion: urgent work waiting behind less urgent work. If the prefetch reads its priority before that wait is registered, its returned observation can precede escalation. Do not assert one numeric result from this script.

Children inherit their parent's priority by default, but explicit child priorities and later escalation affect the effective value. A higher priority does not promise earlier completion, reserve a thread, or authorize shared-state access. [SE-0304 — task priorities and escalation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#task-priorities)

### Cancellation handlers can notify an external operation

A cancellation handler is a synchronous callback registered with `withTaskCancellationHandler`. It can ask an external operation to stop even while the task's main operation is suspended. It does not replace cooperation inside the operation.

**Setup:** Standalone runnable script. Top-level statements and the ordinary `Task` share the main actor, so cancellation is requested before the task body gets its turn.

```swift
let request = Task {
    try await withTaskCancellationHandler {
        print("operation entered")
        try Task.checkCancellation()
        return "payload"
    } onCancel: {
        print("cancellation handler ran")
    }
}
request.cancel()
do {
    _ = try await request.value
} catch is CancellationError {
    print("caller observed cancellation")
}
```

**Walkthrough:** Because the task is already cancelled when the handler is registered, the handler runs first. The operation still enters; its explicit check throws. The output is `cancellation handler ran`, `operation entered`, then `caller observed cancellation`.

In a real adapter, the handler might call a thread-safe `cancel()` on an external request handle. A handler can run concurrently with its operation, so shared mutable registration state needs synchronization. See [[D9 — Continuations and Bridging Callback APIs]] for a full adapter rather than copying an unsynchronized captured variable.

## Rules and pitfalls

### Check freshness on success, error, and cleanup

**Shared setup — search service:** Prepend this to each version. Both versions are **compile-only** state owners; their caller owns and cancels the request tasks. The injected service may ignore cancellation and finish later with either values or an ordinary error.

```swift
import Foundation

typealias SearchService = @Sendable (String) async throws -> [String]
```

**Common pitfall:** Only successful results are checked for freshness, and query text is treated as the identity of an individual request.

**Mistake:**

```swift
@MainActor final class SearchState {
    var results: [String] = []
    var errorMessage: String?
    var activeQuery: String?

    func load(_ query: String, using service: SearchService) async {
        activeQuery = query
        defer { activeQuery = nil }
        do {
            let loaded = try await service(query)
            guard activeQuery == query else { return }
            results = loaded
        } catch {
            errorMessage = "Search failed"
        }
    }
}
```

**Consequence:** Request A starts, request B replaces it, and B succeeds. If A later fails, its `catch` can replace the current error state. Its `defer` can also clear B's active request. A sequence of queries A → B → A exposes another bug: comparing text cannot distinguish the two A requests.

**Why:** Main-actor serialization prevents simultaneous access to these properties, but suspension allows another request to change them. Query equality describes the resource; it does not prove which operation still owns the right to publish.

**Correction:** Give each invocation a fresh token and check both token freshness and cancellation on every publication path.

```swift
@MainActor final class SearchState {
    var results: [String] = []
    var errorMessage: String?
    var activeRequest: UUID?

    func load(_ query: String, using service: SearchService) async {
        let request = UUID()
        activeRequest = request
        defer {
            if activeRequest == request { activeRequest = nil }
        }
        do {
            let loaded = try await service(query)
            guard !Task.isCancelled, activeRequest == request else { return }
            results = loaded
            errorMessage = nil
        } catch {
            guard !Task.isCancelled, activeRequest == request else { return }
            errorMessage = "Search failed"
        }
    }
}
```

**What changes:** A superseded invocation cannot publish a late value, publish an error, or clear the replacement's token. Cancellation also suppresses publication when there is no replacement yet. Cleanup checks identity rather than cancellation so the current cancelled invocation can still clear its own slot. This method does not implement task ownership or cancellation of the external service; those remain explicit caller responsibilities.

### Cancelling losing work and joining it are separate responsibilities

**Shared setup — timed operation:** Prepend this to each version. Both are **compile-only** helpers, not tests that assert elapsed milliseconds.

```swift
enum TimedResult: Sendable { case value(String), timedOut }

func slowRequest() async throws -> String {
    try await Task.sleep(for: .seconds(5))
    return "profile"
}
```

**Common pitfall:** Returning a timeout result from a regular task-group body is expected to stop the other child automatically.

**Mistake:**

```swift
func race() async throws -> TimedResult {
    try await withThrowingTaskGroup(of: TimedResult.self) { group in
        group.addTask { .value(try await slowRequest()) }
        group.addTask {
            try await Task.sleep(for: .milliseconds(10))
            return .timedOut
        }
        return try await group.next()!
    }
}
```

**Consequence:** If the timer result is collected first, the body returns `.timedOut`, but the enclosing group call still waits for the slow request. Normal body return alone does not request cancellation.

**Why:** The group owns both children's lifetimes. Selecting the first result decides what to return, not how to complete the remaining child. Even a cancellation-aware sleep keeps waiting if nobody asks it to cancel.

**Correction:** Request cancellation on every body-exit path.

```swift
func race() async throws -> TimedResult {
    try await withThrowingTaskGroup(of: TimedResult.self) { group in
        defer { group.cancelAll() }
        group.addTask { .value(try await slowRequest()) }
        group.addTask {
            try await Task.sleep(for: .milliseconds(10))
            return .timedOut
        }
        return try await group.next()!
    }
}
```

**What changes:** The losing sleep receives cancellation and can throw, letting the scope finish sooner. The unwrap relies on adding two children unconditionally. Replace them with conditional additions and that assumption needs reconsideration. A service that ignores cancellation can still delay return; this correction does not promise a hard deadline.

### Use dependencies to establish order, not priorities

**Common pitfall:** A higher-priority task is expected to finish a prerequisite before another task begins.

**Mistake:** Standalone runnable script; the two messages have no required order.

```swift
let loading = Task(priority: .userInitiated) { print("profile loaded") }
let rendering = Task(priority: .background) { print("render requested") }
await loading.value
await rendering.value
```

**Consequence:** `render requested` is allowed before `profile loaded`. Awaiting both handles afterward does not retroactively order their bodies.

**Why:** Both tasks were created before either result was awaited. Priority expresses urgency, not a happens-before relationship between the two operations.

**Correction:** Start the dependent step after the prerequisite completes.

```swift
let loading = Task(priority: .userInitiated) { print("profile loaded") }
await loading.value
print("render requested")
```

**What changes:** The messages now have a required order because the render line is reached only after joining the loader. A direct sequential async call would also express that dependency without introducing an extra task.

## Worked example

### A cancelled consumer can still receive a shared producer's result

**Shared setup — test gate:** Prepend this actor only to examples that explicitly name **test gate**. It suspends waiters until opened and tolerates opening before waiting. It deliberately does not react to cancellation; this lets a test control when a suspended operation resumes.

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

**Setup:** Prepend **test gate**, then run this block.

**Predict:** Does cancelling `consumer` cancel `producer` or make awaiting its value throw?

```swift
let release = Gate(), waiting = Gate()
let producer = Task {
    await release.wait()
    return "shared profile"
}
let consumer = Task {
    await waiting.open()
    return await producer.value
}
await waiting.wait()
consumer.cancel()
precondition(!producer.isCancelled)
await release.open()
let value = await consumer.value
precondition(value == "shared profile")
print(value)
```

**Outcome:** The producer remains uncancelled and the consumer receives `shared profile`.

**Why:** The handles refer to two unstructured tasks. Awaiting one task's value creates a wait, not a structured cancellation relationship. Consumer cancellation neither cancels the producer nor makes this nonthrowing value access throw. The driver releases the producer and joins both paths through the consumer's await.

If the result is no longer publishable, the consumer must check its own cancellation before publishing. If consumers need prompt cancellation while a shared producer continues, design a cancellable waiting/subscription layer; merely adding a check after `value` does not interrupt the wait. The producer's owner decides when the shared operation itself should be cancelled.

## Interview questions

### Why is cancellation in Swift cooperative rather than preemptive?

**Answer:** Code needs safe points at which it can abandon work without leaving its invariants or resource ownership half-updated. A cancellation request therefore changes task state, and the operation decides how to leave safely.

**Example:** Standalone runnable script. As in the handler example, the main-actor task is cancelled before it enters its body.

```swift
func processRequest() async throws {
    defer { print("cleanup finished") }
    try Task.checkCancellation()
    print("processing would begin")
}

let request = Task { try await processRequest() }
request.cancel()
do {
    try await request.value
} catch is CancellationError {
    print("owner observed completion")
}
```

**Walkthrough:** The explicit check throws before processing begins. Stack unwinding runs `defer`, then the error reaches the awaiting owner. `cleanup finished` appears before `owner observed completion`; the processing message is absent. `cancel()` alone would not have inserted that check.

**Boundary:** Already completed external effects remain completed. Cancellation-aware APIs can use their own cancellation errors, so adapters should understand those contracts rather than assuming every cancellation is spelled `CancellationError`.

### What does a child task inherit from its parent?

**Answer:** A structured child participates in parent cancellation and inherits priority by default and task-local bindings. Its lifetime is bounded by the structured scope. Actor execution depends on the specific child closure and callee; context inheritance does not mean every child runs on the parent's actor.

**Example:** Standalone runnable script.

```swift
enum RequestTrace {
    @TaskLocal static var id = "unbound"
}

await RequestTrace.$id.withValue("search-17") {
    await withTaskGroup(of: (String, TaskPriority).self) { group in
        group.addTask {
            (RequestTrace.id, Task.currentPriority)
        }
        let child = await group.next()!
        precondition(child.0 == "search-17")
        print(child.0)
        // child.1 is the effective priority, including any escalation.
    }
}
```

**Walkthrough:** The binding surrounds child creation, so the child sees `search-17`. No explicit child priority is supplied; the runtime derives it from the parent, subject to later escalation. The group collects and joins the child before leaving the binding. The unwrap is safe because exactly one child was added.

**Boundary:** `Task {}` also inherits selected context, but has unstructured lifetime and no automatic parent cancellation relationship. `Task.detached` does not inherit task-local bindings, the creating task's priority, or its actor context. Explicit priority arguments remain possible.

### Why does repeated cell configuration need a unique request identity, and why must stale error handling check cancellation too?

**Answer:** A resource identifier can repeat while distinct requests for that resource overlap. A unique operation token identifies which invocation still owns publication, including its error and cleanup paths.

**Example:** Standalone runnable script showing A → B → A without a timing-dependent service. This slot models the publication boundary of a cell or view model.

```swift
import Foundation

@MainActor final class RequestSlot {
    var query = ""
    var current: UUID?
    var errorMessage: String?

    func begin(_ query: String) -> UUID {
        self.query = query
        let request = UUID()
        current = request
        errorMessage = nil
        return request
    }

    func fail(_ request: UUID, message: String) {
        guard !Task.isCancelled, current == request else { return }
        errorMessage = message
    }

    func finish(_ request: UUID) {
        if current == request { current = nil }
    }
}

let slot = RequestSlot()
let firstA = slot.begin("A")
_ = slot.begin("B")
let latestA = slot.begin("A")
slot.fail(firstA, message: "old A failed")
slot.finish(firstA)
precondition(slot.errorMessage == nil && slot.current == latestA)
slot.fail(latestA, message: "current A failed")
precondition(slot.errorMessage == "current A failed")
```

**Walkthrough:** Both A invocations have identical query text, but their tokens differ. The old failure and cleanup cannot affect the latest slot. The current failure can publish. The same identity check belongs on success, as shown in the corrected `SearchState`.

**Boundary:** The cancellation check matters when a service returns or throws after its task is cancelled, even if no replacement has started. This slot demonstrates freshness decisions; it does not itself create, cancel, or join task handles. Test that ownership separately.

### When does a policy need elapsed time instead of a wall-clock date? How do ContinuousClock and SuspendingClock differ during system sleep?

**Answer:** Use elapsed time for duration budgets such as debounce, retry delay, and timeouts. Wall-clock dates describe calendar deadlines and can change when the clock is adjusted. `ContinuousClock` includes system sleep; `SuspendingClock` excludes it. [SE-0329 — clocks](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0329-clock-instant-duration.md)

**Example:** Standalone runnable script. Injecting the observed instant makes the boundary check deterministic.

```swift
func budgetExpired(start: ContinuousClock.Instant,
                   now: ContinuousClock.Instant,
                   budget: Duration) -> Bool {
    start.duration(to: now) >= budget
}

let clock = ContinuousClock()
let start = clock.now
precondition(!budgetExpired(start: start,
    now: start.advanced(by: .seconds(1)), budget: .seconds(2)))
precondition(budgetExpired(start: start,
    now: start.advanced(by: .seconds(2)), budget: .seconds(2)))

let awakeClock = SuspendingClock()
let deadlineIncludingSleep = start.advanced(by: .seconds(10))
let deadlineExcludingSleep = awakeClock.now.advanced(by: .seconds(10))
// Sleep APIs can await these deadlines on their corresponding clocks.
```

**Walkthrough:** The first check is inside the two-second budget; the second is exactly at its boundary and is expired. Both checks construct instants rather than waiting. If a machine later sleeps for eight of ten elapsed seconds, the continuous clock includes all ten seconds while the suspending clock advances only for the awake interval. The two instant types are not interchangeable.

**Boundary:** Choose whether the product's budget should include sleep. This script verifies comparison boundaries, not actual system sleep or wall-clock adjustment. Scheduling can resume a sleeper after its deadline; a clock choice is not a hard real-time scheduling guarantee.

### If a timeout wins a structured race, who cancels and joins the losing work? Why can the function still return after its time budget?

**Answer:** The race owner requests cancellation, and the enclosing structured scope joins the loser. A loser that ignores cancellation can delay return beyond the budget.

**Example:** Standalone **compile-only** helper. The operation must be safe to run as a child, and its cancellation/side-effect behavior remains part of its own contract.

```swift
struct TimeoutError: Error {}

func withTimeout<Value: Sendable>(
    after budget: Duration,
    operation: @escaping @Sendable () async throws -> Value
) async throws -> Value {
    try Task.checkCancellation()
    return try await withThrowingTaskGroup(of: Value.self) { group in
        defer { group.cancelAll() }
        group.addTask {
            try Task.checkCancellation()
            return try await operation()
        }
        group.addTask {
            try await ContinuousClock().sleep(for: budget)
            throw TimeoutError()
        }
        return try await group.next()!
    }
}
```

**Walkthrough:** The helper starts an operation and a timer. A successful operation result causes normal body return; `defer` cancels the timer. If the timer's error is collected first, it escapes the body, requesting sibling cancellation as well. In both paths the group still waits for its children. The unwrap relies on adding two children unconditionally. This is a first-observed-completion policy, so a service error can win too.

**Boundary:** The timer starts when its child runs; this helper is not a strict entry-to-return deadline. A hard budget cannot be enforced by abandoning a structured child. Shielded cleanup can also extend the wait. The exercise verifies cooperative cleanup separately from choosing a timeout winner.

### Extension — Does a shield uncancel the task, or change how the current context observes cancellation?

**Answer:** It temporarily changes how the current context observes cancellation and can shield enclosed children from automatic parent cancellation. It does not clear the task's underlying cancelled state. [SE-0504 — cancellation shields](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md)

**Example:** Standalone runnable Swift 6.4 script targeting macOS 27. Both synchronous and asynchronous shield forms require `anyAppleOS 27.0` in the selected SDK.

```swift
let cleanup = Task {
    withUnsafeCurrentTask { $0?.cancel() }
    let inside = withTaskCancellationShield {
        Task.isCancelled
    }
    let childInside = await withTaskCancellationShield {
        async let childCancelled = Task.isCancelled
        return await childCancelled
    }
    return (inside, childInside, Task.isCancelled)
}
let observed = await cleanup.value
precondition(!observed.0 && !observed.1 && observed.2)
precondition(cleanup.isCancelled)
```

**Walkthrough:** The task first cancels itself. Static cancellation observation is masked inside the synchronous shield. The child created inside the async shield also avoids automatic cancellation from that already-cancelled parent. After the shields exit, static observation sees cancellation again. The handle's `isCancelled` remains true.

| Observation after cancellation | Inside a shield | After leaving |
| --- | --- | --- |
| Static `Task.isCancelled` | `false` | `true` |
| `Task.checkCancellation()` | Does not throw because of that cancellation | Throws |
| The task handle's `isCancelled` | Underlying state remains `true` | Still `true` |

**Boundary:** Shield only bounded required cleanup. Explicit `group.cancelAll()` still cancels group children, and an unawaited `async let` still receives scope-exit cancellation. Shielding only `group.addTask` does not shield the whole child's execution. Awaiting cleanup in `defer` is useful but does not itself install a shield.

The earlier recorded nested-handler probe suppressed both inner and outer handlers while shielded, for self-cancellation and external-handle cancellation; its unshielded control invoked both. Preserve that as a runtime observation, not a universal promise about every handler nesting pattern. The small example above checks observation and child propagation, not that historical nested-handler experiment.

## Focused exercise

**Task:** Cancel an operation before entry and during execution. Verify cancellation observation and joined asynchronous cleanup without ordering events through elapsed-time sleeps.

**Acceptance checks:**

- Cancellation before work entry and after its started acknowledgment both reach `CancellationError`.
- Every wait actually entered by the test is released, and the driver joins its task.
- Cleanup finishes exactly once before task completion is observed.
- No assertion depends on an arbitrary delay or child scheduling order.

**Setup:** Prepend **test gate** from the worked example. The entry gate controls when work can begin; the started gate acknowledges that the first check passed; the release gate controls the next check. The cleanup actor yields before incrementing its count, so the test observes completion of an awaited cleanup operation, not just entry into a synchronous `defer`.

**Solution:**

```swift
actor Cleanup {
    private(set) var count = 0
    func finish() async {
        await Task.yield()
        count += 1
    }
}

func work(started: Gate, release: Gate, cleanup: Cleanup) async throws {
    defer { await cleanup.finish() }
    try Task.checkCancellation()
    await started.open()
    await release.wait()
    try Task.checkCancellation()
}

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
    do {
        try await task.value
        preconditionFailure("expected cancellation")
    } catch is CancellationError {
        let count = await cleanup.count
        precondition(count == 1)
    } catch {
        preconditionFailure("unexpected error: \(error)")
    }
}
print("before-entry and during-work cancellation joined cleanup")
```

**Reasoning:** In the first case cancellation precedes opening the entry gate, so the first check throws. The release gate is never entered. In the second case the started acknowledgment proves work reached the controlled suspension. The driver requests cancellation, releases that wait, and observes the second check throwing. In each case `task.value` completes only after awaited cleanup increments its count. The gates establish causality; a sleep could merely make the test appear to work on one machine.

This test does not prove prompt cancellation of arbitrary services, UI freshness, real system-sleep behavior, or every shielding/handler interaction. Those are separate contracts with separate examples and checks.

## In practice

**Choose it when:** Define cancellation as part of an operation's API: who owns the task, what becomes invalid, how cancellation reaches external work, and what completion means after cleanup.

**Investigate by:** Record request tokens and separate cancellation-request, cancellation-observed, cleanup-finished, and task-completed events. If a timeout is late, identify the child the scope is still joining.

**Strong evidence:** Explain shared-producer ownership, protect success/error/cleanup publication, choose clock semantics deliberately, and verify the before-entry and during-work paths without timing guesses. Use priorities for urgency and explicit dependencies for correctness.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]] — Task ownership, suspension, and scope exit.
- [[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]] — Group result/error policies and bounded work.
- [[D7 — Actor Reentrancy and Logic Races]] — Freshness across actor suspension.
- [[D9 — Continuations and Bridging Callback APIs]] — Cancellation handlers and external request registration.
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]] — Subscription and shared-producer lifetime.
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]] — Unstructured ownership and inherited context.
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]] — Synchronizing cancellation-handler state.
- [[F1 — Testing Strategy in Modern Swift]] and [[F2 — Debugging Swift code with LLDB and compiler diagnostics]] — Controlled time and task evidence.

### Official sources

- [The Swift Programming Language — Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/) — Tasks, groups, and cooperative cancellation.
- [SE-0304 — Structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) — Cancellation, priority inheritance/escalation, and handlers.
- [SE-0311 — Task-local values](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0311-task-locals.md) — Scoped context propagation.
- [SE-0329 — Clock, Instant, and Duration](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0329-clock-instant-duration.md) — Elapsed-time clocks and system sleep.
- [SE-0493 — Async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md) — Awaited cleanup before completion.
- [SE-0504 — Task cancellation shields](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md) — Cancellation observation and propagation under shields.
- [Apple — TaskGroup](https://developer.apple.com/documentation/swift/taskgroup) — Group lifetime and cancellation.
- [Apple — detached(name:priority:operation:)](https://developer.apple.com/documentation/swift/task/detached%28name%3apriority%3aoperation%3a%29-795w1) — Detached context and ownership.
- [Apple — Beyond the basics of structured concurrency](https://developer.apple.com/videos/play/wwdc2023/10170/) — Task trees, cancellation, and priorities.
