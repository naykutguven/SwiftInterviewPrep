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

**Learning goals:**

- Distinguish source naming, callable library symbols, client-visible bodies, mandatory inlining, and library-side specialization.
- Choose exposed bodies/helpers around real client rebuild and behavior-update requirements.
- Verify unchanged/rebuilt consumers, interfaces and symbols, and define arithmetic correctness before optimization.

**Prerequisites:** [[B11 — Library Evolution and Resilience-Aware API Design]] and [[E7 — Resilience, ABI, and Module Stability]] for resilience/replacement, [[E6 — Import Visibility and Dependency Leakage]] for import exposure, [[C9 — Existential, generic, and allocation performance model]]/[[C10 — Compiler Optimization Behavior and Debug vs Release Differences]] for specialization/inlining evidence, and [[F3 — Performance investigation habits]] for measurement.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A library fix arrives, but an old executable still uses an implementation copied into it. Giving clients a function body is a compatibility choice: it allows their compiler to emit or optimize that body, and replacing the library cannot necessarily update those client copies.

Keep **source access** (what callers can name), **symbol export** (callable library entries), and **body visibility** (what client compilers can use) separate. @inlinable exposes a body; it does not require every call to inline. @export(implementation) can emit a client definition even with @inline(never). [Definition visibility](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Four controls put behavior in different places

Shared E8Visibility v1 library. Compile with the baseline plus -parse-as-library -O -enable-library-evolution -emit-module -c; emit its textual interface. Replacement controls build a dylib from the same source with a stable @rpath install name. Version 2 changes exactly these four returned literals from 1 to 2, with compatible signatures/module name.

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

**Walkthrough:** libraryRevision has a callable library implementation without an exported body. clientRevision supplies a client-emitted definition and explicitly avoids inlining. visibleRevision gives the compiler an optimization-visible body while retaining a library entry. forcedRevision mandates eligible direct inlining. These controls arrived before 6.4; they describe different commitments.

### Specialization need not expose the generic algorithm

Shared containsZero source for E8Utilities. The chosen specialization is [Int], while the public generic contract still accepts Sequences of BinaryInteger elements. Keep this as a declaration-only source for the later library assembly.

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

**Walkthrough:** The library may build a selected concrete path and retain a generic fallback. This does not add new Sequence semantics or prove a faster caller. The question below tests multiple input types and inspects the emitted interface/SIL separately.

## Rules and pitfalls

### Remainder does not make an increment safe

**Common pitfall:** Using (index + 1) % capacity without validating capacity or overflow. Compile-only; invalid arithmetic paths are never executed.

**Mistake:** Compile-only.

```swift
func invalidNext(after index: Int, capacity: Int) -> Int { (index + 1) % capacity }
```

**Consequence:** Zero capacity divides by zero, and index + 1 can overflow before remainder is computed.

**Why:** The expression has hidden input requirements. An optimization attribute cannot validate or repair them.

**Correction:** Define the complete domain and choose the wrap case before incrementing. Shared nextIndex setup.

```swift
public enum IndexFailure: Error, Equatable {
    case invalidCapacity
    case outOfBounds
}

@inlinable
public func nextIndex(after index: Int, capacity: Int)
    throws(IndexFailure) -> Int
{
    guard capacity > 0 else { throw .invalidCapacity }
    guard index >= 0, index < capacity else { throw .outOfBounds }
    return index == capacity - 1 ? 0 : index + 1
}
```

**What changes:** For positive capacity and 0 <= index < capacity, the last index wraps to zero; otherwise the increment is representable. This works even at Int.max without allocating a buffer. The error cases and any exposed body remain API commitments.

### A private helper cannot be referenced from an exposed public body

**Common pitfall:** Exposing a function body while retaining an inaccessible implementation dependency.

**Mistake:** Compile-only; intentionally rejected.

```swift
private func clamp(_ value: Int) -> Int {
    max(0, min(value, 100))
}

@inlinable public func clamped(_ value: Int) -> Int {
    clamp(value)  // Expected rejection: private helper in an exposed body.
}
```

**Consequence:** The compiler rejects the private helper reference from @inlinable.

**Why:** Clients must be allowed to compile every referenced declaration in an exposed body. Public signature types alone are insufficient.

**Correction:** Use an internal ABI-visible helper only when the exposed caller is a deliberate commitment; otherwise leave the caller opaque.

```swift
@usableFromInline internal func clamp(_ value: Int) -> Int { max(0, min(value, 100)) }
@inlinable public func clamped(_ value: Int) -> Int { clamp(value) }
precondition(clamped(-1) == 0 && clamped(101) == 100)
```

**What changes:** @usableFromInline keeps source access internal while making the helper an ABI dependency. It must remain available for old clients that reference it; making the helper public would add an unnecessary naming contract.

The attempted private annotation is independently **intentionally rejected**; usableFromInline applies to internal/package declarations:

```swift
@usableFromInline
private func clamp(_ value: Int) -> Int {
    max(0, min(value, 100))
}
```

### Client-visible policy can outlive a library policy update

**Common pitfall:** Marking every convenience method @inlinable, including username and event-name policy.

**Mistake:** Compile-only.

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

**Consequence:** Already-built clients may retain those policy bodies and Foundation dependencies after the library changes.

**Why:** Exposed bodies can be copied/optimized into clients. That is different from a normal call reaching replaceable library code.

**Correction:** Keep policy bodies opaque; expose the small arithmetic body only if its optimization/ABI tradeoff is justified. Shared E8Utilities source; prepend nextIndex and containsZero declarations when building this module.

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

**What changes:** The interface omits policy bodies and the hidden Foundation import, but exposes the clamp body/helper declaration. This illustrative username transformation is not a production Unicode/account-identity policy. Previously shipped client copies are not retroactively removed by hiding a later version’s body.

## Worked example

### An unchanged client keeps three old definitions

**Setup:** Compile this optimized ordinary client against E8Visibility v1. The direct block check links its object; the replacement control uses the exact source with a dylib/rpath build.

**Predict:** Does @inline(never) guarantee a call reaches the new dylib?

```swift
import E8Visibility

@main struct Client {
    static func main() {
        print(
            libraryRevision(), clientRevision(),
            visibleRevision(), forcedRevision())
    }
}
```

**Outcome:** V1 gives 1 1 1 1. After compatible library replacement, the identical executable gives 2 1 1 1; rebuilding against v2 gives 2 2 2 2.

**Why:** The ordinary library entry updates. The client-emitted no-inline definition, optimized visible body, and mandatory-inlined body retain their checked old values. Interfaces/symbols distinguish mechanisms that runtime output alone cannot identify.

## Interview questions

### Why does replacing the dylib update an interface-exported function but leave the client-emitted function unchanged?

**Answer:** The first reaches a callable library implementation; the second has a definition emitted into the original executable. Preventing inlining does not move that definition back to the library.

**Example:** Ordinary optimized v1 client; compare the separately preserved replacement executable.

```swift
import E8Visibility

print(libraryRevision(), clientRevision(), visibleRevision(), forcedRevision())
```

**Walkthrough:** The direct v1 check prints four ones. The preserved worked executable’s replacement result is 2 1 1 1, and its hash remains unchanged. Symbol inspection finds the library/interface entries but no library clientRevision entry.

**Boundary:** These are selected-source/compiler observations, not proof that every @inlinable call always inlines. Runtime behavior, interface visibility, and symbol ownership must be checked together.

### When is @usableFromInline preferable to a public helper, and when should the caller stay opaque?

**Answer:** Use it for a deliberate ABI-visible implementation helper that clients should not name in source. Keep the public caller opaque when behavior updates and implementation freedom outweigh measured client optimization.

**Example:** Ordinary external E8Utilities client; no testable import.

```swift
import E8Utilities

precondition(UtilityKit.normalizedUsername("  ADA\n") == "ada")
precondition(UtilityKit.analyticsEventName("OPEN") == "ios_v2_open")
precondition(UtilityKit.clampedPercentage(-1) == 0)
precondition(UtilityKit.clampedPercentage(101) == 100)
```

**Walkthrough:** The public client can call the intended utilities but cannot name clampPercentage. The interface retains the helper declaration for exposed code; the checked library exports that helper’s symbol.

**Boundary:** Changing a helper can update calls that still reach it, but cannot recover policy already copied by old clients. Keep stable arithmetic exposed only with evidence; avoid publishing helpers solely to satisfy the compiler.

An ordinary client naming the source-internal helper is **intentionally rejected**:

```swift
import E8Utilities

let invalid = clampPercentage(50)
```

### How can prespecialization retain a generic API without exposing its implementation?

**Answer:** The library can build selected specializations and redispatch internally while preserving a generic signature and fallback. Clients need not receive the algorithm body or a separate specialized source API.

**Example:** Ordinary optimized E8Utilities client.

```swift
import E8Utilities

precondition(containsZero([1, 0, 2]))
precondition(!containsZero([Int]()))
precondition(!containsZero([1, 2, 3]))
precondition(containsZero([Int8(1), 0]))
precondition(containsZero(-2...2))
```

**Walkthrough:** The same public generic accepts the selected [Int] case, another integer type, and a range. Optimized library SIL separately shows the exact-type test, concrete array path, and generic witness fallback. The interface retains the signature/annotation without the body.

**Boundary:** This establishes emitted structure and equivalent semantics, not a speedup. Match module/optimization settings and representative caller types before measuring. [Explicit specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0460-specialized.md)

### Extension: How is mandatory inline(always) different from body export and an optimization hint?

**Answer:** It requires eligible direct calls to inline, whereas body export controls where a definition is available/emitted and the older underscored spelling is an optimizer hint. Dynamic targets and ABI compatibility are separate questions.

**Example:** Standalone runnable example.

```swift
@inline(always) func complement(_ value: UInt8) -> UInt8 { 255 - value }
@inline(never) public func complementProbe(_ value: UInt8) -> UInt8 { complement(value) }
for value in UInt8.min...UInt8.max { precondition(complementProbe(value) == 255 - value) }
```

**Walkthrough:** All byte inputs preserve the bounded result. Separate SILGen versus mandatory/optimized SIL controls identify whether the eligible direct helper call was removed; a function name or output alone does not identify inlining.

**Boundary:** Mandatory inlining does not establish a faster app or a safe private helper in a public exposed body. [[C10 — Compiler Optimization Behavior and Debug vs Release Differences|C10]]/[[B11 — Library Evolution and Resilience-Aware API Design|B11]] retain their separate visibility and ABI evidence. [Mandatory inlining](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)

## Focused exercise

**Task:** Compare a callable-library function with a client-visible definition across compatible replacement; review arithmetic and helper boundaries with an ordinary consumer.

**Acceptance checks:**

- The identical original executable runs with v1/v2 and retains the documented client values.
- Rebuilding yields the new values; interface/symbol inspection explains the difference.
- Public utility clients check clamp extremes, index domain/boundaries and multiple generic inputs without naming helpers.

**Solution:** Preserve the earlier E8Visibility client and change only the four returned literals in a separate v2 library. Use [[E7 — Resilience, ABI, and Module Stability|E7]]’s matching module/install-name/rpath build pattern, hash the executable before/after, and rebuild into a different output. The complete ordinary utility client below exercises the separate E8Utilities sources:

```swift
import E8Utilities

@main struct Checks {
    static func main() throws {
        precondition(UtilityKit.normalizedUsername("  ADA\n") == "ada")
        precondition(UtilityKit.analyticsEventName("OPEN") == "ios_v2_open")
        for (input, expected) in [
            (Int.min, 0), (-1, 0), (0, 0),
            (37, 37), (100, 100), (Int.max, 100),
        ] {
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
            (Int.max, Int.max, IndexFailure.outOfBounds),
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

**Reasoning:** Int.min/Int.max clamp inputs, capacity one, Int.max capacity and invalid indices test correctness independently of optimizer choices. Equal generic results do not measure speed. The compatible v2 replacement tests body ownership rather than changing public signatures or running an incompatible layout.

## In practice

**Choose it when:** Choose callable entries and exposed definitions around actual client rebuild schedules and performance evidence.

**Investigate by:** Inspect interfaces, symbols, helper eligibility and unchanged/rebuilt outputs; test domain boundaries before optimizing.

**Strong evidence:** Keep source access, client emission, mandatory inlining, library specialization and measured runtime costs distinct.

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
