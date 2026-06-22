---
tags:
  - swift
  - ios
  - interview-prep
  - interop-modules-distribution
  - resilience
  - abi
  - module-stability
  - binary-frameworks
---
## 0. Rubric snapshot

**Rubric expectation**

Know the practical distinction between **source compatibility**, **ABI stability**, and **module stability**. The rubric frames this as a strong senior / emerging staff topic, and as part of staff-level public SDK design under SemVer and binary compatibility constraints.

**Caveats**

Many teams confuse these and accidentally promise compatibility they do not actually provide.

**You should be able to answer**

- What is the difference between ABI stability and source compatibility?
- Why does module stability matter for binary-distributed frameworks?

**You should be able to do**

- Explain how to ship a binary SDK across Xcode versions.
- Identify which API and implementation changes are breaking.
- Decide when to enable library evolution / `BUILD_LIBRARY_FOR_DISTRIBUTION`.

---

## 1. Core mental model

There are three different compatibility questions:

```text
Source compatibility:
Can client source code still compile after the library changes?

ABI compatibility:
Can an already-compiled client binary keep running with a newer library binary?

Module stability:
Can a Swift compiler import a prebuilt Swift module produced by a different Swift compiler version?
```

These solve different problems.

**Source compatibility** is about the user’s code. If they update your SDK and recompile, does their source still build? Renaming a public method, changing an argument label, removing an enum case, or tightening a generic constraint can break source compatibility.

**ABI compatibility** is about already-compiled binaries. If an app was compiled against SDK version 1.0, can it load SDK version 1.1 without recompiling? This is what matters for OS frameworks, plug-in systems, and binary SDKs distributed separately from clients.

**Module stability** is about Swift compiler compatibility. Swift’s binary `.swiftmodule` format is compiler-version-specific, so a client using a different Xcode / Swift compiler may fail to import a prebuilt Swift framework unless the framework ships a stable textual module interface, usually a `.swiftinterface`. Swift.org describes module stability as allowing Swift modules built with different compiler versions to be used together in one app. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))

The key idea:

```text
Source compatibility protects recompilation.
ABI stability protects already-compiled binaries.
Module stability protects cross-compiler import of prebuilt Swift modules.
```

Swift 5.0 introduced stable ABI for Swift itself on Apple platforms, meaning apps can use the Swift runtime and standard library from the OS instead of bundling their own copy. That does **not** automatically make every third-party Swift framework ABI-stable. For your own binary SDK, you need to build and evolve it under the library-evolution model. ([Swift.org](https://swift.org/blog/abi-stability-and-more/ "ABI Stability and More | Swift.org"))

---

## 2. Essential mechanics

### Source compatibility

Source compatibility means the client can update the dependency and recompile without changing their source.

Source-compatible additive change:

```swift
// SDK v1
public struct AnalyticsClient {
    public func track(_ event: Event) {}
}

// SDK v2
public struct AnalyticsClient {
    public func track(_ event: Event) {}

    // Additive public API: usually source-compatible.
    public func flush() async throws {}
}
```

Source-breaking change:

```swift
// SDK v1
public func track(_ event: Event)

// SDK v2
public func track(event: Event)
// Existing client code calling track(myEvent) no longer compiles.
```

Source compatibility is not the same as behavioral compatibility. This can be source-compatible but still dangerous:

```swift
// SDK v1
public func retry(count: Int = 1)

// SDK v2
public func retry(count: Int = 5)
```

Existing source still compiles, but behavior changes after recompilation. For SDKs, changing defaults is often a behavioral breaking change even if the compiler accepts it.

---

### ABI stability and library evolution

ABI compatibility means already-compiled clients can keep working with a newer framework binary.

Without library evolution, the compiler can make aggressive assumptions about public type layout and implementation details. That is good for optimization but bad for binary replacement.

With library evolution enabled, the compiler preserves flexibility across the resilience boundary. Swift.org describes library evolution support as allowing binary frameworks to make certain additive API changes while remaining binary-compatible with previous versions. It should be used when a framework is built and updated separately from its clients. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))

In Xcode, this is enabled with:

```text
BUILD_LIBRARY_FOR_DISTRIBUTION = YES
```

Swift.org states that this Xcode setting turns on both library evolution and module stability. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))

Command-line equivalent:

```bash
swiftc Sources/*.swift \
  -module-name MySDK \
  -emit-library \
  -emit-module \
  -emit-module-interface \
  -enable-library-evolution
```

Important implication:

```text
Do not wait until v2.0 to enable library evolution for a binary SDK.
Turning it on later is itself binary-incompatible.
```

Swift.org explicitly notes that frameworks built without library evolution do not provide binary compatibility guarantees, and enabling it later is binary-incompatible. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))

---

### Module stability

Module stability lets clients import a prebuilt Swift module even when their compiler is not the exact same one used to build the framework.

Without module stability:

```text
Vendor builds MySDK.swiftmodule with Swift compiler A.
Client tries to import it with Swift compiler B.
Import may fail because .swiftmodule is compiler-version-specific.
```

With module stability:

```text
Vendor ships MySDK.swiftinterface.
Client compiler reads the textual interface.
Client can import the binary framework across supported Swift compiler versions.
```

This matters for `.xcframework` distribution:

```text
MySDK.xcframework
├── ios-arm64
│   ├── MySDK.framework
│   └── Modules/MySDK.swiftmodule
│       ├── arm64-apple-ios.swiftinterface
│       └── arm64-apple-ios.abi.json / related metadata
```

Module stability is not a promise that every future Swift language feature can be consumed by older compilers. A `.swiftinterface` is source-like Swift; if it exposes syntax or declarations the client compiler does not understand, import can still fail. The practical rule is: build binary SDKs with a toolchain and public interface compatible with the client Xcode versions you claim to support.

---

### Resilience boundaries

A resilience boundary is the boundary between a library and a client where the client compiler must not assume too much about the library’s future implementation.

For example, with library evolution enabled, a public non-`@frozen` struct can add stored properties later without breaking ABI:

```swift
// SDK v1
public struct UserProfile {
    public var id: String
    public var displayName: String
}

// SDK v2, resilient if built correctly
public struct UserProfile {
    public var id: String
    public var displayName: String

    // ABI-resilient for a non-frozen struct under library evolution.
    private var experimentBucket: Int = 0
}
```

But if the struct is `@frozen`, you promised its stored layout will not change:

```swift
@frozen
public struct UserProfile {
    public var id: String
    public var displayName: String
}
```

Adding, removing, or reordering stored properties in a frozen public struct is ABI-breaking. SE-0260 defines library evolution mode and explains that `@frozen` opts a type out of layout flexibility for optimization. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0260-library-evolution.md "swift-evolution/proposals/0260-library-evolution.md at main · swiftlang/swift-evolution · GitHub"))

---

## 3. Common traps and misconceptions

### Trap 1: “Swift has ABI stability, so my SDK is ABI-stable”

Bad assumption:

```text
Swift ABI stability exists on Apple platforms.
Therefore, my third-party Swift framework can be safely replaced without recompiling clients.
```

Better:

```text
Swift's runtime / standard library ABI stability is not the same as my SDK's ABI stability.
A binary SDK needs library evolution enabled from the beginning and must avoid ABI-breaking public changes.
```

Swift 5 ABI stability on Apple platforms means the OS can provide Swift runtime and standard library compatibility. It does not mean your framework’s public ABI is automatically resilient. ([Swift.org](https://swift.org/blog/abi-stability-and-more/ "ABI Stability and More | Swift.org"))

---

### Trap 2: “Source-compatible means binary-compatible”

A change can be source-compatible but ABI-breaking.

Example:

```swift
// SDK v1
@frozen
public struct Point {
    public var x: Double
    public var y: Double
}

// SDK v2
@frozen
public struct Point {
    public var x: Double
    public var y: Double
    public var z: Double
}
```

Client source may still compile after adjustments or if it never constructs `Point` directly, but an already-compiled client may have assumptions about layout. For a frozen public struct, adding a stored property breaks ABI.

---

### Trap 3: “Module stability means I can use any Swift feature in a binary SDK”

Module stability helps compilers import prebuilt modules across compiler versions, but the public `.swiftinterface` still has to be understandable to the client compiler.

Risky:

```swift
// SDK public interface built with a newer compiler feature
public func transform<T: ~Copyable>(_ value: consuming T)
```

A client using an older Swift compiler that does not understand the exposed syntax cannot import the interface.

Better:

```swift
// Keep the public interface within the supported compiler matrix.
public func transform(_ value: Data) -> Data
```

Use newer language features internally where possible. Avoid exposing them publicly unless you intentionally raise your minimum supported compiler version.

---

### Trap 4: “`@frozen` is just an optimization annotation”

`@frozen` is a promise. It tells clients they may rely on the enum cases or struct stored layout not changing in ways that would require resilience.

Risky:

```swift
@frozen
public enum PaymentState {
    case pending
    case paid
}
```

Later:

```swift
@frozen
public enum PaymentState {
    case pending
    case paid
    case refunded // Breaks the frozen promise.
}
```

For non-frozen public enums in resilient libraries, clients should switch with `@unknown default`. SE-0192 introduced handling for future enum cases and explains why plain `default` hides useful exhaustiveness diagnostics. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md "swift-evolution/proposals/0192-non-exhaustive-enums.md at main · swiftlang/swift-evolution · GitHub"))

---

## 4. Direct answers to rubric questions

### Q1. What is the difference between ABI stability and source compatibility?

**Source compatibility** means client source code still compiles after updating the library.

**ABI stability** means an already-compiled client binary can keep running with a newer version of the library binary, without recompilation.

Example:

```swift
// SDK v1
public struct Config {
    public var timeout: TimeInterval
}

// SDK v2
public struct Config {
    public var timeout: TimeInterval
    private var retryJitter: TimeInterval = 0.1
}
```

This can be ABI-compatible if `Config` is resilient and the SDK was built with library evolution enabled. But if the struct was `@frozen`, the same change is ABI-breaking.

Interview version:

> Source compatibility asks whether clients can recompile their source after a library update. ABI stability asks whether already-compiled clients can run against a newer binary without recompiling. They often overlap, but they are not the same. For a Swift binary SDK, I need library evolution enabled and I need to treat ABI-public declarations as a contract, especially public types, protocol requirements, frozen structs/enums, public symbols, and inlinable code.

---

### Q2. Why does module stability matter for binary-distributed frameworks?

Because Swift’s normal `.swiftmodule` files are tied to a specific compiler version. A binary SDK vendor cannot assume all clients use the exact same Xcode version.

Module stability lets the SDK ship a stable textual `.swiftinterface` so the client’s compiler can import the framework across supported compiler versions. Swift.org describes module stability as allowing Swift modules built with different compiler versions to be used together in one app. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))

Interview version:

> Module stability matters because binary frameworks are consumed at compile time. If I ship only compiler-specific `.swiftmodule` files, clients may fail to import the framework when they use a different Xcode. Building for distribution emits stable module interfaces, so clients can import the SDK across supported compiler versions. It solves the compiler-import problem; it does not by itself make all API changes binary-compatible.

---

## 5. Code probe

There is no code probe in the rubric for E7. Following the section template’s fallback style, use a minimal example, counterexample, and production example instead.

### Minimal example: resilient public type

```swift
// SDK v1
public struct AccountSummary {
    public let id: String
    public let balance: Decimal

    public init(id: String, balance: Decimal) {
        self.id = id
        self.balance = balance
    }
}
```

SDK built with:

```text
BUILD_LIBRARY_FOR_DISTRIBUTION = YES
```

Later:

```swift
// SDK v2
public struct AccountSummary {
    public let id: String
    public let balance: Decimal

    // Private implementation detail.
    private let riskScore: Int

    public init(id: String, balance: Decimal) {
        self.id = id
        self.balance = balance
        self.riskScore = 0
    }
}
```

### What happens?

```text
No runtime output.

Compatibility result:
- Source compatibility: usually preserved.
- ABI compatibility: preserved only if the type is resilient and the framework was built with library evolution.
- Module stability: preserved only if the framework ships a stable module interface compatible with the client's compiler.
```

### Why?

The client interacts through public accessors and metadata instead of assuming the exact stored layout. Across a resilience boundary, the compiler is more conservative so the library can change certain implementation details later.

```text
Client source
   |
   | imports stable .swiftinterface
   v
Client binary
   |
   | calls ABI-public symbols/accessors
   v
SDK binary
   |
   | owns resilient layout/implementation
   v
Runtime behavior
```

---

### Counterexample: frozen layout promise

```swift
// SDK v1
@frozen
public struct AccountSummary {
    public let id: String
    public let balance: Decimal
}
```

Later:

```swift
// SDK v2
@frozen
public struct AccountSummary {
    public let id: String
    public let balance: Decimal
    private let riskScore: Int
}
```

### What happens?

```text
No direct compiler error necessarily at the point of editing the source.

Compatibility result:
- This is an ABI-breaking change for a frozen ABI-public struct.
- Existing binaries may have layout assumptions that are no longer valid.
- The SDK has broken its @frozen promise.
```

### Why?

`@frozen` trades future layout flexibility for optimization. SE-0260 describes `@frozen` as a promise that stored properties for a struct, or cases for an enum, will not be added, removed, or reordered. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0260-library-evolution.md "swift-evolution/proposals/0260-library-evolution.md at main · swiftlang/swift-evolution · GitHub"))

---

### Production example: binary SDK distribution

```text
Goal:
Ship MyAnalyticsSDK.xcframework to clients using multiple Xcode versions.

Required:
- Build as XCFramework.
- Enable BUILD_LIBRARY_FOR_DISTRIBUTION.
- Ship .swiftinterface files.
- Keep public interface compatible with supported Swift compiler versions.
- Avoid exposing implementation-only dependency types.
- Treat public ABI as a SemVer-governed contract.
- Run CI import tests using each supported Xcode version.
```

A production SDK release pipeline should test at least:

```bash
# Pseudocode: each CI job uses a different supported Xcode.
xcodebuild \
  -scheme MySDK-Package-Consumer \
  -destination "generic/platform=iOS" \
  build
```

And for app integration:

```bash
xcodebuild \
  -workspace ConsumerApp.xcworkspace \
  -scheme ConsumerApp \
  -destination "generic/platform=iOS" \
  build
```

---

## 6. Exercise

### Problem

You need to ship a binary SDK across Xcode versions. Explain what guarantees matter and which changes are breaking.

### Bad / naive version

```text
"We will ship an XCFramework. Swift is ABI-stable now, so clients should be fine across Xcode versions."
```

### What is wrong?

```text
This confuses Swift runtime ABI stability with the SDK's own ABI stability.

It ignores:
- module stability
- library evolution
- dependency stability
- .swiftinterface compatibility
- public ABI promises
- SemVer boundaries
- client Xcode compatibility testing
```

### Improved version

```text
Binary SDK compatibility plan

1. Distribution format
   - Ship an XCFramework.
   - Include device and simulator slices as needed.
   - Sign/notarize as required by the distribution channel.

2. Swift build settings
   - Enable BUILD_LIBRARY_FOR_DISTRIBUTION=YES from the first public binary release.
   - This emits stable module interfaces and enables library evolution behavior.

3. Public API policy
   - Treat every public/open declaration as source API.
   - Treat every ABI-public declaration as binary API.
   - Avoid exposing third-party dependency types in public signatures.
   - Avoid @frozen unless the layout/cases are truly permanent.
   - Avoid @inlinable unless cross-module optimization is worth freezing implementation details.

4. Compatibility matrix
   - Define supported Xcode / Swift compiler versions.
   - Define minimum OS deployment target. For modern iOS app work here: iOS 18+.
   - Test importing and building from a sample consumer app under every supported Xcode.

5. SemVer policy
   - Patch: bug fixes, no public source/ABI break.
   - Minor: additive APIs, ABI-compatible additions, behavior changes only when safe.
   - Major: source-breaking or ABI-breaking changes.

6. Dependency policy
   - Dependencies used in public API become part of the SDK contract.
   - Binary Swift dependencies must also be module-stable if clients import them.
   - Prefer wrapping dependency types behind SDK-owned public types.

7. Release validation
   - Build old client against SDK v1.
   - Replace runtime SDK binary with SDK v2.
   - Confirm old client still launches and exercises the SDK.
   - Rebuild client source against SDK v2.
   - Confirm source compatibility and warnings.
```

### Breaking-change matrix

|Change|Source-breaking?|ABI-breaking?|Notes|
|---|--:|--:|---|
|Add a new public function|Usually no|Usually no|Can still create overload ambiguity|
|Remove a public function|Yes|Yes|Existing source and binaries depend on it|
|Rename a public type|Yes|Yes|Equivalent to remove + add|
|Change argument label|Yes|Usually yes|Swift labels are part of API shape|
|Change return type|Yes|Yes|Existing call sites and symbols change|
|Add defaulted parameter to existing function|Often yes/no depending overloads|Risky|Prefer adding overloads carefully|
|Change default argument value|No|Usually no symbol break|Behavioral break after recompilation|
|Add requirement to public protocol|Yes|ABI-breaking for conformers|Existing conforming types may fail|
|Add case to non-frozen public enum|Usually source-compatible with `@unknown default`|ABI-compatible under resilience|Clients must handle future cases|
|Add case to `@frozen` public enum|Yes/risky|Yes|Breaks frozen promise|
|Add private stored property to resilient public struct|No|No, if library evolution enabled|Layout stays opaque across boundary|
|Add stored property to `@frozen` public struct|Maybe source-compatible|Yes|Layout promise broken|
|Change internal helper not ABI-public|No|No|Unless used by `@inlinable` code|
|Change `@usableFromInline` symbol|Usually no source break|Can be ABI-breaking|It is ABI-public despite not being source-public|
|Enable library evolution after v1|N/A|Yes|Swift.org notes this is binary-incompatible|
|Expose new Swift syntax in `.swiftinterface`|Maybe|Import-breaking for older compilers|Raises minimum supported compiler|

### Why this is better

It separates the three actual compatibility dimensions:

```text
Can source recompile?
Can old binaries run?
Can supported compilers import?
```

That is the difference between casually shipping a framework and operating a serious binary SDK.

---

## 7. Production guidance

Use this in production when:

```text
- Shipping a closed-source Swift SDK.
- Shipping an XCFramework to external clients.
- Supporting multiple client Xcode versions.
- Maintaining public APIs under SemVer.
- Building SDKs consumed by multiple app teams on different release schedules.
- Designing OS-like frameworks, plugin APIs, analytics SDKs, payments SDKs, or enterprise SDKs.
```

Be careful when:

```text
- Marking structs/enums @frozen.
- Adding @inlinable for performance.
- Exposing third-party types in public signatures.
- Adding public protocol requirements.
- Changing default arguments.
- Changing public generic constraints.
- Relying on generated Codable/Equatable/Hashable behavior in public models.
- Using new Swift language features in public APIs while claiming older Xcode support.
```

Avoid when:

```text
- The library is always built from source together with the app.
- The package is internal and all clients recompile together.
- You do not need binary distribution.
- You want maximum optimization and have no binary-compatibility boundary.
```

Debugging checklist:

```text
1. Is the failure at compile time, link time, load time, or runtime?
2. Is the client consuming source, .swiftmodule, or .swiftinterface?
3. Was the framework built with BUILD_LIBRARY_FOR_DISTRIBUTION?
4. Did a public or ABI-public symbol change?
5. Did a public dependency type leak into the interface?
6. Did the .swiftinterface expose syntax unsupported by the client compiler?
7. Did a non-frozen enum require @unknown default at client switch sites?
8. Did a supposedly internal helper become ABI-public through @usableFromInline?
9. Did a frozen struct/enum change layout or cases?
10. Was the old client binary tested against the new framework binary without recompilation?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> ABI stability means Swift apps work across versions, and module stability means modules work across compiler versions.

This is too vague and risks mixing runtime, compiler, and library promises.

### Senior answer

> Source compatibility means clients can recompile after an SDK update. ABI compatibility means already-built clients can run with a new SDK binary. Module stability means a client compiler can import a prebuilt Swift framework from another compiler version via stable interfaces. For binary frameworks, I would enable `BUILD_LIBRARY_FOR_DISTRIBUTION`, avoid freezing public layout too early, and test supported Xcode versions.

### Staff-level answer

> I separate compatibility into source, ABI, and compiler-import dimensions. For a binary Swift SDK, Swift’s platform ABI stability is only the baseline; my SDK still needs library evolution enabled from its first binary release, stable module interfaces, dependency hygiene, and a SemVer policy that distinguishes source breaks, ABI breaks, and behavioral breaks. I would design public types to preserve resilience, avoid exposing dependency types, treat `@frozen`, `@inlinable`, and `@usableFromInline` as long-term contracts, and run CI that validates both recompilation and binary replacement across supported Xcode versions.

Staff-level questions to ask:

```text
Which Xcode / Swift compiler versions do we explicitly support?
Are clients consuming source, binary frameworks, or both?
Can clients recompile, or must old binaries continue to run?
Do public signatures mention third-party dependency types?
Which public structs/enums are truly stable enough to freeze?
Are we using @inlinable for measured performance reasons or prematurely?
Do we test old-client/new-framework binary replacement?
What is our SemVer policy for source, ABI, and behavioral changes?
```

---

## 9. Interview-ready summary

Source compatibility, ABI stability, and module stability answer different questions. Source compatibility means client code still compiles after a library update. ABI stability means an already-compiled client binary can run with a newer library binary. Module stability means a Swift compiler can import a prebuilt module built with a different Swift compiler version, usually through `.swiftinterface` files. For a binary Swift SDK, I would enable `BUILD_LIBRARY_FOR_DISTRIBUTION` from the first release, avoid unnecessary `@frozen` and `@inlinable`, keep dependency types out of public signatures, and test both recompilation and binary replacement across the supported Xcode matrix.

---

## 10. Flashcards

Q: What is source compatibility?
A: Client source code still compiles after updating the dependency.

Q: What is ABI compatibility?
A: An already-compiled client binary can run with a newer compatible library binary without recompilation.

Q: What is module stability?
A: The ability for Swift modules built with one compiler version to be imported by clients using another supported compiler version, typically through `.swiftinterface` files.

Q: Does Swift ABI stability automatically make a third-party SDK ABI-stable?
A: No. Swift’s ABI stability covers the Swift runtime / standard library on Apple platforms. A third-party binary SDK needs library evolution and careful public ABI management.

Q: What does `BUILD_LIBRARY_FOR_DISTRIBUTION` do?
A: In Xcode, it enables library evolution and module stability for framework distribution.

Q: Why is enabling library evolution later dangerous?
A: Frameworks built without library evolution provide no binary compatibility guarantee; enabling it later is itself binary-incompatible.

Q: What does `@frozen` promise?
A: A public struct’s stored properties or enum’s cases will not change in layout-relevant ways, allowing client-side optimizations but reducing future evolution flexibility.

Q: Why is `@unknown default` important?
A: It handles future enum cases while still letting the compiler warn when known cases are not explicitly handled.

Q: Can a source-compatible change be ABI-breaking?
A: Yes. For example, adding a stored property to a `@frozen` public struct may not obviously break source but breaks the ABI promise.

Q: Can an ABI-compatible change be behavior-breaking?
A: Yes. Changing a default retry count, timeout, sorting order, or error behavior may preserve ABI but still break client expectations.

---

## 11. Related sections

- [[B11 — Library evolution and resilience-aware API design]]
- [[E6 — Import visibility and dependency leakage]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[A7 — Functions, parameter labels, defaults, and API surface]]
- [[A12 — Availability and conditional compilation]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — E7, tier placement, and staff-level compatibility emphasis.
- Swift.org — “Library Evolution in Swift.” ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))
- Swift.org — “ABI Stability and More.” ([Swift.org](https://swift.org/blog/abi-stability-and-more/ "ABI Stability and More | Swift.org"))
- Swift Evolution — SE-0260: “Library Evolution for Stable ABIs.” ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0260-library-evolution.md "swift-evolution/proposals/0260-library-evolution.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Evolution — SE-0192: “Handling Future Enum Cases.” ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md "swift-evolution/proposals/0192-non-exhaustive-enums.md at main · swiftlang/swift-evolution · GitHub"))