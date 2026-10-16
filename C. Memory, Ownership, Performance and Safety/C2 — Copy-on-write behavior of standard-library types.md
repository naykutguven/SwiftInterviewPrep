---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - copy-on-write
---

# C2 — Copy-on-Write Behavior of Standard-Library Types

**Goal:** Mutate a nonzero-index ArraySlice while preserving the source and an owned copy.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Array copies are logically independent even when storage is shared. Mutation must preserve other array values and may copy shared storage, materialize a representation, or grow capacity. Unique storage alone does not guarantee allocation-free append. ArraySlice preserves a subrange's original indices and can retain the base allocation beyond its visible elements. [ArraySlice](https://developer.apple.com/documentation/swift/arrayslice)

## Rules and pitfalls

### Separate container values from element identity

A copied array of class elements still contains references to the same objects. Replacing an element edits one sequence; mutating a referenced object changes what all aliases observe. An owned Array(slice) bounds the new collection's retention but still retains references to its visible class elements.

**Common pitfall:** Keeping a tiny slice as long-lived UI state can keep a large base buffer and invisible elements alive. Materialize an owned copy when that retention matters, then measure the actual memory tradeoff.

### Audit every custom CoW mutation path

A private reference storage object can own an Array that itself shares a buffer. Outer uniqueness does not establish inner uniqueness. Mutation must preserve logical independence through setters, inout projections, append/remove, and storage exposure.

**Common pitfall:** isKnownUniquelyReferenced is neither a lock nor an Array-buffer inspection API. Confine and synchronize access separately; see [[C3 — Exclusivity enforcement and `inout`]].

## Interview questions

### Why can an element write or append require O(n) work even though array passing can be cheap?

An element write can require separating shared storage; append can also require growth or representation conversion. Both can copy O(n) elements even when passing or assigning an array often shares storage cheaply. Element operation costs remain additional to the container's complexity.

### How can a tiny ArraySlice retain invisible elements, and what does Array(slice) still retain?

A slice can retain the entire backing allocation, including invisible elements. Array(slice) creates an owned sequence of the visible values and rebases indices, but visible class elements still retain their objects and reachable graphs. It is not a deep clone.

### Can a unique outer storage object contain a shared Array buffer, and which mutation paths must preserve independence?

Yes. A unique wrapper storage object can contain an Array value sharing storage with a snapshot. Its mutation must rely on Array's own value behavior as well as the wrapper's uniqueness policy. Audit every exposed mutation path and prevent mutable storage references from escaping.

## Focused exercise

**Task:** Mutate an ArraySlice and compare the source value. Explain its retained base storage and check nonzero indices and an owned slice copy.

**Acceptance checks:**

- Slice indices remain nonzero and valid for that slice.
- Slice edits preserve the source and earlier owned copy.
- An empty end slice is handled without assuming index zero.

**Solution:**

Standalone slice/value probe:

```swift
let source = [10, 20, 30, 40]
var slice = source[1...2]
precondition(slice.startIndex == 1)
let owned = Array(slice)
slice[slice.startIndex] = 99
precondition(source == [10, 20, 30, 40])
precondition(Array(slice) == [99, 30])
precondition(owned == [20, 30] && owned.startIndex == 0)
let empty = source[source.endIndex..<source.endIndex]
precondition(empty.isEmpty)
```

Use the retention probe below for object lifetime; these integer results establish values and indices, not allocation counts.

**Reasoning:** Slice shares a value-semantic subsequence contract while keeping base indices. Mutation preserves the original array. Materializing Array changes ownership/retention and index origin, so code must not keep using old slice indices against the new array.

## In practice

**Choose it when:** Use slices for temporary views; choose owned copies when long-lived retention warrants the copy.

**Investigate by:** Retain old values for semantic tests and use memory/allocation tools for cost evidence.

**Strong evidence:** Compare retention and optimized workload cost under the required ownership contract; UniqueArray is a different contract, not a drop-in snapshot optimization.

### Retention observations — optional

```swift
final class Item {
    let id: Int
    init(_ id: Int) { self.id = id }
}

struct WeakItem {
    weak var value: Item?
    init(_ value: Item) { self.value = value }
}

final class RetentionFixture {
    let probes: [WeakItem]
    var slice: ArraySlice<Item>?
    var owned: [Item]?

    @inline(never)
    init(keepSlice: Bool) {
        let items = (0..<3).map(Item.init)
        probes = items.map(WeakItem.init)
        if keepSlice {
            slice = items[1..<2]
        } else {
            owned = Array(items[1..<2])
        }
    }

    func alive() -> [Bool] { probes.map { $0.value != nil } }
}

func retentionProbe() {
    let sliced = RetentionFixture(keepSlice: true)
    precondition(sliced.slice?.map(\.id) == [1])
    print(sliced.alive())
    sliced.slice = nil
    print(sliced.alive())

    let copied = RetentionFixture(keepSlice: false)
    precondition(copied.owned?.map(\.id) == [1])
    print(copied.alive())
    copied.owned = nil
    print(copied.alive())
}
```

Calling retentionProbe in the checked debug and optimized configurations prints:

> [true, true, true]
> [false, false, false]
> [false, true, false]
> [false, false, false]

Only Item 1 is visible through either stored collection. The slice fixture retains all three objects after initialization returns. The owned selected array retains only Item 1. Clearing the respective stored collection lets those remaining objects deinitialize in this fixture. The weak probes observe liveness without keeping the items alive themselves.

This replaces the old claim that item deinitializers necessarily print in a particular index order. The example uses a noninlined initializer as a bounded setup function and checks states after it returns; it does not establish a general lexical-scope release rule, deinitialization ordering, resident-memory reduction, or byte allocation count. Pure Swift Item objects are used; no autorelease/bridging behavior is inferred.

For a real image-result incident, inspect the retaining path and compare before/after memory evidence. A small visible count is not evidence of a small retained object graph. Other owners, selected payloads, caches, autorelease pools, and the allocator can keep memory resident after a slice has been removed.

## References

### Related guides

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[C1 — ARC fundamentals]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[E2 — Foundation and Objective-C Bridging Behavior]]
- [[F3 — Performance investigation habits]]

### Official sources

- [ArraySlice](https://developer.apple.com/documentation/swift/arrayslice)
- [Swift Array documentation/source](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift)
- [Swift ArraySlice documentation/source](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/ArraySlice.swift)
- [isKnownUniquelyReferenced documentation/source](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/ManagedBuffer.swift)
- [SE-0377: Borrowing and consuming parameter modifiers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)
- [SE-0527: Unique array proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0527-rigidarray-uniquearray.md)
