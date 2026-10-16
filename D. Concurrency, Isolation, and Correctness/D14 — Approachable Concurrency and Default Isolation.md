---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - swift6-4
  - main-actor
  - approachable-concurrency
aliases:
  - D14 — Swift 6.2 Approachable Concurrency and New Defaults
---

# D14 — Approachable Concurrency and Default Isolation

**Goal:** Predict the actor used by three async execution contracts from actor-isolated and nonisolated callers.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Declaration isolation says who may access a declaration. Async execution policy says where a call executes. Under this baseline, an ordinary unannotated type remains nonisolated, while a plain nonisolated async helper uses its caller's actor. A nonisolated caller supplies no actor. Suspension does not imply a background thread. [SE-0461: async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md#detailed-design)

Approachable Concurrency and Default Actor Isolation are separate settings. The bundle's additional Swift 6 opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`; other bundled features are already enabled by Swift 6 mode. Explicit isolation and superclass, protocol, or conformance inference can still apply. [SE-0466: default isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)

## Rules and pitfalls

### Make an off-actor boundary explicit

Plain nonisolated async and explicit `nonisolated(nonsending)` use caller isolation here. `@concurrent` switches off the actor and permits its work to run concurrently with work on the caller's actor. It creates no new task: cancellation, task locals, and structured lifetime still belong to the current task. Cross-boundary arguments and results must be safely transferable.

**Common pitfall:** Making a parser async, or moving it into a nonisolated target, does not offload its CPU work. Use `@concurrent` when leaving the actor is an API requirement, then measure responsiveness and check cancellation. Do not infer a physical thread from an actor identity.

### Keep ownership and defaults separate

Explicit `@MainActor` is appropriate for UI-owned mutable state. A UI-only target may alternatively choose a MainActor default; that is an override of this collection's baseline. General libraries often benefit from nonisolated defaults so callers can choose their execution context.

**Common pitfall:** Changing client defaults does not recompile an imported module's isolation contract. Inspect the callee's interface and compilation policy. A legacy plain async helper can retain off-actor behavior even when its client enables caller-isolated defaults.

### Infer a conformance only where its witnesses are valid

With `InferIsolatedConformances`, a MainActor type can infer a MainActor-isolated conformance when its witnesses need that actor. The conformance can be used only in a context that respects that isolation. Witnesses that can satisfy requirements nonisolated may instead support a nonisolated conformance; explicitly design reusable value conformances rather than assuming every conformance of an isolated type is isolated. [SE-0470: isolated conformances](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md#inferring-isolated-conformances)

**Common pitfall:** An isolated conformance does not make arbitrary off-actor generic use safe. Check the generic caller and its required conformance domain; see [[B7 — Synthesized conformances and semantic correctness]].

## Interview questions

### Why does enabling Approachable Concurrency leave an ordinary type nonisolated here?

The baseline explicitly keeps Default Actor Isolation nonisolated. The feature bundle changes async and conformance rules without choosing MainActor as a module default. Annotate a UI model explicitly when its state belongs to MainActor.

### Which actor does a MainActor caller's plain async helper use?

It uses MainActor under the baseline. `@concurrent` switches off that actor; a nonisolated caller instead supplies no actor to either caller-isolated spelling. Synchronous nonisolated code continues on the caller's execution context but has no actor-isolated access rights merely because of that execution.

### When should explicit execution annotations replace reliance on defaults?

Use `nonisolated(nonsending)` to stabilize caller-isolated behavior across modules with differing settings, and `@concurrent` when off-actor execution is the promised contract. Do not repeat `nonisolated` on ordinary declarations where it changes nothing.

### How can a MainActor type provide a nonisolated conformance?

Its witnesses must be callable without the actor, for example by using immutable sendable state or pure computations. If a witness needs protected mutable state, retain an isolated conformance and constrain its callers accordingly. Verify both allowed actor use and rejected off-actor use.

## Focused exercise

**Task:** Call plain async, explicit caller-isolated async, and concurrent helpers from MainActor and from a genuinely nonisolated execution context.

**Acceptance checks:**

- The MainActor call observes an actor for the first two helpers and none for the third.
- The off-actor call observes none for all three.
- No result is presented as a thread guarantee or a scheduling-order guarantee.

**Solution:** Standalone executable; compile with F5's baseline flags and `-parse-as-library`.

```swift
func plain() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
nonisolated(nonsending) func callerBound() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
@concurrent func concurrentWork() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
@main struct Probe {
    @MainActor static func main() async {
        let first = await plain()
        let second = await callerBound()
        let third = await concurrentWork()
        precondition(first && second && !third)
        await checkOutside()
    }
    @concurrent static func checkOutside() async {
        let first = await plain()
        let second = await callerBound()
        let third = await concurrentWork()
        precondition(!first && !second && !third)
    }
}
```

**Reasoning:** `checkOutside` establishes an off-actor caller; an unannotated async wrapper alone would inherit MainActor under the baseline. Await results into locals before synchronous assertion autoclosures. The optional actor local avoids relying on a previously observed compiler issue with direct macro comparisons.

## In practice

**Choose it when:** Establish execution intent at reusable API boundaries. Keep stateful services on an appropriate actor and offload independent expensive computation explicitly.

**Investigate by:** Inspect emitted flags, interfaces, and `#isolation` probes; use profiling for CPU placement and responsiveness.

**Strong evidence:** Compile a library and clients with different settings, verify isolation and conformance use at their boundary, and distinguish a source contract change from runtime observations.

### Alternative settings — optional

| Default isolation | Caller-isolated default | Eligible ordinary class | Explicit plain nonisolated async |
| --- | --- | --- | --- |
| nonisolated | enabled (baseline) | nonisolated | caller's actor |
| nonisolated | disabled | nonisolated | off actor |
| MainActor | enabled | MainActor, subject to inference | caller's actor |
| MainActor | disabled | MainActor, subject to inference | off actor |

Explicit `nonisolated(nonsending)` and `@concurrent` retain their respective behavior in every row. To compare MainActor defaults, keep explicit `nonisolated` on the *comparison* helper so inference does not change the declaration being tested. `Task {}` in a lexically nonisolated helper does not gain actor inheritance merely because that helper executes on the caller's actor; see [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]].

## References

### Related guides

- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[D13 — Swift 6 Strict Concurrency Migration]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [SE-0461: async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md#detailed-design)
- [SE-0466: default isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
- [SE-0470: isolated conformances](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md#inferring-isolated-conformances)
- [Swift.org. "Swift 6.2 Released." Swift.org Blog](https://swift.org/blog/swift-6.2-released/)
- [GitHub. "Run Nonisolated Async Functions on the Caller's Actor by Default." Swift Evolution SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)
