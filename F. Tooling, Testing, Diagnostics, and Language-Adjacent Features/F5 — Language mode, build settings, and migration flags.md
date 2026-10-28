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

# F5 — Language Mode, Build Settings, and Migration Flags

**Learning goals:**

- Reproduce compiler, language, isolation, SDK, deployment and warning settings independently.
- Choose concurrency and memory-safety checks from the violated contract rather than diagnostic volume.
- Verify warning policy and imported API isolation with actual negative clients and a repaired capture.

**Prerequisites:** [[D6 — `Sendable`, `@Sendable`, and `sending`]]/[[D13 — Swift 6 Strict Concurrency Migration]] for Sendable and migration, [[D14 — Approachable Concurrency and Default Isolation]] for declaration/async defaults, [[C11 — Strict Memory Safety Mode and Safe Systems Programming]] for strict memory safety, and [[E5 — Modules, packages, targets, products, and resources]]/[[E6 — Import Visibility and Dependency Leakage]] for package/import settings.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Two targets compile the same file and disagree. Before changing the source, compare the actual compiler invocations. The **compiler release** supplies language features; **language mode** selects compatibility/checking rules; the **SDK** supplies declarations; the **deployment target** sets the oldest supported runtime. These can differ within one Xcode workspace.

Swift 6 mode enables complete concurrency checking. Approachable Concurrency changes selected inference and async defaults, while Default Actor Isolation independently affects eligible unannotated declarations. Imported APIs retain the contract from their defining module. A quieter diagnostic is not proof of a safer program. [Migration settings](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Record the complete command, not only an Xcode version

Save the next source as Policy.swift. This is the collection’s baseline, with an explicit target, SDK and module cache. The deployment target here is OS 27 for host validation, not a claim that every operation in the file needs OS 27.

```sh
xcrun --sdk macosx swiftc -version
xcrun --sdk macosx swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -target arm64-apple-macos27.0 -module-cache-path /tmp/f5-module-cache \
  -warnings-as-errors Policy.swift -o /tmp/policy
/tmp/policy
```

```swift
@MainActor final class DisplayState { var title = "Ready" }
func label(_ value: Int) -> String { "Item \(value)" }
let state = DisplayState()
state.title = label(7)
precondition(state.title == "Item 7")
```

**Walkthrough:** The module default is nonisolated; the UI-owned class deliberately opts into MainActor. Top-level code can use it on that actor. Changing the target’s default would infer isolation for other eligible declarations too; it is not a lock added to arbitrary references.

The two opt-ins have separate jobs. NonisolatedNonsendingByDefault keeps ordinary nonisolated async work in the caller’s isolation unless the API explicitly chooses concurrent execution. InferIsolatedConformances permits appropriate protocol conformances to inherit an isolated type’s boundary. Neither option changes the separate nonisolated declaration default or makes an unsafe reference Sendable. [[D14 — Approachable Concurrency and Default Isolation|D14]] teaches these inference rules in detail.

### One capture exposes language-mode staging

Shared Capture.swift source. accept intentionally does not run the closure: this first probe tests the Sendable capture contract, not scheduling. Compile in Swift 5 complete-checking mode without warnings-as-errors for the stated warning.

```swift
final class Cache { var count = 0 }
func accept(_ operation: @Sendable @escaping () -> Void) {}
func probe() {
    let cache = Cache()
    accept { cache.count += 1 }
}
```

**Walkthrough:** The same 6.4 compiler warns in Swift 5 compatibility mode and rejects the same capture in Swift 6 mode. The mutable class is captured by a closure that must be safe to send; the checker is not relying on accept’s current empty body. The exercise later changes that actual contract.

### A declaration’s module fixes its imported isolation

Shared BuildPolicy source. This direct block builds the neutral module with the baseline; configuration controls build this exact source separately with MainActor declaration defaults. It is also the package target source below.

```swift
public final class Catalog {
    public var count = 0
    public init() {}
    public func increment() { count += 1 }
}
```

**Walkthrough:** A neutral client can construct/increment this neutral module synchronously. When the module is instead built with MainActor defaults, those public operations carry that isolation into clients. Changing the importing target’s defaults does not reinterpret the library.

Save that source as `/tmp/BuildPolicy.swift` and build its neutral module with the shared flags. The object and module must come from the same compilation. To compare a MainActor-default library, emit to a separate directory and replace only `-default-isolation nonisolated` with `-default-isolation MainActor`.

```sh
swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -warnings-as-errors -parse-as-library -module-name BuildPolicy \
  -emit-module -emit-module-path /tmp/BuildPolicy.swiftmodule \
  -c /tmp/BuildPolicy.swift -o /tmp/BuildPolicy.o
```

## Rules and pitfalls

### A release number is not a language-mode argument

**Common pitfall:** Passing the compiler’s minor version to -swift-version, or assuming a tools-version line selects every target’s language mode.

**Mistake:** This actual command is intentionally rejected:

```sh
swiftc -swift-version 6.4 -c Policy.swift -o /tmp/Policy.o
```

**Consequence:** Argument parsing fails before checking the source.

**Why:** The compiler release is 6.4, but the mode spelling is 6. A package’s swift-tools-version controls tools/manifest compatibility, separately from its target language settings.

**Correction:** Use the valid mode and record the remaining baseline arguments too:

```sh
swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances -c Policy.swift -o /tmp/Policy.o
```

**What changes:** The compiler can apply the intended language/checking rules. The selected SDK, target, optimization, and warning flags still need to match when comparing targets.

### Warning suppression cannot repair a native Swift 6 capture error

**Common pitfall:** Using a diagnostic annotation to hide the violated isolation/transfer contract.

**Mistake:** Compile-only; intentionally rejected.

```swift
final class Cache { var count = 0 }
func accept(_ operation: @Sendable @escaping () -> Void) {}
@diagnose(SendableClosureCaptures, as: ignored, reason: "Teaching negative: this is a native error")
func probe() {
    let cache = Cache()
    accept { cache.count += 1 }
}
```

**Consequence:** The Swift 6 capture still rejects.

**Why:** The annotation controls warning behavior in its lexical scope. A native error is not demoted by that policy.

**Correction:** Isolate mutable state and make the operation asynchronous so it can respect that boundary. Shared repaired capture setup; unlike the stub, execute actually awaits the operation.

```swift
actor Cache {
    private var count = 0
    func increment() { count += 1 }
    func value() -> Int { count }
}
func execute(_ operation: @Sendable () async -> Void) async { await operation() }
func repairedCapture() async -> Int {
    let cache = Cache()
    await execute { await cache.increment() }
    return await cache.value()
}
let count = await repairedCapture()
precondition(count == 1)
```

**What changes:** The operation’s capture is safe, and the actor performs the mutation. This changes the boundary instead of merely changing severity. The awaited result verifies the executed repair.

### Concurrency checking does not validate a raw address lifetime

**Common pitfall:** Accepting a pointer escape because the file passes Swift 6 concurrency checks. Compile-only; the invalid read is never executed.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
func invalidRead() -> UInt8 {
    let values: [UInt8] = [1, 2]
    let address = values.withUnsafeBufferPointer { $0.baseAddress! }
    return address[0]
}
```

**Consequence:** The pointer has no supported post-closure access lifetime despite successful ordinary compilation.

**Why:** Concurrency checking and strict memory safety address different obligations. Even an unsafe acknowledgment cannot extend a borrowed pointer’s validity.

**Correction:** Read the owned Array through its checked API; this operation does not need an unsafe address at all. Compile the correction with strict memory safety enabled.

```swift
func firstByte(_ values: [UInt8]) -> UInt8? {
    values.first
}
precondition(firstByte([1, 2]) == 1 && firstByte([]) == nil)
```

**What changes:** The Array supplies the value through checked access and handles empty input without an address. [[C11 — Strict Memory Safety Mode and Safe Systems Programming|C11]] shows how to audit a native boundary when unsafe access really is required.

## Worked example

### Give one warning a local policy without weakening native errors

**Setup:** Standalone deprecation fixture under warnings-as-errors. The local exception suppresses this known compatibility call only.

**Predict:** Does a scoped exception change runtime behavior or remove errors elsewhere?

```swift
@available(*, deprecated, message: "Use currentValue")
func legacyValue() -> Int { 7 }
func currentValue() -> Int { 7 }
@diagnose(
    DeprecatedDeclaration, as: ignored, reason: "Compatibility fixture; remove with legacy support")
func compatibilityValue() -> Int { legacyValue() }
precondition(compatibilityValue() == currentValue())
```

**Outcome:** The scoped call compiles and returns the same result. A separate unsuppressed/promoted call rejects as a warning-policy control.

**Why:** The group name comes from the diagnostic. The exception has a reason and removal condition, not a general promise of safety. Diagnostic promotion must be tested against a known failing call. [Source warning policy](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0522-source-warning-control.md)

## Interview questions

### What do strict concurrency, default isolation, and strict memory safety surface?

**Answer:** Concurrency checks cross-isolation access and transfer, declaration defaults infer an owner for eligible unannotated declarations, and strict memory safety highlights unsafe boundaries. None replaces the others.

**Example:** Standalone runnable example.

```swift
@MainActor final class ScreenState {
    private(set) var count = 0
    func increment() { count += 1 }
}
actor Counter {
    private var count = 0
    func increment() { count += 1 }
    func value() -> Int { count }
}
let screen = ScreenState()
screen.increment()
let counter = Counter()
await counter.increment()
let snapshot = await counter.value()
precondition(screen.count == 1 && snapshot == 1)
```

**Walkthrough:** Both owners are explicit under a neutral module default. Actor calls obey their isolation; the earlier pointer example shows why passing those checks would not prove a raw lifetime safe.

**Boundary:** A default can serialize eligible work or alter an API; it is not a general synchronization retrofit. Audit unsafe memory independently.

The memory check catches a different obligation. This independent source is **intentionally rejected** with `-strict-memory-safety -warnings-as-errors`; it is never executed:

```swift
func rawByte(_ address: UnsafePointer<UInt8>) -> UInt8 {
    address.pointee
}
```

The pointer API is an unsafe boundary even without concurrent work. Strict memory safety highlights that obligation; it does not derive an address lifetime or prove a caller's storage valid. The earlier checked Array correction avoids the address entirely. Concurrency isolation and declaration defaults answer separate questions.

### Why can two targets diagnose the same source differently?

**Answer:** Because each target supplies its own language/feature/isolation/SDK/warning settings. Reproduce the actual invocation before changing source, and separately inspect the imported module’s contract.

**Example:** Ordinary BuildPolicy client. Use -I with the module directory and link its object.

```swift
import BuildPolicy

func clientValue() -> Int {
    let catalog = Catalog()
    catalog.increment()
    return catalog.count
}
precondition(clientValue() == 1)
```

Save this client as `/tmp/PolicyClient.swift`, then use the emitted module and object:

```sh
swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances -warnings-as-errors \
  -I /tmp /tmp/BuildPolicy.o /tmp/PolicyClient.swift -o /tmp/policy-client
/tmp/policy-client
```

**Walkthrough:** This ordinary client succeeds with the neutral BuildPolicy module. A separate same-source module built with MainActor defaults makes this synchronous nonisolated caller reject; an actor-owned caller is the repair.

**Boundary:** A test target cannot re-infer production isolation by importing it. Compiler version alone also does not establish SDK declarations or runtime support.

### Can @diagnose suppress a Swift 6 Sendable error or re-infer imported isolation?

**Answer:** No. Warning policy changes reporting, while import preserves the defining module’s API. Repair the transfer/isolation relationship for a native error.

**Example:** Independent intentionally rejected Swift 6 source.

```swift
@diagnose(
    RegionIsolation, as: ignored, reason: "Teaching negative: retained state is not transferred")
func receive(_ owner: Owner) async {
    let value = await owner.retained()
    value.count += 1
}
final class State { var count = 0 }
actor Owner {
    private let state = State()
    func retained() -> State { state }
}
```

**Walkthrough:** The retained non-Sendable actor result still rejects despite warning suppression. Returning a Sendable value snapshot or performing the requested operation inside Owner restores the contract.

**Boundary:** Compiler-specific warning exceptions remain observations, not general policy. [[B6 — Any, AnyObject, Metatypes, and Runtime Type Information|B6]]/[[B11 — Library Evolution and Resilience-Aware API Design|B11]] retain their scoped promotion limits; this case tests an actual error.

### Extension: How do equivalent SwiftPM settings make the baseline reviewable?

**Answer:** Put complete target policies in the manifest, then inspect the actual build commands. The tools version alone neither selects every language mode nor replaces SDK/destination checks.

**Example:** Complete temporary Package.swift, built as a package rather than an independent script.

```swift
// swift-tools-version: 6.4
import PackageDescription

let baseline: [SwiftSetting] = [
    .swiftLanguageMode(.v6), .defaultIsolation(nil),
    .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
    .enableUpcomingFeature("InferIsolatedConformances"),
]
let package = Package(
    name: "PolicyNotes", products: [.library(name: "BuildPolicy", targets: ["BuildPolicy"])],
    targets: [.target(name: "BuildPolicy", swiftSettings: baseline)])
```

**Walkthrough:** The package target uses the earlier BuildPolicy source saved as Sources/BuildPolicy/Policy.swift. The real temporary package build checks this complete manifest and records its effective compiler settings.

**Boundary:** Xcode app/test/generated-code targets can have different settings. Keep an exception ledger and verify each affected target; the package does not set app deployment or signing.

## Focused exercise

**Task:** Compile the same capture in Swift 5 complete-checking and Swift 6 modes, then verify the actor-based repair.

**Acceptance checks:**

- The Swift 5 object build warns and succeeds without warnings-as-errors.
- The Swift 6 object build rejects the unchanged capture.
- The executed actor repair increments once; record SDK/target and both opt-ins, not just the diagnostic text.

**Solution:** This is the same Capture.swift source introduced earlier, now intentionally rejected under the Swift 6 baseline. The repaired capture setup above is its runnable solution.

```swift
final class Cache { var count = 0 }
func accept(_ operation: @Sendable @escaping () -> Void) {}
func probe() {
    let cache = Cache()
    accept { cache.count += 1 }
}
```

**Reasoning:** Only language mode changes between the two original-source comparisons. The actor repair changes the actual ownership/effect contract. Both object compilation and an awaited execution result matter.

```sh
# Keep the same compiler/SDK/target/cache and both baseline opt-ins.
swiftc -swift-version 5 -strict-concurrency=complete -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances -c Capture.swift -o /tmp/Capture5.o
swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances -c Capture.swift -o /tmp/Capture6.o
```

## In practice

**Choose it when:** Define explicit per-target semantics and warning policy when modules migrate at different speeds.

**Investigate by:** Compare full compiler commands, SDK/target, imported interfaces and known-failure gates before treating a diagnostic difference as a source defect.

**Strong evidence:** Repair the boundary, verify the actual caller and preserve warning/error status; do not equate fewer diagnostics with safety.

## References

### Related guides

- [[D13 — Swift 6 Strict Concurrency Migration]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[C11 — Strict Memory Safety Mode and Safe Systems Programming]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F3 — Performance investigation habits]]

### Official sources

- [Swift migration guide: enabling data-race safety](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)
- [SE-0522: source warning control](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0522-source-warning-control.md)
- [Swift.org. "Swift 6.2 Released." Swift.org Blog](https://swift.org/blog/swift-6.2-released/)
- [Swift.org. "Using Upcoming Feature Flags." Swift.org Blog](https://swift.org/blog/using-upcoming-feature-flags/)
- [Swift Evolution. "Control Default Actor Isolation." SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
- [Swift Evolution. "Formalize Language Mode Terminology." SE-0441](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0441-formalize-language-mode-terminology.md)
- [Swift Evolution. "Opt-in Strict Memory Safety Checking." SE-0458](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md)
- [Apple Developer. "Xcode 16 Release Notes." Xcode Release Notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes)
- [Apple Developer. "Embracing Swift concurrency." WWDC25](https://developer.apple.com/videos/play/wwdc2025/268/)
