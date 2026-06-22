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
## 0. Rubric snapshot

**Rubric expectation**

Understand representation, flexibility, and performance tradeoffs between `any P`, `some P`, and generic parameters. The rubric specifically tests the difference between `func f() -> some P` and `func f() -> any P`, why replacing generics with existentials can hurt ergonomics and performance, and how to refactor an API returning `any Sequence` into a more type-preserving design.

**Caveats**

Existentials box and erase type relationships. Opaque types preserve a concrete hidden type, but that hidden concrete type must stay consistent.

**You should be able to answer**

- What is the difference between `func f() -> some P` and `func f() -> any P`?
- Why might replacing generics with existentials hurt both ergonomics and performance?
- When should you use `any`, `some`, or a generic parameter?

**You should be able to do**

- Refactor an API returning `any Sequence` into a more type-preserving design.
- Explain why the B2 code probe fails to compile.

---

## 1. Core mental model

Swift has three major ways to abstract over protocol-conforming types:

```swift
func generic<T: P>(_ value: T)          // caller chooses T; compiler knows T
func opaque() -> some P                 // implementation chooses one hidden concrete type
func existential() -> any P             // runtime box can hold any conforming type
```

`some P` is **not** “return any type conforming to `P`.” It means: “this declaration returns one specific concrete type that conforms to `P`, but clients are not allowed to name that type.” The Swift Book describes this as preserving type identity: an opaque type refers to one specific type, while a boxed protocol type can refer to any conforming type. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/?utm_source=chatgpt.com "Opaque and Boxed Protocol Types - Documentation"))

`any P` is an **existential type**. It is a runtime container whose concrete stored value can be `Remote`, `Local`, `MockImageSource`, or anything else conforming to `P`. Swift introduced explicit `any` syntax to make existential use visible, because existentials erase type information and have semantic/performance consequences. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/existential-any/?utm_source=chatgpt.com "Existential any (ExistentialAny) | Documentation"))

Generics and opaque types preserve more static information than existentials. With a generic `T: P`, the compiler knows the concrete `T` at the call site and can preserve relationships like “both parameters have the same type.” With `some P`, the compiler still knows the underlying type, even if clients do not. With `any P`, the compiler only knows “there is a value inside this existential box that conforms to `P`.”

The key idea:

```text
generic T: P     = caller chooses concrete type; type relationships preserved
some P           = implementation chooses one hidden concrete type; identity preserved
any P            = runtime box can hold different conforming types; identity erased
```

Swift guarantees that an opaque return type has a single underlying concrete type for that declaration. It does **not** guarantee that `some P` can dynamically switch between multiple conforming concrete types. A function returning an opaque type must return values sharing a single underlying type. ([docs.swift.org](https://docs.swift.org/swift-book/ReferenceManual/Types.html?utm_source=chatgpt.com "Types | Documentation"))

---

## 2. Essential mechanics

### Mechanic 1: `some P` return type must have one underlying concrete type

Good:

```swift
protocol ImageSource {}

struct Remote: ImageSource {}

func makeSource() -> some ImageSource {
    Remote()
}
```

The caller sees only `some ImageSource`, but the compiler knows the hidden concrete type is `Remote`.

Bad:

```swift
protocol ImageSource {}

struct Remote: ImageSource {}
struct Local: ImageSource {}

func makeSource(_ local: Bool) -> some ImageSource {
    if local {
        Local()
    } else {
        Remote()
    }
}
```

This fails because the function tries to return two different underlying concrete types: `Local` and `Remote`.

### Mechanic 2: `any P` allows runtime heterogeneity

```swift
protocol ImageSource {}

struct Remote: ImageSource {}
struct Local: ImageSource {}

func makeSource(_ local: Bool) -> any ImageSource {
    if local {
        Local()
    } else {
        Remote()
    }
}
```

This compiles because the function returns an existential container. At runtime, that container can hold either `Local` or `Remote`.

But the caller has lost the concrete type relationship:

```swift
let source: any ImageSource = makeSource(Bool.random())

// The caller knows:
// - source conforms to ImageSource
//
// The caller does not statically know:
// - whether this is Local or Remote
// - whether two returned values have the same concrete type
// - concrete-type-specific APIs not exposed by ImageSource
```

### Mechanic 3: generics preserve caller-chosen type relationships

```swift
protocol ImageSource {
    func load() async throws -> Data
}

func loadTwice<S: ImageSource>(_ source: S) async throws -> (Data, Data) {
    let first = try await source.load()
    let second = try await source.load()
    return (first, second)
}
```

Here `S` is one concrete type for the whole call. The compiler can type-check and optimize based on that relationship.

Compare:

```swift
func loadTwice(_ source: any ImageSource) async throws -> (Data, Data) {
    let first = try await source.load()
    let second = try await source.load()
    return (first, second)
}
```

This may be perfectly fine at an architectural boundary, but the concrete type is erased. In hot paths, generic APIs often optimize better because the compiler can specialize them. Swift compiler documentation explicitly points out that when the compiler knows the concrete type, it has more optimization opportunity than with existential abstraction. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/existential-type/?utm_source=chatgpt.com "Existential Types and Performance (ExistentialType)"))

### Mechanic 4: `some` in parameter position is generic shorthand

These are similar for many practical purposes:

```swift
func render(_ source: some ImageSource) {
    // ...
}
```

```swift
func render<S: ImageSource>(_ source: S) {
    // ...
}
```

But full generic syntax is still needed when you need to name relationships:

```swift
func compare<S: ImageSource>(_ lhs: S, _ rhs: S) {
    // lhs and rhs must have the same concrete type S
}
```

You cannot express that same relationship with two independent `some` parameters:

```swift
func compare(_ lhs: some ImageSource, _ rhs: some ImageSource) {
    // lhs and rhs may be different concrete types
}
```

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `some P` means “any conforming type”

Bad:

```swift
func makeSource(_ local: Bool) -> some ImageSource {
    if local {
        Local()
    } else {
        Remote()
    }
}
```

Better, when runtime choice is genuinely required:

```swift
func makeSource(_ local: Bool) -> any ImageSource {
    if local {
        Local()
    } else {
        Remote()
    }
}
```

Better, when you want static opacity but one concrete wrapper:

```swift
enum Source: ImageSource {
    case local(Local)
    case remote(Remote)
}

func makeSource(_ local: Bool) -> some ImageSource {
    if local {
        Source.local(Local())
    } else {
        Source.remote(Remote())
    }
}
```

The second version returns one concrete type: `Source`.

### Trap 2: Replacing generics with `any` because the syntax is shorter

Bad:

```swift
func process(_ items: any Sequence<Int>) {
    for item in items {
        print(item)
    }
}
```

This is not automatically wrong, but it erases the concrete sequence type. If the caller’s type matters for optimization or API relationships, prefer a generic:

```swift
func process<S: Sequence<Int>>(_ items: S) {
    for item in items {
        print(item)
    }
}
```

### Trap 3: Using `any View` or `AnyView` casually in SwiftUI

In SwiftUI, `some View` is usually the right default for `body` and view-producing APIs because SwiftUI relies heavily on static structural type information.

Bad habit:

```swift
func makeRow() -> AnyView {
    AnyView(Text("Row"))
}
```

Better default:

```swift
func makeRow() -> some View {
    Text("Row")
}
```

Use type erasure in SwiftUI only when the heterogeneity is real and the API boundary justifies it.

### Trap 4: Forgetting that `any P` is itself a concrete type

This is subtle but important:

```swift
let source: any ImageSource = Remote()
```

The type of `source` is not `Remote`. The type of `source` is `any ImageSource`.

That existential container currently stores a `Remote`, but the variable’s static type is the box.

---

## 4. Direct answers to rubric questions

### Q1. What is the difference between `func f() -> some P` and `func f() -> any P`?

`func f() -> some P` returns one specific concrete type chosen by the function implementation, hidden from callers. All return paths must produce the same underlying concrete type.

`func f() -> any P` returns an existential container that can hold any conforming type. Different calls or branches can return different concrete types.

Interview version:

> `some P` is opaque but still statically one concrete type. The caller cannot name the type, but the compiler preserves its identity. `any P` is an existential box: it erases the concrete type and allows runtime heterogeneity. I use `some` when I want to hide implementation while preserving static type information, and `any` when I genuinely need to store or return different conforming types behind one value-level abstraction.

### Q2. Why might replacing generics with existentials hurt both ergonomics and performance?

Replacing generics with existentials erases concrete type information. That can hurt ergonomics because you lose same-type relationships and access to concrete-type-specific or associated-type-dependent APIs. It can hurt performance because the compiler has fewer specialization opportunities, calls may go through witness tables, and existential storage can involve boxing.

Example of lost type relationship:

```swift
func areSameType<S: ImageSource>(_ lhs: S, _ rhs: S) -> Bool {
    true
}
```

This requires both arguments to have the same concrete type.

With existentials:

```swift
func areSameType(_ lhs: any ImageSource, _ rhs: any ImageSource) -> Bool {
    // The static relationship is gone.
    true
}
```

Interview version:

> Generics preserve type relationships. If a function says `<T: P>`, the compiler knows every use of `T` in that call is the same concrete type. With `any P`, that relationship is erased into a runtime container. That can make APIs less expressive, block certain associated-type or `Self`-based operations, and reduce optimization opportunities. I would not replace generics with existentials just to make a signature shorter.

### Q3. When should you use `any`, `some`, or a generic parameter?

Use a **generic parameter** when the caller supplies the type and relationships matter:

```swift
func cache<S: ImageSource>(_ source: S, as key: String) {
    // caller chooses S
}
```

Use **`some P`** for return values when the implementation wants to hide the concrete type but still preserve static type identity:

```swift
func makeSource() -> some ImageSource {
    Remote()
}
```

Use **`any P`** when you need value-level heterogeneity or runtime substitution:

```swift
let sources: [any ImageSource] = [
    Remote(),
    Local()
]
```

Interview version:

> My default is concrete types inside a module, generics when the caller’s type should be preserved, `some` when returning a hidden but stable implementation type, and `any` when the program genuinely needs heterogeneous values or runtime polymorphism. Existentials are a good tool, but they are not a harmless replacement for generics.

---

## 5. Code probe

Given:

```swift
protocol ImageSource {}
struct Remote: ImageSource {}
struct Local: ImageSource {}

func makeSource(_ local: Bool) -> some ImageSource {
    if local {
        return Local()
    } else {
        return Remote()
    }
}
```

### What happens?

Compiler error, verified with Swift 6.2.1:

```text
/tmp/B2.swift:4:6: error: function declares an opaque return type 'some ImageSource', but the return statements in its body do not have matching underlying types [#OpaqueTypeInference]
 2 | struct Remote: ImageSource {}
 3 | struct Local: ImageSource {}
 4 | func makeSource(_ local: Bool) -> some ImageSource {
   |      `- error: function declares an opaque return type 'some ImageSource', but the return statements in its body do not have matching underlying types [#OpaqueTypeInference]
 5 |  if local {
 6 |  return Local()
   |         `- note: return statement has underlying type 'Local'
 7 |  } else {
 8 |  return Remote()
   |         `- note: return statement has underlying type 'Remote'
 9 |  }
10 | }
```

This matches Swift’s opaque result type rule: all return statements must infer one matching underlying type. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/opaque-type-inference/?utm_source=chatgpt.com "Underlying type inference for opaque result types ..."))

### Why?

`some ImageSource` means:

```text
makeSource(_:) -> hidden concrete type X where X: ImageSource
```

The compiler tries to infer `X`.

But the function body says:

```text
if local:
    X == Local
else:
    X == Remote
```

That is impossible unless `Local` and `Remote` are the same concrete type, which they are not.

The mistake is treating this:

```swift
-> some ImageSource
```

as if it meant this:

```swift
-> any ImageSource
```

They are different abstractions.

```text
some ImageSource:

Caller sees:        ImageSource capability
Compiler knows:     exact hidden type
Allowed returns:    one concrete type only

any ImageSource:

Caller sees:        ImageSource capability
Compiler knows:     existential container
Allowed returns:    multiple conforming concrete types
```

### Fix or redesign

#### Option 1: Use `any ImageSource` when runtime heterogeneity is the point

```swift
protocol ImageSource {}
struct Remote: ImageSource {}
struct Local: ImageSource {}

func makeSource(_ local: Bool) -> any ImageSource {
    if local {
        return Local()
    } else {
        return Remote()
    }
}
```

This is correct if the caller does not need to know whether it received `Local` or `Remote`.

#### Option 2: Use one concrete wrapper type with `some`

```swift
protocol ImageSource {}

struct Remote: ImageSource {}
struct Local: ImageSource {}

enum Source: ImageSource {
    case local(Local)
    case remote(Remote)
}

func makeSource(_ local: Bool) -> some ImageSource {
    if local {
        return Source.local(Local())
    } else {
        return Source.remote(Remote())
    }
}
```

Now both branches return `Source`, so the opaque underlying type is consistent.

#### Option 3: Make the caller choose the concrete type with generics

```swift
protocol ImageSource {
    init()
}

struct Remote: ImageSource {}
struct Local: ImageSource {}

func makeSource<S: ImageSource>(_ type: S.Type) -> S {
    S()
}

let local = makeSource(Local.self)
let remote = makeSource(Remote.self)
```

This is appropriate only when the caller, not the callee, should decide the concrete type.

### Why these fixes are correct

`any ImageSource` changes the contract to value-level polymorphism: “I may return different conforming types.”

The enum wrapper keeps the opaque contract: “I return one hidden concrete type.” The enum internally models the variability.

The generic version changes ownership of the type decision: “The caller chooses `S`; I construct that `S`.”

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`-> any ImageSource`|Runtime-selected source types, plugin systems, heterogeneous collections|Erases concrete type relationships; possible boxing/dynamic dispatch|
|`-> some ImageSource` with wrapper enum|You want static opacity but have finite internal variants|Requires wrapper/delegation; enum cases may become part of internal design|
|Generic `<S: ImageSource>`|Caller should choose the concrete type|Not useful when implementation must choose based on runtime state|
|Concrete return type `Source`|The wrapper type is meaningful public API|Exposes implementation shape|
|Type-erased wrapper like `AnyImageSource`|You need custom forwarding, stable ABI surface, or stored closures|More boilerplate; can hide costs|

---

## 6. Exercise

### Problem

Refactor an API returning `any Sequence` into a more type-preserving design and explain the benefits.

### Bad / naive version

```swift
struct User {
    let id: Int
    let isActive: Bool
}

func visibleIDs(from users: [User]) -> any Sequence<Int> {
    users.lazy
        .filter(\.isActive)
        .map(\.id)
}
```

### What is wrong?

```text
The API erases the concrete lazy sequence type even though the function always returns one stable sequence pipeline.

The caller only receives `any Sequence<Int>`.
That loses static type information and can reduce specialization opportunities.

There is no real heterogeneity here, so existential abstraction is unnecessary.
```

This is a common senior-level smell: using `any` because the real return type is ugly.

### Improved version

```swift
struct User {
    let id: Int
    let isActive: Bool
}

func visibleIDs<C: Collection<User>>(from users: C) -> some Sequence<Int> {
    users.lazy
        .filter(\.isActive)
        .map(\.id)
}
```

Usage:

```swift
let users = [
    User(id: 1, isActive: true),
    User(id: 2, isActive: false),
    User(id: 3, isActive: true)
]

let ids = visibleIDs(from: users)

print(Array(ids))
// [1, 3]
```

### Why this is better

The improved API hides the ugly concrete lazy pipeline while preserving the fact that there is one concrete return type. The caller does not need to know the full type, but the compiler still can.

It also generalizes the input from `[User]` to any `Collection<User>`, which is more flexible without erasing the output.

```text
Bad:
[User] -> any Sequence<Int>
Concrete input, erased output

Better:
C: Collection<User> -> some Sequence<Int>
Generic input, opaque-but-preserved output
```

### When `any Sequence` would still be valid

Use `any Sequence` when the function genuinely returns different sequence implementations and you intentionally do not want to expose or preserve that difference:

```swift
func ids(fromCache: Bool) -> any Sequence<Int> {
    if fromCache {
        return [1, 2, 3]
    } else {
        return Set([1, 2, 3])
    }
}
```

But if ordering matters, returning `Set` behind `any Sequence` may be a semantic bug. In that case, return `[Int]`, `some Collection<Int>`, or a domain-specific type instead.

---

## 7. Production guidance

Use `some` in production when:

```text
You return one concrete implementation type.
You want to hide implementation details from clients.
You want better static optimization than an existential usually allows.
You are building SwiftUI views or lazy transformation APIs.
You want to avoid exposing long generic concrete types.
```

Use `any` in production when:

```text
You need heterogeneous storage, such as [any ImageSource].
You need runtime substitution.
You are crossing an architectural boundary where concrete type identity should be erased.
You are building plugin registries, dependency containers, or type-erased adapters.
The concrete type really can vary by branch or runtime condition.
```

Use generics in production when:

```text
The caller should choose the concrete type.
Same-type relationships matter.
Associated types or Self requirements matter.
You are writing hot-path library code where specialization is valuable.
You want the strongest compile-time guarantees.
```

Be careful when:

```text
You replace <T: P> with any P just to reduce syntax.
You return any Sequence because the concrete lazy type is ugly.
You use AnyView or any View to work around SwiftUI type errors.
You erase a protocol with associated types and then discover you need the associated type relationship back.
You expose any P in public API without considering future source and performance impact.
```

Avoid when:

```text
Using any as the default abstraction everywhere.
Using some when the implementation genuinely needs multiple concrete return types.
Using generics when the type parameter appears only once and no relationship is being expressed.
Exposing a complex concrete return type publicly when some P would preserve flexibility.
```

Debugging checklist:

```text
Does this API need heterogeneity, or did we erase the type too early?
Who should choose the concrete type: caller or callee?
Do two parameters/results need to be the same concrete type?
Are we losing associated-type information?
Is this in a hot path where specialization matters?
Is this SwiftUI code fighting the type system because branches produce different View types?
Would an enum model the finite variants better than an existential?
Would a concrete domain type be clearer than any Protocol?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `some` hides the type and `any` means any protocol type.

This is incomplete. It misses the central rule: `some` hides **one concrete type**, while `any` erases into a runtime existential container.

### Senior answer

> `some P` is opaque: the concrete type is hidden from the caller but fixed for the declaration. `any P` is existential: it can hold different conforming concrete types at runtime. I use `some` for returned implementation hiding, generics for preserving caller-chosen type relationships, and `any` for heterogeneity or architectural boundaries.

### Staff-level answer

> I treat `some`, `any`, and generics as API-design tools, not syntax preferences. Generics preserve caller-selected type relationships and specialization. Opaque return types let a module hide implementation detail without giving up static type identity. Existentials are value-level abstraction and are appropriate for heterogeneous storage, plugin boundaries, or runtime substitution, but they erase relationships and can introduce dispatch/boxing costs. In public APIs, I choose deliberately because changing between concrete, generic, opaque, and existential shapes can become a source-compatibility and performance issue.

Staff-level questions to ask:

```text
Is the concrete type chosen by the caller or the implementation?
Does the API need to preserve same-type relationships?
Is the protocol being used as a capability, a boundary, or a storage type?
Is heterogeneity real, or are we hiding an inconvenient concrete type?
Will this be called in a hot path where specialization matters?
Would an enum or concrete domain type better model the variants?
Does this public API accidentally commit us to type erasure?
```

---

## 9. Interview-ready summary

`some P` and `any P` are both abstractions over protocol-conforming types, but they solve different problems. `some P` is an opaque type: the implementation returns one fixed concrete type, hidden from clients but preserved for the compiler. `any P` is an existential: a runtime container that can hold different conforming types and erases the concrete type relationship. Generics preserve the most type information because the caller chooses the concrete type and the compiler can enforce relationships across parameters and returns. My default is generics when relationships matter, `some` when returning a hidden stable implementation type, and `any` only when I genuinely need heterogeneity or value-level type erasure.

---

## 10. Flashcards

Q: What does `func f() -> some P` mean?

A: The function returns one specific concrete type conforming to `P`, hidden from callers but known to the compiler.

Q: What does `func f() -> any P` mean?

A: The function returns an existential container that can hold any concrete type conforming to `P`.

Q: Why does `some P` fail when different branches return different conforming types?

A: Opaque return types must have one consistent underlying concrete type across all return paths.

Q: Who chooses the concrete type in `func f<T: P>(_ value: T)`?

A: The caller chooses `T`; the function is generic over that concrete type.

Q: Who chooses the concrete type in `func f() -> some P`?

A: The implementation chooses the concrete type; callers only see the protocol capabilities.

Q: Why can `any P` hurt ergonomics?

A: It erases concrete type information, including same-type relationships and access to APIs not exposed by the protocol.

Q: Why can `any P` hurt performance?

A: It can reduce specialization opportunities and may require existential storage, boxing, and witness-table dispatch.

Q: When is `any P` the right tool?

A: When heterogeneous values, runtime substitution, plugin boundaries, or deliberate type erasure are part of the design.

Q: When is `some P` the right return type?

A: When the implementation returns one stable concrete type but wants to hide that concrete type from clients.

Q: What is a common smell involving `any Sequence`?

A: Returning `any Sequence` just because the real lazy sequence type is verbose, even though there is no real heterogeneity.

---

## 11. Related sections

- [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]
- [[B3 — Generics, constraints, and same-type relationships]]
- [[B4 — Dispatch model: static, class virtual, witness-table, and protocol extension dispatch]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — B2 rubric, questions, exercise, and code probe.
- The Swift Programming Language — “Opaque and Boxed Protocol Types.” ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/?utm_source=chatgpt.com "Opaque and Boxed Protocol Types - Documentation"))
- The Swift Programming Language Reference — opaque types and the single-underlying-type rule. ([docs.swift.org](https://docs.swift.org/swift-book/ReferenceManual/Types.html?utm_source=chatgpt.com "Types | Documentation"))
- Swift compiler diagnostics — opaque result type inference. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/opaque-type-inference/?utm_source=chatgpt.com "Underlying type inference for opaque result types ..."))
- Swift compiler diagnostics — existential `any`. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/existential-any/?utm_source=chatgpt.com "Existential any (ExistentialAny) | Documentation"))
- Swift compiler diagnostics — existential types and performance. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/existential-type/?utm_source=chatgpt.com "Existential Types and Performance (ExistentialType)"))
- Swift Evolution SE-0244 — Opaque Result Types. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md?utm_source=chatgpt.com "swift-evolution/proposals/0244-opaque-result-types.md at ..."))
- Swift Evolution SE-0335 — Introduce existential `any`. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md?utm_source=chatgpt.com "swift-evolution/proposals/0335-existential-any.md at main"))