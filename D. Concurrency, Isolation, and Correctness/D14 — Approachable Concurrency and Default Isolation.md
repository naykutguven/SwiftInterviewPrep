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

**Learning goals:**

- Separate default declaration isolation from caller-isolated async execution and conformance inference.
- Predict plain async, explicit caller-bound async, and concurrent calls from MainActor and off-actor contexts.
- Verify isolated/nonisolated conformances and imported contracts under actual module settings.

**Prerequisites:** [[D5 — Global Actors and `@MainActor`]]–[[D6 — `Sendable`, `@Sendable`, and `sending`]] for global-actor conformances and safe transfer; [[F5 — Language mode, build settings, and migration flags]] for target settings. Thread identity is deliberately not used as evidence.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Two settings answer different questions. **Default Actor Isolation** decides whether eligible unannotated declarations belong to MainActor or are nonisolated. **Approachable Concurrency** changes selected inference and execution rules; it does not itself choose MainActor as the target default.

Here the default remains nonisolated. In Swift 6 mode the two additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`; other features in the bundle are already enabled by Swift 6 mode. Explicit annotations and superclass/protocol rules can still impose isolation.

With caller-isolated defaults, an ordinary nonisolated async helper uses the caller's actor. An off-actor caller supplies no actor. Explicit `nonisolated(nonsending)` fixes caller-bound behavior; `@concurrent` explicitly leaves the actor while retaining the current task. Declaration permission, execution context, and physical thread identity are different facts. [SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Shared setup — execution contracts

These compile-only helpers return whether their execution has an actor. A typed local stores the #isolation observation before comparison; it is not a thread probe.

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
```

### A MainActor caller supplies its actor

**Setup:** Prepend **execution contracts**. Runnable baseline.

```swift
@MainActor func checkOnMain() async -> [Bool] {
    let first = await plain()
    let second = await callerBound()
    let third = await concurrentWork()
    return [first, second, third]
}
let observed = await checkOnMain()
precondition(observed == [true, true, false])
```

**Walkthrough:** The first two calls retain the caller's actor, while concurrentWork leaves it. There is no new Task initializer in any helper; task cancellation and task-local context remain attached to the current task.

### A genuinely off-actor caller supplies no actor

**Setup:** Prepend **execution contracts**. Runnable.

```swift
@concurrent func checkOutside() async -> [Bool] {
    let first = await plain()
    let second = await callerBound()
    let third = await concurrentWork()
    return [first, second, third]
}
let observed = await checkOutside()
precondition(observed == [false, false, false])
```

**Walkthrough:** The explicit concurrent entry establishes an off-actor caller. Simply calling another ordinary async wrapper from MainActor would not establish this condition under the baseline.

### Shared setup — separately compiled contracts

Save this source as Contracts.swift. Build it twice with the D1 baseline command's settings and `-parse-as-library -emit-module -c`: once as DefaultsOff with the caller-isolated opt-in omitted, once as DefaultsOn with that opt-in present. Keep the default isolation nonisolated in both modules. Use `-module-name DefaultsOff -emit-module-path DefaultsOff.swiftmodule -o DefaultsOff.o` and the corresponding DefaultsOn names.

**DefaultsOff source:** Compile-only library, explicitly built without `NonisolatedNonsendingByDefault`. The explicit nonisolated spelling keeps comparisons about async execution rather than declaration inference.

```swift
public nonisolated func actorPresent() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
public nonisolated(nonsending) func explicitCaller() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
@concurrent public func explicitConcurrent() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
```

**DefaultsOn source:** The identical source below is built under the full baseline. Only its module name and compilation settings differ.

```swift
public nonisolated func actorPresent() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
public nonisolated(nonsending) func explicitCaller() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
@concurrent public func explicitConcurrent() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
```

Library interfaces retain those contracts. For runnable clients add the module directory with `-I .` and link both library objects. For an object-only client, imports need only `-I .`.

## Rules and pitfalls

### Async does not promise CPU offloading

**Common pitfall:** A parser is made async and expected to leave MainActor.

**Mistake:** Standalone runnable counterexample under the baseline.

```swift
func parse(_ values: [Int]) async -> [Int] {
    let actor: (any Actor)? = #isolation
    precondition(actor === MainActor.shared)
    return values.sorted()
}
@MainActor func render() async -> [Int] { await parse([2, 1]) }
let values = await render()
precondition(values == [1, 2])
```

**Consequence:** The synchronous sort still occupies MainActor. The async signature permits suspension but does not insert one into sorting.

**Why:** Caller-isolated execution retained the UI actor.

**Correction:** Standalone runnable explicit off-actor boundary.

```swift
@concurrent func parse(_ values: [Int]) async throws -> [Int] {
    let actor: (any Actor)? = #isolation
    precondition(actor == nil)
    try Task.checkCancellation()
    let result = values.sorted()
    try Task.checkCancellation()
    return result
}
@MainActor func render() async throws -> [Int] { try await parse([2, 1]) }
let values = try await render()
precondition(values == [1, 2])
```

**What changes:** The operation leaves the caller's actor without creating an unstructured task. Inputs/results are transferable values. Sorting itself does not poll cancellation, and off-actor does not mean a particular physical thread, more parallelism, or faster execution.

### Client settings do not rewrite a library's contract

**Common pitfall:** Enabling caller-isolated defaults in the client is expected to change an already-compiled library's ordinary async helper.

**Mistake:** Compile-only client after building **DefaultsOff**. The caller is MainActor; the callee still returns false.

```swift
import DefaultsOff
@MainActor func caller() async -> Bool { await DefaultsOff.actorPresent() }
```

**Consequence:** The imported plain helper keeps its off-actor contract even though this client enables the new default.

**Why:** The client compiles calls to an imported interface; it does not recompile the library's declarations under its own flags.

**Correction:** Runnable client after building **DefaultsOff**. Use an explicitly stable API contract when that is the requirement.

```swift
import DefaultsOff
let stays = await DefaultsOff.explicitCaller()
let leaves = await DefaultsOff.explicitConcurrent()
precondition(stays && !leaves)
```

**What changes:** Explicit caller-bound and concurrent functions retain their intended behavior across settings. A library can instead deliberately change its plain-function contract by rebuilding; that is an API-policy change, not a client-only switch.

### Isolated conformance is not universal conformance permission

**Shared setup — display:** The MainActor class's witness reads mutable UI state. With InferIsolatedConformances enabled, its conformance is inferred isolated. Prepend **display** where named.

```swift
protocol Display { func text() -> String }
@MainActor final class Model: Display {
    var title = "Profile"
    func text() -> String { title }
}
func render<T: Display>(_ value: T) -> String { value.text() }
```

**Common pitfall:** A transferable reference to Model is treated as permission to use its isolated Display conformance from anywhere.

**Mistake:** Prepend **display**. **Intentionally rejected** client.

```swift
@concurrent func outside(_ model: Model) async -> String { render(model) }
```

**Consequence:** The generic use crosses the conformance's allowed domain.

**Why:** Sendability of the reference and permission to invoke its witness are independent requirements.

**Correction:** Prepend **display**. Runnable MainActor client.

```swift
@MainActor func inside() -> String { render(Model()) }
precondition(inside() == "Profile")
```

**What changes:** The witness is used in its permitted context. A protocol intended for off-actor use needs nonisolated witnesses over independent data, demonstrated below. [SE-0470](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md)

## Worked example

### Compare two library contracts from one client

**Setup:** Build **DefaultsOff** and **DefaultsOn**, then link both objects with this runnable baseline client.

**Predict:** Can the same client observe different behavior from identical library source compiled under different settings?

```swift
import DefaultsOff
import DefaultsOn
let oldPlain = await DefaultsOff.actorPresent()
let newPlain = await DefaultsOn.actorPresent()
let oldExplicit = await DefaultsOff.explicitCaller()
let newExplicit = await DefaultsOn.explicitCaller()
precondition(!oldPlain && newPlain && oldExplicit && newExplicit)
```

**Outcome:** Yes: plain returns false from DefaultsOff and true from DefaultsOn. Explicit caller-bound behavior is true from both when called on MainActor.

**Why:** The libraries' interfaces encode different ordinary async contracts. The client has one settings set and one actor context, so the difference comes from the callee module. The validation record also checks the same imported contracts with caller defaults omitted in the client.

## Interview questions

### Why does enabling Approachable Concurrency leave an ordinary type nonisolated here? Where is explicit MainActor appropriate?

**Answer:** The baseline separately chooses nonisolated default isolation. The bundle does not choose MainActor for every type. Explicitly isolate UI-owned mutable state.

**Example:** Standalone runnable contrast.

```swift
final class Service { var count = 0 }
@MainActor final class Screen { var title = "Profile" }
@concurrent func useIndependentService() async -> Int {
    let service = Service()
    service.count += 1
    return service.count
}
let count = await useIndependentService()
let screen = Screen()
precondition(count == 1 && screen.title == "Profile")
```

**Walkthrough:** The ordinary service can be created and used entirely in the off-actor operation. The screen requires MainActor permission, provided by top-level code.

**Boundary:** Superclasses, protocols, explicit annotations, and other inference can still isolate a declaration. Do not infer every declaration's contract from the module default alone. [SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)

### Which actor does a MainActor caller's plain nonisolated async helper use, and what changes with concurrent or an off-actor caller?

**Answer:** The plain helper uses MainActor under the baseline. Concurrent explicitly leaves it; a genuinely off-actor caller supplies no actor to either caller-bound spelling.

**Example:** Prepend **execution contracts**. Runnable local comparison.

```swift
@MainActor func fromUI() async -> [Bool] {
    let plainResult = await plain()
    let boundResult = await callerBound()
    let concurrentResult = await concurrentWork()
    return [plainResult, boundResult, concurrentResult]
}
@concurrent func fromOutside() async -> [Bool] {
    let plainResult = await plain()
    let boundResult = await callerBound()
    let concurrentResult = await concurrentWork()
    return [plainResult, boundResult, concurrentResult]
}
let ui = await fromUI()
let outside = await fromOutside()
precondition(ui == [true, true, false] && outside == [false, false, false])
```

**Walkthrough:** The explicit caller annotations establish which context is supplied before comparing the three callees.

**Boundary:** A synchronous nonisolated helper also executes inline without gaining protected-state access. Actor observations are not thread or scheduling guarantees.

### When should explicit nonisolated(nonsending) or concurrent replace reliance on target defaults?

**Answer:** Use explicit caller-bound behavior to stabilize a reusable API across module settings, and concurrent when leaving the caller's actor is the promised contract.

**Example:** Standalone **compile-only** signatures with different execution intent.

```swift
nonisolated(nonsending) func inspect(_ values: [Int]) async -> Int {
    values.count
}
@concurrent func prepare(_ values: [Int]) async -> [Int] { values.sorted() }
```

**Walkthrough:** Inspect retains the caller's actor even in a module without the caller-default opt-in. Prepare leaves it even when that opt-in is present.

**Boundary:** Neither creates a new task or relaxes transfer checking. General libraries should document intent; redundant nonisolated modifiers on ordinary baseline declarations add no behavior.

### How can a MainActor type infer an isolated conformance, or provide a nonisolated conformance instead?

**Answer:** With inference enabled, witnesses requiring the actor can establish an isolated conformance. Witnesses that genuinely operate nonisolated over safe data can support a conformance usable outside that actor.

**Example:** Standalone runnable nonisolated witness.

```swift
@MainActor struct Label: CustomStringConvertible {
    nonisolated let name: String
    nonisolated var description: String { name }
}
func text<T: CustomStringConvertible>(_ value: T) -> String { value.description }
@concurrent func outside(_ label: Label) async -> String { text(label) }
let result = await outside(Label(name: "Lea"))
precondition(result == "Lea")
```

**Walkthrough:** The witness uses an independent immutable String, so it does not require live UI-state access. The generic helper can use this conformance off actor. The earlier Display example requires MainActor and rejects that same kind of use.

**Boundary:** Do not add a nonisolated annotation to a witness that still reads mutable UI state. Verify both the allowed actor client and the rejected off-actor client.

### Extension — How do changed caller defaults or default actor isolation affect a separately compiled module?

**Answer:** They can change the module's inferred declarations and ordinary async contracts when that module is compiled. Client settings alone do not rewrite existing imported contracts.

**Example:** Build both named libraries first. Runnable client; it intentionally omits NonisolatedNonsendingByDefault while calling the same imported APIs.

```swift
import DefaultsOff
import DefaultsOn
let first = await DefaultsOff.actorPresent()
let second = await DefaultsOn.actorPresent()
let explicit = await DefaultsOff.explicitCaller()
precondition(!first && second && explicit)
```

**Walkthrough:** The imported callee contracts remain distinguishable even when the client changes its own caller-default setting. The explicit caller-bound API still receives top-level MainActor.

| Default declaration isolation | Caller-isolated default | Eligible ordinary class | Explicit nonisolated plain async |
| --- | --- | --- | --- |
| nonisolated | enabled: baseline | nonisolated | caller's actor |
| nonisolated | omitted | nonisolated | off actor |
| MainActor | enabled | MainActor, subject to inference | caller's actor |
| MainActor | omitted | MainActor, subject to inference | off actor |

**Boundary:** For MainActor-default comparisons, explicitly mark the comparison helper nonisolated so default declaration inference does not change the question. Explicit caller-bound and concurrent helpers retain their own intent in every row. A UI-only target can deliberately choose a MainActor default; it is an override, not this collection's baseline.

## Focused exercise

**Task:** Compare three execution contracts from MainActor and a genuinely off-actor caller, without checking physical threads.

**Acceptance checks:**

- Under the baseline, MainActor sees [true, true, false].
- The off-actor entry sees [false, false, false].
- Alternative-setting comparisons keep declaration inference separate from async defaults.

**Solution:** Standalone runnable probe. Explicit nonisolated on the comparison helper is intentional because the validation matrix recompiles it under MainActor defaults too.

```swift
nonisolated func plain() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
nonisolated(nonsending) func bound() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
@concurrent func concurrent() async -> Bool {
    let actor: (any Actor)? = #isolation
    return actor != nil
}
@MainActor func checkUI() async {
    let first = await plain()
    let second = await bound()
    let third = await concurrent()
    #if hasFeature(NonisolatedNonsendingByDefault)
    precondition(first && second && !third)
    #else
    precondition(!first && second && !third)
    #endif
}
@concurrent func checkOutside() async {
    let first = await plain()
    let second = await bound()
    let third = await concurrent()
    precondition(!first && !second && !third)
}
await checkUI()
await checkOutside()
```

**Reasoning:** The concurrent entry provides a real off-actor context. The conditional expectation lets the same source check the ordinary async default with the opt-in present or absent. The explicit bound/concurrent cases retain their contract. Locals are awaited before synchronous assertion autoclosures.

## In practice

**Choose it when:** Make execution intent explicit at reusable boundaries and UI ownership explicit where required.

**Investigate by:** Inspect emitted settings and imported interfaces. Use isolation probes for permissions and profiling for actual responsiveness/performance.

**Strong evidence:** Compile libraries and ordinary clients under different settings; verify conformance permission independently of transferability and scheduling observations.

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
