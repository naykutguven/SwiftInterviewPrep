---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - generics
  - parameter-packs
---
## 0. Rubric snapshot

**Rubric expectation**

Know what parameter packs solve, where pack iteration helps, and why they matter for removing overload walls. B12 is listed as a Tier 2 “Strong Senior / Emerging Staff Depth” topic in the rubric.

**Caveats**

Parameter packs are powerful but advanced. They are mostly useful in libraries, macro-generated code, and generic infrastructure, not ordinary app feature code.

**You should be able to answer**

- What limitation of pre-Swift-5.9/6 code do parameter packs address?
- When is variadic generics actually the right abstraction instead of a simpler overload set?

**You should be able to do**

- Sketch a tuple or builder-style API that becomes materially cleaner with parameter packs.

---

## 1. Core mental model

Variadic generics let generic code abstract over **both type and arity**.

Before parameter packs, Swift could abstract over a single unknown type:

```swift
func identity<T>(_ value: T) -> T
```

And Swift could accept a variable number of values of one type:

```swift
func collect<T>(_ values: T...) -> [T]
```

But Swift could not naturally express “accept any number of arguments, each with its own static type, and preserve those types in the result.” The common workaround was an overload wall: write one overload for one argument, another for two, another for three, and so on. SE-0393 explicitly identifies these workarounds: `Any...`, tuple-wrapping, or fixed-arity overloads with artificial limits. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md "swift-evolution/proposals/0393-parameter-packs.md at main · swiftlang/swift-evolution · GitHub"))

Parameter packs solve that by introducing a compile-time “list” of types and values. A **type parameter pack** is declared with `each`, and a **pack expansion** is written with `repeat`. SE-0393 defines packs as a way to abstract over a variable number of type and value parameters with distinct types. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md "swift-evolution/proposals/0393-parameter-packs.md at main · swiftlang/swift-evolution · GitHub"))

The key idea:

```text
Generic T abstracts over one type.
Generic each T abstracts over zero or more types.
repeat each T expands the pack into a comma-separated list.
```

This is not the same as an array. An array has one element type and a runtime count. A parameter pack has statically known shape at the call site and can preserve different element types.

```swift
func printAll<each Value>(_ values: repeat each Value) {
    for value in repeat each values {
        print(value)
    }
}

printAll(1, "two", true)
```

Output:

```text
1
two
true
```

Here the pack shape is:

```text
each Value == { Int, String, Bool }
values     == { 1, "two", true }
```

---

## 2. Essential mechanics

### Type packs and value packs

A type parameter pack is introduced with `each`:

```swift
func describe<each Value>(_ values: repeat each Value) {
    repeat print(type(of: each values))
}

describe(1, "two", true)
```

Output:

```text
Int
String
Bool
```

`each Value` is not one type. It is a pack of types. At the call site above, it is conceptually:

```text
each Value = { Int, String, Bool }
```

`repeat each Value` expands that pack into a parameter list:

```text
(Int, String, Bool)
```

SE-0393’s terminology matters: a type pack is a list of scalar types; a value pack is a list of scalar values; pack expansions produce lists of types or values in places that accept such lists. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md "swift-evolution/proposals/0393-parameter-packs.md at main · swiftlang/swift-evolution · GitHub"))

### `repeat` expands a pattern

`repeat` does not mean “loop” in the ordinary runtime sense. It expands a pattern across every element of the pack.

```swift
struct Pair<First, Second> {
    let first: First
    let second: Second
}

func makePairs<each First, each Second>(
    firsts first: repeat each First,
    seconds second: repeat each Second
) -> (repeat Pair<each First, each Second>) {
    (repeat Pair(first: each first, second: each second))
}

let pairs = makePairs(firsts: 1, "id", seconds: true, 42.0)
print(pairs)
```

Conceptually:

```text
First  == { Int, String }
Second == { Bool, Double }

Return type:
(Pair<Int, Bool>, Pair<String, Double>)
```

The captured packs in the same expansion must have the same shape. You are not building a dynamic list. You are building a type-preserving expansion.

### Pack iteration in Swift 6

Swift 6 added pack iteration, which lets you iterate over a value pack using `for-in repeat`. SE-0408 was implemented in Swift 6.0 and was designed to remove the awkward “repeat expression only” limitation, especially when you need `break`, `continue`, or early `return`. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0408-pack-iteration.md "swift-evolution/proposals/0408-pack-iteration.md at main · swiftlang/swift-evolution · GitHub"))

```swift
func allEqual<each Element: Equatable>(
    _ lhs: (repeat each Element),
    _ rhs: (repeat each Element)
) -> Bool {
    for (left, right) in repeat (each lhs, each rhs) {
        guard left == right else { return false }
    }

    return true
}

print(allEqual((1, "a"), (1, "a")))
print(allEqual((1, "a"), (2, "a")))
```

Output:

```text
true
false
```

This is the important readability improvement. Without pack iteration, short-circuiting over a pack required unnatural workarounds such as throwing out of helper functions. SE-0408 specifically uses tuple equality as a motivating example. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0408-pack-iteration.md "swift-evolution/proposals/0408-pack-iteration.md at main · swiftlang/swift-evolution · GitHub"))

### Variadic generic types

SE-0398 extended the pack model from generic functions to generic types. It was implemented in Swift 5.9 and lets structs, classes, actors, and type aliases abstract over packs. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0398-variadic-types.md "swift-evolution/proposals/0398-variadic-types.md at main · swiftlang/swift-evolution · GitHub"))

```swift
struct Route<each Input> {
    let build: (repeat each Input) -> String

    func callAsFunction(_ input: repeat each Input) -> String {
        build(repeat each input)
    }
}

let userRoute = Route<Int, String> { id, section in
    "/users/\(id)/\(section)"
}

print(userRoute(42, "profile"))
```

Output:

```text
/users/42/profile
```

This is useful for typed builders, typed routing, parser combinators, dependency containers, tuple-like wrappers, test DSLs, and macro-generated APIs.

Current limitation: variadic generic enums are still not supported by the SE-0398 design.

```swift
enum E<each T> {
    case value(repeat each T)
}
```

Compiler error with Swift 6.2:

```text
error: enums cannot declare a type pack
```

SE-0398 also notes that variadic generic types require runtime support and are not backward-deployable to older runtimes; replacing a non-variadic generic type with a variadic generic type is not binary-compatible. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0398-variadic-types.md "swift-evolution/proposals/0398-variadic-types.md at main · swiftlang/swift-evolution · GitHub"))

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `T...` and `repeat each T` solve the same problem

`T...` is a normal variadic parameter. Every argument must have the same inferred `T`.

Bad:

```swift
func collect<T>(_ values: T...) -> [T] {
    values
}

let mixed = collect(1, "two")
```

Compiler error:

```text
error: conflicting arguments to generic parameter 'T' ('String' vs. 'Int')
```

Better:

```swift
func printAll<each Value>(_ values: repeat each Value) {
    for value in repeat each values {
        print(value)
    }
}

printAll(1, "two")
```

Output:

```text
1
two
```

Use `T...` for homogeneous values. Use packs when each argument may have a different static type and that type relationship matters.

### Trap 2: Treating packs like arrays

A pack is not a runtime collection.

Bad mental model:

```text
each T is like [Any.Type]
repeat each value is like [Any]
```

Better mental model:

```text
each T is a compile-time shape: { T0, T1, T2, ... }
repeat each value expands into arguments or tuple elements.
```

That means packs are great when the arity is statically known at the call site, but bad when the input is truly dynamic.

Bad fit:

```swift
let values = [1, 2, 3]

// You do not turn this runtime array into a type pack.
// This is just [Int], not { Int, Int, Int } as a generic shape.
```

Use arrays, collections, or `AnySequence` when runtime length is the core abstraction.

### Trap 3: Using packs because the syntax looks clever

Parameter packs make APIs harder to read if the problem is not truly arity-polymorphic.

Bad:

```swift
func log<each Value>(_ values: repeat each Value) {
    repeat print(each values)
}
```

This is usually unnecessary. A simpler API may be enough:

```swift
func log(_ message: String) {
    print(message)
}

func log(_ values: Any...) {
    values.forEach { print($0) }
}
```

Parameter packs are justified when preserving the individual static types matters. They are not a default replacement for arrays, tuples, result builders, or simple overloads.

### Trap 4: Forgetting API and ABI consequences

For internal app code, packs are mostly a readability and maintainability decision. For public libraries or binary SDKs, they are also an evolution decision.

SE-0398 says variadic generic types are not backward-deployable to older runtimes and replacing a non-variadic generic type with a variadic one is not binary-compatible. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0398-variadic-types.md "swift-evolution/proposals/0398-variadic-types.md at main · swiftlang/swift-evolution · GitHub"))

That means a staff-level decision is not only:

```text
Can we write this with parameter packs?
```

It is also:

```text
Can we expose this in a public API without creating migration, source-compatibility, ABI, or deployment problems?
```

---

## 4. Direct answers to rubric questions

### Q1. What limitation of pre-Swift-5.9/6 code do parameter packs address?

They address Swift’s old inability to express “a variable number of generic parameters and values while preserving each argument’s distinct static type.”

Before packs, you had three main choices:

```text
1. Use Any... and erase type information.
2. Accept one tuple and make callers manually package values.
3. Write fixed-arity overloads up to an arbitrary maximum.
```

SE-0393 calls out exactly this problem and uses tuple comparison overloads as an example: without variadic generics, the standard library needed overloads for tuple comparison up to fixed arities; with packs, that shape can be represented as one generic declaration. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md "swift-evolution/proposals/0393-parameter-packs.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> Parameter packs remove overload walls. Before Swift 5.9, if an API needed to preserve the static types of a variable number of arguments, you either erased to `Any`, accepted a tuple, or wrote overloads for one, two, three, four arguments and chose an arbitrary limit. Variadic generics let the API abstract over both the number of arguments and their individual types.

### Q2. When is variadic generics actually the right abstraction instead of a simpler overload set?

Use variadic generics when all of these are true:

```text
The API is naturally arity-polymorphic.
Each argument can have a different static type.
The result type or behavior depends on preserving those individual types.
The overload set would otherwise grow mechanically.
The API is infrastructure/library-level enough to justify the syntax cost.
```

A small overload set is still better when the API has genuinely different semantics per arity or only supports two or three cases for domain reasons.

Good fit:

```swift
struct Request<Response> {
    let response: Response
}

func perform<each Response>(
    _ requests: repeat Request<each Response>
) -> (repeat each Response) {
    (repeat (each requests).response)
}

let responses = perform(
    Request(response: 200),
    Request(response: "OK"),
    Request(response: true)
)

print(responses)
```

Output:

```text
(200, "OK", true)
```

Here the result statically preserves:

```text
(Int, String, Bool)
```

That is materially better than returning `[Any]`.

Bad fit:

```swift
func configure(title: String)
func configure(title: String, subtitle: String)
```

Do not replace this with a parameter pack. The two overloads are clearer because they model real API intent, not a mechanical generic pattern.

Interview version:

> Variadic generics are right when the API is fundamentally generic over arity and preserving each argument’s type is part of the value proposition. They are usually wrong for ordinary app code where an array, a result builder, or a couple of semantic overloads are clearer. I would reach for packs in libraries, typed builders, parser combinators, tuple operations, dependency containers, or macro-generated infrastructure.

---

## 5. Code probe

The rubric has no code probe for B12. Here are the minimal probe, counterexample, and production-oriented example.

### Minimal probe

Given:

```swift
func debugValues<each Value>(_ values: repeat each Value) {
    repeat print(type(of: each values))
}

debugValues(1, "two", true)
```

Output:

```text
Int
String
Bool
```

Why:

```text
each Value == { Int, String, Bool }

repeat print(type(of: each values))

expands conceptually into:

print(type(of: 1))
print(type(of: "two"))
print(type(of: true))
```

### Counterexample: normal variadics are homogeneous

Given:

```swift
func collect<T>(_ values: T...) -> [T] {
    values
}

let mixed = collect(1, "two")
```

Compiler error:

```text
error: conflicting arguments to generic parameter 'T' ('String' vs. 'Int')
```

Why:

```text
T... means:
- any number of values
- one shared element type T

repeat each T means:
- any number of values
- each position may have its own type
```

### Production-oriented example: typed request fan-out

Naive version:

```swift
struct Request<Response> {
    let response: Response
}

func perform(_ requests: Request<Any>...) -> [Any] {
    requests.map(\.response)
}
```

Problem:

```text
The result loses all static type information.
The caller must downcast.
The API accepts invalid combinations because everything becomes Any.
```

Improved version:

```swift
struct Request<Response> {
    let response: Response
}

func perform<each Response>(
    _ requests: repeat Request<each Response>
) -> (repeat each Response) {
    (repeat (each requests).response)
}

let responses = perform(
    Request(response: 200),
    Request(response: "OK"),
    Request(response: true)
)

print(responses)
```

Output:

```text
(200, "OK", true)
```

Why this fix is correct:

```text
Input shape:
Request<Int>, Request<String>, Request<Bool>

Pack:
each Response == { Int, String, Bool }

Output:
(Int, String, Bool)
```

No downcasting. No `[Any]`. No arbitrary overload limit.

Alternative fixes and tradeoffs:

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Fixed overloads|You only support 2–3 cases and each has semantic meaning|Grows mechanically if arity expands|
|`[Any]` / `Any...`|Logging, diagnostics, or truly dynamic mixed values|Loses type information|
|Tuple parameter|Caller already owns a tuple-shaped value|Awkward call sites and weaker API ergonomics|
|Parameter packs|Type-preserving arity-polymorphic infrastructure|Advanced syntax and public API complexity|
|Result builder|Building homogeneous or type-erased declarative structures|Builder transform can hide control flow and type errors|

---

## 6. Exercise

### Problem

Sketch a tuple or builder-style API that becomes materially cleaner with parameter packs.

### Bad / naive version

```swift
struct Parser<Input, Output> {
    let parse: (Input) -> Output
}

func zip<A, B>(
    _ a: Parser<String, A>,
    _ b: Parser<String, B>
) -> Parser<String, (A, B)> {
    Parser<String, (A, B)> { input in
        (a.parse(input), b.parse(input))
    }
}

func zip<A, B, C>(
    _ a: Parser<String, A>,
    _ b: Parser<String, B>,
    _ c: Parser<String, C>
) -> Parser<String, (A, B, C)> {
    Parser<String, (A, B, C)> { input in
        (a.parse(input), b.parse(input), c.parse(input))
    }
}

// Then zip4, zip5, zip6...
```

### What is wrong?

```text
The implementation is mechanically repeated.
The API has an arbitrary maximum arity.
Every new arity increases maintenance and test surface.
The type relationship is simple but cannot be expressed without packs.
```

### Improved version

```swift
struct Parser<Input, Output> {
    let parse: (Input) -> Output
}

func zip<Input, each Output>(
    _ parsers: repeat Parser<Input, each Output>
) -> Parser<Input, (repeat each Output)> {
    Parser<Input, (repeat each Output)> { input in
        (repeat (each parsers).parse(input))
    }
}

let intParser = Parser<String, Int> { _ in 42 }
let stringParser = Parser<String, String> { _ in "name" }
let boolParser = Parser<String, Bool> { _ in true }

let combined = zip(intParser, stringParser, boolParser)

print(combined.parse("ignored"))
```

Output:

```text
(42, "name", true)
```

### Why this is better

This API is materially cleaner because the type relationship is the point:

```text
Parser<Input, Int>
Parser<Input, String>
Parser<Input, Bool>

becomes:

Parser<Input, (Int, String, Bool)>
```

There is no type erasure, no arbitrary maximum arity, and no duplicated overload body. This is exactly the kind of infrastructure-level API where parameter packs earn their complexity.

---

## 7. Production guidance

Use this in production when:

```text
You are building generic infrastructure, not ordinary feature glue.
The API currently has a mechanical overload wall.
Each argument position has its own static type.
The return type depends on preserving the exact argument shape.
You are designing parser combinators, tuple operations, route builders, DI factories, request fan-out APIs, or macro-generated helpers.
```

Be careful when:

```text
The API is public or binary-distributed.
The minimum runtime matters.
The syntax would make diagnostics harder for consumers.
A small overload set is clearer.
A result builder already gives better call-site ergonomics.
You are mixing pack overloads with existing scalar overloads.
```

Avoid when:

```text
The input is a runtime collection.
The element type is homogeneous.
The return type does not preserve per-position type information.
The only benefit is avoiding two readable overloads.
The team will not be able to maintain the abstraction confidently.
```

Debugging checklist:

```text
Is this a runtime-length problem or a static-arity problem?
Are we preserving useful type information or just showing off syntax?
Would [T], [Any], a tuple, a result builder, or two overloads be clearer?
Are pack shapes inferred as expected?
Are all packs in a repeat expansion the same shape?
Is pack iteration needed for early return, break, or continue?
Is this public API safe from a source/ABI/deployment perspective?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Parameter packs let functions take many generic arguments.

This is incomplete. It misses the core distinction between homogeneous variadics and type-preserving arity polymorphism.

### Senior answer

> Parameter packs let Swift express APIs that are generic over both the number of arguments and their individual types. They remove overload walls and avoid `Any` when the result type should preserve the input shape. I would use them for library-style APIs, not typical app code.

### Staff-level answer

> Parameter packs are a tool for eliminating mechanical overload walls while preserving static type relationships across variable arity. The decision is not just syntactic: I would check whether the API is truly arity-polymorphic, whether preserving each position’s type matters, whether pack iteration improves correctness, and whether exposing packs publicly creates source, ABI, runtime, or diagnostic costs. For most app code, arrays, semantic overloads, or builders are clearer. For generic infrastructure, packs can remove entire families of brittle overloads.

Staff-level questions to ask:

```text
Is the overload wall mechanical or semantically meaningful?
Does the caller benefit from preserving the exact tuple/result shape?
Will the diagnostics be acceptable for library consumers?
Can this remain internal, or will it become public API surface?
Does this need to work across older Apple runtimes or binary SDK boundaries?
Would a result builder or collection-based design be more maintainable?
```

---

## 9. Interview-ready summary

Variadic generics let Swift generic code abstract over a variable number of types and values while preserving the static type of each position. They solve the old overload-wall problem where libraries had to write `zip2`, `zip3`, `zip4`, or fixed tuple-comparison overloads with arbitrary limits. The syntax is based on `each` for declaring/referencing a pack and `repeat` for expanding a pattern. Swift 6 pack iteration makes these APIs more practical because you can use normal control flow like early `return`. I would use parameter packs for library and infrastructure APIs where arity polymorphism and type preservation are central, but I would avoid them in ordinary app code where arrays, result builders, or a few explicit overloads are clearer.

---

## 10. Flashcards

Q: What problem do parameter packs solve?

A: They let Swift abstract over a variable number of generic types and values while preserving each argument’s distinct static type.

Q: How is `T...` different from `repeat each T`?

A: `T...` accepts many values of one shared type `T`; `repeat each T` expands a pack where each position may have its own type.

Q: What does `each T` mean?

A: It declares or references a type parameter pack: a compile-time list of zero or more types.

Q: What does `repeat` do?

A: It expands a pattern containing pack references into a comma-separated list of types or values.

Q: Why is a parameter pack not an array?

A: An array has one element type and runtime length; a pack has statically inferred shape and can preserve different types per position.

Q: What did Swift 6 pack iteration add?

A: It allows `for-in repeat` over value packs, making normal control flow like `return`, `break`, and `continue` possible.

Q: When are parameter packs a bad choice?

A: When the data is runtime-length, homogeneous, or the API would be clearer with a collection, result builder, or small overload set.

Q: What is the staff-level concern with public variadic generic types?

A: They affect API evolution, diagnostics, runtime support, and binary compatibility; SE-0398 notes that variadic generic types require runtime support and are not binary-compatible replacements for non-variadic generic types.

---

## 11. Related sections

- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B3 — Generics, constraints, and same-type relationships]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[B14 — Result builders and DSL construction]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[E7 — Resilience, ABI, and module stability]]
- [[F4 — Macros and compile-time code generation]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist, B12 and tiering.
- Swift Evolution SE-0393: Value and Type Parameter Packs. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md "swift-evolution/proposals/0393-parameter-packs.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Evolution SE-0398: Allow Generic Types to Abstract Over Packs. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0398-variadic-types.md "swift-evolution/proposals/0398-variadic-types.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Evolution SE-0408: Pack Iteration. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0408-pack-iteration.md "swift-evolution/proposals/0408-pack-iteration.md at main · swiftlang/swift-evolution · GitHub"))
- Apple WWDC23: “Generalize APIs with parameter packs.” ([developer.apple.com](https://developer.apple.com/videos/play/wwdc2023/10168/ "Generalize APIs with parameter packs - WWDC23 - Videos - Apple Developer"))