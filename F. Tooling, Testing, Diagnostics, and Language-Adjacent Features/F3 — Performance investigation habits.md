---
tags:
  - swift
  - ios
  - interview-prep
  - tooling
  - performance
  - instruments
---

# F3 — Performance Investigation Habits

**Goal:** Measure one equivalent optimization in matching release builds and report its scope.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Start with a reproducible user-visible scenario, establish a baseline, instrument the relevant cost, change one thing, then verify behavior and regression policy. Wall time, CPU samples, waits, allocations, and retained memory are different quantities. A profile locates work; a benchmark measures a chosen boundary; correctness tests protect behavior. [Improving app performance](https://developer.apple.com/documentation/xcode/improving-your-app-s-performance)

## Rules and pitfalls

### Define the experiment before changing code

Record input size/distribution, warmup, optimization, architecture, SDK, concurrency policy, module boundaries, sample count, and what setup is timed. Use signposts around complete user operations and inspect CPU versus waiting before rewriting abstractions.

**Common pitfall:** A correct checksum and accurate timer can still measure optimized-away work, constant-specialized inputs, or setup unrelated to the hypothesis. Consume results externally and vary representative runtime inputs with equivalent behavior.

### Preserve ownership and task semantics

Comparing Array, UniqueArray, and Span requires the same ownership requirement; an independent snapshot cannot be replaced by a unique owner merely to win a benchmark. Offloading also needs cancellation, joining, and publication freshness.

**Common pitfall:** A passing measurement test without a baseline and threshold is not a regression gate. Define an acceptable distribution or relative change, account for noise, and recheck user-visible behavior before adopting complexity.

## Interview questions

### What would you measure before rewriting an abstraction-heavy path, and how would you distinguish CPU work, contention, and blocking?

Measure the user interval, CPU stacks, allocation traffic, and wait/contended intervals in matching optimized builds. CPU samples show active work; long elapsed time with little CPU suggests waits or blocking. Signposts and event histories connect the interval to its owner. Do not rewrite generic syntax before locating the actual cost.

### How do you compare Array, UniqueArray, and Span without changing the ownership requirement or timing the wrong work?

Hold element operations, input, retained snapshots, and output semantics fixed. Time mutation, conversion, construction, or end-to-end work according to the stated hypothesis and report excluded setup. Unique ownership and borrowed views cannot substitute for a required copyable retained snapshot without changing the comparison.

### Why can correct output and an accurate timer still produce a misleading benchmark?

The benchmark can have unrealistic inputs, insufficient samples, optimizer elimination, mismatched builds, changed effect order, or the wrong timed boundary. A timer does not validate experimental design. Check generated code and external result consumption where relevant, then report uncertainty and representativeness.

### Which user-visible gain justifies the complexity, memory, and API cost of a proposed optimization?

Require a measured gain in an actual latency, throughput, energy, or memory constraint large enough to justify implementation, API, deployment, and maintenance costs. A microbenchmark win is supporting evidence, not an automatic product improvement. Keep the simpler design when the effect is immaterial.

## Focused exercise

**Task:** Define one representative workload, metric, and baseline. Compare one change in matching optimized builds, verify equivalent behavior, and report measurement limits.

**Acceptance checks:**

- Both paths return identical membership results, including misses.
- The candidate interval includes building its lookup storage.
- Collect repeated optimized samples and state memory/setup limitations before claiming a useful gain.

**Solution:**

Standalone macOS 13+ elapsed-time experiment. Workload: 10,000 values and 1,000 membership queries; Set construction is included in the candidate interval. Compile both paths together with -O and baseline flags.

```swift
@available(macOS 13, *)
func experiment() {
    let values = Array(0..<10_000)
    let queries = (0..<1_000).map { $0 * 17 }
    let clock = ContinuousClock()
    var baseline: [Bool] = []
    var candidate: [Bool] = []
    let first = clock.measure { baseline = queries.map { values.contains($0) } }
    let second = clock.measure {
        let lookup = Set(values)
        candidate = queries.map { lookup.contains($0) }
    }
    precondition(baseline == candidate)
    print(baseline.filter { $0 }.count, first, second)
}
if #available(macOS 13, *) { experiment() }
```

**Reasoning:** The candidate changes algorithmic work and adds storage. This one-shot executable is a measurement fixture, not a regression gate or proof of UI benefit. Reverse/interleave sample order, use runtime-derived representative workloads, inspect optimization, and measure memory before an adoption decision.

## In practice

**Choose it when:** A falsifiable performance hypothesis connects to a real product constraint.

**Investigate by:** Combine complete-operation signposts, CPU/wait tools, allocations, and controlled release benchmarks.

**Strong evidence:** Report distribution, equivalent behavior, excluded work, and a defensible regression threshold alongside the user-visible result.

### Storage comparison and earlier experiment evidence — optional

#### Measure Swift 6.4 ownership and optimization choices

UniqueArray supports unique ownership and noncopyable elements; that changes what programs can express. It still uses heap storage and can allocate on growth. Span is a borrowed view, not an alternative owning collection. Borrow/mutate accessors expose stored values with lifetime/exclusivity constraints; they do not guarantee faster end-to-end execution. [SE-0527](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0527-rigidarray-uniquearray.md), [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

These complete kernels make useful controlled experiments. Arithmetic intentionally wraps modulo UInt64's range. UniqueArray requires Apple OS 27 in the checked SDK; constructing an Array's span requires 26, though the core indexed Span helper below can compile at earlier deployment targets.

```swift
public enum StorageWork {
    public static func updateArray(_ values: inout [UInt64], salt: UInt64) -> UInt64 {
        var checksum: UInt64 = 0
        for index in values.indices {
            values[index] &+= salt
            checksum &+= values[index]
        }
        return checksum
    }

    @available(anyAppleOS 27.0, *)
    public static func updateUnique(_ values: inout UniqueArray<UInt64>, salt: UInt64) -> UInt64 {
        var checksum: UInt64 = 0
        for index in 0..<values.count {
            values[index] &+= salt
            checksum &+= values[index]
        }
        return checksum
    }

    public static func sumArray(_ values: borrowing [UInt64]) -> UInt64 {
        var checksum: UInt64 = 0
        for index in values.indices { checksum &+= values[index] }
        return checksum
    }

    public static func sumSpan(_ values: Span<UInt64>) -> UInt64 {
        var checksum: UInt64 = 0
        for index in 0..<values.count { checksum &+= values[index] }
        return checksum
    }
}
```

The indexed Array sum is deliberate: replacing its body with ordinary `for value in values` attempts to consume the explicitly borrowed parameter in this compiler and rejects. A legal source rewrite and a faster generated program are separate questions. The examples use the shipped Swift APIs; C8 records the differences from proposal-only Containers/RigidArray text.

Compare distinct questions separately:

| Experiment | Hold constant | Interpretation |
| --- | --- | --- |
| Array with one live owner versus UniqueArray | Input, salt sequence, capacity/setup boundary, result checks | A container comparison for this mutation workload |
| Array mutation while an earlier snapshot stays live | Same mutation plus a verified earlier value | Cost under a sharing requirement; UniqueArray without an equivalent snapshot does different work |
| Borrowed Array versus Span read | Same initialized values, same checksum, no mutation during the borrow | Read-path comparison; neither owns a replacement persistent snapshot |
| Onone versus O versus Osize | Compiler, source, module boundaries, data | Configuration-specific generated-code behavior, not a general ranking of Swift features |

The temporary measurement fixture calls these kernels through a separately compiled library without inlinable bodies, validates outputs, and records local elapsed samples. That limits caller-side elimination but also preserves a module boundary and call overhead. It is not equivalent to a whole-module application build. Input creation and unique-buffer construction are outside mutation intervals; shared-snapshot validation happens after each interval. Allocation counts and reference-count traffic require separate evidence.

When considering `@inline(always)`, `@specialized`, `@inlinable`, or exported implementation detail, distinguish optimization requests, compiler feasibility, and API/ABI commitments. Inspect the relevant optimized SIL/assembly only after identifying a concrete question. Count runtime behavior with suitable tools; static retain instructions are not automatically executed retain counts. See [[B11 — Library Evolution and Resilience-Aware API Design]] and [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]].

Combine this with StorageWork above and call it on Apple OS 27 or newer:

```swift
@available(anyAppleOS 27.0, *)
func storageProbe() {
    let original: [UInt64] = [1, 2, 3]
    var ordinary = original
    var unique = UniqueArray<UInt64>()
    for value in original { unique.append(value) }
    let arrayResult = StorageWork.updateArray(&ordinary, salt: 2)
    let uniqueResult = StorageWork.updateUnique(&unique, salt: 2)
    print(original, ordinary, arrayResult, uniqueResult, StorageWork.sumSpan(ordinary.span))
}
```

The output is `[1, 2, 3] [3, 4, 5] 12 12 12`. It proves the illustrated values and original snapshot survive as expected. It does not establish which storage is faster, how many allocations occurred, or whether a real feed will hitch less. That distinction is the interview probe: identify the missing evidence before recommending a rewrite.

A feed hitches while scrolling and applying a filter. A proposed fix replaces all arrays with UniqueArray and starts one detached task per row.

1. Define a reproducible device/build/data/cache scenario and the metric that will determine success. Include a sustained run and enough events to support the reported distribution.
2. Mark preparation, publication/diffing, image decode, and cell/layout work separately. Capture an end-to-end interval too; a fast preparation kernel can still feed a slow UI update.
3. Select a hitch. Check executing CPU stacks, task/executor context, waits, rendering, and allocations. For SwiftUI, inspect update frequency and Cause & Effect dependencies; for UIKit, inspect configuration, reuse, layout, and snapshot application.
4. Form one evidence-based hypothesis. Examples include repeated image decoding, repeated formatting, large diff updates, broad Observation dependencies, shared-buffer mutation, or blocking I/O. Do not label any of these as measured until the trace supports it.
5. Change the implicated path. If offloading preparation, use an explicitly owned task design such as [[D7 — Actor Reentrancy and Logic Races#Focused exercise|the gated publication model]]. Bound parallelism where useful; preserve source identity and test stale success/error, repeated identical queries, cancellation, and reset.
6. Validate equivalent outputs before comparing storage or algorithms. Run a focused benchmark with setup, result consumption, ownership, module boundaries, and optimization documented.
7. Repeat the original device scenario and compare relevant metrics plus memory/energy tradeoffs. Keep raw traces and sample counts. Configure a regression baseline only after establishing stable measurements.

An acceptable report says what was observed, what changed, and which user-visible outcome improved. Invented before/after millisecond numbers, a simulator-only scrolling judgment, or a kernel timing presented as an app result do not satisfy the exercise.

## References

### Related guides

- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F5 — Language mode, build settings, and migration flags]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [Improving app performance](https://developer.apple.com/documentation/xcode/improving-your-app-s-performance)
- [SE-0527](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0527-rigidarray-uniquearray.md)
- [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)
- [Apple: Profile, fix, and verify](https://developer.apple.com/videos/play/wwdc2026/268/)
- [Apple: Optimize SwiftUI performance with Instruments](https://developer.apple.com/videos/play/wwdc2025/306/)
- [Apple: Creating performant scrollable stacks](https://developer.apple.com/documentation/swiftui/creating-performant-scrollable-stacks)
- [Apple: OSSignposter](https://developer.apple.com/documentation/os/ossignposter)
- [Apple: Writing and running performance tests](https://developer.apple.com/documentation/xcode/writing-and-running-performance-tests)
- [Swift.org: Introducing Benchmark](https://www.swift.org/blog/benchmarks/)
- [current Benchmark project](https://github.com/ordo-one/benchmark)
