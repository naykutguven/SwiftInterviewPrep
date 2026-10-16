---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - pattern-matching
  - control-flow
---

# A5 — Pattern Matching and Exhaustive Control Flow

**Goal:** Map every response case while preserving the unguarded remainder of a payload.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A pattern recognizes a shape and may bind values. A switch selects the first matching pattern whose condition holds. Exhaustiveness proves coverage, not business correctness: a catch-all can compile while hiding an unsupported status or a newly added domain case. Use explicit enum cases where changes should trigger review. [Patterns](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/patterns/)

## Rules and pitfalls

### Cover the remainder after a guarded case

A where clause selects a subset of a case's payloads. A success status range does not cover other statuses of the same response case. Add a deliberate remainder policy and test its boundary values.

**Common pitfall:** Treating a guarded enum case as fully covered produces an incomplete switch or tempts an overly broad default. Write an unguarded case for the remaining payloads so future enum cases remain visible.

### Read grouped patterns and fallthrough literally

Each comma-separated pattern has its own where condition. Grouped alternatives must bind compatible names and types. Fallthrough transfers control into the next body without testing its pattern; it does not mean “try the next case.”

**Common pitfall:** Putting a guard only after the last grouped pattern does not constrain every alternative. Repeat the condition or split the cases. Prefer explicit shared helpers to fallthrough when the next pattern's test matters.

### Distinguish external enum checking from evolution

An external nonfrozen enum requires a future-case policy across resilient module boundaries. @unknown default also warns when known cases were omitted. It can still execute for an omitted known case; it is not a runtime filter for future values. See [[B11 — Library Evolution and Resilience-Aware API Design]].

## Interview questions

### Why does this guarded case need remaining payload coverage?

The guard only covers payloads satisfying its condition. For .status(code), a 200..<300 branch leaves every other Int uncovered. Add an unguarded .status branch, then distinguish the domain consequences instead of routing every unknown situation to success.

### Which external enum omission is an error in Swift 6 mode, and which warns while still reaching a fallback?

Omitting the required future-case fallback for an external nonfrozen enum is an error in Swift 6. Covering known cases through @unknown default can warn while still reaching that fallback. Compile with a real separate library: a same-module enum cannot establish the external resilience behavior. Warning promotion must be checked on the selected compiler.

## Focused exercise

**Task:** Map a response enum using guarded status cases. Check success, unexpected status, failure, and cancellation; explain remaining coverage.

**Acceptance checks:**

- Check success range edges and an unexpected status below and above the range.
- Failure and cancellation remain different outcomes.
- Removing the unguarded status branch must make the switch nonexhaustive.

**Solution:**

Standalone synchronous mapping:

```swift
enum Response {
    case status(Int)
    case failed(String)
    case cancelled
}
func message(_ response: Response) -> String {
    switch response {
    case let .status(code) where (200..<300).contains(code): "success"
    case let .status(code): "unexpected \(code)"
    case let .failed(reason): "failed \(reason)"
    case .cancelled: "cancelled"
    }
}
precondition(message(.status(200)) == "success")
precondition(message(.status(299)) == "success")
precondition(message(.status(300)) == "unexpected 300")
precondition(message(.failed("offline")) == "failed offline")
precondition(message(.cancelled) == "cancelled")
```

**Reasoning:** The unguarded status pattern owns the remainder without hiding other enum cases. Adding a new Response case causes a compile-time review. Coverage still does not prove the status policy is correct; assertions establish the chosen mapping.

## In practice

**Choose it when:** Use exhaustive switching for closed domain mappings and selective patterns for event filtering.

**Investigate by:** List each case's payload space, then subtract guarded subsets in source order.

**Strong evidence:** Exercise boundary payloads and verify the fallback against a separately compiled resilient library.

### External enum fixture — optional

Build the following as module SDKStates without library evolution. The attribute makes its case set nonexhaustive to outside consumers:

```swift
@nonexhaustive
public enum SDKStatus: Sendable {
    case ready
    case waiting
}
```

Build this in a separate consumer module, importing that compiled module:

```swift
import SDKStates

func describeStatus(_ status: SDKStatus) -> String {
    switch status {
    case .ready: "ready"
    case .waiting: "waiting"
    @unknown default: "future"
    }
}
```

The consumer compiles and returns ready/waiting for the known cases. Removing only the fallback is rejected in Swift 6 mode for possible additional unknown values. Keeping the fallback but removing .waiting compiles with a known-case warning and returns future for .waiting. Replacing @unknown default with plain default accepts that same omission without the warning.

| Boundary / source change | Checked consequence |
| --- | --- |
| Local closed enum; omit a case with no catch-all | Error: incomplete switch |
| External @nonexhaustive enum; handle known cases but omit fallback | Error in Swift 6 mode |
| Same consumer in Swift 5 mode | Warning for possible unknown values |
| @unknown default but omit a known case | Warning in the tested Swift 6 compiler; fallback still runs |
| Plain default and omit a known case | Accepted without that warning |
| Add a new local Response case | Explicit render must change; lossyRender still compiles |

@nonexhaustive is a Swift 6.2.3 feature, revalidated for this baseline. Its warn form supports staged source migration; it does not enable binary resilience. Same-module/package code is co-developed and has different checking. B11 records checked plain/resilient/frozen/warn/same-package variants and an unchanged client executing a newly added resilient enum case. This chapter's source-only example does not promise an ABI-stable drop-in library replacement. [SE-0487](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md), [SE-0192](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md)

Warnings are review evidence; do not assume a missing known case is a hard error just because the build uses Swift 6. B11 also records a diagnostic-policy limitation in this particular toolchain. Record the real exit status and diagnostics of the gate your project uses.

## References

### Related guides

- [[A4 — Optionals and Nil Modeling]]
- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[A9 — Enums, Associated Values, Recursive Enums, and State Modeling]]
- [[A12 — Availability and Conditional Compilation]]
- [[A13 — Error handling and typed throws]]
- [[A14 — Operators, Literals, and Language Affordances]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [Patterns](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/patterns/)
- [SE-0487](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md)
- [SE-0192](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md)
- [Statements](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/statements/)
