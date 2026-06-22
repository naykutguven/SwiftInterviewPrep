---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - main-actor
  - global-actors
---
## 0. Rubric snapshot

**Rubric expectation**

Understand when `@MainActor` belongs on whole types versus individual members, and how UI isolation should be expressed. The rubric caveat is that `@MainActor` is an isolation contract, not a generic substitute for synchronization.

**You should be able to answer**

- When should a whole type be `@MainActor` instead of individual methods?
- Why is marking too much code `@MainActor` a design smell?

**You should be able to do**

- Refactor a view model so UI mutation is main-actor isolated while background work remains off the main actor.

---

## 1. Core mental model

A **global actor** is a single, globally shared actor isolation domain. Instead of protecting state inside one actor instance, it lets declarations across different types, files, or modules agree: “this code must run through the same serialized executor.”

`@MainActor` is Swift’s built-in global actor for main-thread/UI-affine work. SE-0316 describes global actors as extending actor isolation beyond one actor instance, especially for state and operations that must run on the UI/main thread. It also says a declaration isolated to a global actor can be synchronously accessed only from the same global actor, and must be reached asynchronously from elsewhere. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md "swift-evolution/proposals/0316-global-actors.md at main · swiftlang/swift-evolution · GitHub"))

The key idea:

```text
@MainActor = compile-time UI isolation boundary, not “sprinkle DispatchQueue.main everywhere”
```

`@MainActor` protects UI state from data races by making mutation of that state happen in one isolation domain. It does **not** make a feature logically correct, cancel stale tasks, prevent actor reentrancy bugs, make background work cheap, or turn every dependency into a good design.

For iOS, the practical rule is:

```text
UI-facing mutable state → MainActor
networking/parsing/database/CPU-heavy work → not MainActor unless intentionally UI-affine
```

---

## 2. Essential mechanics

### 2.1 `@MainActor` can isolate properties, methods, closures, protocols, or whole types

```swift
@MainActor
final class ProfileViewModel {
    private(set) var title = ""

    func updateTitle(_ title: String) {
        self.title = title
    }
}
```

Here, all instance members are main-actor isolated. Calling `updateTitle` from outside the main actor is a cross-actor call.

```swift
func render(_ model: ProfileViewModel) async {
    await model.updateTitle("Aykut")
}
```

A synchronous nonisolated call is illegal:

```swift
@MainActor
final class ViewModel {
    var title = ""
    func update() { title = "Done" }
}

func render(_ model: ViewModel) {
    model.update()
}
```

Compiled with Swift 6 mode:

```text
error: call to main actor-isolated instance method 'update()' in a synchronous nonisolated context [#ActorIsolatedCall]
note: calls to instance method 'update()' from outside of its actor context are implicitly asynchronous
note: add '@MainActor' to make global function 'render' part of global actor 'MainActor'
```

This is the compiler enforcing the isolation boundary. Swift’s diagnostics describe the same rule for global actors like `@MainActor`: calling a main-actor-isolated method from a synchronous nonisolated context is invalid. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/?utm_source=chatgpt.com "Calling an actor-isolated method from a synchronous ..."))

---

### 2.2 Whole-type `@MainActor` is appropriate when the type’s identity is UI-affine

Good candidates:

```swift
@MainActor
@Observable
final class SearchViewModel {
    var query = ""
    var results: [SearchResult] = []
    var isLoading = false
}
```

A whole type should usually be `@MainActor` when:

```text
- It is owned by UI.
- Its mutable state directly drives UI rendering.
- Most methods read or mutate UI-visible state.
- Calling it off the main actor would usually be a bug.
```

This is common for:

```text
- SwiftUI observable view models
- UIKit/AppKit view controllers
- UI coordinators
- presentation state containers
```

SE-0316 explicitly notes that when an entire type predominantly requires main-thread execution, the type itself can be annotated with a global actor, and members can opt out with `nonisolated` when needed. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md "swift-evolution/proposals/0316-global-actors.md at main · swiftlang/swift-evolution · GitHub"))

---

### 2.3 Member-level `@MainActor` is better for mixed-responsibility types

```swift
final class ImagePipeline {
    func decode(_ data: Data) -> UIImage {
        // CPU-heavy, should not be main-actor isolated.
    }

    @MainActor
    func display(_ image: UIImage, in imageView: UIImageView) {
        imageView.image = image
    }
}
```

Only the UI-touching member is isolated. The service itself remains usable from non-UI contexts.

This is usually better for:

```text
- services
- repositories
- caches
- network clients
- parsers
- SDK abstractions
- domain logic
```

Marking the whole service `@MainActor` would accidentally serialize all work through the UI executor.

---

### 2.4 `MainActor.run` is a boundary tool, not an architecture

Sometimes you are in non-main async code and need to publish one UI update:

```swift
let value = try await service.loadValue()

await MainActor.run {
    viewModel.state = .loaded(value)
}
```

This is fine for narrow bridging. But if every function is full of `MainActor.run`, the design probably has unclear ownership of UI state. Prefer putting UI-owned mutable state inside a `@MainActor` type.

Bad smell:

```swift
func load() async {
    await MainActor.run { isLoading = true }

    let data = try? await service.fetch()

    await MainActor.run { result = data }
    await MainActor.run { isLoading = false }
}
```

Better:

```swift
@MainActor
final class ViewModel {
    private let service: Service
    private(set) var state: State = .idle

    func load() {
        state = .loading

        Task {
            do {
                let data = try await service.fetch()
                state = .loaded(data)
            } catch {
                state = .failed(error)
            }
        }
    }
}
```

---

### 2.5 Swift 6.2 caveat: default main-actor isolation changes the baseline

Swift 6.2 introduced an option to isolate code to the main actor by default, intended for scripts, UI code, and executable targets. It also introduced `@concurrent` to explicitly request concurrent execution where needed. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))

For iOS app targets in Xcode 26, this matters because a UI-heavy module may be main-actor isolated by default. That reduces annotation noise, but it can also hide architectural mistakes.

Good split:

```text
App/UI target:
- default isolation: MainActor can be reasonable

Core/domain/networking/storage package:
- default isolation: nonisolated is usually healthier
```

Do not let default `MainActor` isolation leak into reusable packages unless the package is explicitly UI-affine.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `@MainActor` means “this starts a background task safely”

Bad:

```swift
@MainActor
final class FeedViewModel {
    var items: [Item] = []

    func load() async {
        let data = try! Data(contentsOf: url) // blocking work on MainActor
        items = decode(data)
    }
}
```

This is main-actor isolated, but it can still freeze the UI.

Better:

```swift
@MainActor
final class FeedViewModel {
    private let service: FeedService
    private(set) var items: [Item] = []

    func load() {
        Task {
            do {
                let items = try await service.loadFeed()
                self.items = items
            } catch {
                // publish error state
            }
        }
    }
}

struct FeedService: Sendable {
    func loadFeed() async throws -> [Item] {
        let (data, _) = try await URLSession.shared.data(from: feedURL)
        return try await decode(data)
    }

    @concurrent
    private func decode(_ data: Data) async throws -> [Item] {
        // CPU-heavy parsing/decoding can stay off MainActor.
    }
}
```

`@MainActor` protects UI state. It does not make blocking work acceptable.

---

### Trap 2: Marking services `@MainActor` because the caller is UI

Bad:

```swift
@MainActor
final class APIClient {
    func fetchProfile() async throws -> Profile {
        // Network/client code should not be UI-isolated by default.
    }
}
```

Better:

```swift
struct APIClient: Sendable {
    func fetchProfile() async throws -> Profile {
        // Non-UI dependency.
    }
}

@MainActor
final class ProfileViewModel {
    private let apiClient: APIClient
    private(set) var state: State = .idle
}
```

Keep UI isolation at the UI boundary. Do not infect lower layers.

---

### Trap 3: Using `Task.detached` to “escape” `@MainActor`

Bad:

```swift
@MainActor
final class ViewModel {
    var title = ""

    func load() {
        Task.detached {
            self.title = "Done"
        }
    }
}
```

This is wrong because `Task.detached` discards actor context. It tries to access main-actor state from a detached task.

Better:

```swift
@MainActor
final class ViewModel {
    var title = ""

    func load() {
        Task {
            title = "Done"
        }
    }
}
```

`Task {}` created inside a main-actor-isolated context inherits that actor context. `Task.detached {}` does not.

---

### Trap 4: Believing `@MainActor` solves stale-result bugs

```swift
@MainActor
final class SearchViewModel {
    var results: [Result] = []

    func search(_ query: String) {
        Task {
            results = try await service.search(query)
        }
    }
}
```

This is data-race safe, but logically wrong if the user types quickly:

```text
search("s")
search("sw")
search("swift")
```

The `"s"` request might finish last and overwrite `"swift"` results. `@MainActor` serialized the mutation, but it did not decide which mutation is still valid.

Better designs use cancellation, task IDs, or stale-result suppression.

---

## 4. Direct answers to rubric questions

### Q1. When should a whole type be `@MainActor` instead of individual methods?

A whole type should be `@MainActor` when the type’s purpose is UI-facing and most of its mutable state is only valid on the main actor.

Good examples:

```text
- SwiftUI observable view model
- UIKit view controller
- UI coordinator
- presentation state model
- object whose properties directly drive rendered UI
```

Use member-level `@MainActor` when only part of the type touches UI.

```swift
final class AnalyticsService {
    func track(_ event: Event) {
        // non-UI
    }

    @MainActor
    func showDebugOverlay() {
        // UI-only
    }
}
```

Interview version:

> I mark a whole type `@MainActor` when the type is fundamentally UI-affine: its mutable state drives rendering, and most operations should be illegal off the main actor. For mixed types, I isolate only the UI-touching members. I do not mark services, parsers, repositories, or caches `@MainActor` just because a view model calls them.

---

### Q2. Why is marking too much code `@MainActor` a design smell?

Because it collapses unrelated responsibilities into the UI isolation domain.

Consequences:

```text
- CPU-heavy work can accidentally run on the main actor.
- Lower layers become coupled to UI execution.
- Reusable modules become harder to use from background contexts.
- Concurrency bugs may be hidden instead of modeled correctly.
- Tests and packages inherit unnecessary actor constraints.
- You serialize work that could safely happen elsewhere.
```

`@MainActor` is correct for UI state. It is suspicious when applied to networking, decoding, persistence, analytics, SDK clients, or business logic without a strong reason.

Interview version:

> Overusing `@MainActor` is a smell because it uses UI isolation as a blanket synchronization mechanism. It may silence compiler diagnostics, but it also couples non-UI code to the main actor, creates accidental serialization, and can move expensive work onto the UI executor. A better design isolates UI mutation on the main actor and keeps background-capable dependencies nonisolated, actor-isolated elsewhere, or explicitly concurrent.

---

## 5. Code probe

The rubric has no dedicated D5 code probe, so use this minimal compiler probe.

Given:

```swift
@MainActor
final class ViewModel {
    var title = ""
    func update() { title = "Done" }
}

func render(_ model: ViewModel) {
    model.update()
}
```

### What happens?

Compiled with Swift 6 mode:

```text
/tmp/d5_call.swift:8:11: error: call to main actor-isolated instance method 'update()' in a synchronous nonisolated context [#ActorIsolatedCall]
 2 | final class ViewModel {
 3 |     var title = ""
 4 |     func update() { title = "Done" }
   |          `- note: calls to instance method 'update()' from outside of its actor context are implicitly asynchronous
 5 | }
 6 | 
 7 | func render(_ model: ViewModel) {
   |      `- note: add '@MainActor' to make global function 'render' part of global actor 'MainActor'
 8 |     model.update()
   |           `- error: call to main actor-isolated instance method 'update()' in a synchronous nonisolated context [#ActorIsolatedCall]
```

### Why?

`update()` is isolated to the main actor because the whole class is `@MainActor`. `render` is synchronous and nonisolated. A synchronous nonisolated function cannot directly call a main-actor-isolated method because that would cross an actor boundary without a suspension point.

Actor-isolated declarations can be accessed directly only from the same isolation domain. Cross-actor calls are asynchronous because they may need to enqueue work on that actor’s executor. SE-0306 describes cross-actor asynchronous calls as messages placed in the actor’s mailbox, with actor-isolated code processed one-at-a-time. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

### Fix 1: Make the caller async and cross the boundary explicitly

```swift
func render(_ model: ViewModel) async {
    await model.update()
}
```

### Fix 2: Make the caller main-actor isolated too

```swift
@MainActor
func render(_ model: ViewModel) {
    model.update()
}
```

### Fix 3: Move only UI mutation into `@MainActor`

```swift
final class Renderer {
    func prepareTitle() -> String {
        "Done"
    }

    @MainActor
    func render(_ model: ViewModel) {
        model.update()
    }
}
```

### Alternative fixes and tradeoffs

|Option|When appropriate|Tradeoff|
|---|---|---|
|`await model.update()`|Non-UI async caller needs to publish UI state|Caller becomes async-aware|
|Mark caller `@MainActor`|Caller is UI code too|Can spread main-actor isolation upward|
|Mark whole type `@MainActor`|Type is UI-owned|Non-UI members become isolated unless opted out|
|Mark individual members `@MainActor`|Mixed UI/non-UI type|More annotations, but cleaner boundaries|
|`MainActor.run`|One-off UI mutation from async code|Repeated use often signals poor ownership|

---

## 6. Exercise

### Problem

Refactor a view model so UI mutation is main-actor isolated while background work remains off the main actor.

### Bad / naive version

```swift
import Foundation

final class SearchViewModel {
    var isLoading = false
    var results: [Int] = []
    var errorMessage: String?

    func search(_ query: String) {
        Task.detached {
            self.isLoading = true

            let ids = try await self.fetchIDs(matching: query)
            let ranked = self.rank(ids)

            self.results = ranked
            self.isLoading = false
        }
    }

    private func fetchIDs(matching query: String) async throws -> [Int] {
        try await Task.sleep(for: .milliseconds(100))
        return Array(0..<100_000)
    }

    private func rank(_ ids: [Int]) -> [Int] {
        ids.sorted(by: >)
    }
}
```

### What is wrong?

```text
1. The view model owns UI-visible mutable state but is not main-actor isolated.
2. Task.detached loses actor/lifecycle context.
3. The detached task captures self and mutates shared reference state.
4. CPU-heavy ranking is mixed into the view model.
5. There is no cancellation or stale-result handling.
6. Multiple booleans/optionals allow invalid UI states.
```

In Swift 6 mode, a simpler version of this pattern produces a data-race diagnostic:

```swift
final class BadSearchViewModel {
    var title = ""

    func load() {
        Task.detached {
            self.title = "Done"
        }
    }
}
```

Compiler output:

```text
/tmp/d5_bad.swift:7:14: error: passing closure as a 'sending' parameter risks causing data races between code in the current task and concurrent execution of the closure
 5 | 
 6 |     func load() {
 7 |         Task.detached {
   |              `- error: passing closure as a 'sending' parameter risks causing data races between code in the current task and concurrent execution of the closure
 8 |             self.title = "Done"
   |                        `- note: closure captures 'self' which is accessible to code in the current task
 9 |         }
10 |     }
```

### Improved version

```swift
import Foundation

enum ViewState: Equatable {
    case idle
    case loading
    case loaded([Int])
    case failed(String)
}

struct SearchService: Sendable {
    func fetchIDs(matching query: String) async throws -> [Int] {
        try await Task.sleep(for: .milliseconds(100))
        return Array(0..<100_000)
    }

    @concurrent
    func rank(_ ids: [Int]) async -> [Int] {
        ids.sorted(by: >)
    }
}

@MainActor
final class SearchViewModel {
    private(set) var state: ViewState = .idle

    private let service: SearchService
    private var currentTask: Task<Void, Never>?

    init(service: SearchService) {
        self.service = service
    }

    func search(_ query: String) {
        currentTask?.cancel()
        state = .loading

        currentTask = Task { [service] in
            do {
                let ids = try await service.fetchIDs(matching: query)
                let ranked = await service.rank(ids)

                try Task.checkCancellation()

                state = .loaded(ranked)
            } catch is CancellationError {
                // Keep the latest visible state.
            } catch {
                state = .failed(String(describing: error))
            }
        }
    }

    func cancel() {
        currentTask?.cancel()
        currentTask = nil
        state = .idle
    }
}
```

### Why this is better

```text
- UI state is isolated by the whole @MainActor view model.
- Background-capable work is moved into SearchService.
- CPU-heavy rank work is explicitly marked @concurrent in Swift 6.2.
- Task cancellation prevents older searches from publishing stale results.
- ViewState removes invalid combinations like isLoading == true and errorMessage != nil.
- The view model owns presentation state, not networking/parsing details.
```

Important nuance: `Task {}` inside a `@MainActor` method inherits the main actor for its body, so state mutation is safe. The expensive work must be placed behind APIs that do not perform CPU-heavy work on the main actor. In Swift 6.2, `@concurrent` is the explicit tool for work that should run on the concurrent executor rather than remaining serialized on the current actor. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))

---

## 7. Production guidance

Use `@MainActor` in production when:

```text
- A type owns UI-visible mutable state.
- A method/property directly touches UIKit/AppKit/SwiftUI state.
- A closure must be executed in a UI context.
- A protocol requirement is UI-affine by design.
- You want the compiler to enforce main-thread UI access.
```

Be careful when:

```text
- A type mixes UI and non-UI responsibilities.
- A method performs parsing, image decoding, sorting, compression, crypto, database work, or file I/O.
- A package target may be reused outside UI contexts.
- You are using Swift 6.2 default MainActor isolation and forget which target has which default.
- You are tempted to add @MainActor only to silence Sendable or isolation diagnostics.
```

Avoid when:

```text
- The type is a repository, API client, cache, parser, formatter, SDK model, or pure domain service.
- You need independent serialization unrelated to UI. Use an actor, Mutex, lock, or immutable model instead.
- You need parallel CPU work. Use non-main isolated code, task groups, or @concurrent where appropriate.
```

Debugging checklist:

```text
1. Which mutable state is UI-visible?
2. Which actor owns that state?
3. Is expensive work accidentally executing on MainActor?
4. Is Task.detached losing isolation or lifecycle?
5. Are stale tasks able to publish old results?
6. Is @MainActor applied because it is semantically correct, or because it silences diagnostics?
7. Would moving code into a non-UI target clarify isolation?
8. Does the test target have the same actor-isolation settings as the app target?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `@MainActor` makes code run on the main thread, so use it for UI updates.

This is incomplete. It misses isolation semantics, caller rules, design boundaries, and performance implications.

### Senior answer

> `@MainActor` is a global actor used to isolate UI-affine state. I usually mark view models or UI controllers as `@MainActor` when their mutable state drives UI, but I keep services and expensive work non-main isolated. Cross-actor calls require `await`, and `Task.detached` is usually wrong from UI objects because it loses actor context.

### Staff-level answer

> `@MainActor` should express ownership of UI state, not serve as a blanket synchronization mechanism. I’d isolate presentation state at the UI boundary, keep domain/network/storage layers non-main isolated, and explicitly move CPU-heavy work away from the main actor. In Swift 6.2, I’d also treat default actor isolation as a module-level architecture decision: reasonable for app/UI targets, dangerous for reusable core packages unless intentionally UI-affine.

Staff-level questions to ask:

```text
Which target owns the UI isolation default?
Are domain packages accidentally inheriting MainActor?
Can this service be used from background contexts?
Where does cancellation/stale-result suppression live?
Does the isolation boundary match ownership, or just silence compiler errors?
```

---

## 9. Interview-ready summary

`@MainActor` is Swift’s built-in global actor for UI-affine code. I use it to make UI state mutation a compile-time isolation contract. A whole view model should be `@MainActor` when its identity and mutable state are presentation-owned; otherwise, I annotate only the UI-touching members. Overusing `@MainActor` is a smell because it couples non-UI code to the UI executor, can move expensive work onto the main actor, and hides ownership problems. The clean design is: UI state on `MainActor`, services/domain work off `MainActor`, explicit cancellation and stale-result handling around async workflows.

---

## 10. Flashcards

Q: What is a global actor?  
A: A globally shared actor isolation domain that can isolate declarations across types, files, and modules.

Q: What is `@MainActor`?  
A: Swift’s built-in global actor for main-thread/UI-affine work.

Q: Does `@MainActor` mean “always dispatch asynchronously to the main queue”?  
A: No. It is an actor isolation contract. Calls from the same actor can be synchronous; calls from outside the actor require an async boundary.

Q: When should a whole type be `@MainActor`?  
A: When the type is UI-owned and most of its mutable state/methods are UI-affine.

Q: When should only a member be `@MainActor`?  
A: When a mostly non-UI type has a small number of UI-touching methods or properties.

Q: Why is overusing `@MainActor` dangerous?  
A: It serializes unrelated work through the UI executor, couples lower layers to UI, and can hide poor ownership design.

Q: Is `Task.detached` a good way to leave the main actor from a view model?  
A: Usually no. It loses actor context, priority/task-local inheritance, and structured/lifecycle expectations.

Q: Does `@MainActor` prevent stale network results from updating UI?  
A: No. It prevents data races on UI state, but stale-result handling requires cancellation, IDs, or validation.

Q: In Swift 6.2, what is `@concurrent` useful for?  
A: Explicitly marking async work that should run concurrently instead of remaining serialized on the caller’s actor.

Q: Should reusable networking packages default to `MainActor` isolation?  
A: Usually no. Keep reusable non-UI packages nonisolated unless their purpose is explicitly UI-affine.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D4 — Actors and actor isolation]]
- [[D6 — Sendable and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D8 — Isolation control: `nonisolated`, isolated parameters, and `assumeIsolated`]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D14 — Swift 6.2 approachable concurrency and new defaults]]
- [[F1 — Testing async and concurrent code]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- Swift Senior/Staff Rubric — D5 Global actors and `@MainActor`.
- SE-0316: Global Actors. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md "swift-evolution/proposals/0316-global-actors.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0306: Actors and actor isolation. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))
- Swift compiler diagnostic: actor-isolated calls from synchronous nonisolated contexts. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/?utm_source=chatgpt.com "Calling an actor-isolated method from a synchronous ..."))
- Swift 6.2 Released: default main-actor isolation and `@concurrent`. ([Swift.org](https://swift.org/blog/swift-6.2-released/ "Swift 6.2 Released | Swift.org"))