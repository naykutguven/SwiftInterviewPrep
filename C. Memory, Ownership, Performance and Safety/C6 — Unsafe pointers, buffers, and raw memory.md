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
## 0. Rubric snapshot

**Rubric expectation**

Know pointer lifetime, initialization, binding, rebinding, alignment, and when to prefer safe wrappers over unsafe memory APIs. The rubric explicitly calls out C6 as “Unsafe pointers, buffers, and raw memory” and uses pointer escape from `withUnsafeBufferPointer` as the core code probe.

**Caveats**

Most unsafe pointer bugs are lifetime or aliasing bugs, not syntax bugs. The code can compile, pass tests, and still be invalid.

**You should be able to answer**

- What is the difference between raw memory and typed memory in Swift?
- When is `withUnsafeBytes` / `withUnsafeBufferPointer` safer than storing an unsafe pointer for later?

**You should be able to do**

- Review code that escapes a pointer from `withUnsafeBytes` / `withUnsafeBufferPointer`.
- Explain exactly why it is invalid.
- Redesign it so the lifetime is owned and explicit.

---

## 1. Core mental model

Swift is memory-safe by default: initialized-before-use, no use-after-free in normal Swift code, bounds checks, exclusivity enforcement, and ARC-managed object lifetime. Unsafe pointer APIs are the escape hatch. They let you talk to C APIs, manually manage buffers, reinterpret bytes, and avoid some abstraction overhead, but Swift stops proving the full safety of the operation for you.

The word `Unsafe` is literal. Apple’s “Unsafe Swift” session describes unsafe operations as operations whose documented assumptions are not fully verified; violating those assumptions can produce undefined behavior rather than a clean Swift trap. ([Apple Developer, "Unsafe Swift"](https://developer.apple.com/videos/play/wwdc2020/10648/))

A pointer is **not ownership**. It is an address plus a type-level view. It does not keep an `Array`, `Data`, object, or manually allocated region alive. A buffer pointer adds a count, but it is still non-owning. Apple’s `UnsafeBufferPointer` documentation describes it as a non-owning collection interface to contiguous memory. ([Apple Developer, "UnsafeBufferPointer"](https://developer.apple.com/documentation/swift/unsafebufferpointer))

Raw memory is just bytes. Typed memory is memory that Swift is allowed to treat as containing initialized values of a specific type. Apple’s `UnsafeRawPointer` documentation says raw pointers provide no automated memory management, no type safety, and no alignment guarantees; you are responsible for the lifetime of memory accessed through them. ([Apple Developer, "UnsafeRawPointer"](https://developer.apple.com/documentation/swift/unsaferawpointer))

The key idea:

```text
Unsafe pointer = temporary address view, not ownership, not lifetime, not type safety.
```

Swift gives you temporary unsafe access through closure-scoped APIs such as `withUnsafeBufferPointer`. Those APIs are usually safer because the pointer’s lifetime is lexically scoped. Apple’s docs state that the pointer argument is valid only during the method’s execution and should not be stored or returned for later use. ([Apple Developer, "withUnsafeBufferPointer(_:)"](https://developer.apple.com/documentation/swift/arrayslice/withunsafebufferpointer%28_%3a%29))

---

## 2. Essential mechanics

### Pointer lifetime

Unsafe pointers do not extend the lifetime of the memory they point to.

```swift
let bytes: [UInt8] = [1, 2, 3]

bytes.withUnsafeBufferPointer { buffer in
    // Safe only inside this closure.
    print(buffer[0])
}
```

Inside the closure, Swift guarantees access to the array’s contiguous storage for the duration of the call. After the closure returns, the pointer is no longer valid.

Bad:

```swift
let bytes: [UInt8] = [1, 2, 3]
var escaped: UnsafePointer<UInt8>?

bytes.withUnsafeBufferPointer { buffer in
    escaped = buffer.baseAddress
}

print(escaped![0]) // Undefined behavior.
```

The pointer may appear to work because the array storage has not been reused yet. That does not make it valid.

---

### Initialization and deinitialization

Allocated memory is not automatically initialized.

```swift
let pointer = UnsafeMutablePointer<Int>.allocate(capacity: 3)

pointer.initialize(repeating: 0, count: 3)
pointer[0] = 42

print(pointer[0])

pointer.deinitialize(count: 3)
pointer.deallocate()
```

Manual memory has three separate states you must reason about:

```text
allocated raw memory
→ initialized typed values
→ deinitialized memory
→ deallocated memory
```

Common bugs:

```text
read before initialize
initialize twice without deinitialize
deinitialize memory that was never initialized
deallocate while still initialized
use after deallocate
```

For trivial types like `UInt8`, some mistakes are less visibly catastrophic. For reference-containing or nontrivial types, wrong initialization/deinitialization can leak, double-release, or corrupt object lifetime.

---

### Raw memory vs typed memory

Raw memory is byte-addressable and untyped.

```swift
var number: UInt32 = 0x01020304

withUnsafeBytes(of: &number) { rawBuffer in
    for byte in rawBuffer {
        print(byte)
    }
}
```

Typed memory is accessed through `UnsafePointer<T>` / `UnsafeMutablePointer<T>`.

```swift
var value = 42

withUnsafePointer(to: &value) { pointer in
    print(pointer.pointee) // Int
}
```

Raw buffer access is useful for serialization, hashing, C interop, compression, crypto, image/audio processing, and inspecting byte layouts. But raw bytes are not a stable cross-platform representation for arbitrary Swift values. Endianness, padding, alignment, reference fields, and resilience can matter.

Apple’s `UnsafeRawBufferPointer` documentation describes it as a view of raw bytes where each byte is viewed as a `UInt8`, independent of the type of values held in that memory. ([Apple Developer, "UnsafeRawBufferPointer"](https://developer.apple.com/documentation/swift/unsaferawbufferpointer))

---

### Binding, rebinding, and assuming binding

Swift distinguishes:

```text
raw memory
typed memory bound to T
temporary rebound memory
```

Important APIs:

```swift
bindMemory(to:capacity:)
assumingMemoryBound(to:)
withMemoryRebound(to:capacity:_:)
```

Use the right one:

|API|Meaning|Risk|
|---|---|---|
|`bindMemory(to:capacity:)`|Bind raw memory to a type.|Wrong if memory is already bound incompatibly.|
|`assumingMemoryBound(to:)`|Assert memory is already bound to that type.|Undefined behavior if your assertion is false.|
|`withMemoryRebound(to:capacity:_:)`|Temporarily view memory as a layout-compatible type.|Pointer must not escape the closure.|

Swift’s migration guide for `UnsafeRawPointer` explains that raw pointer conversion was tightened so Swift can enforce type safety around unsafe pointer conversion; it explicitly points to `assumingMemoryBound`, `bindMemory`, and `withMemoryRebound` as the intended APIs. ([Swift.org, "UnsafeRawPointer Migration"](https://swift.org/migration-guide-swift3/se-0107-migrate.html))

Example: C socket APIs often require rebinding `sockaddr_in` to `sockaddr`.

```swift
var address = sockaddr_in()

withUnsafePointer(to: &address) { pointer in
    pointer.withMemoryRebound(to: sockaddr.self, capacity: 1) { sockaddrPointer in
        // Pass sockaddrPointer to C API here.
        // Do not store sockaddrPointer.
    }
}
```

The temporary pointer must not escape. Swift’s migration guide states that the closure argument for `withMemoryRebound` must not escape the closure and that memory is rebound back after the closure returns. ([Swift.org, "UnsafeRawPointer Migration"](https://swift.org/migration-guide-swift3/se-0107-migrate.html))

---

### Alignment

Typed memory access must respect alignment.

```swift
let bytes: [UInt8] = [1, 0, 0, 0]

bytes.withUnsafeBytes { rawBuffer in
    let value = rawBuffer.load(as: UInt32.self)
    print(value)
}
```

This is only valid if the address is correctly aligned for `UInt32`. For byte streams where alignment is not guaranteed, use APIs that explicitly support unaligned loads.

```swift
bytes.withUnsafeBytes { rawBuffer in
    let value = rawBuffer.loadUnaligned(as: UInt32.self)
    print(value)
}
```

Wrong alignment can be undefined behavior or a hardware fault depending on platform and optimization.

---

## 3. Common traps and misconceptions

### Trap 1: “It printed the right value, so it is safe”

Bad:

```swift
let bytes: [UInt8] = [1, 2, 3]
var saved: UnsafePointer<UInt8>?

bytes.withUnsafeBufferPointer { buffer in
    saved = buffer.baseAddress
}

print(saved![0]) // May print 1. Still invalid.
```

Better:

```swift
let bytes: [UInt8] = [1, 2, 3]

bytes.withUnsafeBufferPointer { buffer in
    print(buffer[0])
}
```

The pointer’s lifetime is the closure. Escaping it creates a dangling pointer.

---

### Trap 2: Treating `UnsafeBufferPointer` as owning memory

`UnsafeBufferPointer` has a count and collection-like APIs, but it does not own the memory.

```swift
struct PacketView {
    let bytes: UnsafeBufferPointer<UInt8>
}
```

This type is dangerous unless it is guaranteed to be used only inside the owner’s lifetime. Prefer `Array`, `Data`, `ContiguousArray`, or a scoped closure.

Better:

```swift
struct Packet {
    private let storage: [UInt8]

    init(_ storage: [UInt8]) {
        self.storage = storage
    }

    func withBytes<R>(_ body: (UnsafeBufferPointer<UInt8>) throws -> R) rethrows -> R {
        try storage.withUnsafeBufferPointer(body)
    }
}
```

Here the public API does not expose a storable pointer.

---

### Trap 3: Confusing byte compatibility with type compatibility

Bad:

```swift
struct Header {
    var length: UInt16
    var kind: UInt16
}

let bytes: [UInt8] = [0x10, 0x00, 0x01, 0x00]

bytes.withUnsafeBytes { rawBuffer in
    let header = rawBuffer.load(as: Header.self)
    print(header)
}
```

This is fragile. It assumes alignment, layout, padding, endianness, and that Swift’s struct layout is suitable for the binary format.

Better:

```swift
struct Header {
    let length: UInt16
    let kind: UInt16
}

func parseHeader(from bytes: [UInt8]) throws -> Header {
    guard bytes.count >= 4 else {
        throw ParseError.truncated
    }

    let length = UInt16(bytes[0]) | (UInt16(bytes[1]) << 8)
    let kind = UInt16(bytes[2]) | (UInt16(bytes[3]) << 8)

    return Header(length: length, kind: kind)
}

enum ParseError: Error {
    case truncated
}
```

For wire formats, parse the format. Do not blindly reinterpret Swift memory layout as protocol data unless you fully control layout, alignment, and architecture assumptions.

---

### Trap 4: Returning unsafe pointers from safe-looking APIs

Bad:

```swift
func bytesPointer(for bytes: [UInt8]) -> UnsafePointer<UInt8> {
    bytes.withUnsafeBufferPointer { buffer in
        buffer.baseAddress!
    }
}
```

This is invalid because the returned pointer is only valid inside the closure.

Better:

```swift
func withBytesPointer<R>(
    for bytes: [UInt8],
    _ body: (UnsafePointer<UInt8>, Int) throws -> R
) rethrows -> R {
    try bytes.withUnsafeBufferPointer { buffer in
        try body(buffer.baseAddress!, buffer.count)
    }
}
```

This design makes the pointer lifetime impossible to misunderstand.

---

## 4. Direct answers to rubric questions

### Q1. What is the difference between raw memory and typed memory in Swift?

Raw memory is untyped bytes. Typed memory is memory that Swift is allowed to access as initialized values of a specific type.

Raw memory APIs:

```swift
UnsafeRawPointer
UnsafeMutableRawPointer
UnsafeRawBufferPointer
UnsafeMutableRawBufferPointer
```

Typed memory APIs:

```swift
UnsafePointer<T>
UnsafeMutablePointer<T>
UnsafeBufferPointer<T>
UnsafeMutableBufferPointer<T>
```

Raw memory has no element type beyond bytes. Typed memory carries a `Pointee` type, so operations like `pointee`, subscript, initialization, and deinitialization are interpreted as operations on `T`.

The dangerous part is that raw memory can be bound to a type, rebound temporarily, or accessed under an assumed binding. If the memory is uninitialized, incorrectly bound, misaligned, or not actually storing values of that type, typed access is undefined behavior.

Interview version:

> Raw memory is just bytes; typed memory is memory Swift may treat as initialized instances of a specific type. `UnsafeRawPointer` lets me inspect or copy bytes, but it does not give type safety, lifetime management, or alignment guarantees. To access values as `T`, the memory must be initialized, correctly aligned, and bound to `T`, or temporarily rebound under strict rules.

---

### Q2. When is `withUnsafeBytes` safer than storing an unsafe pointer for later?

`withUnsafeBytes` is safer when the callee only needs temporary access. The closure expresses the lifetime boundary directly: the pointer exists only for the duration of the closure call.

Good use:

```swift
let data: Data = ...

let checksum = data.withUnsafeBytes { rawBuffer in
    computeChecksum(rawBuffer)
}
```

Dangerous use:

```swift
var saved: UnsafeRawPointer?

data.withUnsafeBytes { rawBuffer in
    saved = rawBuffer.baseAddress
}

useLater(saved!) // Invalid.
```

The closure-based API lets the owner — `Array`, `Data`, `String`, etc. — keep its storage valid only for the operation. Storing the pointer breaks that contract. Apple’s documentation for array buffer access says the pointer argument is valid only for the duration of the method’s execution. ([Apple Developer, "withUnsafeBufferPointer(_:)"](https://developer.apple.com/documentation/swift/array/withunsafebufferpointer%28_%3a%29))

Interview version:

> `withUnsafeBytes` is safer because it scopes the unsafe pointer to a closure, so the owner controls the storage lifetime. Storing an unsafe pointer for later is usually wrong because the pointer does not retain or own the backing storage. After the closure returns, the collection may move, deallocate, copy, or reuse its storage, so the saved pointer becomes dangling.

---

### Q3. What makes unsafe pointer bugs hard to diagnose?

They often violate assumptions the compiler and runtime do not fully check.

A bad pointer may:

```text
print the expected value
work in debug
fail only in release
fail only on device
fail only after unrelated allocation changes
corrupt memory far from the actual bug
```

Unsafe bugs are usually caused by:

```text
lifetime escape
use after free
incorrect binding
wrong alignment
out-of-bounds access
incorrect ownership transfer with C APIs
aliasing mutable memory in unsupported ways
```

Interview version:

> Unsafe bugs are hard because undefined behavior is not required to fail near the source of the bug. The program may appear correct until optimization, allocator behavior, or unrelated code changes expose it. A senior engineer should treat unsafe code as a small audited boundary with documented lifetime, ownership, binding, and alignment invariants.

---

## 5. Code probe

Given:

```swift
let bytes: [UInt8] = [1, 2, 3]
var saved: UnsafePointer<UInt8>?

bytes.withUnsafeBufferPointer { buffer in
    saved = buffer.baseAddress
}

print(saved![0])
```

### What happens?

Observed with Swift 6.2.1 in a normal compile/run:

```text
1
```

But the language-level answer is:

```text
Undefined behavior. No output is guaranteed.
```

With Swift 6.2.1 and `-strict-memory-safety`, the compiler also warns that unsafe constructs are being used without explicit unsafe acknowledgement:

```text
warning: expression uses unsafe constructs but is not marked with 'unsafe' [#StrictMemorySafety]
```

Swift’s strict memory safety diagnostics are opt-in and are intended to surface uses of memory-unsafe constructs such as unsafe pointer operations. ([Swift.org, "Strict Memory Safety"](https://docs.swift.org/compiler/documentation/diagnostics/strict-memory-safety/))

### Why?

Step by step:

```text
bytes owns Array storage
        │
        ▼
withUnsafeBufferPointer creates temporary non-owning buffer view
        │
        ▼
buffer.baseAddress points into bytes storage
        │
        ▼
saved stores that address outside the closure
        │
        ▼
closure returns
        │
        ▼
the lifetime guarantee for buffer/baseAddress ends
        │
        ▼
saved is now a dangling pointer
```

The important point: `saved` does not keep `bytes` alive, does not keep the array buffer pinned, and does not preserve the temporary validity contract created by `withUnsafeBufferPointer`.

Even worse, this code is deceptively likely to print `1` in simple runs because `bytes` is still in scope and the buffer may not have moved. But Swift does not guarantee that this pointer remains valid after the closure. Apple’s “Unsafe Swift” session explicitly warns that temporary pointers obtained from Swift collections are valid only for the call duration, and escaping them leads to undefined behavior. ([Apple Developer, "Unsafe Swift"](https://developer.apple.com/videos/play/wwdc2020/10648/))

### Fix or redesign

#### Fix 1: Use the pointer inside the closure

```swift
let bytes: [UInt8] = [1, 2, 3]

bytes.withUnsafeBufferPointer { buffer in
    guard let baseAddress = buffer.baseAddress else {
        return
    }

    print(baseAddress[0])
}
```

Output:

```text
1
```

#### Fix 2: Copy the value, not the pointer

```swift
let bytes: [UInt8] = [1, 2, 3]
let first = bytes.withUnsafeBufferPointer { buffer in
    buffer[0]
}

print(first)
```

Output:

```text
1
```

#### Fix 3: If you need long-lived memory, own it manually

```swift
final class ByteBuffer {
    private let pointer: UnsafeMutablePointer<UInt8>
    let count: Int

    init(bytes: [UInt8]) {
        self.count = bytes.count
        self.pointer = UnsafeMutablePointer<UInt8>.allocate(capacity: bytes.count)

        pointer.initialize(from: bytes, count: bytes.count)
    }

    deinit {
        pointer.deinitialize(count: count)
        pointer.deallocate()
    }

    func withUnsafeBufferPointer<R>(
        _ body: (UnsafeBufferPointer<UInt8>) throws -> R
    ) rethrows -> R {
        let buffer = UnsafeBufferPointer(start: pointer, count: count)
        return try body(buffer)
    }
}

let buffer = ByteBuffer(bytes: [1, 2, 3])

buffer.withUnsafeBufferPointer { bytes in
    print(bytes[0])
}
```

Output:

```text
1
```

### Why this fix is correct

The first two fixes keep the pointer inside the closure lifetime. No pointer escapes.

The third fix changes the ownership model. Instead of borrowing an array’s temporary storage, `ByteBuffer` allocates and owns memory explicitly. Its `deinit` releases the memory. The unsafe pointer is still not exposed as a long-lived public value; users get scoped access through `withUnsafeBufferPointer`.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Use pointer only inside `withUnsafeBufferPointer`|Passing array storage to a synchronous C API|Cannot support APIs that retain the pointer|
|Copy to `Array` / `Data`|You need owned bytes with safe lifetime|May allocate/copy|
|Own memory with `UnsafeMutablePointer.allocate`|C API needs long-lived manually managed memory|Must handle initialize/deinitialize/deallocate correctly|
|Use `ManagedBuffer` / custom storage object|Library-level storage abstraction|More complex API and invariants|
|Wrap unsafe API behind safe closure-based API|Production SDK or module boundary|Requires careful wrapper design|

---

## 6. Exercise

### Problem

Review code that escapes a pointer from `withUnsafeBytes`; explain exactly why it is invalid.

### Bad / naive version

```swift
import Foundation

final class DigestContext {
    private var savedPointer: UnsafeRawPointer?
    private var savedCount: Int = 0

    func update(with data: Data) {
        data.withUnsafeBytes { rawBuffer in
            savedPointer = rawBuffer.baseAddress
            savedCount = rawBuffer.count
        }
    }

    func finish() -> UInt8 {
        guard let savedPointer else {
            return 0
        }

        let bytes = savedPointer.assumingMemoryBound(to: UInt8.self)
        return bytes[0]
    }
}
```

### What is wrong?

```text
savedPointer points into Data's temporary byte storage.
The pointer is valid only inside withUnsafeBytes.
After update(with:) returns, savedPointer is dangling.
finish() reads from memory whose lifetime is no longer guaranteed.
assumingMemoryBound(to:) adds another unchecked assertion.
The code may appear to work until Data storage changes, moves, or deallocates.
```

This is a lifetime bug first. It may also become a binding bug depending on how the memory is later accessed.

### Improved version A: consume the bytes synchronously

```swift
import Foundation

struct Digest {
    static func firstByte(of data: Data) -> UInt8? {
        data.withUnsafeBytes { rawBuffer in
            guard let baseAddress = rawBuffer.baseAddress, rawBuffer.count > 0 else {
                return nil
            }

            return baseAddress.assumingMemoryBound(to: UInt8.self)[0]
        }
    }
}
```

This keeps pointer access scoped to the closure.

### Improved version B: store owned bytes, not borrowed pointer

```swift
import Foundation

final class DigestContext {
    private var storage = Data()

    func update(with data: Data) {
        storage.append(data)
    }

    func finish() -> UInt8? {
        storage.first
    }
}
```

This is the right default design for app and SDK code. Store safe owned values. Borrow pointers only at interop boundaries.

### Improved version C: long-lived manual storage

```swift
final class OwnedRawBytes {
    private let pointer: UnsafeMutableRawPointer
    let count: Int

    init(bytes: [UInt8]) {
        self.count = bytes.count
        self.pointer = UnsafeMutableRawPointer.allocate(
            byteCount: bytes.count,
            alignment: MemoryLayout<UInt8>.alignment
        )

        pointer.copyMemory(from: bytes, byteCount: bytes.count)
    }

    deinit {
        pointer.deallocate()
    }

    func withUnsafeBytes<R>(
        _ body: (UnsafeRawBufferPointer) throws -> R
    ) rethrows -> R {
        let buffer = UnsafeRawBufferPointer(start: pointer, count: count)
        return try body(buffer)
    }
}
```

### Why this is better

The design makes ownership explicit:

```text
Data / Array version:
safe owned storage → temporary unsafe borrow → no escape

Manual version:
manual owned allocation → scoped unsafe borrow → deallocate in deinit
```

A production-quality wrapper should also document whether the memory is mutable, whether it can be shared across threads, whether C is allowed to retain the pointer, and who owns deallocation.

---

## 7. Production guidance

Use this in production when:

```text
You are wrapping a C / Objective-C / system API.
You are implementing a performance-sensitive primitive with measured need.
You are working with binary formats, audio, video, graphics, crypto, compression, or networking buffers.
You can isolate the unsafe code behind a small safe API.
You can write down the ownership, lifetime, binding, and alignment invariants.
```

Be careful when:

```text
A pointer escapes a closure.
A C API may store the pointer after returning.
You reinterpret bytes as a Swift struct.
The memory may contain object references.
You use assumingMemoryBound(to:).
You mutate through multiple aliases.
You rely on debug behavior or a single successful run.
You cross actor/task/thread boundaries with unsafe storage.
```

Avoid when:

```text
Array, Data, ContiguousArray, String, Span, or a safe wrapper expresses the same thing.
The unsafe code exists only to avoid a tiny unmeasured copy.
The API exposes UnsafePointer as public long-lived state.
The ownership contract depends on comments nobody will read.
You are parsing external data by loading Swift structs directly from bytes.
```

Debugging checklist:

```text
Who owns this memory?
Who deallocates it?
Is the memory initialized?
What type is the memory bound to?
Is the access aligned for that type?
Does the pointer escape a withUnsafe... closure?
Can the owner move, copy, or deallocate the backing storage?
Does a C API retain the pointer after returning?
Can mutation happen through another alias?
Does the bug reproduce only in release or only on device?
Have Address Sanitizer, Thread Sanitizer, Undefined Behavior Sanitizer, Zombies, or Guard Malloc been tried where applicable?
Can the unsafe region be replaced with Data, Array, or a closure-scoped wrapper?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Unsafe pointers let Swift access memory directly. You need to be careful because they can crash.

### Senior answer

> Unsafe pointers are non-owning views into memory. `withUnsafeBufferPointer` gives a temporary pointer valid only inside the closure. If you store the pointer and use it later, you have a dangling pointer and undefined behavior, even if it prints the expected value. I would redesign the code to either copy the value/data or keep pointer access closure-scoped.

### Staff-level answer

> Unsafe memory should be quarantined behind a small audited boundary. The API should encode ownership and lifetime so callers cannot misuse it: safe value types for storage, closure-scoped borrows for temporary pointer access, and explicit manually managed storage only when an external API truly requires long-lived memory. I would document and test the invariants around initialization, binding, alignment, aliasing, deallocation, and whether C can retain the pointer. For Swift 6.2-era code, I would also consider strict memory safety diagnostics as a way to make unsafe boundaries explicit during migration and review.

Staff-level questions to ask:

```text
Can this unsafe API become a closure-scoped borrow instead of returning a pointer?
Can we store Data / Array instead of an UnsafePointer?
Does the C API consume the pointer synchronously, retain it, mutate it, or take ownership?
What is the bound type of this memory at every point?
Are we relying on Swift struct layout for an external binary format?
What happens under optimization, on arm64, and with sanitizers enabled?
Can strict memory safety mode make this boundary more auditable?
Is this unsafe code private to one module, or leaking into public API?
```

---

## 9. Interview-ready summary

Unsafe pointers in Swift are non-owning views into memory. They do not manage lifetime, initialization, binding, alignment, or aliasing for you. Raw memory is bytes; typed memory is memory Swift may access as initialized values of a specific type. `withUnsafeBytes` and `withUnsafeBufferPointer` are safer because they scope the pointer lifetime to a closure. Escaping the pointer is invalid because the backing storage is only guaranteed during the closure call. In production, unsafe memory should be isolated behind safe APIs that encode ownership and lifetime, and long-lived unsafe memory should be manually owned and released with clearly documented invariants.

---

## 10. Flashcards

Q: What does `UnsafePointer<T>` own?

A: Nothing. It is a non-owning typed address view. It does not keep the pointed-to memory alive.

Q: Why is escaping a pointer from `withUnsafeBufferPointer` invalid?

A: The pointer is only valid for the duration of the closure. After the closure returns, using the saved pointer is undefined behavior.

Q: What is raw memory?

A: Untyped bytes accessed through `UnsafeRawPointer` / `UnsafeRawBufferPointer`. It has no safe typed interpretation until binding or loading rules are satisfied.

Q: What is typed memory?

A: Memory that is initialized, correctly aligned, and bound to a specific type so Swift can access it as values of that type.

Q: What does `assumingMemoryBound(to:)` mean?

A: “I assert this memory is already bound to this type.” If that assertion is false, behavior is undefined.

Q: When should you use `withMemoryRebound(to:capacity:_:)`?

A: When temporarily passing memory to an API that expects a layout-compatible type, often C APIs like socket functions. The rebound pointer must not escape.

Q: Why can unsafe pointer bugs pass tests?

A: Undefined behavior can coincidentally produce expected results until optimization, allocation patterns, or platform details change.

Q: What is the safest default wrapper style?

A: Store safe owned data, such as `Array` or `Data`, and expose temporary pointer access through a `withUnsafe...` closure.

---

## 11. Related sections

- [[C1 — ARC fundamentals]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C5 — Collection protocols, complexity, and index invalidation]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[C11 — Strict memory safety mode and safe systems programming]]
- [[C12 — Span, InlineArray, and modern low-level abstractions]]
- [[E3 — C interoperability and unsafe boundaries]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — C6 unsafe pointers/buffers/raw memory section.
- Apple Developer. "UnsafeRawPointer." Apple Developer Documentation. https://developer.apple.com/documentation/swift/unsaferawpointer
- Apple Developer. "UnsafeBufferPointer." Apple Developer Documentation. https://developer.apple.com/documentation/swift/unsafebufferpointer
- Apple Developer. "withUnsafeBufferPointer(_:)." Apple Developer Documentation. https://developer.apple.com/documentation/swift/array/withunsafebufferpointer%28_%3a%29
- Apple Developer. "UnsafeRawBufferPointer." Apple Developer Documentation. https://developer.apple.com/documentation/swift/unsaferawbufferpointer
- Swift.org. "UnsafeRawPointer Migration." Swift 3 Migration Guide. https://swift.org/migration-guide-swift3/se-0107-migrate.html
- Swift.org. "Strict Memory Safety." Swift Compiler Diagnostics. https://docs.swift.org/compiler/documentation/diagnostics/strict-memory-safety/
- Apple Developer. "Unsafe Swift." WWDC20. https://developer.apple.com/videos/play/wwdc2020/10648/
