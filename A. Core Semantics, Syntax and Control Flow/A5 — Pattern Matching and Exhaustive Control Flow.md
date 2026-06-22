---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - pattern-matching
  - control-flow
---
## 0. Rubric snapshot

**Rubric expectation**

Know `switch`, `if case`, `guard case`, `where`, tuple matching, enum associated values, and `@unknown default`.

**Caveats**

`default` can accidentally hide newly added enum cases. `fallthrough` rarely belongs in idiomatic Swift.

**You should be able to answer**

- Why is `@unknown default` preferable to plain `default` for some external enums?
- How would you destructure and validate an enum with associated values in one branch?

**You should be able to do**

- Write a `switch` over a networking result type that preserves exhaustiveness without a catch-all branch.

---

## 1. Core mental model

Pattern matching is Swift’s way of asking:

```text
Does this runtime value have this shape, and if so, what parts should I bind?
```

A `switch` is not just a cleaner `if/else`. It is a compile-time checked decomposition of a value space. When the switched value has a finite known shape, especially an enum, Swift can force you to handle every possible case.

That exhaustiveness check is the main reason Swift enums are powerful. A good `switch` over an enum turns “did I remember all states?” into a compiler-enforced property.

The key idea:

```text
Enum defines the value space.
switch proves you handled the value space.
patterns destructure the matching shape.
where refines a matching shape with a Boolean condition.
```

Swift patterns are used in `switch` cases, `catch` clauses, and `if`, `while`, `guard`, and `for-in` case conditions. The Swift reference lists enum-case patterns, optional patterns, expression patterns, and type-casting patterns among the forms used for full pattern matching. ([docs.swift.org](https://docs.swift.org/swift-book/ReferenceManual/Patterns.html?utm_source=chatgpt.com "Patterns - Documentation | Swift.org"))

`if case` and `guard case` are for matching one interesting shape without writing a full `switch`. Swift’s documentation describes `if case` as using the same patterns you can write in a `switch` case. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/?utm_source=chatgpt.com "Control Flow | Documentation - Swift Programming Language"))

---

## 2. Essential mechanics

### 2.1 `switch` is exhaustive for known finite domains

For local Swift enums, a `switch` without `default` must handle all cases.

```swift
enum LoadState {
    case idle
    case loading
    case loaded([String])
    case failed(Error)
}

func title(for state: LoadState) -> String {
    switch state {
    case .idle:
        "Idle"
    case .loading:
        "Loading"
    case .loaded(let items):
        "Loaded \(items.count)"
    case .failed:
        "Failed"
    }
}
```

This is stronger than an `if/else` chain. If you later add:

```swift
case cancelled
```

the compiler forces every exhaustive `switch` to decide what cancellation means.

That is the point.

### 2.2 Associated values are destructured by the pattern

Associated values are not fields you access later; they are payloads you can bind as part of the matching branch.

```swift
enum LoginEvent {
    case submitted(email: String, password: String)
    case succeeded(userID: String)
    case failed(message: String)
}

func handle(_ event: LoginEvent) {
    switch event {
    case let .submitted(email, password) where email.contains("@") && password.count >= 8:
        print("Valid login attempt from \(email)")

    case .submitted:
        print("Invalid login form")

    case let .succeeded(userID):
        print("Logged in user \(userID)")

    case let .failed(message):
        print("Login failed: \(message)")
    }
}
```

Important detail: the `where` clause refines the branch. It does not make the entire enum case handled unless another branch handles the same case without that condition.

This is exhaustive:

```swift
case let .submitted(email, password) where isValid(email, password):
    ...
case .submitted:
    ...
```

This is not exhaustive for `.submitted`, because invalid submissions do not match:

```swift
case let .submitted(email, password) where isValid(email, password):
    ...
```

### 2.3 `if case` and `guard case` are single-pattern checks

Use `if case` when one shape matters and the rest can be ignored.

```swift
if case let .loaded(items) = state, !items.isEmpty {
    print("Show list")
}
```

Use `guard case` when the rest of the function requires a specific shape.

```swift
func showDetails(from state: LoadState) {
    guard case let .loaded(items) = state else {
        return
    }

    print("Showing \(items.count) items")
}
```

This avoids nested `switch` code when you only need one case.

### 2.4 Tuple matching combines dimensions

Tuple matching is useful when control flow depends on multiple values at once.

```swift
enum Reachability {
    case online
    case offline
}

enum AuthState {
    case anonymous
    case authenticated
}

func action(reachability: Reachability, auth: AuthState) -> String {
    switch (reachability, auth) {
    case (.online, .authenticated):
        "Fetch user data"
    case (.online, .anonymous):
        "Show login"
    case (.offline, _):
        "Show offline mode"
    }
}
```

The last branch covers both:

```swift
(.offline, .anonymous)
(.offline, .authenticated)
```

Tuple matching is good when the combined state is truly cross-product logic. If the tuple grows beyond two or three dimensions, it often means the state model wants its own enum.

### 2.5 `@unknown default` is for non-frozen external enums

External framework enums may gain new cases in future SDKs. SE-0192 introduced the frozen versus non-frozen enum distinction to let libraries add cases without breaking source compatibility for clients; the proposal specifically says adding a case to a non-frozen enum is source-compatible. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md "swift-evolution/proposals/0192-non-exhaustive-enums.md at main · swiftlang/swift-evolution · GitHub"))

Swift’s statement reference says that when switching over a nonfrozen enum, you need a default-like catch-all, and `@unknown default` indicates the branch is meant only for cases added in the future; Swift warns if it matches any case known at compile time. ([docs.swift.org](https://docs.swift.org/swift-book/ReferenceManual/Statements.html?utm_source=chatgpt.com "Statements | Documentation - Swift Programming Language"))

Example shape:

```swift
switch externalStatus {
case .knownA:
    handleA()
case .knownB:
    handleB()
@unknown default:
    logUnexpectedSDKCase()
    useSafeFallback()
}
```

This is better than:

```swift
default:
    useSafeFallback()
```

because plain `default` silently hides future known cases after an SDK update. `@unknown default` keeps the code source-compatible while still pushing you to handle newly known cases deliberately.

---

## 3. Common traps and misconceptions

### Trap 1: Using `default` because it is shorter

Bad:

```swift
enum PaymentState {
    case idle
    case processing
    case succeeded
    case failed
}

func message(for state: PaymentState) -> String {
    switch state {
    case .idle:
        "Ready"
    case .processing:
        "Processing"
    default:
        "Done"
    }
}
```

This compiles, but it destroys the compiler’s ability to help you. If you later add:

```swift
case requiresAuthentication
```

the compiler will not complain. The new state falls into `"Done"`.

Better:

```swift
func message(for state: PaymentState) -> String {
    switch state {
    case .idle:
        "Ready"
    case .processing:
        "Processing"
    case .succeeded:
        "Succeeded"
    case .failed:
        "Failed"
    }
}
```

For internal enums, prefer explicit cases. Use `default` only when the exact value genuinely does not matter.

### Trap 2: Thinking `where` handles the whole enum case

Bad:

```swift
enum HTTPResponse {
    case completed(statusCode: Int)
    case failed(Error)
}

func handle(_ response: HTTPResponse) {
    switch response {
    case let .completed(statusCode) where (200..<300).contains(statusCode):
        print("Success")
    case .failed:
        print("Failure")
    }
}
```

Exact compiler error with Swift 6.2.1:

```text
error: switch must be exhaustive
note: add missing case: '.completed(statusCode: let statusCode)'
```

Why? `.completed(statusCode: 404)` does not match the first branch, and no later branch handles `.completed`.

Better:

```swift
func handle(_ response: HTTPResponse) {
    switch response {
    case let .completed(statusCode) where (200..<300).contains(statusCode):
        print("Success")
    case let .completed(statusCode):
        print("Unexpected status: \(statusCode)")
    case .failed:
        print("Failure")
    }
}
```

### Trap 3: Using `fallthrough` to share behavior

Bad:

```swift
switch state {
case .idle:
    print("Idle")
    fallthrough
case .loading:
    print("Show spinner")
case .loaded:
    print("Show content")
}
```

This is almost always worse than expressing the shared behavior directly.

Better:

```swift
switch state {
case .idle, .loading:
    print("Show spinner or placeholder")
case .loaded:
    print("Show content")
}
```

Or split shared behavior into a helper:

```swift
switch state {
case .idle:
    showPlaceholder()
case .loading:
    showSpinner()
case .loaded:
    showContent()
}
```

Swift does not implicitly fall through from one `case` to the next. If you are reaching for `fallthrough`, ask whether grouped cases, helper functions, or a better state model would be clearer.

---

## 4. Direct answers to rubric questions

### Q1. Why is `@unknown default` preferable to plain `default` for some external enums?

`@unknown default` preserves compiler help when an external enum grows new cases. Plain `default` swallows both future cases and currently unhandled known cases. `@unknown default` says: “I handled all cases I know about; this branch exists only for cases this compiler does not know about yet.”

For SDK enums, this matters because Apple or another framework author may add cases in a future SDK. With plain `default`, your code may keep compiling while applying an inappropriate fallback to a new meaningful case. With `@unknown default`, Swift can warn you when a newly known case is not handled explicitly. SE-0192 exists specifically to support future enum cases while preserving exhaustiveness checking as much as possible. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md "swift-evolution/proposals/0192-non-exhaustive-enums.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> For external non-frozen enums, `@unknown default` is a compatibility branch, not a laziness branch. It lets the app keep compiling against future SDK cases, but it still preserves the compiler’s ability to warn me when a new known case appears. A plain `default` would hide that signal and can turn an SDK evolution event into a silent behavior bug.

### Q2. How would you destructure and validate an enum with associated values in one branch?

Use an enum-case pattern with value binding, then add a `where` clause for validation. Add a second branch for the same case if the `where` condition does not cover all associated values.

```swift
enum SignupEvent {
    case submitted(email: String, password: String)
    case cancelled
}

func handle(_ event: SignupEvent) {
    switch event {
    case let .submitted(email, password)
        where email.contains("@") && password.count >= 8:
        print("Submit valid form")

    case .submitted:
        print("Show validation error")

    case .cancelled:
        print("Dismiss")
    }
}
```

The first branch both destructures and validates:

```swift
case let .submitted(email, password) where ...
```

The second branch is still needed because invalid `.submitted` values do not match the first branch.

Interview version:

> I would bind the associated values directly in the case pattern and use `where` to refine the match. But I would remember that a `where` branch only handles values satisfying the predicate, so I usually need another branch for the same enum case to preserve exhaustiveness.

---

## 5. Code probe

A5 has no rubric-provided code probe, so this is a focused probe for the same concept. The answer template expects code-probe-style treatment when useful.

Given:

```swift
enum Response {
    case success(Int)
    case failure(String)
    case cancelled
}

let response: Response = .success(204)

switch response {
case .success(let code) where (200..<300).contains(code):
    print("ok \(code)")
case .success(let code):
    print("unexpected \(code)")
case .failure(let message):
    print("failure \(message)")
case .cancelled:
    print("cancelled")
}
```

### What happens?

Exact output with Swift 6.2.1:

```text
ok 204
```

### Why?

The runtime value is:

```text
Response.success(204)
```

Swift tests the cases in order:

```text
case .success(let code) where 200..<300 contains code
    .success matches
    code = 204
    condition is true
    branch executes

remaining cases are ignored
```

The second `.success` branch is still necessary because this first branch only handles successful responses whose status-like value is in `200..<300`.

### Counterexample

Given:

```swift
enum Response {
    case success(Int)
    case failure(String)
    case cancelled
}

func render(_ response: Response) -> String {
    switch response {
    case .success(let code):
        return "success \(code)"
    case .failure(let message):
        return "failure \(message)"
    }
}
```

Exact compiler error with Swift 6.2.1:

```text
A5Bad.swift:8:5: error: switch must be exhaustive
 6 | 
 7 | func render(_ response: Response) -> String {
 8 |     switch response {
   |     |- error: switch must be exhaustive
   |     `- note: add missing case: '.cancelled'
 9 |     case .success(let code):
10 |         return "success \(code)"
```

### Fix or redesign

```swift
func render(_ response: Response) -> String {
    switch response {
    case .success(let code):
        return "success \(code)"
    case .failure(let message):
        return "failure \(message)"
    case .cancelled:
        return "cancelled"
    }
}
```

### Why this fix is correct

The fixed version preserves the mapping:

```text
.success(Int)    -> explicit behavior
.failure(String) -> explicit behavior
.cancelled       -> explicit behavior
```

No branch is hidden behind `default`, so adding a new `Response` case later forces the code to be revisited.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Explicit cases|Best default for internal enums|More lines, but safest and clearest|
|Grouped cases|Multiple cases genuinely share behavior|Can blur domain semantics if overused|
|`@unknown default`|External non-frozen SDK enums|Fallback path is hard to test|
|Plain `default`|Truly irrelevant remainder, often non-enum or large value spaces|Can hide new enum cases and domain changes|

---

## 6. Exercise

### Problem

Write a `switch` over a networking result type that preserves exhaustiveness without a catch-all branch.

### Bad / naive version

```swift
import Foundation

enum NetworkResult<Success> {
    case success(Success)
    case failure(NetworkError)
}

enum NetworkError: Error {
    case transport(URLError)
    case server(statusCode: Int, body: Data?)
    case decoding(DecodingError)
    case cancelled
}

struct User: Decodable {
    let id: Int
    let name: String
}

enum UserListViewState {
    case users([User])
    case loginRequired
    case retryableError(String)
    case empty
}

func mapToViewState(_ result: NetworkResult<[User]>) -> UserListViewState {
    switch result {
    case .success(let users):
        return users.isEmpty ? .empty : .users(users)
    default:
        return .retryableError("Something went wrong")
    }
}
```

### What is wrong?

```text
1. default hides the specific failure cases.
2. cancellation becomes an error UI.
3. 401/403 authentication failures are not distinguished from server errors.
4. decoding failures are treated like retryable network failures.
5. adding a new NetworkResult case later may be silently swallowed by default.
6. adding a new NetworkError case later may also be silently swallowed if nested matching uses default.
```

This is the exact kind of bug that exhaustiveness is supposed to prevent.

### Improved version

```swift
import Foundation

enum NetworkResult<Success> {
    case success(Success, statusCode: Int)
    case failure(NetworkError)
}

enum NetworkError: Error {
    case transport(URLError)
    case server(statusCode: Int, body: Data?)
    case decoding(DecodingError)
    case cancelled
}

struct User: Decodable {
    let id: Int
    let name: String
}

enum UserListViewState: Equatable {
    case users([User])
    case empty
    case loginRequired
    case offline
    case cancelled
    case retryableError(message: String)
    case nonRetryableError(message: String)
}

func mapToViewState(_ result: NetworkResult<[User]>) -> UserListViewState {
    switch result {
    case let .success(users, statusCode) where (200..<300).contains(statusCode):
        return users.isEmpty ? .empty : .users(users)

    case let .success(_, statusCode):
        return .nonRetryableError(message: "Unexpected success status: \(statusCode)")

    case .failure(.cancelled):
        return .cancelled

    case let .failure(.transport(error)) where error.code == .notConnectedToInternet:
        return .offline

    case let .failure(.transport(error)):
        return .retryableError(message: error.localizedDescription)

    case let .failure(.server(statusCode, _)) where statusCode == 401 || statusCode == 403:
        return .loginRequired

    case let .failure(.server(statusCode, _)) where (500..<600).contains(statusCode):
        return .retryableError(message: "Server error: \(statusCode)")

    case let .failure(.server(statusCode, _)):
        return .nonRetryableError(message: "Request failed: \(statusCode)")

    case let .failure(.decoding(error)):
        return .nonRetryableError(message: "Invalid response: \(error.localizedDescription)")
    }
}
```

### Why this is better

This switch is exhaustive without a catch-all branch.

It handles every `NetworkResult` case:

```text
.success
.failure
```

It also handles every nested `NetworkError` case:

```text
.cancelled
.transport
.server
.decoding
```

The `where` clauses refine branches, but every refined branch has a fallback branch for the same shape:

```text
.success where 2xx
.success all other status codes

.transport where offline
.transport all other transport errors

.server where 401/403
.server where 5xx
.server all other status codes
```

That is the senior-level pattern: use `where` to express business rules, but never let `where` accidentally create an uncovered subspace.

---

## 7. Production guidance

Use this in production when:

```text
- Modeling finite domain state with enums.
- Rendering view state from loading/result state.
- Handling nested errors where specific cases matter.
- Mapping networking, persistence, authentication, payment, or permission states.
- Switching over SDK enums where new cases may appear.
```

Be careful when:

```text
- A branch uses where; make sure the same enum case has a fallback branch.
- Tuple matching combines many state dimensions.
- You are switching over external framework enums.
- You are tempted to use default for convenience.
- You are matching optional-heavy state that should probably be an enum.
```

Avoid when:

```text
- if/else is clearer for simple Boolean logic.
- The value space is not finite or not meaningful to enumerate.
- Patterns become so clever that the business rule is hidden.
- A tuple switch is compensating for a missing domain model.
```

Debugging checklist:

```text
Did we use default on an internal enum?
Would adding a new enum case produce a compiler error?
Does every where-refined branch have a fallback for the same shape?
Are cancellation, authentication, offline, server, and decoding states semantically distinct?
Are multiple booleans or optionals hiding an enum state machine?
Are we switching over an external enum that should use @unknown default?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Pattern matching means using `switch` with enum cases and `let` values. Swift makes switches exhaustive.

### Senior answer

> Pattern matching lets me decompose enums, tuples, optionals, and associated values directly in control flow. I avoid `default` for internal enums because explicit cases preserve compiler checking. I use `where` carefully because it only covers the values satisfying the condition.

### Staff-level answer

> Pattern matching is a design tool, not just syntax. I want domain states modeled as finite enums so `switch` becomes a compiler-checked proof that every state has behavior. In app code, I avoid catch-all branches for owned enums because they hide domain evolution. For external non-frozen enums, I use `@unknown default` to keep forward compatibility without losing the compiler’s warning when new SDK cases appear. In large codebases, this directly affects API evolution, state-machine correctness, and migration safety.

Staff-level questions to ask:

```text
Is this switch proving exhaustiveness, or hiding missing behavior?
Is this default branch legitimate, or just avoiding a compiler error?
Should this tuple switch become a named domain enum?
Do where clauses leave uncovered subspaces?
Is this enum owned by us, or can an external SDK/library add cases?
```

---

## 9. Interview-ready summary

Pattern matching in Swift is about matching the shape of a value and binding its parts. `switch` is especially powerful because it can prove exhaustiveness for enums, which turns missing state handling into a compiler error. I use explicit cases for internal enums, `if case` or `guard case` for single-shape checks, `where` to refine associated values, and tuple matching when behavior depends on multiple dimensions. I avoid plain `default` for owned enums because it hides future cases. For external non-frozen SDK enums, I use `@unknown default` so the code remains forward-compatible while still getting compiler warnings when new known cases appear.

---

## 10. Flashcards

Q: What does pattern matching do in Swift?  
A: It checks whether a value has a particular shape and optionally binds parts of that value.

Q: Why should you avoid `default` for internal enums?  
A: It hides newly added enum cases and prevents the compiler from forcing explicit behavior.

Q: What does `where` do in a `switch` case?  
A: It adds a Boolean condition to a pattern match. The branch only handles values that match the pattern and satisfy the condition.

Q: Why does a `where` branch often need a fallback branch for the same enum case?  
A: Because values that match the enum case but fail the `where` condition are still possible and must be handled.

Q: When should you use `if case`?  
A: When only one specific pattern matters and you do not need full exhaustive branching.

Q: When should you use `guard case`?  
A: When the rest of the function requires a value to have a specific shape.

Q: What is `@unknown default` for?  
A: Handling future or unknown cases of external non-frozen enums while preserving compiler warnings for newly known cases.

Q: Why is tuple matching useful?  
A: It lets you match combinations of values in a single control-flow structure.

Q: Why is `fallthrough` rare in idiomatic Swift?  
A: Swift cases do not fall through by default, and shared behavior is usually clearer with grouped cases or helper functions.

Q: What is the production value of exhaustive switches?  
A: They make domain evolution safer because adding a new enum case forces all relevant behavior to be revisited.

---

## 11. Related sections

- [[A4 — Optionals and nil modeling]]
- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[A9 — Enums, associated values, recursive enums, and state modeling]]
- [[A12 — Availability and conditional compilation]]
- [[B11 — Library evolution and resilience-aware API design]]

---

## 12. Sources

- Swift Senior/Staff Rubric — A5 pattern matching and exhaustive control flow.
- Swift Documentation — Control Flow. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/?utm_source=chatgpt.com "Control Flow | Documentation - Swift Programming Language"))
- Swift Documentation — Patterns. ([docs.swift.org](https://docs.swift.org/swift-book/ReferenceManual/Patterns.html?utm_source=chatgpt.com "Patterns - Documentation | Swift.org"))
- Swift Documentation — Statements / `@unknown default`. ([docs.swift.org](https://docs.swift.org/swift-book/ReferenceManual/Statements.html?utm_source=chatgpt.com "Statements | Documentation - Swift Programming Language"))
- Swift Evolution SE-0192 — Handling Future Enum Cases. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md "swift-evolution/proposals/0192-non-exhaustive-enums.md at main · swiftlang/swift-evolution · GitHub"))