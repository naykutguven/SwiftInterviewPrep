---
tags:
  - interview-prep
  - swift
  - structured-concurrency
  - async-await
  - concurrency
---

# D1 — Structured Concurrency Fundamentals

**Learning goals:**

- Explain what a task is, what `async` and `await` permit, and how suspension differs from blocking a thread.
- Choose sequential calls or structured child tasks from the dependencies between operations.
- Trace child lifetime, error observation, and implicit waiting when a scope exits.
- Recognize blocking bridges and unowned tasks, then replace them with an approach that has a clear completion path.

**Prerequisites:** Functions, throwing errors, and closures. Actors and continuations are introduced briefly where needed; [[D4 — Actors and Actor Isolation]] and [[D9 — Continuations and Bridging Callback APIs]] develop them further.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Imagine loading a profile screen. It needs a display name and an unread-message count. The requests can be independent, but the screen should not report that loading finished while either required request is still running.

A **task** is a unit of asynchronous work. It can call many functions without creating more tasks. An `async` function is allowed to **suspend**: pause its progress and release its executing thread so other work can use that thread. The task later resumes where it left off. `await` identifies an operation at which suspension is possible; it does not promise that a pause occurs on every call. [SE-0296 — async/await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md)

**Structured concurrency** gives child tasks a bounded lifetime. A child created with `async let` or a task group must finish before its enclosing scope finishes. A **scope** is the enclosing function or block that owns those children. To **join** a child means to wait until it finishes. Requesting cancellation and joining are separate actions: a child can receive cancellation and still need time to finish. [SE-0304 — child tasks](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#child-tasks)

**Concurrency** means operations can make progress over overlapping periods. **Parallelism** means they execute at the same instant. Two children can overlap while sharing a thread through suspension. Creating children does not promise two CPU cores, a background thread, or a particular completion order.

An **executor** schedules pieces of task work onto threads. Swift's cooperative executor expects tasks to let other work make progress by suspending instead of blocking a worker while waiting for future task work. A **worker pool** is the set of threads servicing that work. Holding all those workers in blocking waits can prevent the very tasks needed to release the waits from running. [Apple — Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)

## Essential mechanics

### Example setup and how to run it

Most examples below are independent Swift scripts. Save the relevant block as `Example.swift`; prepend a shared setup only when the example explicitly names it. Compile and run with:

```sh
swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -module-cache-path /tmp/swift-interview-module-cache \
  Example.swift -o /tmp/swift-interview-example
/tmp/swift-interview-example
```

Top-level `await` is supported in these scripts. Top-level statements are main-actor isolated; this does not change the module's `nonisolated` default. Examples marked **compile-only** contain function declarations without a driver; check them with the same flags and `-typecheck` instead of `-o ...`. Do not combine independent examples into one file: some deliberately reuse function names.

**Shared setup — profile requests:** These small services yield to the scheduler, then return recognizable values. `Task.yield()` provides an opportunity for other work to run; it neither guarantees a suspension nor simulates network latency. The examples do not use timing to establish ordering.

```swift
func fetchDisplayName() async -> String {
    await Task.yield()
    return "Lea"
}

func fetchUnreadCount() async -> Int {
    await Task.yield()
    return 3
}
```

### Calling two async functions is still sequential

**Setup:** Prepend **profile requests**, then run this block.

```swift
func loadSequentially() async -> String {
    let name = await fetchDisplayName()
    let unread = await fetchUnreadCount()
    return "\(name): \(unread) unread"
}

print(await loadSequentially())
```

**Walkthrough:** The current task enters `fetchDisplayName()`. It may suspend there, but it does not start `fetchUnreadCount()` until the first call returns. It then loads the count and builds the string. The output is `Lea: 3 unread`. There is one task following a sequence of calls, not two child tasks.

Sequential calls are the right choice when the second operation needs the first result. If the count required an account ID returned by the first call, starting it earlier would not remove that dependency.

### `async let` starts a structured child

**Setup:** Prepend **profile requests**, then run this block.

```swift
func loadTogether() async -> String {
    async let name = fetchDisplayName()
    async let unread = fetchUnreadCount()
    let loaded = await (name, unread)
    return "\(loaded.0): \(loaded.1) unread"
}

print(await loadTogether())
```

**Walkthrough:** Each `async let` declaration creates a child task. The parent reaches the second declaration without first waiting for the name. Reading the bindings at the tuple `await` waits for both results. The bindings retain their meanings even if the count finishes first, so the output remains `Lea: 3 unread`.

The declarations permit overlap. They do not establish which child starts first or guarantee parallel execution. For throwing initializers, `try` is required when their results are read; [[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]] explains error-observation order. [SE-0317 — async let](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md)

### Scope exit always waits, but cancellation depends on the construct

A task group is a block in which the parent can add children dynamically. This first example needs only one child to demonstrate its lifetime.

**Setup:** Standalone runnable script.

```swift
await withTaskGroup(of: Void.self) { group in
    group.addTask {
        await Task.yield()
        print("child finished")
    }
    print("group body finished")
}
print("group returned")
```

**Walkthrough:** `addTask` starts a child. Reaching the end of the body does not allow the outer call to return with that child still running. The first two messages may appear in either order; `group returned` must be last. Normal group-body return does not cancel the child.

| Construct and exit | What happens to unfinished children? |
| --- | --- |
| Unawaited `async let` at scope exit | Cancellation is requested, then the children are joined. |
| Regular task group, normal body return | Children are joined; normal return alone does not cancel them. |
| Regular throwing group, error escaping the body | Remaining children receive cancellation and are joined before the error leaves the group call. |
| Explicit `group.cancelAll()` | Cancellation is requested; the group still joins its children. |

Parent cancellation also normally propagates to structured children. [[D2 — Task Hierarchy, Cancellation, and Priorities]] explains cooperation and the optional cancellation-shield exception. No entry in this table means that cancellation forcibly stops arbitrary instructions. [SE-0304 — task groups](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#task-groups)

## Rules and pitfalls

### Start independent children when overlap is required

**Common pitfall:** Adding `await` to each call is mistaken for starting both requests together.

**Mistake:** Prepend **profile requests**. Runnable, but it does not meet the requirement to start independent child work.

```swift
let name = await fetchDisplayName()
let unread = await fetchUnreadCount()
print("\(name): \(unread) unread")
```

**Consequence:** The unread-count call starts only after the name call returns. An asynchronous implementation can still make unnecessarily sequential requests.

**Why:** `await` permits the current task to suspend. It does not create a second task to perform the next line while the first call is unfinished.

**Correction:** Use the same shared setup; run this block separately.

```swift
async let name = fetchDisplayName()
async let unread = fetchUnreadCount()
let loaded = await (name, unread)
print("\(loaded.0): \(loaded.1) unread")
```

**What changes:** The two declarations create children whose progress can overlap. Both versions produce the same string, so matching output alone is not evidence that requests overlapped. Choose this correction only because the requests are independent and overlap is useful.

### An async wrapper does not make blocking work asynchronous

**Shared setup — legacy file API:** Prepend this setup to both versions below. The API performs a synchronous file read. Both wrapper examples are **compile-only**; they need a caller-supplied file URL to perform a read.

```swift
import Foundation
import Dispatch

func readLegacyFile(_ url: URL) throws -> Data {
    try Data(contentsOf: url)
}
```

**Common pitfall:** Changing the wrapper's signature to `async` is expected to release the caller's thread during the read.

**Mistake:**

```swift
func loadFile(_ url: URL) async throws -> Data {
    try readLegacyFile(url)
}
```

**Consequence:** While `readLegacyFile` runs, its executing thread remains occupied by synchronous I/O. Calling this from the main actor can hold up UI work.

**Why:** The wrapper contains no mechanism that suspends while another execution resource performs the read. Under this baseline, a plain nonisolated async function follows caller isolation. Neither `async` nor `@concurrent` turns a synchronous read into nonblocking I/O. [SE-0461 — async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)

**Correction:** Run the legacy operation on a separately managed serial queue and suspend until its one completion arrives. A **continuation** is the handle used to resume that suspended operation.

```swift
private let fileQueue = DispatchQueue(label: "study.legacy-file")

func loadFile(_ url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        fileQueue.async {
            continuation.resume(with: Result { try readLegacyFile(url) })
        }
    }
}
```

**What changes:** The queue performs the blocking read, and the task can suspend while it waits for the continuation to resume. The serial queue limits simultaneous reads to one; it does not bound its waiting backlog. The read still blocks a queue thread, and cancelling the Swift task does not automatically cancel the read. A real adapter also needs an admission policy and whatever cancellation the legacy API can support; see [[D9 — Continuations and Bridging Callback APIs]].

### `Task {}` needs an explicit owner

**Common pitfall:** Creating `Task {}` inside an async function is mistaken for creating a structured child of that function.

**Mistake:** Prepend **profile requests**. Runnable, with intentionally unspecified output timing.

```swift
func refresh() async {
    Task {
        let name = await fetchDisplayName()
        print(name)
    }
}

await refresh()
print("refresh returned")
```

**Consequence:** `refresh returned` can appear before the name. Returning from `refresh` does not wait for the task, and cancellation of the caller does not automatically cancel this new unstructured task.

**Why:** The `Task` initializer creates work with a separate lifetime. Context inheritance, such as actor isolation or task-local values, does not establish structured parent-child lifetime. Dropping a task handle also does not cancel the work.

**Correction:** For one operation belonging to `refresh`, call it directly in the current task.

```swift
func refresh() async {
    let name = await fetchDisplayName()
    print(name)
}

await refresh()
print("refresh returned")
```

**What changes:** `Lea` is printed before `refresh returned`. No extra task was necessary. For several independent operations, use `async let` or a group. When an unstructured task is appropriate, retain its handle where its lifetime is managed, observe its result or error, and define cancellation and completion policy; [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]] develops that design.

## Worked example

### Returning from a scope with an unawaited child

**Setup:** Standalone runnable script. The loop is a small demonstration of cooperative cancellation, not a recommended way to wait for production events.

**Predict:** Does `runScope()` return before the child prints its cleanup message?

```swift
func watchForCancellation() async {
    while !Task.isCancelled {
        await Task.yield()
    }
    print("child finished cleanup")
}

func runScope() async {
    async let _: Void = watchForCancellation()
    print("parent body finished")
}

await runScope()
print("parent returned")
```

**Outcome:** The messages are `parent body finished`, `child finished cleanup`, then `parent returned`.

**Why:** The parent reaches its closing brace without explicitly reading the child binding. Swift requests cancellation of that child and waits for it to finish. The child checks its cancellation state, leaves the loop, and prints before completing. Only then can the parent return. The child may start before or after the parent's first print; the stated message order does not depend on that scheduling choice.

A child that neither finishes nor responds to cancellation could hold the scope open indefinitely. The structured guarantee is that the child cannot outlive the scope, not that scope exit always happens promptly.

## Interview questions

### What does `await` actually mean semantically?

**Answer:** It marks a possible suspension point in the current task. While suspended, that task does not need to occupy a thread; when the operation completes, execution continues after the call.

**Example:** Prepend **profile requests**, then run this block.

```swift
print("before request")
let name = await fetchDisplayName()
print("after request: \(name)")
```

**Walkthrough:** The first print happens before the call. The task enters the async function and may give other work an opportunity to run. The final print happens only after the function returns `Lea`. The caller has not created a child task by writing `await`.

**Boundary:** An async function can return without actually suspending. `await` does not imply background work or a change of actor. An **actor** is an isolation domain for protected state; an await can involve an actor transition when the called declaration requires one, but many awaits do not. After a genuine suspension, mutable state may have changed; [[D7 — Actor Reentrancy and Logic Races]] explains how to revalidate it.

### Why can a semaphore wait inside a task deadlock when its signal depends on work queued to the same cooperative pool?

**Answer:** A semaphore wait occupies a thread. If all available workers are waiting for tasks that need those same workers, none of the signaling tasks can run.

**Example:** Standalone **compile-only; do not call**. Depending on isolation and scheduling, this can deadlock. It is not a safe runtime experiment.

The SDK rejects a direct semaphore `wait()` in an async function. A synchronous helper can hide that call from the diagnostic without removing the blocking behavior:

```swift
import Dispatch

func blockingWait(_ semaphore: DispatchSemaphore) {
    semaphore.wait()
}

func unsafeWait() async {
    let semaphore = DispatchSemaphore(value: 0)
    Task {
        semaphore.signal()
    }
    blockingWait(semaphore)
}
```

**Walkthrough:** `Task {}` queues work; it does not promise that the work runs before the next statement. `wait()` then blocks the executing thread until `signal()` runs. On a constrained cooperative pool, enough such waits can occupy all workers. If the new task inherits an actor currently occupied by the blocking call, serialization can prevent progress even before pool exhaustion becomes relevant.

A suspending bridge avoids occupying the caller's thread:

```swift
import Dispatch

func waitForQueueCallback() async {
    await withCheckedContinuation { continuation in
        DispatchQueue.global().async {
            continuation.resume()
        }
    }
}

await waitForQueueCallback()
print("callback completed")
```

**Boundary:** The correction has a one-shot callback that always resumes the continuation exactly once. It does not make arbitrary blocking code safe. The queue does only a callback here; genuinely blocking work needs a separately managed queue and resource policy.

### How would you adapt an asynchronous DispatchQueue callback without waiting synchronously, and where should a genuinely blocking legacy operation run?

**Answer:** Resume a checked continuation from the asynchronous callback. Put genuinely blocking legacy work on a suitable separately managed queue, then bridge its completion the same way.

**Example:** Standalone runnable callback adapter.

```swift
import Dispatch

func cachedName(completion: @escaping @Sendable (String) -> Void) {
    DispatchQueue.global().async {
        completion("Lea")
    }
}

func cachedName() async -> String {
    await withCheckedContinuation { continuation in
        cachedName { name in
            continuation.resume(returning: name)
        }
    }
}

print(await cachedName())
```

**Walkthrough:** The adapter registers the callback. The task can suspend without a semaphore. The callback returns `Lea` through the continuation, allowing the awaiting call to finish. Resuming before suspension has actually occurred is also supported.

For the blocking case, prepend **legacy file API** to this **compile-only** adapter:

```swift
private let legacyReadQueue = DispatchQueue(label: "study.blocking-read")

func readFileAsynchronously(_ url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        legacyReadQueue.async {
            continuation.resume(with: Result { try readLegacyFile(url) })
        }
    }
}
```

Here the read still occupies a queue thread, but the Swift caller can suspend. The queue serializes simultaneous reads; callers must separately limit admission if a large waiting backlog is possible.

**Boundary:** The continuation wrapper does not add cancellation to an API that lacks it. Its callback must resume exactly once on every path. Avoid blocking the cooperative executor and consult [[D9 — Continuations and Bridging Callback APIs]] for cancellation, duplicate callbacks, and registration races.

### Why is structured concurrency safer than ad hoc callback trees or detached work?

**Answer:** It makes the owner of child completion visible in the code. A scope cannot finish while its structured children remain unfinished, and parent cancellation normally reaches those children.

**Example:** Prepend **profile requests**. This runnable example adds a failing name request so the caller can observe the error.

```swift
enum ProfileError: Error { case unavailable }

func requiredName(fail: Bool) async throws -> String {
    if fail { throw ProfileError.unavailable }
    return await fetchDisplayName()
}

func requiredProfile(fail: Bool) async throws -> (String, Int) {
    async let name = requiredName(fail: fail)
    async let unread = fetchUnreadCount()
    return try await (name, unread)
}

do {
    _ = try await requiredProfile(fail: true)
} catch ProfileError.unavailable {
    print("caller observed failure")
}
```

**Walkthrough:** The function owns both children. Reading `name` observes its failure; leaving the function cancels and joins any unawaited child before throwing to the caller. The caller receives the error through ordinary throwing control flow. Neither child can silently continue beyond that scope.

**Boundary:** Structure does not guarantee automatic observation of every possible error: regular throwing-group errors must be consumed or explicitly drained according to policy, as D3 explains. It also does not choose retries, make cancellation prompt, or prevent stale UI updates. SwiftUI `.task` has framework-managed lifetime; it is distinct from language-level `async let` and task-group child structure.

### Extension — Can an early return suspend while deferred async cleanup completes?

**Answer:** Yes. In Swift 6.4, an async function can await inside `defer`. That cleanup must finish before an early return completes. [SE-0493 — async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)

**Example:** Standalone runnable Swift 6.4 script. The actor represents a resource whose cleanup has asynchronous work.

```swift
actor Lease {
    private(set) var closed = false

    func close() async {
        await Task.yield()
        closed = true
        print("lease closed")
    }
}

func readNothing(using lease: Lease) async -> String {
    defer { await lease.close() }
    print("return requested")
    return "empty"
}

let lease = Lease()
let value = await readNothing(using: lease)
let closed = await lease.closed
precondition(value == "empty" && closed)
print("caller received value")
```

**Walkthrough:** The return value is selected, then deferred cleanup runs. `close()` can suspend, so the caller waits until it finishes. The printed order is `return requested`, `lease closed`, `caller received value`. The same scope-exit reasoning applies when a child must be implicitly joined, even without a visible final `await` at the closing brace.

**Boundary:** Awaited cleanup does not automatically shield cancellation, prevent cleanup failure, or make actor state atomic across suspension. D2 covers shielding; D7 covers actor invariants. Keep this extension separate from the basic meaning of `await`.

## Focused exercise

**Task:** Start two children for a profile operation. Return the required name, leave a supplementary watcher unawaited, and trace why the parent still waits for the watcher's cancellation cleanup.

**Acceptance checks:**

- Both children acknowledge starting before the test releases the required request.
- The required result remains associated with the name request.
- The watcher observes cancellation and finishes before the parent returns.
- Event ordering follows explicit acknowledgments, not sleeps or assumed child scheduling order.

**Setup:** This standalone runnable script introduces two test helpers. `Gate` is a one-way door: `wait()` suspends until `open()` is called, including when opening happens first. It deliberately ignores cancellation so the driver controls the required request. `EventLog` records events through an actor so concurrent tasks do not mutate an array unsafely. These helpers support the test; neither is a production cancellation primitive.

**Solution:**

```swift
actor Gate {
    private var isOpen = false
    private var waiters: [CheckedContinuation<Void, Never>] = []

    func wait() async {
        if isOpen { return }
        await withCheckedContinuation { waiters.append($0) }
    }

    func open() {
        isOpen = true
        let pending = waiters
        waiters.removeAll()
        for waiter in pending { waiter.resume() }
    }
}

actor EventLog {
    private(set) var events: [String] = []
    func record(_ event: String) { events.append(event) }
}

func requiredName(started: Gate, release: Gate, log: EventLog) async -> String {
    await started.open()
    await release.wait()
    await log.record("name finished")
    return "Lea"
}

func watch(started: Gate, log: EventLog) async {
    await started.open()
    while !Task.isCancelled { await Task.yield() }
    await log.record("watcher finished cleanup")
}

func profile(nameStarted: Gate, watcherStarted: Gate,
             release: Gate, log: EventLog) async -> String {
    async let name = requiredName(started: nameStarted, release: release, log: log)
    async let _: Void = watch(started: watcherStarted, log: log)
    let result = await name
    await log.record("parent body finished")
    return result
}

let nameStarted = Gate(), watcherStarted = Gate(), release = Gate()
let log = EventLog()
let parent = Task {
    await profile(nameStarted: nameStarted, watcherStarted: watcherStarted,
                  release: release, log: log)
}
await nameStarted.wait()
await watcherStarted.wait()
await release.open()
let name = await parent.value
await log.record("parent returned")
let events = await log.events
precondition(name == "Lea")
precondition(events == ["name finished", "parent body finished",
                        "watcher finished cleanup", "parent returned"])
print(events.joined(separator: " -> "))
```

**Reasoning:** The driver owns and joins the outer unstructured task so it can release the gate while the parent runs. Both children have started when `release.open()` lets the required request finish. The parent explicitly awaits that request but leaves the watcher unread. Scope exit therefore cancels and joins the watcher. Its cleanup record must precede `parent returned`. This verifies unfinished-child lifetime; it does not claim which child started first or how quickly arbitrary services cancel.

## In practice

**Choose it when:** Work belongs to one operation and the caller needs a clear completion boundary. Keep real dependencies sequential; use fixed children for independent known requests and groups for a dynamic collection.

**Investigate by:** Identify each task creator, each possible suspension, and every blocking call. For an unstructured task, locate its owner and error observation. For a returning scope, ask what it still has to join.

**Strong evidence:** Explain both the successful path and the path where a child fails or never cooperates with cancellation. A correct bridge keeps the task suspended while external work proceeds, defines admission limits, and documents cancellation separately from completion.

## References

### Related guides

- [[D2 — Task Hierarchy, Cancellation, and Priorities]] — Cancellation cooperation, task context, clocks, and bounded cleanup.
- [[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]] — Dynamic groups, result ordering, and error policy.
- [[D4 — Actors and Actor Isolation]] and [[D5 — Global Actors and `@MainActor`]] — Isolation and execution context.
- [[D7 — Actor Reentrancy and Logic Races]] — Revalidating state after suspension.
- [[D9 — Continuations and Bridging Callback APIs]] — Correct one-shot adapters and cancellation races.
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]] — Streaming instead of accumulating unbounded work.
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]] — Explicit ownership of unstructured work.
- [[D13 — Swift 6 Strict Concurrency Migration]] and [[D14 — Approachable Concurrency and Default Isolation]] — Migration and settings.
- [[F1 — Testing Strategy in Modern Swift]] and [[F2 — Debugging Swift code with LLDB and compiler diagnostics]] — Deterministic checks and investigation.

### Official sources

- [SE-0296 — async/await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md) — Potential suspension and thread use.
- [SE-0304 — Structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) — Child lifetimes, groups, cancellation, and unstructured tasks.
- [SE-0317 — async let](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md) — Bindings, result reads, and scope-exit cancellation.
- [SE-0461 — Async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md) — Caller isolation under this baseline.
- [SE-0493 — Async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md) — Cleanup completed before return.
- [Apple — Explore structured concurrency in Swift](https://developer.apple.com/videos/play/wwdc2021/10134/) — Task trees and scoped lifetime.
- [Apple — Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/) — Cooperative execution and forward progress.
- [Apple — Beyond the basics of structured concurrency](https://developer.apple.com/videos/play/wwdc2023/10170/) — Task ownership, cancellation, and tracing.
- [SwiftUI task(id:priority:_:)](https://developer.apple.com/documentation/swiftui/view/task%28id%3Aname%3Apriority%3Afile%3Aline%3A_%3A%29) — Framework-managed view task lifetime.
