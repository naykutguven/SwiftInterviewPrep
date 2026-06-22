---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - mutability
---
## 0. Rubric snapshot

**Rubric expectation**

Understand how mutability is expressed for value and reference types, including `mutating` and `nonmutating`. The rubric specifically asks you to reason about `let`, `var`, `mutating`, class reference storage, `Sendable`, and intentionally `nonmutating` setters.

**Caveats**

- A `nonmutating` setter or method can still mutate through reference storage.
- Protocol requirements can force awkward mutability shapes.
- Mutability choices affect synthesized conformances and whether a type is safe to share across concurrency boundaries.
- The answer format follows the project’s Swift section template.

**You should be able to answer**

- Why does a struct method that changes stored properties need `mutating`, but a class method does not?
- What does `let` mean for a struct instance versus a class instance?
- How can one mutable reference-typed stored property change a type’s `Sendable` story or make a synthesized conformance less meaningful?

**You should be able to do**

- Design a property wrapper or computed property whose setter is intentionally `nonmutating`.
- Explain the code probe where a `let` struct instance appears to mutate successfully.

---

## 1. Core mental model

Swift’s mutability model is not “can this object change?” It is “does this operation require write access to this binding’s value?” That distinction is the entire topic.

For a value type, such as a `struct` or `enum`, changing a stored property means changing the value itself. A method that does this must be marked `mutating`, because calling it requires write access to `self`. Swift’s official documentation describes `mutating` methods as opt-in behavior that lets a structure or enumeration modify its properties or even assign a new value to `self`. ([Swift.org, "Methods"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/methods/))

For a class, the variable stores a reference to an object. Calling a method that changes the object’s stored properties does not change the reference stored in the local binding. This is why a class method does not need `mutating`. A `let` class binding freezes the reference, not the object behind the reference. Swift’s basics documentation defines constants as bindings whose value can’t be changed, while variables can be set to a different value later. ([Swift.org, "The Basics"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/thebasics/))

A computed property setter is normally mutating for value types, because setting a property usually modifies `self`. But Swift allows an explicit `nonmutating set`. That says: “assigning this property does not require write access to the value itself.” It can still mutate something else, such as heap storage referenced by the value. Swift’s grammar explicitly allows mutation modifiers such as `mutating` and `nonmutating` on setters. ([Swift.org, "Summary of the Grammar"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/summaryofthegrammar/))

The key idea:

```text
let/var controls write access to the binding’s value.
mutating/nonmutating controls whether a member needs write access to self.
Reference storage can change behind an otherwise immutable value.
```

Swift guarantees that `mutating` operations on value types require mutable access to the value. Swift does **not** guarantee deep immutability just because something is bound with `let`.

---

## 2. Essential mechanics

### Value type methods are nonmutating by default

A `struct` method cannot change stored properties unless it is marked `mutating`.

```swift
struct Counter {
    private(set) var value = 0

    mutating func increment() {
        value += 1
    }
}

var counter = Counter()
counter.increment()
```

This works because `counter` is a `var`. If you change it to `let`, Swift rejects the call:

```swift
struct Point {
    var x = 0
    mutating func moveRight() { x += 1 }
}

let point = Point()
point.moveRight()
```

Compiler error:

```text
error: cannot use mutating member on immutable value: 'point' is a 'let' constant
note: change 'let' to 'var' to make it mutable
```

A `mutating` method is conceptually allowed to replace the entire value:

```swift
struct Toggle {
    var isOn = false

    mutating func reset() {
        self = Toggle()
    }
}
```

That is why Swift treats it as requiring write access to the whole value, not merely to one field.

### Class methods mutate through identity

Classes have identity. A class variable holds a reference. Mutating a property through that reference does not replace the reference itself.

```swift
final class Counter {
    var value = 0

    func increment() {
        value += 1
    }
}

let counter = Counter()
counter.increment()

print(counter.value) // 1
```

This compiles because `let counter` means the binding cannot be reassigned:

```swift
counter = Counter() // invalid
```

It does **not** mean the object’s internal state is frozen:

```swift
counter.value = 10 // valid
```

This is one of the most common Swift interview traps: `let` on a class reference is not deep immutability.

### `nonmutating set` means the setter does not mutate `self`

A computed property on a value type can opt into a `nonmutating` setter:

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
```

The setter does not assign to `self` or any stored property of the struct. It mutates `storage.value`, where `storage` is a reference to a heap object.

That means this is legal:

```swift
let counter = Counter()
counter.value = 3
print(counter.value) // 3
```

The value’s stored property `storage` is still the same reference. The object behind it changed.

### Protocol requirements can shape mutability

Protocols can require mutating behavior:

```swift
protocol Resettable {
    mutating func reset()
}

struct FormState: Resettable {
    var text = ""

    mutating func reset() {
        text = ""
    }
}

final class FormController: Resettable {
    var text = ""

    func reset() {
        text = ""
    }
}
```

The struct needs `mutating`. The class does not, because classes mutate through reference identity. Swift’s protocol documentation calls out `mutating` method requirements for methods that modify value-type instances. ([Swift.org, "Protocols"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/))

Protocols can also force awkward designs around setters:

```swift
protocol DraftStore {
    var draft: String { get nonmutating set }
}
```

This says conforming value types must allow setting `draft` without requiring mutable `self`. That usually implies reference-backed storage, external storage, or some framework-managed mechanism.

### Property wrappers can hide the same mechanism

A property wrapper can synthesize backing storage for a property. Swift documentation states that wrapper storage is synthesized using an underscore-prefixed property, such as `_someProperty`. ([Swift.org, "Attributes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/))

A wrapper can also expose a `nonmutating` `wrappedValue` setter:

```swift
@propertyWrapper
struct Boxed<Value> {
    final class Storage {
        var value: Value

        init(_ value: Value) {
            self.value = value
        }
    }

    private let storage: Storage

    var wrappedValue: Value {
        get { storage.value }
        nonmutating set { storage.value = newValue }
    }

    init(wrappedValue: Value) {
        self.storage = Storage(wrappedValue)
    }
}

struct DraftViewModel {
    @Boxed var draft = ""
}

let model = DraftViewModel()
model.draft = "Hello"

print(model.draft) // Hello
```

This is mechanically similar to the code probe. It can be useful, but it is also easy to abuse.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `let` means deep immutability

Bad:

```swift
final class Session {
    var token: String?

    init(token: String?) {
        self.token = token
    }
}

let session = Session(token: nil)
session.token = "abc" // Valid
```

`let` freezes the local binding `session`; it does not freeze the instance.

Better:

```swift
struct Session: Sendable {
    let token: String?
}
```

Or, if identity and mutation are required:

```swift
actor SessionStore {
    private var token: String?

    func updateToken(_ token: String?) {
        self.token = token
    }

    func currentToken() -> String? {
        token
    }
}
```

For shared mutable state, `actor` is often a clearer concurrency boundary than “immutable-looking” class references.

### Trap 2: Assuming a struct is automatically value-semantic

A struct containing a mutable reference may behave like shared mutable state:

```swift
struct Counter {
    final class Storage {
        var value = 0
    }

    let storage = Storage()
}

let a = Counter()
let b = a

a.storage.value = 1
print(b.storage.value) // 1
```

The struct was copied, but the reference inside it was copied too. Both values point to the same `Storage`.

A better value-semantic version:

```swift
struct Counter: Sendable, Equatable {
    private(set) var value = 0

    mutating func increment() {
        value += 1
    }
}
```

This version has obvious mutation semantics and supports structural reasoning.

### Trap 3: Hiding shared mutable state behind `nonmutating`

This is legal:

```swift
struct Preferences {
    private final class Storage {
        var isEnabled = false
    }

    private let storage = Storage()

    var isEnabled: Bool {
        get { storage.isEnabled }
        nonmutating set { storage.isEnabled = newValue }
    }
}
```

But it is surprising because this works:

```swift
let preferences = Preferences()
preferences.isEnabled = true
```

That should make you ask:

```text
Is this intentionally reference-backed state?
Is sharing across copies intended?
Is it thread-safe?
Does this still deserve to look like a value type?
```

### Trap 4: Accidentally breaking `Sendable`

Apple defines `Sendable` as a protocol for thread-safe types whose values can be shared across concurrent contexts without introducing data races. ([Apple Developer, "Sendable"](https://developer.apple.com/documentation/Swift/Sendable))

This looks harmless:

```swift
struct Counter: Sendable {
    private final class Storage {
        var value = 0
    }

    private let storage = Storage()
}
```

But in Swift 6 mode, this is rejected:

```text
error: stored property 'storage' of 'Sendable'-conforming struct 'Counter' has non-Sendable type 'Counter.Storage'
note: class 'Storage' does not conform to the 'Sendable' protocol
```

The problem is not that classes are always forbidden. The problem is that mutable class storage is not automatically safe to transfer across concurrency domains.

---

## 4. Direct answers to rubric questions

### Q1. Why does a struct method that changes stored properties need `mutating`, but a class method does not?

A struct method that changes stored properties changes the value itself, so it needs write access to `self`. A class method changes the object behind a reference, not the reference binding itself, so the method does not need `mutating`.

Interview version:

> For value types, mutation means changing the value, so Swift requires the method to be marked `mutating` and the receiver must be a `var`. For classes, the binding stores a reference. A method can mutate the referenced object without changing the reference stored in the binding, so class methods do not need `mutating`. That distinction is why `let` behaves very differently for structs and classes.

### Q2. What does `let` mean for a struct instance versus a class instance?

For a struct, `let` makes the whole value immutable through that binding. You cannot assign to stored properties or call `mutating` methods. For a class, `let` makes the reference immutable; you cannot reassign the binding, but you can still mutate the object’s `var` properties.

```swift
struct UserValue {
    var name: String
}

let value = UserValue(name: "A")
// value.name = "B" // invalid
```

```swift
final class UserObject {
    var name: String

    init(name: String) {
        self.name = name
    }
}

let object = UserObject(name: "A")
object.name = "B" // valid
```

Interview version:

> `let` freezes the binding’s value. For a struct, the binding’s value is the whole struct, so stored-property mutation and mutating methods are blocked. For a class, the binding’s value is the reference, so reassignment is blocked, but the object behind the reference can still mutate. `let` is not a deep immutability feature.

### Q3. How can one mutable reference-typed stored property change a type’s `Sendable` story or make a synthesized conformance less meaningful?

A value type with mutable reference storage may no longer be safely shareable across concurrency domains, because copying the value can still share the same mutable object. This can block `Sendable` synthesis or make explicit `Sendable` conformance invalid unless the reference storage is itself safely synchronized or immutable. Since `Sendable` is about safe cross-concurrency transfer, hidden mutable references are a serious design issue. ([Apple Developer, "Sendable"](https://developer.apple.com/documentation/Swift/Sendable))

It can also make synthesized conformances misleading. A struct may look value-semantic, but if equality, hashing, or encoding includes a reference-backed object, the synthesized behavior may reflect storage mechanics rather than domain semantics.

Bad:

```swift
struct Profile {
    final class Storage {
        var displayName: String

        init(displayName: String) {
            self.displayName = displayName
        }
    }

    let storage: Storage
}
```

Better:

```swift
struct Profile: Sendable, Equatable, Codable {
    var displayName: String
}
```

Or, if shared mutable identity is required:

```swift
actor ProfileStore {
    private var displayName: String

    init(displayName: String) {
        self.displayName = displayName
    }

    func updateDisplayName(_ displayName: String) {
        self.displayName = displayName
    }

    func snapshot() -> String {
        displayName
    }
}
```

Interview version:

> One mutable reference inside a struct can destroy the assumptions people usually make about value types. Copies may share state, `let` may not mean observable immutability, synthesized equality may describe storage rather than domain identity, and `Sendable` may fail because the hidden class is not safe to transfer across concurrency domains. At senior level, I would either keep the type truly value-semantic, isolate the mutable reference behind an actor or lock, or make the reference semantics explicit in the API.

---

## 5. Code probe

Given:

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

### What happens?

```text
3
```

### Why?

`counter` is a `let` constant, so the `Counter` value itself cannot be mutated. But `value` is a computed property whose setter is explicitly marked `nonmutating`.

That setter does not change `self.storage`. It does not assign a new `Storage` reference. It only mutates the object pointed to by `storage`.

```text
counter: Counter
└── storage: reference ──────────────┐
                                     │
heap object: Counter.Storage         │
└── value: 0  ── set to 3 ◀──────────┘
```

The binding remains the same. The struct’s stored property remains the same reference. The heap object’s internal state changes.

This is legal because the setter promises it does not need write access to `self`. It is surprising because the syntax looks like ordinary mutation of a `let` value.

### Fix or redesign

If the type should behave like a normal value type, use stored value state and a `mutating` method:

```swift
struct Counter: Sendable, Equatable {
    private(set) var value = 0

    mutating func setValue(_ value: Int) {
        self.value = value
    }

    mutating func increment() {
        value += 1
    }
}

var counter = Counter()
counter.setValue(3)

print(counter.value) // 3
```

### Why this fix is correct

This version aligns the API with the semantics:

```text
Changing the counter changes the value.
Changing the value requires var.
Copies are independent.
Sendable and Equatable remain structurally meaningful.
```

It is less magical and easier to reason about in interviews, production code, and concurrency migrations.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Plain value type with `mutating` methods|Default choice for domain models, state snapshots, reducers, request states|Requires `var` at mutation sites|
|Reference type|Real identity is required, such as controllers, shared services, object graphs|`let` does not imply immutable state; concurrency needs care|
|Actor|Shared mutable state crosses concurrency domains|Async access and actor isolation affect API shape|
|Reference-backed struct with `nonmutating set`|Framework-managed state, SwiftUI-style storage, copy handles, bindings|Surprising semantics; can harm `Sendable` and value reasoning|
|Property wrapper with `nonmutating wrappedValue`|You intentionally want value syntax over external/reference storage|Can hide lifecycle, threading, and ownership complexity|

---

## 6. Exercise

### Problem

Design a property wrapper or computed property whose setter is intentionally `nonmutating`, and explain why.

### Bad / naive version

```swift
@propertyWrapper
struct UnsafeShared<Value> {
    final class Storage {
        var value: Value

        init(_ value: Value) {
            self.value = value
        }
    }

    private let storage: Storage

    var wrappedValue: Value {
        get { storage.value }
        nonmutating set { storage.value = newValue }
    }

    init(wrappedValue: Value) {
        self.storage = Storage(wrappedValue)
    }
}

struct Settings {
    @UnsafeShared var username = ""
}

let settings = Settings()
settings.username = "aykut"
```

### What is wrong?

```text
The wrapper makes a let value appear mutable.
Copies of Settings share the same Storage.
There is no synchronization.
Sendable is not safe.
The API does not communicate reference semantics.
```

This might be fine for a tiny demo, but it is dangerous as general-purpose production infrastructure.

### Improved version: explicit reference-backed handle

```swift
@propertyWrapper
struct SharedValue<Value> {
    final class Storage {
        private var value: Value

        init(_ value: Value) {
            self.value = value
        }

        func get() -> Value {
            value
        }

        func set(_ newValue: Value) {
            value = newValue
        }
    }

    private let storage: Storage

    var wrappedValue: Value {
        get { storage.get() }
        nonmutating set { storage.set(newValue) }
    }

    var projectedValue: Storage {
        storage
    }

    init(wrappedValue: Value) {
        self.storage = Storage(wrappedValue)
    }
}

struct Draft {
    @SharedValue var text = ""
}

let draft = Draft()
draft.text = "Hello"

print(draft.text) // Hello
```

### Why this is better

It still uses a `nonmutating` setter, but the design is now honest about being reference-backed:

```text
The wrapper name says “shared”.
The projected value exposes that storage exists.
The type should not be treated as pure value-semantic.
It can be evolved toward locking, actor isolation, or framework-managed storage.
```

However, this is still not automatically thread-safe. If it must cross concurrency domains, you need stronger design.

### Production-grade actor-isolated version

```swift
actor DraftStore {
    private var text = ""

    func updateText(_ text: String) {
        self.text = text
    }

    func currentText() -> String {
        text
    }
}
```

This is usually the better choice when the state is actually shared and mutable across async boundaries.

### SwiftUI-style justification

SwiftUI’s `@State` is a real-world example of state stored outside the immediate view value. Apple describes `@State` as a single source of truth for a value stored in a view hierarchy and recommends declaring it private to avoid conflicting with SwiftUI’s storage management. ([Apple Developer, "State"](https://developer.apple.com/documentation/swiftui/state))

That is a valid reason to use non-obvious storage semantics: the framework owns the lifecycle and update model. In your own code, you need the same level of justification.

---

## 7. Production guidance

Use this in production when:

```text
The type is genuinely value-semantic: use struct + var + mutating.
The type has real identity: use class, but make reference semantics explicit.
The state is shared across concurrency domains: prefer actor or explicit synchronization.
The property is framework-managed or externally stored: a nonmutating setter may be justified.
```

Be careful when:

```text
A struct contains a class.
A let value can still produce observable mutation.
A property wrapper hides storage or lifecycle behavior.
A type claims Sendable but stores mutable reference state.
Synthesized Equatable, Hashable, Codable, or Sendable no longer matches domain semantics.
```

Avoid when:

```text
nonmutating set is used just to avoid changing let to var.
A value type secretly shares mutable state across copies.
A property wrapper hides thread-safety problems.
A class is used only because mutating value semantics were not understood.
```

Debugging checklist:

```text
Is this mutation changing the value, or an object referenced by the value?
Is the receiver a let or var binding?
Is the member mutating or nonmutating?
Does a property wrapper synthesize hidden storage?
Do copies share reference-backed storage?
Would this type safely conform to Sendable in Swift 6?
Are synthesized conformances still semantically correct?
Should this be a struct, class, actor, or explicit handle type?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Structs need `mutating` to change properties. Classes do not. `let` means constant and `var` means variable.

This is true but too shallow.

### Senior answer

> For value types, mutation changes the value itself, so Swift requires `mutating` and a `var` receiver. For classes, mutation happens through a reference, so `let` freezes the reference but not the object. A `nonmutating` setter on a value type can still mutate reference storage, which is legal but potentially surprising.

This is a good senior answer.

### Staff-level answer

> I treat mutability as an API contract, not just syntax. A value type with `mutating` methods communicates local, explicit value mutation. A class communicates identity and shared mutable state. A `nonmutating` setter on a struct is a specialized tool for reference-backed or framework-managed storage, but it can break value-semantic expectations, make `let` misleading, block `Sendable`, and make synthesized conformances semantically wrong. In production, I would either preserve true value semantics, make identity explicit with a class, or isolate shared mutable state behind an actor or a narrow synchronization boundary.

That is the level expected for staff-level Swift design.

Staff-level questions to ask:

```text
Does this type promise value semantics or identity?
Do copies share mutable state?
Does let imply observable immutability for users of this API?
Would this type pass Swift 6 Sendable checking?
Are synthesized conformances domain-correct or merely structural?
Should mutation be local, reference-backed, actor-isolated, or framework-managed?
Is a nonmutating setter communicating a real abstraction or hiding a design smell?
```

---

## 9. Interview-ready summary

Swift separates binding mutability from object mutability. For structs and enums, changing stored properties changes the value, so the method must be `mutating` and the receiver must be a `var`. For classes, a `let` binding freezes the reference, not the object, so class methods can mutate internal state without `mutating`. A `nonmutating` setter on a value type is legal when setting the property does not require write access to `self`, but it can still mutate reference storage behind the value. That is powerful for framework-managed or reference-backed storage, but dangerous when it makes value types look immutable, breaks `Sendable`, or makes synthesized conformances misleading.

---

## 10. Flashcards

Q: Why does a struct method need `mutating` to change a stored property?  
A: Because changing a stored property changes the value itself, so the method needs write access to `self`.

Q: Why does a class method not need `mutating`?  
A: Because the binding stores a reference; the method mutates the object behind the reference, not the reference value itself.

Q: What does `let` mean for a struct?  
A: The entire value is immutable through that binding; stored properties cannot be assigned and `mutating` methods cannot be called.

Q: What does `let` mean for a class?  
A: The reference cannot be reassigned, but the referenced object can still mutate.

Q: What does `nonmutating set` mean?  
A: The setter does not require write access to `self`; it may still mutate external or reference-backed storage.

Q: Why can a `let` struct instance with a `nonmutating` computed property setter still appear mutable?  
A: The setter mutates something behind the value, such as a class instance, without changing the struct’s stored properties.

Q: Why can a mutable class stored inside a struct hurt `Sendable`?  
A: Copies of the struct can share mutable reference state, which may not be safe to transfer across concurrency domains.

Q: When is `nonmutating set` justified?  
A: When the property is intentionally backed by external, shared, or framework-managed storage and the API clearly communicates that behavior.

---

## 11. Related sections

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A15 — Type choices: struct, class, enum, protocol, and actor]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[B13 — Property wrappers and generated storage semantics]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D12 — Synchronization library, mutexes, and choosing between actors and locks]]

---

## 12. Sources

- "Swift Senior/Staff Rubric and Prioritized Study Checklist."
- Swift.org. "The Basics." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/thebasics/
- Swift.org. "Methods." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/methods/
- Swift.org. "Protocols." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/
- Swift.org. "Summary of the Grammar." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/summaryofthegrammar/
- Swift.org. "Attributes." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/
- Apple Developer. "Sendable." Apple Developer Documentation. https://developer.apple.com/documentation/Swift/Sendable
- Apple Developer. "State." Apple Developer Documentation. https://developer.apple.com/documentation/swiftui/state
