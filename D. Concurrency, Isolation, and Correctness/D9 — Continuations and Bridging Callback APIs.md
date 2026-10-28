---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - continuations
  - async-await
---

# D9 — Continuations and Bridging Callback APIs

**Learning goals:**

- Bridge one callback result with exactly one completion, including synchronous registration callbacks.
- Arbitrate duplicate callbacks, cancellation, and request installation under synchronization, then perform external actions after unlocking.
- Choose checked or noncopyable continuation handles from payload, failure, and deployment requirements.

**Prerequisites:** [[D1 — Structured Concurrency Fundamentals]]–[[D2 — Task Hierarchy, Cancellation, and Priorities]] for suspension and cancellation; [[D6 — `Sendable`, `@Sendable`, and `sending`]] for Sendable captures. Mutex requires macOS 15/iOS 18; the Continuation extension requires OS 27.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A continuation is a handle that lets a callback complete one suspended async operation. The continuation body registers the callback synchronously. A callback may run before registration returns, and resuming early is supported; the async call still waits for the registration body to return.

Exactly-once completion is an obligation, not a property added to an unreliable source automatically. A duplicate checked resume traps; an unfinished retained handle can leave the task suspended. Resuming does not unregister the callback or cancel its request. Treat result selection, cancellation, and resource ownership as explicit state transitions. [SE-0300](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0300-continuation.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### A synchronous callback is valid

**Setup:** Standalone runnable one-result API.

```swift
func legacy(_ completion: (Int) -> Void) { completion(7) }
func value() async -> Int {
    await withCheckedContinuation { continuation in
        legacy { continuation.resume(returning: $0) }
    }
}
let result = await value()
precondition(result == 7)
```

**Walkthrough:** The callback resumes before `legacy` returns. The factory handles that ordering; no semaphore is necessary. This source calls once, so it needs no duplicate arbitration.

### Shared setup — first-callback gate

The mutex stores an optional checked handle. The winner takes it while locked; later callbacks find nil. Resume occurs after the lock closure returns. Prepend **first-callback gate** to the simple adapter examples.

```swift
import Synchronization
final class OneShot: Sendable {
    private let state: Mutex<CheckedContinuation<Int, Never>?>
    init(_ continuation: CheckedContinuation<Int, Never>) { state = Mutex(continuation) }
    func finish(_ value: Int) {
        let selected = state.withLock { stored in
            let selected = stored
            stored = nil
            return selected
        }
        selected?.resume(returning: value)
    }
}
func adapt(start: (_ complete: @escaping @Sendable (Int) -> Void) -> Void) async -> Int {
    await withCheckedContinuation { continuation in
        let gate = OneShot(continuation)
        start { gate.finish($0) }
    }
}
```

**Walkthrough:** Checking and removing the handle are one critical section. The continuation cannot be selected twice, including by concurrent callbacks. This gate does not yet own a native request or connect cancellation.

## Rules and pitfalls

### A plain Boolean is not duplicate-callback synchronization

**Common pitfall:** An unsynchronized flag is used inside a Sendable callback to prevent duplicate resume.

**Mistake:** Standalone **intentionally rejected** capture. The source may invoke its callback concurrently.

```swift
func start(_ callback: @escaping @Sendable (Int) -> Void) { callback(7) }
func broken() async -> Int {
    await withCheckedContinuation { continuation in
        var finished = false
        start { value in
            if !finished {
                finished = true
                continuation.resume(returning: value)
            }
        }
    }
}
```

**Consequence:** The captured mutable flag violates the concurrency contract. Bypassing the diagnostic still allows two callbacks to observe false and resume twice.

**Why:** Check-and-set must be atomic with selecting the handle.

**Correction:** Prepend **first-callback gate**. Runnable duplicated synchronous callbacks.

```swift
let result = await adapt { complete in complete(7); complete(9) }
precondition(result == 7)
```

**What changes:** Only the first callback obtains the handle. This policy intentionally ignores duplicate delivery for an API promising one result. For genuinely repeated events, use AsyncSequence instead of throwing events away.

### Remember cancellation before installation

**Shared setup — request handle:** A Sendable cancellation closure may reenter the adapter. It must be called outside its state lock.

```swift
import Synchronization
struct CancelRequest: Sendable { let cancel: @Sendable () -> Void }
```

**Common pitfall:** Cancelling before a native handle arrives is assumed to be equivalent to cancelling it afterward.

**Mistake:** Prepend **request handle**. **Compile-only** incomplete registration state.

```swift
final class Registration: Sendable {
    private let state = Mutex<CancelRequest?>(nil)
    func install(_ request: CancelRequest) { state.withLock { $0 = request } }
    func cancel() {
        let request = state.withLock { $0.take() }
        request?.cancel()
    }
}
```

**Consequence:** If cancel sees nil, it forgets cancellation. A subsequently installed request keeps running.

**Why:** Absence of a handle is not evidence that cancellation can no longer matter.

**Correction:** Prepend **request handle**. **Compile-only** single-install state.

```swift
final class Registration: Sendable {
    private struct State { var request: CancelRequest?; var cancelled = false }
    private let state = Mutex(State())
    func install(_ request: CancelRequest) {
        let cancelNow = state.withLock { state in
            if state.cancelled { return true }
            state.request = request
            return false
        }
        if cancelNow { request.cancel() }
    }
    func cancel() {
        let request = state.withLock { state in
            state.cancelled = true
            return state.request.take()
        }
        request?.cancel()
    }
}
```

**What changes:** Cancellation remains part of state, and a late handle is cancelled outside the lock. This small model permits one installation; the complete adapter also chooses the continuation's terminal result.

## Worked example

### A complete first-terminal-event adapter

**Setup — complete adapter:** Standalone **compile-only** implementation for OS 27, used by the following drivers. Value is Copyable and Sendable because the reusable Result callback shares its payload. The noncopyable-success extension below uses a different source contract.

**Predict:** Can cancellation occur before continuation installation, while registration is returning a handle, or after a callback already won?

```swift
import Synchronization

enum BridgeError: Error {
    case failed
    case couldNotStart
    case cancelled
    case ownerEnded
    case invalidCallbackState
}

struct CancelRequest: Sendable {
    let cancel: @Sendable () -> Void
}

@available(anyAppleOS 27.0, *)
final class ResumeGate<Value: Sendable>: Sendable {
    private struct State: ~Copyable {
        var continuation: Continuation<Value, BridgeError>?
        var request: CancelRequest?
        var finished = false
        var cancelled = false
    }
    private struct Action: ~Copyable {
        var continuation: Continuation<Value, BridgeError>?
        var request: CancelRequest?
    }
    private let state = Mutex(State())

    // Called once, before registration. Only cancellation can precede it.
    func install(_ continuation: consuming Continuation<Value, BridgeError>) -> Bool {
        var incoming: Continuation<Value, BridgeError>? = consume continuation
        let shouldStart = state.withLock { state in
            guard !state.finished else { return false }
            state.continuation = incoming.take()
            return true
        }
        incoming?.resume(throwing: .cancelled)
        return shouldStart
    }

    // Called at most once after the registration call returns a handle.
    func installRequest(_ request: CancelRequest) {
        let cancelNow = state.withLock { state in
            if !state.finished {
                state.request = request
                return false
            }
            return state.cancelled
        }
        if cancelNow { request.cancel() }
    }

    func complete(_ result: Result<Value, BridgeError>) {
        let action = state.withLock { state -> Action in
            guard !state.finished else { return Action() }
            state.finished = true
            return Action(
                continuation: state.continuation.take(),
                request: state.request.take()
            )
        }
        action.continuation?.resume(with: result)
        // Retain then release the request outside the lock.
        withExtendedLifetime(action.request) {}
    }

    func cancel() {
        let action = state.withLock { state -> Action in
            guard !state.finished else { return Action() }
            state.finished = true
            state.cancelled = true
            return Action(
                continuation: state.continuation.take(),
                request: state.request.take()
            )
        }
        action.continuation?.resume(throwing: .cancelled)
        action.request?.cancel()
    }
}

@available(anyAppleOS 27.0, *)
nonisolated(nonsending)
func awaitLegacy<Value: Sendable>(
    start: @Sendable (
        @escaping @Sendable (Result<Value, BridgeError>) -> Void
    ) -> CancelRequest?
) async throws(BridgeError) -> Value {
    let gate = ResumeGate<Value>()
    return try await withTaskCancellationHandler {
        () async throws(BridgeError) -> Value in
        try await withContinuation(of: Value.self, throwing: BridgeError.self) {
            continuation in
            guard gate.install(consume continuation) else { return }
            if let request = start({ gate.complete($0) }) {
                gate.installRequest(request)
            } else {
                gate.complete(.failure(.couldNotStart))
            }
        }
    } onCancel: {
        gate.cancel()
    }
}
```

**Outcome:** The first terminal event to take state wins. Completion selects its result and releases the native request. Cancellation selects `BridgeError.cancelled`, cancels an installed request, or remembers that a late handle must be cancelled.

**Why:** `install` runs once before registration. Only cancellation can precede it. Each terminal operation marks finished and removes the continuation/request together under the mutex. It then resumes, invokes cancellation, and releases resources after unlocking. A reentrant callback sees finished state and cannot resume again. A callback during registration can win even when the registration call later returns nil.

This gate resumes before invoking external cancellation. The async caller therefore does not receive a guarantee that external teardown has completed. A source that silently discards its callback still violates its completion contract; retained unfinished state can hang until explicit cancellation or an external lifetime rule completes it. A task-group timeout cannot rescue an adapter whose cancellation path never resumes.

**Driver:** Prepend **complete adapter**. Runnable synchronous success, duplicate callback, failed registration, and completion-before-nil-registration checks.

```swift
let value: Int = try await awaitLegacy { complete in
    complete(.success(7)); complete(.success(9))
    return nil
}
precondition(value == 7)
do {
    let _: Int = try await awaitLegacy { _ in nil }
    preconditionFailure("expected registration failure")
} catch BridgeError.couldNotStart { }
```

## Interview questions

### Why does the rubric's duplicate-callback adapter compile and then trap before returning a normal result?

**Answer:** A checked continuation can be copied, so the compiler does not reject a second resume. Its runtime check traps when the callback resumes the same handle twice.

**Example:** Standalone **compile-only; do not call**. This intentionally unsafe fixture is not executed.

```swift
func duplicate(_ complete: (Int) -> Void) { complete(1); complete(2) }
func broken() async -> Int {
    await withCheckedContinuation { continuation in
        duplicate { continuation.resume(returning: $0) }
    }
}
```

**Walkthrough:** The first callback selects a result. The second reaches the runtime duplicate-resume check while the synchronous registration body is still running. That body must return before the async factory completes normally, so this is not a valid two-value result.

**Boundary:** Use the first-callback gate for an API promising one completion, or AsyncSequence for real repeated events. Changing the handle type alone does not repair a missing callback or connect native cancellation.

### Why can't an arbitrary escaping callback simply consume a captured noncopyable continuation?

**Answer:** The callback can be invoked repeatedly. It cannot consume the same captured noncopyable handle on every invocation; single ownership must be selected before resuming.

**Example:** Standalone **intentionally rejected** OS 27 adapter.

```swift
func legacy(_ callback: @escaping @Sendable (Int) -> Void) { callback(7) }
func invalid() async -> Int {
    await withContinuation(of: Int.self) { continuation in
        legacy { value in continuation.resume(returning: value) }
    }
}
```

**Walkthrough:** Each call to the escaping closure would consume the same capture. The compiler rejects that ownership model. The complete adapter instead stores one continuation and transfers it to one selected action.

**Boundary:** A noncopyable handle does not turn a repeated-event API into a one-result API. Use an event sequence when repeated callbacks are meaningful.

### Can onCancel precede operation entry, or precede installation of the request handle?

**Answer:** Yes. A task can already be cancelled when its handler is installed, and cancellation can race with the synchronous registration call before that call returns its handle.

**Example:** Prepend **complete adapter**. Runnable cancellation-before-entry check.

```swift
let starts = Mutex(0)
let task = Task {
    try await awaitLegacy { (_: @escaping @Sendable (Result<Int, BridgeError>) -> Void) in
        starts.withLock { $0 += 1 }
        return nil
    }
}
task.cancel()
do { _ = try await task.value; preconditionFailure("expected cancellation") }
catch BridgeError.cancelled { }
precondition(starts.withLock { $0 } == 0)
```

**Walkthrough:** Top-level MainActor execution cancels the ordinary task before its body runs. The handler records cancellation; continuation installation then resumes with the selected error and does not start registration.

**Boundary:** If registration already began, it may still return a handle. `installRequest` must then forward remembered cancellation. A handler must not assume a mutable captured handle has already been assigned.

### Why select the terminal action under a lock, then resume, cancel, and release resources outside it? Which callbacks or destructors could reenter?

**Answer:** The lock protects deciding who owns completion. External actions can invoke callbacks or destructors that reenter; calling them under a nonrecursive mutex can deadlock.

**Example:** Prepend **complete adapter**. Runnable reentrant cancellation.

```swift
let gate = ResumeGate<Int>()
let cancellations = Mutex(0)
do {
    let _: Int = try await withContinuation(of: Int.self, throwing: BridgeError.self) {
        continuation in
        let installed = gate.install(consume continuation)
        precondition(installed)
        gate.installRequest(CancelRequest {
            cancellations.withLock { $0 += 1 }
            gate.complete(.success(99)) // synchronous reentry
        })
        gate.cancel()
    }
    preconditionFailure("expected cancellation")
} catch BridgeError.cancelled { }
precondition(cancellations.withLock { $0 } == 1)
```

**Walkthrough:** Cancellation takes both handles and unlocks before resuming and calling native cancel. That callback immediately reenters `complete`, sees terminal state, and returns. The operation finishes once with cancellation.

**Boundary:** Releasing the final reference to a request can also run custom destruction. Taking that reference out of protected state ensures its final release occurs outside the lock. This is a lock-order rule, not a guarantee that all native cleanup is synchronous.

### Can success still win if cancellation is requested afterwards? What policy does the adapter implement?

**Answer:** Yes. Under first-terminal-event wins, later cancellation does not retroactively replace a result already selected by completion.

**Example:** Prepend **complete adapter**. Runnable controlled selection.

```swift
let gate = ResumeGate<Int>()
let value = try await withContinuation(of: Int.self, throwing: BridgeError.self) {
    continuation in
    let installed = gate.install(consume continuation)
        precondition(installed)
    gate.complete(.success(7))
    gate.cancel()
}
precondition(value == 7)
```

**Walkthrough:** Completion removes the handle first. Cancel finds finished state and has nothing left to resume or cancel. This is an explicit winner policy, not a claim that cancellation always loses races.

**Boundary:** A caller needing cancellation-preferred publication can check its own task state before publishing. That is a separate rule from the adapter's already-selected terminal outcome.

### Extension — Which handle supports noncopyable success, and why are typed failures not exclusive to withContinuation in this SDK?

**Answer:** The OS 27 `Continuation` supports noncopyable success. The selected SDK also provides explicitly typed checked throwing factories; handle ownership, failure typing, and availability are separate choices.

**Example:** Standalone runnable OS 27 noncopyable payload.

```swift
struct Packet: ~Copyable, Sendable { let byte: UInt8 }
func packet() async -> sending Packet {
    await withContinuation(of: Packet.self) { continuation in
        continuation.resume(returning: Packet(byte: 7))
    }
}
let received = await packet()
precondition(received.byte == 7)
```

**Walkthrough:** Resume consumes the handle and transfers the packet. Consuming the same handle again is a compiler error. Dropping a new handle without resuming traps; retaining an unfinished handle can still hang.

This independent runnable example uses a typed checked throwing factory on this SDK:

```swift
enum LoadError: Error { case unavailable }
func checked() async throws(LoadError) -> Int {
    try await withCheckedThrowingContinuation {
        (continuation: CheckedContinuation<Int, LoadError>) in
        continuation.resume(throwing: .unavailable)
    }
}
do { _ = try await checked(); preconditionFailure("expected failure") }
catch LoadError.unavailable { }
```

**Boundary:** Checked double-resume is a runtime trap; unsafe double-resume is undefined behavior. Do not execute those failures as normal examples. The new handle's consumed-use error is checked separately. [SE-0528](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0528-noncopyable-continuation.md)

## Focused exercise

**Task:** Adapt a one-result callback and verify synchronous and concurrent duplicate delivery without resuming twice.

**Acceptance checks:**

- Synchronous delivery returns the first callback value.
- Two concurrent callbacks select one permitted result.
- Resume is outside the mutex; cancellation policy is explicit.

**Solution:** Prepend **first-callback gate**. Runnable; Dispatch starts callbacks rather than blocking on future task work.

```swift
import Dispatch
let synchronous = await adapt { complete in complete(7); complete(9) }
precondition(synchronous == 7)
let concurrent = await adapt { complete in
    DispatchQueue.global().async { complete(7) }
    DispatchQueue.global().async { complete(9) }
}
precondition([7, 9].contains(concurrent))
```

**Reasoning:** The mutex selects one handle atomically, then resume happens after unlocking. Either concurrent callback can win. This focused adapter does not connect native cancellation; the complete adapter above supplies that policy.

### Extension — Normalize optional value/error callbacks

**Setup:** Prepend **complete adapter** for BridgeError. Runnable exhaustive callback-shape policy.

```swift
struct User: Sendable { let id: Int }
func normalize(user: User?, error: BridgeError?) -> Result<User, BridgeError> {
    switch (user, error) {
    case let (user?, nil): .success(user)
    case let (nil, error?): .failure(error)
    case (.some, .some), (nil, nil): .failure(.invalidCallbackState)
    }
}
let combinations: [(User?, BridgeError?)] = [(User(id: 1), nil), (nil, .failed),
                                           (User(id: 1), .failed), (nil, nil)]
for (index, pair) in combinations.enumerated() {
    switch normalize(user: pair.0, error: pair.1) {
    case .success: precondition(index == 0)
    case .failure: precondition(index != 0)
    }
}
```

Two independent if-let resumes would mishandle both-present and both-absent callbacks. Normalization chooses one result; the synchronized gate separately handles repeated delivery.

## In practice

**Choose it when:** A callback really describes one eventual result and its completion/resource contract can be stated.

**Investigate by:** Control registration, callback, cancellation, and teardown separately. Check synchronous completion, late handles, duplicate callbacks, and reentrant cancellation.

**Strong evidence:** State the winner policy and distinguish task completion from external teardown. A missing callback still needs cancellation, timeout cooperation, or an external lifetime guarantee.

## References

### Related guides

- [[A13 — Error handling and typed throws]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

### Official sources

- [SE-0300](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0300-continuation.md)
- [SE-0528 — Noncopyable Continuation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0528-noncopyable-continuation.md)
- [CheckedContinuation](https://developer.apple.com/documentation/swift/checkedcontinuation)
- [withCheckedThrowingContinuation](https://developer.apple.com/documentation/swift/withcheckedthrowingcontinuation%28isolation%3Afunction%3A_%3A%29)
- [withTaskCancellationHandler](https://developer.apple.com/documentation/swift/withtaskcancellationhandler%28operation%3Aoncancel%3Aisolation%3A%29)
- [Current checked-continuation implementation](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/CheckedContinuation.swift)
