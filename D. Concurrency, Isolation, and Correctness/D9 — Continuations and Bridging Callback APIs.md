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

**Goal:** Select one terminal callback result under synchronization and resume outside the lock.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A continuation bridges one asynchronous result into the current task. The bridging body runs synchronously and may resume before it returns; completion still waits until that body returns. Resuming does not cancel or unregister the underlying request. A separate synchronized state machine must arbitrate completion, cancellation, and resource release. [SE-0300](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0300-continuation.md)

## Rules and pitfalls

### Define terminal-action ownership

Select a winning action under a lock, remove its stored continuation/request handles, then resume or cancel outside the lock. Unknown callbacks and destructors can reenter; do not call them while holding a nonrecursive mutex.

**Common pitfall:** A plain Boolean guard does not prevent simultaneous duplicate callbacks. Use synchronized state, and make the winner policy explicit.

### Choose the handle and cancellation contract

Checked handles diagnose duplicate resume; leaked unfinished handles can still hang. The newer noncopyable Continuation supports noncopyable success with OS 27 requirements here. Typed checked-throwing factories also exist in this SDK.

**Common pitfall:** Assuming onCancel only runs after request installation misses cancellation-before-entry and installation races. Store cancellation in the terminal state so a later installed handle is cancelled/released correctly. Use AsyncSequence for genuinely repeated events.

## Interview questions

### Which handle supports noncopyable success, and why are typed failures not exclusive to withContinuation in this SDK?

The newer noncopyable Continuation can support noncopyable success values under its API constraints. Typed errors are not exclusive to it: this SDK accepts explicitly typed checked throwing factories too. Verify handle ownership, factory signature, and runtime availability as separate questions.

### Why can't an arbitrary escaping callback simply consume a captured noncopyable continuation?

An arbitrary escaping callback can be invoked repeatedly, so its capture cannot simply consume the same noncopyable continuation each time. Put single ownership behind a synchronized one-shot state, transfer the handle to the winning action, and make later callbacks no-ops or protocol violations according to policy.

### Can onCancel precede operation entry, or precede installation of the request handle?

Yes to both. Cancellation can be observed before the operation body enters or while a native start call has not returned its handle. The state machine must remember a terminal cancellation and dispose of a handle installed afterwards. A callback can also complete synchronously during registration.

### Why does this adapter select its terminal action under the lock, then resume, cancel, and release resources outside it? Which callbacks or destructors could reenter?

The lock protects only choosing and detaching terminal state. Resume, request.cancel, unregister callbacks, and releasing captured resources can invoke unknown code or destructors that reenter. Performing them after unlocking prevents lock-order and recursive-entry deadlocks.

### Can success still win if cancellation is requested afterwards? What policy does the adapter implement?

Yes, under a first-terminal-action policy completion can win before cancellation takes the state. Cancellation requested afterwards does not retroactively change that selected result. State whether the API uses first-wins, cancellation-preferred, or another explicit policy; tests must exercise the chosen boundary.

## Focused exercise

**Task:** Adapt a one-result callback using a synchronized terminal state. Check synchronous completion and duplicate delivery; state the cancellation policy.

**Acceptance checks:**

- Synchronous callback delivery completes successfully.
- Duplicate delivery cannot resume the continuation twice.
- No resume occurs under the lock; cancellation policy is stated rather than implied.

**Solution:**

Standalone one-shot adapter using Synchronization.Mutex, macOS 15 / iOS 18 or newer. Policy: first callback wins; task cancellation is not connected to a native request in this focused fixture.

```swift
import Synchronization
@available(macOS 15, iOS 18, *)
final class OneShot: Sendable {
    private let state: Mutex<CheckedContinuation<Int, Never>?>
    init(_ continuation: CheckedContinuation<Int, Never>) { state = Mutex(continuation) }
    func finish(_ value: Int) {
        let continuation = state.withLock { stored in
            let selected = stored
            stored = nil
            return selected
        }
        continuation?.resume(returning: value)
    }
}
@available(macOS 15, iOS 18, *)
func adapt(start: (_ completion: @escaping @Sendable (Int) -> Void) -> Void) async -> Int {
    await withCheckedContinuation { continuation in
        let state = OneShot(continuation)
        start { state.finish($0) }
    }
}
@main struct Probe {
    static func main() async {
        if #available(macOS 15, iOS 18, *) {
            let result = await adapt { completion in
                completion(7)
                completion(9)
            }
            precondition(result == 7)
        }
    }
}
```

**Reasoning:** Taking the optional handle under the mutex gives exactly one callback the resume responsibility. This fixture intentionally has no registration resource or cancellation hook; the complete adapter below adds those state transitions rather than treating this small example as production-complete.

## In practice

**Choose it when:** Bridge a single-result callback with explicit result and request ownership.

**Investigate by:** Gate cancellation, registration, synchronous completion, and duplicate callbacks independently.

**Strong evidence:** Verify installation races and reentrant cleanup, including which terminal result is allowed to win.

### Complete cancellation and registration state machine — optional

#### Contract

Wrap a callback registration function that returns an optional cancellation handle. Callbacks may be synchronous, duplicated, or concurrent. A nil handle means registration failed unless an earlier callback already completed the operation. Cancellation is a nonblocking request and may itself trigger the callback synchronously.

Use a first-terminal-event policy. Cancellation before continuation installation must avoid starting the request. Cancellation during registration may allow the request to start, but must be forwarded when its handle becomes available. No external action or request destruction should occur under the state lock.

#### Swift 6.4 implementation

This standalone implementation uses `Continuation` in noncopyable state. The adapter deliberately restricts `Value` to Copyable Sendable payloads so a reusable `Result` callback can safely share its values; the simpler `packet` probe covers noncopyable success separately.

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

The only event allowed before `install` is cancellation, so the late incoming handle can be resumed with `.cancelled` without buffering a general early result. Each operation creates a fresh gate; continuation and request installation are each called at most once.

If completion wins, the gate releases the request without cancelling it. If cancellation wins, it extracts the current request or remembers that a later one must be cancelled. Late callbacks see terminal state and do nothing. The cancellation error is this adapter's typed `BridgeError.cancelled`, not necessarily `CancellationError`.

The gate resumes before calling the legacy cancellation action, outside the lock. That permits a synchronous reentrant callback without a second resume. It does not guarantee the async caller waits for external teardown.

For older deployment targets, use a checked-handle implementation of the same state machine. Changing handle type alone is insufficient; preserve registration, resource-release, and winner-selection behavior.

#### Optional value/error pairs

Reusing `BridgeError` and `CancelRequest` from the implementation, normalize all four callback shapes:

```swift
struct User: Sendable {
    let id: Int
}

func normalize(
    user: User?, error: BridgeError?
) -> Result<User, BridgeError> {
    switch (user, error) {
    case let (user?, nil): .success(user)
    case let (nil, error?): .failure(error)
    case (.some, .some), (nil, nil): .failure(.invalidCallbackState)
    }
}

@available(anyAppleOS 27.0, *)
nonisolated(nonsending)
func loadUser(
    start: @Sendable (
        @escaping @Sendable (User?, BridgeError?) -> Void
    ) -> CancelRequest?
) async throws(BridgeError) -> User {
    try await awaitLegacy { complete in
        start { user, error in
            complete(normalize(user: user, error: error))
        }
    }
}
```

The naive pair of independent `if let` statements would resume twice for two non-nil fields and never resume for two nil fields. Normalization chooses one result; the shared gate additionally arbitrates repeated callbacks.

#### Verification tasks

- Synchronous success, synchronous failure, nil registration, and a callback followed by nil registration.
- Two concurrent callbacks: one result wins without a duplicate-resume trap.
- Cancellation before operation entry: registration count stays zero.
- Cancellation while registration is returning its request, and cancellation after request installation.
- A cancellation action that calls completion synchronously.
- Callback success winning before cancellation, and cancellation winning before callback.
- A legacy owner that explicitly reports `ownerEnded` before releasing its callback.
- Every optional value/error combination.
- Dropped-handle trap, checked leak/timeout, and consumed-handle diagnostic in isolated probes.

An owner that silently discards a callback still violates the completion contract; the gate cannot infer that external event. The waiting operation retains its gate. Explicit cancellation can release it, but an uncancelled request that never reports completion still requires a timeout or external lifetime guarantee. A task-group timeout alone can also wait forever if its cancelled adapter child never resumes; fix the adapter's cancellation path first.

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
- [withTaskCancellationHandler](https://developer.apple.com/documentation/swift/withtaskcancellationhandler(operation:oncancel:isolation:)
- [Current checked-continuation implementation](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/CheckedContinuation.swift)
