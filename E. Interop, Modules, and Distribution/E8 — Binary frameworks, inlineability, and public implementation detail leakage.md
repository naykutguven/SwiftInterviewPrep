---
tags:
  - swift
  - ios
  - interview-prep
  - interop-modules-distribution
  - binary-frameworks
  - inlinable
  - usableFromInline
---

# E8 — Binary Frameworks, Inlineability, and Public Implementation Detail Leakage

**Goal:** Show which implementation an unchanged optimized client retains after a compatible library update.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A library and its clients may rebuild on different schedules. Source visibility controls naming; symbol export controls library entry points; body visibility allows client emission/optimization. Mandatory inlining and library-side specialization make still different commitments. A client-emitted implementation can survive a library update even when it was not inlined into its caller. [SE-0497](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)

## Rules and pitfalls

### Expose bodies only for a justified boundary

A normal library call can reach updated code. An inlinable or implementation-exported body can leave old copies in clients. @usableFromInline keeps a helper source-internal but makes it an ABI dependency for exposed implementations.

**Common pitfall:** Making an internal helper public solely to satisfy an exposed body creates an unnecessary source contract. Keep the caller opaque unless client optimization justifies the binary commitment.

### Separate optimization controls from correctness

@inline(always) requests mandatory eligible direct inlining; it differs from the older optimizer hint. @specialized can create selected library-side specializations and redispatch while preserving a generic API. These controls arrived in 6.3 and are available in 6.4.

**Common pitfall:** A specialization attribute does not prove a speedup or validate arithmetic. Define input range/overflow policy and compare equivalent optimized work; see [[F3 — Performance investigation habits]].

## Interview questions

### Why does replacing the dylib update an interface-exported function but leave the client-emitted function unchanged?

The interface-exported call reaches the library's callable implementation, so replacement changes its result. The client-emitted function's definition already belongs to the original executable, so replacing the dylib cannot change it. @inline(never) prevents inlining but does not move that definition back into the library.

### When is @usableFromInline preferable to a public helper, and when should the caller stay opaque?

Use @usableFromInline when a source-internal helper must be referenced by a deliberately exposed body and its ABI commitment is acceptable. Keep the public caller opaque when implementation freedom and behavior updates matter more than client optimization. Public helper access is appropriate only for a real supported client use.

### How can prespecialization retain a generic API without exposing its implementation?

The library builds selected specializations and chooses them from its generic implementation path without giving clients the algorithm body or publishing a separate specialized source API. Verify emitted interfaces/symbols and measure actual callers. Specialization is a build/representation choice, not new generic semantics.

## Focused exercise

**Task:** Compare a library-called function with a client-visible body across a compatible library replacement. Explain which output an unchanged client may retain.

**Acceptance checks:**

- Run the identical original executable with v1 and a compatible v2 library.
- Compare it with a rebuilt v2 consumer and inspect emitted interface/symbols.
- Report observed client retention without claiming every inlinable call always inlines.

**Solution:**

Use separate E8Visibility library and client builds with -enable-library-evolution -O, baseline concurrency flags, stable install names, and recorded executable hashes:

### An unchanged client and a replaced library

Build this as module `E8Visibility`, with `-enable-library-evolution -O` and binary/textual module output. It is version 1:

```swift
@export(interface)
public func libraryRevision() -> Int { 1 }

@export(implementation) @inline(never)
public func clientRevision() -> Int { 1 }

@inlinable
public func visibleRevision() -> Int { 1 }

@inline(always)
public func forcedRevision() -> Int { 1 }
```

Build this client separately with `-O`:

```swift
import E8Visibility

@main struct Client {
    static func main() {
        print(libraryRevision(), clientRevision(),
              visibleRevision(), forcedRevision())
    }
}
```

Make version 2 by changing all four returned literals from 1 to 2. Preserve the module name, dynamic-library install name, and compatible signatures. Replace only the dylib and start a new process; this is not live hot-swapping.

| Library / client | Observed output in the earlier recorded macOS builds |
| --- | --- |
| v1 / original optimized executable | `1 1 1 1` |
| v2 / identical original executable | `2 1 1 1` |
| v2 / rebuilt executable | `2 2 2 2` |

The executable hash is unchanged during replacement. The emitted interface hides `libraryRevision`'s body and exposes the other three. Exported-symbol inspection finds `libraryRevision`, `visibleRevision`, and `forcedRevision`, but no `clientRevision` entry point. The `@inline(never)` function still retains 1 because its definition belongs to the old client. These are observations for this fixture and settings; do not predict every `@inlinable` call will always be inlined.

**Reasoning:** The four functions deliberately separate library calls, client emission, exposed optimization, and mandatory direct inlining. Runtime output must be interpreted alongside interface and symbol evidence, because the same value alone does not identify which mechanism produced it.

## In practice

**Choose it when:** Set binary-library visibility policy around independent client rebuild schedules.

**Investigate by:** Inspect callable symbols, interface bodies, and unchanged-client output separately.

**Strong evidence:** Justify each exposed helper/body and validate arithmetic domain, compatibility, and measured performance.

### Helper visibility, arithmetic, and specialization — optional

#### A private dependency in an exposed body

Expected rejection, checked in all three compilation configurations:

```swift
private func clamp(_ value: Int) -> Int {
    max(0, min(value, 100))
}

@inlinable public func clamped(_ value: Int) -> Int {
    clamp(value) // Expected rejection: private helper in an exposed body.
}
```

The compiler reports that `clamp` is private and cannot be referenced from an `@inlinable` function. Either keep the caller opaque, or deliberately make the helper usable from inlineable code as in the optional helper redesign below. Making it public is appropriate only if clients should actually use it.

This attempted shortcut is also rejected:

```swift
@usableFromInline
private func clamp(_ value: Int) -> Int {
    max(0, min(value, 100))
}
```

The diagnostic says `@usableFromInline` applies to internal or package declarations, not this private function. These are fresh Swift 6.4 checks; exact source locations and diagnostic formatting are not API contracts.

#### Review an overexposed utility API

This compiles, but mixes product policy with a stable arithmetic operation:

```swift
public import Foundation

public enum UtilityKit {
    @inlinable public static func normalizedUsername(_ value: String) -> String {
        value.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
    }

    @inlinable public static func analyticsEventName(_ value: String) -> String {
        "ios_v2_" + value.lowercased()
    }

    @inlinable public static func clampedPercentage(_ value: Int) -> Int {
        max(0, min(value, 100))
    }
}
```

Decide which behavior must change for already-built clients when the library changes. Explain the dependency implications of the exposed Foundation calls. Do not assume this illustrative username transformation is an appropriate account-identity or Unicode normalization policy.

A candidate revision hides the two policy bodies. The arithmetic body remains exposed only to demonstrate the ABI-visible helper tradeoff; remove that annotation too unless measurements justify it.

```swift
internal import Foundation

public enum UtilityKit {
    @export(interface)
    public static func normalizedUsername(_ value: String) -> String {
        value.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
    }

    @export(interface)
    public static func analyticsEventName(_ value: String) -> String {
        eventPrefix() + value.lowercased()
    }

    @inlinable public static func clampedPercentage(_ value: Int) -> Int {
        clampPercentage(value)
    }
}

private func eventPrefix() -> String { "ios_v2_" }

@usableFromInline
internal func clampPercentage(_ value: Int) -> Int {
    max(0, min(value, 100))
}
```

The resilient interface contains the two policy signatures without their bodies or Foundation import. It contains the clamp body and a source-internal helper declaration. An ordinary external client can use the public API but cannot name `clampPercentage`. The helper has an exported symbol in the checked library. Changing its implementation can update calls that still reach it; it does not recover any policy already copied by previously shipped exposed bodies.

#### Define the arithmetic domain before considering optimization

A storage-dependent `isFull` body can commit clients to a representation. A formula such as `(index + 1) % capacity` also hides two bugs: zero capacity and overflow before remainder. A complete checked index operation is a clearer exercise than an incomplete ring buffer:

```swift
public enum IndexFailure: Error, Equatable {
    case invalidCapacity
    case outOfBounds
}

@inlinable
public func nextIndex(after index: Int, capacity: Int)
    throws(IndexFailure) -> Int {
    guard capacity > 0 else { throw .invalidCapacity }
    guard index >= 0, index < capacity else { throw .outOfBounds }
    return index == capacity - 1 ? 0 : index + 1
}
```

The contract accepts positive capacity and `0 <= index < capacity`. At the last index it returns zero; otherwise incrementing is safe. Capacity `Int.max` needs no allocation and the arithmetic still works. The error cases and exposed algorithm are compatibility commitments. A real queue needs separate occupancy, insertion/removal, and optional-element semantics; this function does not implement a queue.

#### Keep a generic algorithm inside the library

Add this to `E8Utilities` with the revised utility and index files:

```swift
@specialized(where S == [Int])
public func containsZero<S: Sequence>(_ values: S) -> Bool
    where S.Element: BinaryInteger {
    for value in values {
        if value == 0 { return true }
    }
    return false
}
```

The optimized SIL contains an exact `S` versus `Array<Int>` metatype check, an array-specific path, and the generic iterator/witness path. The textual interface retains the signature and specialization annotation without the algorithm body. This demonstrates prespecialization structure, not a speedup claim.

#### Exercise the external contract

This is an ordinary client of `E8Utilities`, not a testable import:

```swift
import E8Utilities

@main struct Checks {
    static func main() throws {
        precondition(UtilityKit.normalizedUsername("  ADA\n") == "ada")
        precondition(UtilityKit.analyticsEventName("OPEN") == "ios_v2_open")
        for (input, expected) in [(Int.min, 0), (-1, 0), (0, 0),
                                  (37, 37), (100, 100), (Int.max, 100)] {
            precondition(UtilityKit.clampedPercentage(input) == expected)
        }
        let one = try nextIndex(after: 0, capacity: 1)
        let wrapped = try nextIndex(after: Int.max - 1, capacity: Int.max)
        let advanced = try nextIndex(after: Int.max - 2, capacity: Int.max)
        precondition(one == 0 && wrapped == 0 && advanced == Int.max - 1)
        for (index, capacity, expected) in [
            (0, 0, IndexFailure.invalidCapacity),
            (0, -1, IndexFailure.invalidCapacity),
            (-1, 3, IndexFailure.outOfBounds),
            (3, 3, IndexFailure.outOfBounds),
            (Int.max, Int.max, IndexFailure.outOfBounds)
        ] {
            do {
                _ = try nextIndex(after: index, capacity: capacity)
                preconditionFailure("Expected rejection")
            } catch {
                precondition(error == expected)
            }
        }
        precondition(containsZero([1, 0, 2]))
        precondition(!containsZero([Int]()))
        precondition(!containsZero([1, 2, 3]))
        precondition(containsZero([Int8(1), 0]))
        precondition(containsZero((-2...2)))
        print("utility boundaries passed")
    }
}
```

It prints `utility boundaries passed` in both macOS settings. The cases cover clamp extremes, capacity one, the largest valid indices, invalid domains, empty/no-match arrays, a different integer element type, and a range. The two client sources also compile for the iOS deployment target; iOS runtime behavior is not tested here.

## References

### Related guides

- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[E7 — Resilience, ABI, and Module Stability]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [SE-0497](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)
- [SE-0193 — Cross-module inlining and specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)
- [SE-0496 — `@inline(always)`](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)
- [SE-0460 — Explicit specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0460-specialized.md)
- [Library evolution in Swift](https://www.swift.org/blog/library-evolution/)
