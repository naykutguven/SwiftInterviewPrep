---
tags:
  - "#swift"
  - interview-prep
  - core-semantics
  - defer
  - clean-up
  - control-flow
---

# A6 — `defer`, Early Exits, and Cleanup Semantics

**Goal:** Trace reached cleanup registrations across nested return and throw paths.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Defer registers work when execution reaches it and runs that work on exit from the enclosing lexical scope. Registered actions run in reverse order. An inner do or loop scope exits before the outer function, so cleanup belongs to that scope rather than an assumed function-end queue. [Control flow: deferred actions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/)

## Rules and pitfalls

### Register cleanup immediately after acquisition

Place defer after a resource is successfully acquired, before any later operation can return or throw. Keep business decisions in ordinary control flow; cleanup should restore the resource invariant.

**Common pitfall:** Registering after a guard or throwing operation leaves earlier exits unprotected. Move registration to the first point at which the resource needs cleanup. A defer never reached cannot run.

### Separate scope exit, cancellation, and fatal termination

Cancellation is a cooperative state change; it does not itself unwind a running function. A fatal trap or process termination does not guarantee cleanup. Return expressions are evaluated before deferred work; a copied result can differ from a returned reference whose object is later mutated.

**Common pitfall:** Treating cancellation as immediate teardown overlooks work that ignores it. Check cancellation at suitable boundaries and bound cleanup independently.

### Await cleanup deliberately

Swift 6.4 permits awaited operations in defer inside async contexts. The enclosing return or error propagation waits for them. Awaiting does not shield cancellation; a cancellation-sensitive cleanup can still observe cancellation. Shield only required bounded teardown, with an explicit failure policy. [SE-0493](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)

## Interview questions

### Can a defer await, and when is its cleanup complete relative to return or catch?

Yes, an async context can await in defer in Swift 6.4. Cleanup finishes before the return reaches its caller or the error reaches an outer catch. The task may already be cancelled; use a deliberate shielding policy when necessary, rather than assuming await disables cancellation.

### In what order do multiple `defer` blocks execute?

They execute in reverse registration order within each exited scope. Inner scopes finish their reached defers before outer scopes finish theirs. A registration skipped by control flow contributes no action. This order follows control flow, not the textual position of every defer in the file.

### Where does `defer` become clearer than duplicating cleanup code in `catch` and `return` paths?

It is clearer when several successful acquisition paths share the same release requirement despite different exits. One immediately registered cleanup prevents return and catch branches from drifting. If cleanup can fail, decide whether to log, combine, or report that failure without hiding the original outcome.

## Focused exercise

**Task:** Register cleanup in nested scopes with return and throw paths. Trace reached defers and verify reverse registration order in each scope.

**Acceptance checks:**

- Both return and throw run inner cleanup before reverse-ordered outer cleanup.
- The catch observes completed cleanup.
- An early exit before a new defer registration does not run that registration.

**Solution:**

Standalone synchronous trace; ordinary errors only:

```swift
enum Failure: Error { case stopped }
func trace(throwing: Bool) -> [String] {
    var events: [String] = []
    func work() throws {
        defer { events.append("outer first") }
        defer { events.append("outer second") }
        do {
            defer { events.append("inner") }
            events.append("body")
            if throwing { throw Failure.stopped }
            return
        }
    }
    do { try work() } catch { events.append("caught") }
    return events
}
precondition(trace(throwing: false) == ["body", "inner", "outer second", "outer first"])
precondition(trace(throwing: true) == ["body", "inner", "outer second", "outer first", "caught"])
```

**Reasoning:** Returning from work first exits the inner do scope and then the function scope. The enclosing catch runs only after work has unwound. This checks lexical cleanup; it makes no promise about fatal termination or a noncooperating task.

## In practice

**Choose it when:** Pair acquisition and release in one lexical scope; use explicit lifecycle APIs when ownership outlives that scope.

**Investigate by:** Record reached registrations and each scope exit, including thrown errors.

**Strong evidence:** Define cancellation and cleanup-failure policy, then verify async teardown completion with controlled gates and bounded timeouts.

### Asynchronous cleanup and shielding — optional

#### Waiting for cleanup and shielding cancellation are separate decisions

Async `defer` does not itself mask cancellation. If teardown calls a cancellation-sensitive API, decide whether to shield that small operation. `withTaskCancellationShield` requires the newer runtime; the installed SDK marks it `@available(anyAppleOS 27.0, *)`. See D2 for the full cancellation contract.

```swift
@available(anyAppleOS 27.0, *)
func performWithProtectedCleanup() async {
    defer {
        await withTaskCancellationShield {
            await cleanup()
        }
    }
    print("body")
}
```

A shield does not make failing I/O succeed or ensure a hung operation completes. Give cleanup a failure policy and bounded resource usage. Handle cleanup errors inside the defer; use an explicit finalization path if callers need to receive that error. Do not hold a thread-owned lock across suspension. [SE-0504](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md)

#### Swift 6.4 probe: cleanup completes before return or catch

Standalone probe; compile with `-parse-as-library -swift-version 6`. The async step yields deliberately; the assertions check ordering, not elapsed time.

```swift
enum Failure: Error { case work }
@MainActor var events: [String] = []
@MainActor func cleanup(_ label: String) async {
    await Task.yield()
    events.append(label)
}
@MainActor func work(_ fail: Bool) async throws {
    defer { events.append("outer") }
    defer { await cleanup("async") }
    events.append("body")
    if fail { throw Failure.work }
}
@main struct Main {
    static func main() async {
        try! await work(false)
        precondition(events == ["body", "async", "outer"])
        events = []
        do { try await work(true) } catch { events.append("caught") }
        precondition(events == ["body", "async", "outer", "caught"])
        print(events.joined(separator: ","))
    }
}
```

Observed on the 6.4 baseline: `body,async,outer,caught`.

##### Nested early return still waits in reverse order

```swift
@MainActor
func nestedCleanup() async {
    func record(_ event: String) async {
        await Task.yield()
        print(event)
    }
    defer { print("outer") }
    do {
        defer { await record("first") }
        defer { await record("second") }
        print("body")
        return
    }
}
```

Awaiting nestedCleanup() prints body, second, first, outer. Return is an implicit waiting point for the two async defers; they are not started as parallel unstructured tasks.

## References

### Related guides

- [[A3 — Initialization model]]
- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A13 — Error handling and typed throws]]
- [[C1 — ARC fundamentals]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]

### Official sources

- [Control flow: deferred actions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/)
- [SE-0493](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)
- [SE-0504](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md)
- [Swift.org. "Error Handling." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/)
- [Swift Forums. "Details of defer statement in Swift."](https://forums.swift.org/t/details-of-defer-statement-in-swift/4483)
- [Apple Developer. "Mutex." Apple Developer Documentation](https://developer.apple.com/documentation/Synchronization/Mutex)
