---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - operators
  - literals
  - api-design
---

# A14 — Operators, Literals, and Language Affordances

**Learning goals:**

- Separate operator grouping, overload selection, operand evaluation, and domain behavior.
- Choose checked, wrapping, or reporting arithmetic and distinguish literals from explicit runtime conversion.
- Trace predicate construction versus matching and understand Swift 6.4 collection trailing-closure syntax.

**Prerequisites:** [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]/[[A8 — Closures, Escaping, and Capture Semantics]] for function parameters and autoclosures; [[A13 — Error handling and typed throws]] for recoverable validation errors.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

The expression left && right() has several stages: the parser groups it, the compiler chooses an overload, arguments are supplied under that overload’s contract, and the body performs an operation. Copying &&’s precedence onto your own operator copies its grouping, not its evaluation behavior.

**Precedence** chooses which operators bind more tightly; **associativity** chooses how repeated operators group. To skip work on the right, an API needs a closure or autoclosure and an implementation that conditionally invokes it. Symbols should express familiar domain meaning without hiding units, failure, mutation, or cost. [Advanced operators](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/advancedoperators/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Grouping changes policy

Standard && groups more tightly than ||. Parentheses can deliberately choose a different policy.

```swift
let paid = false
let active = false
let administrator = true
let independentAdmin = (paid && active) || administrator
let paymentRequired = paid && (active || administrator)
precondition(independentAdmin)
precondition(!paymentRequired)
```

**Walkthrough:** The first policy admits an administrator independently. The second still requires payment. This is a business decision, not a style-only choice of parentheses.

### Overflow behavior is part of the operation

Normal integer + checks overflow; &+ deliberately wraps; reporting overflow returns both a partial value and a flag.

```swift
func checked(_ a: UInt8, _ b: UInt8) -> UInt8 { a + b }
precondition(checked(1, 2) == 3)
let wrapped = UInt8.max &+ 1
let reported = UInt8.max.addingReportingOverflow(1)
precondition(wrapped == 0)
precondition(reported.partialValue == 0 && reported.overflow)
```

**Walkthrough:** Wrapping and reporting both produce partial value 0 here, but only reporting tells the caller that the mathematical sum was not representable. Calling checked with max and 1 would trap under the checked arithmetic contract; that call is never run. Unchecked optimization modes are not a domain validation policy.

## Rules and pitfalls

### Precedence does not make an operator lazy

**Common pitfall:** Receiving an ordinary Bool right operand and expecting internal && to undo its evaluation.

**Mistake:** Standalone runnable example.

```swift
infix operator &&& : LogicalConjunctionPrecedence
func &&& (lhs: Bool, rhs: Bool) -> Bool { lhs && rhs }
func probe() {
    var calls = 0
    func expensive() -> Bool {
        calls += 1
        return true
    }
    let result = false &&& expensive()
    precondition(!result && calls == 1)
}
probe()
```

**Consequence:** expensive runs even though lhs is false.

**Why:** The ordinary right argument was evaluated before entering &&&. Its implementation cannot undo that side effect.

**Correction:** Defer the expression and invoke it conditionally. This teaching operator uses an autoclosure.

```swift
infix operator &&? : LogicalConjunctionPrecedence
func &&? (lhs: Bool, rhs: @autoclosure () -> Bool) -> Bool { lhs && rhs() }
func probe() {
    var calls = 0
    func expensive() -> Bool {
        calls += 1
        return true
    }
    precondition(!(false &&? expensive()) && calls == 0)
    precondition(true &&? expensive())
    precondition(calls == 1)
}
probe()
```

**What changes:** The parameter contract now permits skipping evaluation, and the body actually skips it. Standard && already provides this behavior for ordinary production Boolean code.

### A literal precondition is runtime validation

**Common pitfall:** Claiming that ExpressibleByIntegerLiteral checks a percentage at compile time. This function is compile-only; invoking it would trap.

**Mistake:** Compile-only. The unsafe path is not executed.

```swift
struct Percentage: ExpressibleByIntegerLiteral {
    let value: Int
    init(integerLiteral value: Int) {
        precondition((0...100).contains(value))
        self.value = value
    }
}
func invalidLiteral() {
    let value: Percentage = 101
    _ = value
}
```

**Consequence:** The source compiles, but constructing the invalid literal would fail a runtime precondition.

**Why:** A literal initializer is ordinary initialization code. Its required witness cannot throw or fail to report ordinary invalid input.

**Correction:** Use checked construction for recoverable or untrusted input.

```swift
struct Percentage: Equatable {
    let value: Int
    enum ValidationError: Error, Equatable { case outOfRange(Int) }
    init(validating value: Int) throws(ValidationError) {
        guard (0...100).contains(value) else { throw .outOfRange(value) }
        self.value = value
    }
}
let zero = try Percentage(validating: 0)
let hundred = try Percentage(validating: 100)
precondition(zero.value == 0 && hundred.value == 100)
for invalid in [-1, 101] {
    do {
        _ = try Percentage(validating: invalid)
        preconditionFailure()
    } catch { precondition(error == .outOfRange(invalid)) }
}
```

**What changes:** The caller receives an explicit error and no invalid Percentage. This whole-number 0...100 policy does not model arbitrary ratios or rounding rules.

## Worked example

### Predicate construction and predicate execution happen at different times

**Setup:** Standalone script. Predicate is a small custom type, independent of Foundation.Predicate. Its conjunction returns a new matching closure.

**Predict:** Does a false left match prevent building the right predicate?

```swift
struct Predicate<Value> { let matches: (Value) -> Bool }
infix operator &&& : LogicalConjunctionPrecedence
func &&& <Value>(lhs: Predicate<Value>, rhs: Predicate<Value>) -> Predicate<Value> {
    Predicate { lhs.matches($0) && rhs.matches($0) }
}
func probe() -> [String] {
    var events: [String] = []
    func make(_ name: String, result: Bool) -> Predicate<Int> {
        events.append("build \(name)")
        return Predicate { _ in
            events.append("match \(name)")
            return result
        }
    }
    let combined = make("left", result: false) &&& make("right", result: true)
    precondition(events == ["build left", "build right"])
    precondition(!combined.matches(1))
    return events
}
precondition(probe() == ["build left", "build right", "match left"])
```

**Outcome:** Both predicates are built. Only the left matcher executes.

**Why:** The operator accepts already constructed predicates. Its returned closure uses standard && to short-circuit later matching. Deferring matching is distinct from deferring construction.

## Interview questions

### When does a symbol improve a domain API, and what can it hide?

**Answer:** Use a symbol when established notation expresses a stable operation clearly. Prefer a named method when units, failure, effects, or cost need to be visible. Domain types can preserve units independently of notation.

**Example:** Standalone runnable example.

```swift
struct Milliseconds: Equatable {
    let value: Int
    enum Failure: Error { case overflow }
    func adding(_ other: Self) throws(Failure) -> Self {
        let result = value.addingReportingOverflow(other.value)
        guard !result.overflow else { throw .overflow }
        return Self(value: result.partialValue)
    }
}
let total = try Milliseconds(value: 120).adding(Milliseconds(value: 80))
precondition(total.value == 200)
```

**Walkthrough:** The type states the unit; the named operation and try reveal recoverable arithmetic failure. A bare + between Ints would communicate neither the unit nor this failure policy.

**Boundary:** An operator could also be throwing, but would still need discoverable documentation and compatible overload/precedence design. Concision alone is not a reason to add symbolic vocabulary.

### Which parts of Predicate conjunction are eager, and what actually skips right-hand matching?

**Answer:** Both ordinary Predicate operands are constructed before the conjunction body runs. The resulting closure can later skip the right matcher with standard &&. Construction costs and matching costs are separate.

**Example:** Standalone runnable example.

```swift
struct Predicate<T> { let matches: (T) -> Bool }
func both<T>(_ left: Predicate<T>, _ right: Predicate<T>) -> Predicate<T> {
    Predicate { left.matches($0) && right.matches($0) }
}
func checkPredicates() {
    var builds = 0
    var rightMatches = 0
    func right() -> Predicate<Int> {
        builds += 1
        return Predicate { _ in
            rightMatches += 1
            return true
        }
    }
    let combined = both(Predicate { _ in false }, right())
    precondition(builds == 1 && rightMatches == 0)
    precondition(!combined.matches(1))
    precondition(rightMatches == 0)
    let reached = both(Predicate { _ in true }, right())
    precondition(reached.matches(1))
    precondition(builds == 2 && rightMatches == 1)
}
checkPredicates()
```

**Walkthrough:** right() runs once to build each right operand, including the one whose matcher is never called. Only a successful left match reaches the right matching closure.

**Boundary:** A closure or autoclosure parameter could defer construction too, but that would be a different API contract. These local mutable counters are synchronous test instrumentation, not Sendable predicates.

### Why does a literal construct a custom value while a String variable needs conversion? Can the literal initializer throw?

**Answer:** Literal syntax invokes a contextual type’s literal initializer. A String variable already has a type and needs explicit conversion. The literal witness must be nonthrowing and nonfailable; conformance does not also grant string interpolation.

**Example:** Standalone runnable example.

```swift
struct Label: ExpressibleByStringLiteral {
    let value: String
    init(_ value: String) { self.value = value }
    init(stringLiteral value: String) { self.init(value) }
}
let literal: Label = "Ada"
let raw: String = "Ben"
let converted = Label(raw)
precondition(literal.value == "Ada" && converted.value == "Ben")
```

**Walkthrough:** "Ada" is constructed as Label because of context. raw must pass through Label’s explicit initializer. [ExpressibleByStringLiteral](https://developer.apple.com/documentation/swift/expressiblebystringliteral)

**Boundary:** Runtime validation belongs in a separate checked initializer when failure is recoverable. A precondition in a trusted literal API is a runtime assertion, as shown in the pitfall.

Prepend that Label setup. These are separate **intentionally rejected** conversions:

```swift
let runtimeText = String("Ada")
let invalid: Label = runtimeText
```

```swift
let runtimeText = String("Ada")
let invalid: Label = "Hi \(runtimeText)"
```

Each standalone literal-conformance declaration below is also **intentionally rejected**, because its witness fails the nonfailable/nonthrowing requirement:

```swift
struct FailableLiteral: ExpressibleByStringLiteral {
    let value: String
    init?(stringLiteral value: String) {
        guard !value.isEmpty else { return nil }
        self.value = value
    }
}
```

```swift
struct ThrowingLiteral: ExpressibleByStringLiteral {
    typealias StringLiteralType = String
    typealias ExtendedGraphemeClusterLiteralType = String
    typealias UnicodeScalarLiteralType = String
    init(stringLiteral value: String) throws {}
}
```

### How can parentheses change eligibility, and how can precedence/overload changes affect clients?

**Answer:** Parentheses choose which conditions combine first and can therefore change who qualifies. Public precedence or overload changes can regroup or resolve existing expressions differently on rebuild, so they are source API changes.

**Example:** Standalone runnable example.

```swift
func policies(paid: Bool, active: Bool, admin: Bool) -> (Bool, Bool) {
    ((paid && active) || admin, paid && (active || admin))
}
let administrator = policies(paid: false, active: false, admin: true)
precondition(administrator.0 && !administrator.1)
let subscriber = policies(paid: true, active: true, admin: false)
precondition(subscriber.0 && subscriber.1)
```

**Walkthrough:** An unpaid administrator is admitted by the first expression and rejected by the second. An active paid subscriber satisfies both.

**Boundary:** A rebuilt client can change behavior even when the expression’s text is unchanged after an operator API update. Keep mixed-operator call sites and explicit policy cases in compatibility review.

### Extension: what does Swift 6.4 collection trailing-closure syntax add?

**Answer:** It permits the syntax to attach a trailing closure to array/dictionary expressions. It does not add standard builder initializers or make every array value callable. The selected initializer or callAsFunction must still exist.

**Example:** Standalone runnable example.

```swift
extension Array {
    init(buildValues: () -> [Element]) { self = buildValues() }
}
extension Dictionary {
    init(buildPairs: () -> [(Key, Value)]) {
        self.init()
        for (key, value) in buildPairs() { self[key] = value }
    }
}
let values = [Int] { [1, 2, 3] }
let settings = [String: Int] { [("attempts", 1), ("attempts", 2)] }
precondition(values == [1, 2, 3])
precondition(settings == ["attempts": 2])
```

**Walkthrough:** The user-defined initializers accept the closures. Duplicate keys deliberately retain the last supplied value. No result builder is involved. [SE-0508](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md)

**Boundary:** Removing those extensions makes these calls fail. In 6.4, an otherwise independent bare closure on the next line after an array literal can attach to it; name the closure or insert an explicit statement boundary. [[B14 — Result Builders and DSL Construction|B14]] covers result builders and callable collection values.

Without those initializer extensions, this independent block is **intentionally rejected**:

```swift
let values = [Int] { [1, 2, 3] }
```

## Focused exercise

**Task:** Compare an eager Boolean operator with an autoclosure-based short-circuiting operator.

**Acceptance checks:**

- False lhs still evaluates the eager operator’s right expression.
- False lhs skips the lazy operator’s right expression; true lhs evaluates it once.
- Both operators use the same precedence, isolating the parameter-contract difference.

**Solution:** Standalone synchronous event trace.

```swift
infix operator &&& : LogicalConjunctionPrecedence
infix operator &&? : LogicalConjunctionPrecedence
func &&& (lhs: Bool, rhs: Bool) -> Bool { lhs && rhs }
func &&? (lhs: Bool, rhs: @autoclosure () -> Bool) -> Bool { lhs && rhs() }
func evaluationProbe() -> [String] {
    var events: [String] = []
    func mark(_ event: String) -> Bool {
        events.append(event)
        return true
    }
    _ = false && mark("standard")
    _ = false &&& mark("eager")
    _ = false &&? mark("lazy")
    _ = true &&? mark("needed")
    return events
}
precondition(evaluationProbe() == ["eager", "needed"])
```

**Reasoning:** The eager right expression executes before &&& begins. The autoclosure version moves that expression behind its body’s conditional invocation. Both custom operators are teaching fixtures; ordinary Boolean code already has standard &&.

## In practice

**Choose it when:** Add symbols only when they make the domain contract clearer than a named alternative.

**Investigate by:** Trace parsing, overload resolution, operand construction, and later invocation as distinct stages.

**Strong evidence:** Verify units, failure, side effects, and compatibility under representative mixed-operator expressions.

## References

### Related guides

- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A10 — Extensions and Retroactive Modeling]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[B14 — Result Builders and DSL Construction]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

### Official sources

- [Advanced operators](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/advancedoperators/)
- [SE-0508](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md)
- [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [Basic Operators](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/basicoperators/)
- [Declarations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/)
- [ExpressibleByStringLiteral](https://developer.apple.com/documentation/swift/expressiblebystringliteral)
- [ExpressibleByStringInterpolation](https://developer.apple.com/documentation/swift/expressiblebystringinterpolation)
- [ExpressibleByIntegerLiteral](https://developer.apple.com/documentation/swift/expressiblebyintegerliteral)
