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

**Goal:** Reproduce a concurrency diagnostic under two language modes and explain which setting changes its severity.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A target's compiler settings are part of its semantics. The compiler release determines available language features; language mode selects compatibility and checking rules. The SDK supplies declarations, while the deployment target states the oldest supported runtime. Installing Swift 6.4 does not make every target use Swift 6 mode or make every SDK API deployable.

Swift 6 mode enables complete data-race checking. Approachable Concurrency changes selected inference and async execution rules; Default Actor Isolation independently determines eligible unannotated declarations. Imported APIs keep the contract established by their defining module. [Swift migration guide: enabling data-race safety](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)

## Rules and pitfalls

### Record a complete configuration

Use `-swift-version 6`, not `6.4`. A package's `swift-tools-version: 6.4` sets its minimum tools and manifest version, independently of per-target language mode. These are the baseline compiler arguments:

```sh
-swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances
```

Equivalent SwiftPM target settings, inside a tools-version 6.4 manifest:

```swift
swiftSettings: [
    .swiftLanguageMode(.v6),
    .defaultIsolation(nil),
    .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
    .enableUpcomingFeature("InferIsolatedConformances")
]
```

**Common pitfall:** Comparing only Xcode versions misses target-specific inference and warning policy. Capture effective compiler commands for app, package, extension, generated-code, and test targets, including SDK, architecture, optimization, and deployment target.

### Match the safety control to the problem

Strict concurrency checks isolation and transfer, including `Sendable` captures. Default isolation changes inference; it supplies no general-purpose synchronization. Strict memory safety diagnoses unsafe memory operations and APIs; enabling it does not prove a pointer wrapper correct. Its audit belongs in [[C11 — Strict Memory Safety Mode and Safe Systems Programming]].

**Common pitfall:** Adding default `MainActor` isolation merely to reduce errors changes the API and can serialize expensive work. Keep this collection's nonisolated default; apply explicit `@MainActor` to UI-owned state. Use [[D14 — Approachable Concurrency and Default Isolation]] to decide execution contracts.

### Treat warning policy as a migration tool

A warning exception needs a reason and removal condition. `@diagnose` can control warning groups, but cannot demote a native Swift 6 error. Warnings-as-errors helps prevent regressions after a target migrates; first verify that the specific diagnostic actually fails the build. [SE-0522: source warning control](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0522-source-warning-control.md)

## Interview questions

### What do strict concurrency, default isolation, and strict memory safety surface?

They address cross-isolation safety, inferred declaration isolation, and unsafe memory boundaries respectively. A target can pass concurrency checking while containing a dangling pointer, or compile with nonisolated defaults while explicitly isolating its UI model. Choose the control from the violated invariant.

### Why can two targets diagnose the same source differently?

Each target compiles under its own configuration. Compare language mode, feature flags, isolation, imported SDK annotations, and warning policy before changing source. A test importing production code must respect production isolation; its defaults do not rewrite the imported API.

### Can `@diagnose` suppress a Swift 6 Sendable error or re-infer imported isolation?

No to both. Repair the ownership or isolation boundary for a native error. Warning policy changes reporting, and client defaults apply to client declarations; neither rewrites a separately compiled module's contract.

## Focused exercise

**Task:** Compile the same capture with Swift 5 complete checking and Swift 6 mode. Keep the compiler, SDK, nonisolated default, and both feature opt-ins fixed.

**Acceptance checks:**

- Swift 5 emits a concurrency warning and succeeds without warnings-as-errors.
- Swift 6 rejects the unsafe capture; an actor-based replacement compiles.
- The report records commands and exit statuses rather than relying on diagnostic wording.

**Solution:** Save this standalone compile-only fixture as `Capture.swift`. `accept` is deliberately a stub: this exercise tests capture checking, not scheduling.

```swift
final class Cache { var count = 0 }
func accept(_ operation: @Sendable @escaping () -> Void) {}
func probe() {
    let cache = Cache()
    accept { cache.count += 1 }
}
```

Compile to an object with each mode; supply the two opt-ins shown above to both commands:

```sh
swiftc -swift-version 5 -strict-concurrency=complete -default-isolation nonisolated -enable-upcoming-feature NonisolatedNonsendingByDefault -enable-upcoming-feature InferIsolatedConformances -c Capture.swift -o Capture5.o
swiftc -swift-version 6 -default-isolation nonisolated -enable-upcoming-feature NonisolatedNonsendingByDefault -enable-upcoming-feature InferIsolatedConformances -c Capture.swift -o Capture6.o
```

Repair by replacing `Cache` with an actor and accepting an async sendable operation:

```swift
actor Cache {
    private var count = 0
    func increment() { count += 1 }
}
func accept(_ operation: @Sendable @escaping () async -> Void) {}
func probe() {
    let cache = Cache()
    accept { await cache.increment() }
}
```

**Reasoning:** The original closure can be invoked across isolation domains while retaining a mutable ordinary class. The actor serializes its state and makes the closure's capture safe. This repairs the contract rather than changing diagnostic policy.

## In practice

**Choose it when:** Define shared settings when modules migrate at different speeds. Use a documented exception ledger for dependencies and a clean build of every affected target.

**Investigate by:** Compare emitted compiler invocations and separately compile a client against the public module interface.

**Strong evidence:** Reproduce a diagnostic, repair its boundary, and enforce an intentional per-target policy in CI, including a deliberately failing diagnostic gate.

### Diagnostic edge cases — optional

The earlier toolchain review found scoped warning-promotion exceptions for metatype captures and `@unknown default`. Those observations are not general rules about warning policy; reproduce them before building a release gate. See [[B6 — Any, AnyObject, Metatypes, and Runtime Type Information]] and [[B11 — Library Evolution and Resilience-Aware API Design]]. `InternalImportsByDefault` and `MemberImportVisibility` are separate controls covered by [[E6 — Import Visibility and Dependency Leakage]].

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
- [Swift.org. "Enable data-race safety checking." Swift 6 Concurrency Migration Guide](https://swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)
- [Swift.org. "Using Upcoming Feature Flags." Swift.org Blog](https://swift.org/blog/using-upcoming-feature-flags/)
- [Swift Evolution. "Control Default Actor Isolation." SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
- [Swift Evolution. "Formalize Language Mode Terminology." SE-0441](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0441-formalize-language-mode-terminology.md)
- [Swift Evolution. "Opt-in Strict Memory Safety Checking." SE-0458](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0458-strict-memory-safety.md)
- [Apple Developer. "Xcode 16 Release Notes." Xcode Release Notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes)
- [Apple Developer. "Embracing Swift concurrency." WWDC25](https://developer.apple.com/videos/play/wwdc2025/268/)
