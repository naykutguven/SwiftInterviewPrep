---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - existentials
  - generics
---

# C9 — Existential, Generic, and Allocation Performance Model

**Learning goals:**

- Separate type relationships, dispatch, representation, allocation, and measured runtime cost.
- Compare concrete/generic/opaque/existential/enum/closure-erased choices using the same bounded work and effect order.
- Inspect visible bodies and reachable SIL across real module boundaries before making optimization claims.

**Prerequisites:** [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]/[[B3 — Generics, Constraints, and Same-Type Relationships]] for opaque/existential/generic relationships, [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]] for dispatch, [[B11 — Library Evolution and Resilience-Aware API Design]] for module/body visibility, and [[C2 — Copy-on-write behavior of standard-library types]] for array value semantics.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Two pixel pipelines can compute identical bytes yet take different routes to an implementation. A generic parameter preserves a type relationship; it does not require specialization. An existential holds a runtime-selected conformer; a known concrete value may still let the optimizer remove its packaging or dispatch. A concrete eraser can hide a stored closure call.

Keep five questions separate: what types the API preserves, how a call selects its implementation, how values are represented, which allocations remain, and how the representative workload performs. MemoryLayout answers only part of the representation question. Establish equal work and effects before comparing optimized artifacts or measurements. [Existential any](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Start with the same bounded operation

Shared PixelTransform setup. UInt8 inversion and clamping keep every result representable; no unrelated Int-overflow policy contaminates the comparison.

```swift
public protocol PixelTransform { func transform(_ value: UInt8) -> UInt8 }
struct Invert: PixelTransform {
    func transform(_ value: UInt8) -> UInt8 { 255 - value }
}
struct Clamp: PixelTransform {
    let maximum: UInt8
    func transform(_ value: UInt8) -> UInt8 { min(value, maximum) }
}
precondition(Invert().transform(0) == 255 && Invert().transform(255) == 0)
precondition(Clamp(maximum: 200).transform(255) == 200)
```

**Walkthrough:** Both implementations fulfill one scalar operation. Neither protocol syntax nor these assertions establish which call will inline, whether storage allocates, or how fast a complete image operation runs.

### Hold traversal and storage mutation fixed

Shared apply setup; prepend PixelTransform. Both functions visit the same indices in the same order and edit the same Array representation.

```swift
func genericApply<T: PixelTransform>(_ transform: T, to pixels: inout [UInt8]) {
    for index in pixels.indices { pixels[index] = transform.transform(pixels[index]) }
}
func existentialApply(_ transform: any PixelTransform, to pixels: inout [UInt8]) {
    for index in pixels.indices { pixels[index] = transform.transform(pixels[index]) }
}
func applicationProbe() {
    let input: [UInt8] = [0, 127, 255]
    var typed = input
    var erased = input
    genericApply(Invert(), to: &typed)
    existentialApply(Invert(), to: &erased)
    precondition(typed == [255, 128, 0] && erased == typed)
    precondition(input == [0, 127, 255])
}
applicationProbe()
```

**Walkthrough:** The original stays unchanged while both outputs match. That is semantic equivalence and value independence, not an allocation count. A known Invert can be visible in either formulation; an unknown generic T can still use witnesses.

### Choose the runtime-choice contract deliberately

Shared alternatives setup; prepend PixelTransform. PixelRule closes the conformer choices into cases. AnyPixelTransform is a concrete conforming wrapper whose private closure can capture any supported transform.

```swift
enum PixelRule: PixelTransform {
    case invert
    case clamp(maximum: UInt8)
    func transform(_ value: UInt8) -> UInt8 {
        switch self {
        case .invert: 255 - value
        case .clamp(let maximum): min(value, maximum)
        }
    }
}
struct AnyPixelTransform: PixelTransform {
    private let body: (UInt8) -> UInt8
    init<T: PixelTransform>(_ transform: T) { body = transform.transform }
    func transform(_ value: UInt8) -> UInt8 { body(value) }
}
func makeFixedTransform() -> some PixelTransform { Invert() }
func makeDynamicTransform(_ invert: Bool) -> any PixelTransform {
    if invert { Invert() } else { Clamp(maximum: 200) }
}
func makeClosedTransform(_ invert: Bool) -> PixelRule {
    if invert { .invert } else { .clamp(maximum: 200) }
}
precondition(makeFixedTransform().transform(1) == 254)
precondition(makeDynamicTransform(false).transform(255) == 200)
precondition(makeClosedTransform(true).transform(1) == 254)
precondition(AnyPixelTransform(Invert()).transform(1) == 254)
```

**Walkthrough:** The opaque factory hides one concrete identity. The existential factory permits unrelated runtime conformers. The enum keeps one concrete type with runtime case branching. The eraser keeps one concrete wrapper type with a closure implementation. Those are modeling choices; none proves a winner.

| Surface | Relationship retained | Cost still requiring evidence |
| --- | --- | --- |
| Concrete or generic T | A concrete type or one named generic relationship | Specialization, witnesses, element/storage cost |
| Opaque some result | One hidden identity per declaration/generic context | Client-visible implementation and remaining calls |
| Existential any value | Runtime choice through the protocol | Packaging, dispatch, payload/storage allocations |
| Closed enum | One concrete type with declared cases | Branching, payloads, extensibility tradeoff |
| Concrete closure eraser | One wrapper type with private behavior | Captures, indirect calls, closure elimination |

### Shared trace setup

Recorded logs calls before performing the bounded transform. The log is used synchronously; this is effect-order evidence, not timing instrumentation.

```swift
final class Trace { var events: [String] = [] }
struct Recorded: PixelTransform {
    let name: String
    let log: Trace
    let body: (UInt8) -> UInt8
    func transform(_ value: UInt8) -> UInt8 {
        log.events.append(name + ":" + String(value))
        return body(value)
    }
}
func recorded(_ log: Trace) -> [Recorded] {
    [
        Recorded(name: "invert", log: log, body: { 255 - $0 }),
        Recorded(name: "clamp", log: log, body: { min($0, 200) }),
    ]
}
```

## Rules and pitfalls

### Fusing loops can change observable effects

**Common pitfall:** Comparing a per-pixel fused pipeline with a per-transform pipeline as if they performed the same observable work. Prepend PixelTransform, apply, and trace setups.

**Mistake:** Standalone runnable example.

```swift
func orderingProbe() {
    let log = Trace()
    let transforms = recorded(log)
    var pixels: [UInt8] = [1, 2]
    for index in pixels.indices {
        for transform in transforms { pixels[index] = transform.transform(pixels[index]) }
    }
    precondition(pixels == [200, 200])
    precondition(log.events == ["invert:1", "clamp:254", "invert:2", "clamp:253"])
    precondition(log.events != ["invert:1", "invert:2", "clamp:254", "clamp:253"])
}
orderingProbe()
```

**Consequence:** The final bytes match the pure example, but the call trace differs from the required per-transform order.

**Why:** Fusion changes which operation happens next. A nonmutating requirement can still log, read shared state, or have other effects; equal final bytes alone do not establish full equivalence.

**Correction:** Preserve the specified order for the comparison, or explicitly require a pure/equivalent effect contract before fusing. Prepend the same setups.

```swift
let log = Trace()
var pixels: [UInt8] = [1, 2]
for transform in recorded(log) { genericApply(transform, to: &pixels) }
precondition(pixels == [200, 200])
precondition(log.events == ["invert:1", "invert:2", "clamp:254", "clamp:253"])
```

**What changes:** The comparison now preserves the declared pass order. Any speedup hypothesis must hold this semantic choice, data, storage, and result handling fixed.

### A wrapper can move a witness call into a helper

**Common pitfall:** Searching only the wrapper body for witness_method and concluding dispatch disappeared.

**Mistake:** Compile-only inspection source. Prepend PixelTransform and apply setups. The no-inline wrapper is an observation point, not a performance recommendation.

```swift
@inline(never)
public func unknownWrapper(_ transform: any PixelTransform, pixels: inout [UInt8]) {
    existentialApply(transform, to: &pixels)
}
```

**Consequence:** A wrapper can contain a function_ref to existentialApply while that reachable helper still performs the transform witness call.

**Why:** Looking at one function omits the code it invokes. Even an optimizer-generated specialization can remain generic over the opened runtime type.

**Correction:** Inspect the reachable call path. Save the assembled Swift source as Example.swift, then emit the actual SIL under the baseline rather than guessing from source names:

```sh
xcrun --sdk macosx swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -target arm64-apple-macos27.0 -module-cache-path /tmp/c9-module-cache \
  -module-name PixelProbe -Onone -emit-sil Example.swift -o /tmp/PixelProbe.sil
rg -n -A 40 'unknownWrapper' /tmp/PixelProbe.sil
rg -n -A 90 'existentialApply' /tmp/PixelProbe.sil
```

**What changes:** The inspection follows function_ref targets and identifies the requirement call where it actually occurs. The validation also inspects optimized reachable functions. Neither the absence of an instruction in one wrapper nor the presence of “specialized” in a symbol establishes allocation count or throughput.

## Worked example

### Compare existential, opened, and closed pipelines

**Setup:** Prepend PixelTransform, apply, and alternatives setups. The same input and per-transform order are used everywhere.

**Predict:** Does opening each existential turn the stored transform list into a compile-time homogeneous list?

```swift
func existentialPipeline(_ transforms: [any PixelTransform], to pixels: inout [UInt8]) {
    for transform in transforms { existentialApply(transform, to: &pixels) }
}
func openedPipeline(_ transforms: [any PixelTransform], to pixels: inout [UInt8]) {
    for transform in transforms { genericApply(transform, to: &pixels) }
}
func genericPipeline<T: PixelTransform>(_ transforms: [T], to pixels: inout [UInt8]) {
    for transform in transforms { genericApply(transform, to: &pixels) }
}
func pipelineProbe() -> [[UInt8]] {
    let input: [UInt8] = [0, 127, 255]
    var erased = input
    var opened = input
    var closed = input
    let transforms: [any PixelTransform] = [Invert(), Clamp(maximum: 200)]
    existentialPipeline(transforms, to: &erased)
    openedPipeline(transforms, to: &opened)
    genericPipeline([PixelRule.invert, .clamp(maximum: 200)], to: &closed)
    return [input, erased, opened, closed]
}
precondition(pipelineProbe() == [[0, 127, 255], [200, 128, 0], [200, 128, 0], [200, 128, 0]])
```

**Outcome:** Every output is [200, 128, 0] and the source stays [0, 127, 255]. Opening does not change the array’s existential storage contract.

**Why:** The generic call temporarily names one runtime conformer. It does not make any PixelTransform generally self-conforming or recover one compile-time type shared by unrelated elements. [Existential opening](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)

## Interview questions

### What does the optimizer know at this call and across a module boundary?

**Answer:** It knows constraints, visible bodies, and any concrete facts recovered during optimization. A separately compiled public generic body can still receive a conformer unknown to that library. Merely writing a generic does not make that body available for client specialization.

**Example:** Compile-only library PixelCore. Add -parse-as-library -O -enable-library-evolution -module-name PixelCore -emit-module -emit-module-path /tmp/PixelCore.swiftmodule -c -o /tmp/PixelCore.o to the baseline; separate clients add -I /tmp and link PixelCore.o.

```swift
public protocol PixelTransform { func transform(_ value: UInt8) -> UInt8 }
public func genericApply<T: PixelTransform>(_ transform: T, to pixels: inout [UInt8]) {
    for index in pixels.indices { pixels[index] = transform.transform(pixels[index]) }
}
public func existentialApply(_ transform: any PixelTransform, to pixels: inout [UInt8]) {
    for index in pixels.indices { pixels[index] = transform.transform(pixels[index]) }
}
```

**Walkthrough:** Build this separate PixelCore module under -O -enable-library-evolution, with object and binary/textual module output. No @inlinable body is exported. The ordinary client below supplies its own conformer and obtains matching results; separate SIL controls inspect the library’s unknown generic and existential paths.

**Boundary:** Body export, resilience, whole-module/cross-module optimization, and target settings are independent choices. This current-compiler experiment does not establish a timing result or a general old-client binary contract.

Separate runnable PixelCore client; no testable import or chapter-local protocol is prepended:

```swift
import PixelCore

struct ClientMask: PixelTransform {
    let mask: UInt8
    func transform(_ value: UInt8) -> UInt8 { value ^ mask }
}
let input: [UInt8] = [0, 127, 255]
var typed = input
var erased = input
genericApply(ClientMask(mask: 1), to: &typed)
existentialApply(ClientMask(mask: 1), to: &erased)
precondition(typed == [1, 126, 254] && erased == typed && input == [0, 127, 255])
```

### Why can [any P] retain costs, and why does [T] alone not prove their removal?

**Answer:** Existential elements retain runtime choice and can require representation/witness work. [T] retains one element type, but unspecialized generics can still use witnesses and T can itself be a closure eraser. Representation, dispatch, and allocation must be investigated separately.

**Example:** Prepend PixelTransform, apply, alternatives, and worked pipeline setups.

```swift
let input: [UInt8] = [0, 127, 255]
var erased = input
var closed = input
var wrapped = input
existentialPipeline([Invert(), Clamp(maximum: 200)], to: &erased)
genericPipeline([PixelRule.invert, .clamp(maximum: 200)], to: &closed)
genericPipeline(
    [AnyPixelTransform(Invert()), AnyPixelTransform(Clamp(maximum: 200))], to: &wrapped)
precondition(erased == [200, 128, 0] && closed == erased && wrapped == erased)
```

**Walkthrough:** PixelRule is one closed enum type; AnyPixelTransform is one concrete wrapper type holding different captured operations. Matching bytes do not prove that the stored closure calls or allocations disappeared.

**Boundary:** A closed case set changes extensibility. A large/reference enum payload can still retain storage, and a concrete eraser can retain reference state. Measure representative unknown and known callers before choosing by cost.

### Extension: What do value-layout sizes establish about existential allocation?

**Answer:** They establish the size of those value representations for the selected target. They do not count payload allocations, closure contexts, Array buffers, retained object graphs, or process memory.

**Example:** Prepend PixelTransform setup; exact sizes are target observations.

```swift
struct SmallMask: PixelTransform {
    let mask: UInt8
    func transform(_ value: UInt8) -> UInt8 { value ^ mask }
}
struct LargeMask: PixelTransform {
    let a: UInt64, b: UInt64, c: UInt64, d: UInt64
    func transform(_ value: UInt8) -> UInt8 { value ^ UInt8(truncatingIfNeeded: a ^ b ^ c ^ d) }
}
protocol ReferenceTransform: AnyObject { func transform(_ value: UInt8) -> UInt8 }
final class ObjectMask: PixelTransform, ReferenceTransform {
    let mask: UInt8
    init(mask: UInt8) { self.mask = mask }
    func transform(_ value: UInt8) -> UInt8 { value ^ mask }
}
func layoutProbe() -> [Int] {
    [
        MemoryLayout<Int>.size, MemoryLayout<SmallMask>.size, MemoryLayout<LargeMask>.size,
        MemoryLayout<any PixelTransform>.size, MemoryLayout<any ReferenceTransform>.size,
    ]
}
precondition(layoutProbe() == [8, 1, 32, 40, 16])
```

**Walkthrough:** Those are the checked arm64 Apple value sizes. The ordinary unconstrained existential can use inline payload space plus metadata/witness information or out-of-line payload storage as required by layout/alignment; a class-constrained representation refers to an object with its own lifetime.

**Boundary:** An Array/String payload can fit its value representation while retaining other storage. Optimizations can remove temporary containers entirely. Inspect allocations when allocation is the question; these numbers are not a portable heap-footprint guarantee.

Prepend PixelTransform, alternatives, and layout setups. This runnable closure-erasure lifetime probe checks retaining edges rather than allocation counts:

```swift
func capturedObjectProbe() {
    weak var observed: ObjectMask?
    var wrapper: AnyPixelTransform?
    do {
        let object = ObjectMask(mask: 1)
        observed = object
        wrapper = AnyPixelTransform(object)
    }
    precondition(observed != nil && wrapper?.transform(0) == 1)
    wrapper = nil
    precondition(observed == nil)
}
capturedObjectProbe()
```

### Extension: What can a focused SIL experiment establish for known and unknown transforms?

**Answer:** It can show the emitted implementation for the selected sources/settings, including whether reachable transform witness calls remain. Known concrete facts may eliminate them for both a generic call and a known existential; unknown conformers retain different opportunities.

**Example:** Compile-only observation wrappers; prepend PixelTransform and apply setups.

```swift
@inline(never) public func genericInvert(_ pixels: inout [UInt8]) {
    genericApply(Invert(), to: &pixels)
}
@inline(never) public func knownExistentialInvert(_ pixels: inout [UInt8]) {
    let transform: any PixelTransform = Invert()
    existentialApply(transform, to: &pixels)
}
@inline(never) public func unknownExistentialApply(
    _ transform: any PixelTransform, to pixels: inout [UInt8]
) {
    existentialApply(transform, to: &pixels)
}
```

**Walkthrough:** The validation emits SILGen/Onone, mandatory SIL/Onone, and optimized SIL/O from the assembled source. It follows chapter helper targets: known-Invert reachable paths lose transform witness calls in the optimized artifact, while the unknown path keeps one.

**Boundary:** This is compiler evidence, not a permanent instruction sequence, universal specialization, speedup, or allocation measurement. The no-inline wrappers retain observation points. [[B11 — Library Evolution and Resilience-Aware API Design|B11]] separately checks exported-body and binary-replacement contracts.

The opaque choice boundary remains a source-level constraint, independent of optimization. Prepend PixelTransform setup for this **intentionally rejected** declaration:

```swift
func invalidOpaque(_ invert: Bool) -> some PixelTransform {
    if invert { return Invert() }
    return Clamp(maximum: 200)
}
```

## Focused exercise

**Task:** Compare equivalent generic and existential pipelines, keeping pure work, order, input ownership, and expected results fixed.

**Acceptance checks:**

- Empty, edge, and all-UInt8 inputs produce equal results through each selected contract.
- Every pipeline applies invert then clamp in the same pass order and preserves its original input value.
- The report identifies compiler/runtime measurements still needed before claiming a speedup.

**Solution:** Prepend PixelTransform, apply, alternatives, and worked pipeline setups. These are semantic checks rather than a benchmark:

```swift
let fixtures: [[UInt8]] = [[], [0, 127, 255], Array(UInt8.min...UInt8.max)]
for input in fixtures {
    let original = input.map(Int.init)
    for maximum: UInt8 in [0, 1, 127, 200, 255] {
        let expected = input.map { min(255 - $0, maximum) }
        var erased = input
        var opened = input
        var closed = input
        var wrapped = input
        existentialPipeline([Invert(), Clamp(maximum: maximum)], to: &erased)
        openedPipeline([Invert(), Clamp(maximum: maximum)], to: &opened)
        genericPipeline([PixelRule.invert, .clamp(maximum: maximum)], to: &closed)
        genericPipeline(
            [AnyPixelTransform(Invert()), AnyPixelTransform(Clamp(maximum: maximum))], to: &wrapped)
        precondition(
            erased == expected && opened == expected && closed == expected && wrapped == expected)
        precondition(input.map(Int.init) == original)
    }
}
```

**Reasoning:** The bounded operations avoid overflow and each comparison preserves the actual pass order. Before a performance claim, use the shipped optimization and module boundaries, representative unknown/known inputs, warmup/repeated samples, externally consumed output, and separate CPU/allocation/elapsed-time hypotheses. [[C2 — Copy-on-write behavior of standard-library types|C2]] supplies the storage/retention review.

## In practice

**Choose it when:** Choose extensibility and retained type relationships first, then investigate a demonstrated cost.

**Investigate by:** Hold semantics and effects fixed, inspect visible bodies and reachable optimized calls, and measure allocations separately from value layout.

**Strong evidence:** Explain what each artifact proves and what still requires workload measurement; do not treat generic/any/some/enum/closure syntax as a performance verdict.

## References

### Related guides

- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F3 — Performance investigation habits]]

### Official sources

- [Existential any](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md)
- [SE-0352](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)
- [SE-0244: Opaque result types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)
- [Swift compiler optimization tips](https://github.com/swiftlang/swift/blob/main/docs/OptimizationTips.rst)
- [SE-0193: Cross-module inlining and specialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)
- [SE-0496: Inline always](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)
- [SE-0497: Definition visibility](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)
