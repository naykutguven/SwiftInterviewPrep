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

**Learning goals:**

- Choose owned fixed/dynamic storage separately from borrowed Span/Ref access and exclusive owners.
- Keep initialized count, capacity, bounds, lifetime, and wire representation distinct.
- Validate a fixed-width field with overflow-safe bounds and explicit endian order, checking each selected operation’s deployment boundary.

**Prerequisites:** [[C3 — Exclusivity enforcement and `inout`]]/[[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]] for access lifetimes, [[C6 — Unsafe pointers, buffers, and raw memory]] for initialized memory and raw bytes, and [[C8 — Copyable, noncopyable types, and `~Copyable`]] for unique owners. [[B3 — Generics, Constraints, and Same-Type Relationships]] supplies generic default suppression.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A parser can own a packet or borrow its bytes while reading one field. Ownership and the view used for access are different choices. Array owns a copyable dynamic sequence; InlineArray owns a fixed-count inline value; UniqueArray owns exclusive dynamic storage. Span borrows initialized contiguous elements without owning their lifetime. Ref provides dependent access to one value.

“Inline” describes representation inside an owner, not guaranteed stack placement. “Safe load” describes supported checked memory access, not a promise that invalid bounds return nil or that a Swift struct is a wire format. Establish the owner, initialized range, field format, and per-operation availability before loading. [Span](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Process a borrowed view and return an owned scalar

Shared sum setup. Span indices describe its initialized contiguous elements; the array remains its owner.

```swift
func sum(_ values: Span<Int>) -> Int {
    var result = 0
    for index in values.indices { result += values[index] }
    return result
}
@available(anyAppleOS 26.0, *)
func spanProbe() {
    let values = [10, 20, 30]
    let view = values.span
    precondition(sum(view) == 60)
    precondition(values == [10, 20, 30])
    let empty: [Int] = []
    precondition(sum(empty.span) == 0)
}
if #available(anyAppleOS 26.0, *) { spanProbe() }
```

**Walkthrough:** The borrowed view stays inside the array’s lifetime and the result is an owned Int. Array.span needs OS 26 here, while the basic sum operation and temporary-output-span path have separate older-target controls. The small sum does not overflow, and index-based traversal does not require Iterable.

### Fixed count is part of the InlineArray type

Standalone OS 26+ indexing example. Header can be a value inside another owner, including a class instance.

```swift
struct Header { var bytes: [4 of UInt8] }
final class Frame { var header = Header(bytes: [1, 2, 3, 4]) }
func checksum(_ header: [4 of UInt8]) -> UInt16 {
    var result: UInt16 = 0
    for index in header.indices { result += UInt16(header[index]) }
    return result
}
let frame = Frame()
precondition(checksum(frame.header.bytes) == 10)
precondition(frame.header.bytes.count == 4)
precondition(MemoryLayout<[4 of UInt8]>.size == 4)
```

**Walkthrough:** [4 of UInt8] is InlineArray<4, UInt8>; the count belongs to the type and literal arity is checked. Inline storage remains inside its containing value. The observed size is value layout, not evidence about Frame’s allocation location or total memory. [Integer generic parameters](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0452-integer-generic-parameters.md), [InlineArray](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md)

### Capacity and initialized count are different numbers

Prepend sum setup. The helper creates temporary capacity, and each append initializes one element.

```swift
let total = withTemporaryAllocation(of: Int.self, capacity: 3) { output in
    precondition(output.count == 0 && output.capacity == 3)
    output.append(10)
    output.append(20)
    precondition(output.count == 2 && output.capacity == 3)
    return sum(output.span)
}
precondition(total == 30)
let empty = withTemporaryAllocation(of: Int.self, capacity: 0) { output in
    precondition(output.isEmpty)
    return sum(output.span)
}
precondition(empty == 0)
```

**Walkthrough:** Only the two appended values are readable through the initialized span. The unused third position is capacity, not an extra zero. The helper destroys initialized values and releases storage on normal/throwing exit; it does not promise stack allocation for every capacity. [Temporary output spans](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md)

## Rules and pitfalls

### A Swift struct layout is not an external header format

**Common pitfall:** Loading a Swift struct directly from five protocol bytes and assuming padding, alignment, and byte order match. Compile-only; this invalid access is never run.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
struct Header: BitwiseCopyable {
    var version: UInt8
    var length: UInt32
}
func invalidHeader(_ bytes: [UInt8]) -> Header {
    bytes.withUnsafeBufferPointer { buffer in
        UnsafeRawPointer(buffer.baseAddress!).load(as: Header.self)
    }
}
```

**Consequence:** The load can have insufficient/alignment-incompatible storage, and even a permitted native representation would not specify the wire format or endian order.

**Why:** MemoryLayout and BitwiseCopyable do not define portable field offsets or valid wire interpretation. A typed load does not validate packet semantics.

**Correction:** Parse declared fields separately and construct the model value. This standalone format uses version byte 1 followed by a four-byte big-endian length.

```swift
struct Header {
    let version: UInt8
    let length: UInt32
}
func parseHeader(_ bytes: Span<UInt8>) -> Header? {
    guard bytes.count >= 5, bytes[0] == 1 else { return nil }
    let length = bytes.bytes.load(fromByteOffset: 1, as: UInt32.self, .bigEndian)
    return Header(version: bytes[0], length: length)
}
func headerProbe() {
    let valid: [UInt8] = [1, 0, 0, 1, 2]
    precondition(parseHeader(valid.span)?.length == 258)
    let short: [UInt8] = [1, 0]
    precondition(parseHeader(short.span) == nil)
    let unknown: [UInt8] = [2, 0, 0, 1, 2]
    precondition(parseHeader(unknown.span) == nil)
}
headerProbe()
```

**What changes:** The field begins at deliberately unaligned offset one, but the documented safe load supports it. Length and version are checked before access, and byte order is explicit. A complete protocol parser must also validate semantic length limits and the remaining message.

### A blanket availability annotation can hide supported older operations

**Common pitfall:** Requiring OS 27 for a helper that only uses basic Span indices/subscripts, then calling it from an older-target context. This mistake is compiled for macOS 26.

**Mistake:** Compile-only; intentionally rejected.

```swift
@available(anyAppleOS 27.0, *)
func unnecessarilyRestricted(_ values: Span<Int>) -> Int {
    var result = 0
    for index in values.indices { result += values[index] }
    return result
}
func olderCaller(_ values: [Int]) -> Int { unnecessarilyRestricted(values.span) }
```

**Consequence:** The caller rejects because of the helper’s own unnecessary restriction.

**Why:** Availability belongs to selected declarations/operations. It cannot be inferred from a chapter about other new APIs or a blanket label on Span.

**Correction:** Use the supported basic operation contract and validate the actual deployment target. This source is compiled for macOS 13 and run on the OS 27 host.

```swift
func basicSum(_ values: Span<Int>) -> Int {
    var result = 0
    for index in values.indices { result += values[index] }
    return result
}
let total = withTemporaryAllocation(of: Int.self, capacity: 3) { output in
    output.append(1)
    output.append(2)
    output.append(3)
    return basicSum(output.span)
}
precondition(total == 6)
```

**What changes:** The basic helper and temporary-output-span construction compile for the older selected target. Array.span separately needs OS 26, and explicit ByteOrder raw loads below need OS 27. Older-target compilation is distinct from execution on an older runtime.

The opposite overgeneralization also fails. This unguarded ByteOrder load is **intentionally rejected** for macOS 26:

```swift
func tooNew(_ bytes: RawSpan) -> UInt32 {
    bytes.load(fromByteOffset: 0, as: UInt32.self, .bigEndian)
}
```

## Worked example

### Guard a newer load while keeping an explicit older operation

**Setup:** Standalone borrowed-byte parser compiled for macOS 26, executed on OS 27. The fallback is called directly too, so its arithmetic is tested without claiming an older-runtime run.

**Predict:** Does checking module/type presence alone authorize the new endian overload?

```swift
func fallback(_ bytes: Span<UInt8>, at offset: Int) -> UInt32? {
    guard offset >= 0, offset <= bytes.count, bytes.count - offset >= 4 else { return nil }
    return (UInt32(bytes[offset]) << 24) | (UInt32(bytes[offset + 1]) << 16)
        | (UInt32(bytes[offset + 2]) << 8) | UInt32(bytes[offset + 3])
}
func portableRead(_ bytes: Span<UInt8>, at offset: Int) -> UInt32? {
    guard offset >= 0, offset <= bytes.count, bytes.count - offset >= 4 else { return nil }
    if #available(anyAppleOS 27.0, *) {
        return bytes.bytes.load(fromByteOffset: offset, as: UInt32.self, .bigEndian)
    }
    return fallback(bytes, at: offset)
}
func portableProbe() {
    let bytes: [UInt8] = [0xFF, 0, 0, 1, 2]
    precondition(portableRead(bytes.span, at: 1) == 258)
    precondition(fallback(bytes.span, at: 1) == 258)
    precondition(portableRead(bytes.span, at: Int.max) == nil)
    precondition(fallback(bytes.span, at: -1) == nil)
}
portableProbe()
```

**Outcome:** No: the newer operation needs a runtime availability context. Both tested paths return 258, and invalid offsets return nil before indexing.

**Why:** The guards establish a safe subtraction and four remaining bytes. Only then are offset + 1/2/3 computed. A newer load handles the same explicit byte order behind its guard; type/module availability alone would not establish that permission.

## Interview questions

### How do Array, InlineArray, UniqueArray, Span, and Ref differ in ownership and lifetime?

**Answer:** Array gives copyable dynamic value storage; InlineArray gives fixed-count inline owned storage; UniqueArray gives exclusive dynamic storage. Span and Ref borrow access rather than owning the referent’s lifetime. Their copy/resize/retain permissions differ.

**Example:** Standalone runnable example.

```swift
func ownershipProbe() {
    var array = [1, 2]
    let snapshot = array
    array[0] = 9
    precondition(snapshot == [1, 2] && array == [9, 2])
    var inline: [2 of Int] = [1, 2]
    let inlineSnapshot = inline
    inline[0] = 9
    precondition(inlineSnapshot[0] == 1 && inline[0] == 9)
    var unique = UniqueArray<Int>()
    unique.append(7)
    let removed = unique.removeLast()
    precondition(removed == 7 && unique.count == 0)
    let view = array.span
    let value = 42
    let reference = Ref(value)
    precondition(view[0] == 9 && reference.value == 42)
}
ownershipProbe()
```

**Walkthrough:** The first two copyable-element containers produce independent logical snapshots. UniqueArray transfers its element out rather than silently copying the owner. The span/reference stay within their owners’ function-local lifetimes.

**Boundary:** This combined comparison runs on OS 27. Inline storage can be inside another allocated owner and need not live on a stack. Ref of an ordinary getter can borrow a temporary instead of original storage; [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`|C7]] checks that distinct case. No allocation winner follows from these values.

### What must a parser validate before a safe raw load?

**Answer:** Validate nonnegative offset, enough remaining bytes without overflowing arithmetic, the operation’s supported representation/width and byte order, and the format’s semantic fields. Bounds-checked loads may trap, so a parser returning failure must guard first.

**Example:** Standalone runnable example.

```swift
@available(anyAppleOS 27.0, *)
func readUInt32BE(_ bytes: RawSpan, at offset: Int) -> UInt32? {
    guard offset >= 0, offset <= bytes.byteCount,
        bytes.byteCount - offset >= 4
    else { return nil }
    return bytes.load(fromByteOffset: offset, as: UInt32.self, .bigEndian)
}
func readProbe() {
    let bytes: [UInt8] = [0xFF, 0, 0, 1, 2]
    precondition(readUInt32BE(bytes.span.bytes, at: 1) == 258)
    precondition(readUInt32BE(bytes.span.bytes, at: 2) == nil)
    precondition(readUInt32BE(bytes.span.bytes, at: Int.max) == nil)
}
readProbe()
```

**Walkthrough:** Offset one is the final valid four-byte window. The operation supports its unaligned location and interprets it as big-endian. Offset two/Int.max reject before a load or overflowing offset + width.

**Boundary:** UInt32’s supported memory conversion says nothing about whether 258 is a legal protocol length. Swift struct layout and BitwiseCopyable alone also do not supply this raw conversion contract. [Safe raw operations](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0525-rawspan-safe-loading-api.md)

This independent BitwiseCopyable struct still lacks the safe load’s required conversion conformance; it is **intentionally rejected**:

```swift
struct Pair: BitwiseCopyable {
    let first: UInt16
    let second: UInt16
}
func unsupported(_ bytes: RawSpan) -> Pair { bytes.load(fromByteOffset: 0, as: Pair.self) }
```

### Extension: Why is borrowing iteration different from Sequence/Collection and from OS 26 indexing?

**Answer:** Iterable supplies borrowed elements and potentially typed iteration failure. A type can support that for-in surface without Sequence/Collection conformance. The checked new iteration APIs require OS 27; existing index/subscript operations have a separate boundary.

**Example:** Standalone runnable example.

```swift
struct Token: ~Copyable { let id: Int }
func iterationProbe() {
    let tokens: [2 of Token] = [Token(id: 1), Token(id: 2)]
    var ids: [Int] = []
    for token in tokens { ids.append(token.id) }
    let view = tokens.span
    for token in view { ids.append(token.id) }
    precondition(ids == [1, 2, 1, 2] && tokens.count == 2)
}
iterationProbe()
```

**Walkthrough:** Both loops borrow, so the owner still has two Tokens afterward. Binding view uses the current checked lifetime form; the historical direct loop over tokens.span produced a diagnostic. [[C5 — Collection Protocols, Complexity, and Index Invalidation|C5]] supplies the generic visitor and preserves I.Failure. [Iterable](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md)

**Boundary:** This does not turn the inline owner into Sequence/Collection or add their arbitrary algorithms. Do not consume borrowed noncopyable elements or resize a live view’s owner. On the supported OS 26 path, use indices/subscripts instead.

### Extension: What cleanup does withTemporaryAllocation provide on a throwing exit?

**Answer:** It destroys the initialized typed prefix and releases temporary storage on normal return or throw. Capacity is not initialized count, and no borrowed view may escape as an owned result.

**Example:** Standalone runnable example.

```swift
final class Log { var destroyed: [Int] = [] }
final class Item {
    let id: Int
    let log: Log
    init(_ id: Int, log: Log) {
        self.id = id
        self.log = log
    }
    deinit { log.destroyed.append(id) }
}
enum BuildError: Error { case stopped }
func temporary(_ log: Log, fail: Bool) throws -> Int {
    try withTemporaryAllocation(of: Item.self, capacity: 2) { output in
        output.append(Item(1, log: log))
        if fail { throw BuildError.stopped }
        output.append(Item(2, log: log))
        return output.count
    }
}
let success = Log()
let count = try temporary(success, fail: false)
precondition(count == 2 && success.destroyed.sorted() == [1, 2])
let failure = Log()
do {
    _ = try temporary(failure, fail: true)
    preconditionFailure()
} catch BuildError.stopped {}
precondition(failure.destroyed == [1])
```

**Walkthrough:** Success destroys the two initialized Item objects; failure after the first append destroys only item 1. No destruction order is imposed on the two-element case. The unused capacity is not treated as an initialized object.

**Boundary:** This is synchronous cleanup with no thread affinity. Negative capacity and append beyond capacity violate preconditions and are not executed. Temporary allocation is not a promise of stack placement at arbitrary sizes.

Standalone **intentionally rejected** span escape. The helper’s owned result requirement cannot be satisfied by its borrowed temporary view:

```swift
let escaped = withTemporaryAllocation(of: Int.self, capacity: 1) { output in
    output.append(1)
    return output.span
}
```

## Focused exercise

**Task:** Read one fixed-width big-endian field from borrowed bytes after overflow-safe validation.

**Acceptance checks:**

- The last valid offset one reads an unaligned four-byte field as 258.
- Negative/Int.min, oversized/Int.max, and empty inputs return nil.
- The view remains local to its owner and the result is an owned UInt32 value.

**Solution:** Prepend readUInt32BE/readProbe from the parser answer. This runs on the OS 27 baseline required by the ByteOrder overload:

```swift
func parserExercise() {
    let bytes: [UInt8] = [0xFF, 0, 0, 1, 2]
    let view = bytes.span
    precondition(readUInt32BE(view.bytes, at: 1) == 258)
    for offset in [Int.min, -1, 2, 5, Int.max] {
        precondition(readUInt32BE(view.bytes, at: offset) == nil)
    }
    let empty: [UInt8] = []
    precondition(readUInt32BE(empty.span.bytes, at: 0) == nil)
}
parserExercise()
```

**Reasoning:** The guard proves offset is in range before subtraction, then proves a complete field remains. The explicit endian operation supplies representation; protocol-level meaning remains a separate validation step. No out-of-bounds load is executed.

## In practice

**Choose it when:** Choose owned storage for retained data and borrowed views for scoped access, with fixed versus dynamic count determined by the actual model.

**Investigate by:** Check initialized count, capacity, owner lifetime, exclusive access, byte representation, and each selected declaration’s availability.

**Strong evidence:** Verify last-valid and invalid parser boundaries and cleanup on throw, then measure allocations separately from inline/heap or safe/unsafe source labels.

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
