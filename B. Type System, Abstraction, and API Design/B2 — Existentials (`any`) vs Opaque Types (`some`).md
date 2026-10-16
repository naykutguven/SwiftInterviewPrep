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

**Goal:** Choose a generic or existential boundary that permits the intended operations on two implementations.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Generics let the caller determine a type under constraints. An opaque result lets the implementation choose one hidden type for the declaration's generic context, while retaining its identity. An existential stores a runtime-selected conformer under an erased static type. Parameter some introduces an unnamed generic parameter; it is not an opaque result. [Opaque and boxed protocol types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/)

## Rules and pitfalls

### Retain only the abstraction you need

Use concrete types for direct APIs, generics for caller-selected relationships, opaque results to hide an implementation while preserving identity, and existentials when runtime variation is the requirement. Associated-type constraints can retain useful relationships after erasure.

**Common pitfall:** Returning different conformers from one some result violates the fixed underlying-type rule. Return an existential, one conforming wrapper, or builder-generated structural type when that is the intended contract.

### Separate semantic choice from representation cost

Existentials need not allocate and generics need not specialize. Opening an existential supports many generic calls without granting any P general conformance to P.

**Common pitfall:** Selecting generics only because “any is slow” replaces measurement with folklore. Choose the required operations first, then measure representative optimized clients; see [[C9 — Existential, generic, and allocation performance model]].

## Interview questions

### Why can two calls to one opaque factory compare while two factories returning Int behind some Equatable cannot?

Calls to the same opaque factory share its declaration-specific opaque identity, so its Equatable operations can relate the results. Two separate factories introduce distinct opaque identities even when both currently hide Int. The caller cannot assume those hidden types are equal.

### Why can a plain existential enter a generic call while an invariant Holder result or two independent existential arguments fail?

Opening supplies a temporary type for one existential's underlying value. That type cannot generally escape in an invariant Holder result whose static type would expose the hidden identity. Two independent existentials may hold different concrete types, so one shared generic T is not automatically available for them.

### What does any Sequence<Int> preserve, and what do neither any Sequence<Int> nor some Sequence<Int> promise about ordering, eagerness, or performance?

It retains the Element == Int relationship, enabling typed iteration. Neither spelling promises a particular element order, eagerness of production, random access, allocation count, or specialization. Sequence's basic iteration contract is narrower than these implementation properties.

### How do some P? and any P? relate to the older parenthesized spellings? Why can bare nil or an optional existential still fail at a generic parameter?

The new spellings denote the same optional opaque or existential types as their parenthesized forms. Bare nil can leave the wrapped type uninferred; an optional existential can block opening where the generic relationship would escape or an invariant use requires a single type. Syntax does not erase those constraints.

### How do AnyView, any View, and a ViewBuilder result differ?

AnyView is a concrete SwiftUI eraser that conforms to View. any View is an existential and does not automatically satisfy every View-constrained context. ViewBuilder transforms branches into one structural result type, permitting different branch expressions while preserving a single opaque result identity. SwiftUI is an Apple framework dependency.

## Focused exercise

**Task:** Expose two implementations through a generic or existential API. Explain who selects the concrete type and check the intended caller operations.

**Acceptance checks:**

- The generic consumer accepts both concrete implementations.
- The existential factory varies its stored type at runtime.
- Opening permits this single-input generic call without claiming general existential self-conformance.

**Solution:**

Standalone runtime-choice and generic-consumer example:

```swift
protocol Label { var text: String { get } }
struct Compact: Label { let text = "short" }
struct Detailed: Label { let text = "long" }
func render<L: Label>(_ label: L) -> String { label.text }
func make(detailed: Bool) -> any Label {
    if detailed { return Detailed() }
    return Compact()
}
precondition(render(Compact()) == "short")
precondition(render(Detailed()) == "long")
let runtime = make(detailed: true)
precondition(render(runtime) == "long")
```

**Reasoning:** The caller selects L for direct generic calls; the factory selects the dynamic conformer for the existential result. The consumer returns String, so the opened hidden type need not escape in an invariant generic result.

## In practice

**Choose it when:** Match the boundary to who chooses the concrete implementation and which relationships callers need.

**Investigate by:** Reduce rejected uses to one input/result type position.

**Strong evidence:** Validate caller operations and measure costs before introducing manual erasure.

### Opening and optional-type limits — optional

#### Existential opening is useful, but bounded

These independent specialist snippets share the following declarations. Compile rejected variants separately:

```swift
protocol ImageSource { var name: String { get } }
struct Local: ImageSource { var name: String { "local" } }
struct Remote: ImageSource { var name: String { "remote" } }
func name<S: ImageSource>(_ source: S) -> String { source.name }
func twoNames<S: ImageSource>(_ first: S, _ second: S) -> [String] {
    [first.name, second.name]
}
```

```swift
func identity<S: ImageSource>(_ source: S) -> S { source }

func openedName(_ source: any ImageSource) -> String {
    name(source)
}

func erasedAgain(_ source: any ImageSource) -> any ImageSource {
    identity(source)
}
```

The compiler can open the argument and bind a generic parameter to its underlying type for this call. A suitable result can be erased again. These calls do not establish a general self-conformance for `any ImageSource`. [SE-0352](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)

Expected failure: a generic wrapper cannot escape with this opened identity.

```swift
struct Holder<S: ImageSource> {
    let source: S
}

func wrap<S: ImageSource>(_ source: S) -> Holder<S> {
    Holder(source: source)
}

func cannotKeepOpenedType(_ source: any ImageSource) {
    _ = wrap(source)
}
```

`Holder<S>` is invariant in `S`; this result cannot simply become `Holder<any ImageSource>`. The compiler rejects the call with a protocol-conformance diagnostic.

Expected failure, reusing `twoNames`:

```swift
func cannotProveSameType(
    _ first: any ImageSource,
    _ second: any ImageSource
) -> [String] {
    twoNames(first, second)
}
```

Two existential arguments do not establish one shared underlying type, even if a particular caller stored two `Remote` values. Keep the relationship generic before erasure when the API needs it.

Expected failure:

```swift
func cannotReturnExistentialAsOpaque(
    _ source: any ImageSource
) -> some ImageSource {
    source
}
```

A nongeneric opaque factory cannot select a different hidden underlying type on each call from the existential argument. Use an existential result, a concrete conforming eraser, or a generic input with a valid underlying type expression.

#### Swift 6.4 simplifies optional spelling

```swift
protocol Identified {
    var id: Int { get }
}
extension Remote: Identified {
    var id: Int { 7 }
}

func optionalSource(_ present: Bool) -> some ImageSource? {
    present ? Remote() : nil
}

func optionalName(_ source: some ImageSource?) -> String {
    source?.name ?? "none"
}

func optionalPair() -> some (ImageSource & Identified)? {
    Remote()
}

func optionalErasure(_ present: Bool) -> any ImageSource? {
    present ? Local() : nil
}
```

`some ImageSource?` means `(some ImageSource)?`; `any ImageSource?` means `(any ImageSource)?`. Old parenthesized spelling remains valid for 6.2/6.3-compatible source. Compositions need grouping. This syntax change does not relax hidden-type inference or existential opening. A4 covers nested optionals, IUOs, and suppression syntax. [SE-0521](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0521-improved-optional-opaque-and-any.md)

Expected failures, checked separately with the relevant declarations:

```swift
func uninferredArgument() -> String {
    optionalName(nil)
}
```

Use `optionalName(Optional<Remote>.none)` when that is the intended argument type.

```swift
func uninferredResult() -> some ImageSource? {
    nil
}
```

Bare `nil` supplies no underlying conforming type. `optionalSource` above supplies one through its `Remote` branch.

```swift
func ungroupedComposition() -> any ImageSource & Identified? {
    Remote()
}
```

The diagnostic asks for parentheses; write `any (ImageSource & Identified)?`.

```swift
func cannotOpenOptional(_ source: any ImageSource?) -> String {
    optionalName(source)
}
```

Opening a plain existential argument does not make this optional argument work. Unwrap first, then call the plain generic parameter function:

```swift
func nameAfterUnwrapping(_ source: any ImageSource?) -> String {
    guard let source else { return "none" }
    return name(source)
}
```

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
