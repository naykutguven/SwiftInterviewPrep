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

**Goal:** Audit a synchronous native wrapper’s initialized bounds, lifetime, and cleanup obligations.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Strict memory safety is an opt-in diagnostic and audit facility for unsafe constructs. It makes operations whose full preconditions Swift cannot prove visible. Unsafe acknowledgment identifies a responsibility; it does not check that responsibility or repair invalid code. This is independent of strict concurrency checking. [SE-0458](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md)

## Rules and pitfalls

### Minimize and document the unsafe surface

Prefer checked spans, owned Swift values, and safe raw loading where their contracts fit. Confine unavoidable native calls behind safe input/output types, stating lifetime, bounds, alignment, retention, and initialization assumptions.

**Common pitfall:** Marking a whole subsystem unsafe hides where reasoning is needed. Put acknowledgment at the narrow audited expression and require native dependencies to state whether they retain pointers or return borrowed storage.

### Keep safety assertions separate from proof

@unsafe marks an unsafe declaration; unsafe acknowledges an expression. @safe is the author's claim that an implementation encapsulates the obligations behind a safe contract, even with unsafe-looking types in an allowed signature.

**Common pitfall:** A scoped closure can return or save an unsafe pointer, and @safe does not stop it. A warning-free build can still contain a lifetime bug. Use lifetime-dependent safe views where possible and verify the implementation's actual behavior.

## Interview questions

### What does @safe assert, and why can a scoped callback still escape a pointer?

It asserts that callers may rely on the declaration's safe contract because the implementation encapsulates its unsafe obligations. It is not a verifier of the body. A callback wrapper can still leak a raw pointer through returned or stored values; closure syntax alone does not supply a nonescapable lifetime relationship.

### What kinds of constructs does strict memory safety mode want you to acknowledge?

Unsafe pointer types/operations, raw memory access, unsafe imported APIs, and other declarations marked unsafe need explicit audit treatment under the checker. Distinguish an unsafe declaration's caller obligations from a safe wrapper's implementation responsibility; configure warning/error policy separately.

### Why is making unsafe code explicit valuable even if the code is known-correct?

It concentrates review and testing on the boundaries where compiler proofs stop, making future changes to counts, lifetimes, and native retention easier to audit. A known-correct implementation can still depend on assumptions that an innocent refactor violates. Explicit contracts preserve that knowledge.

## Focused exercise

**Task:** Audit one native wrapper's pointer access. Identify initialization, bounds, and cleanup obligations and explain where unsafe acknowledgment belongs.

**Acceptance checks:**

- Empty input avoids a pointer dereference and returns the stated identity.
- Array supplies initialized bytes and the count within the access scope.
- The native function retains nothing; no pointer or allocation escapes.

**Solution:**

Native fixture dependency: CChecksum declares and implements the following synchronous function. It reads count initialized bytes, retains nothing, accepts null only for count zero, and returns the sum modulo 2^32.

```c
#include <stddef.h>
#include <stdint.h>
uint32_t c_checksum(const uint8_t *bytes, size_t count) {
    uint32_t result = 0;
    for (size_t i = 0; i < count; ++i) result += bytes[i];
    return result;
}
```

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
```

Compile the Swift client with -strict-memory-safety and the baseline flags, linking the C object and a module map exposing its header.

**Reasoning:** The public wrapper accepts owned Swift storage. The unsafe expression acknowledges a native call whose read-only synchronous lifetime is separately documented. No manual allocation needs cleanup here; a retaining API would require an owner through completion and a different wrapper.

## In practice

**Choose it when:** Audit a genuine low-level boundary rather than adding acknowledgments indiscriminately.

**Investigate by:** Combine strict diagnostics, native-source review, bounds fixtures, and appropriate sanitizers.

**Strong evidence:** Explain remaining trusted assumptions and verify the native contract without equating warning suppression with correctness.

### Checked replacement operations — optional

#### Prefer the checked Swift 6.4 operation

| Existing obligation | Alternative to consider | Obligation that remains |
| --- | --- | --- |
| Track initialized temporary elements and clean them up after a throw | withTemporaryAllocation with OutputSpan / OutputRawSpan | Respect capacity, keep views local, and distinguish initialized count from capacity. |
| Load raw bytes through an unchecked pointer | RawSpan.load for ConvertibleFromBytes values | Validate length and offsets before a trapping check; define endian order and semantic field constraints. |
| Expose a temporary pointer/reference to owned storage | Span, Ref, or MutableRef where supported | Keep the owner alive and obey shared/exclusive access and availability requirements. |
| Duplicate ownership of an external resource | A noncopyable owner or UniqueBox | Audit raw-resource adoption, destruction, thread affinity, and aliases inside reference payloads. |

These are choices about contracts, not automatic performance improvements. C6–C8/C12 contain checked examples; some APIs require OS 27, while core temporary allocation and span access support older targets. Enabling diagnostics does not change deployment availability. The release adds useful alternatives to acknowledge-and-audit workflows, while external pointer contracts still require review.

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
