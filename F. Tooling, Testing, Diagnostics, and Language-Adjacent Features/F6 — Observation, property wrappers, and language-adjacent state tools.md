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

**Goal:** Track one observable property and distinguish tracked, unrelated, ignored, and re-registered changes.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Observation records property reads executed inside a tracking scope. Dependencies follow actual branches and nested accesses, not a computed property's name. @Observable generates registrar/accessor/storage support; it does not add synchronization, persistence, view identity, or lossless event delivery. UI-owned models still need explicit MainActor under this collection's nonisolated defaults. [SE-0395](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0395-observability.md)

## Rules and pitfalls

### Keep dependency reads on every valid path

An ignored cache can hide the source property's read after the first computation. Cache hits must still register the dependency, and external storage needs explicit access/mutation instrumentation where automatic backing storage is absent.

**Common pitfall:** The first read works while later cache-hit scopes observe no source read and fail to invalidate. Reestablish the source dependency on every path or move the cache behind an appropriately instrumented boundary.

### Separate model identity, binding, and isolation

State retains a SwiftUI model according to view identity; Bindable creates projections and can retain its model. Neither wrapper makes arbitrary state thread-safe. Keep callback/task/token lifetime explicit and use a unique request identity for asynchronous publication.

**Common pitfall:** Cancelling an old search and comparing query text can still permit a repeated-query stale result or error. Guard success, error, and cleanup by the current operation identity.

## Interview questions

### Why can an ignored cache work on its first read but fail to invalidate after a cache-hit read?

The first getter call reads the tracked source while filling its ignored cache. A cache-hit branch can return without reading that source, so the new tracking scope has no dependency to invalidate. Register or read the source on hits too, and invalidate cached data consistently when it changes.

### Which actor runs a synchronous change callback versus a continuous callback, and what owns the observation's lifetime?

A synchronous change callback runs in the mutation's execution context, but its Sendable type does not grant actor-isolated access to the model. A continuous callback uses its inherited registration isolation and is scheduled under that API's contract. The subscription owner retains and cancels/releases its token or task; callback delivery alone is not lifecycle ownership.

### Why can cancelling the older task and comparing query text still publish a stale search result?

Cancellation is cooperative and query strings can repeat across distinct operations. An older service can return or throw after a newer identical query began. A unique operation identity must authorize each publication and cleanup; MainActor serializes access without selecting the intended winner.

## Focused exercise

**Task:** Track one observable property. Check unrelated and ignored writes, explain one-shot registration, and state the model's required isolation.

**Acceptance checks:**

- Only the actually read count triggers the original registration.
- Unrelated and ignored writes do not trigger it.
- The original callback is one-shot; a new scope observes a later change without reading actor state in the callback.

**Solution:**

Standalone executable: Observation requires macOS 14 / iOS 17; this synchronized callback counter additionally requires macOS 15 / iOS 18. The model is explicitly UI-owned.

```swift
import Observation
import Synchronization
@available(macOS 15, iOS 18, *)
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
@main struct Probe {
    @MainActor static func main() {
        if #available(macOS 15, iOS 18, *) {
            let model = Model()
            let log = ChangeLog()
            _ = withObservationTracking { model.count } onChange: { log.record() }
            model.unrelated = 1
            model.bookkeeping = 1
            precondition(log.count() == 0)
            model.count = 1
            model.count = 2
            precondition(log.count() == 1)
            _ = withObservationTracking { model.count } onChange: { log.record() }
            model.count = 3
            precondition(log.count() == 2)
        }
    }
}
```

**Reasoning:** The log has its own synchronization, while Model has MainActor ownership. This separates callback capture safety from observation dependency tracking. The newer options/events/tokens require OS 27 in this SDK; Observations requires OS 26. They do not turn state invalidation into an ordered event log.

## In practice

**Choose it when:** Observe current state through a model with explicit identity, lifetime, and isolation.

**Investigate by:** Trace actual getter reads, cache-hit paths, token ownership, and request publication.

**Strong evidence:** Validate dependency coverage and teardown across synchronous, continuous, and async surfaces without assuming observation supplies synchronization.

### Cached and external storage; advanced observation — optional

#### Cached getters must register dependencies on a cache hit

The old cache example returned ignored cached storage before reading `posts`. A tracking scope entered after the cache warmed therefore had no dependency on the source array. Fix that path too:

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
        let source = posts // Register the dependency even on a cache hit.
        if let cachedSortedPosts { return cachedSortedPosts }
        let result = source.sorted {
            $0.rank == $1.rank ? $0.id < $1.id : $0.rank > $1.rank
        }
        cachedSortedPosts = result
        return result
    }
}
```

This cache is main-actor-confined and contains value-type posts. Mutating nested reference objects would require a separate invalidation design. Another reasonable design computes a stored `sortedPosts` when inputs change. Neither approach proves a performance improvement without measurement.

#### External storage requires an observation boundary

Forwarding a computed property to an unobservable store does not automatically register that property. This in-memory adapter manually instruments reads and writes:

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
```

All updates in this sample enter through the adapter. It is not a `UserDefaults` observer, disk store, or cross-process notification bridge. A real external writer must notify this boundary; merely adding `@ObservationIgnored` to a dependency does not supply notifications. Likewise, ignored caches are safe only when their public results still depend on correctly tracked inputs.

#### Synchronous advanced events and matching

The model below is explicitly nonisolated, confined to this synchronous function, and never sent to the callback. The callback captures only a synchronized log and reads event metadata. It demonstrates matching without claiming that an actor-isolated property's key path can be formed in an arbitrary callback.

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
```

The checked ordinary result is `["will:true:false", "did:true:false"]`; cancelling in the first event yields only `["will:true:false"]`. The second assignment does not cause this dependency expression to be re-observed. A separate timing probe confirms old/new stored values for the two phases. The options overload also passes a focused check returning a noncopyable value and propagating a typed failure. Do not infer the ordering of different observers from one run.

`Event.Kind` is a struct with known static values, not an exhaustive four-case enum. Handle unknown kinds when branching. For deallocation events, `matches` does not identify a changed property. Do not capture an object strongly merely to observe its eventual destruction, or assume a deallocation notification proves every destructor side effect has completed.

**Runtime/source discrepancy:** In the checked distribution, a combined `[.willSet, .didSet, .deinit]` registration emits the first mutation pair and then no deallocation callback. A separate deinit-only registration does fire. SE-0506 illustrates a later deallocation callback after that pair; the local result differs. Do not make resource cleanup depend on receiving all three events from one one-shot registration.

#### Continuous callback adapter

This fence reuses `Model` and targets Apple OS 27. The initial callback is scheduled; construction does not synchronously render. Later callbacks execute on the inherited main actor and read current state.

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

The owner retains `CounterBinding` for the desired lifetime. `stop()` drops its token; token destruction unregisters the observation. A directly held token can instead be consumed with `token.cancel()`. The weak model capture prevents this adapter from prolonging the model's life, but a `render` closure that strongly captures the binding's owner can still form a cycle. Already executed UI work is not undone by cancellation.

For async snapshot consumers, the following earlier API is often sufficient. It also reuses `Model`; it requires Apple OS 26:

```swift
import Observation

@available(anyAppleOS 26.0, *)
@MainActor
func counterValues(_ model: Model) -> Observations<Int, Never> {
    Observations { model.count }
}
```

Consume it in a task whose lifetime you own, and cancel that task when observation should end. This closure strongly captures `model` for the sequence's lifetime. Validation uses explicit initial/update handshakes to check coalescing; a fixed number of `Task.yield()` calls is not a reliable synchronization test. Async sequence cancellation/termination is covered in D10.

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
