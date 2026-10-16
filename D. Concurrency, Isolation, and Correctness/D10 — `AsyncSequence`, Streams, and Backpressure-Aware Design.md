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

**Goal:** Finish a three-event stream and define an explicit owner for early-exit cleanup.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

AsyncSequence models repeated future values through an iterator whose next can suspend. AsyncStream adapts callback-style production and buffers according to policy. Consumer pull syntax does not necessarily slow the external source: yield is synchronous, and buffering/drop behavior is part of correctness. A single async function instead models one outcome. [AsyncStream](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md)

## Rules and pitfalls

### Match loss policy to the domain

A latest-value UI signal can tolerate dropping obsolete updates; an audit log cannot. Unbounded buffering risks memory growth, while bounded oldest/newest policies intentionally lose values. Inspect yield results and choose an actual backpressure mechanism when producers must wait.

**Common pitfall:** Treating AsyncStream as a lossless async array ignores external source rate and buffer limits. State whether every event, the newest state, or a sampled value is required.

### Own the subscription lifetime

Normal finish, failure, cancellation, consumer break, and owner release are different events. Provide an explicit close/unregister path when source resources must end. Do not assume one stream broadcasts independently to many consumers.

**Common pitfall:** Breaking a loop while retaining the stream and continuation need not trigger onTermination. The subscription owner must close the source when the consumer stops early; see [[F6 — Observation, property wrappers, and language-adjacent state tools]].

## Interview questions

### When should an API return `AsyncSequence` instead of a callback or delegate?

Use AsyncSequence for an ordered series of future elements with a defined completion/failure contract and a lifetime the consumer can own. A callback/delegate may remain appropriate for existing framework integration, synchronous hooks, or multiple coordinated operations. Use one async result for one completion.

### What cancellation and resource-lifetime issues arise with `AsyncStream`?

The producer may retain subscriptions, callbacks, buffers, or tasks after the consumer stops. Connect termination and explicit close to resource teardown, avoid capture cycles, use transferable payloads, and define whether cancellation interrupts a suspended next or merely requests source shutdown.

### What happens to onTermination when the consumer breaks but both stream and continuation remain alive?

A bare break does not guarantee onTermination in that retained configuration. The iterator stopped asking, but the source and continuation can remain alive. Explicitly finish/cancel/unregister through an owned subscription rather than using callback timing as a release guarantee.

### Can a channel send return without successful delivery? Who releases a producer after early consumer exit?

A channel send may resume due to cancellation or closure rather than successful delivery, depending on its contract. Handoff also does not mean processing completed. An explicit owner must close the channel or cancel/join the producer after early consumer exit; otherwise a producer can remain suspended forever.

## Focused exercise

**Task:** Return three events and explicitly finish a stream. Check normal completion and define who closes the source after early consumer exit.

**Acceptance checks:**

- Normal iteration yields three events in source order and completes.
- The early-exit owner explicitly finishes the retained source.
- A yield after finish reports termination rather than being treated as delivery.

**Solution:**

Standalone finite stream. The finite source owns production and always finishes:

```swift
func events() -> AsyncStream<Int> {
    let pair = AsyncStream<Int>.makeStream()
    for value in 1...3 { pair.continuation.yield(value) }
    pair.continuation.finish()
    return pair.stream
}
@main struct Probe {
    static func main() async {
        var collected: [Int] = []
        for await value in events() { collected.append(value) }
        precondition(collected == [1, 2, 3])
        let pair = AsyncStream<Int>.makeStream()
        pair.continuation.yield(1)
        for await _ in pair.stream { break }
        pair.continuation.finish()
        let result = pair.continuation.yield(2)
        if case .terminated = result { } else { preconditionFailure("source still open") }
    }
}
```

**Reasoning:** This bounded finite production uses default buffering and finishes before returning. It proves ordering and completion for three values, not sustainable backpressure. A real callback source's close must also unregister its external resource, and any producer task must be joined.

## In practice

**Choose it when:** Match event/state semantics, buffering, loss, and lifetime to the source.

**Investigate by:** Test normal finish, retained early exit, cancellation, yield results, and producer cleanup.

**Strong evidence:** Specify whether send completion means handoff or processed acknowledgment and verify resource release.

### Yield and backpressure policies — optional

#### `AsyncStream` is not always enough for real backpressure

`AsyncStream` can buffer and report `YieldResult`, but it does not naturally make synchronous producers suspend until the consumer catches up.

For true producer/consumer backpressure between tasks, `AsyncAlgorithms.AsyncChannel` is often a better fit:

```swift
import AsyncAlgorithms

func transfer() async {
    let channel = AsyncChannel<Int>()
    await withTaskGroup(of: Void.self) { group in
        group.addTask {
            defer { channel.finish() }
            for value in 0..<3 {
                guard !Task.isCancelled else { return }
                await channel.send(value)
            }
        }
        for await value in channel {
            print(value)
            // An early break still reaches the cleanup below.
        }
        channel.finish() // release a producer waiting in send
        group.cancelAll()
    }
}
```

AsyncChannel.send suspends for handoff, not completion of arbitrary processing in the loop body. A send can also return because of cancellation or channel termination; returning is not a delivery acknowledgment. The structured group above owns the producer, and explicit finish/cancel releases it on early consumer exit. This dependency is not part of the Swift standard library. The Swift Async Algorithms documentation describes `AsyncChannel` as a communication type where backpressure from `send(_:)` prevents production from exceeding consumption. ([GitHub, "Channel"](https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Channel.md)) The implementation comments also describe `send(_:)` as suspending after enqueueing until the next iterator call or finish. ([GitHub, "AsyncChannel.swift"](https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/Channels/AsyncChannel.swift))

#### Yield results and early exit

```swift
func bufferProbe() async {
    let pair = AsyncStream<Int>.makeStream(bufferingPolicy: .bufferingNewest(1))
    pair.continuation.yield(1)
    if case .dropped(let dropped) = pair.continuation.yield(2) {
        print("dropped", dropped)
    }
    pair.continuation.finish()
    for await value in pair.stream { print(value) }
    if case .terminated = pair.continuation.yield(3) { print("terminated") }
}
```

This prints dropped 1, then 2, then terminated. bufferingOldest(1) instead rejects the newly offered 2 and retains 1. A zero-capacity stream has no buffer: an offer without a waiting consumer is dropped. finish is idempotent and does not convert a failed yield into a delivered event.

#### Observation is state tracking, not a message log

Swift 6.4 adds configurable/continuous Observation tracking (SE-0506). Choose tracking for dependencies on observable state and an AsyncSequence for a domain's values over time. Neither label establishes lossless event delivery, multicast, or backpressure. Define update timing and token lifetime; F6 verifies the shipped withContinuousObservation spelling, Apple OS 27 requirement, and event-lifecycle differences from the proposal. Do not replace a protocol's ordered event stream with coalesced UI-state observation. [SE-0506](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0506-advanced-observation-tracking.md)

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
