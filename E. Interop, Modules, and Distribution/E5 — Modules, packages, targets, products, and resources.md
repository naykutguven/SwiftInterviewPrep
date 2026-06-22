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
## 0. Rubric snapshot

**Rubric expectation**

Know SwiftPM structure, target boundaries, resource scoping, test support patterns, and package access control. The rubric explicitly flags E5 as a senior iOS/macOS non-negotiable topic.

**Caveats**

Poor target factoring leaks dependencies, overexposes implementation details, creates dependency cycles, and slows builds.

**You should be able to answer**

- What is the difference between a target and a product in SwiftPM?
- Why should resources be scoped deliberately to the target that owns them?

**You should be able to do**

- Split a monolithic package into targets for core logic, UI adapters, and test support without creating dependency cycles.

---

## 1. Core mental model

SwiftPM has several layers, and mixing them up causes bad package architecture:

```text
Package
 ├─ Products: what external clients are allowed to depend on
 └─ Targets: build/module/test boundaries inside the package
      ├─ Sources
      ├─ Dependencies
      └─ Resources
```

A **package** is the distribution and dependency-resolution unit. It has a `Package.swift` manifest, declares supported platforms, package dependencies, products, targets, resources, and build settings. The `PackageDescription` library is what you use in `Package.swift` to configure these details. ([Swift.org, "PackageDescription"](https://docs.swift.org/swiftpm/documentation/packagedescription/))

A **target** is the basic build boundary. SwiftPM compiles each target’s source files into a module or test suite; a target may depend on other targets in the same package and on products vended by external package dependencies. ([Swift.org, "Target"](https://docs.swift.org/swiftpm/documentation/packagedescription/target/))

A **product** is the vendable API surface of the package. It is how clients consume the package. A library product is created to let clients that declare a dependency on the package use the package’s functionality, and it is made from one or more targets. ([Swift.org, "library(name:type:targets:)"](https://docs.swift.org/swiftpm/documentation/packagedescription/product/library%28name%3Atype%3Atargets%3A%29/))

The key idea:

```text
Targets are how you build and isolate code.
Products are what you intentionally sell/export to clients.
Resources belong to the target whose code owns their runtime meaning.
```

This distinction matters because a target can exist only for internal package structure without becoming a public product. That is how you keep implementation helpers, adapters, generated code, mocks, fixtures, and test support out of the external API surface.

Swift 5.9 added the `package` access level, which lets modules in the same package share APIs without making them public to external clients. SwiftPM automatically configures this for packages. This is especially useful when splitting a large module into smaller targets without turning internal seams into public API. ([Swift.org, "Swift 5.9 Released"](https://swift.org/blog/swift-5.9-released/))

---

## 2. Essential mechanics

### 2.1 Package manifest: the graph lives in `Package.swift`

A package manifest describes the package’s products, targets, dependencies, resources, platforms, and build settings.

```swift
// swift-tools-version: 6.2

import PackageDescription

let package = Package(
    name: "SearchFeature",
    platforms: [
        .iOS(.v18)
    ],
    products: [
        .library(
            name: "SearchFeature",
            targets: ["SearchFeatureUI"]
        )
    ],
    dependencies: [
        .package(url: "https://github.com/example/analytics-sdk", from: "1.0.0")
    ],
    targets: [
        .target(
            name: "SearchCore"
        ),
        .target(
            name: "SearchFeatureUI",
            dependencies: [
                "SearchCore",
                .product(name: "AnalyticsSDK", package: "analytics-sdk")
            ],
            resources: [
                .process("Resources")
            ]
        ),
        .target(
            name: "SearchTestSupport",
            dependencies: [
                "SearchCore"
            ]
        ),
        .testTarget(
            name: "SearchCoreTests",
            dependencies: [
                "SearchCore",
                "SearchTestSupport"
            ],
            resources: [
                .process("Fixtures")
            ]
        )
    ]
)
```

Important reading:

```text
Package: distribution / dependency-resolution unit
Product: externally consumable vendable unit
Target: build/module/test boundary
Module: compiler-level import boundary, usually produced by a Swift target
Resource: runtime asset scoped to a target
```

### 2.2 Target ≈ module boundary

For ordinary Swift targets, each target becomes a module you can import.

```swift
// Sources/SearchCore/SearchQuery.swift

public struct SearchQuery: Sendable, Equatable {
    public var text: String

    public init(text: String) {
        self.text = text
    }
}
```

```swift
// Sources/SearchFeatureUI/SearchViewModel.swift

import SearchCore

@MainActor
public final class SearchViewModel {
    private var query = SearchQuery(text: "")
}
```

`internal` means “visible inside this module,” not “visible inside this package.” Once you split code into multiple targets, `internal` no longer crosses the boundary.

Use `package` when multiple targets in the same package need to share something that external clients must not see:

```swift
// Sources/SearchCore/SearchRequestBuilder.swift

package struct SearchRequestBuilder {
    package func makeRequest(for query: SearchQuery) -> URLRequest {
        // ...
    }
}
```

This is better than making the type `public` just so `SearchFeatureUI` can use it.

### 2.3 Product = vendable surface

This product exposes only `SearchFeatureUI` as the client-facing module:

```swift
products: [
    .library(
        name: "SearchFeature",
        targets: ["SearchFeatureUI"]
    )
]
```

A package can have many targets but only a few products:

```swift
targets: [
    .target(name: "SearchCore"),
    .target(name: "SearchNetworking"),
    .target(name: "SearchFeatureUI"),
    .target(name: "SearchTestSupport")
]
```

Not every target deserves to be a product.

Good product design:

```text
Expose stable capabilities.
Hide implementation seams.
Do not vend helper targets unless clients genuinely need them.
```

Bad product design:

```swift
products: [
    .library(name: "SearchCore", targets: ["SearchCore"]),
    .library(name: "SearchNetworking", targets: ["SearchNetworking"]),
    .library(name: "SearchInternalHelpers", targets: ["SearchInternalHelpers"]),
    .library(name: "SearchTestSupport", targets: ["SearchTestSupport"]),
    .library(name: "SearchFeatureUI", targets: ["SearchFeatureUI"])
]
```

This turns internal decomposition into external API.

### 2.4 Resources are target-scoped

SwiftPM resources are intentionally scoped to targets. SE-0271 says resources are part of a target just like source files, and SwiftPM generates `Bundle.module` for each module that has resources. ([GitHub, "Package Manager Resources"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0271-package-manager-resources.md))

```swift
.target(
    name: "SearchFeatureUI",
    resources: [
        .process("Resources")
    ]
)
```

Access them from code in the same target:

```swift
let url = Bundle.module.url(
    forResource: "EmptyState",
    withExtension: "json"
)
```

Do not use `Bundle.main` for package resources:

```swift
// Wrong for SwiftPM package resources
let image = UIImage(named: "empty_state")

// Better
let image = UIImage(
    named: "empty_state",
    in: .module,
    compatibleWith: nil
)
```

`Bundle.module` is generated internally for the module that owns the resources, so another target should not casually reach into that bundle. If another target needs the resource, expose behavior from the owning target:

```swift
public enum SearchFeatureAssets {
    public static func emptyStateImage() -> UIImage? {
        UIImage(named: "empty_state", in: .module, compatibleWith: nil)
    }
}
```

### 2.5 Test support target pattern

A test support target is useful when multiple test targets need shared factories, fixtures, stubs, or assertion helpers.

```swift
.target(
    name: "SearchTestSupport",
    dependencies: [
        "SearchCore"
    ]
),
.testTarget(
    name: "SearchCoreTests",
    dependencies: [
        "SearchCore",
        "SearchTestSupport"
    ]
),
.testTarget(
    name: "SearchFeatureUITests",
    dependencies: [
        "SearchFeatureUI",
        "SearchTestSupport"
    ]
)
```

Keep test support as non-product by default. Only vend it as a product when downstream clients genuinely need testing helpers.

---

## 3. Common traps and misconceptions

### Trap 1: “A target and product are basically the same”

They are not.

Bad:

```swift
products: [
    .library(name: "InternalNetworking", targets: ["InternalNetworking"]),
    .library(name: "InternalModels", targets: ["InternalModels"]),
    .library(name: "InternalDebugTools", targets: ["InternalDebugTools"])
]
```

Better:

```swift
products: [
    .library(name: "FeatureKit", targets: ["FeatureUI"])
]

targets: [
    .target(name: "FeatureCore"),
    .target(name: "FeatureNetworking", dependencies: ["FeatureCore"]),
    .target(name: "FeatureUI", dependencies: ["FeatureCore", "FeatureNetworking"])
]
```

The better version keeps internal decomposition internal.

### Trap 2: Splitting by layer in a way that creates cycles

Bad:

```text
Core imports UI
UI imports Core
Networking imports Core
Core imports Networking
```

This means the boundaries are fake.

Better:

```text
Core
 ├─ domain models
 ├─ protocols
 └─ pure business rules

Networking -> Core
UI -> Core
Composition/App -> UI + Networking
Tests -> TestSupport + targets under test
```

The core target should define abstractions, not depend on concrete adapters.

### Trap 3: Putting resources wherever they “look convenient”

Bad:

```text
Sources/SearchCore/Resources/loading.json
Sources/SearchFeatureUI/SearchView.swift
```

Then `SearchFeatureUI` starts trying to access resources owned by `SearchCore`.

Better:

```text
Sources/SearchFeatureUI/Resources/loading.json
Sources/SearchFeatureUI/SearchView.swift
```

The target that interprets the resource should usually own it.

### Trap 4: Making everything `public` after modularization

Bad:

```swift
public struct InternalSearchPipeline {
    public init() {}
    public func run() {}
}
```

Better:

```swift
package struct InternalSearchPipeline {
    package init() {}
    package func run() {}
}
```

Use `public` for client API. Use `package` for cross-target package internals. Use `internal` for same-target internals.

### Trap 5: Creating too many tiny targets

Target splitting can improve incremental builds and ownership boundaries, but every target adds graph complexity, module interfaces, build planning, dependency edges, and import overhead.

Bad target split:

```text
SearchStringFormatting
SearchDateFormatting
SearchButtonStyle
SearchLabelStyle
SearchOneProtocol
SearchOneEnum
```

Better target split:

```text
SearchCore
SearchUI
SearchNetworking
SearchTestSupport
```

Split around meaningful ownership and dependency boundaries, not around every file group.

---

## 4. Direct answers to rubric questions

### Q1. What is the difference between a target and a product in SwiftPM?

A **target** is a build/module/test boundary. A **product** is what the package vends to external clients.

A target contains source files, resources, dependencies, and build settings. SwiftPM compiles targets into modules or test suites. A product groups one or more targets into something clients can depend on, such as a library product. ([Swift.org, "Target"](https://docs.swift.org/swiftpm/documentation/packagedescription/target/))

Interview version:

> A target is how SwiftPM builds and isolates code; for Swift sources it usually becomes a module. A product is the package’s exported consumable unit. I can have many internal targets for architecture and build hygiene, but only expose a small number of products to clients. That distinction prevents implementation decomposition from becoming public API.

### Q2. Why should resources be scoped deliberately to the target that owns them?

Because resources are runtime dependencies of the code that interprets them. SwiftPM’s resource model scopes resources to targets and generates `Bundle.module` for modules with resources, so package code should not assume resources live in the main bundle or in the same physical bundle as compiled code. ([GitHub, "Package Manager Resources"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0271-package-manager-resources.md))

If resources are placed in the wrong target, you get hidden coupling:

```text
Target A owns the code.
Target B owns the resource.
Target A now depends on B’s bundle layout or public accessors.
```

That is a design smell.

Interview version:

> I scope resources to the target whose code owns their meaning. Package resources are not app-main-bundle resources, and SwiftPM intentionally gives each resource-owning module its own `Bundle.module`. If another target needs a resource, I prefer exposing behavior or typed accessors from the owning target rather than having other modules know the bundle layout.

---

## 5. Code probe

The rubric has no code probe for E5. Instead, use these examples.

### Minimal example

```swift
// swift-tools-version: 6.2

import PackageDescription

let package = Package(
    name: "ImageLoading",
    platforms: [.iOS(.v18)],
    products: [
        .library(name: "ImageLoading", targets: ["ImageLoadingUI"])
    ],
    targets: [
        .target(
            name: "ImageLoadingCore"
        ),
        .target(
            name: "ImageLoadingUI",
            dependencies: ["ImageLoadingCore"],
            resources: [.process("Resources")]
        ),
        .testTarget(
            name: "ImageLoadingCoreTests",
            dependencies: ["ImageLoadingCore"]
        )
    ]
)
```

What this means:

```text
ImageLoadingCore is an internal architectural target.
ImageLoadingUI is the public product target.
Only the ImageLoading product is vended to clients.
Resources belong to ImageLoadingUI.
Tests depend directly on the target they test.
```

### Counterexample

```swift
// swift-tools-version: 6.2

import PackageDescription

let package = Package(
    name: "ImageLoading",
    products: [
        .library(name: "ImageLoadingCore", targets: ["ImageLoadingCore"]),
        .library(name: "ImageLoadingUI", targets: ["ImageLoadingUI"]),
        .library(name: "ImageLoadingTestSupport", targets: ["ImageLoadingTestSupport"]),
        .library(name: "ImageLoadingInternalMocks", targets: ["ImageLoadingInternalMocks"])
    ],
    targets: [
        .target(
            name: "ImageLoadingCore",
            dependencies: ["ImageLoadingUI"]
        ),
        .target(
            name: "ImageLoadingUI",
            dependencies: ["ImageLoadingCore"]
        ),
        .target(
            name: "ImageLoadingTestSupport"
        ),
        .target(
            name: "ImageLoadingInternalMocks"
        )
    ]
)
```

What happens conceptually:

```text
This design is invalid or unstable as architecture.

ImageLoadingCore <-> ImageLoadingUI creates a dependency cycle.
Internal test/helper modules are exposed as products.
The product list leaks implementation details.
The package has no clear public surface.
```

### Production example

```swift
// swift-tools-version: 6.2

import PackageDescription

let package = Package(
    name: "CheckoutFeature",
    platforms: [.iOS(.v18)],
    products: [
        .library(
            name: "CheckoutFeature",
            targets: ["CheckoutUI"]
        )
    ],
    dependencies: [
        .package(url: "https://github.com/example/payments-sdk", from: "3.0.0")
    ],
    targets: [
        .target(
            name: "CheckoutCore"
        ),
        .target(
            name: "CheckoutPayments",
            dependencies: [
                "CheckoutCore",
                .product(name: "PaymentsSDK", package: "payments-sdk")
            ]
        ),
        .target(
            name: "CheckoutUI",
            dependencies: [
                "CheckoutCore",
                "CheckoutPayments"
            ],
            resources: [
                .process("Resources")
            ]
        ),
        .target(
            name: "CheckoutTestSupport",
            dependencies: [
                "CheckoutCore"
            ]
        ),
        .testTarget(
            name: "CheckoutCoreTests",
            dependencies: [
                "CheckoutCore",
                "CheckoutTestSupport"
            ],
            resources: [
                .process("Fixtures")
            ]
        ),
        .testTarget(
            name: "CheckoutUITests",
            dependencies: [
                "CheckoutUI",
                "CheckoutTestSupport"
            ]
        )
    ]
)
```

Dependency direction:

```text
CheckoutCore
   ↑
CheckoutPayments
   ↑
CheckoutUI

CheckoutTestSupport -> CheckoutCore
Tests -> targets under test + CheckoutTestSupport
```

No production target imports tests. Core does not import UI. UI composes core and adapter targets.

---

## 6. Exercise

### Problem

Split a monolithic package into targets for core logic, UI adapters, and test support without creating dependency cycles.

### Bad / naive version

```swift
// swift-tools-version: 6.2

import PackageDescription

let package = Package(
    name: "ProfileFeature",
    platforms: [.iOS(.v18)],
    products: [
        .library(name: "ProfileFeature", targets: ["ProfileFeature"])
    ],
    dependencies: [
        .package(url: "https://github.com/example/analytics-sdk", from: "1.0.0"),
        .package(url: "https://github.com/example/networking-sdk", from: "2.0.0")
    ],
    targets: [
        .target(
            name: "ProfileFeature",
            dependencies: [
                .product(name: "AnalyticsSDK", package: "analytics-sdk"),
                .product(name: "NetworkingSDK", package: "networking-sdk")
            ],
            resources: [
                .process("Resources"),
                .process("TestFixtures")
            ]
        ),
        .testTarget(
            name: "ProfileFeatureTests",
            dependencies: ["ProfileFeature"]
        )
    ]
)
```

### What is wrong?

```text
Core domain, UI, networking, analytics, resources, and test fixtures are all coupled.
Test fixtures are accidentally part of the production target.
Any file in the target can import analytics/networking.
Builds are less incremental because every change hits the monolith.
The package has no internal seams for focused testing.
The only isolation boundary is the whole feature.
```

### Improved version

```swift
// swift-tools-version: 6.2

import PackageDescription

let package = Package(
    name: "ProfileFeature",
    platforms: [.iOS(.v18)],
    products: [
        .library(
            name: "ProfileFeature",
            targets: ["ProfileUI"]
        )
    ],
    dependencies: [
        .package(url: "https://github.com/example/analytics-sdk", from: "1.0.0"),
        .package(url: "https://github.com/example/networking-sdk", from: "2.0.0")
    ],
    targets: [
        .target(
            name: "ProfileCore"
        ),
        .target(
            name: "ProfileNetworking",
            dependencies: [
                "ProfileCore",
                .product(name: "NetworkingSDK", package: "networking-sdk")
            ]
        ),
        .target(
            name: "ProfileAnalytics",
            dependencies: [
                "ProfileCore",
                .product(name: "AnalyticsSDK", package: "analytics-sdk")
            ]
        ),
        .target(
            name: "ProfileUI",
            dependencies: [
                "ProfileCore",
                "ProfileNetworking",
                "ProfileAnalytics"
            ],
            resources: [
                .process("Resources")
            ]
        ),
        .target(
            name: "ProfileTestSupport",
            dependencies: [
                "ProfileCore"
            ]
        ),
        .testTarget(
            name: "ProfileCoreTests",
            dependencies: [
                "ProfileCore",
                "ProfileTestSupport"
            ],
            resources: [
                .process("Fixtures")
            ]
        ),
        .testTarget(
            name: "ProfileUITests",
            dependencies: [
                "ProfileUI",
                "ProfileTestSupport"
            ]
        )
    ]
)
```

Example core API:

```swift
// Sources/ProfileCore/Profile.swift

public struct Profile: Sendable, Equatable {
    public let id: String
    public let displayName: String

    public init(id: String, displayName: String) {
        self.id = id
        self.displayName = displayName
    }
}

public protocol ProfileLoading: Sendable {
    func loadProfile(id: String) async throws -> Profile
}

package enum ProfileValidation {
    package static func isValidDisplayName(_ value: String) -> Bool {
        !value.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty
    }
}
```

Example adapter:

```swift
// Sources/ProfileNetworking/NetworkProfileLoader.swift

import ProfileCore
import NetworkingSDK

public struct NetworkProfileLoader: ProfileLoading {
    public init() {}

    public func loadProfile(id: String) async throws -> Profile {
        // Call NetworkingSDK, map DTO -> Profile.
        Profile(id: id, displayName: "Aykut")
    }
}
```

Example UI resource access:

```swift
// Sources/ProfileUI/ProfileAssets.swift

import UIKit

public enum ProfileAssets {
    public static func placeholderAvatar() -> UIImage? {
        UIImage(
            named: "placeholder_avatar",
            in: .module,
            compatibleWith: nil
        )
    }
}
```

Example test support:

```swift
// Sources/ProfileTestSupport/ProfileFactory.swift

import ProfileCore

public enum ProfileFactory {
    public static func make(
        id: String = "user-1",
        displayName: String = "Test User"
    ) -> Profile {
        Profile(id: id, displayName: displayName)
    }
}
```

### Why this is better

```text
ProfileCore is pure and easy to test.
ProfileUI owns UI resources.
Networking and analytics dependencies do not leak into core logic.
Test fixtures live in test targets, not production targets.
ProfileTestSupport is reusable by tests without being a client-facing product.
The public product exposes the feature, not every internal seam.
```

The dependency graph is acyclic:

```text
ProfileCore
   ↑       ↑
   |       |
ProfileNetworking   ProfileAnalytics
        ↑             ↑
         \           /
          ProfileUI

ProfileTestSupport -> ProfileCore
Tests -> target under test + ProfileTestSupport
```

---

## 7. Production guidance

Use this in production when:

```text
You have a feature large enough to deserve explicit boundaries.
You want faster incremental builds by reducing unnecessary recompilation.
You want to keep third-party dependencies out of core logic.
You want test helpers without polluting production code.
You want to share internals across package targets without public exposure.
You are building an SDK and need a deliberate public product surface.
```

Be careful when:

```text
A target split forces public APIs that should not exist.
A target has only one tiny file and no meaningful dependency boundary.
A target dependency starts pointing “up” from core to UI.
A resource is needed by code in a different target.
A test support target starts depending on production UI or networking.
A product exposes an implementation target because it was convenient.
```

Avoid when:

```text
The package is small and target splitting adds more ceremony than clarity.
The split mirrors folders but not real ownership or dependency boundaries.
The graph creates cycles.
The split exists only to hide bad coupling instead of fixing it.
```

Debugging checklist:

```text
Can I draw the target dependency graph?
Which targets are products, and why?
Can clients import only what they should import?
Did I accidentally expose an internal target as a product?
Does core logic depend on UI, networking, analytics, or persistence?
Are resources in the same target as the code that interprets them?
Are test fixtures excluded from production targets?
Can package access replace unnecessary public declarations?
Is a build slowdown caused by one target importing too many dependencies?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> A package has targets and products. Targets contain code, products expose libraries.

### Senior answer

> A target is a module/build boundary, and a product is the vendable surface. I use targets to isolate core logic, adapters, UI, resources, and test support. I avoid exposing every target as a product because that leaks implementation details and creates compatibility obligations.

### Staff-level answer

> I treat SwiftPM structure as architecture, not just build configuration. Targets encode dependency direction, build isolation, resource ownership, and test seams. Products encode the external contract. I keep core targets dependency-light, put adapters at the edges, scope resources to the owning target, use `package` access for cross-target internals, and expose only stable products. When a package grows, I inspect the graph for cycles, dependency leakage, public API creep, build-time hotspots, and whether test support is helping or becoming another hidden production dependency.

Staff-level questions to ask:

```text
Which targets are architectural boundaries, and which are just folders?
Which targets must be products, and which should stay internal?
What public APIs exist only because of poor target factoring?
Can core logic build without UIKit, SwiftUI, networking, analytics, persistence, or app-specific resources?
Do resources belong to the target that interprets them?
Should shared internals be package access instead of public?
Are test helpers reusable without becoming client-facing API?
Where would adding a new platform adapter fit in the graph?
Which dependency edges dominate build time?
What changes would be SemVer-breaking for package clients?
```

---

## 9. Interview-ready summary

In SwiftPM, a target is the build and module boundary, while a product is the package’s exported consumable surface. I use targets to separate core logic, UI, platform adapters, resources, and test support, but I only expose products that clients should intentionally depend on. Resources should live in the target whose code owns their meaning and should be accessed through `Bundle.module`, not through assumptions about the app bundle. In modern Swift, `package` access is important because it lets multiple targets in the same package share implementation details without making them public API. Staff-level package design is about keeping the dependency graph acyclic, minimizing leaked dependencies, preserving build performance, and making the public product surface stable.

---

## 10. Flashcards

Q: What is a SwiftPM target?

A: A build/module/test boundary. SwiftPM compiles a target’s sources into a module or test suite, and the target declares its own dependencies, resources, and build settings.

Q: What is a SwiftPM product?

A: A vendable unit of a package, such as a library product, that external clients can depend on. Products are composed from one or more targets.

Q: Should every target become a product?

A: No. Targets are often internal architecture. Only stable client-facing capabilities should become products.

Q: What access level should be used for APIs shared across targets in the same package but hidden from clients?

A: `package`, introduced in Swift 5.9.

Q: Why is making an internal helper target a product risky?

A: It exposes implementation details, creates compatibility obligations, and lets clients depend on seams you may want to change or delete.

Q: Where should package resources live?

A: In the target whose code owns and interprets them.

Q: How should SwiftPM package resources usually be accessed?

A: Through `Bundle.module` from code in the resource-owning target.

Q: Why is `Bundle.main` usually wrong for SwiftPM resources?

A: Package resources are not guaranteed to live in the app’s main bundle; SwiftPM provides target/module-scoped resource bundles.

Q: What is a good direction for feature target dependencies?

A: Core is dependency-light; adapters depend on core; UI composes core and adapters; tests depend on targets under test and test support.

Q: What is a common sign of bad target factoring?

A: Dependency cycles, public APIs created only to cross target boundaries, or core logic importing UI/platform/third-party SDKs.

---

## 11. Related sections

- [[A11 — Access control]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[E6 — Import visibility and dependency leakage]]
- [[E7 — Resilience, ABI, and module stability]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — E5 expectation, caveats, questions, and exercise.
- Swift.org. "Swift Package Manager." Swift Package Manager Documentation. https://docs.swift.org/swiftpm/documentation/packagemanagerdocs/
- Swift.org. "PackageDescription." PackageDescription. https://docs.swift.org/swiftpm/documentation/packagedescription/
- Swift.org. "Target." PackageDescription. https://docs.swift.org/swiftpm/documentation/packagedescription/target/
- Swift.org. "library(name:type:targets:)." PackageDescription. https://docs.swift.org/swiftpm/documentation/packagedescription/product/library%28name%3Atype%3Atargets%3A%29/
- GitHub. "Package Manager Resources." Swift Evolution SE-0271. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0271-package-manager-resources.md
- Swift.org. "Swift 5.9 Released." Swift.org Blog. https://swift.org/blog/swift-5.9-released/
