---
tags:
  - interview-prep
  - swift
  - ios
---

# Swift Senior/Staff Rubric and Prioritized Study Checklist

Verified against modern Swift 6-era sources, including:
- Swift.org Documentation
- The Swift Programming Language
- Swift API Design Guidelines
- "Announcing Swift 6" (September 17, 2024)
- "Swift 6.2 Released" (September 15, 2025)
- Apple docs for `Sendable`, macros, Swift Package Manager settings, and strict concurrency adoption

This document is aimed at a seasoned iOS/macOS engineer. It focuses on Swift language knowledge, the caveats that matter in real codebases, and the judgment signals that distinguish senior from staff-level engineers.

## How to Use This

Use the rubric in two ways:
- As an interview rubric: probe specific concepts with the questions and exercises below.
- As a study checklist: work through the priority tiers at the end.

### Scoring Scale

Use a 0-4 scale per topic:
- `0`: No working understanding.
- `1`: Can repeat definitions, but cannot reason through edge cases.
- `2`: Can use the feature in normal code, but misses caveats and failure modes.
- `3`: Can explain semantics, tradeoffs, and common bugs; applies it correctly in production code.
- `4`: Can teach the topic, design APIs around it, predict interactions with other features, and debug subtle failures.

## Interview Use: Suggested Weighting

For a senior iOS/macOS engineer:
- Must be strong in `A1-A16`, `B1-B11`, `B13-B14`, `C1-C10`, `D1-D13`, `E1-E3`, `E5`, `F1-F3`.
- Should have at least working familiarity with `B12`, `C11-C12`, `D14`, `E4`, `E6-E8`, `F4-F6`.

For a staff-level or principal candidate:
- Expect strong reasoning across all senior areas.
- Expect clear judgment on API design, migration strategy, module boundaries, resilience, and concurrency tradeoffs.
- Expect them to explain not just "what Swift does" but "how to structure a codebase so the language works for you instead of against you."

## Prioritized Study Checklist

### Tier 1: Non-Negotiable for Senior iOS/macOS Work

- `A1` Value vs reference semantics, identity, CoW
- `A2` `let`/`var`/`mutating`
- `A3` Initialization model
- `A4` Optionals and nil modeling
- `A5` Pattern matching and exhaustiveness
- `A6` `defer` and cleanup semantics
- `A7` Functions, labels, defaults, and `inout`
- `A8` Closures, escaping, and captures
- `A9` Enum-based state modeling
- `A10` Extensions and retroactive modeling
- `A11` Access control
- `A12` Availability and conditional compilation
- `A13` Error handling and typed throws
- `A14` Operators and literal affordances
- `A15` Type choices: `struct`/`class`/`enum`/`protocol`/`actor`
- `A16` Regex and standard-library idioms
- `B1` Protocols, associated types, `Self`
- `B2` `any` vs `some`
- `B3` Generics and same-type constraints
- `B4` Dispatch model and protocol extension dispatch
- `B5` Key paths
- `B6` `Any`, metatypes, dynamic casting
- `B7` Synthesized conformances
- `B8` `Equatable`/`Hashable`
- `B9` `Codable` evolution
- `B10` Swift API design
- `B11` Resilience-aware public API thinking
- `B13` Property wrappers
- `B14` Result builders
- `C1` ARC fundamentals
- `C2` CoW behavior of stdlib types
- `C3` Exclusivity and `inout`
- `C4` String/Unicode correctness
- `C5` Collection complexity
- `C6` Unsafe memory basics
- `C9` Generic vs existential performance model
- `C10` Debug vs release behavior
- `D1` Structured concurrency fundamentals
- `D2` Cancellation and task hierarchy
- `D3` `async let` and task groups
- `D4` Actors
- `D5` `@MainActor`
- `D6` `Sendable` and `@Sendable`
- `D7` Actor reentrancy
- `D8` Isolation control
- `D9` Continuations
- `D10` `AsyncSequence` and streams
- `D11` Detached tasks
- `D13` Swift 6 strict concurrency migration
- `E1` Objective-C interop
- `E2` Foundation bridging
- `E3` C interop boundaries
- `E5` SwiftPM targets, products, resources
- `F1` Testing async and concurrent code
- `F2` Diagnostics and LLDB
- `F3` Performance investigation habits

### Tier 2: Strong Senior / Emerging Staff Depth

- `B12` Variadic generics and parameter packs
- `C7` Ownership modifiers
- `C8` `Copyable` and noncopyable types
- `C11` Strict memory safety mode
- `C12` `Span` and `InlineArray`
- `D12` Actors vs locks vs other synchronization
- `D14` Swift 6.2 approachable concurrency behavior
- `E6` Import visibility and dependency leakage
- `E7` ABI, source compatibility, module stability
- `E8` `@inlinable` and `@usableFromInline`
- `F4` Macros
- `F5` Language/build settings literacy
- `F6` Observation and generated semantics

### Tier 3: Staff-Level Differentiators and Specialized Mastery

- `E4` C++ interoperability
- `C12` Deep low-level use of `Span`/`InlineArray`
- `C7-C8` Ownership-driven API design for libraries
- `B11`, `E7-E8` Public SDK design under SemVer and binary-compatibility constraints
- `D14`, `F5` Migration strategy across multiple targets, modules, or client-facing packages
- `F4` Macro architecture decisions in library or infrastructure code

## Study Plan Order

Work in this order if building toward staff-level competence:
1. Learn semantic correctness first:
   - `A1-A16`, `B1-B6`, `C1-C6`
2. Build design judgment:
   - `B7-B11`, `B13-B14`, `E1-E3`, `E5`, `F1-F3`
3. Build concurrency mastery:
   - `D1-D13`
4. Add performance and low-level ownership depth:
   - `C7-C12`, `B12`
5. Add library and ecosystem depth:
   - `E4`, `E6-E8`, `F4-F6`, `D14`

## High-Signal Practice Formats

Use these to measure real understanding instead of passive recognition:
- Whiteboard or talk through a bug caused by protocol-extension dispatch.
- Refactor a completion-handler API to `async/await` and explain cancellation and isolation.
- Diagnose a retain cycle with closures and tasks.
- Evolve a `Codable` model without breaking old persisted data.
- Review an API and reduce its access surface while preserving testability.
- Explain how one `await` inside an actor method can introduce a logic bug.
- Compare an existential-based design to a generic one for a hot path.
- Wrap a C API behind a safe Swift interface with minimal unsafe surface area.
- Outline a migration plan from Swift 5.x to Swift 6 strict concurrency.
- Explain whether a library feature should be implemented with a macro, wrapper, protocol, or plain code.

## Minimum Interview Loop for a Senior/Staff Candidate

If you only have time for a tight interview loop, probe these:
- `A1`, `A3`, `A8`
- `B1`, `B2`, `B4`, `B10`
- `C1`, `C3`, `C4`, `C6`
- `D1`, `D4`, `D6`, `D7`, `D9`, `D13`
- `E1`, `E2`, `E5`
- `F1`, `F2`

These topics expose most of the semantic, design, and debugging weaknesses that matter in real Swift codebases.

## Detailed Rubric

### A. Core Semantics, Syntax, and Control Flow

- `A1. Value semantics, reference semantics, identity, and copy-on-write`
  - Expect: Understand the semantic difference between copying a value and sharing identity; know that many standard-library value types use copy-on-write.
  - Caveats: `let` on a class only freezes the reference, not interior mutable state; CoW is an optimization strategy, not the same thing as value semantics.
  - Measure with:
  - Ask: "What is the difference between `===` and `==`, and when is using `===` a design smell?"
  - Ask: "Why can `Array` usually behave like a value even though copying it may be cheap until mutation?"
  - Exercise: "Given a `struct` containing an `Array` and a `class` containing an `Array`, predict which mutations are visible across copies."
  - Code Probe:
    ```swift
    struct ValueBox {
        var items = [1]
    }

    final class RefBox {
        var items = [1]
    }

    var a = ValueBox()
    var b = a
    b.items.append(2)

    let c = RefBox()
    let d = c
    d.items.append(2)

    print(a.items, b.items, c.items, d.items)
    ```
  - Probe Ask: "What prints, and why?"
  - Probe Ask: "Which part of the behavior comes from value semantics, which part comes from reference semantics, and where does copy-on-write fit into the explanation?"

- `A2. `let`, `var`, `mutating`, and method semantics`
  - Expect: Understand how mutability is expressed for value and reference types, including `mutating` and `nonmutating`.
  - Caveats: A nonmutating setter or method can still mutate through reference storage; protocol requirements can force awkward mutability shapes; mutability choices also affect synthesized conformances and whether a type is safe to share across concurrency boundaries.
  - Measure with:
  - Ask: "Why does a `struct` method that changes stored properties need `mutating`, but a class method does not?"
  - Ask: "What does `let` mean for a `struct` instance versus a `class` instance?"
  - Ask: "How can one mutable reference-typed stored property change a type's `Sendable` story or make a synthesized conformance less meaningful?"
  - Exercise: "Design a property wrapper or computed property whose setter is intentionally `nonmutating`, and explain why."
  - Code Probe:
    ```swift
    struct Counter {
        private final class Storage {
            var value = 0
        }

        private let storage = Storage()

        var value: Int {
            get { storage.value }
            nonmutating set { storage.value = newValue }
        }
    }

    let counter = Counter()
    counter.value = 3
    print(counter.value)
    ```
  - Probe Ask: "Why does this compile even though `counter` is bound with `let`?"
  - Probe Ask: "Why is this legal, why might it still be surprising, and when would this design be justified or dangerous?"

- `A3. Initialization model`
  - Expect: Know designated vs convenience initializers, two-phase initialization, memberwise initialization, failable init, `required`, and initialization safety rules.
  - Caveats: Property observers do not fire during initialization; using `self` before full initialization is restricted; subclass initialization ordering matters.
  - Measure with:
  - Ask: "Why can't you access `self` freely before all stored properties are initialized?"
  - Ask: "When does Swift synthesize a memberwise initializer, and when does it disappear?"
  - Exercise: "Given a class hierarchy with stored properties, identify which initializer must call `super.init` first and why."

- `A4. Optionals and nil modeling`
  - Expect: Understand `Optional` as an enum, optional chaining, `map` vs `flatMap`, `guard let`, `if let`, implicitly unwrapped optionals, pattern matching, and when `nil` is a domain concept versus a smell.
  - Caveats: Force unwraps in infrastructure code often mask design problems; nested optionals are sometimes valid but usually indicate poor modeling; `!` should almost never be treated as ordinary control flow.
  - Measure with:
  - Ask: "What is the difference between `foo?.bar()` and `foo!.bar()` beyond syntax?"
  - Ask: "When is `T?` the right model, and when should you use an enum instead?"
  - Ask: "What are implicitly unwrapped optionals actually for, and why are they usually a legacy interop or lifecycle tool rather than a preferred model?"
  - Exercise: "Refactor a view model that uses multiple booleans and optionals into a single enum-based state machine."

- `A5. Pattern matching and exhaustive control flow`
  - Expect: Know `switch`, `if case`, `guard case`, `where`, tuple matching, enum associated values, and `@unknown default`.
  - Caveats: `default` can accidentally hide newly added enum cases; `fallthrough` rarely belongs in idiomatic Swift.
  - Measure with:
  - Ask: "Why is `@unknown default` preferable to plain `default` for some external enums?"
  - Ask: "How would you destructure and validate an enum with associated values in one branch?"
  - Exercise: "Write a switch over a networking result type that preserves exhaustiveness without a catch-all branch."

- `A6. `defer`, early exits, and cleanup semantics`
  - Expect: Understand `defer` ordering, especially around `throw`, `return`, and nested scopes.
  - Caveats: `defer` runs when leaving the current scope, not at function end in the abstract.
  - Measure with:
  - Ask: "In what order do multiple `defer` blocks execute?"
  - Ask: "Where does `defer` become clearer than duplicating cleanup code in `catch` and `return` paths?"
  - Exercise: "Use `defer` to guarantee cleanup of a file descriptor or lock in a function with multiple exit points."

- `A7. Functions, parameter labels, defaults, and API surface`
  - Expect: Understand argument labels, default arguments, variadics, `inout`, function types, and how function signatures contribute to API clarity.
  - Caveats: Default arguments are baked at call sites; `inout` is not plain pass-by-reference.
  - Measure with:
  - Ask: "Why are argument labels part of Swift API design rather than syntax decoration?"
  - Ask: "What are the semantic pitfalls of changing a default parameter value in a library?"
  - Exercise: "Redesign an overly positional API into a Swifty signature with clear labels and defaults."

- `A8. Closures, escaping, and capture semantics`
  - Expect: Know escaping vs nonescaping closures, capture behavior for value and reference types, capture lists, and `@autoclosure`.
  - Caveats: Capturing a mutable local boxes storage; capturing `self` strongly in long-lived async work frequently leaks.
  - Measure with:
  - Ask: "What changes when a closure becomes `@escaping`?"
  - Ask: "How does `[weak self]` differ from `[unowned self]`, and when is each justified?"
  - Exercise: "Diagnose a retain cycle involving a view controller, a service callback, and a stored closure."
  - Code Probe:
    ```swift
    final class Screen {
        var onTap: (() -> Void)?

        deinit {
            print("Screen deinit")
        }

        func bind() {
            onTap = {
                print(self)
            }
        }
    }

    var screen: Screen? = Screen()
    screen?.bind()
    screen = nil
    ```
  - Probe Ask: "What prints, and why?"
  - Probe Ask: "What retain cycle exists here, and what are the tradeoffs between `[weak self]` and `[unowned self]` for fixing it?"

- `A9. Enums, associated values, recursive enums, and state modeling`
  - Expect: Use enums to model finite state machines and invalid states away.
  - Caveats: Multiple booleans often indicate an enum is missing; associated values should encode semantics, not anonymous blobs.
  - Measure with:
  - Ask: "Why is an enum often a better model than a set of optional properties?"
  - Ask: "When do you need `indirect`, and what is the runtime implication?"
  - Exercise: "Model a request lifecycle with cancel/retry semantics as an enum and explain why it is safer than booleans."
  - Code Probe:
    ```swift
    struct LoadState {
        var isLoading = false
        var data: [Int]? = nil
        var errorMessage: String? = nil
    }
    ```
  - Probe Ask: "What invalid states does this model allow?"
  - Probe Ask: "Redesign it as an enum so the impossible states become unrepresentable."

- `A10. Extensions and retroactive modeling`
  - Expect: Know how extensions organize behavior, add protocol conformances, and change discoverability.
  - Caveats: Retroactive conformances are global and can create semantic collisions across modules.
  - Measure with:
  - Ask: "When is adding a conformance in an extension a good idea, and when is it dangerous?"
  - Ask: "Why can an extension-based conformance in one module affect code elsewhere?"
  - Exercise: "Review a package that adds `Codable` to an external type; explain the compatibility and ownership risks."

- `A11. Access control`
  - Expect: Understand `private`, `fileprivate`, `internal`, `package`, `public`, and `open`.
  - Caveats: `public` does not imply overridable; `open` has library evolution cost; overexposing APIs becomes a maintenance burden.
  - Measure with:
  - Ask: "What is the practical difference between `public` and `open`?"
  - Ask: "Why is `package` useful in multi-target Swift packages?"
  - Exercise: "Given a package with app, core, and test-support targets, choose the minimum access level for several symbols and justify each."

- `A12. Availability and conditional compilation`
  - Expect: Know `@available`, deprecation messaging, `#available`, `#if`, `canImport`, and source-compatibility guards.
  - Caveats: Runtime availability and compile-time conditional compilation solve different problems.
  - Measure with:
  - Ask: "What is the difference between `#available` and `#if canImport`?"
  - Ask: "How would you deprecate an API while giving users an actionable migration path?"
  - Exercise: "Write an API that uses a modern framework when available and a fallback otherwise, without duplicating the public surface."

- `A13. Error handling and typed throws`
  - Expect: Know `throws`, `rethrows`, `Result`, `try?`, `try!`, and modern typed throws in Swift 6.
  - Caveats: `try?` collapses information; typed throws improve precision but can complicate API surfaces if overused.
  - Measure with:
  - Ask: "When should you return `Result` instead of throwing?"
  - Ask: "What does `throws(Never)` mean semantically?"
  - Exercise: "Refactor a generic transform API from `rethrows` or untyped `throws` to typed throws and explain the tradeoff."

- `A14. Operators, literals, and language affordances`
  - Expect: Understand custom operators, precedence groups, literal protocols, and when DSL-like code helps or hurts.
  - Caveats: Custom operators can destroy readability and tooling ergonomics; literal conformances can create surprising behavior.
  - Measure with:
  - Ask: "When is a custom operator justified in Swift?"
  - Ask: "What are the maintenance risks of overusing `ExpressibleBy...Literal` conformances?"
  - Exercise: "Review a codebase that introduces several operators for business logic; identify which ones improve readability and which ones are a mistake."

- `A15. Type choices: struct, class, enum, protocol, and actor`
  - Expect: Know how to choose between structs, classes, enums, protocols, and actors based on value semantics, identity, finite-state modeling, abstraction boundaries, and isolated mutable state.
  - Caveats: Reaching for classes or protocols by default often creates incidental complexity; actors are for isolated mutable state, not a generic replacement for all model types.
  - Measure with:
  - Ask: "When is a `class` genuinely required instead of a `struct`?"
  - Ask: "When should mutable shared state live behind an actor rather than a class plus manual synchronization?"
  - Exercise: "Take a small feature with domain models, shared cache, request state, and service abstraction, and choose `struct`, `class`, `enum`, `protocol`, or `actor` for each piece with justification."

- `A16. Regex and standard-library idioms`
  - Expect: Know how to use Swift regex literals/DSL and standard-library algorithms and collection APIs before inventing custom parsing, looping, or transformation helpers.
  - Caveats: Regexes and fluent chains can become unreadable if they encode too much logic at once; a simple loop is sometimes clearer than a clever pipeline.
  - Measure with:
  - Ask: "When is Swift Regex a better fit than `NSRegularExpression` or ad hoc string splitting?"
  - Ask: "When do `map`, `compactMap`, `flatMap`, `reduce(into:)`, `lazy`, and collection helpers improve code, and when do they make code harder to read?"
  - Exercise: "Refactor imperative text-processing or collection-transformation code into clearer standard-library and regex-based code without making the logic more opaque."

### B. Type System, Abstraction, and API Design

- `B1. Protocols, associated types, `Self` requirements, and compositions`
  - Expect: Know when protocols model behavior well, when associated types are necessary, how protocol compositions help express capability sets, and why protocols with `Self` or associated types behave differently from simple existential-friendly protocols.
  - Caveats: Over-protocolization can produce unusable APIs and erase important type relationships; marker protocols should communicate real semantic guarantees rather than serve as empty taxonomy labels.
  - Measure with:
  - Ask: "Why can't every protocol be used directly as a type?"
  - Ask: "When should a protocol have an associated type instead of using an existential payload?"
  - Ask: "When is a protocol composition like `P & Q` better than creating a new umbrella protocol, and when is a marker protocol justified?"
  - Exercise: "Design a `Cache` abstraction and decide whether it should use associated types, generics, or type erasure."

- `B2. Existentials (`any`) vs opaque types (`some`)`
  - Expect: Understand representation, flexibility, and performance tradeoffs between `any P`, `some P`, and generic parameters.
  - Caveats: Existentials box and erase type relationships; opaque types preserve a concrete hidden type but must stay consistent.
  - Measure with:
  - Ask: "What is the difference between `func f() -> some P` and `func f() -> any P`?"
  - Ask: "Why might replacing generics with existentials hurt both ergonomics and performance?"
  - Exercise: "Refactor an API returning `any Sequence` into a more type-preserving design and explain the benefits."
  - Code Probe:
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
  - Probe Ask: "Why does this fail to compile?"
  - Probe Ask: "How would the behavior and tradeoffs change if the return type were `any ImageSource` instead?"

- `B3. Generics, constraints, and same-type relationships`
  - Expect: Know generic constraints, `where` clauses, conditional conformances, and expressing relationships between types.
  - Caveats: If you erase too early, you lose compile-time guarantees and specialization opportunities.
  - Measure with:
  - Ask: "What problem does a same-type constraint solve that a simple protocol constraint does not?"
  - Ask: "When does a generic API provide meaningfully stronger guarantees than `any Protocol`?"
  - Exercise: "Write a generic function that only accepts two collections with the same element type and explain why that matters."

- `B4. Dispatch model: static, class virtual, witness-table, and protocol extension dispatch`
  - Expect: Know the difference between static dispatch, class virtual dispatch, Objective-C dynamic dispatch when opted into, witness-table dispatch for protocol requirements, and non-requirement methods added in protocol extensions.
  - Caveats: Many engineers think all protocol extension methods are dynamically dispatched; they are not; choosing `dynamic` or `@objc` changes both behavior and performance characteristics.
  - Measure with:
  - Ask: "How do dispatch rules differ between a struct method, a class method, a protocol requirement, and a non-requirement method in a protocol extension?"
  - Ask: "Why can calling a method through a protocol-typed value produce different behavior than calling it on a concrete type, and when does Objective-C dynamic dispatch enter the picture?"
  - Exercise: "Given a protocol, an extension, and a conforming type, predict which implementation is called in three different call sites."
  - Code Probe:
    ```swift
    protocol P {
        func f()
    }

    extension P {
        func f() { print("default f") }
        func g() { print("extension g") }
    }

    struct S: P {
        func f() { print("S.f") }
        func g() { print("S.g") }
    }

    let x: P = S()
    x.f()
    x.g()
    ```
  - Probe Ask: "What prints, and why?"
  - Probe Ask: "Which call uses witness-table dispatch, which call does not, and how would you redesign this if `g()` needed polymorphic behavior?"

- `B5. Key paths and strongly typed indirection`
  - Expect: Understand key paths as compile-time safe references to properties, including writable and reference-writable variants.
  - Caveats: Key paths are not just strings with type safety; they preserve structure and can improve reuse.
  - Measure with:
  - Ask: "When is a key path better than passing a closure?"
  - Ask: "What is the difference between `WritableKeyPath` and `ReferenceWritableKeyPath`?"
  - Exercise: "Build a generic sorting or mapping helper using key paths and justify it against alternative designs."

- `B6. `Any`, `AnyObject`, metatypes, and runtime type information`
  - Expect: Understand dynamic casting, `is`/`as?`/`as!`, metatypes, factory-style APIs based on `T.Type`, `Self`-driven construction patterns, and the limits of reflection in Swift.
  - Caveats: Heavy use of `Any` often means type information was thrown away too early.
  - Measure with:
  - Ask: "What is the difference between `T.Type`, `Any.Type`, and `AnyObject`?"
  - Ask: "When does a metatype-based factory or registry make sense, and what concurrency or type-erasure pitfalls appear when you start passing metatypes around?"
  - Ask: "What architectural smell does frequent downcasting usually reveal?"
  - Exercise: "Review a plugin-style registry built on `Any`; propose a safer typed alternative."

- `B7. Synthesized conformances and semantic correctness`
  - Expect: Know when Swift synthesizes `Equatable`, `Hashable`, `Codable`, `Sendable`, and `CaseIterable`, and when synthesis is semantically wrong even if legal.
  - Caveats: Synthesized correctness is structural, not domain-aware.
  - Measure with:
  - Ask: "Why might synthesized `Equatable` be technically valid but semantically wrong for a model?"
  - Ask: "When should you hand-write `Hashable` instead of using synthesis?"
  - Exercise: "Given an entity type with stable identity and frequently changing fields, decide what equality should mean and implement it."

- `B8. `Equatable`, `Hashable`, and collection correctness`
  - Expect: Understand equality/hash invariants and how they affect dictionaries and sets.
  - Caveats: Mutable hashed state is a correctness bug; hash values are not persistent IDs.
  - Measure with:
  - Ask: "What invariant must always hold between `==` and `hash(into:)`?"
  - Ask: "Why is storing mutable fields in a `Hashable` key dangerous?"
  - Exercise: "Debug a bug where a value disappears from a `Set` after a property mutation."
  - Code Probe:
    ```swift
    final class User: Hashable {
        let id: Int
        var name: String

        init(id: Int, name: String) {
            self.id = id
            self.name = name
        }

        static func == (lhs: User, rhs: User) -> Bool {
            lhs.id == rhs.id && lhs.name == rhs.name
        }

        func hash(into hasher: inout Hasher) {
            hasher.combine(id)
            hasher.combine(name)
        }
    }

    let a = User(id: 1, name: "A")
    let b = User(id: 1, name: "B")
    var users: Set<User> = [a, b]
    a.name = "B"

    let values = users.map { "\($0.id)-\($0.name)" }.sorted()
    print(users.count)
    print(values)
    print(Set(values).count)
    ```
  - Probe Ask: "Why is this a correctness bug even though the code compiles?"
  - Probe Ask: "How would you redefine equality and hashing to match stable collection semantics?"

- `B9. `Codable` and schema evolution`
  - Expect: Know synthesis, custom coding keys, lossy decoding tradeoffs, backward compatibility, and versioning concerns.
  - Caveats: `Codable` makes simple cases easy, not complex schema evolution trivial.
  - Measure with:
  - Ask: "What breaks when you rename a property in a persisted `Codable` model?"
  - Ask: "When is it worth writing a custom `init(from:)`?"
  - Exercise: "Evolve a persisted settings model by renaming a field and adding a new enum case without breaking older data."
  - Code Probe:
    ```swift
    struct Settings: Codable {
        var sortMode: SortMode
    }

    enum SortMode: String, Codable {
        case byName
        case byDate
    }
    ```
  - Probe Ask: "Suppose older app versions persisted `{"sortOrder":"name"}` instead. How would you evolve this model without breaking existing data?"
  - Probe Ask: "When would synthesis stop being enough here, and what would a custom `init(from:)` need to handle?"

- `B10. API design guidelines and Swift-native surface design`
  - Expect: Understand naming, argument labels, mutating vs nonmutating APIs, fluent vs explicit style, and minimizing surface area.
  - Caveats: A technically correct API can still be un-Swifty and expensive to maintain.
  - Measure with:
  - Ask: "Why are naming and labels part of correctness in Swift API design?"
  - Ask: "What signs show an API was ported mechanically from another language without adapting to Swift idioms?"
  - Exercise: "Review a small SDK API and rewrite three signatures to better fit Swift API Design Guidelines."

- `B11. Library evolution and resilience-aware API design`
  - Expect: Know what becomes hard to change once a symbol is public, and how resilience affects enums, structs, and inlining choices.
  - Caveats: Public API is a promise; convenience today can become ABI or source-compatibility debt later.
  - Measure with:
  - Ask: "Why is making something `public` in a library more than an access-control decision?"
  - Ask: "When would `@frozen` or `@inlinable` be a mistake?"
  - Exercise: "Evaluate whether a public enum in an SDK should be frozen and how clients should switch over it."

- `B12. Variadic generics and parameter packs`
  - Expect: Know what parameter packs solve, where pack iteration helps, and why they matter for removing overload walls.
  - Caveats: Powerful but advanced; useful mainly in libraries, macro-generated code, and generic infrastructure.
  - Measure with:
  - Ask: "What limitation of pre-Swift-5.9/6 code do parameter packs address?"
  - Ask: "When is variadic generics actually the right abstraction instead of a simpler overload set?"
  - Exercise: "Sketch a tuple or builder-style API that becomes materially cleaner with parameter packs."

- `B13. Property wrappers and generated storage semantics`
  - Expect: Know wrapped value, projected value, initialization rules, access control effects, and wrapper composition.
  - Caveats: Wrappers affect storage, mutability, isolation, and sometimes API shape in non-obvious ways.
  - Measure with:
  - Ask: "What code does a property wrapper effectively synthesize around a property?"
  - Ask: "How can a wrapper accidentally hide thread-safety or actor-isolation problems?"
  - Exercise: "Implement a small wrapper that validates inputs or logs mutations, and explain its initialization behavior."
  - Code Probe:
    ```swift
    @propertyWrapper
    struct Clamped {
        private var value: Int

        init(wrappedValue: Int) {
            value = min(max(wrappedValue, 0), 100)
        }

        var wrappedValue: Int {
            get { value }
            set { value = min(max(newValue, 0), 100) }
        }
    }

    struct Score {
        @Clamped var value = 150
    }

    print(Score().value)
    ```
  - Probe Ask: "What prints, and why?"
  - Probe Ask: "What storage does Swift effectively synthesize for `@Clamped var value`, and how would custom memberwise initialization interact with the wrapper?"

- `B14. Result builders and DSL construction`
  - Expect: Understand how builders transform syntax and why they are useful for declarative APIs.
  - Caveats: Builders can create cryptic type-checking failures and hide control-flow semantics from readers.
  - Measure with:
  - Ask: "What problem does a result builder solve better than plain closures?"
  - Ask: "What are the readability and compile-time costs of overusing builders?"
  - Exercise: "Design a tiny builder-based DSL and explain whether it is actually better than an array of closures or plain structs."

### C. Memory, Ownership, Performance, and Safety

- `C1. ARC fundamentals`
  - Expect: Know retain/release semantics, strong cycles, `weak`, `unowned`, and deinitialization timing.
  - Caveats: ARC is deterministic at scope/lifetime boundaries but can be extended by captures, async work, autorelease pools, and bridging.
  - Measure with:
  - Ask: "When is `unowned` correct, and when is it a crash waiting to happen?"
  - Ask: "Why can an object survive longer than expected even without an obvious strong reference in your current scope?"
  - Exercise: "Trace a retain cycle involving a view controller, a timer, and a closure."
  - Code Probe:
    ```swift
    final class Owner {
        var child: Child?

        deinit {
            print("Owner deinit")
        }
    }

    final class Child {
        var onDone: (() -> Void)?

        deinit {
            print("Child deinit")
        }
    }

    func make() {
        let owner = Owner()
        let child = Child()
        owner.child = child
        child.onDone = { print(owner) }
    }

    make()
    ```
  - Probe Ask: "What deinitializers run, and why?"
  - Probe Ask: "Identify the cycle and fix it without changing the outward behavior of the callback."

- `C2. Copy-on-write behavior of standard-library types`
  - Expect: Understand uniqueness checks, shared storage, and when mutation triggers actual copying.
  - Caveats: Bridging to Objective-C can defeat expectations; slices often share backing storage.
  - Measure with:
  - Ask: "Why can passing an `Array` around be cheap until mutation?"
  - Ask: "What hidden retention or memory-footprint issues can `ArraySlice` create?"
  - Exercise: "Diagnose why holding onto a small slice of a huge array keeps the whole buffer alive."

- `C3. Exclusivity enforcement and `inout``
  - Expect: Understand exclusive access rules, dynamic exclusivity traps, and the borrow-like nature of `inout`.
  - Caveats: `inout` is not a stable alias; overlapping access bugs can compile and still trap at runtime.
  - Measure with:
  - Ask: "Why is `inout` not the same as C++ references or pointers?"
  - Ask: "What kind of overlapping accesses does Swift try to forbid?"
  - Exercise: "Explain why passing the same variable to two `inout` parameters is invalid and how to redesign the API."
  - Code Probe:
    ```swift
    func swapValues(_ a: inout Int, _ b: inout Int) {
        let temp = a
        a = b
        b = temp
    }

    var x = 1
    swapValues(&x, &x)
    ```
  - Probe Ask: "Why is this invalid in Swift?"
  - Probe Ask: "What exclusivity rule is being protected, and what API redesigns avoid the overlapping-access problem?"

- `C4. String and Unicode correctness`
  - Expect: Know that `String` is a collection of extended grapheme clusters, not bytes or UTF-16 code units.
  - Caveats: Indexing is not integer-based; apparent character count and storage representation differ.
  - Measure with:
  - Ask: "Why is `String` indexing not O(1) by integer offset?"
  - Ask: "What bugs happen when engineers assume one user-visible character equals one Unicode scalar or one UTF-16 code unit?"
  - Exercise: "Design a truncation function for user-visible text that does not split grapheme clusters."
  - Code Probe:
    ```swift
    let s = "\u{1F1FA}\u{1F1F8}e\u{301}"
    print(s.count)
    print(s.utf16.count)
    ```
  - Probe Ask: "Why are these counts different?"
  - Probe Ask: "Why is truncating or slicing user-visible text by UTF-16 or byte offset a correctness bug?"

- `C5. Collection protocols, complexity, and index invalidation`
  - Expect: Understand `Sequence` vs `Collection` vs `BidirectionalCollection` vs `RandomAccessCollection`, and the complexity contracts they imply.
  - Caveats: Not every collection supports cheap indexing; mutating a collection can invalidate indices.
  - Measure with:
  - Ask: "Why is taking an `Array` parameter more restrictive than taking a `Collection`?"
  - Ask: "What assumptions are safe only for `RandomAccessCollection`?"
  - Exercise: "Refactor an algorithm to accept the weakest useful collection constraint without losing the required complexity."

- `C6. Unsafe pointers, buffers, and raw memory`
  - Expect: Know pointer lifetime, initialization, binding, rebinding, alignment, and when to prefer safe wrappers over unsafe memory APIs.
  - Caveats: Most pointer bugs are lifetime or aliasing bugs, not syntax bugs.
  - Measure with:
  - Ask: "What is the difference between raw memory and typed memory in Swift?"
  - Ask: "When is `withUnsafeBytes` safer than storing an unsafe pointer for later?"
  - Exercise: "Review code that escapes a pointer from `withUnsafeBytes`; explain exactly why it is invalid."
  - Code Probe:
    ```swift
    let bytes: [UInt8] = [1, 2, 3]
    var saved: UnsafePointer<UInt8>?

    bytes.withUnsafeBufferPointer { buffer in
        saved = buffer.baseAddress
    }

    print(saved![0])
    ```
  - Probe Ask: "Why is this invalid even if it appears to work?"
  - Probe Ask: "What lifetime guarantee does `withUnsafeBufferPointer` actually provide, and how would you redesign this safely?"

- `C7. Ownership modifiers: `borrowing`, `consuming`, and `consume``
  - Expect: Understand Swift's direction toward explicit ownership for performance-sensitive and move-aware code.
  - Caveats: Ownership annotations are semantically meaningful, not micro-optimizations to sprinkle randomly.
  - Measure with:
  - Ask: "What problem do `borrowing` and `consuming` solve?"
  - Ask: "Why is explicit ownership more than a performance feature for low-level APIs?"
  - Exercise: "Explain how a resource-holding type should expose an operation that consumes the resource exactly once."

- `C8. `Copyable`, noncopyable types, and `~Copyable``
  - Expect: Know what `Copyable` means, when noncopyable modeling is useful, and how modern Swift lets generic code work with both copyable and noncopyable types.
  - Caveats: Noncopyable types change API design, generic constraints, and control-flow patterns; suppressing copyability should model ownership truth, not aesthetic preference.
  - Measure with:
  - Ask: "What does `Copyable` mean semantically, and when do you need to spell it or suppress it?"
  - Ask: "Why would you make a Swift type noncopyable?"
  - Ask: "How does noncopyability interact with generics and protocol design, especially when you want one API to support both copyable and noncopyable inputs?"
  - Exercise: "Model a file handle or lock token as a noncopyable type and explain what misuse it prevents."

- `C9. Existential, generic, and allocation performance model`
  - Expect: Know that abstraction choices affect specialization, boxing, dynamic dispatch, and allocations.
  - Caveats: Performance intuition should be evidence-driven, but engineers should still predict likely hotspots.
  - Measure with:
  - Ask: "When will a generic API usually optimize better than an existential-based one?"
  - Ask: "Why can storing protocol values in arrays allocate or inhibit specialization?"
  - Exercise: "Compare two designs for a hot path: one generic, one existential; predict the likely optimization differences."

- `C10. Compiler optimization behavior and debug vs release differences`
  - Expect: Know that debug builds can mislead performance investigations; understand specialization, inlining, optimization levels, assertion behavior, and when inspecting lowered/generated output such as SIL is warranted for ownership or performance questions.
  - Caveats: Measuring debug builds and reasoning from them is often wrong.
  - Measure with:
  - Ask: "Why can a data structure or algorithm seem slow in debug and fine in release?"
  - Ask: "When would you inspect SIL or compiler-generated output rather than keep guessing about a performance or ownership issue?"
  - Ask: "What is the difference in intent between `assert`, `precondition`, and `fatalError`?"
  - Exercise: "Given a crash that appears only in release or only in debug, outline a debugging plan."

- `C11. Strict memory safety mode and safe systems programming`
  - Expect: Know what Swift 6.2's strict memory safety checking is trying to expose and why explicit unsafe acknowledgments matter.
  - Caveats: The absence of warnings is not proof that unsafe code is correct; the presence of unsafe APIs should be intentional and auditable.
  - Measure with:
  - Ask: "What kinds of constructs does strict memory safety mode want you to acknowledge?"
  - Ask: "Why is making unsafe code explicit valuable even if the code is known-correct?"
  - Exercise: "Review a low-level wrapper around a C API and identify where explicit unsafe boundaries should exist."

- `C12. `Span`, `InlineArray`, and modern low-level abstractions`
  - Expect: Know that modern Swift increasingly offers safe low-level tools for contiguous memory and fixed-size inline storage.
  - Caveats: These are specialized tools; misuse comes from adopting them without a concrete memory-layout or performance need.
  - Measure with:
  - Ask: "What problem does `Span` solve better than raw pointers?"
  - Ask: "When is a fixed-size inline container a better fit than `Array`?"
  - Exercise: "Describe a real scenario in graphics, parsing, crypto, or DSP where `InlineArray` or `Span` would be appropriate."

### D. Concurrency, Isolation, and Correctness

- `D1. Structured concurrency fundamentals`
  - Expect: Understand `async/await`, suspension points, task trees, and the difference between concurrency and parallelism.
  - Caveats: `await` is a suspension point, not merely syntax noise; every suspension can change interleaving assumptions.
  - Measure with:
  - Ask: "What does `await` actually mean semantically?"
  - Ask: "Why is structured concurrency safer than ad hoc callback trees or detached work?"
  - Exercise: "Rewrite a callback-based API to `async/await` and explain what correctness properties improve."

- `D2. Task hierarchy, cancellation, and priorities`
  - Expect: Know cooperative cancellation, task-local propagation, parent-child relationships, and priority inheritance basics.
  - Caveats: Cancellation does nothing unless code checks or uses cancellation-aware APIs.
  - Measure with:
  - Ask: "Why is cancellation in Swift cooperative rather than preemptive?"
  - Ask: "What does a child task inherit from its parent?"
  - Exercise: "Design an image-loading pipeline that cancels stale work when cells are reused."

- `D3. `async let`, task groups, and choosing the right concurrency primitive`
  - Expect: Know when to use `async let`, throwing task groups, discarding task groups, and plain sequential `await`.
  - Caveats: Not all concurrent-looking code should be parallelized; overhead and cancellation behavior differ.
  - Measure with:
  - Ask: "When is `async let` better than a task group?"
  - Ask: "What are the failure and cancellation semantics of a throwing task group?"
  - Exercise: "Parallelize three independent network calls and then explain how failure in one should affect the others."

- `D4. Actors and actor isolation`
  - Expect: Understand actor-isolated state, message sends across actor boundaries, and why actors prevent data races but not all correctness issues.
  - Caveats: Actors serialize access to their isolated state, not the entire logical workflow of a feature.
  - Measure with:
  - Ask: "What kind of problem does an actor solve, and what kind of problem does it not solve?"
  - Ask: "Why can actor-based code still have stale-read or check-then-act logic bugs?"
  - Exercise: "Review a token-refresh actor and identify a logic race that can still happen despite actor isolation."

- `D5. Global actors and `@MainActor``
  - Expect: Understand when `@MainActor` belongs on types versus members, and how UI isolation should be expressed.
  - Caveats: `@MainActor` is an isolation contract; the main actor is not a generic substitute for all synchronization.
  - Measure with:
  - Ask: "When should a whole type be `@MainActor` instead of individual methods?"
  - Ask: "Why is marking too much code `@MainActor` a design smell?"
  - Exercise: "Refactor a view model so UI mutation is main-actor isolated while background work remains off the main actor."

- `D6. `Sendable` and `@Sendable``
  - Expect: Understand sendability rules for value types, classes, actors, closures, and generic APIs.
  - Caveats: `@unchecked Sendable` is a proof obligation, not a compiler silencer; sendability is about safe cross-isolation transfer, not just thread use.
  - Measure with:
  - Ask: "Why can a `final` immutable class sometimes be `Sendable`, and why are many mutable classes not?"
  - Ask: "What does `@Sendable` require of captured values?"
  - Exercise: "Take a non-sendable service type used in async code and propose the safest fix among actorization, value modeling, or unchecked conformance."
  - Code Probe:
    ```swift
    final class Cache {
        var store: [String: Int] = [:]
    }

    func run(_ operation: @Sendable @escaping () async -> Void) {}

    let cache = Cache()
    run {
        cache.store["a"] = 1
    }
    ```
  - Probe Ask: "Why does Swift 6 complain about this?"
  - Probe Ask: "What are the safest fixes, and when would you choose actor isolation, value semantics, or `@unchecked Sendable`?"

- `D7. Actor reentrancy and logic races`
  - Expect: Know that actor methods can interleave at suspension points and that reentrancy is a central design concern.
  - Caveats: Many engineers think actors imply non-reentrancy by default; they do not.
  - Measure with:
  - Ask: "What does actor reentrancy mean in practice?"
  - Ask: "How can an `await` in the middle of an actor method invalidate assumptions made earlier in the method?"
  - Exercise: "Debug an actor method that checks a cache, awaits network work, then writes stale data."
  - Code Probe:
    ```swift
    actor Counter {
        var value = 0

        func incrementIfZero() async {
            if value == 0 {
                try? await Task.sleep(nanoseconds: 10_000_000)
                value += 1
            }
        }
    }
    ```
  - Probe Ask: "If two tasks call `incrementIfZero()` concurrently, can `value` become `2`? Why?"
  - Probe Ask: "What actor reentrancy behavior creates the bug, and how would you fix the logic rather than just moving code around?"

- `D8. Isolation control: `nonisolated`, isolated parameters, and `assumeIsolated``
  - Expect: Understand when behavior should or should not be isolated, and how to express that safely.
  - Caveats: Removing isolation to satisfy the compiler can reintroduce races or semantic confusion.
  - Measure with:
  - Ask: "What does `nonisolated` mean on a member of an actor or globally isolated type?"
  - Ask: "When is `assumeIsolated` appropriate, and what makes it dangerous?"
  - Exercise: "Given an actor API, identify which helpers can be `nonisolated` because they don't touch isolated state."

- `D9. Continuations and bridging callback APIs`
  - Expect: Know `withCheckedContinuation`, `withCheckedThrowingContinuation`, correctness rules, and exactly-once resume semantics.
  - Caveats: Multiple resumes and forgotten resumes are correctness bugs; checked continuations help detect them but do not design the API for you.
  - Measure with:
  - Ask: "What invariants must hold when using a continuation?"
  - Ask: "Why can adapting a multi-callback API to a single continuation be the wrong abstraction?"
  - Exercise: "Wrap a legacy completion-handler API and explain how you guarantee exactly one resume."
  - Code Probe:
    ```swift
    func legacy(_ completion: @escaping (Result<Int, Error>) -> Void) {
        completion(.success(1))
        completion(.success(2))
    }

    func adapted() async throws -> Int {
        try await withCheckedThrowingContinuation { continuation in
            legacy { result in
                continuation.resume(with: result)
            }
        }
    }
    ```
  - Probe Ask: "What is broken about this adaptation?"
  - Probe Ask: "What invariant does `withCheckedThrowingContinuation` require, and how would you handle a multi-event callback correctly instead?"

- `D10. `AsyncSequence`, streams, and backpressure-aware design`
  - Expect: Understand when a single async result should become an async sequence, and how streams model event sources.
  - Caveats: Async streams can hide buffering and cancellation problems if treated like magic.
  - Measure with:
  - Ask: "When should an API return `AsyncSequence` instead of a callback or delegate?"
  - Ask: "What cancellation and resource-lifetime issues arise with `AsyncStream`?"
  - Exercise: "Model location updates or websocket messages as an async sequence and explain how consumers stop the stream safely."
  - Code Probe:
    ```swift
    func makeStream() -> AsyncStream<Int> {
        AsyncStream { continuation in
            for i in 0..<3 {
                continuation.yield(i)
            }
        }
    }
    ```
  - Probe Ask: "Why might a consumer hang forever waiting for this stream to finish?"
  - Probe Ask: "What does this teach you about stream lifetime, completion, and resource cleanup?"

- `D11. Detached tasks, task inheritance, and isolation loss`
  - Expect: Know the difference between `Task {}` and `Task.detached {}`.
  - Caveats: Detached tasks lose actor context, task-local values, and structured-parent guarantees; many uses are wrong.
  - Measure with:
  - Ask: "Why is `Task.detached` often a footgun?"
  - Ask: "What does `Task {}` inherit that a detached task does not?"
  - Exercise: "Review code that uses detached tasks from a view model and explain why it may violate isolation or lifecycle expectations."
  - Code Probe:
    ```swift
    @MainActor
    final class ViewModel {
        var title = ""

        func load() {
            Task.detached {
                self.title = "Done"
            }
        }
    }
    ```
  - Probe Ask: "Why is `Task.detached` the wrong tool here?"
  - Probe Ask: "What context is lost relative to `Task {}`, and what is the correct way to structure this work?"

- `D12. Synchronization library, mutexes, and choosing between actors and locks`
  - Expect: Know that actors are not the only concurrency primitive; low-level synchronization still matters in some contexts.
  - Caveats: Locking can be the correct tool for tightly scoped shared mutable state, but it needs disciplined invariants.
  - Measure with:
  - Ask: "When is an actor a better fit than a mutex, and when is a mutex still appropriate?"
  - Ask: "Why can wrapping everything in a lock be worse than modeling ownership explicitly?"
  - Exercise: "Choose between a lock, an actor, and immutability for a shared cache used from UI and background work."

- `D13. Swift 6 strict concurrency migration`
  - Expect: Know language-mode migration strategy, warning-to-error progression, and pragmatic use of annotations like `@preconcurrency`.
  - Caveats: Silencing diagnostics without rethinking ownership/isolation is not a migration.
  - Measure with:
  - Ask: "What did complete strict concurrency checking provide before Swift 6 language mode?"
  - Ask: "When is `@preconcurrency` acceptable, and why is it rarely the final design?"
  - Exercise: "Outline a migration plan for a medium-sized app moving from Swift 5.x concurrency warnings to Swift 6 strict checking."

- `D14. Swift 6.2 approachable concurrency and new defaults`
  - Expect: Know about optional default isolation to `MainActor`, `@concurrent`, and the evolving execution semantics of `nonisolated async`.
  - Caveats: This area is version-sensitive; staff engineers should know which behavior is language-version or build-setting dependent.
  - Measure with:
  - Ask: "What problem is default actor isolation to `MainActor` trying to solve, and what does it trade off?"
  - Ask: "Why does Swift 6.2's change to `nonisolated async` behavior matter for app code, and when would you reach for `@concurrent` to make concurrency explicit?"
  - Exercise: "Review a UI-heavy package and decide whether default isolation to `MainActor` is appropriate, partial, or harmful."

### E. Interop, Modules, and Distribution

- `E1. Objective-C interoperability`
  - Expect: Know `@objc`, `dynamic`, selector-based APIs, `NSObject` constraints, NSError bridging, and what Swift features do not map cleanly to ObjC.
  - Caveats: Opting into Objective-C runtime behavior affects dispatch, reflection, KVC/KVO compatibility, and API shape.
  - Measure with:
  - Ask: "When do you need `@objc` or `dynamic`, and what are the tradeoffs?"
  - Ask: "Why do some Swift language features not export cleanly to Objective-C?"
  - Exercise: "Expose a Swift API to Objective-C and identify what must change about its signature and types."

- `E2. Foundation and Objective-C bridging behavior`
  - Expect: Understand how Swift standard-library types bridge to Foundation types and where costs or semantic shifts occur.
  - Caveats: Bridging can copy, box, or change mutability assumptions.
  - Measure with:
  - Ask: "What happens when Swift `String` or `Array` crosses into Objective-C APIs?"
  - Ask: "Why can bridging hide performance costs in hot paths?"
  - Exercise: "Investigate a performance issue around repeated bridging between Swift collections and Foundation APIs."

- `E3. C interoperability and unsafe boundaries`
  - Expect: Know how Swift interacts with C APIs, pointers, nullability, and imported macros/constants at a practical level.
  - Caveats: C interop often imports unsafety into otherwise safe Swift code.
  - Measure with:
  - Ask: "What design steps would you take to wrap a C API safely for the rest of a Swift codebase?"
  - Ask: "Why should most unsafe pointer manipulation be quarantined to a narrow layer?"
  - Exercise: "Design a Swifty wrapper for a C API that uses output pointers and manual cleanup."

- `E4. C++ interoperability`
  - Expect: Know that Swift can interoperate with C++, including important constraints around value/reference mapping, lifetime, view/reference-returning APIs, and emerging safe interop annotations.
  - Caveats: The feature is powerful but still evolving; not every C++ construct maps cleanly or safely; borrowed references and view types are especially dangerous if exposed without a wrapper layer.
  - Measure with:
  - Ask: "How does Swift generally import C++ value types, and why does that matter?"
  - Ask: "Why are C++ APIs that return references, views, or borrowed storage risky in Swift, and what do lifetime or escapability annotations buy you?"
  - Ask: "What kinds of C++ APIs are risky to expose directly to general Swift application code?"
  - Exercise: "Review a proposal to expose a C++ collection-heavy API directly to Swift app layers; explain what wrapper boundaries you would add."

- `E5. Modules, packages, targets, products, and resources`
  - Expect: Know SwiftPM structure, target boundaries, resource scoping, test support patterns, and package access control.
  - Caveats: Poor target factoring leaks dependencies and slows builds.
  - Measure with:
  - Ask: "What is the difference between a target and a product in SwiftPM?"
  - Ask: "Why should resources be scoped deliberately to the target that owns them?"
  - Exercise: "Split a monolithic package into targets for core logic, UI adapters, and test support without creating dependency cycles."

- `E6. Import visibility and dependency leakage`
  - Expect: Understand access-controlled imports and the general problem of leaking implementation-only dependencies into public APIs.
  - Caveats: Public API surfaces that mention dependency types effectively lock in that dependency.
  - Measure with:
  - Ask: "Why is import visibility part of API design and not just a build detail?"
  - Ask: "What happens if a public type signature exposes a type from an implementation dependency?"
  - Exercise: "Review a public SDK API that returns a concrete type from a third-party library; redesign it to avoid dependency leakage."

- `E7. Resilience, ABI, and module stability`
  - Expect: Know the practical distinction between source compatibility, ABI stability, and module stability.
  - Caveats: Many teams confuse these and make incorrect promises to clients.
  - Measure with:
  - Ask: "What is the difference between ABI stability and source compatibility?"
  - Ask: "Why does module stability matter for binary-distributed frameworks?"
  - Exercise: "You need to ship a binary SDK across Xcode versions; explain what guarantees matter and which changes are breaking."

- `E8. Binary frameworks, inlineability, and public implementation detail leakage`
  - Expect: Understand the cost of `@inlinable`, `@usableFromInline`, and exposing internals for performance.
  - Caveats: Performance annotations can freeze implementation detail into a public contract.
  - Measure with:
  - Ask: "Why is `@inlinable` not just a free performance win?"
  - Ask: "When is `@usableFromInline` the right compromise?"
  - Exercise: "Review a public utility package and decide which, if any, helpers should become inlineable."

### F. Tooling, Testing, Diagnostics, and Language-Adjacent Features

- `F1. Testing strategy in modern Swift`
  - Expect: Know XCTest and modern Swift Testing, async tests, actor-isolated tests, deterministic testing of concurrent code, and what to unit-test versus integration-test.
  - Caveats: Concurrency bugs often need design-for-testability, not just `sleep` and retries; proving a concurrent fix is correct usually requires testing cancellation, ordering, stale-result suppression, and isolation assumptions explicitly.
  - Measure with:
  - Ask: "How do you test async code without introducing flakiness?"
  - Ask: "How would you prove a concurrency fix is correct rather than merely less flaky?"
  - Ask: "What should an interview candidate do if a feature is hard to test because the design hides too much global state or concurrency?"
  - Exercise: "Write a test strategy for a debounced search pipeline with cancellation and stale-result suppression."

- `F2. Debugging Swift code with LLDB and compiler diagnostics`
  - Expect: Understand reading type errors, fix-its, isolation diagnostics, memory-graph clues, async stepping, task context, named-task debugging, and basic LLDB workflows.
  - Caveats: Senior engineers should not treat compiler diagnostics as noise; they encode the type and isolation model.
  - Measure with:
  - Ask: "How do you approach a 20-line generic or result-builder compiler error without thrashing?"
  - Ask: "How do async backtraces, task context, and task names help you debug a concurrent hang or stale-state bug?"
  - Ask: "What debugging signals suggest a retain cycle versus a data race versus a stale-state bug?"
  - Exercise: "Given an isolation diagnostic about crossing actor boundaries with a non-sendable type, explain how you would narrow the real issue."

- `F3. Performance investigation habits`
  - Expect: Use evidence: Instruments, allocation traces, time profiles, and targeted benchmarks rather than superstition.
  - Caveats: Many performance discussions in Swift are wrong because they assume cost instead of measuring it.
  - Measure with:
  - Ask: "What would you measure first before rewriting an abstraction-heavy hot path?"
  - Ask: "How do you decide whether a performance fix is worth the complexity cost?"
  - Exercise: "Outline a profiling plan for a scrolling hitch in a data-heavy SwiftUI/UIKit screen."

- `F4. Macros and compile-time code generation`
  - Expect: Know freestanding vs attached macros, additive expansion model, package support, and macro tradeoffs.
  - Caveats: Macros move complexity to compile time; they do not eliminate it.
  - Measure with:
  - Ask: "What kinds of boilerplate are good candidates for macros, and what kinds are better handled with normal code?"
  - Ask: "Why can macro-heavy codebases become harder to build, debug, or reason about?"
  - Exercise: "Evaluate whether a request/response endpoint definition should use a macro, a property wrapper, a protocol, or plain code."

- `F5. Language mode, build settings, and migration flags`
  - Expect: Know Swift language versioning, strict concurrency settings, default isolation settings, warning groups, and strict memory safety mode at a high level.
  - Caveats: Staff-level engineers should know which behavior comes from language mode, build settings, or framework behavior rather than mixing them together.
  - Measure with:
  - Ask: "What kinds of issues are surfaced by strict concurrency, default isolation, and strict memory safety, respectively?"
  - Ask: "Why is build-setting literacy important when debugging behavior differences between targets?"
  - Exercise: "Given two targets with different language settings, explain how you would prevent migration drift and inconsistent diagnostics."

- `F6. Observation, property wrappers, and language-adjacent state tools`
  - Expect: Know that some modern Apple-platform patterns are macro- and wrapper-driven, and understand the generated semantics rather than treating them as magic.
  - Caveats: Observation tools are easy to overtrust; generated code still obeys ownership, isolation, and mutation rules.
  - Measure with:
  - Ask: "What kinds of hidden coupling or update behavior can property-wrapper or observation-heavy code create?"
  - Ask: "Why is understanding generated semantics important even when a framework makes the syntax look trivial?"
  - Exercise: "Review an observable model with thread-sensitive state and explain where language-level isolation still needs to be explicit."
