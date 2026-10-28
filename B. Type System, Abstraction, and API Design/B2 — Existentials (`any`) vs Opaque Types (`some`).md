---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - existentials
  - opaque-types
  - generics
  - api-design
---

# B2 — Existentials (`any`) vs Opaque Types (`some`)

**Learning goals:**

- Distinguish caller-selected generics, parameter some, hidden opaque results, and runtime-selected existentials.
- Preserve associated-type relationships and explain both useful existential opening and its limits.
- Apply optional opaque syntax and SwiftUI erasure/builders without inferring allocation or performance from spelling.

**Prerequisites:** [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]] for protocols, associated types, and Self. [[A4 — Optionals and Nil Modeling]] covers Optional; [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] covers generic-looking function signatures. The SwiftUI extension requires the Apple SwiftUI framework.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A function can let its caller choose an implementation, hide one implementation it chooses itself, or choose among implementations at runtime. Those are different contracts. A named generic parameter T lets the caller choose a type satisfying constraints. A parameter spelled some P introduces an unnamed generic parameter. A result spelled some P gives the caller one hidden but stable type identity for that declaration and its generic arguments.

An existential, any P, stores a runtime-selected conformer under an erased static interface. Its dynamic value can change types. Choose among these forms by the type relationships callers need; the keywords alone do not establish allocation counts or specialization. [Opaque and boxed protocol types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Parameter some is generic; result some hides a chosen type

Each parameter some below introduces a separate type parameter. A named T can instead relate two inputs.

```swift
protocol Label { var text: String { get } }
struct Compact: Label { let text = "short" }
struct Detailed: Label { let text = "long" }
func render(_ value: some Label) -> String { value.text }
func pair<T: Label>(_ first: T, _ second: T) -> [String] { [first.text, second.text] }
func unrelated(_ first: some Label, _ second: some Label) -> [String] { [first.text, second.text] }
func fixed() -> some Label { Compact() }
precondition(render(Compact()) == "short")
precondition(render(Detailed()) == "long")
precondition(pair(Compact(), Compact()) == ["short", "short"])
precondition(unrelated(Compact(), Detailed()) == ["short", "long"])
precondition(fixed().text == "short")
```

**Walkthrough:** render accepts either type because the caller selects its generic argument. pair requires one shared T; unrelated introduces two independent parameters. fixed’s implementation chooses Compact and hides that choice. [SE-0341](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)

### Opaque identity belongs to a declaration and its generic context

The generic factory defines a family of hidden types, one for each T. Calls with the same T share an opaque identity.

```swift
struct Box<Value: Equatable>: Equatable { let value: Value }
func boxed<T: Equatable>(_ value: T) -> some Equatable { Box(value: value) }
let first = boxed(7)
let second = boxed(7)
precondition(first == second)
let text = boxed("Ada")
precondition(text == boxed("Ada"))
```

**Walkthrough:** The two Int calls can compare. The String calls can compare with each other too. The generic input can change the factory’s underlying type expression; it does not permit arbitrary type variation within one instantiated result contract.

## Rules and pitfalls

### One opaque result cannot choose unrelated hidden types per call

**Common pitfall:** Returning Local or Remote directly from one nongeneric some result.

**Mistake:** Compile-only; intentionally rejected.

```swift
protocol Source { var name: String { get } }
struct Local: Source { let name = "local" }
struct Remote: Source { let name = "remote" }
func make(_ local: Bool) -> some Source {
    if local { return Local() }
    return Remote()
}
```

**Consequence:** The compiler rejects the nonmatching underlying result types.

**Why:** The caller is promised one hidden type identity, but the branches propose two unrelated types.

**Correction:** Use an existential when runtime variation is the requirement. An enum wrapper is another option demonstrated in the exercise.

```swift
protocol Source { var name: String { get } }
struct Local: Source { let name = "local" }
struct Remote: Source { let name = "remote" }
func make(_ local: Bool) -> any Source {
    if local { return Local() }
    return Remote()
}
precondition(make(true).name == "local" && make(false).name == "remote")
```

**What changes:** The factory now advertises erased runtime choice rather than a stable hidden concrete identity.

### Do not discard required runtime variation to avoid imagined existential costs

**Common pitfall:** Forcing a heterogeneous source list into a single concrete generic specialization.

**Mistake:** Compile-only; intentionally rejected.

```swift
protocol Source { var name: String { get } }
struct Local: Source { let name = "local" }
struct Remote: Source { let name = "remote" }
struct Holder<S: Source> { let source: S }
let sources: [Holder<Local>] = [Holder(source: Local()), Holder(source: Remote())]
```

**Consequence:** The Remote element cannot satisfy Holder<Local>’s concrete type contract.

**Why:** A generic specialization retains one type argument. It does not become heterogeneous simply because the generic constraint is a protocol.

**Correction:** Use the runtime-selected interface when that is the storage requirement.

```swift
protocol Source { var name: String { get } }
struct Local: Source { let name = "local" }
struct Remote: Source { let name = "remote" }
let sources: [any Source] = [Local(), Remote()]
precondition(sources.map(\.name) == ["local", "remote"])
```

**What changes:** The array can hold both conformers and preserves Source operations. Existentials need not allocate, and generics need not specialize; [[C9 — Existential, generic, and allocation performance model|C9]] supplies the measurement framework for costs.

## Worked example

### Opening an existential for a single generic operation

**Setup:** Standalone script. Opening temporarily gives the hidden concrete type a generic name inside one call.

**Predict:** Can the caller read the source’s name without recovering its concrete type?

```swift
protocol Source { var name: String { get } }
struct Local: Source { let name = "local" }
func name<S: Source>(_ source: S) -> String { source.name }
func identity<S: Source>(_ source: S) -> S { source }
let selected: any Source = Local()
precondition(name(selected) == "local")
let erasedAgain: any Source = identity(selected)
precondition(erasedAgain.name == "local")
```

**Outcome:** The generic call returns local, and identity’s result can erase back to any Source.

**Why:** The hidden type is usable temporarily inside the generic operation. These results do not require exposing it as a type argument of a generic wrapper. [SE-0352](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)

## Interview questions

### Why can two calls to one opaque factory compare while two factories hiding Int cannot?

**Answer:** Each opaque declaration defines its own hidden type identity. Two calls to the same declaration share that identity for the same generic context; different declarations do not expose an equality between their hidden types.

**Example:** Standalone runnable example.

```swift
func firstFactory() -> some Equatable { 7 }
func secondFactory() -> some Equatable { 7 }
let first = firstFactory()
let again = firstFactory()
precondition(first == again)
let second = secondFactory()
precondition(second == secondFactory())
```

**Walkthrough:** Each pair can use its own Equatable witness because both operands have one opaque identity. The common current implementation Int is hidden from callers.

**Boundary:** This is a source contract, not a promise to keep using Int forever. An opaque result can be a generic type expression, so changing generic arguments can also change its instantiated identity.

Prepend that answer’s factory setup. Comparing the independent opaque results is **intentionally rejected**:

```swift
let invalid = firstFactory() == secondFactory()
```

### Why can a plain existential enter a generic call while Holder results or independent arguments fail?

**Answer:** Opening can bind one temporary underlying type for a call. That identity cannot generally escape inside an invariant generic wrapper, and two independent existential inputs do not prove one common hidden type.

**Example:** Standalone runnable example.

```swift
protocol Source { var name: String { get } }
struct Local: Source { let name = "local" }
func name<S: Source>(_ source: S) -> String { source.name }
func twoNames<S: Source>(_ first: S, _ second: S) -> [String] { [first.name, second.name] }
struct Holder<S: Source> { let source: S }
func wrap<S: Source>(_ source: S) -> Holder<S> { Holder(source: source) }
let selected: any Source = Local()
precondition(name(selected) == "local")
precondition(twoNames(Local(), Local()) == ["local", "local"])
```

**Walkthrough:** name returns String, which does not expose the opened type. wrap would return Holder<S>; that wrapper is invariant, so erasing the value does not turn it into Holder<any Source>. twoNames requires one shared S.

**Boundary:** An existential is not generally self-conforming just because an opened call works. Named generic arguments are checked after permitted conversions; even two Base values can contain different dynamic subclasses.

These separate **compile-only, intentionally rejected** functions prepend that Source/Holder setup:

```swift
func invalidWrap(_ source: any Source) { _ = wrap(source) }
```

```swift
func invalidPair(_ first: any Source, _ second: any Source) -> [String] {
    twoNames(first, second)
}
```

```swift
func invalidOpaque(_ source: any Source) -> some Source { source }
```

The dynamic-subclass boundary is a different issue. This standalone runnable example has one static T == Base even though the two objects have different dynamic classes:

```swift
class Base {}
final class First: Base {}
final class Second: Base {}
func sameStaticType<T>(_ first: T, _ second: T) -> Bool { T.self == Base.self }
let first: Base = First()
let second: Base = Second()
precondition(sameStaticType(first, second))
precondition(type(of: first) != type(of: second))
```

### What does any Sequence<Int> preserve, and what does neither some nor any promise?

**Answer:** It preserves Element == Int, enabling typed iteration. Neither spelling promises a particular order, eagerness, random access, allocation count, or specialization. Those properties need additional API contracts or measurement.

**Example:** Standalone runnable example.

```swift
func reversedNumbers() -> some Sequence<Int> { [1, 2, 3].reversed() }
func total(_ values: any Sequence<Int>) -> Int { values.reduce(0, +) }
let forward: any Sequence<Int> = [1, 2, 3]
let backward: any Sequence<Int> = reversedNumbers()
precondition(total(forward) == 6 && total(backward) == 6)
precondition(Array(forward) == [1, 2, 3])
precondition(Array(reversedNumbers()) == [3, 2, 1])
```

**Walkthrough:** Both expose Int elements and support the same summation. Their observed orders differ because the implementations differ, not because some means ordered or any means unordered.

**Boundary:** Some Sequence implementations are single-pass. If repeatable traversal or random access matters, ask for Collection or RandomAccessCollection and still state the domain’s ordering policy.

### How do some P? and any P? relate to older spellings, and why can nil or optional opening fail?

**Answer:** Swift 6.4 makes them alternatives to (some P)? and (any P)?. It does not infer a wrapped type from bare nil or make every optional existential openable at a generic parameter.

**Example:** Standalone runnable example.

```swift
protocol Source { var name: String { get } }
struct Local: Source { let name = "local" }
protocol Identified { var id: Int { get } }
extension Local: Identified { var id: Int { 7 } }
func optionalSource(_ present: Bool) -> some Source? { present ? Local() : nil }
func optionalName(_ source: some Source?) -> String { source?.name ?? "none" }
func erased(_ present: Bool) -> any Source? { present ? Local() : nil }
func combined() -> any (Source & Identified)? { Local() }
func plain<S: Source>(_ source: S) -> String { source.name }
func afterUnwrapping(_ source: any Source?) -> String {
    guard let source else { return "none" }
    return plain(source)
}
precondition(optionalName(Optional<Local>.none) == "none")
precondition(optionalSource(true)?.name == "local" && optionalSource(false) == nil)
precondition(afterUnwrapping(erased(true)) == "local")
precondition(afterUnwrapping(erased(false)) == "none")
precondition(combined()?.id == 7)
```

**Walkthrough:** The concrete Local branch establishes the opaque wrapped type, including on the absent path. Explicit Optional<Local>.none supplies a generic argument type. Unwrapping exposes a plain existential that can enter plain. [SE-0521](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0521-improved-optional-opaque-and-any.md)

**Boundary:** Compositions still need parentheses. Keep older parenthesized spelling for older compilers; syntax improvements do not relax underlying identity or lifetime constraints.

Each **compile-only, intentionally rejected** variant prepends that optional Source setup:

```swift
func missingArgumentType() -> String { optionalName(nil) }
```

```swift
func missingOpaqueType() -> some Source? { nil }
```

```swift
func invalidComposition() -> any Source & Identified? { nil }
```

```swift
func invalidOptional(_ source: any Source?) -> String { optionalName(source) }
```

### How do AnyView, any View, and a ViewBuilder result differ?

**Answer:** AnyView is a concrete SwiftUI eraser that conforms to View. any View is an existential and works only where opening or erasure can satisfy the operation. ViewBuilder combines branches into one structural type, which can be returned as one opaque result.

**Example:** Standalone runnable macOS script with SwiftUI. MainActor annotations describe UI-owned construction; there is no rendered application.

```swift
import SwiftUI

@MainActor func accept<V: View>(_ view: V) -> Int { 1 }
@MainActor @ViewBuilder func choice(_ compact: Bool) -> some View {
    if compact { Text("Compact") } else { Image(systemName: "doc") }
}
let erased = AnyView(Text("Erased"))
let existential: any View = Text("Existential")
precondition(accept(erased) == 1)
precondition(accept(choice(true)) == 1)
precondition(accept(existential) == 1)  // This single-input generic call can open it.
precondition(type(of: choice(true)) == type(of: choice(false)))
```

**Walkthrough:** AnyView supplies a concrete conforming wrapper. The builder’s branches produce one structural result type. accept’s Int result does not expose an opened existential type, so that call also works.

**Boundary:** An invariant wrapper such as ViewHolder<V> cannot generally retain the opened existential type. This example checks type behavior without rendering UI or benchmarking SwiftUI identity/update costs.

Prepend the SwiftUI answer’s setup. This **intentionally rejected** invariant wrapper cannot keep the existential’s opened identity:

```swift
struct ViewHolder<V: View> { let view: V }
let invalid = ViewHolder(view: existential)
```

## Focused exercise

**Task:** Expose two source implementations through three deliberate contracts: caller-selected generic, runtime existential, and one opaque wrapper.

**Acceptance checks:**

- The generic reader accepts Local and Remote.
- The existential factory selects either implementation at runtime.
- A conforming enum permits runtime branch choice while retaining one opaque result type.

**Solution:** Standalone script.

```swift
protocol Source { var name: String { get } }
struct Local: Source { let name = "local" }
struct Remote: Source { let name = "remote" }
func read<S: Source>(_ source: S) -> String { source.name }
func runtime(_ local: Bool) -> any Source {
    if local { return Local() }
    return Remote()
}
enum Selected: Source {
    case local(Local)
    case remote(Remote)
    var name: String {
        switch self {
        case .local(let source): source.name
        case .remote(let source): source.name
        }
    }
}
func opaque(_ local: Bool) -> some Source {
    local ? Selected.local(Local()) : Selected.remote(Remote())
}
precondition(read(Local()) == "local" && read(Remote()) == "remote")
precondition(read(runtime(true)) == "local" && read(runtime(false)) == "remote")
precondition(read(opaque(true)) == "local" && read(opaque(false)) == "remote")
```

**Reasoning:** The caller chooses S for direct generic calls. runtime chooses a hidden dynamic conformer. opaque always returns Selected, whose cases describe the runtime alternatives. These contracts differ in operations and evolution, not a source-level promise of speed.

## In practice

**Choose it when:** Choose the boundary from who selects the concrete type and what relationships callers need.

**Investigate by:** Reduce rejection to one input/result position; identify what must escape after opening.

**Strong evidence:** Validate the intended caller operations and measure representative optimized clients before introducing manual erasure or claiming specialization.

## References

### Related guides

- [[A4 — Optionals and Nil Modeling]]
- [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]

### Official sources

- [Opaque and boxed protocol types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/)
- [SE-0352](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)
- [SE-0521](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0521-improved-optional-opaque-and-any.md)
- [SE-0244: Opaque result types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)
- [SE-0335: Existential any](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md)
- [SE-0341: Opaque parameters](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)
- [SE-0353: Constrained existential types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0353-constrained-existential-types.md)
- [Existential performance diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/existential-type/)

- [SwiftUI AnyView](https://developer.apple.com/documentation/swiftui/anyview)
- [SwiftUI ViewBuilder](https://developer.apple.com/documentation/swiftui/viewbuilder)
