---
tags:
  - swift
  - ios
  - interview-prep
  - interop-modules-distribution
  - swiftpm
  - modules
  - packages
---

# E5 — Modules, Packages, Targets, Products, and Resources

**Learning goals:**

- Distinguish package, product, target, module, source access, and resource ownership.
- Verify an acyclic multi-target package with tests and an ordinary external resource consumer.
- Record effective build settings/artifact paths and distinguish build-derived SBOMs from graph inventories and schema validation.

**Prerequisites:** [[A11 — Access Control]] for access, [[B10 — API Design Guidelines and Swift-Native Surface Design]] for public construction, [[D14 — Approachable Concurrency and Default Isolation]] for isolation settings, and [[F1 — Testing Strategy in Modern Swift]] for test evidence. [[E6 — Import Visibility and Dependency Leakage]]/[[E7 — Resilience, ABI, and Module Stability]] cover dependency exposure and binary delivery.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A consumer selects a package product named ProfileKit but imports modules named ProfileCore, ProfileStore, and ProfileUI. Those names describe different boundaries: a **package** owns a manifest/dependency graph; a **target** supplies sources/resources; an ordinary Swift target builds a **module**; a **product** exposes targets for consumers.

The dependency graph must be acyclic. Access control still decides which declarations callers may use, and each resource target owns its generated bundle. Neither a product name nor Bundle.main creates a package-wide namespace. A **software bill of materials (SBOM)** records dependency inventory; later examples distinguish the resolved graph from dependencies used by a selected build. [PackageDescription](https://docs.swift.org/package-manager/PackageDescription/PackageDescription.html)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Define the graph and its build policies

Shared ProfileFeature fixture. Create three sibling directories ProfileFeature, DeviceMetrics, and Consumer under one temporary parent. All files below are complete; compile them as the package, not independent Swift scripts. The Swift target settings select the shared baseline. DeviceMetrics is a local conditional dependency so SBOM/platform behavior needs no network download.

ProfileUI → ProfileStore → ProfileCore is acyclic; the product exposes ProfileUI and its required modules. The iOS-only edge additionally supplies DeviceMetrics. More targets add interfaces and overhead; this graph alone makes no faster-build claim.

**File:** ProfileFeature/Package.swift

```swift
// swift-tools-version: 6.4
import PackageDescription

let baseline: [SwiftSetting] = [
    .swiftLanguageMode(.v6), .defaultIsolation(nil),
    .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
    .enableUpcomingFeature("InferIsolatedConformances"),
]
let package = Package(
    name: "ProfileFeature", platforms: [.macOS(.v13), .iOS(.v16)],
    products: [.library(name: "ProfileKit", targets: ["ProfileUI"])],
    dependencies: [.package(path: "../DeviceMetrics")],
    targets: [
        .target(name: "ProfileCore", swiftSettings: baseline),
        .target(
            name: "ProfileStore", dependencies: ["ProfileCore"],
            resources: [.process("Resources"), .copy("Static")], swiftSettings: baseline),
        .target(
            name: "ProfileUI",
            dependencies: [
                "ProfileCore", "ProfileStore",
                .product(
                    name: "DeviceMetrics", package: "DeviceMetrics",
                    condition: .when(platforms: [.iOS])),
            ],
            swiftSettings: baseline),
        .testTarget(
            name: "ProfileFeatureTests",
            dependencies: ["ProfileCore", "ProfileStore", "ProfileUI"],
            resources: [.copy("Fixtures")], swiftSettings: baseline),
    ])
```

**File:** ProfileFeature/Sources/ProfileCore/Profile.swift

```swift
public struct Profile: Sendable, Equatable {
    public let id: String
    public let name: String
    public init(id: String, name: String) {
        self.id = id
        self.name = name
    }
}
package func profileLabel(_ profile: Profile) -> String { profile.name }
```

**Walkthrough:** Profile has an explicit public initializer. profileLabel is available to coordinating modules within this package identity, while an ordinary external package cannot call it. A client can import a module carried by the product’s target graph; the product is not a source-access firewall.

### Keep resource lookup in the owning target

Create ProfileStore/Resources/profiles.json containing {"ada":"Ada"} and greeting.txt containing Hello plus a newline. Create ProfileStore/Static/metadata.txt containing copied directory plus a newline. process applies resource processing rules; copy preserves the Static directory. The methods below load those actual paths.

**File:** ProfileFeature/Sources/ProfileStore/Store.swift

```swift
import Foundation
import ProfileCore

public enum StoreError: Error { case missingResource, missingProfile }
public struct BundledProfileStore: Sendable {
    public init() {}
    public func profile(id: String) async throws -> Profile {
        let url = try Self.resource("profiles", extension: "json")
        let values = try JSONDecoder().decode([String: String].self, from: Data(contentsOf: url))
        guard let name = values[id] else { throw StoreError.missingProfile }
        return Profile(id: id, name: name)
    }
    public static func greeting() throws -> String {
        try String(contentsOf: resource("greeting", extension: "txt"), encoding: .utf8)
            .trimmingCharacters(in: .whitespacesAndNewlines)
    }
    public static func metadata() throws -> String {
        try String(
            contentsOf: resource("metadata", extension: "txt", subdirectory: "Static"),
            encoding: .utf8
        )
        .trimmingCharacters(in: .whitespacesAndNewlines)
    }
    private static func resource(
        _ name: String, extension suffix: String, subdirectory: String? = nil
    ) throws -> URL {
        guard
            let url = Bundle.module.url(
                forResource: name, withExtension: suffix, subdirectory: subdirectory)
        else {
            throw StoreError.missingResource
        }
        return url
    }
}
```

**Walkthrough:** Bundle.module is generated for ProfileStore because that target declares resources. Lookup stays behind public store methods rather than exposing the bundle to callers. Missing resources throw instead of silently substituting test/app content. These tiny trusted resources use bounded synchronous reads; they are not an asynchronous file-I/O design.

### A consumer selects the product and imports modules

**File:** ProfileFeature/Sources/ProfileUI/Presenter.swift

```swift
import ProfileCore
import ProfileStore

#if os(iOS)
    import DeviceMetrics
#endif
public struct ProfilePresenter {
    public init() {}
    public func title(for profile: Profile) throws -> String {
        try BundledProfileStore.greeting() + ": " + profileLabel(profile)
    }
    public static var platformLabel: String {
        #if os(iOS)
            DeviceMetrics.label
        #else
            "macOS sample"
        #endif
    }
}
```

**File:** DeviceMetrics/Package.swift

```swift
// swift-tools-version: 6.4
import PackageDescription

let package = Package(
    name: "DeviceMetrics", platforms: [.iOS(.v16)],
    products: [.library(name: "DeviceMetrics", targets: ["DeviceMetrics"])],
    targets: [.target(name: "DeviceMetrics")], swiftLanguageModes: [.v6])
```

**File:** DeviceMetrics/Sources/DeviceMetrics/Metrics.swift

```swift
public enum DeviceMetrics { public static let label = "iOS sample" }
```

**File:** Consumer/Package.swift

```swift
// swift-tools-version: 6.4
import PackageDescription

let package = Package(
    name: "Consumer", platforms: [.macOS(.v13), .iOS(.v16)],
    dependencies: [.package(path: "../ProfileFeature")],
    targets: [
        .executableTarget(
            name: "Consumer",
            dependencies: [.product(name: "ProfileKit", package: "ProfileFeature")],
            swiftSettings: [
                .swiftLanguageMode(.v6), .defaultIsolation(nil),
                .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ])
    ])
```

**File:** Consumer/Sources/Consumer/Consumer.swift

```swift
import Foundation
import ProfileCore
import ProfileStore
import ProfileUI

@main struct Consumer {
    @MainActor static func main() async throws {
        let profile = try await BundledProfileStore().profile(id: "ada")
        let title = try ProfilePresenter().title(for: profile)
        guard title == "Hello: Ada", try BundledProfileStore.metadata() == "copied directory" else {
            fatalError("External resource/API contract failed")
        }
        print(title)
        print(ProfilePresenter.platformLabel)
    }
}
```

**Walkthrough:** The product dependency is ProfileKit; import ProfileKit is not the module spelling. The ordinary consumer constructs only public types and loads resources through their owner. ProfileUI alone uses the package helper. No testable import supplies hidden privileges.

## Rules and pitfalls

### Bundle.main is not the owning package target’s bundle

**Common pitfall:** Looking for a package resource in the consumer/app bundle.

**Mistake:** Compile-only.

```swift
import Foundation

func wrongGreeting() -> URL? { Bundle.main.url(forResource: "greeting", withExtension: "txt") }
```

**Consequence:** The resource can be missing even when the package’s generated resource bundle is correctly built and copied.

**Why:** Bundle.main describes the process/application bundle. ProfileStore’s resource declarations generate a separate target-owned bundle.

**Correction:** Call the owner’s public lookup rather than reimplementing its bundle path.

```swift
import ProfileStore

func expectedGreeting() throws -> String { try BundledProfileStore.greeting() }
```

**What changes:** The owning target controls the process/copy layout and reports packaging failure explicitly. The separate consumer and tests verify those files are actually present.

### A product name need not be an importable module

**Common pitfall:** Importing the library product spelling as if it were a Swift module.

**Mistake:** Intentionally rejected ordinary consumer.

```swift
import ProfileKit
```

**Consequence:** The ordinary consumer rejects the nonexistent ProfileKit module.

**Why:** The product names the consumable target graph; ProfileCore/ProfileStore/ProfileUI are the modules compiled from its Swift targets.

**Correction:** Use the module whose declarations the source needs.

```swift
import ProfileCore

func publicProfile() -> Profile { Profile(id: "ada", name: "Ada") }
```

**What changes:** The client uses the explicit public initializer from ProfileCore. It still does not acquire package access merely by depending on the same product.

### Graph inventory is not a selected-build inventory

**Common pitfall:** Generating a package-graph SBOM and treating every resolved conditional edge as shipped in the macOS artifact.

**Mistake:** This actual graph command can include DeviceMetrics even though the macOS build does not use that iOS-only edge:

```sh
swift package generate-sbom --product ProfileKit --sbom-output-dir /tmp/profile-package-sboms
```

**Consequence:** The inventory can overstate dependencies used by the selected build.

**Why:** A resolved graph and a platform-filtered build are different inputs. Generation success also does not establish schema validity or complete runtime-loaded component coverage.

**Correction:** Generate from the actual selected build and record destination/product/settings; compare the graph document deliberately:

```sh
swift build --product ProfileKit --sbom-spec cyclonedx --sbom-spec spdx \
  --sbom-output-dir /tmp/profile-build-sboms
```

**What changes:** The build route can apply build-time conditions. The validation compares macOS, iOS, and graph outputs, parses their metadata, and preserves schema warnings. An SBOM is inventory rather than a vulnerability/license verdict or reproducible-build proof. [SBOM generation](https://docs.swift.org/latest/documentation/packagemanagerdocs/generatingsboms)

## Worked example

### A test target has its own resource bundle

**Setup:** Add ProfileFeature/Tests/ProfileFeatureTests/Fixtures/greeting.txt containing Test fixture plus a newline. The test target declared above owns that resource; it does not replace ProfileStore’s greeting. This complete four-test source uses the public API and one package helper available under the package identity.

**Predict:** Does Bundle.module in a test source select the same resource bundle as Bundle.module inside ProfileStore?

**File:** ProfileFeature/Tests/ProfileFeatureTests/FeatureTests.swift

```swift
import Foundation
import ProfileCore
import ProfileStore
import ProfileUI
import Testing

@Test @MainActor func profileAndTitle() async throws {
    let profile = try await BundledProfileStore().profile(id: "ada")
    #expect(try ProfilePresenter().title(for: profile) == "Hello: Ada")
    #expect(profileLabel(profile) == "Ada")
}
@Test @MainActor func copiedResource() throws {
    #expect(try BundledProfileStore.metadata() == "copied directory")
}
@Test @MainActor func testBundleIsSeparate() throws {
    let url = try #require(
        Bundle.module.url(forResource: "greeting", withExtension: "txt", subdirectory: "Fixtures"))
    #expect(
        try String(contentsOf: url, encoding: .utf8).trimmingCharacters(in: .whitespacesAndNewlines)
            == "Test fixture")
    #expect(try BundledProfileStore.greeting() == "Hello")
}
@Test @MainActor func missingProfile() async {
    do {
        _ = try await BundledProfileStore().profile(id: "missing")
        Issue.record("Expected absence")
    } catch StoreError.missingProfile {} catch { Issue.record("Unexpected error: \(error)") }
}
```

**Outcome:** The tests distinguish Test fixture from Hello and check metadata, public presentation, package helper use, and missing-profile failure.

**Why:** Bundle.module is generated in the declaring target’s context. A successful test alone can miss external packaging defects, so the ordinary Consumer is a separate acceptance gate.

## Interview questions

### Why can a client depend on ProfileKit, import ProfileCore, and still fail to use its package helper?

**Answer:** The product supplies a module graph, while package access requires matching package identity. An external consumer does not share that identity just because it selects the product.

**Example:** This independent ordinary-client source is intentionally rejected in the consumer package:

**File:** variants/PackageAccess.swift

```swift
import ProfileCore

let profile = Profile(id: "ada", name: "Ada")
let invalid = profileLabel(profile)
```

**Walkthrough:** Public construction succeeds, but the package function is inaccessible. ProfileUI and the package test target can call it because they belong to ProfileFeature.

**Boundary:** Public declarations in a reachable module can still be imported by a consumer; a product is not an access firewall. Internal declarations remain module-only, a narrower boundary than package access.

### How do process/copy rules and resource ownership change lookup in tests and consumers?

**Answer:** The declaring target determines Bundle.module. process applies supported processing rules, while copy preserves the named directory. Call the owner’s API and test the resulting artifact instead of assuming source paths survive unchanged.

**Example:** Independent ordinary-client helper; its methods invoke ProfileStore’s real processed and copied resources:

**File:** variants/ResourceCalls.swift

```swift
import ProfileStore

func resourceChecks() throws {
    let greeting = try BundledProfileStore.greeting()
    let metadata = try BundledProfileStore.metadata()
    precondition(greeting == "Hello" && metadata == "copied directory")
}
```

**Walkthrough:** The greeting lives in the processed resource output and metadata in the preserved Static directory. The worked test uses a different fixture bundle; neither lookup substitutes Bundle.main.

**Boundary:** Processing can be platform-dependent. Validate packaged resources with the actual consumer/destination, and report a missing/case-changed path as a packaging error.

### Why do macOS-build, iOS-build, and standalone package-graph SBOMs differ?

**Answer:** The build route can filter conditional edges for its destination, while a graph inventory may retain resolved edges not used in that build. This fixture’s iOS edge makes the distinction observable.

**Example:** Run from ProfileFeature using the same selected toolchain and explicit SDK/target options for each build:

```sh
swift build --product ProfileKit --sbom-spec cyclonedx --sbom-spec spdx \
  --sbom-output-dir /tmp/macos-profile-sboms
swift build --product ProfileKit --sdk "$IOS_SDK" --triple arm64-apple-ios16.0 \
  --sbom-spec cyclonedx --sbom-spec spdx --sbom-output-dir /tmp/ios-profile-sboms
swift package generate-sbom --product ProfileKit --sbom-output-dir /tmp/graph-profile-sboms
```

**Walkthrough:** Inspect DeviceMetrics in all three documents, along with product, platform and generator metadata. Record actual emitted schema versions and warnings; the earlier installed distribution skipped schema validation because its schema bundle was absent.

**Boundary:** Parsed valid JSON does not prove schema validity. Local path packages can have unknown versions, and runtime-loaded dependencies may require separate inventory. --target plus --sbom-spec is an invalid combination; --sbom-warning-only weakens the failure gate. CLI options override corresponding SWIFTPM_BUILD_SBOM_* settings. [SE-0509](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0509-swift-sboms-via-swiftpm.md)

### Extension: How should build scripts locate outputs under SwiftPM 6.4?

**Answer:** Query the bin path using the same package, scratch directory, configuration, destination and settings as the build. Swift Build is the default; a hard-coded historical .build/debug path can refer to the wrong output.

**Example:** Run from Consumer:

```sh
swift build --scratch-path /tmp/profile-consumer-build --configuration release --product Consumer
swift build --scratch-path /tmp/profile-consumer-build --configuration release --product Consumer --show-bin-path
swift run --scratch-path /tmp/profile-consumer-build --configuration release Consumer
```

**Walkthrough:** The query reports the selected backend’s artifact directory; swift run executes the matching product. The validation retains the actual path rather than guessing where Swift Build put its executable or resource bundles.

**Boundary:** Separate test runners must all be checked, including their true exit status. --build-system native is a documented diagnostic fallback for a Swift Build regression, not a silent baseline change; native/xcode engines are deprecated in this distribution. More targets and an explicit-dependency flag alone prove neither speed nor a fully enforced dependency graph. [SwiftPM 6.4 notes](https://docs.swift.org/latest/documentation/packagemanagerdocs/6.4/)

## Focused exercise

**Task:** Load target-owned resources from a separate ordinary package consumer and verify public construction and packaging failures.

**Acceptance checks:**

- Build/test ProfileFeature and run the real Consumer without @testable imports.
- Confirm Hello: Ada, copied directory metadata, and the selected platform label.
- Wrong product import/package helper access fail; missing/wrong resource paths fail separately.

**Solution:** Use all complete shared package files above, including the ordinary Consumer source. Run these actual commands from their stated directories:

```sh
# From ProfileFeature
swift test
# From Consumer
swift build --product Consumer
swift build --product Consumer --show-bin-path
swift run Consumer
```

**Reasoning:** Tests verify the declaring targets; the external executable verifies the public graph and resource delivery without testing privileges. Additional controls change the copied metadata lookup path and introduce a target cycle in separate temporary copies; neither defect is hidden by a test-only resource. Keep native build sandbox workarounds specific to the validation environment.

## In practice

**Choose it when:** Split targets around ownership and dependency contracts, with explicit access and resource APIs.

**Investigate by:** Inspect graph edges, actual artifact locations and resource bundles, then test ordinary consumers and conditional destinations.

**Strong evidence:** Preserve compiler/build/test statuses and distinguish build inventory, graph inventory, parsed JSON and schema validity.

## References

### Related guides

- [[A11 — Access Control]]
- [[A12 — Availability and Conditional Compilation]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[E2 — Foundation and Objective-C Bridging Behavior]]
- [[E4 — C++ interoperability]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[E7 — Resilience, ABI, and Module Stability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F4 — Macros and compile-time code generation]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [PackageDescription](https://docs.swift.org/package-manager/PackageDescription/PackageDescription.html)
- [SwiftPM 6.4 changes](https://docs.swift.org/latest/documentation/packagemanagerdocs/6.4/)
- [SBOM guide](https://docs.swift.org/latest/documentation/packagemanagerdocs/generatingsboms)
- [SE-0509](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0509-swift-sboms-via-swiftpm.md)
- [SE-0386: package access](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0386-package-access-modifier.md)
- [SE-0491: module selectors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0491-module-selectors.md)
- [Swift 6.4 release](https://www.swift.org/blog/swift-6.4-released/)
