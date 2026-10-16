---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - mutability
---

# A2 — `let`, `var`, `mutating`, and Method Semantics

**Goal:** Determine whether an operation needs mutable access to a binding or only reaches shared reference state.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Mutability describes access to a value, not whether anything reachable can change. A struct's mutating method can replace self and needs writable access to that value. A class method can change its object without replacing the reference held by the caller. This explains why let prevents value mutation but permits mutation through an ordinary class reference. [Methods](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/methods/)

## Rules and pitfalls

### Make mutation requirements honest

A value's ordinary setter mutates self. A nonmutating setter promises that the wrapper value itself need not be written; reference-backed storage can still change. Name that sharing explicitly and define its isolation.

**Common pitfall:** Adding nonmutating to make an API work through let can hide live shared state behind a value-shaped interface. Use ordinary value storage for independent snapshots or document a handle's identity and ownership.

### Preserve exclusivity through accessors

Swift 6.4 borrow/mutate accessors expose existing storage without ordinary get/set copying. Stored-value mutate access still needs a mutable owner and exclusive access. It supplies no lock. [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

**Common pitfall:** Treating a projection as a way around let or overlapping access violates the owner's access requirements. Keep the access scoped, or copy an independent value when the type supports copying.

## Interview questions

### Why does a `struct` method that changes stored properties need `mutating`, but a class method does not?

A mutating struct method requests writable access to self because it may change or replace the value. A class method changes the referenced object rather than the reference in the caller's binding, so mutating is neither needed nor permitted as a class-method modifier.

### What does `let` mean for a `struct` instance versus a `class` instance?

For a struct, let prevents operations requiring write access to the bound value. For a class, let prevents assigning another reference; it does not make the object's mutable properties immutable. Reference fields inside a let struct retain this distinction.

### Why can weak let observe nil while still rejecting reassignment, and what Sendable constraints remain?

A weak let fixes which object the weak reference initially names; destruction can still make its observation nil. It cannot be reassigned to another object. Weakness describes lifetime observation, not synchronization: Sendable requirements and isolation still apply to the stored property's type and use.

### How can one mutable reference-typed stored property change a type's `Sendable` story or make a synthesized conformance less meaningful?

A mutable non-Sendable object field can prevent checked Sendable conformance. Synthesized equality, hashing, or coding first requires the stored types to conform, then uses their semantics, which may encode identity or mutable state rather than a snapshot. Choose the model's intended behavior before requesting synthesis.

### Why can a reference-backed nonmutating setter work through let while a stored-value mutate accessor cannot?

The nonmutating setter writes through a reference without changing the wrapper's stored reference. A stored-value mutate accessor exposes part of self for modification and therefore needs mutable, exclusive access to that owner. The two APIs promise different sharing behavior.

## Focused exercise

**Task:** Compare a stored-value mutation with a reference-backed nonmutating setter through let. Identify which owner must be mutable and what state is shared.

**Acceptance checks:**

- Value mutation requires var.
- A let handle can write its reference-backed property, observed through its copy.
- Separate storage instances remain independent.

**Solution:**

Standalone; the two handles deliberately share storage:

```swift
struct ValueCounter {
    var count = 0
    mutating func increment() { count += 1 }
}
final class Storage { var count = 0 }
struct Handle {
    let storage: Storage
    var count: Int {
        get { storage.count }
        nonmutating set { storage.count = newValue }
    }
}
var value = ValueCounter()
value.increment()
let handle = Handle(storage: Storage())
let alias = handle
handle.count = 4
precondition(value.count == 1 && alias.count == 4)
```

Changing value to let intentionally rejects the increment call; compile that variant separately. The handle assignment remains legal because it does not write the wrapper.

**Reasoning:** The syntactic property assignment is not enough to identify what changes. Trace the setter and its access to self. A stored-value accessor cannot adopt this handle behavior merely by returning an inout projection.

## In practice

**Choose it when:** Use nonmutating setters for deliberate handles such as framework bindings, with documented shared storage.

**Investigate by:** Check accessor signatures, owner mutability, and aliases rather than only call-site syntax.

**Strong evidence:** Validate isolation and exclusive access along with the visible mutation behavior.

### Accessor and weak-storage detail — optional

#### Weak immutable references can still observe nil

Swift 6.4 accepts `weak let` for optional object references. The binding cannot be assigned another reference, but observing it returns nil after the object is destroyed. This is distinct from a strong let reference keeping the object alive. Checked local and stored-reference probes observe the object while it is retained and nil after releasing its last strong owner; assigning nil to the weak let binding is rejected.

SE-0481 records the feature as implemented in Swift 6.3, so this is catch-up context for a 6.2 migration rather than a new 6.4 introduction. An immutable weak property can participate in checked Sendable conformance when its referenced type is Sendable; it does not make an arbitrary referenced object thread-safe. C1 and A8 cover lifetime and capture policy in more depth. [SE-0481](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)

#### Swift 6.4 borrow/mutate accessors do not bypass mutability

These accessors project access to existing storage. They let a value expose a noncopyable member without an owned getter result:

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
func demo() {
    var slot = Slot(1)
    print(slot.token.id)
    slot.token.id += 1
    print(slot.token.id)
}
demo()
```

Output is `1`, then `2`. The mutation still requires a mutable owner and exclusive access. With the preceding Token/Slot definitions, this separate example is rejected:

```swift
func invalid() {
    let slot = Slot(1)
    slot.token.id += 1
}
```

This differs from the reference-backed nonmutating setter above: that setter changes a separate object without mutating the struct's stored reference. A mutate accessor here exposes the struct's own stored Token. Neither mechanism supplies a lock, actor isolation, deep immutability, or permission for conflicting access. [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

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
