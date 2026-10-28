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

# F6 — Observation, Property Wrappers, and Language-Adjacent State Tools

**Learning goals:**

- Trace actual tracked reads through branches, caches and external storage.
- Separate generated storage, SwiftUI identity/bindings, actor ownership and observation lifetime.
- Use request identity for async publication and distinguish one-shot, continuous and async observation contracts.

**Prerequisites:** [[B13 — Property Wrappers and Generated Storage Semantics]] for wrappers, [[D5 — Global Actors and `@MainActor`]]/[[D6 — `Sendable`, `@Sendable`, and `sending`]]/[[D7 — Actor Reentrancy and Logic Races]] for isolation and freshness, [[C1 — ARC fundamentals]] for lifetime, and [[F1 — Testing Strategy in Modern Swift]] for controlled tests. The SwiftUI examples are compiled declarations rather than rendered UI.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A computed result redraws after its first read but stops updating after a cache hit. Observation tracks property reads that actually happened inside a scope; the computed property’s name does not automatically supply every dependency.

@Observable generates accessors, storage and a registrar that records those reads/mutations. It does not supply synchronization, persistence, stable view identity or a lossless event log. This collection uses neutral declaration defaults, so UI-owned models explicitly select MainActor. [Observation model](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0395-observability.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Track one read and keep callback capture safety separate

Shared Model/ChangeLog setup. Observation itself needs macOS 14/iOS 17; the synchronized log uses Mutex, requiring macOS 15/iOS 18. The baseline runs on OS 27.

```swift
import Observation
import Synchronization

final class ChangeLog: Sendable {
    private let value = Mutex(0)
    func record() { value.withLock { $0 += 1 } }
    func count() -> Int { value.withLock { $0 } }
}
@MainActor @Observable final class Model {
    var count = 0
    var unrelated = 0
    @ObservationIgnored var bookkeeping = 0
}
@MainActor func initialProbe() {
    let model = Model()
    let log = ChangeLog()
    _ = withObservationTracking {
        model.count
    } onChange: {
        log.record()
    }
    model.unrelated = 1
    model.bookkeeping = 1
    precondition(log.count() == 0)
    model.count = 1
    model.count = 2
    precondition(log.count() == 1 && model.count == 2)
}
initialProbe()
```

**Walkthrough:** Only count was read. Unrelated/ignored writes do not affect that dependency, and the original callback fires once before the relevant write. The Sendable callback captures a synchronized log, not actor-owned model state.

### State ownership and Bindable projection answer different questions

Compile-only SwiftUI declarations; prepend Model/ChangeLog setup. State retains its model according to stable view identity, while Bindable supplies property bindings for an existing model.

```swift
import SwiftUI

@MainActor struct CounterScreen: View {
    @State private var model = Model()
    var body: some View { CounterEditor(model: model) }
}
@MainActor struct CounterEditor: View {
    @Bindable var model: Model
    var body: some View { Stepper("Count: \(model.count)", value: $model.count) }
}
```

**Walkthrough:** Reconstructing a View value is different from replacing its identity. The child’s projection edits the same model; Bindable can also retain that model. Neither wrapper synchronizes arbitrary storage.

### A cache and an external store need explicit dependency paths

The pitfalls below use value-type Post inputs and a main-actor cache. Reference payload mutation would require its own invalidation policy. External storage is demonstrated with an in-memory adapter; all its writes enter through the adapter, not a disk/UserDefaults/cross-process observer.

## Rules and pitfalls

### A cache-hit path can omit the tracked source read

**Common pitfall:** Returning ignored cached storage before reading posts. The complete bad cache is used only to observe missing invalidation.

**Mistake:** Standalone runnable example.

```swift
import Observation

struct Post: Equatable, Sendable {
    let id: Int
    let rank: Int
}

@MainActor @Observable
final class BadFeedModel {
    var posts: [Post] = [] {
        didSet { cachedSortedPosts = nil }
    }

    @ObservationIgnored private var cachedSortedPosts: [Post]?

    var sortedPosts: [Post] {
        if let cachedSortedPosts { return cachedSortedPosts }
        let result = posts.sorted {
            $0.rank == $1.rank ? $0.id < $1.id : $0.rank > $1.rank
        }
        cachedSortedPosts = result
        return result
    }
}

let model = BadFeedModel()
model.posts = [Post(id: 1, rank: 1)]
_ = model.sortedPosts
let log = ChangeLog()
_ = withObservationTracking {
    model.sortedPosts
} onChange: {
    log.record()
}
model.posts = [Post(id: 2, rank: 2)]
precondition(log.count() == 0)
```

**Consequence:** The new tracking scope read only the ignored cached value, so changing posts did not notify it.

**Why:** The first computation happened to read posts, but the later cache-hit path returned before doing so. Dependencies follow executed reads, not the earlier computation’s history.

**Correction:** Read the source on every getter path and invalidate the cache when it changes. Shared FeedModel setup; prepend ChangeLog only when running the probe.

```swift
import Observation

struct Post: Equatable, Sendable {
    let id: Int
    let rank: Int
}

@MainActor @Observable
final class FeedModel {
    var posts: [Post] = [] {
        didSet { cachedSortedPosts = nil }
    }

    @ObservationIgnored private var cachedSortedPosts: [Post]?

    var sortedPosts: [Post] {
        let source = posts  // Register the dependency even on a cache hit.
        if let cachedSortedPosts { return cachedSortedPosts }
        let result = source.sorted {
            $0.rank == $1.rank ? $0.id < $1.id : $0.rank > $1.rank
        }
        cachedSortedPosts = result
        return result
    }
}

@MainActor func cacheProbe() {
    let model = FeedModel()
    model.posts = [Post(id: 1, rank: 1)]
    _ = model.sortedPosts
    let log = ChangeLog()
    _ = withObservationTracking {
        model.sortedPosts
    } onChange: {
        log.record()
    }
    model.posts = [Post(id: 2, rank: 2)]
    precondition(log.count() == 1 && model.sortedPosts == [Post(id: 2, rank: 2)])
}
cacheProbe()
```

**What changes:** The cache hit now registers posts, and its next write invalidates that dependency. The sort still has stable rank/id policy; caching is not a measured speed improvement.

### A forwarded computed property does not automatically observe its store

**Common pitfall:** Reading an unobservable store through a computed property and expecting the macro to infer its mutation boundary.

**Mistake:** Standalone runnable example.

```swift
import Observation

@MainActor final class RawStore { var enabled = false }
@MainActor @Observable final class UntrackedSettings {
    private let store = RawStore()
    var enabled: Bool {
        get { store.enabled }
        set { store.enabled = newValue }
    }
}
let settings = UntrackedSettings()
let log = ChangeLog()
_ = withObservationTracking {
    settings.enabled
} onChange: {
    log.record()
}
settings.enabled = true
precondition(log.count() == 0)
```

**Consequence:** The computed forwarding getter/setter never register or notify the relevant observation access.

**Why:** The plain store is not Observable, and forwarding syntax alone does not instrument this property.

**Correction:** Use an access/mutation boundary in the adapter and route external changes through it. Shared SettingsModel setup.

```swift
import Observation

@MainActor
final class SettingsStore {
    var enabled: Bool
    init(enabled: Bool) { self.enabled = enabled }
}

@MainActor @Observable
final class SettingsModel {
    private let store: SettingsStore

    init(enabled: Bool) { store = SettingsStore(enabled: enabled) }

    var enabled: Bool {
        get {
            access(keyPath: \.enabled)
            return store.enabled
        }
        set {
            withMutation(keyPath: \.enabled) {
                store.enabled = newValue
            }
        }
    }

    func applyExternalValue(_ value: Bool) { enabled = value }
}

let settings = SettingsModel(enabled: false)
let log = ChangeLog()
_ = withObservationTracking {
    settings.enabled
} onChange: {
    log.record()
}
settings.applyExternalValue(true)
precondition(log.count() == 1 && settings.enabled)
```

**What changes:** The adapter records reads and wraps writes in withMutation. A real external writer must notify it; adding an ignored field does not subscribe to a backing store.

### Equal query text does not identify the same operation

**Common pitfall:** Authorizing an old completion by comparing only query text, even after a newer identical query began.

**Mistake:** Standalone runnable example.

```swift
@MainActor final class QueryOnly {
    var query = ""
    var results: [String] = []
    func complete(query: String, values: [String]) {
        if query == self.query { results = values }
    }
}
let state = QueryOnly()
state.query = "A"
state.query = "B"
state.query = "A"
state.complete(query: "A", values: ["new"])
state.complete(query: "A", values: ["old"])
precondition(state.results == ["old"])
```

**Consequence:** A stale completion passes the text check and overwrites newer state without a data race.

**Why:** Cancellation can be ignored or observed late, and A → B → A contains three identities. MainActor serializes access but does not decide which result is current.

**Correction:** Use a unique identity for every operation and guard success, error and cleanup. Shared SearchModel setup; start returns the task so its owner/tests can join superseded work.

```swift
import Observation

@MainActor @Observable final class SearchModel {
    private final class Request {}
    var query = ""
    private(set) var results: [String] = []
    private(set) var lastError: String?
    private(set) var isLoading = false
    @ObservationIgnored private var current: Request?
    @ObservationIgnored private var task: Task<Void, Never>?
    @discardableResult
    func start(_ query: String, search: @escaping @Sendable (String) async throws -> [String])
        -> Task<Void, Never>
    {
        task?.cancel()
        let request = Request()
        current = request
        self.query = query
        lastError = nil
        isLoading = true
        let created = Task { [weak self] in
            defer {
                if let self, self.current === request {
                    self.task = nil
                    self.current = nil
                    self.isLoading = false
                }
            }
            do {
                let values = try await search(query)
                guard let self, self.current === request, !Task.isCancelled else { return }
                self.results = values
            } catch {
                guard let self, self.current === request, !Task.isCancelled else { return }
                self.lastError = String(describing: error)
            }
        }
        task = created
        return created
    }
    func cancel() {
        task?.cancel()
        task = nil
        current = nil
        isLoading = false
    }
}
```

**What changes:** The result/error can publish only for its current identity, and stale cleanup cannot clear a newer slot. The task captures the model weakly through the suspension. Explicit cancel revokes publication and leaves the last successful results intact.

In start, the new Request becomes current before the task can publish. After suspension, both success and error check that exact object identity and cancellation. The defer block clears state only while it still owns the current slot, so an older completion cannot turn off a newer loading indicator. Explicit cancel revokes that slot immediately.

Shared ControlledSearch test setup for the question below. It stores pending continuations by call number and acknowledges registration, so tests choose completion order without sleeps. It intentionally ignores cancellation until resumed to exercise the publication guard.

```swift
actor ControlledSearch {
    private var nextID = 0
    private var pending: [Int: CheckedContinuation<[String], any Error>] = [:]
    private var observers: [(Int, CheckedContinuation<Void, Never>)] = []
    func call(_ query: String) async throws -> [String] {
        nextID += 1
        let id = nextID
        return try await withCheckedThrowingContinuation { continuation in
            pending[id] = continuation
            let ready = observers.filter { $0.0 <= nextID }
            observers.removeAll { $0.0 <= nextID }
            for (_, observer) in ready { observer.resume() }
        }
    }
    func registered(_ count: Int) async {
        if nextID >= count { return }
        await withCheckedContinuation { observers.append((count, $0)) }
    }
    func finish(_ id: Int, with result: Result<[String], any Error>) {
        let continuation = pending.removeValue(forKey: id)!
        continuation.resume(with: result)
    }
    func remaining() -> Int { pending.count }
}
```

## Worked example

### One-shot registration must be established again

**Setup:** Prepend Model/ChangeLog setup. This synchronous MainActor probe never reads actor-owned state in its Sendable callback.

**Predict:** Will two writes produce two callbacks from the first registration?

```swift
let counter = Model()
let log = ChangeLog()
_ = withObservationTracking {
    counter.count
} onChange: {
    log.record()
}
counter.count = 1
counter.count = 2
precondition(log.count() == 1)
_ = withObservationTracking {
    counter.count
} onChange: {
    log.record()
}
counter.count = 3
precondition(log.count() == 2)
```

**Outcome:** The first registration fires once; the second scope observes a later write.

**Why:** Only the expression’s actual reads register dependencies. The callback occurs before the relevant write, so delivery is an invalidation signal rather than an ordered sequence of every stored value.

## Interview questions

### Why can an ignored cache work on its first read but fail after a cache-hit read?

**Answer:** Because the first computation reads tracked inputs, while a later hit can return only ignored storage. Every scope needs its own executed dependency read.

**Example:** Prepend Model/ChangeLog then FeedModel setups.

```swift
let feed = FeedModel()
feed.posts = [Post(id: 2, rank: 1), Post(id: 1, rank: 1)]
precondition(feed.sortedPosts.map(\.id) == [1, 2])
let log = ChangeLog()
_ = withObservationTracking {
    feed.sortedPosts
} onChange: {
    log.record()
}
feed.posts.append(Post(id: 3, rank: 2))
precondition(log.count() == 1 && feed.sortedPosts.map(\.id) == [3, 1, 2])
```

**Walkthrough:** The cache is warm before tracking, yet the getter still reads posts. Appending invalidates that scope and recomputes the stable rank/id order.

**Boundary:** Reference-object payload edits, off-actor writers and actual sort costs need separate policies/evidence. A correctly invalidating cache is not automatically a useful cache.

### Which context runs synchronous versus continuous callbacks, and what owns their lifetime?

**Answer:** The synchronous callback runs during the mutation but its Sendable type grants no actor-isolated model access. The continuous API inherits registration isolation and schedules callbacks under its contract; the owner retains/releases its token.

**Example:** Compile-only OS 27 CounterBinding setup; prepend Model/ChangeLog.

```swift
import Observation

@available(anyAppleOS 27.0, *)
@MainActor
final class CounterBinding {
    private var token: ObservationTracking.Token?

    init(
        model: Model,
        render: @escaping @MainActor @Sendable (Int) -> Void
    ) {
        token = withContinuousObservation(options: [.didSet, .deinit]) { [weak model] event in
            guard let model else {
                event.cancel()
                return
            }
            render(model.count)
        }
    }

    func stop() { token = nil }
}
```

**Walkthrough:** The initial continuous callback is scheduled, not a synchronous render from init. Later callbacks run under the registered MainActor contract and read current state. stop drops the stored token; an owned local token can instead be consumed by cancel().

**Boundary:** The weak model capture avoids prolonging its life, but render can still retain an owner and form a different cycle. Cancellation does not undo UI work already performed. This is the installed withContinuousObservation spelling, distinct from proposal examples.

The synchronous callback’s lack of actor permission is an independent **intentionally rejected** source, despite mutation occurring on MainActor:

```swift
@MainActor func invalidCallback(_ model: Model) {
    _ = withObservationTracking {
        model.count
    } onChange: {
        print(model.count)
    }
}
```

### Why can cancelling the older task and comparing query text still publish a stale result?

**Answer:** Cancellation is cooperative and text can repeat. A unique operation identity must authorize each success, error and cleanup after suspension, independently of whether the service observes cancellation.

**Example:** Prepend SearchModel then ControlledSearch setups.

```swift
let service = ControlledSearch()
let model = SearchModel()
let old = model.start("A") { try await service.call($0) }
await service.registered(1)
let middle = model.start("B") { try await service.call($0) }
await service.registered(2)
let latest = model.start("A") { try await service.call($0) }
await service.registered(3)
await service.finish(3, with: .success(["new"]))
await latest.value
await service.finish(1, with: .success(["old"]))
enum Failure: Error { case old }
await service.finish(2, with: .failure(Failure.old))
await old.value
await middle.value
precondition(
    model.query == "A" && model.results == ["new"] && model.lastError == nil && !model.isLoading)
let remaining = await service.remaining()
precondition(remaining == 0)
```

**Walkthrough:** The newest A finishes first. Older A success and B error arrive afterward but cannot publish or clear its state; every owned task and continuation is joined/released.

**Boundary:** This is an explicit one-result search policy, not lossless events or deduplication. If cancellation must complete immediately when a dependency never returns, the owner needs a separate terminal-state protocol.

### Extension: What do advanced event matching and cancellation establish?

**Answer:** They identify the registered property/phase and can cancel that observation. They do not make Event.Kind an exhaustive enum, imply arbitrary actor access, or guarantee delivery of every event from a single registration.

**Example:** Standalone runnable example.

```swift
import Observation
import Synchronization

@available(anyAppleOS 27.0, *)
@Observable final class Gauge {
    var value = 0
    var other = 0
}

@available(anyAppleOS 27.0, *)
func eventProbe(cancelOnWill: Bool = false) -> [String] {
    let gauge = Gauge()
    let log = Mutex<[String]>([])
    _ = withObservationTracking(options: [.willSet, .didSet]) {
        gauge.value + gauge.other
    } onChange: { event in
        let phase = event.kind == .willSet ? "will" : "did"
        let row = "\(phase):\(event.matches(\Gauge.value)):\(event.matches(\Gauge.other))"
        log.withLock { $0.append(row) }
        if cancelOnWill && event.kind == .willSet { event.cancel() }
    }
    gauge.value = 1
    gauge.value = 2
    return log.withLock { $0 }
}

precondition(eventProbe() == ["will:true:false", "did:true:false"])
precondition(eventProbe(cancelOnWill: true) == ["will:true:false"])
```

**Walkthrough:** The confined nonisolated Gauge is never sent to the callback; only synchronized metadata is recorded. Cancelling on will suppresses the paired did callback, and a later assignment does not re-register the expression.

**Boundary:** These APIs need OS 27. The earlier combined will/did/deinit observation differed from the proposal while a deinit-only registration fired; current controls reproduce that distinction. Never make resource cleanup depend on receiving all three events.

### Extension: When is the async Observations sequence sufficient?

**Answer:** When the consumer needs current state snapshots and accepts coalescing rather than a lossless per-write event log. Own and cancel the consuming task, and account for the sequence’s model capture.

**Example:** Compile-only Observations setup; prepend Model/ChangeLog.

```swift
import Observation

@available(anyAppleOS 26.0, *)
@MainActor
func counterValues(_ model: Model) -> Observations<Int, Never> {
    Observations { model.count }
}
```

**Walkthrough:** The closure reads count under MainActor and strongly captures model for the sequence lifetime. Controls use initial/update handshakes to observe coalescing; a fixed number of yields is not synchronization.

**Boundary:** This API needs OS 26, while advanced options/tokens need 27. Neither adds persistence, synchronization or an automatic consumer lifecycle.

### Extension: What does a Bindable projection establish without rendering a view?

**Answer:** It can write through to the same observable model. View identity and State lifecycle are separate framework behaviors; compiling or constructing a binding alone does not render or prove a view lifecycle.

**Example:** Prepend Model/ChangeLog setup.

```swift
import SwiftUI

@MainActor func bindingProbe() {
    let model = Model()
    @Bindable var editing = model
    $editing.count.wrappedValue = 7
    precondition(model.count == 7)
}
bindingProbe()
```

**Walkthrough:** The binding changes the shared model count. The earlier View declarations compile the State/Bindable composition, while actual identity changes would require a UI lifecycle test.

**Boundary:** Bindable can retain its model and projected closures can retain other owners. It adds no general synchronization, validation or persistence policy.

## Focused exercise

**Task:** Track one observable property and verify unrelated/ignored writes, one-shot behavior and registration ownership.

**Acceptance checks:**

- Only count triggers the tracked registration.
- Unrelated/ignored writes do not trigger it.
- A second tracking scope observes the later write; callbacks capture synchronized state rather than actor-owned model access.

**Solution:** Prepend Model/ChangeLog setup. Run on MainActor under the declared baseline.

```swift
let model = Model()
let log = ChangeLog()
_ = withObservationTracking {
    model.count
} onChange: {
    log.record()
}
model.unrelated = 1
model.bookkeeping = 1
precondition(log.count() == 0)
model.count = 1
model.count = 2
precondition(log.count() == 1)
_ = withObservationTracking {
    model.count
} onChange: {
    log.record()
}
model.count = 3
precondition(log.count() == 2 && model.count == 3)
```

**Reasoning:** The model’s owner and the callback’s Sendable capture have separate safety mechanisms. The dependency expression registers reads; it does not promise every intermediate value as an event log or an ordering among observers.

## In practice

**Choose it when:** Observe current state with explicit identity, isolation and subscription ownership.

**Investigate by:** Trace every getter/cache-hit/external-write path and guard async publication by operation identity.

**Strong evidence:** Verify invalidation and teardown across synchronous/continuous/async APIs while keeping UI lifecycle and historical runtime limits explicit.

## References

### Related guides

- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A9 — Enums, Associated Values, Recursive Enums, and State Modeling]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B5 — Key paths and strongly typed indirection]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[B13 — Property Wrappers and Generated Storage Semantics]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F4 — Macros and compile-time code generation]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [SE-0395](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0395-observability.md)
- [Swift 6.4 release](https://www.swift.org/blog/swift-6.4-released/)
- [SE-0506 — Advanced observation tracking](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0506-advanced-observation-tracking.md)
- [Apple — Observation](https://developer.apple.com/documentation/observation)
- [Apple — Managing model data](https://developer.apple.com/documentation/swiftui/managing-model-data-in-your-app)
- [Apple — Bindable](https://developer.apple.com/documentation/swiftui/bindable)
- [State](https://developer.apple.com/documentation/swiftui/state)
- [SE-0258 — Property wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md)
