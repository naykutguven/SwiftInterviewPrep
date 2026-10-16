---
tags:
  - interview-prep
  - swift
  - structured-concurrency
  - async-await
  - concurrency
---

# D1 — Structured Concurrency Fundamentals

**Goal:** Identify child lifetime and potential suspension without confusing await with thread blocking.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Async functions can suspend and later resume; suspension releases the executing thread for other work. Structured concurrency adds lexical ownership of child tasks through async let and task groups. Concurrency permits overlapping progress, while parallelism requires actual simultaneous execution. Neither async nor await guarantees a particular thread or parallel execution. [SE-0296](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md)

## Rules and pitfalls

### Preserve forward progress

Await a genuinely asynchronous dependency rather than blocking a cooperative worker on a semaphore or condition whose signal needs future task work. A pool cannot safely assume blocked workers will be replaced without limit.

**Common pitfall:** Wrapping a blocking legacy call in an async function leaves it blocking on the caller's execution context. Dispatch genuinely blocking work to an appropriate separately managed executor/queue and suspend the task through a callback bridge. See [[D9 — Continuations and Bridging Callback APIs]].

### Own child lifetime and errors

Async let and groups create children that cannot outlive their scope. Unfinished children are cancelled and joined on scope exit; cancellation is cooperative, so exit can wait. Task {} and detached tasks instead need an explicit owner and error observation.

**Common pitfall:** Calling Task {} inside an async method does not make that work a structured child. Retain, cancel, and join it when unstructured ownership is actually required.

## Interview questions

### Can an early return suspend while deferred async cleanup completes?

Yes. Swift 6.4 async defer can await cleanup before an early return completes. Structured child scope exit can also perform implicit cancellation and joining without a visible final await. Account for those suspension points when reasoning about actor invariants and latency.

### What does `await` actually mean semantically?

It marks a possible suspension boundary at an async operation and the required isolation transition. It does not require an actual pause, start a new task by itself, or mean background work. Under this baseline, a plain nonisolated async helper uses caller isolation; @concurrent explicitly leaves that actor.

### Why can a semaphore wait inside a task deadlock when its signal depends on work queued to the same cooperative pool?

The waiting task occupies a worker instead of suspending. If all available workers wait for signals from tasks queued to the same cooperative pool, none of those tasks can run. Use suspension for asynchronous completion; do not rely on thread-pool expansion to repair a circular progress dependency.

### How would you adapt an asynchronous DispatchQueue callback without waiting synchronously, and where should a genuinely blocking legacy operation run?

Resume a checked continuation from the queue's callback instead of waiting synchronously. Put truly blocking legacy work on a dedicated suitable queue/executor with bounded resource policy, then bridge its completion. Simply adding async or @concurrent does not turn blocking I/O into a nonblocking operation.

### Why is structured concurrency safer than ad hoc callback trees or detached work?

The scope owns completion, cancellation propagation, and error observation for its children, reducing orphaned work and lifetime ambiguity. It does not automatically supply correct retries, ordering, or prompt cancellation. SwiftUI .task has framework-managed lifetime, which is distinct from language-level async-let/group child structure.

## Focused exercise

**Task:** Trace two child operations and the parent's scope exit. Identify potential suspension and explain why an unfinished child must be joined.

**Acceptance checks:**

- Both values retain their request association.
- The scope cannot return while a child remains unfinished.
- No assertion relies on which child starts or completes first.

**Solution:**

Standalone async executable; no output order is claimed:

```swift
func request(_ id: Int) async -> Int { id * 10 }
func combined() async -> (Int, Int) {
    async let first = request(1)
    async let second = request(2)
    return await (first, second)
}
@main struct Probe {
    static func main() async {
        let result = await combined()
        precondition(result.0 == 10 && result.1 == 20)
    }
}
```

Compile with -parse-as-library and the baseline flags. Each initializer creates a child; the tuple await observes both before combined returns.

**Reasoning:** The requests are independent, so fixed child declarations express potential overlap. The small functions may finish without meaningful suspension; use controlled gates for lifetime/cancellation evidence rather than timing print statements.

## In practice

**Choose it when:** Preserve the task tree when work belongs to one operation.

**Investigate by:** Trace task ownership, suspension, blocking, and implicit joins.

**Strong evidence:** Design a bridge that preserves forward progress and records who cancels, joins, and handles errors.

### Implicit joining at scope exit — optional

#### Implicit cancellation and waiting at scope exit

Standalone with -parse-as-library:

```swift
func childWork() async {
    while !Task.isCancelled { await Task.yield() }
    print("child observed cancellation")
}

func parentScope() async {
    async let _: Void = childWork()
    print("parent body")
}

@main struct Main {
    static func main() async {
        await parentScope()
        print("parent returned")
    }
}
```

The trace is parent body, child observed cancellation, parent returned. The unawaited async-let child is cancelled and awaited when the scope exits. This proof uses cooperative child work; a child that never responds or completes can hold up scope exit. A regular task group has different normal-exit cancellation rules; see D3.

## References

### Related guides

- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]]
- [[D4 — Actors and Actor Isolation]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[D13 — Swift 6 Strict Concurrency Migration]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

### Official sources

- [SE-0296](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md)
- [SwiftUI task(id:priority:_:)](https://developer.apple.com/documentation/swiftui/view/task%28id%3Aname%3Apriority%3Afile%3Aline%3A_%3A%29)
- [GitHub. "Structured Concurrency." Swift Evolution SE-0304](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [Apple Developer. "Explore Structured Concurrency in Swift." WWDC21](https://developer.apple.com/videos/play/wwdc2021/10134/)
- [Apple Developer. "Beyond the Basics of Structured Concurrency." WWDC23](https://developer.apple.com/videos/play/wwdc2023/10170/)
- [SE-0493 — Async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)
