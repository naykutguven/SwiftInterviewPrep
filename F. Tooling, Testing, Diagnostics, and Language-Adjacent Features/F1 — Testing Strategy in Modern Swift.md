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

**Learning goals:**

- Choose assertions and test boundaries that explain what behavior was verified.
- Control registration, time, completion and cancellation separately, then join owned work.
- Verify runner failures, interoperability and retained diagnostic artifacts with real test executions.

**Prerequisites:** [[D1 — Structured Concurrency Fundamentals]]/[[D2 — Task Hierarchy, Cancellation, and Priorities]] for task ownership and cancellation, [[D9 — Continuations and Bridging Callback APIs]] for continuations, [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]] for synchronized state, and [[F5 — Language mode, build settings, and migration flags]] for the compiler/SDK distinction. The examples use a separate Swift Testing target, without an app or network dependency.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A debounce test passes after a sleep on your laptop and fails on CI. The sleep established elapsed wall time, but it did not establish that the worker registered its wait, consumed cancellation, or finished publication.

A useful test controls the events that determine behavior and asserts the public contract after the relevant work completes. Unit tests isolate policy; integration and UI tests cover real framework delivery, external I/O and view lifecycle. An acknowledgment, a virtual-time advance, cancellation and joining a task each prove something different. [Swift Testing](https://developer.apple.com/xcode/swift-testing/)

## Essential mechanics

### Create a real test target

Create `/tmp/TestingNotes/Package.swift` with the manifest below. Put each subsequent file under `Tests/GuideTests/` using its stated filename. There is one shared support file. Keep every file in this package: the deliberately failing suites are selected independently by filter and are excluded from the passing command. Every test example is compiled into the target; function declarations alone are not evidence that a test ran.

```swift
// swift-tools-version: 6.4
import PackageDescription

let package = Package(
    name: "TestingNotes",
    platforms: [.macOS(.v15)],
    targets: [
        .testTarget(
            name: "GuideTests",
            swiftSettings: [
                .swiftLanguageMode(.v6),
                .defaultIsolation(nil),
                .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ])
    ]
)
```

```sh
cd /tmp/TestingNotes
swift test --filter PassingTests
swift test --filter ExpectRequirementProbe   # Deliberately exits unsuccessfully.
```

Use the selected Xcode/Swift 6.4 toolchain. The host validation also passed `--disable-sandbox -Xswiftc -disable-sandbox` because its sandbox cannot launch compiler macro plugins otherwise; these are environment accommodations, not language semantics. Keep the actual `swift test` process status if formatting output. For example, Bash `set -o pipefail` prevents `swift test 2>&1 | xcsift -f toon` from hiding a test failure behind formatter success. Tests requiring newer APIs state their own deployment availability.

### Assertions describe outcomes; support types control events

`Support.swift` below supplies a suite and three small actors. Gate acknowledges an explicitly opened event. Completion records a separate terminal state. CallbackSource lets a test decide when a callback occurs; installation does not deliver it.

```swift
import Testing

@Suite struct PassingTests {}
actor Gate {
    private var opened = false
    private var waiters: [CheckedContinuation<Void, Never>] = []
    func wait() async {
        if opened { return }
        await withCheckedContinuation { waiters.append($0) }
    }
    func open() {
        opened = true
        let pending = waiters
        waiters.removeAll()
        for waiter in pending { waiter.resume() }
    }
}
actor Completion {
    private var finished = false
    func finish() { finished = true }
    func value() -> Bool { finished }
}
actor CallbackSource {
    private var callback: (@Sendable () -> Void)?
    func install(_ callback: @escaping @Sendable () -> Void) { self.callback = callback }
    func fire() { callback?() }
    func clear() { callback = nil }
}
```

**Walkthrough:** Each test creates fresh instances. Opening a gate resumes its waiters; it does not join those tasks or promise they have completed their next statement. The callback is Sendable because delivery may cross isolation boundaries.

`BasicTests.swift` gives the smallest outcome test. Awaiting the owned task is the completion boundary; an arbitrary delay would add no useful proof.

```swift
import Testing

extension PassingTests {
    @Test func joinsResult() async {
        let worker = Task { "loaded" }
        let result = await worker.value
        #expect(result == "loaded")
    }
}
```

**Walkthrough:** This checks a returned value and joins its worker. It says nothing about a database, timer or UI delivery path because none is part of this fixture.

### A controllable time dependency checks arithmetic

`ManualTime.swift` is shared by the clock question and exercise. It models integer ticks, not the standard Clock protocol. Registration acknowledgments prevent advancing before waits exist. Cancelled waiters remain pending until their deadline advances; the policy checks cancellation before emission. Production clocks need a separate prompt-cancellation policy.

```swift
actor ManualTime {
    private var now = 0
    private var waits: [(Int, CheckedContinuation<Void, Never>)] = []
    private var observers: [(Int, CheckedContinuation<Void, Never>)] = []
    func instant() -> Int { now }
    func sleep(until deadline: Int) async {
        if deadline <= now { return }
        await withCheckedContinuation { continuation in
            waits.append((deadline, continuation))
            let ready = observers.filter { $0.0 <= waits.count }
            observers.removeAll { $0.0 <= waits.count }
            for (_, observer) in ready { observer.resume() }
        }
    }
    func registered(_ count: Int = 1) async {
        if waits.count >= count { return }
        await withCheckedContinuation { observers.append((count, $0)) }
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
func debounced(_ value: String, delayTicks: Int, time: ManualTime) async throws -> String {
    precondition(delayTicks >= 0)
    try Task.checkCancellation()
    let deadline = await time.instant() + delayTicks
    await time.sleep(until: deadline)
    try Task.checkCancellation()
    return value
}
```

**Walkthrough:** The delay is added to the current virtual instant. Tests use small bounded integers, so overflow is outside this toy policy. A simple manually released gate has no clock arithmetic and cannot detect an off-by-one deadline.

## Rules and pitfalls

### A confirmation scope does not wait for future delivery

**Common pitfall:** Registering a callback in a confirmation body and returning before the callback fires.

**Mistake:** `ConfirmationProbe.swift`; intentionally failing suite. Run `swift test --filter ConfirmationProbe` separately.

```swift
import Testing

@Suite struct ConfirmationProbe {
    @Test func returnsBeforeDelivery() async {
        let source = CallbackSource()
        await confirmation("callback") { confirm in
            await source.install { confirm() }
        }
        await source.fire()
        await source.clear()
    }
}
```

**Consequence:** The scope finishes with zero confirmations; firing afterward cannot repair that failure.

**Why:** Confirmation validates calls during its body. It is not an implicit waiter for arbitrary future work.

**Correction:** `CallbackTests.swift` keeps delivery and the worker join inside the scope.

```swift
import Testing

extension PassingTests {
    @Test func confirmsOwnedDelivery() async {
        let source = CallbackSource()
        await confirmation("callback") { confirm in
            await source.install { confirm() }
            let worker = Task { await source.fire() }
            await worker.value
            await source.clear()
        }
    }
}
```

**What changes:** The expected callback happens before the confirmation body exits. A real callback API still needs an adapter with defined terminal behavior and cleanup; do not assume this fixture validates framework delivery.

### Resuming a continuation does not join the producer

**Common pitfall:** Treating an intermediate callback as proof that all work after it finished.

**Mistake:** `ResumeProbe.swift`; intentionally failing suite. The worker is held at a separate finish gate, so this failure is deterministic.

```swift
import Testing

@Suite struct ResumeProbe {
    @Test func resumeIsNotCompletion() async {
        let finish = Gate()
        let completion = Completion()
        var worker: Task<Void, Never>?
        await withCheckedContinuation { continuation in
            worker = Task {
                continuation.resume()
                await finish.wait()
                await completion.finish()
            }
        }
        let finishedTooEarly = await completion.value()
        #expect(finishedTooEarly)
        await finish.open()
        await worker?.value
    }
}
```

**Consequence:** The test resumed, but Completion is still false. Even the failing test releases and joins its worker.

**Why:** Resumption makes the awaiting task eligible to run. It does not complete the producer's later statements.

**Correction:** `CompletionTests.swift` acknowledges arrival, explicitly releases the last stage, then joins the owned task.

```swift
import Testing

extension PassingTests {
    @Test func joinsTerminalStage() async {
        let arrived = Gate()
        let finish = Gate()
        let completion = Completion()
        let worker = Task {
            await arrived.open()
            await finish.wait()
            await completion.finish()
        }
        await arrived.wait()
        #expect(await completion.value() == false)
        await finish.open()
        await worker.value
        #expect(await completion.value())
    }
}
```

**What changes:** Arrival and completion have distinct observable events. A test can now explain exactly which boundary an assertion follows.

### A wall-time delay cannot advance injected time

**Common pitfall:** Sleeping in the test while the implementation waits on a different time dependency.

**Mistake:** `TimingAssumption.swift` is a compiled helper, deliberately not an @Test. Do not use the delay as a scheduler assertion.

```swift
import Testing

func timingAssumption() async throws {
    let time = ManualTime()
    let worker = Task { try await debounced("query", delayTicks: 10, time: time) }
    await time.registered()
    try await Task.sleep(for: .milliseconds(10))
    #expect(await time.pending() == 0)  // Wrong: virtual time is still zero.
    await time.advance(to: 10)
    _ = try await worker.value
}
```

**Consequence:** Waiting ten real milliseconds leaves this virtual deadline pending; using guessed sleeps for real scheduling also depends on load.

**Why:** The injected clock and wall clock are different dependencies, and elapsed time alone does not acknowledge worker progress.

**Correction:** `ClockQuestionTests.swift` registers, advances virtual time to the boundary, and joins the result.

```swift
import Testing

extension PassingTests {
    @Test func advancesChosenClock() async throws {
        let time = ManualTime()
        let worker = Task { try await debounced("query", delayTicks: 10, time: time) }
        await time.registered()
        await time.advance(to: 9)
        #expect(await time.pending() == 1)
        await time.advance(to: 10)
        #expect(try await worker.value == "query")
        #expect(await time.pending() == 0)
    }
}
```

**What changes:** The test verifies the policy's deadline calculation and emission without wall-time sleeps. Real scheduling, app suspension and external timer cancellation remain integration concerns.

## Worked example

### A failed expectation continues; a failed requirement throws

**Setup:** `ExpectRequirementProbe.swift` is an intentionally failing suite selected independently with `swift test --filter ExpectRequirementProbe`.

**Predict:** Which print statements run, and how many issues does the runner record?

```swift
import Testing

@Suite struct ExpectRequirementProbe {
    @Test func expectationVersusRequirement() throws {
        let condition = Bool(false)
        #expect(condition)
        print("after expect")
        try #require(condition)
        print("after require")
    }

}
```

**Outcome:** Two issues are recorded, `after expect` prints, `after require` does not, and the test process exits unsuccessfully.

**Why:** #expect records failure and continues. #require throws, ending ordinary execution of that test. Preserve that process status even when a formatter reports a readable summary.

## Interview questions

### What does a manual gate prove, and what still needs a clock or integration test?

**Answer:** A gate proves behavior around an explicitly chosen event. It has no elapsed-time semantics, so it cannot verify deadline arithmetic, clock choice or actual framework scheduling.

**Example:** `GateQuestionTests.swift`; uses shared Gate and Completion.

```swift
import Testing

extension PassingTests {
    @Test func gateControlsRelease() async {
        let arrived = Gate()
        let release = Gate()
        let completion = Completion()
        let worker = Task {
            await arrived.open()
            await release.wait()
            await completion.finish()
        }
        await arrived.wait()
        #expect(await completion.value() == false)
        await release.open()
        await worker.value
        #expect(await completion.value())
    }
}
```

**Walkthrough:** Before release, the terminal action cannot have occurred. Joining establishes completion after release. No number here represents milliseconds.

**Boundary:** Add virtual time for duration policy and integration tests for real delivery, OS sleep policy or I/O cancellation.

### How do you verify a debounce deadline without sleeping?

**Answer:** Start from a known virtual instant, wait for registration, advance to just before the calculated deadline, then to the boundary and join. Include a nonzero starting instant so a hard-coded absolute deadline cannot accidentally pass.

**Example:** `DeadlineTests.swift`; uses shared ManualTime/debounced.

```swift
import Testing

extension PassingTests {
    @Test func deadlineIsRelativeToNow() async throws {
        let time = ManualTime()
        await time.advance(to: 100)
        let worker = Task { try await debounced("relative", delayTicks: 10, time: time) }
        await time.registered()
        await time.advance(to: 109)
        #expect(await time.pending() == 1)
        await time.advance(to: 110)
        #expect(try await worker.value == "relative")
        await time.advance(to: 111)
        #expect(await time.pending() == 0)
    }
}
```

**Walkthrough:** The deadline is 110, so 109 leaves one waiter and 110 releases it. Advancing beyond it leaves no duplicate wait or emission.

**Boundary:** This is a single-result delay policy, not a complete UI debounce owner. Replacement/publication ordering is covered by request-identity examples in [[F6 — Observation, property wrappers, and language-adjacent state tools|F6]]; production Clock conformance and prompt cancellation require separate design.

### Why must deliberate failures test both interop directions and preserve runner status?

**Answer:** XCTest and Swift Testing bridge selected assertions/issues according to mode and runner. A helper's assertion can be reported differently when called under the other framework, and a successful formatter can conceal a failing process.

**Example:** `InteropProbes.swift`; both suites intentionally contain a defect. Select each direction separately.

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

**Walkthrough:** The Swift Testing probe calls an XCTest assertion with duplicate values. The XCTest case records a failing Swift Testing expectation. The assertion contract is false in both directions; the runner must surface that fact for the migration gate to be useful.

```sh
SWIFT_TESTING_XCTEST_INTEROP_MODE=limited swift test --filter bridgeXCTestFailure
SWIFT_TESTING_XCTEST_INTEROP_MODE=complete swift test --filter bridgeXCTestFailure
SWIFT_TESTING_XCTEST_INTEROP_MODE=complete swift test --filter BridgeTestingFailure
```

**Boundary:** ST-0021 formally targets Corelibs XCTest; verify the XCTest shipped with Xcode independently. Modes are `none`, `limited`, `complete`, and `strict`. The checked Xcode runner preserves the failing Swift Testing-to-XCTest direction under complete; its limited mode can leave the opposite direction passing with a warning. Strict misuse can terminate the process. The recorded tools-version 6.3 default is limited and 6.4 default is complete on this runner, so language mode alone does not determine interop. Do not infer all XCTest expectations, waits, skip APIs, traits or UI/performance infrastructure interoperate. [ST-0021](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0021-targeted-interoperability-swift-testing-and-xctest.md)

### Extension: How do warning issues and Test.cancel affect evidence?

**Answer:** A warning adds non-failing diagnostic context. Test.cancel ends the current case by throwing and cancels its task hierarchy; it does not erase earlier failures or prove the system under test handles cancellation.

**Example:** `DiagnosticTests.swift`; warning and parameterized cancellation run in the passing suite.

```swift
import Testing

extension PassingTests {

    @Test func recordsDiagnosticWarning() {
        Issue.record("Fixture used its documented fallback", severity: .warning)
        #expect(2 + 2 == 4)
    }

    @Test(arguments: [false, true])
    func optionalFixture(_ available: Bool) throws {
        guard available else { try Test.cancel("Fixture unavailable for this case") }
        #expect(available)
    }

}
```

**Walkthrough:** The warning case still asserts its result. Of the two available values, false cancels that case and true reaches its assertion.

**Boundary:** Prefer an enabled/disabled trait for prerequisites known before execution. Do not catch cancellation and claim missing coverage. Unstructured Task handles still need explicit cancellation and joining. [Issue severity](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0013-issue-severity-warning.md), [test cancellation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0016-test-cancellation.md)

### Extension: What does CustomTestReflectable change?

**Answer:** It changes the fields exposed for test diagnostics, independently of ordinary reflection. It does not change equality or production semantics.

**Example:** `ReflectionTests.swift`; keep the test-specific conformance in the test target.

```swift
import Testing

struct SearchSnapshot: Equatable, CustomTestReflectable {
    let query: String
    let resultIDs: [Int]
    let generation: Int

    var customTestMirror: Mirror {
        Mirror(
            self,
            children: [
                "query": query,
                "resultCount": resultIDs.count,
                "generation": generation,
            ], displayStyle: .struct)
    }
}

extension PassingTests {
    @Test func snapshotReflection() {
        let value = SearchSnapshot(query: "swift", resultIDs: [1, 2], generation: 3)
        let labels = Mirror(reflectingForTest: value).children.compactMap(\.label)
        #expect(labels == ["query", "resultCount", "generation"])
        #expect(
            Mirror(reflecting: value).children.compactMap(\.label)
                == ["query", "resultIDs", "generation"])
    }

}
```

**Walkthrough:** The test mirror exposes resultCount; the ordinary mirror still exposes resultIDs. Both paths are asserted explicitly.

**Boundary:** A mirror is not comprehensive redaction: parent descriptions, operands, comments and attachments can still include storage. Use synthetic fixtures and decide what artifacts may contain. [Custom reflection](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0022-customtestreflectable.md)

### Extension: How do attachments become retained test evidence?

**Answer:** Create a diagnostic representation, record the attachment, configure a writable output directory, and inspect the produced file. A successful Attachment.record call alone does not verify retention.

**Example:** `AttachmentTests.swift`; the Transferable overlay needs macOS 15.2/iOS 18.2, independently of the Swift 6.4 compiler.

```swift
import CoreTransferable
import Foundation
import Testing
import UniformTypeIdentifiers

struct SearchTrace: Transferable {
    let text: String

    static var transferRepresentation: some TransferRepresentation {
        DataRepresentation(exportedContentType: .plainText) { trace in
            Data(trace.text.utf8)
        }
    }
}

extension PassingTests {
    @available(macOS 15.2, iOS 18.2, *)
    @Test func attachesTrace() async throws {
        let trace = SearchTrace(text: "request=2 state=loaded\n")
        let attachment = try await Attachment(exporting: trace, as: .plainText, named: "search.txt")
        Attachment.record(attachment)
    }

}
```

```sh
mkdir -p /tmp/testing-notes-attachments
swift test --filter attachesTrace --attachments-path /tmp/testing-notes-attachments
```

**Walkthrough:** The async exporter turns the trace into plain-text data, and the runner writes the named artifact. The checked file contains `request=2 state=loaded` followed by a newline.

**Boundary:** Export can throw; decide whether that diagnostic failure should fail the test or be reported separately from an original assertion. Do not assume every CI runner retains attachments without configuration. [Transferable attachments](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0023-attachments-transferable.md)

### Extension: Can a later passing repetition erase an earlier failed attempt?

**Answer:** Repetition helps reproduce behavior but does not explain or repair failure. Keep every attempt and the actual command status; eligible test cases repeat independently.

**Example:** `RepetitionProbe.swift`; an intentionally stateful diagnostic probe, excluded from PassingTests. Its first attempt fails and a later attempt passes.

```swift
import Synchronization
import Testing

private let attempts = Mutex(0)
@Suite struct RepetitionProbe {
    @Test func failsThenPasses() {
        let attempt = attempts.withLock { value in
            value += 1
            return value
        }
        #expect(attempt > 1)
    }
}
```

```sh
swift test --filter RepetitionProbe --maximum-repetitions 3 --repeat-until pass
```

**Walkthrough:** The fixture makes attempt ordering visible without a random failure. The checked runner eventually summarizes this test as passed but the swift test process still exits 1 after its first failed attempt.

**Boundary:** This shared counter is diagnostic machinery, not independent test setup to copy into a suite. Real parameterized/repeated cases need fresh setup and cleanup. A repetition trait shown as future work is not baseline API. [Per-case repetition](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0024-per-test-case-repetitions.md)

### Extension: Does serialized execution give unrelated tests a global lock?

**Answer:** No. Serialization applies to the annotated suite's members or a parameterized test's cases. Prefer independent fixtures; do not depend on unrelated suites running in a particular order.

**Example:** `IndependentCaseTests.swift`; each argument builds its own input/result without shared mutable state.

```swift
import Foundation
import Testing

extension PassingTests {
    @Test(.serialized, arguments: [" swift ", " actor "])
    func trimsIndependentInput(_ input: String) {
        let result = input.trimmingCharacters(in: .whitespaces)
        #expect(result == input.filter { $0 != " " })
    }
}
```

**Walkthrough:** The serialized trait applies to this test’s argument cases; unrelated tests can still overlap. Each case owns its values, so its result does not depend on another case’s setup. A truly shared external resource needs its own isolation/ownership boundary.

**Boundary:** A suite-local serialized trait is useful when required by a scenario but cannot coordinate unrelated suites, processes or external services.

## Focused exercise

**Task:** Verify before/at/after a virtual deadline, then cancel another operation and prove no successful emission. Join both tasks and release every waiter.

**Acceptance checks:**

- Time 9 leaves the deadline-10 operation pending; time 10 releases its value and 11 leaves no waiter.
- Cancellation prevents the second operation's successful value after its deadline is released.
- The final pending count is zero, with no real sleeps or abandoned tasks.

**Solution:** `ClockExerciseTests.swift`; uses ManualTime/debounced from the shared support.

```swift
import Testing

extension PassingTests {
    @Test func boundaryAndCancellation() async throws {
        let time = ManualTime()
        let first = Task { try await debounced("query", delayTicks: 10, time: time) }
        await time.registered()
        await time.advance(to: 9)
        #expect(await time.pending() == 1)
        await time.advance(to: 10)
        #expect(try await first.value == "query")
        await time.advance(to: 11)
        #expect(await time.pending() == 0)

        let second = Task { try await debounced("cancelled", delayTicks: 10, time: time) }
        await time.registered()
        second.cancel()
        await time.advance(to: 21)
        do {
            _ = try await second.value
            Issue.record("unexpected successful emission")
        } catch is CancellationError {}
        #expect(await time.pending() == 0)
    }
}
```

**Reasoning:** Registration acknowledges that the deadline exists. The clock verifies arithmetic; the task's cancellation check prevents publication; joining establishes completion. This deliberately simple dependency does not provide prompt cancellation until time advances, and it does not replace integration coverage of real timers or UI lifecycle.

## In practice

**Choose it when:** Test the smallest controllable boundary that exposes a defect, then cover external delivery through integration tests.

**Investigate by:** Separate arrival, deadline, cancellation, publication and task completion. Run known failures through the actual runner.

**Strong evidence:** Show assertions, released waiters, joined work, retained artifacts and original process status; explain what remains outside the fixture.

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
