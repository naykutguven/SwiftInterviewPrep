---
tags:
  - swift
  - ios
  - interview-prep
  - tooling
  - observation
  - property-wrappers
  - swiftui
  - macros
---
## 0. Rubric snapshot

**Rubric expectation**

Modern Apple-platform Swift uses a lot of macro- and property-wrapper-driven patterns. You should understand the generated semantics instead of treating `@Observable`, `@State`, `@Bindable`, `@Environment`, `@AppStorage`, etc. as magic.

**Caveats**

Observation tools are convenient, but they do **not** suspend Swift’s normal rules. Generated code still obeys ownership, isolation, mutation, access control, initialization, and concurrency rules.

**You should be able to answer**

- What kinds of hidden coupling or update behavior can property-wrapper or observation-heavy code create?
- Why is understanding generated semantics important even when a framework makes the syntax look trivial?

**You should be able to do**

- Review an observable model with thread-sensitive state and explain where language-level isolation still needs to be explicit.

---

## 1. Core mental model

Observation and property wrappers are **syntax compression**, not semantic escape hatches.

A property wrapper changes the storage/access pattern of a property. Swift synthesizes backing storage and computed accessors around `wrappedValue`; if the wrapper exposes `projectedValue`, the `$property` projection becomes available. SE-0258 defines wrappers as types marked with `@propertyWrapper` that must provide `wrappedValue`, and explains that applying a wrapper makes the source property computed while introducing synthesized storage. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md "swift-evolution/proposals/0258-property-wrappers.md at main · swiftlang/swift-evolution · GitHub"))

`@Observable` is an attached macro. Apple’s docs describe `Observable()` as a macro that adds observation support and conforms a type to `Observable`; SwiftUI’s model-data guidance says the macro generates code at compile time. ([Apple Developer](https://developer.apple.com/documentation/observation/observable%28%29?utm_source=chatgpt.com "Observable() | Apple Developer Documentation"))

The important part is what the macro expands into. SE-0395 says `@Observable` synthesizes `Observable` conformance, an observation registrar, helper methods for tracking access/mutation, and transforms stored properties into tracked computed properties backed by underscored storage. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md "swift-evolution/proposals/0395-observability.md at main · swiftlang/swift-evolution · GitHub"))

The key idea:

```text
@Observable / @State / @Bindable reduce boilerplate;
they do not decide ownership, isolation, lifetime, or domain invariants for you.
```

Observation tracks **property access** and invalidates dependents when accessed properties change. That is a very different model from “notify the whole world whenever anything changes.” SE-0395 describes `withObservationTracking` as capturing tracked-property accesses in a scope and calling `onChange` when one of those accessed properties changes. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md "swift-evolution/proposals/0395-observability.md at main · swiftlang/swift-evolution · GitHub"))

For iOS 18+ SwiftUI code, the default mental model should be:

```text
@State owns local view state.
@Observable makes reference models observable.
@Bindable creates bindings into mutable observable models.
@Environment injects shared context.
@MainActor / actors / Sendable still define concurrency correctness.
```

---

## 2. Essential mechanics

### Mechanic 1: `@Observable` turns stored properties into tracked computed properties

Conceptually:

```swift
import Observation

@Observable
final class SearchModel {
    var query = ""
    var results: [String] = []
}
```

is closer to:

```swift
final class SearchModel: Observable {
    private let registrar = ObservationRegistrar()

    private var _query = ""

    var query: String {
        get {
            registrar.access(self, keyPath: \.query)
            return _query
        }
        set {
            registrar.withMutation(of: self, keyPath: \.query) {
                _query = newValue
            }
        }
    }

    private var _results: [String] = []

    var results: [String] {
        get {
            registrar.access(self, keyPath: \.results)
            return _results
        }
        set {
            registrar.withMutation(of: self, keyPath: \.results) {
                _results = newValue
            }
        }
    }
}
```

That is not exact public source code, but it is the right mental model. The proposal explicitly says the macro adds registrar storage, helper methods, and converts each stored property to a computed property with underscored ignored storage. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md "swift-evolution/proposals/0395-observability.md at main · swiftlang/swift-evolution · GitHub"))

### Mechanic 2: SwiftUI observes what the view reads

```swift
@Observable
final class Library {
    var books: [Book] = []
    var searchText = ""

    var filteredBooks: [Book] {
        books.filter { $0.title.localizedCaseInsensitiveContains(searchText) }
    }
}

struct LibraryView: View {
    @State private var library = Library()

    var body: some View {
        List(library.filteredBooks) { book in
            Text(book.title)
        }
    }
}
```

The view reads `library.filteredBooks`, and that computed property reads `books` and `searchText`. The dependency is therefore broader than it looks at the call site.

SE-0395 explicitly states that computed properties deriving from stored properties are automatically tracked because they rely on tracked properties. It also says computed properties backed by remote storage or other indirection require manual tracking through generated access/mutation helpers. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md "swift-evolution/proposals/0395-observability.md at main · swiftlang/swift-evolution · GitHub"))

### Mechanic 3: `@Bindable` creates bindings into an observable model; it does not own the model

```swift
@Observable
final class Book {
    var title = ""
    var isFavorite = false
}

struct BookEditor: View {
    @Bindable var book: Book

    var body: some View {
        Form {
            TextField("Title", text: $book.title)
            Toggle("Favorite", isOn: $book.isFavorite)
        }
    }
}
```

Apple’s `Bindable` documentation says it creates bindings to mutable properties of a data model object that conforms to `Observable`. ([Apple Developer](https://developer.apple.com/documentation/swiftui/bindable?utm_source=chatgpt.com "Bindable | Apple Developer Documentation"))

That means:

```text
@Bindable is not ownership.
@Bindable is not synchronization.
@Bindable is a binding projection into an observable object.
```

### Mechanic 4: `@State` can own observable reference models in modern SwiftUI

```swift
struct SearchScreen: View {
    @State private var model = SearchModel()

    var body: some View {
        SearchContent(model: model)
    }
}
```

Apple’s `State` documentation says `@State` can store observable objects created with the `Observable()` macro. ([Apple Developer](https://developer.apple.com/documentation/SwiftUI/State%28%29?utm_source=chatgpt.com "State() | Apple Developer Documentation"))

For iOS 18+ code, this is the normal replacement for many old `@StateObject` / `ObservableObject` patterns when using the new Observation framework.

### Mechanic 5: `@ObservationIgnored` opts a property out of observation

```swift
@MainActor
@Observable
final class SearchModel {
    var query = ""
    var results: [String] = []

    @ObservationIgnored
    private var currentTask: Task<Void, Never>?
}
```

Use this for implementation detail that should not trigger UI updates: tasks, caches, loggers, services, formatters, dependencies, locks, and other plumbing.

Apple documents `ObservationIgnored()` as disabling observation tracking for a property. SE-0395 also explains that developers can apply it to stored properties that should not participate in tracking. ([Apple Developer](https://developer.apple.com/documentation/Observation/ObservationIgnored%28%29?utm_source=chatgpt.com "ObservationIgnored() | Apple Developer Documentation"))

---

## 3. Common traps and misconceptions

### Trap 1: “`@Observable` makes my model thread-safe”

It does not.

Bad:

```swift
import Observation

@Observable
final class SearchModel {
    var results: [String] = []

    func refresh() {
        Task.detached {
            self.results = ["A", "B", "C"]
        }
    }
}
```

This mixes UI-observed state with detached background mutation. Observation can register access/mutation; it does not decide which executor owns the state.

Better:

```swift
import Observation

@MainActor
@Observable
final class SearchModel {
    private(set) var results: [String] = []

    func refresh(using service: SearchService) {
        Task {
            let loaded = await service.search()
            results = loaded
        }
    }
}
```

If the model drives SwiftUI state, make that isolation explicit with `@MainActor`.

### Trap 2: Hiding global state behind property wrappers

Bad:

```swift
struct SettingsView: View {
    @AppStorage("showExperimentalUI") private var showExperimentalUI = false

    var body: some View {
        Toggle("Experimental UI", isOn: $showExperimentalUI)
    }
}
```

This is fine for a small local preference. It becomes a problem when business-critical app behavior is scattered across many views through implicit storage keys.

Better:

```swift
@MainActor
@Observable
final class SettingsModel {
    var showExperimentalUI: Bool {
        get { settingsStore.showExperimentalUI }
        set { settingsStore.showExperimentalUI = newValue }
    }

    @ObservationIgnored
    private let settingsStore: SettingsStore

    init(settingsStore: SettingsStore) {
        self.settingsStore = settingsStore
    }
}
```

Centralize policy. Keep wrappers at the edge.

### Trap 3: Assuming computed properties are “free”

```swift
@Observable
final class FeedModel {
    var posts: [Post] = []

    var sortedPosts: [Post] {
        posts.sorted { $0.date > $1.date }
    }
}
```

Every view read of `sortedPosts` may do real work. Observation tracks dependencies; it does not memoize expensive computation for you.

Better options:

```swift
@Observable
final class FeedModel {
    private(set) var sortedPosts: [Post] = []

    func updatePosts(_ posts: [Post]) {
        sortedPosts = posts.sorted { $0.date > $1.date }
    }
}
```

or use a carefully invalidated cache:

```swift
@Observable
final class FeedModel {
    var posts: [Post] = [] {
        didSet {
            cachedSortedPosts = nil
        }
    }

    @ObservationIgnored
    private var cachedSortedPosts: [Post]?

    var sortedPosts: [Post] {
        if let cachedSortedPosts {
            return cachedSortedPosts
        }

        let sorted = posts.sorted { $0.date > $1.date }
        cachedSortedPosts = sorted
        return sorted
    }
}
```

Be careful: caching inside an observable type is now mutation. If this model is UI-bound, isolate it.

### Trap 4: Treating wrappers as harmless decoration

```swift
@propertyWrapper
struct Locked<Value> {
    private var value: Value

    var wrappedValue: Value {
        get { value }
        set { value = newValue }
    }

    init(wrappedValue: Value) {
        self.value = wrappedValue
    }
}
```

The name suggests synchronization, but this wrapper does nothing thread-safe. Worse, even a real lock wrapper often cannot safely support compound operations like:

```swift
counter += 1
```

because that can become a separate get and set. SE-0258’s historical notes explicitly call out that atomic property-wrapper examples can encourage races because useful atomic operations cannot be built from independent load/store wrappers. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md "swift-evolution/proposals/0258-property-wrappers.md at main · swiftlang/swift-evolution · GitHub"))

---

## 4. Direct answers to rubric questions

### Q1. What kinds of hidden coupling or update behavior can property-wrapper or observation-heavy code create?

Observation-heavy code can hide coupling through implicit read tracking, computed-property dependencies, environment injection, wrapper projections, and global storage.

A view might look like it depends on one property:

```swift
Text(model.summary)
```

but `summary` might read:

```swift
var summary: String {
    "\(user.name) — \(cart.items.count) — \(featureFlags.currentVariant)"
}
```

Now the view update behavior is coupled to `user.name`, `cart.items`, and `featureFlags.currentVariant`.

Common hidden couplings:

```text
Computed property reads
Environment values
@AppStorage / SceneStorage keys
@Bindable mutations from child views
Wrapper-provided projected values
Shared observable reference models
Observable arrays/nested reference models
ObservationIgnored implementation details
```

The coupling is not always bad. It is often the point of declarative UI. The staff-level skill is knowing when the coupling is intentional, local, and testable versus implicit, global, and hard to reason about.

Interview version:

> Observation tracks what code reads, not what the type author had in mind. A view can become coupled to a model through computed properties, environment values, bindings, and wrapper projections. That is powerful because it reduces boilerplate, but it can create surprising invalidation, hidden global dependencies, or under-updates if state is hidden behind `@ObservationIgnored` or external storage. I treat wrappers and macros as generated code with ownership and isolation implications, not as decoration.

### Q2. Why is understanding generated semantics important even when a framework makes the syntax look trivial?

Because the generated code affects storage, access control, initialization, mutation, observation, performance, actor isolation, and public API compatibility.

`@Observable` looks like one line:

```swift
@Observable
final class Model {
    var count = 0
}
```

But it changes the shape of the type. Stored properties become tracked computed properties with backing storage and registrar calls. SE-0395 also notes that changing a type to `@Observable` has the same ABI impact as changing a property from stored to computed, and removing `@Observable` removes a conformance, which is ABI breaking. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md "swift-evolution/proposals/0395-observability.md at main · swiftlang/swift-evolution · GitHub"))

Property wrappers are similar. A simple wrapper:

```swift
@Clamped var score = 120
```

is not “a stored `Int` with decoration.” It is wrapper storage plus synthesized accessors around `wrappedValue`. SE-0258 explains that property wrappers introduce stored wrapper storage and make the source property computed. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md "swift-evolution/proposals/0258-property-wrappers.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> The syntax is intentionally small, but the generated semantics are not small. A wrapper changes storage and accessors; `@Observable` adds a registrar and turns stored properties into tracked computed properties. That matters for initialization, performance, public API resilience, debugging, `Sendable`, actor isolation, and view invalidation. If I do not understand the expansion, I cannot reliably debug why a view updates, why it does not update, or why Swift 6 reports an isolation problem.

---

## 5. Code probe replacement

The F6 rubric has no code probe. This section uses a minimal example, a counterexample with a Swift 6.2 diagnostic, and a production-oriented redesign.

### Minimal example

```swift
import Observation
import SwiftUI

@MainActor
@Observable
final class CounterModel {
    var count = 0

    func increment() {
        count += 1
    }
}

struct CounterView: View {
    @State private var model = CounterModel()

    var body: some View {
        Button("Count: \(model.count)") {
            model.increment()
        }
    }
}
```

### What happens?

```text
The view reads model.count.
SwiftUI tracks that read.
When increment() mutates count, Observation reports the mutation.
SwiftUI invalidates the dependent view.
```

### Why?

```text
CounterView.body
    reads model.count
        -> access is tracked

Button action
    calls model.increment()
        -> mutates model.count
        -> mutation is reported
        -> body can be recomputed
```

`@MainActor` is not for observation. It is for isolation: this model is UI state, so mutation belongs on the main actor.

### Counterexample

```swift
@MainActor
final class SearchModel {
    var results: [String] = []

    func refresh() {
        Task.detached {
            self.results = ["A"]
        }
    }
}
```

### Swift 6.2 compiler error

```text
error: main actor-isolated property 'results' can not be mutated from a nonisolated context
note: mutation of this property is only permitted within the actor
note: consider declaring an isolated method on 'MainActor' to perform the mutation
```

### Why?

```text
SearchModel is MainActor-isolated.
Task.detached does not inherit MainActor isolation.
The closure runs in a nonisolated context.
Mutating self.results crosses the actor boundary incorrectly.
```

`@Observable` would not fix this. Observation reports changes; it does not grant permission to mutate actor-isolated state from the wrong executor.

### Fix or redesign

```swift
@MainActor
@Observable
final class SearchModel {
    private(set) var results: [String] = []

    func refresh(using service: SearchService) {
        Task {
            let loaded = await service.search()
            results = loaded
        }
    }
}

struct SearchService: Sendable {
    func search() async -> [String] {
        ["A"]
    }
}
```

### Why this fix is correct

`Task {}` created from a main-actor-isolated context preserves the actor context for accesses to `self`. The asynchronous service work can suspend, but mutation of `results` is still performed through the model’s main-actor isolation.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`@MainActor @Observable` model|UI-facing view model/state object|Easy to reason about, but don’t put heavy work on the main actor|
|Actor-backed store + main-actor observable adapter|Shared mutable cache, database, token store, cross-feature state|More boilerplate, clearer isolation boundary|
|Lock/mutex around a small value|Very small synchronous shared state, hot path, no async suspension needed|Requires disciplined invariants; easy to misuse|
|`Task.detached` + explicit `await MainActor.run`|Rare boundary to fully detached work|Easy to lose cancellation, priority, task-local values, and lifecycle semantics|

---

## 6. Exercise

### Problem

Review an observable model with thread-sensitive state and explain where language-level isolation still needs to be explicit.

### Bad / naive version

```swift
import Observation

@Observable
final class SearchViewModel {
    var query = ""
    var results: [SearchResult] = []
    var isLoading = false
    var errorMessage: String?

    private let service: SearchService
    private var currentTask: Task<Void, Never>?

    init(service: SearchService) {
        self.service = service
    }

    func search() {
        currentTask?.cancel()

        currentTask = Task.detached {
            self.isLoading = true
            self.errorMessage = nil

            do {
                let results = try await self.service.search(self.query)
                self.results = results
                self.isLoading = false
            } catch {
                self.errorMessage = error.localizedDescription
                self.isLoading = false
            }
        }
    }
}

struct SearchResult: Identifiable {
    let id: UUID
    let title: String
}

struct SearchService {
    func search(_ query: String) async throws -> [SearchResult] {
        []
    }
}
```

### What is wrong?

```text
1. @Observable does not make the model thread-safe.
2. Task.detached loses actor context.
3. UI-facing mutable state has no explicit @MainActor isolation.
4. Multiple properties allow invalid intermediate states:
   - isLoading == true with stale results
   - errorMessage != nil while results are non-empty
   - query changes while old request completes
5. currentTask is implementation detail but is observable by default unless ignored.
6. service crosses concurrency boundaries without an explicit Sendable story.
7. Stale-result suppression is not modeled.
```

### Improved version

```swift
import Observation

@MainActor
@Observable
final class SearchViewModel {
    enum State: Equatable {
        case idle
        case loading(query: String)
        case loaded(query: String, results: [SearchResult])
        case failed(query: String, message: String)
    }

    var query = ""
    private(set) var state: State = .idle

    @ObservationIgnored
    private let service: SearchService

    @ObservationIgnored
    private var currentTask: Task<Void, Never>?

    init(service: SearchService) {
        self.service = service
    }

    func search() {
        let submittedQuery = query.trimmingCharacters(in: .whitespacesAndNewlines)

        guard !submittedQuery.isEmpty else {
            currentTask?.cancel()
            state = .idle
            return
        }

        currentTask?.cancel()
        state = .loading(query: submittedQuery)

        currentTask = Task { [service] in
            do {
                let results = try await service.search(submittedQuery)
                try Task.checkCancellation()

                guard query == submittedQuery else {
                    return
                }

                state = .loaded(query: submittedQuery, results: results)
            } catch is CancellationError {
                // Keep current state or let the next search replace it.
            } catch {
                guard query == submittedQuery else {
                    return
                }

                state = .failed(
                    query: submittedQuery,
                    message: error.localizedDescription
                )
            }
        }
    }

    deinit {
        currentTask?.cancel()
    }
}

struct SearchResult: Identifiable, Equatable, Sendable {
    let id: UUID
    let title: String
}

struct SearchService: Sendable {
    var search: @Sendable (String) async throws -> [SearchResult]
}
```

### Why this is better

```text
UI state is explicitly MainActor-isolated.
Observation drives view invalidation.
Concurrency isolation drives mutation correctness.
State enum prevents invalid UI states.
Task storage is ignored by observation.
Service and result values have a Sendable story.
Cancellation and stale-result suppression are explicit.
```

This is the core F6 lesson: `@Observable` handles dependency tracking, but actor isolation, cancellation, sendability, and state modeling remain your job.

### Production version with a background actor

For shared cache or thread-sensitive storage, keep the observable model as a UI adapter and move cross-task mutable state behind an actor:

```swift
actor SearchCache {
    private var values: [String: [SearchResult]] = [:]

    func results(for query: String) -> [SearchResult]? {
        values[query]
    }

    func store(_ results: [SearchResult], for query: String) {
        values[query] = results
    }
}

@MainActor
@Observable
final class SearchViewModel {
    enum State: Equatable {
        case idle
        case loading(query: String)
        case loaded(query: String, results: [SearchResult])
        case failed(query: String, message: String)
    }

    var query = ""
    private(set) var state: State = .idle

    @ObservationIgnored private let service: SearchService
    @ObservationIgnored private let cache: SearchCache
    @ObservationIgnored private var currentTask: Task<Void, Never>?

    init(service: SearchService, cache: SearchCache) {
        self.service = service
        self.cache = cache
    }

    func search() {
        let submittedQuery = query.trimmingCharacters(in: .whitespacesAndNewlines)

        currentTask?.cancel()
        state = .loading(query: submittedQuery)

        currentTask = Task { [service, cache] in
            do {
                if let cached = await cache.results(for: submittedQuery) {
                    guard query == submittedQuery else { return }
                    state = .loaded(query: submittedQuery, results: cached)
                    return
                }

                let loaded = try await service.search(submittedQuery)
                try Task.checkCancellation()

                await cache.store(loaded, for: submittedQuery)

                guard query == submittedQuery else { return }
                state = .loaded(query: submittedQuery, results: loaded)
            } catch is CancellationError {
                // No-op.
            } catch {
                guard query == submittedQuery else { return }
                state = .failed(
                    query: submittedQuery,
                    message: error.localizedDescription
                )
            }
        }
    }
}
```

Isolation split:

```text
SearchViewModel
    @MainActor
    owns UI-facing observable state

SearchCache
    actor
    owns shared mutable background state

SearchService
    Sendable
    can cross concurrency boundaries

Observation
    updates views when tracked UI-facing properties change
```

---

## 7. Production guidance

Use this in production when:

```text
You target iOS 17+ / iOS 18+ and use SwiftUI.
A reference model naturally owns UI state.
You want fine-grained view invalidation based on property reads.
You want less boilerplate than ObservableObject + @Published.
You can clearly define ownership and isolation boundaries.
```

Be careful when:

```text
The observable model contains mutable shared state.
The model is used from both UI and background code.
Computed properties do expensive work.
The model stores tasks, services, locks, caches, or formatters.
State is spread across @AppStorage, @Environment, @State, and observable references.
Nested reference models or arrays make update behavior less obvious.
```

Avoid when:

```text
You need a domain model that should remain UI-framework independent.
You are using @Observable as a substitute for actor isolation.
You are hiding business-critical global state in wrappers.
You are building a public SDK where generated API shape/resilience matters.
You cannot explain what owns the state and which executor may mutate it.
```

Debugging checklist:

```text
Which view read caused the dependency?
Is this property actually tracked?
Is any field marked @ObservationIgnored incorrectly?
Is this a computed property that reads more state than expected?
Is expensive work happening inside body via a computed property?
Who owns this observable object?
Is mutation isolated to @MainActor or another actor?
Did a Task.detached lose actor context?
Are service/result types Sendable where they cross concurrency boundaries?
Is there stale-result suppression after await?
Is state modeled as one enum or many inconsistent booleans/optionals?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `@Observable` updates SwiftUI when properties change. Property wrappers like `@State` and `@Bindable` help manage state.

### Senior answer

> `@Observable` is a macro that generates observation support. It tracks property access and reports mutations, so SwiftUI can invalidate views that depend on specific properties. But it does not solve ownership, state modeling, thread safety, actor isolation, or cancellation. I still need `@MainActor`, actors, `Sendable`, and clear source-of-truth rules.

### Staff-level answer

> I treat Observation and property wrappers as generated code with architectural consequences. `@Observable` changes property storage/access semantics and can become part of a public API contract. SwiftUI’s read tracking can create implicit coupling through computed properties and environment values. In a production architecture, I keep UI-facing observable state main-actor-isolated, move shared mutable state into actors or disciplined synchronization, keep implementation details `@ObservationIgnored`, model UI state as enums, and make state ownership explicit at feature boundaries. I also check macro expansion or compiler diagnostics when behavior is unclear instead of guessing.

Staff-level questions to ask:

```text
What owns this observable object?
Which actor is allowed to mutate it?
Which properties should be observable and which should be ignored?
Are computed properties hiding expensive work or extra dependencies?
Is this state local, feature-scoped, app-scoped, or persistent?
Does a child view need @Bindable, or should mutation flow through methods?
Are we exposing generated observable API in a public module?
Can tests prove cancellation, stale-result suppression, and isolation?
```

---

## 9. Interview-ready summary

Observation and property wrappers are powerful because they compress boilerplate, but they do not remove Swift’s semantics. A property wrapper synthesizes backing storage and accessors around `wrappedValue`; `@Observable` is a macro that adds observation support, registrar storage, and tracked access/mutation around properties. SwiftUI uses that tracking to update views based on what the view reads. The risk is hidden coupling through computed properties, environment values, bindings, and wrapper projections. For production SwiftUI, I use `@Observable` for UI-facing state, `@State` for ownership, `@Bindable` for child mutation, `@ObservationIgnored` for implementation detail, and explicit `@MainActor`, actors, and `Sendable` for concurrency correctness.

---

## 10. Flashcards

Q: What does `@Observable` do at a high level?  
A: It is an attached macro that adds observation support to a type, including `Observable` conformance and tracked access/mutation for properties.

Q: Does `@Observable` make a model thread-safe?  
A: No. Observation tracks changes; actor isolation, locks, immutability, and `Sendable` handle concurrency correctness.

Q: What is the difference between `@State` and `@Bindable` with observable models?  
A: `@State` owns local view state; `@Bindable` creates bindings into mutable properties of an existing observable model.

Q: When should you use `@ObservationIgnored`?  
A: For implementation details that should not trigger observation: tasks, services, caches, locks, loggers, formatters, and derived storage.

Q: Why can computed properties create hidden observation coupling?  
A: A computed property can read multiple tracked properties, so a view that reads one computed value may become dependent on all underlying reads.

Q: Why is `Task.detached` dangerous in an observable view model?  
A: It loses actor context and structured task inheritance, so mutating UI-observed state from it can violate isolation.

Q: What does a property wrapper synthesize?  
A: Backing wrapper storage plus computed accessors that read/write `wrappedValue`; optionally a `$property` projection from `projectedValue`.

Q: What is the staff-level view of Observation?  
A: Use it as a UI invalidation mechanism, not as an architecture. Keep source of truth, isolation, state modeling, and public API boundaries explicit.

---

## 11. Related sections

- [[B13 — Property wrappers and generated storage semantics]]
- [[F4 — Macros and compile-time code generation]]
- [[F5 — Language mode, build settings, and migration flags]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[A9 — Enums, associated values, recursive enums, and state modeling]]

---

## 12. Sources

- Swift Senior/Staff Rubric — F6 expectation, caveats, questions, and exercise.
- Apple Developer Documentation — Observation framework overview. ([Apple Developer](https://developer.apple.com/documentation/observation?utm_source=chatgpt.com "Observation | Apple Developer Documentation"))
- Apple Developer Documentation — `Observable()` macro. ([Apple Developer](https://developer.apple.com/documentation/observation/observable%28%29?utm_source=chatgpt.com "Observable() | Apple Developer Documentation"))
- Apple Developer Documentation — Managing model data in SwiftUI. ([Apple Developer](https://developer.apple.com/documentation/swiftui/managing-model-data-in-your-app?utm_source=chatgpt.com "Managing model data in your app"))
- Apple Developer Documentation — `Bindable`. ([Apple Developer](https://developer.apple.com/documentation/swiftui/bindable?utm_source=chatgpt.com "Bindable | Apple Developer Documentation"))
- Apple Developer Documentation — `State`. ([Apple Developer](https://developer.apple.com/documentation/SwiftUI/State%28%29?utm_source=chatgpt.com "State() | Apple Developer Documentation"))
- Apple Developer Documentation — `ObservationIgnored()`. ([Apple Developer](https://developer.apple.com/documentation/Observation/ObservationIgnored%28%29?utm_source=chatgpt.com "ObservationIgnored() | Apple Developer Documentation"))
- SE-0395 — Observation. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md "swift-evolution/proposals/0395-observability.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0258 — Property Wrappers. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md "swift-evolution/proposals/0258-property-wrappers.md at main · swiftlang/swift-evolution · GitHub"))