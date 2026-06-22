---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - availability
  - conditional-compilation
---
## 0. Rubric snapshot

**Rubric expectation**

Know `@available`, deprecation messaging, `#available`, `#if`, `canImport`, and source-compatibility guards. The main caveat is that runtime availability and compile-time conditional compilation solve different problems.

**Caveats**

`#available` answers:

```text
Can this already-compiled program safely execute this API on the current OS/runtime?
```

`#if` / `#if canImport` answers:

```text
Should this source code be compiled into this target at all?
```

Swift’s language reference documents `@available` as a declaration attribute for describing a declaration’s availability lifecycle, including introduced, deprecated, obsoleted, unavailable, message, and renamed information. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/?utm_source=chatgpt.com "Attributes | Documentation - Swift Programming Language"))

**You should be able to answer**

- What is the difference between `#available` and `#if canImport`?
- How would you deprecate an API while giving users an actionable migration path?

**You should be able to do**

- Write an API that uses a modern framework when available and a fallback otherwise, without duplicating the public surface.

---

## 1. Core mental model

Availability is about **time** and **environment**.

A Swift codebase can be constrained by at least four different dimensions:

```text
compiler version
Swift language mode
SDK/module availability
runtime OS availability
```

Those are not the same thing.

For example, with Xcode 26, your compiler may understand a new syntax and your SDK may contain an iOS 26 API, but your app may still run on iOS 18 because your minimum deployment target is iOS 18. In that case, the code can compile only if the compiler can prove the iOS 26 API is guarded by a runtime availability check.

The key idea:

```text
#if decides what source exists for this build.
#available decides which already-compiled path is safe at runtime.
@available describes the lifecycle and use constraints of a declaration.
```

`@available` belongs on declarations. It tells clients and the compiler when something was introduced, deprecated, obsoleted, renamed, or unavailable. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/?utm_source=chatgpt.com "Attributes | Documentation - Swift Programming Language"))

`#available` belongs in control flow. It lets you use a newer API inside a guarded block while still supporting older OS versions. Apple’s Xcode documentation says Swift uses `#available` to run code conditionally for a specific platform or OS version. ([Apple Developer](https://developer.apple.com/documentation/xcode/running-code-on-a-specific-version?utm_source=chatgpt.com "Running code on a specific platform or OS version"))

`#if` belongs to compilation. Swift’s language reference says conditional compilation blocks are compiler control statements; only code in the active branch is compiled for that build. ([Swift Belgeleri](https://docs.swift.org/swift-book/ReferenceManual/Statements.html?utm_source=chatgpt.com "Statements | Documentation - Swift Programming Language"))

---

## 2. Essential mechanics

### 2.1 `@available`: annotate declaration lifecycle

Use `@available` to describe when a declaration can be used.

```swift
@available(iOS 26.0, *)
func useNewSystemFeature() {
    // Calls APIs introduced in iOS 26.
}
```

That means the function itself is only available on iOS 26 or newer. A caller in code that supports iOS 18 must guard the call:

```swift
if #available(iOS 26.0, *) {
    useNewSystemFeature()
} else {
    // Fallback for iOS 18–25.
}
```

Use `@available` for deprecation too:

```swift
@available(
    *,
    deprecated,
    renamed: "loadUserProfile(id:)",
    message: "Use loadUserProfile(id:) to get async cancellation and clearer labeling."
)
func fetchUserProfile(userID: User.ID) async throws -> UserProfile {
    try await loadUserProfile(id: userID)
}

func loadUserProfile(id: User.ID) async throws -> UserProfile {
    // New implementation.
}
```

Good deprecations are not passive warnings. They provide:

```text
new symbol
reason
migration path
compatibility bridge if possible
planned removal only when justified
```

### 2.2 `#available`: runtime availability check

Use `#available` when the source can compile against the SDK, but the app might run on an older OS.

```swift
func configureButton(_ button: UIButton) {
    if #available(iOS 26.0, *) {
        // iOS 26-only behavior.
        button.configuration?.buttonSize = .large
    } else {
        // iOS 18–25 fallback.
        button.contentEdgeInsets = UIEdgeInsets(top: 10, left: 16, bottom: 10, right: 16)
    }
}
```

`#available` does **not** hide syntax from an older compiler. It only guards runtime execution.

That matters for library authors supporting multiple Xcode versions. If a declaration or syntax does not exist in an older compiler, `#available` is too late. You need `#if compiler(...)`, `#if swift(...)`, `#if canImport(...)`, or a source split.

### 2.3 `#if`: compile-time source selection

Use `#if` when one branch should not be compiled for a target.

```swift
#if os(iOS)
import UIKit

typealias PlatformImage = UIImage
#elseif os(macOS)
import AppKit

typealias PlatformImage = NSImage
#endif
```

This is not runtime branching. The nonmatching branch is removed from that build.

Common conditions:

```swift
#if os(iOS)
#endif

#if targetEnvironment(simulator)
#endif

#if canImport(UIKit)
#endif

#if compiler(>=6.2)
#endif

#if swift(>=6)
#endif

#if DEBUG
#endif
```

`canImport` tests whether a module can be imported. It does not import the module by itself. Swift’s language reference documents conditional compilation conditions including `os`, `arch`, `swift`, `compiler`, `canImport`, and `targetEnvironment`. ([Swift Belgeleri](https://docs.swift.org/swift-book/ReferenceManual/Statements.html?utm_source=chatgpt.com "Statements | Documentation - Swift Programming Language"))

### 2.4 `#unavailable`: clearer fallback-first code

Use `#unavailable` when the old path is the main branch you want to explain.

```swift
func configureFeature() {
    guard #unavailable(iOS 26.0) else {
        configureModernFeature()
        return
    }

    configureFallbackFeature()
}
```

This often reads better when the fallback is the real implementation and the new API is a narrow fast path.

---

## 3. Common traps and misconceptions

### Trap 1: Using `#available` when the framework might not compile

Bad:

```swift
if #available(iOS 26.0, *) {
    let engine = ModernFrameworkEngine()
}
```

This only works if the compiler and SDK know `ModernFrameworkEngine`.

If the module itself might not exist for some builds, guard the import and implementation:

```swift
#if canImport(ModernTextFramework)
import ModernTextFramework
#endif

func makeTextService() -> any TextServiceBackend {
    #if canImport(ModernTextFramework)
    if #available(iOS 26.0, macOS 26.0, *) {
        return ModernTextFrameworkBackend()
    }
    #endif

    return FallbackTextServiceBackend()
}
```

The point is not the specific framework. The point is the two-level guard:

```text
Can this module/type compile?
Can this code safely run on the current OS?
```

### Trap 2: Thinking `canImport` means “this API exists”

`canImport(UIKit)` means UIKit can be imported. It does **not** mean a specific UIKit API exists, nor that the current runtime OS supports it.

Bad:

```swift
#if canImport(UIKit)
button.someIOS26OnlyAPI()
#endif
```

Better:

```swift
#if canImport(UIKit)
if #available(iOS 26.0, *) {
    button.someIOS26OnlyAPI()
} else {
    fallback()
}
#endif
```

Use both dimensions when necessary:

```text
can the module compile?
can this API run on this OS?
```

### Trap 3: Duplicating public API across availability branches

Bad:

```swift
#if canImport(ModernTextFramework)
public struct TextGenerator {
    public func generate(prompt: String) async throws -> String {
        // Modern framework implementation.
    }
}
#else
public struct TextGenerator {
    public func generate(prompt: String) async throws -> String {
        // Fallback implementation.
    }
}
#endif
```

This creates two public declarations that can drift.

Better:

```swift
public struct TextGenerator {
    private let backend: any TextGeneratingBackend

    public init() {
        self.backend = Self.makeBackend()
    }

    public func generate(prompt: String) async throws -> String {
        try await backend.generate(prompt: prompt)
    }

    private static func makeBackend() -> any TextGeneratingBackend {
        #if canImport(ModernTextFramework)
        if #available(iOS 26.0, macOS 26.0, *) {
            return ModernTextFrameworkBackend()
        }
        #endif

        return FallbackTextGeneratingBackend()
    }
}
```

One public surface. Multiple private implementations.

### Trap 4: Using `#if os(iOS)` when `canImport` is the real constraint

Bad:

```swift
#if os(iOS)
import UIKit
#endif
```

This often works in app code, but packages and cross-platform code can be more precise with capability checks:

```swift
#if canImport(UIKit)
import UIKit
#endif
```

Use `os(...)` when the platform itself is the semantic condition. Use `canImport(...)` when the module is the real dependency boundary.

### Trap 5: Deprecating without a migration path

Bad:

```swift
@available(*, deprecated)
func loadData() async throws -> Data
```

Better:

```swift
@available(
    *,
    deprecated,
    renamed: "data(for:)",
    message: "Use data(for:) because it preserves cancellation and returns response metadata."
)
func loadData() async throws -> Data {
    try await data(for: defaultRequest).data
}
```

A deprecation should reduce migration ambiguity. A warning that only says “deprecated” is usually noise.

---

## 4. Direct answers to rubric questions

### Q1. What is the difference between `#available` and `#if canImport`?

`#available` is a **runtime availability check**. It lets already-compiled code choose a safe execution path depending on the OS version. `#if canImport` is a **compile-time conditional compilation check**. It decides whether source code that depends on a module should be compiled at all.

Interview version:

> `#available` protects calls to APIs that exist in the SDK but might not exist on the runtime OS. `#if canImport` protects source that depends on a module that may not be available for the current build target. They answer different questions: “Can I run this API now?” versus “Can this code be compiled for this target?” In real code, using a new optional framework often needs both: `#if canImport` around the import and implementation, and `#available` around the runtime path.

### Q2. How would you deprecate an API while giving users an actionable migration path?

Use `@available(*, deprecated, renamed:message:)` when there is a replacement symbol. Keep a compatibility wrapper where reasonable. The message should explain the replacement and why users should migrate.

```swift
@available(
    *,
    deprecated,
    renamed: "fetchProfile(id:)",
    message: "Use fetchProfile(id:) because it supports cancellation and avoids ambiguous userID labeling."
)
public func getUser(_ userID: User.ID) async throws -> Profile {
    try await fetchProfile(id: userID)
}

public func fetchProfile(id: User.ID) async throws -> Profile {
    // New implementation.
}
```

Interview version:

> I would avoid a vague deprecation. I would introduce the new API first, make the old API forward to it if possible, and annotate the old symbol with `@available(*, deprecated, renamed:message:)`. The warning should tell clients exactly what to call and why. For a public SDK, I would also document the removal timeline and avoid `obsoleted` until a major version or clearly communicated breaking-change window.

---

## 5. Code probe / examples

There is no rubric code probe for A12. Use these three examples instead.

### 5.1 Minimal example

```swift
func configureSearchExperience() {
    if #available(iOS 26.0, *) {
        configureModernSearchExperience()
    } else {
        configureLegacySearchExperience()
    }
}

@available(iOS 26.0, *)
private func configureModernSearchExperience() {
    // Use iOS 26-only APIs here.
}

private func configureLegacySearchExperience() {
    // iOS 18-compatible fallback.
}
```

What happens:

```text
The iOS 26-only function is callable only inside an iOS 26 availability context.
The fallback remains valid for the iOS 18 minimum deployment target.
```

Why:

```text
The compiler uses the #available branch to prove that iOS 26-only APIs are reached only on iOS 26+.
```

### 5.2 Counterexample

```swift
func configureSearchExperience() {
    configureModernSearchExperience()
}

@available(iOS 26.0, *)
private func configureModernSearchExperience() {
    // Use iOS 26-only APIs here.
}
```

Representative compiler diagnostic with an iOS 18 deployment target:

```text
error: 'configureModernSearchExperience()' is only available in iOS 26.0 or newer
```

Why:

```text
The caller is available on iOS 18, but it unconditionally calls a function that is only available on iOS 26+.
```

Fix:

```swift
func configureSearchExperience() {
    if #available(iOS 26.0, *) {
        configureModernSearchExperience()
    } else {
        configureLegacySearchExperience()
    }
}
```

### 5.3 Production example: one public API, modern framework when available

```swift
public struct TextGenerator {
    private let backend: any TextGeneratingBackend

    public init() {
        self.backend = Self.makeBackend()
    }

    public func generate(prompt: String) async throws -> String {
        try await backend.generate(prompt: prompt)
    }

    private static func makeBackend() -> any TextGeneratingBackend {
        #if canImport(ModernTextFramework)
        if #available(iOS 26.0, macOS 26.0, *) {
            return ModernTextFrameworkBackend()
        }
        #endif

        return FallbackTextGeneratingBackend()
    }
}

private protocol TextGeneratingBackend {
    func generate(prompt: String) async throws -> String
}

private struct FallbackTextGeneratingBackend: TextGeneratingBackend {
    func generate(prompt: String) async throws -> String {
        "Fallback response for: \(prompt)"
    }
}

#if canImport(ModernTextFramework)
import ModernTextFramework

@available(iOS 26.0, macOS 26.0, *)
private struct ModernTextFrameworkBackend: TextGeneratingBackend {
    func generate(prompt: String) async throws -> String {
        // Real ModernTextFramework integration would live here.
        // Keep ModernTextFramework types out of the public API.
        "Modern response for: \(prompt)"
    }
}
#endif
```

Why this design is correct:

```text
Public API:
TextGenerator.generate(prompt:)

Private implementation choices:
ModernTextFrameworkBackend on supported SDK + runtime
FallbackTextGeneratingBackend otherwise
```

The public surface does not duplicate. The optional framework does not leak into the public API. The fallback remains available for iOS 18.

---

## 6. Exercise

### Problem

Write an API that uses a modern framework when available and a fallback otherwise, without duplicating the public surface.

### Bad / naive version

```swift
#if canImport(ModernTextFramework)
import ModernTextFramework

public struct TextGenerator {
    public init() {}

    public func generate(prompt: String) async throws -> String {
        // Modern framework implementation.
        "Modern: \(prompt)"
    }
}
#else
public struct TextGenerator {
    public init() {}

    public func generate(prompt: String) async throws -> String {
        // Fallback implementation.
        "Fallback: \(prompt)"
    }
}
#endif
```

### What is wrong?

```text
1. The public type is declared twice.
2. The two versions can drift.
3. Documentation, tests, and symbol behavior can diverge.
4. The optional dependency shape leaks into the module design.
5. Runtime availability is not handled if the module exists but the OS is older.
```

### Improved version

```swift
public struct TextGenerator {
    private let backend: any TextGeneratingBackend

    public init() {
        self.backend = Self.defaultBackend()
    }

    public func generate(prompt: String) async throws -> String {
        try await backend.generate(prompt: prompt)
    }

    private static func defaultBackend() -> any TextGeneratingBackend {
        #if canImport(ModernTextFramework)
        if #available(iOS 26.0, macOS 26.0, *) {
            return ModernTextFrameworkBackend()
        }
        #endif

        return RuleBasedTextGeneratingBackend()
    }
}

private protocol TextGeneratingBackend {
    func generate(prompt: String) async throws -> String
}

private struct RuleBasedTextGeneratingBackend: TextGeneratingBackend {
    func generate(prompt: String) async throws -> String {
        "Fallback response for: \(prompt)"
    }
}

#if canImport(ModernTextFramework)
import ModernTextFramework

@available(iOS 26.0, macOS 26.0, *)
private struct ModernTextFrameworkBackend: TextGeneratingBackend {
    func generate(prompt: String) async throws -> String {
        // Integrate ModernTextFramework here.
        "Modern framework response for: \(prompt)"
    }
}
#endif
```

### Why this is better

The public API is stable:

```swift
TextGenerator.generate(prompt:)
```

The implementation is flexible:

```text
new SDK + new OS       -> modern framework backend
old OS or unavailable  -> fallback backend
```

The design also respects module boundaries. Callers do not need to import the modern framework, and your public SDK does not expose modern framework types. That matters for source compatibility, binary distribution, and dependency hygiene.

---

## 7. Production guidance

Use this in production when:

```text
You support an older minimum deployment target.
You adopt newly introduced Apple APIs.
You build cross-platform packages.
You support multiple Xcode or Swift language versions.
You maintain public APIs and need deprecation paths.
You ship SDKs that must avoid leaking optional dependencies.
```

Be careful when:

```text
A new API exists in the SDK but not on your minimum runtime OS.
A module exists on some platforms but not others.
The code must compile with multiple compiler versions.
A public API signature mentions types from optional frameworks.
You use conditional compilation around public declarations.
```

Avoid when:

```text
A normal runtime feature flag would be clearer.
You are using #if DEBUG to hide correctness problems.
You are duplicating entire public types across #if branches.
You are using #if os(...) when capability or module availability is the actual condition.
You are using deprecation warnings without a replacement or rationale.
```

Debugging checklist:

```text
Is this a compiler-version problem, SDK problem, module-import problem, or runtime OS problem?
Does the symbol need @available?
Does the call site need #available?
Does the import or implementation need #if canImport?
Does a public API expose an optional dependency type?
Are old and new implementations tested through the same public API?
Does the deprecation tell users exactly what to do next?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `#available` checks OS versions, and `#if` compiles code conditionally.

### Senior answer

> `#available` is a runtime guard used when the code can compile but may run on an older OS. `#if` is compile-time source selection. `canImport` checks module availability, not individual API availability. I avoid duplicating public APIs across conditional branches and keep conditional logic behind stable internal abstractions.

### Staff-level answer

> Availability is part of API and release engineering. I separate compiler, language mode, SDK, module, and runtime constraints. For public libraries, I keep optional dependencies out of public signatures, use `@available` and deprecation annotations as client-facing contracts, and centralize fallback selection behind one public surface. I also test both modern and fallback paths, because availability code often rots when teams only run on the latest OS.

Staff-level questions to ask:

```text
What is the minimum deployment target?
Which Xcode and SDK versions must compile this package?
Is the dependency optional at compile time, runtime, or both?
Does the public API expose types from a framework that may not exist everywhere?
Can the fallback and modern implementation share the same tests?
Is the deprecation source-compatible, binary-compatible, and actually actionable?
```

---

## 9. Interview-ready summary

Availability in Swift has three different layers. `@available` annotates declarations and communicates lifecycle: introduced, deprecated, obsoleted, renamed, or unavailable. `#available` is runtime control flow that lets code safely call newer APIs while supporting older OS versions. `#if`, including `#if canImport`, is compile-time source selection that decides whether code exists in a build at all. A senior design keeps one public API surface and hides platform/framework variation behind private implementations. A staff-level design also treats availability as part of compatibility strategy: migration messages, fallback testing, dependency boundaries, and long-term source stability.

---

## 10. Flashcards

Q: What does `@available` do?

A: It annotates a declaration’s lifecycle or constraints, such as introduced version, deprecation, obsoletion, unavailability, rename, or platform-specific availability.

Q: What does `#available` do?

A: It checks runtime OS availability in control flow and lets the compiler verify that newer APIs are only used in safe branches.

Q: What does `#if canImport(Module)` do?

A: It checks at compile time whether a module can be imported for the current build target. It does not import the module by itself.

Q: Why is `#available` not enough when a framework might not exist?

A: Because the compiler still needs to parse and type-check the code. If the module or type is absent from the SDK, the source cannot compile unless excluded with conditional compilation.

Q: Why can `canImport` be insufficient for a new API?

A: Because the module may exist while a specific API is available only on newer OS versions. You still need `#available` for runtime safety.

Q: What is the difference between `compiler(>=6.2)` and `swift(>=6)`?

A: `compiler` checks the compiler version. `swift` checks the Swift language mode being compiled.

Q: What makes a deprecation actionable?

A: It names the replacement, explains the reason, preserves compatibility where possible, and gives clients a clear migration path.

Q: Why avoid duplicating public declarations in `#if` branches?

A: Duplicated public surfaces drift over time and create inconsistent behavior, documentation, tests, and compatibility guarantees.

---

## 11. Related sections

- [[A7 — Functions, parameter labels, defaults, and API surface]]
- [[A10 — Extensions and retroactive modeling]]
- [[A11 — Access control]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[E6 — Import visibility and dependency leakage]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — A12 availability and conditional compilation.    
- Swift Book — Attributes / `@available`: [https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/) ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/?utm_source=chatgpt.com "Attributes | Documentation - Swift Programming Language"))
- Swift Book — Statements / conditional compilation, `#if`, `canImport`, `compiler`, `swift`, `#available`, `#unavailable`: [https://docs.swift.org/swift-book/ReferenceManual/Statements.html](https://docs.swift.org/swift-book/ReferenceManual/Statements.html) ([Swift Belgeleri](https://docs.swift.org/swift-book/ReferenceManual/Statements.html?utm_source=chatgpt.com "Statements | Documentation - Swift Programming Language"))
- Apple Developer Documentation — Running code on a specific platform or OS version: [https://developer.apple.com/documentation/xcode/running-code-on-a-specific-version](https://developer.apple.com/documentation/xcode/running-code-on-a-specific-version) ([Apple Developer](https://developer.apple.com/documentation/xcode/running-code-on-a-specific-version?utm_source=chatgpt.com "Running code on a specific platform or OS version"))