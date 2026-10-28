---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - enums
  - state-modeling
---

# A9 — Enums, Associated Values, Recursive Enums, and State Modeling

**Learning goals:**

- Represent mutually exclusive states while preserving valid refresh-with-previous-data states.
- Separate case shape, payload validation, transition validation, and execution of effects.
- Explain recursive storage and the ownership of reference and noncopyable payloads.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] for value/reference behavior, [[A4 — Optionals and Nil Modeling]] for absence, [[A5 — Pattern Matching and Exhaustive Control Flow]] for pattern matching, and [[A13 — Error handling and typed throws]] for errors. The lifecycle extension uses [[D2 — Task Hierarchy, Cancellation, and Priorities]] cancellation concepts.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Separate isLoading, hasError, and value flags can represent combinations the product forbids. An enum instead stores exactly one **case**, with the data that case needs as its **associated value**. loaded(values) cannot simultaneously be idle. However, loading while displaying the previous successful value may be a valid refresh state, so the model must preserve it deliberately.

An enum constrains shape. It does not validate every Int payload, authorize every transition, start a request, or cancel a task. Those are separate responsibilities. First choose valid states, then define which events may change them and who executes any resulting effects. [Enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Cases carry only the data they need

Shared setup named LoadState. Value is a generic payload type; the same state shape can hold different value models. visibleValue extracts the data the UI is allowed to display.

```swift
enum LoadState<Value> {
    case idle
    case loading(previous: Value?)
    case loaded(Value)
    case failed(message: String, previous: Value?)
    var visibleValue: Value? {
        switch self {
        case .idle: nil
        case .loading(let previous), .failed(_, let previous): previous
        case .loaded(let value): value
        }
    }
    var isLoading: Bool { if case .loading = self { true } else { false } }
    var errorMessage: String? {
        if case .failed(let message, _) = self { message } else { nil }
    }
}
extension LoadState: Equatable where Value: Equatable {}
let state = LoadState<[Int]>.loading(previous: [1, 2])
precondition(state.isLoading && state.visibleValue == [1, 2])
precondition(state.errorMessage == nil)
```

**Walkthrough:** The loading case can retain content without claiming the refresh finished. It cannot also contain an error message: only failed carries one. Payload validity is still a separate question.

### An enum copy follows its payload semantics

Reference payloads remain shared; value payloads retain their normal independent value behavior.

```swift
final class Payload { var count = 1 }
enum ReferenceState { case loaded(Payload) }
let object = Payload()
let first = ReferenceState.loaded(object)
let second = first
object.count = 9
if case .loaded(let value) = second { precondition(value.count == 9) }
enum ValueState { case loaded([Int]) }
let saved = ValueState.loaded([1])
var edited = saved
if case .loaded(var values) = edited {
    values.append(2)
    edited = .loaded(values)
}
precondition({ if case .loaded(let values) = saved { values == [1] } else { false } }())
```

**Walkthrough:** The case tag does not clone a class object. By contrast, extracting and updating the array value leaves saved’s sequence intact. [[A1 — Value semantics, reference semantics, identity, and copy-on-write|A1]] explains the underlying storage contract.

## Rules and pitfalls

### Do not exclude a legitimate refresh state

**Common pitfall:** Replacing loaded content with a payload-free loading case when the UI should keep showing it.

**Mistake:** Standalone runnable example.

```swift
enum State {
    case idle, loading
    case loaded([Int])
    case failed(String)
}
var state = State.loaded([1])
state = .loading
let visible: [Int]? = { if case .loaded(let values) = state { values } else { nil } }()
precondition(visible == nil)
```

**Consequence:** Beginning refresh loses the only representation of the previous data.

**Why:** The enum excluded a valid product state rather than merely excluding contradictions.

**Correction:** Use LoadState from Essential mechanics and carry the previous snapshot into loading and failure.

```swift
var current = LoadState<[Int]>.loaded([1])
current = .loading(previous: current.visibleValue)
precondition(current.isLoading && current.visibleValue == [1])
current = .failed(message: "offline", previous: current.visibleValue)
precondition(!current.isLoading && current.visibleValue == [1])
precondition(current.errorMessage == "offline")
```

**What changes:** The state now records both the refresh phase and the content that remains valid to display. Retaining another user’s data would need a different policy.

### Case shape does not validate a payload

**Common pitfall:** Assuming an enum case makes every associated retry count valid.

**Mistake:** Standalone runnable example.

```swift
enum State { case retry(attempt: Int) }
let state = State.retry(attempt: -1)
if case .retry(let attempt) = state { precondition(attempt < 0) }
```

**Consequence:** A negative attempt is constructible.

**Why:** The case constrains the payload’s type to Int, not its allowed domain.

**Correction:** Put the invariant into a validated value and require it in the case.

```swift
struct Attempt: Equatable {
    let number: Int
    init?(_ number: Int) {
        guard number > 0 else { return nil }
        self.number = number
    }
}
enum State { case retry(Attempt) }
precondition(Attempt(-1) == nil && Attempt(0) == nil)
let state = State.retry(Attempt(1)!)
if case .retry(let attempt) = state { precondition(attempt.number == 1) }
```

**What changes:** All callers using this initializer receive the same validation. Keeping a transition setter private additionally controls when a retry may begin; it solves a different invariant.

## Worked example

### A failed refresh preserves the prior snapshot

**Setup:** Prepend LoadState setup. This is a synchronous value-model trace.

**Predict:** What changes in the original snapshot after refreshing and failing?

```swift
var current = LoadState<[Int]>.loaded([1, 2])
let original = current
current = .loading(previous: current.visibleValue)
precondition(current.isLoading && current.visibleValue == [1, 2])
current = .failed(message: "offline", previous: current.visibleValue)
precondition(current.errorMessage == "offline" && current.visibleValue == [1, 2])
precondition(original == .loaded([1, 2]))
```

**Outcome:** The current state is failed with retained content; the original is still loaded.

**Why:** Each enum assignment creates a new logical state. The array payload has value behavior, so the original remains a valid previous snapshot.

## Interview questions

### Which combinations should this product preserve, and which should the type exclude?

**Answer:** Preserve legitimate loading-with-content and failure-with-content states. Exclude contradictory phases and require each case’s necessary data. Validate payloads and event order separately.

**Example:** Prepend LoadState setup.

```swift
let initial = LoadState<[Int]>.loading(previous: nil)
let refresh = LoadState<[Int]>.loading(previous: [1])
let emptySuccess = LoadState<[Int]>.loaded([])
let failure = LoadState<[Int]>.failed(message: "offline", previous: [1])
precondition(initial.visibleValue == nil)
precondition(refresh.visibleValue == [1])
precondition(emptySuccess.visibleValue == [])
precondition(failure.visibleValue == [1] && failure.errorMessage == "offline")
```

**Walkthrough:** Initial absence, retained refresh data, empty success, and failed refresh are distinct. No value has two case tags at once.

**Boundary:** This representation does not stop a stale completion from assigning a different case. It also does not validate the contents of [Int] or decide whether retained data belongs to the current user.

### How can a retry timer or cancellation acknowledgment arrive after replacement, and what prevents publication?

**Answer:** Requesting cancellation does not prove the operation stopped. An event can already be queued. Give each attempt or timer a distinct identity and accept its event only while that identity and phase remain current.

**Example:** Standalone runnable example.

```swift
final class ID {}
enum Phase {
    case idle
    case waiting(ID)
    case running(ID)
    case cancelling(ID)
    case loaded(Int)
}
struct Model {
    private(set) var phase: Phase = .idle
    mutating func begin() -> ID {
        let id = ID()
        phase = .running(id)
        return id
    }
    mutating func waitToRetry() -> ID {
        let ticket = ID()
        phase = .waiting(ticket)
        return ticket
    }
    mutating func timer(_ ticket: ID) {
        guard case .waiting(let current) = phase, current === ticket else { return }
        phase = .running(ticket)
    }
    mutating func acknowledgeCancellation(_ id: ID) {
        guard case .cancelling(let current) = phase, current === id else { return }
        phase = .idle
    }
    mutating func complete(_ value: Int, id: ID) {
        guard case .running(let current) = phase, current === id else { return }
        phase = .loaded(value)
    }
}
var model = Model()
let old = model.begin()
let timer = model.waitToRetry()
let replacement = model.begin()
model.timer(timer)
model.complete(1, id: old)
model.acknowledgeCancellation(old)
model.complete(2, id: replacement)
if case .loaded(2) = model.phase {} else { preconditionFailure() }
```

**Walkthrough:** The old timer no longer matches a waiting phase; the old completion and acknowledgment no longer match their required phases and identities. Only the replacement may publish 2.

**Boundary:** The simplified model illustrates event filtering and does not execute effects. The complete optional lifecycle below adds cancellation requests, acknowledgment, retry budgets, and delay commands.

### Extension: why does a recursive enum need indirect storage?

**Answer:** A directly recursive stored payload would require an infinitely large inline layout. indirect breaks that layout cycle while preserving enum value behavior. A collection payload can supply the indirection too.

**Example:** Standalone runnable example.

```swift
enum EvaluationError: Error { case overflow }
indirect enum Expression {
    case number(Int)
    case add(Expression, Expression)
}
func evaluate(_ expression: Expression) throws(EvaluationError) -> Int {
    switch expression {
    case .number(let value): return value
    case .add(let left, let right):
        let a = try evaluate(left)
        let b = try evaluate(right)
        let result = a.addingReportingOverflow(b)
        guard !result.overflow else { throw .overflow }
        return result.partialValue
    }
}
enum Tree {
    case leaf(Int)
    indirect case branch(Tree, Tree)
}
enum Menu {
    case item(String)
    case group([Menu])
}
let value = try evaluate(.add(.number(2), .add(.number(3), .number(4))))
precondition(value == 9)
do {
    _ = try evaluate(.add(.number(Int.max), .number(1)))
    preconditionFailure("Expected overflow")
} catch { precondition(error == .overflow) }
```

**Walkthrough:** The evaluator follows the tree and checks arithmetic overflow explicitly. Tree uses indirect only on its recursive case; Menu stores recursion behind Array’s finite representation. [Recursive enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/#Recursive-Enumerations)

**Boundary:** indirect does not guarantee allocation counts, deep copies, or iterative evaluation. Adversarial depth needs a bound or iterative traversal to avoid exhausting the call stack.

Separate **intentionally rejected** recursive layout:

```swift
enum InvalidRecursive {
    case end
    case next(InvalidRecursive)
}
```

### Extension: how does a noncopyable payload change enum ownership?

**Answer:** The containing enum must also permit noncopyability. You can borrow it repeatedly for inspection, then consume it once to finish the payload’s lifetime.

**Example:** Standalone runnable example.

```swift
final class ReleaseLog { var serials: [Int] = [] }
struct Lease: ~Copyable {
    let serial: Int
    let log: ReleaseLog
    consuming func finish() {}
    deinit { log.serials.append(serial) }
}
enum LeaseState: ~Copyable {
    case empty
    case ready(Lease)
}
func serial(_ state: borrowing LeaseState) -> Int? {
    switch state {
    case .empty: nil
    case .ready(let lease): lease.serial
    }
}
func finish(_ state: consuming LeaseState) {
    switch consume state {
    case .empty: break
    case .ready(let lease): lease.finish()
    }
}
func checkLease() {
    let log = ReleaseLog()
    let state = LeaseState.ready(Lease(serial: 7, log: log))
    precondition(serial(state) == 7 && serial(state) == 7)
    finish(consume state)
    precondition(log.serials == [7])
}
checkLease()
```

**Walkthrough:** Two borrows observe one lease; consuming the state destroys the payload once and records 7. The log is a lifetime fixture, not an OS resource. [SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)

**Boundary:** A generic parameter still defaults to Copyable unless declared otherwise. Generalized protocol requirements do not guarantee automatic conformance synthesis for all noncopyable enums; [[B7 — Synthesized conformances and semantic correctness|B7]]/[[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] document the limits.

Prepend the declarations from that answer. This **compile-only, intentionally rejected** function uses consumed state:

```swift
func invalidReuse(_ state: consuming LeaseState) -> Int? {
    finish(consume state)
    return serial(state)
}
```

## Focused exercise

**Task:** Represent initial load, refresh, success, and failure while rejecting stale completions.

**Acceptance checks:**

- Initial loading has no previous data; refresh and failure retain it.
- Empty success remains distinct from absence.
- A replaced request cannot publish; only a matching loading request accepts success or failure.

**Solution:** Prepend LoadState setup. Each ID is a new object identity, avoiding integer-generation exhaustion in this teaching model.

```swift
final class RequestID {}
struct Model {
    private(set) var state: LoadState<[Int]> = .idle
    private var current: RequestID?
    mutating func begin() -> RequestID {
        let id = RequestID()
        current = id
        state = .loading(previous: state.visibleValue)
        return id
    }
    mutating func succeed(_ values: [Int], id: RequestID) {
        guard current === id, case .loading = state else { return }
        current = nil
        state = .loaded(values)
    }
    mutating func fail(_ message: String, id: RequestID) {
        guard current === id, case .loading(let previous) = state else { return }
        current = nil
        state = .failed(message: message, previous: previous)
    }
}
var model = Model()
let first = model.begin()
precondition(model.state == .loading(previous: nil))
model.succeed([1], id: first)
let refresh = model.begin()
precondition(model.state == .loading(previous: [1]))
model.fail("offline", id: refresh)
precondition(model.state == .failed(message: "offline", previous: [1]))
let old = model.begin()
let current = model.begin()
model.succeed([0], id: old)
model.fail("old failure", id: old)
precondition(model.state == .loading(previous: [1]))
model.succeed([], id: current)
model.fail("duplicate", id: current)
precondition(model.state == .loaded([]))
```

**Reasoning:** The enum constrains state shape; private transition methods enforce identity and phase. Clearing current after success/failure rejects duplicate terminal events. No task is started here: an operation owner must execute work, deliver events under an isolation policy, and cancel/join it as needed.

## In practice

**Choose it when:** Use cases for mutually exclusive alternatives and transition methods for event-dependent invariants.

**Investigate by:** List late success, retry, cancellation acknowledgment, reset, and cleanup events; drive them in controlled orders.

**Strong evidence:** Keep valid stale-content states representable while proving stale events cannot publish or clear a replacement’s work handle.

### Controlled cancellation and retry lifecycle — optional

This larger model applies the same state/transition/effect separation to retries. Read the core examples first. RequestLifecycle is the snapshot; RequestModel owns authorized transitions; RequestEffect is a command for the separate operation owner. The first block is **compile-only shared setup**, named RequestModel setup. The subsequent probes prepend it.

**Problem:** Model initial work, delayed retry, cancellation, replacement, and reset. Reject stale completions and timer events. Keep state mutation on MainActor and return explicit commands for the operation owner to execute. Retry attempts must increase within a limit; cancellation acknowledgment must retain the originally chosen reason.

```swift
import Foundation

final class RequestID: Hashable, Sendable {
    fileprivate init() {}
    static func == (lhs: RequestID, rhs: RequestID) -> Bool { lhs === rhs }
    func hash(into hasher: inout Hasher) { hasher.combine(ObjectIdentifier(self)) }
}

struct RetryDelay: Equatable, Sendable {
    let duration: Duration
    init?(_ duration: Duration) {
        guard duration >= .zero else { return nil }
        self.duration = duration
    }
}

enum Recovery: Equatable, Sendable {
    case never
    case retry(after: RetryDelay)
}

struct RequestFailure: Error, Equatable, Sendable {
    let message: String
    let recovery: Recovery
}

enum CancellationReason: Equatable, Sendable {
    case userInitiated, viewDisappeared
}

enum RetryPlan: Equatable, Sendable {
    case none
    case available(nextAttempt: Int, delay: RetryDelay)
}

enum RequestLifecycle<Value: Sendable>: Sendable {
    case idle
    case inFlight(requestID: RequestID, attempt: Int)
    case loaded(Value)
    case failed(RequestFailure, retryPlan: RetryPlan)
    case waitingToRetry(ticket: RequestID, nextAttempt: Int)
    case cancelling(requestID: RequestID, reason: CancellationReason)
    case cancelled(reason: CancellationReason)
}

enum RequestEffect: Equatable, Sendable {
    case start(requestID: RequestID, attempt: Int)
    case cancelRequest(RequestID)
    case waitForRetry(ticket: RequestID, delay: RetryDelay)
    case cancelRetry(RequestID)
}

@MainActor
final class RequestModel<Value: Sendable> {
    private(set) var state: RequestLifecycle<Value> = .idle
    private let maximumAttempts: Int

    init(maximumAttempts: Int = 3) {
        precondition(maximumAttempts > 0)
        self.maximumAttempts = maximumAttempts
    }

    private func cancellationEffects() -> [RequestEffect] {
        switch state {
        case .inFlight(let id, _), .cancelling(let id, _): [.cancelRequest(id)]
        case .waitingToRetry(let ticket, _): [.cancelRetry(ticket)]
        case .idle, .loaded, .failed, .cancelled: []
        }
    }

    func start() -> [RequestEffect] {
        let cancel = cancellationEffects()
        let id = RequestID()
        state = .inFlight(requestID: id, attempt: 1)
        return cancel + [.start(requestID: id, attempt: 1)]
    }

    func complete(requestID: RequestID, with value: Value) {
        guard case .inFlight(let current, _) = state, current == requestID else { return }
        state = .loaded(value)
    }

    func fail(requestID: RequestID, with failure: RequestFailure) {
        guard case .inFlight(let current, let attempt) = state, current == requestID else { return }
        let plan: RetryPlan
        if case .retry(let delay) = failure.recovery, attempt < maximumAttempts {
            plan = .available(nextAttempt: attempt + 1, delay: delay)
        } else {
            plan = .none
        }
        state = .failed(failure, retryPlan: plan)
    }

    func retry() -> [RequestEffect] {
        guard case .failed(_, .available(let nextAttempt, let delay)) = state else { return [] }
        let ticket = RequestID()
        state = .waitingToRetry(ticket: ticket, nextAttempt: nextAttempt)
        return [.waitForRetry(ticket: ticket, delay: delay)]
    }

    func retryDelayElapsed(ticket: RequestID) -> [RequestEffect] {
        guard case .waitingToRetry(let current, let attempt) = state, current == ticket else {
            return []
        }
        state = .inFlight(requestID: ticket, attempt: attempt)
        return [.start(requestID: ticket, attempt: attempt)]
    }

    func cancel(reason: CancellationReason) -> [RequestEffect] {
        switch state {
        case .inFlight(let id, _):
            state = .cancelling(requestID: id, reason: reason)
            return [.cancelRequest(id)]
        case .waitingToRetry(let ticket, _):
            state = .cancelled(reason: reason)
            return [.cancelRetry(ticket)]
        case .idle, .loaded, .failed, .cancelling, .cancelled:
            return []
        }
    }

    func finishCancellation(requestID: RequestID) {
        guard case .cancelling(let current, let reason) = state, current == requestID else {
            return
        }
        state = .cancelled(reason: reason)
    }

    func reset() -> [RequestEffect] {
        let effects = cancellationEffects()
        state = .idle
        return effects
    }
}
```

RequestID uses a fresh object's identity for each initial request or retry ticket. Two models cannot accidentally accept each other's first integer generation, and an outstanding token keeps its identity alive. The tokens are local process identities, not durable IDs or serialized keys. Their private construction boundary and the model's private state setter prevent outside-file callers from supplying a new arbitrary state or attempt. The enum itself still allows independently constructed snapshots; its declaration is not a universal transition validator.

RetryDelay rejects negative Duration values. The model derives nextAttempt from its current attempt and maximumAttempts; checking attempt < maximumAttempts also makes increment safe at Int.max. A new start begins a new retry budget at attempt 1. A failure with recovery never, or one that reaches the limit, has no retry plan. Retrying moves to waitingToRetry and emits a delay command even for a zero delay; it does not start a request until the matching timer completion is delivered.

| Event | Accepted state and effect | What stale events do |
| --- | --- | --- |
| start | Any state; retire active work/timer and start a fresh identity at attempt 1 | Earlier results cannot match the new identity |
| complete / fail | Matching inFlight identity only | Ignored during cancellation, waiting, after reset, or for another identity |
| retry | Failed with an available plan; create one retry ticket | Repeated retry while waiting produces no command |
| retryDelayElapsed | Matching waiting ticket; start that ticket's attempt | An old/repeated/cancelled timer event does nothing |
| cancel | In-flight work enters cancelling; a waiting retry is retired | Repeated cancellation does not replace the stored reason |
| finishCancellation | Matching cancelling request; use its stored reason | An older acknowledgment cannot cancel newer work |
| reset | Retire current work/timer and return idle | Outstanding callbacks cannot leave idle |

The operation owner must execute these effects: retain task/timer handles by token, cancel the identified work, deliver events on MainActor, and join work when its lifecycle requires completion. retryDelayElapsed is an acknowledgment from that owner that the requested wait completed; the model does not read a clock to prove it. finishCancellation must be delivered when the owner has observed that operation finish. A cancelled UI state alone is not proof that every command or scheduled task has finished. Returning cancelRetry retires the ticket immediately so a late timer cannot restart work.

This complete state/effect model intentionally contains no hidden network or Task implementation. Its deterministic tests drive every event and inspect commands; [[D2 — Task Hierarchy, Cancellation, and Priorities|D2]]/[[F1 — Testing Strategy in Modern Swift|F1]] cover actual task cancellation and joining. Retry eligibility and a finite attempt count do not establish request idempotency, server retry policy, or backoff/jitter requirements. Those belong to the operation contract.

A short stale-completion probe, using the model above:

```swift
@MainActor
func lifecycleProbe() -> [Int]? {
    let model = RequestModel<[Int]>()
    guard case .start(let first, _)? = model.start().last else { return nil }
    _ = model.cancel(reason: .userInitiated)
    guard case .start(let second, _)? = model.start().last else { return nil }
    model.complete(requestID: first, with: [1])
    model.finishCancellation(requestID: first)
    model.complete(requestID: second, with: [2])
    guard case .loaded(let value) = model.state else { return nil }
    return value
}

precondition(lifecycleProbe() == [2])
```

It returns [2]. Replacement yields a distinct token, and neither the first result nor its cancellation acknowledgment replaces the current request. The historical lifecycle fixture additionally tested delayed retries, attempt exhaustion, zero-delay eligibility, nonretryable failures, cancellation-reason preservation, cross-model tokens, late/duplicate timer events, and resets from every state shape.

The following runnable checks prepend RequestModel setup. They exercise retry eligibility and timer retirement without running a clock or HTTP request. An effect is an instruction; only the operation owner can confirm actual completion.

```swift
@MainActor func checkRetryLifecycle() {
    precondition(RetryDelay(.seconds(-1)) == nil)
    let delay = RetryDelay(.zero)!
    let offline = RequestFailure(message: "offline", recovery: .retry(after: delay))
    let model = RequestModel<Int>(maximumAttempts: 2)
    guard case .start(let first, 1)? = model.start().last else { preconditionFailure() }
    model.fail(requestID: first, with: offline)
    guard case .waitForRetry(let ticket, let requestedDelay)? = model.retry().first
    else { preconditionFailure() }
    precondition(requestedDelay == delay)
    precondition(model.retry().isEmpty)
    guard case .start(let retry, 2)? = model.retryDelayElapsed(ticket: ticket).first
    else { preconditionFailure() }
    precondition(retry === ticket)
    precondition(model.retryDelayElapsed(ticket: ticket).isEmpty)
    model.fail(requestID: retry, with: offline)
    precondition(model.retry().isEmpty)  // Budget exhausted.

    guard case .start(let next, 1)? = model.start().last else { preconditionFailure() }
    model.fail(requestID: next, with: offline)
    guard case .waitForRetry(let retired, _)? = model.retry().first else { preconditionFailure() }
    precondition(model.cancel(reason: .viewDisappeared) == [.cancelRetry(retired)])
    precondition(model.retryDelayElapsed(ticket: retired).isEmpty)
    guard case .cancelled(.viewDisappeared) = model.state else { preconditionFailure() }

    guard case .start(let active, _)? = model.start().last else { preconditionFailure() }
    precondition(model.cancel(reason: .userInitiated) == [.cancelRequest(active)])
    precondition(model.cancel(reason: .viewDisappeared).isEmpty)
    model.complete(requestID: active, with: 99)
    model.finishCancellation(requestID: active)
    guard case .cancelled(.userInitiated) = model.state else { preconditionFailure() }

    guard case .start(let fresh, _)? = model.start().last else { preconditionFailure() }
    precondition(model.reset() == [.cancelRequest(fresh)])
    model.complete(requestID: fresh, with: 99)
    model.finishCancellation(requestID: fresh)
    guard case .idle = model.state else { preconditionFailure() }
    let other = RequestModel<Int>()
    guard case .start(let foreign, _)? = other.start().last else { preconditionFailure() }
    guard case .start(let own, _)? = model.start().last else { preconditionFailure() }
    model.complete(requestID: foreign, with: 99)
    model.complete(requestID: own, with: 2)
    guard case .loaded(2) = model.state else { preconditionFailure() }
}
checkRetryLifecycle()
```

**Walkthrough:** The first retry emits a wait command, and only its matching elapsed event starts attempt 2. Exhaustion removes retry eligibility. Cancelling a wait retires its ticket immediately; a late timer does nothing. Cancelling active work preserves its first reason, and reset or a token from another model cannot publish stale data. These checks verify transitions and emitted commands, not the execution of cancellation, real time, or server idempotency.

## References

### Related guides

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A4 — Optionals and Nil Modeling]]
- [[A5 — Pattern Matching and Exhaustive Control Flow]]
- [[A13 — Error handling and typed throws]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[B9 — Codable and Schema Evolution]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [Enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/)
- [Recursive enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/#Recursive-Enumerations)
- [SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [SE-0427](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md)
- [SE-0487: Nonexhaustive enums](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md)
