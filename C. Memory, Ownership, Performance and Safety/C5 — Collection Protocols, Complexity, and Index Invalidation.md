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

**Goal:** Search a sorted ArraySlice while returning its original collection index.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Collection protocols describe traversal and complexity, not just available syntax. Sequence can be single-pass or infinite. Collection is finite and multipass with indices; BidirectionalCollection adds backward traversal; RandomAccessCollection gives constant-time index distance and offsetting under its contract. None implies contiguous storage or integer indices. [Collection](https://developer.apple.com/documentation/swift/collection)

## Rules and pitfalls

### Choose constraints from the algorithm

Binary search needs sorted input and efficient midpoint/index operations to justify logarithmic index work. Use RandomAccessCollection for that guarantee and return its Index rather than inventing an Int offset.

**Common pitfall:** Repeatedly offsetting from startIndex on a forward collection can erase the intended complexity gain. Separate comparisons, traversal, and element-comparison cost when explaining the algorithm.

### Respect index and iteration lifetimes

Structural mutation can invalidate saved indices; an index is not a durable element ID. Swift 6.4 Iterable supports borrowing iteration independently from Sequence/Collection, including noncopyable elements and typed iteration failure.

**Common pitfall:** Assuming every for-in source conforms to Sequence, or that Array directly conforms to Iterable in this SDK, hides actual constraints. Use the declared iteration surface and suppress Copyable/Escapable defaults only where the borrowed contract requires it.

## Interview questions

### Which protocol preserves this algorithm's semantics and complexity?

Use the weakest protocol that preserves both required operations and promised cost. One forward pass can use Sequence; indexed repeated access needs Collection; logarithmic binary-search index work needs RandomAccessCollection plus sorted input. A stronger protocol may still be inappropriate if the algorithm does not benefit from it.

### Why can InlineArray support for-in without Sequence conformance, while Array does not directly satisfy Iterable in this SDK?

For-in can use the new borrowing iteration machinery rather than Sequence.Iterator. InlineArray exposes that supported surface. Array's SDK surface uses its established protocols and borrowing adapters rather than direct Iterable conformance. Check conformance and adapter availability separately from syntax acceptance.

### Which owner, element, and lifetime defaults must a generic borrowing visitor suppress?

A truly general borrowing visitor must account for a possibly noncopyable/non-escapable owner and element, and for the iterator/view's dependent lifetime. Suppress the relevant defaults in each type position that supports them, borrow instead of taking ownership, and propagate typed iteration failure as required.

## Focused exercise

**Task:** Search a sorted ArraySlice while preserving original indices. Check empty, missing, and boundary matches; explain the required collection constraint.

**Acceptance checks:**

- Boundary matches preserve ArraySlice indices 1 and 3.
- Empty and missing inputs return nil.
- Sorted input is a declared precondition; comparison cost is separate.

**Solution:**

Standalone sorted-input binary search returning the original index:

```swift
func find<C: RandomAccessCollection>(
    _ target: C.Element, in values: C
) -> C.Index? where C.Element: Comparable {
    var low = values.startIndex
    var high = values.endIndex
    while low != high {
        let middle = values.index(low, offsetBy: values.distance(from: low, to: high) / 2)
        if values[middle] < target { low = values.index(after: middle) }
        else { high = middle }
    }
    return low != values.endIndex && values[low] == target ? low : nil
}
let source = [0, 10, 20, 30, 40]
let slice = source[1...3]
precondition(find(10, in: slice) == 1)
precondition(find(30, in: slice) == 3)
precondition(find(25, in: slice) == nil)
precondition(find(10, in: source[0..<0]) == nil)
```

**Reasoning:** The half-open search interval shrinks without integer-index assumptions. Random-access midpoint work is constant time, yielding logarithmic comparisons/index work. It does not make an expensive Comparable implementation cheap or validate sortedness without additional work.

## In practice

**Choose it when:** Preserve an algorithm's semantics and complexity through explicit constraints.

**Investigate by:** Test nonzero indices, mutation invalidation, and expensive index/element operations.

**Strong evidence:** Distinguish protocol guarantees, borrowing lifetimes, and measured workload cost.

### Borrowing and throwing iteration — optional

#### Borrowing and throwing iteration in Swift 6.4

The visitor below borrows each element and preserves the iteration failure type. Suppress defaults on both the input type and its Element: a noncopyable owner, nonescapable Span, and noncopyable element each exercise a different constraint. The body here cannot throw; `throws(I.Failure)` describes failure while obtaining elements.

```swift
@available(anyAppleOS 27.0, *)
func visit<I: Iterable & ~Copyable & ~Escapable>(
    _ values: borrowing I,
    _ body: (borrowing I.Element) -> Void
) throws(I.Failure) where I.Element: ~Copyable {
    for try value in values {
        body(value)
    }
}

struct Token: ~Copyable { let id: Int }
@available(anyAppleOS 27.0, *)
func demo() {
    var tokens = UniqueArray<Token>()
    tokens.append(Token(id: 1))
    tokens.append(Token(id: 2))
    visit(tokens) { print($0.id) }
    let view = tokens.span
    visit(view) { print($0.id) }
    print(tokens.count)
}
if #available(anyAppleOS 27.0, *) { demo() }
```

Output is `1, 2, 1, 2, 2`, one value per line. UniqueArray and Span use Failure == Never, so these calls need no try. Generic iteration uses `for try`; removing try is rejected because an arbitrary I may fail. The published example checks a generic throwing signature and nonthrowing concrete calls; it does not implement a custom iterator that throws at runtime.

The iterator supplies borrowed spans through `nextSpan(maxCount:)`; the loop exposes elements while their borrow is valid. Do not consume a borrowed noncopyable element or resize its owner while that access is live. Iteration failure is separate from async suspension. Generic code cannot assume that an iterator remains usable after an error. [SE-0516](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md)

In this SDK, Array itself does not satisfy Iterable. Existing Sequence algorithms still accept it. To use this visitor with an array, bind a local view, for example `let view = values.span`, and pass that view on the available OS path. InlineArray and UniqueArray support borrowing iteration but do not thereby become Sequence or Collection.

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
