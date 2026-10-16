---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - span
  - inlinearray
  - low-level-swift
---

# C12 — `Span`, `InlineArray`, and Modern Low-Level Abstractions

**Goal:** Read a fixed-width big-endian field from borrowed bytes after overflow-safe bounds validation.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Owned storage and borrowed views solve different problems. Array owns a copyable dynamic sequence; InlineArray owns fixed-count inline elements; UniqueArray owns noncopyable-capable dynamic storage. Span borrows initialized contiguous elements without owning their lifetime. Ref borrows one value. Inline describes representation inside its owner, not guaranteed stack placement. [Span](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md)

## Rules and pitfalls

### Validate the format before the load

A checked raw load enforces memory bounds and supported bit-pattern conversion. It does not establish packet length, protocol version, byte order, field meaning, or valid combinations. Avoid offset + width on untrusted offsets until overflow has been ruled out.

**Common pitfall:** Loading a Swift struct as a wire header assumes an undocumented layout and endianness. Parse explicit fixed-width fields and validate domain constraints separately.

### Select operations, not just type names

Span, InlineArray, borrowing iteration, temporary/output spans, raw loads, and byte-order overloads have different availability. Iterable is separate from Sequence/Collection. Temporary allocation's capacity is not its initialized count.

**Common pitfall:** Assuming every Span API requires OS 27 needlessly removes supported alternatives; assuming every newer overload back-deploys makes older runtimes invalid. Check each selected declaration and preserve an owned-storage fallback when required.

## Interview questions

### How do Array, InlineArray, UniqueArray, Span, and Ref differ in ownership and lifetime?

Array is a copyable CoW value; InlineArray is fixed-size inline owned storage; UniqueArray has exclusive dynamic ownership. Span is a nonowning nonescapable view dependent on storage lifetime. Ref represents dependent access to a value and can borrow a getter temporary. Each choice changes who may retain, copy, resize, or mutate the data.

### What must a parser validate before a safe raw load?

Validate nonnegative offset, sufficient remaining bytes without overflowing arithmetic, protocol byte order, supported field width/type, and any domain constraints. A memory-safe checked load can still trap on invalid bounds, so a parser returning failure must validate first. An unaligned wire field should use the documented unaligned-safe operation.

## Focused exercise

**Task:** Read one fixed-width field from borrowed bytes with explicit byte order. Check the last valid offset, negative input, and an oversized offset.

**Acceptance checks:**

- Offset one is the last valid four-byte field and deliberately unaligned.
- Negative, too-large, overflowing-size, and empty cases return nil.
- The borrowed view never escapes its owner or returns as an owned result.

**Solution:**

Standalone Apple OS 27 example; the explicit byte-order RawSpan load needs that runtime. Older basic Span operations have separate availability.

```swift
@available(anyAppleOS 27.0, *)
func readUInt32BE(_ bytes: RawSpan, at offset: Int) -> UInt32? {
    guard offset >= 0, offset <= bytes.byteCount,
          bytes.byteCount - offset >= 4 else { return nil }
    return bytes.load(fromByteOffset: offset, as: UInt32.self, .bigEndian)
}
@available(anyAppleOS 27.0, *)
func probe() {
    let bytes: [UInt8] = [0xFF, 0, 0, 1, 2]
    let span = bytes.span
    precondition(readUInt32BE(span.bytes, at: 1) == 258)
    precondition(readUInt32BE(span.bytes, at: 2) == nil)
    precondition(readUInt32BE(span.bytes, at: -1) == nil)
    precondition(readUInt32BE(span.bytes, at: Int.max) == nil)
    let empty: [UInt8] = []
    precondition(readUInt32BE(empty.span.bytes, at: 0) == nil)
}
if #available(anyAppleOS 27.0, *) { probe() }
```

**Reasoning:** The guards establish the safe subtraction before checking the remaining field width. The API's big-endian conversion supplies the wire interpretation explicitly. The result proves only this field's representation; a complete parser must still validate the rest of its format.

## In practice

**Choose it when:** Use borrowed views for scoped processing and owned containers for retained data.

**Investigate by:** Check owner lifetime, initialized count, bounds, representation, and per-operation availability.

**Strong evidence:** Compare ownership/deployment alternatives and measure allocation needs without assuming inline means stack or safe means nontrapping.

### Inline storage, iteration, and initialized temporary storage — optional

#### `InlineArray` has size in the type

`InlineArray` uses integer generic parameters. SE-0452 introduced integer generic parameters in Swift 6.2, allowing generic types to be parameterized by literal integer values. ([GitHub, "Integer Generic Parameters"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0452-integer-generic-parameters.md)) SE-0453 then uses that capability for `InlineArray<let count: Int, Element>`. ([GitHub, "InlineArray, a Fixed-Size Array"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md))

The type sugar introduced in Swift 6.2 remains available in 6.4:

```swift
@available(anyAppleOS 26.0, *)
func checksum(_ header: [4 of UInt8]) -> UInt16 {
    var result: UInt16 = 0

    for i in header.indices {
        result += UInt16(header[i])
    }

    return result
}

@available(anyAppleOS 26.0, *)
func demoChecksum() {
    let header: [4 of UInt8] = [1, 2, 3, 4]
    print(checksum(header))
}

if #available(anyAppleOS 26.0, *) { demoChecksum() }
```

Output:

> 10

`[4 of UInt8]` is sugar for `InlineArray<4, UInt8>`. SE-0483 introduced this type sugar and states that `[5 of Int]` is equivalent to `InlineArray<5, Int>`. ([GitHub, "InlineArray Type Sugar"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0483-inline-array-sugar.md))

#### Borrowing iteration is separate from Sequence/Collection

`InlineArray` still does not conform to `Sequence` or `Collection`. Swift 6.4 adds `Iterable` support, so a plain `for` loop can borrow its elements, including noncopyable elements. This does not add ordinary collection algorithms or transfer ownership out of the container. The checked operations require OS 27. [SE-0516](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md)

```swift
struct Token: ~Copyable { var id: Int }
@available(anyAppleOS 27.0, *)
func demo() {
    let tokens: [2 of Token] = [Token(id: 1), Token(id: 2)]
    for token in tokens { print(token.id) }
    let view = tokens.span
    for token in view { print(token.id) }
    print(tokens.count)
}
if #available(anyAppleOS 27.0, *) { demo() }
```

Output is `1, 2, 1, 2, 2`, one value per line. Both loops borrow; the two elements remain in the owner. In this compiler build, directly writing `for token in tokens.span` produced a lifetime diagnostic. Binding the span to `view` first compiles and runs; use that checked form here.

On the OS 26 path, use indices and subscripting instead:

```swift
@available(anyAppleOS 26.0, *)
func printBytes(_ bytes: [4 of UInt8]) {
    for i in bytes.indices {
        print(bytes[i])
    }
}
```

#### Temporary allocation tracks initialized elements

`withTemporaryAllocation` supplies an inout `OutputSpan` or `OutputRawSpan`. Capacity is available storage; count is the initialized prefix. The helper cleans up initialized typed elements and releases storage on normal or throwing exit. The closure is synchronous and the borrowed view cannot be returned as its owned result. It does not promise stack allocation for every capacity. [SE-0524](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md)

```swift
func sum(_ values: Span<Int>) -> Int {
    var result = 0
    for i in values.indices { result += values[i] }
    return result
}
let total = withTemporaryAllocation(of: Int.self, capacity: 3) { output in
    output.append(10)
    output.append(20)
    print(output.count, output.capacity)
    return sum(output.span)
}
print(total)
```

Output is `2 3`, then `30`. Capacity three does not initialize a third value. This exact source also compiles targeting macOS 13 and iOS 18 with the 6.4 toolchain; no OS 27 guard is needed for these operations.

Rejected example, in a separate file:

```swift
let escaped = withTemporaryAllocation(of: Int.self, capacity: 1) { output in
 output.append(1)
 return output.span
}
```

The result type must satisfy the helper's implicit Escapable requirement; Span does not. Return an owned aggregate or scalar computed inside the closure.

## References

### Related guides

- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[C11 — Strict Memory Safety Mode and Safe Systems Programming]]
- [[E3 — C Interoperability and Unsafe Boundaries]]
- [[F3 — Performance investigation habits]]

### Official sources

- [Span](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md)
- [GitHub, "Integer Generic Parameters"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0452-integer-generic-parameters.md)
- [GitHub, "InlineArray, a Fixed-Size Array"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md)
- [GitHub, "InlineArray Type Sugar"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0483-inline-array-sugar.md)
- [SE-0516](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md)
- [SE-0524](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md)
- [SE-0525: Safe raw-span loading](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0525-rawspan-safe-loading-api.md)
- [Swift.org. "Swift 6.2 Released." Swift.org Blog](https://swift.org/blog/swift-6.2-released/)
- [Apple Developer. "Span." Apple Developer Documentation](https://developer.apple.com/documentation/swift/span)
