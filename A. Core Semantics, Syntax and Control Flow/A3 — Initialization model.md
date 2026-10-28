---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - initialization
---

# A3 — Initialization Model

**Learning goals:**

- Trace stored-property initialization, permitted self access, and property-observer behavior.
- Explain designated, convenience, required, and failable class initializers.
- Predict memberwise initializer synthesis and access, including the Swift 6.4 private-property rule.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] and [[A2 — `let`, `var`, `mutating`, and method semantics]] for stored values, references, and mutation. [[A11 — Access Control]] covers module access levels.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An initializer must turn uninitialized storage into a valid instance before callers can use it. For a struct, every stored property must receive a value. For a class, initialization also has to establish the inherited part of the object before ordinary methods can rely on it.

**Definite initialization** is the compiler’s proof that storage is ready before a read. It permits some direct reads of already initialized stored fields while still rejecting a computed property or method that uses self before all fields are initialized. A **failable initializer** returns nil for recoverable invalid input; a throwing initializer can explain the failure. These are better input boundaries than a precondition when the caller is allowed to provide invalid data. [Initialization](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Initialize storage before using the whole instance

A direct field read can depend on an earlier assignment without calling through self.

```swift
struct Rectangle {
    let width: Int
    let height: Int
    let area: Int
    init(width: Int, height: Int) {
        self.width = width
        self.height = height
        self.area = self.width * self.height
    }
}
let rectangle = Rectangle(width: 3, height: 4)
precondition(rectangle.area == 12)
```

**Walkthrough:** width and height have values before area reads them. This is access to initialized stored fields, not permission to call any method before the instance is complete.

### Direct initialization does not invoke observers

A property observer such as didSet reacts to ordinary writes. Direct assignments during initialization establish initial state.

```swift
struct Counter {
    var changes = 0
    var value: Int { didSet { changes += 1 } }
    init() { value = 1 }
    mutating func update() { value = 2 }
}
var counter = Counter()
precondition(counter.changes == 0)
counter.update()
precondition(counter.changes == 1)
```

**Walkthrough:** The initializer’s assignment does not call didSet. update runs after initialization and performs an observed write. A helper called after storage is initialized can also make ordinary observed writes.

### Class delegation has an order

A designated initializer initializes the subclass’s own fields, calls super.init, then customizes inherited state. A convenience initializer delegates with self.init.

```swift
class Item {
    var title: String
    required init(title: String) { self.title = title }
}
final class Episode: Item {
    let minutes: Int
    init(title: String, minutes: Int) {
        self.minutes = minutes
        super.init(title: title)
        self.title += " episode"
    }
    required convenience init(title: String) { self.init(title: title, minutes: 0) }
    convenience init() { self.init(title: "Untitled", minutes: 0) }
}
let episode = Episode(title: "Swift", minutes: 20)
precondition(episode.title == "Swift episode" && episode.minutes == 20)
precondition(Episode().minutes == 0)
let factory: Item.Type = Episode.self
precondition(factory.init(title: "Factory") is Episode)
```

**Walkthrough:** minutes is ready before delegation. Inherited title is customized afterward. factory holds the class type Episode.self. required guarantees that this type can be constructed through Item.Type’s required initializer. Other initializers can have different implementations.

## Rules and pitfalls

### A computed read can use self too early

**Common pitfall:** Treating a computed property as though it were a direct initialized-field read.

**Mistake:** Compile-only; intentionally rejected.

```swift
struct Rectangle {
    let width: Int
    let area: Int
    var doubledWidth: Int { width * 2 }
    init(width: Int) {
        self.width = width
        self.area = doubledWidth
    }
}
```

**Consequence:** The compiler rejects using self before area has been initialized.

**Why:** The computed getter is a call on the instance. It is not analyzed as an unrestricted shortcut to the already initialized width field.

**Correction:** Compute from the input, then assign all storage.

```swift
struct Rectangle {
    let width: Int
    let area: Int
    var doubledWidth: Int { width * 2 }
    init(width: Int) {
        self.width = width
        self.area = width * 2
    }
}
precondition(Rectangle(width: 3).area == 6)
```

**What changes:** The calculation uses the initializer parameter and does not require an incompletely initialized self.

### Recoverable input needs a recoverable failure path

**Common pitfall:** Using a precondition to reject caller-supplied invalid duration. Compile-only; the invalid call would trap and is never run.

**Mistake:** Compile-only. The unsafe path is not executed.

```swift
struct Episode {
    let minutes: Int
    init(minutes: Int) {
        precondition(minutes >= 0)
        self.minutes = minutes
    }
}
func invalidInput() { _ = Episode(minutes: -1) }
```

**Consequence:** A negative duration terminates normal execution instead of giving the caller a result it can handle.

**Why:** A precondition expresses a programmer obligation, not an input-validation result.

**Correction:** Use a failable initializer when callers need only valid/invalid, or throw a domain error when they need a reason.

```swift
struct Episode {
    let minutes: Int
    init?(minutes: Int) {
        guard minutes >= 0 else { return nil }
        self.minutes = minutes
    }
}
precondition(Episode(minutes: -1) == nil)
precondition(Episode(minutes: 0)?.minutes == 0)
precondition(Episode(minutes: 20)?.minutes == 20)
```

**What changes:** Invalid data produces nil; zero and positive durations remain valid. The initializer now states the domain boundary explicitly.

### Subclass storage must precede superclass delegation

**Common pitfall:** Calling super.init before the subclass has initialized its own stored property.

**Mistake:** Compile-only; intentionally rejected.

```swift
class Item {
    var title: String
    init(title: String) { self.title = title }
}
final class Episode: Item {
    let minutes: Int
    init(title: String, minutes: Int) {
        super.init(title: title)
        self.minutes = minutes
    }
}
```

**Consequence:** The compiler rejects the delegation because minutes has not been initialized.

**Why:** Superclass initialization must not expose an object with uninitialized subclass storage. Inherited-property customization also belongs after super.init.

**Correction:** Initialize minutes first, delegate, then customize title.

```swift
class Item {
    var title: String
    init(title: String) { self.title = title }
}
final class Episode: Item {
    let minutes: Int
    init(title: String, minutes: Int) {
        self.minutes = minutes
        super.init(title: title)
        self.title += " episode"
    }
}
let episode = Episode(title: "Swift", minutes: 20)
precondition(episode.title == "Swift episode" && episode.minutes == 20)
```

**What changes:** The first phase establishes storage; the second permits ordinary customization of the initialized object.

## Worked example

### Preserving the memberwise initializer

**Setup:** Standalone script. A struct’s initializer in an extension can coexist with synthesized initialization.

**Predict:** Does adding init(square:) remove init(width:height:)?

```swift
struct Size {
    let width: Int
    let height: Int
}
extension Size {
    init(square side: Int) { self.init(width: side, height: side) }
}
let rectangle = Size(width: 3, height: 4)
let square = Size(square: 5)
precondition(rectangle.width == 3 && rectangle.height == 4)
precondition(square.width == 5 && square.height == 5)
```

**Outcome:** Both calls compile; the first uses synthesis and the second delegates to it.

**Why:** A custom initializer in the primary struct declaration suppresses memberwise synthesis. An extension initializer preserves it. Synthesized access is not automatically public.

## Interview questions

### Why can an initializer read one stored field but not freely use self, and when do observers run?

**Answer:** A direct read can use storage already known to be initialized. A computed getter or method requires a sufficiently initialized self. Direct initialization assignments skip observers; ordinary writes after initialization can invoke them.

**Example:** Standalone runnable example.

```swift
struct Sample {
    var observations = 0
    var first: Int { didSet { observations += 1 } }
    let second: Int
    init(_ input: Int) {
        first = input
        second = first + 1
        adjust()
    }
    mutating func adjust() { first += 1 }
}
let sample = Sample(3)
precondition(sample.first == 4 && sample.second == 4)
precondition(sample.observations == 1)
```

**Walkthrough:** second reads initialized storage. Once all fields have values, adjust performs an ordinary mutation and didSet counts it. The direct first assignment did not count.

**Boundary:** Do not depend on observers to establish initial invariants; initialize those explicitly. Property wrappers add their own initialization rules (B15).

### When is a memberwise initializer synthesized, and what access does it have?

**Answer:** A struct without a custom initializer in its primary declaration receives memberwise synthesis. Extension initializers preserve it. Its visibility follows stored-property and initializer access rules; a public struct does not automatically receive a public memberwise initializer.

**Example:** Standalone runnable example.

```swift
struct User {
    var name: String
    var attempts = 0
}
extension User {
    init(guest: Bool) { self.init(name: guest ? "Guest" : "Member") }
}
let member = User(name: "Ada", attempts: 2)
let guest = User(guest: true)
precondition(member.attempts == 2)
precondition(guest.name == "Guest" && guest.attempts == 0)
```

**Walkthrough:** The stored default lets callers omit attempts. The extension delegates to the generated initializer.

**Boundary:** Library clients need an explicit public initializer when construction is public API. Swift 6.4 changes the treatment of initialized private properties, demonstrated below; it does not make synthesized initializers public.

### How do designated, convenience, required, and failable initializers differ?

**Answer:** Designated initializers establish a class’s stored state and delegate upward. Convenience initializers delegate within the class. required preserves a construction requirement in subclasses. Failable initialization represents invalid input with nil.

**Example:** Standalone runnable example.

```swift
class Named {
    let name: String
    required init?(name: String) {
        guard !name.isEmpty else { return nil }
        self.name = name
    }
    convenience init?() { self.init(name: "Guest") }
}
final class Member: Named {
    let points: Int
    required init?(name: String) {
        points = 0
        super.init(name: name)
    }
}
let type: Named.Type = Member.self
precondition(type.init(name: "") == nil)
precondition(type.init(name: "Ada") is Member)
precondition(Named()?.name == "Guest")
```

**Walkthrough:** Member initializes its own points before super.init. Failure in the superclass propagates. The metatype call is valid because the initializer is required.

**Boundary:** Initializer inheritance is conditional, not blanket inheritance of every convenience signature. A class initializer cannot use ordinary self methods before its initialization phase permits it.

### Extension: what changes for initialized private stored properties in Swift 6.4 memberwise synthesis?

**Answer:** Initialized private properties can be omitted from the accessible memberwise signature instead of needlessly making all construction private. Uninitialized private storage still needs a value and can still restrict the generated initializer.

**Example:** Standalone runnable example.

```swift
struct Profile {
    var name: String
    private var cache = 0
    private var token: Int?
    var cached: Int { cache }
    static func seeded() -> Profile { Profile(name: "Ada", cache: 7, token: nil) }
}
let plain = Profile(name: "Ada")
precondition(plain.cached == 0)
precondition(Profile.seeded().cached == 7)
```

**Walkthrough:** The accessible call supplies name and leaves private state at its initial values, including implicit nil. The private compatibility form remains usable inside Profile. [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)

**Boundary:** Do not generalize this to every private property or wrapper. A matching hand-written extension initializer can now collide with the new synthesized signature; a bare Profile.init reference can also select a different arity. [[A11 — Access Control|A11]] demonstrates construction across a module boundary.

Separate **intentionally rejected** example: uninitialized private storage still has to be supplied, and the resulting initializer is private outside the type.

```swift
struct Secret {
    var name: String
    private var token: Int
}
let secret = Secret(name: "Ada", token: 7)
```

## Focused exercise

**Task:** Initialize a subclass with its own duration and an inherited title, plus a convenience construction path. Separate recoverable invalid input from object initialization order.

**Acceptance checks:**

- Negative duration fails; zero and positive durations succeed.
- Subclass duration is initialized before super.init; the inherited title is customized afterward.
- A convenience initializer delegates instead of duplicating validation.

**Solution:** Standalone script; a throwing initializer reports which input was invalid.

```swift
enum EpisodeError: Error { case emptyTitle, negativeDuration }
class Item {
    var title: String
    init(title: String) { self.title = title }
}
final class Episode: Item {
    let minutes: Int
    init(title: String, minutes: Int) throws(EpisodeError) {
        guard !title.isEmpty else { throw .emptyTitle }
        guard minutes >= 0 else { throw .negativeDuration }
        self.minutes = minutes
        super.init(title: title)
        self.title += " episode"
    }
    convenience init(shortTitle title: String) throws(EpisodeError) {
        try self.init(title: title, minutes: 0)
    }
}
let short = try Episode(shortTitle: "Swift")
let long = try Episode(title: "Swift", minutes: 20)
precondition(short.minutes == 0 && long.minutes == 20)
precondition(long.title == "Swift episode")
do {
    _ = try Episode(title: "Swift", minutes: -1)
    preconditionFailure("Expected validation failure")
} catch { precondition(error == .negativeDuration) }
do {
    _ = try Episode(title: "", minutes: 0)
    preconditionFailure("Expected validation failure")
} catch { precondition(error == .emptyTitle) }
```

**Reasoning:** The designated initializer validates input, initializes minutes, delegates to Item, and then customizes inherited title. The rejected ordering in Rules and pitfalls explains why these steps cannot be freely rearranged. The convenience path delegates with a zero duration, so it cannot accidentally skip validation. Throwing communicates a reason; a failable initializer would be sufficient for a simple optional parse.

## In practice

**Choose it when:** Use initialization to establish domain validity and expose only the construction paths callers should rely on.

**Investigate by:** Follow each assignment and delegation in order; distinguish a stored read from a getter call. Inspect the generated module interface for public construction.

**Strong evidence:** Test invalid and boundary inputs, subclass construction obligations, and ordinary external clients rather than relying only on same-file or @testable access.

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
