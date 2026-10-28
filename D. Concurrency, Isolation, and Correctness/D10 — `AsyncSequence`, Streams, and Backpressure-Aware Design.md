---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - asyncsequence
  - asyncstream
---

# D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design

**Learning goals:**

- Choose one async result or a sequence of events and define completion, failure, and payload ownership.
- Explain buffering, drops, backpressure, and the difference between handoff and processed acknowledgment.
- Own normal completion, cancellation, and early-exit resource cleanup explicitly.

**Prerequisites:** [[D1 — Structured Concurrency Fundamentals]]–[[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]] for tasks and joining; [[D9 — Continuations and Bridging Callback APIs]] for one-result callbacks. Stream examples use only the standard library; channel examples name their separate dependency.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

One async function returns one eventual outcome. An **AsyncSequence** produces values over time through an iterator whose `next()` can suspend and returns nil at completion. `for await` expresses that repeated demand.

`AsyncStream` bridges an external callback producer into this interface. Its `yield` is synchronous: it can enqueue or drop a value without slowing the producer. **Backpressure** means production actually waits for downstream capacity. Buffering policy, delivery policy, and resource lifetime therefore remain part of correctness. An audit log and a latest-temperature display need different loss policies. [SE-0314](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Finishing makes a finite stream complete

**Setup:** Standalone runnable stream.

```swift
func events() -> AsyncStream<Int> {
    let pair = AsyncStream<Int>.makeStream()
    for value in 1...3 { pair.continuation.yield(value) }
    pair.continuation.finish()
    return pair.stream
}
var values: [Int] = []
for await value in events() { values.append(value) }
precondition(values == [1, 2, 3])
```

**Walkthrough:** The three buffered values are consumed in source order, then `finish()` allows next to return nil. Without finish, a retained open producer can leave a consumer waiting for another event forever.

### Shared setup — subscription owner

This fixture records resource close exactly once. In a real callback adapter, closing the registration would unregister a delegate, invalidate a timer, or cancel the external source. The termination closure captures the registration rather than the subscription, avoiding a subscription → continuation → subscription cycle. Prepend **subscription owner** where named.

```swift
import Synchronization
final class Registration: Sendable {
    private let state = Mutex(0)
    func close() { state.withLock { if $0 == 0 { $0 = 1 } } }
    var closeCount: Int { state.withLock { $0 } }
}
final class Subscription: Sendable {
    let stream: AsyncStream<Int>
    let registration: Registration
    private let continuation: AsyncStream<Int>.Continuation
    init() {
        let pair = AsyncStream<Int>.makeStream()
        let registration = Registration()
        self.stream = pair.stream
        self.continuation = pair.continuation
        self.registration = registration
        pair.continuation.onTermination = { _ in registration.close() }
    }
    func emit(_ value: Int) { continuation.yield(value) }
    func close() { registration.close(); continuation.finish() }
}
```

**Walkthrough:** Normal finish and consumer cancellation can reach termination cleanup. Explicit close additionally gives the owner a reliable shutdown path after a retained early break. Registration close is idempotent so these paths can converge.

### A throwing stream ends with an explicit failure

**Setup:** Standalone runnable stream with one buffered value and a terminal error.

```swift
enum StreamFailure: Error { case disconnected }
let stream = AsyncThrowingStream<Int, Error> { continuation in
    continuation.yield(1)
    continuation.finish(throwing: StreamFailure.disconnected)
}
var received: [Int] = []
do {
    for try await value in stream { received.append(value) }
    preconditionFailure("expected a disconnected source")
} catch StreamFailure.disconnected { }
precondition(received == [1])
```

**Walkthrough:** The consumer drains the buffered value, then observes the failure rather than ordinary nil completion. A real source must route each terminal path into finish and unregister its external resources; repeated finish calls are idempotent.

## Rules and pitfalls

### A buffer limit is a loss policy, not automatic backpressure

**Common pitfall:** A stream is treated as a lossless async array while an external producer can outrun consumption.

**Mistake:** Standalone **compile-only** producer; default buffering is unbounded.

```swift
func readings(_ input: [Int]) -> AsyncStream<Int> {
    let pair = AsyncStream<Int>.makeStream()
    for value in input { pair.continuation.yield(value) }
    pair.continuation.finish()
    return pair.stream
}
```

**Consequence:** A large or indefinitely fast source can retain an unacceptable backlog. Synchronous yield never waits for the consumer to catch up.

**Why:** Consumer pull syntax does not impose demand control on an arbitrary callback source.

**Correction:** For a latest-state display, explicitly keep only the newest buffered reading. Standalone runnable policy demonstration.

```swift
let pair = AsyncStream<Int>.makeStream(bufferingPolicy: .bufferingNewest(1))
pair.continuation.yield(1)
let offered = pair.continuation.yield(2)
if case .dropped(let old) = offered { precondition(old == 1) }
else { preconditionFailure("expected a displaced reading") }
pair.continuation.finish()
var values: [Int] = []
for await value in pair.stream { values.append(value) }
precondition(values == [2])
```

**What changes:** The backlog is bounded by intentionally discarding an obsolete state. This is inappropriate for events that must all survive. `bufferingOldest(1)` would retain 1 and reject the newly offered 2; zero capacity drops an offer when no consumer is waiting. Use source throttling or a real backpressure mechanism for lossless work.

### Breaking iteration does not necessarily close a retained source

**Common pitfall:** A bare break is expected to unregister the source automatically.

**Mistake:** Prepend **subscription owner**. Runnable retained-subscription example.

```swift
let subscription = Subscription()
subscription.emit(1)
for await _ in subscription.stream { break }
precondition(subscription.registration.closeCount == 0)
subscription.close() // driver cleanup, absent from the mistaken owner policy
```

**Consequence:** The retained stream/source can remain open after the loop stops. The last line is necessary test cleanup, not an effect of break.

**Why:** Stopping this iterator's demand is not the same operation as finishing or unregistering the retained producer.

**Correction:** Prepend **subscription owner**. Runnable explicit lifetime boundary.

```swift
let subscription = Subscription()
subscription.emit(1)
do {
    defer { subscription.close() }
    for await _ in subscription.stream { break }
}
precondition(subscription.registration.closeCount == 1)
```

**What changes:** Leaving the owner scope explicitly closes the registration even on early exit. Any owned producer task also needs cancellation and joining; closing a continuation is not a substitute for observing that task's completion.

## Worked example

### Buffering results and terminal state are observable

**Setup:** Standalone runnable script.

**Predict:** What remains after two offers into one oldest-value slot, and what does yielding after finish mean?

```swift
let pair = AsyncStream<Int>.makeStream(bufferingPolicy: .bufferingOldest(1))
pair.continuation.yield(1)
if case .dropped(let rejected) = pair.continuation.yield(2) {
    precondition(rejected == 2)
} else { preconditionFailure("expected a rejected new value") }
pair.continuation.finish()
var received: [Int] = []
for await value in pair.stream { received.append(value) }
precondition(received == [1])
if case .terminated = pair.continuation.yield(3) { }
else { preconditionFailure("finished source accepted work") }
```

**Outcome:** One is consumed, two is dropped, and the post-finish offer is terminated.

**Why:** BufferingOldest protects the already-enqueued value. Finish closes further production but allows buffered values to drain. Neither an enqueued offer nor a successful loop iteration proves downstream business processing was durably acknowledged.

## Interview questions

### When should an API return AsyncSequence instead of a callback or delegate?

**Answer:** Use it for repeated future values with a defined ordering, completion/failure contract, and consumer-owned lifetime. One result belongs in an async function; existing synchronous framework hooks may still need callbacks.

**Example:** Standalone runnable event sequence.

```swift
func statusChanges() -> AsyncStream<String> {
    let pair = AsyncStream<String>.makeStream()
    pair.continuation.yield("connecting")
    pair.continuation.yield("ready")
    pair.continuation.finish()
    return pair.stream
}
var states: [String] = []
for await state in statusChanges() { states.append(state) }
precondition(states == ["connecting", "ready"])
```

**Walkthrough:** Two domain events become two elements, and finish ends iteration. Wrapping both events in a single-result continuation would either drop one or misuse its completion handle.

**Boundary:** A sequence is not automatically multicast. Coalesced Observation state tracking is not an ordered event log; continuous tracking and its OS 27 API are covered in F6. Choose the domain semantics before choosing the adapter.

### What cancellation and resource-lifetime issues arise with AsyncStream?

**Answer:** A source can retain callbacks, buffers, registrations, and producer tasks after the consumer stops. Connect termination and explicit owner shutdown to resource teardown without capture cycles.

**Example:** Prepend **subscription owner**. Runnable cancellation path.

```swift
let subscription = Subscription()
let consumer = Task {
    for await _ in subscription.stream { }
}
consumer.cancel()
await consumer.value
precondition(subscription.registration.closeCount == 1)
subscription.close() // idempotent owner shutdown
```

**Walkthrough:** The cancelled consumer's iteration terminates and its termination callback closes the registration. The driver joins the consumer; explicit close remains valid even when termination already performed cleanup.

**Boundary:** This fixture's unregister is synchronous and idempotent. Real native teardown may need a separate completion acknowledgment. Payloads crossing tasks also need an appropriate Sendable/transfer contract.

### What happens to onTermination when the consumer breaks but stream and continuation remain alive?

**Answer:** A bare break does not guarantee termination in that retained configuration. Explicitly close through the owner when the source must end.

**Example:** Standalone runnable retained-pair check on the selected runtime.

```swift
import Synchronization
let terminations = Mutex(0)
let pair = AsyncStream<Int>.makeStream()
pair.continuation.onTermination = { _ in terminations.withLock { $0 += 1 } }
pair.continuation.yield(1)
for await _ in pair.stream { break }
precondition(terminations.withLock { $0 } == 0)
pair.continuation.finish()
precondition(terminations.withLock { $0 } == 1)
```

**Walkthrough:** Retaining the producer side keeps it available to offer more values even though this loop stopped. Finish supplies the explicit terminal event.

**Boundary:** Do not generalize this observation into a promise of no termination under every iterator/ownership arrangement. Define close/unregister explicitly instead of depending on destruction timing.

### Can a channel send return without successful delivery? Who releases the producer after early consumer exit?

**Answer:** Yes. AsyncChannel send can resume because of cancellation or channel finish, so return alone is not proof of delivery. The owner must finish the channel or cancel and join its producer after early exit.

#### Channel package setup — external dependency

AsyncChannel belongs to Swift Async Algorithms, not the standard library. For the following two independent channel examples, create an executable package with this Package.swift, put one example at a time in `Sources/ChannelStudy/main.swift`, and run `swift run ChannelStudy`. The pinned dependency is deliberately explicit. The package's client settings match this repository's baseline.

```swift
// swift-tools-version: 6.4
import PackageDescription
let package = Package(
    name: "ChannelStudy",
    platforms: [.macOS(.v15)],
    dependencies: [.package(url: "https://github.com/apple/swift-async-algorithms.git", exact: "1.0.4")],
    targets: [.executableTarget(name: "ChannelStudy", dependencies: [
        .product(name: "AsyncAlgorithms", package: "swift-async-algorithms")
    ], swiftSettings: [
        .swiftLanguageMode(.v6), .defaultIsolation(nil),
        .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
        .enableUpcomingFeature("InferIsolatedConformances")
    ])]
)
```

**Example:** Runnable with the **channel package setup**. This channel is already closed.

```swift
import AsyncAlgorithms
let channel = AsyncChannel<Int>()
channel.finish()
await channel.send(7)
var received: [Int] = []
for await value in channel { received.append(value) }
precondition(received.isEmpty)
```

**Walkthrough:** Send returns but no value is delivered. The terminal state is enough to release pending or subsequent send operations.

This second independent runnable package example owns the producer through a group:

```swift
import AsyncAlgorithms
let channel = AsyncChannel<Int>()
await withTaskGroup(of: Void.self) { group in
    group.addTask {
        defer { channel.finish() }
        for value in 1...3 {
            guard !Task.isCancelled else { return }
            await channel.send(value)
        }
    }
    for await value in channel {
        precondition(value == 1)
        break
    }
    channel.finish()
    group.cancelAll()
}
```

The owner explicitly releases a producer potentially suspended in send, then the group joins it. The producer does not need to finish sending all three values for the early-exit contract.

**Boundary:** Channel backpressure concerns handoff/iteration, not completion of arbitrary work in the loop body. Use a separate acknowledgment if durable processing matters. Source implementation and cancellation rules, not the name channel, establish the contract.

## Focused exercise

**Task:** Produce three ordered events and finish. Then retain a subscription, exit early, and explicitly close it.

**Acceptance checks:**

- Normal iteration receives exactly three values and completes.
- Early exit has an explicit resource owner and closes once.
- A post-finish offer reports termination.

**Solution:** Prepend **subscription owner**. Runnable.

```swift
let complete = Subscription()
for value in 1...3 { complete.emit(value) }
complete.close()
var values: [Int] = []
for await value in complete.stream { values.append(value) }
precondition(values == [1, 2, 3] && complete.registration.closeCount == 1)
let early = Subscription()
early.emit(1)
for await _ in early.stream { break }
early.close()
precondition(early.registration.closeCount == 1)
let pair = AsyncStream<Int>.makeStream()
pair.continuation.finish()
if case .terminated = pair.continuation.yield(4) { }
else { preconditionFailure("expected termination") }
```

**Reasoning:** The finite source finishes independently of how quickly its values are consumed. The early-exit owner makes shutdown explicit. Neither fixture proves sustainable throughput, live-source teardown, or processing acknowledgment.

## In practice

**Choose it when:** Match event versus state semantics, loss, buffering, and lifetime to the source.

**Investigate by:** Test normal finish, failure, cancellation, retained early exit, and post-finish yields separately.

**Strong evidence:** Name the resource owner and distinguish enqueued, handed off, processed, and acknowledged events.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [AsyncStream](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md)
- [GitHub, "Channel"](https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Channel.md)
- [GitHub, "AsyncChannel.swift"](https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/Channels/AsyncChannel.swift)
- [SE-0506](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0506-advanced-observation-tracking.md)
- [GitHub. "AsyncSequence." Swift Evolution SE-0298](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0298-asyncsequence.md)
- [Apple Developer. "AsyncStream." Apple Developer Documentation](https://developer.apple.com/documentation/swift/asyncstream)
