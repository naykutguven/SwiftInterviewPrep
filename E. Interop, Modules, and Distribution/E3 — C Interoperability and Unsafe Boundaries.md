---
tags:
  - swift
  - ios
  - interview-prep
  - interop
  - c-interop
  - unsafe-pointers
---

# E3 — C Interoperability and Unsafe Boundaries

**Learning goals:**

- Translate native counts, initialization, nullability, ownership, errors, and callback lifetime into a small Swift API.
- Verify failure and zero-length cleanup against a real C implementation rather than relying on annotations.
- Distinguish generated safe overloads, raw C ABI implementations, exported headers, and confinement.

**Prerequisites:** [[C6 — Unsafe pointers, buffers, and raw memory]] for pointer storage states, [[C11 — Strict Memory Safety Mode and Safe Systems Programming]] for strict memory safety, [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]] for Span, and [[A13 — Error handling and typed throws]] for errors. [[C1 — ARC fundamentals]] and [[D9 — Continuations and Bridging Callback APIs]] explain retained callback context and asynchronous registration.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A C function receives an address and a count. Its type does not tell Swift whether the bytes are initialized, whether the function saves the address, or who frees a returned allocation after failure. Start with the native contract, then use scoped borrowed input and owned Swift output to make those obligations local.

**Ownership** tells you who must release a resource; **nullability** tells you whether a pointer may be null. Neither tells you allocation size. The **ABI** is the binary calling contract, including the pointer/count parameters and result representation. Bounds and lifetime annotations can generate safer Swift overloads, but they remain promises about the implementation. [C interoperability](https://www.swift.org/documentation/c-interop/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Begin with a synchronous counted byte operation

Shared E3Safe fixture. Save the header, module map, and C source in one temporary directory. The implementation reads exactly count initialized bytes synchronously, retains no pointer, and returns the sum modulo 2^32. Null with zero is valid; null with positive count is not.

```c
#pragma once
#include <stddef.h>
#include <stdint.h>
#include <ptrcheck.h>
#include <lifetimebound.h>
uint32_t e3_sum(const uint8_t * _Nullable __counted_by(count) bytes __noescape, size_t count);
uint32_t e3_swift_sum(const uint8_t * _Nullable __counted_by(count) bytes __noescape, size_t count);
```

```text
module E3Safe { header "E3Safe.h" export * }
```

```c
#include "E3Safe.h"
uint32_t e3_sum(const uint8_t *bytes, size_t count) {
    uint32_t result = 0;
    for (size_t i = 0; i < count; ++i) result += bytes[i];
    return result;
}
```

Build with the selected Xcode SDK/target: xcrun --sdk macosx clang -std=c11 -Wall -Wextra -Werror -c E3Safe.c -o /tmp/E3Safe.o. Swift clients add -I with the header directory and link that object. Validation records the explicit matching SDK/target. No experimental safe-wrapper flag is used.

The counted/noescape declaration generates a Span overload. Shared safeSum setup; Array.span separately needs Apple OS 26.

```swift
import E3Safe

func safeSum(_ bytes: Span<UInt8>) -> UInt32 { e3_sum(bytes) }
@available(anyAppleOS 26.0, *)
func sumProbe() {
    let bytes: [UInt8] = [0, 1, 255]
    precondition(safeSum(bytes.span) == 256)
    let empty: [UInt8] = []
    precondition(safeSum(empty.span) == 0)
}
if #available(anyAppleOS 26.0, *) { sumProbe() }
```

**Walkthrough:** Swift callers pass an initialized borrowed view rather than inventing a pointer/count pair. The C call still has its original ABI and must honor its annotations. The empty check exercises the documented null/zero behavior without reading storage. [Safe interoperability](https://www.swift.org/documentation/cxx-interop/safe-interop/)

### A complete owned-handle fixture

Shared E3Native setup. The native decoder copies input during creation and returns inverted owned output bytes. Every nonnull handle/output belongs to the caller, even on failure or with logical count zero. Only the matching destroy/free function releases it. Input and callbacks are synchronous/nonretaining. Fixture counters and failure modes are sequential instrumentation, not synchronized library state.

Save the following header, module map, and implementation together and build E3Native.c with the same command as E3Safe.c. Swift clients import E3Native, add its directory with -I, and link E3Native.o.

```c
#pragma once
#include <stddef.h>
#include <stdint.h>
#define E3_MAX_INPUT 4096
#define E3_VERSION "1.0.0"
#define E3_MIN(a, b) ((a) < (b) ? (a) : (b))
enum { E3_OK = 0, E3_INVALID_INPUT = 1, E3_OUT_OF_MEMORY = 2, E3_DECODE_FAILED = 3 };
typedef struct E3Decoder E3Decoder;
// Copies input synchronously. Every nonnull output handle, even on failure,
// belongs to the caller and must be destroyed with e3_decoder_destroy.
int e3_decoder_create(const uint8_t * _Nullable bytes, size_t count,
                      E3Decoder * _Nullable * _Nonnull output);
void e3_decoder_destroy(E3Decoder * _Nullable decoder);
// Every nonnull output allocation, even on failure or zero count, must be
// released with e3_bytes_free. Nil plus zero is valid empty success.
int e3_decoder_copy_bytes(E3Decoder * _Nonnull decoder,
                         uint8_t * _Nullable * _Nonnull output,
                         size_t * _Nonnull count);
void e3_bytes_free(uint8_t * _Nullable bytes);
typedef void (*E3Visitor)(uint8_t value, void * _Nonnull context);
// Synchronous serial calls; stores neither input, callback, nor context.
void e3_visit(const uint8_t * _Nullable bytes, size_t count,
              E3Visitor _Nonnull visitor, void * _Nonnull context);
// Fixture controls/counters: use only sequentially, reset with no live objects.
void e3_fixture_reset(int mode);
int e3_live_handles(void);
int e3_live_buffers(void);
int e3_destroyed_handles(void);
int e3_freed_buffers(void);
```

```text
module E3Native { header "E3Native.h" export * }
```

The implementation is included so cleanup claims can be checked. Modes 0–10 inject normal operation, creation failure, partial handle failure, missing handle, output failure, nil/positive count, nil/zero, allocated/zero, over-limit count, SIZE_MAX count, and an unknown status. Modes 8/9 deliberately advertise more than allocated and must be rejected before reading.

```c
#include "E3Native.h"
#include <assert.h>
#include <stdlib.h>
#include <string.h>
struct E3Decoder { uint8_t bytes[E3_MAX_INPUT]; size_t count; int mode; };
static int fixture_mode, handles, buffers, destroyed, freed;
void e3_fixture_reset(int mode) {
    assert(handles == 0 && buffers == 0);
    fixture_mode = mode; destroyed = 0; freed = 0;
}
int e3_live_handles(void) { return handles; }
int e3_live_buffers(void) { return buffers; }
int e3_destroyed_handles(void) { return destroyed; }
int e3_freed_buffers(void) { return freed; }
int e3_decoder_create(const uint8_t *bytes, size_t count, E3Decoder **output) {
    *output = NULL;
    if (count > E3_MAX_INPUT || (count > 0 && bytes == NULL)) return E3_INVALID_INPUT;
    if (fixture_mode == 1) return E3_OUT_OF_MEMORY;
    if (fixture_mode == 3) return E3_OK; // Deliberately malformed success.
    if (fixture_mode == 10) return 77; // Unknown status.
    E3Decoder *decoder = malloc(sizeof(*decoder));
    if (decoder == NULL) return E3_OUT_OF_MEMORY;
    ++handles; decoder->count = count; decoder->mode = fixture_mode;
    if (count > 0) memcpy(decoder->bytes, bytes, count);
    *output = decoder;
    return fixture_mode == 2 ? E3_INVALID_INPUT : E3_OK;
}
void e3_decoder_destroy(E3Decoder *decoder) {
    if (decoder == NULL) return;
    free(decoder); --handles; ++destroyed;
}
int e3_decoder_copy_bytes(E3Decoder *decoder, uint8_t **output, size_t *count) {
    *output = NULL; *count = 0;
    if (decoder->mode == 5) { *count = 3; return E3_OK; } // Malformed nil/count.
    if (decoder->mode == 6) return E3_OK;
    size_t allocation = decoder->count;
    if (decoder->mode == 7 || decoder->mode == 8 || decoder->mode == 9) allocation = 1;
    if (allocation == 0) return E3_OK;
    uint8_t *bytes = malloc(allocation);
    if (bytes == NULL) return E3_OUT_OF_MEMORY;
    ++buffers; *output = bytes;
    if (decoder->mode == 7) return E3_OK; // Owned allocation, zero logical count.
    if (decoder->mode == 8) { *count = E3_MAX_INPUT + 1; return E3_OK; }
    if (decoder->mode == 9) { *count = SIZE_MAX; return E3_OK; }
    for (size_t i = 0; i < decoder->count; ++i) bytes[i] = decoder->bytes[i] ^ 0xff;
    *count = decoder->count;
    return decoder->mode == 4 ? E3_DECODE_FAILED : E3_OK;
}
void e3_bytes_free(uint8_t *bytes) {
    if (bytes == NULL) return;
    free(bytes); --buffers; ++freed;
}
void e3_visit(const uint8_t *bytes, size_t count, E3Visitor visitor, void *context) {
    for (size_t i = 0; i < count; ++i) visitor(bytes[i], context);
}
```

### Adopt, copy, and close through one confined owner

Shared ByteDecoder setup. It limits input/output to 4096 bytes, translates known and unknown statuses, and registers output cleanup before status/count checks. Class aliases share close state. Compile this source with -strict-memory-safety and link E3Native; it is a declaration-only block.

```swift
import E3Native
import Foundation

public enum ByteDecoderError: Error, Equatable {
    case inputTooLarge, invalidInput, allocationFailed, decodeFailed, invalidOutput, closed
    case unknownStatus(CInt)
    init(status: CInt) {
        switch Int(status) {
        case E3_INVALID_INPUT: self = .invalidInput
        case E3_OUT_OF_MEMORY: self = .allocationFailed
        case E3_DECODE_FAILED: self = .decodeFailed
        default: self = .unknownStatus(status)
        }
    }
}

@safe
public final class ByteDecoder: ~Sendable {
    private var handle: OpaquePointer?
    public static let maximumInputCount = Int(E3_MAX_INPUT)
    public static let libraryVersion = E3_VERSION

    private init(adopting handle: OpaquePointer) { unsafe self.handle = handle }
    deinit { close() }

    public static func make(_ input: Data) throws -> ByteDecoder {
        guard input.count <= maximumInputCount else { throw ByteDecoderError.inputTooLarge }
        return unsafe try input.withUnsafeBytes { raw in
            var created: OpaquePointer?
            let status = unsafe e3_decoder_create(
                raw.bindMemory(to: UInt8.self).baseAddress, raw.count, &created
            )
            guard status == E3_OK else {
                unsafe e3_decoder_destroy(created)
                throw ByteDecoderError(status: status)
            }
            guard let created = unsafe created else { throw ByteDecoderError.invalidOutput }
            return unsafe ByteDecoder(adopting: created)
        }
    }

    public func copyBytes() throws -> Data {
        guard let handle = unsafe handle else { throw ByteDecoderError.closed }
        var output: UnsafeMutablePointer<UInt8>?
        var count = 0
        let status = unsafe e3_decoder_copy_bytes(handle, &output, &count)
        defer { unsafe e3_bytes_free(output) }
        guard status == E3_OK else { throw ByteDecoderError(status: status) }
        guard count >= 0, count <= Self.maximumInputCount else {
            throw ByteDecoderError.invalidOutput
        }
        if count == 0 { return Data() }
        guard let output = unsafe output else { throw ByteDecoderError.invalidOutput }
        return unsafe Data(bytes: output, count: count)
    }

    public func close() {
        guard let owned = unsafe handle else { return }
        unsafe handle = nil
        unsafe e3_decoder_destroy(owned)
    }
}
```

**Walkthrough:** make destroys a partial created handle before throwing and adopts a valid successful handle. copyBytes registers defer immediately after receiving output, then validates status, count, and nonnull storage. Data copies successful bytes before native release. close clears the owned slot before destruction and is idempotent; deinit is a backstop. @safe asserts the wrapper’s contract, and ~Sendable states confinement; neither verifies C or provides a lock. E3_MAX_INPUT/E3_VERSION import as simple constants; function-like E3_MIN is not a Swift function.

## Rules and pitfalls

### Checking failure before registering output cleanup leaks owned failure output

**Common pitfall:** Assuming a failure status means no ownership was transferred. Prepend ByteDecoder setup; this incorrect function is never called.

**Mistake:** Compile-only.

```swift
func lateCleanup(_ decoder: OpaquePointer) throws -> Data {
    var output: UnsafeMutablePointer<UInt8>?
    var count = 0
    let status = unsafe e3_decoder_copy_bytes(decoder, &output, &count)
    guard status == E3_OK else { throw ByteDecoderError.decodeFailed }
    defer { unsafe e3_bytes_free(output) }
    guard count > 0, let output else { return Data() }
    return Data(bytes: output, count: count)
}
```

**Consequence:** Mode 4 can transfer an allocation and then report failure; the early throw skips its release.

**Why:** The API transfers every nonnull output regardless of status. Cleanup registration belongs immediately after that transfer, before any throwing branch.

**Correction:** Use the shared wrapper, whose defer is registered before status validation. Prepend ByteDecoder setup; reset only with no live fixture resources.

```swift
e3_fixture_reset(4)
let decoder = try ByteDecoder.make(Data([1, 2]))
do {
    _ = try decoder.copyBytes()
    preconditionFailure()
} catch ByteDecoderError.decodeFailed {}
precondition(e3_live_buffers() == 0 && e3_freed_buffers() == 1)
decoder.close()
precondition(e3_live_handles() == 0 && e3_destroyed_handles() == 1)
```

**What changes:** The output is freed exactly once even when no Data result is produced. This verifies the injected branch, not real allocation exhaustion or arbitrary native capacity.

### Zero logical length does not erase allocation ownership

**Common pitfall:** Returning before release because count is zero. Prepend ByteDecoder setup; this bad function is compile-only.

**Mistake:** Compile-only.

```swift
func ignoresOwnedEmpty(_ decoder: OpaquePointer) -> Data {
    var output: UnsafeMutablePointer<UInt8>?
    var count = 0
    _ = unsafe e3_decoder_copy_bytes(decoder, &output, &count)
    if count == 0 { return Data() }
    unsafe e3_bytes_free(output)
    return Data()
}
```

**Consequence:** Mode 7 returns a nonnull owned allocation with zero logical count; this function forgets it.

**Why:** Count describes readable output, while the native contract describes release responsibility. A zero count can accompany either null or allocated storage.

**Correction:** Return empty Data after cleanup has already been registered. Prepend ByteDecoder setup.

```swift
e3_fixture_reset(7)
let decoder = try ByteDecoder.make(Data())
let result = try decoder.copyBytes()
precondition(result.isEmpty && e3_freed_buffers() == 1 && e3_live_buffers() == 0)
decoder.close()
decoder.close()
precondition(e3_destroyed_handles() == 1)
```

**What changes:** Both empty-result forms are supported, and close does not destroy the same handle twice. A borrowed static sentinel would require its own different contract.

### An unretained context address is not a new owner

**Common pitfall:** Returning an address to a local object and expecting it to remain alive.

**Mistake:** Compile-only.

```swift
final class VisitBox { var values: [UInt8] = [] }
func invalidContext() -> UnsafeMutableRawPointer {
    Unmanaged.passUnretained(VisitBox()).toOpaque()
}
```

**Consequence:** Using that address after the local owner is gone would access a destroyed object. No such use is executed.

**Why:** passUnretained records a pointer without adding a strong retaining edge. The C API must finish every use within an independently guaranteed lifetime.

**Correction:** Keep the box alive through the complete synchronous native call. Shared visitBytes setup; E3Native calls serially and retains no input, function, or context.

```swift
import E3Native

private final class VisitBox { var values: [UInt8] = [] }
private func visitThunk(_ value: UInt8, _ context: UnsafeMutableRawPointer) {
    let box = unsafe Unmanaged<VisitBox>.fromOpaque(context).takeUnretainedValue()
    box.values.append(value)
}
func visitBytes(_ values: [UInt8]) -> [UInt8] {
    let box = VisitBox()
    return withExtendedLifetime(box) {
        values.withUnsafeBufferPointer { buffer in
            unsafe e3_visit(
                buffer.baseAddress, buffer.count, visitThunk,
                Unmanaged.passUnretained(box).toOpaque())
        }
        return box.values
    }
}
precondition(visitBytes([]).isEmpty)
precondition(visitBytes(Array(UInt8.min...UInt8.max)) == Array(UInt8.min...UInt8.max))
```

**What changes:** Every callback finishes while box and input storage are valid. An asynchronous/overlapping registration needs retained ownership, synchronization, and teardown after all callbacks stop; this synchronous wrapper does not implement that protocol.

## Worked example

### Returned Data survives close, while aliases share handle state

**Setup:** Prepend ByteDecoder setup. Sequential mode-0 fixture.

**Predict:** Does copying the class reference create another native decoder?

```swift
e3_fixture_reset(0)
let decoder = try ByteDecoder.make(Data([0, 1, 255]))
let alias = decoder
let result = try alias.copyBytes()
precondition(result == Data([255, 254, 0]))
precondition(e3_live_buffers() == 0 && e3_freed_buffers() == 1)
decoder.close()
precondition(e3_destroyed_handles() == 1 && e3_live_handles() == 0)
do {
    _ = try alias.copyBytes()
    preconditionFailure()
} catch ByteDecoderError.closed {}
precondition(result == Data([255, 254, 0]))
```

**Outcome:** There is one destroyed handle. The alias observes closed state, but returned Data remains independently owned.

**Why:** Class-reference assignment shares one owner object. Data copies the native result before its buffer is freed; closing the handle cannot invalidate those bytes.

## Interview questions

### Who owns an output returned on failure or with zero length, and where must cleanup be registered?

**Answer:** Whoever the native contract names as owner, independently of status or length. Register the matching release immediately after transfer, before a failure or empty-result return.

**Example:** Prepend ByteDecoder setup.

```swift
for mode in [4, 7] {
    e3_fixture_reset(CInt(mode))
    let decoder = try ByteDecoder.make(Data([1]))
    do {
        let result = try decoder.copyBytes()
        precondition(mode == 7 && result.isEmpty)
    } catch ByteDecoderError.decodeFailed { precondition(mode == 4) }
    precondition(e3_live_buffers() == 0 && e3_freed_buffers() == 1)
    decoder.close()
}
```

**Walkthrough:** The failure and allocated-empty paths each release one buffer. Destruction of the handle is a separate responsibility covered by close.

**Boundary:** Do not substitute free for a library release or apply an owned-output rule to a borrowed/static result. Capacity remains a native implementation invariant.

### Why does a counted/noescape declaration provide a safe Span overload while its Swift @implementation still takes pointer/count?

**Answer:** The overload improves Swift calls; it does not change the C ABI. The Swift implementation must match the original raw signature, then justify constructing its checked borrowed view.

**Example:** Compile-only implementation block. Import E3Safe and use the matching native declaration; link E3Safe.o for clients.

```swift
import E3Safe

func nativeSum(_ bytes: Span<UInt8>) -> UInt32 {
    var total: UInt32 = 0
    for i in bytes.indices { total &+= UInt32(bytes[i]) }
    return total
}

@c @implementation
public func e3_swift_sum(
    _ bytes: UnsafePointer<UInt8>?, _ count: Int
) -> UInt32 {
    let span = unsafe Span(_unsafeElements: UnsafeBufferPointer(start: bytes, count: count))
    return nativeSum(span)
}

func callSwiftImplementation(_ bytes: Span<UInt8>) -> UInt32 {
    E3Safe.e3_swift_sum(bytes)
}
```

**Walkthrough:** The raw implementation accepts nullable pointer plus Int count, constructs a span under that C contract, and uses checked indexing. Swift callers can select the imported generated overload through E3Safe.e3_swift_sum. The native consumer below checks null/zero and real arrays.

**Boundary:** A C caller that violates the annotations still violates the contract. @implementation is not a verifier; it does not appear as a new declaration in the generated export header.

An attempted safe-overload signature is **intentionally rejected** as the raw implementation:

```swift
import E3Safe

@c @implementation
func e3_swift_sum(_ bytes: Span<UInt8>) -> UInt32 { 0 }
```

### When is passUnretained valid for callback context, and what changes if callbacks can escape or overlap?

**Answer:** Only when an independent owner spans every callback and teardown. The synchronous serial contract here permits a bounded lifetime; escaped or overlapping callbacks need a different retained registration and terminal-state protocol.

**Example:** Prepend visitBytes setup.

```swift
let input = Array(UInt8.min...UInt8.max)
precondition(visitBytes(input) == input)
precondition(visitBytes([]) == [])
```

**Walkthrough:** visitBytes keeps its box alive until C finishes every callback and returns copied Swift values. There is no callback after return or concurrent mutation of that box.

**Boundary:** A retained registration must release exactly once after native use stops; releasing during one callback can race another. [[D9 — Continuations and Bridging Callback APIs|D9]] teaches the asynchronous state machine rather than silently extending this fixture.

### Extension: What proves a Swift C export is usable by a native consumer?

**Answer:** A generated C-compatible header, representable signatures, and a compiled/executed C client using that header. A guessed prototype proves less.

**Example:** Compile-only E3Exports module. Combine with the raw implementation block, add -parse-as-library -module-name E3Exports -emit-module -c -emit-objc-header-path /tmp/E3Exports.h.

```swift
@c(E3Mode)
public enum ExportMode: CInt {
    case identity = 0
    case invert = 1
}

@c(E3Transform)
public func transform(_ value: UInt8, _ mode: ExportMode) -> UInt8 {
    switch mode {
    case .identity: value
    case .invert: 255 - value
    }
}
```

**Walkthrough:** The integer enum and E3Transform are emitted into the C-compatible header section. @c uses the C convention and chosen symbol name; valid enum cases select bounded UInt8 arithmetic.

**Boundary:** These attributes arrived in Swift 6.3. Changing from an underscored export can affect symbols/ABI; choose names deliberately and do not feed invalid enum values. [C-compatible declarations](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0495-cdecl.md)

Compile this [[C11 — Strict Memory Safety Mode and Safe Systems Programming|C11]] client after emitting E3Exports.h; link E3Safe.o, the Swift export/implementation object, and the Swift runtime with the Swift driver. It exercises every valid enum byte combination and actual raw native calls:

```c
#include "E3Exports.h"
#include "E3Safe.h"
#include <assert.h>
#include <stdio.h>
int main(void) {
    uint8_t bytes[] = {0, 1, 255};
    assert(e3_sum(bytes, 3) == 256);
    assert(e3_swift_sum(bytes, 3) == 256);
    assert(e3_sum(NULL, 0) == 0);
    assert(e3_swift_sum(NULL, 0) == 0);
    for (int value = 0; value < 256; ++value) {
        assert(E3Transform((uint8_t)value, E3ModeIdentity) == value);
        assert(E3Transform((uint8_t)value, E3ModeInvert) == 255 - value);
    }
    puts("C client called generated exports and a Swift implementation of a C header.");
    return 0;
}
```

The native compiler and linker commands, emitted-header inspection, and execution are retained in the validation fixtures. The generated header flag keeps its historical Objective-C spelling even for its C-compatible section.

### Extension: What do nullability and non-sendability actually reject?

**Answer:** They reject particular invalid calls or transfers, rather than proving the whole library safe. A nonnull parameter does not accept nil, and an explicitly non-Sendable owner cannot satisfy a Sendable requirement.

**Example:** Prepend ByteDecoder setup. Intentionally rejected.

```swift
func requireSendable<T: Sendable>(_ value: T) {}
func sendDecoder(_ decoder: ByteDecoder) { requireSendable(decoder) }
```

**Walkthrough:** This call is rejected because ByteDecoder explicitly suppresses Sendable. The owner’s class aliases still share state, so confinement must cover all of them.

**Boundary:** The fixture’s global failure mode/counters are also sequential. Neither ~Sendable nor a safe wrapper serializes arbitrary native globals.

Independent **intentionally rejected** nullability, representability, and ABI-mismatch controls:

```swift
import E3Native

func nilRequiredHandle() {
    var bytes: UnsafeMutablePointer<UInt8>?
    var count = 0
    _ = unsafe e3_decoder_copy_bytes(nil, &bytes, &count)
}
```

```swift
struct NotCRepresentable { let value: Int }
@c
func invalidExport(_ value: NotCRepresentable) {}
```

```swift
import E3Safe

@c @implementation
func e3_swift_sum(_ bytes: UnsafePointer<UInt8>?, _ count: Double) -> UInt32 { 0 }
```

The raw pointer/count call is also **intentionally rejected** under -strict-memory-safety and -warnings-as-errors. Safe overload availability does not make the raw call safe:

```swift
import E3Safe

func unacknowledgedRawCall(_ bytes: UnsafePointer<UInt8>?, _ count: Int) -> UInt32 {
    e3_sum(bytes, count)
}
```

## Focused exercise

**Task:** Wrap synchronous native byte processing with empty-input and failure cleanup, then state whether input may be retained.

**Acceptance checks:**

- Success copies output before release and preserves it after close.
- Empty input and allocated-empty output return empty Data.
- Partial failure releases its buffer; close is idempotent and malformed output is rejected before access.

**Solution:** Prepend ByteDecoder setup. Reset only after each case has closed its handle.

```swift
for input in [Data(), Data([0, 255]), Data(UInt8.min...UInt8.max)] {
    e3_fixture_reset(0)
    let decoder = try ByteDecoder.make(input)
    let first = try decoder.copyBytes()
    let second = try decoder.copyBytes()
    precondition(first == Data(input.map { $0 ^ 0xff }) && second == first)
    decoder.close()
    decoder.close()
    precondition(e3_live_handles() == 0 && e3_live_buffers() == 0)
    precondition(e3_destroyed_handles() == 1)
}
for mode in [4, 5, 8, 9] {
    e3_fixture_reset(CInt(mode))
    let decoder = try ByteDecoder.make(Data([1]))
    do {
        _ = try decoder.copyBytes()
        preconditionFailure("Expected invalid result")
    } catch {}
    decoder.close()
    precondition(e3_live_handles() == 0 && e3_live_buffers() == 0)
}
```

**Reasoning:** The native code copies input synchronously and retains no borrowed address. Each output is either copied under validated bounds or rejected before reading, with cleanup already registered. Additional controls check all creation modes, boundaries, alias/deinit paths, and input mutation; simulated failure does not establish a general leak-free library.

## In practice

**Choose it when:** Translate a real documented native contract into owned Swift values and a small confined owner.

**Investigate by:** Inspect actual headers/implementation, count every owned resource, and exercise native failure output and generated clients.

**Strong evidence:** Verify raw ABI, generated safe calls, cleanup, and consumers separately; annotations and successful compilation are supporting evidence, not capacity or lifetime proofs.

## References

### Related guides

- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[A12 — Availability and Conditional Compilation]]
- [[A13 — Error handling and typed throws]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C1 — ARC fundamentals]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C11 — Strict Memory Safety Mode and Safe Systems Programming]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[E1 — Objective-C Interoperability]]
- [[E2 — Foundation and Objective-C Bridging Behavior]]
- [[E4 — C++ interoperability]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [C interoperability](https://www.swift.org/documentation/c-interop/)
- [SE-0495](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0495-cdecl.md)
- [Safe interop documentation](https://www.swift.org/documentation/cxx-interop/safe-interop/)
- [Swift importer documentation](https://github.com/swiftlang/swift/blob/main/docs/HowSwiftImportsCAPIs.md)
- [Swift 6.3 release](https://www.swift.org/blog/swift-6.3-released/)
- [Swift 6.4 release](https://www.swift.org/blog/swift-6.4-released/)
- [SE-0458: strict memory-safety checking](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md)
- [Official native-client implementation test](https://github.com/swiftlang/swift/blob/main/test/Interpreter/c_implementation.swift)
