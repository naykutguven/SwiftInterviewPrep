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

**Goal:** Use an explicitly MainActor-isolated conformance from an allowed caller and reject a nonisolated use.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A global actor is one shared isolation domain spanning declarations across types and files. MainActor fits UI-owned state; an instance actor instead protects one actor instance's state. Isolating a whole type, one member, and one protocol conformance are separate choices. This collection keeps target defaults nonisolated and states required UI ownership explicitly. [Global actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)

## Rules and pitfalls

### Put UI state under one clear owner

Use whole-type MainActor when the object's identity and mutable state are UI-affine. Use member isolation for a genuinely mixed capability with a narrow UI operation. Avoid scattering MainActor.run blocks around an otherwise unspecified ownership model.

**Common pitfall:** Marking a reusable service MainActor just because its caller is UI forces all clients through that actor. Keep independent services under their appropriate value, actor, or synchronization contract.

### Separate UI ownership from CPU execution

Nonblocking I/O can suspend without occupying MainActor. A synchronous sort called on MainActor occupies that actor until it finishes. A plain nonisolated async helper inherits caller isolation here; @concurrent explicitly leaves it while preserving current-task ownership.

**Common pitfall:** Actor safety does not prevent stale results. Guard successful updates, ordinary errors after cancellation, and cleanup with the current request identity.

## Interview questions

### When should a whole type be MainActor, and when is a member annotation enough?

Use whole-type isolation when its invariants and identity belong to the UI domain. A member annotation is enough when the type's remaining API can correctly operate elsewhere. The choice should follow ownership, not a desire to eliminate a diagnostic at one call site.

### What does MainActor isolation on a conformance permit that a nonisolated witness cannot do? Compare an isolated consumer with a generic consumer that crosses isolation.

It permits witnesses to read and mutate MainActor-owned state while restricting use of that conformance to compatible contexts. A synchronous generic helper can preserve the caller's allowed conformance use; a nonisolated caller cannot use the isolated conformance, and a consumer crossing isolation must satisfy that boundary too. Sendable value transfer is a separate question.

### Why does awaiting nonblocking I/O differ from sorting synchronously on MainActor, and what does @concurrent guarantee?

Awaiting genuinely nonblocking I/O permits other actor work while suspended. Synchronous CPU work occupies the actor. @concurrent switches the async function off the caller's actor, requiring safe argument/result transfer; it does not create a new task or guarantee a particular physical thread, parallelism, or faster execution.

### Why must a cancelled search guard ordinary errors and cleanup as well as successful results?

A cancellation-insensitive request can return success or throw an ordinary error after replacement. Old cleanup can also clear current state. Check both cancellation and the unique request identity wherever state is published or cleared; query equality alone fails when a query repeats.

## Focused exercise

**Task:** Give a MainActor model an explicit MainActor protocol conformance. Call the same generic helper from MainActor and from a nonisolated caller; explain which use is permitted.

**Acceptance checks:**

- MainActor can construct the model and use its conformance through render.
- The same render call in outside rejects.
- No MainActor target default or unsafe assumeIsolated escape is needed.

**Solution:**

Standalone successful client; compile with -parse-as-library:

```swift
protocol Display { func text() -> String }
@MainActor final class Model: @MainActor Display {
    var title = "Hello"
    func text() -> String { title }
}
func render<T: Display>(_ value: T) -> String { value.text() }
@main struct Probe {
    @MainActor static func main() {
        precondition(render(Model()) == "Hello")
    }
}
```

Add this function in a separate intentionally rejected variant, reusing the same declarations:

```swift
func outside(_ value: Model) -> String { render(value) }
```

The error concerns use of the MainActor-isolated conformance from a nonisolated context, not the helper's method spelling.

**Reasoning:** render uses a protocol witness synchronously. Its caller must be permitted to use the supplied conformance; the UI caller is, outside is not. A reusable nonisolated conformance would need witnesses that do not require protected mutable UI state.

## In practice

**Choose it when:** State and conformance ownership are intrinsically tied to one global actor.

**Investigate by:** Compile callers inside and outside the intended actor, including generic consumers.

**Strong evidence:** Design a useful UI protocol boundary and separately validate CPU placement, cancellation, and freshness.

### Custom global actors — optional

#### A custom global actor is one shared domain

This standalone declaration isolates members across separate declarations to one global actor. It does not allocate a dedicated thread or make arbitrary external state safe.

```swift
@globalActor
actor MetadataActor {
    static let shared = MetadataActor()
    private init() {}
}

@MetadataActor
enum MetadataCache {
    static var revision = 0
    static func bump() { revision += 1 }
}

@MetadataActor
func refreshMetadata() {
    MetadataCache.bump()
}
```

A separate instance actor would have a separate isolation domain per instance. A custom global actor is useful when declarations intentionally share one domain; it is not a performance fix by itself. The Apple UI main-thread integration belongs to MainActor, not every global actor or every platform. Prefer isolation checks over thread-ID assumptions in portable examples.

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
