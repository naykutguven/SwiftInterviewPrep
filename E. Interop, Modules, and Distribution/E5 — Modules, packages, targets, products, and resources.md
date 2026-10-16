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

**Goal:** Load a resource through its owning target from an ordinary external package consumer.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A package defines dependencies and targets; a product exposes targets for consumption; an ordinary Swift target builds a module. The product name need not be an importable module name. Resources belong to targets and their generated resource bundles, while access control independently governs what consumers may name and construct. [PackageDescription](https://docs.swift.org/package-manager/PackageDescription/PackageDescription.html)

## Rules and pitfalls

### Put lookup behind the resource owner

Use Bundle.module in code belonging to the resource target. Process and copy rules can create different lookup paths; test the actual processed artifact. Bundle.main refers to the process/application bundle, not a package-wide asset namespace.

**Common pitfall:** A test-only resource or main app bundle can hide a missing package resource during local tests. Load through the public API from an ordinary consumer executable too.

### Treat the package graph as a build contract

Keep dependency edges acyclic and deliberate, with package access only for coordinating modules sharing package identity. More targets add boundaries and overhead rather than guaranteeing faster builds. Swift Build is SwiftPM 6.4's default; discover artifact paths with the same build arguments.

**Common pitfall:** An SBOM from a resolved package graph can include conditional edges absent from one selected build. Generation success also does not establish schema validation or complete runtime dependency coverage.

## Interview questions

### Why can a client depend on ProfileKit, import ProfileCore, and still fail to use its package helper?

ProfileKit is the product dependency, while ProfileCore is a module supplied by its target graph. The external client has a different package identity and therefore cannot use package-only helpers. A product assembles targets; it is not an access-control firewall or automatically a same-named module.

### How do process/copy rules and the owning target change resource lookup in tests and consumers?

Process applies platform/resource processing rules; copy preserves the declared directory structure. Bundle.module belongs to the target declaring those resources, including a separate test target's own fixture bundle. Clients should call that owner's public API, not reach into an assumed common bundle or use Bundle.main.

### Why do the macOS build, iOS build, and standalone package-graph SBOMs differ?

Build-derived inventories can reflect platform-conditional edges, so an iOS-only dependency appears in the iOS build and not macOS. A standalone graph inventory can retain the resolved edge without that build filtering. Record product, destination, and generator route and inspect metadata/schema validation separately.

## Focused exercise

**Task:** Place a resource in its owning package target and load it from a separate consumer. Check public construction and resource lookup.

**Acceptance checks:**

- A separate package imports ProfileCore via product ProfileKit.
- Public construction and target-owned resource lookup succeed.
- Removing the resource or changing its expected path is tested as a separate packaging failure.

**Solution:**

Create sibling ResourceKit and Consumer directories. ResourceKit/Sources/ProfileCore/Resources/greeting.txt contains Hello followed by a newline.

```swift
// ResourceKit/Package.swift
// swift-tools-version: 6.4
import PackageDescription
let package = Package(name: "ResourceKit",
    products: [.library(name: "ProfileKit", targets: ["ProfileCore"])],
    targets: [.target(name: "ProfileCore", resources: [.process("Resources")],
        swiftSettings: [.swiftLanguageMode(.v6), .defaultIsolation(nil),
            .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
            .enableUpcomingFeature("InferIsolatedConformances")])])
```

```swift
// ResourceKit/Sources/ProfileCore/Greeting.swift
import Foundation
public struct Greeting {
    public init() {}
    public func load() throws -> String {
        guard let url = Bundle.module.url(forResource: "greeting", withExtension: "txt") else {
            throw CocoaError(.fileNoSuchFile)
        }
        return try String(contentsOf: url, encoding: .utf8)
            .trimmingCharacters(in: .whitespacesAndNewlines)
    }
}
```

```swift
// Consumer/Package.swift
// swift-tools-version: 6.4
import PackageDescription
let package = Package(name: "Consumer",
    dependencies: [.package(path: "../ResourceKit")],
    targets: [.executableTarget(name: "Consumer",
        dependencies: [.product(name: "ProfileKit", package: "ResourceKit")],
        swiftSettings: [.swiftLanguageMode(.v6), .defaultIsolation(nil),
            .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
            .enableUpcomingFeature("InferIsolatedConformances")])])
```

```swift
// Consumer/Sources/Consumer/main.swift
import ProfileCore
let text = try Greeting().load()
precondition(text == "Hello")
```

Build and run Consumer from its directory. This tiny trusted resource uses synchronous file reading; it is not a general asynchronous I/O design.

**Reasoning:** The product/module distinction is visible at the consumer, while resource lookup stays with its owner. A successful source build must carry the resource bundle alongside executable/library artifacts when packaged for deployment.

## In practice

**Choose it when:** Split targets around ownership and dependency contracts, with consumer evidence.

**Investigate by:** Inspect effective settings, resource bundles, graph edges, and discovered output locations.

**Strong evidence:** Validate tests and external consumers, then distinguish build SBOM evidence from graph inventory and schema validation.

### Conditional graph and SBOM detail — optional

#### Swift Build and SBOMs in 6.4

Swift Build is now SwiftPM's default build system. Existing scripts must discover output locations rather than assume .build/debug. Use the same package, scratch path, configuration, SDK, architecture, and other build arguments when querying the bin path. There is a separate test runner per test target; aggregate every runner's result and preserve the actual command exit status. [SwiftPM 6.4 changes](https://docs.swift.org/latest/documentation/packagemanagerdocs/6.4/)

Run from ProfileFeature with the selected Swift 6.4 toolchain; directories below are example output locations:

```sh
swift test
swift build --product ProfileKit
swift build --product ProfileKit --show-bin-path
swift build --product ProfileKit --sbom-spec cyclonedx --sbom-spec spdx --sbom-output-dir /tmp/profile-build-sboms
swift package generate-sbom --product ProfileKit --sbom-output-dir /tmp/profile-package-sboms
```

| SBOM route | Meaning |
| --- | --- |
| swift build with Swift Build and SBOM flags | Uses the selected build, including build-time conditions |
| swift package generate-sbom | Describes the resolved package graph without compiling; conditional edges may overstate that build's contents |
| native build-system SBOM route | Also lacks the Swift Build conditional filtering; do not assume equivalent precision |

The standalone command without a specification requests all supported formats. The checked distribution emits CycloneDX 1.7 and SPDX 3.0.1 JSON. With this graph, macOS build output excludes DeviceMetrics, iOS build output includes it, and standalone graph output includes it. These local-path fixtures have unknown versions; use real resolved source identities and inspect metadata for a release artifact.

Use --product to select a product and --sbom-filter to filter package/product entries. Combining --target and --sbom-spec is rejected. Generation errors normally fail the command; --sbom-warning-only weakens that gate. CLI options take precedence over the corresponding SWIFTPM_BUILD_SBOM_* variables. [SBOM guide](https://docs.swift.org/latest/documentation/packagemanagerdocs/generatingsboms), [SE-0509](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0509-swift-sboms-via-swiftpm.md)

The installed toolchain reports a missing SwiftPM_SBOMModel schema bundle and skips schema validation while successfully writing files. We parsed and inspected the contents; we did not establish schema validity. An SBOM is dependency inventory, not a vulnerability scan, license verdict, reproducible-build proof, or guarantee that every runtime-loaded component is represented. Preserve warnings and review the document alongside the actual release configuration.

For a Swift Build regression, the documented diagnostic fallback is --build-system native. That engine and the xcode engine are deprecated choices in this CLI. Record the failing command and compare engines if needed; do not silently bake a fallback into the new baseline. The release notes list limitations involving generated Linux SDKs, explicit target dependency import checks, and swift-java build-tool-plugin combinations. A flag named --explicit-target-dependency-import-check is therefore not sufficient evidence that every dependency edge is correctly declared in this release.

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
