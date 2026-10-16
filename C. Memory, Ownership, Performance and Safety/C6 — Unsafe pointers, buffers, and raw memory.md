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

**Goal:** Replace an escaped pointer with scoped processing or an owned byte copy.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A pointer is an address with a typed view, not ownership of storage. Its valid use depends on allocation, binding, initialization, alignment, bounds, and lifetime. UnsafeBufferPointer adds a count but does not keep its memory alive. A correct-looking result after a lifetime violation is not evidence of safety. [Unsafe Swift](https://developer.apple.com/videos/play/wwdc2020/10648/)

## Rules and pitfalls

### Keep borrowed addresses scoped

Array/Data unsafe access closures grant a temporary valid view under their API contract. Complete synchronous processing inside that scope or copy the required data into owned storage. A closure wrapper alone does not prevent returning or storing its pointer.

**Common pitfall:** Saving baseAddress and using it after withUnsafeBytes returns leaves a dangling or invalidated pointer. Return a computed result or owned bytes instead. An asynchronous native API that retains the buffer needs explicit ownership until completion.

### Track allocation state explicitly

Allocate/deallocate pairs manage storage. Initialize/deinitialize pairs manage values in it. Binding establishes the stored type; assumingMemoryBound does not perform a conversion or establish a missing binding. Alignment and aliasing requirements remain independent.

**Common pitfall:** Deinitializing uninitialized memory, rebinding without a valid scope, or reading an unaligned typed pointer is undefined behavior. Prefer checked spans and raw-byte operations where available; keep a small audited native boundary.

## Interview questions

### Why can a closure wrapper still leak an unsafe pointer, and what does a span improve?

A closure can return an unsafe pointer as its result or store it externally, so lexical API shape alone does not enforce lifetime. Span expresses a borrowed nonescapable view with compiler-checked dependencies, bounds, and operation constraints. It does not own storage or permit every pointer-like operation.

### What is the difference between raw memory and typed memory in Swift?

Raw memory is untyped byte storage; typed memory is bound to a particular type with valid initialization state. Correct conversion requires the documented binding, alignment, and lifetime operations. A matching byte count does not prove arbitrary typed loads valid.

### When is `withUnsafeBytes` safer than storing an unsafe pointer for later?

It is safer when all pointer use finishes synchronously while the owning value supplies valid storage. It is not safe if the closure escapes the pointer or the native function retains it beyond the call. In that case use an owned buffer and a completion-driven cleanup contract.

## Focused exercise

**Task:** Replace an escaped buffer pointer with scoped processing or owned storage. Check empty input and explain initialization and lifetime obligations.

**Acceptance checks:**

- Every pointer read occurs inside the access closure.
- Empty input returns zero without dereferencing nil.
- The returned byte copy remains valid independently of the access scope.

**Solution:**

Standalone scoped sum plus owned-copy alternative. Empty buffers need no forced base address:

```swift
func checksum(_ bytes: [UInt8]) -> UInt64 {
    bytes.withUnsafeBufferPointer { buffer in
        buffer.reduce(UInt64(0)) { $0 + UInt64($1) }
    }
}
func ownedCopy(_ bytes: [UInt8]) -> [UInt8] {
    bytes.withUnsafeBufferPointer { Array($0) }
}
let bytes: [UInt8] = [1, 2, 3]
precondition(checksum(bytes) == 6)
precondition(checksum([]) == 0)
var copy = ownedCopy(bytes)
copy[0] = 9
precondition(bytes == [1, 2, 3] && copy == [9, 2, 3])
```

This small sum's domain cannot overflow UInt64. Production aggregation still needs a declared arithmetic policy.

**Reasoning:** Array supplies initialized UInt8 elements and the buffer bounds; the helper does not allocate or bind raw memory manually. Returning an Array copies the value contract rather than retaining the pointer. No undefined dangling-pointer example is executed.

## In practice

**Choose it when:** Confine native interop to a narrow boundary and return Swift-owned values.

**Investigate by:** Audit who retains each buffer and exactly which event ends that ownership.

**Strong evidence:** Document initialization, bounds, alignment, retention, and cleanup, then combine compiler checking with targeted sanitizers and native integration tests.

### Binding and checked Swift 6.4 operations — optional

#### Binding, rebinding, and assuming binding

Swift distinguishes:

> raw memory
> typed memory bound to T
> temporary rebound memory

Important APIs:

> bindMemory(to:capacity:)
> assumingMemoryBound(to:)
> withMemoryRebound(to:capacity:_:)

Use the right one:

|API|Meaning|Risk|
|---|---|---|
|`bindMemory(to:capacity:)`|Bind raw memory to a type.|Wrong if memory is already bound incompatibly.|
|`assumingMemoryBound(to:)`|Assert memory is already bound to that type.|Undefined behavior if your assertion is false.|
|`withMemoryRebound(to:capacity:_:)`|Temporarily view memory as a layout-compatible type.|Pointer must not escape the closure.|

Equal byte size alone does not establish legal rebinding. Use the API's documented layout/binding rules, and do not mix incompatible accesses while a rebound view is active.

Swift’s migration guide for `UnsafeRawPointer` explains that raw pointer conversion was tightened so Swift can enforce type safety around unsafe pointer conversion; it explicitly points to `assumingMemoryBound`, `bindMemory`, and `withMemoryRebound` as the intended APIs. ([Swift.org, "UnsafeRawPointer Migration"](https://swift.org/migration-guide-swift3/se-0107-migrate.html))

Example: C socket APIs often require rebinding `sockaddr_in` to `sockaddr`.

Darwin-specific shape; no socket operation is performed here:

```swift
import Darwin

var address = sockaddr_in()

withUnsafePointer(to: &address) { pointer in
    pointer.withMemoryRebound(to: sockaddr.self, capacity: 1) { sockaddrPointer in
        // Pass sockaddrPointer to C API here.
        // Do not store sockaddrPointer.
    }
}
```

The temporary pointer must not escape. Swift’s migration guide states that the closure argument for `withMemoryRebound` must not escape the closure and that memory is rebound back after the closure returns. ([Swift.org, "UnsafeRawPointer Migration"](https://swift.org/migration-guide-swift3/se-0107-migrate.html))

#### Swift 6.4 alternatives: temporary spans and safe byte operations

Prefer withTemporaryAllocation when a synchronous computation needs temporary initialized storage. OutputSpan tracks a typed initialized prefix; OutputRawSpan tracks initialized bytes. The helper handles cleanup on return or throw, while a returned Span is rejected by its nonescapable contract. Capacity is not initialized count. C12 includes the typed version and cleanup-on-error probe. [SE-0524](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md)

For byte formats, use the checked raw-span overloads when the value types meet their representation constraints. The following has no pointer conversion and passes strict memory-safety checking with that warning group treated as errors:

```swift
@available(anyAppleOS 27.0, *)
func demo() {
    let field: UInt32 = withTemporaryAllocation(byteCount: 6, alignment: 1) { output in
        output.append(0xFF, as: UInt8.self)
        output.append(258, as: UInt32.self, .bigEndian)
        let bytes = output.bytes
        print(bytes.byteCount)
        return bytes.load(fromByteOffset: 1, as: UInt32.self, .bigEndian)
    }
    print(field)
}
if #available(anyAppleOS 27.0, *) { demo() }
```

Output is `5`, then `258`: six bytes of capacity contain five initialized bytes. The four-byte field begins at offset one and is encoded/decoded explicitly as big-endian. These ByteOrder overloads require Apple OS 27. The underlying temporary-allocation helper itself supports the older targets recorded in C12.

Safe raw loads require ConvertibleFromBytes; safe stores/appends use ConvertibleToBytes and their other stated constraints. BitwiseCopyable alone does not establish validity of every bit pattern or initialized padding. Keep unsafe generic loads for genuinely audited cases. Never infer a stable network/file encoding from arbitrary Swift struct layout. [SE-0525](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0525-rawspan-safe-loading-api.md)

Bounds-checked APIs may trap. For external data, validate offset and remaining length before a load; prove offset is in 0...byteCount before subtracting it. C12's parser checks avoid overflowing offset + width. A safe memory operation does not validate a packet's semantics.

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
