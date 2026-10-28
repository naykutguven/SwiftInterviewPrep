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

**Learning goals:**

- Separate source, ABI, textual-module, behavior, and deployment compatibility.
- Compare an unchanged executable with a rebuilt/maintained client under a compatible resilient library update.
- Inspect frozen-layout commitments and verify interfaces, platform slices, linking, embedding, and runtime resolution as separate release gates.

**Prerequisites:** [[B11 — Library Evolution and Resilience-Aware API Design]] for library evolution and [[A5 — Pattern Matching and Exhaustive Control Flow]]/[[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for enum fallbacks/default arguments. [[E5 — Modules, packages, targets, products, and resources]]/[[E6 — Import Visibility and Dependency Leakage]] cover package, module loading and linkage; [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]] explains client-visible function bodies.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A customer keeps yesterday’s executable while installing tomorrow’s library. Its old instructions, copied defaults, and existing conformances remain in that executable. The new library must still understand that client’s calls and values.

**Source compatibility** concerns rebuilding source. **ABI compatibility** concerns already-built code. **Module stability** concerns importing a supported textual interface with other compilers. Behavior and deployment are additional contracts. Building an XCFramework packages slices; it does not by itself establish any complete release matrix. [Library evolution](https://www.swift.org/blog/library-evolution/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Build a version clients can remain linked to

Shared E7SDK v1 source. Build each library version separately with the baseline plus -parse-as-library -O -enable-library-evolution -emit-module -emit-library, the same module name and stable @rpath install name. Keep v1/v2 outputs separate; do not compile both versions together. The published-block checks additionally object-compile each module; replacement controls produce the dylibs.

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

**Walkthrough:** Reading hides its representation behind public construction/access. DeliveryState is a nonfrozen resilient enum. The client implements MessageProvider.title. attempts supplies a caller-side default argument. These are separate places where evolution can matter.

### Make a deliberate compatible update

Shared E7SDK v2 source, compiled into its own directory. The initializer/getter retain their public contract, while Reading’s private representation changes. The enum gains paused, the protocol gains a requirement with a suitable same-module default witness, and attempts changes its default.

```swift
public struct Reading {
    private let implementationLabel: String
    private let storage: [Int]
    public init(_ value: Int) {
        implementationLabel = "v2"
        storage = [value, 99]
    }
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

**Walkthrough:** Library evolution allows this chosen nonfrozen representation change. The old client can use a new enum case through its fallback and an added suitable default witness, yet retain its original default argument. That is ABI viability with changed behavior, not behavioral identity or permission for arbitrary protocol changes.

### Frozen includes private stored fields

Shared FrozenSDK v1, a separate comparison-only module. Frozen representation is an ABI commitment, including storage not visible to ordinary source clients.

```swift
@frozen public struct FrozenReading {
    public let value: Int
    public init(_ value: Int) { self.value = value }
}
```

## Rules and pitfalls

### An unchanged initializer does not make a frozen layout edit compatible

**Common pitfall:** Adding a private field to a previously shipped frozen type. This is a separately compiled v2 control; no incompatible binary replacement is run.

**Mistake:** Compile-only.

```swift
@frozen public struct FrozenReading {
    public let value: Int
    private let reserved: Int
    public init(_ value: Int) {
        self.value = value
        reserved = 0
    }
}
```

**Consequence:** Both versions compile with the same source initializer, yet the new private field changes the promised layout.

**Why:** Existing clients can contain layout assumptions for a frozen value. Source visibility of the field does not erase that ABI commitment.

**Correction:** Choose resilient representation before shipping when independent storage evolution is required. This is a separate repair model, not permission to remove frozen from a shipped ABI.

```swift
public struct EvolvingReading {
    private let storage: [Int]
    public init(_ value: Int) { storage = [value, 99] }
    public var value: Int { storage[0] }
}
```

**What changes:** The unexposed representation can evolve under the library-evolution contract. The real E7SDK v1/v2 pair tests compatible storage replacement; the frozen pair is inspected with the ABI digester, never executed incompatibly.

### A rebuild is a different experiment from replacement

**Common pitfall:** Recompiling the consumer each time a library changes, then calling that an old-client binary test.

**Mistake:** This command replaces the consumer before testing v2:

```sh
swiftc -O -I v2 Client.swift -L v2 -lE7SDK -o client
./client
```

**Consequence:** New enum knowledge, conformances and caller-emitted defaults can conceal what an already-shipped executable would do.

**Why:** A rebuild gives the compiler a new interface and creates different client instructions. ABI replacement must preserve the old executable.

**Correction:** Preserve the client and change only the compatible dylib between new process launches:

```sh
shasum -a 256 client
cp v2/libE7SDK.dylib live/libE7SDK.dylib
./client
shasum -a 256 client
```

**What changes:** Matching hashes establish that the client was not rebuilt. The fixture builds that client against v1 with an rpath to live, then compares a separate rebuilt and maintained consumer. No in-process hot swap is performed.

## Worked example

### One old client encounters four changed library behaviors

**Setup:** Compile this ordinary optimized client against E7SDK v1. The block check links its object; the separate replacement fixture builds the dylib/rpath variant of this exact source.

**Predict:** Which results come from old client instructions and which from the loaded library?

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
        print(
            copy.value, label(currentState()), render(ClientProvider()), attempts(),
            separator: " | ")
    }
}
```

**Outcome:** With v1 the output is 42 | queued | hello | 1. Loading v2 with the identical executable gives 42 | unrecognized | hello + default | 1. Rebuilding the old source changes the last value to 5 and warns about paused.

**Why:** Reading’s accessor preserves its value, the old enum switch uses its fallback, the evolved protocol supplies its suitable default witness, and the old caller keeps default 1. A maintained switch below explicitly handles paused.

## Interview questions

### Why does an unchanged client retain its old default argument while calling a new protocol default witness?

**Answer:** The default argument was emitted into the original caller, while this added requirement uses the resilient protocol/default-witness machinery supplied by the evolved library. They are distinct locations of behavior.

**Example:** Ordinary optimized E7SDK v1 consumer. Replacement uses the same calls in the preserved worked client.

```swift
import E7SDK

struct Provider: MessageProvider { func title() -> String { "hello" } }
print(render(Provider()), attempts(), separator: " | ")
```

**Walkthrough:** The v1-linked check prints hello | 1. In the replacement experiment, the unchanged caller prints hello + default | 1, while its v2 rebuild prints hello + default | 5.

**Boundary:** This default must satisfy the supported protocol-evolution conditions. Removing the default in a separate v2 source control breaks source conformance; do not run that incompatible replacement.

### What does an interface-only import prove with one installed compiler, and what still needs a supported-Xcode matrix?

**Answer:** It proves that this compiler/SDK/target can reconstruct the supplied module from its textual interface after the binary module is withheld. Module stability across supported compilers still needs those actual combinations.

**Example:** Separate ordinary v2 client; validation repeats it with interface-only module loading.

```swift
import E7SDK

func readPublicValue() -> Int { Reading(42).value }
precondition(readPublicValue() == 42)
```

**Walkthrough:** The direct block checks the public operation; the additional control stages only v2’s swiftinterface and recompiles this source in an isolated search directory. It cannot silently use a binary swiftmodule.

**Boundary:** One local Swift 6.4 compiler is one data point. Interface feature syntax, SDK/runtime availability, and platform architecture still constrain distribution.

### Why can a successfully built binary package still fail to load its framework?

**Answer:** Build, link, framework delivery, and runtime resolution occur at different stages. A framework can be linked and copied beside an executable while its @rpath install name remains unreachable. A **runpath** is a directory the loader searches when resolving that install name.

**Example:** After producing the E7SDK.xcframework from the checked platform slices, use this complete local binary consumer manifest:

```swift
// swift-tools-version: 6.4
import PackageDescription

let package = Package(
    name: "BinaryConsumer", platforms: [.macOS(.v13), .iOS(.v16)],
    targets: [
        .binaryTarget(name: "E7SDK", path: "Artifacts/E7SDK.xcframework"),
        .executableTarget(
            name: "BinaryConsumer", dependencies: ["E7SDK"],
            swiftSettings: [
                .defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ],
            linkerSettings: [
                .unsafeFlags(
                    [
                        "-Xlinker", "-rpath", "-Xlinker",
                        "@executable_path",
                    ], .when(platforms: [.macOS]))
            ]),
    ], swiftLanguageModes: [.v6])
```

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

**Walkthrough:** The final macOS root-executable manifest adds @executable_path to its runpaths. The checked default Swift Build backend copies the macOS framework beside the executable; the process can then resolve @rpath/E7SDK.framework/E7SDK and prints XCFramework consumer: 42. A separate manifest control without that runpath distinguishes successful build/link from a loader failure.

**Boundary:** Inspect link commands/symbols for link errors, delivered bundle files for embedding, and install names/rpaths plus loader diagnostics for launch errors. Apps need their normal embedding/signing configuration. This local executable’s unsafe linker flags are not a universal dependency setting; a diagnostic DYLD_FRAMEWORK_PATH override is not a deployment recipe.

### Extension: What does adding a known enum case require in maintained source?

**Answer:** A resilient fallback permits unknown values, but a rebuilt client should deliberately handle a newly known case. Keep unknown behavior meaningful rather than mapping every future value to success.

**Example:** Compile against E7SDK v2 as a separate client, not alongside the old @main declaration.

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
        print(
            copy.value, label(currentState()), render(ClientProvider()), attempts(),
            separator: " | ")
    }
}
```

**Walkthrough:** The v2 maintained client prints 42 | paused | hello + default | 5 without the old-source missing-known-case warning.

**Boundary:** ABI-compatible enum evolution still needs behavior review. The separate old-source rebuild retains unrecognized while warning; runtime fallback is not a substitute for maintaining source.

## Focused exercise

**Task:** Build an optimized consumer against resilient v1, replace only its compatible library, then compare that executable with rebuilt and maintained consumers.

**Acceptance checks:**

- Preserve the original executable hash across v1/v2 runs.
- Observe the stated representation, enum, witness and default-argument outcomes.
- Rebuild old source separately, then handle paused in maintained source.
- Do not execute incompatible frozen-layout or missing-default replacements.

**Solution:** Use the complete v1/v2 and worked Client sources above. Build the libraries with matching module names and install names. This actual command fragment records the distinguishing steps; add the shared baseline/SDK/target/module-cache flags to each compile:

```sh
mkdir -p v1 v2 live
swiftc -parse-as-library -O -enable-library-evolution -emit-module -emit-library \
  -module-name E7SDK LibraryV1.swift -emit-module-path v1/E7SDK.swiftmodule \
  -emit-module-interface-path v1/E7SDK.swiftinterface \
  -Xlinker -install_name -Xlinker @rpath/libE7SDK.dylib -o v1/libE7SDK.dylib
swiftc -parse-as-library -O -enable-library-evolution -emit-module -emit-library \
  -module-name E7SDK LibraryV2.swift -emit-module-path v2/E7SDK.swiftmodule \
  -emit-module-interface-path v2/E7SDK.swiftinterface \
  -Xlinker -install_name -Xlinker @rpath/libE7SDK.dylib -o v2/libE7SDK.dylib
cp v1/libE7SDK.dylib live/libE7SDK.dylib
swiftc -parse-as-library -O -I v1 Client.swift -L live -lE7SDK \
  -Xlinker -rpath -Xlinker "$PWD/live" -o client
shasum -a 256 client
./client
cp v2/libE7SDK.dylib live/libE7SDK.dylib
./client
shasum -a 256 client
```

**Reasoning:** The unchanged and rebuilt consumer answer different questions. Record outputs and warnings, not just process success. The ABI digester’s frozen-layout warning is likewise reviewed even if its exit status is zero with error-on-abi-breakage requested. Interface-only imports, separate macOS/iOS/device/Simulator slices, wrong-slice rejection and the binary-package run are independent distribution checks, reported with their actual scope.

## In practice

**Choose it when:** Enable library evolution before promising independent binary/client updates, and declare the supported compiler/platform/runtime matrix.

**Investigate by:** Preserve old executables and inspect interfaces, ABI diagnostics, slice metadata, linked dependencies and runtime delivery separately.

**Strong evidence:** Use actual unchanged-client and distributed-consumer execution, with known exclusions rather than assuming successful packaging proves compatibility.

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
