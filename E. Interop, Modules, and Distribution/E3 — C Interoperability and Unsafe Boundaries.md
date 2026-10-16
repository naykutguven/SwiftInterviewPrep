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

**Goal:** Wrap a synchronous C pointer/count operation with explicit empty-input and failure cleanup.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A C declaration rarely states the full ownership contract. Pointer type and count do not prove allocation size, initialization, retention, or who releases output on failure. A Swift wrapper must combine the actual native contract with scoped inputs, owned outputs, status mapping, and confinement. [C interoperability](https://www.swift.org/documentation/c-interop/)

## Rules and pitfalls

### Register cleanup for every owned output

If a native API transfers an allocation even on failure or with zero length, register the matching release immediately after receiving ownership, before status-based returns. Match allocator and deallocator exactly.

**Common pitfall:** Checking status before registering cleanup leaks failure output. A zero count does not mean a nonnull owned allocation can be ignored; ownership follows the contract, not size.

### Keep borrowed input within the native use

Use scoped buffer access for a synchronous nonretaining call. A retaining operation needs an owner lasting through completion and cancellation. PassUnretained callback context is valid only while another guaranteed owner spans every callback.

**Common pitfall:** Bounds annotations, @safe, or unsafe acknowledgment do not verify C implementation behavior. Inspect and run the native fixture; check generated Span overloads separately from a pointer/count ABI implementation.

## Interview questions

### Who owns an output returned on failure or with zero length, and where must cleanup be registered?

The caller owns it whenever the C contract transfers ownership, including failure and zero-length cases. Register matching cleanup immediately after ownership is established, then inspect status/count. Treat borrowed outputs and static sentinels according to their different contracts.

### Why does a counted/noescape declaration provide a safe Span overload while its Swift @implementation still takes pointer/count?

The importer can generate a Swift-facing Span overload from bounds and noescape annotations, encoding safer access for callers. The exported/imported C ABI still uses pointers and counts, so a Swift @implementation supplies that raw signature. A safe overload does not rewrite the ABI or prove the implementation.

### When is passUnretained valid for callback context, and what changes if callbacks can escape or overlap?

Only when a separate owner guarantees the context remains alive for all callbacks and no use occurs after teardown. Escaping or overlapping callbacks need a retained registration owner, synchronized terminal state, and exactly matched release after all native access ends. Avoid releasing context from one callback while another can still use it.

## Focused exercise

**Task:** Wrap a synchronous pointer/count C call. Check empty input and failure cleanup; state whether the callee may retain the input.

**Acceptance checks:**

- Empty input follows the documented null/count and ownership rules.
- A failure that returns owned output still releases it once.
- Successful output is copied before native release; the input is not retained.

**Solution:**

The complete native fixture below is part of this solution. Use its exact owned-output/release API rather than substituting an unrelated free. The wrapper accepts Swift-owned bytes, lends them only for the synchronous call, registers output cleanup before checking failure, then copies successful output to Swift-owned data.

**Native fixture contract:** Wrap E3Native using Data input/results and Swift errors. Input is copied during creation. Every nonnull output belongs to the wrapper, including partial failure and zero-length allocation. Support bounded input/output, valid empty results, deterministic idempotent close, fallback deinitialization, and use-after-close rejection. Keep the owner explicitly non-Sendable because the fixture supplies no shared-access guarantee.

The complete E3Native.h used by the wrapper is:

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

Import it as a Clang module using temporary module.modulemap:

> module E3Native { header "E3Native.h" export * }

The enum constants above import as Int in this fixture, while the status return is CInt. The wrapper converts the status to Int for matching and retains the original CInt for unknown codes. E3_MAX_INPUT and E3_VERSION import as simple constants; E3_MIN does not become a Swift function. C preprocessor macros and Swift compiler macros are different systems.

Reference implementation:

```swift
import Foundation
import E3Native

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
        guard count >= 0, count <= Self.maximumInputCount else { throw ByteDecoderError.invalidOutput }
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

The output allocation is registered for cleanup before checking status/count. A zero count returns empty Data while still freeing any nonnull allocation. A nonzero count requires a nonnull output and must fit the resource policy. Data copies the bytes before native release, so returned data survives handle closure. The wrapper cannot independently measure an arbitrary pointer's real capacity; that invariant still comes from the audited C implementation.

The class reference deliberately has shared identity; it is not a value-semantic copy of a decoder. ~Sendable states the confinement intent, and @safe states that safe callers need not manage its pointer contract themselves. Neither attribute synchronizes the C implementation or forces a particular thread for cleanup. A thread-affine library needs an owner/teardown design that actually honors that requirement.

These independent negatives use the header and wrapper respectively. A nil handle cannot call a _Nonnull decoder parameter, and the explicitly non-Sendable wrapper cannot satisfy a Sendable generic requirement.

```swift
import E3Native
func nilRequiredHandle() {
    var bytes: UnsafeMutablePointer<UInt8>?
    var count = 0
    _ = unsafe e3_decoder_copy_bytes(nil, &bytes, &count)
}
```
```swift
func requireSendable<T: Sendable>(_ value: T) {}
func sendDecoder(_ decoder: ByteDecoder) { requireSendable(decoder) }
```

### Real fixture implementation and failure modes

E3Native.c below owns allocations with malloc/free, pairs every resource with the documented release function, and copies input. Its modes are deliberately test-only: 0 normal; 1 simulated creation allocation failure; 2 partial handle failure; 3 successful status with missing handle; 4 partial output failure; 5 nil with positive count; 6 nil/zero success; 7 allocated/zero success; 8 over-limit count; 9 SIZE_MAX count; 10 unknown status. Modes 8/9 must be rejected before any read of their deliberately insufficient allocation.

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

**Acceptance cases:** empty input; every byte value and boundary size; copied input despite caller mutation; independent owned output after native cleanup; repeated decode; normal close twice; closing through an alias; deinit without explicit close; oversized input; each creation/decode fault mode; exact live/destroy/free counters; synchronous context callbacks; generated C export/enum calls; expected compiler rejections; strict-memory-safety and Address Sanitizer checks. Do not execute the escaped-pointer example or violate annotated native bounds to test a crash.

The validation harness uses bounded sequential cases and checks that native live counts return to zero. Simulated allocation failure proves that branch, not real system exhaustion. Counters establish cleanup for these calls, not a universal absence of leaks or races in an arbitrary C library.

**Reasoning:** The safety argument comes from the fixture's explicit allocation/retention behavior plus the wrapper's cleanup order. Aliases of a class owner share close state; non-sendability alone does not synchronize native globals. Check the native API's concurrency and failure contracts independently.

## In practice

**Choose it when:** Expose a small Swift contract that hides native allocation and error conventions.

**Investigate by:** Inspect real headers and implementation, then test native failure output and cleanup counters.

**Strong evidence:** Validate generated headers, ordinary native consumers, and strict-memory diagnostics without treating annotations as proof.

### Generated C interfaces and safe overloads — optional

#### Generate a C header from Swift

The following emits E3Mode with its integer cases and E3Transform into the C-compatible section of the generated header. The C client below calls both cases for every UInt8 value. @c uses the C calling convention and a C name; choose collision-resistant names for a real library. Supply only valid enum values under this exported closed-enum contract.

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

Use the generated header flag even though its historical spelling mentions Objective-C: -emit-objc-header -emit-objc-header-path E3Exports.h. The tested header compiles as C11. @c/@implementation are 6.3 additions, not new 6.4 syntax; moving from an underscored @_cdecl export can have symbol/ABI consequences and requires consumer review. [SE-0495](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0495-cdecl.md)

#### Import bounds and lifetime into a safe overload

Temporary E3Safe.h annotates a read-only pointer with its element count and synchronous nonescape contract. The C implementation of e3_sum adds bytes modulo 2^32. e3_swift_sum is declared here but implemented in Swift in the next block.

```c
#pragma once
#include <stddef.h>
#include <stdint.h>
#include <ptrcheck.h>
#include <lifetimebound.h>
uint32_t e3_sum(const uint8_t * _Nullable __counted_by(count) bytes __noescape, size_t count);
uint32_t e3_swift_sum(const uint8_t * _Nullable __counted_by(count) bytes __noescape, size_t count);
```
> module E3Safe { header "E3Safe.h" export * }
```c
#include "E3Safe.h"
uint32_t e3_sum(const uint8_t *bytes, size_t count) {
    uint32_t result = 0;
    for (size_t i = 0; i < count; ++i) result += bytes[i];
    return result;
}
```

The installed compiler produces a Span<UInt8> overload from this nullable counted/noescape pointer declaration without an experimental flag. The checked empty case uses nil with count zero. The nonoptional Span API does not imply that nil with a positive count is valid for native callers. Bounds annotations must describe reality; __noescape is a lifetime promise, not an instruction to copy/retain input.

```swift
import E3Safe

func sum(_ bytes: Span<UInt8>) -> UInt32 {
    e3_sum(bytes)
}

@available(anyAppleOS 26.0, *)
func sumArray(_ bytes: [UInt8]) -> UInt32 {
    e3_sum(bytes.span)
}
```

sum takes an existing Span and compiles for the older targets used here. sumArray uses Array.span and therefore carries the separate Apple OS 26 availability requirement. Scope, declaration annotations, SDK availability, and strict-memory-safety checking remain distinct. [Safe interop documentation](https://www.swift.org/documentation/cxx-interop/safe-interop/)

#### Implement a pre-existing C declaration in Swift

The raw implementation matches the imported pointer/count signature. Its unsafe construction trusts the C caller's annotated bounds/lifetime contract; nativeSum then operates on a checked borrowed view. Swift callers can select the generated imported overload through E3Safe.e3_swift_sum. The generated E3Exports.h does not redeclare the @implementation function: native consumers keep using E3Safe.h.

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

A complete native client follows. Compile the Swift export and implementation under module E3Exports to emit E3Exports.h, compile this C source plus E3Safe.c, and link the objects with the Swift driver/runtime. It uses real generated/imported declarations, not a guessed Swift-compatible C prototype.

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

#### Rejected signatures and an explicit unsafe call

Each negative is independent. The first cannot export an ordinary Swift struct. The next two use E3Safe: one changes the imported size parameter to Double; the other attempts to use the safe Span signature as the raw @implementation. Both fail matching/representability checks.

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
```swift
import E3Safe
@c @implementation
func e3_swift_sum(_ bytes: Span<UInt8>) -> UInt32 { 0 }
```

The following raw call can compile in ordinary mode but is rejected when StrictMemorySafety diagnostics are promoted to errors. Its pointer/count signature remains unsafe even though a safe overload exists. Adding unsafe acknowledges the raw contract; prefer the safe overload when it fits the operation.

```swift
import E3Safe
func unacknowledgedRawCall(_ bytes: UnsafePointer<UInt8>?, _ count: Int) -> UInt32 {
    e3_sum(bytes, count)
}
```

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
