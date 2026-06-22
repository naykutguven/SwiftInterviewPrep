---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - performance
  - safety
  - collections
---
## 0. Rubric snapshot

**Rubric expectation**

Understand `Sequence` vs `Collection` vs `BidirectionalCollection` vs `RandomAccessCollection`, and the complexity contracts they imply.

**Caveats**

Not every collection supports cheap indexing. Mutating a collection can invalidate indices.

**You should be able to answer**

- Why is taking an `Array` parameter more restrictive than taking a `Collection`?
- What assumptions are safe only for `RandomAccessCollection`?

**You should be able to do**

- Refactor an algorithm to accept the weakest useful collection constraint without losing the required complexity.

---

## 1. Core mental model

Swift’s collection protocols are not just “containers.” They are **semantic and complexity contracts**.

`Sequence` is the weakest abstraction. It means values can be produced one after another. It does **not** promise indexing, finiteness, repeatable traversal, or cheap counting. A `Sequence` can be single-pass, lazy, destructive, or infinite. Use it when your algorithm only needs one forward pass.

`Collection` strengthens `Sequence`. It promises a finite, multipass collection whose elements can be accessed through indices. Apple’s documentation describes a collection as multipass and index-subscriptable; unlike a plain sequence, elements can be repeatedly accessed by saving an index. ([Apple Developer](https://developer.apple.com/documentation/swift/collection?utm_source=chatgpt.com "Collection | Apple Developer Documentation"))

`BidirectionalCollection` adds the ability to move backward from an index. This matters for algorithms needing reverse traversal, `last`, suffix scanning, or two-pointer movement from both ends.

`RandomAccessCollection` adds the key performance contract: moving an index by an arbitrary distance and measuring distance between indices are `O(1)`. Apple’s docs explicitly distinguish random-access collections by constant-time index movement and distance measurement; `count` is also `O(1)` for random-access collections instead of requiring traversal. ([GitHub](https://github.com/apple/swift/blob/main/stdlib/public/core/RandomAccessCollection.swift?utm_source=chatgpt.com "swift/stdlib/public/core/RandomAccessCollection.swift at main"))

The key idea:

```text
Sequence → one-pass iteration
Collection → finite + multipass + indexed
BidirectionalCollection → can move backward
RandomAccessCollection → index distance/offset is O(1)
```

The staff-level point: choose the weakest protocol that still preserves your algorithm’s **correctness and Big-O complexity**. Too concrete makes APIs unnecessarily narrow. Too weak silently makes algorithms slower or semantically wrong.

---

## 2. Essential mechanics

### Protocol hierarchy encodes capability

```swift
func logAll<S: Sequence>(_ values: S) {
    for value in values {
        print(value)
    }
}
```

This only needs a single pass, so `Sequence` is enough.

```swift
func firstAndLast<C: BidirectionalCollection>(_ values: C) -> (C.Element, C.Element)? {
    guard let first = values.first, let last = values.last else {
        return nil
    }

    return (first, last)
}
```

This needs access to both ends, so `BidirectionalCollection` is more honest than `Sequence`.

```swift
func binarySearch<C: RandomAccessCollection>(
    _ collection: C,
    for target: C.Element
) -> C.Index? where C.Element: Comparable {
    var lower = collection.startIndex
    var upper = collection.endIndex

    while lower != upper {
        let distance = collection.distance(from: lower, to: upper)
        let middle = collection.index(lower, offsetBy: distance / 2)
        let value = collection[middle]

        if value == target {
            return middle
        } else if value < target {
            lower = collection.index(after: middle)
        } else {
            upper = middle
        }
    }

    return nil
}
```

Binary search should require `RandomAccessCollection`, not merely `Collection`, because the algorithm’s expected `O(log n)` behavior depends on cheap distance and offset operations.

### `Collection` does not mean integer indexing

A generic collection’s `Index` is an associated type. It is not necessarily `Int`.

```swift
func firstElement<C: Collection>(_ collection: C) -> C.Element? {
    guard !collection.isEmpty else {
        return nil
    }

    return collection[collection.startIndex]
}
```

This is correct.

Bad:

```swift
func firstElement<C: Collection>(_ collection: C) -> C.Element? {
    collection[0] // Wrong mental model: C.Index is not necessarily Int.
}
```

Even `ArraySlice` demonstrates the trap:

```swift
let array = [10, 20, 30, 40]
let slice = array[1...2]

print(slice.startIndex)
print(slice[slice.startIndex])
```

Exact output:

```text
1
20
```

The slice starts at index `1`, not `0`. This is why generic APIs should return `C.Index`, not `Int`, unless they truly require integer positions.

### `count` and offsetting are not always cheap

For `Collection`, `count` can be `O(n)`. Apple’s documentation for `Collection.count` says `count` is `O(1)` only when the collection conforms to `RandomAccessCollection`; otherwise it can be `O(n)`. ([Apple Developer](https://developer.apple.com/documentation/swift/collection/count?utm_source=chatgpt.com "count | Apple Developer Documentation"))

Bad:

```swift
func everyOther<C: Collection>(_ collection: C) -> [C.Element] {
    var result: [C.Element] = []

    for offset in 0..<collection.count {
        if offset.isMultiple(of: 2) {
            let index = collection.index(collection.startIndex, offsetBy: offset)
            result.append(collection[index])
        }
    }

    return result
}
```

This looks innocent, but for a non-random-access collection, repeated offsetting from `startIndex` can make it much worse than linear.

Better:

```swift
func everyOther<C: Collection>(_ collection: C) -> [C.Element] {
    var result: [C.Element] = []
    var index = collection.startIndex
    var shouldTake = true

    while index != collection.endIndex {
        if shouldTake {
            result.append(collection[index])
        }

        shouldTake.toggle()
        collection.formIndex(after: &index)
    }

    return result
}
```

This walks indices forward once. It works with a weaker constraint and preserves linear complexity.

### Mutating can invalidate indices

Indices are only valid under the rules of the specific collection and mutation operation. Apple’s `Collection` documentation notes that saved indices may become invalid after mutating operations. ([Apple Developer](https://developer.apple.com/documentation/Swift/Collection?utm_source=chatgpt.com "Collection | Apple Developer Documentation")) Apple’s `Array.insert(_:at:)` documentation also warns that calling the method may invalidate existing indices for that collection. ([Apple Developer](https://developer.apple.com/documentation/swift/array/insert%28_%3Aat%3A%29-88yqz?utm_source=chatgpt.com "insert(_:at:) | Apple Developer Documentation"))

Bad:

```swift
var messages = ["A", "B", "C"]

let index = messages.firstIndex(of: "B")!
messages.insert("New", at: messages.startIndex)

// Do not assume `index` still identifies the same logical element.
print(messages[index])
```

Better:

```swift
var messages = ["A", "B", "C"]

messages.insert("New", at: messages.startIndex)

if let index = messages.firstIndex(of: "B") {
    print(messages[index])
}
```

After structural mutation, recompute indices unless the collection’s documentation gives a stronger guarantee.

---

## 3. Common traps and misconceptions

### Trap 1: “Generic over `Collection` is always better than `Array`”

Not if your algorithm needs random access.

Bad:

```swift
func median<C: Collection>(_ sortedValues: C) -> C.Element? {
    guard !sortedValues.isEmpty else {
        return nil
    }

    let middle = sortedValues.index(
        sortedValues.startIndex,
        offsetBy: sortedValues.count / 2
    )

    return sortedValues[middle]
}
```

This compiles, but `count` and `index(_:offsetBy:)` may be linear for non-random-access collections.

Better:

```swift
func median<C: RandomAccessCollection>(_ sortedValues: C) -> C.Element? {
    guard !sortedValues.isEmpty else {
        return nil
    }

    let middle = sortedValues.index(
        sortedValues.startIndex,
        offsetBy: sortedValues.count / 2
    )

    return sortedValues[middle]
}
```

The constraint documents and preserves the intended complexity.

### Trap 2: Assuming indices are stable logical IDs

An index is a position in a specific collection state. It is not a durable identity.

Bad:

```swift
struct Selection {
    var selectedIndex: Array<String>.Index
}
```

This can be okay for temporary internal state, but it is fragile across insertions, removals, sorting, filtering, pagination, and data reloads.

Better for app state:

```swift
struct Message: Identifiable {
    let id: UUID
    var text: String
}

struct Selection {
    var selectedMessageID: Message.ID
}
```

Store domain identity, not a collection index, when the selection must survive structural changes.

### Trap 3: Treating `RandomAccessCollection` as “array-like storage”

`RandomAccessCollection` does **not** mean:

```text
Index == Int
startIndex == 0
storage is contiguous
mutation is cheap
elements are stored in an Array
```

It only promises efficient index movement and distance measurement. If you need contiguous memory, that is a different requirement.

---

## 4. Direct answers to rubric questions

### Q1. Why is taking an `Array` parameter more restrictive than taking a `Collection`?

Taking `Array` requires callers to pass a specific concrete storage type. Taking `Collection`, `Sequence`, `BidirectionalCollection`, or `RandomAccessCollection` lets callers pass other valid collection types without materializing an array.

For example, this is unnecessarily narrow:

```swift
func renderTitles(_ titles: [String]) {
    for title in titles {
        print(title)
    }
}
```

The function only iterates. It should be:

```swift
func renderTitles<S: Sequence>(_ titles: S) where S.Element == String {
    for title in titles {
        print(title)
    }
}
```

Or, if it needs `isEmpty`, `count`, indices, or repeated traversal:

```swift
func renderTitles<C: Collection>(_ titles: C) where C.Element == String {
    guard !titles.isEmpty else {
        return
    }

    for title in titles {
        print(title)
    }
}
```

Concrete `Array` parameters are fine when you specifically need array semantics: storage ownership, mutation, random access with array-specific behavior, Objective-C bridging, or API simplicity at a boundary.

Interview version:

> Taking `Array` bakes in a concrete representation. If the algorithm only needs iteration or indexed collection behavior, a protocol constraint such as `Sequence`, `Collection`, or `RandomAccessCollection` is more flexible and avoids forcing callers to allocate or convert. The key is not to make everything generic blindly; the constraint should match the weakest capability that still preserves correctness and complexity.

### Q2. What assumptions are safe only for `RandomAccessCollection`?

You can assume constant-time distance measurement and arbitrary index offsetting. That makes algorithms like binary search, midpoint access, partitioning by index ranges, and repeated indexed jumps efficient. Apple’s standard library documentation describes random-access collections as supporting `O(1)` index movement and distance measurement. ([GitHub](https://github.com/apple/swift/blob/main/stdlib/public/core/RandomAccessCollection.swift?utm_source=chatgpt.com "swift/stdlib/public/core/RandomAccessCollection.swift at main"))

You cannot assume:

```text
Index == Int
startIndex == 0
contiguous memory
cheap insertion/removal
stable indices after mutation
sortedness
thread safety
```

Interview version:

> `RandomAccessCollection` is the right constraint when the algorithm’s Big-O depends on jumping around by index. It gives me constant-time offset and distance operations, so binary search can remain logarithmic. But it is not the same as `Array`: the index may not be `Int`, the storage may not be contiguous, and mutation can still invalidate indices.

---

## 5. Code probe

C5 has no rubric code probe. Minimal examples, counterexample, and production example are included instead.

### Minimal example: weakest useful constraint

```swift
func containsZero<S: Sequence>(_ values: S) -> Bool where S.Element == Int {
    for value in values {
        if value == 0 {
            return true
        }
    }

    return false
}

print(containsZero([1, 2, 0, 3]))
```

Exact output:

```text
true
```

Why this is correct:

```text
The algorithm only needs one forward pass.
Therefore Sequence is enough.
Requiring Array or Collection would be unnecessarily restrictive.
```

### Counterexample: too weak for the intended complexity

```swift
func binarySearch<C: Collection>(
    _ collection: C,
    for target: C.Element
) -> C.Index? where C.Element: Comparable {
    var lower = collection.startIndex
    var upper = collection.endIndex

    while lower != upper {
        let distance = collection.distance(from: lower, to: upper)
        let middle = collection.index(lower, offsetBy: distance / 2)

        if collection[middle] == target {
            return middle
        } else if collection[middle] < target {
            lower = collection.index(after: middle)
        } else {
            upper = middle
        }
    }

    return nil
}
```

What happens:

```text
This can compile, but the generic constraint is misleading.
For non-random-access collections, distance and offset operations may be linear.
The algorithm no longer has the expected O(log n) complexity.
```

Better:

```swift
func binarySearch<C: RandomAccessCollection>(
    _ collection: C,
    for target: C.Element
) -> C.Index? where C.Element: Comparable {
    var lower = collection.startIndex
    var upper = collection.endIndex

    while lower != upper {
        let distance = collection.distance(from: lower, to: upper)
        let middle = collection.index(lower, offsetBy: distance / 2)
        let value = collection[middle]

        if value == target {
            return middle
        } else if value < target {
            lower = collection.index(after: middle)
        } else {
            upper = middle
        }
    }

    return nil
}
```

### Production example: preserving generic indices

```swift
struct SearchResult<Index> {
    let index: Index
}

func findFirstVisibleItem<C: Collection>(
    in items: C,
    isVisible: (C.Element) -> Bool
) -> SearchResult<C.Index>? {
    guard let index = items.firstIndex(where: isVisible) else {
        return nil
    }

    return SearchResult(index: index)
}
```

This returns `C.Index`, not `Int`, because generic collections are not necessarily integer-indexed.

---

## 6. Exercise

### Problem

Refactor an algorithm to accept the weakest useful collection constraint without losing the required complexity.

### Bad / naive version

```swift
func binarySearch(
    _ values: [Int],
    for target: Int
) -> Int? {
    var lower = values.startIndex
    var upper = values.endIndex

    while lower != upper {
        let middle = lower + (upper - lower) / 2

        if values[middle] == target {
            return middle
        } else if values[middle] < target {
            lower = middle + 1
        } else {
            upper = middle
        }
    }

    return nil
}
```

### What is wrong?

```text
The algorithm only accepts Array<Int>.
It excludes ArraySlice<Int>, ContiguousArray<Int>, and other random-access collections.
It returns Int, which assumes array-style integer indices.
The algorithm's real requirement is not Array; it is sorted random-access traversal.
```

### Improved version

```swift
func binarySearch<C: RandomAccessCollection>(
    _ collection: C,
    for target: C.Element
) -> C.Index? where C.Element: Comparable {
    var lower = collection.startIndex
    var upper = collection.endIndex

    while lower != upper {
        let distance = collection.distance(from: lower, to: upper)
        let middle = collection.index(lower, offsetBy: distance / 2)
        let value = collection[middle]

        if value == target {
            return middle
        } else if value < target {
            lower = collection.index(after: middle)
        } else {
            upper = middle
        }
    }

    return nil
}
```

Usage:

```swift
let values = [1, 3, 5, 7, 9]
let index = binarySearch(values, for: 7)

print(index!)
print(values[index!])
```

Exact output:

```text
3
7
```

Array slice usage:

```swift
let values = [1, 3, 5, 7, 9]
let slice = values[1...]

let index = binarySearch(slice, for: 7)

print(index!)
print(slice[index!])
```

Exact output:

```text
3
7
```

The returned index is `3`, not `2`, because the slice preserves the base array’s indices.

### Why this is better

The improved version accepts the weakest useful constraint: `RandomAccessCollection`.

It is weaker than `Array`, because it accepts more input types. It is stronger than `Collection`, because the algorithm’s expected complexity depends on random-access index operations.

```text
Too concrete:
Array<Int>                  → unnecessarily narrow

Too weak:
Collection                  → may lose O(log n)

Correct:
RandomAccessCollection      → flexible + preserves complexity
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`[Int]`|App-level convenience API where callers naturally have arrays|Simple, but unnecessarily narrow|
|`Collection`|Single-pass or forward-index algorithms where linear traversal is fine|Not suitable for binary search complexity|
|`RandomAccessCollection`|Algorithms relying on midpoint jumps, distance, or offset|More restrictive than `Collection`, but honestly preserves Big-O|
|Overload both|Public API that wants convenience plus generic performance|More API surface to maintain|

---

## 7. Production guidance

Use this in production when:

```text
Use Sequence when you only need one forward pass.
Use Collection when you need finiteness, multipass traversal, indices, isEmpty, or firstIndex.
Use BidirectionalCollection when you need reverse traversal or two-ended movement.
Use RandomAccessCollection when your complexity depends on O(1) distance or offset operations.
Use concrete Array when you truly need Array storage, mutation behavior, bridging, or API simplicity.
```

Be careful when:

```text
You call count inside loops.
You repeatedly call index(_:offsetBy:) from startIndex.
You store indices across mutations.
You assume generic indices are Int.
You return Int positions from generic collection algorithms.
You use ArraySlice and expect startIndex == 0.
```

Avoid when:

```text
Avoid Collection if Sequence is enough.
Avoid RandomAccessCollection if a single pass would work.
Avoid Array in library APIs unless concrete storage is part of the contract.
Avoid storing collection indices as long-lived model identity.
Avoid using generic Collection while secretly depending on array-like complexity.
```

Debugging checklist:

```text
Is this algorithm accidentally O(n²)?
Does this function really need Array, or only Sequence/Collection?
Does this function need RandomAccessCollection to preserve Big-O?
Are saved indices reused after insert/remove/sort/filter?
Are we assuming startIndex is zero?
Are we returning Int where we should return C.Index?
Are we materializing arrays just to satisfy an overly concrete API?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `Array` is a collection, and `Collection` is more generic. `RandomAccessCollection` is faster for indexing.

### Senior answer

> I choose collection constraints based on the operations the algorithm needs. If I only iterate once, I use `Sequence`. If I need multipass indexed access, I use `Collection`. If I need reverse traversal, I use `BidirectionalCollection`. If the algorithm depends on cheap distance or offset, like binary search, I require `RandomAccessCollection`.

### Staff-level answer

> Collection protocol choices are API design and performance contracts. I avoid concrete `Array` in reusable APIs unless storage is part of the promise, but I also avoid weakening constraints so much that the advertised complexity becomes false. For public APIs, I document complexity, return generic indices instead of `Int`, avoid persisting indices across mutation, and design overloads only when they materially improve usability or performance.

Staff-level questions to ask:

```text
What is the weakest protocol that preserves this algorithm's correctness?
What is the weakest protocol that preserves this algorithm's Big-O complexity?
Does this API force callers to allocate an Array unnecessarily?
Are returned indices meaningful outside the immediate collection state?
Should the API return C.Index, an offset, or a domain identifier?
Does the public documentation state complexity honestly?
```

---

## 9. Interview-ready summary

Swift collection protocols encode capability and complexity. `Sequence` is for one-pass iteration. `Collection` adds finite, multipass indexed access. `BidirectionalCollection` adds backward movement. `RandomAccessCollection` adds constant-time distance and offset operations, which is what algorithms like binary search need. Taking `Array` is often too concrete because it forces a specific storage type, but taking plain `Collection` can be too weak if the algorithm relies on random access. A strong Swift engineer chooses the weakest constraint that still preserves semantics and complexity, avoids assuming `Index == Int`, and treats indices as valid only for a specific collection state.

---

## 10. Flashcards

Q: What does `Sequence` guarantee?

A: It guarantees values can be iterated. It does not guarantee indexing, finiteness, multipass traversal, or cheap counting.

Q: What does `Collection` add over `Sequence`?

A: Finite, multipass traversal and indexed subscript access using the collection’s `Index` type.

Q: What does `BidirectionalCollection` add?

A: The ability to move backward from an index, enabling reverse traversal and efficient access from the end.

Q: What does `RandomAccessCollection` add?

A: Constant-time distance measurement and arbitrary index offsetting, preserving complexity for algorithms that jump by index.

Q: Why is `Array` more restrictive than `Collection`?

A: `Array` requires a concrete storage type. `Collection` allows any conforming type that satisfies the required semantics.

Q: Why should generic collection algorithms return `C.Index` instead of `Int`?

A: Generic collection indices are not necessarily integers, and even integer-indexed slices may not start at zero.

Q: When is `Collection` too weak?

A: When the algorithm’s expected complexity depends on random index jumps, such as binary search.

Q: Why is storing collection indices across mutation dangerous?

A: Structural mutations can invalidate indices or make them refer to different logical elements.

---

## 11. Related sections

- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C4 — String and Unicode correctness]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[B3 — Generics, constraints, and same-type relationships]]
- [[B10 — API design guidelines and Swift-native surface design]]

---

## 12. Sources

- Swift Senior/Staff Rubric, C5 — Collection protocols, complexity, and index invalidation.
- Apple Developer Documentation — `Collection`: multipass, indexed collection semantics. ([Apple Developer](https://developer.apple.com/documentation/swift/collection?utm_source=chatgpt.com "Collection | Apple Developer Documentation"))
- Swift standard library documentation/source — `RandomAccessCollection`: `O(1)` distance and index movement. ([GitHub](https://github.com/apple/swift/blob/main/stdlib/public/core/RandomAccessCollection.swift?utm_source=chatgpt.com "swift/stdlib/public/core/RandomAccessCollection.swift at main"))
- Apple Developer Documentation — `Collection.count`: `O(1)` for `RandomAccessCollection`, otherwise potentially `O(n)`. ([Apple Developer](https://developer.apple.com/documentation/swift/collection/count?utm_source=chatgpt.com "count | Apple Developer Documentation"))
- Apple Developer Documentation — collection indices may become invalid after mutation. ([Apple Developer](https://developer.apple.com/documentation/Swift/Collection?utm_source=chatgpt.com "Collection | Apple Developer Documentation"))
- Apple Developer Documentation — `Array.insert(_:at:)` may invalidate existing indices. ([Apple Developer](https://developer.apple.com/documentation/swift/array/insert%28_%3Aat%3A%29-88yqz?utm_source=chatgpt.com "insert(_:at:) | Apple Developer Documentation"))