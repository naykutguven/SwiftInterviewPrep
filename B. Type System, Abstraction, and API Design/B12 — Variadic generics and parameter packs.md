---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - generics
  - parameter-packs
---

# B12 — Variadic Generics and Parameter Packs

**Goal:** Implement typed pack equality across zero, one, and heterogeneous positions.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A type pack is an ordered list of type parameters whose arity is known through the generic relationship. each references a pack element and repeat expands a pattern. This preserves each position's type without a mechanical family of arity overloads. It is not a runtime array, a work scheduler, or a guarantee of machine-code unrolling. [SE-0393](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0393-parameter-packs.md)

## Rules and pitfalls

### Preserve corresponding positions

An expansion using multiple packs requires compatible shape. Corresponding values can then share the required per-position type relationship. A homogeneous runtime collection instead expresses one element type and a runtime count.

**Common pitfall:** Treating pack correspondence like zip overlooks that zip can truncate at the shorter runtime sequence. Packs establish typed arity rather than a minimum runtime length.

### Keep evaluation and availability explicit

Pack iteration permits ordinary early return, but call arguments are evaluated before entering the function. Check variadic nominal-type runtime availability separately from functions and aliases. Ordinary unconstrained pack elements still have ownership defaults such as Copyable.

**Common pitfall:** Using packs to launch independent async work adds complexity without creating concurrency. Use structured concurrency for scheduling; choose packs only when preserving positional type relationships helps callers.

## Interview questions

### Why does one tuple expansion collapse to a scalar, while passing a concrete tuple as one argument does not unpack it?

A one-element tuple type collapses to its scalar type in Swift, so a one-position expansion can produce a scalar. Passing a concrete tuple as one ordinary argument keeps it one argument; the compiler does not automatically unpack that tuple into a parameter pack. Arity belongs to the declared expansion context.

### What does same shape mean, and how does it differ from runtime zip truncation?

Same shape means the expansion's packs have corresponding statically related positions. It is a type-checking relationship, not a promise that two runtime sequences happen to have equal counts. Runtime zip stops at the shorter sequence instead of diagnosing an arity mismatch.

### Which evaluations does an early return skip, and which call arguments have already been evaluated?

Early return skips subsequent comparisons or transformations in the function's pack iteration. It cannot skip constructing arguments whose expressions already ran at the call site. Trace argument construction separately from iteration to explain effects accurately.

### When would a collection or a few meaningful overloads be clearer than a pack API?

A collection is clearer for runtime-length homogeneous data. A few overloads are clearer when arities have different domain meaning or defaults. Packs fit mechanical repetitions with useful heterogeneous type relationships, provided diagnostics and deployment costs are justified.

## Focused exercise

**Task:** Implement a pack-based equality helper. Check zero, one, and heterogeneous inputs, plus an incompatible shape or element type.

**Acceptance checks:**

- Zero positions compare equal; one position uses its Equatable operation.
- Heterogeneous corresponding positions preserve their types.
- A differing value returns false; incompatible types or shape reject.

**Solution:**

Standalone pack-iteration example:

```swift
func equal<each T: Equatable>(
    _ lhs: (repeat each T), _ rhs: (repeat each T)
) -> Bool {
    for (left, right) in repeat (each lhs, each rhs) {
        if left != right { return false }
    }
    return true
}
precondition(equal((), ()))
precondition(equal(1, 1))
precondition(equal((1, "a", true), (1, "a", true)))
precondition(!equal((1, "a"), (1, "b")))
```

Compile equal((1, "a"), (1, 2)) separately as an intended element-type mismatch; also check incompatible arity with explicit types.

**Reasoning:** Each T supplies Equatable for its corresponding pair. The loop exits after the first unequal comparison, while argument construction has already occurred. A homogeneous [Any] would lose these per-position requirements and need a different comparison contract.

## In practice

**Choose it when:** Replace mechanical overload families while preserving typed positional relationships.

**Investigate by:** Test zero, one, many, incompatible shapes, and side-effectful argument construction.

**Strong evidence:** Compare the API's clarity and deployment constraints against a collection or meaningful overloads.

### Nominal runtime boundary and pack limitations — optional

#### Variadic generic types have their own runtime boundary

```swift
@available(macOS 14.0, iOS 17.0, *)
struct Route<each Input> {
    let build: (repeat each Input) -> String

    func callAsFunction(_ input: repeat each Input) -> String {
        build(repeat each input)
    }
}

@available(macOS 14.0, iOS 17.0, *)
func routeExample() -> String {
    let route = Route<Int, String> { id, section in
        "/users/\(id)/\(section)"
    }
    return route(42, "profile")
}
```

This returns `/users/42/profile`. It is a typed formatter example, not a URL-escaping or routing implementation. The stored closure is not automatically Sendable or concurrent.

The compiler rejects this generic type without availability protection when targeting macOS 13 or iOS 16, and accepts targets macOS 14/iOS 17. Pack functions in this chapter compile for the older targets; do not apply a blanket runtime minimum to every use of packs. A variadic typealias is another distinct case:

```swift
typealias Renderer<each Input> = (repeat each Input) -> String

func rendererExample() -> String {
    let render: Renderer<Int, String> = { "\($0):\($1)" }
    return render(42, "profile")
}
```

This typealias/function example compiles for macOS 13 and iOS 16. Variadic nominal types require metadata/runtime support, while a typealias does not introduce a new nominal runtime type. Replacing an existing non-variadic generic type with a variadic generic type is not a binary-compatible edit. [SE-0398](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0398-variadic-types.md)

#### Homogeneous variadics can still erase types through context

```swift
func collect<Value>(_ values: Value...) -> [Value] { values }

func explicitlyErased() -> [Any] {
    let values: [Any] = collect(1, "two")
    return values
}
```

The uncontextualized `collect(1, "two")` is rejected in the checked compiler for conflicting inferred types. The explicit `[Any]` result context above chooses one common element type and compiles. The distinction is one shared type versus preserved per-position types, not “mixed values are impossible in every ordinary variadic call.”

#### Type-preserving projection is not work scheduling

```swift
struct ValueBox<Value> { let value: Value }

func unpack<each Value>(
    _ boxes: repeat ValueBox<each Value>
) -> (repeat each Value) {
    (repeat (each boxes).value)
}
```

`unpack(ValueBox(value: 200), ValueBox(value: "OK"), ValueBox(value: true))` yields `(Int, String, Bool)`. It reads stored values; it does not fetch, launch child tasks, or fan out requests. A real asynchronous API needs an execution, cancellation, and sendability design.

A `ValueBox<Int>` also does not implicitly become `ValueBox<Any>`. Explicitly constructing `ValueBox<Any>(value: intBox.value)` is an erasure step. Pack-based projection avoids that step when the static relationship is the API's purpose.

#### Overload migration needs call-site checks

```swift
func choose() -> String { "zero" }
func choose<Value>(_ value: Value) -> String { "scalar" }
func choose<each Value>(_ values: repeat each Value) -> String { "pack" }
```

The checked calls select `zero` for no arguments, `scalar` for one, and `pack` for `(1, "a")`. Do not turn this example into an unconditional rule that every scalar overload wins: conversions and requirements still affect ranking. Two pack overloads whose labels differ can both accept no arguments, making an empty call ambiguous.

#### Current restrictions still matter

This intentionally fails with `enums cannot declare a type pack`:

```swift
enum PackedEnum<each Value> {
    case values(repeat each Value)
}
```

The checked compiler also rejects a generic type declaring two packs and subclassing a variadic generic class. Functions such as makePairs can declare multiple packs; that does not remove the nominal-type restriction. The unconstrained `each Value` examples retain an implicit Copyable requirement: passing a noncopyable token to packCount fails. See C8 before generalizing a helper to other ownership forms.

Parameter packs and result builders can cooperate: a builder can use a pack-based buildBlock to preserve component types. Builders are not limited to homogeneous or erased results. See B14 for the DSL transform and its tradeoffs.

## References

### Related guides

- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[B14 — Result Builders and DSL Construction]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[E7 — Resilience, ABI, and Module Stability]]
- [[F4 — Macros and compile-time code generation]]

### Official sources

- [SE-0393](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0393-parameter-packs.md)
- [SE-0398](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0398-variadic-types.md)
- [SE-0399: Tuple-of-pack expansion](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0399-tuple-of-value-pack-expansion.md)
- [SE-0408: Pack iteration](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0408-pack-iteration.md)
