---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - unsafe-memory
---

# C6 — Unsafe Pointers, Buffers, and Raw Memory

**Learning goals:**

- Separate ownership, allocation, binding, initialization, alignment, bounds, and access lifetime.
- Replace escaping borrowed pointers with scoped processing or owned results and audit manual cleanup on throw.
- Use checked temporary/raw spans when they fit, while keeping native retention and wire-format obligations explicit.

**Prerequisites:** [[C1 — ARC fundamentals]]/[[A6 — `defer`, Early Exits, and Cleanup Semantics]] for lifetime and cleanup, [[C3 — Exclusivity enforcement and `inout`]] for scoped exclusive access, and [[C4 — String and Unicode correctness]] for encoded bytes. [[C11 — Strict Memory Safety Mode and Safe Systems Programming]] adds strict memory-safety diagnostics; [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]] develops safe span parsing.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An address does not tell you who owns the storage or whether a String value has been initialized there. A pointer’s safe use depends on several independent facts: allocated capacity, type binding, initialized elements, alignment, bounds, and lifetime. UnsafeBufferPointer adds a count but does not retain the memory owner.

A **raw pointer** carries no specific element type in its static view; the memory can still already be bound to a type. A **typed pointer** claims a Pointee type, but that claim must match the actual storage state. Allocate/deallocate manage storage; initialize/deinitialize manage the values living in it. [Unsafe Swift](https://developer.apple.com/videos/play/wwdc2020/10648/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Initialize values before reading and destroy them before freeing storage

Standalone manual typed allocation. Both elements are initialized before the buffer copy, and defer reverses the registered cleanup order.

```swift
func labels() -> [String] {
    let pointer = UnsafeMutablePointer<String>.allocate(capacity: 2)
    defer { pointer.deallocate() }
    pointer.initialize(to: "A")
    pointer.advanced(by: 1).initialize(to: "B")
    defer { pointer.deinitialize(count: 2) }
    return Array(UnsafeBufferPointer(start: pointer, count: 2))
}
precondition(labels() == ["A", "B"])
```

**Walkthrough:** Allocation alone creates room, not initialized Strings. The later defer first destroys the two initialized values, then the earlier defer frees their storage. Returning an Array retains an owned sequence, not this pointer. If initialization could throw partway through, cleanup must track the actual initialized prefix.

### Binding establishes a type; assuming binding only asserts it

Standalone raw allocation aligned for Int. All accesses remain inside the function’s owned lifetime.

```swift
func boundValue() -> Int {
    let raw = UnsafeMutableRawPointer.allocate(
        byteCount: MemoryLayout<Int>.stride, alignment: MemoryLayout<Int>.alignment
    )
    defer { raw.deallocate() }
    let typed = raw.bindMemory(to: Int.self, capacity: 1)
    typed.initialize(to: 42)
    defer { typed.deinitialize(count: 1) }
    let same = raw.assumingMemoryBound(to: Int.self)
    return same.pointee
}
precondition(boundValue() == 42)
```

**Walkthrough:** bindMemory establishes the intended type for this fresh storage. The later assumption is justified because that binding already exists. Neither operation converts an Int number into another numeric type. Alignment, initialized count, and lifetime are still separate facts.

## Rules and pitfalls

### A scoped access closure can still leak an unsafe pointer

**Common pitfall:** Saving baseAddress and dereferencing it after the owner’s access closure returns. Compile-only; the invalid access is never executed.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
func invalidChecksum() -> UInt8 {
    let bytes: [UInt8] = [1, 2, 3]
    let saved = bytes.withUnsafeBufferPointer { $0.baseAddress! }
    return saved[0]
}
```

**Consequence:** The saved address has no supported lifetime for the later read, even if a particular run keeps the buffer in place.

**Why:** The closure grants temporary access; returning an UnsafePointer does not retain or extend that access. A nonescaping closure parameter does not constrain its result to safe values.

**Correction:** Finish the read inside the closure and return an owned scalar. This checksum deliberately sums modulo 2^64.

```swift
func checksum(_ bytes: [UInt8]) -> UInt64 {
    bytes.withUnsafeBufferPointer { buffer in
        buffer.reduce(UInt64(0)) { $0 &+ UInt64($1) }
    }
}
precondition(checksum([1, 2, 3]) == 6)
precondition(checksum([]) == 0)
```

**What changes:** Every buffer read occurs while Array supplies initialized valid storage. Empty input uses the identity without dereferencing a nil base address. A retaining asynchronous native API needs a different owner-through-completion contract.

### Capacity is not the number of initialized values

**Common pitfall:** Deinitializing two Strings after initializing only one. Compile-only; destroying uninitialized memory is never run.

**Mistake:** Compile-only; the invalid cleanup is never called.

```swift
func invalidCleanup() {
    let pointer = UnsafeMutablePointer<String>.allocate(capacity: 2)
    pointer.initialize(to: "A")
    pointer.deinitialize(count: 2)
    pointer.deallocate()
}
```

**Consequence:** The second destruction reads storage that never contained a valid initialized String.

**Why:** Allocation reserves capacity; initialization establishes which positions contain values. Destruction must cover exactly the initialized elements, including after partial failure.

**Correction:** Track the initialized prefix and register cleanup before any throwing step. This standalone probe observes one or two destroyed objects without exposing their addresses.

```swift
final class Log { var destroyed = 0 }
final class Item {
    let id: Int
    let log: Log
    init(_ id: Int, log: Log) {
        self.id = id
        self.log = log
    }
    deinit { log.destroyed += 1 }
}
enum BuildError: Error { case stopped }
func build(log: Log, stopAfterFirst: Bool) throws -> [Int] {
    let pointer = UnsafeMutablePointer<Item>.allocate(capacity: 2)
    var initialized = 0
    defer {
        pointer.deinitialize(count: initialized)
        pointer.deallocate()
    }
    pointer.advanced(by: initialized).initialize(to: Item(1, log: log))
    initialized += 1
    if stopAfterFirst { throw BuildError.stopped }
    pointer.advanced(by: initialized).initialize(to: Item(2, log: log))
    initialized += 1
    return [pointer[0].id, pointer[1].id]
}
let success = Log()
let result = try build(log: success, stopAfterFirst: false)
precondition(result == [1, 2] && success.destroyed == 2)
let failure = Log()
do {
    _ = try build(log: failure, stopAfterFirst: true)
    preconditionFailure()
} catch BuildError.stopped {}
precondition(failure.destroyed == 1)
```

**What changes:** Cleanup destroys only values whose initialization completed, then frees storage on return and throw. withTemporaryAllocation can automate this bookkeeping when its scoped output-span contract fits.

### A matching byte size does not justify an incompatible typed view

**Common pitfall:** Using assumingMemoryBound to read an initialized Int as an unrelated Double. Compile-only; this invalid binding access is never run.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
func invalidConversion() -> Double {
    let pointer = UnsafeMutablePointer<Int>.allocate(capacity: 1)
    pointer.initialize(to: 42)
    defer {
        pointer.deinitialize(count: 1)
        pointer.deallocate()
    }
    let other = UnsafeRawPointer(pointer).assumingMemoryBound(to: Double.self)
    return other.pointee
}
```

**Consequence:** The view asserts a binding/interpretation not established by the actual Int storage. Similar size is insufficient.

**Why:** assumingMemoryBound performs no numeric conversion and does not create permission for arbitrary incompatible aliasing.

**Correction:** If the intent is numeric conversion, read the actual Int then convert its value.

```swift
func convertedValue() -> Double {
    let pointer = UnsafeMutablePointer<Int>.allocate(capacity: 1)
    pointer.initialize(to: 42)
    defer {
        pointer.deinitialize(count: 1)
        pointer.deallocate()
    }
    return Double(pointer.pointee)
}
precondition(convertedValue() == 42.0)
```

**What changes:** The pointer access matches the initialized type, and Double’s value initializer performs the conversion. Bit-level formats require their own documented representation rules; temporary rebinding is not a universal conversion tool.

## Worked example

### Checked temporary bytes distinguish capacity from initialized length

**Setup:** Standalone Apple OS 27 example, also compiled with strict memory safety and warnings as errors. The explicit ByteOrder overloads need that runtime.

**Predict:** Does six-byte capacity make six bytes readable after appending one byte and a four-byte field?

```swift
let field: UInt32 = withTemporaryAllocation(byteCount: 6, alignment: 1) { output in
    output.append(0xFF, as: UInt8.self)
    output.append(258, as: UInt32.self, .bigEndian)
    let bytes = output.bytes
    precondition(bytes.byteCount == 5)
    return bytes.load(fromByteOffset: 1, as: UInt32.self, .bigEndian)
}
precondition(field == 258)
```

**Outcome:** Only five bytes are initialized, and the field beginning at unaligned offset one decodes as 258.

**Why:** OutputRawSpan tracks initialized bytes and the helper releases temporary storage. The safe load supports this primitive representation and explicit byte order. It does not make unvalidated arbitrary offsets or Swift struct layouts into a wire format. [Temporary allocation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md), [safe raw-byte operations](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0525-rawspan-safe-loading-api.md)

## Interview questions

### What is the difference between raw and typed memory?

**Answer:** A raw pointer has no Pointee type in its view, while typed access asserts a particular type and compatible storage state. Actual binding and initialization are properties of the memory, not something an arbitrary cast repairs.

**Example:** Standalone runnable example.

```swift
func example() -> UInt32 {
    let raw = UnsafeMutableRawPointer.allocate(
        byteCount: MemoryLayout<UInt32>.stride, alignment: MemoryLayout<UInt32>.alignment
    )
    defer { raw.deallocate() }
    let value = raw.bindMemory(to: UInt32.self, capacity: 1)
    value.initialize(to: 258)
    defer { value.deinitialize(count: 1) }
    return raw.assumingMemoryBound(to: UInt32.self).pointee
}
precondition(example() == 258)
```

**Walkthrough:** The type is established before initialization and read. Values are deinitialized before allocation release. A raw-pointer view does not undo the binding, so the later typed assumption uses the same UInt32.

**Boundary:** Native representation here is not a portable byte encoding. Alignment and bounds still apply; parsing an external unaligned field should use the documented checked byte operation with explicit byte order.

### When is withUnsafeBytes safer than storing a pointer for later?

**Answer:** When the complete operation finishes synchronously within the owner’s access scope and the callee retains no address. Return a scalar or owned data; a retaining native call cannot use this lifetime contract.

**Example:** Standalone runnable example.

```swift
import Foundation

func checksum(_ data: Data) -> UInt64 {
    data.withUnsafeBytes { (buffer: UnsafeRawBufferPointer) in
        buffer.reduce(UInt64(0)) { $0 &+ UInt64($1) }
    }
}
func selectedBytes(_ data: Data) -> [UInt8] {
    data.withUnsafeBytes { (buffer: UnsafeRawBufferPointer) in Array(buffer) }
}
let data = Data([1, 2, 3])
precondition(checksum(data) == 6 && checksum(Data()) == 0)
var owned = selectedBytes(data)
owned[0] = 9
precondition(Array(data) == [1, 2, 3] && owned == [9, 2, 3])
```

**Walkthrough:** Data supplies the initialized bytes during each closure. The reduction returns a scalar, and Array(buffer) returns owned values that remain valid afterward. Empty data never requires a forced base address.

**Boundary:** The modulo sum is a teaching checksum, not a cryptographic hash. Native APIs must separately document pointer retention, result ownership, callbacks, and the event that allows release.

### Extension: Why can a closure wrapper leak a pointer, and what does a span improve?

**Answer:** UnsafePointer is an escapable address value, so a closure can return or save it. Span instead expresses a borrowed nonescapable view with compiler-enforced lifetime dependencies and checked element access. It still does not own storage.

**Example:** Standalone runnable example.

```swift
func sum(_ bytes: Span<UInt8>) -> UInt64 {
    var result: UInt64 = 0
    for index in bytes.indices { result &+= UInt64(bytes[index]) }
    return result
}
let bytes: [UInt8] = [1, 2, 3]
let view = bytes.span
precondition(sum(view) == 6)
let total = withTemporaryAllocation(of: UInt8.self, capacity: 2) { output in
    output.append(4)
    output.append(5)
    return sum(output.span)
}
precondition(total == 9)
```

**Walkthrough:** Both calls return an owned scalar while the borrowed view stays within its owner’s lifetime. The output span tracks initialized elements and handles cleanup. No raw address is manufactured or retained.

**Boundary:** A span cannot outlive its owner, be resized like an owned Array, or establish domain validity of packet bytes. A safe type also has per-operation deployment constraints; [[C12 — `Span`, `InlineArray`, and modern low-level abstractions|C12]] checks those explicitly.

Standalone **intentionally rejected** attempt to return the temporary span. The result’s Escapable requirement does not permit this borrowed view:

```swift
let escaped = withTemporaryAllocation(of: Int.self, capacity: 1) { output in
    output.append(1)
    return output.span
}
```

### Extension: When is temporary rebinding appropriate?

**Answer:** Use it only for the documented compatible layout/native boundary, keeping the rebound pointer inside its closure and avoiding conflicting access through the original type. Equal size alone is not the rule.

**Example:** Standalone runnable example.

```swift
import Darwin

var address = sockaddr_in()
address.sin_family = sa_family_t(AF_INET)
let family = withUnsafePointer(to: &address) { pointer in
    pointer.withMemoryRebound(to: sockaddr.self, capacity: 1) { rebound in
        Int(rebound.pointee.sa_family)
    }
}
precondition(family == AF_INET)
```

**Walkthrough:** The Darwin socket structures provide the native compatible prefix used at this interop boundary. The example reads only that initialized family field, performs no socket operation, and lets the memory return to its original binding after the closure. [Raw-pointer migration and binding rules](https://swift.org/migration-guide-swift3/se-0107-migrate.html)

**Boundary:** Do not copy this into arbitrary Swift structs with similar sizes. Assuming an existing binding, establishing one, and temporarily rebinding are distinct operations; audit the exact native layout and retention contract.

## Focused exercise

**Task:** Replace an escaped buffer pointer with a scoped result or an owned byte sequence.

**Acceptance checks:**

- All pointer reads finish within the owner’s access closure.
- Empty input returns the sum identity without dereferencing nil.
- The returned byte copy can be edited independently after the closure ends.

**Solution:** Standalone scoped processing and owned-result alternatives:

```swift
func checksum(_ bytes: [UInt8]) -> UInt64 {
    bytes.withUnsafeBufferPointer { buffer in
        buffer.reduce(UInt64(0)) { $0 &+ UInt64($1) }
    }
}
func ownedCopy(_ bytes: [UInt8]) -> [UInt8] {
    bytes.withUnsafeBufferPointer { Array($0) }
}
let bytes: [UInt8] = [1, 2, 3]
precondition(checksum(bytes) == 6 && checksum([]) == 0)
precondition(ownedCopy([]).isEmpty)
var copy = ownedCopy(bytes)
copy[0] = 9
precondition(bytes == [1, 2, 3] && copy == [9, 2, 3])
```

**Reasoning:** Array supplies initialized elements and bounds. The return values have owned contracts, while the pointer never escapes. The checksum uses wrapping UInt64 addition explicitly. Manual native allocation would additionally need state tracking, alignment/binding, and completion-driven cleanup.

## In practice

**Choose it when:** Keep unsafe/native interoperability narrow and prefer scoped checked views when their contracts fit.

**Investigate by:** Audit the owner and initialized prefix, every binding/alignment assumption, and the exact event ending native retention before reviewing individual pointer syntax.

**Strong evidence:** Verify successful and throwing cleanup with valid inputs, compile invalid lifetime examples without executing them, and use targeted native/sanitizer integration evidence for real boundaries.

## References

### Related guides

- [[C1 — ARC fundamentals]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[C11 — Strict Memory Safety Mode and Safe Systems Programming]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[E3 — C Interoperability and Unsafe Boundaries]]

### Official sources

- [Unsafe Swift](https://developer.apple.com/videos/play/wwdc2020/10648/)
- [Swift.org, "UnsafeRawPointer Migration"](https://swift.org/migration-guide-swift3/se-0107-migrate.html)
- [SE-0524](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md)
- [SE-0525](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0525-rawspan-safe-loading-api.md)
- [Apple Developer. "UnsafeRawPointer." Apple Developer Documentation](https://developer.apple.com/documentation/swift/unsaferawpointer)
- [Apple Developer. "UnsafeBufferPointer." Apple Developer Documentation](https://developer.apple.com/documentation/swift/unsafebufferpointer)
- [Apple Developer. "withUnsafeBufferPointer(_:)." Apple Developer Documentation](https://developer.apple.com/documentation/swift/array/withunsafebufferpointer%28_%3a%29)
- [Apple Developer. "UnsafeRawBufferPointer." Apple Developer Documentation](https://developer.apple.com/documentation/swift/unsaferawbufferpointer)
- [Swift.org. "Strict Memory Safety." Swift Compiler Diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/strict-memory-safety/)
