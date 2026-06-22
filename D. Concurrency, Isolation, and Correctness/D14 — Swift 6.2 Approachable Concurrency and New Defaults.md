---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - swift6-2
  - main-actor
  - approachable-concurrency
---
## 0. Rubric snapshot

**Rubric expectation**

Understand Swift 6.2’s approachable concurrency changes: optional default isolation to `MainActor`, `@concurrent`, and the changed/evolving execution semantics of `nonisolated async`. The rubric explicitly calls this section version-sensitive and expects staff-level awareness of what depends on language version or build settings.

**Caveats**

This is not just syntax. These settings change what unannotated code _means_.

You should be able to answer:

- What problem is default actor isolation to `MainActor` trying to solve, and what does it trade off?
- Why does Swift 6.2’s change to `nonisolated async` behavior matter for app code?
- When would you reach for `@concurrent` to make concurrency explicit?

You should be able to do:

- Review a UI-heavy package and decide whether default isolation to `MainActor` is appropriate, partial, or harmful.

---

## 1. Core mental model

Swift 6 made data-race safety explicit, but many app codebases are not architected as “everything can run anywhere.” UI-heavy iOS/macOS code often already assumes a single UI isolation domain: the main actor. Swift 6.2’s approachable concurrency features let you tell the compiler that assumption once at the module/target level instead of sprinkling `@MainActor` everywhere. Swift.org describes this as lowering the barrier to concurrent programming by reducing boilerplate while preserving data-race safety. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))

The key idea:

```text
Old default: unannotated code is nonisolated.
Swift 6.2 option: unannotated code in this module is MainActor-isolated.
```

This does **not** mean Swift is globally `@MainActor` by default. Default actor isolation is controlled per module/target. SE-0466 defines a `-default-isolation` compiler flag and a SwiftPM setting; if unspecified, the module default remains `nonisolated`. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md "swift-evolution/proposals/0466-control-default-actor-isolation.md at main · swiftlang/swift-evolution · GitHub"))

The second major change is about `async` execution. Historically, a `nonisolated async` function switched off the caller’s actor to the generic executor. With Swift 6.2’s upcoming-feature behavior, a nonisolated async function can instead run in the caller’s execution context, so calling it from the main actor does not automatically mean it leaves the main actor. SE-0461 calls this `nonisolated(nonsending)` behavior, and says `@concurrent` is the explicit spelling for the old “switch off actor” behavior. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md "swift-evolution/proposals/0461-async-function-isolation.md at main · swiftlang/swift-evolution · GitHub"))

The staff-level point:

```text
async does not mean background.
nonisolated does not necessarily mean off-main.
@concurrent means “I intentionally want this async function to run concurrently off the actor.”
```

---

## 2. Essential mechanics

### 2.1 Default actor isolation is a module/target setting

With default isolation set to `MainActor`, unannotated declarations in that module are inferred as `@MainActor` unless another isolation rule applies. SE-0466 says the valid compiler values are `MainActor` and `nonisolated`; no setting means `nonisolated`. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md "swift-evolution/proposals/0466-control-default-actor-isolation.md at main · swiftlang/swift-evolution · GitHub"))

SwiftPM example:

```swift
// swift-tools-version: 6.2

let package = Package(
    name: "ProfileFeature",
    targets: [
        .target(
            name: "ProfileUI",
            swiftSettings: [
                .defaultIsolation(MainActor.self)
            ]
        ),
        .target(
            name: "ProfileCore",
            swiftSettings: [
                .defaultIsolation(nil) // Equivalent to nonisolated
            ]
        )
    ]
)
```

In an app target or UI feature target, this can reduce noise:

```swift
// Built with -default-isolation MainActor

final class ProfileViewModel {
    var title = ""

    func updateTitle(_ title: String) {
        self.title = title
    }
}
```

Conceptually, this behaves like:

```swift
@MainActor
final class ProfileViewModel {
    var title = ""

    func updateTitle(_ title: String) {
        self.title = title
    }
}
```

### 2.2 Default isolation does not apply everywhere

SE-0466 lists important exceptions: explicit actor isolation wins; actor types have their own isolation; declarations that cannot have global actor isolation are excluded; protocol/superclass/conformance inference can override the module default; nested types inside nonisolated types are also special. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md "swift-evolution/proposals/0466-control-default-actor-isolation.md at main · swiftlang/swift-evolution · GitHub"))

Example:

```swift
// Built with -default-isolation MainActor

actor ImageCache {
    private var storage: [URL: Data] = [:]

    func data(for url: URL) -> Data? {
        storage[url]
    }
}
```

`ImageCache` is an actor. Its isolated state belongs to `ImageCache`, not to `MainActor`.

### 2.3 `Task {}` and `Task.detached {}` still differ

Under default `MainActor` isolation, a `Task {}` created inside a main-actor-isolated context inherits that isolation. `Task.detached {}` does not. SE-0466 explicitly shows `Task {}` inheriting the enclosing context and `Task.detached {}` remaining nonisolated. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md "swift-evolution/proposals/0466-control-default-actor-isolation.md at main · swiftlang/swift-evolution · GitHub"))

```swift
// Built with -default-isolation MainActor

final class SearchViewModel {
    var query = ""

    func start() {
        Task {
            // MainActor-isolated because `start()` is MainActor-isolated.
            query = "Swift"
        }

        Task.detached {
            // Not MainActor-isolated.
            // Cannot safely mutate `query` here.
        }
    }
}
```

### 2.4 `nonisolated async` no longer necessarily means “runs elsewhere”

With the new Swift 6.2 upcoming-feature behavior, `nonisolated async` can run on the caller’s actor. SE-0461 says `nonisolated(nonsending)` functions run on the caller’s actor and do not send arguments/results across an isolation boundary by default. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md "swift-evolution/proposals/0461-async-function-isolation.md at main · swiftlang/swift-evolution · GitHub"))

```swift
// Built with default MainActor isolation
// and NonisolatedNonsendingByDefault behavior.

final class ImageViewModel {
    func load() async {
        let image = await ImageDecoder.decode(data)
        // `decode` may still execute on the MainActor if called from here.
    }
}

nonisolated enum ImageDecoder {
    static func decode(_ data: Data) async -> UIImage {
        // Do not assume this is off-main just because it is async.
        UIImage(data: data) ?? UIImage()
    }
}
```

That is the dangerous misconception: `async` is about suspension, not automatic background execution.

### 2.5 Use `@concurrent` when off-actor execution is intentional

`@concurrent` says the function should switch off the caller’s actor and run concurrently with other tasks on that actor. SE-0461 describes it as the explicit spelling for the old/default nonisolated async behavior in Swift language modes up to Swift 6, and Swift.org presents it as the tool for introducing concurrency explicitly. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md "swift-evolution/proposals/0461-async-function-isolation.md at main · swiftlang/swift-evolution · GitHub")) ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))

```swift
struct ImageDecoder {
    @concurrent
    static func decode(_ data: Data) async throws -> Image {
        try await Task.checkCancellation()
        return try expensiveDecode(data)
    }
}
```

Use `@concurrent` for expensive CPU-bound work, parsing, image decoding, compression, cryptography, or other work that must not remain serialized on the main actor.

---

## 3. Common traps and misconceptions

### Trap 1: “Swift 6.2 makes everything MainActor by default”

Wrong. Default actor isolation is opt-in per module/target. SE-0466 explicitly says existing/default behavior remains `nonisolated` unless the module opts into `MainActor`. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md "swift-evolution/proposals/0466-control-default-actor-isolation.md at main · swiftlang/swift-evolution · GitHub"))

Bad mental model:

```text
Swift 6.2 == MainActor by default everywhere
```

Better mental model:

```text
Each module has a default isolation policy.
UI modules may choose MainActor.
Reusable/domain/server modules often should remain nonisolated.
```

### Trap 2: “async means background”

Bad:

```swift
@MainActor
final class FeedViewModel {
    func refresh() async {
        let models = await parseLargeJSON(data)
        self.models = models
    }
}

nonisolated func parseLargeJSON(_ data: Data) async -> [FeedModel] {
    // May still run on the caller's actor under new semantics.
    expensiveParse(data)
}
```

Better:

```swift
@MainActor
final class FeedViewModel {
    func refresh() async throws {
        let models = try await parseLargeJSON(data)
        self.models = models
    }
}

@concurrent
func parseLargeJSON(_ data: Data) async throws -> [FeedModel] {
    try Task.checkCancellation()
    return try expensiveParse(data)
}
```

Why: under the new `nonisolated async` behavior, the first version may keep CPU work on the main actor. `@concurrent` makes the off-actor intent explicit.

### Trap 3: Blanket `MainActor` isolation for packages

A package named `FeatureKit` often contains mixed concerns:

```text
FeatureKit
- SwiftUI screens
- View models
- Domain models
- Networking
- Persistence
- Image decoding
- Test fixtures
```

Making the whole package default to `MainActor` is usually too broad. It turns domain and infrastructure code into UI-bound code. SE-0466 explicitly notes that `MainActor` is the wrong default for many modules, including general libraries and highly concurrent server applications. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md "swift-evolution/proposals/0466-control-default-actor-isolation.md at main · swiftlang/swift-evolution · GitHub"))

Better package split:

```text
FeatureUI        -> defaultIsolation(MainActor.self)
FeatureDomain    -> nonisolated
FeatureNetworking -> nonisolated
FeatureImageIO   -> nonisolated, with selected @concurrent work
```

### Trap 4: Using `nonisolated` as a performance escape hatch

`nonisolated` removes actor isolation from a declaration. Under Swift 6.2’s new async behavior, it does not necessarily force execution onto a background executor. Use `@concurrent` when the actual requirement is off-actor concurrent execution. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md "swift-evolution/proposals/0461-async-function-isolation.md at main · swiftlang/swift-evolution · GitHub"))

---

## 4. Direct answers to rubric questions

### Q1. What problem is default actor isolation to `MainActor` trying to solve, and what does it trade off?

It solves annotation overload for code that is already logically single-threaded or UI-bound. Instead of requiring every view model, UI service, observable model, and UI mutation method to say `@MainActor`, the module can infer that by default. Swift.org says this option is ideal for scripts, UI code, and executable targets. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))

The tradeoff is that it can accidentally make too much code main-actor-isolated: domain models, pure transformations, networking helpers, parsers, persistence models, test utilities, and public library APIs. This harms reuse, creates unnecessary actor hops, and can put expensive work on the main actor. It is also source-incompatible to change a module’s default isolation later. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md "swift-evolution/proposals/0466-control-default-actor-isolation.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> Default `MainActor` isolation is a migration and ergonomics tool for code that is already UI-main-thread-oriented. It reduces boilerplate and aligns the compiler with the app’s real isolation model. The tradeoff is architectural: if you apply it too broadly, you accidentally turn general-purpose code into UI-bound code. I would use it for app/UI targets, but keep domain, networking, persistence, parsing, and reusable library targets nonisolated unless there is a clear reason.

### Q2. Why does Swift 6.2’s change to `nonisolated async` behavior matter for app code?

Because old intuition says “a `nonisolated async` function leaves the actor,” but the Swift 6.2 upcoming-feature behavior says it can run in the caller’s execution context. That makes app code safer for non-`Sendable` values because arguments are not necessarily sent across an isolation boundary, but it also means expensive async work called from the main actor may remain on the main actor. SE-0461 calls out both source-compatibility and performance implications. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md "swift-evolution/proposals/0461-async-function-isolation.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> The change matters because `async` is no longer a reliable signal that work leaves the main actor. This improves data-race safety and makes class-heavy app code easier to migrate, but it means CPU-heavy async functions need explicit design. If I want parsing or decoding to leave the main actor, I should say that with `@concurrent`, not rely on old `nonisolated async` behavior.

### Q3. When would you reach for `@concurrent`?

Use `@concurrent` when the function’s semantic contract is: “this work should run concurrently off the caller’s actor.” Common cases are image decoding, JSON parsing, compression, media processing, cryptography, expensive normalization, and other CPU-bound work that would make the main actor unresponsive. Swift.org’s own example uses `@concurrent` to fetch/decode image data while keeping the main actor free. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))

Interview version:

> I use `@concurrent` when off-actor execution is part of the API’s correctness or performance contract. It is not a random optimization annotation. It tells readers and the compiler that values may cross an isolation boundary and that the work can run concurrently with the caller’s actor, so I need the arguments and result model to be safe for that.

---

## 5. Code probe

The rubric has no D14 code probe. Use these three probes instead.

### Probe A — minimal example

Given:

```swift
// Build setting:
// -default-isolation MainActor

final class CounterViewModel {
    var count = 0

    func increment() {
        count += 1
    }
}
```

### What happens?

```text
Compiles.

`CounterViewModel`, `count`, and `increment()` are inferred as MainActor-isolated.
There is no printed output.
```

### Why?

```text
Module default isolation = MainActor
        ↓
Unannotated class declaration
        ↓
Implicit MainActor isolation
        ↓
Mutable UI-ish state is protected by the main actor
```

This is the happy path: a UI/view-model type that was already conceptually main-thread-only becomes compiler-checked without repeated `@MainActor`.

---

### Probe B — counterexample

Given:

```swift
// Build setting:
// -default-isolation MainActor

struct SearchIndex {
    private var entries: [String: [Int]] = [:]

    mutating func insert(_ token: String, documentID: Int) {
        entries[token, default: []].append(documentID)
    }
}
```

### What happens?

```text
Compiles, but the type is inferred as MainActor-isolated.
There is no printed output.
```

### Why this is suspicious

`SearchIndex` looks like a pure domain/data structure. If it is inferred as `@MainActor`, it becomes awkward to build, mutate, or test from background indexing work. The fix is not to add more annotations randomly. The better design is to keep this type in a nonisolated target or explicitly mark it `nonisolated` if the module default is `MainActor`.

Better:

```swift
nonisolated struct SearchIndex {
    private var entries: [String: [Int]] = [:]

    mutating func insert(_ token: String, documentID: Int) {
        entries[token, default: []].append(documentID)
    }
}
```

Even better in a real codebase: move it to a nonisolated `SearchCore` target.

---

### Probe C — production example

Bad:

```swift
// Build setting:
// -default-isolation MainActor
// NonisolatedNonsendingByDefault enabled

final class ImageViewModel {
    var image: Image?

    func load(from url: URL) async throws {
        let data = try await HTTPClient.fetch(url)
        image = await ImageDecoder.decode(data)
    }
}

nonisolated enum ImageDecoder {
    static func decode(_ data: Data) async -> Image {
        expensiveDecode(data)
    }
}
```

### What happens?

```text
This may compile, but `decode(_:)` should not be assumed to run off the main actor.
There is no deterministic printed output.
```

### Why?

Under the new behavior, a `nonisolated async` function can inherit the caller’s execution context. If `load(from:)` is main-actor-isolated, `decode(_:)` may execute on the main actor unless explicitly made concurrent.

Better:

```swift
final class ImageViewModel {
    var image: Image?

    func load(from url: URL) async throws {
        let data = try await HTTPClient.fetch(url)
        image = try await ImageDecoder.decode(data)
    }
}

nonisolated enum ImageDecoder {
    @concurrent
    static func decode(_ data: Data) async throws -> Image {
        try Task.checkCancellation()
        return try expensiveDecode(data)
    }
}
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`@concurrent` function|CPU-heavy async work should leave the caller’s actor|Requires safe crossing of isolation boundaries|
|Separate nonisolated target|Domain/infrastructure code should be reusable from any actor|More module factoring and dependency management|
|Explicit `nonisolated` declaration|A few declarations in a `MainActor` default target are pure/general-purpose|Can become noisy if overused|
|Actor-owned worker|Stateful background service needs serialized mutable state|Actor reentrancy and API design still matter|
|MainActor default everywhere|Small script or simple UI app|Harmful for libraries and performance-sensitive internals|

---

## 6. Exercise

### Problem

Review a UI-heavy package and decide whether default isolation to `MainActor` is appropriate, partial, or harmful.

### Naive package

```text
ShoppingFeature
├── CartView.swift
├── CartViewModel.swift
├── CartState.swift
├── CartItem.swift
├── CartPriceCalculator.swift
├── CartAPIClient.swift
├── CartImageDecoder.swift
├── CartRepository.swift
└── CartSearchIndex.swift
```

Naive setting:

```swift
.target(
    name: "ShoppingFeature",
    swiftSettings: [
        .defaultIsolation(MainActor.self)
    ]
)
```

### What is wrong?

```text
The package is UI-heavy but not UI-only.
```

`CartView` and `CartViewModel` probably belong on the main actor. But `CartItem`, `CartState`, `CartPriceCalculator`, `CartAPIClient`, `CartImageDecoder`, `CartRepository`, and `CartSearchIndex` are not inherently UI-bound. Blanket default `MainActor` isolation would leak UI isolation into pure domain and infrastructure code.

### Improved version

```text
ShoppingFeatureUI
├── CartView.swift
└── CartViewModel.swift

ShoppingFeatureDomain
├── CartState.swift
├── CartItem.swift
└── CartPriceCalculator.swift

ShoppingFeatureData
├── CartAPIClient.swift
├── CartRepository.swift
└── CartSearchIndex.swift

ShoppingFeatureMedia
└── CartImageDecoder.swift
```

SwiftPM sketch:

```swift
// swift-tools-version: 6.2

let package = Package(
    name: "ShoppingFeature",
    targets: [
        .target(
            name: "ShoppingFeatureUI",
            dependencies: [
                "ShoppingFeatureDomain",
                "ShoppingFeatureData",
                "ShoppingFeatureMedia"
            ],
            swiftSettings: [
                .defaultIsolation(MainActor.self)
            ]
        ),

        .target(
            name: "ShoppingFeatureDomain",
            swiftSettings: [
                .defaultIsolation(nil)
            ]
        ),

        .target(
            name: "ShoppingFeatureData",
            dependencies: ["ShoppingFeatureDomain"],
            swiftSettings: [
                .defaultIsolation(nil)
            ]
        ),

        .target(
            name: "ShoppingFeatureMedia",
            swiftSettings: [
                .defaultIsolation(nil)
            ]
        )
    ]
)
```

Example implementation:

```swift
// ShoppingFeatureUI
final class CartViewModel {
    var state: CartState = .empty

    private let repository: CartRepository

    init(repository: CartRepository) {
        self.repository = repository
    }

    func refresh() async throws {
        state = .loading

        do {
            let cart = try await repository.currentCart()
            state = .loaded(cart)
        } catch {
            state = .failed(error)
        }
    }
}
```

```swift
// ShoppingFeatureDomain
nonisolated enum CartState: Equatable {
    case empty
    case loading
    case loaded(Cart)
    case failed(any Error)
}
```

```swift
// ShoppingFeatureMedia
nonisolated enum CartImageDecoder {
    @concurrent
    static func decode(_ data: Data) async throws -> CartImage {
        try Task.checkCancellation()
        return try decodeImage(data)
    }
}
```

### Decision

```text
Default MainActor isolation is appropriate for ShoppingFeatureUI.
It is harmful for the whole package.
It is unnecessary for pure value models and domain logic.
It should be avoided for reusable networking, decoding, repository, and indexing code.
```

### Why this is better

This design lets the compiler protect UI state without contaminating pure code. It also makes performance intent explicit: UI mutation stays serialized on the main actor; CPU-heavy decoding uses `@concurrent`; domain models remain portable across isolation domains.

---

## 7. Production guidance

Use default `MainActor` isolation when:

```text
- The target is an app target.
- The target is mostly SwiftUI/UIKit/AppKit UI code.
- Most mutable state is UI-facing.
- The module is not intended as a general-purpose library.
- You are migrating a UI-heavy codebase to Swift 6 strict concurrency.
```

Be careful when:

```text
- The target mixes UI, networking, persistence, parsing, and domain logic.
- Public APIs are consumed from many isolation domains.
- Tests need to construct and mutate models outside the main actor.
- You have expensive async functions that were assumed to run off-main.
- You are comparing behavior across targets with different Swift settings.
```

Avoid default `MainActor` isolation when:

```text
- The target is a reusable SDK.
- The target is domain/model-only.
- The target is server-side Swift.
- The target contains hot-path parsing, indexing, media processing, or crypto.
- The module’s API should be usable from any actor.
```

Debugging checklist:

```text
1. What is this target’s Default Actor Isolation setting?
2. Is this declaration explicitly isolated, inferred isolated, or nonisolated?
3. Is this async function actually leaving the actor?
4. Does this function need `@concurrent`, or should it remain serialized?
5. Is a pure type accidentally MainActor-isolated because of the target setting?
6. Are two targets compiling the same conceptual code under different settings?
7. Is a public API leaking MainActor isolation to clients?
8. Is a performance regression caused by CPU work now running on the main actor?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift 6.2 makes concurrency easier by using `MainActor` by default and adding `@concurrent`.

This is incomplete and dangerously imprecise.

### Senior answer

> Swift 6.2 lets a target opt into default `MainActor` isolation, which is useful for UI code. It also changes the model for `nonisolated async` under an upcoming feature so async functions can run in the caller’s context. If I need expensive work to leave the main actor, I use `@concurrent`.

Good.

### Staff-level answer

> I would treat Swift 6.2 approachable concurrency as a module-boundary and migration strategy, not just a compiler convenience. App/UI targets can use default `MainActor` isolation to reduce annotation noise and encode their actual UI isolation model. Reusable domain, networking, persistence, media, and SDK targets should usually stay nonisolated. I would audit every expensive async path because `nonisolated async` no longer necessarily means off-main under the new behavior. For intentional off-actor CPU work, I would use `@concurrent` and make the Sendable/isolation boundary explicit.

Staff-level questions to ask:

```text
Which targets are UI-only enough to default to MainActor?
Which declarations become public API promises if inferred MainActor?
Which async functions used to rely on generic-executor execution?
Which expensive paths now need @concurrent?
Which package boundaries should be split before enabling the setting?
How will tests verify main-actor state, cancellation, and off-main work?
```

---

## 9. Interview-ready summary

Swift 6.2’s approachable concurrency changes are about making data-race safety fit real app architecture. Default `MainActor` isolation lets a module infer `@MainActor` for unannotated code, which is useful for UI-heavy targets but harmful if applied to pure domain, networking, persistence, or reusable library code. The other major change is that `nonisolated async` can run in the caller’s execution context under the new behavior, so `async` does not mean background and `nonisolated` does not necessarily mean off-main. When off-actor execution is the real intent, especially for CPU-heavy work, use `@concurrent` and design the API boundary accordingly.

---

## 10. Flashcards

Q: Does Swift 6.2 make all code `@MainActor` by default?  
A: No. Default `MainActor` isolation is an opt-in module/target setting. If unspecified, the default remains `nonisolated`.

Q: What problem does default `MainActor` isolation solve?  
A: It reduces annotation noise for code that is already logically UI/main-thread-bound and helps migration to strict concurrency.

Q: What is the biggest risk of default `MainActor` isolation?  
A: Accidentally making pure, reusable, or performance-sensitive code UI-bound.

Q: Does `async` mean “runs on a background thread”?  
A: No. `async` means the function can suspend. It says nothing by itself about background execution.

Q: What changed about `nonisolated async` in Swift 6.2’s new behavior?  
A: It can run in the caller’s execution context instead of automatically switching off the caller’s actor.

Q: What does `@concurrent` express?  
A: The function intentionally switches off the caller’s actor and can run concurrently with other tasks on that actor.

Q: When should a UI package use default `MainActor` isolation?  
A: When the target is genuinely UI-facing and most state/mutations are main-actor-bound.

Q: When should a package stay `nonisolated`?  
A: For domain models, reusable SDKs, networking, persistence, parsing, media processing, server code, or APIs meant to be called from any actor.

Q: What is the migration danger with Swift 6.2?  
A: Different targets may have different isolation defaults, so the same-looking code can have different isolation semantics.

Q: What should a staff engineer do before enabling default `MainActor` isolation?  
A: Split targets by responsibility, audit public APIs, identify CPU-heavy async functions, and decide where `@concurrent` is required.

---

## 11. Related sections

- [[D5 — Global actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and assumeIsolatedUntitled]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D13 — Swift 6 strict concurrency migration]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- Swift Senior/Staff Rubric, D14 — Swift 6.2 approachable concurrency and new defaults.
- Swift.org — “Swift 6.2 Released,” Approachable Concurrency section. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))
- SE-0466 — Control default actor isolation inference. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md "swift-evolution/proposals/0466-control-default-actor-isolation.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0461 — Run nonisolated async functions on the caller’s actor by default. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md "swift-evolution/proposals/0461-async-function-isolation.md at main · swiftlang/swift-evolution · GitHub"))