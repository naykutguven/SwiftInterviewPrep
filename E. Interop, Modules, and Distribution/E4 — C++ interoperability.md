---
tags:
  - swift
  - ios
  - interview-prep
  - interop
  - cpp-interop
---

# E4 — C++ Interoperability

**Learning goals:**

- Read imported copy/move/destruction and invalidation from the real C++ API.
- Separate stable Span inputs, conditional return wrappers, owned snapshots, and native exception handling.
- Keep native mutable ownership confined and validate the actual consumer dialect, module, and runtime.

**Prerequisites:** [[C6 — Unsafe pointers, buffers, and raw memory]]/[[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]/[[C8 — Copyable, noncopyable types, and `~Copyable`]]/[[C12 — `Span`, `InlineArray`, and modern low-level abstractions]] for memory, ownership, and spans; [[D4 — Actors and Actor Isolation]] for actors; [[E3 — C Interoperability and Unsafe Boundaries]] for native contracts. [[E5 — Modules, packages, targets, products, and resources]]/[[E6 — Import Visibility and Dependency Leakage]] explain consumer build settings and dependency exposure. This chapter is relevant when the role uses C++.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Assigning an imported C++ class value does not automatically share an Objective-C-style object. Its native copy constructor decides whether copying duplicates a vector, shares storage, or merely copies a raw address. A borrowed native view is still invalidated by whatever operations its owner permits.

Review copy/move/destruction, borrowed lifetime, mutation, exceptions, and concurrency separately. A Swift actor can serialize access, and an owned Swift snapshot can outlive native mutation, but neither repairs a wrong C++ lifetime annotation. [C++ interoperability](https://www.swift.org/documentation/cxx-interop/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Build the actual C++20 contract

Shared E4Core setup. Save the following complete header as E4Core.hpp with the module map beside it. Swift builds add -I with that directory, -cxx-interoperability-mode=default, and -Xcc -std=c++20. C++20 is separate from Swift 6 mode; Apple validation uses the selected SDK and libc++. A SwiftPM target uses .interoperabilityMode(.Cxx) and the package cxxLanguageStandard: .cxx20. [Build settings](https://www.swift.org/documentation/cxx-interop/project-build-setup/)

E4Value deliberately copies its vector independently and records sequential copy/live counters. E4Buffer deletes copying, catches replacement exceptions into status values, and exposes a borrowed view invalidated by replacement. Instrumentation is not a thread-safe production API.

```cpp
#pragma once
#include <swift/bridging>
#include <lifetimebound.h>
#include <span>
#include <vector>
#include <string>
#include <optional>
#include <cstdint>
#include <stdexcept>

using E4ReadOnlyBytes = std::span<const uint8_t>;
using E4MutableBytes = std::span<uint8_t>;
inline uint32_t e4Sum(E4ReadOnlyBytes bytes __noescape) noexcept {
    uint32_t result = 0;
    for (auto byte : bytes) result += byte;
    return result;
}
inline E4ReadOnlyBytes e4Tail(E4ReadOnlyBytes bytes __lifetimebound) noexcept {
    return bytes.empty() ? bytes : bytes.subspan(1);
}
inline void e4Invert(E4MutableBytes bytes __noescape) noexcept {
    for (auto &byte : bytes) byte ^= 0xff;
}
// Deliberately lacks the noescape annotation; used only as a compile control.
inline uint32_t e4Unannotated(E4ReadOnlyBytes bytes) noexcept { return e4Sum(bytes); }

class SWIFT_ESCAPABLE E4Buffer {
    std::vector<uint8_t> storage;
    bool failNext = false;
public:
    E4Buffer() noexcept = default;
    E4Buffer(const E4Buffer &) = delete;
    E4Buffer &operator=(const E4Buffer &) = delete;
    E4Buffer(E4Buffer &&) noexcept = default;
    E4Buffer &operator=(E4Buffer &&) noexcept = default;
    void failNextReplacement() noexcept { failNext = true; } // Fixture only.
    int replace(E4ReadOnlyBytes input __noescape) noexcept {
        if (input.size() > 4096) return 1;
        try {
            if (failNext) { failNext = false; throw std::runtime_error("injected"); }
            std::vector<uint8_t> replacement;
            if (!input.empty()) replacement.assign(input.begin(), input.end());
            storage.swap(replacement);
            return 0;
        } catch (...) { return 2; } // Never unwind a C++ exception through Swift.
    }
    E4ReadOnlyBytes view() const noexcept __lifetimebound { return storage; }
    size_t count() const noexcept { return storage.size(); }
};

// Copy/lifetime counters are sequential test instrumentation.
inline int e4Copies = 0, e4LiveValues = 0;
class SWIFT_ESCAPABLE E4Value {
    std::vector<int> storage;
public:
    E4Value() : storage{1, 2, 3} { ++e4LiveValues; }
    E4Value(const E4Value &other) : storage(other.storage) { ++e4Copies; ++e4LiveValues; }
    E4Value(E4Value &&other) noexcept : storage(std::move(other.storage)) { ++e4LiveValues; }
    ~E4Value() { --e4LiveValues; }
    int first() const noexcept { return storage.front(); }
    void setFirst(int value) noexcept { storage.front() = value; }
};
inline int e4CopyCount() noexcept { return e4Copies; }
inline int e4LiveCount() noexcept { return e4LiveValues; }
inline void e4ResetCopies() noexcept { e4Copies = 0; }

inline std::string e4Name() { return "Swift + C++"; }
inline std::optional<int> e4Optional(bool present) {
    return present ? std::optional<int>(42) : std::nullopt;
}
```

```text
module E4Core { header "E4Core.hpp" requires cplusplus export * }
```

### Observe a native value copy before discussing performance

Shared valueProbe setup. Native counters describe these bounded calls; they are reset only without concurrent fixture use.

```swift
import E4Core

func valueProbe() -> (CInt, CInt, CInt) {
    e4ResetCopies()
    let original = E4Value()
    var copy = original
    copy.setFirst(9)
    return (original.first(), copy.first(), e4CopyCount())
}
func borrowedRead(_ value: borrowing E4Value) -> CInt { value.first() }

let state = valueProbe()
precondition(state.0 == 1 && state.1 == 9 && state.2 >= 1)
precondition(e4LiveCount() == 0)
```

**Walkthrough:** Mutating the copy leaves the original’s first element equal to 1. At least one native copy was observed in this configuration; all probe-owned values are destroyed by return. The number/order of copies is not a portable Swift guarantee.

### Use the supported borrowed input overload

Shared bridge/invert setup. Strict checking is enabled; no experimental return feature is used. Explicitly guard empty spans for the installed overlay’s conversion limitation.

```swift
import E4Core

func bridge(_ bytes: Span<UInt8>) -> UInt32 {
    bytes.isEmpty ? 0 : e4Sum(bytes)
}
func invert(_ bytes: inout MutableSpan<UInt8>) {
    if !bytes.isEmpty { e4Invert(&bytes) }
}
```

**Walkthrough:** Nonempty input uses a generated std::span conversion for the noescape call. Empty read returns the sum identity and empty mutation does nothing before conversion. This avoids the earlier recorded empty-default-span trap without executing it again. Array.span needs OS 26 separately.

### Convert standard-library results explicitly

Shared converted setup. Native std::string and optional are different types from Swift String/Optional.

```swift
import CxxStdlib
import E4Core

func converted() -> (String, CInt?, CInt?) {
    (String(e4Name()), Optional(fromCxx: e4Optional(true)), Optional(fromCxx: e4Optional(false)))
}

let result = converted()
precondition(result.0 == "Swift + C++" && result.1 == 42 && result.2 == nil)
```

**Walkthrough:** String copies the supported native text representation and Optional(fromCxx:) maps present/absent CInt. This bounded conversion says nothing about validation of arbitrary encoding or allocation-failure recovery.

## Rules and pitfalls

### A borrowed view must not be retained across invalidating mutation

**Common pitfall:** Saving an unsafe native view then replacing its vector. Compile-only; no invalidated access is executed.

**Mistake:** Compile-only.

```swift
import CxxStdlib
import E4Core

func invalidView() -> UInt32 {
    var buffer = E4Buffer()
    let bytes: [UInt8] = [1, 2]
    _ = buffer.replace(bytes.span)
    let view = unsafe buffer.view()
    _ = unsafe buffer.replace(E4ReadOnlyBytes())
    let expired = unsafe Span(_unsafeCxxSpan: view)
    return expired.isEmpty ? 0 : UInt32(expired[0])
}
```

**Consequence:** The view still describes storage that replacement may have released; converting its wrapper does not restore validity.

**Why:** std::span owns no vector storage. The unsafe conversion cannot infer an owner relationship that has been erased into the imported native view.

**Correction:** Copy while borrowing the owner, with no mutation or suspension, and return only owned bytes. Shared copyBytes setup.

```swift
import CxxStdlib
import E4Core

func copyBytes(_ buffer: borrowing E4Buffer) -> [UInt8] {
    let nativeView = unsafe buffer.view()
    let view = unsafe Span(_unsafeCxxSpan: nativeView)
    var result: [UInt8] = []
    result.reserveCapacity(view.count)
    for i in view.indices { result.append(view[i]) }
    return result
}
```

**What changes:** The owner stays alive through the complete copy. Both view values are discarded before returning, so later mutation cannot invalidate the Array. The unsafe audit still depends on E4Buffer’s real implementation and confinement.

### Imported native values do not automatically become Sendable

**Common pitfall:** Satisfying a Sendable consumer just because the type imports as a value.

**Mistake:** Compile-only; intentionally rejected.

```swift
import E4Core

func needsSendable<T: Sendable>(_ value: T) {}
func sendNativeValue() { needsSendable(E4Value()) }
```

**Consequence:** The generic call is rejected; the native type has no checked Sendable promise.

**Why:** Value import does not prove independent nested resources or synchronized native operations. A facade must keep its live native owner in a valid access domain.

**Correction:** Keep E4Buffer private to an actor, serialize replacement, and export owned Sendable snapshots. Prepend copyBytes setup; this shared SampleService source uses synchronous bounded native work.

```swift
import E4Core

public struct SampleSnapshot: Sendable, Equatable {
    public let bytes: [UInt8]
    public let checksum: UInt32
}
public enum SampleError: Error, Equatable { case tooLarge, nativeFailure }
func replace(_ buffer: inout E4Buffer, with bytes: [UInt8]) throws(SampleError) {
    guard bytes.count <= 4096 else { throw .tooLarge }
    let status = bytes.withUnsafeBufferPointer { raw in
        if raw.isEmpty { return unsafe buffer.replace(E4ReadOnlyBytes()) }
        let view = unsafe Span(_unsafeElements: raw)
        return buffer.replace(view)
    }
    switch status {
    case 0: return
    case 1: throw .tooLarge
    default: throw .nativeFailure
    }
}
public actor SampleService {
    private var storage = E4Buffer()
    public init() {}
    public func replaceSamples(_ bytes: [UInt8]) throws(SampleError) {
        try replace(&storage, with: bytes)
    }
    public func snapshot() -> SampleSnapshot {
        let bytes = copyBytes(storage)
        return SampleSnapshot(bytes: bytes, checksum: bytes.reduce(0) { $0 &+ UInt32($1) })
    }
}
```

**What changes:** The actor never transfers E4Buffer to another executor. snapshot copies bytes and computes the checksum from those same owned bytes before returning, with no intervening await. C++ exceptions are caught in replace, then mapped to SampleError. This does not promise automatic cancellation or a dedicated native thread.

### A missing lifetime contract cannot be supplied by call-site optimism

**Common pitfall:** Passing Swift Span to an otherwise identical unannotated std::span parameter.

**Mistake:** Compile-only; intentionally rejected.

```swift
import E4Core

func missingContract(_ bytes: Span<UInt8>) -> UInt32 { e4Unannotated(bytes) }
```

**Consequence:** The baseline call rejects because this generated input overload requires the supported noescape contract.

**Why:** The importer must know that borrowed Swift storage does not escape through the call. An arbitrary native function is not automatically nonretaining.

**Correction:** Use the annotated operation whose implementation actually honors nonescape. Prepend bridge/invert setup.

```swift
let bytes: [UInt8] = [0, 1, 255]
precondition(bridge(bytes.span) == 256)
precondition(bridge(Span<UInt8>()) == 0)
var mutable: [UInt8] = [0, 1, 255]
mutable.withUnsafeMutableBufferPointer { raw in
    var view = unsafe MutableSpan(_unsafeElements: raw)
    invert(&view)
}
precondition(mutable == [255, 254, 0])
```

**What changes:** The real native input and mutation execute under the supported contract, and empty input is handled before conversion. No borrowed view escapes.

## Worked example

### Owned snapshots survive later replacement

**Setup:** Prepend copyBytes and SampleService setups. This actor has a single private native buffer.

**Predict:** Will a saved snapshot or caller input change after native replacement?

```swift
let service = SampleService()
var input: [UInt8] = [1, 2, 3]
try await service.replaceSamples(input)
input[0] = 9
let saved = await service.snapshot()
precondition(saved.bytes == [1, 2, 3] && saved.checksum == 6)
try await service.replaceSamples([8])
let next = await service.snapshot()
precondition(saved.bytes == [1, 2, 3] && next.bytes == [8] && next.checksum == 8)
```

**Outcome:** Saved state remains [1, 2, 3], while the new snapshot contains [8]. Caller mutation is independent too.

**Why:** replace copies the input into native storage; snapshot separately copies that native storage into a Swift Array. These are explicit ownership contracts, not an inferred deep-copy promise from an imported class keyword.

## Interview questions

### Which native operations can copy, invalidate a view, or throw, and how should a Swift API encode each contract?

**Answer:** Read each operation’s native contract: E4Value copies independently, E4Buffer replacement can invalidate its views, and replace catches its own exceptions into a status. Borrow for a bounded read, copy for retention, and translate declared failure.

**Example:** Prepend copyBytes and SampleService setups.

```swift
func failureProof() throws {
    var buffer = E4Buffer()
    try replace(&buffer, with: [1, 2])
    buffer.failNextReplacement()
    do {
        try replace(&buffer, with: [9])
        preconditionFailure()
    } catch SampleError.nativeFailure {}
    precondition(copyBytes(buffer) == [1, 2])
    try replace(&buffer, with: [9])
    precondition(copyBytes(buffer) == [9])
}
try failureProof()
```

**Walkthrough:** The injected native exception is caught before leaving C++, translated into nativeFailure, and leaves the old vector intact. Retry succeeds because the native failure flag is one-shot.

**Boundary:** E4Value constructors/copies and std::string conversion are only bounded successful probes; this does not define allocation-failure recovery for every C++ standard-library operation. Never unwind an uncaught C++ exception through Swift.

### Why does a Swift-only snapshot not make the engine Sendable or remove a dependent module’s C++ setting?

**Answer:** The snapshot changes the public values that cross the boundary. The implementation still owns a native object, and consumer compilation can still require its C++ module/settings according to the emitted module dependencies.

**Example:** Prepend copyBytes and SampleService setups.

```swift
let service = SampleService()
try await service.replaceSamples([0, 255])
let snapshot: SampleSnapshot = await service.snapshot()
func accept<T: Sendable>(_ value: T) {}
accept(snapshot)
precondition(snapshot.bytes == [0, 255] && snapshot.checksum == 255)
```

**Walkthrough:** The Sendable generic accepts SampleSnapshot, whose fields are owned Swift values. The native owner remains inside the actor. A separate ordinary module/client build checks the consumer settings rather than assuming the signature erases import dependencies.

**Boundary:** Do not mark the engine unchecked Sendable to satisfy a call. [[E6 — Import Visibility and Dependency Leakage|E6]] separates module loading from linkage; a source-only facade does not establish binary distribution or synchronized native globals.

### What evidence distinguishes stable span input support from the conditional return-overload examples?

**Answer:** Compile and run inputs under ordinary settings, then check return wrappers separately with their required annotations/features. Successful input conversion is not evidence that a lifetime-dependent return is production-ready.

**Example:** Compile-only, intentionally rejected under the ordinary interop baseline. Import E4Core.

```swift
import E4Core

func tailSum(_ bytes: Span<UInt8>) -> UInt32 {
    let tail: Span<UInt8> = e4Tail(bytes)
    return e4Sum(tail)
}
func borrowedChecksum(_ buffer: borrowing E4Buffer) -> UInt32 {
    let view: Span<UInt8> = buffer.view()
    return e4Sum(view)
}
```

**Walkthrough:** Under ordinary settings these return-overload calls reject. Separate object-compilation controls add SafeInteropWrappers and LifetimeDependence and test the return declarations without executing them. The earlier driver failed with signal 11 in both tested isolation configurations; one Address Sanitizer run reported stack overflow, and the root cause was not established. This revision object-compiles the experimental declarations and does not claim successful return execution.

**Boundary:** The historical release announcement is broader than this installed compiler result. Keep the owned-copy path as the checked recipe and preserve the limitation. [Importer implementation](https://github.com/swiftlang/swift/blob/main/lib/ClangImporter/SwiftifyDecl.cpp)

With -enable-experimental-feature SafeInteropWrappers -enable-experimental-feature LifetimeDependence, the following independent lifetime violations are **intentionally rejected**. No dangling read or overlapping mutation is executed:

```swift
import E4Core

@available(anyAppleOS 26.0, *)
func escaped(_ anchor: Span<UInt8>) -> Span<UInt8> {
    let local: [UInt8] = [1, 2]
    return e4Tail(local.span)
}
```

```swift
import E4Core

func overlap() -> UInt32 {
    var buffer = E4Buffer()
    let view: Span<UInt8> = buffer.view()
    buffer.failNextReplacement()
    return e4Sum(view)
}
```

### Extension: Which ownership and conversion boundaries remain source errors?

**Answer:** A deleted-copy owner cannot be transferred twice, and native string/optional results require explicit Swift conversion. These are separate constraints, not signs that every imported C++ type behaves alike.

**Example:** Standalone runnable example.

```swift
import CxxStdlib
import E4Core

let text = String(e4Name())
let present: CInt? = Optional(fromCxx: e4Optional(true))
let absent: CInt? = Optional(fromCxx: e4Optional(false))
precondition(text == "Swift + C++" && present == 42 && absent == nil)
```

**Walkthrough:** The value conversions are explicit and preserve the native CInt payload. The negatives below expose transfer and implicit-conversion assumptions independently.

**Boundary:** SWIFT_SHARED_REFERENCE requires real retain/release functions, while immortal/unsafe reference mappings have different lifetimes. This fixture checks value/unique ownership rather than claiming those reference mappings were run.

```swift
import E4Core

func take(_ buffer: consuming E4Buffer) {}
func twice() {
    let buffer = E4Buffer()
    take(buffer)
    take(buffer)
}
```

```swift
import E4Core

func implicitString() -> String { e4Name() }
```

```swift
import E4Core

func implicitOptional() -> CInt? { e4Optional(true) }
```

A synchronous nonisolated call into the actor is independently **intentionally rejected**; prepend copyBytes and SampleService setups:

```swift
func wrongIsolation(_ service: SampleService) -> SampleSnapshot { service.snapshot() }
```

## Focused exercise

**Task:** Copy an imported C++ value, mutate it, and verify the native copy/lifetime contract alongside a bounded empty-input case.

**Acceptance checks:**

- Original/copy retain independent native values and the observed copy count is at least one.
- No probe-owned values remain after return; borrowing adds no observed copy in the checked build.
- Empty/nonempty supported inputs execute; no invalid borrowed return is used.

**Solution:** Prepend valueProbe and bridge/invert setups.

```swift
let result = valueProbe()
precondition(result.0 == 1 && result.1 == 9 && result.2 >= 1)
precondition(e4LiveCount() == 0)
func borrowedProof() {
    let value = E4Value()
    e4ResetCopies()
    let before = e4CopyCount()
    precondition(borrowedRead(value) == 1)
    precondition(e4CopyCount() == before)
}
borrowedProof()
precondition(e4LiveCount() == 0)
precondition(bridge(Span<UInt8>()) == 0)
let bytes = Array(UInt8.min...UInt8.max)
precondition(bridge(bytes.span) == bytes.reduce(0) { $0 &+ UInt32($1) })
```

**Reasoning:** The native deep-copy constructor explains independence; counters report this call path under recorded optimization. A native consumer and additional snapshot/failure/4096-boundary controls verify other contracts. Simulated failure and bounded sanitizer checks are not general allocation, leak, or speed proofs.

## In practice

**Choose it when:** Use C++ interop when the native API is part of the product, and choose confinement plus owned snapshots when retaining views is unnecessary.

**Investigate by:** Read the actual native members, annotations, dialect and consumer module settings; test empty input and native failure before broadening use.

**Strong evidence:** Separate stable input runtime, conditional return compilation, native consumer execution, and bounded copy/cleanup observations.

## References

### Related guides

- [[A12 — Availability and Conditional Compilation]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C11 — Strict Memory Safety Mode and Safe Systems Programming]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D4 — Actors and Actor Isolation]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[E3 — C Interoperability and Unsafe Boundaries]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[E7 — Resilience, ABI, and Module Stability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [Swift/C++ interoperability](https://www.swift.org/documentation/cxx-interop/)
- [Build setup](https://www.swift.org/documentation/cxx-interop/project-build-setup/)
- [supported configurations](https://www.swift.org/documentation/cxx-interop/status/)
- [Swift 6.4 release](https://www.swift.org/blog/swift-6.4-released/)
- [importer implementation](https://github.com/swiftlang/swift/blob/main/lib/ClangImporter/SwiftifyDecl.cpp)
- [Safe interop annotations](https://www.swift.org/documentation/cxx-interop/safe-interop/)
- [feature definitions](https://github.com/swiftlang/swift/blob/main/include/swift/Basic/Features.def)
- [Official span interface fixture](https://github.com/swiftlang/swift/blob/main/test/Interop/Cxx/stdlib/Inputs/std-span.h)
