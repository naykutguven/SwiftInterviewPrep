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

**Learning goals:**

- Separate source, ABI, module-interface, behavioral, and persisted-schema compatibility.
- Check resilient/frozen/nonexhaustive enum contracts across real module and package boundaries.
- Distinguish callable symbols, client-visible bodies, mandatory inlining, and library-side specialization using unchanged and rebuilt clients.

**Prerequisites:** [[A5 — Pattern Matching and Exhaustive Control Flow]] for exhaustive matching, [[A11 — Access Control]] for public construction/access, [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]] for dispatch, and [[B9 — Codable and Schema Evolution]] for wire schemas. [[E7 — Resilience, ABI, and Module Stability]]/[[E8 — Binary frameworks, inlineability, and public implementation detail leakage]] expand binary distribution and body visibility.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A client may rebuild immediately with a library update, or continue running an old executable. These are different release scenarios. Source compatibility asks whether supported client source rebuilds with its intended meaning. **ABI compatibility** asks whether compiled code can still use a replacement library’s binary calling and representation contract. **Module stability** concerns importing a textual interface with supported compiler versions.

Behavioral compatibility and persisted schemas are separate again: a function can keep its symbol while changing policy, and a source-compatible property can change a JSON key. A single successful rebuild cannot certify all these contracts. Choose the distribution model and release checks before exposing layout or implementation. [Library evolution](https://www.swift.org/blog/library-evolution/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Representation and implementation are different commitments

Library evolution (`-enable-library-evolution`) keeps eligible public nonfrozen representation behind resilient boundaries. @frozen deliberately publishes layout or closed enum shape under its rules; even a private field can affect a frozen struct’s binary layout. Source access does not undo that commitment. A visible body can also be copied into a client, independently of whether a function’s source name is public or internal.

### Declare the supported construction and observation surface

Compile-only example under -enable-library-evolution. Token hides its stored representation; Point publishes a fixed layout.

```swift
public struct Token {
    private let text: String
    public init(_ text: String) { self.text = text.lowercased() }
    public var value: String { text }
}
@frozen public struct Point {
    public var x: Int
    public var y: Int
    public init(x: Int, y: Int) {
        self.x = x
        self.y = y
    }
}
```

**Walkthrough:** Callers use Token’s initializer/getter without relying on its private storage. Frozen Point’s stored layout is a stronger distribution commitment. Neither declaration alone specifies a persisted schema.

### Shared separate-module setup: StatusSDK v1

Build this as StatusSDK.swift using baseline flags plus `-parse-as-library -enable-library-evolution -module-name StatusSDK -emit-module -emit-module-path /tmp/StatusSDK.swiftmodule -c -o /tmp/StatusSDK.o`. Separate runnable clients add `-I /tmp` and link `/tmp/StatusSDK.o`. Validation uses an isolated temporary directory. The dynamic replacement experiment uses the same source with `-emit-library`, a stable install name and a client rpath, instead of object linkage.

```swift
public enum Status { case ready, stopped }
public func current() -> Status { .ready }
```

## Rules and pitfalls

### All known cases do not close an external resilient enum

**Common pitfall:** Omitting future-case behavior in a separate client of StatusSDK.

**Mistake:** Compile-only; intentionally rejected.

```swift
import StatusSDK

func label(_ value: Status) -> String {
    switch value {
    case .ready: "ready"
    case .stopped: "stopped"
    }
}
```

**Consequence:** Swift 6 rejects the possible unknown values left without a branch.

**Why:** The public nonfrozen enum is imported across a resilient boundary. The compiler cannot assume today’s known cases are its entire future shape.

**Correction:** Add an explicit future policy while still naming known cases.

```swift
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

**What changes:** The fallback supports future values, and @unknown default keeps a diagnostic when a rebuilt source omits a newly known case. It can execute for an omitted known case too.

### An exposed body can expose its helper obligations

**Common pitfall:** Adding public mandatory inlining while calling an ordinary internal helper.

**Mistake:** Compile-only; intentionally rejected.

```swift
internal func implementation() -> Int { 1 }
@inline(always) public func revision() -> Int { implementation() }
```

**Consequence:** The compiler rejects the insufficiently visible helper in the otherwise exposed public body.

**Why:** The new @inline(always) has body-visibility consequences; publishing an implementation creates dependencies for clients that compile it.

**Correction:** Keep policy behind an interface-backed call when clients should not receive the body. This compile-only correction explicitly controls visibility.

```swift
private func implementation() -> Int { 1 }
@export(interface) @inline(always)
public func revision() -> Int { implementation() }
```

**What changes:** Explicit interface export keeps a callable symbol and withholds the body from outside clients. If body exposure is justified, the working @usableFromInline internal helper pattern is a different ABI commitment.

## Worked example

### A future fallback becomes a known-case warning after rebuild

**Setup:** Separate StatusSDK client. The driver prints its result so the same executable can be run against v1 and v2.

**Predict:** What should the old executable do if a compatible v2 returns a new paused case?

```swift
import StatusSDK

func label(_ value: Status) -> String {
    switch value {
    case .ready: "ready"
    case .stopped: "stopped"
    @unknown default: "future"
    }
}
print(label(current()))
```

**Outcome:** With v1 it prints ready. The controlled resilient replacement prints future from the unchanged executable when v2 adds paused and returns it. Rebuilding the unchanged source against v2 warns about paused and still uses future.

**Why:** The binary already contains fallback behavior. A rebuilt source knows paused and receives a maintenance diagnostic. These are separate observations; pasting both declarations into one source module would remove the tested boundary.

## Interview questions

### Which changes need a rebuild, and what must an unchanged binary tolerate?

**Answer:** A source change is evaluated by recompiling supported clients. A compatible resilient binary release must keep existing clients working through supported symbols and permitted representation changes. Private nonfrozen storage can evolve; frozen layout is a stronger promise. Neither check alone certifies behavior or schemas.

**Example:** Compile-only library RepresentationAPI. Use the shared module process with -enable-library-evolution.

```swift
public struct Token {
    private let raw: String
    public init(_ raw: String) { self.raw = raw.lowercased() }
    public var value: String { raw }
}
@frozen public struct Point {
    public let x: Int
    public let y: Int
    public init(x: Int, y: Int) {
        self.x = x
        self.y = y
    }
}
```

**Walkthrough:** Build RepresentationAPI as a resilient module. Its ordinary client below constructs Token and Point through explicit public initializers. The replacement control changes Token’s private String storage to a private reference-storage/generation representation while retaining this accessor contract; the unchanged client still prints abc.

**Boundary:** This demonstrates the specific checked representation change, not arbitrary API removal or layout edits to frozen Point. Textual-interface import by this compiler is not proof that an older compiler understands all new attributes.

Separate runnable RepresentationAPI client. Link RepresentationAPI.o for the basic run; the replacement control links a stable dylib instead:

```swift
import RepresentationAPI

let token = Token("ABC")
let point = Point(x: 1, y: 2)
precondition(point.x == 1 && point.y == 2)
print(token.value)
```

A private field does not evade frozen layout visibility. This independent **intentionally rejected** resilient-library source has an ineligible field type:

```swift
private struct Hidden { let value: Int }
@frozen public struct Frozen {
    private let hidden: Hidden
}
```

### How do ordinary, resilient, frozen, nonexhaustive, and nonexhaustive(warn) enums differ?

**Answer:** They define different source-checking and binary boundaries. A plain nonresilient enum or resilient frozen enum can expose a closed shape. A public resilient nonfrozen enum requires future handling. @nonexhaustive imposes that source policy independently of resilience; its warn form stages the future-value diagnostic.

**Example:** Compile-only DeliveryAPI module; add -package-name DeliveryKit. This explicit-nonexhaustive source is initially built without library evolution to isolate source checking.

```swift
@nonexhaustive
public enum DeliveryStatus: Sendable { case pending, delivered }
public func latestStatus() -> DeliveryStatus { .pending }
```

**Walkthrough:** Build this as separate module DeliveryAPI with package identity DeliveryKit. The consumer below handles both known cases and future values. Additional controls build ordinary, resilient, frozen, explicit nonexhaustive and warn variants from this source and compare external versus same-package clients. [SE-0487](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md)

**Boundary:** The attribute does not enable binary resilience. Same-package code is co-developed and has different source checking. Missing a known case without any fallback remains an error even for the warn form.

Separate runnable external consumer with a different package identity:

```swift
import DeliveryAPI

func statusLabel(_ status: DeliveryStatus) -> String {
    switch status {
    case .pending: "pending"
    case .delivered: "delivered"
    @unknown default: "unknown"
    }
}
precondition(statusLabel(.pending) == "pending" && statusLabel(.delivered) == "delivered")
```

| Separate module configuration | External all-known switch without fallback | Same-package all-known switch |
| --- | --- | --- |
| Plain, nonresilient | Accepted | Accepted |
| Plain, resilient | Rejected for future values in Swift 6 | Accepted |
| Frozen, resilient | Accepted as closed shape | Accepted |
| Explicit @nonexhaustive, nonresilient | Rejected for future values | Accepted |
| @nonexhaustive(warn), nonresilient | Warning for future values | Accepted |

This matrix is checked using actual emitted modules and matching/different -package-name values. @unknown default with an omitted known case warns and executes that fallback. Plain default accepts the omission without that warning. The selected compiler does not reliably promote this known-case warning with -warnings-as-errors; the validation records actual diagnostics and exit status rather than assuming promotion.

The external missing-fallback client is **intentionally rejected** against the published DeliveryAPI:

```swift
import DeliveryAPI

func incomplete(_ status: DeliveryStatus) -> String {
    switch status {
    case .pending: "pending"
    case .delivered: "delivered"
    }
}
```

### Why can library replacement update an interface-backed call while an old client keeps other bodies?

**Answer:** The interface-backed call reaches a library symbol whose implementation is replaced. Client-visible code can already reside in the old executable, so replacing the dylib cannot replace those instructions. Client emission is distinct from inlining.

**Example:** Compile-only RevisionAPI source. Use -O -enable-library-evolution and binary/textual module output; the dynamic replacement build additionally emits a dylib.

```swift
@export(interface) public func libraryRevision() -> Int { 1 }
@export(implementation) @inline(never) public func clientRevision() -> Int { 1 }
@inlinable public func visibleRevision() -> Int { 1 }
@inline(always) public func forcedRevision() -> Int { 1 }
```

**Walkthrough:** The separate optimized client below prints 1,1,1,1 with v1. After changing the four literals to 2 in a compatible v2 library, its unchanged executable prints 2,1,1,1. Rebuilding produces 2,2,2,2. Interface and symbol inspection distinguish the mechanisms. [SE-0497](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)

**Boundary:** visibleRevision’s retained value is an observation for this optimized fixture, not a promise that every inlinable call always inlines. @inline(never) on clientRevision still leaves its definition in the client.

Separate runnable optimized client, initially linking RevisionAPI.o. For replacement use -L/-lRevisionAPI and a stable @rpath install name/client rpath; keep the original executable byte-for-byte unchanged for the middle run.

```swift
import RevisionAPI

print(
    [libraryRevision(), clientRevision(), visibleRevision(), forcedRevision()].map(String.init)
        .joined(separator: ","))
```

### Does specialized publish a new specialized ABI, and is inline(always) just inline(__always)?

**Answer:** @specialized requests selected library-side specializations while retaining a generic API; it does not necessarily publish a new specialized ABI. @inline(always) is mandatory eligible direct inlining with public body-visibility consequences, unlike the older optimizer hint @inline(__always).

**Example:** Compile-only SpecializationAPI with -O -enable-library-evolution and textual module output.

```swift
@specialized(where S == [Int])
public func containsZero<S: Sequence>(_ values: S) -> Bool where S.Element: BinaryInteger {
    for value in values { if value == 0 { return true } }
    return false
}
```

**Walkthrough:** The SpecializationAPI client below exercises Array<Int> and other supported sequence types. The validation inspects optimized SIL for the selected Array<Int> path and generic fallback, and inspects the public interface/symbols without claiming a separate public specialized entry point. [SE-0460](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0460-specialized.md)

**Boundary:** The body performs the same generic predicate under both paths; no speedup is inferred. Partial specialization constraints or a misspelled attribute are not substitutes for the supported declaration. Public inline(always) and helper eligibility are checked separately below.

Separate runnable SpecializationAPI client, linking its object:

```swift
import SpecializationAPI

precondition(containsZero([1, 0, 2]))
precondition(!containsZero([Int]()))
precondition(containsZero([Int8(1), 0]))
precondition(containsZero(AnySequence([1, 0])))
let values = [1, 2, 0]
precondition(containsZero(values[1...]))
```

The older hint accepts this separate compile-only source, while the public inline(always) counterpart in the pitfall rejects its helper. Their spellings are not interchangeable. [SE-0496](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0496-inline-always.md)

```swift
internal func helper() -> Int { 1 }
@inline(__always) public func hinted() -> Int { helper() }
```

For a deliberately exposed body, this working helper policy keeps the helper source-internal while accepting its ABI commitment:

```swift
@usableFromInline internal func helper() -> Int { 1 }
@inlinable public func visible() -> Int { helper() }
```

The selected compiler still rejects this shortcut, despite broader helper eligibility described in the export proposal. It is an **intentionally rejected** implementation limit, so the working example above uses @usableFromInline:

```swift
@export(interface) internal func helper() -> Int { 1 }
@inlinable public func visible() -> Int { helper() }
```

## Focused exercise

**Task:** Add a public enum case and examine both a separate-source rebuild and an unchanged resilient client.

**Acceptance checks:**

- The v1 external consumer handles all known cases and has a future fallback.
- A v2 returning paused reaches future in the unchanged executable; rebuilding unchanged source warns.
- The maintained v2 client explicitly handles paused, and the report keeps source versus binary results separate.

**Solution:** Standalone StatusSDK client using the shared v1 module. For v2, add paused to Status and return it from current(). The replacement control uses a compatible dynamic library.

```swift
import StatusSDK

func label(_ status: Status) -> String {
    switch status {
    case .ready: "ready"
    case .stopped: "stopped"
    @unknown default: "future"
    }
}
print(label(current()))
```

**Reasoning:** The old executable already contains future behavior. A rebuilt source can see paused and warns that it was not named. Add an explicit paused branch in the maintained consumer; a plain default would hide that maintenance signal.

For an independent maintained-v2 example, build the next source as the resilient module **StatusSDKv2** using the shared module command with that module name. The suffix keeps this study fixture separate from v1. In the actual replacement experiment the module remains StatusSDK.

```swift
public enum Status { case ready, stopped, paused }
public func current() -> Status { .paused }
```

Compile this ordinary client with `-I /tmp` and link `/tmp/StatusSDKv2.o`, using the same isolated directory for both artifacts:

```swift
import StatusSDKv2

func label(_ status: Status) -> String {
    switch status {
    case .ready: "ready"
    case .stopped: "stopped"
    case .paused: "paused"
    @unknown default: "future"
    }
}
precondition(label(.ready) == "ready" && label(.stopped) == "stopped")
precondition(label(current()) == "paused")
```

The maintained consumer now names paused and keeps a fallback for later cases. The unchanged-binary experiment separately demonstrates that the old executable had only its future policy.

## In practice

**Choose it when:** Define release gates for the actual source/binary distribution model before publishing layout, helpers, or implementation bodies.

**Investigate by:** Compare ordinary source clients, emitted interfaces, callable symbols, and unchanged executables under recorded configurations.

**Strong evidence:** Demonstrate unchanged and rebuilt behavior, preserve executable hashes during replacement, and distinguish these experiments from an older-compiler or full distribution matrix.

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
