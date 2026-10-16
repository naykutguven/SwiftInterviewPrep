---
tags:
  - swift
  - ios
  - interview-prep
  - tooling-testing-diagnostics
  - testing-strategy
  - swift-testing
  - xctest
  - concurrency
---

# F1 — Testing Strategy in Modern Swift

**Goal:** Advance a controllable time dependency to verify a debounce deadline and cancelled result.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A useful test controls the events that determine behavior and observes the actual contract. Starting a task, knowing it reached a boundary, requesting cancellation, and joining completion are different steps. Unit tests isolate policy; integration/UI tests cover real framework delivery, external I/O, and view lifecycle. A manually released gate alone does not verify elapsed-time arithmetic. [Swift Testing](https://developer.apple.com/xcode/swift-testing/)

## Rules and pitfalls

### Control time and completion separately

Inject a Clock or a controllable time dependency when debounce/timeout behavior depends on duration. Test before, at, and after the boundary, plus replacement and cancellation. Join owned tasks before ending the test.

**Common pitfall:** Sleeping for a guessed duration makes scheduler speed part of the assertion. Advance virtual time and use an explicit registration acknowledgment so the test knows the operation is waiting.

### Verify the test runner's failure path

#expect usually records and continues; #require throws on failure. Confirmation scopes do not wait for arbitrary future callbacks by themselves. Serialized tests are not a global lock or a reliable ordering contract for unrelated suites.

**Common pitfall:** Framework interoperability can leave an assertion reported through the wrong runner. Run deliberate failures separately and retain the original test process status through output formatting; a passing later repetition does not erase an earlier failure.

## Interview questions

### What does a manual debounce gate prove, and what still needs a virtual-clock or integration test?

It proves behavior after a manually chosen release and can verify request/cancellation ordering. It does not prove a ten-millisecond threshold or correct clock choice. A virtual clock/time dependency checks deadline arithmetic; integration tests still cover real scheduling, framework callbacks, and external effects.

### How would a controllable clock verify debounce or timeout boundaries without sleeping? Which scheduler or external-I/O behavior still needs integration coverage?

Register a wait against a known deadline, advance to just before it, confirm work remains pending, then advance to the boundary and join the emitted result. Cancel another operation and verify no successful emission. Keep separate integration coverage for real timer scheduling, system sleep policy, and external I/O cancellation.

### Why can limited interop leave an XCTest failure inside Swift Testing passing, and why must a runner's exit status survive output formatting?

Interoperability depends on direction, mode, runner, and XCTest implementation. Under limited interop an XCTest assertion inside a Swift Testing test may not fail that test as expected. A formatter can also hide failure if a pipeline returns only its own status. Test a known failure and preserve the actual runner exit status.

### How do warning severity, Test.cancel, CustomTestReflectable, attachments, and per-case repetitions affect evidence without replacing assertions?

Warning issues, cancellation diagnostics, custom reflected values, attachments, and repetitions improve evidence and debugging context. They do not replace a failing assertion on the contract. Record which cases/repetitions failed and check platform/framework availability for each feature.


## Worked example

**Setup:** A separate Swift Testing target that is expected to fail; keep it out of the passing clock suite.

**Predict:** Which statements run after a failed expectation and a failed requirement?

```swift
import Testing
@Test func expectationVersusRequirement() throws {
    let condition = Bool(false)
    #expect(condition)
    print("after expect")
    try #require(condition)
    print("after require")
}
```

**Outcome:** The runner records two issues, prints after expect, skips after require, and exits unsuccessfully.

**Why:** The expectation records failure and continues. The requirement throws, ending this test's ordinary execution. Preserve that failing process status when formatting its output.

## Focused exercise

**Task:** Test a debounce policy with an injected clock or controllable time dependency. Advance time, check emission and cancellation, and join the owned work.

**Acceptance checks:**

- Time 9 leaves the deadline-10 operation pending; time 10 releases its value.
- A cancelled operation produces no successful value after release.
- Every operation is joined and every registered wait is released without real sleeps.

**Solution:**

Swift Testing test file, with no app or network dependency. This integer-tick clock deliberately releases cancelled waiters only when their deadline is advanced; the policy checks cancellation before returning a value. A production clock must define prompt cancellation separately.

```swift
import Testing
actor ManualTime {
    private var now = 0
    private var waits: [(Int, CheckedContinuation<Void, Never>)] = []
    private var observers: [CheckedContinuation<Void, Never>] = []
    func sleep(until deadline: Int) async {
        if deadline <= now { return }
        await withCheckedContinuation { continuation in
            waits.append((deadline, continuation))
            let pending = observers
            observers.removeAll()
            for observer in pending { observer.resume() }
        }
    }
    func registered() async {
        if !waits.isEmpty { return }
        await withCheckedContinuation { observers.append($0) }
    }
    func advance(to instant: Int) {
        precondition(instant >= now)
        now = instant
        let ready = waits.filter { $0.0 <= now }
        waits.removeAll { $0.0 <= now }
        for (_, continuation) in ready { continuation.resume() }
    }
    func pending() -> Int { waits.count }
}
func debounced(_ value: String, until deadline: Int, time: ManualTime) async throws -> String {
    try Task.checkCancellation()
    await time.sleep(until: deadline)
    try Task.checkCancellation()
    return value
}
@Test func debounceBoundaryAndCancellation() async throws {
    let time = ManualTime()
    let first = Task { try await debounced("query", until: 10, time: time) }
    await time.registered()
    await time.advance(to: 9)
    let pending = await time.pending()
    #expect(pending == 1)
    await time.advance(to: 10)
    let emitted = try await first.value
    #expect(emitted == "query")

    let second = Task { try await debounced("cancelled", until: 20, time: time) }
    await time.registered()
    second.cancel()
    await time.advance(to: 20)
    do { _ = try await second.value; Issue.record("unexpected emission") }
    catch is CancellationError { }
    let remaining = await time.pending()
    #expect(remaining == 0)
}
```

**Reasoning:** The registration acknowledgment prevents advancing before the sleeper is installed. The test verifies this debounce timing/cancellation policy, not a full UI debounce owner or a production cancellation-aware Clock implementation. Replacement/publication tests and real integration coverage remain separate.

## In practice

**Choose it when:** Control the smallest boundary that exposes the defect, then add integration coverage for what remains external.

**Investigate by:** Inject time and services, gate arrival/completion, and assert cleanup after joining.

**Strong evidence:** Define what the test proves and demonstrate that its runner actually fails on a known defect.

### Framework interoperability and diagnostics — optional

#### Swift 6.4 interoperability is targeted

ST-0021 adds assertion/issue interoperability for gradual migration and defines `none`, `limited`, `complete`, and `strict` modes. Its formal scope is Corelibs XCTest; verify the proprietary XCTest shipped with Xcode separately. It does not make Swift Testing traits, XCTest UI/performance infrastructure, waits, or skip APIs universally interchangeable. [ST-0021 — Targeted interoperability](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0021-targeted-interoperability-swift-testing-and-xctest.md)

The following are **intentionally failing migration probes**, not tests to paste into the green suite. Run each direction independently:

```swift
import Testing
import XCTest

func legacyUniquenessCheck(_ values: [Int]) {
    XCTAssertEqual(Set(values).count, values.count)
}

@Test func bridgeXCTestFailure() {
    legacyUniquenessCheck([1, 2, 1])
}

final class BridgeTestingFailure: XCTestCase {
    func testModernIssue() {
        #expect(Bool(false), "intentional interop failure")
    }
}
```

Use `SWIFT_TESTING_XCTEST_INTEROP_MODE` to select the runtime mode. The direction matters:

| Mode | Failing XCTest assertion in a Swift Testing test | Failing Swift Testing expectation in an XCTest case |
| --- | --- | --- |
| `none` | May be lost; do not use as a migration correctness gate | May be lost |
| `limited` | Warning-level report; test may remain passing | Failure |
| `complete` | Failure plus migration warning | Failure |
| `strict` | Interoperability misuse can terminate the test process | Failure |

The local Xcode evidence is recorded alongside these modes; do not assume a successful XCTest assertion necessarily produces a warning or strict-mode termination. Validate actual failure paths. Prefer explicit configuration in a migration gate over guessing defaults from the compiler version. In the tested SwiftPM runner, a tools-version 6.3 manifest defaults to limited, while 6.4 defaults to complete, with the same 6.4 compiler. Tools version and `-swift-version 6` have different purposes.

Do not use XCTest expectations/waiters inside Swift Testing merely because assertion bridging works. `throw XCTSkip` is not a Swift Testing skip; use the appropriate traits or `try Test.cancel`. Interoperability is a bridge for selected APIs, not a reason to replace helpers without testing that deliberate failures reach CI.

#### catch-up: warning issues and test cancellation

Warning severity records a non-failing runtime issue. It is distinct from compiler warnings and does not establish that a violated requirement is acceptable. Test cancellation ends the current test/case by throwing and cancels its task hierarchy; it does not erase failures already recorded. Cancelling the test is different from testing cancellation of the system under test. [ST-0013 — Issue severity](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0013-issue-severity-warning.md), [ST-0016 — Test cancellation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0016-test-cancellation.md)

```swift
import Testing

@Test func recordsDiagnosticWarning() {
    Issue.record("Fixture used its documented fallback", severity: .warning)
    #expect(2 + 2 == 4)
}

@Test(arguments: [false, true])
func optionalFixture(_ available: Bool) throws {
    guard available else { try Test.cancel("Fixture unavailable for this case") }
    #expect(available)
}
```

Prefer a disabled/enabled trait for prerequisites known before execution. Parameterized cancellation affects the current case; do not catch its error and pretend the missing scenario was verified. Ordinary `Task {}` handles still need explicit cancellation/joining; test cancellation does not turn them into structured children.

#### diagnostics: custom test reflection

`CustomTestReflectable` supplies `customTestMirror` for test output, independently of ordinary `CustomReflectable`. `Mirror(reflectingForTest:)` uses it. Keep test-specific conformances/wrappers in a test target. A custom mirror changes presentation, not equality or production semantics. In the checked failure output, custom child fields appear while the parent value description still includes its original resultIDs storage. [ST-0022 — Custom reflection](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0022-customtestreflectable.md)

```swift
import Testing

struct SearchSnapshot: Equatable, CustomTestReflectable {
    let query: String
    let resultIDs: [Int]
    let generation: Int

    var customTestMirror: Mirror {
        Mirror(self, children: [
            "query": query,
            "resultCount": resultIDs.count,
            "generation": generation
        ], displayStyle: .struct)
    }
}

@Test func snapshotReflection() {
    let value = SearchSnapshot(query: "swift", resultIDs: [1, 2], generation: 3)
    let labels = Mirror(reflectingForTest: value).children.compactMap(\.label)
    #expect(labels == ["query", "resultCount", "generation"])
    #expect(Mirror(reflecting: value).children.compactMap(\.label)
        == ["query", "resultIDs", "generation"])
}
```

A mirror is not comprehensive redaction: descriptions, assertion operands, comments, attachments, and other diagnostics can still expose data. Use synthetic fixtures and an explicit log/attachment policy where needed. Do not hide information that reviewers need to understand a failure.

#### Attachments and repetition

Text/byte attachments provide reproducible diagnostics; image attachments were added in 6.3 through platform overlays. Swift 6.4 adds an async throwing `Attachment(exporting:as:named:)` initializer for `Transferable` values. Import the correctly spelled `CoreTransferable`. The tested overlay requires macOS 15.2 / iOS 18.2, not Apple OS 27. [ST-0023 — Transferable attachments](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0023-attachments-transferable.md)

```swift
import Foundation
import CoreTransferable
import UniformTypeIdentifiers
import Testing

struct SearchTrace: Transferable {
    let text: String

    static var transferRepresentation: some TransferRepresentation {
        DataRepresentation(exportedContentType: .plainText) { trace in
            Data(trace.text.utf8)
        }
    }
}

@available(macOS 15.2, iOS 18.2, *)
@Test func attachesTrace() async throws {
    let trace = SearchTrace(text: "request=2 state=loaded\n")
    let attachment = try await Attachment(exporting: trace, as: .plainText, named: "search.txt")
    Attachment.record(attachment)
}
```

Export can fail; decide whether that diagnostic failure should also fail the test or be handled separately from the original issue. An attachment call alone does not prove your runner retained a file. The SwiftPM runner accepts an existing writable directory through `--attachments-path`; inspect the produced artifact.

Swift 6.4 also exposes per-test-case repetition in `swift test`. With a stopping condition, eligible cases repeat independently; this does not rerun every case merely because one failed. A repetition trait illustrated as future work is not baseline API. [ST-0024 — Test-case repetition](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0024-per-test-case-repetitions.md)

```sh
swift test --filter NormalizerTests --maximum-repetitions 3
swift test --filter NormalizerTests --maximum-repetitions 10 --repeat-until fail
swift test --filter attachesTrace --attachments-path /path/to/existing/attachments
```

Repetition helps investigate flaky behavior. A later pass does not explain an earlier failure or prove the cause fixed. Record all attempts and the runner's final status. In the checked `--repeat-until pass` fixture, the eventual test summary says passed, but `swift test` still exits 1 after the earlier failing attempt. Preserve the process exit status; a formatter's success summary is insufficient. Keep independent setup/cleanup for every case and repeat; use a controlled interleaving to validate a concurrency fix.

## References

### Related guides

- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [Swift Testing](https://developer.apple.com/xcode/swift-testing/)
- [ST-0021 — Targeted interoperability](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0021-targeted-interoperability-swift-testing-and-xctest.md)
- [ST-0013 — Issue severity](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0013-issue-severity-warning.md)
- [ST-0016 — Test cancellation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0016-test-cancellation.md)
- [ST-0022 — Custom reflection](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0022-customtestreflectable.md)
- [ST-0023 — Transferable attachments](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0023-attachments-transferable.md)
- [ST-0024 — Test-case repetition](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0024-per-test-case-repetitions.md)
- [Swift 6.4 release](https://www.swift.org/blog/swift-6.4-released/)
- [Swift 6.3 release](https://www.swift.org/blog/swift-6.3-released/)
- [Go further with Swift Testing](https://developer.apple.com/videos/play/wwdc2024/10195/)
- [Asynchronous tests and expectations](https://developer.apple.com/documentation/xctest/asynchronous-tests-and-expectations)
