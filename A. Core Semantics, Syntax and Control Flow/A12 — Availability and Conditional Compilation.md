---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - availability
  - conditional-compilation
---

# A12 — Availability and Conditional Compilation

**Goal:** Select a runtime guard and fallback without confusing SDK, compiler, or module availability.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Compiler support, language mode, SDK declarations, and runtime support are independent requirements. An installed SDK can describe APIs unavailable on the deployment target. Runtime guards narrow availability inside a branch; compile-time conditions exclude source before type checking. A runtime guard cannot teach an older compiler a new syntax or manufacture an absent module. [Attributes: availability](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/)

## Rules and pitfalls

### Match the guard to the missing capability

Use #if compiler for compiler syntax support, #if swift for the effective language version, canImport for module visibility, and #available or an enclosing @available declaration for runtime API availability. Inspect the selected SDK's exact introduction versions.

**Common pitfall:** canImport confirms a module, not every member or a supported runtime. Nest the appropriate runtime guard around newer API use and keep a fallback with the same public behavior.

### Make deprecation actionable

Mark an old declaration deprecated with a useful message or renamed replacement when semantics align. Keep a compatibility adapter long enough for supported clients, and test warning policy separately from runtime behavior.

**Common pitfall:** Removing or silently changing the old operation while adding a deprecation message is still a compatibility break. Document the replacement's behavioral differences and adoption period.

## Interview questions

### What is the difference between `#available` and `#if canImport`?

#available checks runtime platform versions and gives the compiler a narrowed availability context. #if canImport selects source based on module availability during compilation. A module can import successfully while a particular call still requires a newer SDK or runtime, so one guard does not replace the other.

### Why can a call guarded by anyAppleOS 26 still fail for macOS, and why is that guard not a test for an Apple target?

The shorthand's aligned version can be lower than a specific API's macOS introduction, requiring an explicit macOS override. The wildcard also makes availability conditions true on unlisted platforms; it is not an Apple-target test. Combine os/canImport source guards with exact platform availability when needed.

### How would you deprecate an API while giving users an actionable migration path?

Introduce a replacement with a clear contract, annotate the old declaration with deprecated and an actionable message or renamed entry, and provide a forwarding adapter if behavior is equivalent. Compile a consumer to check the diagnostic and test both paths; do not use deprecation to conceal a new policy.

## Focused exercise

**Task:** Guard one newer API behind a fallback. Explain runtime availability versus compiler/module guards and compile for the older deployment target.

**Acceptance checks:**

- The source compiles against the current SDK with the older deployment target.
- The newer call has a sufficient availability context.
- The fallback exposes the same units; older-runtime execution is reported separately.

**Solution:**

Standalone macOS example, Foundation required. Compile with a macOS 12 deployment target using the installed SDK; ContinuousClock requires macOS 13.

```swift
import Foundation
func elapsedWork() -> Double {
    if #available(macOS 13, *) {
        let start = ContinuousClock.now
        let duration = start.duration(to: .now)
        return Double(duration.components.seconds)
            + Double(duration.components.attoseconds) / 1e18
    } else {
        let start = ProcessInfo.processInfo.systemUptime
        return ProcessInfo.processInfo.systemUptime - start
    }
}
precondition(elapsedWork() >= 0)
```

This measures an intentionally empty interval to demonstrate guarding, not a performance benchmark. An older-runtime fallback must also run on that runtime before claiming deployment validation.

**Reasoning:** SDK compilation checks guarded use, while executing on a current OS only exercises the modern branch. A production elapsed-time abstraction should use an injected Clock or equivalent dependency when tests need deterministic time.

## In practice

**Choose it when:** Maintain one public contract across supported compiler and runtime combinations.

**Investigate by:** Record compiler version, SDK, deployment target, and module imports independently.

**Strong evidence:** Compile the supported matrix and run both branches on eligible runtimes rather than equating guarded compilation with deployment success.

### Aligned Apple versions — optional

#### `anyAppleOS` in Swift 6.4

`@available(anyAppleOS 27.0, *)` and `#available(anyAppleOS 27.0, *)` cover the aligned Apple OS versions. Use versions 26.0 or later; older OS releases need their original platform-specific numbers. A specific platform requirement overrides the shared value. [Apple’s announcement](https://forums.swift.org/t/introducing-anyappleos/85728)

The following compile-only probe deliberately needs a stronger guard on macOS:

```swift
@available(anyAppleOS 26.0, macOS 27.0, *)
func modernFeature() {}

func useFeature() {
    if #available(anyAppleOS 26.0, *) {
        modernFeature() // error when targeting macOS 13; accepted targeting iOS 18
    }
}
```

For that declaration, fix the call with `if #available(anyAppleOS 26.0, macOS 27.0, *)`. Do not silently lower its macOS requirement.

`#if os(anyAppleOS)` is compile-time platform selection. An availability check with `*` is not an Apple-platform selection test and does not establish that UIKit, AppKit, or another module exists. Keep `canImport` where module availability is the condition:

```swift
#if os(anyAppleOS)
@available(anyAppleOS 27.0, *)
func appleOnlyFeature() {}
#endif
```

Migration from 6.2/6.3: keep explicit platform lists when older compilers must build the source, or isolate the newer spelling behind an appropriate compiler-version branch. The older spelling remains valid in 6.4.

## References

### Related guides

- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A10 — Extensions and Retroactive Modeling]]
- [[A11 — Access Control]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [Attributes: availability](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/)
- [Apple’s announcement](https://forums.swift.org/t/introducing-anyappleos/85728)
- [Swift.org. "Statements." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/statements/)
- [Apple Developer. "Running Code on a Specific Platform or OS Version." Apple Developer Documentation](https://developer.apple.com/documentation/xcode/running-code-on-a-specific-version)
