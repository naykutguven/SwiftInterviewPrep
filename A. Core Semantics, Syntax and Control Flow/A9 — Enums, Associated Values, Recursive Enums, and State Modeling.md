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

**Goal:** Represent refresh-with-previous-data without permitting contradictory workflow flags.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An enum stores exactly one case with that case's payload. Cases constrain shape; they do not validate arbitrary payloads, authorize transitions, or execute effects. Decide which combinations the product needs before replacing flags. Loading while showing an earlier successful value can be a valid refresh state. [Enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/)

## Rules and pitfalls

### Separate representation from transitions

Use cases to exclude impossible combinations and transition methods to reject stale or invalid events. Keep effect execution, such as starting or cancelling a task, under an explicit owner. A request identity should distinguish repeated attempts for the same input.

**Common pitfall:** A loaded/loading/failed enum that discards previous data cannot represent a valid stale-content refresh. Put previous data on the loading and failed cases when the product retains it.

### Respect payload ownership

Copying a copyable enum copies its payload values, including shared class references. Indirect cases break recursive inline layout; they do not make arbitrary graph operations iterative or guarantee a cost. A noncopyable payload can require noncopyable enum ownership.

**Common pitfall:** Assuming the enum validates associated values leaves negative retries or mismatched IDs constructible. Introduce validated domain types or guarded construction where those invariants matter. See [[C8 — Copyable, noncopyable types, and `~Copyable`]].

## Interview questions

### Which combinations should this product preserve, and which should the type exclude?

Preserve combinations required by the product, such as refreshing with visible previous data or refresh failure with retained content. Exclude contradictory shapes such as simultaneously idle and failed. Cases should require the data each state needs; construction and transitions still validate IDs, payload domains, and event ordering.

### How can a retry timer or cancellation acknowledgment arrive after replacement, and what prevents it from changing the current state?

Timers and cancellation acknowledgments are asynchronous events and may arrive after replacement. Tag them with the operation identity and apply them only if that operation is still current. Cancellation requests stop permission to publish when the model says so, but do not prove the underlying operation stopped. Old cleanup must not clear a replacement's handle.

## Focused exercise

**Task:** Represent loading, loaded, and failed states while retaining previous data where required. Check allowed transitions and excluded combinations.

**Acceptance checks:**

- Initial loading has no previous data; refresh retains the previous snapshot.
- Failure retains that snapshot; empty success remains distinct from absence.
- An old completion cannot replace the current request.

**Solution:**

Standalone synchronous transition model; effects are intentionally external:

```swift
enum State: Equatable {
    case idle
    case loading(previous: [Int]?)
    case loaded([Int])
    case failed(previous: [Int]?, message: String)
}
struct Model {
    private(set) var state: State = .idle
    private(set) var request = 0
    mutating func begin() -> Int {
        let previous: [Int]?
        switch state {
        case .loaded(let values): previous = values
        case .loading(let values), .failed(let values, _): previous = values
        case .idle: previous = nil
        }
        request += 1
        state = .loading(previous: previous)
        return request
    }
    mutating func succeed(_ values: [Int], request id: Int) {
        guard id == request, case .loading = state else { return }
        state = .loaded(values)
    }
    mutating func fail(_ message: String, request id: Int) {
        guard id == request, case .loading(let old) = state else { return }
        state = .failed(previous: old, message: message)
    }
}
var model = Model()
let first = model.begin()
model.succeed([1], request: first)
let refresh = model.begin()
precondition(model.state == .loading(previous: [1]))
model.fail("offline", request: refresh)
precondition(model.state == .failed(previous: [1], message: "offline"))
let old = model.begin()
let current = model.begin()
model.succeed([0], request: old)
precondition(model.state == .loading(previous: [1]))
model.succeed([], request: current)
precondition(model.state == .loaded([]))
```

**Reasoning:** The enum excludes contradictory flags while the model enforces request freshness. This counter is a bounded teaching fixture; production identity can use a unique token and needs an exhaustion policy if numeric revisions are used. Different-user refresh also needs an explicit rule before retaining previous data.

## In practice

**Choose it when:** Model mutually exclusive workflow alternatives, with transitions next to their invariants.

**Investigate by:** Enumerate events, including late success, retry, cancellation acknowledgment, and cleanup.

**Strong evidence:** Test controlled event orders and demonstrate that valid stale-content states remain representable.

### Recursive and noncopyable representation — optional

#### Recursive inline storage needs a break in the layout cycle

```swift
enum EvaluationError: Error { case overflow }

indirect enum Expression: Equatable, Sendable {
    case number(Int)
    case add(Expression, Expression)
    case multiply(Expression, Expression)
}

func evaluate(_ expression: Expression) throws(EvaluationError) -> Int {
    switch expression {
    case let .number(value):
        return value
    case let .add(left, right):
        let a = try evaluate(left), b = try evaluate(right)
        let result = a.addingReportingOverflow(b)
        guard !result.overflow else { throw .overflow }
        return result.partialValue
    case let .multiply(left, right):
        let a = try evaluate(left), b = try evaluate(right)
        let result = a.multipliedReportingOverflow(by: b)
        guard !result.overflow else { throw .overflow }
        return result.partialValue
    }
}
```

The expression add(number(2), multiply(number(3), number(4))) evaluates to 14. Overflow throws the modeled error instead of silently wrapping or relying on optimization-dependent arithmetic checks. The evaluator is recursive: it is not suitable for arbitrary adversarial depth without an explicit depth limit or iterative traversal.

This direct recursive definition is **intentionally rejected**:

```swift
enum InvalidRecursive {
    case end
    case next(InvalidRecursive)
}
```

You can place indirect only on the recursive case. A collection can also supply the required storage indirection without an indirect enum declaration:

```swift
enum Tree: Equatable, Sendable {
    case leaf(Int)
    indirect case branch(Tree, Tree)
}

enum Menu: Equatable, Sendable {
    case item(String)
    case group([Menu])
}
```

Tree's recursive branch and Menu's Array payload both have finite representations. indirect adds representation indirection while preserving enum value behavior. Do not infer an exact heap-allocation count, stack layout, deep copy, or performance ranking from the keyword. Optimizations and reference payloads still matter. [Recursive enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/#Recursive-Enumerations)

#### A value type is not necessarily Copyable

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

func leaseSerial(_ state: borrowing LeaseState) -> Int? {
    switch state {
    case .empty: nil
    case let .ready(lease): lease.serial
    }
}

func finishLease(_ state: consuming LeaseState) {
    switch consume state {
    case .empty: break
    case let .ready(lease): lease.finish()
    }
}
```

The same LeaseState can be borrowed repeatedly to inspect its serial. finishLease consumes the state and its Lease payload; the synthetic log records destruction once. The Lease is a lifetime teaching fixture, not an OS resource. Noncopyable support and borrowing/consuming rules predate 6.4 and remain relevant to this baseline. [SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md), [SE-0427](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md)

With the declarations above, this function is **intentionally rejected during ownership checking**:

```swift
func invalidReuse(_ state: consuming LeaseState) -> Int? {
    finishLease(consume state)
    return leaseSerial(state)
}
```

A containing enum with a Lease payload must also admit noncopyability. A generic parameter's default Copyable requirement does not disappear merely because its argument is an enum. C7/C8 contain the broader ownership rules, and B7 records a current noncopyable-enum synthesis limitation. Do not assume generalized protocol requirements imply every automatic conformance is synthesized.


### Controlled cancellation and retry lifecycle — optional

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
        case let .inFlight(id, _), let .cancelling(id, _): [.cancelRequest(id)]
        case let .waitingToRetry(ticket, _): [.cancelRetry(ticket)]
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
        guard case let .inFlight(current, _) = state, current == requestID else { return }
        state = .loaded(value)
    }

    func fail(requestID: RequestID, with failure: RequestFailure) {
        guard case let .inFlight(current, attempt) = state, current == requestID else { return }
        let plan: RetryPlan
        if case let .retry(delay) = failure.recovery, attempt < maximumAttempts {
            plan = .available(nextAttempt: attempt + 1, delay: delay)
        } else {
            plan = .none
        }
        state = .failed(failure, retryPlan: plan)
    }

    func retry() -> [RequestEffect] {
        guard case let .failed(_, .available(nextAttempt, delay)) = state else { return [] }
        let ticket = RequestID()
        state = .waitingToRetry(ticket: ticket, nextAttempt: nextAttempt)
        return [.waitForRetry(ticket: ticket, delay: delay)]
    }

    func retryDelayElapsed(ticket: RequestID) -> [RequestEffect] {
        guard case let .waitingToRetry(current, attempt) = state, current == ticket else { return [] }
        state = .inFlight(requestID: ticket, attempt: attempt)
        return [.start(requestID: ticket, attempt: attempt)]
    }

    func cancel(reason: CancellationReason) -> [RequestEffect] {
        switch state {
        case let .inFlight(id, _):
            state = .cancelling(requestID: id, reason: reason)
            return [.cancelRequest(id)]
        case let .waitingToRetry(ticket, _):
            state = .cancelled(reason: reason)
            return [.cancelRetry(ticket)]
        case .idle, .loaded, .failed, .cancelling, .cancelled:
            return []
        }
    }

    func finishCancellation(requestID: RequestID) {
        guard case let .cancelling(current, reason) = state, current == requestID else { return }
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

This complete state/effect model intentionally contains no hidden network or Task implementation. Its deterministic tests drive every event and inspect commands; D2/F1 cover actual task cancellation and joining. Retry eligibility and a finite attempt count do not establish request idempotency, server retry policy, or backoff/jitter requirements. Those belong to the operation contract.

A short stale-completion probe, using the model above:

```swift
@MainActor
func lifecycleProbe() -> [Int]? {
    let model = RequestModel<[Int]>()
    guard case let .start(first, _)? = model.start().last else { return nil }
    _ = model.cancel(reason: .userInitiated)
    guard case let .start(second, _)? = model.start().last else { return nil }
    model.complete(requestID: first, with: [1])
    model.finishCancellation(requestID: first)
    model.complete(requestID: second, with: [2])
    guard case let .loaded(value) = model.state else { return nil }
    return value
}
```

It returns [2]. Replacement yields a distinct token, and neither the first result nor its cancellation acknowledgment replaces the current request. The full fixture additionally tests delayed retries, attempt exhaustion, zero-delay eligibility, nonretryable failures, cancellation-reason preservation, cross-model tokens, late/duplicate timer events, and resets from every state shape.

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
