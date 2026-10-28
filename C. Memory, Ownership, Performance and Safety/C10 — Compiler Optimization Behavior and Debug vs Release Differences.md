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

**Learning goals:**

- Record optimization, assertion policy, source conditions, language/isolation, SDK, and target separately.
- Keep required effects and recoverable failure in ordinary control flow and verify equivalent behavior across builds.
- Distinguish mandatory inlining, body visibility, specialization, emitted artifacts, and real runtime measurements.

**Prerequisites:** [[A13 — Error handling and typed throws]] for recoverable errors, [[B11 — Library Evolution and Resilience-Aware API Design]] for exported bodies/ABI, [[C3 — Exclusivity enforcement and `inout`]] for dynamic enforcement, and [[C9 — Existential, generic, and allocation performance model]] for equivalent performance work.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A startup works in a debug build but silently skips preparation in release because the work was hidden inside assert. That is a configuration-dependent API effect, not evidence that optimization randomly broke valid Swift.

Optimization can change inlining, ARC traffic, checks, and debugger visibility while preserving well-defined behavior. Some APIs explicitly depend on configuration: assert’s autoclosure normally stops evaluating under -O/-Osize, while ordinary precondition still evaluates. Assertion configuration can override those defaults. Treat optimization level, assertion policy, and source conditions such as DEBUG as separate inputs. [assert](https://developer.apple.com/documentation/swift/assert(_:_:file:line:))

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Observe assertion evaluation without executing a failed contract

Shared assertionProbe/preconditionProbe setup. Every condition is true; the counters deliberately reveal evaluation and are not a pattern for required work.

```swift
func assertionProbe() -> Int {
    var checks = 0
    func valid(_ value: Int) -> Bool {
        checks += 1
        return value >= 0
    }
    for value in 0..<3 { assert(valid(value)) }
    return checks
}
func preconditionProbe() -> Int {
    var checks = 0
    func valid(_ value: Int) -> Bool {
        checks += 1
        return value >= 0
    }
    for value in 0..<3 { precondition(valid(value)) }
    return checks
}
print(assertionProbe(), preconditionProbe())
```

**Walkthrough:** The baseline default -Onone build prints 3 3. The validation compiles the exact source under the configurations below and checks output externally, so an unchecked build cannot remove the harness’s own assertions.

| Effective configuration | assert counter | precondition counter |
| --- | --- | --- |
| -Onone, default | 3 | 3 |
| -O or -Osize, default | 0 | 3 |
| -Ounchecked, default | 0 | 0 |
| -Onone -assert-config Release | 0 | 3 |
| -O -assert-config Debug | 3 | 3 |

These are valid true-condition controls. A violated unchecked assumption can produce undefined behavior; no failed unchecked contract is executed. Required effects belong outside both autoclosures when they must survive every supported configuration.

### Shared startup setup

Cache is a deterministic preparation model, not a disk cache. It records one attempt and the requested success result. start reports recoverable failure through an ordinary error path.

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
```

## Rules and pitfalls

### Required preparation must not live inside an assertion

**Common pitfall:** Calling preparation only from assert. Prepend startup setup; this mistake tests only successful preparation so no assertion fails.

**Mistake:** Standalone runnable example.

```swift
let cache = Cache()
assert(cache.prepare(succeed: true))
print(cache.attempts, cache.ready)
```

**Consequence:** The default debug build prints 1 true, while ordinary -O prints 0 false because preparation was never called.

**Why:** assert receives an autoclosure whose evaluation depends on assertion configuration. Moving the effect into precondition would still depend on unchecked assertion policy.

**Correction:** Execute preparation in ordinary control flow and use the diagnostic check only for an already established invariant. Prepend startup setup.

```swift
let cache = Cache()
try start(cache, succeed: true)
precondition(cache.attempts == 1 && cache.ready)
print(cache.attempts, cache.ready)
```

**What changes:** Both -Onone and -O print 1 true. start’s guard performs the work and can throw; its assert only inspects state after success. The exercise verifies ordinary failure independently of assertions.

### Onone does not define the DEBUG source condition

**Common pitfall:** Inferring optimization from a #if DEBUG branch without knowing whether -D DEBUG was supplied.

**Mistake:** Standalone runnable example.

```swift
#if DEBUG
    let label = "debug"
#else
    let label = "release"
#endif
print(label)
```

**Consequence:** The baseline -Onone build prints release because DEBUG is absent.

**Why:** Custom source conditions are build inputs. They are not derived automatically from the optimization level or a target’s user-facing configuration name.

**Correction:** Use a source condition for its actual declared purpose and supply it explicitly. This example adds -D DEBUG to the baseline command.

```swift
#if DEBUG
    let diagnosticLogging = true
#else
    let diagnosticLogging = false
#endif
print("diagnostic logging:", diagnosticLogging)
```

**What changes:** The flag-enabled example prints diagnostic logging: true under either -Onone or -O. Record both flags when reproducing a configuration-dependent failure; language mode, isolation, architecture, and SDK are separate too.

## Worked example

### Recoverable failure remains observable when diagnostics are disabled

**Setup:** Prepend startup setup. One success and one failure use ordinary control flow.

**Predict:** Will -O remove the failed preparation attempt or turn it into success?

```swift
let cache = Cache()
try start(cache, succeed: true)
do {
    try start(cache, succeed: false)
    preconditionFailure("Expected failure")
} catch StartupFailure.preparation {}
precondition(cache.attempts == 2 && !cache.ready)
print(cache.attempts, cache.ready)
```

**Outcome:** Both ordinary debug and optimized builds print 2 false.

**Why:** Preparation executes before the success invariant check. Failure throws from the guard regardless of assert configuration. The two attempts are real modeled work; the assertion is optional diagnostic evaluation.

## Interview questions

### Why can debug timings misrepresent a shipped workload without fixing algorithmic complexity?

**Answer:** Unoptimized code can retain ARC/check/indirection/loop costs that a shipped build removes, changing relative timings. That does not grant a better complexity contract. Compare algorithms and equivalent results separately from the optimized machine implementation.

**Example:** Standalone runnable example.

```swift
func matches(_ queries: [Int], in values: [Int], comparisons: inout Int) -> [Bool] {
    queries.map { query in
        values.contains { value in
            comparisons += 1
            return query == value
        }
    }
}
let queries = [4, 3, 2, 1]
let values = [1, 2, 3, 4]
var comparisons = 0
let repeatedScan = matches(queries, in: values, comparisons: &comparisons)
let membership = Set(values)
let hashed = queries.map { membership.contains($0) }
precondition(repeatedScan == [true, true, true, true] && hashed == repeatedScan)
precondition(comparisons == 10)
```

**Walkthrough:** The repeated-scan algorithm performs ten instrumented source comparisons here. Set construction changes the algorithm and adds its own hashing/storage cost. The logged count is not a CPU instruction count or benchmark.

**Boundary:** Use representative data and the shipped flags for timing/allocation questions. Account for preprocessing, hash quality, result consumption, and module boundaries; no universal Set speedup follows from this small fixture.

### When does assert evaluate, how can that be overridden, and how should preparation failure be modeled?

**Answer:** Effective assertion configuration controls evaluation, not just the usual optimization default. Required work must run separately, and recoverable failure should throw or return an explicit outcome in every supported build.

**Example:** Prepend startup setup.

```swift
func attempt(_ succeed: Bool) -> (Int, Bool, Bool) {
    let cache = Cache()
    do {
        try start(cache, succeed: succeed)
        return (cache.attempts, cache.ready, true)
    } catch {
        return (cache.attempts, cache.ready, false)
    }
}
let success = attempt(true)
let failure = attempt(false)
precondition(success.0 == 1 && success.1 && success.2)
precondition(failure.0 == 1 && !failure.1 && !failure.2)
```

**Walkthrough:** Both paths perform one preparation. Success and failure remain distinct under -Onone/-O and the separately tested assertion overrides. The initial counter matrix shows why assert itself must not supply that work.

**Boundary:** A failing precondition remains a programmer-contract trap in ordinary -O/-Osize. That is different from this recoverable failure path and from undefined behavior after an unchecked assumption is violated.

### What do inline(always) diagnostics and mandatory-pass SIL establish, and which runtime/ABI claims need other evidence?

**Answer:** Diagnostics establish accepted attribute/body combinations. Mandatory-pass SIL can establish that a named eligible direct call was inlined. It does not establish an app speedup, every dynamic-call target, or old-client ABI compatibility.

**Example:** Standalone runnable example.

```swift
@inline(always)
func complement(_ value: UInt8) -> UInt8 { 255 - value }
@inline(never)
public func complementProbe(_ value: UInt8) -> UInt8 { complement(value) }
for value in UInt8.min...UInt8.max { precondition(complementProbe(value) == 255 - value) }
```

**Walkthrough:** All inputs produce the bounded complement. The validation emits SILGen and mandatory/optimized SIL: the initial direct reference to complement disappears from complementProbe after the mandatory pass, where the subtraction is visible. [Mandatory inlining](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)

**Boundary:** The non-underscored control predates 6.4 and is stronger than @inline(__always). The no-inline wrapper is an observation point. Dynamic function values and witnesses are separate; client body visibility and specialization require their own evidence.

Three standalone **intentionally rejected** attribute cases, object-compiled so mandatory-pass diagnostics run:

```swift
internal func helper() -> Int { 1 }
@inline(always) public func exposed() -> Int { helper() }
```

```swift
class Base { @inline(always) func value() -> Int { 1 } }
```

```swift
@inline(always) func recursive(_ value: UInt8) -> UInt8 {
    value == 0 ? 0 : recursive(value - 1)
}
```

The older hint accepts the ordinary internal helper; the two spellings are not a mechanical migration. This source is compile-only:

```swift
internal func helper() -> Int { 1 }
@inline(__always) public func hinted() -> Int { helper() }
```

| Control | Question it addresses |
| --- | --- |
| @inline(always) / @inline(never) | Must/must not this eligible call inline? |
| @inlinable | Can a client inspect/copy this body and its ABI-visible dependencies? |
| @export(interface) | Is there a callable library symbol with the body hidden from clients? |
| @export(implementation) | Is the implementation supplied for client emission? |
| @specialized(where ...) | Should selected internal specializations and generic redispatch be emitted? |

[[B11 — Library Evolution and Resilience-Aware API Design|B11]]’s current separate-module/interface/symbol/replacement experiments establish its specific exported-body outcomes and helper-eligibility limitation. They are distinct evidence, not consequences inferred from complementProbe. [Definition visibility](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)

### Extension: Does inlining or a scoped callback prevent an unsafe pointer from escaping?

**Answer:** No. Neither changes the unsafe pointer’s ownership/lifetime contract. A generic callback result can still be an address whose permitted access scope has ended; optimization is not a lifetime proof.

**Example:** Compile-only; the invalid escaped function is never invoked.

```swift
@inline(always)
func withBytes<R>(_ bytes: [UInt8], _ body: (UnsafeBufferPointer<UInt8>) -> R) -> R {
    bytes.withUnsafeBufferPointer(body)
}
func escaped(_ bytes: [UInt8]) -> UnsafePointer<UInt8> {
    withBytes(bytes) { $0.baseAddress! }
}
```

**Walkthrough:** The declaration compiles, but escaped returns an unsupported borrowed address and is never called. For empty input its force would also fail. The safe alternative below returns an owned optional byte.

**Boundary:** Compile-only review example. A callback’s nonescaping closure lifetime does not constrain an escapable UnsafePointer result. [[C6 — Unsafe pointers, buffers, and raw memory|C6]]/[[C11 — Strict Memory Safety Mode and Safe Systems Programming|C11]]/[[C12 — `Span`, `InlineArray`, and modern low-level abstractions|C12]] explain scoped processing, safe views, and auditable native retention.

Prepend withBytes/escaped declarations without calling escaped. This corrected API returns an owned optional scalar, including empty input:

```swift
func firstByte(_ bytes: [UInt8]) -> UInt8? { withBytes(bytes) { $0.first } }
precondition(firstByte([]) == nil)
precondition(firstByte([0]) == 0 && firstByte([255]) == 255)
```

## Focused exercise

**Task:** Move required preparation out of an assertion and compare valid success/failure behavior under matching debug and optimized settings.

**Acceptance checks:**

- -Onone and -O both print 2 false.
- Preparation executes exactly once per attempted start.
- A failed preparation remains an ordinary error independently of assert evaluation.

**Solution:** Prepend startup setup. The validation also runs valid unchecked/assertion-override controls using external output checks:

```swift
let cache = Cache()
try start(cache, succeed: true)
precondition(cache.ready && cache.attempts == 1)
do {
    try start(cache, succeed: false)
    preconditionFailure("Expected failure")
} catch StartupFailure.preparation {}
precondition(!cache.ready && cache.attempts == 2)
print(cache.attempts, cache.ready)
```

**Reasoning:** The guard owns the preparation effect; assert only checks already computed success. Reproduction records optimization, assertion configuration, source definitions, language/isolation, target, and SDK. No invalid unchecked contract or unsafe pointer path is executed.

## In practice

**Choose it when:** Investigate a configuration-dependent failure with a fixed semantic contract and the actual effective build settings.

**Investigate by:** Reduce inputs while preserving flags/dependencies, inspect the relevant compiler stage, and validate required effects and errors independently of diagnostic autoclosures.

**Strong evidence:** Verify the semantic repair under matching shipped conditions, keeping algorithm analysis, emitted implementation, and measured runtime behavior separate.

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

- [assert](<https://developer.apple.com/documentation/swift/assert(_:_:file:line:)>)
- [SE-0496](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)
- [SE-0497](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)
- [Swift compiler optimization guidance](https://github.com/swiftlang/swift/blob/main/docs/OptimizationTips.rst)
- [SE-0460: Explicit specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0460-specialized.md)
- [SE-0193: Cross-module inlining and specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)
