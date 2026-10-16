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

**Goal:** Predict which edits remain independent after copying a reference-containing draft.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Copying an ordinary value produces a second logical value; copying an object reference preserves the object identity. A struct copies each stored property's value, including any references or closures. It does not recursively clone an object graph. Copy-on-write can preserve independent array values while sharing storage until a mutation requires separation. That is a representation strategy, not a promise of deep copying or a fixed allocation count. [Structures and classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)

## Rules and pitfalls

### Define the observable snapshot

Describe which state a caller may edit independently. Array element replacement changes one array value, while mutating a class element reaches the same object through every retained alias. Extracting an array property from a class creates a separate array value even though the owning class is shared.

**Common pitfall:** Calling a struct a snapshot without checking its reference fields or captured environments allows edits to leak. Store value data, explicitly clone the required graph, or document intentional sharing.

### Separate identity, ownership, and isolation

Use equality for an agreed comparison, object identity for the same instance, and a domain identifier for an entity across reloads. Noncopyable ownership restricts duplication of the wrapper, not all aliases reachable inside it. Sendable checking is another contract; it does not establish an independent snapshot.

**Common pitfall:** A let reference or private field does not freeze the referenced object. Isolation or synchronization must protect shared mutable state. See [[D6 — `Sendable`, `@Sendable`, and `sending`]].

## Interview questions

### What question do ==, ===, and explicit domain IDs each answer?

`==` asks whether values satisfy their equality rule. `===` asks whether object references identify the same instance, including optional references; two nil references do not establish that an object exists. Domain IDs identify entities according to an explicit lifetime and scope. Equal content, entity identity, and object identity can change independently.

### Why do object mutation, array-element replacement, and extracting an array property have different aliasing effects?

Object mutation changes shared storage. Replacing an array element changes that array's sequence of reference values, preserving the other array. Extracting an array property copies its value contract, so later sequence edits need not mutate the owner's property. Follow the particular operation rather than classifying every value reached through a class as reference-semantic.

### How can a copied struct containing a closure share mutable state, and why does noncopyable ownership not guarantee a deeply unique graph?

A closure can retain shared mutable captured storage, so copying a struct holding that closure can share its counter or service state. A noncopyable wrapper can still hold a reference aliased elsewhere. Neither syntax proves a deeply unique object graph; review the reachable state and the permitted mutation paths.

## Focused exercise

**Task:** Copy a draft containing an array and a class payload. Check independent array edits, shared object mutation, and the original snapshot.

**Acceptance checks:**

- Appending preserves the original array sequence.
- Payload mutation is shared; replacing the reference affects only the edited draft.
- Check an empty items array and a separately constructed payload.

**Solution:**

Standalone, synchronous model probe:

```swift
final class Payload { var count = 1 }
struct Draft { var items: [Int]; var payload: Payload }
let original = Draft(items: [1], payload: Payload())
var edited = original
edited.items.append(2)
edited.payload.count = 9
precondition(original.items == [1])
precondition(edited.items == [1, 2])
precondition(original.payload.count == 9)
let shared = original.payload
edited.payload = Payload()
precondition(original.payload === shared)
precondition(edited.payload !== shared)
```

**Reasoning:** The draft copies an Array value and an object reference. Array's independence covers the sequence of references or integers stored in it, not arbitrary state behind a reference. A full immutable snapshot would convert Payload into independent value data before returning the draft.

## In practice

**Choose it when:** Use values for independent edits and snapshots; references suit shared services and lifecycle identity.

**Investigate by:** Draw the aliases and test mutation through every exposed path, including closures.

**Strong evidence:** Define the snapshot boundary, verify old snapshots after commit, and choose an explicit conflict policy for stale drafts.

### Ownership and transfer boundaries — optional

#### Noncopyable does not mean reference type or deeply unique graph

This independent specialist fixture uses a mutable reference payload:

```swift
final class Counter {
    var value: Int
    init(_ value: Int) { self.value = value }
}
```

```swift
struct Ticket: ~Copyable {
    let number: Int
    consuming func redeem() -> Int { number }
}

struct UniqueCounterOwner: ~Copyable {
    let counter: Counter
}

func ownershipProbe() {
    let ticket = Ticket(number: 7)
    let transferred = consume ticket
    print(transferred.redeem())

    let shared = Counter(1)
    let owner = UniqueCounterOwner(counter: shared)
    shared.value = 9
    print(owner.counter.value)
}
```

ownershipProbe prints `7`, then `9`. The Ticket is transferred and consumed; no duplicate usable ticket is created. UniqueCounterOwner cannot be copied, but its class payload still has the separately retained alias named shared. Unique ownership of one wrapper does not erase aliases already inside its object graph.

This is **intentionally rejected** when compiled with the Ticket definition:

```swift
func cannotReuse() -> Int {
    let ticket = Ticket(number: 7)
    let transferred = consume ticket
    return transferred.redeem() + ticket.number
}
```

The original ticket is used after it has been consumed. Ownership diagnostics need actual compilation, not only a parser or a shallow type-checking experiment. Noncopyable structs/enums predate 6.4; they correct the older blanket statement that every struct can be copied. C7/C8 explain borrowing, consuming, and resource lifetimes. [SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)

Swift 6.4's unique-storage and borrowing tools add representation choices. A UniqueArray is not an interchangeable copyable Array snapshot; choosing it changes the ownership contract. Borrowed views do not become owned snapshots just because their syntax looks value-like. C8/C12 give the checked API and deployment requirements; F3 measures a limited workload without claiming universal speedups.

#### Value behavior and Sendable safety are separate

```swift
struct NumberSnapshot: Sendable {
    var items: [Int]
}

final class Catalog: Sendable {
    let names: [String]
    init(names: [String]) { self.names = names }
}

func requiresSendable<T: Sendable>(_ value: T) {}

func sendableProbe() {
    requiresSendable(NumberSnapshot(items: [1]))
    requiresSendable(Catalog(names: ["Book"]))
}
```

Both types satisfy Sendable. NumberSnapshot contains independently stored Int array values. Catalog is a final immutable reference type whose stored data satisfies the required checking. That does not imply every immutable reference binding, or every class with let properties, is safe: those properties can themselves refer to mutable state.

This declaration is **intentionally rejected** with the Counter declaration in this subsection:

```swift
struct BadTransfer: Sendable {
    let counter: Counter
}
```

Its stored Counter reference is non-Sendable. Spelling struct or let does not repair that. Do not add unchecked conformance merely to make this example pass; choose an appropriate value snapshot, immutable representation, actor, or audited synchronization model. D6 covers region transfers and isolation-specific rules beyond this simple declared-conformance check. [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)

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
