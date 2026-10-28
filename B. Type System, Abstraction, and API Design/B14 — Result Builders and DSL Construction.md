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

**Learning goals:**

- Translate a small builder body into expression, component, combination, control-flow, and final-result steps.
- Explain eager evaluation, explicit-return opt-out, partial blocks, and typed pack-based construction.
- Justify a DSL against plain data construction and distinguish builder APIs from Swift 6.4 trailing-closure parsing.

**Prerequisites:** [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]/[[A8 — Closures, Escaping, and Capture Semantics]] for closure parameters and evaluation, [[A14 — Operators, Literals, and Language Affordances]] for literal/type syntax, and [[B12 — Variadic generics and parameter packs]] before the pack-based extension.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A menu can be built from an ordered list of labels, with an optional item in the middle. A **result builder** lets a closure spell those contributions as separate expressions. The compiler translates the selected body into calls to static builder methods; those calls decide which expression types are accepted and how values combine.

Keep three types distinct: the **expression** you write (String), the **component** produced for combination ([String]), and the **final result** (perhaps [String], perhaps a Menu). The generated calls still execute at runtime when the receiving API invokes the closure. Builder syntax supplies no purity, laziness, actor isolation, or Sendable policy. [Result-builder transform](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md#the-result-builder-transform)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Convert expressions, then combine components

Standalone minimal builder. list executes its nonescaping closure once; the builder determines how that closure produces [String].

```swift
@resultBuilder
enum ListBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildBlock(_ parts: [String]...) -> [String] {
        parts.flatMap { $0 }
    }
}
func list(@ListBuilder _ body: () -> [String]) -> [String] { body() }
let labels = list {
    "first"
    "last"
}
let expanded = ListBuilder.buildBlock(
    ListBuilder.buildExpression("first"),
    ListBuilder.buildExpression("last")
)
precondition(labels == ["first", "last"] && labels == expanded)
precondition(list {} == [])
```

**Walkthrough:** Each String becomes a one-element component. buildBlock flattens components in order, including zero components for the empty body. expanded spells the important calls directly; result-builder transformation is not a new collection type.

### Control flow needs its own hooks

Shared MenuBuilder setup. This builder accepts individual labels and an array-returning helper, supports branches and loops, and converts its [String] component to a Menu result.

```swift
struct Menu: Equatable { let labels: [String] }
@resultBuilder
enum MenuBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildExpression(_ value: [String]) -> [String] { value }
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
    static func buildOptional(_ part: [String]?) -> [String] { part ?? [] }
    static func buildEither(first: [String]) -> [String] { first }
    static func buildEither(second: [String]) -> [String] { second }
    static func buildArray(_ parts: [[String]]) -> [String] { parts.flatMap { $0 } }
    static func buildFinalResult(_ part: [String]) -> Menu { Menu(labels: part) }
}
func menu(@MenuBuilder _ body: () -> Menu) -> Menu { body() }
func extras() -> [String] { ["help", "about"] }
let showAdmin = false
let result = menu {
    "home"
    if showAdmin { "admin" }
    if showAdmin { "sign out" } else { "sign in" }
    for label in ["search", "settings"] { label }
    extras()
}
precondition(result.labels == ["home", "sign in", "search", "settings", "help", "about"])
```

**Walkthrough:** The absent if contributes []; buildEither selects sign in; the loop supplies an array of components to buildArray. extras is accepted by buildExpression([String]), independently of loop support. buildFinalResult then wraps the combined labels in Menu.

## Rules and pitfalls

### A loop is not an ordinary array expression

**Common pitfall:** Adding a for loop to a builder that only defines expression conversion and block combination.

**Mistake:** Compile-only; intentionally rejected.

```swift
@resultBuilder
enum SmallBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
}
@SmallBuilder func labels() -> [String] {
    for label in ["A", "B"] { label }
}
```

**Consequence:** The compiler rejects the unsupported control-flow construct.

**Why:** The loop produces multiple component results, and the transform needs buildArray to combine them. Having [String] components alone does not supply that hook.

**Correction:** Add the specific loop operation. This standalone correction keeps the same expression/component types.

```swift
@resultBuilder
enum SmallBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
    static func buildArray(_ parts: [[String]]) -> [String] { parts.flatMap { $0 } }
}
@SmallBuilder func labels() -> [String] {
    for label in ["A", "B"] { label }
}
precondition(labels() == ["A", "B"])
```

**What changes:** The loop results can now combine. An if without else would separately need buildOptional; an array-valued helper still needs a matching expression conversion.

### Trailing-closure parsing does not supply an Array builder API

**Common pitfall:** Assuming Swift 6.4 provides a universal Array initializer accepting a builder closure.

**Mistake:** Compile-only; intentionally rejected.

```swift
let labels = [String] {
    "A"
    "B"
}
```

**Consequence:** The syntax parses, but no matching standard Array initializer accepts this closure.

**Why:** SE-0508 adds a spelling for an applicable initializer or callable value. The initializer and any builder must still exist in the program.

**Correction:** Provide the API explicitly. The nongeneric StringListBuilder is used as @StringListBuilder, with no invented generic argument.

```swift
@resultBuilder
enum StringListBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildExpression(_ value: [String]) -> [String] { value }
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
}
extension Array where Element == String {
    init(@StringListBuilder build: () -> [String]) { self = build() }
}
let concise = [String] {
    "A"
    "B"
}
let explicit = [String].init {
    "A"
    "B"
}
precondition(concise == ["A", "B"] && explicit == concise)
```

**What changes:** Both spellings now call the same user-defined initializer. The direct spelling is new in 6.4; .init remains useful for older compiler source compatibility. The library policy comes from this code. [Array-expression trailing closures](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md)

## Worked example

### Construction effects and stored actions occur at different times

**Setup:** Standalone data-only Route builder. Log records synchronous construction and later action invocation. It performs no router installation or asynchronous work.

**Predict:** Which events exist immediately after routes returns, and which appear only after calling an action?

```swift
final class Log { var events: [String] = [] }
struct Route {
    let path: String
    let action: () -> String
}
@resultBuilder
enum RouteBuilder {
    static func buildBlock(_ routes: Route...) -> [Route] { routes }
}
func routes(@RouteBuilder _ body: () -> [Route]) -> [Route] { body() }
func makeRoute(_ path: String, log: Log) -> Route {
    log.events.append("build " + path)
    return Route(
        path: path,
        action: {
            log.events.append("use " + path)
            return path
        })
}
let log = Log()
let built = routes {
    makeRoute("/home", log: log)
    makeRoute("/help", log: log)
}
precondition(log.events == ["build /home", "build /help"])
precondition(built.map(\.path) == ["/home", "/help"])
precondition(built[0].action() == "/home")
precondition(log.events == ["build /home", "build /help", "use /home"])
```

**Outcome:** Both build events occur during body execution. The use /home event appears only after the explicit action call.

**Why:** makeRoute is an ordinary effectful function evaluated to produce a component. The closure it stores is a value whose body is not invoked by RouteBuilder. The builder attribute alone would tell us neither fact without inspecting the receiving API and component code.

## Interview questions

### Which hooks transform a list expression, an optional branch, a for-in loop, and the final result?

**Answer:** buildExpression converts a supported expression to a component. buildOptional handles an if without else; buildArray combines loop results; buildFinalResult converts the combined component to the declared result. buildBlock or a partial-block pair performs ordered combination.

**Example:** Prepend MenuBuilder setup.

```swift
func makeMenu(includeHelp: Bool) -> Menu {
    menu {
        "home"
        if includeHelp { extras() }
        for label in ["search", "settings"] { label }
    }
}
precondition(makeMenu(includeHelp: false).labels == ["home", "search", "settings"])
precondition(makeMenu(includeHelp: true).labels == ["home", "help", "about", "search", "settings"])
```

**Walkthrough:** The String home becomes [home]. extras already returns [String], accepted through its own expression overload. buildOptional supplies or omits that component, and buildArray flattens the loop’s results before final Menu conversion.

**Boundary:** buildArray does not automatically accept arbitrary array-valued expressions. Add only syntax and conversions the DSL intends to support; broad conversions can make mistakes harder to diagnose.

### Why does an explicit return of the required array compile, and when does the compiler warn that the builder is disabled?

**Answer:** An explicit return can opt the body out of builder transformation and return its required result directly. A declared builder function with that return produces a warning in this compiler; a closure argument can opt out without the same warning.

**Example:** Standalone runnable example.

```swift
@resultBuilder
enum ListBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
}
@ListBuilder func direct() -> [String] {
    return ["A", "B"]
}
func list(@ListBuilder _ body: () -> [String]) -> [String] { body() }
precondition(direct() == ["A", "B"])
precondition(list { return ["A", "B"] } == ["A", "B"])
```

**Walkthrough:** Compile without -warnings-as-errors for the stated disabled-transform warning. Neither returned array needs the builder’s String conversion. A successful result therefore does not prove that all builder hooks were invoked.

**Boundary:** An explicit return still has to produce the required result type. Do not rely on builder inference to repair an ordinary wrong return type after opting out.

### What must exist for [String] { ... } to type-check, and what changed from the older .init spelling?

**Answer:** Array needs a matching initializer accepting the closure. A builder-based initializer additionally needs the actual builder declaration and supported hooks. Swift 6.4 changes parsing of the direct array/dictionary expression spelling, not the API requirements.

**Example:** Prepend StringListBuilder and custom Array initializer from the corrected trailing-closure pitfall.

```swift
func helperLabels() -> [String] { ["B", "C"] }
let direct = [String] {
    "A"
    helperLabels()
}
let explicitInit = [String].init {
    "A"
    helperLabels()
}
precondition(direct == ["A", "B", "C"] && direct == explicitInit)
```

**Walkthrough:** The custom initializer receives a closure transformed by StringListBuilder. helperLabels works because that builder explicitly accepts [String] expressions. Both spellings invoke the same implementation.

**Boundary:** Ordinary array values are not callable unless an appropriate callAsFunction API also exists. The parser change is not tied to result builders, as the dictionary example below shows.

Standalone dictionary counterpart. This closure is ordinary Swift, and the initializer deliberately chooses the last duplicate-key value:

```swift
extension Dictionary where Key == String, Value == Int {
    init(buildEntries: () -> [(String, Int)]) {
        self.init(buildEntries(), uniquingKeysWith: { _, new in new })
    }
}
let attempts = [String: Int] { [("attempts", 1), ("attempts", 2)] }
precondition(attempts == ["attempts": 2])
```

### When are a plain array or model constructors clearer, and which builder changes could break a client’s source?

**Answer:** Use ordinary construction when it already expresses the structure clearly. Builder expression overloads, component types, and combination hooks form a source API: removing or broadening them can reject clients or change inference and meaning.

**Example:** Prepend MenuBuilder setup.

```swift
func plain(includeHelp: Bool) -> Menu {
    var labels = ["home"]
    if includeHelp { labels += extras() }
    labels += ["search", "settings"]
    return Menu(labels: labels)
}
func withDSL(includeHelp: Bool) -> Menu {
    menu {
        "home"
        if includeHelp { extras() }
        for label in ["search", "settings"] { label }
    }
}
for include in [false, true] {
    precondition(plain(includeHelp: include) == withDSL(includeHelp: include))
}
```

**Walkthrough:** Both versions visibly produce the same labels for both branches. The DSL saves some combination syntax; the ordinary form needs fewer custom language hooks. Removing MenuBuilder.buildExpression([String]) would break the extras() contribution even though buildArray remains.

**Boundary:** Adding a broad conversion is also a compatibility decision because it changes what expressions can type-check. Test representative clients and use typed helpers to localize inference problems; no transform supplies SwiftUI identity or observation semantics.

Prepend MenuBuilder setup. This separate builder intentionally omits the array-expression conversion; using extras() is **rejected** even though loop support exists:

```swift
@resultBuilder
enum NarrowBuilder {
    static func buildExpression(_ value: String) -> [String] { [value] }
    static func buildBlock(_ parts: [String]...) -> [String] { parts.flatMap { $0 } }
    static func buildArray(_ parts: [[String]]) -> [String] { parts.flatMap { $0 } }
}
@NarrowBuilder func unsupported() -> [String] { extras() }
```

### Extension: How do partial blocks and final conversion interact?

**Answer:** When the applicable first/accumulated partial-block pair is available, a nonempty block uses it instead of the nonempty buildBlock overload. An empty block still needs its empty combination. buildFinalResult can then change the combined type.

**Example:** Standalone runnable example.

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
func total(@SumBuilder _ content: () -> String) -> String { content() }
precondition(total {} == "total=0")
precondition(
    total {
        1
        2
        3
    } == "total=6")
```

**Walkthrough:** The empty component is 0. The nonempty body combines 1, 2, and 3 through partial blocks, so the recognizable -1 buildBlock result is never used. Final conversion produces a String. [Partial blocks](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0348-buildpartialblock.md)

**Boundary:** Combination follows production of expression components; it does not imply that later expressions wait for an accumulation call. The trace below tests that distinction. Repeated array concatenation can still be costly; partial blocks are not a benchmark result.

Standalone trace of expression production versus accumulation. The Log reference is used only synchronously; no global mutable state is needed:

```swift
final class Log { var events: [String] = [] }
struct Part {
    let value: Int
    let log: Log
}
func part(_ value: Int, log: Log) -> Part {
    log.events.append("expression")
    return Part(value: value, log: log)
}
@resultBuilder
enum TraceBuilder {
    static func buildPartialBlock(first: Part) -> Part {
        first.log.events.append("first")
        return first
    }
    static func buildPartialBlock(accumulated: Part, next: Part) -> Part {
        accumulated.log.events.append("combine")
        return Part(value: accumulated.value + next.value, log: accumulated.log)
    }
    static func buildFinalResult(_ part: Part) -> Int {
        part.log.events.append("final")
        return part.value
    }
}
func traced(@TraceBuilder _ body: () -> Int) -> Int { body() }
let log = Log()
let sum = traced {
    part(1, log: log)
    part(2, log: log)
    part(3, log: log)
}
precondition(sum == 6)
precondition(
    log.events == [
        "expression", "expression", "expression", "first", "combine", "combine", "final",
    ])
```

### Extension: Must every builder produce one homogeneous collection?

**Answer:** No. A pack-based buildBlock can preserve each component’s static type in a tuple. Its zero, single, and many-position results follow the same pack/tuple rules as an ordinary pack function.

**Example:** Standalone runnable example.

```swift
@resultBuilder
enum TupleBuilder {
    static func buildBlock<each Value>(_ values: repeat each Value) -> (repeat each Value) {
        (repeat each values)
    }
}
func makeTuple<Value>(@TupleBuilder _ content: () -> Value) -> Value { content() }
let empty: Void = makeTuple {}
let single: Int = makeTuple { 42 }
let values: (Int, String, Bool) = makeTuple {
    1
    "A"
    true
}
precondition(single == 42)
precondition(values.0 == 1 && values.1 == "A" && values.2)
_ = empty
```

**Walkthrough:** The results are Void, Int, and the stated heterogeneous tuple. No array of Any or mechanical fixed-arity buildBlock family is needed.

**Boundary:** This builder has no optional, branch, or loop hooks. It supports statically related arity, not an arbitrary runtime-length tuple. See [[B12 — Variadic generics and parameter packs|B12]] for availability and ownership boundaries.

Prepend TupleBuilder setup. Applying it directly to a function exposes the intended missing-optional-hook diagnostic in this **rejected** conditional example:

```swift
@TupleBuilder func conditional(_ include: Bool) -> Int? {
    if include { 1 }
}
```

### Extension: Why should closure-valued DSL expressions after array literals be made explicit?

**Answer:** The 6.4 parser may attach a following brace to an array or dictionary expression as a trailing closure. Name the intended closure value, then contribute that name to the builder.

**Example:** Standalone runnable example.

```swift
@resultBuilder
enum ArrayBuilder<Element> {
    static func buildExpression(_ value: Element) -> [Element] { [value] }
    static func buildBlock(_ parts: [Element]...) -> [Element] { parts.flatMap { $0 } }
}
func makeList<Element>(@ArrayBuilder<Element> _ body: () -> [Element]) -> [Element] {
    body()
}
func deferredCounts() -> [() -> Int] {
    makeList {
        let numbers = [1, 2, 3]
        let action = { numbers.count }
        action
    }
}
let actions = deferredCounts()
precondition(actions.count == 1 && actions[0]() == 3)
```

**Walkthrough:** ArrayBuilder and makeList are fully declared here. The named action is one closure value, stored without invocation; calling it later returns 3. The rejected bare-closure form below shows the parsing edge. [Migration edge](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md#source-compatibility)

**Boundary:** A statement terminator can also separate the expressions, but naming the action makes the intent easier to read. This parser change does not make ordinary Array values callable.

Prepend ArrayBuilder/makeList setup. The following bare closure attaches to the array literal; it is **intentionally rejected**:

```swift
func fragileCounts() -> [() -> Int] {
    makeList {
        let numbers = [1, 2, 3]
        { numbers.count }
    }
}
```

Compiler-diagnostic limit: the historical validation found two malformed generic calls for which this compiler failed to produce a useful diagnostic. Typed ordinary-closure and direct TupleBuilder-function probes expose the real type/missing-hook errors. These failures are recorded as toolchain behavior, not a language rule, and are not presented as successful intended rejections.

## Focused exercise

**Task:** Build an ordered String list DSL supporting expressions and an optional branch; compare it with ordinary array construction.

**Acceptance checks:**

- An empty body produces [].
- Both optional-branch outcomes preserve first/optional/last ordering.
- The corresponding plain array construction has the same result for both inputs.

**Solution:** Standalone runnable implementation and checks:

```swift
@resultBuilder
enum ListBuilder {
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
func plain(include: Bool) -> [String] {
    var values = ["first"]
    if include { values.append("optional") }
    values.append("last")
    return values
}
precondition(list {} == [])
precondition(make(include: false) == ["first", "last"])
precondition(make(include: true) == ["first", "optional", "last"])
for include in [false, true] { precondition(make(include: include) == plain(include: include)) }
```

**Reasoning:** The String expression type becomes a [String] component. buildOptional maps an absent contribution to [], and combination preserves order. list invokes its closure once; no stored, asynchronous, or reactive description is implied by the attribute.

## In practice

**Choose it when:** Use a DSL when nested structural composition becomes clearer and its supported syntax repays the custom API and diagnostic cost.

**Investigate by:** Reduce failures to expression conversion, component combination, a missing control-flow hook, or final-result conversion; then inspect closure invocation and ordinary effects separately.

**Strong evidence:** Compare equivalent plain construction, check empty and control-flow cases, and test representative client inference before changing public hooks or overloads.

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
