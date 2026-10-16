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

**Goal:** Trace eager operand evaluation and deliberate short-circuiting in two operators.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Operator syntax controls grouping; the selected implementation controls behavior. Precedence and associativity do not defer argument evaluation. To skip a right-hand expression, the API must receive it as a closure or autoclosure and conditionally invoke it. Clear domain meaning matters more than reducing characters. [Advanced operators](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/advancedoperators/)

## Rules and pitfalls

### Separate grouping from evaluation

Use parentheses for policy expressions whose meaning is easy to misread. A custom Boolean operator with ordinary Bool parameters receives already evaluated operands; a short-circuiting implementation needs a deferred right side.

**Common pitfall:** Copying &&-like precedence onto an eager function does not copy && semantics. Test right-side effects and document evaluation behavior, including throws and cost.

### Keep numeric and literal contracts explicit

Normal integer arithmetic checks overflow; wrapping operators intentionally discard overflow, while reporting APIs return the partial result and overflow flag. Choose the behavior from the domain. Literal conformance constructs a value from literal syntax, not an implicit conversion from any same-looking variable.

**Common pitfall:** A nonthrowing, nonfailable literal requirement cannot report ordinary validation failure. A precondition in it is a runtime trap, not compile-time validation. Use a checked initializer for untrusted values and specify units and arithmetic bounds.

## Interview questions

### When does a symbol improve a domain API, and what failure, unit, effect, or cost does it hide?

A symbol helps when established notation expresses a stable, widely understood operation. Prefer a named method when it reveals units, failure, mutation, or expensive work that the symbol hides. Compare representative call sites and the public evolution cost rather than assuming concision improves an API.

### Which parts of Predicate conjunction are eager, and what actually skips its right-hand matching work?

Both Predicate operands can be constructed before conjunction runs. The resulting matcher can later skip invoking the right predicate when the left fails. Deferring matching is different from deferring construction; a side effect or cost in a predicate factory can still happen eagerly.

### Why does a literal construct a custom value while assigning a String variable fails? Can its literal initializer throw?

Literal syntax invokes the contextual type's literal initializer. A String variable already has a type and needs explicit conversion; literal conformance does not add general implicit conversion or string interpolation. The required literal initializer cannot throw or fail, so recoverable validation needs another API.

### How can changing parentheses alter an eligibility policy, and how can public precedence/overload changes affect existing clients?

Parentheses choose which eligibility conditions combine first, potentially changing the policy. Public precedence or overload additions can regroup or resolve existing expressions differently on rebuild. Treat operator vocabulary as source API and test old call sites, especially mixed operators.

## Focused exercise

**Task:** Compare an eager Boolean operator with an autoclosure-based short-circuiting operator. Trace which right-hand expressions execute.

**Acceptance checks:**

- The eager operator evaluates the right side despite false lhs.
- The autoclosure operator skips it for false and invokes it once for true.
- Grouping is identical; the parameter contracts explain the difference.

**Solution:**

Standalone synchronous evaluation trace:

```swift
infix operator &&&: LogicalConjunctionPrecedence
infix operator &&?: LogicalConjunctionPrecedence
func &&& (lhs: Bool, rhs: Bool) -> Bool { lhs && rhs }
func &&? (lhs: Bool, rhs: @autoclosure () -> Bool) -> Bool {
    lhs ? rhs() : false
}
func probe() {
    var calls = 0
    func right() -> Bool { calls += 1; return true }
    let eager = false &&& right()
    precondition(!eager && calls == 1)
    let deferred = false &&? right()
    precondition(!deferred && calls == 1)
    let reached = true &&? right()
    precondition(reached && calls == 2)
}
probe()
```

**Reasoning:** The eager implementation's internal && cannot undo evaluation that occurred before entry. The autoclosure moves that evaluation behind an explicit decision. These teaching operators add no advantage over standard && in production.

## In practice

**Choose it when:** Add symbolic vocabulary only when it makes a domain contract clearer than a named alternative.

**Investigate by:** Trace parsing, overload resolution, construction, and later invocation separately.

**Strong evidence:** Check side effects, failure, unit semantics, and compatibility under representative client expressions.

### Literal and Swift 6.4 syntax detail — optional

#### Literal protocols cannot express ordinary initialization failure

Each block below is an independent **expected rejection**:

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
    let value: String
    init(stringLiteral value: String) throws {
        self.value = value
    }
}
```

Swift 6.4 rejects the failable witness for a non-failable requirement and the throwing witness for a non-throwing requirement. An implementation can still precondition or trap; protocol conformance does not prove that every literal succeeds. A precondition in a literal initializer executes at runtime and is not a compiler check of the domain rule.

Represent recoverable invalid input explicitly:

```swift
struct Percentage: Equatable {
    let value: Int
    enum ValidationError: Error { case outOfRange(Int) }

    init(validating value: Int) throws {
        guard (0...100).contains(value) else {
            throw ValidationError.outOfRange(value)
        }
        self.value = value
    }
}
```

Zero and 100 are valid; -1 and 101 throw and no Percentage is created. This is a whole-number percentage with an explicit 0...100 policy. It does not claim to model arbitrary ratios, rounding, or financial discount rules. A literal API for trusted fixtures can deliberately trap, but its documentation should say so. Do not describe that tradeoff as compile-time validation or silently accept invalid production state.

#### Swift 6.4 array/dictionary trailing closures

These are user-defined initializers; neither uses a result builder:

```swift
extension Array {
    init(buildValues: () -> [Element]) {
        self = buildValues()
    }
}

extension Dictionary {
    init(buildPairs: () -> [(Key, Value)]) {
        self.init()
        for (key, value) in buildPairs() { self[key] = value }
    }
}

func collectionSyntaxExample() -> ([Int], [String: Int]) {
    let values = [Int] { [1, 2, 3] }
    let settings = [String: Int] { [("attempts", 1), ("attempts", 2)] }
    return (values, settings)
}
```

The results are `[1, 2, 3]` and `["attempts": 2]`; duplicate keys deliberately use the last supplied value. Explicit `[Int].init { ... }` also works. Removing the corresponding extensions makes these examples fail: new parsing does not add those initializers. Similarly, applying a trailing closure to an array *value* needs a suitable callAsFunction. B14 contains that case and the result-builder examples.

When migrating from 6.2/6.3, a bare closure on the line after an array literal can now attach to that expression. Name the closure or use an explicit statement boundary when it is meant to be independent. SE-0508 does not enable trailing closures after every scalar literal or introduce a standard ArrayBuilder. Do not promote its future directions into baseline exercises. [SE-0508](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md)

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
