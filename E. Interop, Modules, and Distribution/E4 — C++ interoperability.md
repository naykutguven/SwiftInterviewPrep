---
tags:
  - swift
  - ios
  - interview-prep
  - interop
  - cpp-interop
---

# E4 — C++ Interoperability

**Goal:** Copy and mutate an imported C++ value while tracing the native copy and lifetime contract.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Supported C++ classes and structs generally import as Swift values whose copy/move/destruction behavior follows C++ special members. Value syntax does not establish deep independence: a vector-owning value and a borrowed-pointer value have different semantics. Importability, lifetime, exception handling, concurrency confinement, and consumer build requirements are separate review questions. [Swift/C++ interoperability](https://www.swift.org/documentation/cxx-interop/)

## Rules and pitfalls

### Inspect native ownership before adapting it

Review constructors, copy/move operations, destructors, invalidating mutations, and borrowed return values. Convert retained results into Swift-owned snapshots when clients should not depend on native lifetime. Keep throwing C++ behavior behind a supported error boundary.

**Common pitfall:** Copying a native value that contains a shared pointer or borrowed view can still share the underlying resource. Do not infer semantics from the imported Swift type category.

### Verify build and lifetime surfaces independently

Use a real C++20 module and the required Swift C++ interop mode for dependent targets. Stable Span input bridging and experimental lifetime-dependent return overloads have different support limits.

**Common pitfall:** A Swift-only return DTO does not make the native engine Sendable or necessarily remove transitive C++ build requirements. Compile an ordinary consumer and run empty/mutation/lifetime cases before claiming the boundary works.

## Interview questions

### Which native operations can copy, invalidate a view, or throw, and how would your Swift API encode each contract?

Native copy constructors define copying; container growth, mutation, or destruction can invalidate views; exceptions can arise from native operations under their C++ contract. Encode independent snapshots, scoped borrows, and fallible operations explicitly in Swift. Prevent unsupported exceptions from escaping the interop boundary.

### Why does a Swift-only snapshot not make the engine Sendable or remove a dependent module's C++ setting?

Converting an output does not synchronize the engine's mutable internals or global state. Imported declarations can still require C++ interoperability in dependent compilation. Keep the engine confined and validate the ordinary consumer's actual manifest and imports.

### What evidence distinguishes the stable span input support from the conditional return-overload examples?

Stable input support is established by compiling/running the supported C++20 span input functions without the experimental return feature. Conditional return overloads require their additional annotations/flags and still show lifetime limitations in this toolchain. Separate successful inputs from rejected return uses and keep the latter outside baseline recommendations.

## Focused exercise

**Task:** Copy an imported C++ value, mutate the copy, and observe native copy/destruction behavior. Explain which result depends on the native contract.

**Acceptance checks:**

- Copy the imported value and mutate the copy; inspect both values.
- Observe copy/destruction counters without asserting an optimizer-independent count or destruction order.
- Compile and run a native-backed empty-input case and an ordinary interop-enabled consumer.

**Solution:**

Use the real C++ module, value type, and Swift probe below. The source defines what native copying means; it is not inferred from Swift assignment syntax.

### Build a real C++20 module

Use -cxx-interoperability-mode=default and -Xcc -std=c++20 for Swift. Compile native consumers as C++20 with the same SDK/target and C++ standard library. The Apple checks use libc++. std::span requires the C++20 dialect; the Swift language mode is separately -swift-version 6.

For SwiftPM, .interoperabilityMode(.Cxx) belongs in the Swift target's settings and cxxLanguageStandard: .cxx20 selects the package dialect. An umbrella header/module map exposes the C++ target. Review downstream requirements before enabling interop in an existing package. E5 covers manifests and E6 covers import visibility. [Build setup](https://www.swift.org/documentation/cxx-interop/project-build-setup/), [supported configurations](https://www.swift.org/documentation/cxx-interop/status/)

Complete temporary E4Core.hpp:

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

Temporary module.modulemap:

> module E4Core { header "E4Core.hpp" requires cplusplus export * }

E4Buffer starts empty and cannot be copied; its move operations allow native ownership transfer. replace copies input into a temporary vector, then swaps only after success. Its status is 0 success, 1 over the 4096-byte policy, or 2 native failure. The injected exception is caught before returning to Swift. failNextReplacement and the E4Value counters are test instrumentation, not a thread-safe production monitoring API.

E4Value, e4Name, and the conversion examples are bounded successful copy/conversion probes. They do not define a recoverable allocation-failure policy for every C++ standard-library constructor or copy. Audit all potentially throwing native operations in a real adapter; the complete exception-to-status contract here belongs to E4Buffer.replace.

### Value semantics, copies, and destruction

The following probe keeps original and copy observable after mutating the copy. It returns original first element 1, copied first element 9, and an observed native copy count of at least one. After the function returns, the fixture's live-value count is zero. A separate borrowedRead call leaves the copy count unchanged in the checked debug and optimized builds.

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
```

These counters establish what this fixture did under the recorded settings. They do not prove that every argument pass copies, that borrowing eliminates all allocations inside a function, or that Swift and C++ containers share a copy-on-write model. E4Value's copy constructor deliberately deep-copies its vector. A C++ copy constructor could instead retain shared storage or copy an unsafe pointer.

C++ types with suitable deleted-copy/move members can import as noncopyable. SWIFT_NONCOPYABLE also expresses a Swift-side restriction for a native type that is otherwise copyable. Reference semantics require a different mapping: SWIFT_SHARED_REFERENCE names real retain/release functions, and return-ownership annotations must match the native API. SWIFT_IMMORTAL_REFERENCE and SWIFT_UNSAFE_REFERENCE describe different lifetime obligations. Do not infer Swift reference counting just from the C++ class keyword. These reference mappings are source-reviewed context; this fixture tests value and unique ownership. [C++ mapping documentation](https://www.swift.org/documentation/cxx-interop/)

### Explicit standard-library conversions

The imported std::string and std::optional<int> remain C++ types. This returns ("Swift + C++", 42, nil); the optional payload is CInt, matching C++ int in the tested targets.

```swift
import E4Core
import CxxStdlib

func converted() -> (String, CInt?, CInt?) {
    (String(e4Name()), Optional(fromCxx: e4Optional(true)), Optional(fromCxx: e4Optional(false)))
}
```

Do not assume a conversion implies valid application data, independent nested resources, or negligible cost. std::string_view is a borrowed view and needs an owner/lifetime policy even when an explicit String conversion eventually creates owned text.

**Reasoning:** The native implementation determines whether copies are independent. Counter observations describe one build and call path, not a language-guaranteed number of copies. The borrowed-view and output-snapshot fixtures retain their confinement requirements below.

## In practice

**Choose it when:** A role or product genuinely requires C++ interoperability.

**Investigate by:** Compile actual headers/dialects and inspect native ownership, invalidation, and exception contracts.

**Strong evidence:** Verify stable inputs, conditional returns, native execution, and consumer settings separately.

### Span support and confined snapshot facade — optional

#### Stable read-only and mutable span inputs

These compile with the ordinary interop settings and strict-memory-safety diagnostics promoted to errors, without SafeInteropWrappers. e4Sum uses std::span<const uint8_t>; e4Invert uses std::span<uint8_t>. The latter imports a safe overload taking inout MutableSpan<UInt8>. The signatures do not imply ownership transfer or asynchronous retention. The explicit empty checks are necessary for this distribution: directly calling the generated e4Sum overload with Span<UInt8>() traps with signal 5, while an empty array-backed span passed. The local Cxx overlay requires a nonnil buffer pointer during its span conversion. Keep empty handling in the audited boundary; do not infer identical behavior for all empty backing representations.

```swift
import E4Core

func bridge(_ bytes: Span<UInt8>) -> UInt32 {
    bytes.isEmpty ? 0 : e4Sum(bytes)
}
func invert(_ bytes: inout MutableSpan<UInt8>) {
    if !bytes.isEmpty { e4Invert(&bytes) }
}
```

For the tested noescape calls, the generated overload performs the boundary conversion. For nonempty input, no manually constructed C++ span is needed at the Swift call site. Array.span itself requires Apple OS 26; the temporary older-deployment driver constructs a Swift Span from a scoped buffer with an explicit unsafe acknowledgment. That scope/annotation audit remains separate from API availability.

An otherwise identical function without the noescape annotation does not accept a Swift Span in the baseline configuration:

```swift
import E4Core
func missingContract(_ bytes: Span<UInt8>) -> UInt32 { e4Unannotated(bytes) }
```

#### Lifetime-dependent returns: a checked limitation

The release announcement describes Span/std::span bridging in both directions. The installed compiler accepts the input overloads above but rejects the following return-overload calls with ordinary settings. The corresponding Clang importer source explicitly gates lifetimebound wrapper generation behind the legacy SafeInteropWrappers feature. [Swift 6.4 release](https://www.swift.org/blog/swift-6.4-released/), [importer implementation](https://github.com/swiftlang/swift/blob/main/lib/ClangImporter/SwiftifyDecl.cpp)

**Compile-only research example:** Both functions below compile in the tested configuration after adding -enable-experimental-feature SafeInteropWrappers -enable-experimental-feature LifetimeDependence. The experimental runtime driver nevertheless failed with signal 11 in both isolation configurations. One diagnostic run with AddressSanitizer reached the first empty-tail case and reported stack-overflow during the subsequent native sum. The root cause is not established; neither function receives a successful runtime-validation claim from that driver. Do not execute these examples as a production recipe or enable the flags project-wide on this evidence. The baseline owned-copy exercise below passes its separate runtime checks.

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

The following independent negatives use that same experimental configuration. The first returns a view into a local array; the anchor parameter deliberately makes a return lifetime expressible so the diagnostic concerns the wrong local dependency. The second tries to mutate a buffer while its view remains in use. Both are rejected for their intended lifetime/exclusivity violation. The Array.span control separately has OS 26 availability.

```swift
import E4Core
@available(anyAppleOS 26.0, *)
func escaped(_ anchor: Span<UInt8>) -> Span<UInt8> { let local: [UInt8] = [1, 2];return e4Tail(local.span) }
```
```swift
import E4Core
func overlap() -> UInt32 { var buffer = E4Buffer();let view: Span<UInt8> = buffer.view();buffer.failNextReplacement();return e4Sum(view) }
```

No dangling access or overlapping mutation is executed. Do not replace lifetime annotations with a false independent-value annotation to make a compiler error disappear. The historical safe-interop page describes the 6.2 experiment; its return examples should not be copied as an unconditional 6.4 build recipe. [Safe interop annotations](https://www.swift.org/documentation/cxx-interop/safe-interop/)

#### Ownership and conversion negatives

Each snippet is independent and rejected in the ordinary baseline. E4Buffer cannot be consumed twice. The C++ string and optional require explicit conversion, and a C++ value does not automatically satisfy a Sendable constraint.

```swift
import E4Core
func take(_ buffer: consuming E4Buffer) {}
func twice() { let buffer = E4Buffer();take(buffer);take(buffer) }
```
```swift
import E4Core
func implicitString() -> String { e4Name() }
```
```swift
import E4Core
func implicitOptional() -> CInt? { e4Optional(true) }
```
```swift
import E4Core
func needsSendable<T: Sendable>(_ value: T) {}
func sendNativeValue() { needsSendable(E4Value()) }
```

**Task:** Replace a feature-layer native vector/span API with a bounded service. Preserve old state on failure, keep native mutation serialized, and return independently owned bytes/checksum. Verify the real implementation and show which unsafe assumptions remain inside the adapter.

The baseline copy helper deliberately does not return a borrowed view. It borrows the E4Buffer owner for the entire synchronous operation, obtains the native view under an explicit unsafe acknowledgment, copies its bytes, and discards both views before returning. No native mutation or suspension occurs inside the helper. The unsafe conversion's Swift lifetime tracking alone does not recover the hidden native owner relationship; the helper's contract and implementation supply that audit.

```swift
import E4Core
import CxxStdlib

func copyBytes(_ buffer: borrowing E4Buffer) -> [UInt8] {
    let nativeView = unsafe buffer.view()
    let view = unsafe Span(_unsafeCxxSpan: nativeView)
    var result: [UInt8] = []
    result.reserveCapacity(view.count)
    for i in view.indices { result.append(view[i]) }
    return result
}
```

Combine that helper with the following service:

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
    public func replaceSamples(_ bytes: [UInt8]) throws(SampleError) { try replace(&storage, with: bytes) }
    public func snapshot() -> SampleSnapshot {
        let bytes = copyBytes(storage)
        return SampleSnapshot(bytes: bytes, checksum: bytes.reduce(0) { $0 &+ UInt32($1) })
    }
}
```

For empty replacement, the adapter calls the raw overload with an explicitly constructed empty C++ span, avoiding the generated conversion while preserving the native clear operation. For nonempty replacement, replace uses a synchronous borrowed input; the native function copies before returning and catches its own exceptions. The owner stays private to the actor. snapshot copies while isolated and computes the checksum from those same owned bytes, so the pair is coherent. There is no await between acquiring native state and finishing the snapshot. Callers can keep or mutate their arrays without changing stored native data, and subsequent replacement cannot invalidate an earlier snapshot.

The public actor boundary does not pass E4Buffer to another executor. The helper's inout access covers the native mutation; the synchronous scoped input does not escape. This is still a bounded synchronous native operation, with no automatic cancellation or dedicated-thread guarantee.

With the full exercise in scope, this nonisolated synchronous call is rejected:

```swift
func wrongIsolation(_ service: SampleService) -> SampleSnapshot { service.snapshot() }
```

**Acceptance cases:** empty/single/representative/all-byte/4096-byte inputs; bitwise inversion; copied input despite later caller mutation; independent snapshots after replacement; 4097-byte rejection in Swift and C++; injected C++ failure translated to nativeFailure with old data unchanged; retry success; multiple serialized actor snapshot callers; value-copy independence/destruction; intended compiler rejections; native header/consumer integration; bounded sanitizer runs. Simulated failure is not a real memory-exhaustion test.

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
