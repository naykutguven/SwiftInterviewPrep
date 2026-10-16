---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - initialization
---

# A3 — Initialization Model

**Goal:** Construct a subclass in the required initialization order and explain an invalid ordering.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Initialization establishes storage before general instance use. Domain validity still needs explicit checks. Class construction proceeds through inherited layers: each designated initializer initializes its own stored properties, delegates upward, then customizes inherited state. Only after the required initialization can code freely use self or dispatch through instance methods. [Initialization](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/)

## Rules and pitfalls

### Respect the layer that owns storage

A subclass designated initializer initializes new storage before calling a superclass designated initializer. A convenience initializer delegates through self.init before ordinary customization. Use required when subclasses must preserve a construction entry point; it is an obligation, not automatic validation.

**Common pitfall:** Calling super.init first leaves subclass storage uninitialized when superclass construction could interact with the instance. Establish the subclass layer first, then delegate.

### Keep synthesis separate from public construction

A synthesized memberwise initializer is a convenience for structs, not a stable public API. Defining an initializer in the primary declaration suppresses memberwise synthesis; placing an additional initializer in an extension can preserve it. Write a deliberate public initializer when clients need one.

**Common pitfall:** Relying on synthesis makes a storage edit change construction access or parameters. Verify call sites when adding private state and validate domain invariants inside explicit initializers.

## Interview questions

### How does adding a private initialized cache differ from adding a private property that requires a value?

In Swift 6.4, an initialized private cache can be excluded from a more accessible memberwise initializer. A private property requiring a value must participate and can restrict initializer access. A private optional can have implicit nil initialization, so test its synthesized surface rather than assuming every private field creates the same parameter.

### Why can a direct stored-field read succeed where a computed-property read fails during initialization? When can an observer run from an initializer?

Definite initialization tracks direct stored-field accesses individually, but a computed property can access arbitrary self and requires the whole instance to be ready. Direct initial storage writes skip observers; an ordinary helper mutation after initialization, or inherited-property customization after super.init, can invoke observers.

### Why can't you access `self` freely before all stored properties are initialized?

Uninitialized storage must not be read, escape, or be reached by dynamically dispatched code. Permitting unrestricted self access would expose an incomplete object. A direct read of an already initialized stored field is a narrower operation and can be allowed where invoking an instance accessor is not.

### When does Swift synthesize a memberwise initializer, and when does it disappear?

Structs synthesize memberwise initialization when eligible and no initializer in the primary declaration suppresses it. Stored-property access and defaults shape the signature; extension initializers can preserve it. Synthesis does not create a public constructor and should not substitute for a documented client contract.

### In what order must a designated subclass initializer initialize its own stored properties, call super.init, and modify inherited properties? How does a convenience initializer delegate through self.init?

Initialize the subclass's own stored properties, call super.init, then modify inherited mutable properties. A convenience initializer delegates to self.init and eventually reaches a designated initializer. It does not jump directly to super.init or bypass the layer's construction invariants.

## Focused exercise

**Task:** Initialize a subclass with one new and one inherited property. Show valid delegation order and explain a rejected ordering.

**Acceptance checks:**

- The subclass stores duration before delegation.
- Inherited title changes after super.init; convenience construction reaches the designated path.
- Zero duration is accepted; negative input intentionally violates the stated precondition.

**Solution:**

Standalone class-hierarchy probe:

```swift
class Item {
    var title: String
    init(title: String) { self.title = title }
}
final class Episode: Item {
    let duration: Int
    init(title: String, duration: Int) {
        precondition(duration >= 0)
        self.duration = duration
        super.init(title: title)
        self.title += " episode"
    }
    convenience init(duration: Int) {
        self.init(title: "Untitled", duration: duration)
    }
}
let episode = Episode(duration: 0)
precondition(episode.duration == 0 && episode.title == "Untitled episode")
```

Move super.init before self.duration in a separate compile-only variant: it must reject because subclass storage is not initialized.

**Reasoning:** The initializer creates a fully initialized object and separately enforces a duration domain. Use a failable or throwing initializer when invalid external input is recoverable; a precondition fits this fixture's programmer contract.

## In practice

**Choose it when:** Use explicit initializers for invariants and stable construction surfaces.

**Investigate by:** Reduce a diagnostic to one stored property and one delegation step; compare direct storage with computed access.

**Strong evidence:** Evolve private storage while preserving supported public construction and testing the rejected initialization paths.

### Memberwise synthesis detail — optional

#### Memberwise initialization is synthesized for structs, with caveats

Swift gives structures a memberwise initializer by default. Stored properties with default values can be omitted from the memberwise call. ([Swift.org, "Initialization"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/))

```swift
struct Size {
    var width = 0.0
    var height = 0.0
}

let explicit = Size(width: 100, height: 200)
let defaulted = Size(height: 200)
```

The synthesized memberwise initializer disappears when you declare a custom initializer inside the main struct declaration:

```swift
struct User {
    let id: Int
    let name: String

    init(id: Int) {
        self.id = id
        self.name = "Guest"
    }
}

let user = User(id: 1, name: "Aykut")
```

Swift 6.4 rejects this example: the custom initializer removes memberwise synthesis, so the name argument is extra.

To keep the synthesized memberwise initializer, put custom initializers in an extension:

```swift
struct User {
    let id: Int
    let name: String
}

extension User {
    init(id: Int) {
        self.init(id: id, name: "Guest")
    }
}

let named = User(id: 1, name: "Aykut")
let guest = User(id: 2)
```

Access control still matters: synthesis never automatically creates a public initializer. Swift 6.4 excludes initializable properties that have initial values and are less accessible than the maximum candidate access level, capped at `internal`. Implicitly nil-initialized optional properties count as initialized.

```swift
struct Profile {
    var name: String
    private var cache = 0
    private var token: Int?
}
let profile = Profile(name: "Ada")
print(profile.name) // Ada
```

A private property requiring a value is different:

```swift
struct Secret {
    var name: String
    private var token: Int
}
let value = Secret(name: "Ada", token: 1) // expected inaccessible-initializer error
```

Both outcomes were checked in Swift 6.4. A type with only equally private properties does not acquire an internal memberwise initializer from this rule.

**Migration:** a compatibility overload preserves applicable old signatures, but unbound `Type.init` references and extension initializers can still need review. Check the actual overload set when upgrading a model or macro-generated storage. [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)

The ordinary-property compatibility overload remains usable from an allowed access context:

```swift
struct Cache {
    var name: String
    private var count = 0
    var cached: Int { count }

    static func seeded() -> Cache {
        Cache(name: "Ada", count: 7)
    }
}
print(Cache(name: "Ada").cached, Cache.seeded().cached) // 0 7
```

A bare Cache.init reference can select the new shorter signature. An extension already declaring that signature can now collide with synthesis; both cases were checked. Private initialized property wrappers had their own omission behavior before 6.4, so the ordinary-property compatibility rule must not be mechanically applied to them; see B13. A separate default initializer can also exist when every stored property has a default: that does not make the private memberwise signature accessible.

## References

### Related guides

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A4 — Optionals and Nil Modeling]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B13 — Property Wrappers and Generated Storage Semantics]]
- [[C1 — ARC fundamentals]]

### Official sources

- [Initialization](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/)
- [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)
- [Swift.org. "Declarations." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/)
- [Swift.org. "Access Control." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/)
- [Swift.org. "Protocols." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [Apple Developer. "Swift."](https://developer.apple.com/swift/)
