---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - main-actor
  - global-actors
---

# D5 — Global Actors and `@MainActor`

**Learning goals:**

- Choose whole-type, member, or global-actor-isolated conformance from state ownership.
- Separate UI-state permission from CPU execution and safely transfer inputs to off-actor work.
- Protect successful publication, errors, and cleanup when a UI request is replaced.

**Prerequisites:** [[D4 — Actors and Actor Isolation]] for actor isolation and [[D2 — Task Hierarchy, Cancellation, and Priorities]] for request freshness and cancellation.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An instance actor protects one instance. A **global actor** supplies one shared isolation domain for declarations across types and files. `MainActor` is the usual owner of UI state on Apple platforms. Annotating a declaration tells callers which isolation is required; it does not make an entire asynchronous workflow atomic.

A type annotation, a single member annotation, and a conformance annotation answer different questions: who owns this object's state, who may call this operation, and who may use its protocol witnesses? This repository keeps the target default `nonisolated` and expresses UI ownership explicitly. [SE-0316 — global actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Whole-type and member isolation

**Setup:** Standalone runnable script. The model belongs to the UI; the formatter only has one UI-owned member.

```swift
@MainActor final class ProfileModel {
    var title = "Profile"
    func rename(_ name: String) { title = name }
}
struct Formatter {
    func text(_ name: String) -> String { "Hello, \(name)" }
    @MainActor func apply(_ name: String, to model: ProfileModel) {
        model.rename(text(name))
    }
}
let model = ProfileModel()
Formatter().apply("Lea", to: model)
precondition(model.title == "Hello, Lea")
```

**Walkthrough:** Model state requires MainActor permission. Formatting independent string data does not. The member `apply` crosses that policy boundary explicitly. Top-level script execution already has MainActor permission; a nonisolated caller would await an actor-bound call.

### A custom global actor shares one domain

**Setup:** Standalone runnable script.

```swift
@globalActor actor MetadataActor {
    static let shared = MetadataActor()
    private init() {}
}
@MetadataActor enum MetadataCache {
    static var revision = 0
    static func bump() { revision += 1 }
}
@MetadataActor func refreshMetadata() { MetadataCache.bump() }
await refreshMetadata()
let revision = await MetadataCache.revision
precondition(revision == 1)
```

**Walkthrough:** The function and cache intentionally share the same global actor. Different instance actors would own different domains. A custom global actor does not allocate a dedicated thread, and Apple's MainActor integration is not a property of every actor on every platform.

## Rules and pitfalls

### Reusable computation does not need UI ownership

**Common pitfall:** A reusable service is marked MainActor solely because its first caller is a view model.

**Mistake:** Standalone **compile-only** declaration.

```swift
@MainActor struct Sorter {
    func sorted(_ values: [Int]) -> [Int] { values.sorted() }
}
```

**Consequence:** All callers must respect MainActor, and a large synchronous sort occupies that actor until completion.

**Why:** The annotation describes permission, not a request to run expensive work elsewhere.

**Correction:** Standalone **compile-only** service with transferable values and an explicit execution boundary.

```swift
struct Sorter: Sendable {
    @concurrent func sorted(_ values: [Int]) async throws -> [Int] {
        try Task.checkCancellation()
        let result = values.sorted()
        try Task.checkCancellation()
        return result
    }
}
```

**What changes:** The async call leaves the caller's actor while staying in the current task. Sorting itself does not check cancellation internally, so checks only bound the surrounding work. This does not promise a background thread or faster sorting.

### Cancellation does not replace request identity

**Common pitfall:** An old request's success is guarded, but its error and cleanup can still overwrite the latest UI state.

**Mistake:** Standalone **compile-only** model.

```swift
import Foundation
@MainActor final class SearchModel {
    var current: UUID?
    var error: String?
    func load(using service: @Sendable () async throws -> String) async {
        let id = UUID(); current = id
        defer { current = nil }
        do { _ = try await service(); guard current == id else { return } }
        catch { self.error = "Failed" }
    }
}
```

**Consequence:** After B replaces A, A's late ordinary error can set the error message and its defer can clear B's slot.

**Why:** Suspension allows another MainActor operation to replace the identity; actor safety does not preserve A's authority.

**Correction:** Standalone **compile-only** version.

```swift
import Foundation
@MainActor final class SearchModel {
    var current: UUID?
    var value: String?
    var error: String?
    func load(using service: @Sendable () async throws -> String) async {
        let id = UUID(); current = id
        defer { if current == id { current = nil } }
        do {
            let loaded = try await service()
            guard !Task.isCancelled, current == id else { return }
            value = loaded; error = nil
        } catch {
            guard !Task.isCancelled, current == id else { return }
            self.error = "Failed"
        }
    }
}
```

**What changes:** Every publication path checks cancellation and identity; cleanup clears only its own slot. The caller still owns the task and must decide when to cancel and join it. D2 supplies controlled late-success and late-error tests for this pattern.

## Worked example

### Isolate the conformance that reads UI state

**Shared setup — display model:** Standalone declarations; prepend them where named.

```swift
protocol Display { func text() -> String }
@MainActor final class Model: @MainActor Display {
    var title = "Hello"
    func text() -> String { title }
}
func render<T: Display>(_ value: T) -> String { value.text() }
```

**Setup:** Prepend **display model**. Runnable.

**Predict:** Can the synchronous generic helper use this conformance from MainActor?

```swift
let model = Model()
precondition(render(model) == "Hello")
```

**Outcome:** Yes. The conformance is explicitly MainActor-isolated, so its witness can read `title` when used from the permitted domain.

**Why:** `render` is generic and invokes the conformance's witness. An allowed synchronous caller can use it without erasing the conformance restriction. The nonisolated variant in the exercise rejects. [SE-0470 — isolated conformances](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md)

## Interview questions

### When should a whole type be MainActor, and when is a member annotation enough?

**Answer:** Isolate the whole type when its mutable invariants and identity belong to the UI. Use a member annotation when the remaining API has an independently valid contract outside that domain.

**Example:** Standalone **compile-only** mixed capability.

```swift
struct ProfilePresenter {
    func summary(name: String) -> String { "Profile: \(name)" }
    @MainActor func present(name: String, update: @MainActor (String) -> Void) {
        update(summary(name: name))
    }
}
```

**Walkthrough:** `summary` works with independent data. `present` requires permission to invoke the UI callback. This avoids making formatting itself depend on MainActor.

**Boundary:** A type full of UI-owned mutable properties usually benefits from one whole-type annotation rather than many scattered member annotations.

### What does MainActor isolation on a conformance permit that a nonisolated witness cannot do? Compare an isolated consumer with a generic consumer that crosses isolation.

**Answer:** It lets the witness access MainActor-owned state while restricting use of that conformance. Transferability of the value does not grant permission to use its isolated conformance anywhere.

**Example:** Prepend **display model**. This **intentionally rejected** nonisolated caller cannot use that conformance.

```swift
func outside(_ model: Model) -> String { render(model) }
```

**Walkthrough:** The class can be referenced outside MainActor, but the generic call needs its `Display` conformance. That use requires MainActor permission. The same `render` call succeeds from the worked example's actor context.

**Boundary:** A consumer that deliberately crosses isolation must also preserve a valid conformance domain. If a protocol must be usable everywhere, provide genuinely nonisolated witnesses over independent data rather than bypassing the check.

### Why does awaiting nonblocking I/O differ from sorting synchronously on MainActor, and what does @concurrent guarantee?

**Answer:** A nonblocking operation can suspend and free the actor for other work; synchronous CPU work keeps occupying it. `@concurrent` establishes an async boundary off the caller's actor without creating a new task.

**Example:** Standalone runnable script.

```swift
@concurrent func sortedOffActor(_ values: [Int]) async -> [Int] {
    precondition(#isolation == nil)
    return values.sorted()
}
@MainActor func updateList() async -> [Int] {
    await sortedOffActor([3, 1, 2])
}
let values = await updateList()
precondition(values == [1, 2, 3])
```

**Walkthrough:** The model awaits the off-actor function, whose static isolation is nil. When the call returns, the model resumes with its own actor permission. `#isolation` concerns actor isolation, not a physical thread ID. Nonblocking I/O needs no detached task merely because it waits.

**Boundary:** The sort remains synchronous inside the off-actor function and may need chunking for responsive cancellation. A plain nonisolated async helper follows caller isolation under this baseline; merely adding `async` would not establish this boundary.

### Why must a cancelled search guard ordinary errors and cleanup as well as successful results?

**Answer:** A cancelled service can finish late with either outcome, and old cleanup can erase a replacement. Check every path that writes shared UI state.

**Example:** Standalone runnable publication model; an old invocation has a different identity even if the input text repeats.

```swift
import Foundation
@MainActor final class Publication {
    var current = UUID()
    var error: String?
    func fail(_ id: UUID) {
        guard !Task.isCancelled, current == id else { return }
        error = "Failed"
    }
    func finish(_ id: UUID) { if current == id { current = UUID() } }
}
let state = Publication()
let old = state.current
state.current = UUID()
let latest = state.current
state.fail(old)
state.finish(old)
precondition(state.error == nil && state.current == latest)
```

**Walkthrough:** The old failure cannot publish and its cleanup cannot change the replacement's identity. Cancellation additionally handles a cancelled current request when there is no replacement.

**Boundary:** MainActor permission protects access, not the freshness policy. Owning a task handle, cancelling its external work, and prompt waiter cancellation are separate requirements.

## Focused exercise

**Task:** Use one explicit MainActor conformance through the same generic helper from allowed and disallowed callers.

**Acceptance checks:**

- MainActor can construct the model and call `render`.
- A truly nonisolated caller rejects use of its isolated conformance.
- Neither MainActor target defaults nor `assumeIsolated` is needed.

**Solution:** Prepend **display model**. Runnable permitted caller.

```swift
@MainActor func inside() -> String { render(Model()) }
precondition(inside() == "Hello")
```

Compile this separate **intentionally rejected** caller with the same shared declarations:

```swift
@concurrent func crossBoundary(_ model: Model) async -> String {
    render(model)
}
```

**Reasoning:** The conformance domain is checked at the generic use, even though the class reference is transferable. The permitted call retains that domain; the concurrent consumer intentionally leaves it.

## In practice

**Choose it when:** Ownership of state, operations, or witnesses belongs to a shared global actor.

**Investigate by:** Compile both UI and nonisolated clients. Check execution placement separately from actor access and request freshness.

**Strong evidence:** Explain why each annotation exists and how CPU work, cancellation, and protocol use behave at the boundary.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D4 — Actors and Actor Isolation]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [Global actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)
- [GitHub. "Actors." Swift Evolution SE-0306](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)
- [Swift.org. "Calling an Actor-Isolated Method from a Synchronous Nonisolated Context." Swift Compiler Diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/)
- [Swift.org. "Swift 6.2 Released." Swift.org Blog](https://swift.org/blog/swift-6.2-released/)
- [SE-0461 — Async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)
- [SE-0466 — Control default actor isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)

- [SE-0470 — isolated conformances](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md) — Supporting source used in the chapter.
