---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - debug-release
  - compiler-optimization
---

# C10 — Compiler Optimization Behavior and Debug vs Release Differences

**Goal:** Move required preparation outside an assertion and compare debug and optimized behavior.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Optimization changes implementation choices such as inlining, ARC traffic, and debugger visibility. It must preserve well-defined program semantics, except where an API explicitly depends on configuration. Assertion autoclosures are one such boundary: assert ordinarily disappears under optimized release settings, while precondition remains in ordinary -O/-Osize. Effective assertion configuration can override those defaults. [assert](https://developer.apple.com/documentation/swift/assert(_:_:file:line:))

## Rules and pitfalls

### Keep required effects in normal control flow

Call preparation before an assertion. Handle recoverable failure with an error or explicit outcome. If a programmer invariant also needs checking, check the already computed result separately.

**Common pitfall:** assert(cache.prepare()) omits preparation when assertions are disabled. Moving it into precondition still makes required work configuration-dependent under unchecked settings. Execute required effects outside both autoclosures.

### Record the full build configuration

-Onone does not itself define DEBUG; source conditions come from build flags such as -D DEBUG. Record optimization, assertion policy, language/isolation settings, architecture, SDK, and module boundaries before comparing failures.

**Common pitfall:** Debug timing exaggerates abstraction overhead but optimization cannot repair a poor asymptotic algorithm. Diagnose release-only failures with matching inputs and a reduced reproducer; keep unsafe undefined behavior outside performance comparisons.

## Interview questions

### Why can debug timings misrepresent a shipped workload without fixing algorithmic complexity?

Debug builds can retain checks, indirect operations, ARC work, and poor loop optimization that optimized builds remove. This can change relative costs substantially without changing an algorithm's asymptotic work. Benchmark the shipped configuration and realistic data; use algorithm analysis to identify growth that optimization cannot remove.

### When does assert evaluate, how can that be overridden, and how should recoverable preparation failure be modeled?

Assert evaluates only in an assertion-enabled configuration, whose effective setting can differ from the usual optimization default. Required work must execute separately. Recoverable preparation failure should return an outcome or throw, preserving failure behavior in every supported build rather than trapping or disappearing with a diagnostic check.

### What do the inline(always) diagnostics and mandatory-pass SIL establish, and which runtime/ABI claims need different evidence?

Diagnostics establish which attribute combinations the compiler accepts; mandatory-pass SIL can show direct inlining for a named call. They do not establish an app speedup, all dynamic-call behavior, or unchanged-client ABI compatibility. Those require measurements and separately compiled/replacement-client experiments.

## Focused exercise

**Task:** Move required work out of an assert autoclosure. Compare debug and optimized behavior while holding input and relevant settings fixed.

**Acceptance checks:**

- Both ordinary debug and optimized builds print 2 false.
- Preparation runs once per attempted start.
- Failure remains an ordinary error path even when assert is disabled.

**Solution:**

Standalone deterministic preparation probe. Build and run with both -Onone and -O, holding all other baseline flags fixed:

```swift
enum StartupFailure: Error { case preparation }
final class Cache {
    private(set) var attempts = 0
    private(set) var ready = false
    func prepare(succeed: Bool) -> Bool {
        attempts += 1
        ready = succeed
        return succeed
    }
}
func start(_ cache: Cache, succeed: Bool) throws {
    guard cache.prepare(succeed: succeed) else { throw StartupFailure.preparation }
    assert(cache.ready)
}
let cache = Cache()
try start(cache, succeed: true)
precondition(cache.ready && cache.attempts == 1)
do { try start(cache, succeed: false); preconditionFailure("expected failure") }
catch StartupFailure.preparation { }
precondition(!cache.ready && cache.attempts == 2)
print(cache.attempts, cache.ready)
```

**Reasoning:** The guard executes preparation independently of diagnostics. Assert merely checks the established success invariant. An unchecked build's assumptions and undefined-behavior risks require separate review; this passing fixture is not a recommendation to ship unchecked code.

## In practice

**Choose it when:** Investigate configuration-dependent behavior with a fixed semantic contract.

**Investigate by:** Reduce the failure while preserving effective flags and dependencies, then inspect optimized output.

**Strong evidence:** Validate a semantic repair in matching release conditions without relying on an optimizer accident.

### Assertion and export controls — optional

#### Assertions evaluate their autoclosures according to configuration

```swift
func assertionProbe() -> Int {
    var checks = 0
    func isValid(_ value: Int) -> Bool {
        checks += 1
        return value >= 0
    }
    for value in 0..<3 { assert(isValid(value)) }
    return checks
}
```

```swift
func preconditionProbe() -> Int {
    var checks = 0
    func isValid(_ value: Int) -> Bool {
        checks += 1
        return value >= 0
    }
    for value in 0..<3 { precondition(isValid(value)) }
    return checks
}
```

These counters deliberately expose evaluation; their side effects are an anti-pattern for required production work. Every tested condition is true, including in unchecked builds.

| Compiler configuration | assertionProbe | preconditionProbe |
| --- | --- | --- |
| Onone, default assertion configuration | 3 | 3 |
| O or Osize, default | 0 | 3 |
| Ounchecked, default | 0 | 0 |
| Onone plus assert-config Release | 0 | 3 |
| O plus assert-config Debug | 3 | 3 |

The selected SDK implements assert/precondition with autoclosures and configuration checks. In ordinary O/Osize builds, a failing precondition traps even though assertions are disabled. Under unchecked configuration, a violated assumed contract can produce undefined behavior; do not demonstrate it by executing an invalid program. Required work belongs outside both assertion and precondition conditions when it must survive every supported configuration.

#### Mandatory inlining is different from implementation visibility

```swift
@inline(always)
func complement(_ value: UInt8) -> UInt8 { 255 - value }

@inline(never)
public func complementProbe(_ value: UInt8) -> UInt8 {
    complement(value)
}
```

Swift 6.3 introduced the non-underscored @inline(always) control. For direct eligible calls it requires inlining, including at Onone; unsupported direct cycles are diagnosed. Dynamically selected function values or witness calls are a separate case: the attribute does not make every runtime target statically resolvable. It is stronger than the older @inline(__always) hint and is not a mechanical spelling replacement. [SE-0496](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)

The checked mandatory/optimized SIL contains the complement arithmetic inside complementProbe without a function_ref to complement. This checks the named direct call, not an overall app speedup. The never attribute keeps an outer observation point for this compiler experiment.

| Control | Primary concern | Review implication |
| --- | --- | --- |
| @inline(always) | Mandatory direct inlining where applicable | Public/package bodies normally inherit inlinable restrictions; explicit export visibility can alter the boundary |
| @inline(never) | Prevent inlining | Does not by itself hide a body from client analysis or specialization |
| @inlinable | Make a body available to clients | Client-generated copies and ABI-visible dependencies affect future changes |
| @export(interface) | Callable library symbol, hidden client body | Distinct from source-level public/private access |
| @export(implementation) | Body supplied for clients to emit | Does not provide the tested library's exported callable entry point |
| @specialized(where ...) | Selected internal specializations and generic redispatch | Does not change source overload resolution or publish a new specialized API |

B11 already verifies these export/inlining combinations with emitted interfaces, symbols, and an unchanged client against a replacement library. It also records a current helper-eligibility limitation: an internal/private export(interface) helper is rejected from an inlinable body in this compiler, despite the broader proposal description. Use that evidence; do not infer every interaction from the attribute's name. E8 covers distribution decisions. [SE-0497](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)

## References

### Related guides

- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[A13 — Error handling and typed throws]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C11 — Strict Memory Safety Mode and Safe Systems Programming]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [assert](https://developer.apple.com/documentation/swift/assert(_:_:file:line:)
- [SE-0496](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)
- [SE-0497](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)
- [Swift compiler optimization guidance](https://github.com/swiftlang/swift/blob/main/docs/OptimizationTips.rst)
- [SE-0460: Explicit specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0460-specialized.md)
- [SE-0193: Cross-module inlining and specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)
