---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - swift6
  - strict-concurrency
---
## 0. Rubric snapshot

**Rubric expectation**

Know language-mode migration strategy, warning-to-error progression, and pragmatic use of annotations like `@preconcurrency`. The rubric explicitly frames D13 as a migration-strategy topic, not just a concurrency-syntax topic.

**Caveats**

Silencing diagnostics without rethinking ownership/isolation is not a migration. `@preconcurrency`, `@unchecked Sendable`, `nonisolated(unsafe)`, and blanket `@MainActor` can keep a build moving, but they can also preserve the exact race-prone design Swift 6 is trying to expose.

**You should be able to answer**

- What did complete strict concurrency checking provide before Swift 6 language mode?
- When is `@preconcurrency` acceptable, and why is it rarely the final design?

**You should be able to do**

- Outline a migration plan for a medium-sized app moving from Swift 5.x concurrency warnings to Swift 6 strict checking.

---

## 1. Core mental model

Swift 6 strict concurrency migration is about turning implicit concurrency assumptions into compiler-checkable isolation contracts.

Before Swift 6, many iOS codebases relied on undocumented assumptions:

```text
"This callback probably comes on the main queue."
"This service is only called from one place."
"This class is mutable, but we don't mutate it concurrently."
"This global cache is safe because we are careful."
```

Swift 6 asks you to encode those assumptions:

```swift
@MainActor
actor
Sendable
@Sendable
nonisolated
Mutex
immutable value types
```

The goal is not “make the warnings disappear.” The goal is to make the compiler understand why the code is safe.

Swift 6 introduced an opt-in language mode that extends Swift’s safety guarantees to diagnose potential data races as compiler errors. Before Swift 6 language mode, Swift 5.10 already allowed data-race safety diagnostics as warnings via complete strict concurrency checking, which made it possible to stage the migration. ([Swift.org, "Announcing Swift 6"](https://swift.org/blog/announcing-swift-6/))

The key idea:

```text
Swift 5 complete checking = rehearse Swift 6 as warnings.
Swift 6 language mode = enforce those rules as errors.
```

Apple’s WWDC migration guidance is explicitly incremental: enable complete concurrency checking per target while still in Swift 5 mode, fix the warnings, then enable Swift 6 for that target, and repeat. Apple also warns against mixing this with large unrelated refactors. ([Apple Developer, "Migrate Your App to Swift 6"](https://developer.apple.com/videos/play/wwdc2024/10169/))

---

## 2. Essential mechanics

### 2.1 Swift compiler version is not the same as Swift language mode

You can use a Swift 6.x compiler while a target still compiles in Swift 5 language mode.

That means this can happen:

```text
Compiler: Swift 6.2
Language mode: Swift 5
Strict concurrency: Complete
Result: Swift 6-style concurrency issues appear mostly as warnings.
```

Then later:

```text
Compiler: Swift 6.2
Language mode: Swift 6
Result: many of those warnings become errors.
```

This distinction matters in multi-target apps. Your app target, feature packages, test support target, mock target, and internal libraries may not all migrate at once.

### 2.2 Complete checking before Swift 6 is a staging mode

Representative example:

```swift
import Foundation

final class ImageCache {
    var store: [String: Data] = [:]
}

func run(_ operation: @Sendable @escaping () async -> Void) {}

func migrate() {
    let cache = ImageCache()

    run {
        cache.store["avatar"] = Data()
    }
}
```

With Swift 5 language mode plus complete strict concurrency checking:

```text
warning: capture of 'cache' with non-Sendable type 'ImageCache' in a '@Sendable' closure; this is an error in the Swift 6 language mode [#SendableClosureCaptures]
```

With Swift 6 language mode:

```text
error: capture of 'cache' with non-Sendable type 'ImageCache' in a '@Sendable' closure [#SendableClosureCaptures]
```

Same underlying issue. Different enforcement level.

Why? `@Sendable` means the closure may cross concurrency domains. Capturing a mutable reference type that is not `Sendable` is unsafe because multiple tasks could access the same mutable object concurrently.

### 2.3 Strict concurrency checks isolation, not just async/await

Swift 6 migration is not only about code that says `async`.

It also surfaces:

```swift
var globalCache = ImageCache()
```

Potential issue:

```text
Mutable global shared state.
```

Possible fixes:

```swift
let globalConfiguration = AppConfiguration(...)
```

or:

```swift
@MainActor
var uiOnlyCache = ImageCache()
```

or:

```swift
actor ImageCache {
    private var store: [String: Data] = [:]

    func value(for key: String) -> Data? {
        store[key]
    }

    func insert(_ data: Data, for key: String) {
        store[key] = data
    }
}
```

Apple’s WWDC example calls out global variables as a common source of shared mutable state and shows fixes like immutability, global actor isolation, or unsafe opt-out only when an external mechanism genuinely protects the state. ([Apple Developer, "Migrate Your App to Swift 6"](https://developer.apple.com/videos/play/wwdc2024/10169/))

### 2.4 `Sendable` is a semantic promise

Good:

```swift
struct UserProfile: Sendable {
    let id: UUID
    let name: String
}
```

Usually bad:

```swift
final class Session: Sendable {
    var token: String = ""
}
```

A mutable class is not automatically safe just because it is `final`. A final immutable class can sometimes be safely `Sendable`; a mutable class usually needs actor isolation, a lock, or a redesign.

Better:

```swift
struct SessionSnapshot: Sendable {
    let token: String
}

actor SessionStore {
    private var token = ""

    func snapshot() -> SessionSnapshot {
        SessionSnapshot(token: token)
    }

    func updateToken(_ token: String) {
        self.token = token
    }
}
```

### 2.5 `@preconcurrency` is a temporary compatibility tool

`@preconcurrency import SomeOldSDK` tells the compiler to soften some concurrency diagnostics for types imported from a module that has not fully adopted concurrency annotations.

Example:

```swift
@preconcurrency import LegacyAnalyticsSDK
```

Acceptable use:

```text
A third-party SDK has not yet annotated its APIs with Sendable/global actor information, and you need to keep migrating your own code.
```

Unacceptable use:

```text
You own the code, the compiler is right, and you use @preconcurrency to avoid redesigning unsafe shared mutable state.
```

SE-0337 defines `@preconcurrency` as part of incremental migration. The proposal says the workflow is to enable checking, solve problems, temporarily use `@preconcurrency import` for missing dependency annotations, and later remove it once the dependency has proper annotations. It also states that Swift 6 mode checks all code completely for missing `Sendable` conformances and concurrency violations, generally as errors. ([GitHub, "Support Incremental Migration to Concurrency Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md))

---

## 3. Common traps and misconceptions

### Trap 1: Treating warnings as noise

Bad:

```swift
// "It is only a warning in Swift 5 mode."
run {
    cache.store["avatar"] = Data()
}
```

Better:

```swift
actor ImageCache {
    private var store: [String: Data] = [:]

    func insert(_ data: Data, for key: String) {
        store[key] = data
    }
}
```

Swift 5 complete-checking warnings often explicitly say they will become errors in Swift 6. Ignoring them is just delaying the migration cost.

### Trap 2: Blanket `@MainActor` on everything

Bad:

```swift
@MainActor
final class ImageProcessingService {
    func resize(_ image: Image) async -> Image {
        // CPU-heavy work
    }
}
```

Better:

```swift
@MainActor
final class AvatarViewModel {
    private let processor: ImageProcessingService

    func loadAvatar() async {
        let image = await processor.resize(...)
        self.avatar = image
    }
}

struct ImageProcessingService: Sendable {
    func resize(_ image: Image) async -> Image {
        // non-UI work
    }
}
```

`@MainActor` is correct for UI state. It is not a trash bin for concurrency diagnostics.

### Trap 3: Using `@unchecked Sendable` as a silencer

Bad:

```swift
final class TokenStore: @unchecked Sendable {
    var token: String = ""
}
```

Better:

```swift
actor TokenStore {
    private var token: String = ""

    func getToken() -> String {
        token
    }

    func setToken(_ token: String) {
        self.token = token
    }
}
```

Use `@unchecked Sendable` only when you can prove thread safety that the compiler cannot see, usually because all mutable state is protected by a lock or another synchronization primitive.

### Trap 4: Migrating everything in one huge PR

Bad migration plan:

```text
1. Turn on Swift 6 for the whole workspace.
2. Add @MainActor, @unchecked Sendable, and @preconcurrency until it builds.
3. Ship.
```

Better migration plan:

```text
1. Pick one target.
2. Enable complete checking while staying in Swift 5 mode.
3. Fix warnings by category.
4. Enable Swift 6 for that target.
5. Add a CI gate.
6. Repeat.
7. Revisit temporary opt-outs.
```

Apple’s migration guidance recommends migrating target by target, resolving warnings first, then enabling Swift 6 mode, and avoiding combining migration with large unrelated refactoring. ([Apple Developer, "Migrate Your App to Swift 6"](https://developer.apple.com/videos/play/wwdc2024/10169/))

---

## 4. Direct answers to rubric questions

### Q1. What did complete strict concurrency checking provide before Swift 6 language mode?

Complete strict concurrency checking in Swift 5.x provided a preview of Swift 6’s data-race-safety enforcement. It checked a whole module for many actor-isolation and `Sendable` problems that would become hard errors in Swift 6 language mode, but it generally reported them as warnings so teams could migrate incrementally.

Interview version:

> Complete checking before Swift 6 was basically a staging tool. It let a Swift 5 target ask the compiler, “Show me the concurrency problems Swift 6 would reject,” without immediately breaking the build. That made it possible to fix warnings target by target, then switch each target to Swift 6 once it was clean.

Swift.org states that Swift 6 diagnoses potential data races as compiler errors, while those checks were previously available as warnings through complete strict concurrency checking in Swift 5.10. ([Swift.org, "Announcing Swift 6"](https://swift.org/blog/announcing-swift-6/))

### Q2. When is `@preconcurrency` acceptable, and why is it rarely the final design?

`@preconcurrency` is acceptable when you are interacting with code that has not fully adopted Swift concurrency annotations yet, especially third-party SDKs, older Apple SDK surfaces, C/Objective-C imports, or public libraries that need staged source/ABI compatibility.

It is rarely final because it weakens the compiler’s ability to prove safety. It says, effectively:

```text
"Treat this dependency as pre-concurrency-aware for now."
```

That is useful during migration, but not a proof that crossing isolation boundaries is safe.

Interview version:

> I would use `@preconcurrency` as a temporary compatibility boundary, mainly for dependencies I do not control or for public APIs that need staged adoption. I would not use it to hide problems in code I own. The final design should usually make isolation explicit with actors, global actors, value types, `Sendable`, or a narrow synchronized wrapper.

SE-0337 describes `@preconcurrency import` as a temporary escape hatch for dependencies without complete concurrency annotations, and says the compiler should warn when the attribute becomes unnecessary so it can be removed. ([GitHub, "Support Incremental Migration to Concurrency Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md))

---

## 5. Code probe

D13 has no rubric code probe. Use this representative migration diagnostic instead.

Given:

```swift
import Foundation

final class ImageCache {
    var store: [String: Data] = [:]
}

func run(_ operation: @Sendable @escaping () async -> Void) {}

func migrate() {
    let cache = ImageCache()

    run {
        cache.store["avatar"] = Data()
    }
}
```

### What happens?

Swift 5 mode with complete strict concurrency checking:

```text
/tmp/D13Probe2.swift:13:9: warning: capture of 'cache' with non-Sendable type 'ImageCache' in a '@Sendable' closure; this is an error in the Swift 6 language mode [#SendableClosureCaptures]
 1 | import Foundation
 2 | 
 3 | final class ImageCache {
   |             `- note: class 'ImageCache' does not conform to the 'Sendable' protocol
 4 |     var store: [String: Data] = [:]
 5 | }
   :
11 | 
12 |     run {
13 |         cache.store["avatar"] = Data()
   |         `- warning: capture of 'cache' with non-Sendable type 'ImageCache' in a '@Sendable' closure; this is an error in the Swift 6 language mode [#SendableClosureCaptures]
14 |     }
15 | }
```

Swift 6 language mode:

```text
/tmp/D13Probe2.swift:13:9: error: capture of 'cache' with non-Sendable type 'ImageCache' in a '@Sendable' closure [#SendableClosureCaptures]
 1 | import Foundation
 2 | 
 3 | final class ImageCache {
   |             `- note: class 'ImageCache' does not conform to the 'Sendable' protocol
 4 |     var store: [String: Data] = [:]
 5 | }
   :
11 | 
12 |     run {
13 |         cache.store["avatar"] = Data()
   |         `- error: capture of 'cache' with non-Sendable type 'ImageCache' in a '@Sendable' closure [#SendableClosureCaptures]
14 |     }
15 | }
```

### Why?

`ImageCache` is a mutable reference type:

```swift
final class ImageCache {
    var store: [String: Data] = [:]
}
```

The closure passed to `run` is `@Sendable`:

```swift
func run(_ operation: @Sendable @escaping () async -> Void) {}
```

A `@Sendable` closure can execute in a different concurrency domain. Capturing a mutable non-`Sendable` class instance would allow shared mutable state to cross that boundary.

```text
migrate()
 └─ cache: ImageCache
      └─ mutable dictionary storage

@Sendable closure
 └─ captures cache
      └─ may execute concurrently elsewhere

Problem:
shared mutable reference crosses a concurrency boundary without isolation.
```

### Fix or redesign

Actor-based fix:

```swift
import Foundation

actor ImageCache {
    private var store: [String: Data] = [:]

    func insert(_ data: Data, for key: String) {
        store[key] = data
    }

    func data(for key: String) -> Data? {
        store[key]
    }
}

func run(_ operation: @Sendable @escaping () async -> Void) {}

func migrate() {
    let cache = ImageCache()

    run {
        await cache.insert(Data(), for: "avatar")
    }
}
```

### Why this fix is correct

The mutable dictionary is now actor-isolated. The closure can safely capture the actor because actors are designed to protect their isolated mutable state. Access to `store` must go through `await`, making the concurrency boundary explicit.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Use an `actor`|Shared mutable async state, cache, session store, token refresh, database facade|Requires `await`; actor reentrancy still needs design care|
|Use immutable value snapshots|Data crosses actor/task boundaries but does not need mutation|May require copying or redesigning state flow|
|Use `Mutex`/lock|Tiny synchronous critical section, hot path, low-level primitive|You must maintain locking invariants manually|
|`@unchecked Sendable`|Internally synchronized type where compiler cannot verify safety|Proof obligation stays with you; easy to misuse|
|`@MainActor`|UI state or state truly confined to main actor|Wrong for heavy work or domain services|

---

## 6. Exercise

### Problem

Outline a migration plan for a medium-sized app moving from Swift 5.x concurrency warnings to Swift 6 strict checking.

Assume:

```text
- iOS app with app target, networking package, persistence package, UI feature modules, test support.
- Some GCD, callbacks, delegates, Combine, mutable services, and third-party SDKs.
- Target environment: iOS 18+, Swift 6.2+, Xcode 26.
```

### Bad / naive version

```text
1. Flip the whole workspace to Swift 6.
2. Add @MainActor to most classes.
3. Add @unchecked Sendable to classes that complain.
4. Add @preconcurrency import everywhere.
5. Fix anything that still blocks compilation.
```

### What is wrong?

```text
- It confuses build success with correctness.
- It pushes non-UI work onto the main actor.
- It hides unsafe mutable reference sharing.
- It creates permanent compatibility debt.
- It makes review impossible because every file changes at once.
- It does not leave the codebase with clearer ownership/isolation boundaries.
```

### Improved version

#### Phase 0 — Baseline and inventory

Create a migration tracking document:

```text
Targets:
- App
- DesignSystem
- FeatureSearch
- FeatureProfile
- Networking
- Persistence
- TestSupport

For each target:
- Swift language mode
- Strict concurrency level
- Number of warnings
- Third-party dependencies
- Known unsafe opt-outs
- Owner
```

Also inventory risky patterns:

```text
- Mutable globals/statics
- Shared mutable classes
- Singleton services
- Callback/delegate APIs with unclear queue guarantees
- Task.detached usage
- Stored closures
- Combine pipelines crossing queues
- Non-Sendable models crossing actors/tasks
- Objective-C/C dependencies
```

#### Phase 1 — Keep Swift 5 mode, enable stricter diagnostics

For one target at a time:

```text
Swift language mode: Swift 5
Strict concurrency checking: Complete
```

Do not start with the whole app unless the codebase is small.

For Xcode projects, this is a per-target build setting. For packages, use package-level/target-level Swift settings where appropriate.

Apple’s WWDC migration example uses complete checking first while staying in Swift 5 mode, resolves warnings, then enables Swift 6 mode for that target. ([Apple Developer, "Migrate Your App to Swift 6"](https://developer.apple.com/videos/play/wwdc2024/10169/))

#### Phase 2 — Fix warnings by category, not randomly

Recommended order:

```text
1. Immutable globals/statics
2. UI isolation
3. Sendable value models
4. Shared mutable services
5. Callback/delegate boundaries
6. Unstructured tasks
7. Dependency boundaries
8. Tests and mocks
```

Example: global mutable state.

Before:

```swift
var analytics = AnalyticsClient()
```

After, if immutable and `Sendable`:

```swift
let analytics = AnalyticsClient(...)
```

After, if UI-only:

```swift
@MainActor
var analytics = AnalyticsClient()
```

After, if mutable and shared:

```swift
actor AnalyticsClient {
    private var pendingEvents: [Event] = []

    func track(_ event: Event) {
        pendingEvents.append(event)
    }
}
```

#### Phase 3 — Make UI isolation explicit

For SwiftUI/UIKit app code:

```swift
@MainActor
final class ProfileViewModel {
    private let service: ProfileService

    var state: State = .idle

    func load() async {
        state = .loading

        do {
            let profile = try await service.fetchProfile()
            state = .loaded(profile)
        } catch {
            state = .failed(error)
        }
    }
}
```

Do not make the service `@MainActor` unless it genuinely owns UI state.

```swift
struct ProfileService: Sendable {
    func fetchProfile() async throws -> Profile {
        // network work
    }
}
```

#### Phase 4 — Convert unsafe shared services

Before:

```swift
final class TokenStore {
    var token: String?
}
```

Better:

```swift
actor TokenStore {
    private var token: String?

    func currentToken() -> String? {
        token
    }

    func updateToken(_ token: String?) {
        self.token = token
    }
}
```

Alternative for tiny synchronous state:

```swift
import Synchronization

final class TokenStore: @unchecked Sendable {
    private let token = Mutex<String?>(nil)

    func currentToken() -> String? {
        token.withLock { $0 }
    }

    func updateToken(_ newValue: String?) {
        token.withLock { $0 = newValue }
    }
}
```

This `@unchecked Sendable` is defensible because all mutable state is protected by the mutex. It still needs code review.

#### Phase 5 — Handle dependencies deliberately

Use:

```swift
@preconcurrency import LegacySDK
```

only with a tracking note:

```swift
// TODO(Swift6Migration): Remove once LegacySDK ships Sendable/global actor annotations.
// Owner: Platform
// Review date: 2026-08-01
@preconcurrency import LegacySDK
```

Do not scatter it without ownership.

SE-0337’s intended workflow is temporary: use `@preconcurrency import` for missing annotations, then remove it once updated dependencies make the import unnecessary. ([GitHub, "Support Incremental Migration to Concurrency Checking"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md))

#### Phase 6 — Enable Swift 6 per target

Once a target has zero complete-checking warnings:

```text
Swift language mode: Swift 6
CI: warnings as errors for that target
```

Then prevent regressions:

```text
- New code must compile cleanly under Swift 6.
- New models crossing boundaries must be Sendable.
- New UI state must have explicit actor isolation.
- New unsafe opt-outs require review.
```

#### Phase 7 — Revisit temporary opt-outs

Search for:

```text
@preconcurrency
@unchecked Sendable
nonisolated(unsafe)
Task.detached
MainActor.run
DispatchQueue
```

Classify each as:

```text
- Remove now
- Replace with actor/value model
- Keep with documented invariant
- Blocked by dependency
```

---

## 7. Production guidance

Use strict concurrency migration in production when:

```text
- You want to reduce hard-to-reproduce data races.
- You are adopting async/await more broadly.
- You have shared mutable services, caches, session state, or delegate callbacks.
- You maintain packages used by other Swift 6 adopters.
- You want new code to have compiler-enforced isolation guarantees.
```

Be careful when:

```text
- A target has many old Objective-C/C dependencies.
- Third-party SDKs lack Sendable or global actor annotations.
- Your architecture relies heavily on mutable singletons.
- You have large test/mocking layers with non-Sendable classes.
- You are tempted to combine migration with big architecture rewrites.
```

Avoid:

```text
- Blanket @MainActor.
- Blanket @unchecked Sendable.
- Permanent @preconcurrency imports.
- Treating nonisolated(unsafe) as normal application code.
- Flipping every target at once in a medium/large app.
```

Debugging checklist:

```text
1. What value crosses the actor/task boundary?
2. Is it Sendable?
3. If it is a class, is it immutable, synchronized, or actor-isolated?
4. Is this UI state? If yes, should it be @MainActor?
5. Is this domain/background state? If yes, why would MainActor be correct?
6. Is this warning from code I own or a dependency?
7. Is @preconcurrency hiding a dependency gap or my own design problem?
8. Is this fix preserving behavior, or just silencing the compiler?
9. Can the invariant be tested?
10. Should this become an actor, value type, mutex-protected type, or API boundary?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift 6 makes concurrency warnings into errors, so you fix `Sendable` and actor warnings before switching.

### Senior answer

> I would migrate incrementally. First I would enable complete strict concurrency checking in Swift 5 mode for one target, fix warnings by category, then switch that target to Swift 6. I would prefer actors or value types for shared mutable state, `@MainActor` for UI state, and use `@preconcurrency` only for dependencies that are not annotated yet.

### Staff-level answer

> I would treat Swift 6 migration as an architecture and ownership audit, not a compiler-cleanup task. The plan is per-target, CI-gated, and category-driven: globals, UI isolation, Sendable models, service isolation, callback boundaries, dependencies, tests. I would track every unsafe opt-out with an owner and removal plan. The goal is to leave the codebase with clearer isolation boundaries, not just fewer diagnostics.

Staff-level questions to ask:

```text
Which targets should migrate first, and why?
Which shared services should become actors, and which should become immutable values?
Which classes are incorrectly crossing concurrency domains?
Which dependencies require @preconcurrency, and who owns removing it?
Which warnings reveal architectural issues rather than local syntax issues?
How do we prevent new Swift 5-style concurrency debt after one target migrates?
How do test doubles and mocks need to change under Sendable checking?
```

---

## 9. Interview-ready summary

Swift 6 strict concurrency migration is the process of moving from warning-based concurrency checking to enforced data-race safety. In Swift 5 mode, complete strict concurrency checking lets you preview many Swift 6 failures as warnings. The practical migration strategy is target-by-target: enable complete checking, fix warnings by making isolation and sendability explicit, then enable Swift 6 language mode and gate regressions in CI. `@preconcurrency` is acceptable as a temporary bridge for dependencies that lack concurrency annotations, but it is rarely a final design because it weakens the compiler’s ability to prove safety. Strong migration work improves the architecture: UI state becomes `@MainActor`, shared mutable state becomes actor-isolated or synchronized, and cross-boundary data becomes `Sendable`.

---

## 10. Flashcards

Q: What is the difference between Swift compiler version and Swift language mode?  
A: The compiler version is the toolchain you use; the language mode controls which source rules are enforced for a target. A Swift 6 compiler can still compile a target in Swift 5 mode.

Q: What did complete strict concurrency checking provide before Swift 6 language mode?  
A: It surfaced many Swift 6 concurrency violations as warnings while staying in Swift 5 mode.

Q: Why is Swift 5 complete checking useful?  
A: It lets teams fix data-race-safety issues incrementally before turning them into Swift 6 errors.

Q: What does `@preconcurrency import` do?  
A: It weakens some concurrency diagnostics for imported declarations from modules that have not fully adopted concurrency annotations.

Q: When is `@preconcurrency` acceptable?  
A: For dependencies or compatibility boundaries you do not fully control, especially during staged migration.

Q: Why is `@preconcurrency` rarely final?  
A: Because it suppresses or downgrades diagnostics instead of proving the code is safe.

Q: What is the wrong use of `@MainActor` during migration?  
A: Marking non-UI services or heavy background work as `@MainActor` just to silence diagnostics.

Q: What is the danger of `@unchecked Sendable`?  
A: It shifts proof of thread safety from the compiler to the developer; if the proof is wrong, the code can still race.

Q: What should happen before enabling Swift 6 for a target?  
A: The target should compile cleanly under complete strict concurrency checking in Swift 5 mode.

Q: What is the staff-level migration goal?  
A: Build clear ownership and isolation boundaries so the compiler can enforce safety for future changes.

---

## 11. Related sections

- [[D4 — Actors and actor isolation]]
- [[D5 — Global actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and assumeIsolatedUntitled]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D12 — Synchronization library, mutexes, and choosing between actors and locks]]
- [[D14 — Swift 6.2 approachable concurrency and new defaults]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — D13 Swift 6 strict concurrency migration
- Swift.org. "Announcing Swift 6." Swift.org Blog. https://swift.org/blog/announcing-swift-6/
- Apple Developer. "Migrate Your App to Swift 6." WWDC24. https://developer.apple.com/videos/play/wwdc2024/10169/
- GitHub. "Support Incremental Migration to Concurrency Checking." Swift Evolution SE-0337. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md
