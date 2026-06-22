---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - safety
  - systems-programming
  - swift6-2
---
## 0. Rubric snapshot

**Rubric expectation**

Understand what Swift 6.2’s opt-in strict memory-safety checking is trying to expose, and why explicit unsafe acknowledgments matter.

**Caveats**

The absence of warnings is not proof that unsafe code is correct. The presence of unsafe APIs should be intentional, localized, documented, and auditable.

**You should be able to answer**

- What kinds of constructs does strict memory safety mode want you to acknowledge?
- Why is making unsafe code explicit valuable even if the code is known-correct?

**You should be able to do**

- Review a low-level wrapper around a C API and identify where explicit unsafe boundaries should exist.

---

## 1. Core mental model

Swift is memory-safe by default in normal code: it checks definite initialization, prevents use-after-free in ordinary references, performs bounds checks, and enforces exclusive access rules. But Swift also has deliberate escape hatches for systems programming: unsafe pointers, raw memory, unsafe unowned references, unchecked exclusivity, C/C++ interop, and low-level standard-library APIs. Strict memory safety mode is about making those escape hatches visible. Swift 6.2 introduced this as an opt-in feature that flags unsafe constructs so they can either be replaced with safer APIs or explicitly acknowledged in source. ([Swift.org, "Memory Safety"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety/))

The important point: strict memory safety mode is **diagnostic/auditing infrastructure**, not a magic proof of correctness. SE-0458 explicitly says the checking does not itself make code more memory-safe; it identifies constructs that are not safe, encourages safe alternatives, and makes unsafe behavior easier to audit. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

The key idea:

```text
Safe Swift by default
+ explicit unsafe boundaries where Swift cannot prove lifetime/bounds/initialization/aliasing
= auditable systems programming
```

This is similar in spirit to `try` and `await`: the keyword is not the mechanism that makes the operation safe. It marks the semantic boundary. With strict memory safety, `unsafe` marks a place where the compiler cannot prove memory safety and the programmer must uphold the contract manually. The proposal describes `unsafe` as an expression marker that acknowledges unsafe constructs without changing the type or requiring the enclosing function to become unsafe. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

For senior/staff-level Swift work, this matters most at module boundaries: C APIs, C++ APIs, custom memory buffers, performance-critical parsers, audio/DSP, graphics, cryptography, compression, embedded work, and binary SDKs. App code should rarely traffic in unsafe pointers directly. A well-designed Swift module pushes unsafety into a narrow adapter layer and exposes ordinary Swift value types, `Span`, `Data`, `Array`, safe handles, or lifetime-scoped closure APIs above it.

---

## 2. Essential mechanics

### 2.1 Strict memory safety is opt-in

Swift 6.2’s strict memory safety checking is not the default for every project. It can be enabled for modules/packages. SwiftPM exposes this through `SwiftSetting.strictMemorySafety(_:)`, first available in PackageDescription 6.2. ([Swift.org, "strictMemorySafety(_:)"](https://docs.swift.org/swiftpm/documentation/packagedescription/swiftsetting/strictmemorysafety%28_%3A%29/))

```swift
// swift-tools-version: 6.2

let package = Package(
    name: "LowLevelCore",
    targets: [
        .target(
            name: "LowLevelCore",
            swiftSettings: [
                .strictMemorySafety()
            ]
        )
    ]
)
```

For stricter CI enforcement, the proposal describes enabling the compiler flag and escalating the `StrictMemorySafety` warning group to errors:

```bash
swiftc -strict-memory-safety -Werror StrictMemorySafety
```

SE-0458 states that strict memory-safety diagnostics are warnings in the `StrictMemorySafety` diagnostic group, so teams can choose whether to treat them as warnings or errors. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

### 2.2 Unsafe types make declarations unsafe

Types like `UnsafePointer`, `UnsafeMutablePointer`, `UnsafeRawPointer`, `UnsafeBufferPointer`, `UnsafeMutableBufferPointer`, `OpaquePointer`, and similar APIs do not carry enough information for Swift to prove lifetime, bounds, initialization, or aliasing safety. SE-0458 describes these standard-library unsafe pointer families as `@unsafe`, and a declaration whose signature includes unsafe types is implicitly unsafe. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

```swift
// Implicitly unsafe because the signature contains UnsafePointer.
func sumBytes(_ pointer: UnsafePointer<UInt8>, count: Int) -> Int {
    var sum = 0

    for i in 0..<count {
        sum += Int(pointer[i])
    }

    return sum
}
```

The danger is not the spelling of `UnsafePointer`. The danger is that the function signature cannot express:

```text
- Is pointer non-nil?
- Is count valid?
- Is the memory initialized?
- Does the buffer live for the whole call?
- Is the memory being mutated elsewhere?
- Is the pointer correctly aligned and bound?
```

### 2.3 `unsafe` acknowledges an unsafe expression

Under strict memory safety, executable use of unsafe constructs must be acknowledged with `unsafe`. SE-0458 compares this marker to `try` and `await`: it marks an effect at the expression level, but it does not propagate outward like throwing or async behavior. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

```swift
func checksum(_ bytes: [UInt8]) -> UInt32 {
    guard !bytes.isEmpty else { return 0 }

    return bytes.withUnsafeBufferPointer { buffer in
        unsafe c_checksum(buffer.baseAddress!, buffer.count)
    }
}
```

The `unsafe` marker means:

```text
At this expression, Swift cannot prove memory safety.
The programmer is asserting that the API contract is satisfied here.
```

It does **not** mean:

```text
This code is now safe.
The compiler verified the pointer lifetime.
Bounds, initialization, aliasing, and alignment are automatically correct.
```

### 2.4 `@unsafe` marks unsafe declarations

Use `@unsafe` when the API itself requires callers to reason about memory safety.

```swift
@unsafe
public func readRegister(at rawAddress: UInt) -> UInt32 {
    let pointer = UnsafePointer<UInt32>(bitPattern: rawAddress)!
    return unsafe pointer.pointee
}
```

This is an unsafe API because callers must know that:

```text
- rawAddress is valid
- memory is mapped
- the type and alignment are correct
- reading has acceptable side effects
- lifetime and ownership are meaningful for that address
```

A safe public API should usually not expose this.

### 2.5 `@safe` marks declarations with unsafe-looking signatures that encapsulate safety

Some functions necessarily mention unsafe types but still manage the safety boundary themselves. The proposal uses `Array.withUnsafeBufferPointer` as the canonical kind of example: the method vends an unsafe buffer pointer to a closure while the array controls the buffer lifetime for the duration of the call. That function can be marked `@safe`, while the closure’s use of the unsafe pointer remains the caller’s responsibility. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

```swift
extension Packet {
    @safe
    public func withBytes<R>(
        _ body: (UnsafeRawBufferPointer) throws -> R
    ) rethrows -> R {
        try storage.withUnsafeBytes { buffer in
            try body(buffer)
        }
    }
}
```

This pattern is safer than returning a pointer:

```swift
// Dangerous: pointer can escape the valid lifetime.
public var bytes: UnsafeRawBufferPointer {
    storage.withUnsafeBytes { $0 }
}
```

### 2.6 Strict memory safety interacts strongly with C/C++ interop

C and C++ APIs often express memory contracts informally: comments, naming, documentation, or convention. Swift cannot infer enough from a raw `const char *`, borrowed view, reference return, or pointer-length pair unless the imported API carries safety/lifetime annotations or is wrapped in Swift.

Swift’s safe C++ interop documentation says strict safety mode flips the default assumption for C++ types: types not known to be safe are treated as unsafe, and diagnostics are emitted for their use without `unsafe`. It also describes lifetime and escapability annotations that let Swift enforce more safety at compile time. ([Swift.org, "Safely Mixing Swift and C/C++"](https://swift.org/documentation/cxx-interop/safe-interop/))

---

## 3. Common traps and misconceptions

### Trap 1: “No warning means safe”

No. Strict memory safety is not a complete verifier. Unsafe code can still be logically wrong after every diagnostic is silenced.

Bad:

```swift
func firstByte(_ bytes: [UInt8]) -> UInt8 {
    bytes.withUnsafeBufferPointer { buffer in
        unsafe buffer.baseAddress!.pointee
    }
}
```

What is wrong?

```text
The unsafe expression is acknowledged, but empty arrays still produce nil baseAddress.
The code can still crash.
```

Better:

```swift
func firstByte(_ bytes: [UInt8]) -> UInt8? {
    bytes.withUnsafeBufferPointer { buffer in
        guard let baseAddress = buffer.baseAddress else {
            return nil
        }

        return unsafe baseAddress.pointee
    }
}
```

Even better for normal app code:

```swift
func firstByte(_ bytes: [UInt8]) -> UInt8? {
    bytes.first
}
```

### Trap 2: Marking everything `unsafe` instead of shrinking the unsafe region

Bad:

```swift
func parsePacket(_ bytes: [UInt8]) -> Packet {
    unsafe parsePacketUnsafe(bytes)
}
```

Better:

```swift
func parsePacket(_ bytes: [UInt8]) throws -> Packet {
    guard bytes.count >= Header.byteCount else {
        throw PacketError.truncatedHeader
    }

    return try bytes.withUnsafeBufferPointer { buffer in
        try unsafe parseHeaderAndPayload(buffer)
    }
}
```

The better version separates safe validation from the unsafe memory operation. That matters during code review.

### Trap 3: Exposing unsafe pointers as public API

Bad:

```swift
public struct AudioBuffer {
    public let samples: UnsafePointer<Float>
    public let count: Int
}
```

This makes every consumer responsible for lifetime and bounds correctness.

Better:

```swift
public struct AudioBuffer {
    private let storage: [Float]

    public var count: Int {
        storage.count
    }

    public subscript(index: Int) -> Float {
        storage[index]
    }

    public func withUnsafeSamples<R>(
        _ body: (UnsafeBufferPointer<Float>) throws -> R
    ) rethrows -> R {
        try storage.withUnsafeBufferPointer(body)
    }
}
```

The wrapper exposes safe operations by default and keeps the unsafe view lifetime-scoped.

### Trap 4: Confusing strict memory safety with strict concurrency

Strict concurrency is about data-race safety, actor isolation, `Sendable`, and cross-isolation transfer. Strict memory safety is about unsafe memory constructs: raw pointers, unsafe references, unchecked exclusivity, unsafe imported APIs, unsafe conformances, and unsafe declarations.

They overlap in some cases. SE-0458 lists `nonisolated(unsafe)` and `@preconcurrency` imports as unsafe in the relevant strict-concurrency context because they can undermine safety guarantees. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

---

## 4. Direct answers to rubric questions

### Q1. What kinds of constructs does strict memory safety mode want you to acknowledge?

Strict memory safety mode wants explicit acknowledgment of constructs that can undermine Swift’s normal memory-safety guarantees.

Important categories:

```text
- Unsafe pointer and buffer pointer types
- Raw memory APIs
- OpaquePointer / CVaListPointer-like low-level pointers
- Unsafe declarations and unsafe signatures
- Unsafe conformances
- Unsafe overrides
- unowned(unsafe)
- unsafeAddressor / unsafeMutableAddressor
- @exclusivity(unchecked)
- nonisolated(unsafe), where strict concurrency applies
- @preconcurrency imports, where they suppress relevant safety diagnostics
- C/C++ imported APIs whose safety/lifetime contracts Swift cannot prove
```

SE-0458 explicitly identifies unsafe language constructs such as `unowned(unsafe)`, `unsafeAddressor`, `unsafeMutableAddressor`, `@exclusivity(unchecked)`, and, in strict-concurrency contexts, `nonisolated(unsafe)` and `@preconcurrency` imports. It also identifies unsafe standard-library pointer families and unsafe C/C++ interop surfaces. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

Interview version:

> Strict memory safety mode is looking for places where Swift’s normal guarantees no longer apply: unsafe pointers, raw memory, unchecked exclusivity, unsafe unowned references, unsafe conformances, unsafe overrides, and imported C/C++ APIs with unverifiable lifetime or bounds contracts. The point is not to ban them. The point is to force those boundaries to be visible and auditable.

### Q2. Why is making unsafe code explicit valuable even if the code is known-correct?

Because “known-correct” is usually local, contextual, and fragile. The correctness depends on invariants that the compiler cannot see: pointer lifetime, initialization, capacity, alignment, aliasing, thread access, ownership transfer, and C API cleanup rules.

Making unsafe code explicit gives you:

```text
- Code review focus
- Searchability
- Auditability
- Better migration planning
- Smaller unsafe regions
- Safer public APIs
- Clear distinction between safe interface and unsafe implementation
- CI enforcement through warning groups
```

SE-0458 states that applying `unsafe`/`@unsafe` can be automated, but the remaining job is for programmers to audit those locations and ensure the unsafe behavior is correctly encapsulated. ([GitHub, "Opt-in Strict Memory Safety Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md))

Interview version:

> Even correct unsafe code should be explicit because its correctness depends on invariants outside the type system. The marker tells future maintainers where to look when changing lifetimes, storage, interop calls, or threading. It also prevents unsafe APIs from silently spreading through a module. The goal is not ceremony; it is containment and auditability.

---

## 5. Code probe

The C11 rubric does not include a code probe. Below are a minimal example, counterexample, and production-style example.

### 5.1 Minimal example

```swift
@unsafe
func loadByte(from pointer: UnsafePointer<UInt8>) -> UInt8 {
    unsafe pointer.pointee
}
```

Expected strict-memory-safety behavior:

```text
The API is unsafe because its signature contains UnsafePointer.
The dereference must be acknowledged with unsafe.
Callers must uphold lifetime, alignment, initialization, and validity.
```

### 5.2 Counterexample: unsafe marker does not fix a bug

```swift
func lastByte(_ bytes: [UInt8]) -> UInt8 {
    bytes.withUnsafeBufferPointer { buffer in
        unsafe buffer.baseAddress![bytes.count - 1]
    }
}
```

This still crashes for an empty array.

Better:

```swift
func lastByte(_ bytes: [UInt8]) -> UInt8? {
    bytes.withUnsafeBufferPointer { buffer in
        guard let baseAddress = buffer.baseAddress, !bytes.isEmpty else {
            return nil
        }

        return unsafe baseAddress[bytes.count - 1]
    }
}
```

Best for normal Swift:

```swift
func lastByte(_ bytes: [UInt8]) -> UInt8? {
    bytes.last
}
```

### 5.3 Production example: C checksum wrapper

Assume a C API:

```c
uint32_t c_checksum(const uint8_t *bytes, size_t count);
```

Naive Swift wrapper:

```swift
public func checksum(pointer: UnsafePointer<UInt8>, count: Int) -> UInt32 {
    unsafe c_checksum(pointer, count)
}
```

Problem:

```text
The public API exposes an unsafe pointer contract to all clients.
Every caller must know the pointer lifetime, count validity, initialization, and nullability rules.
```

Better wrapper:

```swift
public enum Checksum {
    public static func calculate(_ bytes: [UInt8]) -> UInt32 {
        guard !bytes.isEmpty else {
            return 0
        }

        return bytes.withUnsafeBufferPointer { buffer in
            unsafe c_checksum(buffer.baseAddress!, buffer.count)
        }
    }
}
```

Why this is better:

```text
- Public API accepts safe Swift storage.
- Pointer lifetime is scoped to withUnsafeBufferPointer.
- Empty input is handled before force-unwrapping baseAddress.
- Unsafe C call is localized to one expression.
- Callers do not need to reason about pointer validity.
```

---

## 6. Exercise

### Problem

Review a low-level wrapper around a C API and identify where explicit unsafe boundaries should exist.

### Bad / naive version

Assume a C API:

```c
typedef struct Decoder Decoder;

Decoder *decoder_create(void);
void decoder_destroy(Decoder *);
int decoder_decode(Decoder *, const uint8_t *bytes, size_t count);
const char *decoder_last_error(Decoder *);
```

Naive Swift wrapper:

```swift
public final class DecoderWrapper {
    private var decoder: UnsafeMutablePointer<Decoder>?

    public init() {
        decoder = decoder_create()
    }

    deinit {
        decoder_destroy(decoder)
    }

    public func decode(_ pointer: UnsafePointer<UInt8>, count: Int) -> Int {
        decoder_decode(decoder, pointer, count)
    }

    public var lastError: String {
        String(cString: decoder_last_error(decoder))
    }
}
```

### What is wrong?

```text
1. The public decode API exposes UnsafePointer.
2. decoder is optional everywhere, but the wrapper does not model creation failure.
3. deinit may pass nil depending on C API expectations.
4. decode does not validate count.
5. lastError assumes the returned C string is non-null and valid long enough.
6. Pointer lifetime and ownership are undocumented.
7. Thread-safety is unspecified.
8. Unsafe boundaries are spread across the whole type instead of localized.
```

### Improved version

```swift
public enum DecoderError: Error {
    case creationFailed
    case decodeFailed(code: Int, message: String?)
}

public final class DecoderWrapper {
    private let decoder: OpaquePointer

    public init() throws {
        guard let decoder = unsafe decoder_create() else {
            throw DecoderError.creationFailed
        }

        self.decoder = decoder
    }

    deinit {
        unsafe decoder_destroy(decoder)
    }

    public func decode(_ bytes: [UInt8]) throws {
        let code: Int32

        if bytes.isEmpty {
            code = unsafe decoder_decode(decoder, nil, 0)
        } else {
            code = bytes.withUnsafeBufferPointer { buffer in
                unsafe decoder_decode(decoder, buffer.baseAddress!, buffer.count)
            }
        }

        guard code == 0 else {
            throw DecoderError.decodeFailed(
                code: Int(code),
                message: lastErrorMessage()
            )
        }
    }

    private func lastErrorMessage() -> String? {
        guard let pointer = unsafe decoder_last_error(decoder) else {
            return nil
        }

        return unsafe String(cString: pointer)
    }
}
```

### Where the explicit unsafe boundaries should exist

```text
- Creating the C object: decoder_create()
- Destroying the C object: decoder_destroy()
- Passing array storage into decoder_decode()
- Reading the C error string pointer
- Converting the C string into Swift String
```

### Why this is better

```text
- The public API is safe Swift: [UInt8] in, throws out.
- Unsafe pointers do not escape.
- C resource ownership is represented by DecoderWrapper lifetime.
- Creation failure is modeled explicitly.
- Cleanup is centralized in deinit.
- C integer error codes are translated into Swift errors.
- Unsafe calls are searchable and reviewable.
```

### Remaining production questions

```text
- Is Decoder thread-safe?
- Can decode be called concurrently?
- Does decoder_last_error return thread-local, decoder-owned, or borrowed global memory?
- Is the error pointer valid after another decode call?
- Does decoder_decode retain the input pointer after returning?
- Does decoder_destroy accept null?
- Are count and size_t conversions safe on all supported platforms?
```

If `Decoder` is not thread-safe, a production wrapper should either document single-threaded use, make the type actor-isolated, protect it with a lock, or design it as non-Sendable and keep it confined to one isolation domain.

---

## 7. Production guidance

Use strict memory safety in production when:

```text
- Building low-level libraries
- Wrapping C/C++ APIs
- Handling audio, video, graphics, crypto, compression, parsing, or binary formats
- Maintaining SDKs consumed by other teams
- Hardening security-sensitive modules
- Auditing unsafe pointer usage before refactors
- Migrating old C-style Swift wrappers toward safer APIs
```

Be careful when:

```text
- Enabling it on a large legacy module all at once
- Treating every warning as a mechanical annotation task
- Marking broad APIs @unsafe instead of designing safe wrappers
- Assuming unsafe code is correct because diagnostics disappeared
- Exposing unsafe pointer types in public API
- Combining unsafe memory with concurrent access
```

Avoid when:

```text
- You are using it as a substitute for API design
- You are silencing warnings without auditing invariants
- You are adding unsafe low-level abstractions for ordinary app code without a measured need
- A standard-library API already solves the problem safely
```

Debugging checklist:

```text
1. Where exactly does unsafe memory enter the module?
2. Can the unsafe API be replaced by Array, Data, Span, String, or a safe handle?
3. Does the pointer escape a withUnsafe... closure?
4. Who owns allocation and deallocation?
5. Is memory initialized before reads?
6. Is memory bound to the expected type?
7. Are alignment assumptions valid?
8. Are bounds checked before pointer arithmetic?
9. Can another thread mutate the memory?
10. Does the public API expose unsafe types unnecessarily?
11. Are unsafe calls documented with the required preconditions?
12. Can CI escalate StrictMemorySafety warnings to errors for this target?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Strict memory safety mode warns when you use unsafe pointers, and you can add `unsafe` to silence it.

### Senior answer

> Strict memory safety mode is an opt-in audit tool for unsafe memory constructs. It forces unsafe pointer use, unsafe declarations, unsafe conformances, unsafe references, and unsafe interop boundaries to be explicit. I would use it to localize unsafe code and replace public unsafe APIs with safe Swift wrappers.

### Staff-level answer

> Strict memory safety mode is a codebase-design tool. It lets a team define where memory unsafety is allowed, make those boundaries searchable, and prevent unsafe contracts from leaking across module boundaries. I would enable it first in low-level targets, review every unsafe marker, wrap C/C++ APIs behind safe Swift surfaces, and eventually treat the `StrictMemorySafety` warning group as errors in hardened modules. I would not treat the absence of warnings as proof; I would still audit lifetime, bounds, initialization, aliasing, ownership, and concurrency invariants.

Staff-level questions to ask:

```text
Which targets should allow unsafe code at all?
Can unsafe APIs be internal instead of public?
Can public pointer APIs become withUnsafe... scoped closure APIs?
Can Span or InlineArray replace raw pointers or heap arrays?
Should unsafe wrappers be isolated behind a package target?
Should StrictMemorySafety warnings be errors in CI?
Who owns the safety proof for each unsafe marker?
Is the unsafe code covered by tests, fuzzing, sanitizers, or stress tests?
Does the API document lifetime and ownership preconditions?
Can unsafe C/C++ contracts be expressed through annotations?
```

---

## 9. Interview-ready summary

Swift 6.2’s strict memory safety mode is an opt-in checking mode that makes unsafe memory constructs explicit. It flags unsafe pointers, raw memory, unsafe references, unsafe conformances, unsafe declarations, and unsafe C/C++ interop surfaces so the programmer must either replace them with safer APIs or acknowledge them with `unsafe`/`@unsafe`. The important production judgment is that `unsafe` does not make code correct; it marks where Swift cannot prove correctness. Senior engineers localize those regions. Staff engineers design module boundaries so unsafe code is rare, auditable, documented, and hidden behind safe Swift APIs.

---

## 10. Flashcards

Q: What is strict memory safety mode in Swift 6.2?  
A: An opt-in diagnostic mode that identifies uses of unsafe memory constructs and requires explicit acknowledgment or safer alternatives.

Q: Does strict memory safety prove unsafe code is correct?  
A: No. It only exposes places where Swift cannot prove memory safety. Humans still need to audit the invariants.

Q: What does `unsafe` mean at an expression?  
A: It acknowledges that the expression uses unsafe constructs. It does not change the type, lifetime, bounds, or correctness of the operation.

Q: What is the difference between `@unsafe` and `unsafe`?  
A: `@unsafe` marks a declaration as unsafe to use. `unsafe` marks an expression that uses unsafe constructs.

Q: When should an API be `@unsafe`?  
A: When callers must manually uphold memory-safety conditions such as lifetime, bounds, initialization, alignment, or aliasing.

Q: When is `@safe` useful?  
A: When a declaration’s signature mentions unsafe types but the declaration itself safely encapsulates the memory contract, such as a lifetime-scoped `withUnsafe...` API.

Q: Why are unsafe pointer signatures dangerous in public APIs?  
A: They push lifetime, ownership, bounds, and initialization obligations onto every caller.

Q: What is the preferred shape for safe wrappers around C APIs?  
A: Safe Swift input/output at the public boundary, narrow unsafe calls inside, and explicit ownership/cleanup rules.

Q: What is a common misuse of strict memory safety migration?  
A: Mechanically adding `unsafe` everywhere without reducing unsafe surface area or auditing invariants.

Q: How does this differ from strict concurrency?  
A: Strict concurrency checks data-race and isolation safety; strict memory safety checks unsafe memory constructs and interop boundaries.

---

## 11. Related sections

- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[E3 — C interoperability and unsafe boundaries]]
- [[E4 — C++ interoperability]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>)
- Swift.org. "Swift 6.2 Released." Swift.org Blog. https://swift.org/blog/swift-6.2-released/
- GitHub. "Opt-in Strict Memory Safety Checking." Swift Evolution SE-0458. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md
- Swift.org. "strictMemorySafety(_:)." PackageDescription. https://docs.swift.org/swiftpm/documentation/packagedescription/swiftsetting/strictmemorysafety%28_%3A%29/
- Swift.org. "Safely Mixing Swift and C/C++." Swift.org Documentation. https://swift.org/documentation/cxx-interop/safe-interop/
