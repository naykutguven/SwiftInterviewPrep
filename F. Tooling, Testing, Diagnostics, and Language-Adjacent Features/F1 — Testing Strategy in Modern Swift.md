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
## 0. Rubric snapshot

**Rubric expectation**

Know XCTest and modern Swift Testing, async tests, actor-isolated tests, deterministic testing of concurrent code, and what to unit-test versus integration-test.

**Caveats**

Concurrency bugs often need design-for-testability, not just sleeps and retries. Proving a concurrent fix is correct usually requires testing cancellation, ordering, stale-result suppression, and isolation assumptions explicitly.

**You should be able to answer**

- How do you test async code without introducing flakiness?
- How would you prove a concurrency fix is correct rather than merely less flaky?
- What should an interview candidate do if a feature is hard to test because the design hides too much global state or concurrency?

**You should be able to do**

- Write a test strategy for a debounced search pipeline with cancellation and stale-result suppression.

This answer follows the uploaded section template structure.

---

## 1. Core mental model

Testing modern Swift is not mainly about choosing XCTest versus Swift Testing. The deeper skill is designing code so that important behavior is observable, controllable, and assertable.

For synchronous pure logic, tests can call a function and compare output. For async and concurrent code, the hard parts are different: time, task lifetime, cancellation, ordering, actor hops, stale results, and shared state. If those are hidden inside `Task {}`, `Task.sleep`, `DispatchQueue.main.asyncAfter`, singletons, or global mutable state, tests become timing guesses instead of correctness proofs.

Swift Testing gives modern syntax with `@Test`, `#expect`, parameterized tests, and good integration with Swift Concurrency. Apple describes Swift Testing as a framework for Swift packages and Xcode projects, and its `#expect` API captures expression values to explain failures. Swift Testing tests also integrate with Swift Concurrency and run in parallel by default. ([Apple Developer, "Swift Testing"](https://developer.apple.com/documentation/testing))

XCTest is still relevant. It remains widely used in existing Apple-platform projects, supports UI tests, performance tests, test plans, expectations, and legacy asynchronous styles. Apple’s XCTest guidance says to use expectations when there is no Swift async alternative, such as Objective-C callbacks, delegate methods, completion handlers, or Combine-style futures/promises. ([Apple Developer, "Asynchronous Tests and Expectations"](https://developer.apple.com/documentation/xctest/asynchronous-tests-and-expectations))

The senior/staff-level move is to separate **business semantics** from **execution mechanics**. For example, a debounced search feature should expose rules like “only the latest query can publish results” and “previous in-flight work is cancelled” independently from wall-clock time or UIKit lifecycle details.

The key idea:

```text
Good async tests do not wait longer; they control time, ordering, cancellation, and isolation.
```

Swift guarantees that `await` marks a suspension point and that actor isolation protects actor-isolated state from data races. Swift does **not** guarantee that tasks run in the order your mental model expects, that sleeping for 300 ms is enough in CI, or that actor isolation prevents stale-result logic bugs.

---

## 2. Essential mechanics

### 2.1 Swift Testing: modern default for Swift-first unit tests

Swift Testing uses ordinary Swift functions annotated with `@Test`. Assertions are written with `#expect` and can be used in async tests by marking the test function `async`. Apple’s async testing docs show the basic model: mark the test `async`, then `await` the asynchronous operation and assert normally. ([Apple Developer, "Testing asynchronous code"](https://developer.apple.com/documentation/testing/testing-asynchronous-code))

```swift
import Testing

struct SearchNormalizerTests {
    @Test
    func trimsWhitespace() {
        let normalized = SearchNormalizer.normalize("  swift  ")

        #expect(normalized == "swift")
    }

    @Test
    func emptyQueryIsIgnored() async {
        let pipeline = SearchPipeline(
            search: { _ in [] },
            debounce: .immediate
        )

        let result = await pipeline.submit("   ")

        #expect(result == .ignored)
    }
}
```

Use Swift Testing for new Swift-first unit and integration tests unless your project has a strong reason to stay on XCTest for a specific layer.

Good uses:

```text
Pure domain tests
Async/await unit tests
Parameterized input/output tests
Actor-isolated model tests
Package tests
Concurrency behavior tests
```

Be careful with:

```text
Shared mutable fixtures
Global singletons
Tests assuming serial execution
Tests depending on XCTest-only APIs
```

Swift Testing runs tests in parallel by default. Apple’s WWDC material says this improves speed, but also means shared mutable state in tests becomes more dangerous; `.serialized` exists, but Apple recommends refactoring tests to run safely in parallel where possible. ([Apple Developer, "Go further with Swift Testing"](https://developer.apple.com/videos/play/wwdc2024/10195/))

---

### 2.2 XCTest: still important for legacy, UI, performance, and expectation-based tests

XCTest remains valuable in existing codebases and for Apple-platform testing workflows. The important point is not “XCTest old, Swift Testing new.” The point is choosing the tool based on what you need to observe.

```swift
import XCTest

final class LegacyCallbackTests: XCTestCase {
    func testCallbackAPI() {
        let expectation = expectation(description: "callback called")

        LegacyService.fetch { value in
            XCTAssertEqual(value, 42)
            expectation.fulfill()
        }

        wait(for: [expectation], timeout: 1.0)
    }
}
```

This is acceptable for a legacy callback API. But for native async Swift code, prefer an async test:

```swift
func testAsyncAPI() async throws {
    let value = try await service.fetch()

    XCTAssertEqual(value, 42)
}
```

Bad XCTest async tests often mix `Task {}` and expectations unnecessarily:

```swift
func testBad() {
    let expectation = expectation(description: "done")

    Task {
        let value = await service.fetch()
        XCTAssertEqual(value, 42)
        expectation.fulfill()
    }

    wait(for: [expectation], timeout: 1.0)
}
```

Better:

```swift
func testBetter() async {
    let value = await service.fetch()

    XCTAssertEqual(value, 42)
}
```

The second version preserves structured control flow. The test itself awaits the result, so failure, cancellation, and thrown errors stay attached to the test.

---

### 2.3 Async tests should control time, not sleep through it

This is the core rule for debounce, retry, polling, timeout, and cancellation tests.

Bad:

```swift
@Test
func debouncesQuery_bad() async {
    let sut = SearchViewModel(searchClient: .stub)

    sut.queryChanged("swift")

    try? await Task.sleep(for: .milliseconds(350))

    #expect(sut.results == [.swift])
}
```

This test is flaky because it assumes:

```text
350 ms is enough on every machine
the task was scheduled promptly
the debounce task already started
the search completed
the MainActor update already happened
no unrelated test affected shared state
```

Better: inject a scheduler/clock-like dependency or model the debounce as an explicit dependency that the test can advance.

```swift
protocol Debouncer: Sendable {
    func wait() async throws
}

struct ImmediateDebouncer: Debouncer {
    func wait() async throws {}
}

struct SleepDebouncer: Debouncer {
    let duration: Duration

    func wait() async throws {
        try await Task.sleep(for: duration)
    }
}
```

In unit tests, use `ImmediateDebouncer` or a controllable test debouncer. In production, use `SleepDebouncer(duration: .milliseconds(300))`.

The goal is not to test Swift’s clock. The goal is to test your feature’s behavior when the debounce completes.

---

### 2.4 Actor-isolated tests should match the isolation of the system under test

If a type is `@MainActor`, test it from `@MainActor`.

```swift
import Testing

@MainActor
struct SearchViewModelTests {
    @Test
    func startsInIdleState() {
        let viewModel = SearchViewModel(search: { _ in [] })

        #expect(viewModel.state == .idle)
    }
}
```

This avoids fake “fixes” like unnecessary `await MainActor.run {}` blocks everywhere.

For actor types, test through the actor boundary:

```swift
actor SearchCache {
    private var values: [String: [SearchResult]] = [:]

    func store(_ results: [SearchResult], for query: String) {
        values[query] = results
    }

    func results(for query: String) -> [SearchResult]? {
        values[query]
    }
}

@Test
func cacheStoresResults() async {
    let cache = SearchCache()

    await cache.store([.swift], for: "swift")

    let results = await cache.results(for: "swift")
    #expect(results == [.swift])
}
```

Do not remove isolation to make tests easier. That is usually a design smell. Instead, expose behavior through a testable API.

---

### 2.5 Confirmations/expectations are for events, not for ordinary async results

For a single async result, prefer direct `await`.

```swift
@Test
func fetchesUser() async throws {
    let user = try await client.user(id: 1)

    #expect(user.id == 1)
}
```

For callback/event-style APIs, Swift Testing provides confirmations. Apple describes confirmations as a way to confirm that asynchronous events occur during a function invocation; by default a confirmation is expected once, and you can specify a different expected count. ([Apple Developer, "Expectations and confirmations"](https://developer.apple.com/documentation/testing/expectations))

Conceptually:

```swift
@Test
func emitsThreeEvents() async {
    await confirmation(expectedCount: 3) { event in
        emitter.onValue = {
            event()
        }

        await emitter.start()
    }
}
```

Use confirmations or XCTest expectations when the API is genuinely event/callback-shaped. Do not turn every async function into an expectation test.

---

## 3. Common traps and misconceptions

### Trap 1: “Add sleep until the test passes”

Bad:

```swift
@Test
func searchReturnsLatestResult_bad() async {
    viewModel.queryChanged("s")
    viewModel.queryChanged("sw")
    viewModel.queryChanged("swift")

    try? await Task.sleep(for: .seconds(1))

    #expect(viewModel.results == [.swift])
}
```

This does not prove correctness. It only proves that one run happened to settle within one second.

Better:

```swift
@Test
func searchReturnsLatestResult() async {
    let search = ControlledSearchClient()
    let viewModel = SearchViewModel(
        search: search.search,
        debouncer: ImmediateDebouncer()
    )

    await viewModel.queryChanged("slow")
    await viewModel.queryChanged("fast")

    await search.complete(query: "fast", with: [.fast])
    await search.complete(query: "slow", with: [.slow])

    #expect(viewModel.results == [.fast])
}
```

The better test controls completion order and proves stale-result suppression.

---

### Trap 2: Testing implementation details instead of behavior

Bad:

```swift
#expect(viewModel.currentTask != nil)
#expect(viewModel.debounceTask != nil)
```

This couples the test to internals. A refactor from task storage to `AsyncSequence` would break the test even if behavior stays correct.

Better:

```swift
#expect(search.startedQueries == ["swift"])
#expect(viewModel.state == .loaded([.swift]))
```

Assert externally meaningful behavior:

```text
Which query triggered search?
Which result became visible?
Was stale output ignored?
Was previous work cancelled?
Did loading/error state transition correctly?
```

---

### Trap 3: Hiding global state inside tests

Bad:

```swift
final class SearchService {
    static let shared = SearchService()

    var baseURL = URL(string: "https://production.example.com")!
}
```

Tests now depend on global mutable state and can interfere with each other, especially when Swift Testing runs tests in parallel by default. ([Apple Developer, "Go further with Swift Testing"](https://developer.apple.com/videos/play/wwdc2024/10195/))

Better:

```swift
struct SearchEnvironment: Sendable {
    var search: @Sendable (String) async throws -> [SearchResult]
    var debounce: any Debouncer
}
```

Pass dependencies in explicitly. This makes tests parallel-safe and behavior-specific.

---

### Trap 4: Assuming actor isolation proves feature correctness

Actor isolation prevents data races on actor-isolated state. It does not automatically prove business invariants.

Bad mental model:

```text
It is an actor, so the search result cannot be stale.
```

Correct mental model:

```text
The actor serializes isolated state access, but awaits can still allow interleaving.
I still need a latest-query token, cancellation, or sequence number to suppress stale results.
```

Example:

```swift
actor SearchModel {
    private var latestQuery = ""
    private var results: [SearchResult] = []

    func search(_ query: String, client: SearchClient) async {
        latestQuery = query
        let response = try? await client.search(query)
        results = response ?? []
    }
}
```

This can still publish stale results if an older request completes after a newer request. The actor does not know your semantic rule unless you encode it.

Better:

```swift
actor SearchModel {
    private var generation = 0
    private var results: [SearchResult] = []

    func search(_ query: String, client: SearchClient) async {
        generation += 1
        let currentGeneration = generation

        let response = try? await client.search(query)

        guard currentGeneration == generation else {
            return
        }

        results = response ?? []
    }
}
```

---

## 4. Direct answers to rubric questions

### Q1. How do you test async code without introducing flakiness?

Use structured async tests, inject controllable dependencies, avoid arbitrary sleeps, and make ordering/cancellation observable.

Concrete rules:

```text
Prefer async test functions over Task + expectation.
Inject clocks, schedulers, clients, stores, and ID generators.
Use immediate or controllable debounce in unit tests.
Control completion order with stubs.
Assert final behavior and important intermediate transitions.
Avoid shared mutable global state.
Use actor isolation in tests instead of bypassing isolation.
```

Interview version:

> I avoid testing async code by waiting for time to pass. I make async boundaries explicit: the network client, debounce delay, scheduler, clock, and storage are injected. For ordinary async functions, the test itself is async and awaits the operation. For multi-event callbacks, I use confirmations or expectations. For concurrency-sensitive behavior, I control completion order and cancellation explicitly so the test proves the behavior rather than relying on CI timing.

---

### Q2. How would you prove a concurrency fix is correct rather than merely less flaky?

Test the failure modes directly.

For a stale-result bug, do not just assert that the latest query usually wins. Force the older request to complete after the newer request and assert that the older result is ignored.

For cancellation, do not just check that UI looks right. Instrument the dependency and assert that the cancelled operation observes cancellation.

For actor reentrancy, do not just run the test many times. Create a controlled suspension point and resume operations in the order that used to break the invariant.

Good proof shape:

```text
1. Reproduce the dangerous interleaving deterministically.
2. Assert the invariant that must always hold.
3. Verify cancellation or stale suppression explicitly.
4. Run test without wall-clock sleeps.
5. Keep test isolated from global mutable state.
```

Interview version:

> I would first identify the specific bad interleaving or lifecycle issue the fix is supposed to prevent. Then I would write a deterministic test that forces that ordering: for example, complete request B before request A, then complete A and assert that A cannot overwrite B. If cancellation is part of the fix, I would use a stub that records cancellation. The test should fail on the old implementation and pass on the new one without arbitrary sleeps.

---

### Q3. What should an interview candidate do if a feature is hard to test because the design hides too much global state or concurrency?

Say that the design is the problem. Then propose a seam.

Do not say:

```text
I would add longer timeouts.
I would make the method public for tests.
I would use sleeps until it stabilizes.
I would mock the singleton globally.
```

Say:

```text
I would extract the global dependency behind an injected protocol or closure.
I would isolate mutable shared state behind an actor or value model.
I would inject time/scheduling instead of using Task.sleep directly.
I would move pure decision logic out of UI/lifecycle code.
I would expose behavior-level outputs, not internal task handles.
```

Interview version:

> If a feature is hard to test because it hides global state or starts unstructured work internally, I would treat that as a design smell. I’d introduce explicit dependencies for time, networking, storage, and execution boundaries, and move the business rule into a small unit that can be driven deterministically. I would not solve that by adding sleeps or making internals public just for tests.

---

## 5. Code probe

No rubric code probe is provided for F1. The relevant probe is a design/correctness probe: can you distinguish a flaky async test from a deterministic one?

### Minimal example

```swift
import Testing

struct SearchServiceTests {
    @Test
    func returnsResults() async throws {
        let service = SearchService(
            fetch: { query in
                [SearchResult(title: query)]
            }
        )

        let results = try await service.search("swift")

        #expect(results == [SearchResult(title: "swift")])
    }
}
```

### What happens?

```text
The test deterministically passes if SearchService forwards the query and returns the injected result.
No wall-clock timing is involved.
```

### Why?

The async dependency is controlled by the test. The test awaits the operation directly. There is no detached task, no sleep, and no hidden production network call.

```text
Test
 └─ awaits SearchService.search("swift")
     └─ calls injected fetch("swift")
         └─ returns controlled value
```

### Counterexample

```swift
@Test
func returnsResults_bad() async throws {
    let viewModel = SearchViewModel.live()

    viewModel.queryChanged("swift")

    try await Task.sleep(for: .milliseconds(500))

    #expect(viewModel.results.count > 0)
}
```

### What happens?

```text
This may pass locally and fail in CI.
It may fail under load.
It may pass even if stale-result suppression is broken.
It may hit real network/state unless SearchViewModel.live() is carefully isolated.
```

### Why?

The test does not control:

```text
Debounce time
Task scheduling
Network completion
MainActor update timing
Cancellation
Shared global state
```

It waits and hopes.

### Production example

```swift
struct SearchPipeline: Sendable {
    var debounce: any Debouncer
    var search: @Sendable (String) async throws -> [SearchResult]

    func results(for query: String) async throws -> [SearchResult] {
        let trimmed = query.trimmingCharacters(in: .whitespacesAndNewlines)

        guard !trimmed.isEmpty else {
            return []
        }

        try Task.checkCancellation()
        try await debounce.wait()
        try Task.checkCancellation()

        return try await search(trimmed)
    }
}
```

### Why this design is testable

```text
Debounce is injected.
Search is injected.
Cancellation is explicit.
Pure query normalization is local.
The test can use ImmediateDebouncer.
The test does not need to wait for real time.
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Direct async test|Single async result|Not enough for multi-event callback APIs|
|Swift Testing confirmation|Event/callback fires one or more times|Confirmation scope must cover the event timing|
|XCTest expectation|Legacy Objective-C, delegate, callback, Combine/Future code|Easier to misuse with timeouts|
|Injected debouncer/clock|Debounce, timeout, retry, polling|Requires designing dependencies explicitly|
|Controlled stub with manual completion|Ordering, stale result, cancellation tests|More test infrastructure|
|`.serialized` test suite|Temporary migration workaround for shared state|Can hide design problems and reduces parallelism|

---

## 6. Exercise

### Problem

Write a test strategy for a debounced search pipeline with cancellation and stale-result suppression.

We want to test this feature:

```text
When the user types:
1. Ignore empty/whitespace queries.
2. Debounce non-empty queries.
3. Start a search only after debounce.
4. Cancel previous pending/in-flight search when a newer query arrives.
5. If an older request completes after a newer request, ignore the older result.
6. Publish loading/results/error state on the correct isolation domain, usually MainActor for UI state.
```

---

### Bad / naive version

```swift
@MainActor
final class SearchViewModel {
    private let service = SearchService.shared
    private var task: Task<Void, Never>?

    var results: [SearchResult] = []

    func queryChanged(_ query: String) {
        task?.cancel()

        task = Task {
            try? await Task.sleep(for: .milliseconds(300))

            let results = try? await service.search(query)

            self.results = results ?? []
        }
    }
}
```

Naive test:

```swift
@Test
@MainActor
func debouncedSearch_bad() async {
    let viewModel = SearchViewModel()

    viewModel.queryChanged("s")
    viewModel.queryChanged("sw")
    viewModel.queryChanged("swift")

    try? await Task.sleep(for: .seconds(1))

    #expect(viewModel.results.contains { $0.title == "swift" })
}
```

### What is wrong?

```text
The service is global.
The debounce duration is real time.
The test does not prove only one search happened.
The test does not prove older work was cancelled.
The test does not force stale completion order.
The test may pass even if "s" or "sw" temporarily overwrites "swift".
The production code ignores CancellationError from sleep and may still continue.
The test is coupled to scheduler timing.
```

There is also a subtle production bug:

```swift
try? await Task.sleep(for: .milliseconds(300))
```

If the task is cancelled during sleep, `Task.sleep` throws. `try?` converts that into `nil`, then execution continues to the search. That means cancellation may not actually stop the operation.

Better:

```swift
try await Task.sleep(for: .milliseconds(300))
try Task.checkCancellation()
```

or handle `CancellationError` explicitly and return.

---

### Improved design

```swift
import Foundation

struct SearchResult: Equatable, Sendable {
    let title: String

    static let slow = SearchResult(title: "slow")
    static let fast = SearchResult(title: "fast")
    static let swift = SearchResult(title: "swift")
}

enum SearchState: Equatable, Sendable {
    case idle
    case loading(query: String)
    case loaded(query: String, results: [SearchResult])
    case failed(query: String, message: String)
}

protocol Debouncer: Sendable {
    func wait() async throws
}

struct ImmediateDebouncer: Debouncer {
    func wait() async throws {}
}

struct SleepDebouncer: Debouncer {
    let duration: Duration

    func wait() async throws {
        try await Task.sleep(for: duration)
    }
}

@MainActor
final class SearchViewModel {
    private let debouncer: any Debouncer
    private let search: @Sendable (String) async throws -> [SearchResult]

    private var task: Task<Void, Never>?
    private var generation = 0

    private(set) var state: SearchState = .idle

    init(
        debouncer: any Debouncer,
        search: @escaping @Sendable (String) async throws -> [SearchResult]
    ) {
        self.debouncer = debouncer
        self.search = search
    }

    func queryChanged(_ rawQuery: String) {
        let query = rawQuery.trimmingCharacters(in: .whitespacesAndNewlines)

        task?.cancel()
        generation += 1

        guard !query.isEmpty else {
            state = .idle
            return
        }

        let currentGeneration = generation
        state = .loading(query: query)

        task = Task { [debouncer, search] in
            do {
                try await debouncer.wait()
                try Task.checkCancellation()

                let results = try await search(query)
                try Task.checkCancellation()

                await MainActor.run {
                    guard currentGeneration == self.generation else {
                        return
                    }

                    self.state = .loaded(query: query, results: results)
                }
            } catch is CancellationError {
                // Cancellation is expected when a newer query replaces this one.
            } catch {
                await MainActor.run {
                    guard currentGeneration == self.generation else {
                        return
                    }

                    self.state = .failed(query: query, message: String(describing: error))
                }
            }
        }
    }
}
```

### Why this is better

```text
Debounce is injectable.
Search is injectable.
Cancellation is not swallowed accidentally.
The latest-generation check suppresses stale results.
UI state is MainActor-isolated.
Tests can drive behavior without real network or real time.
```

The `generation` token is important. Cancellation is necessary but not sufficient. Some dependencies may ignore cancellation or complete anyway. Stale-result suppression protects correctness even when cancellation is cooperative but not immediate.

---

### Controlled search test double

```swift
actor ControlledSearchClient {
    private struct Pending {
        let query: String
        let continuation: CheckedContinuation<[SearchResult], Error>
    }

    private var pending: [Pending] = []
    private(set) var startedQueries: [String] = []
    private(set) var cancelledQueries: [String] = []

    func search(_ query: String) async throws -> [SearchResult] {
        startedQueries.append(query)

        return try await withTaskCancellationHandler {
            try await withCheckedThrowingContinuation { continuation in
                pending.append(Pending(query: query, continuation: continuation))
            }
        } onCancel: {
            Task {
                await self.recordCancellation(query)
            }
        }
    }

    func complete(query: String, with results: [SearchResult]) {
        guard let index = pending.firstIndex(where: { $0.query == query }) else {
            return
        }

        let pending = pending.remove(at: index)
        pending.continuation.resume(returning: results)
    }

    private func recordCancellation(_ query: String) {
        cancelledQueries.append(query)
    }
}
```

This test double gives you control over completion order.

---

### Test 1: empty queries are ignored

```swift
import Testing

@MainActor
struct SearchViewModelTests {
    @Test
    func emptyQueryIsIgnored() async {
        let client = ControlledSearchClient()
        let viewModel = SearchViewModel(
            debouncer: ImmediateDebouncer(),
            search: client.search
        )

        viewModel.queryChanged("   ")

        #expect(viewModel.state == .idle)

        let started = await client.startedQueries
        #expect(started.isEmpty)
    }
}
```

What this proves:

```text
Whitespace does not trigger debounce/search.
The state model has an explicit idle state.
No network call is made.
```

---

### Test 2: latest query wins even if older request completes last

```swift
@MainActor
@Test
func latestQueryWinsWhenOlderRequestCompletesLast() async {
    let client = ControlledSearchClient()
    let viewModel = SearchViewModel(
        debouncer: ImmediateDebouncer(),
        search: client.search
    )

    viewModel.queryChanged("slow")
    viewModel.queryChanged("fast")

    await client.complete(query: "fast", with: [.fast])
    await client.complete(query: "slow", with: [.slow])

    #expect(viewModel.state == .loaded(query: "fast", results: [.fast]))
}
```

What this proves:

```text
The dangerous interleaving is forced.
The old request cannot overwrite the newer result.
The generation check works.
```

This is a real correctness proof. A sleep-based test would not prove this.

---

### Test 3: newer query cancels previous work

```swift
@MainActor
@Test
func newerQueryCancelsPreviousWork() async {
    let client = ControlledSearchClient()
    let viewModel = SearchViewModel(
        debouncer: ImmediateDebouncer(),
        search: client.search
    )

    viewModel.queryChanged("old")
    viewModel.queryChanged("new")

    // Give the cancellation handler a chance to record.
    await Task.yield()

    let cancelled = await client.cancelledQueries

    #expect(cancelled.contains("old"))
}
```

This test checks the cancellation behavior explicitly.

Caveat: cancellation is cooperative. A dependency may not stop instantly. That is why you should test both:

```text
Previous task observes cancellation.
Older completion cannot publish stale state.
```

---

### Test 4: loading state is isolated and visible

```swift
@MainActor
@Test
func nonEmptyQueryEntersLoadingState() async {
    let client = ControlledSearchClient()
    let viewModel = SearchViewModel(
        debouncer: ImmediateDebouncer(),
        search: client.search
    )

    viewModel.queryChanged("swift")

    #expect(viewModel.state == .loading(query: "swift"))
}
```

This proves UI-visible state transitions without leaving the `MainActor`.

---

### Test 5: error from latest query is published

```swift
struct SearchFailure: Error, Sendable {}

actor FailingSearchClient {
    func search(_ query: String) async throws -> [SearchResult] {
        throw SearchFailure()
    }
}

@MainActor
@Test
func latestErrorIsPublished() async {
    let client = FailingSearchClient()
    let viewModel = SearchViewModel(
        debouncer: ImmediateDebouncer(),
        search: client.search
    )

    viewModel.queryChanged("swift")

    await Task.yield()

    if case .failed(query: "swift", _) = viewModel.state {
        #expect(true)
    } else {
        #expect(Bool(false), "Expected failed state")
    }
}
```

This test is less ideal because `Task.yield()` still depends on scheduler progress. A stronger design would expose the pipeline as an async function or an `AsyncSequence<SearchState>` so the test can await the next emitted state directly.

---

### Stronger staff-level design: test the pipeline separately from the UI adapter

Split the feature into:

```text
SearchPipeline
- normalizes query
- debounces
- calls search
- handles cancellation
- suppresses stale results

SearchViewModel
- observes pipeline output
- publishes MainActor UI state
```

Then most tests target `SearchPipeline`, not the view model.

```swift
struct SearchPipeline {
    let debouncer: any Debouncer
    let search: @Sendable (String) async throws -> [SearchResult]

    func searchResults(for query: String) async throws -> [SearchResult] {
        let query = query.trimmingCharacters(in: .whitespacesAndNewlines)

        guard !query.isEmpty else {
            return []
        }

        try await debouncer.wait()
        try Task.checkCancellation()

        return try await search(query)
    }
}
```

The view model becomes thin:

```swift
@MainActor
final class SearchViewModel {
    private let pipeline: SearchPipeline
    private var task: Task<Void, Never>?
    private var generation = 0

    private(set) var state: SearchState = .idle

    init(pipeline: SearchPipeline) {
        self.pipeline = pipeline
    }

    func queryChanged(_ query: String) {
        task?.cancel()
        generation += 1

        let currentGeneration = generation

        task = Task {
            do {
                let results = try await pipeline.searchResults(for: query)

                await MainActor.run {
                    guard currentGeneration == self.generation else { return }
                    self.state = .loaded(query: query, results: results)
                }
            } catch is CancellationError {
            } catch {
                await MainActor.run {
                    guard currentGeneration == self.generation else { return }
                    self.state = .failed(query: query, message: String(describing: error))
                }
            }
        }
    }
}
```

This makes test strategy cleaner:

```text
Unit-test SearchPipeline heavily.
Unit-test ViewModel state transitions lightly.
Integration-test real debounce/search wiring once.
UI-test only the user-visible search behavior.
```

---

## 7. Production guidance

Use this in production when:

```text
Testing async services, repositories, view models, actors, and state machines.
Testing cancellation-sensitive user flows such as search, image loading, checkout, login, token refresh, and syncing.
Migrating XCTest-heavy suites toward Swift Testing.
Creating package-level tests that should run fast and in parallel.
Validating Swift 6 concurrency migrations.
```

Be careful when:

```text
Tests use shared fixtures, singletons, static mutable state, or real clocks.
Tests assume serial execution.
A test starts Task {} internally and does not await its result.
The production code uses try? around cancellation-aware APIs.
The feature has hidden DispatchQueue, Timer, RunLoop, or Task.sleep behavior.
The code under test is @MainActor but the test fights actor isolation.
```

Avoid when:

```text
A unit test hits real network, real database, production analytics, or global mutable state.
A test uses arbitrary sleeps to wait for async behavior.
A test only verifies "eventually something happened" but not ordering/cancellation invariants.
A test reaches into private implementation details instead of behavior.
A test disables concurrency/parallelism instead of fixing shared state.
```

Debugging checklist:

```text
Can the test control time?
Can the test control completion order?
Can the test observe cancellation?
Can the test force the old bad interleaving?
Can an older result overwrite a newer one?
Is UI state isolated to MainActor?
Does the test use Task.sleep as a substitute for determinism?
Does production code swallow CancellationError?
Does the code start unstructured work that the test cannot await?
Does the test rely on global shared state?
Would this test still pass if run in parallel with the rest of the suite?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> I test async code by using XCTest expectations or marking the test async and waiting for the result.

This is incomplete. It does not address flakiness, time control, cancellation, stale results, or design seams.

### Senior answer

> I prefer async tests that await the system under test directly. For debounced or delayed behavior, I inject a clock or debouncer instead of using real sleeps. For callback APIs, I use XCTest expectations or Swift Testing confirmations. I avoid global state and make dependencies injectable so tests are deterministic and parallel-safe.

This is solid senior-level.

### Staff-level answer

> I design the feature so the concurrency policy is testable: time, search execution, cancellation, and output publication are explicit boundaries. For a debounced search pipeline, I would unit-test normalization, debounce triggering, cancellation observation, latest-query-wins behavior, stale-result suppression, and MainActor state publication separately. I would force the dangerous interleavings deterministically using controlled stubs. If the feature is hard to test, I would treat that as feedback that the design hides too much execution policy, not as a reason to add sleeps or make internals public.

That is the signal you want.

Staff-level questions to ask:

```text
What is the invariant this concurrent code must preserve?
Can I force the interleaving that used to break it?
Is cancellation relied on for correctness, or only for efficiency?
What happens if a dependency ignores cancellation?
Which layer owns time/debounce policy?
Which layer owns UI isolation?
Can this test run safely in parallel?
Does this test fail on the old implementation?
Can this behavior be tested below the UI layer?
```

---

## 9. Interview-ready summary

Modern Swift testing is about deterministic control over behavior, not just using XCTest or Swift Testing syntax. For async code, I prefer async test functions that directly `await` the system under test. For callbacks or repeated events, I use expectations or Swift Testing confirmations. For debounce, retry, cancellation, and stale-result problems, I avoid real sleeps and inject time or scheduling dependencies. To prove a concurrency fix, I force the dangerous ordering explicitly, such as completing an older request after a newer one, and assert that stale output cannot be published. If a feature is hard to test because it hides global state or unstructured tasks, I treat that as a design problem and introduce seams for dependencies, isolation, and time.

---

## 10. Flashcards

Q: What is the biggest source of flakiness in async Swift tests?

A: Hidden timing and scheduling assumptions, especially `Task.sleep`, `DispatchQueue.asyncAfter`, real network calls, and unawaited unstructured tasks.

Q: When should you use an async test instead of an XCTest expectation?

A: Use an async test when the API already returns a single async result. Use expectations/confirmations for callback or event-style APIs that cannot be awaited directly.

Q: Why is `try? await Task.sleep(...)` dangerous in cancellable code?

A: Cancellation causes `Task.sleep` to throw. `try?` swallows that error and execution may continue, so cancelled work may still run.

Q: What does stale-result suppression mean?

A: Older async work must not be allowed to publish results after a newer query/request has become current.

Q: Why is cancellation not enough to prevent stale results?

A: Cancellation is cooperative. A dependency may ignore cancellation or complete anyway, so the publishing layer should also check a generation/token/latest-query invariant.

Q: How do you prove a “latest query wins” fix?

A: Start an old query, start a newer query, complete the newer query first, then complete the old query, and assert that only the newer result is visible.

Q: Why can actor-isolated code still need stale-result tests?

A: Actors prevent data races on isolated state, but `await` can allow interleaving. Business invariants like “latest result wins” must still be encoded and tested.

Q: Why can Swift Testing expose bad tests after migration?

A: Swift Testing runs tests in parallel by default, so tests that depend on shared mutable state or serial execution may become unreliable. ([Apple Developer, "Go further with Swift Testing"](https://developer.apple.com/videos/play/wwdc2024/10195/))

Q: What is the right response when a feature is hard to test?

A: Identify the hidden dependency or concurrency boundary and introduce a seam: injected client, clock/debouncer, actor, state machine, or async sequence.

Q: What should a debounced search test strategy cover?

A: Empty-query suppression, debounce behavior, search triggering, cancellation, stale-result suppression, loading/error/result state, and actor isolation.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D4 — Actors and actor isolation]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D13 — Swift 6 strict concurrency migration]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F3 — Performance investigation habits]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — F1 testing strategy section.
- Apple Developer. "Swift Testing." Apple Developer Documentation. https://developer.apple.com/documentation/testing
- Apple Developer. "Swift Testing." Xcode. https://developer.apple.com/xcode/swift-testing/
- Apple Developer. "Testing asynchronous code." Apple Developer Documentation. https://developer.apple.com/documentation/testing/testing-asynchronous-code
- Apple Developer. "Asynchronous Tests and Expectations." Apple Developer Documentation. https://developer.apple.com/documentation/xctest/asynchronous-tests-and-expectations
- Apple Developer. "Expectations and confirmations." Apple Developer Documentation. https://developer.apple.com/documentation/testing/expectations
- Apple Developer. "Go further with Swift Testing." WWDC24. https://developer.apple.com/videos/play/wwdc2024/10195/
