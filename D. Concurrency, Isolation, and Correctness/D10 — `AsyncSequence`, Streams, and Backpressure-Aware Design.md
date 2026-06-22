---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - asyncsequence
  - asyncstream
---
## 0. Rubric snapshot

**Rubric expectation**

Understand when a single async result should become an `AsyncSequence`, and how streams model event sources. The rubric specifically probes `AsyncSequence`, streams, cancellation, resource lifetime, and a stream that never finishes.

**Caveats**

`AsyncStream` is useful, but it is not magic. It can hide:

- missing termination
- unbounded buffering
- cancellation leaks
- producer lifetime bugs
- weak or absent backpressure

**You should be able to answer**

- When should an API return `AsyncSequence` instead of a callback or delegate?
- What cancellation and resource-lifetime issues arise with `AsyncStream`?

**You should be able to do**

- Model location updates or websocket messages as an async sequence.
- Explain why a consumer may hang forever if a stream never calls `finish()`.

---

## 1. Core mental model

`async` functions model **one future result**.

```swift
func loadUser() async throws -> User
```

`AsyncSequence` models **many future results over time**.

```swift
var messages: AsyncThrowingStream<Message, Error>
```

The important shift is this:

```text
async function = one result later
AsyncSequence = repeated awaits until termination
```

A consumer pulls values by repeatedly awaiting `next()` through `for await`. Each iteration may suspend. The sequence ends only when its iterator returns `nil` or throws. SE-0298 introduced `AsyncSequence` to make asynchronous iteration over many values feel like ordinary `for-in`, but with `await` marking suspension. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0298-asyncsequence.md "swift-evolution/proposals/0298-asyncsequence.md at main · swiftlang/swift-evolution · GitHub"))

`AsyncStream` and `AsyncThrowingStream` are convenience root sequences. They let you bridge callback/delegate-style APIs into Swift concurrency without manually writing an `AsyncIteratorProtocol`. Their continuation is the producer side; the stream is the consumer side. SE-0314 describes them as a bridge from non-`async/await` asynchronous behavior into async contexts, especially for delegate or multi-callback APIs. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md "swift-evolution/proposals/0314-async-stream.md at main · swiftlang/swift-evolution · GitHub"))

The trap is that `AsyncStream` is **buffered push-to-pull adaptation**, not automatically a fully backpressured pipeline. A fast producer may keep yielding while a slow consumer is not keeping up. Buffering policy decides what happens to excess elements, but it does not necessarily slow the upstream producer.

---

## 2. Essential mechanics

### 2.1 `AsyncSequence` is pull-based at the consumer boundary

A consumer asks for the next element:

```swift
for await value in values {
    print(value)
}
```

Conceptually:

```text
consumer calls next()
↓
next() suspends until value / nil / error
↓
loop body runs
↓
consumer calls next() again
```

This is different from a delegate, where the producer calls you whenever it wants.

### 2.2 `AsyncStream` has a producer-side continuation

```swift
func numbers() -> AsyncStream<Int> {
    AsyncStream { continuation in
        continuation.yield(1)
        continuation.yield(2)
        continuation.finish()
    }
}
```

`yield(_:)` sends a value. `finish()` terminates the stream. Without `finish()`, a consumer that tries to keep iterating may suspend forever after the last yielded value.

Apple’s documentation describes `AsyncStream` as conforming to `AsyncSequence` and being suited for adapting callback/delegate APIs to `async/await`. ([Apple Developer](https://developer.apple.com/documentation/swift/asyncstream?utm_source=chatgpt.com "AsyncStream | Apple Developer Documentation")) The continuation is used to yield values and then terminate the stream normally with `finish()`. ([Apple Developer](https://developer.apple.com/documentation/swift/asyncstream/continuation?utm_source=chatgpt.com "AsyncStream.Continuation | Apple Developer Documentation"))

### 2.3 Buffering policy is part of correctness

Default unbounded buffering is dangerous for high-frequency sources.

```swift
let stream = AsyncStream<Int>(
    bufferingPolicy: .bufferingNewest(1)
) { continuation in
    continuation.yield(1)
    continuation.yield(2)
    continuation.yield(3)
    continuation.finish()
}
```

Important policies:

```text
.unbounded
.bufferingOldest(n)   // keep oldest buffered values; drop new overflow
.bufferingNewest(n)   // keep newest buffered values; drop old overflow
```

Apple documents `bufferingNewest(_:)` as keeping at most the specified number of newest values and discarding older values when the buffer is full. ([Apple Developer](https://developer.apple.com/documentation/swift/asyncstream/continuation/bufferingpolicy/bufferingnewest%28_%3A%29?utm_source=chatgpt.com "AsyncStream.Continuation.BufferingPolicy. ..."))

For UI state, `.bufferingNewest(1)` is often right: stale states are worthless. For audit logs, telemetry, payment events, or ordered network messages, dropping values is usually wrong.

### 2.4 Termination is not optional

A finite stream must finish:

```swift
func finiteStream() -> AsyncStream<Int> {
    AsyncStream { continuation in
        for i in 0..<3 {
            continuation.yield(i)
        }
        continuation.finish()
    }
}
```

A non-finite stream still needs cleanup:

```swift
func notifications(
    named name: Notification.Name
) -> AsyncStream<Notification> {
    AsyncStream { continuation in
        let token = NotificationCenter.default.addObserver(
            forName: name,
            object: nil,
            queue: nil
        ) { notification in
            continuation.yield(notification)
        }

        continuation.onTermination = { @Sendable _ in
            NotificationCenter.default.removeObserver(token)
        }
    }
}
```

`onTermination` is where you stop observation, cancel tasks, close sockets, invalidate timers, or release delegate ownership. Apple’s `onTermination` docs state that task cancellation during stream iteration invokes the callback, allowing cleanup. ([Apple Developer](https://developer.apple.com/documentation/swift/asyncthrowingstream/continuation/ontermination?utm_source=chatgpt.com "onTermination | Apple Developer Documentation"))

### 2.5 `AsyncStream` is not always enough for real backpressure

`AsyncStream` can buffer and report `YieldResult`, but it does not naturally make synchronous producers suspend until the consumer catches up.

For true producer/consumer backpressure between tasks, `AsyncAlgorithms.AsyncChannel` is often a better fit:

```swift
import AsyncAlgorithms

let channel = AsyncChannel<Int>()

Task {
    for i in 0..<100 {
        await channel.send(i)
    }
    channel.finish()
}

for await value in channel {
    print(value)
}
```

`AsyncChannel.send(_:)` suspends until consumption progresses, so production cannot outrun consumption in the same way. The Swift Async Algorithms documentation describes `AsyncChannel` as a communication type where backpressure from `send(_:)` prevents production from exceeding consumption. ([GitHub](https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Channel.md "swift-async-algorithms/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Channel.md at main · apple/swift-async-algorithms · GitHub")) The implementation comments also describe `send(_:)` as suspending after enqueueing until the next iterator call or finish. ([GitHub](https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/Channels/AsyncChannel.swift "swift-async-algorithms/Sources/AsyncAlgorithms/Channels/AsyncChannel.swift at main · apple/swift-async-algorithms · GitHub"))

---

## 3. Common traps and misconceptions

### Trap 1: Treating `AsyncStream` as “async array”

Bad:

```swift
func makeStream() -> AsyncStream<Int> {
    AsyncStream { continuation in
        for i in 0..<3 {
            continuation.yield(i)
        }
        // Missing finish()
    }
}
```

Better:

```swift
func makeStream() -> AsyncStream<Int> {
    AsyncStream { continuation in
        for i in 0..<3 {
            continuation.yield(i)
        }
        continuation.finish()
    }
}
```

An async stream has a lifetime. If you do not finish a finite stream, the consumer cannot know that there are no more values.

### Trap 2: Using unbounded buffering for high-frequency events

Bad:

```swift
func sensorValues() -> AsyncStream<Double> {
    AsyncStream { continuation in
        sensor.onValue = { value in
            continuation.yield(value)
        }
    }
}
```

This can accumulate memory if the producer emits faster than the consumer processes.

Better:

```swift
func latestSensorValue() -> AsyncStream<Double> {
    AsyncStream(bufferingPolicy: .bufferingNewest(1)) { continuation in
        sensor.onValue = { value in
            continuation.yield(value)
        }

        continuation.onTermination = { @Sendable _ in
            sensor.stop()
        }

        sensor.start()
    }
}
```

For “latest state wins” streams, bounded buffering is usually the right design.

### Trap 3: Forgetting that stream creation can start resources

This is subtle:

```swift
let stream = websocketMessages()
// socket may already be created/started depending on implementation
```

If the `AsyncStream` builder starts a monitor, opens a socket, registers a delegate, or starts a timer, merely creating the stream may consume resources. SE-0314’s example starts monitoring inside the stream builder and uses `onTermination` to stop monitoring. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md "swift-evolution/proposals/0314-async-stream.md at main · swiftlang/swift-evolution · GitHub"))

### Trap 4: Assuming one stream means broadcast semantics

`AsyncSequence` itself does not promise multicast behavior. Some sequences are single-consumer, some support multiple iterators, and behavior can be surprising. The Async Algorithms channel proposal explicitly notes that `AsyncSequence` itself makes no assumption about whether multiple consumers are supported. ([GitHub](https://github.com/apple/swift-async-algorithms/blob/main/Evolution/0016-mutli-producer-single-consumer-channel.md "swift-async-algorithms/Evolution/0016-mutli-producer-single-consumer-channel.md at main · apple/swift-async-algorithms · GitHub"))

For app code, document this:

```swift
/// Returns a unicast stream. Create one stream per consumer.
func messages() -> AsyncThrowingStream<Message, Error>
```

or expose a proper broadcast abstraction.

---

## 4. Direct answers to rubric questions

### Q1. When should an API return `AsyncSequence` instead of a callback or delegate?

Return `AsyncSequence` when the API produces **multiple values over time** and the consumer should process them with structured async control flow.

Good candidates:

```text
location updates
websocket messages
Bluetooth events
notifications
file lines
download progress
speech recognition events
audio buffers
UI action streams
```

Use a single `async` function when there is exactly one logical result:

```swift
func fetchProfile() async throws -> Profile
```

Use `AsyncSequence` when the caller needs to consume an ongoing stream:

```swift
func messages() -> AsyncThrowingStream<WebSocketMessage, Error>
```

Interview version:

> I return `AsyncSequence` when the operation is not “one result later” but “many values over time.” It gives the caller structured consumption with `for await`, cancellation through the consuming task, and composition through async sequence operators. I still need to design termination, buffering, backpressure, and cleanup explicitly; `AsyncStream` only gives me the bridge, not the whole lifecycle design.

### Q2. What cancellation and resource-lifetime issues arise with `AsyncStream`?

The stream producer may own resources: delegates, sockets, timers, observers, tasks, file handles, or hardware sessions. If the consumer cancels or breaks out of the loop, the producer must be stopped. Otherwise the app can leak work or keep producing values nobody consumes.

You usually handle this with `continuation.onTermination`:

```swift
continuation.onTermination = { @Sendable termination in
    task.cancel()
    socket.cancel()
    locationManager.stopUpdatingLocation()
}
```

Also, if the producer is finite, it must call `finish()`. If it can fail, use `AsyncThrowingStream` and call `finish(throwing:)`.

Interview version:

> The main issue is ownership. `AsyncStream` separates the consumer from the producer, so the producer may keep running after the consumer stops unless I wire cancellation back through `onTermination`. I also need to call `finish()` for finite streams, choose a buffering policy, and avoid unbounded memory growth. A stream API should make it clear who starts the resource, who stops it, and what happens when the consumer cancels.

---

## 5. Code probe

Given:

```swift
func makeStream() -> AsyncStream<Int> {
    AsyncStream { continuation in
        for i in 0..<3 {
            continuation.yield(i)
        }
    }
}
```

### What happens?

`makeStream()` itself compiles and prints nothing.

If consumed like this:

```swift
for await value in makeStream() {
    print(value)
}
print("done")
```

the visible output is:

```text
0
1
2
```

Then the task suspends forever waiting for another element. `"done"` is never printed.

### Why?

The producer yields three values but never terminates the stream.

```text
AsyncStream created
↓
yield(0)
yield(1)
yield(2)
↓
no finish()
↓
consumer receives 0, 1, 2
↓
consumer asks for next()
↓
next() suspends forever because stream is neither finished nor producing
```

For a stream, “I have no more values” must be represented explicitly by `finish()`.

### Fix or redesign

```swift
func makeStream() -> AsyncStream<Int> {
    AsyncStream { continuation in
        for i in 0..<3 {
            continuation.yield(i)
        }
        continuation.finish()
    }
}
```

Now:

```swift
for await value in makeStream() {
    print(value)
}
print("done")
```

prints:

```text
0
1
2
done
```

### Why this fix is correct

`finish()` tells the iterator that the sequence has reached its terminal state. After the buffered values are consumed, `next()` returns `nil`, and the `for await` loop exits.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`continuation.finish()`|Finite non-throwing streams|Caller cannot distinguish success from “no more values” beyond normal termination|
|`AsyncThrowingStream` + `finish(throwing:)`|Stream can fail|Caller must use `for try await`|
|`AsyncChannel`|Producer and consumer are async tasks and you need backpressure|Requires Swift Async Algorithms dependency|
|Plain `[Int]` or `some Sequence<Int>`|Values are already available synchronously|No async cancellation or suspension semantics|
|`async throws -> [Int]`|One batched result is the real domain model|Caller waits for all values before processing|

---

## 6. Exercise

### Problem

Model websocket messages as an async sequence and explain how consumers stop the stream safely.

### Bad / naive version

```swift
func messages(from task: URLSessionWebSocketTask)
    -> AsyncThrowingStream<URLSessionWebSocketTask.Message, Error>
{
    AsyncThrowingStream { continuation in
        Task {
            while true {
                let message = try await task.receive()
                continuation.yield(message)
            }
        }
    }
}
```

### What is wrong?

```text
1. The producer task is unstructured and not canceled when the consumer stops.
2. The websocket task is not canceled on termination.
3. Errors are not sent to the stream with finish(throwing:).
4. Normal termination is not modeled.
5. Buffering policy is implicit.
6. The loop may keep running after the caller no longer needs messages.
```

### Improved version

```swift
import Foundation

func messages(
    from webSocketTask: URLSessionWebSocketTask
) -> AsyncThrowingStream<URLSessionWebSocketTask.Message, Error> {
    AsyncThrowingStream(
        bufferingPolicy: .bufferingNewest(100)
    ) { continuation in
        let receiveTask = Task {
            do {
                while !Task.isCancelled {
                    let message = try await webSocketTask.receive()
                    continuation.yield(message)
                }

                continuation.finish()
            } catch {
                continuation.finish(throwing: error)
            }
        }

        continuation.onTermination = { @Sendable _ in
            receiveTask.cancel()
            webSocketTask.cancel(with: .goingAway, reason: nil)
        }

        webSocketTask.resume()
    }
}
```

Usage:

```swift
let webSocketTask = URLSession.shared.webSocketTask(with: url)

let consumer = Task {
    do {
        for try await message in messages(from: webSocketTask) {
            print(message)
        }
    } catch {
        print("WebSocket failed:", error)
    }
}

// Later:
consumer.cancel()
```

### Why this is better

The consumer owns cancellation through its task. When the task is canceled, stream termination runs cleanup:

```text
consumer task canceled
↓
for try await stops
↓
AsyncThrowingStream terminates
↓
onTermination runs
↓
receive task canceled
↓
websocket canceled
```

This design also makes failure explicit: `receive()` errors terminate the stream by throwing into the consumer’s `for try await` loop.

### Production refinement

In a real client, I would usually wrap this in a type that owns the socket lifetime instead of exposing a free function:

```swift
actor WebSocketClient {
    private let url: URL
    private var task: URLSessionWebSocketTask?

    init(url: URL) {
        self.url = url
    }

    func connect() -> AsyncThrowingStream<URLSessionWebSocketTask.Message, Error> {
        let webSocketTask = URLSession.shared.webSocketTask(with: url)
        task = webSocketTask

        return AsyncThrowingStream(
            bufferingPolicy: .bufferingNewest(100)
        ) { continuation in
            let receiveTask = Task {
                do {
                    webSocketTask.resume()

                    while !Task.isCancelled {
                        let message = try await webSocketTask.receive()
                        continuation.yield(message)
                    }

                    continuation.finish()
                } catch {
                    continuation.finish(throwing: error)
                }
            }

            continuation.onTermination = { @Sendable _ in
                receiveTask.cancel()
                webSocketTask.cancel(with: .goingAway, reason: nil)
            }
        }
    }
}
```

For a high-throughput or lossless protocol, `.bufferingNewest(100)` may be wrong. You may need no dropping, explicit acknowledgments, backpressure, or `AsyncChannel`.

---

## 7. Production guidance

Use `AsyncSequence` in production when:

```text
- the domain naturally emits many values over time
- consumers should use structured cancellation
- values can be processed incrementally
- composition with map/filter/prefix/debounce/merge is useful
- you are replacing callbacks, delegates, notification observers, or repeated polling
```

Be careful when:

```text
- the source emits faster than consumers process
- every value must be delivered
- multiple consumers need the same events
- the stream owns hardware/network resources
- the producer is started eagerly
- the stream never naturally completes
- cancellation needs to propagate upstream
```

Avoid `AsyncStream` when:

```text
- the operation has exactly one result
- a simple async function is enough
- you need strong producer backpressure
- you need durable delivery guarantees
- you need complex multicast/replay semantics
- you are using it to hide bad lifecycle ownership
```

Debugging checklist:

```text
Does the stream ever call finish() or finish(throwing:)?
Who owns the continuation?
Who owns the producer resource?
What happens when the consuming task is canceled?
Is onTermination installed before production starts?
Can values be dropped?
Is the buffering policy intentional?
Can the producer outrun the consumer?
Is this stream unicast, multicast, or unspecified?
Are multiple consumers allowed?
Does the stream capture self strongly?
Is a Task created inside the stream canceled?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `AsyncSequence` is like an async array. You can use `for await` to receive values.

### Senior answer

> `AsyncSequence` models values over time. `AsyncStream` is a bridge for callback or delegate APIs, but I need to explicitly handle `finish`, cancellation, buffering, and cleanup with `onTermination`.

### Staff-level answer

> I treat stream APIs as ownership and flow-control boundaries. Before exposing an `AsyncSequence`, I define whether it is finite or infinite, unicast or multicast, lossy or lossless, bounded or unbounded, and who owns cancellation. `AsyncStream` is fine for adapting callbacks, but for serious producer-consumer backpressure I would consider `AsyncChannel`, a custom `AsyncSequence`, or a domain-specific protocol instead of pretending buffering solves the architecture.

Staff-level questions to ask:

```text
Is this event source finite or infinite?
Is the sequence lossy or lossless?
Can consumers cancel independently?
Can there be multiple consumers?
Does the producer need true backpressure?
What is the maximum buffer size?
What happens during app backgrounding?
What happens if the stream is created but never consumed?
Who closes the socket / stops location / removes observer?
Should this be AsyncSequence, Combine Publisher, delegate, NotificationCenter, or a one-shot async function?
```

---

## 9. Interview-ready summary

`AsyncSequence` is the right abstraction for many asynchronous values over time, while an `async` function is for one future result. `AsyncStream` and `AsyncThrowingStream` are useful bridges from callbacks and delegates into `for await`, but they require explicit lifecycle design: call `finish()` for finite streams, use `finish(throwing:)` for failures, set `onTermination` to clean up resources, and choose a buffering policy intentionally. For true backpressure, `AsyncStream` may not be enough; a channel or custom sequence can be a better design.

---

## 10. Flashcards

Q: When should an API return `AsyncSequence`?  
A: When it produces multiple values over time and the caller should consume them incrementally with cancellation-aware async control flow.

Q: What is the difference between `async -> T` and `AsyncSequence<T>`?  
A: `async -> T` produces one result later; `AsyncSequence<T>` produces zero or more values over time until it terminates.

Q: What happens if an `AsyncStream` yields values but never calls `finish()`?  
A: A consumer may receive the yielded values and then suspend forever waiting for the next value.

Q: What is `onTermination` for?  
A: It propagates downstream termination or cancellation back to producer cleanup: cancel tasks, remove observers, close sockets, stop hardware updates.

Q: Why is `.unbounded` buffering risky?  
A: A fast producer can accumulate memory if the consumer is slow or paused.

Q: When is `.bufferingNewest(1)` appropriate?  
A: When only the latest value matters, such as UI state, progress, or sensor snapshots.

Q: When is `.bufferingNewest(1)` dangerous?  
A: When every event matters, such as ordered messages, transactions, logs, or protocol frames.

Q: Does `AsyncStream` provide true backpressure?  
A: Not necessarily. It buffers and can report yield results, but it does not inherently suspend a synchronous producer until consumption catches up.

Q: What is a better fit for task-to-task backpressure?  
A: `AsyncChannel` from Swift Async Algorithms, or a custom `AsyncSequence` with explicit flow control.

Q: What is the biggest production smell with `AsyncStream`?  
A: Creating a stream that starts resources but has no clear termination, cancellation, or ownership story.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive]]
- [[D9 — Continuations and bridging callback APIs]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D12 — Synchronization library, mutexes, and choosing between actors and locks]]
- [[F1 — Testing async and concurrent code]]

---

## 12. Sources

- Swift Senior/Staff Rubric, section D10.
- SE-0298: `AsyncSequence`, implemented in Swift 5.5; introduced async iteration over many values over time. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0298-asyncsequence.md "swift-evolution/proposals/0298-asyncsequence.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0314: `AsyncStream` and `AsyncThrowingStream`; bridge callback/delegate-style multi-value APIs into async contexts. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md "swift-evolution/proposals/0314-async-stream.md at main · swiftlang/swift-evolution · GitHub"))
- Apple Developer Documentation: `AsyncStream`, `AsyncStream.Continuation`, buffering policy, and termination behavior. ([Apple Developer](https://developer.apple.com/documentation/swift/asyncstream?utm_source=chatgpt.com "AsyncStream | Apple Developer Documentation"))
- Swift Async Algorithms: `AsyncChannel` and backpressure semantics. ([GitHub](https://github.com/apple/swift-async-algorithms/blob/main/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Channel.md "swift-async-algorithms/Sources/AsyncAlgorithms/AsyncAlgorithms.docc/Guides/Channel.md at main · apple/swift-async-algorithms · GitHub"))