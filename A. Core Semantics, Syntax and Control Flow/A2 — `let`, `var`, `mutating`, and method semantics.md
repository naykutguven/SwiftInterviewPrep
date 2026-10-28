---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - mutability
---

# A2 — `let`, `var`, `mutating`, and Method Semantics

**Learning goals:**

- Explain what let prevents for value and reference types, and when a value method must be mutating.
- Recognize reference-backed nonmutating setters and the limits of synthesized conformances.
- Apply weak-let and modern borrow/mutate accessor rules without confusing them with synchronization.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] for values, references, and aliases. The extension uses [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]/[[C8 — Copyable, noncopyable types, and `~Copyable`]] ownership syntax.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

When a declaration says let, ask what is held in that binding. A let struct cannot have its stored value changed through the binding. A let class reference cannot be replaced, but the referenced object may still have mutable properties. var permits changing the binding’s value; it does not grant access that an API otherwise hides.

For a value type, a method marked **mutating** needs write access to self. It can change fields or replace self entirely. A class method edits the object behind a reference and does not use mutating. A computed property can instead forward a write to shared reference storage with a **nonmutating setter**. The spelling describes access to the wrapper, not whether anything anywhere changes. [Methods](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/methods/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Mutating self versus editing an object

The struct method needs a mutable receiver; the class method does not replace the reference.

```swift
struct ValueCounter {
    var count = 0
    mutating func increment() { count += 1 }
    mutating func reset() { self = ValueCounter() }
}
final class ObjectCounter {
    var count = 0
    func increment() { count += 1 }
}
var value = ValueCounter()
value.increment()
precondition(value.count == 1)
value.reset()
precondition(value.count == 0)
let object = ObjectCounter()
object.increment()
precondition(object.count == 1)
```

**Walkthrough:** increment and reset write the struct’s value, so value is var. The object binding stays unchanged when its count changes.

### A nonmutating setter can forward a write

Shared setup named Handle. This block is runnable; prepend it to the answer that explicitly names it.

```swift
final class Storage { var count = 0 }
struct Handle {
    let storage: Storage
    var count: Int {
        get { storage.count }
        nonmutating set { storage.count = newValue }
    }
}
let first = Handle(storage: Storage())
let second = first
second.count = 8
precondition(first.count == 8)
```

**Walkthrough:** The setter changes Storage.count without replacing Handle or its stored reference. This permits a set through a let Handle, but copies intentionally share the state.

## Rules and pitfalls

### A value-shaped wrapper may still share mutation

**Common pitfall:** Expecting two copies of a struct with a reference-backed property to hold independent state.

**Mistake:** Standalone runnable example.

```swift
final class Box { var value = 0 }
struct Counter { let box: Box }
let saved = Counter(box: Box())
let draft = saved
draft.box.value = 5
precondition(saved.box.value == 5)
```

**Consequence:** A draft edit reaches the saved counter.

**Why:** Both wrappers copied the same reference. A let field holds that reference constant; it does not freeze its target.

**Correction:** Keep independently editable state in a stored value.

```swift
struct Counter { var value = 0 }
let saved = Counter()
var draft = saved
draft.value = 5
precondition(saved.value == 0 && draft.value == 5)
```

**What changes:** The copied counter now owns independent integer state. Deliberate shared handles should document that contract instead.

### An accessor does not bypass a let value

**Common pitfall:** Trying to call a mutating value method through a let binding.

**Mistake:** Compile-only; intentionally rejected.

```swift
struct Counter {
    var value = 0
    mutating func increment() { value += 1 }
}
let counter = Counter()
counter.increment()
```

**Consequence:** The compiler rejects a mutating call on an immutable value.

**Why:** The method requires exclusive write access to counter. Making the method a different spelling of a stored-value write would not remove that requirement.

**Correction:** Make the binding mutable when local mutation is intended.

```swift
struct Counter {
    var value = 0
    mutating func increment() { value += 1 }
}
var counter = Counter()
counter.increment()
precondition(counter.value == 1)
```

**What changes:** var permits the required write. It supplies no lock and no exemption from exclusivity or actor-isolation rules.

## Worked example

### Reading a value does not keep a live view

**Setup:** Standalone script. Compare a copied Int with a copied reference.

**Predict:** Which count changes after increment?

```swift
final class Counter { var count = 0 }
let counter = Counter()
let recorded = counter.count
let alias = counter
alias.count += 1
precondition(recorded == 0)
precondition(counter.count == 1)
```

**Outcome:** recorded stays 0; both object references observe 1.

**Why:** Reading an Int property yields an Int value. Copying the object reference keeps access to that object’s later state.

## Interview questions

### Why does a struct method need mutating while a class method does not?

**Answer:** A struct method must declare that it writes self. A class method can edit the referenced object while leaving the reference binding unchanged.

**Example:** Standalone runnable example.

```swift
struct Position {
    var x = 0
    mutating func move() { x += 1 }
}
final class Cursor {
    var x = 0
    func move() { x += 1 }
}
var position = Position()
let cursor = Cursor()
position.move()
cursor.move()
precondition(position.x == 1 && cursor.x == 1)
```

**Walkthrough:** position needs write access to its stored value. cursor retains its identity throughout the call.

**Boundary:** mutating does not mean thread-safe. An enum method that replaces self has the same mutating requirement as a struct method.

### What does let prevent for a struct versus a class?

**Answer:** It prevents changing the bound value. For a struct that includes its stored fields; for a class that means the reference, while mutable object properties remain editable.

**Example:** Standalone runnable example.

```swift
struct Settings { var retries: Int }
final class Session { var retries = 2 }
let settings = Settings(retries: 2)
var draft = settings
draft.retries = 4
let session = Session()
session.retries = 4
precondition(settings.retries == 2 && draft.retries == 4)
precondition(session.retries == 4)
```

**Walkthrough:** The copy draft can change because it is var. session can expose changing state despite its let binding.

**Boundary:** A class API can enforce immutability by exposing no mutation and containing immutable state. The binding keyword alone does not establish that design.

### How can weak let become nil without reassignment, and does it make an object Sendable?

**Answer:** A weak reference does not keep an object alive. weak let prevents explicitly rebinding the reference, but Automatic Reference Counting (ARC), Swift’s object-lifetime system, can still zero it when the object dies. This does not make the referenced type Sendable.

**Example:** Standalone runnable example.

```swift
final class Owner {}
var strong: Owner? = Owner()
weak let observer = strong
precondition(observer != nil)
strong = nil
precondition(observer == nil)
```

**Walkthrough:** The first check observes the living object. Releasing the only strong reference destroys it and zeroes observer. weak let is supported by this Swift 6.4 compiler. [SE-0481](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)

**Boundary:** The separate ImmutableWeakCaptures feature concerns closure captures. Neither feature supplies isolation or permits unrestricted access to a mutable object across actors.

### How do reference fields affect Sendable and synthesized Equatable, Hashable, or Codable?

**Answer:** Checked Sendable requires the stored state to satisfy its safety rules. Other synthesis follows the fields’ conformances and their meanings; it does not turn reference state into an independent snapshot.

**Example:** Standalone runnable example.

```swift
import Foundation

final class Payload: Equatable, Codable {
    var count: Int
    init(_ count: Int) { self.count = count }
    static func == (lhs: Payload, rhs: Payload) -> Bool { lhs.count == rhs.count }
}
struct Record: Equatable, Codable { let payload: Payload }
let a = Record(payload: Payload(1))
let b = Record(payload: Payload(1))
precondition(a == b)
b.payload.count = 2
precondition(a != b)
let decoded = try JSONDecoder().decode(Record.self, from: JSONEncoder().encode(a))
precondition(decoded == a)
precondition(decoded.payload !== a.payload)
```

**Walkthrough:** Record’s equality delegates to Payload. Encoding stores data, and this decoding constructs a fresh Payload; that behavior comes from Codable implementation, not assignment.

**Boundary:** Synthesis requires eligible properties to conform. Hashable keys whose hash-relevant reference state later changes can break collection invariants ([[B8 — Equatable, Hashable, and Collection Correctness|B8]]). A mutable Payload also prevents checked Sendable synthesis; [[A1 — Value semantics, reference semantics, identity, and copy-on-write|A1]] shows the rejection.

### How can a nonmutating setter work through let while a stored-value mutate accessor cannot?

**Answer:** A nonmutating setter can write an external object. An accessor that lends mutable access to the struct’s own storage needs a mutable receiver.

**Example:** Prepend the Handle setup from Essential mechanics.

```swift
let handle = Handle(storage: Storage())
handle.count = 12
precondition(handle.count == 12)
struct ValueSlot {
    private var storage = 0
    var count: Int {
        borrow { storage }
        mutate { &storage }
    }
}
var valueSlot = ValueSlot()
valueSlot.count = 12
let immutable = valueSlot
precondition(immutable.count == 12)  // Reading is allowed; writing needs var.
```

**Walkthrough:** The let handle keeps the same stored reference and forwards its write to Storage. ValueSlot lends its own stored integer through mutate, so the write requires valueSlot to be var; immutable permits only the read.

**Boundary:** Modern mutate accessors yield exclusive access to storage; they are not a way to bypass let. The extension below demonstrates this difference.

### Extension: how do borrow and mutate accessors expose noncopyable stored values?

**Answer:** borrow lends read access without copying the value. mutate lends exclusive write access, so the receiver must be mutable. These Swift 6.4 accessors preserve the ownership rules of the stored value.

**Example:** Standalone runnable example.

```swift
struct Token: ~Copyable { var id: Int }
struct Slot: ~Copyable {
    private var storage: Token
    init(_ id: Int) { storage = Token(id: id) }
    var token: Token {
        borrow { storage }
        mutate { &storage }
    }
}
func useSlot() -> Int {
    var slot = Slot(1)
    let before = slot.token.id
    slot.token.id = 7
    return before + slot.token.id
}
precondition(useSlot() == 8)
```

**Walkthrough:** Reading id borrows the token. Assigning id uses mutate to reach the stored token without copying it. [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

**Boundary:** This is a language/toolchain extension, not a concurrency primitive. Calling the same write through let is rejected; [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`|C7]]/[[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] cover the full lifetime rules.

With the Token and Slot definitions from the extension answer, the following **compile-only, intentionally rejected** function attempts a write through let:

```swift
func immutableSlot() {
    let slot = Slot(1)
    slot.token.id = 2
}
```

## Focused exercise

**Task:** Implement a value counter and a shared handle. Demonstrate their different behavior through copies and let bindings.

**Acceptance checks:**

- The value counter changes only its mutable copy.
- A let handle can forward a mutation and both aliases observe it.
- Resetting a value changes only that value.

**Solution:** Standalone script.

```swift
struct ValueCounter {
    var count = 0
    mutating func increment() { count += 1 }
    mutating func reset() { self = ValueCounter() }
}
final class Storage { var count = 0 }
struct Handle {
    let storage: Storage
    var count: Int {
        get { storage.count }
        nonmutating set { storage.count = newValue }
    }
}
let saved = ValueCounter()
var draft = saved
draft.increment()
precondition(saved.count == 0 && draft.count == 1)
draft.reset()
precondition(saved.count == 0 && draft.count == 0)
let first = Handle(storage: Storage())
let second = first
second.count = 3
precondition(first.count == 3 && second.count == 3)
```

**Reasoning:** Both types are structs, but their stored representation establishes different mutation behavior. Explain that contract at the API boundary; the nonmutating keyword alone is not a promise of purity.

## In practice

**Choose it when:** Use mutating for value changes and a nonmutating setter only when the wrapper intentionally forwards effects elsewhere.

**Investigate by:** Inspect the storage reached by an accessor and the conformances of every stored property.

**Strong evidence:** Demonstrate let/var behavior, copy behavior, and isolation separately; verify collection keys remain stable after insertion.

## References

### Related guides

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[B13 — Property Wrappers and Generated Storage Semantics]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]

### Official sources

- [Methods](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/methods/)
- [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)
- [SE-0481](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)
- [Swift.org. "The Basics." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/thebasics/)
- [Swift.org. "Protocols." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [Swift.org. "Summary of the Grammar." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/summaryofthegrammar/)
- [Swift.org. "Attributes." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/)
- [Apple Developer. "Sendable." Apple Developer Documentation](https://developer.apple.com/documentation/Swift/Sendable)
- [Apple Developer. "State." Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/state)
