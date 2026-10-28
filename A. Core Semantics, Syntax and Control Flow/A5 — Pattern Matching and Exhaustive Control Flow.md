---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - pattern-matching
  - control-flow
---

# A5 — Pattern Matching and Exhaustive Control Flow

**Learning goals:**

- Trace first-match pattern order, payload bindings, guarded subsets, and optional/tuple patterns.
- Distinguish per-pattern where clauses and fallthrough from ordinary case selection.
- Check closed-enum coverage and an external enum’s future-case policy in a separate module.

**Prerequisites:** [[A4 — Optionals and Nil Modeling]] for Optional cases. [[A9 — Enums, Associated Values, Recursive Enums, and State Modeling]] explains domain-state enums; [[B11 — Library Evolution and Resilience-Aware API Design]] develops binary resilience.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A response can be successful, failed, or cancelled. A pattern asks whether the value has a particular shape, and a binding gives a name to a payload inside that shape. A switch tries cases in source order and selects the first matching pattern whose condition holds.

**Exhaustiveness** means every possible value has a branch. It does not mean every branch implements the right business rule. An unconditional default can hide a new response case; explicit cases can instead require a compiler-guided review. [Patterns](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/patterns/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Bind a payload, then narrow it

The first status branch handles only a range. The following unguarded status branch handles its remainder.

```swift
enum Response {
    case status(Int)
    case failed(String)
    case cancelled
}
func message(_ response: Response) -> String {
    switch response {
    case .status(let code) where (200..<300).contains(code): "ok \(code)"
    case .status(let code): "unexpected \(code)"
    case .failed(let reason): "failure \(reason)"
    case .cancelled: "cancelled"
    }
}
precondition(message(.status(204)) == "ok 204")
precondition(message(.status(404)) == "unexpected 404")
```

**Walkthrough:** let binds code for the guard and body. The first matching case wins; putting unguarded status first would swallow the success branch.

### Filter with patterns and combine tuple conditions

for case skips nonmatching elements. A tuple pattern can inspect several components at once; grouped alternatives share one body.

```swift
let values: [Int?] = [1, nil, 3]
var present: [Int] = []
for case .some(let value) in values { present.append(value) }
precondition(present == [1, 3])
func action(_ state: (Bool, Int)) -> String {
    switch state {
    case (true, 0), (false, 0): "empty"
    case (true, let count) where count > 0: "send"
    default: "wait"
    }
}
precondition(action((false, 0)) == "empty")
precondition(action((true, 2)) == "send")
precondition(action((true, -1)) == "wait")
```

**Walkthrough:** nil elements do not enter the loop body. The tuple switch’s grouped zero patterns need no bindings; the positive branch binds count. if case and guard case use the same pattern idea for a single value.

### fallthrough enters a body without retesting

This is an explicit control-flow jump. Ordinary Swift cases do not fall through by default.

```swift
func trace(_ input: Int) -> [String] {
    var events: [String] = []
    switch input {
    case 2:
        events.append("two")
        fallthrough
    case 99:
        events.append("next body")
    default:
        events.append("other")
    }
    return events
}
precondition(trace(2) == ["two", "next body"])
```

**Walkthrough:** The second body runs even though 2 does not match 99. Use a shared function when the intent is to share work without this jump; fallthrough does not create bindings for the destination pattern.

## Rules and pitfalls

### A guarded case does not cover its whole payload space

**Common pitfall:** Handling only successful status integers while assuming the status case is exhausted.

**Mistake:** Compile-only; intentionally rejected.

```swift
enum Response {
    case status(Int)
    case failed, cancelled
}
func render(_ response: Response) -> String {
    switch response {
    case .status(let code) where (200..<300).contains(code): "ok"
    case .failed: "failed"
    case .cancelled: "cancelled"
    }
}
```

**Consequence:** The compiler reports a nonexhaustive switch: status values outside the range have no branch.

**Why:** The where clause covers only a subset of Int payloads.

**Correction:** Add an explicit unguarded remainder for that case.

```swift
enum Response {
    case status(Int)
    case failed, cancelled
}
func render(_ response: Response) -> String {
    switch response {
    case .status(let code) where (200..<300).contains(code): "ok"
    case .status(let code): "unexpected \(code)"
    case .failed: "failed"
    case .cancelled: "cancelled"
    }
}
precondition(render(.status(199)) == "unexpected 199")
precondition(render(.status(200)) == "ok")
```

**What changes:** The remainder is deliberate, and a newly added enum case still requires review.

### where belongs to its individual pattern

**Common pitfall:** Writing one guard after a comma-separated group and expecting it to constrain every alternative. This warning-bearing script is run without -warnings-as-errors.

**Mistake:** Runnable with the warning described below; compile without `-warnings-as-errors`.

```swift
enum Event {
    case progress(Int)
    case finished(Int)
}
func positive(_ event: Event) -> Bool {
    switch event {
    case let .progress(value), let .finished(value) where value > 0:
        _ = value
        return true
    default: return false
    }
}
precondition(positive(.progress(0)))  // Unwanted true.
```

**Consequence:** progress(0) matches the first unguarded pattern. This compiler also warns that where applies only to the second pattern.

**Why:** Comma-separated patterns are alternatives, each with its own condition.

**Correction:** Repeat the condition or split the cases. This separate corrected script repeats it.

```swift
enum Event {
    case progress(Int)
    case finished(Int)
}
func positive(_ event: Event) -> Bool {
    switch event {
    case .progress(let value) where value > 0,
        .finished(let value) where value > 0:
        true
    default: false
    }
}
precondition(!positive(.progress(0)))
precondition(!positive(.finished(0)))
precondition(positive(.progress(1)) && positive(.finished(1)))
```

**What changes:** Both alternatives now test their payload. Grouped patterns that bind values must bind compatible names and types.

## Worked example

### A catch-all can hide a domain change

**Setup:** Standalone script. The same enum is rendered with explicit cases and with an overly broad fallback.

**Predict:** Does compilation prove that cancellation is handled correctly?

```swift
enum Response {
    case ready(String)
    case failed, cancelled
}
func explicit(_ value: Response) -> String {
    switch value {
    case .ready(let text): text
    case .failed: "failed"
    case .cancelled: "cancelled"
    }
}
func lossy(_ value: Response) -> String {
    switch value {
    case .ready(let text): text
    default: "failed"
    }
}
precondition(explicit(.cancelled) == "cancelled")
precondition(lossy(.cancelled) == "failed")
```

**Outcome:** Both functions compile, but lossy merges cancellation into failure.

**Why:** Exhaustiveness checks branch coverage. Assertions check the chosen meaning. Adding a case would force explicit to change while lossy would keep compiling.

### Shared external-enum setup

@nonexhaustive tells outside consumers that the case list is not closed, so they need a future-case policy even when they handle all current cases. The attribute was introduced in Swift 6.2.3 and is supported by this baseline. This source-checking rule is separate from a library’s binary-resilience contract. [SE-0487](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md)

Save the next block as SDKStates.swift and compile it as a separate module named SDKStates. Using the baseline flags, add `-parse-as-library -module-name SDKStates -emit-module -emit-module-path /tmp/SDKStates.swiftmodule -c -o /tmp/SDKStates.o`. Clients add `-I /tmp`; runnable clients also link `/tmp/SDKStates.o`. Do not paste this declaration into the client: that would remove the module boundary being tested. The validation uses an isolated temporary directory in place of /tmp.

```swift
@nonexhaustive
public enum SDKStatus: Sendable { case ready, waiting }
```

## Interview questions

### Why does a guarded case need remaining payload coverage?

**Answer:** Its where condition excludes some payloads. An unguarded branch for the same enum case covers those values without hiding other domain cases.

**Example:** Standalone runnable example.

```swift
enum Response {
    case status(Int)
    case failed(String)
    case cancelled
}
func message(_ response: Response) -> String {
    switch response {
    case .status(let code) where (200..<300).contains(code): "success"
    case .status(let code): "unexpected \(code)"
    case .failed(let reason): "failed \(reason)"
    case .cancelled: "cancelled"
    }
}
precondition(message(.status(299)) == "success")
precondition(message(.status(300)) == "unexpected 300")
precondition(message(.cancelled) == "cancelled")
```

**Walkthrough:** 299 matches the first branch; 300 fails its guard and reaches the next. Cancellation has an independent branch.

**Boundary:** Correct coverage does not establish whether every 2xx code should have the same business behavior; that is the application’s policy.

### Which external enum omission is an error in Swift 6 mode, and which warns while still reaching fallback?

**Answer:** For this external @nonexhaustive enum, omitting future-case fallback is an error in Swift 6 mode. With @unknown default, omitting a known case produces a warning and that case still executes the fallback.

**Example:** Separate runnable client of Shared external-enum setup; add the client import/search/link flags described there.

```swift
import SDKStates

func describe(_ status: SDKStatus) -> String {
    switch status {
    case .ready: "ready"
    case .waiting: "waiting"
    @unknown default: "future"
    }
}
precondition(describe(.ready) == "ready")
precondition(describe(.waiting) == "waiting")
```

**Walkthrough:** Explicit branches handle both current cases. The fallback supplies behavior if the external API introduces a different case. [SE-0487](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md)

**Boundary:** This attribute controls source checking; it does not enable binary resilience. [[B11 — Library Evolution and Resilience-Aware API Design|B11]] separately validates resilient/frozen/same-package variants and library replacement. @unknown default is not a runtime filter limited to future cases.

This separate client variant is **intentionally rejected** in Swift 6 mode. Use the compiled SDKStates module; object compilation needs only `-I /tmp`.

```swift
import SDKStates

func incomplete(_ status: SDKStatus) -> String {
    switch status {
    case .ready: "ready"
    case .waiting: "waiting"
    }
}
```

This separate variant is **warning-bearing and runnable**. Compile without `-warnings-as-errors` to observe the warning and result. The selected compiler’s warning-promotion limitation is recorded in [[B11 — Library Evolution and Resilience-Aware API Design|B11]]; inspect actual diagnostics and exit status in your build.

```swift
import SDKStates

func omittedKnown(_ status: SDKStatus) -> String {
    switch status {
    case .ready: "ready"
    @unknown default: "fallback"
    }
}
precondition(omittedKnown(.waiting) == "fallback")
```

A plain default also reaches fallback for waiting, but suppresses the useful omitted-known-case warning:

```swift
import SDKStates

func plainDefault(_ status: SDKStatus) -> String {
    switch status {
    case .ready: "ready"
    default: "fallback"
    }
}
precondition(plainDefault(.waiting) == "fallback")
```

## Focused exercise

**Task:** Map a response enum using guarded success statuses and a deliberate remainder.

**Acceptance checks:**

- Check 199, 200, 299, and 300.
- Failure and cancellation retain separate outcomes.
- Removing the unguarded status branch is rejected, as in the pitfall.

**Solution:** Standalone script.

```swift
enum Response {
    case status(Int)
    case failed(String)
    case cancelled
}
func message(_ response: Response) -> String {
    switch response {
    case .status(let code) where (200..<300).contains(code): "success"
    case .status(let code): "unexpected \(code)"
    case .failed(let reason): "failed \(reason)"
    case .cancelled: "cancelled"
    }
}
for code in [199, 200, 299, 300] {
    let expected = (200..<300).contains(code) ? "success" : "unexpected \(code)"
    precondition(message(.status(code)) == expected)
}
precondition(message(.failed("offline")) == "failed offline")
precondition(message(.cancelled) == "cancelled")
```

**Reasoning:** The success branch subtracts one range from the status payload space. The unguarded status branch covers everything left without absorbing a future Response case.

## In practice

**Choose it when:** Use explicit exhaustive cases for closed domain mappings and selective patterns for filtering events.

**Investigate by:** List each case’s payload space, then subtract guarded subsets in source order. Review where conditions individually.

**Strong evidence:** Test boundary payloads and the actual external-module fallback, separating source warnings from ABI and deployment guarantees.

## References

### Related guides

- [[A4 — Optionals and Nil Modeling]]
- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[A9 — Enums, Associated Values, Recursive Enums, and State Modeling]]
- [[A12 — Availability and Conditional Compilation]]
- [[A13 — Error handling and typed throws]]
- [[A14 — Operators, Literals, and Language Affordances]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [Patterns](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/patterns/)
- [SE-0487](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md)
- [SE-0192](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0192-non-exhaustive-enums.md)
- [Statements](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/statements/)
