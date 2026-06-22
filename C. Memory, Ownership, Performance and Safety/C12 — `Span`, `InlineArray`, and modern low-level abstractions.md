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
## 0. Rubric snapshot

**Rubric expectation**

Know that modern Swift increasingly offers safe low-level tools for contiguous memory and fixed-size inline storage. The rubric positions C12 as a strong senior / emerging staff topic, with deeper `Span` / `InlineArray` expertise as a staff-level differentiator.

**Caveats**

These are specialized tools. Misuse usually comes from adopting them because they look “more performant,” without a concrete memory-layout, safety, or allocation reason.

**You should be able to answer**

- What problem does `Span` solve better than raw pointers?
- When is a fixed-size inline container a better fit than `Array`?

**You should be able to do**

- Describe a real scenario in graphics, parsing, crypto, or DSP where `InlineArray` or `Span` would be appropriate.

---

## 1. Core mental model

Swift’s older low-level story often forced a tradeoff:

```text
Safe but higher-level: Array, Data, Collection
Fast and contiguous but unsafe: UnsafeBufferPointer, raw pointers
```

`Span` and `InlineArray` reduce that gap.

`Span<Element>` is a **non-owning, non-escaping view** into contiguous initialized memory. It is not a container. It does not own storage. It borrows storage from something else and gives you local, bounds-checked, memory-safe access to that storage. Swift 6.2 describes `Span` as safe direct access to contiguous memory where validity is enforced while the span is in use, avoiding pointer-style use-after-free bugs. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org")) Apple’s documentation similarly describes `Span` as a non-owning, non-escaping view into memory whose lifetime is inherited from the owning container. ([Apple Developer](https://developer.apple.com/documentation/swift/span?utm_source=chatgpt.com "Span | Apple Developer Documentation"))

`InlineArray<N, Element>` is a **fixed-size inline storage container**. Its count is part of the type. Unlike `Array`, it does not have separate heap storage just to hold its elements, does not grow, does not shrink, and does not use copy-on-write semantics. Swift 6.2 introduced it as a fixed-size array with inline storage that can live on the stack or directly inside another type. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org")) SE-0453 defines it as a fixed-size, contiguously inline allocated array whose storage follows the natural allocation pattern of the containing value and does not introduce an implicit heap allocation just for the elements. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md "swift-evolution/proposals/0453-vector.md at main · swiftlang/swift-evolution · GitHub"))

The relationship:

```text
InlineArray = owns fixed-size inline storage
Span        = borrows contiguous storage owned by something else
```

Use `InlineArray` when the **size is a semantic fact**. Use `Span` when the **caller owns storage, but your API only needs temporary contiguous access**.

Swift guarantees more than C pointers here, but not magic. `Span` improves temporal safety, spatial safety, definite initialization, and type safety, but it is not owned storage and cannot safely escape. `InlineArray` removes dynamic array growth and separate element storage allocation, but it is not a drop-in `Array` replacement and should not be used casually.

For iOS app work with an iOS 18.0 minimum deployment target, availability is a real constraint. Apple documentation search snippets list `InlineArray` members as iOS 26.0+, so these APIs generally need availability-gated use or fallbacks in iOS 18-compatible app code. ([Apple Developer](https://developer.apple.com/documentation/swift/inlinearray/index%28after%3A%29?utm_source=chatgpt.com "index(after:) | Apple Developer Documentation"))

---

## 2. Essential mechanics

### `Span` is borrowed contiguous access, not ownership

A `Span` lets an API say:

```text
I need temporary access to initialized contiguous elements.
I do not need to own, retain, resize, or store the container.
```

Conceptual API shape:

```swift
@available(iOS 26.0, macOS 26.0, *)
func hasPNGSignature(_ bytes: Span<UInt8>) -> Bool {
    guard bytes.count >= 8 else { return false }

    return bytes[0] == 0x89 &&
           bytes[1] == 0x50 &&
           bytes[2] == 0x4E &&
           bytes[3] == 0x47 &&
           bytes[4] == 0x0D &&
           bytes[5] == 0x0A &&
           bytes[6] == 0x1A &&
           bytes[7] == 0x0A
}
```

This is better than accepting `UnsafeBufferPointer<UInt8>` for normal parser logic because the unsafe boundary does not spread through the codebase. SE-0447 specifically calls out that unsafe buffer pointers are unsafe because the pointer is unmanaged, subscripting is only debug-bounds-checked in client code, and the pointer can escape the closure duration. `Span` is designed to be the safe “currency type” for local processing over contiguous memory. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md?utm_source=chatgpt.com "swift-evolution/proposals/0447-span-access-shared-contiguous-storage.md at main · swiftlang/swift-evolution · GitHub"))

### `Span` access is bounds-checked, but unchecked access exists

`Span` provides count, indices, and subscript-like buffer access. SE-0447 specifies that normal subscript access has always-on bounds checking to preserve spatial safety, while `subscript(unchecked:)` exists for cases where the caller has proven index validity in a tight loop. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md?utm_source=chatgpt.com "swift-evolution/proposals/0447-span-access-shared-contiguous-storage.md at main · swiftlang/swift-evolution · GitHub"))

Safe default:

```swift
@available(iOS 26.0, macOS 26.0, *)
func sum(_ bytes: Span<UInt8>) -> UInt64 {
    var result: UInt64 = 0

    for i in bytes.indices {
        result += UInt64(bytes[i])
    }

    return result
}
```

Unsafe variant only after proof:

```swift
@available(iOS 26.0, macOS 26.0, *)
func sumKnownNonEmptyPrefix(_ bytes: Span<UInt8>) -> UInt64 {
    precondition(bytes.count >= 4)

    return UInt64(bytes[unchecked: 0]) +
           UInt64(bytes[unchecked: 1]) +
           UInt64(bytes[unchecked: 2]) +
           UInt64(bytes[unchecked: 3])
}
```

The second version is not “more senior.” It is only justified if profiling proves the bounds checks matter and the invariant is local, obvious, and tested.

### `InlineArray` has size in the type

`InlineArray` uses integer generic parameters. SE-0452 introduced integer generic parameters in Swift 6.2, allowing generic types to be parameterized by literal integer values. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0452-integer-generic-parameters.md "swift-evolution/proposals/0452-integer-generic-parameters.md at main · swiftlang/swift-evolution · GitHub")) SE-0453 then uses that capability for `InlineArray<let count: Int, Element>`. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md "swift-evolution/proposals/0453-vector.md at main · swiftlang/swift-evolution · GitHub"))

Modern Swift 6.2 also has type sugar:

```swift
@available(iOS 26.0, macOS 26.0, *)
func checksum(_ header: [4 of UInt8]) -> UInt16 {
    var result: UInt16 = 0

    for i in header.indices {
        result += UInt16(header[i])
    }

    return result
}

@available(iOS 26.0, macOS 26.0, *)
func demoChecksum() {
    let header: [4 of UInt8] = [1, 2, 3, 4]
    print(checksum(header))
}
```

Output:

```text
10
```

`[4 of UInt8]` is sugar for `InlineArray<4, UInt8>`. SE-0483 introduced this type sugar and states that `[5 of Int]` is equivalent to `InlineArray<5, Int>`. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0483-inline-array-sugar.md "swift-evolution/proposals/0483-inline-array-sugar.md at main · swiftlang/swift-evolution · GitHub"))

### `InlineArray` is not `Array`

Bad:

```swift
@available(iOS 26.0, macOS 26.0, *)
func bad() {
    var bytes: [4 of UInt8] = [1, 2, 3, 4]
    bytes.append(5)
}
```

This does not compile because `InlineArray` is fixed-size. There is no `append`.

Better:

```swift
@available(iOS 26.0, macOS 26.0, *)
func replaceLastByte() {
    var bytes: [4 of UInt8] = [1, 2, 3, 4]
    bytes[3] = 5
    print(bytes[3])
}
```

Output:

```text
5
```

SE-0453 explicitly says `InlineArray` cannot append or remove elements and requires every element to be initialized. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md "swift-evolution/proposals/0453-vector.md at main · swiftlang/swift-evolution · GitHub"))

### `InlineArray` does not currently conform to `Sequence` or `Collection`

This matters. You can iterate through `indices`, but you should not assume every `Array`-like API is available.

```swift
@available(iOS 26.0, macOS 26.0, *)
func printBytes(_ bytes: [4 of UInt8]) {
    for i in bytes.indices {
        print(bytes[i])
    }
}
```

SE-0453 says `InlineArray` intentionally does not conform to `Sequence` or `Collection` because there is no copy-on-write behavior and generic collection APIs could accidentally introduce implicit eager copies. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md "swift-evolution/proposals/0453-vector.md at main · swiftlang/swift-evolution · GitHub"))

---

## 3. Common traps and misconceptions

### Trap 1: Treating `Span` as a stored buffer

`Span` is borrowed. It should be used for local processing, not saved for later.

Bad, conceptually:

```swift
@available(iOS 26.0, macOS 26.0, *)
struct Parser {
    var savedBytes: Span<UInt8> // Wrong model: Span is non-escaping borrowed access.
}
```

Better:

```swift
struct Parser {
    var ownedBytes: [UInt8]
}
```

Or expose parsing as a local operation:

```swift
@available(iOS 26.0, macOS 26.0, *)
struct HeaderParser {
    func parse(_ bytes: Span<UInt8>) -> Header? {
        guard bytes.count >= 4 else { return nil }
        return Header(
            magic0: bytes[0],
            magic1: bytes[1],
            magic2: bytes[2],
            magic3: bytes[3]
        )
    }
}

struct Header {
    let magic0: UInt8
    let magic1: UInt8
    let magic2: UInt8
    let magic3: UInt8
}
```

### Trap 2: Replacing every small `Array` with `InlineArray`

This is usually premature. `Array` is still the right default for dynamic data, app-level models, decoded network payloads, UI collections, and any sequence whose size is not a compile-time semantic invariant.

Bad:

```swift
@available(iOS 26.0, macOS 26.0, *)
struct SearchResults {
    var items: [20 of SearchResult]
}
```

This is probably wrong because search results are not semantically exactly 20 elements.

Better:

```swift
struct SearchResults {
    var items: [SearchResult]
}
```

Use `InlineArray` only when “exactly N” is a domain rule or memory-layout rule.

### Trap 3: Assuming inline means always stack

`InlineArray` means the elements are stored inline with the containing value. If the containing value lives on the stack, the inline storage is usually stack storage. If the containing value is a class instance, the storage is inline inside the heap-allocated object. SE-0453 explicitly defines “inline” as the natural allocation pattern of the context, not “always stack.” ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md "swift-evolution/proposals/0453-vector.md at main · swiftlang/swift-evolution · GitHub"))

```swift
@available(iOS 26.0, macOS 26.0, *)
final class Palette {
    // Stored inline inside the class object, which itself is heap allocated.
    var colors: [4 of UInt32] = [0, 0, 0, 0]
}
```

### Trap 4: Believing `InlineArray` has CoW behavior

It does not. `Array` has copy-on-write storage. `InlineArray` is inline storage; copying a copyable `InlineArray` copies its elements eagerly.

This is why generic collection conformances are tricky: ordinary collection algorithms may accidentally copy the whole inline buffer.

### Trap 5: Ignoring availability in app code

For an iOS 18.0-minimum app, do not expose `InlineArray` or `Span` directly in public APIs used by all deployment paths unless the API is availability-gated.

Bad:

```swift
public struct Packet {
    public var header: [4 of UInt8]
}
```

Better for broad deployment:

```swift
public struct Packet {
    public var header: Header
}

public struct Header {
    private var b0: UInt8
    private var b1: UInt8
    private var b2: UInt8
    private var b3: UInt8
}
```

Then optionally add modern fast paths behind availability:

```swift
extension Header {
    @available(iOS 26.0, macOS 26.0, *)
    public init(_ bytes: [4 of UInt8]) {
        self.b0 = bytes[0]
        self.b1 = bytes[1]
        self.b2 = bytes[2]
        self.b3 = bytes[3]
    }
}
```

---

## 4. Direct answers to rubric questions

### Q1. What problem does `Span` solve better than raw pointers?

`Span` solves the problem of passing temporary contiguous memory through an API without losing Swift’s memory-safety model.

Raw pointers are fast, but they make the programmer manually prove lifetime, bounds, initialization, type binding, and non-escaping behavior. `Span` gives you a borrowed, non-owning view into initialized contiguous storage with compiler-enforced lifetime constraints and bounds-checked access. SE-0447 describes `Span` as preserving temporal safety, spatial safety, definite initialization, and type safety while enabling pointer-like local processing over contiguous memory. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md?utm_source=chatgpt.com "swift-evolution/proposals/0447-span-access-shared-contiguous-storage.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> `Span` is the safe API currency for temporary contiguous memory. It lets a function operate over memory owned by `Array`, `Data`, `String.UTF8View`, or another contiguous container without taking ownership and without exposing raw pointers across the codebase. Compared with `UnsafeBufferPointer`, it keeps the lifetime tied to the owner, bounds-checks normal access, and prevents the usual use-after-free and escaped-pointer bugs. I would still keep truly unsafe pointer conversion at narrow interop boundaries, but I would prefer `Span` for parser, crypto, graphics, or DSP code that only needs local read access.

### Q2. When is a fixed-size inline container a better fit than `Array`?

Use a fixed-size inline container when the count is known at compile time, the count is part of the domain invariant, and dynamic allocation / growth / CoW behavior are unnecessary or harmful.

Good examples:

```text
RGBA = exactly 4 channels
AES block = exactly 16 bytes
SHA-256 block = exactly 64 bytes
Matrix4x4 = exactly 16 scalars
Audio biquad coefficients = exactly 5 coefficients
SIMD-ish small work buffer = fixed, hot, local
```

`Array` is better when the count is dynamic, when you need append/remove, when you need ordinary collection APIs, or when clarity matters more than memory layout.

Interview version:

> `InlineArray` is appropriate when “exactly N elements” is a semantic or layout requirement, not merely today’s expected size. It avoids separate heap storage and dynamic resizing overhead, but it gives up `Array`’s flexibility and copy-on-write behavior. I would use it for fixed protocol headers, crypto blocks, graphics matrices, or tight DSP kernels after measuring. I would not use it for normal app collections like table rows, search results, network models, or anything whose count can naturally vary.

---

## 5. Code probe / examples

The rubric has no code probe for C12, so use these instead.

### Minimal example: fixed-size protocol header

```swift
@available(iOS 26.0, macOS 26.0, *)
struct PacketHeader {
    private var bytes: [4 of UInt8]

    init(_ bytes: [4 of UInt8]) {
        self.bytes = bytes
    }

    var version: UInt8 {
        bytes[0]
    }

    var kind: UInt8 {
        bytes[1]
    }

    var payloadLength: UInt16 {
        UInt16(bytes[2]) << 8 | UInt16(bytes[3])
    }
}

@available(iOS 26.0, macOS 26.0, *)
func demoHeader() {
    let header = PacketHeader([1, 7, 0, 42])
    print(header.version)
    print(header.kind)
    print(header.payloadLength)
}
```

Output:

```text
1
7
42
```

### Why this example fits

```text
PacketHeader is exactly 4 bytes.
The fixed size is a protocol invariant.
The type prevents accidentally constructing a 3-byte or 5-byte header.
No append/remove/growth behavior is needed.
```

### Counterexample: dynamic UI state

```swift
@available(iOS 26.0, macOS 26.0, *)
struct BadSearchState {
    var results: [20 of SearchResult]
}

struct SearchResult {
    let title: String
}
```

This is the wrong abstraction. Search results are not naturally exactly 20 elements. You would end up inventing sentinel values, empty placeholders, or extra count tracking.

Better:

```swift
struct SearchState {
    var results: [SearchResult]
}
```

### Production-style example: parser API over borrowed bytes

```swift
@available(iOS 26.0, macOS 26.0, *)
struct BinaryParser {
    func readUInt32BE(from bytes: Span<UInt8>, at offset: Int) -> UInt32? {
        guard offset >= 0 else { return nil }
        guard bytes.count >= offset + 4 else { return nil }

        return UInt32(bytes[offset]) << 24 |
               UInt32(bytes[offset + 1]) << 16 |
               UInt32(bytes[offset + 2]) << 8 |
               UInt32(bytes[offset + 3])
    }
}
```

This API says the parser does not own the bytes. It only needs local contiguous access.

---

## 6. Exercise

### Problem

Describe a real scenario in graphics, parsing, crypto, or DSP where `InlineArray` or `Span` would be appropriate.

### Scenario: crypto block compression

A SHA-256 compression function operates on 512-bit blocks, i.e. exactly 64 bytes. That is a strong fit for `InlineArray` because the size is not incidental; it is part of the algorithm. A higher-level hash function may accept arbitrary input as borrowed contiguous bytes, which is a strong fit for `Span`.

### Bad / naive version

```swift
struct SHA256Block {
    var bytes: [UInt8]

    init(bytes: [UInt8]) {
        self.bytes = bytes
    }
}

func compress(_ block: SHA256Block) {
    // Assumes block.bytes.count == 64.
}
```

### What is wrong?

```text
The type allows invalid states:
- 0 bytes
- 63 bytes
- 65 bytes
- any dynamically resized byte array

The invariant “exactly 64 bytes” is only documented or checked at runtime.
The storage has dynamic Array behavior even though the algorithm does not need growth.
```

### Improved version

```swift
@available(iOS 26.0, macOS 26.0, *)
struct SHA256Block {
    private var bytes: [64 of UInt8]

    init(_ bytes: [64 of UInt8]) {
        self.bytes = bytes
    }

    subscript(index: Int) -> UInt8 {
        bytes[index]
    }
}

@available(iOS 26.0, macOS 26.0, *)
struct SHA256Compressor {
    func compress(_ block: SHA256Block) {
        // The block is guaranteed to contain exactly 64 bytes.
        // Compression rounds can safely assume the block-size invariant.
    }
}
```

### Span-based input processing

```swift
@available(iOS 26.0, macOS 26.0, *)
struct SHA256InputProcessor {
    func process(_ input: Span<UInt8>) {
        var offset = 0

        while offset + 64 <= input.count {
            // In real code, construct a SHA256Block from this 64-byte region.
            // Span is appropriate here because input is borrowed, not owned.
            offset += 64
        }

        // Remaining bytes are handled by padding logic.
    }
}
```

### Why this is better

```text
InlineArray models the fixed 64-byte block invariant in the type.
Span models borrowed input without forcing ownership or unsafe pointer APIs.
The unsafe boundary, if any, stays at the edge where external memory enters Swift.
The algorithm code can talk in domain terms: block, input span, offset.
```

### Production caveat

For iOS 18-minimum shipping code, this exact `InlineArray` implementation must be availability-gated or replaced by a compatible representation, such as two `UInt64`s, a tuple-backed internal type, or a validated `[UInt8]` fallback. The design judgment is still useful: encode fixed-size invariants in types where the platform allows it.

---

## 7. Production guidance

Use this in production when:

```text
- The size is a compile-time invariant: 4 channels, 16-byte block, 64-byte block, 4x4 matrix.
- You are writing hot low-level code where allocation and CoW traffic are measurable costs.
- You need safe local contiguous-memory access without spreading UnsafePointer APIs.
- You are wrapping C/C++/system APIs and want unsafe code quarantined at the boundary.
- You are building parser, codec, crypto, graphics, embedded, or DSP infrastructure.
```

Be careful when:

```text
- Your app deployment target is below the availability of these APIs.
- You are exposing these types in public SDK APIs.
- You are tempted to use unchecked subscripts.
- You are replacing Array without profiling.
- The fixed size is a current implementation detail rather than a domain invariant.
- You need Sequence/Collection conformances.
```

Avoid when:

```text
- The number of elements is naturally dynamic.
- You need append/remove/filter/map-style ordinary collection workflows.
- The code is UI/model/business logic where clarity beats micro-layout control.
- You cannot availability-gate the API.
- Your team does not have tests or profiling around the low-level path.
```

Debugging checklist:

```text
Is the size truly fixed by the domain?
Is the Span escaping, stored, captured, or returned incorrectly?
Is the unsafe boundary narrow and auditable?
Is the use of unchecked indexing locally proven?
Did you measure allocation or copy costs before introducing InlineArray?
Does this code need to run on iOS 18?
Did the abstraction leak platform availability into public API?
Is a simpler tuple, struct, Array, Data, or ContiguousArray enough?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `InlineArray` is like a fixed-size array, and `Span` is like a safer pointer.

### Senior answer

> `InlineArray` owns fixed-size inline storage, so it is useful when the count is a real invariant and dynamic allocation is unnecessary. `Span` is a borrowed non-owning view into contiguous memory, useful for local parser or codec APIs that should not own storage or expose unsafe pointers. I would use both only with availability checks and after measuring the hot path.

### Staff-level answer

> I would treat `Span` and `InlineArray` as API-design tools for making low-level invariants explicit. `Span` belongs at boundaries where an algorithm needs temporary contiguous access but should not own or escape storage. `InlineArray` belongs where fixed size is a semantic or layout guarantee, such as protocol headers, crypto blocks, matrices, or DSP coefficients. I would not leak these types broadly into an SDK without thinking about platform availability, source compatibility, resilience, and migration. For an iOS 18-minimum app, I would usually hide them behind availability-gated internal fast paths or keep using stable fallback representations until the deployment target moves.

Staff-level questions to ask:

```text
Is this fixed size a domain invariant or an optimization detail?
Can the public API remain stable if we later remove InlineArray?
Does this need to work on older OS versions?
Where is the unsafe pointer boundary, and can we quarantine it?
What benchmarks prove this is worth the complexity?
Can tests prove bounds, lifetime, and malformed-input behavior?
Could this be modeled as a tuple, small struct, Array, Data, or ContiguousArray instead?
```

---

## 9. Interview-ready summary

`Span` and `InlineArray` are part of Swift’s newer safe-systems-programming direction. `Span` is a borrowed, non-owning, non-escaping view into contiguous initialized memory; it is a safer API currency than raw pointers for local parsing, crypto, graphics, and DSP algorithms. `InlineArray` is fixed-size inline storage where the count is part of the type, useful when “exactly N elements” is a real invariant and separate dynamic array storage is unnecessary. The senior-level judgment is not to replace `Array` everywhere, but to use these tools where they encode correctness, reduce unsafe surface area, or remove measured allocation overhead. For iOS app code, availability and public API leakage are major design constraints.

---

## 10. Flashcards

Q: What is the core difference between `Span` and `InlineArray`?  
A: `Span` borrows contiguous storage owned by something else; `InlineArray` owns fixed-size inline storage.

Q: Why is `Span` safer than `UnsafeBufferPointer`?  
A: It is non-escaping, lifetime-tied to the owner, normally bounds-checked, and represents initialized memory, reducing use-after-free and escaped-pointer bugs.

Q: Is `Span` an owned collection?  
A: No. It is a borrowed view for local processing. Store owned data in `Array`, `Data`, `InlineArray`, or another owning type.

Q: When is `InlineArray` better than `Array`?  
A: When the count is a compile-time semantic invariant and dynamic growth, heap storage, and CoW behavior are unnecessary or harmful.

Q: Does `InlineArray` conform to `Sequence` or `Collection`?  
A: No, not currently. You can use `indices` and subscripting, but you should not assume ordinary `Array` APIs.

Q: Does inline storage always mean stack storage?  
A: No. It means storage is inline with the containing value. If the containing value is heap allocated, the inline storage is inside that heap object.

Q: What is a good `InlineArray` use case?  
A: AES block `[16 of UInt8]`, SHA-256 block `[64 of UInt8]`, RGBA `[4 of Float]`, or matrix `[16 of Float]`.

Q: What is a bad `InlineArray` use case?  
A: Search results, table rows, network lists, or anything whose count naturally varies.

Q: What should you check before using these in an iOS 18-minimum app?  
A: API availability, fallback design, and whether the types leak into public APIs that must be usable on older OS versions.

Q: What is the staff-level concern with exposing `InlineArray` in a public SDK?  
A: It couples clients to Swift version, standard-library availability, platform availability, and a fixed representation that may become compatibility debt.

---

## 11. Related sections

- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[C11 — Strict memory safety mode and safe systems programming]]
- [[E3 — C interoperability and unsafe boundaries]]
- [[F3 — Performance investigation habits]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist.
- Swift.org Blog — Swift 6.2 Released. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))
- SE-0447 — `Span`: Safe Access to Contiguous Storage. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md?utm_source=chatgpt.com "swift-evolution/proposals/0447-span-access-shared-contiguous-storage.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0452 — Integer Generic Parameters. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0452-integer-generic-parameters.md "swift-evolution/proposals/0452-integer-generic-parameters.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0453 — `InlineArray`, a fixed-size array. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md "swift-evolution/proposals/0453-vector.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0483 — `InlineArray` Type Sugar. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0483-inline-array-sugar.md "swift-evolution/proposals/0483-inline-array-sugar.md at main · swiftlang/swift-evolution · GitHub"))
- Apple Developer Documentation — `Span` and `InlineArray` API references. ([Apple Developer](https://developer.apple.com/documentation/swift/span?utm_source=chatgpt.com "Span | Apple Developer Documentation"))