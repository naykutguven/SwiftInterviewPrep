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

**Learning goals:**

- Separate compiler version, language mode, SDK/module visibility, and runtime availability.
- Choose sufficient #available/@available guards, including anyAppleOS platform overrides.
- Provide an actionable deprecation and a fallback with one observable contract.

**Prerequisites:** [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for function APIs and [[A11 — Access Control]] for public contracts. [[F5 — Language mode, build settings, and migration flags]] explains compiler and language-mode settings.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Installing a new SDK does not upgrade the OS on a user’s device. The SDK can declare an API that your deployment target cannot call unguarded. Runtime availability narrows the OS versions where a call executes. Conditional compilation instead selects which source is compiled.

Keep four requirements separate: the compiler must understand the syntax; the selected language mode must permit the feature; the SDK/module must declare the symbol; the runtime must support executing it. A runtime check cannot manufacture an absent module or teach an older compiler new syntax. [Availability attributes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Choose a guard for the missing capability

This standalone script records compile-time facts. compiler tests the toolchain; swift tests the effective language mode; canImport tests module visibility.

```swift
#if compiler(>=6.4)
    let modernCompiler = true
#else
    let modernCompiler = false
#endif
#if swift(>=6)
    let swiftSixMode = true
#else
    let swiftSixMode = false
#endif
#if canImport(Foundation)
    import Foundation
    let hasFoundation = true
#else
    let hasFoundation = false
#endif
precondition(modernCompiler && hasFoundation)
print("Swift 6 mode: \(swiftSixMode)")
```

**Walkthrough:** On the baseline compiler, compiler is true. A separate Swift 5-mode check keeps that true while swift(>=6) becomes false. Foundation’s existence says nothing about the availability of an individual member.

### A runtime guard narrows a deployment context

Compile this script for macOS 12 using the current SDK: override the baseline target with `-target arm64-apple-macos12.0`. ContinuousClock requires macOS 13.

```swift
import Foundation

func clockKind() -> String {
    if #available(macOS 13, *) {
        _ = ContinuousClock()
        return "continuous"
    } else {
        _ = ProcessInfo.processInfo.systemUptime
        return "uptime"
    }
}
precondition(["continuous", "uptime"].contains(clockKind()))
```

**Walkthrough:** The modern branch gives the compiler a sufficient availability context. Current-host execution uses that branch; compiling the fallback does not prove it ran on a macOS 12 machine.

## Rules and pitfalls

### canImport is not a member/runtime guarantee

**Common pitfall:** Using a module check as the only protection around a newer runtime API. Compile for macOS 12.

**Mistake:** Compile-only; intentionally rejected.

```swift
#if canImport(Foundation)
    import Foundation
    func useClock() { _ = ContinuousClock() }
#endif
```

**Consequence:** The module exists, but the compiler rejects unguarded ContinuousClock use for the older target.

**Why:** canImport asks whether a module is visible during compilation. It does not raise deployment availability.

**Correction:** Keep the module condition when needed and add an appropriate runtime guard.

```swift
#if canImport(Foundation)
    import Foundation
    func clockSupported() -> Bool {
        if #available(macOS 13, *) {
            _ = ContinuousClock()
            return true
        } else {
            return false
        }
    }
    _ = clockSupported()
#endif
```

**What changes:** Both source visibility and runtime requirements are represented. A member absent from the selected SDK still cannot be named merely because an OS guard exists.

### Deprecation does not excuse a silent behavior change

**Common pitfall:** Changing the old name API to normalize case while adding a migration message. Compile/run this warning-bearing script without -warnings-as-errors.

**Mistake:** Runnable with the warning described below; compile without `-warnings-as-errors`.

```swift
func normalizedName(_ text: String) -> String { text.lowercased() }
@available(*, deprecated, message: "Use normalizedName")
func oldName(_ text: String) -> String { normalizedName(text) }
precondition(oldName("Ada") == "ada")  // Legacy spelling was preserved before this change.
```

**Consequence:** Existing callers now receive different text even before they migrate.

**Why:** A warning changes diagnostics, not the old API’s semantic compatibility obligation.

**Correction:** Keep the old behavior behind a forwarding adapter to an equivalent replacement; offer normalization as an explicit choice. This separate script also permits deprecation warnings.

```swift
func verbatimName(_ text: String) -> String { text }
func normalizedName(_ text: String) -> String { text.lowercased() }
@available(*, deprecated, renamed: "verbatimName(_:)")
func oldName(_ text: String) -> String { verbatimName(text) }
precondition(oldName("Ada") == "Ada")
precondition(verbatimName("Ada") == "Ada")
precondition(normalizedName("Ada") == "ada")
```

**What changes:** The old contract remains intact, while the diagnostic points to a behaviorally equivalent API. Normalization remains a deliberate caller decision.

## Worked example

### A platform-specific version overrides the aligned version

**Setup:** Compile-only, targeting macOS 13. The declaration is a teaching API with an intentionally stronger macOS requirement.

**Predict:** Is an anyAppleOS 26 guard sufficient for this function?

```swift
@available(anyAppleOS 26.0, macOS 27.0, *)
func modernFeature() {}
func useFeature() {
    if #available(anyAppleOS 26.0, macOS 27.0, *) { modernFeature() }
}
```

**Outcome:** The matching override compiles. Omitting macOS 27 from the guard would leave a gap on macOS.

**Why:** Aligned availability is shorthand for platform versions, not permission to lower an explicit platform requirement. anyAppleOS accepts aligned versions starting at 26; older versions require their original platform numbers. [Apple’s announcement](https://forums.swift.org/t/introducing-anyappleos/85728)

## Interview questions

### What is the difference between #available and #if canImport?

**Answer:** #available checks runtime platform versions and narrows the compiler’s availability context in that branch. #if canImport chooses source based on whether a module is available during compilation.

**Example:** Runnable script compiled for macOS 12 with the current SDK. Only the current-host runtime branch is executed.

```swift
#if canImport(Foundation)
    import Foundation
    func readUptime() -> Double {
        if #available(macOS 13, *) {
            let clock = ContinuousClock()
            _ = clock.now
        }
        return ProcessInfo.processInfo.systemUptime
    }
    precondition(readUptime() >= 0)
#endif
```

**Walkthrough:** Foundation must be visible before this source is compiled. The inner guard independently permits the newer clock API while preserving a public result on older macOS.

**Boundary:** A runtime guard cannot compensate for an old SDK without that declaration. Use compiler/source guards or a compatible source alternative when the missing requirement occurs at compile time.

### Why can anyAppleOS 26 still fail for macOS, and why is it not an Apple-target test?

**Answer:** A declaration can override the aligned version with a stronger macOS requirement. Also, the availability wildcard covers unlisted platforms; it does not select Apple-only source. Use #if os(anyAppleOS) for target selection and canImport for a required module.

**Example:** Compile-only macOS 13-target source.

```swift
#if os(anyAppleOS)
    @available(anyAppleOS 26.0, macOS 27.0, *)
    func appleFeature() -> String { "ready" }
    func label() -> String {
        if #available(anyAppleOS 26.0, macOS 27.0, *) { appleFeature() } else { "fallback" }
    }
#endif
```

**Walkthrough:** The source guard selects Apple targets. Within that source, the runtime guard honors the macOS-specific override. On a non-Apple target the source is excluded rather than relying on an availability wildcard.

**Boundary:** This check compiles macOS source only. It does not claim Linux or other Apple-target execution. For older compilers, retain explicit platform lists or isolate the newer spelling behind a compiler-version condition.

Separate **intentionally rejected** macOS 13-target call with an insufficient guard:

```swift
@available(anyAppleOS 26.0, macOS 27.0, *)
func modernFeature() {}
func invalid() {
    if #available(anyAppleOS 26.0, *) { modernFeature() }
}
```

### How would you deprecate an API with an actionable migration path?

**Answer:** Introduce an equivalent replacement, keep the old behavior through a compatibility adapter, and annotate it with renamed or a useful message. Check the consumer diagnostic separately from runtime behavior.

**Example:** Standalone warning-bearing runnable script; compile without -warnings-as-errors.

```swift
func displayName(for name: String) -> String { name.isEmpty ? "Guest" : name }
@available(*, deprecated, renamed: "displayName(for:)")
func label(_ name: String) -> String { displayName(for: name) }
precondition(label("") == "Guest")
precondition(label("Ada") == displayName(for: "Ada"))
```

**Walkthrough:** The old spelling still forwards the same behavior, and a consumer receives a rename diagnostic naming the replacement. The two tests cover missing display text and a normal name.

**Boundary:** Deprecation is not removal or an automatic source rewrite. Keep the supported adoption period and explain any behavioral differences instead of presenting a non-equivalent rename as safe.

## Focused exercise

**Task:** Guard a newer API behind a fallback with the same observable units and compile for an older deployment target.

**Acceptance checks:**

- The current SDK compiles the code for macOS 12.
- ContinuousClock is used only within macOS 13 availability.
- Both branches return elapsed seconds; current-host execution is distinguished from older-runtime validation.

**Solution:** Standalone macOS script with Foundation. Override target to arm64-apple-macos12.0.

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

**Reasoning:** Guarded compilation establishes that the API use is valid for the deployment target. Execution on this host exercises the modern branch only. The intentionally empty interval demonstrates units and guards, not performance; deterministic timing tests should inject a clock or equivalent dependency.

## In practice

**Choose it when:** Maintain one public contract across supported compiler, SDK, module, and runtime combinations.

**Investigate by:** Record those four axes separately and choose the guard that addresses the actual missing capability.

**Strong evidence:** Compile the supported target matrix and run each branch on an eligible runtime before claiming deployment validation.

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
