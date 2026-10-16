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

**Goal:** Compare equivalent generic and existential work without inferring a speedup from type syntax.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Generics preserve type relationships; specialization is a compiler decision. An unspecialized generic body can use metadata and witnesses. An existential stores a runtime-selected conformer, but a compiler that knows the concrete value can remove packaging or dispatch. Opaque results retain hidden identity without guaranteeing that a client's optimizer sees the implementation. [Existential any](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md)

## Rules and pitfalls

### Compare the same semantic workload

Hold data, operations, ownership, module boundaries, and result handling fixed. A closed enum or closure eraser may be viable but changes the extensibility or representation contract. MemoryLayout describes value layout, not total heap allocations.

**Common pitfall:** Comparing fused generic work with a multipass existential pipeline can change effect order, temporary storage, and algorithmic work. Equal final bytes do not prove equivalent observable effects. Establish equivalence before timing.

### Follow optimization evidence through calls

Known concrete inputs can specialize or devirtualize in either formulation. Unknown conformers across separately compiled modules retain different opportunities. A concrete eraser with a stored closure can still perform an indirect call.

**Common pitfall:** Finding no witness instruction in a wrapper is inconclusive if the wrapper calls a helper that performs the dispatch. Follow reachable functions in SIL and measure representative optimized callers; see [[F3 — Performance investigation habits]].

## Interview questions

### What does the optimizer know at this call and across the module boundary?

It knows the static constraints, visible bodies, and any concrete facts available after optimization. Separate module compilation, resilience, body export, and unknown client conformers limit that knowledge. Inspect the actual optimized artifact for the target build rather than assuming a source-level generic always specializes.

### Why can [any P] retain representation/dispatch costs, and why does [T] alone not prove their removal?

Heterogeneous existential elements can require packaging and witness selection, although known uses may optimize away. [T] preserves one element type but an unspecialized caller can still use metadata/witnesses, and T can itself be a concrete closure eraser. Representation, dispatch, and allocation are related but separate measurements.

## Focused exercise

**Task:** Compare generic and existential pipelines performing identical work. Check equal results and identify what must be measured before claiming a speedup.

**Acceptance checks:**

- Both versions apply the same pure operation in the same traversal order.
- Results and original-input preservation match.
- No time, dispatch count, or allocation conclusion is inferred from passing assertions.

**Solution:**

Standalone equivalent scalar transforms over the same inputs:

```swift
protocol Transform { func apply(_ value: Int) -> Int }
struct Add: Transform {
    let amount: Int
    func apply(_ value: Int) -> Int { value + amount }
}
func generic<T: Transform>(_ transform: T, input: [Int]) -> [Int] {
    input.map { transform.apply($0) }
}
func existential(_ transform: any Transform, input: [Int]) -> [Int] {
    input.map { transform.apply($0) }
}
let input = [0, 1, 2]
let transform = Add(amount: 3)
let typed = generic(transform, input: input)
let erased = existential(transform, input: input)
precondition(typed == [3, 4, 5] && typed == erased)
precondition(input == [0, 1, 2])
```

**Reasoning:** The fixture establishes equivalent behavior, not an interesting optimization winner. A benchmark must introduce representative unknown/known call boundaries, matching optimization, warmup, repeated samples, and externally consumed output. Separate allocation, CPU, and elapsed-time hypotheses.

## In practice

**Choose it when:** Select abstraction by extensibility and caller relationships, then investigate demonstrated cost.

**Investigate by:** Inspect optimized SIL and allocations under the same module and ownership configuration.

**Strong evidence:** Explain both what the compiler artifact establishes and what requires runtime measurement.

### Pipeline and SIL evidence — optional

#### Equivalent generic and existential operations

```swift
public protocol PixelTransform {
    func transform(_ value: UInt8) -> UInt8
}

struct Invert: PixelTransform {
    func transform(_ value: UInt8) -> UInt8 { 255 - value }
}

struct Clamp: PixelTransform {
    let maximum: UInt8
    func transform(_ value: UInt8) -> UInt8 { min(value, maximum) }
}
```

Both implementations below traverse exactly the same input in the same order and mutate the same Array representation:

```swift
func genericApply<T: PixelTransform>(
    _ transform: T, to pixels: inout [UInt8]
) {
    for index in pixels.indices {
        pixels[index] = transform.transform(pixels[index])
    }
}

func existentialApply(
    _ transform: any PixelTransform, to pixels: inout [UInt8]
) {
    for index in pixels.indices {
        pixels[index] = transform.transform(pixels[index])
    }
}
```

For Invert and [0, 127, 255], both produce [255, 128, 0]. This proves a semantic fixture, not which call is faster. A generic call with a concrete Invert can be specialized; an existential created as Invert immediately before the call may be equally visible to the optimizer. Conversely, an unknown generic T can still require witness dispatch.

The Int overflow from unrelated scoring examples is avoided here: subtracting a UInt8 from 255 stays representable, and clamp stays within its input range. Those bounded operations make the comparison's arithmetic contract explicit.

#### Heterogeneous storage and existential opening

```swift
func existentialPipeline(
    _ transforms: [any PixelTransform], to pixels: inout [UInt8]
) {
    for transform in transforms {
        existentialApply(transform, to: &pixels)
    }
}

func openedPipeline(
    _ transforms: [any PixelTransform], to pixels: inout [UInt8]
) {
    for transform in transforms {
        genericApply(transform, to: &pixels)
    }
}

func genericPipeline<T: PixelTransform>(
    _ transforms: [T], to pixels: inout [UInt8]
) {
    for transform in transforms {
        genericApply(transform, to: &pixels)
    }
}
```

The opened form is valid: Swift implicitly opens each existential for that generic call. This does not make any PixelTransform self-conform or recover a compile-time concrete type for every runtime element. Nor does it turn [any PixelTransform] into [Invert]. The array can still contain unrelated conformers and retain its existential storage. [SE-0352](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)

Moving a generic operation around a runtime value is worth investigating only with evidence. It can change where opening occurs, but “generic helper outside the inner loop” does not by itself prove specialization, fewer allocations, or higher throughput.

#### A closed enum preserves runtime choice without existential elements

```swift
enum PixelRule: PixelTransform {
    case invert
    case clamp(maximum: UInt8)

    func transform(_ value: UInt8) -> UInt8 {
        switch self {
        case .invert: 255 - value
        case let .clamp(maximum): min(value, maximum)
        }
    }
}

func makeFixedTransform() -> some PixelTransform {
    Invert()
}

func makeDynamicTransform(_ invert: Bool) -> any PixelTransform {
    if invert { Invert() } else { Clamp(maximum: 200) }
}

func makeClosedTransform(_ invert: Bool) -> PixelRule {
    if invert { .invert } else { .clamp(maximum: 200) }
}
```

A [PixelRule] is homogeneous in its concrete element type while holding different enum cases. It can use genericPipeline. It replaces an open conformer set with a declared case set, and can still require runtime branching. It is a modeling choice, not proof of better performance. An enum with a large/reference payload can still copy, retain, or allocate through that payload; its Array still has storage.

#### Layout, payload storage, and heap allocation are different measurements

```swift
struct SmallMask: PixelTransform {
    let mask: UInt8
    func transform(_ value: UInt8) -> UInt8 { value ^ mask }
}

struct LargeMask: PixelTransform {
    let a: UInt64
    let b: UInt64
    let c: UInt64
    let d: UInt64
    func transform(_ value: UInt8) -> UInt8 {
        value ^ UInt8(truncatingIfNeeded: a ^ b ^ c ^ d)
    }
}

protocol ReferenceTransform: AnyObject {
    func transform(_ value: UInt8) -> UInt8
}

final class ObjectMask: PixelTransform, ReferenceTransform {
    let mask: UInt8
    init(mask: UInt8) { self.mask = mask }
    func transform(_ value: UInt8) -> UInt8 { value ^ mask }
}

func layoutProbe() -> [Int] {
    [MemoryLayout<Int>.size,
     MemoryLayout<SmallMask>.size,
     MemoryLayout<LargeMask>.size,
     MemoryLayout<any PixelTransform>.size,
     MemoryLayout<any ReferenceTransform>.size]
}
```

On the checked arm64 Apple runtime this returns [8, 1, 32, 40, 16]. These are value-layout sizes, not heap footprints, allocation counts, array capacities, or process memory. They also do not prescribe every existential representation on every target.

The usual unconstrained single-protocol existential representation includes an inline value buffer plus metadata and witness information. Payload layout/alignment and runtime representation determine whether out-of-line storage is needed. A class-constrained representation carries an object reference and conformance information; the object already has a separate lifetime. An Array or String payload may fit its value representation while owning other storage. Optimizations can remove temporary representations entirely. Inspect retained allocations when allocation count is the question. [SE-0335](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md)

#### A concrete eraser can still contain an indirect call

```swift
struct AnyPixelTransform: PixelTransform {
    private let body: (UInt8) -> UInt8

    init<T: PixelTransform>(_ transform: T) {
        body = transform.transform
    }

    func transform(_ value: UInt8) -> UInt8 { body(value) }
}
```

[AnyPixelTransform] has one concrete element type, but each stored closure can capture a different underlying transform. Specializing genericPipeline for this wrapper does not automatically specialize every captured operation. The capture can retain reference state; escaping closure allocation/elimination needs separate evidence. The wrapper deliberately makes no Sendable promise.

Using the preceding declarations:

```swift
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
```

The result is [[0, 127, 255], [200, 128, 0], [200, 128, 0], [200, 128, 0]]. The original remains unchanged under Array value semantics. This does not establish an exact CoW allocation count; C2 separates semantic independence from observed storage.

An intentionally rejected opaque-return counterexample:

```swift
func invalidOpaque(_ invert: Bool) -> some PixelTransform {
    if invert {
        return Invert()
    } else {
        return Clamp(maximum: 200)
    }
}
```

The branches have different underlying types. Returning PixelRule or a fixed composed wrapper can preserve an opaque result while representing dynamic choices; returning any PixelTransform permits erasure directly. The choice should follow extensibility and type-relationship needs, then measurement.

#### Focused SIL experiment

These wrappers use the actual genericApply/existentialApply functions above. The public protocol allows an external caller to supply an unknown conformer. The no-inline annotations preserve named observation points for this experiment; they are not a production recommendation.

```swift
@inline(never)
public func genericInvert(_ pixels: inout [UInt8]) {
    genericApply(Invert(), to: &pixels)
}

@inline(never)
public func knownExistentialInvert(_ pixels: inout [UInt8]) {
    let transform: any PixelTransform = Invert()
    existentialApply(transform, to: &pixels)
}

@inline(never)
public func unknownExistentialApply(
    _ transform: any PixelTransform, to pixels: inout [UInt8]
) {
    existentialApply(transform, to: &pixels)
}
```

Compile the protocol, concrete types, application functions, and these wrappers together with the recorded Swift 6 settings. Compare SILGen, mandatory-pass SIL at Onone, and optimized SIL at O. Follow a wrapper's function_ref targets before concluding that its own lack of witness_method removed dispatch; a call can merely move that work into another function.

The checked optimized artifact removes witness dispatch from the reachable known-Invert implementations for both wrappers. The unknown-existential path retains a witness call. This is a compiler observation for this source/toolchain/configuration, not a language guarantee or timing result. The validation record identifies the commands and checked function bodies.

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
