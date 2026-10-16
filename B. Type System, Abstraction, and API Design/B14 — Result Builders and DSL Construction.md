---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
  - result-builders
  - dsl
---

# B14 — Result Builders and DSL Construction

**Goal:** Build a small ordered list DSL and identify the transformations required for an optional branch.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A result builder transforms selected bodies into calls to static builder methods. Expressions produce components, control-flow hooks combine alternatives, and a final conversion may produce another result type. The expressions and generated calls still execute at runtime. Builder syntax alone does not establish purity, lazy evaluation, isolation, or Sendable safety. [SE-0289](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md)

## Rules and pitfalls

### Define expression and component types explicitly

A builder can turn String expressions into [String] components, combine them, and map an absent branch to an empty component. A helper returning several values also needs a supported expression conversion. buildArray handles loop-produced components, not arbitrary array expressions automatically.

**Common pitfall:** Adding a for loop without buildArray or a branch without the required hook fails because the transform has no operation for that syntax. Add only the language constructs the DSL should support.

### Justify the DSL boundary

A builder helps nested structural composition; a flat array of meaningful values is often clearer. Changes to accepted expressions, overloads, and combination hooks can alter client inference and source compatibility.

**Common pitfall:** Assuming Swift 6.4 supplies a universal ArrayBuilder confuses parsing with APIs. The new array-expression trailing-closure syntax still requires a matching initializer; see [[A14 — Operators, Literals, and Language Affordances]].

## Interview questions

### Which hooks transform a list expression, an optional branch, a for-in loop, and the final result?

buildExpression converts each supported expression to a component; buildOptional handles an if without else; buildArray combines loop results; buildFinalResult optionally converts the combined component to the declared result. buildBlock or partial-block methods combine ordered components. Each hook has a separate role.

### Why does an explicit return of the required array compile, and when does the compiler warn that the builder is disabled?

An explicit return can disable builder transformation and return the required array directly. The compiler can warn when a declared builder attribute is effectively bypassed by that return. A type-correct explicit return is therefore not evidence that every builder hook was used.

### What must exist for [String] { ... } to type-check, and what changed from the older .init spelling?

Array must have an applicable initializer accepting that trailing closure with the intended builder attribute. Swift 6.4 permits the expression syntax to parse without the older .init spelling in the supported situation; it does not add the initializer or builder implementation.

### When are a plain array or model constructors clearer, and which builder changes could break a client's source?

Use plain arrays or model constructors when they already show the domain clearly. Builder hook, overload, component, or expression-conversion changes can break source or change inference. Evaluate the maintenance and diagnostic cost before choosing a DSL.

## Focused exercise

**Task:** Build a small list DSL supporting expressions and an optional branch. Check empty input and output order, then compare plain array construction.

**Acceptance checks:**

- Empty input produces an empty list.
- Optional inclusion preserves output order.
- The same result can be expressed with a plain array and conditional append.

**Solution:**

Standalone ordered list builder:

```swift
@resultBuilder
struct ListBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
    static func buildOptional(_ part: [String]?) -> [String] { part ?? [] }
}
func list(@ListBuilder _ body: () -> [String]) -> [String] { body() }
func make(include: Bool) -> [String] {
    list {
        "first"
        if include { "optional" }
        "last"
    }
}
precondition(list {} == [])
precondition(make(include: false) == ["first", "last"])
precondition(make(include: true) == ["first", "optional", "last"])
```

**Reasoning:** String is the expression type and [String] is the component and result type. buildOptional supplies an empty component for the absent branch. The closure executes when list calls it; the builder does not make it a stored or asynchronous description.

## In practice

**Choose it when:** Nested composition becomes clearer and supported control flow is worth the API cost.

**Investigate by:** Reduce a diagnostic to expression conversion, combination, branch, or final-result types.

**Strong evidence:** Compare the DSL against plain construction and test client inference before changing hooks.

### Partial blocks and syntax boundaries — optional

#### Partial blocks and final conversion

```swift
@resultBuilder
enum SumBuilder {
    static func buildExpression(_ value: Int) -> Int { value }
    static func buildBlock() -> Int { 0 }
    static func buildBlock(_ values: Int...) -> Int { -1 }
    static func buildPartialBlock(first: Int) -> Int { first }
    static func buildPartialBlock(accumulated: Int, next: Int) -> Int {
        accumulated + next
    }
    static func buildFinalResult(_ value: Int) -> String { "total=\(value)" }
}

func total(@SumBuilder _ content: () -> String) -> String {
    content()
}
```

`total {}` returns `"total=0"`; `total { 1; 2; 3 }` returns `"total=6"`. The deliberately recognizable -1 overload is not selected for the nonempty block: the available partial-block pair takes precedence. These methods combine already-produced components; they do not imply that later expression evaluation waits for each accumulation step. A temporary trace checks that ordering. [SE-0348](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0348-buildpartialblock.md)

Partial blocks can reduce fixed-arity overload families. They are not an automatic complexity improvement: repeatedly concatenating an ever-growing array can still copy substantial data. This sum example makes no benchmark claim and follows ordinary Int overflow behavior.

A pack-based alternative preserves heterogeneous positions:

```swift
@resultBuilder
enum TupleBuilder {
    static func buildBlock<each Value>(
        _ values: repeat each Value
    ) -> (repeat each Value) {
        (repeat each values)
    }
}

func makeTuple<Value>(@TupleBuilder _ content: () -> Value) -> Value {
    content()
}
```

`makeTuple {}` can produce Void, `makeTuple { 42 }` produces Int, and `makeTuple { 1; "a"; true }` produces `(Int, String, Bool)`. It supports neither arbitrary runtime arity nor control-flow forms without their hooks. See B12 for pack inference, shape, and ownership limits.

#### Array-expression trailing closures still need an API

Use the ListBuilder in this independent syntax fixture. It accepts both a String expression and a [String] helper result, unlike the narrower focused builder:

```swift
@resultBuilder struct ListBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildExpression(_ value: [String]) -> [String] { value }
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
}
```

Define a user initializer:

```swift
extension Array where Element == String {
    init(@ListBuilder<String> build: () -> [String]) {
        self = build()
    }
}

func arraySyntaxExample() -> [[String]] {
    let concise = [String] {
        "A"
        "B"
    }
    let explicit = [String].init {
        "A"
        "B"
    }
    return [concise, explicit]
}
```

Both arrays are `["A", "B"]`. The direct `[String] { ... }` spelling is the 6.4 parser change; the explicit `.init` form remains useful for 6.2/6.3 source compatibility. Without the custom initializer, the new spelling still fails type checking. No built-in result builder is supplied. [SE-0508](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md)

Dictionary expressions benefit too. This separate example uses an ordinary closure, with an explicit last-value-wins duplicate-key policy:

```swift
extension Dictionary where Key == String, Value == Int {
    init(buildEntries: () -> [(String, Int)]) {
        self.init(buildEntries(), uniquingKeysWith: { _, new in new })
    }
}

func dictionarySyntaxExample() -> [String: Int] {
    [String: Int] {
        [("attempts", 1), ("attempts", 2)]
    }
}
```

It returns `["attempts": 2]`. The parser change is not tied to builders, nor does it dictate duplicate handling. Array/dictionary values with a suitable user-defined callAsFunction can also use the syntax; ordinary arrays are not callable without such an API.

#### Recheck standalone closure expressions after literals

A newline alone does not necessarily separate a following closure from an array/dictionary expression under the new parser. For a DSL accumulating closures, make the intended closure value explicit:

```swift
func deferredCounts() -> [() -> Int] {
    makeList {
        let numbers = [1, 2, 3]
        let action = { numbers.count }
        action
    }
}
```

This stores one closure, whose later invocation returns 3. Naming the action separates data initialization from the contributing expression. The migration probe also checks the fragile bare-closure form and a statement-terminator repair. SE-0508 documents this narrow source-compatibility edge and an init-accessor ordering edge; it does not change the ABI. [SE-0508](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md)

#### Building a description does not execute its domain

The route example below only constructs data. It does not install a router or enforce authentication. Likewise, a builder storing a closure has not necessarily executed that closure. Conversely, calling a synchronous side-effecting function to produce a component executes that call while the body runs. Test the wrapper's invocation count and the component effects separately.

The checked compiler fails to produce a useful diagnostic for two malformed generic calls: an inline closure returning String where a list is required, and a conditional in the pack builder without an optional hook. Separating the ordinary closure type or applying the tuple builder directly produces the expected type/missing-hook diagnostics. The validation record retains both forms.

Break large ambiguous expressions into explicitly typed helpers or smaller model/view declarations. Treat overload changes as public API changes. Adding a broad buildExpression overload may change inference or accept mistakes; returning a final array does not guarantee the compiler can infer every earlier expression backward from it.

## References

### Related guides

- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A14 — Operators, Literals, and Language Affordances]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[B12 — Variadic generics and parameter packs]]
- [[B13 — Property Wrappers and Generated Storage Semantics]]
- [[D5 — Global Actors and `@MainActor`]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F4 — Macros and compile-time code generation]]

### Official sources

- [SE-0289](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md)
- [SE-0348](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0348-buildpartialblock.md)
- [SE-0508](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md)
- [SE-0373: Local variables in result builders](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0373-vars-without-limits-in-result-builders.md)
- [Swift language reference: Attributes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/)
- [Apple: ViewBuilder](https://developer.apple.com/documentation/swiftui/viewbuilder)
