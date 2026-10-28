---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - dispatch
  - protocols
  - api-design
---

# B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch

**Learning goals:**

- Predict declaration and witness selection through concrete, generic, existential, and superclass callers.
- Distinguish extension-only helpers, inherited defaults, constrained defaults, and overridable class witnesses.
- Keep optimization evidence, conformance isolation, Objective-C exposure, and native dynamic behavior separate.

**Prerequisites:** [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]] for protocol witnesses, [[B2 — Existentials (`any`) vs Opaque Types (`some`)]] for generic/existential consumers, and [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for class dispatch. The runtime examples require Foundation on Apple platforms.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A concrete renderer can appear to override a default while a generic renderer still prints the default. To explain that result, ask two questions in order: which declaration does the caller select, and how does that declaration find its implementation?

A protocol **requirement** has a conformance witness. An extension-only helper has no requirement slot to replace. A class witness can call through an overridable class method, allowing a subclass override to participate. Optimization can change the machine-level route, but must preserve these selected behaviors. [Protocol extensions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### A requirement and an extension-only helper have different customization paths

The protocol declares render as a requirement. label exists only in the extension.

```swift
protocol Renderer { func render() -> String }
extension Renderer {
    func render() -> String { "default render" }
    func label() -> String { "extension label" }
}
struct Screen: Renderer {
    func render() -> String { "screen render" }
    func label() -> String { "screen label" }
}
func generic<R: Renderer>(_ renderer: R) -> [String] { [renderer.render(), renderer.label()] }
let screen = Screen()
let erased: any Renderer = screen
precondition([screen.render(), screen.label()] == ["screen render", "screen label"])
precondition(generic(screen) == ["screen render", "extension label"])
precondition([erased.render(), erased.label()] == ["screen render", "extension label"])
```

**Walkthrough:** Concrete lookup sees both Screen members. Protocol consumers use Screen’s render witness but select the extension-only label helper. A same-named concrete helper does not add a protocol requirement.

### A class witness can lead to an override

The base explicitly supplies the witness through a non-final class method.

```swift
protocol Renderer { func render() -> String }
class Base: Renderer { func render() -> String { "base" } }
final class Child: Base { override func render() -> String { "child" } }
func generic<R: Renderer>(_ value: R) -> String { value.render() }
let child = Child()
let base: Base = child
let erased: any Renderer = child
precondition(base.render() == "child")
precondition(generic(child) == "child" && erased.render() == "child")
```

**Walkthrough:** The conformance’s witness uses the class method path. That path reaches Child’s override even though the conformance was declared on Base. The next pitfall contrasts a base that never supplied such a method.

## Rules and pitfalls

### A child member cannot replace an inherited default witness

**Common pitfall:** Adding a same-named method to a subclass after its base accepted a protocol-extension default.

**Mistake:** Standalone runnable example.

```swift
protocol Renderer { func render() -> String }
extension Renderer { func render() -> String { "default" } }
class Base: Renderer {}
final class Child: Base { func render() -> String { "child" } }
func generic<R: Renderer>(_ value: R) -> String { value.render() }
let child = Child()
let erased: any Renderer = child
precondition(child.render() == "child")
precondition(generic(child) == "default" && erased.render() == "default")
```

**Consequence:** The concrete call returns child, but both protocol-consumer calls return default.

**Why:** Base’s conformance already chose the extension witness. Child’s new member is not an override of a Base method and does not rebuild that conformance.

**Correction:** Put the intended customization point on the base as an overridable witness.

```swift
protocol Renderer { func render() -> String }
extension Renderer { func render() -> String { "default" } }
class Base: Renderer { func render() -> String { "base" } }
final class Child: Base { override func render() -> String { "child" } }
func generic<R: Renderer>(_ value: R) -> String { value.render() }
let child = Child()
let erased: any Renderer = child
precondition(generic(child) == "child" && erased.render() == "child")
```

**What changes:** The witness now enters the class override path. Subclass customization is a deliberate API design, not a later coincidence of names.

### Optimization does not redo generic lookup

**Common pitfall:** Expecting specialization to find a concrete helper that was not a protocol requirement.

**Mistake:** Standalone runnable example.

```swift
protocol Named {}
extension Named { func name() -> String { "extension" } }
struct User: Named { func name() -> String { "user" } }
func generic<T: Named>(_ value: T) -> String { value.name() }
precondition(User().name() == "user")
precondition(generic(User()) == "extension")
```

**Consequence:** The generic operation keeps extension behavior, including in the separately checked optimized build.

**Why:** Its body was checked against Named and selected the extension helper. Replacing an indirect call with a direct call must preserve that selection.

**Correction:** Make required polymorphism a protocol requirement.

```swift
protocol Named { func name() -> String }
extension Named { func name() -> String { "extension" } }
struct User: Named { func name() -> String { "user" } }
func generic<T: Named>(_ value: T) -> String { value.name() }
precondition(generic(User()) == "user")
```

**What changes:** The generic operation now uses the conformer’s selected witness. Public protocol changes still require an evolution policy for existing conformers and clients.

## Worked example

### Constrained lookup and an unconditional conformance can disagree

**Setup:** Standalone script. The constrained extension is an overload available when Self is Equatable.

**Predict:** Does specializing Box<Int> reselect its protocol witness?

```swift
protocol Categorized { func category() -> String }
extension Categorized { func category() -> String { "general" } }
extension Categorized where Self: Equatable {
    func category() -> String { "equatable" }
}
struct Box<Value>: Categorized { let value: Value }
extension Box: Equatable where Value: Equatable {}
func category<C: Categorized>(_ value: C) -> String { value.category() }
let box = Box(value: 7)
let erased: any Categorized = box
precondition(box.category() == "equatable")
precondition(category(box) == "general" && erased.category() == "general")
```

**Outcome:** Direct lookup returns equatable; the generic and existential callers return general.

**Why:** The unconditional Box<Value>: Categorized conformance was formed for all Value types. It selected a witness valid without Equatable. Direct concrete lookup has additional information; specialization does not rebuild the conformance.

## Interview questions

### Why can a child’s concrete render differ from generic/existential render, and how does an overridable base member fix it?

**Answer:** A child concrete call can find a new method while protocol consumers keep the base conformance’s default witness. An overridable base witness instead provides a class dispatch path to the subclass.

**Example:** Standalone runnable example.

```swift
protocol Renderer { func render() -> String }
extension Renderer { func render() -> String { "default" } }
class DefaultBase: Renderer {}
final class DefaultChild: DefaultBase { func render() -> String { "child" } }
class VirtualBase: Renderer { func render() -> String { "base" } }
final class VirtualChild: VirtualBase { override func render() -> String { "override" } }
func read<R: Renderer>(_ value: R) -> String { value.render() }
let first = DefaultChild()
let second = VirtualChild()
precondition(first.render() == "child" && read(first) == "default")
precondition(second.render() == "override" && read(second) == "override")
let erasedFirst: any Renderer = first
let erasedSecond: any Renderer = second
precondition(erasedFirst.render() == "default" && erasedSecond.render() == "override")
```

**Walkthrough:** DefaultBase selected an extension witness with no overridable member. VirtualBase supplied the class method that VirtualChild overrides.

**Boundary:** A subclass does not get a new copy of an inherited conformance. Design the base witness intentionally when subclass polymorphism is required.

### Why can Box<Int> choose a constrained default while its unconditional conformance uses the general witness?

**Answer:** Direct lookup uses the concrete constraints visible at that call. An unconditional conformance must select a witness valid for every supported generic argument, without later re-running selection when specialized.

**Example:** Standalone runnable example.

```swift
protocol Categorized { func category() -> String }
extension Categorized { func category() -> String { "general" } }
extension Categorized where Self: Equatable { func category() -> String { "equatable" } }
struct Box<Value>: Categorized { let value: Value }
extension Box: Equatable where Value: Equatable {}
func category<C: Categorized>(_ value: C) -> String { value.category() }
let box = Box(value: 7)
precondition(box.category() == "equatable")
precondition(category(box) == "general")
```

**Walkthrough:** The direct member can choose the Equatable overload; the unconditional witness cannot rely on that constraint.

**Boundary:** A conditional conformance is a different contract. It can use the constrained witness but is unavailable for non-Equatable payloads; an explicit witness can also state a policy valid for every payload.

Prepend the Categorized answer’s setup. This runnable alternative explicitly conditions conformance:

```swift
struct EquatableBox<Value> { let value: Value }
extension EquatableBox: Equatable where Value: Equatable {}
extension EquatableBox: Categorized where Value: Equatable {}
precondition(category(EquatableBox(value: 7)) == "equatable")
```

### Which choices are language semantics, and what do SIL or optimized runs establish?

**Answer:** Declaration lookup, conformance-witness selection, and class overrides determine behavior. SIL instructions show how a particular build implements that behavior. An optimized direct call can be devirtualization; it does not create a new lookup rule.

**Example:** Standalone runnable example.

```swift
protocol Renderer { func render() -> String }
extension Renderer { func helper() -> String { "helper" } }
struct Value: Renderer { func render() -> String { "value" } }
class Base { func render() -> String { "base" } }
final class Child: Base { override func render() -> String { "child" } }
func generic<R: Renderer>(_ value: R) -> String { value.render() + value.helper() }
func virtual(_ value: Base) -> String { value.render() }
precondition(generic(Value()) == "valuehelper")
precondition(virtual(Child()) == "child")
```

**Walkthrough:** The generic requirement and class call retain their chosen behavior. The validation also emits unoptimized SIL for this source and checks witness_method, class_method, and direct function references, then runs an -O build.

**Boundary:** Those observations are compiler/build evidence, not promises about permanent instructions, inlining, allocations, or timing. Use `swiftc` with the baseline flags and `-emit-silgen Example.swift` to inspect SIL; use `-O` for the optimized executable comparison.

### Why must callers satisfy conformance isolation as well as selecting a witness?

**Answer:** Dispatch selects an implementation; isolation controls permission to use it. An isolated conformance cannot be made nonisolated merely by passing its Sendable value into a generic function.

**Example:** Standalone runnable example.

```swift
protocol Titled { func title() -> String }
struct Label: Sendable, @MainActor Titled {
    let text: String
    @MainActor func title() -> String { text }
}
@MainActor func show<T: Titled>(_ value: T) -> String { value.title() }
precondition(show(Label(text: "Ada")) == "Ada")
```

**Walkthrough:** show invokes the witness inside its MainActor domain. The shared baseline has nonisolated declaration defaults; Approachable Concurrency alone does not make every ordinary protocol extension MainActor-isolated.

**Boundary:** Imported conformance contracts and explicit actor annotations remain real. [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions|B1]]/[[D5 — Global Actors and `@MainActor`|D5]]/[[D14 — Approachable Concurrency and Default Isolation|D14]] cover inferred isolated conformances and generic-boundary choices.

Prepend that Label/Titled setup. This **compile-only, intentionally rejected** caller drops the witness domain:

```swift
func generic<T: Titled>(_ value: T) -> String { value.title() }
func invalid(_ label: Label) -> String { generic(label) }
```

### When does a selector need @objc, when does KVO need dynamic behavior, and why can a Swift struct have a dynamic method?

**Answer:** @objc exposes supported declarations to Objective-C selectors. KVO property interception needs the appropriate dynamic Objective-C surface. Native Swift dynamic replacement is a separate mechanism, so dynamic does not universally imply Objective-C representation.

**Example:** Standalone runnable example.

```swift
import Foundation

final class ActionTarget: NSObject {
    @objc func title() -> NSString { "selector" }
}
func invokeSelector() -> String? {
    let target = ActionTarget()
    return target.perform(#selector(ActionTarget.title))?.takeUnretainedValue() as? String
}
precondition(invokeSelector() == "selector")
```

**Walkthrough:** The selector is valid because title is exposed to Objective-C. This selector invocation uses that runtime; the declaration does not need dynamic merely to have a selector.

**Boundary:** This is Apple Objective-C interoperability. It does not prove that every ordinary Swift call to an @objc member is Objective-C messaging. The independent KVO and native dynamic examples below demonstrate the other two contracts.

Second independent runnable example: KVO interception and explicit invalidation. Mutex protects the recorder because the callback is concurrency-annotated; the Player itself is mutated synchronously in one context.

```swift
import Foundation
import Synchronization

final class Player: NSObject {
    @objc dynamic var status: String = "idle"
}

@available(macOS 15.0, iOS 18.0, *)
final class StatusRecorder: Sendable {
    private let storage = Mutex<[String]>([])
    func append(_ value: String) { storage.withLock { $0.append(value) } }
    var values: [String] { storage.withLock { $0 } }
}

@available(macOS 15.0, iOS 18.0, *)
func observedStatuses() -> [String] {
    let player = Player()
    let recorder = StatusRecorder()
    let observation = player.observe(\.status, options: [.initial, .new]) {
        _, change in
        if let value = change.newValue { recorder.append(value) }
    }
    player.status = "playing"
    observation.invalidate()
    player.status = "stopped"
    return recorder.values
}

precondition(observedStatuses() == ["idle", "playing"])
```

The callback observes idle and playing, then invalidation prevents the stopped notification. The macOS 15/iOS 18 annotation comes from Mutex, not from KVO being new at those versions. [Using key-value observing in Swift](https://developer.apple.com/documentation/swift/using-key-value-observing-in-swift)

Third independent runnable example: a native value type can declare a dynamic method without NSObject or @objc. No replacement is installed here.

```swift
struct NativeValue { dynamic func value() -> Int { 7 } }
precondition(NativeValue().value() == 7)
```

Native dynamic replacement predates Swift 6.4. The compiler-author explanation describes its Swift 5.0 machinery and warns that the underscored replacement attribute is unsupported; this chapter does not use that attribute. [Compiler-author explanation](https://forums.swift.org/t/how-does-the-hot-reloading-work-in-xcode11/25312/5)

## Focused exercise

**Task:** Predict f and g through concrete, generic, and existential consumers, then make g a requirement.

**Acceptance checks:**

- Concrete g uses S.g, while initial protocol consumers use the extension-only g.
- Required f uses S.f through all surfaces.
- After g becomes a requirement, both protocol-consumer paths use S.g.

**Solution:** Standalone before-change script. The after-change script immediately below is a separate compilation.

```swift
protocol P { func f() -> String }
extension P {
    func f() -> String { "default f" }
    func g() -> String { "extension g" }
}
struct S: P {
    func f() -> String { "S.f" }
    func g() -> String { "S.g" }
}
func generic<T: P>(_ value: T) -> [String] { [value.f(), value.g()] }
let concrete = S()
let erased: any P = concrete
precondition([concrete.f(), concrete.g()] == ["S.f", "S.g"])
precondition(generic(concrete) == ["S.f", "extension g"])
precondition([erased.f(), erased.g()] == ["S.f", "extension g"])
```

**Reasoning:** The initial conformance needs a witness only for f. The extension-only g is statically selected by protocol consumers. Adding a g requirement changes the customization contract; it is not an optimizer hint.

Separate runnable after-change solution:

```swift
protocol P {
    func f() -> String
    func g() -> String
}
extension P {
    func f() -> String { "default f" }
    func g() -> String { "extension g" }
}
struct S: P {
    func f() -> String { "S.f" }
    func g() -> String { "S.g" }
}
func generic<T: P>(_ value: T) -> [String] { [value.f(), value.g()] }
let concrete = S()
let erased: any P = concrete
precondition(generic(concrete) == ["S.f", "S.g"])
precondition([erased.f(), erased.g()] == ["S.f", "S.g"])
```

## In practice

**Choose it when:** Make required polymorphism explicit and keep derived extension helpers deliberately non-customizable through protocol consumers.

**Investigate by:** Inspect the caller’s static types, selected declarations, and inherited witnesses before looking at SIL or timing.

**Strong evidence:** Test the intended overridable witness path, constrained/default alternatives, isolation permission, and ordinary/optimized callers separately.

## References

### Related guides

- [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]
- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[E1 — Objective-C Interoperability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [Protocol extensions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [Compiler-author explanation](https://forums.swift.org/t/how-does-the-hot-reloading-work-in-xcode11/25312/5)
- [Swift ABI discussion](https://github.com/swiftlang/swift/blob/main/docs/ABIStabilityManifesto.md)
- [Library evolution in Swift](https://www.swift.org/blog/library-evolution/)
- [Using key-value observing in Swift](https://developer.apple.com/documentation/swift/using-key-value-observing-in-swift)
