---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - existentials
  - generics
---
## 0. Rubric snapshot

**Rubric expectation**

Know that abstraction choices affect specialization, boxing, dynamic dispatch, and allocations. The rubric frames C9 around predicting how generic and existential designs behave in hot paths, especially around specialization and allocation costs.

**Caveats**

Performance intuition should be evidence-driven, but senior engineers should still be able to predict likely hotspots before measuring.

**You should be able to answer**

- When will a generic API usually optimize better than an existential-based one?
- Why can storing protocol values in arrays allocate or inhibit specialization?

**You should be able to do**

- Compare two designs for a hot path: one generic, one existential; predict the likely optimization differences.

---

## 1. Core mental model

Swift gives you multiple ways to abstract over behavior: concrete types, generics, opaque types, and existentials. They can look similar at the API level, but they give the compiler very different information.

A **generic** function keeps the concrete type as a type parameter:

```swift
func render<R: Renderer>(_ renderer: R)
```

The source code says only `R: Renderer`, but each call site still has a concrete `R`. In optimized builds, Swift can often specialize the generic function for that concrete type, inline calls, remove witness-table dispatch, and optimize layout-specific operations. Swift’s optimizer performs important transformations such as specialization, inlining, and devirtualization in optimized builds; whole-module optimization historically mattered because file/module boundaries can otherwise limit specialization. ([Swift Forums, "Enabling Whole Module Optimizations by Default for Release Builds"](https://forums.swift.org/t/enabling-whole-module-optimizations-by-default-for-release-builds/1764))

An **existential** stores a value whose exact concrete type is erased:

```swift
func render(_ renderer: any Renderer)
```

Now the function receives a runtime box/container that says: “there is some value in here, and here is how to call `Renderer` requirements on it.” Swift Evolution SE-0335 explicitly calls out that existential types have performance implications: they can require dynamic memory unless the value fits in the inline buffer, and they involve pointer indirection and dynamic method dispatch. ([GitHub, "Introduce Existential any"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md))

An **opaque type** using `some` sits closer to generics than existentials. It hides the concrete type from API clients, but the implementation still commits to one concrete underlying type. The Swift book describes opaque types as preserving a specific hidden type, while boxed protocol types can hold any conforming type. ([Swift.org, "Opaque and Boxed Protocol Types"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/))

The key idea:

```text
Concrete/generic/some preserve type identity.
any erases type identity into a runtime container.
Preserved type identity gives the optimizer more room.
Erased type identity gives API flexibility but usually less optimization potential.
```

Swift does **not** guarantee that generic code is always faster. It guarantees type relationships. The optimizer may then exploit those relationships. Likewise, Swift does **not** guarantee that every existential causes a heap allocation. Small values may fit inline, class references are already references, and the optimizer can sometimes eliminate abstraction overhead. But as a default mental model for hot paths: generic/concrete APIs are easier to optimize; existential APIs are more flexible but more runtime-shaped.

---

## 2. Essential mechanics

### 2.1 Generics preserve concrete type information

A generic function is compiled against constraints, but each use has a real concrete type.

```swift
protocol Scorer {
    func score(_ value: Int) -> Int
}

struct MultiplierScorer: Scorer {
    let factor: Int

    func score(_ value: Int) -> Int {
        value * factor
    }
}

func total<S: Scorer>(
    using scorer: S,
    values: [Int]
) -> Int {
    var result = 0

    for value in values {
        result += scorer.score(value)
    }

    return result
}
```

At the call site:

```swift
let scorer = MultiplierScorer(factor: 2)
let result = total(using: scorer, values: Array(0..<1_000))
```

The compiler can see that `S == MultiplierScorer` in optimized contexts. That can enable:

```text
generic function specialized for MultiplierScorer
→ scorer.score may be devirtualized
→ score body may be inlined
→ loop may become easier to optimize
```

This is why generic APIs are often a good fit for performance-sensitive reusable algorithms.

### 2.2 Existentials erase the concrete type

Now compare:

```swift
func total(
    using scorer: any Scorer,
    values: [Int]
) -> Int {
    var result = 0

    for value in values {
        result += scorer.score(value)
    }

    return result
}
```

The function no longer has a type parameter `S`. It has an existential value. Conceptually:

```text
any Scorer =
  value storage
  concrete type metadata
  witness table for Scorer
```

Each call to `score` is made through the existential’s witness table unless the optimizer can prove and eliminate the abstraction. SE-0335 explicitly says existential costs include dynamic memory in some cases, heap allocation/reference counting, pointer indirection, and dynamic method dispatch. ([GitHub, "Introduce Existential any"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md))

This does not mean “never use `any`.” It means `any` should be a deliberate type-erasure boundary.

### 2.3 Arrays of existentials are heterogeneous containers

A generic array is homogeneous:

```swift
func totalAll<S: Scorer>(
    _ scorers: [S],
    values: [Int]
) -> Int {
    var result = 0

    for scorer in scorers {
        for value in values {
            result += scorer.score(value)
        }
    }

    return result
}
```

Here every element is the same concrete `S`.

An existential array can be heterogeneous:

```swift
struct OffsetScorer: Scorer {
    let offset: Int

    func score(_ value: Int) -> Int {
        value + offset
    }
}

let scorers: [any Scorer] = [
    MultiplierScorer(factor: 2),
    OffsetScorer(offset: 10)
]
```

That flexibility has a cost. Each element is stored as an existential container. The optimizer cannot treat the array as `[MultiplierScorer]`, because it is not one. Every element may have a different concrete type.

```text
[MultiplierScorer]       → contiguous concrete values
[any Scorer]             → existential containers; each element may be a different type
```

This inhibits specialization because there is no single `S` to specialize for.

### 2.4 `some` hides type identity from clients but preserves it for the implementation

```swift
func makeScorer() -> some Scorer {
    MultiplierScorer(factor: 2)
}
```

The caller does not know the concrete return type, but the function still returns one specific underlying type. Swift’s opaque result type model requires the implementation to provide a consistent concrete type for that declaration. The proposal for opaque result types also notes that when the function body is visible to the optimizer, the compiler can access the concrete type and eliminate indirection costs. ([GitHub, "Opaque Result Types"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md))

Contrast:

```swift
func makeScorer(_ flag: Bool) -> any Scorer {
    if flag {
        MultiplierScorer(factor: 2)
    } else {
        OffsetScorer(offset: 10)
    }
}
```

This returns different possible concrete types, so existential type erasure is the right abstraction.

---

## 3. Common traps and misconceptions

### Trap 1: “Protocols are slow”

Wrong. Protocols are not automatically slow. The cost depends on how you use them.

Bad mental model:

```text
protocol == slow
generic == fast
```

Better mental model:

```text
generic constraint over protocol → preserves concrete type
any Protocol                    → erases concrete type
protocol extension method       → may be statically dispatched if not a requirement
```

Bad:

```swift
func process(_ items: [any Scorer], values: [Int]) -> Int {
    items.reduce(0) { partial, scorer in
        partial + values.reduce(0) { $0 + scorer.score($1) }
    }
}
```

Better for a homogeneous hot path:

```swift
func process<S: Scorer>(_ items: [S], values: [Int]) -> Int {
    items.reduce(0) { partial, scorer in
        partial + values.reduce(0) { $0 + scorer.score($1) }
    }
}
```

The first version is more flexible. The second version is more optimizable.

### Trap 2: Using `any` too early

A common architectural mistake is erasing type information at the boundary of every layer:

```swift
protocol Parser {
    associatedtype Output

    func parse(_ data: Data) throws -> Output
}

func load(_ parser: any Parser) {
    // Now what is Output?
}
```

This loses type relationships that the rest of the pipeline may need.

Better:

```swift
func load<P: Parser>(
    _ parser: P,
    data: Data
) throws -> P.Output {
    try parser.parse(data)
}
```

Erase only when you truly need heterogeneous storage, dynamic plugin boundaries, or a stable runtime boundary.

### Trap 3: Assuming existential allocation always happens

Existentials **can** allocate, but not every existential value allocates. Small values may fit in the existential inline buffer. Class-constrained existentials often store an object reference. The correct statement is:

```text
Existentials introduce a representation that may require boxing/allocation
and usually introduces indirection/dynamic dispatch.
```

Not:

```text
Every `any P` heap-allocates.
```

### Trap 4: Measuring Debug builds and drawing Release conclusions

Swift Debug builds intentionally preserve debuggability and perform minimal optimization. Swift’s optimization settings distinguish `-Onone`, `-Osize`, and `-O`; optimized builds can drastically change emitted code. ([GitHub, "Optimization Tips"](https://github.com/swiftlang/swift/blob/main/docs/optimizationtips.rst))

For C9, this matters because specialization and inlining are optimizer behavior. A generic abstraction that looks expensive in Debug may mostly disappear in Release.

### Trap 5: Hiding public-library performance problems behind `@inlinable`

`@inlinable` can expose implementation bodies to clients and allow more optimization across module boundaries, but it also leaks implementation detail into the public optimization contract. Opaque result type evolution can also be constrained when combined with `@inlinable`, because the underlying concrete type can become exposed through inlining. ([GitHub, "Opaque Result Types"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md))

Do not treat `@inlinable` as a free speed button.

---

## 4. Direct answers to rubric questions

### Q1. When will a generic API usually optimize better than an existential-based one?

A generic API usually optimizes better when the operation is performance-sensitive, called many times, and the concrete type is known at compile time or visible to the optimizer.

Generics preserve type relationships:

```swift
func run<S: Strategy>(_ strategy: S)
```

This allows the optimizer to specialize `run` for `ConcreteStrategy`, potentially inline protocol calls, remove witness-table lookups, and optimize concrete layout.

Existentials erase type relationships:

```swift
func run(_ strategy: any Strategy)
```

This allows runtime flexibility, but the function cannot assume one concrete type. Calls usually go through witness-table dispatch, and values may require existential storage or boxing.

Interview version:

> A generic API usually optimizes better when the compiler can see the concrete type at the call site. The generic function preserves `T`, so optimized builds can specialize the function, inline calls, and devirtualize protocol requirements. An existential `any P` intentionally erases the concrete type, so it usually introduces an existential container, witness-table dispatch, possible boxing, and less opportunity for specialization. I would prefer generics for homogeneous hot paths and use existentials at dynamic or heterogeneous boundaries.

### Q2. Why can storing protocol values in arrays allocate or inhibit specialization?

Because `[any P]` is an array of existential containers, not an array of one concrete conforming type.

```swift
let items: [any Scorer] = [
    MultiplierScorer(factor: 2),
    OffsetScorer(offset: 10)
]
```

Each element may have a different concrete type. The compiler cannot specialize the loop for a single `S`. The array storage contains existential representations; large value types may need heap boxing, and calls to protocol requirements are made through runtime witness tables unless optimized away.

By contrast:

```swift
let items: [MultiplierScorer] = [
    MultiplierScorer(factor: 2),
    MultiplierScorer(factor: 3)
]
```

This has one concrete element type. The optimizer can reason about layout and calls much more directly.

Interview version:

> An array like `[any P]` is heterogeneous from the compiler’s perspective. Each element is an existential container with erased concrete type information. That can require boxing when a value does not fit inline, and it usually means calls go through witness tables. It also prevents the optimizer from specializing the loop for one concrete element type. `[T] where T: P` is homogeneous and preserves the concrete type, so it is usually a better shape for hot paths.

---

## 5. Code probe / focused examples

The rubric does **not** provide a C9 code probe. For this section, use the following examples instead.

### Minimal example: generic hot path

```swift
protocol PixelTransform {
    func transform(_ value: UInt8) -> UInt8
}

struct Invert: PixelTransform {
    func transform(_ value: UInt8) -> UInt8 {
        255 - value
    }
}

func apply<T: PixelTransform>(
    _ transform: T,
    to pixels: inout [UInt8]
) {
    for index in pixels.indices {
        pixels[index] = transform.transform(pixels[index])
    }
}

var pixels: [UInt8] = [0, 127, 255]
apply(Invert(), to: &pixels)
print(pixels)
```

### What happens?

```text
[255, 128, 0]
```

### Why?

The source-level abstraction is generic, but the call uses `T == Invert`.

```text
apply<T: PixelTransform>
called as apply<Invert>
→ optimizer may specialize for Invert
→ transform call may be inlined
→ abstraction cost may disappear in Release
```

### Counterexample: existential hot path

```swift
func apply(
    _ transform: any PixelTransform,
    to pixels: inout [UInt8]
) {
    for index in pixels.indices {
        pixels[index] = transform.transform(pixels[index])
    }
}

var pixels: [UInt8] = [0, 127, 255]
let transform: any PixelTransform = Invert()

apply(transform, to: &pixels)
print(pixels)
```

### What happens?

```text
[255, 128, 0]
```

### Why?

Semantically, the result is the same. Performance-wise, the function receives an existential.

```text
any PixelTransform
→ erased concrete type
→ call through witness table unless optimized away
→ less direct specialization opportunity
```

### Production example: mixed strategies

Existential storage is justified when the collection is genuinely heterogeneous:

```swift
struct Clamp: PixelTransform {
    func transform(_ value: UInt8) -> UInt8 {
        min(value, 200)
    }
}

let transforms: [any PixelTransform] = [
    Invert(),
    Clamp()
]

func applyPipeline(
    _ transforms: [any PixelTransform],
    to pixels: inout [UInt8]
) {
    for transform in transforms {
        for index in pixels.indices {
            pixels[index] = transform.transform(pixels[index])
        }
    }
}
```

This design is flexible. It allows runtime-composed pipelines. But it is not the most optimizable shape for a tight image-processing loop.

A more performance-oriented design might use an enum when the cases are known:

```swift
enum BuiltInPixelTransform {
    case invert
    case clamp(max: UInt8)

    func transform(_ value: UInt8) -> UInt8 {
        switch self {
        case .invert:
            255 - value
        case .clamp(let max):
            min(value, max)
        }
    }
}
```

This avoids existential storage while still allowing a heterogeneous pipeline of known operations.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Concrete type|Maximum performance, single implementation|Least flexible API|
|Generic `T: Protocol`|Homogeneous hot path, reusable algorithm|Can increase code size through specialization|
|`some Protocol`|Hide concrete return type while preserving one underlying type|Cannot return different concrete types from the same declaration|
|`any Protocol`|Heterogeneous storage, plugin systems, dependency boundaries|Possible boxing, dynamic dispatch, less specialization|
|Enum|Closed set of heterogeneous cases|Less extensible across modules|

---

## 6. Exercise

### Problem

Compare two designs for a hot path: one generic, one existential; predict the likely optimization differences.

### Scenario

You are building a search ranking pipeline. A ranking function runs thousands of times per keystroke over local results.

### Bad / naive version

```swift
protocol Ranker {
    func score(_ item: SearchItem, query: String) -> Double
}

struct SearchItem {
    let title: String
    let popularity: Double
}

struct TitleRanker: Ranker {
    func score(_ item: SearchItem, query: String) -> Double {
        item.title.localizedCaseInsensitiveContains(query) ? 10 : 0
    }
}

struct PopularityRanker: Ranker {
    func score(_ item: SearchItem, query: String) -> Double {
        item.popularity
    }
}

func rank(
    items: [SearchItem],
    query: String,
    rankers: [any Ranker]
) -> [(SearchItem, Double)] {
    items.map { item in
        let score = rankers.reduce(0) { partial, ranker in
            partial + ranker.score(item, query: query)
        }

        return (item, score)
    }
}
```

### What is wrong?

```text
The design erases every ranker into `any Ranker`.
The inner loop calls protocol requirements through existential dispatch.
The array is heterogeneous, so the compiler cannot specialize the loop for one ranker type.
Large value-type rankers may require boxing.
This may be acceptable at a feature boundary, but it is suspicious inside a keystroke-level hot path.
```

This design is not automatically bad. It is bad if profiling shows ranking is hot and the pipeline is frequently executed.

### Improved version 1: generic homogeneous ranker

```swift
func rank<R: Ranker>(
    items: [SearchItem],
    query: String,
    ranker: R
) -> [(SearchItem, Double)] {
    items.map { item in
        (item, ranker.score(item, query: query))
    }
}
```

This is optimal when there is one ranker type per call.

```text
R is preserved
→ function can specialize for TitleRanker
→ score can potentially inline
→ no existential array
```

### Improved version 2: concrete composed ranker

```swift
struct CombinedRanker: Ranker {
    var title = TitleRanker()
    var popularity = PopularityRanker()

    func score(_ item: SearchItem, query: String) -> Double {
        title.score(item, query: query)
        + popularity.score(item, query: query)
    }
}

func rank<R: Ranker>(
    items: [SearchItem],
    query: String,
    ranker: R
) -> [(SearchItem, Double)] {
    items.map { item in
        (item, ranker.score(item, query: query))
    }
}
```

This preserves static structure while still composing behavior.

### Improved version 3: enum pipeline for a closed set

```swift
enum RankingRule {
    case title
    case popularity(weight: Double)

    func score(_ item: SearchItem, query: String) -> Double {
        switch self {
        case .title:
            item.title.localizedCaseInsensitiveContains(query) ? 10 : 0

        case .popularity(let weight):
            item.popularity * weight
        }
    }
}

func rank(
    items: [SearchItem],
    query: String,
    rules: [RankingRule]
) -> [(SearchItem, Double)] {
    items.map { item in
        let score = rules.reduce(0) { partial, rule in
            partial + rule.score(item, query: query)
        }

        return (item, score)
    }
}
```

This is useful when the set of ranking rules is known and controlled by your module.

### Why this is better

```text
Generic design:
- Best when the pipeline is homogeneous or statically composed.
- Enables specialization and inlining.
- Keeps concrete layout visible.

Enum design:
- Best when there is a closed set of heterogeneous behaviors.
- Avoids existential boxing.
- Keeps dynamic choice explicit and switch-based.

Existential design:
- Best when rankers are loaded dynamically, owned by separate modules, or configured at runtime.
- Most flexible.
- Least predictable for hot-path optimization.
```

Staff-level judgment: start with the cleanest design, profile in Release with realistic data, then move existential boundaries outward if the profile shows they are hot.

---

## 7. Production guidance

Use generics in production when:

```text
The algorithm is reusable but operates on one concrete type per call.
The code is in a hot loop.
You need same-type relationships.
You want the optimizer to specialize and inline.
You are building low-level infrastructure, parsers, codecs, ranking, layout, diffing, or rendering code.
```

Use existentials in production when:

```text
You need heterogeneous storage.
You need runtime plugin-style behavior.
You need dependency injection at a module boundary.
You are storing multiple unrelated implementations behind one interface.
The performance cost is not on a measured hot path.
```

Use `some` in production when:

```text
You want to hide a concrete return type but still preserve a single underlying type.
You want API resilience without fully erasing type identity.
You do not need heterogeneous return values from the same declaration.
```

Be careful when:

```text
You put `[any P]` inside tight loops.
You return `any Sequence`, `any Collection`, or `any View`-like abstractions from low-level APIs.
You erase types before associated type relationships have done their job.
You benchmark Debug builds.
You add `@inlinable` to fix performance without accepting the public API cost.
```

Avoid when:

```text
You use `any` merely because a protocol exists.
You use type erasure to hide poor modeling.
You replace a small closed enum with runtime polymorphism.
You design all SDK APIs around existentials before understanding call frequency and client constraints.
```

Debugging checklist:

```text
Is this code actually hot in Instruments?
Am I measuring a Release build with realistic optimization settings?
Is the existential inside an inner loop?
Is the collection homogeneous but typed as `[any P]`?
Can this be generic instead?
Can this be `some P` instead of `any P`?
Is the protocol class-bound, value-based, or associated-type-heavy?
Are allocations visible in Allocations or Time Profiler?
Would an enum better model a closed set of cases?
Is cross-module optimization or `@inlinable` relevant, and is the API cost acceptable?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Generics are faster than protocols, so use generics for performance.

This is too vague and partially wrong.

### Senior answer

> Generics preserve the concrete type and can often be specialized in optimized builds. Existentials erase the concrete type into `any P`, which can introduce boxing, witness-table dispatch, pointer indirection, and fewer optimization opportunities. I would use generics for homogeneous hot paths and existentials for heterogeneous storage or dynamic boundaries.

### Staff-level answer

> I separate semantic API needs from performance representation. If the operation is a hot homogeneous algorithm, I prefer generics or concrete composition because they preserve type identity and allow specialization. If the system needs runtime heterogeneity, plugin boundaries, or dependency inversion, I use `any` deliberately and try to keep that existential boundary outside inner loops. For a public library, I also consider resilience, `@inlinable`, code size from specialization, and whether `some` gives enough abstraction without full type erasure. Then I validate with Release profiling rather than relying on folklore.

Staff-level questions to ask:

```text
Is heterogeneity required, or did we erase type information too early?
Is this API public, and what type relationships must clients preserve?
Is this hot path dominated by dispatch, allocation, ARC, string work, I/O, or something else?
Would specialization increase code size enough to matter?
Can `some` hide implementation detail without existential overhead?
Should the dynamic boundary be moved outward?
Would a closed enum be clearer and faster than open-ended runtime polymorphism?
Are we measuring Release with realistic data?
Is cross-module optimization relevant?
Would `@inlinable` leak implementation detail we may need to change later?
```

---

## 9. Interview-ready summary

Swift abstraction choices affect the optimizer. A generic API like `func f<T: P>(_ value: T)` preserves the concrete type at each call site, so optimized builds can often specialize, inline, and devirtualize. An existential like `any P` erases the concrete type into a runtime container, which can involve boxing, witness-table dispatch, pointer indirection, ARC, and fewer specialization opportunities. That does not make existentials bad; they are the right tool for heterogeneous storage and dynamic boundaries. For hot homogeneous paths, I prefer concrete, generic, or `some`-based designs, then validate with Release profiling.

---

## 10. Flashcards

Q: What is the main performance difference between `T: P` and `any P`?

A: `T: P` preserves the concrete type as a generic parameter; `any P` erases the concrete type into an existential container.

Q: Why can generics optimize well in Swift?

A: Optimized builds can often specialize generic functions for concrete types, then inline and devirtualize calls.

Q: Does every existential allocate?

A: No. Small values may fit inline, and class references are already references. But existentials can require boxing/allocation and usually involve indirection.

Q: Why can `[any P]` be slower than `[ConcreteP]`?

A: `[any P]` stores existential containers and may contain different concrete types, so the compiler cannot specialize the loop for one element type.

Q: When is `any P` the right choice?

A: Heterogeneous storage, dynamic runtime boundaries, plugin systems, dependency injection boundaries, or cases where type erasure is semantically required.

Q: When is `some P` preferable to `any P`?

A: When the implementation returns one concrete type but wants to hide that type from clients.

Q: What is the dangerous performance misconception around protocols?

A: Thinking “protocols are slow.” The real distinction is generic protocol constraints versus existential protocol values.

Q: Why should Debug benchmarks not drive C9 conclusions?

A: Debug builds perform minimal optimization, so specialization and inlining behavior may not reflect Release performance.

---

## 11. Related sections

- [[B2 — Existentials (any) vs opaque types (some)]]
- [[B3 — Generics, constraints, and same-type relationships]]
- [[B4 — Dispatch model: static, class virtual, witness-table, and protocol extension dispatch]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[F3 — Performance investigation habits]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — C9 section.
- GitHub. "Introduce Existential any." Swift Evolution SE-0335. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md
- Swift.org. "Opaque and Boxed Protocol Types." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/
- GitHub. "Opaque Result Types." Swift Evolution SE-0244. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md
- GitHub. "Optimization Tips." swiftlang/swift. https://github.com/swiftlang/swift/blob/main/docs/optimizationtips.rst
- GitHub. "High Level SIL Optimizations." swiftlang/swift. https://github.com/swiftlang/swift/blob/main/docs/highlevelsiloptimizations.rst
