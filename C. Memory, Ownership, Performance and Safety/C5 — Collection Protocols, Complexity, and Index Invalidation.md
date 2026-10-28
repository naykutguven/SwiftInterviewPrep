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

# C5 — Collection Protocols, Complexity, and Index Invalidation

**Learning goals:**

- Choose Sequence, Collection, BidirectionalCollection, or RandomAccessCollection from both semantics and cost.
- Preserve generic indices and their mutation/snapshot lifetime rather than treating offsets as identity.
- Distinguish Iterable’s borrowed elements and typed failure from ordinary Sequence iteration and owned storage.

**Prerequisites:** [[C2 — Copy-on-write behavior of standard-library types]] for slices and value snapshots, [[B3 — Generics, Constraints, and Same-Type Relationships]] for generic constraints, and [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]/[[C8 — Copyable, noncopyable types, and `~Copyable`]] before the borrowing/noncopyable extension.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A for loop asks for traversal, but binary search also needs cheap jumps to a midpoint. These are different contracts. **Sequence** supplies iteration and can be single-pass or infinite. **Collection** is finite and multipass, with reusable indices for a valid unchanged collection. **BidirectionalCollection** adds backward movement. **RandomAccessCollection** promises constant-time index distance and offsetting.

None of those names promises integer indices or contiguous storage. An index identifies a position in the appropriate collection state; it is not a durable element ID. Choose the weakest protocol that preserves the algorithm’s behavior and the complexity you advertise. [Collection](https://developer.apple.com/documentation/swift/collection)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### A Sequence can consume shared iteration state

Standalone finite example. The iterator closure shares one cursor, deliberately making a second traversal empty.

```swift
final class Cursor { var nextValue = 1 }
let cursor = Cursor()
let source = AnySequence<Int> {
    AnyIterator {
        guard cursor.nextValue <= 3 else { return nil }
        defer { cursor.nextValue += 1 }
        return cursor.nextValue
    }
}
precondition(Array(source) == [1, 2, 3])
precondition(Array(source).isEmpty)
```

**Walkthrough:** Sequence alone does not promise that a fresh loop restarts the data. A consumer requiring repeatable indexed passes should ask for Collection or deliberately materialize an owned finite input. An infinite sequence cannot be materialized to completion.

### A generic Index need not be Int

Shared ForwardValues setup. Its custom Index wraps an offset without exposing integer-index arithmetic. StepLog counts calls to index(after:) so later examples can inspect traversal work.

```swift
final class StepLog { var steps = 0 }
struct ForwardValues<Element>: Collection {
    struct Index: Comparable {
        let offset: Int
        static func < (lhs: Self, rhs: Self) -> Bool { lhs.offset < rhs.offset }
    }
    let elements: [Element]
    let log: StepLog
    var startIndex: Index { Index(offset: 0) }
    var endIndex: Index { Index(offset: elements.count) }
    subscript(index: Index) -> Element { elements[index.offset] }
    func index(after index: Index) -> Index {
        log.steps += 1
        return Index(offset: index.offset + 1)
    }
}
let log = StepLog()
let values = ForwardValues(elements: [10, 20, 30], log: log)
let next = values.index(after: values.startIndex)
precondition(values[next] == 20)
precondition(Array(values) == [10, 20, 30])
```

**Walkthrough:** Generic code advances an Index using the collection API. ForwardValues intentionally has no random-access conformance; its default distance/offset operations traverse by repeated index(after:). The backing Array does not automatically grant the wrapper a stronger public contract.

### State binary search’s sortedness and index-cost requirements

Shared find setup. Input must already be sorted in ascending Comparable order. The helper returns C.Index, not an invented Int offset.

```swift
func find<C: RandomAccessCollection>(_ target: C.Element, in values: C) -> C.Index?
where C.Element: Comparable {
    var low = values.startIndex
    var high = values.endIndex
    while low != high {
        let middle = values.index(low, offsetBy: values.distance(from: low, to: high) / 2)
        if values[middle] < target { low = values.index(after: middle) } else { high = middle }
    }
    return low != values.endIndex && values[low] == target ? low : nil
}
let sorted = [10, 20, 20, 30]
precondition(find(20, in: sorted) == 1)
precondition(find(25, in: sorted) == nil)
```

**Walkthrough:** The half-open candidate interval shrinks toward the first value not less than target. The final equality check distinguishes a match. Random-access index operations make this logarithmic index/comparison work; the cost of comparing one Element remains additional. Sorting is a caller precondition, not something the type constraint proves.

## Rules and pitfalls

### Repeated offsets can turn one pass into quadratic traversal

**Common pitfall:** Requesting each successive position by offsetting again from startIndex on a forward-only collection. Prepend ForwardValues setup.

**Mistake:** Standalone runnable example.

```swift
func indexedSum<C: Collection>(_ values: C) -> Int where C.Element == Int {
    var sum = 0
    for offset in 0..<values.count {
        let index = values.index(values.startIndex, offsetBy: offset)
        sum += values[index]
    }
    return sum
}
let work = StepLog()
let source = ForwardValues(elements: [1, 2, 3, 4], log: work)
precondition(indexedSum(source) == 10)
precondition(work.steps > 4)
```

**Consequence:** The result is correct, but repeated forward jumps perform more traversal than the four-element pass requires.

**Why:** On a forward collection, reaching offset k can require k index advances. Repeating that from zero accumulates quadratic index work, in addition to any count traversal.

**Correction:** Express one forward pass as one forward pass. Prepend ForwardValues setup.

```swift
func sum<S: Sequence>(_ values: S) -> Int where S.Element == Int {
    var result = 0
    for value in values { result += value }
    return result
}
let work = StepLog()
let source = ForwardValues(elements: [1, 2, 3, 4], log: work)
precondition(sum(source) == 10)
precondition(work.steps == 4)
```

**What changes:** The algorithm needs only Sequence and traverses each element once. The counter observes calls to this fixture’s index operation, not elapsed time or allocations. Integer arithmetic is bounded by these small inputs.

### for-in syntax does not imply Sequence conformance

**Common pitfall:** Passing InlineArray to a Sequence-only helper merely because a for loop accepts its elements.

**Mistake:** Compile-only; intentionally rejected.

```swift
func sum<S: Sequence>(_ values: S) -> Int where S.Element == Int {
    values.reduce(0, +)
}
let inline: [2 of Int] = [1, 2]
let invalid = sum(inline)
```

**Consequence:** The compiler rejects the missing Sequence conformance in this SDK.

**Why:** The newer borrowing iteration surface is a separate protocol. Syntax acceptance does not silently add Sequence or Collection algorithms.

**Correction:** Use the actual borrowing contract when it is the operation you support. This standalone visitor borrows Int elements and preserves iteration failure.

```swift
func sum<I: Iterable & ~Copyable & ~Escapable>(_ values: borrowing I) throws(I.Failure) -> Int
where I.Element == Int {
    var result = 0
    for try value in values { result += value }
    return result
}
let inline: [2 of Int] = [1, 2]
precondition(sum(inline) == 3)
let array = [1, 2]
let view = array.span
precondition(sum(view) == 3)
```

**What changes:** InlineArray and Span satisfy the checked Iterable surface; these concrete failures are Never, so the calls need no try. Array itself remains usable by Sequence algorithms. The selected Iterable APIs require Apple OS 27.

## Worked example

### A valid integer index can identify the wrong element after mutation

**Setup:** Standalone Array example. The saved value is an index, not a domain ID.

**Predict:** Does an in-bounds saved index still identify the same record after an insertion?

```swift
struct Record { let id: Int }
var records = [Record(id: 10), Record(id: 20)]
let savedIndex = records.startIndex + 1
let savedID = records[savedIndex].id
records.insert(Record(id: 5), at: records.startIndex)
precondition(records[savedIndex].id == 10)
let current = records.firstIndex { $0.id == savedID }
precondition(current == 2 && records[current!].id == 20)
```

**Outcome:** The old numeric position now refers to record 10; looking up the saved ID finds record 20 at its new position.

**Why:** Structural edits change the relationship between positions and elements. This Array example uses an index that happens still to be in bounds; other invalidated indices must not be used at all. Recompute positions or use domain identity according to the operation.

## Interview questions

### Which protocol preserves an algorithm’s semantics and complexity?

**Answer:** Use Sequence for one pass, Collection for repeatable indexed traversal, BidirectionalCollection for backward movement, and RandomAccessCollection when the advertised cost needs constant-time distance/offset operations. Required sortedness and comparison cost are separate.

**Example:** Prepend find setup.

```swift
func total<S: Sequence>(_ values: S) -> Int where S.Element == Int {
    values.reduce(0, +)
}
let source = [0, 10, 20, 30]
let slice = source[1...3]
precondition(total(slice) == 60)
precondition(find(20, in: slice) == 2)
precondition(slice.index(before: slice.endIndex) == 3)
```

**Walkthrough:** total only consumes successive values. find needs efficient indexed midpoint work and preserves the slice’s original index 2. Backward traversal is available because this slice also has that stronger conformance.

**Boundary:** A stronger constraint can exclude useful callers without helping the algorithm. No protocol validates sorting or makes an expensive Comparable implementation constant-time.

Prepend find and ForwardValues setups. A forward-only collection cannot meet the advertised binary-search index-cost contract; this call is **intentionally rejected**:

```swift
let invalid = find(20, in: ForwardValues(elements: [10, 20, 30], log: StepLog()))
```

### Why can InlineArray support for-in without Sequence, while Array does not directly satisfy Iterable here?

**Answer:** The compiler can use borrowing iteration rather than Sequence.Iterator. The installed InlineArray exposes that surface, while Array uses its established protocols and an available span view for this borrowing visitor.

**Example:** Standalone runnable example.

```swift
struct Token: ~Copyable { let id: Int }
func iterationProbe() {
    let tokens: [2 of Token] = [Token(id: 1), Token(id: 2)]
    var ids: [Int] = []
    for token in tokens { ids.append(token.id) }
    precondition(ids == [1, 2] && tokens.count == 2)
    let view = tokens.span
    for token in view { ids.append(token.id) }
    precondition(ids == [1, 2, 1, 2] && tokens.count == 2)
}
iterationProbe()
```

**Walkthrough:** Both loops borrow the noncopyable elements and leave the owner intact. This does not make InlineArray a Sequence/Collection or transfer each token out. [Borrowing iteration](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md)

**Boundary:** These iteration APIs need Apple OS 27. Binding the span to view uses the checked lifetime form; a prior direct loop over tokens.span produced a lifetime diagnostic in this compiler. OS 26 indexing/subscripting is a separate available path.

### Which owner, element, and lifetime defaults must a generic borrowing visitor suppress?

**Answer:** Suppress Copyable and Escapable on the input I, borrow it, and suppress Copyable on I.Element. The Element declaration here still has its own supported lifetime constraints. Propagate I.Failure for obtaining elements instead of erasing it or assuming Never.

**Example:** Standalone runnable example.

```swift
func visit<I: Iterable & ~Copyable & ~Escapable>(
    _ values: borrowing I, _ body: (borrowing I.Element) -> Void
) throws(I.Failure) where I.Element: ~Copyable {
    for try value in values { body(value) }
}
struct Token: ~Copyable { let id: Int }
func visitorProbe() {
    var values = UniqueArray<Token>()
    values.append(Token(id: 1))
    values.append(Token(id: 2))
    var ids: [Int] = []
    visit(values) { ids.append($0.id) }
    let view = values.span
    visit(view) { ids.append($0.id) }
    precondition(ids == [1, 2, 1, 2] && values.count == 2)
}
visitorProbe()
```

**Walkthrough:** UniqueArray exercises a noncopyable owner and element; Span additionally exercises the nonescapable input. Both concrete Failure types are Never, so no try is needed at these calls even though the generic loop needs for try.

**Boundary:** The callback here cannot throw; iteration failure and callback failure would need a combined contract. This example checks the generic throwing signature and nonthrowing concrete cases, not a custom failing iterator runtime. Do not consume a borrowed element or resize its owner during its live access.

Standalone **intentionally rejected** generic omission of try; an arbitrary Iterable can fail while producing elements:

```swift
func invalid<I: Iterable & ~Copyable & ~Escapable>(_ values: borrowing I)
    throws(I.Failure) where I.Element: ~Copyable
{
    for value in values { _ = value }
}
```

Prepend visit. Passing Array directly is **intentionally rejected**; bind values.span when that borrowing path is required:

```swift
func invalidArray() { visit([1, 2]) { _ in } }
```

## Focused exercise

**Task:** Search a sorted ArraySlice while returning its original index and respecting the generic complexity contract.

**Acceptance checks:**

- Boundary matches return slice indices 1 and 3; a duplicate returns the first matching index.
- Missing and empty inputs return nil.
- Sortedness is explicit, and index work is distinguished from element comparison cost.

**Solution:** Prepend find setup. All supplied inputs are sorted:

```swift
let source = [0, 10, 20, 30, 40]
let slice = source[1...3]
precondition(find(10, in: slice) == 1)
precondition(find(30, in: slice) == 3)
precondition(find(25, in: slice) == nil)
precondition(find(5, in: slice) == nil)
precondition(find(50, in: slice) == nil)
precondition(find(10, in: source[0..<0]) == nil)
precondition(find(20, in: [10, 20, 20, 30]) == 1)
```

**Reasoning:** The half-open interval and original Index values avoid zero-based assumptions. RandomAccessCollection supplies the midpoint-cost guarantee; validating or sorting arbitrary input would add work and can change what returned indices mean.

## In practice

**Choose it when:** Select constraints from the operations and cost guarantee you actually expose.

**Investigate by:** Test single-pass input, non-integer/nonzero indices, structural edits, and instrumented traversal work before discussing elapsed time.

**Strong evidence:** Preserve index validity and borrowing lifetimes while distinguishing source algorithms, protocol complexity guarantees, and measured runtime costs.

## References

### Related guides

- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C4 — String and Unicode correctness]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]

### Official sources

- [Collection](https://developer.apple.com/documentation/swift/collection)
- [SE-0516](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md)
- [SE-0514: Additional collection Hashable conformances](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0514-hashable-conformance-for-dictionarykeys-collectionofone-emptycollection.md)
- [GitHub. "RandomAccessCollection.swift." swiftlang/swift](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/RandomAccessCollection.swift)
- [Apple Developer. "count." Apple Developer Documentation](https://developer.apple.com/documentation/swift/collection/count)
- [Apple Developer. "insert(_:at:)." Apple Developer Documentation](https://developer.apple.com/documentation/swift/array/insert%28_%3aat%3a%29-88yqz)
