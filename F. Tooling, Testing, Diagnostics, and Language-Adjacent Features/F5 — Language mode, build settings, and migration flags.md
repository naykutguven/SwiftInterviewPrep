---
tags:
  - swift
  - ios
  - interview-prep
  - tooling
  - build-settings
  - swift6
  - migration
  - concurrency
---
## 0. Rubric snapshot

**Rubric expectation**

Know Swift language versioning, strict concurrency settings, default isolation settings, warning groups, and strict memory safety mode at a high level. The rubric frames this as a staff-level tooling/migration topic, not just a compiler-settings trivia topic.

**Caveats**

Staff-level engineers should distinguish behavior caused by:

```text
Swift language mode
target-level build settings
SwiftPM settings
upcoming feature flags
framework behavior
toolchain version
```

Mixing these together leads to bad migration decisions and confusing diagnosis.

**You should be able to answer**

- What kinds of issues are surfaced by strict concurrency, default isolation, and strict memory safety, respectively?
- Why is build-setting literacy important when debugging behavior differences between targets?

**You should be able to do**

- Given two targets with different language settings, explain how to prevent migration drift and inconsistent diagnostics.

---

## 1. Core mental model

Swift settings are not passive project metadata. They are part of the semantic environment in which a target is compiled.

The same Swift source file can produce different diagnostics, inferred isolation, warning severity, and migration pressure depending on the target’s language mode and build settings. This matters especially in app projects with an app target, extension targets, test targets, SwiftPM packages, generated code, and binary or source dependencies.

Think in layers:

```text
Toolchain version gives capabilities.
Language mode chooses the language semantics/checking baseline.
Build settings choose per-target defaults and diagnostics.
Feature flags let you preview or migrate future behavior.
Frameworks add their own isolation/import/generated-code behavior.
```

Swift 6 language mode is the big semantic switch for data-race safety: the Swift migration guide says Swift 6 enables full data-race safety checking by default. ([Swift.org, "Enable data-race safety checking"](https://swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)) But Swift 6.2 adds more knobs around approachability, especially default actor isolation and `nonisolated async` behavior. Swift 6.2 lets code be main-actor isolated by default, changes how `nonisolated async` can run, and introduces `@concurrent` to explicitly move work off an actor. ([Swift.org, "Swift 6.2 Released"](https://swift.org/blog/swift-6.2-released/))

The staff-level skill is not “turn everything on.” It is knowing which targets should have which semantics. A UI-heavy app target may reasonably use default `MainActor` isolation. A general-purpose networking, parsing, crypto, SDK, or model package usually should not silently become main-actor isolated.

The key idea:

```text
A Swift target is compiled under a policy. F5 is about knowing the policy, making it explicit, and keeping it consistent.
```

---

## 2. Essential mechanics

### 2.1 Swift language mode is not the same as compiler version

The compiler/toolchain version is the binary you use. The language mode is the source-compatibility mode the compiler applies to a target.

Example distinction:

```text
Toolchain: Swift 6.2 / Xcode 26
Language mode: Swift 5 or Swift 6
```

Using a newer compiler does not automatically mean every target is compiled under the newest language mode. Xcode 16 added support for the Swift 6 language mode while still supporting earlier Swift language modes. ([Apple Developer, "Xcode 16 Release Notes"](https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes))

For SwiftPM, language mode can be set per target:

```swift
.target(
    name: "Core",
    swiftSettings: [
        .swiftLanguageMode(.v6)
    ]
)
```

This is useful during migration because you can move modules one by one instead of forcing the whole codebase through Swift 6 diagnostics at once.

### 2.2 Strict concurrency surfaces data-race-safety issues

Strict concurrency checking is about Swift’s isolation and sendability model. It surfaces cases where values or code cross concurrency boundaries unsafely.

Typical diagnostics involve:

```text
non-Sendable values crossing actor boundaries
@Sendable closures capturing mutable non-Sendable state
accessing actor-isolated state from nonisolated code
global/static mutable state without isolation
main-actor state used from background/concurrent code
escaping closures/tasks that lose isolation assumptions
```

In Swift 5 migration mode, these can appear as warnings. In Swift 6 language mode, many become errors because full data-race safety checking is enabled by default. ([Swift.org, "Enable data-race safety checking"](https://swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/))

Bad:

```swift
final class Cache {
    var values: [String: Int] = [:]
}

func run(_ operation: @Sendable @escaping () async -> Void) {}

let cache = Cache()

run {
    cache.values["count"] = 1
}
```

Better:

```swift
actor Cache {
    private var values: [String: Int] = [:]

    func set(_ value: Int, forKey key: String) {
        values[key] = value
    }
}

func run(_ operation: @Sendable @escaping () async -> Void) {}

let cache = Cache()

run {
    await cache.set(1, forKey: "count")
}
```

The fix is not “silence Sendable.” The fix is choosing an ownership/isolation model.

### 2.3 Default actor isolation changes inferred isolation

Swift 6.2 adds a way for a module to infer unannotated declarations as isolated to a global actor, currently `MainActor.self`, or remain `nil`/nonisolated. The SwiftPM docs describe `defaultIsolation` and state that the default is nonisolated when unspecified; valid arguments are `MainActor.self` and `nil`. ([Swift.org, "defaultIsolation(_:_:)"](https://docs.swift.org/swiftpm/documentation/packagedescription/swiftsetting/defaultisolation%28_%3A_%3A%29/))

UI-heavy module:

```swift
.target(
    name: "AppFeature",
    swiftSettings: [
        .swiftLanguageMode(.v6),
        .defaultIsolation(MainActor.self)
    ]
)
```

General-purpose library module:

```swift
.target(
    name: "ParsingCore",
    swiftSettings: [
        .swiftLanguageMode(.v6),
        .defaultIsolation(nil)
    ]
)
```

Apple’s WWDC25 guidance says main-actor-by-default is driven by a build setting and should primarily be used for the main app module and UI-focused modules. It also says Approachable Concurrency is recommended broadly, while default actor isolation to `MainActor` is recommended for UI-focused modules. ([Apple Developer, "Embracing Swift concurrency"](https://developer.apple.com/videos/play/wwdc2025/268/))

### 2.4 `@concurrent` is an explicit escape hatch from actor execution

Under Swift 6.2’s approachable concurrency model, code can remain on the caller’s actor unless you explicitly introduce concurrency. Swift 6.2 introduces `@concurrent` to say “this work should run concurrently/off actor.” ([Swift.org, "Swift 6.2 Released"](https://swift.org/blog/swift-6.2-released/))

```swift
struct ImageDecoder {
    @concurrent
    static func decode(_ data: Data) async throws -> Image {
        // CPU-heavy work that should not block the main actor.
    }
}
```

Do not use `@concurrent` as a magical performance annotation. It changes isolation assumptions. Once code runs concurrently, anything captured or passed across that boundary must satisfy Swift’s safety model.

### 2.5 Upcoming feature flags are migration tools

Upcoming feature flags let you opt into future language behavior target by target. Swift added the generalized `-enable-upcoming-feature FeatureName` mechanism in Swift 5.8, and SwiftPM exposes this as `.enableUpcomingFeature(...)`. ([Swift.org, "Using Upcoming Feature Flags"](https://swift.org/blog/using-upcoming-feature-flags/))

```swift
.target(
    name: "Core",
    swiftSettings: [
        .enableUpcomingFeature("ExistentialAny")
    ]
)
```

They can also be checked from source:

```swift
#if compiler(>=5.8)
#if hasFeature(ExistentialAny)
let service: any ImageLoading
#else
let service: ImageLoading
#endif
#endif
```

Feature flags are most valuable when you want to measure migration cost before adopting a full language mode.

### 2.6 Warning groups and warning severity are migration controls

Swift 6.2 added more precise warning control in SwiftPM, including warning groups and treating warnings as errors. The release notes show `.treatAllWarnings(as: .error)` and `.treatWarning("DeprecatedDeclaration", as: .warning)`. ([Swift.org, "Swift 6.2 Released"](https://swift.org/blog/swift-6.2-released/))

```swift
.target(
    name: "Core",
    swiftSettings: [
        .treatAllWarnings(as: .error),
        .treatWarning("DeprecatedDeclaration", as: .warning)
    ]
)
```

This is useful for preventing regression after a target is migrated. It is dangerous if applied too early to noisy legacy targets because it can freeze migration progress.

### 2.7 Strict memory safety is not strict concurrency

Strict memory safety is about unsafe memory constructs, not actor isolation. Swift 6.2 introduced opt-in strict memory safety to flag unsafe constructs so you can replace them with safe alternatives or explicitly acknowledge them. ([Swift.org, "Swift 6.2 Released"](https://swift.org/blog/swift-6.2-released/)) The SwiftPM documentation describes it as an opt-in compiler feature that identifies constructs or APIs that break memory safety and reports issues as warnings that can be acknowledged with unsafe annotations. ([Swift.org, "strictMemorySafety(_:)"](https://docs.swift.org/swiftpm/documentation/packagedescription/swiftsetting/strictmemorysafety%28_%3A%29/))

Example target:

```swift
.target(
    name: "CWrapper",
    swiftSettings: [
        .swiftLanguageMode(.v6),
        .defaultIsolation(nil),
        .strictMemorySafety()
    ]
)
```

This is more relevant for low-level wrappers, C/C++ interop, parsing, graphics, crypto, DSP, embedded Swift, and security-sensitive code than ordinary UI app code.

---

## 3. Common traps and misconceptions

### Trap 1: “Swift 6 compiler means Swift 6 language mode”

Wrong. The compiler/toolchain can support multiple language modes.

Bad mental model:

```text
Xcode 26 installed → all targets are Swift 6
```

Better mental model:

```text
Xcode 26 installed → Swift 6.x compiler available
Each target still has its own Swift language mode and build settings
```

This matters when one target emits Swift 6 strict-concurrency errors while another only emits Swift 5 warnings.

### Trap 2: “Default MainActor isolation fixes concurrency”

Default `MainActor` isolation reduces annotation noise for UI-heavy modules. It does not make bad architecture good.

Bad:

```swift
// Entire app, networking, database, parsing, and SDK code
// implicitly MainActor because it reduced warnings.
```

Better:

```text
App/UI target: defaultIsolation(MainActor.self)
Core parsing/networking/storage libraries: defaultIsolation(nil)
Shared mutable state: actor, lock, value semantics, or explicit ownership model
```

Main-actor-by-default can be appropriate for app and UI modules, but a general-purpose library should usually stay nonisolated and let clients choose where work runs. Apple’s WWDC25 guidance explicitly recommends nonisolated APIs for general-purpose libraries when clients should decide whether to offload work. ([Apple Developer, "Embracing Swift concurrency"](https://developer.apple.com/videos/play/wwdc2025/268/))

### Trap 3: “Strict concurrency and strict memory safety are the same kind of safety”

They are different.

```text
Strict concurrency:
- actor isolation
- Sendable
- @Sendable captures
- cross-isolation transfer
- data-race safety

Strict memory safety:
- unsafe pointers
- raw memory
- unsafe imported APIs
- unsafe annotations/audit boundaries
- use-after-free / invalid memory access risk
```

A target can be strict-concurrency clean and still have unsafe pointer bugs. A target can have strict memory safety warnings and no actor-isolation issues.

### Trap 4: “Warnings as errors everywhere is staff-level discipline”

Not automatically. It is often a good end state, but a bad migration start.

Better migration posture:

```text
1. Inventory current settings.
2. Enable diagnostics in warnings mode where possible.
3. Fix by module ownership/isolation boundaries.
4. Lock migrated targets with warnings-as-errors.
5. Add CI checks to prevent drift.
```

### Trap 5: “The app target compiles, so the package is migrated”

No. Each target is separate.

You can easily have:

```text
App target: Swift 6, default MainActor, strict diagnostics
Package target: Swift 5, nonisolated default, weaker diagnostics
Test target: different default isolation from production code
Extension target: stale Swift version
```

That is migration drift.

---

## 4. Direct answers to rubric questions

### Q1. What kinds of issues are surfaced by strict concurrency, default isolation, and strict memory safety, respectively?

Strict concurrency surfaces data-race-safety issues: non-Sendable values crossing isolation boundaries, unsafe `@Sendable` captures, actor-isolation violations, global mutable state, and places where tasks/closures lose isolation assumptions. Swift 6 language mode enables full data-race safety checking by default. ([Swift.org, "Enable data-race safety checking"](https://swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/))

Default isolation surfaces inference differences. It changes what unannotated declarations mean in a target. With `defaultIsolation(MainActor.self)`, unannotated code in that module is inferred as main-actor isolated. This can reduce UI boilerplate, but it can also reveal code that should be `nonisolated`, moved into a separate actor, or explicitly marked `@concurrent`. Swift 6.2 introduced this as part of approachable concurrency. ([Swift.org, "Swift 6.2 Released"](https://swift.org/blog/swift-6.2-released/))

Strict memory safety surfaces unsafe memory boundaries: unsafe pointers, raw memory, unsafe imported APIs, and code that must be replaced with safe wrappers or explicitly acknowledged as unsafe. Swift 6.2 introduced it as opt-in because most projects do not need that level of enforcement. ([Swift.org, "Swift 6.2 Released"](https://swift.org/blog/swift-6.2-released/))

Interview version:

> Strict concurrency is about data-race safety: actor isolation, Sendable, @Sendable captures, and safe transfer across isolation boundaries. Default isolation is about what actor context unannotated code belongs to, especially MainActor-by-default for UI-heavy modules in Swift 6.2. Strict memory safety is a separate low-level mode that audits unsafe memory constructs like pointers and raw memory. A staff-level migration keeps those separate instead of treating every diagnostic as “Swift concurrency being annoying.”

### Q2. Why is build-setting literacy important when debugging behavior differences between targets?

Because Swift settings are target-specific. The same source pattern can be legal, warning-only, or an error depending on language mode, strict-concurrency level, default actor isolation, upcoming feature flags, warning-group settings, SwiftPM settings, and toolchain version.

This is especially common in iOS codebases with:

```text
App target
Widget extension
Share extension
Unit tests
UI tests
Local Swift packages
Generated code
Third-party packages
Binary frameworks
```

Example:

```text
App target:
- Swift 6
- default MainActor
- strict concurrency complete

Core package:
- Swift 5
- nonisolated default
- no strict memory safety

Test target:
- Swift 6
- default isolation unset
```

A diagnostic appearing only in tests may not mean tests are wrong. It may mean the test target compiled the same imported code under a different isolation policy or exposed a mismatch between production and test settings.

Interview version:

> Build-setting literacy matters because Swift migration is per target. When diagnostics differ between app, package, extension, and test targets, I first compare language mode, strict concurrency, default actor isolation, upcoming feature flags, and warning severity. Otherwise I might “fix” code that is actually exposing a configuration mismatch. For staff-level migration, settings should be documented, centralized where possible, and enforced in CI.

---

## 5. Code probe

No rubric code probe is provided for F5. Instead, use these three probes.

### 5.1 Minimal example: same source, different default isolation

Given:

```swift
final class ViewModel {
    var title = ""

    func updateTitle() {
        title = "Loaded"
    }
}
```

With `defaultIsolation(nil)`:

```text
ViewModel is not implicitly actor-isolated.
The type is ordinary nonisolated code unless annotated otherwise.
```

With `defaultIsolation(MainActor.self)`:

```text
ViewModel is inferred as MainActor-isolated in that module.
Access from concurrent/non-main-actor code now requires respecting MainActor isolation.
```

Why this matters:

```text
The source did not change.
The target policy changed.
The compiler’s isolation model changed.
```

### 5.2 Counterexample: default MainActor used to hide design problems

Bad:

```swift
// Networking package compiled with defaultIsolation(MainActor.self)

final class NetworkClient {
    private var inflightRequests: [URL: Task<Data, Error>] = [:]

    func data(from url: URL) async throws -> Data {
        if let task = inflightRequests[url] {
            return try await task.value
        }

        let task = Task {
            let (data, _) = try await URLSession.shared.data(from: url)
            return data
        }

        inflightRequests[url] = task
        return try await task.value
    }
}
```

What happens:

```text
The mutable dictionary is protected by MainActor only because the whole package is implicitly MainActor-isolated.
The code may compile with fewer concurrency diagnostics, but the networking layer is now coupled to the main actor.
```

Better:

```swift
actor NetworkClient {
    private var inflightRequests: [URL: Task<Data, Error>] = [:]

    func data(from url: URL) async throws -> Data {
        if let task = inflightRequests[url] {
            return try await task.value
        }

        let task = Task {
            let (data, _) = try await URLSession.shared.data(from: url)
            return data
        }

        inflightRequests[url] = task

        do {
            return try await task.value
        } catch {
            inflightRequests[url] = nil
            throw error
        }
    }
}
```

Why this fix is correct:

```text
The state belongs to the networking subsystem, not the UI.
An actor gives the cache/request table its own isolation domain.
The app can call it from MainActor without making the networking implementation MainActor-bound.
```

### 5.3 Production example: explicit SwiftPM settings

```swift
// swift-tools-version: 6.2

import PackageDescription

let uiSwiftSettings: [SwiftSetting] = [
    .swiftLanguageMode(.v6),
    .defaultIsolation(MainActor.self),
    .treatAllWarnings(as: .error)
]

let coreSwiftSettings: [SwiftSetting] = [
    .swiftLanguageMode(.v6),
    .defaultIsolation(nil),
    .treatAllWarnings(as: .error)
]

let lowLevelSwiftSettings: [SwiftSetting] = [
    .swiftLanguageMode(.v6),
    .defaultIsolation(nil),
    .strictMemorySafety(),
    .treatAllWarnings(as: .error)
]

let package = Package(
    name: "AppModules",
    platforms: [
        .iOS(.v18)
    ],
    products: [
        .library(name: "FeatureUI", targets: ["FeatureUI"]),
        .library(name: "CoreNetworking", targets: ["CoreNetworking"]),
        .library(name: "CWrapper", targets: ["CWrapper"])
    ],
    targets: [
        .target(
            name: "FeatureUI",
            dependencies: ["CoreNetworking"],
            swiftSettings: uiSwiftSettings
        ),
        .target(
            name: "CoreNetworking",
            swiftSettings: coreSwiftSettings
        ),
        .target(
            name: "CWrapper",
            swiftSettings: lowLevelSwiftSettings
        )
    ]
)
```

What happens:

```text
FeatureUI:
- Swift 6
- MainActor by default
- warnings fail the build

CoreNetworking:
- Swift 6
- nonisolated by default
- warnings fail the build

CWrapper:
- Swift 6
- nonisolated by default
- strict memory safety enabled
- warnings fail the build
```

This makes target policy explicit.

---

## 6. Exercise

### Problem

Given two targets with different language settings, explain how you would prevent migration drift and inconsistent diagnostics.

### Bad / naive version

```text
App target:
- Swift Language Version: Swift 6
- Strict Concurrency: Complete
- Default Actor Isolation: MainActor
- Approachable Concurrency: Yes

CoreKit target:
- Swift Language Version: Swift 5
- Strict Concurrency: Minimal
- Default Actor Isolation: unset
- Approachable Concurrency: unset

Tests:
- Whatever Xcode generated
```

Naive process:

```text
Fix warnings only when they appear.
Let each developer change target settings locally.
Do not document why settings differ.
Do not enforce settings in CI.
Use @MainActor, @unchecked Sendable, and @preconcurrency until the build is green.
```

### What is wrong?

```text
The app and library targets have different semantic environments.
Diagnostics become target-dependent.
Test results may not match production behavior.
Warnings fixed in one target can reappear in another.
Developers cannot tell whether a diagnostic is a real code issue or a settings mismatch.
The team may accidentally ship a partially migrated codebase while thinking Swift 6 migration is done.
```

### Improved version

#### Step 1: Inventory current settings per target

Create a migration table:

|Target|Type|Language mode|Strict concurrency|Default isolation|Strict memory safety|Warning policy|Intended policy|
|---|---|--:|---|---|---|---|---|
|App|iOS app|6|Complete|MainActor|Off|Warnings as errors|UI/main-actor module|
|Widget|Extension|6|Complete|MainActor|Off|Warnings as errors|UI/main-actor module|
|CoreKit|SwiftPM library|6|Complete via Swift 6|nil|Off|Warnings as errors|General-purpose library|
|CWrapper|SwiftPM library|6|Complete via Swift 6|nil|On|Warnings as errors|Unsafe boundary|
|Tests|Test target|6|Complete|Match target under test where possible|Off|Warnings as errors|Detect drift|

#### Step 2: Define policy by module category

```text
UI/app-facing modules:
- Swift 6
- defaultIsolation(MainActor.self)
- Approachable Concurrency enabled where available
- warnings as errors after migration

General-purpose libraries:
- Swift 6
- defaultIsolation(nil)
- explicit @MainActor only for UI-specific APIs
- explicit actors/locks/value semantics for mutable state

Low-level interop/memory modules:
- Swift 6
- defaultIsolation(nil)
- strictMemorySafety enabled
- narrow unsafe API boundary
- documented unsafe invariants

Legacy third-party or generated-code boundaries:
- isolate behind adapter modules
- use @preconcurrency only as a temporary compatibility boundary
- track removal issue/date
```

#### Step 3: Centralize settings

For SwiftPM:

```swift
let uiSettings: [SwiftSetting] = [
    .swiftLanguageMode(.v6),
    .defaultIsolation(MainActor.self),
    .treatAllWarnings(as: .error)
]

let librarySettings: [SwiftSetting] = [
    .swiftLanguageMode(.v6),
    .defaultIsolation(nil),
    .treatAllWarnings(as: .error)
]

let unsafeBoundarySettings: [SwiftSetting] = [
    .swiftLanguageMode(.v6),
    .defaultIsolation(nil),
    .strictMemorySafety(),
    .treatAllWarnings(as: .error)
]
```

For Xcode targets, prefer shared `.xcconfig` files or project-generation templates. Do not hand-edit each target forever.

Conceptually:

```text
BaseSwift6.xcconfig
- Swift Language Version = Swift 6
- Strict Concurrency Checking = Complete
- Treat warnings according to migration phase

UIActorIsolation.xcconfig
- Default Actor Isolation = MainActor
- Approachable Concurrency = Yes

LibraryIsolation.xcconfig
- Default Actor Isolation = nonisolated / unset
```

#### Step 4: Enforce in CI

Add CI checks for:

```text
xcodebuild clean build for all app/extension/test targets
swift test for all packages
warnings-as-errors for migrated modules
a script that dumps target Swift settings and fails on unexpected drift
a migration allowlist for temporary @preconcurrency / @unchecked Sendable / unsafe annotations
```

#### Step 5: Make exceptions explicit

Use a migration ledger:

```markdown
| Exception | Target | Reason | Owner | Removal condition |
|---|---|---|---|---|
| @preconcurrency import LegacySDK | App | SDK not Swift 6 annotated | Platform team | Remove after LegacySDK 4.0 |
| @unchecked Sendable ImageCache | CoreKit | Protected by internal Mutex | Core team | Replace with actor or Synchronization.Mutex wrapper |
| strictMemorySafety disabled | CWrapper | Pending pointer wrapper audit | Infra team | Enable after issue #123 |
```

### Why this is better

This prevents migration drift because every target has an intentional policy. It also makes diagnostics explainable:

```text
Diagnostic in UI target?
Likely MainActor/default-isolation or Sendable issue.

Diagnostic only in CoreKit?
Likely true nonisolated library boundary issue.

Diagnostic only in CWrapper?
Likely strict memory safety or unsafe API audit issue.

Diagnostic only in tests?
Likely test target settings mismatch or test crossing isolation incorrectly.
```

Staff-level migration is not “make the build green.” It is “make the compiler policy explicit and keep the codebase moving toward one coherent safety model.”

---

## 7. Production guidance

Use this in production when:

```text
You are migrating to Swift 6 / Swift 6.2.
You have multiple app, extension, package, and test targets.
You need to explain why diagnostics differ across modules.
You are deciding whether a module should default to MainActor.
You are wrapping C/C++ or unsafe memory APIs.
You want CI to prevent regression after migration.
```

Be careful when:

```text
A target contains both UI code and general-purpose library code.
Generated code is compiled under different settings than handwritten code.
Tests use different language mode/default isolation than production.
A package is consumed by clients with different toolchains.
A fix involves @preconcurrency, @unchecked Sendable, unsafe, or broad @MainActor.
```

Avoid when:

```text
You use MainActor-by-default to avoid designing isolation.
You enable warnings-as-errors before triaging migration categories.
You apply strict memory safety to all targets without a plan for unsafe dependencies.
You let Xcode UI settings, Package.swift settings, and CI flags diverge.
```

Debugging checklist:

```text
What target emits the diagnostic?
What Swift toolchain builds it?
What Swift language mode is the target using?
Is strict concurrency complete, targeted, minimal, or implicit via Swift 6?
Is default actor isolation MainActor or nil?
Is Approachable Concurrency enabled?
Are upcoming feature flags enabled?
Is strict memory safety enabled?
Is this source compiled in app, package, test, generated code, or extension context?
Does the fix belong in code, module boundaries, build settings, or dependency isolation?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift 6 has stricter concurrency checks, so you turn on Swift 6 and fix the warnings.

### Senior answer

> Swift 6 language mode enables full data-race safety checking, so I expect Sendable, actor-isolation, @Sendable capture, and global mutable state diagnostics. I would migrate target by target, use complete checking before flipping language mode where useful, and avoid silencing warnings without understanding the isolation model.

### Staff-level answer

> I treat Swift language mode and build settings as part of each module’s API and safety policy. UI modules may use default MainActor isolation; general-purpose libraries should usually remain nonisolated; low-level wrappers may opt into strict memory safety. I centralize settings, document exceptions, enforce consistency in CI, and prevent drift across app, extension, package, and test targets. When diagnostics differ, I first compare target policies before changing code.

Staff-level questions to ask:

```text
Which modules should be MainActor-by-default, and which should remain nonisolated?
Are tests compiled under the same semantic assumptions as the production target?
Which warnings should be errors now, and which warning groups need a temporary exception?
Which @preconcurrency, @unchecked Sendable, and unsafe annotations are temporary migration debt?
Can CI prove that no target silently regressed to Swift 5 or weaker diagnostics?
```

---

## 9. Interview-ready summary

Language mode, build settings, and migration flags define the compiler policy for each Swift target. Swift 6 language mode enables full data-race safety checking, surfacing Sendable and actor-isolation problems. Swift 6.2 adds build-setting-driven approachability features such as default MainActor isolation and explicit `@concurrent` for moving work off actor. Strict memory safety is separate: it audits unsafe memory boundaries. A senior engineer can fix the diagnostics; a staff engineer prevents migration drift by choosing target policies deliberately, centralizing settings, documenting exceptions, and enforcing consistency in CI.

---

## 10. Flashcards

Q: What is the difference between Swift compiler version and Swift language mode?  
A: Compiler version is the toolchain capability; language mode is the source-compatibility/semantic mode used to compile a target.

Q: What does Swift 6 language mode mainly change for concurrency?  
A: It enables full data-race safety checking by default, turning many strict-concurrency issues into errors.

Q: What does default actor isolation control?  
A: It controls what actor isolation unannotated declarations get in a module, for example `MainActor` or nonisolated.

Q: Should every module use `defaultIsolation(MainActor.self)`?  
A: No. It is appropriate mainly for app/UI-focused modules. General-purpose libraries should usually remain nonisolated.

Q: What does strict memory safety check?  
A: Unsafe memory constructs and APIs, such as unsafe pointers, raw memory, and unsafe imported boundaries.

Q: Is strict memory safety the same as strict concurrency?  
A: No. Strict concurrency is about data-race safety; strict memory safety is about unsafe memory access.

Q: Why can diagnostics differ between app and test targets?  
A: They may have different language modes, strict concurrency settings, default isolation, feature flags, or warning policies.

Q: What is migration drift?  
A: When targets in the same codebase silently use different Swift settings, causing inconsistent diagnostics and behavior.

Q: What is the staff-level fix for migration drift?  
A: Inventory target settings, define policy by module type, centralize settings, document temporary exceptions, and enforce with CI.

Q: When is `@preconcurrency` acceptable?  
A: As a temporary compatibility boundary for dependencies not yet annotated for Swift concurrency, not as a final design.

---

## 11. Related sections

- [[D13 — Swift 6 strict concurrency migration]]
- [[D14 — Swift 6.2 approachable concurrency and new defaults]]
- [[C11 — Strict memory safety mode and safe systems programming]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F3 — Performance investigation habits]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — F5 section.
- Swift.org. "Swift 6.2 Released." Swift.org Blog. https://swift.org/blog/swift-6.2-released/
- Swift.org. "Enable data-race safety checking." Swift 6 Concurrency Migration Guide. https://swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/
- Swift.org. "Using Upcoming Feature Flags." Swift.org Blog. https://swift.org/blog/using-upcoming-feature-flags/
- Swift.org. "defaultIsolation(_:_:)." PackageDescription. https://docs.swift.org/swiftpm/documentation/packagedescription/swiftsetting/defaultisolation%28_%3A_%3A%29/
- Swift.org. "SwiftLanguageMode." PackageDescription. https://docs.swift.org/swiftpm/documentation/packagedescription/swiftlanguagemode/
- Swift.org. "strictMemorySafety(_:)." PackageDescription. https://docs.swift.org/swiftpm/documentation/packagedescription/swiftsetting/strictmemorysafety%28_%3A%29/
- Apple Developer. "Xcode 16 Release Notes." Xcode Release Notes. https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes
- Apple Developer. "Embracing Swift concurrency." WWDC25. https://developer.apple.com/videos/play/wwdc2025/268/
