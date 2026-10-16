---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - async-let
  - task-groups
  - structured-concurrency
---

# D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive

**Goal:** Run three independent child requests with an explicit result-association and failure policy.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Choose concurrency primitives from the shape of work. Sequential await expresses a dependency. Async let expresses a fixed set of independent children with values consumed in the same scope. Groups express dynamic child sets and completion-driven handling; discarding groups fit work whose values are not needed. Every structured scope still owns child completion. [Structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)

## Rules and pitfalls

### Associate results deliberately

Async-let bindings retain named roles. A task group produces completed results in completion order, so carry an input index/ID and reorder only if the caller needs input order. Bound active work when input size or native resources are large.

**Common pitfall:** Adding every request to a group can exhaust connections or memory. Seed a bounded number and replenish as results arrive; task creation alone is not a resource policy.

### Observe errors where policy requires

A regular throwing group's child error propagates when its result is observed and allowed to leave the body. Ignoring results is different from handling them. Discarding groups have distinct automatic failure behavior.

**Common pitfall:** An all-or-nothing operation that catches and ignores child errors can report success. Pick fail-whole or per-item outcomes explicitly, and remember cancellation still waits for cooperative children.

## Interview questions

### When is `async let` better than a task group, and how can the order of awaiting its values delay error observation?

Async let is clearer for a small fixed set of independent values. If the first awaited child is slow while another has failed, the error may not be observed until its binding is awaited. A group can observe completions earlier when dynamic or prompt failure handling is the requirement.

### Compare catching `next()`, using `nextResult()`, calling `waitForAll()`, ignoring child results, and a throwing discarding group.

next() throws when the observed child failed; catching it can allow deliberate continued draining. nextResult() returns the outcome as data. waitForAll() drains before throwing under its current contract. Ignoring regular-group results does not automatically propagate every child error. A throwing discarding group cancels siblings on child failure and propagates according to its scope contract.

### Why can a first-result or timeout race remain suspended after choosing its winner?

Choosing a winner does not release the scope from its remaining children. The owner cancels losers, but the scope joins their completion. Cancellation-insensitive work or shielded cleanup can therefore keep the race suspended after winner selection.

## Focused exercise

**Task:** Run three independent requests with async let or a group. Preserve result association and check the selected all-or-nothing or partial-result policy.

**Acceptance checks:**

- Successful results retain request order independent of completion order.
- One selected request failure makes the batch throw rather than return partial data.
- The enclosing scope joins all children on success and error.

**Solution:**

Standalone fixed-child all-or-nothing fixture:

```swift
enum RequestFailure: Error { case unavailable }
func request(_ id: Int, failing: Int?) async throws -> Int {
    try Task.checkCancellation()
    if id == failing { throw RequestFailure.unavailable }
    return id * 10
}
func batch(failing: Int?) async throws -> [Int] {
    async let first = request(1, failing: failing)
    async let second = request(2, failing: failing)
    async let third = request(3, failing: failing)
    return try await [first, second, third]
}
@main struct Probe {
    static func main() async throws {
        let values = try await batch(failing: nil)
        precondition(values == [10, 20, 30])
        do { _ = try await batch(failing: 2); preconditionFailure("expected failure") }
        catch RequestFailure.unavailable { }
    }
}
```

**Reasoning:** Named bindings are enough for three fixed requests. A partial-result policy would return per-request Result values instead of pretending missing data is success. The fixture proves result/failure association without claiming which child starts, finishes, or observes cancellation first.

## In practice

**Choose it when:** Match task shape, resource bounds, ordering, and error ownership.

**Investigate by:** Gate child completions and failures instead of asserting print order.

**Strong evidence:** Verify bounded work and selected failure policy, including slow cooperative cleanup after cancellation.

### Bounded groups and cleanup — optional

#### Bound active work and preserve input order

`addTask` does not impose a concurrency limit. Seed a limited number of children, then add one replacement for each collected result. Only the parent writes the results array. This standalone helper rejects nonpositive limits, returns an empty result for empty input with a positive limit, and propagates errors/cancellation:

```swift
enum FanOutError: Error { case invalidLimit }

func parallelMap<Input: Sendable, Output: Sendable>(
    _ inputs: [Input],
    maxConcurrent: Int,
    transform: @escaping @Sendable (Input) async throws -> Output
) async throws -> [Output] {
    guard maxConcurrent > 0 else { throw FanOutError.invalidLimit }
    try Task.checkCancellation()
    return try await withThrowingTaskGroup(of: (Int, Output).self) { group in
        var nextIndex = 0
        var results = [Output?](repeating: nil, count: inputs.count)

        while nextIndex < min(maxConcurrent, inputs.count) {
            let index = nextIndex
            let input = inputs[index]
            guard group.addTaskUnlessCancelled(operation: {
                (index, try await transform(input))
            }) else { throw CancellationError() }
            nextIndex += 1
        }

        while let (index, value) = try await group.next() {
            try Task.checkCancellation()
            results[index] = .some(value)
            if nextIndex < inputs.count {
                let index = nextIndex
                let input = inputs[index]
                guard group.addTaskUnlessCancelled(operation: {
                    (index, try await transform(input))
                }) else { throw CancellationError() }
                nextIndex += 1
            }
        }
        try Task.checkCancellation()
        // Every position was filled if collection completed without throwing.
        return results.map { $0! }
    }
}
```

The limit applies to child transforms in this invocation, not all app requests, HTTP connections, threads, or work a transform launches internally. It retains O(n) input/output storage and creates O(n) tasks over the whole operation. For unbounded streams, use a streaming design with backpressure rather than materializing every result. `addTaskUnlessCancelled` avoids adding work to an already-cancelled group; cancellation can still race with a successful addition.

#### Cleanup, racing, and cancellation shields

A first-result helper should call `cancelAll()` after selecting its winner, commonly in synchronous `defer` around `next()`. A normal early return from a regular group body does not itself cancel the losers. Even with cancellation, the group waits for every child; a timer child cannot impose a hard timeout on work that ignores cancellation. First completion, first success, and ignoring selected errors are distinct policies.

Swift 6.4 async `defer` can suspend while finishing a group body; both deferred cleanup and child completion must finish before the call returns. On OS 27, a cancellation shield can change parent-to-child propagation, but does not remove the structured lifetime guarantee. Explicit `group.cancelAll()` still cancels its children, and an unawaited async-let child still receives scope-exit cancellation. Shielding only `addTask` does not protect a child of an already-cancelled group. Use shielding for bounded necessary cleanup; do not shield a whole fan-out to make cancellation disappear. See D2 for runtime evidence and D7 for actor invariants during awaited cleanup. These standard-library concepts also apply outside Apple UI code; the recorded execution evidence here is from macOS, not a Linux/Windows runtime test.

## References

### Related guides

- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [Structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [GitHub. "async let." Swift Evolution SE-0317](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md)
- [GitHub. "TaskGroup.swift." swiftlang/swift](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift)
- [Apple Developer. "TaskGroup." Apple Developer Documentation](https://developer.apple.com/documentation/swift/taskgroup)
- [SE-0381 — DiscardingTaskGroups](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0381-task-group-discard-results.md)
- [SE-0504 — Task cancellation shields](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md)
