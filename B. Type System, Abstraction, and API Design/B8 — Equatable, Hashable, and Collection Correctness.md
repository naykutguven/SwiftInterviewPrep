---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - equatable
  - hashable
  - collections
---

# B8 — Equatable, Hashable, and Collection Correctness

**Learning goals:**

- Maintain equality laws, matching hash inputs, and stable collection-observed key state.
- Distinguish membership from retained content after insert, replacement, and dictionary assignment.
- Apply Swift 6.4 collection conformances with their real OS availability and element constraints.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] for copied references versus independent values and [[B7 — Synthesized conformances and semantic correctness]] for synthesized/custom equality and hashing.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A hash table uses a hash to narrow candidate matches, then equality to identify the matching key. Unequal values may share a hash. Equal values must hash using matching essential components. For the domain used as keys, equality should be reflexive, symmetric, and transitive.

Once a key is stored, its equality/hash-relevant state must stay stable as observed by the collection. A separate value copy may change independently; an alias that changes state still read by the stored key breaks that invariant. Entity IDs are often good immutable keys, while mutable snapshot content belongs in the associated value. [Hashable](https://developer.apple.com/documentation/swift/hashable)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### A collision is not equality

This deliberately poor hash implementation makes every key collide while preserving a valid equality/hash contract.

```swift
struct Key: Hashable {
    let id: Int
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(0) }
}
let first = Key(id: 1)
let second = Key(id: 2)
precondition(first != second)
let keys: Set = [first, second, Key(id: 1)]
precondition(keys.count == 2 && keys.contains(first) && keys.contains(second))
```

**Walkthrough:** The Set still distinguishes IDs using equality. Feeding the same hash input for every ID is valid but can perform poorly; it is a teaching example, not a recommended hash function.

### Changing an independent key copy is safe

The stored value does not share mutable reference storage with the local copy.

```swift
struct Key: Hashable { var id: Int }
var local = Key(id: 1)
let stored: Set = [local]
local.id = 2
precondition(stored.contains(Key(id: 1)))
precondition(!stored.contains(local))
```

**Walkthrough:** Only the local copy changes. The Set still observes its stored key as ID 1. The next pitfall contrasts an alias into shared mutable storage.

## Rules and pitfalls

### Do not change state still observed by a stored key

**Common pitfall:** Hashing mutable class content and changing it while the same object is in a Set. Compile-only; the invalid collection state is never executed or inspected.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
final class User: Hashable {
    let id: Int
    var name: String
    init(id: Int, name: String) {
        self.id = id
        self.name = name
    }
    static func == (lhs: User, rhs: User) -> Bool { lhs.id == rhs.id && lhs.name == rhs.name }
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
        hasher.combine(name)
    }
}
func invalid() {
    let user = User(id: 1, name: "old")
    let members: Set = [user]
    user.name = "new"  // Violates the stored-key invariant.
    _ = members.contains(user)
}
```

**Consequence:** The collection’s lookup assumptions are no longer valid. No particular contains result or count after the violation is promised.

**Why:** The collection retained the same reference, so the mutation changes key state behind its stored entry.

**Correction:** Keep stable IDs as keys and mutable content as values.

```swift
struct Snapshot {
    let id: Int
    var name: String
}
let old = Snapshot(id: 1, name: "old")
var byID = [old.id: old]
byID[1]?.name = "new"
precondition(byID.count == 1 && byID[1]?.name == "new")
```

**What changes:** Changing the dictionary value does not mutate its immutable Int key. A reference key with immutable hash-relevant identity can also work, but its aliasing contract must remain valid.

### Membership repair does not imply content replacement

**Common pitfall:** Using ID-only equality/hash and expecting Set.insert to replace an equal member’s label.

**Mistake:** Standalone runnable example.

```swift
struct Snapshot: Hashable {
    let id: Int
    let name: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
var members: Set = [Snapshot(id: 1, name: "old")]
let result = members.insert(Snapshot(id: 1, name: "new"))
precondition(!result.inserted && result.memberAfterInsert.name == "old")
precondition(members.first?.name == "old")
```

**Consequence:** Membership is correct but the retained label is still old.

**Why:** insert finds an existing equal member and returns it rather than replacing its non-identity content.

**Correction:** Use update(with:) for an equal-member replacement, or assign the new snapshot under its stable dictionary key.

```swift
struct Snapshot: Hashable {
    let id: Int
    let name: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
var members: Set = [Snapshot(id: 1, name: "old")]
let previous = members.update(with: Snapshot(id: 1, name: "new"))
precondition(previous?.name == "old" && members.first?.name == "new")
```

**What changes:** The operation now matches the replacement policy. Test retained values as well as count and contains.

## Worked example

### Stable identity and replaceable snapshots

**Setup:** Standalone script. Structural snapshot equality and ID-keyed lookup serve different purposes.

**Predict:** Does changing content require changing the lookup key?

```swift
struct Snapshot: Equatable {
    let id: Int
    let name: String
}
let old = Snapshot(id: 7, name: "old")
let new = Snapshot(id: 7, name: "new")
var byID = [old.id: old]
byID[new.id] = new
precondition(old != new)
precondition(byID.count == 1 && byID[7] == new)
```

**Outcome:** The value changes while one ID key remains.

**Why:** A dictionary separates identity from content without forcing every Snapshot comparison to be ID-only.

## Interview questions

### Why does equal hash not establish equality, and why is hashValue not a persisted key?

**Answer:** Collisions are valid, so equality must distinguish candidate matches. hashValue is seeded and implementation-dependent; it is not a durable domain encoding across processes or versions.

**Example:** Standalone runnable example.

```swift
struct Key: Hashable {
    let id: Int
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(0) }
}
let first = Key(id: 1)
let second = Key(id: 2)
precondition(first.hashValue == second.hashValue)  // Deliberate same-input collision.
precondition(first != second)
precondition(Set([first, second]).count == 2)
let persistedID = first.id
precondition(persistedID == 1)
```

**Walkthrough:** Both hashes deliberately combine the same value, yet the keys are unequal and coexist. Persisting the explicit ID preserves the domain key instead of a hash-table implementation result.

**Boundary:** No exact hash integer or cross-run difference is asserted. ObjectIdentifier is also lifetime/process-scoped identity, not a durable entity key.

### How can a struct containing a reference have the same mutable-key bug as a class?

**Answer:** Copying the struct copies its reference, not the object graph. If equality/hash reads that referenced object’s mutable state, an outside alias can change the key already observed by the collection.

**Example:** Standalone compile-only illustration of an invalid collection invariant; no driver is supplied.

```swift
final class Identity { var id = 1 }
struct Key: Hashable {
    let identity: Identity
    static func == (lhs: Key, rhs: Key) -> Bool { lhs.identity.id == rhs.identity.id }
    func hash(into hasher: inout Hasher) { hasher.combine(identity.id) }
}
func invalidUse() {
    let shared = Identity()
    let key = Key(identity: shared)
    let members: Set = [key]
    shared.id = 2  // Changes state still read by the stored key.
    _ = members.contains(key)
}
```

**Walkthrough:** The outer value does not isolate Identity.id. Calling invalidUse would violate the stored-key contract, so this function is compiled only and never run.

**Boundary:** Store immutable value data in the key or enforce immutable identity behind the reference. The next independent correction copies the ID into the key.

Runnable correction: taking a value snapshot of the identity makes later external reference mutation irrelevant to the stored key.

```swift
final class Identity { var id = 1 }
struct Key: Hashable { let id: Int }
let shared = Identity()
let key = Key(id: shared.id)
let members: Set = [key]
shared.id = 2
precondition(members.contains(Key(id: 1)))
precondition(!members.contains(Key(id: 2)))
```

### Why can insert retain old content after an ID-only fix, and when should replacement be explicit?

**Answer:** Equality determines membership, while insert preserves an existing equal member. update(with:) replaces it; dictionary assignment replaces the value for a stable key. Choose the operation from the retained-content policy.

**Example:** Standalone runnable example.

```swift
struct Snapshot: Hashable {
    let id: Int
    let name: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
let old = Snapshot(id: 1, name: "old")
let new = Snapshot(id: 1, name: "new")
var members: Set = [old]
precondition(members.insert(new).memberAfterInsert.name == "old")
precondition(members.update(with: new)?.name == "old")
precondition(members.first?.name == "new")
var byID = [old.id: old]
byID[new.id] = new
precondition(byID[1]?.name == "new")
```

**Walkthrough:** The first operation detects an equal member; the second replaces it; dictionary assignment updates content under ID 1. Counts alone would miss the stale-label issue.

**Boundary:** If snapshots should compare structurally, keep that equality and use the ID-keyed dictionary for entity lookup instead of changing all content comparisons.

### What does Dictionary.Keys hashing ignore, and why can EmptyCollection<NotHashable> be Hashable?

**Answer:** Dictionary.Keys describes the set of keys, independently of values and iteration order. EmptyCollection has no element values to hash, so its conformance needs no Hashable element. CollectionOfOne still needs its single element to be Hashable.

**Example:** Standalone runnable example.

```swift
struct NotHashable {}
@available(macOS 27.0, iOS 27.0, *)
func counts() -> [Int] {
    let first = ["a": 1, "b": 2]
    let second = ["b": 200, "a": 100]
    let keys = Set([first.keys, second.keys])
    let singles = Set([CollectionOfOne(1), CollectionOfOne(1), CollectionOfOne(2)])
    let empty = Set([EmptyCollection<NotHashable>(), EmptyCollection<NotHashable>()])
    return [keys.count, singles.count, empty.count]
}
precondition(counts() == [1, 2, 1])
```

**Walkthrough:** Different dictionary values still produce equal key sets. The one-element collection distinguishes 1 from 2, while every empty collection of this Element has no values to distinguish. [SE-0514](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0514-hashable-conformance-for-dictionarykeys-collectionofone-emptycollection.md)

**Boundary:** These new conformances require macOS 27/iOS 27 in the selected SDK. For older targets, materialize Set(dictionary.keys) when that is the desired copyable-key value. Generalized Hashable still does not remove Set’s Copyable element constraint.

## Focused exercise

**Task:** Store changed snapshots under stable keys and compare insertion with replacement.

**Acceptance checks:**

- Equal-ID insertion reveals the retained old member.
- Explicit Set replacement retains the new content.
- Dictionary assignment keeps one immutable ID key and updates its snapshot.

**Solution:** Standalone replacement probe.

```swift
struct Snapshot: Hashable {
    let id: Int
    let name: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
let old = Snapshot(id: 1, name: "old")
let new = Snapshot(id: 1, name: "new")
var members: Set = [old]
let insertion = members.insert(new)
precondition(!insertion.inserted && insertion.memberAfterInsert.name == "old")
precondition(members.count == 1 && members.contains(new))
let previous = members.update(with: new)
precondition(previous?.name == "old" && members.first?.name == "new")
var byID = [old.id: old]
byID[new.id] = new
precondition(byID.count == 1 && byID[1]?.name == "new")
precondition(!members.contains(Snapshot(id: 2, name: "new")))
```

**Reasoning:** The immutable ID defines membership, while name is replaceable content. The assertions inspect both membership and the retained value; they never depend on a particular hash integer or an invalid mutated-key collection.

## In practice

**Choose it when:** Separate lookup identity from mutable content in stores, caches, selection, and deduplication.

**Investigate by:** Check equality laws, hash components, aliases into stored keys, and retained values after each collection operation.

**Strong evidence:** Demonstrate valid behavior through replacement and value/reference boundaries, then verify availability and consumer constraints instead of inferring correctness from one hash printout.

## References

### Related guides

- [[B7 — Synthesized conformances and semantic correctness]]
- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]

### Official sources

- [Hashable](https://developer.apple.com/documentation/swift/hashable)
- [SE-0514](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0514-hashable-conformance-for-dictionarykeys-collectionofone-emptycollection.md)
- [Equatable](https://developer.apple.com/documentation/swift/equatable)
- [Hasher.finalize](<https://developer.apple.com/documentation/swift/hasher/finalize()>)
- [Set source and API contracts](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Set.swift)
- [SE-0499: Noncopyable simple protocols](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)
