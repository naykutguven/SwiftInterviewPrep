---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - error-handling
  - typed-throws
---

# A13 — Error Handling and Typed Throws

**Learning goals:**

- Choose throws, rethrows, Result, try?, or an asserted try! from the caller’s recovery needs.
- Trace typed-error propagation and catch analysis without erasing failure information.
- Explain throws(Never) and Swift 6.4 async Result initialization, including cancellation and noncopyable success.

**Prerequisites:** [[A4 — Optionals and Nil Modeling]] for Optional transformations, [[A5 — Pattern Matching and Exhaustive Control Flow]] for exhaustive switch, and [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]/[[A8 — Closures, Escaping, and Capture Semantics]] for closure effects. The async extension uses [[D1 — Structured Concurrency Fundamentals]]/[[D2 — Task Hierarchy, Cancellation, and Priorities]].

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A parser can produce an integer or explain why it could not. throws expresses those alternatives as control flow: return a value, or propagate an error to a handler. Result stores the alternatives as data: success(value) or failure(error). Choose based on whether the caller is executing a sequence or retaining an outcome for later.

**Typed throws** names the error type, such as throws(ParseFailure), rather than permitting any Error. This makes the failure contract explicit, but does not choose retry, presentation, or cancellation policy. The concrete error type is also an API evolution commitment. [SE-0413](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Propagate a failure or retain it as a value

The adapter changes the shape of the outcome without changing the error.

```swift
enum ParseFailure: Error, Equatable { case invalid(String) }
func parse(_ raw: String) throws(ParseFailure) -> Int {
    guard let value = Int(raw) else { throw .invalid(raw) }
    return value
}
func outcome(_ raw: String) -> Result<Int, ParseFailure> {
    do { return .success(try parse(raw)) } catch { return .failure(error) }
}
let number = try parse("42")
precondition(number == 42)
precondition(outcome("bad") == .failure(.invalid("bad")))
```

**Walkthrough:** try marks a potentially throwing call. The unconditional catch receives ParseFailure here because parse has typed throws. Result stores the same failure for later inspection.

### rethrows ties failure to a closure parameter

The helper can throw only as a consequence of calling its throwing input. Nonthrowing input needs no try at the call site.

```swift
func transform<T, U>(_ value: T, using operation: (T) throws -> U) rethrows -> U {
    try operation(value)
}
let doubled = transform(3) { $0 * 2 }
precondition(doubled == 6)
enum Problem: Error { case rejected }
do {
    _ = try transform(3) { _ throws -> Int in throw Problem.rejected }
    preconditionFailure("Expected failure")
} catch { precondition(error is Problem) }
```

**Walkthrough:** The same helper is nonthrowing for the first call and propagates the closure’s error in the second. rethrows does not permit it to introduce an unrelated I/O error.

## Rules and pitfalls

### try? can discard the recovery information

**Common pitfall:** Converting a failure to a default success-looking number.

**Mistake:** Standalone runnable example.

```swift
enum Failure: Error { case invalid(String) }
func parse(_ raw: String) throws -> Int {
    guard let value = Int(raw) else { throw Failure.invalid(raw) }
    return value
}
let value = (try? parse("bad")) ?? 0
precondition(value == 0)
```

**Consequence:** Malformed input becomes indistinguishable from a real zero.

**Why:** try? maps the error to nil, and ?? then replaces it. try! would assert that failure cannot occur and trap for this input.

**Correction:** Keep the typed error while callers still need to recover.

```swift
enum Failure: Error, Equatable { case invalid(String) }
func parse(_ raw: String) throws(Failure) -> Int {
    guard let value = Int(raw) else { throw .invalid(raw) }
    return value
}
do {
    _ = try parse("bad")
    preconditionFailure()
} catch { precondition(error == .invalid("bad")) }
let validZero = try parse("0")
precondition(validZero == 0)
```

**What changes:** The invalid path keeps its input and failure identity; zero remains a distinct valid result. Use try? only when absence is the complete contract.

### Case catches are not exhaustive enum-switch analysis

**Common pitfall:** Assuming separately named typed enum catches prove that a nonthrowing wrapper handles every error.

**Mistake:** Compile-only; intentionally rejected.

```swift
enum Problem: Error { case empty, malformed }
func fail() throws(Problem) -> Int { throw .empty }
func invalidWrapper() -> Int {
    do { return try fail() } catch Problem.empty { return 0 } catch Problem.malformed { return -1 }
}
```

**Consequence:** This compiler rejects the unhandled throwing path despite the two named cases.

**Why:** Catch-pattern coverage is not the same analysis as exhaustive switching over a known enum.

**Correction:** Catch unconditionally, then switch over the typed error.

```swift
enum Problem: Error { case empty, malformed }
func parse(_ text: String) throws(Problem) -> Int {
    guard !text.isEmpty else { throw .empty }
    guard let value = Int(text) else { throw .malformed }
    return value
}
func rendered(_ text: String) -> String {
    do { return String(try parse(text)) } catch {
        switch error {
        case .empty: return "empty"
        case .malformed: return "malformed"
        }
    }
}
precondition(rendered("") == "empty")
precondition(rendered("bad") == "malformed")
precondition(rendered("7") == "7")
```

**What changes:** The unconditional catch handles the throwing path. The inner switch receives a concrete Problem and checks its cases exhaustively.

### Independent failure requires throws

**Common pitfall:** Adding a helper-owned error to a rethrows operation.

**Mistake:** Compile-only; intentionally rejected.

```swift
enum Failure: Error { case storage }
func invalid(_ body: () throws -> Void) rethrows {
    try body()
    throw Failure.storage
}
```

**Consequence:** The compiler rejects an independent throw from a rethrows function.

**Why:** rethrows permits failure derived from its throwing closure parameters, not unrelated helper work.

**Correction:** Declare throws when the operation has its own failure source.

```swift
enum Failure: Error { case storage }
func perform(_ body: () throws -> Void, storageAvailable: Bool) throws {
    try body()
    guard storageAvailable else { throw Failure.storage }
}
do {
    try perform({}, storageAvailable: false)
    preconditionFailure()
} catch { precondition(error is Failure) }
try perform({}, storageAvailable: true)
```

**What changes:** Callers now see that this helper can fail even when their closure cannot. A typed union error can be appropriate when preserving multiple failure origins matters.

## Worked example

### Choose Result for a retained batch outcome

**Setup:** Standalone synchronous script. Each input gets one stored outcome; success is not substituted for invalid input.

**Predict:** Can the caller later identify which input failed?

```swift
enum Failure: Error, Equatable { case invalid(String) }
func parse(_ raw: String) throws(Failure) -> Int {
    guard let value = Int(raw) else { throw .invalid(raw) }
    return value
}
func outcome(_ raw: String) -> Result<Int, Failure> {
    do { return .success(try parse(raw)) } catch { return .failure(error) }
}
let outcomes = ["1", "bad", "2"].map(outcome)
precondition(outcomes == [.success(1), .failure(.invalid("bad")), .success(2)])
```

**Outcome:** The middle entry retains the malformed input and remains in order.

**Why:** An outcome value is useful because the batch stores every result. In a simple sequential operation, throws would avoid this extra wrapping and switching.

## Interview questions

### Why can separate typed enum-case catches fail while an unconditional catch with switch works?

**Answer:** The compiler does not generally prove catch-pattern exhaustiveness by enumerating error cases. An unconditional catch handles the throwing path, then an enum switch can prove coverage of the concrete typed error.

**Example:** Standalone runnable example.

```swift
enum Problem: Error { case missing, invalid }
func load(_ present: Bool) throws(Problem) -> Int {
    guard present else { throw .missing }
    throw .invalid
}
func label(_ present: Bool) -> String {
    do { return String(try load(present)) } catch {
        switch error {
        case .missing: return "missing"
        case .invalid: return "invalid"
        }
    }
}
precondition(label(false) == "missing")
precondition(label(true) == "invalid")
```

**Walkthrough:** Both failure paths enter one catch; error is Problem, so the switch needs no default. The pitfall gives the separately compiled rejected catch-only variant.

**Boundary:** If the operation instead throws any Error, narrowing to Problem does not handle every possible error. Preserve a fallback or propagate the remaining errors deliberately.

### When should you return Result instead of throwing?

**Answer:** Use Result when an outcome must be stored, passed to a callback, or inspected later as data. Use throws for ordinary sequential propagation. Neither changes task lifetime or chooses recovery policy.

**Example:** Standalone runnable example.

```swift
enum Failure: Error, Equatable { case offline }
func fetch(_ online: Bool) throws(Failure) -> String {
    guard online else { throw .offline }
    return "data"
}
func deliver(_ online: Bool, completion: (Result<String, Failure>) -> Void) {
    do { completion(.success(try fetch(online))) } catch { completion(.failure(error)) }
}
var retained: Result<String, Failure>?
deliver(false) { retained = $0 }
precondition(retained == .failure(.offline))
deliver(true) { retained = $0 }
precondition(retained == .success("data"))
```

**Walkthrough:** The callback transports a single outcome value that can be retained. Internally, fetch still uses straightforward throwing control flow.

**Boundary:** This callback runs synchronously exactly once because this implementation does so. Result does not itself promise delivery timing, a queue, or cancellation support.

### What does throws(Never) mean semantically?

**Answer:** It means no error value can be thrown, because Never has no instances. Calls are semantically nonthrowing and can participate in generic typed-throws relationships.

**Example:** Standalone runnable example.

```swift
func doubled(_ value: Int) throws(Never) -> Int { value * 2 }
let operation: (Int) -> Int = doubled
precondition(doubled(3) == 6)
precondition(operation(4) == 8)
```

**Walkthrough:** Neither call needs try. The function value can be used through a nonthrowing type.

**Boundary:** It does not guarantee normal return: a function can trap or never terminate. It also makes no promise to observe cancellation; cancellation only becomes a thrown error when the operation’s contract allows and performs it.

### Extension: does awaiting a Result initializer start a task or handle cancellation?

**Answer:** No. Swift 6.4’s async initializer awaits its operation in the current task and captures the thrown error. It neither creates a child task nor clears or automatically checks cancellation. An explicit closure error annotation can preserve a concrete failure type.

**Example:** Standalone runnable script compiled for macOS 12 using the current SDK. It runs on the current host.

```swift
enum Problem: Error, Equatable { case missing }
func fail() async throws(Problem) -> Int { throw .missing }
let result: Result<Int, Problem> = await Result {
    () async throws(Problem) -> Int in
    try await fail()
}
precondition(result == .failure(.missing))
let task = Task { () -> Bool in
    withUnsafeCurrentTask { $0?.cancel() }
    let captured: Result<Int, any Error> = await Result {
        () async throws -> Int in
        try Task.checkCancellation()
        return 42
    }
    guard case .failure(let error) = captured else { return false }
    let ignored: Result<Int, any Error> = await Result {
        () async throws -> Int in 42
    }
    guard case .success(42) = ignored else { return false }
    return error is CancellationError && Task.isCancelled
}
let observedCancellation = await task.value
precondition(observedCancellation)
```

**Walkthrough:** The annotated first closure preserves Problem. The explicit Task in this test creates the controlled cancellation context; each Result initializer inside it uses that same task. In the cancelled task, checkCancellation throws and Result stores that error; the second operation ignores cancellation and succeeds. The task remains cancelled. [SE-0530](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0530-async-result-support.md)

**Boundary:** Capturing cancellation is not handling it. Decide whether failure(CancellationError()) should propagate rather than enter service retries. This initializer compiles for macOS 12 with the selected SDK; operations called inside it retain their own availability and isolation requirements.

This independent **intentionally rejected** variant omits the closure error annotation and infers an incompatible erased error type in this compiler:

```swift
enum Problem: Error { case missing }
func fail() async throws(Problem) -> Int { throw .missing }
func capture() async {
    let result: Result<Int, Problem> = await Result { try await fail() }
    _ = result
}
```

### Extension: can async Result carry a noncopyable success?

**Answer:** Yes, the selected initializer supports a noncopyable success payload. Consuming the result transfers that payload; it does not make the payload Copyable or define a sharing policy.

**Example:** Standalone runnable example.

```swift
struct Token: ~Copyable { let id: Int }
enum Failure: Error { case missing }
func checkToken() async {
    let result: Result<Token, Failure> = await Result {
        () async throws(Failure) -> Token in
        await Task.yield()
        return Token(id: 42)
    }
    switch consume result {
    case .success(let token): precondition(token.id == 42)
    case .failure: preconditionFailure()
    }
}
await checkToken()
```

**Walkthrough:** The initializer stores one token; the consuming switch moves it into the success branch for use.

**Boundary:** [[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] covers ownership. [[D9 — Continuations and Bridging Callback APIs|D9]] separately checks typed continuation factories and the newer runtime requirements of noncopyable continuation handles; those requirements must not be attributed to every typed Result API.

## Focused exercise

**Task:** Implement a failing transform with typed throws, then expose the same outcome as Result.

**Acceptance checks:**

- Valid input yields its integer.
- Malformed and empty input retain the original text in the typed failure.
- No conversion to nil or fabricated success erases failure.

**Solution:** Standalone synchronous fixture.

```swift
enum ParseFailure: Error, Equatable { case invalid(String) }
func parse(_ raw: String) throws(ParseFailure) -> Int {
    guard let value = Int(raw) else { throw .invalid(raw) }
    return value
}
func outcome(_ raw: String) -> Result<Int, ParseFailure> {
    do { return .success(try parse(raw)) } catch { return .failure(error) }
}
precondition(outcome("42") == .success(42))
precondition(outcome("0") == .success(0))
precondition(outcome("bad") == .failure(.invalid("bad")))
precondition(outcome("") == .failure(.invalid("")))
```

**Reasoning:** The adapter preserves exactly the success/failure alternatives of parse. Int’s accepted grammar is the policy here; [[A16 — Regex and standard-library idioms|A16]] shows stricter grammar and overflow classification. If the transform gains independent failure sources, revise its typed contract deliberately.

## In practice

**Choose it when:** Choose the effect shape from whether callers execute an operation or retain its outcome.

**Investigate by:** Trace every conversion to nil, erased Error, or a domain error, including cancellation.

**Strong evidence:** Define error evolution and cancellation policy at public boundaries, then verify success, ordinary failure, and cancellation separately.

## References

### Related guides

- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[A4 — Optionals and Nil Modeling]]
- [[A5 — Pattern Matching and Exhaustive Control Flow]]
- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[D9 — Continuations and Bridging Callback APIs]]

### Official sources

- [SE-0413: typed throws](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md)
- [SE-0530](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0530-async-result-support.md)
- [Swift.org. "Error Handling." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/)
- [Apple Developer. "Result." Apple Developer Documentation](https://developer.apple.com/documentation/swift/result)
