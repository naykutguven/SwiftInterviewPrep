---
tags:
  - swift
  - ios
  - interview-prep
  - interop
  - c-interop
  - unsafe-pointers
---
## 0. Rubric snapshot

**Rubric expectation**

Understand how Swift interacts with C APIs, pointers, nullability, and imported macros/constants at a practical level. The rubric’s caveat is that C interop often imports unsafety into otherwise safe Swift code. The measured skills are safe wrapper design, quarantining unsafe pointer manipulation, and wrapping C APIs that use output pointers and manual cleanup.

**You should be able to answer**

- What design steps would you take to wrap a C API safely for the rest of a Swift codebase?
- Why should most unsafe pointer manipulation be quarantined to a narrow layer?

**You should be able to do**

- Design a Swifty wrapper for a C API that uses output pointers and manual cleanup.

---

## 1. Core mental model

Swift is safe by default. C is not. When Swift calls C, Swift can preserve some type information, but it cannot magically infer all of C’s hidden contracts: who owns memory, whether a pointer can be null, how many elements a pointer references, whether the pointer escapes, whether a buffer must be freed by a specific function, or whether an integer return code represents an error. Apple’s current guidance frames raw C/C++ pointers as hard to call safely because incorrect use can cause buffer overflows and use-after-free bugs; Swift imports such pointers as unsafe types deliberately. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2025/311/ "Safely mix C, C++, and Swift - WWDC25 - Videos - Apple Developer"))

The key idea:

```text
C boundary = unchecked contracts enter Swift.
Good Swift wrapper = converts unchecked C contracts into checked Swift ownership, errors, lifetimes, and types.
```

Swift interop gives you tools: `UnsafePointer`, `UnsafeMutablePointer`, `UnsafeRawPointer`, `UnsafeMutableRawPointer`, `OpaquePointer`, `withUnsafeBytes`, `withUnsafePointer`, `defer`, `Data`, `Array`, `String`, `Result`, `throws`, and wrapper types. But these are not safety guarantees by themselves. They are vocabulary for expressing C’s memory model in Swift.

A strong wrapper does three things:

1. It **keeps raw pointers private**.
2. It **turns C conventions into Swift conventions**: return codes become `throws`, output pointers become return values, nullable pointers become optionals or thrown errors, ownership becomes `deinit`, and C enums/constants become typed Swift models.
3. It **documents and enforces the C contract** at the boundary.

The staff-level point: do not let C-shaped APIs spread through an app. One unsafe API in a private adapter is manageable. `UnsafeMutablePointer<T>?` leaking across feature modules is architectural debt.

---

## 2. Essential mechanics

### 2.1 C pointer imports

C pointer types map to Swift unsafe pointer types. In Swift’s importer documentation, `const T *` maps to `UnsafePointer<T>`, mutable `T *` maps to `UnsafeMutablePointer<T>`, `void *` maps to raw pointers, and incomplete/opaque C types can be represented as `OpaquePointer`. ([GitHub](https://github.com/swiftlang/swift/blob/master/docs/HowSwiftImportsCAPIs.md "swift/docs/HowSwiftImportsCAPIs.md at main · swiftlang/swift · GitHub"))

```c
// C
void addSecondToFirst(int *x, const long *y);
```

Imported approximately as:

```swift
func addSecondToFirst(
    _ x: UnsafeMutablePointer<CInt>!,
    _ y: UnsafePointer<CLong>!
)
```

The `!` matters. It means the imported pointer is an implicitly unwrapped optional because the C header did not provide enough nullability information.

Better C header:

```c
void addSecondToFirst(int * _Nonnull x, const long * _Nonnull y);
```

Better Swift import:

```swift
func addSecondToFirst(
    _ x: UnsafeMutablePointer<CInt>,
    _ y: UnsafePointer<CLong>
)
```

### 2.2 Nullability annotations shape the Swift API

C pointers can be null, but C itself historically does not encode that in the type system. Swift does. `_Nonnull` imports as a non-optional pointer, `_Nullable` imports as an optional pointer, and unspecified nullability imports as an implicitly unwrapped optional. ([GitHub](https://github.com/swiftlang/swift/blob/master/docs/HowSwiftImportsCAPIs.md "swift/docs/HowSwiftImportsCAPIs.md at main · swiftlang/swift · GitHub"))

Bad C header:

```c
int decoder_create(const uint8_t *bytes, size_t count, Decoder **outDecoder);
```

Swift sees too many IUOs:

```swift
func decoder_create(
    _ bytes: UnsafePointer<UInt8>!,
    _ count: Int,
    _ outDecoder: UnsafeMutablePointer<OpaquePointer?>!
) -> CInt
```

Better C header:

```c
int decoder_create(
    const uint8_t * _Nonnull bytes,
    size_t count,
    Decoder * _Nullable * _Nonnull outDecoder
);
```

Now Swift can distinguish:

```swift
// input bytes: required
// out pointer: required
// produced decoder: nullable until success is checked
```

In production, fix nullability at the C header level when you own it. If you do not own it, fix it in the Swift wrapper.

### 2.3 Temporary pointer lifetimes

`withUnsafeBytes`, `withUnsafeBufferPointer`, and `withUnsafePointer` provide temporary access. The pointer must not escape the closure unless the C API explicitly copies the data synchronously.

Good:

```swift
let status = data.withUnsafeBytes { rawBuffer in
    let bytes = rawBuffer.bindMemory(to: UInt8.self)

    return c_process_bytes(bytes.baseAddress, bytes.count)
}
```

Bad:

```swift
var saved: UnsafePointer<UInt8>?

data.withUnsafeBytes { rawBuffer in
    saved = rawBuffer.bindMemory(to: UInt8.self).baseAddress
}

// Undefined behavior if used later.
// The pointer was only valid during the closure.
print(saved!.pointee)
```

Apple’s unsafe-pointer guidance emphasizes that generated pointer values are temporary and invalidated when the function returns; closure-based unsafe APIs make that lifetime explicit. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10648/?time=536 "Unsafe Swift - WWDC20 - Videos - Apple Developer"))

### 2.4 Output pointers become Swift return values

C commonly uses output pointers:

```c
int get_size(Image *image, int *width, int *height);
```

Raw Swift import:

```swift
var width: CInt = 0
var height: CInt = 0

let status = get_size(imagePointer, &width, &height)
```

Swifty wrapper:

```swift
struct ImageSize: Equatable {
    let width: Int
    let height: Int
}

func size() throws -> ImageSize {
    var width: CInt = 0
    var height: CInt = 0

    let status = c_image_get_size(pointer, &width, &height)
    guard status == C_IMAGE_OK else {
        throw CImageError(status)
    }

    return ImageSize(width: Int(width), height: Int(height))
}
```

The rest of the app should not know this was implemented with output pointers.

### 2.5 Manual cleanup becomes RAII-style Swift ownership

C APIs often look like this:

```c
Decoder *decoder_create(...);
void decoder_destroy(Decoder *);
```

or:

```c
int decoder_create(..., Decoder **outDecoder);
void decoder_destroy(Decoder *);
```

In Swift, use a wrapper whose `deinit` calls the cleanup function:

```swift
public final class Decoder {
    private let pointer: OpaquePointer

    private init(pointer: OpaquePointer) {
        self.pointer = pointer
    }

    deinit {
        c_decoder_destroy(pointer)
    }
}
```

This turns “remember to call cleanup” into normal Swift object lifetime. It is not magic: you still need to know whether the pointer is owned by the caller, borrowed, retained, or returned autoreleased-like. Apple’s unsafe-pointer material stresses that pointers do not manage memory for you; you must know whether the C function takes ownership, stores the pointer, or merely uses it for the duration of the call. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10648/?time=536 "Unsafe Swift - WWDC20 - Videos - Apple Developer"))

### 2.6 C macros and constants

Swift imports simple constant-like C macros as global constants, but C macros are generally not imported as executable Swift macros. The Swift repository documentation says C macros are generally not imported, while constant-defining macros are imported as readonly variables; Apple’s C macro documentation similarly describes simple `#define` constants as imported constants. ([GitHub](https://github.com/swiftlang/swift/blob/master/docs/HowSwiftImportsCAPIs.md "swift/docs/HowSwiftImportsCAPIs.md at main · swiftlang/swift · GitHub"))

C:

```c
#define IMAGE_MAX_WIDTH 4096
#define LIB_VERSION "1.2.0"
#define IMAGE_MIN(a, b) ((a) < (b) ? (a) : (b))
```

Swift import:

```swift
// likely imported
var IMAGE_MAX_WIDTH: CInt { get }
var LIB_VERSION: String { get }

// not imported as a normal Swift function
// IMAGE_MIN(...)
```

Wrap constants into a namespace:

```swift
public enum ImageLibrary {
    public static let maximumWidth = Int(IMAGE_MAX_WIDTH)
    public static let version = LIB_VERSION
}
```

Avoid letting `SCREAMING_C_MACRO_NAMES` leak through your Swift API.

### 2.7 Function pointers and callbacks

C function pointers import as `@convention(c)` function values. They cannot capture Swift context like normal Swift closures. The Swift importer documentation notes that Swift closures have different layout because they include captured context, while C function pointers do not; C function pointers are represented as `@convention(c)`. ([GitHub](https://github.com/swiftlang/swift/blob/master/docs/HowSwiftImportsCAPIs.md "swift/docs/HowSwiftImportsCAPIs.md at main · swiftlang/swift · GitHub"))

C:

```c
typedef void (*LogCallback)(const char *message);
void set_logger(LogCallback callback);
```

Swift:

```swift
let callback: @convention(c) (UnsafePointer<CChar>?) -> Void = { message in
    guard let message else { return }
    print(String(cString: message))
}

set_logger(callback)
```

If you need captured state, C APIs usually require a separate `void *context` parameter. The Swift wrapper must retain/release that context safely, often with `Unmanaged`.

---

## 3. Common traps and misconceptions

### Trap 1: Treating `UnsafePointer` as “just another reference”

Bad:

```swift
final class ImageProcessor {
    private var bytes: UnsafePointer<UInt8>?
    
    func configure(data: Data) {
        data.withUnsafeBytes { buffer in
            bytes = buffer.bindMemory(to: UInt8.self).baseAddress
        }
    }
}
```

This stores a pointer whose lifetime ended when the closure returned.

Better:

```swift
final class ImageProcessor {
    private var data: Data?

    func configure(data: Data) {
        self.data = data
    }

    func process() throws {
        guard let data else { return }

        try data.withUnsafeBytes { buffer in
            let bytes = buffer.bindMemory(to: UInt8.self)
            guard let base = bytes.baseAddress else { return }
            try check(c_process(base, bytes.count))
        }
    }
}
```

Store the owning Swift value, not the borrowed pointer.

### Trap 2: Assuming C return codes are obvious

Bad:

```swift
c_decoder_create(bytes, count, &decoder)
return Decoder(pointer: decoder!)
```

This ignores:

```text
What does nonzero mean?
Can outDecoder be nil on failure?
Can outDecoder be nonnil on partial failure?
Who frees it?
```

Better:

```swift
let status = c_decoder_create(bytes, count, &decoder)

guard status == C_DECODER_OK, let decoder else {
    if let decoder {
        c_decoder_destroy(decoder)
    }
    throw DecoderError(status)
}

return Decoder(pointer: decoder)
```

### Trap 3: Leaking C-shaped APIs upward

Bad public API:

```swift
public func decode(
    _ bytes: UnsafePointer<UInt8>?,
    _ count: Int,
    _ outImage: UnsafeMutablePointer<OpaquePointer?>?
) -> CInt
```

Better public API:

```swift
public func decode(_ data: Data) throws -> DecodedImage
```

The former forces every caller to understand C memory rules. The latter localizes the unsafe knowledge.

### Trap 4: Trusting unspecified nullability

Imported `UnsafePointer<T>!` is not a guarantee. It means the importer did not know whether null is meaningful. Treat it as an API smell at the boundary.

### Trap 5: Forgetting C integer widths

C uses `int`, `size_t`, `long`, `uint32_t`, etc. Swift `Int` is convenient, but do not blindly convert. Validate ranges when converting from Swift values to narrower C types:

```swift
guard count <= Int(CInt.max) else {
    throw DecoderError.inputTooLarge
}

let cCount = CInt(count)
```

### Trap 6: Making unsafe code “look safe” without proving it

A wrapper is not safe because it hides pointers. It is safe only if it enforces:

```text
lifetime
ownership
bounds
nullability
thread-safety
error handling
cleanup
```

Swift 6.2’s strict memory safety mode can help identify unsafe constructs, especially around C/C++ pointers, but it is opt-in and diagnostics are not a proof that every manual contract is correct. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2025/311/ "Safely mix C, C++, and Swift - WWDC25 - Videos - Apple Developer"))

---

## 4. Direct answers to rubric questions

### Q1. What design steps would you take to wrap a C API safely for the rest of a Swift codebase?

Start by reading the C contract, not by translating signatures mechanically. Identify ownership, nullability, pointer lifetime, buffer bounds, error conventions, cleanup rules, threading guarantees, and whether callbacks escape. Then expose a small Swift API that uses Swift-native types: `Data`, `Array`, `String`, typed structs/enums, `throws`, and a wrapper object whose `deinit` performs cleanup.

Concrete steps:

```text
1. Audit the C header.
2. Fix C annotations if possible: nullability, bounds, noescape/lifetime annotations where available.
3. Hide imported C declarations in a private/internal adapter target.
4. Convert return codes to Swift errors.
5. Convert output pointers to return values.
6. Convert manual cleanup to deinit.
7. Convert C constants/macros into typed Swift constants/enums.
8. Keep unsafe pointer lifetimes inside closure scopes.
9. Add tests for success, failure, nil output, cleanup, invalid input, and large input.
10. Document the remaining unsafe assumptions.
```

Interview version:

> I would not expose the imported C API directly. I would first audit the C contract: ownership, nullability, buffer sizes, escaping, cleanup, and error codes. Then I would build a small Swift wrapper that turns output pointers into return values, return codes into `throws`, manual cleanup into `deinit`, and raw buffers into `Data` or typed collections. The rest of the app should see a Swift API, not `UnsafeMutablePointer` and integer status codes.

### Q2. Why should most unsafe pointer manipulation be quarantined to a narrow layer?

Because unsafe pointer code has unchecked failure modes. The compiler cannot fully protect you from dangling pointers, use-after-free, invalid binding, wrong capacity, wrong ownership, double free, or out-of-bounds access. Apple’s unsafe Swift guidance is explicit that unsafe pointers provide low-level control but trust you to use them correctly; incorrect use can lead to undefined behavior and memory corruption. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10648/?time=536 "Unsafe Swift - WWDC20 - Videos - Apple Developer"))

Quarantining unsafe code gives you:

```text
smaller audit surface
fewer callers that need C expertise
clear ownership rules
centralized tests
easier migration if the C API changes
easier replacement with pure Swift later
```

Interview version:

> Unsafe pointer manipulation should be treated like a hazardous boundary. It is sometimes necessary for C interop or performance, but it should be isolated. A narrow wrapper lets the team audit one place for lifetime, ownership, bounds, and cleanup correctness. If unsafe types spread across the app, every caller becomes responsible for C-level invariants, and that is how memory bugs become architecture bugs.

---

## 5. Code probe

The rubric does not include a code probe for E3. Use these focused examples instead.

### Minimal example: output pointer

C-style imported function:

```swift
// Pretend this is imported from C:
//
// int c_image_get_size(OpaquePointer *image, int *width, int *height)
func c_image_get_size(
    _ image: OpaquePointer,
    _ width: UnsafeMutablePointer<CInt>,
    _ height: UnsafeMutablePointer<CInt>
) -> CInt {
    width.pointee = 640
    height.pointee = 480
    return 0
}
```

Swift wrapper:

```swift
struct ImageSize: Equatable {
    let width: Int
    let height: Int
}

enum ImageError: Error {
    case cFailure(CInt)
}

final class ImageHandle {
    private let pointer: OpaquePointer

    init(pointer: OpaquePointer) {
        self.pointer = pointer
    }

    func size() throws -> ImageSize {
        var width: CInt = 0
        var height: CInt = 0

        let status = c_image_get_size(pointer, &width, &height)

        guard status == 0 else {
            throw ImageError.cFailure(status)
        }

        return ImageSize(width: Int(width), height: Int(height))
    }
}
```

### What happens?

```text
No output. The wrapper compiles conceptually and turns C output parameters into a Swift return value.
```

### Why?

```text
C API:
caller allocates output variables
callee writes through pointers
status code indicates success/failure

Swift wrapper:
local variables own temporary output storage
unsafe pointer exposure lasts only for the C call
validated status controls whether values are returned
callers receive a typed Swift value
```

### Counterexample: escaping temporary pointer

```swift
var escaped: UnsafePointer<UInt8>?

let data = Data([1, 2, 3])

data.withUnsafeBytes { rawBuffer in
    escaped = rawBuffer.bindMemory(to: UInt8.self).baseAddress
}

print(escaped!.pointee)
```

### What happens?

```text
Undefined behavior.
```

### Why?

The pointer produced by `withUnsafeBytes` is only valid during the closure. It does not own the `Data` storage, and it must not be saved for later.

### Production example: wrapper around manual cleanup

```swift
import Foundation

// MARK: - Pretend C declarations

// C:
//
// typedef struct decoder decoder_t;
//
// typedef enum decoder_status_t {
//   DECODER_OK = 0,
//   DECODER_INVALID_DATA = 1,
//   DECODER_OUT_OF_MEMORY = 2
// } decoder_status_t;
//
// decoder_status_t decoder_create(
//   const uint8_t *bytes,
//   size_t count,
//   decoder_t **out_decoder
// );
//
// void decoder_destroy(decoder_t *decoder);
//
// decoder_status_t decoder_decode_png(
//   decoder_t *decoder,
//   uint8_t **out_bytes,
//   size_t *out_count
// );
//
// void decoder_free(void *ptr);

typealias decoder_t = OpaquePointer

let DECODER_OK: CInt = 0
let DECODER_INVALID_DATA: CInt = 1
let DECODER_OUT_OF_MEMORY: CInt = 2

func decoder_create(
    _ bytes: UnsafePointer<UInt8>,
    _ count: Int,
    _ outDecoder: UnsafeMutablePointer<decoder_t?>
) -> CInt {
    // Placeholder for imported C implementation.
    outDecoder.pointee = OpaquePointer(bitPattern: 0x1)
    return DECODER_OK
}

func decoder_destroy(_ decoder: decoder_t) {
    // Placeholder for imported C cleanup.
}

func decoder_decode_png(
    _ decoder: decoder_t,
    _ outBytes: UnsafeMutablePointer<UnsafeMutablePointer<UInt8>?>,
    _ outCount: UnsafeMutablePointer<Int>
) -> CInt {
    // Placeholder for imported C implementation.
    let count = 3
    let pointer = UnsafeMutablePointer<UInt8>.allocate(capacity: count)
    pointer.initialize(from: [0x89, 0x50, 0x4E], count: count)

    outBytes.pointee = pointer
    outCount.pointee = count

    return DECODER_OK
}

func decoder_free(_ pointer: UnsafeMutableRawPointer?) {
    pointer?.deallocate()
}

// MARK: - Swifty wrapper

public enum DecoderError: Error, Equatable {
    case invalidData
    case outOfMemory
    case missingOutput
    case unknownStatus(CInt)

    init(status: CInt) {
        switch status {
        case DECODER_INVALID_DATA:
            self = .invalidData
        case DECODER_OUT_OF_MEMORY:
            self = .outOfMemory
        default:
            self = .unknownStatus(status)
        }
    }
}

public final class ImageDecoder {
    private let pointer: decoder_t

    private init(pointer: decoder_t) {
        self.pointer = pointer
    }

    deinit {
        decoder_destroy(pointer)
    }

    public static func make(data: Data) throws -> ImageDecoder {
        try data.withUnsafeBytes { rawBuffer in
            let bytes = rawBuffer.bindMemory(to: UInt8.self)

            guard let baseAddress = bytes.baseAddress else {
                throw DecoderError.invalidData
            }

            var rawDecoder: decoder_t?

            let status = decoder_create(
                baseAddress,
                bytes.count,
                &rawDecoder
            )

            guard status == DECODER_OK else {
                if let rawDecoder {
                    decoder_destroy(rawDecoder)
                }
                throw DecoderError(status: status)
            }

            guard let rawDecoder else {
                throw DecoderError.missingOutput
            }

            return ImageDecoder(pointer: rawDecoder)
        }
    }

    public func decodePNGData() throws -> Data {
        var outputBytes: UnsafeMutablePointer<UInt8>?
        var outputCount: Int = 0

        let status = decoder_decode_png(
            pointer,
            &outputBytes,
            &outputCount
        )

        guard status == DECODER_OK else {
            if let outputBytes {
                decoder_free(UnsafeMutableRawPointer(outputBytes))
            }
            throw DecoderError(status: status)
        }

        guard let outputBytes else {
            throw DecoderError.missingOutput
        }

        defer {
            decoder_free(UnsafeMutableRawPointer(outputBytes))
        }

        return Data(bytes: outputBytes, count: outputCount)
    }
}
```

### Why this wrapper is correct

The wrapper makes the C contract explicit:

```text
decoder_create:
- input Data is borrowed only during withUnsafeBytes
- C must copy input bytes if it needs them later
- output pointer is checked
- failure cleans up partial output if needed
- success becomes ImageDecoder

decoder_destroy:
- called exactly once from deinit

decoder_decode_png:
- C-owned output buffer is copied into Swift Data
- defer guarantees decoder_free
- raw pointer does not escape
```

### Alternative fixes and tradeoffs

|Option|When appropriate|Tradeoff|
|---|---|---|
|`final class` wrapper with `deinit`|C object has identity and manual cleanup|ARC controls lifetime, but thread-safety still needs design|
|`struct` containing `Data`|C output can be copied into Swift-owned memory|Copy cost, but safest API|
|Keep `UnsafePointer` public|Very low-level performance API or systems package|Forces every caller to uphold C invariants|
|Add C header annotations|You own the C API|Best long-term safety, but requires C-side changes|
|Use Swift 6.2 strict memory safety|Security-sensitive or low-level modules|Finds unsafe constructs, but does not replace wrapper design|

---

## 6. Exercise

### Problem

Design a Swifty wrapper for a C API that uses output pointers and manual cleanup.

### Bad / naive version

```swift
public func decodeImage(_ data: Data) -> OpaquePointer? {
    var decoder: OpaquePointer?

    data.withUnsafeBytes { rawBuffer in
        let bytes = rawBuffer.bindMemory(to: UInt8.self)
        decoder_create(bytes.baseAddress!, bytes.count, &decoder)
    }

    return decoder
}
```

### What is wrong?

```text
- Force unwraps baseAddress.
- Ignores empty data.
- Ignores the C status code.
- Returns a raw C pointer to the app.
- Does not document or enforce who destroys the decoder.
- Allows leaks because callers may forget decoder_destroy.
- Does not protect against partial initialization.
- Does not convert C errors into Swift errors.
```

### Improved version

```swift
public final class SafeImageDecoder {
    private let decoder: OpaquePointer

    private init(decoder: OpaquePointer) {
        self.decoder = decoder
    }

    deinit {
        decoder_destroy(decoder)
    }

    public static func create(from data: Data) throws -> SafeImageDecoder {
        guard !data.isEmpty else {
            throw DecoderError.invalidData
        }

        return try data.withUnsafeBytes { rawBuffer in
            let buffer = rawBuffer.bindMemory(to: UInt8.self)

            guard let baseAddress = buffer.baseAddress else {
                throw DecoderError.invalidData
            }

            var output: OpaquePointer?

            let status = decoder_create(
                baseAddress,
                buffer.count,
                &output
            )

            guard status == DECODER_OK else {
                if let output {
                    decoder_destroy(output)
                }
                throw DecoderError(status: status)
            }

            guard let output else {
                throw DecoderError.missingOutput
            }

            return SafeImageDecoder(decoder: output)
        }
    }
}
```

### Why this is better

The improved wrapper changes the API from this:

```text
"Here is a nullable pointer. Good luck."
```

to this:

```text
"Give me Data. I either return a valid decoder or throw. Cleanup is automatic."
```

That is exactly what a Swifty wrapper should do.

---

## 7. Production guidance

Use C interop in production when:

```text
- You depend on a mature C library.
- You need OS/POSIX/Darwin APIs.
- You need performance-critical native code already implemented in C.
- You are gradually migrating a legacy Objective-C/C codebase.
- A C ABI boundary is required for distribution or plugin compatibility.
```

Be careful when:

```text
- The C API stores pointers passed by Swift.
- The C API returns borrowed memory.
- The C API requires caller-allocated buffers.
- The C API uses callbacks with void * context.
- The C API uses global mutable state.
- The C API is not thread-safe.
- Nullability annotations are missing.
- Cleanup differs depending on success/failure.
```

Avoid when:

```text
- A safe Swift/Foundation API already exists.
- You are using C only to avoid learning Swift-native APIs.
- Unsafe pointers leak into feature-layer code.
- Ownership rules are undocumented.
- You cannot write tests for failure and cleanup behavior.
```

Debugging checklist:

```text
Ownership:
- Who owns this pointer?
- Who frees it?
- Which function frees it?
- Is cleanup required on failure?

Lifetime:
- Is the pointer valid after the call returns?
- Does C copy the bytes or store the pointer?
- Did a withUnsafe... pointer escape?

Bounds:
- Where is the count stored?
- Is count in bytes or elements?
- Can count overflow when converting Int <-> CInt/size_t?

Nullability:
- Can input be null?
- Can output be null on success?
- Does the C header encode this?

Errors:
- What does each status code mean?
- Can partial output be produced on failure?

Concurrency:
- Is the C object thread-safe?
- Is there global mutable state?
- Should the Swift wrapper be non-Sendable, actor-isolated, or locked?

Diagnostics:
- Enable Address Sanitizer for pointer bugs.
- Enable Thread Sanitizer for thread-safety issues.
- Consider Swift 6.2 strict memory safety for unsafe-boundary auditing.
```

Apple recommends keeping unsafe API usage to a minimum, choosing safer alternatives where available, using buffer pointers for memory regions, and using tools such as Address Sanitizer to catch unsafe-memory bugs. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10648/?time=536 "Unsafe Swift - WWDC20 - Videos - Apple Developer"))

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift can call C functions using unsafe pointers. You use `withUnsafePointer` or `withUnsafeBytes`, pass pointers to C, and call cleanup functions manually.

### Senior answer

> C interop imports unsafe contracts into Swift. I would wrap the C API so callers use `Data`, `String`, Swift structs, and `throws`. I would hide pointers, validate status codes, use `defer` for cleanup, and use `deinit` for owned C handles.

### Staff-level answer

> I would treat the C boundary as an architecture boundary. The imported module should be isolated behind an adapter or package target. I would fix nullability and bounds annotations in the C headers where possible, design a Swift overlay that expresses ownership and errors natively, prevent raw pointers from leaking into public API, and add tests and sanitizers around failure paths. For security-sensitive modules, I would also consider Swift 6.2 strict memory safety and C/C++ bounds-safety annotations, because the goal is not just to make the call compile; it is to make the unsafe assumptions explicit and auditable. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2025/311/ "Safely mix C, C++, and Swift - WWDC25 - Videos - Apple Developer"))

Staff-level questions to ask:

```text
Who owns each pointer returned by the C API?
Can the C function store a pointer passed from Swift?
Are pointer counts bytes, elements, or capacity?
Can the C API produce partial output on failure?
Can we annotate the C header with nullability and bounds?
Should this wrapper be Sendable, actor-isolated, locked, or explicitly non-thread-safe?
Should this C dependency leak into public API or stay implementation-only?
How will we test cleanup, failure, and invalid input?
Can we replace parts of this C API with Swift-native code over time?
```

---

## 9. Interview-ready summary

Swift can interoperate with C, but C brings unchecked contracts into Swift: pointer lifetime, ownership, nullability, bounds, cleanup, and error conventions. I would keep unsafe pointer manipulation inside a narrow wrapper layer. That wrapper should convert C output pointers into Swift return values, return codes into `throws`, manual cleanup into `deinit` or `defer`, C constants into typed Swift names, and raw buffers into `Data`, `Array`, or carefully scoped unsafe buffers. The rest of the app should not see `UnsafeMutablePointer` unless it is genuinely a low-level API. Staff-level judgment is designing the boundary so Swift’s type system works for the codebase instead of letting C’s unsafety spread through it.

---

## 10. Flashcards

Q: Why is C interop unsafe in Swift?  
A: Because C APIs often encode ownership, lifetime, nullability, bounds, and error contracts only in documentation or convention. Swift can import the function, but it cannot infer every safety rule.

Q: What should a Swifty C wrapper do with output pointers?  
A: Hide them. Use local variables internally, call the C function, validate the status code, then return a Swift value.

Q: What should a Swifty C wrapper do with manual cleanup?  
A: Use `defer` for temporary buffers and `deinit` for owned C handles.

Q: Why is `withUnsafeBytes` safer than storing an `UnsafePointer`?  
A: It scopes the pointer lifetime to a closure, making it harder to accidentally use the pointer after the underlying storage is invalid.

Q: What does unspecified C pointer nullability usually import as?  
A: An implicitly unwrapped optional unsafe pointer, such as `UnsafePointer<T>!`.

Q: Why is `UnsafePointer<T>!` a smell at an API boundary?  
A: It means Swift does not know whether nil is valid. The wrapper should make that explicit as non-optional, optional, or throwing behavior.

Q: When should raw pointers appear in public Swift API?  
A: Rarely; mainly in low-level libraries where callers are expected to understand memory contracts.

Q: What is `OpaquePointer` for?  
A: C pointers to incomplete or opaque types whose pointee layout Swift does not know.

Q: Can C function pointers capture Swift state?  
A: Not like normal Swift closures. C function pointers import as `@convention(c)` and have no captured context. Use a separate context pointer if the C API supports it.

Q: What is the staff-level concern with C interop?  
A: Not just making the call compile, but controlling unsafe surface area, preserving module boundaries, testing failure/cleanup paths, and making hidden contracts explicit.

---

## 11. Related sections

- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C11 — Strict memory safety mode and safe systems programming]]
- [[C12 — Span, InlineArray, and modern low-level abstractions]]
- [[E1 — Objective-C interoperability]]
- [[E2 — Foundation and Objective-C bridging behavior]]
- [[E4 — C++ interoperability]]
- [[E6 — Import visibility and dependency leakage]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- Rubric section E3: C interoperability and unsafe boundaries.
- Swift importer behavior for C functions, pointers, nullability, opaque pointers, function pointers, structs, enums, globals, and macros. ([GitHub](https://github.com/swiftlang/swift/blob/master/docs/HowSwiftImportsCAPIs.md "swift/docs/HowSwiftImportsCAPIs.md at main · swiftlang/swift · GitHub"))
- Apple Developer Documentation: C interoperability and imported C macros/functions. ([Apple Developer](https://developer.apple.com/documentation/swift/c-interoperability?utm_source=chatgpt.com "C Interoperability | Apple Developer Documentation"))
- Apple WWDC20: Unsafe Swift / safely managing unsafe pointer usage and keeping unsafe operations minimal. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10648/?time=536 "Unsafe Swift - WWDC20 - Videos - Apple Developer"))
- Apple WWDC25: Safely mix C, C++, and Swift; strict memory safety, pointer bounds/lifetime annotations, and Swift 6.2 safety direction. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2025/311/ "Safely mix C, C++, and Swift - WWDC25 - Videos - Apple Developer"))
- Swift.org: Safely Mixing Swift and C/C++, including strict safety mode, lifetime annotations, bounds annotations, and Span-related safe imports. ([Swift.org](https://swift.org/documentation/cxx-interop/safe-interop/ "Safely Mixing Swift and C/C++ | Swift.org"))