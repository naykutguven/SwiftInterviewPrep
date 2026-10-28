---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - value-semantics
  - reference-semantics
  - copy-on-write
---

# A1 — Value Semantics, Reference Semantics, Identity, and Copy-on-Write

**Learning goals:**

- Predict what assignment copies, including arrays that contain objects or closures.
- Distinguish equality, object identity, domain identity, ownership, and isolation.
- Explain the observable guarantee of copy-on-write without promising deep copies or allocation counts.

**Prerequisites:** Basic stored properties and function calls. [[A2 — `let`, `var`, `mutating`, and method semantics]] explains mutation syntax; no concurrency knowledge is needed for the core examples.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Suppose an editor copies a saved draft before changing it. The useful question is whether those edits can change the saved version. A **value** has its own logical contents after assignment. A **reference** identifies an object, so assigning the reference gives another path to that same object. A struct copies its stored values; a stored object reference is still a reference after that copy.

Array, String, and Dictionary provide value behavior. Their implementations can share backing storage until a write needs independent storage. This is **copy-on-write**: an implementation strategy that preserves the value contract. It does not recursively clone objects stored inside the collection, and it does not promise an allocation on every assignment or mutation. [Structures and classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Two values, one object

Read each assignment as copying the right-hand value. For a class, that value is a reference to an instance.

```swift
struct Point { var x: Int }
final class Marker { var x = 1 }
var firstPoint = Point(x: 1)
var secondPoint = firstPoint
secondPoint.x = 2
precondition(firstPoint.x == 1)
precondition(secondPoint.x == 2)
let firstMarker = Marker()
let secondMarker = firstMarker
secondMarker.x = 2
precondition(firstMarker.x == 2)
precondition(firstMarker === secondMarker)
```

**Walkthrough:** The struct edit changes secondPoint. Both marker bindings identify the same instance, so either path observes its changed x. The let bindings prevent replacing those references; they do not freeze the marker.

### Collection independence stops at its elements

Replacing an element changes the array sequence. Editing an object reached through an element changes that object.

```swift
final class Item {
    var title: String
    init(_ title: String) { self.title = title }
}
let saved = [Item("Draft")]
var edited = saved
edited[0].title = "Shared edit"
precondition(saved[0].title == "Shared edit")
edited[0] = Item("Replacement")
edited.append(Item("Extra"))
precondition(saved.count == 1)
precondition(saved[0].title == "Shared edit")
precondition(edited[0] !== saved[0])
```

**Walkthrough:** The first edit follows the copied reference into the shared item. Replacement and append change only edited’s sequence. The result establishes value behavior; it does not measure storage sharing or allocations.

## Rules and pitfalls

### Define what a snapshot includes

**Common pitfall:** Calling a copied struct a snapshot while it still exposes mutable reference state.

**Mistake:** Standalone runnable example.

```swift
final class Text { var value = "Saved" }
struct Draft { var text: Text }
let saved = Draft(text: Text())
let edited = saved
edited.text.value = "Changed"
precondition(saved.text.value == "Changed")  // The unwanted change is observable.
```

**Consequence:** The saved draft changes through edited, even though both outer bindings are let.

**Why:** Both drafts contain a reference to the same Text. Copying the outer struct did not clone Text.

**Correction:** Store the data that must be independent as values.

```swift
struct Draft { var text: String }
let saved = Draft(text: "Saved")
var edited = saved
edited.text = "Changed"
precondition(saved.text == "Saved")
precondition(edited.text == "Changed")
```

**What changes:** The text is now inside the value boundary. A deliberate clone can also work, but it must cover every mutable node that the snapshot promises to isolate.

### let and private do not protect a shared object

Sendable is a promise, checked under Swift’s rules, about sharing or transferring values between concurrency domains. It does not promise an independent snapshot. An actor owns an isolation domain: outside callers follow its access rules instead of directly editing its stored state. The following example introduces that distinction; [[D4 — Actors and Actor Isolation]] and [[D6 — `Sendable`, `@Sendable`, and `sending`]] develop it.

**Common pitfall:** Assuming an immutable binding or hidden field makes reachable mutable state safe to transfer.

**Mistake:** Compile-only; intentionally rejected.

```swift
final class Counter { var count = 0 }
struct SharedCounter: Sendable {
    private let storage = Counter()
    func increment() { storage.count += 1 }
}
```

**Consequence:** Swift rejects the checked Sendable conformance because Counter is mutable, non-Sendable reference state.

**Why:** Access control limits which source code can name the field. It does not provide isolation or synchronization for the object.

**Correction:** Make the shared service own an isolation boundary. This separate runnable example uses an actor.

```swift
actor SharedCounter {
    private var count = 0
    func increment() { count += 1 }
    func value() -> Int { count }
}
let counter = SharedCounter()
await counter.increment()
let observed = await counter.value()
precondition(observed == 1)
```

**What changes:** Every access to the actor’s state follows its isolation rules. An actor remains a shared reference; it is not an independent value snapshot. See [[D4 — Actors and Actor Isolation|D4]] and [[D6 — `Sendable`, `@Sendable`, and `sending`|D6]].

## Worked example

### Extracting an array from a shared owner

**Setup:** Standalone script. A Shelf is shared, but its titles property is an Array value.

**Predict:** Does appending to extracted also append to shelf.titles?

```swift
final class Shelf { var titles = ["A"] }
let shelf = Shelf()
let alias = shelf
var extracted = shelf.titles
extracted.append("B")
alias.titles.append("C")
precondition(extracted == ["A", "B"])
precondition(shelf.titles == ["A", "C"])
```

**Outcome:** extracted contains A, B; shelf.titles contains A, C.

**Why:** Reading the property produces an array value. Mutating that local value is different from mutating the property through the shared owner.

## Interview questions

### What question do ==, ===, and explicit domain IDs each answer?

**Answer:** == follows a type’s equality rule; === compares object identity. A domain ID identifies an entity under an application-defined scope and lifetime. These answers need not agree.

**Example:** Standalone runnable example.

```swift
final class User: Equatable {
    let id: Int
    var name: String
    init(id: Int, name: String) {
        self.id = id
        self.name = name
    }
    static func == (lhs: User, rhs: User) -> Bool { lhs.name == rhs.name }
}
let old = User(id: 7, name: "Ada")
let reloaded = User(id: 7, name: "Ada")
precondition(old == reloaded)
precondition(old !== reloaded)
reloaded.name = "Ada L."
precondition(old != reloaded)
precondition(old.id == reloaded.id)
let absent: User? = nil
precondition(absent === nil)
```

**Walkthrough:** Separate allocations can compare equal. Editing content changes this chosen equality result without changing either entity ID. Comparing nil references says nothing about an existing user.

**Boundary:** This equality rule is illustrative. Production models must define whether equality means all content, entity identity, or something else; an ID is useful only with a documented uniqueness scope.

### Why do object mutation, array-element replacement, and extracting an array property have different aliasing effects?

**Answer:** The operation determines which storage changes. Object mutation follows a reference; replacement edits an array value; property extraction reads that array value into another binding.

**Example:** Standalone runnable example.

```swift
final class Item { var count = 0 }
final class Store { var items = [Item()] }
let store = Store()
var draft = store.items
draft[0].count = 5
precondition(store.items[0].count == 5)
draft[0] = Item()
draft.append(Item())
precondition(store.items.count == 1)
precondition(store.items[0].count == 5)
precondition(draft[0].count == 0 && draft.count == 2)
```

**Walkthrough:** The first write reaches the shared item. The next two writes change draft’s sequence of references, leaving the owner’s array intact.

**Boundary:** An array of value elements would also isolate the element’s value edits. Neither case promises how many buffer copies the optimizer performs.

### How can copied closures share state, and why does noncopyable ownership not guarantee a deeply unique graph?

**Answer:** A copied closure can retain the same captured storage. A noncopyable wrapper cannot be duplicated implicitly, but it can contain a class reference that already has other aliases.

**Example:** Standalone runnable example.

```swift
func makeCounter() -> () -> Int {
    var count = 0
    return {
        count += 1
        return count
    }
}
struct Action { let next: () -> Int }
let first = Action(next: makeCounter())
let second = first
precondition(first.next() == 1)
precondition(second.next() == 2)
final class Counter { var count = 0 }
struct Owner: ~Copyable { let counter: Counter }
func probeOwner() {
    let shared = Counter()
    let owner = Owner(counter: shared)
    shared.count = 9
    precondition(owner.counter.count == 9)
}
probeOwner()
```

**Walkthrough:** Both actions call a closure that updates one captured variable. Owner cannot be copied, yet shared and owner.counter still reach the same Counter.

**Boundary:** Ownership, snapshot independence, and Sendable safety are separate contracts. A final immutable class containing Sendable data can be Sendable while still retaining object identity.

### Extension: what does consuming a noncopyable value actually prevent?

**Answer:** It prevents later use of that consumed value unless the variable is reinitialized. It does not make references inside the value deeply unique.

**Example:** Standalone runnable example.

```swift
struct Ticket: ~Copyable {
    let number: Int
    consuming func redeem() -> Int { number }
}
func redeemTicket() -> Int {
    let ticket = Ticket(number: 7)
    let transferred = consume ticket
    return transferred.redeem()
}
precondition(redeemTicket() == 7)
```

**Walkthrough:** Ownership moves into transferred and then into redeem. No second usable ticket is created. [SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)

**Boundary:** [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`|C7]]/[[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] cover borrowing and destruction. UniqueArray and borrowed views have different ownership contracts from a copyable Array snapshot; [[C12 — `Span`, `InlineArray`, and modern low-level abstractions|C12]] covers those APIs and deployment requirements.

With only the Ticket declaration from the preceding answer, this function is **intentionally rejected** during object compilation:

```swift
func cannotReuse() -> Int {
    let ticket = Ticket(number: 7)
    let transferred = consume ticket
    return transferred.redeem() + ticket.number
}
```

## Focused exercise

**Task:** Copy a draft containing an array and a class payload. Identify the independent edits and the shared edits.

**Acceptance checks:**

- Appending preserves the saved array, including when it starts empty.
- Payload mutation is shared, while replacing the reference affects only the edited draft.
- A separately constructed payload does not share changes.

**Solution:** Standalone script. The same probe checks nonempty and empty arrays.

```swift
final class Payload { var count = 1 }
struct Draft {
    var items: [Int]
    var payload: Payload
}
func checkDraft(items: [Int]) {
    let saved = Draft(items: items, payload: Payload())
    var edited = saved
    edited.items.append(2)
    edited.payload.count = 9
    precondition(saved.items == items)
    precondition(edited.items == items + [2])
    precondition(saved.payload.count == 9)
    let originalPayload = saved.payload
    edited.payload = Payload()
    precondition(saved.payload === originalPayload)
    precondition(edited.payload !== originalPayload)
    let separate = Draft(items: items, payload: Payload())
    separate.payload.count = 4
    precondition(saved.payload.count == 9)
}
checkDraft(items: [1])
checkDraft(items: [])
```

**Reasoning:** Each array assertion checks observable value independence. The reference assertions distinguish mutating a shared payload from replacing a reference. An immutable snapshot would store the needed payload data as values or clone the promised mutable graph.

## In practice

**Choose it when:** Use values when callers need independent edits. Use references for intentional sharing or lifecycle identity.

**Investigate by:** Draw every alias, including references stored in containers and variables retained by closures. Mutate through each public path.

**Strong evidence:** State exactly what a snapshot includes, verify that old snapshots survive edits, and define how stale drafts are reconciled.

## References

### Related guides

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B6 — Any, AnyObject, Metatypes, and Runtime Type Information]]
- [[B8 — Equatable, Hashable, and Collection Correctness]]
- [[C1 — ARC fundamentals]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D4 — Actors and Actor Isolation]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[F3 — Performance investigation habits]]

### Official sources

- [Structures and classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)
- [SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
- [The Swift Programming Language: Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)
- [Swift Array documentation/source](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift)
- [SE-0306: Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)
