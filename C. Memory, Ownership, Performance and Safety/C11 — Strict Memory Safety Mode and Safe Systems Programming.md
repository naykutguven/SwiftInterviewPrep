---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - safety
  - systems-programming
  - swift6-4
---

# C11 — Strict Memory Safety Mode and Safe Systems Programming

**Learning goals:**

- Separate strict memory-safety diagnostics and warning policy from actual runtime correctness and concurrency checking.
- Use @unsafe, unsafe expressions, and @safe claims at the boundary they describe.
- Audit a complete native wrapper’s initialized bounds, lifetime, retention, and cleanup; prefer checked owned/borrowed alternatives when they fit.

**Prerequisites:** [[C6 — Unsafe pointers, buffers, and raw memory]] for pointer storage states and native lifetime, [[C3 — Exclusivity enforcement and `inout`]] for exclusivity, and [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]] for checked span/raw operations. [[F5 — Language mode, build settings, and migration flags]] explains diagnostic policy.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A native checksum receives an address and a count. Swift cannot prove the C implementation respects that count or that the address stays valid. **Strict memory safety** is an opt-in diagnostic/audit facility that highlights where those obligations leave the checked language model.

`@unsafe` marks a declaration with caller obligations; `unsafe` acknowledges an expression requiring review. `@safe` is an author’s assertion that a declaration encapsulates those obligations behind a safe contract. None validates the native body, repairs an invalid count, or prevents every pointer escape. Warning/error policy and strict concurrency are separate settings. [Strict memory safety](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Native fixture: CChecksum

Save these three files in one temporary directory. The C function synchronously reads count initialized bytes, retains nothing, accepts null only when count is zero, and returns the sum modulo 2^32. Its implementation allocates nothing and invokes no callbacks. This small dependency makes the audit concrete.

**CChecksum.h:**

```c
#include <stddef.h>
#include <stdint.h>
uint32_t c_checksum(const uint8_t *bytes, size_t count);
```

**CChecksum.c:**

```c
#include "CChecksum.h"
uint32_t c_checksum(const uint8_t *bytes, size_t count) {
    uint32_t result = 0;
    for (size_t i = 0; i < count; ++i) result += bytes[i];
    return result;
}
```

**module.modulemap:**

```text
module CChecksum {
    header "CChecksum.h"
    export *
}
```

From that directory, build the native object:

```sh
xcrun --sdk macosx clang -std=c11 -Wall -Wextra -Werror \
  -c CChecksum.c -o /tmp/CChecksum.o
```

For each independent Swift example importing CChecksum, use the [[D1 — Structured Concurrency Fundamentals|D1]] baseline command plus `-I . /tmp/CChecksum.o`. The checker examples additionally use `-strict-memory-safety`; ordinary accepted cases keep `-warnings-as-errors`. To observe the stated warning, omit warnings-as-errors as labeled. Validation uses an isolated temporary module directory and explicit matching SDK/target. A retaining native API needs a different owner-through-completion contract.

### Enabling the checker exposes an unacknowledged native call

Standalone Swift client of CChecksum. Compile with -strict-memory-safety and without -warnings-as-errors for this diagnostic example; the actual valid C operation still runs.

```swift
import CChecksum

func checksum(_ bytes: [UInt8]) -> UInt32 {
    guard !bytes.isEmpty else { return 0 }
    return bytes.withUnsafeBufferPointer { buffer in
        c_checksum(buffer.baseAddress!, buffer.count)
    }
}
precondition(checksum([]) == 0 && checksum([1, 2, 3]) == 6)
```

**Walkthrough:** The checker reports the unsafe expression. Compiling ordinarily, warning, and promoting that warning to an error are different policies; none changes the C function’s actual pointer contract. The correction below narrows the acknowledgment to the audited call.

## Rules and pitfalls

### A broad unsafe surface can force unnecessary caller acknowledgments

**Common pitfall:** Marking an entire value-based wrapper unsafe even though its implementation can encapsulate the native address/count obligations.

**Mistake:** Standalone runnable example.

```swift
import CChecksum

@unsafe func broadChecksum(_ bytes: [UInt8]) -> UInt32 {
    guard !bytes.isEmpty else { return 0 }
    return bytes.withUnsafeBufferPointer { buffer in
        unsafe c_checksum(buffer.baseAddress!, buffer.count)
    }
}
let result = unsafe broadChecksum([1, 2, 3])
precondition(result == 6)
```

**Consequence:** Every caller must acknowledge the whole wrapper, obscuring which operation actually needs native reasoning.

**Why:** The declaration advertises caller obligations even though this Array-based API can keep initialization, bounds, and lifetime inside its implementation.

**Correction:** Keep the owned Swift input surface safe and acknowledge the precise audited native expression. This shared checksum setup is the working wrapper.

```swift
import CChecksum

func checksum(_ bytes: [UInt8]) -> UInt32 {
    guard !bytes.isEmpty else { return 0 }
    return bytes.withUnsafeBufferPointer { buffer in
        unsafe c_checksum(buffer.baseAddress!, buffer.count)
    }
}
precondition(checksum([]) == 0)
precondition(checksum([1, 2, 3]) == 6)
precondition(checksum([255, 1]) == 256)
```

**What changes:** Callers provide owned bytes, not an arbitrary pointer/count pair. Array supplies initialized UInt8 storage and count inside the scope; the separately reviewed C function retains nothing. No explicit @safe annotation is needed for this already safe signature.

### An author’s @safe claim does not prove a returned address valid

**Common pitfall:** Labeling a borrowed-address result safe while allowing it to leave its actual access scope. Compile-only; the returned pointer is never used.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
@safe func borrowedAddress(_ bytes: [UInt8]) -> UnsafePointer<UInt8>? {
    bytes.withUnsafeBufferPointer { buffer in
        buffer.baseAddress
    }
}
```

**Consequence:** The checker can accept this claimed-safe declaration even though the returned address has no valid later access contract.

**Why:** The annotation is an assertion by the author, not a verifier of the body. Unsafe pointer values are escapable; closure syntax alone does not express a dependent safe lifetime.

**Correction:** Return an owned sequence when retention is required, without exposing a borrowed address.

```swift
func ownedBytes(_ bytes: [UInt8]) -> [UInt8] { bytes }
var source: [UInt8] = [1, 2, 3]
let saved = ownedBytes(source)
source[0] = 9
precondition(saved == [1, 2, 3] && source == [9, 2, 3])
```

**What changes:** The result has Array’s owned value contract rather than an invalid address. This proves logical independence, not a fixed physical-copy count. For scoped processing, a Span plus an owned scalar result can avoid retaining the sequence.

## Worked example

### Audit a caller-specified prefix before crossing the C boundary

**Setup:** Standalone CChecksum client under strict checking with warnings as errors. This wrapper accepts a count only after validating it against the actual owned array.

**Predict:** What happens for negative/oversized counts, zero, and a valid prefix?

```swift
import CChecksum

func prefixChecksum(_ bytes: [UInt8], count: Int) -> UInt32? {
    guard count >= 0, count <= bytes.count else { return nil }
    guard count > 0 else { return 0 }
    return bytes.withUnsafeBufferPointer { buffer in
        unsafe c_checksum(buffer.baseAddress!, count)
    }
}
precondition(prefixChecksum([1, 2, 3], count: 2) == 3)
precondition(prefixChecksum([], count: 0) == 0)
precondition(prefixChecksum([], count: 1) == nil)
precondition(prefixChecksum([1], count: -1) == nil)
precondition(prefixChecksum([1], count: Int.max) == nil)
```

**Outcome:** Valid initialized prefixes return their sums; invalid counts return nil before any native call.

**Why:** The guards establish bounds and avoid a nil dereference on empty input. The scoped owner supplies the address lifetime, and the native non-retention contract keeps that lifetime sufficient. These are reviewable proofs beyond the unsafe spelling.

## Interview questions

### What constructs does strict memory safety want acknowledged?

**Answer:** Unsafe pointer types/operations, unsafe native APIs, raw memory access, and declarations explicitly marked unsafe are audit boundaries. A safe wrapper can encapsulate them, but it must state and uphold their obligations; diagnostic promotion is a separate policy.

**Example:** Prepend the corrected checksum setup and link CChecksum.

```swift
let result = checksum([0, 255, 1])
precondition(result == 256)
precondition(checksum([]) == 0)
```

**Walkthrough:** The shared wrapper keeps the C call’s acknowledgment inside the implementation. An ordinary caller needs no pointer or unsafe marker. The first mechanics example supplies the warning-producing version, and an additional control promotes it to an error.

**Boundary:** This does not establish native memory safety by compilation alone. C source review, initialization/bounds, ownership, and retention still matter; strict concurrency checks another set of obligations.

### Why make unsafe code explicit even when this implementation is known-correct?

**Answer:** It identifies where compiler guarantees stop and which assumptions a future refactor must preserve. A changed count, retained address, or initialization policy can invalidate the audit without changing the function’s Swift types.

**Example:** Prepend prefixChecksum from the worked example and link CChecksum.

```swift
precondition(prefixChecksum([1, 2, 3], count: 3) == 6)
precondition(prefixChecksum([1, 2, 3], count: 4) == nil)
precondition(prefixChecksum([1, 2, 3], count: -1) == nil)
```

**Walkthrough:** The prefix wrapper checks the actual array’s initialized range before making the acknowledged call. Changing the native count expression must preserve that proof; the checker does not derive the C loop’s bounds from the Swift guard.

**Boundary:** The intentionally invalid count probe below compiles too and is never called. An acknowledgment is responsibility, not a runtime check or evidence that the responsibility was met.

Standalone **compile-only**, acknowledged but invalid native count. This function is never invoked; reading one byte beyond the initialized range is not a runtime test:

```swift
import CChecksum

func invalidCount(_ bytes: [UInt8]) -> UInt32 {
    guard !bytes.isEmpty else { return 0 }
    return bytes.withUnsafeBufferPointer { buffer in
        unsafe c_checksum(buffer.baseAddress!, buffer.count + 1)
    }
}
```

### Extension: What does @safe assert, and why can a scoped callback still escape a pointer?

**Answer:** It asserts that the implementation upholds its safe API contract, including allowed uses of unsafe-looking signature types. It does not verify the body. A callback can return an escapable UnsafePointer even when the closure itself does not escape.

**Example:** Compile-only under -strict-memory-safety. Neither function is called with an escaping result.

```swift
@safe func withBorrowedBytes<R>(
    _ bytes: [UInt8], _ body: (UnsafeBufferPointer<UInt8>) -> R
) -> R {
    bytes.withUnsafeBufferPointer(body)
}
@safe func escaped(_ bytes: [UInt8]) -> UnsafePointer<UInt8>? {
    withBorrowedBytes(bytes) { buffer in buffer.baseAddress }
}
```

**Walkthrough:** R can be a raw address value, so this wrapper does not prevent pointer escape. The escaped result is never used. The earlier @safe borrowedAddress example illustrates why the author’s claim cannot replace lifetime reasoning.

**Boundary:** Compile-only diagnostic/lifetime review. A real dependent safe-view return requires a valid supported lifetime contract; use [[C6 — Unsafe pointers, buffers, and raw memory|C6]]/[[C12 — `Span`, `InlineArray`, and modern low-level abstractions|C12]]’s scoped Span processing or an owned result instead of an unsound promise.

### Extension: Which checked operations can reduce manual memory obligations?

**Answer:** Use initialized output spans for temporary storage and safe raw operations for supported byte representations, where their contracts fit. They reduce manual state tracking but do not remove bounds, lifetime, format, or availability requirements.

**Example:** Standalone runnable example.

```swift
func sum(_ values: Span<Int>) -> Int {
    var total = 0
    for index in values.indices { total += values[index] }
    return total
}
let total = withTemporaryAllocation(of: Int.self, capacity: 3) { output in
    output.append(10)
    output.append(20)
    precondition(output.count == 2 && output.capacity == 3)
    return sum(output.span)
}
precondition(total == 30)
```

**Walkthrough:** The helper tracks two initialized values inside capacity three, releases them/storage after the closure, and returns an owned scalar. The strict checker accepts the source without native pointer acknowledgment. [Temporary output spans](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md)

**Boundary:** [[C6 — Unsafe pointers, buffers, and raw memory|C6]]/[[C12 — `Span`, `InlineArray`, and modern low-level abstractions|C12]] demonstrate raw loading and cleanup on throw; [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`|C7]]/[[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] cover dependent references and exclusive owners. A checked load can still trap on invalid bounds, and a unique class payload can still have other aliases. No automatic performance improvement follows.

| Obligation | Alternative to consider | Remaining contract |
| --- | --- | --- |
| Track initialized temporary elements through throw | OutputSpan/OutputRawSpan withTemporaryAllocation | Capacity, initialized count, local lifetime |
| Read raw bytes | RawSpan.load for supported ConvertibleFromBytes values | Bounds, endian order, semantic fields |
| Expose scoped storage access | Span, Ref, MutableRef | Owner lifetime, exclusivity, per-operation availability |
| Prevent accidental resource-owner duplication | Noncopyable owner / UniqueBox | Adoption, release context, class payload aliases |

## Focused exercise

**Task:** Audit the complete CChecksum wrapper’s initialized bounds, lifetime, and cleanup obligations.

**Acceptance checks:**

- Empty input returns zero without obtaining/dereferencing a nonempty base address.
- Array supplies initialized UInt8 elements and the count within one access scope.
- The actual C function retains nothing and allocates nothing; only the native expression needs acknowledgment.

**Solution:** Prepend the corrected checksum setup. Compile with the baseline plus -strict-memory-safety, -I for the C module, and the native object:

```swift
precondition(checksum([]) == 0)
precondition(checksum([1, 2, 3]) == 6)
precondition(checksum([255, 255, 1]) == 511)
let owned: [UInt8] = [0, 1, 2, 3]
precondition(checksum(owned) == 6 && owned == [0, 1, 2, 3])
```

**Reasoning:** The function’s native source and wrapper jointly establish the contract. There is no manual allocation to release here; an allocating or retaining dependency would need an explicit owner and completion/cleanup policy. This exercise verifies its bounded fixture, not arbitrary C implementations.

## In practice

**Choose it when:** Audit a genuine low-level boundary and keep acknowledgments at the smallest meaningful expression.

**Investigate by:** Combine diagnostic controls with native-source review and valid empty/boundary/throwing fixtures; use appropriate sanitizer/integration checks for the real dependency.

**Strong evidence:** Explain the trusted assumptions that remain after a warning-free build and verify the native retention/release contract without treating @safe as a proof engine.

## References

### Related guides

- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[E3 — C Interoperability and Unsafe Boundaries]]
- [[E4 — C++ interoperability]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [SE-0458](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md)
- [SE-0524: Temporary allocation with output spans](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md)
- [SE-0525: Safe raw-span loading/storage](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0525-rawspan-safe-loading-api.md)
- [Swift.org. "Swift 6.2 Released." Swift.org Blog](https://swift.org/blog/swift-6.2-released/)
- [Swift.org. "Safely Mixing Swift and C/C++." Swift.org Documentation](https://swift.org/documentation/cxx-interop/safe-interop/)
