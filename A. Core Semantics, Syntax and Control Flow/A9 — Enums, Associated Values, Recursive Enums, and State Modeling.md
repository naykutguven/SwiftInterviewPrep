---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - enums
  - state-modeling
---
## 0. Rubric snapshot

**Rubric expectation**

Use enums to model finite state machines and make invalid states unrepresentable. The A9 rubric specifically focuses on enums, associated values, recursive enums, and state modeling.

**Caveats**

Multiple booleans often indicate that an enum is missing. Associated values should encode semantics, not anonymous blobs.

**You should be able to answer**

- Why is an enum often a better model than a set of optional properties?
- When do you need `indirect`, and what is the runtime implication?

**You should be able to do**

- Model a request lifecycle with cancel/retry semantics as an enum.
- Redesign a boolean/optional-based `LoadState` so impossible states become unrepresentable.

---

## 1. Core mental model

An enum is a **closed set of mutually exclusive cases**. A value of an enum is exactly one case at a time. That makes enums a strong fit for state modeling, because many real app states are mutually exclusive: a screen is idle, loading, loaded, failed, cancelled, or retrying. It should not be “loading and failed and loaded” unless your model intentionally allows that.

Associated values let each enum case carry only the data that makes sense for that case. Swift enumerations can store associated values of different types for different cases, which is what makes them stronger than plain string/status-code enums. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/?utm_source=chatgpt.com "Enumerations - Documentation | Swift.org"))

The key design move is to stop asking:

```text
Which properties happen to be nil right now?
```

and start asking:

```text
Which domain state is this value in?
```

With booleans and optionals, the compiler accepts many combinations that your domain does not. With an enum, the type system can reject entire classes of bugs because there is no representation for those invalid combinations.

Swift enums are value types, like structs. That means enum state is copied by value unless the associated payload contains reference types. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/?utm_source=chatgpt.com "Structures and Classes - Documentation | Swift.org")) This matters in view models, reducers, state stores, and SwiftUI because you can reason about state transitions as value replacement:

```swift
state = .loading
state = .loaded(items)
state = .failed(error)
```

Recursive enums are enums whose cases contain another value of the same enum type. Swift requires `indirect` for direct recursion so the compiler can insert a layer of indirection instead of trying to lay out an infinitely sized value. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/?utm_source=chatgpt.com "Enumerations - Documentation | Swift.org"))

The key idea:

```text
Enum state modeling = one valid case + only the payload valid for that case.
```

Swift guarantees that an enum value is one of its declared cases. It does not guarantee that your case design is semantically good. You can still make bad enums with vague cases, unlabeled associated values, duplicated state, or payloads that are too weak to encode the domain.

---

## 2. Essential mechanics

### Case exclusivity

An enum value is exactly one case.

```swift
enum ScreenState {
    case idle
    case loading
    case loaded([Int])
    case failed(message: String)
}

let state: ScreenState = .loading
```

This is stronger than:

```swift
struct ScreenState {
    var isLoading: Bool
    var data: [Int]?
    var errorMessage: String?
}
```

The struct can represent contradictory states. The enum cannot be both `.loading` and `.failed` at the same time.

### Associated values encode case-specific data

Each case can carry the data relevant to that state.

```swift
enum ProfileState {
    case idle
    case loading
    case loaded(user: User, isStale: Bool)
    case failed(error: ProfileError, retryCount: Int)
}
```

Good associated values are semantic. Prefer this:

```swift
case failed(error: ProfileError, retryCount: Int)
```

over this:

```swift
case failed(String, Int)
```

The Swift API Design Guidelines emphasize clarity at the point of use and clarity over brevity. That applies directly to enum case names and associated-value labels. ([Swift.org](https://swift.org/documentation/api-design-guidelines/?utm_source=chatgpt.com "API Design Guidelines | Swift.org"))

### Exhaustive switching

A `switch` over an enum forces you to handle all cases unless you add a catch-all branch.

```swift
func render(_ state: ScreenState) {
    switch state {
    case .idle:
        print("Show placeholder")

    case .loading:
        print("Show spinner")

    case .loaded(let values):
        print("Show \(values.count) values")

    case .failed(let message):
        print("Show error: \(message)")
    }
}
```

For internal app enums, avoid `default` unless it is truly intentional. Exhaustiveness is one of the main benefits of enums. A `default` branch hides newly added cases from the compiler.

### Recursive enums require `indirect`

Directly recursive enums need `indirect`.

```swift
indirect enum Expression {
    case number(Int)
    case add(Expression, Expression)
    case multiply(Expression, Expression)
}

let expression = Expression.add(
    .number(2),
    .multiply(.number(3), .number(4))
)
```

Without indirection, the compiler would need to store an `Expression` inside an `Expression` inside an `Expression`, with no finite layout. `indirect` tells Swift to add a level of indirection. The practical runtime cost is extra pointer indirection and, in many implementations, additional allocation/boxing compared with a fully inline payload.

### Enums can contain behavior

Enums are not just data tags. They can have computed properties and methods.

```swift
enum LoadState<Value> {
    case idle
    case loading
    case loaded(Value)
    case failed(message: String)

    var isTerminal: Bool {
        switch self {
        case .loaded, .failed:
            true
        case .idle, .loading:
            false
        }
    }
}
```

This keeps state-specific behavior close to the state model.

---

## 3. Common traps and misconceptions

### Trap 1: Replacing one bad model with another vague enum

Bad:

```swift
enum State {
    case state1
    case state2(String)
    case state3(Bool, String?)
}
```

This technically uses an enum, but it does not encode domain meaning. It just moves ambiguity into cases and unlabeled payloads.

Better:

```swift
enum SearchState {
    case idle
    case typing(query: String)
    case searching(query: String)
    case showingResults(query: String, results: [SearchResult])
    case noResults(query: String)
    case failed(query: String, message: String)
}
```

The better version names the domain states and gives each payload a semantic label.

### Trap 2: Using booleans for mutually exclusive states

Bad:

```swift
struct PlaybackState {
    var isPlaying: Bool
    var isPaused: Bool
    var isBuffering: Bool
    var error: Error?
}
```

This allows nonsense like:

```swift
isPlaying = true
isPaused = true
isBuffering = true
error != nil
```

Better:

```swift
enum PlaybackState {
    case stopped
    case buffering(previousTime: TimeInterval?)
    case playing(position: TimeInterval)
    case paused(position: TimeInterval)
    case failed(error: PlaybackError)
}
```

### Trap 3: Treating associated values as anonymous tuples

This is weak:

```swift
enum DownloadState {
    case failed(String, Int, Bool)
}
```

A future reader has to remember what each value means.

Prefer labeled values or small domain structs:

```swift
struct DownloadFailure: Error, Sendable {
    let message: String
    let statusCode: Int?
    let isRetryable: Bool
}

enum DownloadState {
    case failed(DownloadFailure)
}
```

### Trap 4: Duplicating state outside the enum

Bad:

```swift
struct ViewModelState {
    var state: LoadState
    var isLoading: Bool
    var errorMessage: String?
}
```

Now `state` and the derived flags can disagree.

Better:

```swift
enum LoadState<Value> {
    case idle
    case loading
    case loaded(Value)
    case failed(message: String)

    var isLoading: Bool {
        if case .loading = self { true } else { false }
    }

    var errorMessage: String? {
        if case .failed(let message) = self { message } else { nil }
    }
}
```

Derived information should usually be computed from the enum, not stored beside it.

---

## 4. Direct answers to rubric questions

### Q1. Why is an enum often a better model than a set of optional properties?

An enum is better when the domain has **mutually exclusive states**. Optional properties describe absence or presence independently; they do not describe which combinations are valid.

For example:

```swift
struct LoadState {
    var isLoading: Bool
    var data: [Int]?
    var errorMessage: String?
}
```

This allows:

```swift
LoadState(isLoading: true, data: [1, 2], errorMessage: "Network failed")
```

That probably makes no sense. Are we loading? Did we succeed? Did we fail? The type does not know.

An enum removes the invalid combinations:

```swift
enum LoadState {
    case idle
    case loading
    case loaded([Int])
    case failed(message: String)
}
```

Now there is no value that means “loading, loaded, and failed at once.”

Interview version:

> An enum is better when the states are mutually exclusive and each state has different valid data. Optionals and booleans model independent facts, so they allow invalid combinations unless every call site manually preserves invariants. An enum moves that invariant into the type system: the value is exactly one case, and each case carries only the payload that makes sense for that state.

### Q2. When do you need `indirect`, and what is the runtime implication?

You need `indirect` when an enum case directly stores another value of the same enum type.

```swift
indirect enum Tree<Element> {
    case leaf(Element)
    case node(left: Tree<Element>, right: Tree<Element>)
}
```

Without `indirect`, Swift cannot compute a finite memory layout. A `Tree` would contain a `Tree`, which would contain another `Tree`, forever. Swift’s documentation describes recursive enums as enums that have another instance of the enum as an associated value, and `indirect` tells the compiler to insert the required layer of indirection. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/?utm_source=chatgpt.com "Enumerations - Documentation | Swift.org"))

Runtime implication:

```text
direct enum payload:
    value stores payload inline when possible

indirect enum payload:
    value stores a reference-like indirection to payload storage
```

That means recursive enums are expressive and safe, but they are not free. You should expect extra indirection and possible allocation overhead compared with a flat nonrecursive enum.

Interview version:

> `indirect` is needed when an enum case recursively contains the enum itself. Without it, the compiler cannot form a finite layout for the value. `indirect` adds a level of indirection, which makes the recursive structure representable, but it also means extra pointer chasing and potentially allocation overhead. I would use it for naturally recursive domains like syntax trees, expression trees, routes, nested filters, or filesystem-like structures, not for ordinary flat UI state.

---

## 5. Code probe

Given:

```swift
struct LoadState {
    var isLoading = false
    var data: [Int]? = nil
    var errorMessage: String? = nil
}
```

### What happens?

```text
Compiles successfully.

Exact output:
No output.
```

There is no `print` or executable observation in the probe. The bug is semantic, not syntactic.

### Why?

This model allows these invalid or ambiguous states:

```swift
LoadState(
    isLoading: true,
    data: nil,
    errorMessage: nil
)
// Valid: loading.

LoadState(
    isLoading: false,
    data: [1, 2, 3],
    errorMessage: nil
)
// Valid: loaded.

LoadState(
    isLoading: false,
    data: nil,
    errorMessage: "Network failed"
)
// Valid: failed.

LoadState(
    isLoading: true,
    data: [1, 2, 3],
    errorMessage: nil
)
// Invalid or ambiguous: loading and loaded.

LoadState(
    isLoading: true,
    data: nil,
    errorMessage: "Network failed"
)
// Invalid or ambiguous: loading and failed.

LoadState(
    isLoading: false,
    data: [1, 2, 3],
    errorMessage: "Network failed"
)
// Invalid or ambiguous: loaded and failed.

LoadState(
    isLoading: true,
    data: [1, 2, 3],
    errorMessage: "Network failed"
)
// Invalid: all states at once.

LoadState(
    isLoading: false,
    data: nil,
    errorMessage: nil
)
// Ambiguous: idle, empty, not yet requested, or reset?
```

State diagram:

```text
Boolean/optional struct:

isLoading: Bool     -> 2 states
data: [Int]?        -> 2 states
errorMessage: String? -> 2 states

Total combinations: 2 × 2 × 2 = 8

Domain probably wants:
idle
loading
loaded(data)
failed(message)

Only 4 intended states.
The other combinations are accidental states.
```

### Fix or redesign

```swift
enum LoadState: Equatable, Sendable {
    case idle
    case loading
    case loaded([Int])
    case failed(message: String)
}
```

Usage:

```swift
func render(_ state: LoadState) {
    switch state {
    case .idle:
        print("Show empty placeholder")

    case .loading:
        print("Show spinner")

    case .loaded(let values):
        print("Show \(values.count) values")

    case .failed(let message):
        print("Show error: \(message)")
    }
}
```

### Why this fix is correct

The enum encodes the invariant directly:

```text
LoadState is exactly one of:
- idle
- loading
- loaded(data)
- failed(message)
```

There is no value for “loading with data and an error.” If you want “loading while showing stale data,” model that explicitly:

```swift
enum LoadState: Equatable, Sendable {
    case idle
    case loading(previous: [Int]?)
    case loaded([Int])
    case failed(message: String, previous: [Int]?)
}
```

That is the senior-level point: do not merely remove data from states. Preserve domain truth, but make it explicit.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Simple enum|States are mutually exclusive and payloads are small|Best default; may need evolution as UX gets richer|
|Enum with richer associated values|You need stale data, retry metadata, pagination, timestamps, or source info|More verbose but preserves correctness|
|Struct with private storage and validated initializer|State is not truly exclusive, but invariants still matter|More boilerplate; weaker exhaustiveness than enum|
|Separate reducer/action model|Complex state transitions need auditability and testability|More architecture; may be overkill for small screens|

---

## 6. Exercise

### Problem

Model a request lifecycle with cancel/retry semantics as an enum and explain why it is safer than booleans.

### Bad / naive version

```swift
struct RequestState<Value> {
    var isLoading = false
    var isCancelling = false
    var isRetrying = false

    var value: Value?
    var errorMessage: String?

    var attempt = 0
    var canRetry = false
}
```

### What is wrong?

```text
This allows impossible or unclear combinations:

- isLoading == true and isCancelling == true
- isRetrying == true but attempt == 0
- value != nil and errorMessage != nil
- canRetry == true with no failure
- isCancelling == true with no active request
- isLoading == false, value == nil, errorMessage == nil, but unclear whether idle, cancelled, or reset
```

Every mutation site must manually maintain the invariant. That is fragile in async code, especially with cancellation, retries, stale responses, and multiple overlapping requests.

### Improved version

```swift
struct RequestID: Hashable, Sendable {
    let rawValue: Int
}

struct RequestFailure: Error, Equatable, Sendable {
    let message: String
    let isRetryable: Bool
}

enum CancellationReason: Equatable, Sendable {
    case userInitiated
    case supersededByNewRequest
    case viewDisappeared
}

enum RetryPlan: Equatable, Sendable {
    case none
    case immediate(nextAttempt: Int)
    case afterDelay(nanoseconds: UInt64, nextAttempt: Int)
}

enum RequestLifecycle<Value: Sendable>: Sendable {
    case idle
    case inFlight(requestID: RequestID, attempt: Int)
    case loaded(Value)
    case failed(RequestFailure, retryPlan: RetryPlan)
    case cancelling(requestID: RequestID, reason: CancellationReason)
    case cancelled(reason: CancellationReason)
}
```

Example transitions:

```swift
extension RequestLifecycle {
    mutating func start(requestID: RequestID) {
        self = .inFlight(requestID: requestID, attempt: 1)
    }

    mutating func beginRetry(requestID: RequestID, attempt: Int) {
        self = .inFlight(requestID: requestID, attempt: attempt)
    }

    mutating func cancel(reason: CancellationReason) -> RequestID? {
        switch self {
        case .inFlight(let requestID, _):
            self = .cancelling(requestID: requestID, reason: reason)
            return requestID

        case .idle, .loaded, .failed, .cancelling, .cancelled:
            return nil
        }
    }

    mutating func complete(requestID: RequestID, with value: Value) {
        guard case .inFlight(let currentID, _) = self,
              currentID == requestID
        else {
            return
        }

        self = .loaded(value)
    }

    mutating func fail(
        requestID: RequestID,
        with failure: RequestFailure,
        retryPlan: RetryPlan
    ) {
        guard case .inFlight(let currentID, _) = self,
              currentID == requestID
        else {
            return
        }

        self = .failed(failure, retryPlan: retryPlan)
    }

    mutating func finishCancellation(
        requestID: RequestID,
        reason: CancellationReason
    ) {
        guard case .cancelling(let currentID, _) = self,
              currentID == requestID
        else {
            return
        }

        self = .cancelled(reason: reason)
    }
}
```

Example usage:

```swift
var state = RequestLifecycle<[Int]>.idle

let firstRequest = RequestID(rawValue: 1)
state.start(requestID: firstRequest)

let cancelledID = state.cancel(reason: .supersededByNewRequest)
// Use cancelledID to cancel the underlying Task if non-nil.

let secondRequest = RequestID(rawValue: 2)
state.start(requestID: secondRequest)

state.complete(requestID: firstRequest, with: [1, 2, 3])
// Ignored because firstRequest is stale.

state.complete(requestID: secondRequest, with: [4, 5, 6])
// State becomes .loaded([4, 5, 6])
```

### Why this is better

The enum makes the lifecycle explicit:

```text
idle
  -> inFlight(requestID, attempt)
  -> loaded(value)

idle
  -> inFlight(requestID, attempt)
  -> failed(failure, retryPlan)
  -> inFlight(newRequestID, nextAttempt)

inFlight(requestID, attempt)
  -> cancelling(requestID, reason)
  -> cancelled(reason)
```

The model prevents contradictory states by construction. You cannot have `.loaded` and `.failed` at the same time. You cannot be `.cancelling` without a `requestID`. You cannot have retry metadata unless the state is `.failed`.

The `requestID` also protects against stale async completions. This is important in Swift concurrency code because an older task may complete after a newer request has already started. The enum does not magically cancel tasks, but it gives you a safe state machine for accepting or ignoring completions.

---

## 7. Production guidance

Use this in production when:

```text
- A value has a finite set of mutually exclusive states.
- Different states require different data.
- You want exhaustive switch handling.
- You want to prevent invalid state combinations.
- You are modeling UI state, request state, playback state, authentication state, onboarding state, routing, feature availability, or domain workflows.
```

Be careful when:

```text
- The state space is not actually mutually exclusive.
- The enum grows into a giant god-state for an entire feature.
- Associated values become large anonymous tuples.
- Cases duplicate the same payload repeatedly.
- Public enums are part of a library API and may need future evolution.
- Recursive enums are used in hot paths without considering allocation/indirection costs.
```

Avoid when:

```text
- A simple Bool genuinely represents one independent binary fact.
- The values are open-ended and owned by external systems.
- You need identity, shared mutable state, or lifecycle semantics better modeled by a class or actor.
- You are hiding complex behavior behind vague cases like .custom(Any).
```

Debugging checklist:

```text
- Can this type represent a state that should be impossible?
- Are multiple booleans trying to encode one state machine?
- Are optionals independent, or are they really case-specific payloads?
- Are associated values labeled and semantic?
- Does every switch intentionally handle all cases?
- Did someone add default and accidentally hide future enum cases?
- Are stale async completions guarded by request identity?
- Is retry/cancel behavior represented in the state, or scattered across flags?
- Is the enum public? If yes, what happens when a new case is added?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Enums are useful when you have multiple cases, like loading, success, and failure.

### Senior answer

> Enums are useful for modeling mutually exclusive states. Associated values let each state carry only the data valid for that state, which avoids invalid combinations caused by booleans and optionals. I avoid catch-all switches for internal enums because exhaustiveness is a major safety feature.

### Staff-level answer

> I use enums to encode domain invariants into the type system. For UI and async state, I prefer enum state machines over scattered booleans because they make illegal combinations unrepresentable and make transitions testable. I also think about evolution: internal enums should usually be switched exhaustively, public SDK enums require resilience planning, associated values should be semantic and labeled, and recursive enums should be reserved for genuinely recursive domains because `indirect` has runtime costs.

Staff-level questions to ask:

```text
Which states are truly mutually exclusive?
Which data is valid only in a specific state?
Can stale async completions corrupt this state?
Should transitions be centralized in methods/reducers instead of spread across call sites?
Is this enum internal app state or public API that needs evolution planning?
```

---

## 9. Interview-ready summary

Enums are one of Swift’s best tools for domain modeling because they represent a closed set of mutually exclusive cases. Associated values let each case carry exactly the data that is valid for that state, which makes many invalid combinations impossible to express. This is much safer than modeling state with independent booleans and optionals, where every call site has to manually preserve invariants. For recursive data structures, Swift requires `indirect` so the compiler can insert a level of indirection and produce a finite memory layout. In production, I use enums for UI state, request lifecycles, routing, errors, and finite workflows, but I keep payloads semantic, avoid vague cases, and think carefully about public API evolution.

---

## 10. Flashcards

Q: Why are enums better than multiple booleans for state modeling?  
A: Because enum cases are mutually exclusive, while booleans can combine into invalid states.

Q: What is an associated value?  
A: Data stored with a specific enum case, valid only when the enum is in that case.

Q: What is the main smell in `isLoading`, `data?`, and `error?` state?  
A: The model allows contradictory states like loading with both data and an error.

Q: When do you need `indirect`?  
A: When an enum case directly stores another instance of the same enum type.

Q: What runtime implication does `indirect` have?  
A: It adds a level of indirection, which can mean pointer chasing and allocation overhead.

Q: Should associated values be labeled?  
A: Usually yes when the meaning is not obvious from the case name and type.

Q: Why is `default` often bad in switches over internal enums?  
A: It hides newly added cases and weakens exhaustiveness checking.

Q: What should you do if loading can show stale data?  
A: Model it explicitly, for example `.loading(previous: Value?)`, rather than storing separate flags.

Q: Does an enum automatically make a state machine correct?  
A: No. It prevents invalid representations only if the cases and payloads accurately encode the domain.

Q: Why might public enums require extra care?  
A: Adding or changing cases can affect source compatibility and client switch exhaustiveness.

---

## 11. Related sections

- [[A4 — Optionals and nil modeling]]
- [[A5 — Pattern matching and exhaustive control flow]]
- [[A15 — Type choices: struct, class, enum, protocol, and actor]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D7 — Actor reentrancy and logic races]]

---

## 12. Sources

- Swift Senior/Staff Rubric — A9 enum-based state modeling and code probe.
- Swift Documentation — Enumerations, associated values, and recursive enums. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/?utm_source=chatgpt.com "Enumerations - Documentation | Swift.org"))
- Swift Documentation — Declarations, enum indirection, and recursive layout. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations | Documentation - Swift Programming Language"))
- Swift Documentation — Structures, classes, and value-type behavior for structs/enums. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/?utm_source=chatgpt.com "Structures and Classes - Documentation | Swift.org"))
- Swift API Design Guidelines — clarity at the point of use and clarity over brevity. ([Swift.org](https://swift.org/documentation/api-design-guidelines/?utm_source=chatgpt.com "API Design Guidelines | Swift.org"))