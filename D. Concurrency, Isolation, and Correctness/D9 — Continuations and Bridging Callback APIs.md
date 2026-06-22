---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - continuations
  - async-await
---
## 0. Rubric snapshot

**Rubric expectation**

Know `withCheckedContinuation`, `withCheckedThrowingContinuation`, correctness rules, and exactly-once resume semantics. The rubric explicitly calls out multiple resumes and forgotten resumes as correctness bugs, and asks you to explain why a multi-callback API may not fit a single continuation.

**Caveats**

- A continuation is a bridge from callback/event-style code into one suspended async task.
- It is not a general event stream abstraction.
- Checked continuations detect some misuse, but they do not make a bad adapter correct.
- The core invariant is: resume exactly once on every execution path.

**You should be able to answer**

- What invariants must hold when using a continuation?
- Why can adapting a multi-callback API to a single continuation be the wrong abstraction?

**You should be able to do**

- Wrap a legacy completion-handler API.
- Explain how you guarantee exactly one resume.
- Diagnose the code probe where a legacy API calls the completion twice.

---

## 1. Core mental model

A continuation is a manually held “resume handle” for one suspended async function call.

When an async function reaches `withCheckedContinuation` or `withCheckedThrowingContinuation`, Swift gives your synchronous bridging closure a continuation object. Your job is to pass that continuation into the legacy callback world and resume it when the old API finishes.

The continuation represents **one suspension point**, not the lifetime of a task, not a cancellable operation, and not an event channel. SE-0300 describes continuations as the mechanism that lets async Swift work with callback- or delegate-based systems by suspending the task and later resuming it from synchronous code. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md "swift-evolution/proposals/0300-continuation.md at main · swiftlang/swift-evolution · GitHub"))

The key idea:

```text
async function suspends once → legacy API eventually calls back → continuation resumes the suspended task exactly once
```

Swift guarantees that once the continuation is resumed, the suspended task can continue with a returned value or thrown error. It does **not** guarantee that the old API behaves well, calls back once, calls back at all, respects task cancellation, or uses the right abstraction for multi-event data.

Apple’s `CheckedContinuation` documentation states the central rule directly: a resume method must be called exactly once on every execution path, and resuming more than once is a correctness violation. ([Apple Developer](https://developer.apple.com/documentation/swift/checkedcontinuation?utm_source=chatgpt.com "CheckedContinuation | Apple Developer Documentation"))

---

## 2. Essential mechanics

### 2.1 `withCheckedContinuation` is for one successful result

Use it when the legacy API cannot fail, or when failure is represented in the value domain.

```swift
func legacyLoad(completion: @escaping (Data) -> Void) {
    // Old callback API
}

func load() async -> Data {
    await withCheckedContinuation { continuation in
        legacyLoad { data in
            continuation.resume(returning: data)
        }
    }
}
```

This adapts:

```text
callback: (Data) -> Void
```

into:

```text
async -> Data
```

The continuation closure itself is synchronous. It starts the legacy operation and returns. The task remains suspended until the continuation is resumed. SE-0300 specifies that the continuation operation runs immediately in the current task context, and that `resume` makes the task schedulable rather than necessarily continuing it inline. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md "swift-evolution/proposals/0300-continuation.md at main · swiftlang/swift-evolution · GitHub"))

---

### 2.2 `withCheckedThrowingContinuation` is for one success-or-failure result

Use it for completion handlers shaped like `Result<T, Error>` or `(T?, Error?)`.

```swift
func legacyFetch(completion: @escaping (Result<Data, any Error>) -> Void) {
    // Old callback API
}

func fetch() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetch { result in
            continuation.resume(with: result)
        }
    }
}
```

`resume(with:)` is useful when the callback already uses `Result`.

Equivalent manual form:

```swift
func fetch() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetch { result in
            switch result {
            case .success(let data):
                continuation.resume(returning: data)

            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

---

### 2.3 Checked continuations are runtime-checked, not statically proven

Checked continuations help detect misuse:

```swift
continuation.resume(returning: 1)
continuation.resume(returning: 2) // trap
```

SE-0300 says `CheckedContinuation` traps on multiple resumes and logs a warning if discarded without resuming, leaving the task suspended and leaking resources held by that suspended task. Those checks happen regardless of optimization level. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md "swift-evolution/proposals/0300-continuation.md at main · swiftlang/swift-evolution · GitHub"))

This means:

```text
checked continuation = diagnostic guardrail
not = proof that your adapter is correct
```

---

### 2.4 Continuations do not automatically handle cancellation

Task cancellation does not magically cancel the legacy operation. If the old API returns a cancellable token, operation object, `URLSessionTask`, delegate registration, observer token, or similar handle, your async wrapper must connect Swift task cancellation to that handle.

```swift
func fetchImage(url: URL, client: LegacyClient) async throws -> UIImage {
    var request: LegacyRequest?

    return try await withTaskCancellationHandler {
        request?.cancel()
    } operation: {
        try await withCheckedThrowingContinuation { continuation in
            request = client.fetchImage(url: url) { result in
                continuation.resume(with: result)
            }
        }
    }
}
```

This sketch shows the intent, but in production you also need thread-safe state handling if cancellation and callback can race. SE-0300 explicitly treats a continuation as a single suspension-point mechanism, not a task handle that controls the whole task lifetime. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md "swift-evolution/proposals/0300-continuation.md at main · swiftlang/swift-evolution · GitHub"))

---

## 3. Common traps and misconceptions

### Trap 1: “Checked” means safe

Bad:

```swift
func adapted() async throws -> Int {
    try await withCheckedThrowingContinuation { continuation in
        legacy { result in
            continuation.resume(with: result)
        }
    }
}
```

This is only correct if `legacy` has a real contract that says:

```text
The completion is called exactly once.
```

Better:

```swift
func adapted() async throws -> Int {
    try await withCheckedThrowingContinuation { continuation in
        legacyExactlyOnce { result in
            continuation.resume(with: result)
        }
    }
}
```

The name is not the important part. The contract is.

---

### Trap 2: Using a single continuation for multiple events

Bad:

```swift
func observeValues() async -> Int {
    await withCheckedContinuation { continuation in
        service.onValue = { value in
            continuation.resume(returning: value)
        }
    }
}
```

This only works for the **first** value. The second value attempts to resume an already-resumed continuation.

Better:

```swift
func observeValues() -> AsyncStream<Int> {
    AsyncStream { continuation in
        let token = service.observe { value in
            continuation.yield(value)
        }

        continuation.onTermination = { @Sendable _ in
            token.cancel()
        }
    }
}
```

Use a continuation for one result. Use `AsyncSequence` / `AsyncStream` for a sequence of values.

---

### Trap 3: Forgetting the “no callback” path

Bad:

```swift
func login() async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        auth.login { result in
            continuation.resume(with: result)
        }
    }
}
```

If `auth.login` can silently fail, drop the callback, or be deallocated before callback, the async caller hangs.

Better:

```swift
func login() async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        let didStart = auth.login { result in
            continuation.resume(with: result)
        }

        if !didStart {
            continuation.resume(throwing: LoginError.couldNotStart)
        }
    }
}
```

Every execution path must resume once.

---

### Trap 4: Guarding duplicate callbacks with a plain captured Boolean

Bad:

```swift
func adapted() async throws -> Int {
    try await withCheckedThrowingContinuation { continuation in
        var didResume = false

        legacy { result in
            guard !didResume else { return }
            didResume = true
            continuation.resume(with: result)
        }
    }
}
```

This is not thread-safe if callbacks can arrive concurrently from different queues. In Swift 6 strict concurrency, mutable captured state in escaping callback bridges is also a diagnostic smell.

Better: use a real synchronization primitive, an actor, or redesign the adapter around the legacy API’s actual lifecycle.

---

## 4. Direct answers to rubric questions

### Q1. What invariants must hold when using a continuation?

A continuation must be resumed **exactly once** on every possible path.

That means:

```text
0 resumes  → suspended task hangs / leaks resources
1 resume   → correct
2 resumes  → correctness bug; checked continuation traps
```

You also need to preserve the semantic shape of the original API:

- one completion → continuation is appropriate
    
- multiple values → `AsyncSequence`
    
- delegate lifecycle → stream or higher-level wrapper
    
- cancellable operation → connect task cancellation to legacy cancellation
    
- callback may arrive concurrently → synchronize resume state
    

Interview version:

> A continuation is a one-shot bridge from callback code into one suspended async task. The invariant is exactly-once resume on all paths: not zero, not two. Checked continuations help detect duplicate resumes and leaked continuations, but they do not prove the adapter is semantically correct. I still need to reason about cancellation, callback threading, ownership, and whether the legacy API is actually one-shot.

---

### Q2. Why can adapting a multi-callback API to a single continuation be the wrong abstraction?

Because a continuation returns or throws **once**. A multi-callback API emits multiple events over time.

Examples:

```text
location updates
websocket messages
download progress
delegate callbacks
notification observation
Bluetooth state changes
AVFoundation capture output
```

A single continuation forces you to lie about the API shape. You either drop later values, trap on duplicate resumes, or hang because there is no clear terminal callback.

Use `AsyncStream` or `AsyncThrowingStream` when the API emits multiple values.

Interview version:

> A continuation models one eventual result. If the callback can fire multiple times, the natural async shape is usually `AsyncSequence`, not `async -> Value`. Otherwise the second callback violates the continuation’s exactly-once rule. The staff-level judgment is to preserve the semantic shape of the API instead of mechanically wrapping callbacks.

---

## 5. Code probe

Given:

```swift
func legacy(_ completion: @escaping (Result<Int, Error>) -> Void) {
    completion(.success(1))
    completion(.success(2))
}

func adapted() async throws -> Int {
    try await withCheckedThrowingContinuation { continuation in
        legacy { result in
            continuation.resume(with: result)
        }
    }
}
```

### What happens?

This compiles, but traps at runtime.

On Swift 6.2.1, the meaningful runtime failure is:

```text
_Concurrency/CheckedContinuation.swift:169: Fatal error: SWIFT TASK CONTINUATION MISUSE: adapted() tried to resume its continuation more than once, returning 2!
```

There is no reliable normal output like:

```text
1
```

Because `legacy` calls the completion synchronously twice. The first callback resumes the continuation. The second callback immediately attempts to resume the same continuation again, so the checked continuation traps before the suspended task can complete normally.

### Why?

Step-by-step:

```text
adapted()
 └─ withCheckedThrowingContinuation creates one continuation
     └─ legacy starts
         ├─ completion(.success(1))
         │   └─ continuation.resume(returning: 1)  ✅ first resume
         └─ completion(.success(2))
             └─ continuation.resume(returning: 2)  ❌ second resume → trap
```

The broken invariant:

```text
one continuation instance must be resumed exactly once
this code resumes the same continuation twice
```

This is not a compiler error because Swift cannot generally know how many times an arbitrary callback-based API will invoke its callback. This is a runtime contract between your adapter and the legacy API.

### Fix or redesign

If `legacy` is actually a broken one-shot API and you only want the first result, you can defensively gate the continuation:

```swift
import Synchronization

private enum OneShotState<T: Sendable> {
    case pending(CheckedContinuation<T, any Error>)
    case resumed
}

private final class OneShot<T: Sendable>: Sendable {
    private let state: Mutex<OneShotState<T>>

    init(_ continuation: CheckedContinuation<T, any Error>) {
        self.state = Mutex(.pending(continuation))
    }

    func resume(with result: Result<T, any Error>) {
        let continuation = state.withLock { state -> CheckedContinuation<T, any Error>? in
            switch state {
            case .pending(let continuation):
                state = .resumed
                return continuation

            case .resumed:
                return nil
            }
        }

        continuation?.resume(with: result)
    }
}

func adaptedFirstResultOnly() async throws -> Int {
    try await withCheckedThrowingContinuation { continuation in
        let oneShot = OneShot<Int>(continuation)

        legacy { result in
            oneShot.resume(with: result)
        }
    }
}
```

This returns `1` and ignores the second callback.

But be careful: this is only a defensive adapter. It may hide a serious bug in the legacy API. In many production systems, duplicate completion should be logged, asserted, or treated as an integration failure.

### Better redesign for a multi-event API

If multiple success callbacks are valid, the correct abstraction is a stream:

```swift
func values() -> AsyncThrowingStream<Int, any Error> {
    AsyncThrowingStream { continuation in
        legacy { result in
            switch result {
            case .success(let value):
                continuation.yield(value)

            case .failure(let error):
                continuation.finish(throwing: error)
            }
        }

        // Only valid for this toy API because `legacy` emits synchronously.
        // A real multi-event API needs an explicit completion/cancel callback.
        continuation.finish()
    }
}
```

Consumer:

```swift
for try await value in values() {
    print(value)
}
```

Output for this toy `legacy`:

```text
1
2
```

### Why this fix is correct

The stream version matches the shape of the legacy API:

```text
legacy emits multiple values → AsyncThrowingStream emits multiple values
```

The one-shot continuation version mismatches the shape:

```text
legacy emits multiple values → async throws -> Int returns once
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Plain `withCheckedThrowingContinuation`|Legacy API guarantees exactly one callback|Simple, idiomatic, but relies on a real API contract|
|One-shot guarded continuation|Legacy API is supposed to be one-shot but may duplicate due to bugs|Prevents crash, but may hide upstream correctness problems|
|`AsyncStream` / `AsyncThrowingStream`|Legacy API emits multiple values or events|More lifecycle/cancellation handling required|
|Actor wrapper|Callback state is complex and cross-thread|More overhead and async API design complexity|
|Unsafe continuation|Proven hot path where checked overhead matters|Removes diagnostics; rarely justified in app code|

---

## 6. Exercise

### Problem

Wrap a legacy completion-handler API and explain how you guarantee exactly one resume.

### Bad / naive version

```swift
func loadUser(id: User.ID) async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        legacyClient.loadUser(id: id) { user, error in
            if let user {
                continuation.resume(returning: user)
            }

            if let error {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

### What is wrong?

```text
If both user and error are non-nil → resumes twice.
If both user and error are nil     → never resumes.
If callback can be called twice    → resumes twice.
If task is cancelled               → legacy operation may keep running.
```

This adapter does not define the illegal callback states. It just hopes they do not happen.

### Improved version for a true one-shot completion API

```swift
enum LoadUserError: Error {
    case invalidCallbackState
}

func loadUser(id: User.ID) async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        legacyClient.loadUser(id: id) { user, error in
            switch (user, error) {
            case let (user?, nil):
                continuation.resume(returning: user)

            case let (nil, error?):
                continuation.resume(throwing: error)

            case (.some, .some), (nil, nil):
                continuation.resume(throwing: LoadUserError.invalidCallbackState)
            }
        }
    }
}
```

This guarantees exactly one resume **assuming the completion itself is called exactly once**.

### Improved version with defensive duplicate-callback handling

```swift
import Synchronization

enum LoadUserError: Error {
    case invalidCallbackState
}

private enum OneShotState<T: Sendable> {
    case pending(CheckedContinuation<T, any Error>)
    case resumed
}

private final class OneShot<T: Sendable>: Sendable {
    private let state: Mutex<OneShotState<T>>

    init(_ continuation: CheckedContinuation<T, any Error>) {
        self.state = Mutex(.pending(continuation))
    }

    func resume(returning value: T) {
        resume(with: .success(value))
    }

    func resume(throwing error: any Error) {
        resume(with: .failure(error))
    }

    private func resume(with result: Result<T, any Error>) {
        let continuation = state.withLock { state -> CheckedContinuation<T, any Error>? in
            switch state {
            case .pending(let continuation):
                state = .resumed
                return continuation

            case .resumed:
                return nil
            }
        }

        continuation?.resume(with: result)
    }
}

func loadUserDefensively(id: User.ID) async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        let oneShot = OneShot<User>(continuation)

        legacyClient.loadUser(id: id) { user, error in
            switch (user, error) {
            case let (user?, nil):
                oneShot.resume(returning: user)

            case let (nil, error?):
                oneShot.resume(throwing: error)

            case (.some, .some), (nil, nil):
                oneShot.resume(throwing: LoadUserError.invalidCallbackState)
            }
        }
    }
}
```

### Why this is better

- The callback state is exhaustively handled.
- Invalid `(user, error)` combinations are converted into a real failure.
- Duplicate callbacks cannot resume the continuation twice.
- The one-shot state is protected by `Mutex`, so concurrent callbacks do not race.
- The adapter’s semantic contract is explicit.

Still, this wrapper only solves duplicate resume safety. It does **not** automatically cancel the legacy operation. Add cancellation if the legacy API supports it.

---

## 7. Production guidance

Use continuations in production when:

```text
- You are wrapping a true one-shot callback API.
- You are migrating old completion-handler APIs to async/await.
- You need to bridge Objective-C, C, delegate, or SDK APIs into Swift concurrency.
- You can clearly prove every path resumes exactly once.
```

Be careful when:

```text
- The callback uses optional value + optional error.
- The completion can be called synchronously.
- The completion can be called from arbitrary queues.
- Cancellation must stop underlying work.
- The legacy owner can deallocate before calling back.
- The callback can fire more than once.
```

Avoid continuations when:

```text
- The API emits multiple values.
- There is no completion signal.
- You need backpressure.
- You need long-lived observation.
- You are wrapping a delegate stream, websocket, notification observer, progress callback, or location updates.
```

Debugging checklist:

```text
Did every path resume?
Can any path resume twice?
Can success and failure both happen?
Can neither success nor failure happen?
Can callback happen synchronously before setup completes?
Can callback race with cancellation?
Can callback race with deallocation?
Should this be AsyncSequence instead?
Is the continuation stored? If yes, who owns and clears it?
Is duplicate callback a bug that should be logged/asserted?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Use `withCheckedContinuation` to turn callbacks into async/await, and call `resume` in the completion handler.

### Senior answer

> A checked continuation bridges one callback result into one suspended async task. I must resume exactly once on every path. I need to handle success, failure, invalid callback states, no-callback paths, duplicate callbacks, and cancellation. If the callback can fire multiple times, I should use `AsyncStream` instead.

### Staff-level answer

> Continuations are an interoperability boundary, so I treat them as a small state machine with a proof obligation. I preserve the semantic shape of the legacy API: one-shot completions become `async`, multi-event callbacks become `AsyncSequence`, cancellable operations connect to task cancellation, and delegate lifetimes get explicit termination cleanup. I do not silence continuation misuse; I design the adapter so duplicate, missing, racing, and invalid callback states are either impossible, synchronized, logged, or surfaced as errors.

Staff-level questions to ask:

```text
What is the exact callback contract?
Is the callback guaranteed once, at most once, or many times?
Can it call back synchronously?
Can it call back concurrently?
What owns the operation while the task is suspended?
How does cancellation propagate?
What happens if the legacy object deallocates?
Should duplicate completion be ignored, logged, asserted, or treated as fatal?
Is AsyncSequence the more honest abstraction?
```

---

## 9. Interview-ready summary

A continuation is a one-shot bridge from callback-based code into Swift async/await. `withCheckedContinuation` and `withCheckedThrowingContinuation` suspend the current async task and give legacy code a handle to resume it later. The core rule is exactly-once resume on every execution path. Zero resumes leave the task suspended; multiple resumes are a correctness bug and checked continuations trap or warn at runtime. For one-shot completion handlers, continuations are appropriate. For multi-callback APIs like delegates, progress, notifications, websockets, or location updates, `AsyncStream` / `AsyncThrowingStream` is usually the correct abstraction. In production, the hard part is not calling `resume`; it is proving the adapter handles success, failure, invalid states, duplicate callbacks, cancellation, threading, and lifetime correctly.

---

## 10. Flashcards

Q: What is the core invariant of `CheckedContinuation`?  
A: Resume exactly once on every execution path.

Q: What happens if a checked continuation is resumed twice?  
A: Runtime trap: Swift reports continuation misuse.

Q: What happens if a continuation is never resumed?  
A: The awaiting task remains suspended, and resources held by that suspended task can leak.

Q: When should you use `withCheckedThrowingContinuation` instead of `withCheckedContinuation`?  
A: When the async wrapper needs to throw, usually because the legacy API reports success or failure.

Q: Is a continuation a cancellation handle?  
A: No. It represents one suspension point. You must separately connect task cancellation to the underlying legacy operation.

Q: Why is a multi-callback API a poor fit for a single continuation?  
A: A continuation can return or throw once, while a multi-callback API emits many values.

Q: What should you use for repeated events?  
A: `AsyncStream` or `AsyncThrowingStream`.

Q: Does `resume` immediately continue the async function inline?  
A: No. It makes the task schedulable; the executor resumes it according to concurrency scheduling.

Q: Why is a plain `var didResume = false` often insufficient?  
A: If callbacks can arrive concurrently, the flag is a data race and can still allow duplicate resumes.

Q: What is the staff-level way to approach continuations?  
A: Treat the adapter as a small state machine with explicit ownership, cancellation, threading, and exactly-once semantics.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D12 — Synchronization library, mutexes, and choosing between actors and locks]]
- [[F1 — Testing async and concurrent code]]
- [[F2 — Diagnostics and LLDB]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist, D9 — Continuations section.
- Apple Developer Documentation — `CheckedContinuation`: exactly-once resume rule. ([Apple Developer](https://developer.apple.com/documentation/swift/checkedcontinuation?utm_source=chatgpt.com "CheckedContinuation | Apple Developer Documentation"))
- Apple Developer Documentation — `withCheckedThrowingContinuation(isolation:function:_:)`. ([Apple Developer](https://developer.apple.com/documentation/swift/withcheckedthrowingcontinuation%28isolation%3Afunction%3A_%3A%29?utm_source=chatgpt.com "withCheckedThrowingContinuati..."))
- Apple Developer Documentation — `CheckedContinuation.resume(returning:)`: repeated resume traps. ([Apple Developer](https://developer.apple.com/documentation/swift/checkedcontinuation/resume%28returning%3A%29?utm_source=chatgpt.com "resume(returning:) | Apple Developer Documentation"))
- Swift Evolution SE-0300 — Continuations for interfacing async tasks with synchronous code. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md "swift-evolution/proposals/0300-continuation.md at main · swiftlang/swift-evolution · GitHub"))