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

**Goal:** Expose a throwing transform as a Result without discarding its failure type.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A throwing function returns a value or propagates an error. Result stores those alternatives as data when the outcome must be retained, passed to a callback, or examined later. Typed throws constrains the error contract; it does not choose retry, cancellation, or presentation policy for the caller. [SE-0413: typed throws](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md)

## Rules and pitfalls

### Choose effects from the caller's needs

Prefer throws for ordinary sequential failure propagation. Prefer Result when an outcome is a stored or transported value. Rethrows relates failure to throwing closure parameters; it cannot describe independent I/O failure.

**Common pitfall:** Replacing every throwing API with Result forces callers to unwrap manually without gaining a useful boundary. Conversely, try? merges thrown failure with absence, and try! asserts that failure cannot occur. Preserve errors when callers need recovery.

### Treat typed errors as evolution commitments

A concrete error type makes failure relationships explicit and can improve generic reasoning. Adding cases or changing that type affects callers and resilience policy. throws(Never) means the function has no possible thrown error.

**Common pitfall:** Enumerating typed enum catches does not necessarily satisfy catch exhaustiveness analysis. Use an unconditional catch, then switch exhaustively over the typed error. Keep cancellation recognizable at boundaries that can throw it.

## Interview questions

### Does awaiting a Result initializer start a new task or handle cancellation for you?

No. The Swift 6.4 async Result initializer awaits a closure in the current task; it does not spawn work or clear cancellation. It captures a thrown error as failure, which can include cancellation when the selected failure type permits it. An explicitly typed closure may be needed to preserve a concrete error type.

### Why can separate typed enum-case catches fail exhaustiveness checking while an unconditional catch with a switch works?

Catch-pattern checking is not equivalent to enum-switch exhaustiveness checking. Separate enum-case catches can leave a rejected throwing path even when their names look complete. An unconditional catch captures the concrete typed error; switching on it gives the compiler the enum coverage problem it can prove.

### When should you return `Result` instead of throwing?

Return Result when storing, forwarding, or composing the outcome as a value is the requirement, such as a callback result or retained batch entry. Use throws when callers naturally await or call and handle failure through control flow. Neither inherently changes task lifetime or the error's meaning.

### What does `throws(Never)` mean semantically?

It states that no value can be thrown because Never has no instances. Semantically the error path is impossible. It can participate in generic typed-throws relationships, but does not guarantee that the function returns: traps, nontermination, and cancellation-insensitive work are separate concerns.

## Focused exercise

**Task:** Implement a failing transform with throws, then expose its outcome as Result. Check success and preserved error information.

**Acceptance checks:**

- Valid input produces the parsed value.
- Invalid and empty input retain the original value in the typed failure.
- No try? or fabricated success erases the failure path.

**Solution:**

Standalone synchronous typed-error fixture:

```swift
enum ParseFailure: Error, Equatable { case invalid(String) }
func parse(_ raw: String) throws(ParseFailure) -> Int {
    guard let value = Int(raw) else { throw .invalid(raw) }
    return value
}
func outcome(_ raw: String) -> Result<Int, ParseFailure> {
    do { return .success(try parse(raw)) }
    catch { return .failure(error) }
}
precondition(outcome("42") == .success(42))
precondition(outcome("bad") == .failure(.invalid("bad")))
precondition(outcome("") == .failure(.invalid("")))
```

**Reasoning:** The adapter stores the same success/error alternatives without changing parsing policy. A real parser may need stricter grammar than Int accepts; grammar and overflow tests belong to the domain boundary. If the transform later introduces independent failure, revise the typed error contract deliberately.

## In practice

**Choose it when:** Use the effect shape that matches whether callers execute or retain the outcome.

**Investigate by:** Trace every conversion to nil, erased Error, and domain failure, including cancellation.

**Strong evidence:** Define error evolution and cancellation behavior, then test both ordinary failure and cancellation propagation.

### Async Result and cancellation — optional

#### Swift 6.4: capture async failure in a Result

The catching initializer now has an async overload. Await it to turn an async throwing operation into a stored success/failure value; creating the result does not spawn another task. Explicit closure error annotations can be necessary: removing the annotation in the probe below infers any Error and is rejected as a conversion to Problem in the checked compiler. [SE-0530](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0530-async-result-support.md)

```swift
enum Problem: Error { case missing }
func fail() async throws(Problem) -> Int { throw .missing }

@main struct Main {
    static func main() async {
        let value: Result<Int, Problem> = await Result {
            () async throws(Problem) -> Int in
            try await fail()
        }
        if case .failure(.missing) = value {
            print("typed failure")
        } else {
            preconditionFailure()
        }
    }
}
```

Standalone probe with `-parse-as-library`; observed output: `typed failure`. The async initializer compiles for the older targets recorded here; it does not require OS 27 just because the compiler is 6.4. Calls made inside the operation still have their own availability and isolation requirements.

Capturing an error is not handling it. For an operation that can throw cancellation, decide whether a `.failure(CancellationError())` must propagate rather than be treated as a retryable service failure. Swift 6.2/6.3 required a manual async do/catch wrapper for this conversion. Async defer in 6.4 is complementary: handle teardown errors inside the defer, or use explicit finalization when its error should determine the function's result.

A deterministic cancellation probe (standalone with -parse-as-library):

```swift
@main struct Main {
    static func main() async {
        withUnsafeCurrentTask { $0?.cancel() }
        let value: Result<Int, any Error> = await Result {
            () async throws -> Int in
            try Task.checkCancellation()
            return 42
        }
        if case .failure(let error) = value {
            print(error is CancellationError)
        }
        print(Task.isCancelled)
    }
}
```

Both lines print true. The check inside the operation throws; Result captures that error. The initializer neither checks cancellation automatically nor clears the task's state. An operation that ignores cancellation can still return success from a cancelled task.

At staff depth, the initializer also accepts a noncopyable success value:

```swift
struct Token: ~Copyable { var id: Int }
enum Failure: Error { case missing }

@main struct Main {
    static func main() async {
        let result: Result<Token, Failure> = await Result {
            () async throws(Failure) -> Token in
            await Task.yield()
            return Token(id: 42)
        }
        switch consume result {
        case .success(let token): print(token.id)
        case .failure: print("failed")
        }
    }
}
```

This prints 42. Consume the result to take ownership of its payload; this does not make the token Copyable or infer a concurrency-sharing policy. See C8 for the ownership foundations.

D9 checks typed failures at continuation boundaries: both withContinuation(of:throwing:_:) and an explicitly typed CheckedContinuation<Int, Problem> parameter preserve Problem in the selected Swift 6.4 SDK. The new noncopyable handle also supports noncopyable success and requires OS 27 here; those are separate from typed-throws support. Avoid retaining an older blanket claim that checked factories only construct Never or any Error.

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
