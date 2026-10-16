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

**Goal:** Replace changed snapshot content while preserving stable collection keys.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Hash tables use hashes to narrow candidates and equality to distinguish actual matches. Equal values must hash using matching essential components; collisions between unequal values are valid. A stored key's equality/hash-relevant state must remain stable as observed by the collection, including through reference aliases. [Hashable](https://developer.apple.com/documentation/swift/hashable)

## Rules and pitfalls

### Keep identity keys stable

Use immutable domain IDs for entity lookup and store mutable snapshot content as dictionary values. Mutating an independent copy of a value key is safe; mutating referenced state still observed by a stored key is not.

**Common pitfall:** A struct key whose hash reads a mutable class field has the same aliasing hazard as a class key. Freeze the relevant data or redesign the key around stable identity.

### Distinguish insertion from replacement

Set.insert does not promise to replace an equal stored element with new content. Set.update(with:) replaces the equal member; dictionary assignment replaces the value for a key. Choose the operation from the desired retained-value policy.

**Common pitfall:** Fixing ID-only equality and hashing can restore membership while insert still retains an older label. Test retained content as well as count and contains; see [[B7 — Synthesized conformances and semantic correctness]].

## Interview questions

### Why does equal hash not establish equality, and why are persistent hashValue keys wrong?

Different values may collide, so equal hash is only a candidate match and must be followed by equality. hashValue is seeded and implementation-dependent; it can change between runs or library versions. Persist explicit domain identifiers rather than hashes or ObjectIdentifier values.

### How can a struct containing a reference have the same mutable-key bug as a class?

Its stored reference can be copied while remaining shared. If the struct's equality or hash reads that object's mutable field, an external alias changes the state seen by an already stored key. The outer value declaration does not create deep independence.

### Why can insert retain stale content after an ID-only equality fix, and when should update(with:) or dictionary assignment be used?

Insert recognizes an existing equal member and can retain it, including its older non-identity content. Use update(with:) when replacing an equal Set member is intended, or store snapshots under stable dictionary keys and assign the new value. Membership and replacement are separate contracts.

### What does Dictionary.Keys hashing ignore, and why can EmptyCollection<NotHashable> now be Hashable?

Dictionary.Keys equality/hashing concerns the keys, independent of dictionary values and iteration order. EmptyCollection has no elements to compare/hash, so its new conformance need not require its Element to be Hashable. These additional conformances have SDK/OS availability; they do not relax Set's Copyable element requirement.

## Focused exercise

**Task:** Store snapshots under stable keys, then replace one with changed content. Check membership and the retained value after insertion versus replacement.

**Acceptance checks:**

- Equal-ID insertion preserves membership and reveals the retained old value.
- Explicit replacement retains the new content.
- Dictionary assignment keeps one stable ID key and updates its snapshot.

**Solution:**

Standalone Set and dictionary replacement probe:

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
let inserted = members.insert(new)
precondition(!inserted.inserted && inserted.memberAfterInsert.name == "old")
let replaced = members.update(with: new)
precondition(replaced?.name == "old" && members.first?.name == "new")
var byID = [old.id: old]
byID[new.id] = new
precondition(byID.count == 1 && byID[1]?.name == "new")
```

**Reasoning:** The immutable ID defines membership, while name is replaceable content. Exact hash integers are irrelevant. If structural snapshot equality is preferred, use a separate ID-keyed dictionary rather than forcing every snapshot to use entity equality.

## In practice

**Choose it when:** Separate lookup identity from mutable content in caches, UI stores, and deduplication.

**Investigate by:** Check equality laws, matching hash inputs, aliases, and retained values after each operation.

**Strong evidence:** Demonstrate valid behavior through replacement and aliasing boundaries without relying on one hash printout.

### New collection conformances — optional

#### New collection conformances preserve their own semantics

[SE-0514](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0514-hashable-conformance-for-dictionarykeys-collectionofone-emptycollection.md) adds Hashable to Dictionary.Keys, CollectionOfOne, and EmptyCollection. In this SDK their new conformances require macOS 27 / iOS 27. Dictionary.Keys equality/hash represents the set of keys, irrespective of dictionary values or iteration order. CollectionOfOne requires a Hashable element. EmptyCollection has no element values to hash and does not need its Element type to be Hashable.

```swift
struct NotHashable {}

@available(macOS 27.0, iOS 27.0, *)
func collectionCounts() -> [Int] {
    let first = ["a": 1, "b": 2]
    let second = ["b": 200, "a": 100]
    let keySets = Set([first.keys, second.keys])
    let singles = Set([CollectionOfOne(1), CollectionOfOne(1), CollectionOfOne(2)])
    let empty = Set([EmptyCollection<NotHashable>(), EmptyCollection<NotHashable>()])
    return [keySets.count, singles.count, empty.count]
}
```

Result: `[1, 2, 1]`. These counts do not claim a stable iteration order or durable hash encoding. Do not backdate API availability merely because the 6.4 compiler can parse the source. For older supported OS releases, `Set(dictionary.keys)` can supply an explicit set of copyable keys when that is the desired value, with its own materialization cost.

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
- [Hasher.finalize](https://developer.apple.com/documentation/swift/hasher/finalize()
- [Set source and API contracts](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Set.swift)
- [SE-0499: Noncopyable simple protocols](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)
