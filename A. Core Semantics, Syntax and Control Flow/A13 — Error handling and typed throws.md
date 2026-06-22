---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - error-handling
  - typed-throws
---
## 0. Rubric snapshot

**Rubric expectation**

Know `throws`, `rethrows`, `Result`, `try?`, `try!`, and modern typed throws in Swift 6.

**Caveats**

`try?` collapses error information. Typed throws improve precision, but they can complicate API surfaces if overused. The rubric specifically asks you to explain when to return `Result`, what `throws(Never)` means, and how to refactor a generic transform API from `rethrows` or untyped `throws` to typed throws.

**You should be able to answer**

- When should you return `Result` instead of throwing?
- What does `throws(Never)` mean semantically?
- How do typed throws compare with `rethrows`?

**You should be able to do**

- Refactor a generic transform API from `rethrows` or untyped `throws` to typed throws.
- Explain the precision/evolution tradeoff.

---

## 1. Core mental model

Swift error handling is an alternate return path, not exception-style arbitrary control flow. A throwing function has two possible exits:

```text
normal return: Output
error return:  Error type
```

Historically, Swift’s thrown error type was always type-erased to `any Error`; SE-0413 added the ability to specify that a function or closure only throws a particular concrete error type, implemented in Swift 6.0. ([GitHub, "SE-0413: Typed Throws"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md))

The modern model is:

```swift
func oldStyle() throws -> Value
// roughly: throws(any Error)

func typed() throws(LoginError) -> Value
// throws exactly LoginError

func infallible() throws(Never) -> Value
// cannot throw; equivalent to non-throwing
```

The key idea:

```text
throws(E) makes the error path part of the function type.
```

This gives the compiler more information. A caller of `throws(LoginError)` knows that the error path is `LoginError`, not an arbitrary `any Error`. That helps when the caller can meaningfully handle every case, especially inside a module, package, parser, validation layer, or generic algorithm.

But the type is also an API promise. If a public SDK says `throws(NetworkError)`, changing that to `throws(AuthError)`, `throws(any Error)`, or a wrapper error later affects source compatibility. Swift Evolution explicitly notes that typed throws constrain implementation evolution and that untyped `throws` remains a good default for most code. ([GitHub, "SE-0413: Typed Throws"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md))

Use typed throws when the failure domain is stable and meaningful to clients. Use untyped `throws` when errors are mostly propagated, logged, rendered to the user, or sourced from dependencies that may change. Use `Result` when success/failure must be stored, passed around, combined, or delivered through callback-like boundaries rather than immediately propagated.

---

## 2. Essential mechanics

### 2.1 `throws`, `try`, `do/catch`

A throwing function must be called with `try`, unless its thrown type is statically `Never`.

```swift
enum LoginError: Error {
    case missingEmail
    case invalidPassword
}

func validate(email: String, password: String) throws(LoginError) -> String {
    guard !email.isEmpty else { throw .missingEmail }
    guard password.count >= 8 else { throw .invalidPassword }
    return "ok"
}

do {
    let token = try validate(email: "", password: "12345678")
    print(token)
} catch {
    // error has type LoginError here
    print(error)
}
```

In a typed throwing function, the compiler rejects throwing a different error type:

```swift
enum ParseError: Error { case invalid }
enum NetworkError: Error { case offline }

func parse() throws(ParseError) {
    throw NetworkError.offline
}
```

Compiler error, verified with Swift 6.2.1:

```text
a13_bad.swift:5:24: error: thrown expression type 'NetworkError' cannot be converted to error type 'ParseError'
3 | 
4 | func parse() throws(ParseError) {
5 |     throw NetworkError.offline
  |                        `- error: thrown expression type 'NetworkError' cannot be converted to error type 'ParseError'
6 | }
7 | 
```

### 2.2 `throws(any Error)` and `throws(Never)`

SE-0413 defines untyped `throws` as equivalent to `throws(any Error)`, and `throws(Never)` as equivalent to a non-throwing function. ([GitHub, "SE-0413: Typed Throws"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md))

```swift
func canThrowAnything() throws(any Error) {
    // Same effect as plain `throws`
}

func answer() throws(Never) -> Int {
    42
}

print(answer()) // no `try` needed
```

Output:

```text
42
```

`Never` is uninhabited: no value of type `Never` can exist. Therefore, a function whose error path is `Never` has no possible runtime error value to throw.

### 2.3 `try?` collapses error information

`try?` converts success into an optional value and any thrown error into `nil`.

```swift
enum LoginError: Error { case invalidPassword }

func login() throws(LoginError) -> String {
    throw .invalidPassword
}

let token = try? login()
print(type(of: token))
print(token as Any)
```

Output:

```text
Optional<String>
nil
```

The important part: `try?` discards `LoginError.invalidPassword`. After `try?`, the caller only knows “there was no value.” That is fine for optional-like flows, but it is bad when the failure reason matters.

### 2.4 `try!` is a runtime assertion

`try!` says: “I know this can syntactically throw, but I assert it will not throw here.”

```swift
let token = try! validate(email: "me@example.com", password: "12345678")
```

Use it only when failure would indicate a programmer error or impossible invariant violation. In production app code, repeated `try!` is usually a smell. It turns recoverable error handling into a crash path.

### 2.5 `rethrows`

`rethrows` means a function can throw only as a consequence of calling one of its throwing function parameters.

Classic shape:

```swift
extension Collection {
    func oldTransformed<Output>(
        _ transform: (Element) throws -> Output
    ) rethrows -> [Output] {
        var output: [Output] = []
        output.reserveCapacity(underestimatedCount)

        for element in self {
            output.append(try transform(element))
        }

        return output
    }
}
```

This is useful because calling it with a non-throwing closure does not require `try`:

```swift
let doubled = [1, 2, 3].oldTransformed { $0 * 2 }
```

But `rethrows` is untyped. It expresses “when this throws,” not precisely “what this throws.”

Typed throws can express many rethrowing algorithms more precisely by carrying a generic error type through the closure and the outer function. SE-0413 uses `map` as the canonical example: the operation directly throws the same error as the body; for non-throwing closures, the error type becomes `Never`; for untyped throwing closures, it becomes `any Error`. ([GitHub, "SE-0413: Typed Throws"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md))

---

## 3. Common traps and misconceptions

### Trap 1: Treating `try?` as harmless convenience

Bad:

```swift
func loadProfile() async -> Profile? {
    try? await api.fetchProfile()
}
```

This erases whether the failure was offline, unauthorized, decoding failure, cancellation, server error, or a bug.

Better:

```swift
func loadProfile() async -> Result<Profile, ProfileLoadError> {
    do {
        return .success(try await api.fetchProfile())
    } catch is CancellationError {
        return .failure(.cancelled)
    } catch {
        return .failure(.requestFailed(error))
    }
}
```

Or, if immediate propagation is enough:

```swift
func loadProfile() async throws -> Profile {
    try await api.fetchProfile()
}
```

### Trap 2: Overusing typed throws in unstable public APIs

Bad:

```swift
public func fetchFeed() async throws(URLError) -> Feed
```

This looks precise, but it assumes the only failure forever is `URLError`. The moment you add authentication, decoding, rate limiting, cache corruption, or server-side semantic failures, the API contract becomes too narrow.

Better:

```swift
public enum FeedError: Error {
    case transport(URLError)
    case decoding(DecodingError)
    case unauthorized
    case rateLimited
    case unknown(any Error)
}

public func fetchFeed() async throws(FeedError) -> Feed
```

Or keep it untyped if clients mostly render/log errors:

```swift
public func fetchFeed() async throws -> Feed
```

### Trap 3: Assuming typed throws gives exhaustive `catch` over enum cases

This is subtle. Even with typed throws, Swift’s `do/catch` exhaustiveness is not the same as `switch` exhaustiveness. SE-0413 notes that unconditional `catch` is what makes a `do/catch` exhaustive; type-pattern catches alone do not prove exhaustiveness. ([GitHub, "SE-0413: Typed Throws"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md))

Prefer:

```swift
do {
    try validate(email: email, password: password)
} catch {
    switch error {
    case .missingEmail:
        showMissingEmail()
    case .invalidPassword:
        showInvalidPassword()
    }
}
```

This gives you enum-case exhaustiveness inside the `switch`, where Swift’s exhaustiveness model is strongest.

### Trap 4: Using `Result` everywhere because it is “more explicit”

Bad:

```swift
func parse(_ input: String) -> Result<Model, ParseError>
```

This may be appropriate if you store or batch parse results. But if callers immediately unwrap and propagate the failure, `throws(ParseError)` is clearer:

```swift
func parse(_ input: String) throws(ParseError) -> Model
```

`Result` is a value. `throws` is control flow. Use the one that matches how the failure moves through the program.

---

## 4. Direct answers to rubric questions

### Q1. When should you return `Result` instead of throwing?

Return `Result` when success/failure must be represented as a value instead of immediately controlling execution.

Use `Result` when you need to:

- Store outcomes for later.
- Put successes and failures in a collection.
- Pass a completion value through callback-style APIs.
- Retry, aggregate, debounce, or pipeline outcomes.
- Expose a publisher/stream-like state machine where failure is data.
- Preserve the failure type across an API boundary that cannot be `throws`.

Prefer `throws` when the caller should either handle the error now or propagate it upward.

```swift
// Better as throws: immediate control flow
func decodeUser(from data: Data) throws(DecodingError) -> User

// Better as Result: stored outcome
struct UploadRecord {
    let fileID: UUID
    let result: Result<UploadReceipt, UploadError>
}
```

Interview version:

> I use `throws` for normal call-stack error propagation and `Result` when the outcome itself needs to be a value: stored, passed through callbacks, collected, retried, or modeled as state. `Result` is not a replacement for `throws`; it is for cases where success/failure needs value semantics.

### Q2. What does `throws(Never)` mean semantically?

`throws(Never)` means the function has no possible error value to throw. It is equivalent to being non-throwing.

```swift
func makeID() throws(Never) -> UUID {
    UUID()
}

let id = makeID() // no try
```

Conceptually:

```text
throws(any Error)  -> can throw anything conforming to Error
throws(MyError)    -> can throw MyError
throws(Never)      -> cannot throw
```

This matters for generic APIs:

```swift
func transformed<Output, Failure: Error>(
    _ transform: (Element) throws(Failure) -> Output
) throws(Failure) -> [Output]
```

If `transform` is non-throwing, `Failure == Never`, so the outer function is also non-throwing.

Interview version:

> `throws(Never)` is the typed-throws spelling for a non-throwing function. Since `Never` has no values, there is no possible error value to propagate. It is especially useful in generic typed-throws APIs because a non-throwing closure naturally maps to `Failure == Never`.

### Q3. How do typed throws compare with `rethrows`?

`rethrows` tells you when a function can throw: only if a throwing function parameter throws. Typed throws tells you what type can be thrown.

Classic `rethrows`:

```swift
func transformed<Output>(
    _ transform: (Element) throws -> Output
) rethrows -> [Output]
```

Typed throws:

```swift
func transformed<Output, Failure: Error>(
    _ transform: (Element) throws(Failure) -> Output
) throws(Failure) -> [Output]
```

The typed version preserves the transform’s error type. If the transform throws `ValidationError`, the outer function throws `ValidationError`. If the transform is non-throwing, the outer function is effectively non-throwing.

But typed throws cannot represent every `rethrows` design. If the outer function catches the closure’s error and substitutes a different error type, then it no longer throws the same `Failure`. SE-0413 explicitly calls this out: typed throws models direct propagation well, but not error substitution in the same way. ([GitHub, "SE-0413: Typed Throws"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md))

Interview version:

> `rethrows` is about conditional throwing; typed throws is about the static type of the error path. For higher-order functions that only forward closure errors, typed throws is more precise because it can preserve the closure’s failure type. But if the function translates errors, enriches them, or can fail independently, `rethrows` or a concrete wrapper error may be a better model.

---

## 5. Code probe replacement: minimal, counterexample, production example

The rubric has no explicit code probe for A13, so this section uses compiler-checked examples.

### 5.1 Minimal example

Given:

```swift
enum LoginError: Error, CustomStringConvertible {
    case missingEmail
    case invalidPassword

    var description: String {
        switch self {
        case .missingEmail: "missingEmail"
        case .invalidPassword: "invalidPassword"
        }
    }
}

func validate(email: String, password: String) throws(LoginError) -> String {
    guard !email.isEmpty else { throw .missingEmail }
    guard password.count >= 8 else { throw .invalidPassword }
    return "ok"
}

do {
    print(try validate(email: "", password: "12345678"))
} catch {
    print(type(of: error), error)
}

let collapsed = try? validate(email: "me@example.com", password: "short")
print(collapsed as Any)

func answer() throws(Never) -> Int { 42 }
print(answer())
```

### What happens?

Output, verified with Swift 6.2.1:

```text
LoginError missingEmail
nil
42
```

### Why?

```text
validate("", "12345678")
  -> throws LoginError.missingEmail
  -> catch receives LoginError

try? validate("me@example.com", "short")
  -> throws LoginError.invalidPassword
  -> converted to Optional.none
  -> error detail discarded

answer() throws(Never)
  -> no possible error value
  -> equivalent to non-throwing
  -> no try needed
```

### 5.2 Counterexample

Given:

```swift
enum ParseError: Error { case invalid }
enum NetworkError: Error { case offline }

func parse() throws(ParseError) {
    throw NetworkError.offline
}
```

### What happens?

Compiler error:

```text
a13_bad.swift:5:24: error: thrown expression type 'NetworkError' cannot be converted to error type 'ParseError'
3 | 
4 | func parse() throws(ParseError) {
5 |     throw NetworkError.offline
  |                        `- error: thrown expression type 'NetworkError' cannot be converted to error type 'ParseError'
6 | }
7 | 
```

### Why?

`parse()` promises that its error path is `ParseError`. `NetworkError.offline` is not a `ParseError`, so the compiler rejects it.

Fix:

```swift
enum ParseError: Error {
    case invalid
    case dependency(any Error)
}

func parse() throws(ParseError) {
    do {
        try fetchSchema()
    } catch {
        throw .dependency(error)
    }
}

func fetchSchema() throws(NetworkError) {
    throw .offline
}
```

### 5.3 Production example

A validation layer is a good typed-throws candidate because the error domain is local, stable, and exhaustively meaningful.

```swift
struct SignupForm {
    var email: String
    var password: String
}

enum SignupValidationError: Error, Equatable {
    case missingEmail
    case invalidEmail
    case weakPassword
}

func validate(_ form: SignupForm) throws(SignupValidationError) {
    guard !form.email.isEmpty else {
        throw .missingEmail
    }

    guard form.email.contains("@") else {
        throw .invalidEmail
    }

    guard form.password.count >= 12 else {
        throw .weakPassword
    }
}

func submit(_ form: SignupForm) async {
    do {
        try validate(form)
        // continue with request
    } catch {
        switch error {
        case .missingEmail:
            // focus email field
            break
        case .invalidEmail:
            // show email format message
            break
        case .weakPassword:
            // show password policy message
            break
        }
    }
}
```

Why this is good:

```text
The caller can handle every validation error intentionally.
The error cases are domain-level, not transport-level.
The API is local enough that evolution is manageable.
```

---

## 6. Exercise

### Problem

Refactor a generic transform API from `rethrows` or untyped `throws` to typed throws and explain the tradeoff.

### Bad / naive version

```swift
extension Collection {
    func transformed<Output>(
        _ transform: (Element) throws -> Output
    ) throws -> [Output] {
        var output: [Output] = []
        output.reserveCapacity(underestimatedCount)

        for element in self {
            output.append(try transform(element))
        }

        return output
    }
}
```

### What is wrong?

```text
The function is always throwing, even when the transform is non-throwing.
The thrown error type is erased to any Error.
Callers lose the specific transform failure type.
The signature says the collection operation itself may throw independently,
even though it only propagates transform errors.
```

A better pre-typed-throws version is `rethrows`:

```swift
extension Collection {
    func transformed<Output>(
        _ transform: (Element) throws -> Output
    ) rethrows -> [Output] {
        var output: [Output] = []
        output.reserveCapacity(underestimatedCount)

        for element in self {
            output.append(try transform(element))
        }

        return output
    }
}
```

This fixes conditional throwing, but not error-type precision.

### Improved version with typed throws

```swift
extension Collection {
    func transformed<Output, Failure: Error>(
        _ transform: (Element) throws(Failure) -> Output
    ) throws(Failure) -> [Output] {
        var output: [Output] = []
        output.reserveCapacity(underestimatedCount)

        for element in self {
            output.append(try transform(element))
        }

        return output
    }
}
```

Usage:

```swift
enum ValidationError: Error, CustomStringConvertible {
    case negative(Int)

    var description: String {
        switch self {
        case .negative(let value): "negative(\(value))"
        }
    }
}

func requirePositive(_ value: Int) throws(ValidationError) -> Int {
    guard value >= 0 else { throw .negative(value) }
    return value
}

let numbers = [1, 2, 3]
let doubled = numbers.transformed { $0 * 2 }
print(doubled)

do {
    let checked = try [1, -2, 3].transformed(requirePositive)
    print(checked)
} catch {
    print(type(of: error), error)
}
```

Output, verified with Swift 6.2.1:

```text
[2, 4, 6]
ValidationError negative(-2)
```

### Why this is better

```text
If transform is non-throwing:
  Failure == Never
  transformed is non-throwing at the call site

If transform throws ValidationError:
  transformed throws ValidationError
  caller keeps precise error information

If transform throws any Error:
  Failure == any Error
  behavior matches traditional untyped throws
```

### Tradeoff

|Design|When it is appropriate|Tradeoff|
|---|---|---|
|`throws`|The function can fail independently or error type is intentionally erased|Requires `try` even when closure is non-throwing; loses precision|
|`rethrows`|Pre-Swift-6-compatible higher-order functions that only propagate closure errors|Preserves conditional throwing but not thrown error type|
|`throws(Failure)`|Generic algorithms that directly propagate the closure’s error type|More complex signature; can overcomplicate public APIs|
|`Result<Output, Failure>`|Outcome must be stored, passed, aggregated, or delivered through callbacks|More wrapping/unwrapping; less natural for linear control flow|

---

## 7. Production guidance

Use typed throws in production when:

```text
The error domain is stable and intentionally part of the API.
The caller can meaningfully handle all failure cases.
The API is internal, package-local, or domain-specific.
You are writing generic infrastructure that forwards a closure's error type.
You want to avoid type erasure in a narrow performance-sensitive path.
```

Be careful when:

```text
The function calls dependencies whose error model may evolve.
The API is public and source compatibility matters.
The failure is mostly logged, rendered, or propagated rather than exhaustively handled.
You are tempted to create huge umbrella error enums just to satisfy typed throws.
You need to combine multiple unrelated error domains.
```

Avoid typed throws when:

```text
The precise error type is not useful to callers.
The error domain is unstable.
The API boundary is a large app/service/SDK boundary where dependencies may change.
The signature becomes more complex than the handling logic it improves.
```

Debugging checklist:

```text
Did try? accidentally erase the error I needed?
Is try! hiding a real recoverable failure?
Is this function truly throwing its own errors, or only forwarding a closure error?
Should this be throws, rethrows, throws(Failure), or Result?
Is the thrown type stable enough to be public API?
Am I wrapping dependency errors intentionally, or leaking implementation details?
Does the catch block preserve enum exhaustiveness with switch?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `throws` is for errors, `try?` gives nil, `try!` crashes, and `Result` stores success or failure.

### Senior answer

> Swift error handling is an explicit alternate return path. I use `throws` for propagation, `Result` when success/failure must be a value, avoid `try?` when diagnostics matter, and reserve `try!` for true invariants. With typed throws, I can make the error path precise, but I avoid exposing narrow thrown types when the failure domain is unstable.

### Staff-level answer

> I treat the thrown error type as an API design decision. Untyped `throws` is a good default for broad app and SDK boundaries because errors often evolve and are usually propagated or rendered. Typed throws are valuable in stable domains, validation, parsing, and generic algorithms that forward closure errors. For public APIs, I consider source compatibility, dependency leakage, error wrapping strategy, and whether clients can actually recover from specific cases. I do not use typed throws just to make signatures look precise.

Staff-level questions to ask:

```text
Will clients recover differently from each error case, or just log/render it?
Is this error type stable enough to expose as public API?
Are we leaking dependency implementation details through the thrown type?
Would Result model this better because the outcome must be stored or streamed?
Is this higher-order function forwarding errors, substituting errors, or failing independently?
```

---

## 9. Interview-ready summary

Swift error handling models failure as an explicit alternate return path. Plain `throws` means the error path is type-erased to `any Error`; typed throws lets a function declare a specific thrown type like `throws(LoginError)`; `throws(Never)` means the function cannot throw and is equivalent to non-throwing. I use `throws` for immediate propagation, `Result` when success/failure must be stored or passed as a value, and typed throws when the failure domain is stable and useful to callers. For generic higher-order APIs, typed throws can replace many `rethrows` patterns by preserving the closure’s error type, but it is not always better for public APIs because it can freeze implementation detail into the signature.

---

## 10. Flashcards

Q: What is plain `throws` equivalent to in the typed-throws model?  
A: `throws(any Error)`.

Q: What does `throws(Never)` mean?  
A: The function cannot throw; it is equivalent to a non-throwing function.

Q: What does `try?` do to a thrown error?  
A: It discards the error and returns `nil`.

Q: When should you prefer `Result` over `throws`?  
A: When success/failure must be stored, passed around, collected, retried, streamed, or delivered through callback-style boundaries.

Q: What does `rethrows` express?  
A: The function only throws if one of its throwing function parameters throws.

Q: What does typed throws add beyond `rethrows`?  
A: It can preserve the static type of the error being forwarded.

Q: Why can typed throws be risky in public APIs?  
A: The thrown type becomes part of the API contract and can constrain future implementation changes.

Q: How should you exhaustively handle a typed enum error?  
A: Use an unconditional `catch`, then `switch error` inside the catch.

---

## 11. Related sections

- [[A4 — Optionals and nil modeling]]
- [[A5 — Pattern matching and exhaustive control flow]]
- [[A7 — Functions, parameter labels, defaults, and API surface]]
- [[B3 — Generics, constraints, and same-type relationships]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[D9 — Continuations and bridging callback APIs]]

---

## 12. Sources

- "Swift Senior/Staff Rubric." A13 Error handling and typed throws.
- GitHub. "SE-0413: Typed Throws." swift-evolution. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md
- Swift.org. "Error Handling." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/
- Apple Developer. "Result." Apple Developer Documentation. https://developer.apple.com/documentation/swift/result
