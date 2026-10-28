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

**Learning goals:**

- Define a falsifiable user-visible performance question and the metric/setup boundary that answers it.
- Preserve equivalent work, ownership, output and build settings before comparing algorithms or storage.
- Use repeated optimized measurements, traces and an explicit regression policy without presenting a kernel result as an app result.

**Prerequisites:** [[C2 — Copy-on-write behavior of standard-library types]]/[[C8 — Copyable, noncopyable types, and `~Copyable`]]/[[C9 — Existential, generic, and allocation performance model]]/[[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]/[[C12 — `Span`, `InlineArray`, and modern low-level abstractions]] for storage, ownership and optimization; [[D7 — Actor Reentrancy and Logic Races]] for publication identity; [[F1 — Testing Strategy in Modern Swift]] for correctness tests and [[F5 — Language mode, build settings, and migration flags]] for matching build settings.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A feed hitches during filtering. Replacing generic code or all arrays is a guess until evidence locates the cost. Measure a reproducible operation, separate preparation from publication/rendering, and ask whether time is spent executing, waiting, allocating or retaining data.

**Wall time** is elapsed duration; CPU samples show executing stacks; allocation traffic describes storage churn; retained memory describes what remains live. A profile locates work, a benchmark measures a chosen boundary, and correctness tests protect behavior. None alone proves user benefit. [Performance investigation](https://developer.apple.com/documentation/xcode/improving-your-app-s-performance)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Mark the operation you intend to investigate

Compile-only OSSignposter fixture. Its interval covers synchronous preparation and the supplied publication call, not a future display frame. The caller supplies a real presentation adapter when integrating it into an app.

```swift
import os

@MainActor func applyFilter(_ values: [Int], publish: ([Int]) -> Void) {
    let signposter = OSSignposter(subsystem: "example.study", category: "Filtering")
    let id = signposter.makeSignpostID()
    let interval = signposter.beginInterval("FilterAndPublish", id: id)
    defer { signposter.endInterval("FilterAndPublish", interval) }
    let selected = values.filter { $0.isMultiple(of: 2) }
    publish(selected)
}
```

**Walkthrough:** The interval includes filter and the synchronous publication callback. A UI trace must also cover diff/layout/rendering after that callback; a fast filter alone cannot establish smooth scrolling. Signposts label work, not CPU/wait/allocation causes.

### Hold the algorithm’s outputs and setup boundary fixed

Shared MembershipWork source, compiled as a separate optimized library with no client-visible bodies. linearMembership performs repeated scans; indexedMembership builds its Set inside the operation. Both return one Boolean per query, preserving duplicates, misses and order.

```swift
public func linearMembership(_ queries: [Int], in values: [Int]) -> [Bool] {
    queries.map { values.contains($0) }
}
public func indexedMembership(_ queries: [Int], in values: [Int]) -> [Bool] {
    let lookup = Set(values)
    return queries.map { lookup.contains($0) }
}
```

**Walkthrough:** The candidate changes algorithmic work and adds storage. Its complete call includes index construction, instead of timing only its cheap repeated lookup. A separate module limits caller-side elimination but also preserves call overhead; it is not a whole-module app build.


Save the shared source as `/tmp/MembershipWork.swift`, then emit its object and importable module. Use the same compiler/SDK and flags for the client; changing the library's optimization independently changes the experiment.

```sh
swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -target arm64-apple-macos27.0 -warnings-as-errors -O \
  -parse-as-library -enable-library-evolution -module-name MembershipWork \
  -emit-module -emit-module-path /tmp/MembershipWork.swiftmodule \
  -c /tmp/MembershipWork.swift -o /tmp/MembershipWork.o
```

### Separate a storage contract from its cost

Shared StorageWork source. Arithmetic deliberately wraps modulo UInt64’s range. UniqueArray needs OS 27; Array.span separately needs 26, while basic indexed Span processing can compile for older targets.

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

**Walkthrough:** These kernels define the same per-element operations, but constructing or retaining owners is a separate part of an experiment. Span borrows initialized storage; UniqueArray cannot silently supply Array’s freely copyable snapshot contract. Indexed iteration of the explicitly borrowed Array is deliberate for this compiler.

## Rules and pitfalls

### Timing only lookup can hide the proposed change’s construction cost

**Common pitfall:** Moving Set construction outside the candidate interval while comparing it with a complete scan operation.

**Mistake:** Standalone runnable example.

```swift
import MembershipWork

let values = Array(0..<100)
let queries = [0, 99, 100]
let clock = ContinuousClock()
let lookup = Set(values)  // Excluded from this interval.
var result: [Bool] = []
let elapsed = clock.measure { result = queries.map { lookup.contains($0) } }
precondition(result == [true, true, false])
print("lookup-only", elapsed)
```

**Consequence:** The number describes reuse of an already-built index, not creating an index for this request.

**Why:** An accurate clock cannot repair the wrong boundary. Reuse can be a valid separate hypothesis, but it requires cache/invalidation/retention policy.

**Correction:** Time the complete candidate call when construction belongs to the operation.

```swift
import MembershipWork

let values = Array(0..<100)
let queries = [0, 99, 100]
let clock = ContinuousClock()
var result: [Bool] = []
let elapsed = clock.measure { result = indexedMembership(queries, in: values) }
precondition(result == linearMembership(queries, in: values))
print("construction-and-lookup", elapsed)
```

**What changes:** The measured boundary now includes Set construction. These tiny examples demonstrate boundaries and output checks; their timings are not a performance conclusion.

### A unique owner does not supply an independent retained copy

**Common pitfall:** Transferring a UniqueArray while continuing to use its old binding as though assignment created a snapshot.

**Mistake:** Compile-only; intentionally rejected.

```swift
func invalidSnapshots() {
    var original = UniqueArray<UInt64>()
    original.append(1)
    let saved = original
    original[0] = 9
    _ = saved.count
}
```

**Consequence:** The old binding is used after its value was transferred; the compiler rejects the ownership assumption.

**Why:** Changing the owner can change the program’s allowed operations. A comparison that removes a required snapshot is not equivalent work.

**Correction:** Keep a copyable value when the required contract includes retaining an independent old sequence. Prepend StorageWork setup.

```swift
let saved: [UInt64] = [1, 2, 3]
var edited = saved
let result = StorageWork.updateArray(&edited, salt: 2)
precondition(saved == [1, 2, 3] && edited == [3, 4, 5] && result == 12)
```

**What changes:** The old value survives while mutation preserves independence. A unique-owner alternative must explicitly construct any equivalent retained snapshot and include or disclose that setup cost.

### A passing measurement is not a regression gate

**Common pitfall:** Treating successful benchmark execution as proof that a performance budget was met.

**Mistake:** This command checks correctness and records samples but has no earlier baseline or threshold:

```sh
/tmp/membership-probe 10000 1000 42 Swift6.4-arm64-macos27-O > current.json
```

**Consequence:** It can exit successfully even when later measurements regress.

**Why:** A timer or XCTest measurement without a configured baseline is evidence collection. A gate needs a declared workload, compatible environment, metric and acceptance policy.

**Correction:** Compare repeated samples under an explicit local policy, preserving the raw samples and failure status. Save this complete script as gate.py; it compares the candidate kernel metric between a recorded baseline report and a later matching report:

```python
import json, statistics, sys
baseline = json.load(open(sys.argv[1]))
current = json.load(open(sys.argv[2]))
if baseline["workload"] != current["workload"] or baseline["environment"] != current["environment"] or current["environment"] == "UNRECORDED":
    raise SystemExit("workload mismatch")
old = statistics.median(baseline["candidate_ns"])
new = statistics.median(current["candidate_ns"])
if len(current["candidate_ns"]) < 7 or old <= 0:
    raise SystemExit("insufficient baseline")
limit = old * 1.10  # Example local policy, not a universal acceptable regression.
print({"baseline_median_ns": old, "current_median_ns": new, "limit_ns": limit})
raise SystemExit(0 if new <= limit else 1)
```

**What changes:** A later compatible report can fail the stated policy. A ten-percent threshold is only this example’s local kernel policy, not a justified production budget. Hardware/build/environment metadata and noise must also match; the validation checks gate success and a deliberately slower synthetic report rather than inventing an app regression result.

## Worked example

### Equal storage outputs are not speed or allocation evidence

**Setup:** Prepend StorageWork setup and run on OS 27. Both updates use the same values and salt.

**Predict:** Does the equal checksum establish that one owner allocates less or makes a feed smoother?

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

storageProbe()
```

**Outcome:** The output is [1, 2, 3] [3, 4, 5] 12 12 12.

**Why:** The source snapshot and values are correct. Timing, allocations, reference-count traffic and app rendering need separate evidence; this probe measures none of them.

## Interview questions

### What would you measure before rewriting an abstraction-heavy path, and how would you distinguish CPU work, contention, and blocking?

**Answer:** Start with the complete user interval and representative optimized build, then correlate executing stacks, waits and allocation/retention evidence. Syntax is not a diagnosis.

**Example:** Standalone runnable example.

```swift
import os

func measuredPreparation(_ values: [Int]) -> [Int] {
    let signposter = OSSignposter(subsystem: "example.study", category: "Preparation")
    let state = signposter.beginInterval("Prepare")
    defer { signposter.endInterval("Prepare", state) }
    return values.filter { $0.isMultiple(of: 2) }
}
precondition(measuredPreparation([1, 2, 3, 4]) == [2, 4])
```

**Walkthrough:** The signpost identifies this preparation interval. Instruments CPU stacks can show active work; long wall time with little CPU suggests waiting, but locating the cause needs the corresponding wait/task/lock history.

**Boundary:** No Instruments trace is captured by this source example. Separate image decode, diffing, layout/rendering, blocking I/O and contention before rewriting a generic API.

### How do you compare Array, UniqueArray, and Span without changing ownership requirements or timing the wrong work?

**Answer:** Hold operations, input, retained snapshots and outputs fixed; state whether construction, conversion, mutation or complete work is timed. A borrowed view cannot replace an owning snapshot.

**Example:** Prepend StorageWork setup.

```swift
func storageChecks() {
    let values: [UInt64] = [1, 2, 3]
    let view = values.span
    precondition(StorageWork.sumArray(values) == StorageWork.sumSpan(view))
    var ordinary = values
    var unique = UniqueArray<UInt64>()
    for value in values { unique.append(value) }
    let first = StorageWork.updateArray(&ordinary, salt: 2)
    let second = StorageWork.updateUnique(&unique, salt: 2)
    precondition(first == second && first == 12 && values == [1, 2, 3])
}
storageChecks()
```

**Walkthrough:** The read paths and update outputs agree. Unique-buffer construction is visible before update; retaining values also creates a sharing requirement for ordinary Array. Those facts must be reflected in any timed comparison.

**Boundary:** The actual separate-module controls time mutation/read paths under their declared ownership settings; no result here ranks all storage choices. Unique storage still allocates/grows.

### Why can correct output and an accurate timer still produce a misleading benchmark?

**Answer:** They do not validate input representativeness, sample count, build equivalence, setup boundaries or optimizer behavior. Correctness is necessary; experimental design is another requirement.

**Example:** Separate optimized MembershipWork client.

```swift
import MembershipWork

for (values, queries) in [([Int](), [0]), ([1, 1, 2], [2, 3, 1]), ([1, 2], [Int]())] {
    let baseline = linearMembership(queries, in: values)
    let candidate = indexedMembership(queries, in: values)
    precondition(baseline == candidate)
}
```

**Walkthrough:** These edge checks establish equal membership behavior, including duplicates, misses and empty queries. The repeated runtime-input experiment below additionally records measurement boundaries and consumes each result.

**Boundary:** A checksum alone is not a complete optimization barrier. The separately compiled opaque kernels limit elimination but add a module boundary; they do not recreate every app optimizer opportunity.

### Which user-visible gain justifies the complexity, memory and API cost of a proposed optimization?

**Answer:** A measured improvement in a real latency/throughput/energy/memory constraint large enough to justify the added costs. A kernel win is supporting evidence; repeat the original user scenario before adopting an ownership or deployment change.

**Example:** Collect the local workload evidence with these actual commands, then repeat the separately instrumented app scenario before making an app claim:

```sh
# Build the kernel library and driver together under the recorded -O baseline.
/tmp/membership-probe 10000 1000 42 Swift6.4-arm64-macos27-O > baseline.json
/tmp/membership-probe 10000 1000 42 Swift6.4-arm64-macos27-O > current.json
python3 gate.py baseline.json current.json
```

**Walkthrough:** The script reports a local candidate-kernel median against its explicit policy. The signpost fixtures identify preparation/publication intervals for a later app trace. An app report must also retain data/cache state, device/build, event/sample counts and the memory/API tradeoff; those are not present in these kernel numbers.

**Boundary:** This revision runs the local optimized experiment, not a scrolling-device, energy or allocation campaign. Keep the simpler design if the actual user constraint does not improve. Offloaded work must still preserve task ownership, cancellation, bounded admission and publication identity as in [[D7 — Actor Reentrancy and Logic Races|D7]]/[[F1 — Testing Strategy in Modern Swift|F1]]/[[F6 — Observation, property wrappers, and language-adjacent state tools|F6]].

## Focused exercise

**Task:** Define one representative local workload/metric/baseline, compare one change in matching optimized builds, verify equal outputs and report limits.

**Acceptance checks:**

- Both algorithms preserve Boolean result order, duplicates and misses.
- The candidate interval includes Set construction.
- Seven interleaved-order samples follow warmup, use runtime-supplied bounded input parameters, and consume results externally.
- The report names this kernel boundary and excludes app/allocation claims.

**Solution:** Build MembershipWork separately as stated earlier and save this ordinary client as MembershipProbe.swift. Compile/link with -O, the baseline flags, -I to the emitted module and the library object. Supply the final environment label from the recorded compiler/SDK/hardware/build setup; the gate rejects UNRECORDED or differing labels rather than discovering that setup automatically. Invoke it with count/query-count/seed arguments as shown above.

```swift
import Foundation
import MembershipWork

struct Generator {
    var state: UInt64
    mutating func next() -> UInt64 {
        state = state &* 6_364_136_223_846_793_005 &+ 1_442_695_040_888_963_407
        return state
    }
}
func nanoseconds(_ duration: Duration) -> Double {
    let parts = duration.components
    return Double(parts.seconds) * 1e9 + Double(parts.attoseconds) / 1e9
}
let arguments = CommandLine.arguments
let count = arguments.count > 1 ? Int(arguments[1])! : 10_000
let queryCount = arguments.count > 2 ? Int(arguments[2])! : 1_000
let seed = arguments.count > 3 ? UInt64(arguments[3])! : 42
let environment = arguments.count > 4 ? arguments[4] : "UNRECORDED"
precondition((1...100_000).contains(count) && (1...10_000).contains(queryCount))
var generator = Generator(state: seed)
let values = (0..<count).map { _ in Int(generator.next() % UInt64(count * 2)) }
let queries = (0..<queryCount).map { _ in Int(generator.next() % UInt64(count * 3)) }
let expected = linearMembership(queries, in: values)
precondition(indexedMembership(queries, in: values) == expected)
let clock = ContinuousClock()
var baseline: [Double] = []
var candidate: [Double] = []
for sample in 0..<7 {
    var left: [Bool] = []
    var right: [Bool] = []
    var first: Duration = .zero
    var second: Duration = .zero
    if sample.isMultiple(of: 2) {
        first = clock.measure { left = linearMembership(queries, in: values) }
        second = clock.measure { right = indexedMembership(queries, in: values) }
    } else {
        second = clock.measure { right = indexedMembership(queries, in: values) }
        first = clock.measure { left = linearMembership(queries, in: values) }
    }
    precondition(left == expected && right == expected)
    baseline.append(nanoseconds(first))
    candidate.append(nanoseconds(second))
}
struct Report: Encodable {
    let workload: String
    let environment: String
    let baseline_ns: [Double]
    let candidate_ns: [Double]
    let hits: Int
}
let report = Report(
    workload:
        "count=\(count);queries=\(queryCount);seed=\(seed);set-build=included;module=opaque;optimization=O",
    environment: environment, baseline_ns: baseline, candidate_ns: candidate,
    hits: expected.filter { $0 }.count)
let output = try JSONEncoder().encode(report)
print(String(decoding: output, as: UTF8.self))
```

```sh
swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -target arm64-apple-macos27.0 -warnings-as-errors -O \
  -I /tmp /tmp/MembershipWork.o /tmp/MembershipProbe.swift -o /tmp/membership-probe
/tmp/membership-probe 10000 1000 42 Swift6.4-arm64-macos27-O
```

**Reasoning:** Runtime parameters and externally printed results keep the operation observable; opaque library calls restrict caller-side elimination. Input generation is excluded from both intervals, and Set construction is included in the candidate. Seven samples are a local teaching experiment, not a statistical guarantee or production threshold. Compare raw distributions and repeat on the actual release scenario; allocation/energy and UI work remain separate.

## In practice

**Choose it when:** Begin with a falsifiable user constraint and use traces to choose one implicated change.

**Investigate by:** Record build/data/cache/ownership/module/setup boundaries, equivalent outputs, raw samples and any regression policy.

**Strong evidence:** Connect any kernel gain to a repeated user scenario and disclose memory/API/deployment costs instead of claiming a universal feature ranking.

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
