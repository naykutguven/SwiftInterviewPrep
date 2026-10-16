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

# E7 — Resilience, ABI, and Module Stability

**Goal:** Compare an unchanged executable and rebuilt consumer against a compatible resilient library replacement.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Source, ABI, module-interface, behavioral, and deployment compatibility are different release contracts. An unchanged executable can use a new resilient representation while retaining caller-emitted defaults or bodies. A successful XCFramework build packages artifacts; it does not prove all consumers, runtime paths, or supported compilers work. [Library evolution](https://www.swift.org/blog/library-evolution/)

## Rules and pitfalls

### Choose resilience before promising binary replacement

Build with library evolution when independent library/client updates require it. Keep frozen-layout commitments deliberate, including private storage. Public signatures can remain source-identical while a frozen representation change breaks ABI.

**Common pitfall:** Rebuilding the client during replacement hides binary incompatibility and caller-emitted behavior. Hash and preserve the executable, replace only the compatible dylib between process runs, then rebuild as a separate comparison.

### Test distribution at the consumer

Check supported architecture/platform slices, textual interface imports, linked dependencies, embedding, runtime search paths, resources, and deployment availability. A link failure and a loader failure occur at different stages.

**Common pitfall:** One installed compiler importing an interface cannot establish every supported Xcode version. Maintain a claimed compiler/platform matrix and report untested combinations honestly.

## Interview questions

### Why does an unchanged client retain its old default argument while calling a new protocol default witness?

The default argument expression was emitted by the original client, while the new requirement's default witness is resolved through the evolved protocol/library machinery in this fixture. Those are different locations of behavior. ABI viability does not imply that every old client observes the rebuilt client's defaults.

### What does an interface-only import prove with one installed compiler, and what still needs a supported-Xcode matrix?

It proves that the supplied textual interface is sufficient for that selected compiler and configuration, rather than accidentally relying on a binary swiftmodule. Import with every supported compiler, SDK, and target slice to justify a module-stability claim. One successful compiler remains one data point.

### Why can a successfully built binary package still fail to load its framework? What evidence would distinguish linking, embedding, and runtime search-path failures?

The package can build while its framework is missing, incorrectly embedded, incompatible with the process slice, or unreachable at runtime. Inspect link commands and symbols for link failures, bundle contents for embedding, and loader diagnostics/install names/rpaths for launch-time resolution. Execute the actual distributed consumer.

## Focused exercise

**Task:** Build a consumer against a resilient library, replace it with a compatible version, and compare that unchanged executable with a rebuilt consumer.

**Acceptance checks:**

- The executable hash stays unchanged during library replacement.
- The old fallback and default argument behave as stated; the rebuilt client uses the new default.
- A maintained source switch adds the new known case; no incompatible frozen-layout executable is run.

**Solution:**

Build v1 and v2 as separate versions of E7SDK with library evolution. Use the source and client below; never compile both library versions into one module.

Compile LibraryV1.swift and LibraryV2.swift as separate versions of the same E7SDK module, never together.

LibraryV1.swift:

```swift
public struct Reading {
    private let raw: Int
    public init(_ value: Int) { raw = value }
    public var value: Int { raw }
}
public enum DeliveryState { case queued, finished }
public func currentState() -> DeliveryState { .queued }
public protocol MessageProvider { func title() -> String }
public func render(_ provider: any MessageProvider) -> String { provider.title() }
public func attempts(_ count: Int = 1) -> Int { count }
```

LibraryV2.swift:

```swift
public struct Reading {
    private let implementationLabel: String
    private let storage: [Int]
    public init(_ value: Int) { implementationLabel = "v2"; storage = [value, 99] }
    public var value: Int { storage[0] }
}
public enum DeliveryState { case queued, finished, paused }
public func currentState() -> DeliveryState { .paused }
public protocol MessageProvider {
    func title() -> String
    func suffix() -> String
}
extension MessageProvider {
    public func suffix() -> String { " + default" }
}
public func render(_ provider: any MessageProvider) -> String {
    provider.title() + provider.suffix()
}
public func attempts(_ count: Int = 5) -> Int { count }
```

The public Reading initializer/accessor remain intact; its private representation changes. DeliveryState adds a case, MessageProvider gains a defaulted requirement, and the default argument to attempts changes. These deliberately separate ABI viability from behavioral identity.

Client.swift, compiled against v1:

```swift
import E7SDK

struct ClientProvider: MessageProvider {
    func title() -> String { "hello" }
}
func label(_ state: DeliveryState) -> String {
    switch state {
    case .queued: "queued"
    case .finished: "finished"
    @unknown default: "unrecognized"
    }
}
@main struct Client {
    static func main() {
        let reading = Reading(42)
        let copy = reading
        precondition(copy.value == 42)
        print(copy.value, label(currentState()), render(ClientProvider()), attempts(), separator: " | ")
    }
}
```

Observed with the same optimized Swift 6.4 client binary in the earlier recorded macOS builds:

| Loaded library / client | Output |
| --- | --- |
| v1 / original client | 42 \| queued \| hello \| 1 |
| v2 / identical original client | 42 \| unrecognized \| hello + default \| 1 |
| v2 / original source recompiled | 42 \| unrecognized \| hello + default \| 5 |
| v2 / updated switch source | 42 \| paused \| hello + default \| 5 |

The executable hash stays identical across the first two runs. Each run starts a new process after selecting the library; no in-process replacement is performed. Reading still copies/accesses correctly, the new enum value reaches the old fallback, and the old conformance uses the new default witness. The unchanged client keeps its caller-side default argument.

Updated ClientV2.swift:

```swift
import E7SDK

struct ClientProvider: MessageProvider {
    func title() -> String { "hello" }
}
func label(_ state: DeliveryState) -> String {
    switch state {
    case .queued: "queued"
    case .finished: "finished"
    case .paused: "paused"
    @unknown default: "unrecognized"
    }
}
@main struct Client {
    static func main() {
        let reading = Reading(42)
        let copy = reading
        precondition(copy.value == 42)
        print(copy.value, label(currentState()), render(ClientProvider()), attempts(), separator: " | ")
    }
}
```

The original source rebuild warns about the newly known paused case. This revised switch is warning-free. A consumer that interprets every unknown value as success would still be behaviorally wrong even if the ABI works; unrecognized makes the lack of knowledge explicit.

**Reasoning:** The fixture changes resilient storage, adds an enum case, and adds a suitable defaulted protocol requirement. It demonstrates these chosen compatible changes, not every possible protocol evolution. Exact observed output belongs to the recorded configuration, with behavior review in addition to ABI checks.

## In practice

**Choose it when:** Distribute binaries that can update independently of their clients.

**Investigate by:** Keep unchanged and rebuilt consumers, interface-only imports, and packaged runtime execution as separate gates.

**Strong evidence:** Define a supported release matrix and inspect diagnostics as well as process exit codes.

### Frozen-layout rejection evidence and binary packaging — optional

#### Frozen-layout counterexample

Build these as separate FrozenSDK versions for comparison only.

FrozenV1.swift:

```swift
@frozen public struct FrozenReading {
    public let value: Int
    public init(_ value: Int) { self.value = value }
}
```

FrozenV2.swift:

```swift
@frozen public struct FrozenReading {
    public let value: Int
    private let reserved: Int
    public init(_ value: Int) { self.value = value; reserved = 0 }
}
```

Both definitions compile, and their explicit initializer keeps the same source call shape. The second adds layout to a frozen type, violating the ABI promise. Its private reserved field appears in the emitted interface because clients need the frozen shape.

The checked swift-api-digester ABI comparison reports that reserved was added to a non-resilient type. It returns exit status zero and warning diagnostics even with -error-on-abi-breakage. Retain and inspect the diagnostic output; a release gate cannot rely only on that invocation's status. No incompatible frozen-layout binary is executed.

Build the two SDK versions, preserve the original consumer, and compare recompilation, binary replacement, interface import, and packaging as separate tests. Keep all sources, binaries, and caches in a temporary directory.

A minimal host recipe, using the selected Swift 6.4 compiler:

```sh
mkdir -p v1 v2 live
swiftc -swift-version 6 -enable-upcoming-feature NonisolatedNonsendingByDefault -enable-upcoming-feature InferIsolatedConformances -default-isolation nonisolated -parse-as-library -O -enable-library-evolution -emit-module -emit-library -module-name E7SDK -emit-module-path v1/E7SDK.swiftmodule -emit-module-interface-path v1/E7SDK.swiftinterface -Xlinker -install_name -Xlinker @rpath/libE7SDK.dylib LibraryV1.swift -o v1/libE7SDK.dylib
swiftc -swift-version 6 -enable-upcoming-feature NonisolatedNonsendingByDefault -enable-upcoming-feature InferIsolatedConformances -default-isolation nonisolated -parse-as-library -O -enable-library-evolution -emit-module -emit-library -module-name E7SDK -emit-module-path v2/E7SDK.swiftmodule -emit-module-interface-path v2/E7SDK.swiftinterface -Xlinker -install_name -Xlinker @rpath/libE7SDK.dylib LibraryV2.swift -o v2/libE7SDK.dylib
cp v1/libE7SDK.dylib live/libE7SDK.dylib
swiftc -swift-version 6 -enable-upcoming-feature NonisolatedNonsendingByDefault -enable-upcoming-feature InferIsolatedConformances -default-isolation nonisolated -parse-as-library -O -I v1 Client.swift -L live -lE7SDK -Xlinker -rpath -Xlinker "$PWD/live" -o client
shasum -a 256 client
./client
cp v2/libE7SDK.dylib live/libE7SDK.dylib
./client
shasum -a 256 client
```

The earlier validation record includes additional macOS/iOS compilation configurations with explicit annotations for its comparison settings. This revised source uses the shared nonisolated baseline; see [[Guide Revision Validation]] for the current reruns. Earlier iOS/Simulator evidence is compilation, not execution.

Further acceptance checks:

1. Recompile the same old client source against v2, observe the missing-known-case warning and the new default argument, then compile the updated switch.
2. Stage only the v2 .swiftinterface and compile a client without its binary .swiftmodule. Do this with every toolchain you actually claim to support; only 6.4 is locally checked here.
3. Remove the default suffix implementation from a separate v2 control; confirm the old source no longer conforms. Do not run a knowingly incompatible replacement.
4. Compare frozen-layout ABI dumps and inspect diagnostics, including warnings returned with a successful status.
5. Build distinct framework slices, create an XCFramework, inspect its platform metadata, and compile a consumer against every included slice. Confirm the wrong device/Simulator pairing fails.
6. Run a real binary-package consumer and verify dynamic-framework delivery.

#### Checked SwiftPM binary consumer

Copy the generated E7SDK.xcframework into BinaryConsumer/Artifacts. Package.swift:

```swift
// swift-tools-version: 6.4
import PackageDescription
let package = Package(name: "BinaryConsumer", platforms: [.macOS(.v13), .iOS(.v16)],
    targets: [
        .binaryTarget(name: "E7SDK", path: "Artifacts/E7SDK.xcframework"),
        .executableTarget(name: "BinaryConsumer", dependencies: ["E7SDK"],
                          swiftSettings: [.defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"), .enableUpcomingFeature("InferIsolatedConformances")],
                          linkerSettings: [.unsafeFlags(["-Xlinker", "-rpath", "-Xlinker",
                              "@executable_path"], .when(platforms: [.macOS]))])
    ], swiftLanguageModes: [.v6])
```

Sources/BinaryConsumer/Consumer.swift:

```swift
import E7SDK

@main struct BinaryConsumer {
    static func main() {
        let reading = Reading(42)
        precondition(reading.value == 42)
        print("XCFramework consumer:", reading.value)
    }
}
```

Run swift run BinaryConsumer on macOS. With this final manifest, the checked default Swift Build backend copies the macOS framework beside the executable, the explicit relative runpath resolves it, and the process prints XCFramework consumer: 42.

The runpath is deliberate configuration for this local command-line consumer. Without it, the default backend built successfully and copied the framework, but the process failed to load @rpath/E7SDK.framework/E7SDK. The deprecated native backend passed the original-manifest diagnostic control. A separate DYLD_FRAMEWORK_PATH control also ran the unchanged original executable, confirming the runtime-resolution issue; that environment override is diagnostic evidence, not the published deployment recipe.

The final recipe uses no manual framework copy or environment override. It establishes this arm64/macOS fixture under Swift Build, not a guarantee about every binary target or app embedding layout. An iOS/macOS app should use its normal framework embedding/signing configuration and verify the resulting application. The manifest's unsafe linker flags belong to this root executable example, not a recommended universal dependency setting.

## References

### Related guides

- [[A5 — Pattern Matching and Exhaustive Control Flow]]
- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A12 — Availability and Conditional Compilation]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [Library evolution](https://www.swift.org/blog/library-evolution/)
- [ABI background](https://www.swift.org/blog/abi-stability-and-more/)
- [SE-0260](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0260-library-evolution.md)
- [SE-0192](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md)
- [SE-0487](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md)
- [Apple XCFramework workflow](https://developer.apple.com/documentation/xcode/creating-a-multi-platform-binary-framework-bundle)
- [Module tracking in debug info](https://www.swift.org/blog/module-tracking-in-debug-info/)
