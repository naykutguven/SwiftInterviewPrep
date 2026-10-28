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

**Learning goals:**

- Read each/repeat, typed arity, corresponding positions, and pack iteration through zero, one, and many values.
- Distinguish call-argument evaluation, early return, and explicit work scheduling.
- Choose between packs, collections, and meaningful overloads while checking ownership, overload, and deployment boundaries.

**Prerequisites:** [[B3 — Generics, Constraints, and Same-Type Relationships]] for generic constraints and [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for ordinary parameter lists. [[D2 — Task Hierarchy, Cancellation, and Priorities]] supplies the task-group alternative used in one pitfall.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A logging function might accept an Int, a String, and a Bool without turning them all into Any. A **type parameter pack** names that ordered list of types. A corresponding value pack supplies one value for each position. Its **arity** is the number of positions; it belongs to the generic relationship rather than a runtime collection's count.

`each Value` refers to a pack's element type within an expansion pattern. `repeat` expands the pattern over corresponding positions. The compiler preserves the positional relationships; it does not promise a particular machine-code unrolling strategy. A homogeneous Array instead has one element type and a runtime length. [Parameter packs](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0393-parameter-packs.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Expand a typed projection

ValueBox stores one ordinary generic value. unpack applies the same projection at every position, preserving each result type. This is synchronous local work.

```swift
struct ValueBox<Value> { let value: Value }
func unpack<each Value>(
    _ boxes: repeat ValueBox<each Value>
) -> (repeat each Value) {
    (repeat (each boxes).value)
}
let empty: Void = unpack()
let single: Int = unpack(ValueBox(value: 200))
let response: (Int, String, Bool) = unpack(
    ValueBox(value: 200), ValueBox(value: "OK"), ValueBox(value: true)
)
precondition(single == 200)
precondition(response.0 == 200 && response.1 == "OK" && response.2)
_ = empty
```

**Walkthrough:** Zero positions produce Void, one produces Int, and three produce the specified tuple. The boxes remain ValueBox<Int>, ValueBox<String>, and ValueBox<Bool>; none becomes ValueBox<Any>.

### Iterate corresponding positions

Both tuple parameters expand the same type pack. Each pair therefore shares a type with an Equatable operation. Pack iteration allows ordinary control flow. [Pack iteration](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0408-pack-iteration.md)

```swift
func allEqual<each Element: Equatable>(
    _ lhs: (repeat each Element), _ rhs: (repeat each Element)
) -> Bool {
    for (left, right) in repeat (each lhs, each rhs) {
        if left != right { return false }
    }
    return true
}
precondition(allEqual((), ()))
precondition(allEqual(7, 7))
precondition(allEqual((7, "OK", true), (7, "OK", true)))
precondition(!allEqual((7, "OK"), (7, "bad")))
```

**Walkthrough:** The empty loop returns true. Scalar comparison runs once. The heterogeneous call compares Int with Int, then String with String, then Bool with Bool, without an Any cast. This helper does not add a tuple protocol conformance.

## Rules and pitfalls

### Runtime zip can hide a missing position

**Common pitfall:** Using zip to validate complete correspondence between runtime inputs without first checking their counts.

**Mistake:** Standalone runnable example.

```swift
let expected = [1, 2, 3]
let received = [1, 2]
let accepted = zip(expected, received).allSatisfy { $0 == $1 }
precondition(accepted)  // The third expected value was never examined.
```

**Consequence:** The shorter received input is accepted even though the requirement was complete equality.

**Why:** zip stops when either sequence ends. It describes runtime traversal, not a type-level arity relationship.

**Correction:** For runtime-length arrays, compare the whole arrays. Use a pack only when the API needs statically corresponding, possibly heterogeneous positions.

```swift
let expected = [1, 2, 3]
let received = [1, 2]
precondition(expected != received)
func equalPositions<each Value: Equatable>(
    _ lhs: (repeat each Value), _ rhs: (repeat each Value)
) -> Bool {
    for (left, right) in repeat (each lhs, each rhs) {
        if left != right { return false }
    }
    return true
}
precondition(equalPositions((1, "A"), (1, "A")))
```

**What changes:** Array equality accounts for length. The pack helper establishes a different static contract; its incompatible-arity call is rejected below.

### An expansion supplies repetition, not a scheduler

**Common pitfall:** Expecting a pack-based invocation helper to run independent operations concurrently.

**Mistake:** Standalone runnable example.

```swift
final class Log { var events: [String] = [] }
func invoke<each Value>(_ operations: repeat () -> each Value) -> (repeat each Value) {
    (repeat (each operations)())
}
let log = Log()
let values = invoke(
    {
        log.events.append("first")
        return "A"
    },
    {
        log.events.append("second")
        return "B"
    }
)
precondition(values.0 == "A" && values.1 == "B")
precondition(log.events == ["first", "second"])
```

**Consequence:** Both operations run synchronously as the expansion is evaluated.

**Why:** Neither each nor repeat creates tasks. The helper preserves result positions but has no concurrency mechanism.

**Correction:** When independent operations have one result type and concurrency is the requirement, make task ownership explicit. This local task-group example returns both labels without assuming completion order.

```swift
func loadLabels() async -> [String] {
    await withTaskGroup(of: String.self) { group in
        group.addTask { "A" }
        group.addTask { "B" }
        var labels: [String] = []
        for await label in group { labels.append(label) }
        return labels.sorted()
    }
}
let labels = await loadLabels()
precondition(labels == ["A", "B"])
```

**What changes:** The group creates and joins child tasks. Sorting makes the assertion independent of task completion order. It intentionally uses homogeneous results; an actual heterogeneous async API also needs ownership, cancellation, and sendability decisions.

## Worked example

### A tuple argument is still one position

**Setup:** Standalone arity probe. Counting uses one increment for each expanded argument.

**Predict:** Does passing (1, "A") as a single argument produce one position or two?

```swift
func packCount<each Value>(_ values: repeat each Value) -> Int {
    var count = 0
    for _ in repeat each values { count += 1 }
    return count
}
precondition(packCount() == 0)
precondition(packCount(1) == 1)
precondition(packCount(1, "A") == 2)
precondition(packCount((1, "A")) == 1)
```

**Outcome:** The counts are 0, 1, 2, and 1.

**Why:** Two separate arguments populate two pack positions. The final call passes one tuple value as its only position. Swift does not implicitly splat that tuple into separate arguments.

## Interview questions

### Why does one tuple expansion collapse to a scalar, while passing a concrete tuple as one argument does not unpack it?

**Answer:** Swift has no distinct one-element tuple type. An expansion with one position therefore yields that scalar, while a concrete tuple passed as one argument remains the value of that one position.

**Example:** Standalone runnable example.

```swift
func identity<each Value>(_ values: repeat each Value) -> (repeat each Value) {
    (repeat each values)
}
let scalar: Int = identity(7)
let twoArguments: (Int, String) = identity(7, "A")
let oneTupleArgument: (Int, String) = identity((7, "A"))
precondition(scalar == 7)
precondition(twoArguments.0 == oneTupleArgument.0)
precondition(twoArguments.1 == oneTupleArgument.1)
```

**Walkthrough:** The first result is Int. The other two results have the same final tuple type for different reasons: a two-position expansion versus a one-position expansion whose scalar value is itself a tuple. The arity probe above distinguishes them.

**Boundary:** A matching result shape alone does not reveal how many arguments populated the pack. This does not provide general tuple-unpacking call syntax.

### What does same shape mean, and how does it differ from runtime zip truncation?

**Answer:** Packs expanded together must have corresponding statically related positions. That is an arity constraint imposed by the generic signature. Runtime zip can pair differently sized collections and simply stop early.

**Example:** Standalone runnable example.

```swift
func makePairs<each Left, each Right>(
    _ lhs: (repeat each Left), _ rhs: (repeat each Right)
) -> (repeat (each Left, each Right)) {
    (repeat (each lhs, each rhs))
}
let pairs = makePairs((1, "A"), (true, 2.5))
precondition(pairs.0.0 == 1 && pairs.0.1)
precondition(pairs.1.0 == "A" && pairs.1.1 == 2.5)
precondition(Array(zip([1, 2], [true])).count == 1)
```

**Walkthrough:** makePairs preserves two positions with different Left/Right types in each. Expanding them together requires equal shape, not equal element types. allEqual adds the stronger condition that the two inputs use the same Element pack.

**Boundary:** Shape compatibility is a compiler relationship. It does not validate runtime array lengths or perform a cross-product of all values.

Prepend makePairs. This different-arity call is **intentionally rejected**; the explicit tuple types make the missing position visible:

```swift
let left: (Int, String) = (1, "A")
let right: Bool = true
let invalid = makePairs(left, right)
```

### Which evaluations does an early return skip, and which call arguments have already been evaluated?

**Answer:** It skips later work inside pack iteration, after call arguments have already been evaluated. Distinguish constructing each input from comparing each pair.

**Example:** Standalone runnable example.

```swift
final class Log { var events: [String] = [] }
struct Tracked: Equatable {
    let value: Int
    let log: Log
    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.log.events.append("compare")
        return lhs.value == rhs.value
    }
}
func make(_ value: Int, log: Log) -> Tracked {
    log.events.append("make")
    return Tracked(value: value, log: log)
}
func equal<each Value: Equatable>(
    _ lhs: (repeat each Value), _ rhs: (repeat each Value)
) -> Bool {
    for (left, right) in repeat (each lhs, each rhs) {
        if left != right { return false }
    }
    return true
}
let log = Log()
let result = equal(
    (make(1, log: log), make(2, log: log)),
    (make(9, log: log), make(2, log: log))
)
precondition(!result)
precondition(log.events == ["make", "make", "make", "make", "compare"])
```

**Walkthrough:** All four make calls finish before equal starts. The first pair differs, so only one comparison is recorded. Constructing the second pair was not avoided.

**Boundary:** This is eager argument evaluation. Passing closures could defer work, but that would be a different API whose invocation and effect order must be specified.

### When would a collection or a few meaningful overloads be clearer than a pack API?

**Answer:** Use a collection for runtime-length homogeneous values. Use a few overloads when arities mean different domain operations. Packs fit mechanical repetition whose positional types matter.

**Example:** Standalone runnable example.

```swift
func total(_ values: [Int]) -> Int { values.reduce(0, +) }
func greeting(to name: String) -> String { "Hello, " + name }
func greeting(to first: String, and second: String) -> String {
    "Hello, " + first + " and " + second
}
precondition(total([1, 2, 3]) == 6)
precondition(greeting(to: "Ada") == "Hello, Ada")
precondition(greeting(to: "Ada", and: "Lin") == "Hello, Ada and Lin")
```

**Walkthrough:** total accepts however many runtime values arrive. The greeting overloads make a specific two-person policy explicit. Neither benefits from preserving a heterogeneous type at every position.

**Boundary:** Choose packs when the relationship simplifies the public contract, then evaluate diagnostics and supported deployment targets. Removing a small, meaningful overload is not automatically an improvement.

### Extension: Do pack functions, aliases, and variadic nominal types have the same deployment boundary?

**Answer:** No. A variadic nominal type needs runtime metadata support; a pack-based function or alias must be checked separately. The Route example explicitly protects its macOS 14/iOS 17 nominal-type requirement.

**Example:** Standalone runnable example.

```swift
@available(macOS 14.0, iOS 17.0, *)
struct Route<each Input> {
    let build: (repeat each Input) -> String
    func callAsFunction(_ input: repeat each Input) -> String {
        build(repeat each input)
    }
}
let route = Route<Int, String> { "/users/\($0)/\($1)" }
precondition(route(42, "profile") == "/users/42/profile")
```

**Walkthrough:** On the chapter’s macOS 27 target it returns /users/42/profile. Lower-target controls reject an unguarded variadic nominal declaration for macOS 13/iOS 16 and accept its supported boundary. The function/alias example below compiles for those older targets. [Variadic types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0398-variadic-types.md)

**Boundary:** This is a typed formatter, not a URL-escaping or router implementation. The stored closure has no automatic Sendable or concurrency guarantee. Replacing an existing nonvariadic type with a variadic type is not a binary-compatible edit.

Standalone alias/function counterpart; no nominal Route is used:

```swift
typealias Renderer<each Input> = (repeat each Input) -> String
func renderExample() -> String {
    let render: Renderer<Int, String> = { "\($0):\($1)" }
    return render(42, "profile")
}
precondition(renderExample() == "42:profile")
```

### Extension: Can ordinary homogeneous variadics ever accept mixed concrete values?

**Answer:** Yes, when context explicitly chooses a common element type such as Any. That is one erased type shared by all arguments; a pack preserves each position’s type.

**Example:** Standalone runnable example.

```swift
func collect<Value>(_ values: Value...) -> [Value] { values }
let erased: [Any] = collect(1, "two")
precondition(erased[0] as? Int == 1 && erased[1] as? String == "two")
```

**Walkthrough:** The result context supplies Value == Any and each argument converts into that element type. The separate uncontextualized call below is rejected by this compiler for conflicting inferred types.

**Boundary:** Explicit erasure is a legitimate contract when downstream code needs dynamic values. It cannot provide the compile-time corresponding-type checks of allEqual.

Prepend collect. This uncontextualized mixed call is **intentionally rejected**:

```swift
let mixed = collect(1, "two")
```

### Extension: What overload and ownership restrictions must a migration check?

**Answer:** Check zero, one, and many call sites rather than assuming every scalar overload wins. These unconstrained pack elements also retain the implicit Copyable requirement. Nominal pack declarations have further restrictions.

**Example:** Standalone runnable example.

```swift
func choose() -> String { "zero" }
func choose<Value>(_ value: Value) -> String { "scalar" }
func choose<each Value>(_ values: repeat each Value) -> String { "pack" }
precondition(choose() == "zero")
precondition(choose(1) == "scalar")
precondition(choose(1, "A") == "pack")
```

**Walkthrough:** These three calls choose the named results in the checked compiler. The rejected examples below expose empty-call ambiguity, ownership defaults, and nominal restrictions instead of treating these results as a universal overload-ranking law.

**Boundary:** Conversions and extra constraints can affect overload ranking. Current nominal restrictions do not imply that a function cannot declare multiple packs; makePairs above does.

Standalone **intentionally rejected** empty call: both labeled pack overloads can accept zero arguments.

```swift
func choose<each Value>(left values: repeat each Value) {}
func choose<each Value>(right values: repeat each Value) {}
choose()
```

Standalone **intentionally rejected** noncopyable argument; borrowing alone does not suppress the generic Copyable default:

```swift
struct Ticket: ~Copyable { let id: Int }
func packCount<each Value>(_ values: repeat each Value) -> Int {
    var count = 0
    for _ in repeat each values { count += 1 }
    return count
}
func example() -> Int { packCount(Ticket(id: 1)) }
```

Separate **intentionally rejected** nominal declarations: an enum pack, two nominal packs, and subclassing a variadic generic class.

```swift
enum PackedEnum<each Value> { case values(repeat each Value) }
```

```swift
struct TwoPacks<each Left, each Right> {}
```

```swift
class Base<each Value> {}
class Child: Base<Int, String> {}
```

## Focused exercise

**Task:** Implement pack-based equality across zero, one, and heterogeneous positions. Explain where shape, type, and value mismatches are detected.

**Acceptance checks:**

- Empty inputs compare equal and scalar inputs use their Equatable operation.
- Matching heterogeneous positions compare without casts; an unequal position returns false.
- Explicit incompatible element types and arities are rejected separately.

**Solution:** Standalone runnable solution:

```swift
func equal<each Value: Equatable>(
    _ lhs: (repeat each Value), _ rhs: (repeat each Value)
) -> Bool {
    for (left, right) in repeat (each lhs, each rhs) {
        guard left == right else { return false }
    }
    return true
}
precondition(equal((), ()))
precondition(equal(1, 1))
precondition(!equal(1, 2))
precondition(equal((1, "A", true), (1, "A", true)))
precondition(!equal((1, "A", true), (1, "B", true)))
```

**Reasoning:** The shared Value pack establishes corresponding types before execution. Each iteration then decides value equality. No comparison runs for an empty pack; return false skips later comparisons while preserving the earlier call-argument evaluation rule.

Prepend the solution to each **intentionally rejected** block independently. Fixing the first call cannot make the second shape valid:

```swift
let lhs: (Int, String) = (1, "A")
let rhs: (Int, Int) = (1, 2)
let invalid = equal(lhs, rhs)
```

```swift
let lhs: (Int, String) = (1, "A")
let rhs: Int = 1
let invalid = equal(lhs, rhs)
```

## In practice

**Choose it when:** Use packs for useful positional type relationships and mechanical repetition. Prefer runtime collections for data whose length is known only at runtime.

**Investigate by:** Check empty/scalar/heterogeneous calls, shape/type failures, call-site effects, and ordinary overload selection before publishing the API.

**Strong evidence:** Explain why preserving each position helps callers, then verify availability and ownership without assuming scheduling, unrolling, or universal inference behavior.

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
