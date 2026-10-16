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

**Goal:** Predict requirement and extension-only calls through concrete, generic, and existential surfaces.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Determine the declaration selected by static lookup, then how that declaration obtains its implementation. Protocol requirements use conformance witnesses; extension-only members have no requirement slot. Class overrides can participate when the witness points through an overridable class member. Optimization must preserve these semantic choices. [Protocol extensions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)

## Rules and pitfalls

### Make customization points requirements

A same-named concrete helper does not replace an extension-only operation selected by a generic body. If conformers must customize behavior through protocol consumers, declare the requirement and supply a default witness where useful.

**Common pitfall:** Adding a child-class method after the base inherited a default witness does not replace that conformance's witness. Put an overridable member on the base when subclass customization is the intended path.

### Keep evidence scoped to its level

SIL can show witness_method, class_method, or direct calls for one build. An optimized direct call can reflect devirtualization without changing the language's selected behavior. Isolated conformances add an actor restriction to witness use.

**Common pitfall:** Inferring a new dispatch guarantee from one optimized run confuses implementation evidence with semantics. Test the concrete/generic/existential call surfaces separately and keep their static types explicit.

## Interview questions

### Why can a child class's concrete render differ from its generic or existential render? How does an overridable base member change the witness path?

The child concrete call can find its new member, while generic/existential calls use the base conformance's previously selected default witness. If the base instead supplies an overridable class method as its witness, that witness can dispatch through the override. The conformance does not rebuild itself for each subclass.

### Why can direct Box<Int> lookup select a constrained default while its unconditional conformance still uses the general witness?

Direct lookup sees the concrete Int constraint and can select a constrained overload. The unconditional conformance was checked for all generic arguments and chose a witness valid under those broader requirements. Later specialization does not redo conformance or overload selection.

### Which choices are language semantics, and what do SIL instructions or an optimized run actually establish?

Requirement versus extension-only selection and class override behavior are semantic rules. SIL instructions and optimized output show how a particular compiler/build implements them. They do not guarantee a permanent instruction sequence or that future optimization reruns static overload resolution.

### Why must a caller satisfy a conformance's isolation boundary as well as select the correct witness?

A witness can require a particular actor. Correct dispatch still cannot authorize use from the wrong isolation context; generic callers must preserve that conformance domain. This collection's ordinary protocol extensions do not gain MainActor isolation merely from Approachable Concurrency.

### When does a selector need @objc, when does KVO property access need dynamic behavior, and why can a Swift struct have a dynamic method?

@objc exposes supported declarations to Objective-C selectors. Dynamic behavior is needed for interception such as KVO property dispatch. Native Swift dynamic replacement is a distinct mechanism and can apply to a struct method; dynamic does not universally imply Objective-C representation. Check the framework's actual requirements.

## Focused exercise

**Task:** Predict f and g through concrete, generic, and existential consumers. Make g a requirement and check the changed witness path.

**Acceptance checks:**

- The concrete g uses S; protocol consumers initially use the extension-only helper.
- The required f uses S through every surface.
- Making g a requirement supplies S.g as its witness.

**Solution:**

Standalone synchronous probe:

```swift
protocol P { func f() -> String }
extension P {
    func f() -> String { "default f" }
    func g() -> String { "default g" }
}
struct S: P {
    func f() -> String { "custom f" }
    func g() -> String { "custom g" }
}
func generic<T: P>(_ value: T) -> [String] { [value.f(), value.g()] }
let concrete = S()
let erased: any P = concrete
precondition([concrete.f(), concrete.g()] == ["custom f", "custom g"])
precondition(generic(concrete) == ["custom f", "default g"])
precondition([erased.f(), erased.g()] == ["custom f", "default g"])
```

Then add func g() -> String to P and change both protocol-consumer expected g results to custom g.

**Reasoning:** The original protocol has a witness slot only for f. Adding g changes its customization contract, not just an optimizer hint. Public protocol evolution must consider existing conformers and binary policy separately.

## In practice

**Choose it when:** Make required polymorphism explicit and keep derived helpers intentionally static.

**Investigate by:** Inspect static types and witness selection before SIL or timing.

**Strong evidence:** Design and test an overridable base witness path across inherited conformances.

### Constrained and runtime dispatch — optional

#### Constrained defaults and conditional conformances

Standalone staff-depth example:

```swift
protocol Categorized {
    func category() -> String
}
extension Categorized {
    func category() -> String { "general" }
}
extension Categorized where Self: Equatable {
    func category() -> String { "equatable" }
}

struct Box<Value>: Categorized { let value: Value }
extension Box: Equatable where Value: Equatable {}

func category<C: Categorized>(_ value: C) -> String { value.category() }
```

For `Box(value: 7)`, direct member lookup selects `equatable`, but `category(box)` and `any Categorized` select the conformance witness `general`. The unconditional `Box<Value>: Categorized` conformance was formed without requiring `Value: Equatable`.

Reusing `Categorized` and its extensions, a conditional conformance is a different contract:

```swift
struct EquatableBox<Value> { let value: Value }
extension EquatableBox: Equatable where Value: Equatable {}
extension EquatableBox: Categorized where Value: Equatable {}
```

Now the constrained default can supply the witness; the generic call returns `equatable`. The cost is that boxes of non-equatable values no longer have this conformance. Alternatively, provide an explicit requirement witness whose implementation expresses the intended rule for every supported value type. Do not expect the witness to be reselected at each call.

#### Objective-C exposure and runtime dispatch

Standalone selector example:

```swift
import Foundation

final class ActionTarget: NSObject {
    @objc func title() -> NSString { "selector" }
}

func invokeSelector() -> String? {
    let target = ActionTarget()
    let selector = #selector(ActionTarget.title)
    return target.perform(selector)?.takeUnretainedValue() as? String
}
```

`@objc` exposes the method to Objective-C; this selector invocation uses that runtime. Target/action-style selector use does not require adding `dynamic` to every exposed method. Conversely, `@objc` alone is not a blanket promise that ordinary Swift calls always use Objective-C messaging.

For Swift property access that must participate in KVO interception, use the appropriate observable Objective-C surface. Standalone example:

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
```

The checked host returns `idle`, then `playing`; invalidation stops the later notification. The recorder handles the callback's concurrency annotation without capturing and mutating an unprotected local array. The example mutates `Player` synchronously in one context; the recorder does not make that model safe for arbitrary concurrent mutation. Its minimum deployment annotation comes from `Mutex`, not from introducing KVO in those OS releases.

#### Native dynamic is not necessarily Objective-C

Standalone counterexample to the old blanket rule:

```swift
struct NativeValue {
    dynamic func value() -> Int { 7 }
}
```

This compiles in Swift 6.4 without `@objc` or `NSObject` and returns 7. Swift has native dynamic replacement machinery; do not describe every `dynamic` declaration as Objective-C messaging. The compiler team's explanation traces that machinery to Swift 5.0 and identifies the underscored replacement attribute as unsupported surface. This chapter uses the counterexample to correct the dispatch model, not to recommend an underscored replacement API. [Compiler-author explanation](https://forums.swift.org/t/how-does-the-hot-reloading-work-in-xcode11/25312/5)

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
