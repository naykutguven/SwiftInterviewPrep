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

**Learning goals:**

- Trace reached defer registrations on return, throw, nested scopes, and early exits.
- Register cleanup immediately after acquisition and distinguish scope exit from cancellation or fatal termination.
- Explain Swift 6.4 awaited defer and the separate policy for cancellation shielding.

**Prerequisites:** [[A5 — Pattern Matching and Exhaustive Control Flow]] for control flow and [[A13 — Error handling and typed throws]] for thrown errors. The extension uses [[D1 — Structured Concurrency Fundamentals]]/[[D2 — Task Hierarchy, Cancellation, and Priorities]] task and cancellation concepts.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

After opening a resource, every ordinary exit needs to release it. Repeating release in every return and catch is easy to get wrong. defer puts that obligation next to acquisition: when execution reaches defer, it registers work for exit from the enclosing **lexical scope**, the region delimited by braces.

Registered actions run in reverse order. An inner do block finishes its own actions before the outer function finishes its actions. A defer that execution never reaches registers nothing. This is ordinary scope cleanup, not a guarantee about a fatal trap or process termination. [Deferred actions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Register now, run on scope exit

The trace distinguishes registration order from execution order.

```swift
func trace() -> [String] {
    var events: [String] = []
    func work() {
        defer { events.append("first cleanup") }
        defer { events.append("second cleanup") }
        events.append("body")
    }
    work()
    return events
}
precondition(trace() == ["body", "second cleanup", "first cleanup"])
```

**Walkthrough:** The body runs first. On function exit, second cleanup runs before first cleanup, reversing the order in which they were reached.

### The return expression is evaluated before cleanup

An Int result is already computed when defer runs. A returned object still refers to storage that cleanup can mutate.

```swift
func number() -> Int {
    var value = 1
    defer { value = 2 }
    return value
}
final class Box { var value = 1 }
func object() -> Box {
    let box = Box()
    defer { box.value = 2 }
    return box
}
precondition(number() == 1)
precondition(object().value == 2)
```

**Walkthrough:** number returns the previously evaluated integer. object returns a reference to the same box that defer edits before the caller receives it. [[A1 — Value semantics, reference semantics, identity, and copy-on-write|A1]] explains this value/reference distinction.

## Rules and pitfalls

### Register immediately after acquiring the resource

**Common pitfall:** Putting cleanup after an early return that can already leave the resource acquired.

**Mistake:** Standalone runnable example.

```swift
final class Resource { var open = false }
func work(_ resource: Resource, valid: Bool) {
    resource.open = true
    guard valid else { return }
    defer { resource.open = false }
    _ = valid
}
let resource = Resource()
work(resource, valid: false)
precondition(resource.open)  // The missing cleanup is observable.
```

**Consequence:** The invalid-input path leaves the simulated resource open.

**Why:** Execution returns before reaching defer, so no cleanup was registered.

**Correction:** Register the release immediately after acquisition, before later work can return or throw.

```swift
final class Resource { var open = false }
func work(_ resource: Resource, valid: Bool) {
    resource.open = true
    defer { resource.open = false }
    guard valid else { return }
    _ = valid
}
let resource = Resource()
work(resource, valid: false)
precondition(!resource.open)
work(resource, valid: true)
precondition(!resource.open)
```

**What changes:** Both ordinary exits now share the release obligation. Only register cleanup after acquisition actually succeeds.

### Cancellation does not itself unwind a scope

The test uses withUnsafeCurrentTask only to cancel its own current task synchronously. It neither stores nor transfers that temporary task reference. This establishes the cancellation flag before the next statement without a scheduling or timing assumption.

**Common pitfall:** Expecting a cancellation flag to prevent later work and trigger immediate cleanup.

**Mistake:** Standalone runnable example.

```swift
let task = Task { () -> [String] in
    var events: [String] = []
    func work() {
        defer { events.append("cleanup") }
        withUnsafeCurrentTask { $0?.cancel() }
        precondition(Task.isCancelled)
        events.append("continued")
    }
    work()
    return events
}
let observed = await task.value
precondition(observed == ["continued", "cleanup"])
```

**Consequence:** The task continues after cancellation; cleanup waits for actual scope exit.

**Why:** Cancellation is cooperative state. It does not force a return or throw.

**Correction:** Observe cancellation at a meaningful boundary. This example checks it before continuing the simulated work.

```swift
let task = Task { () -> [String] in
    var events: [String] = []
    func work() throws {
        defer { events.append("cleanup") }
        withUnsafeCurrentTask { $0?.cancel() }
        try Task.checkCancellation()
        events.append("continued")
    }
    do { try work() } catch is CancellationError { events.append("cancelled") } catch {
        preconditionFailure("Unexpected error")
    }
    return events
}
let observed = await task.value
precondition(observed == ["cleanup", "cancelled"])
```

**What changes:** checkCancellation throws, causing an ordinary scope exit and then cleanup. An API that ignores cancellation can still keep its scope alive; [[D2 — Task Hierarchy, Cancellation, and Priorities|D2]] explains propagation.

## Worked example

### Nested scopes finish before the outer return

**Setup:** Standalone script. Returning exits the do scope and then the function scope.

**Predict:** Which cleanup runs before the caller appends returned?

```swift
var events: [String] = []
func work(_ events: inout [String]) {
    defer { events.append("outer") }
    do {
        defer { events.append("inner") }
        events.append("body")
        return
    }
}
work(&events)
events.append("returned")
precondition(events == ["body", "inner", "outer", "returned"])
```

**Outcome:** body, inner, outer, returned.

**Why:** The return exits both scopes. Each scope finishes its registered work before control reaches the caller.

## Interview questions

### In what order do multiple defer blocks execute?

**Answer:** In reverse registration order in each exited scope, with inner-scope cleanup before outer-scope cleanup. Skipped registrations never enter that order.

**Example:** Standalone runnable example.

```swift
func trace(_ early: Bool) -> [String] {
    var events: [String] = []
    func work() {
        defer { events.append("outer first") }
        if early { return }
        defer { events.append("outer second") }
        do {
            defer { events.append("inner") }
            events.append("body")
        }
    }
    work()
    return events
}
precondition(trace(true) == ["outer first"])
precondition(trace(false) == ["body", "inner", "outer second", "outer first"])
```

**Walkthrough:** The early path reaches only the first registration. The other path exits the inner do before the outer scope and then reverses the two outer registrations.

**Boundary:** The order concerns reached lexical scopes. It does not promise cleanup after fatalError, memory corruption, or external process termination.

### Where is defer clearer than duplicated cleanup in catch and return paths?

**Answer:** It is clearer when a successfully acquired resource has one release obligation shared by several exits. Register that obligation once and keep business decisions in ordinary control flow.

**Example:** Standalone runnable example.

```swift
enum Failure: Error { case read }
final class Resource { var releases = 0 }
func read(_ resource: Resource, empty: Bool, fail: Bool) throws -> String {
    defer { resource.releases += 1 }
    if empty { return "" }
    if fail { throw Failure.read }
    return "data"
}
let resource = Resource()
let empty = try read(resource, empty: true, fail: false)
precondition(empty == "")
do { _ = try read(resource, empty: false, fail: true) } catch {}
let data = try read(resource, empty: false, fail: false)
precondition(data == "data")
precondition(resource.releases == 3)
```

**Walkthrough:** The empty return, thrown error, and successful return each run one cleanup. Adding another early return would inherit the same obligation.

**Boundary:** A throwing cleanup needs an explicit policy. Handle its error inside defer or use explicit finalization if the caller must receive it; do not silently replace the original failure.

### Extension: can defer await, and when is cleanup complete relative to return or catch?

**Answer:** Swift 6.4 permits await in defer inside an async context. Return or error propagation waits for that cleanup, in the same reverse order. Awaiting does not automatically shield cancellation.

**Example:** Standalone runnable example.

```swift
enum Failure: Error { case work }
@MainActor final class Journal {
    var events: [String] = []
    func cleanup() async {
        await Task.yield()
        events.append("async cleanup")
    }
}
@MainActor func work(_ journal: Journal, fail: Bool) async throws {
    defer { journal.events.append("outer") }
    defer { await journal.cleanup() }
    journal.events.append("body")
    if fail { throw Failure.work }
}
let journal = Journal()
try await work(journal, fail: false)
precondition(journal.events == ["body", "async cleanup", "outer"])
journal.events = []
do { try await work(journal, fail: true) } catch { journal.events.append("caught") }
precondition(journal.events == ["body", "async cleanup", "outer", "caught"])
```

**Walkthrough:** Task.yield introduces a possible suspension. The caller still observes completed cleanup before receiving a result or entering catch. [SE-0493](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)

**Boundary:** Cancellation-sensitive cleanup can still observe cancellation. Never hold a thread-owned lock across suspension. Bound teardown work and choose how cleanup failure is reported.

### Cancellation shielding — optional

The following standalone script defines all its helpers. withTaskCancellationShield is available on anyAppleOS 27 in this SDK. It is useful only for a required, bounded cleanup step; a shield cannot make a hung operation finish or make failing I/O succeed. No lock is held here. [SE-0504](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md)

```swift
@MainActor final class Resource {
    var released = false
    func cleanup() async {
        await Task.yield()
        released = true
    }
}
@available(anyAppleOS 27.0, *)
@MainActor func protectedWork(_ resource: Resource) async {
    defer {
        await withTaskCancellationShield { await resource.cleanup() }
    }
    withUnsafeCurrentTask { $0?.cancel() }
}
let resource = Resource()
let task = Task { await protectedWork(resource) }
await task.value
precondition(resource.released)
```

## Focused exercise

**Task:** Trace nested cleanup on return, throw, and an early return before a later registration.

**Acceptance checks:**

- Both ordinary return and throw run inner cleanup before reverse-ordered outer cleanup.
- An outer catch observes completed cleanup.
- The early path skips every registration it does not reach.

**Solution:** Standalone script; each event records an actual control-flow step.

```swift
enum Failure: Error { case stopped }
enum Exit { case early, returned, thrown }
func trace(_ exit: Exit) -> [String] {
    var events: [String] = []
    func work() throws {
        defer { events.append("outer first") }
        if exit == .early { return }
        defer { events.append("outer second") }
        do {
            defer { events.append("inner") }
            events.append("body")
            if exit == .thrown { throw Failure.stopped }
            return
        }
    }
    do { try work() } catch { events.append("caught") }
    return events
}
precondition(trace(.early) == ["outer first"])
precondition(trace(.returned) == ["body", "inner", "outer second", "outer first"])
precondition(trace(.thrown) == ["body", "inner", "outer second", "outer first", "caught"])
```

**Reasoning:** A thrown error and a return both leave the inner scope before the outer one. The early path establishes why textual presence of a defer is insufficient: execution must reach it. These assertions concern ordinary control flow, not fatal termination.

## In practice

**Choose it when:** Pair acquisition and release in one lexical scope. Use an explicit lifecycle API when ownership outlives that scope.

**Investigate by:** Record reached registrations and actual scope exits, including cancellation checks and errors.

**Strong evidence:** Define bounded asynchronous teardown, cancellation shielding, and cleanup-failure policy, then verify completion with controlled events rather than elapsed-time sleeps.

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
