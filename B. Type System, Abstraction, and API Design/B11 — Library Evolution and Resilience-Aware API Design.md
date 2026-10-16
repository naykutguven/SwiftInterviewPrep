---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
  - library-evolution
  - resilience
---

# B11 — Library Evolution and Resilience-Aware API Design

**Goal:** Check a public enum addition across a separate client and explain the chosen resilience contract.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Source compatibility asks whether a client rebuilds successfully with supported meaning. ABI compatibility asks whether an unchanged binary can use a replacement library. Module stability concerns importing an interface with supported compilers. Behavioral and persisted-schema compatibility are additional contracts. One successful rebuild establishes none of the others automatically. [Library evolution](https://www.swift.org/blog/library-evolution/)

## Rules and pitfalls

### Publish only the representation clients may rely on

Library-evolution mode enables resilient boundaries; @frozen commits layout or enum shape where its rules apply. Private fields still contribute to a frozen struct's layout. Nonexhaustive source checking does not itself enable binary resilience.

**Common pitfall:** Adding a case and recompiling both sides hides what an unchanged client must handle. Test actual external boundaries and keep the old executable unchanged for binary replacement experiments.

### Separate visible bodies, symbols, and specialization

An exposed implementation can be emitted into clients and retain old behavior after library replacement. A normal library call can reach updated code. Source access, body export, mandatory inlining, and library-side specialization are independent decisions.

**Common pitfall:** An internal helper reachable from an exposed body can become a compatibility obligation. Leave the caller opaque when that commitment is not justified; see [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]].

## Interview questions

### Which changes require recompilation, and which must an unchanged client binary tolerate?

Source-only changes require recompilation and review of changed lookup, defaults, and diagnostics. Under an ABI-compatible resilient release, an unchanged binary must tolerate permitted library representation changes while using supported symbols and runtime behavior. Client-emitted bodies can retain earlier behavior, so source and binary tests need separate expectations.

### How do ordinary, resilient, frozen, nonexhaustive, and nonexhaustive(warn) enums differ across a real module/package boundary?

Ordinary nonresilient and frozen enums can expose closed shapes to external clients under their rules; resilient public nonfrozen enums require future-case handling. nonexhaustive and nonexhaustive(warn) alter source checking at their defined module/package boundary, with warning behavior distinct from binary resilience. Verify the exact boundary and toolchain rather than treating all public enums alike.

### Why can replacing a library update an interface-backed function while an old client retains other function bodies?

The interface-backed call can still invoke a library symbol whose implementation changed. An exposed body may already reside in the old client, so replacing the library cannot replace those client instructions. Inlining is one route; client emission can matter even without an inlined call.

### Does specialized expose a new specialized ABI, and is inline(always) just another spelling of inline(__always)?

@specialized requests library-side specialization without necessarily exposing a new specialized public ABI. @inline(always) has public body-visibility consequences and is distinct from the older optimizer hint @inline(__always). Inspect supported attributes and emitted interfaces; do not infer distribution policy from their similar names.

## Focused exercise

**Task:** Add a case to a public enum and rebuild a separate consumer. Check fallback requirements and explain source versus binary compatibility.

**Acceptance checks:**

- The v1 ordinary external client has a future-case fallback.
- The v2 rebuild surfaces the newly known case.
- The report distinguishes rebuilding source from running an unchanged binary.

**Solution:**

Use two source files in separate modules. Build the library with -enable-library-evolution plus F5's baseline flags.

```swift
// StatusSDK.swift, v1
public enum Status { case ready, stopped }
public func current() -> Status { .ready }
```

```swift
// Client.swift
import StatusSDK
func label(_ value: Status) -> String {
    switch value {
    case .ready: "ready"
    case .stopped: "stopped"
    @unknown default: "future"
    }
}
precondition(label(current()) == "ready")
```

For v2 add case paused and return it from current(). Rebuilding the unchanged client source warns about its omitted known case and uses the fallback. Add an explicit paused case for the maintained client. Run the old executable with a compatible v2 library only as a separate ABI experiment.

**Reasoning:** @unknown default provides future coverage while retaining a diagnostic for omitted known cases. It can execute for that known paused case too. The full fixture below separates warning-promotion observations from native errors and binary replacement evidence.

## In practice

**Choose it when:** Define release gates for the actual source/binary distribution model.

**Investigate by:** Compare ordinary clients, textual interfaces, and unchanged executables under recorded builds.

**Strong evidence:** Demonstrate both unchanged and rebuilt consumer behavior and identify remaining compiler/platform-matrix limits.

### Separate-module evolution fixtures — optional

Compile this as module `DeliveryAPI`, separate from its consumer:

```swift
@nonexhaustive
public enum DeliveryStatus: Sendable {
    case pending, delivered
}

public func latestStatus() -> DeliveryStatus { .pending }
```

Consumer source:

```swift
import DeliveryAPI

func statusLabel(_ status: DeliveryStatus) -> String {
    switch status {
    case .pending: "pending"
    case .delivered: "delivered"
    @unknown default: "unknown"
    }
}
```

An external consumer without the fallback is rejected. With it, the two known values return their labels. If the library adds a new case, recompiling this source warns that the known case is missing. The already-compiled resilient client can use the fallback when a newer binary library supplies that value.

Do not test this distinction by pasting the enum and switch into one source module: that removes the boundary you are trying to examine.

## References

### Related guides

- [[A11 — Access Control]]
- [[A12 — Availability and Conditional Compilation]]
- [[B9 — Codable and Schema Evolution]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B13 — Property Wrappers and Generated Storage Semantics]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[E7 — Resilience, ABI, and Module Stability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [Library evolution](https://www.swift.org/blog/library-evolution/)
- [SE-0193: Cross-module inlining and specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)
- [SE-0487: Nonexhaustive enums](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md)
- [SE-0497: Definition visibility](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)
- [SE-0496: Inline always](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)
- [SE-0460: Explicit specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0460-specialized.md)
