---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - async-let
  - task-groups
  - structured-concurrency
---

# D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive

**Learning goals:**

- Choose sequential calls, fixed `async let` children, dynamic groups, or discarding groups from the dependencies and required results.
- Preserve input/result association when children complete in a different order.
- Explain how result consumption determines regular throwing-group error behavior, and how discarding groups differ.
- Bound active child work and describe cancellation and completion policy for a race.

**Prerequisites:** [[D1 — Structured Concurrency Fundamentals]] introduces task creation and joining; [[D2 — Task Hierarchy, Cancellation, and Priorities]] explains cooperative cancellation. `Sendable` means a value can safely cross isolation boundaries; [[D6 — `Sendable`, `@Sendable`, and `sending`]] gives the full model.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Start by drawing the dependencies. If loading an avatar requires an account ID returned by another request, load the ID first. If a dashboard needs three independent values, their requests can start together. If the number of requests comes from an input array, a task group lets the parent create that many children.

`async let` gives each fixed child a named role, such as `profile` or `notifications`. A group instead hands the parent results as children finish. That **completion order** can differ from the input order, so the result needs an index or ID when the caller cares which input it belongs to.

Every structured scope still joins its children before returning. Choosing the first result or encountering an error does not release the scope from that responsibility. Cancellation asks remaining children to finish early; it does not force them to disappear. [SE-0304 — structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)

A **failure policy** answers a separate question: does one failed request fail the whole batch, or does the caller receive an outcome for each request? Choose that policy explicitly before choosing how to consume group results.

## Essential mechanics

### Example setup

Run independent scripts using the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]], including top-level `await`. Prepend shared declarations only when named. **Compile-only** helpers use the same flags with `-typecheck`. Discarding groups require macOS 14/iOS 17 or later. The selected runtime is macOS 27.

**Shared setup — named requests:** Prepend this block to examples that name **named requests**. It is a small deterministic service, not a network latency simulation. The yield offers other work an opportunity to run; no claimed result order depends on it.

```swift
enum RequestFailure: Error { case unavailable(String) }

func request(_ name: String, failing: String? = nil) async throws -> String {
    try Task.checkCancellation()
    await Task.yield()
    if name == failing { throw RequestFailure.unavailable(name) }
    return "\(name):loaded"
}
```

### Sequential calls preserve a real dependency

**Setup:** Standalone runnable script.

```swift
func fetchAccountID() async -> Int {
    await Task.yield()
    return 42
}

func fetchAvatar(for accountID: Int) async -> String {
    await Task.yield()
    return "avatar for account \(accountID)"
}

let accountID = await fetchAccountID()
let avatar = await fetchAvatar(for: accountID)
print(avatar)
```

**Walkthrough:** The second request cannot be formed until the ID is available. The calls therefore belong in sequence, producing `avatar for account 42`. Trying to create both requests earlier would not remove the data dependency. Sequential asynchronous code still allows suspension without occupying a thread while waiting.

### `async let` fits fixed children, including different result types

**Setup:** Prepend **named requests**, then run this block.

```swift
func unreadCount() async -> Int { 3 }

async let profile = request("profile")
async let unread = unreadCount()
let loaded = try await (profile, unread)
print(loaded.0, loaded.1)
```

**Walkthrough:** The parent creates two children before reading either result. The bindings keep a `String` profile separate from an `Int` count. The tuple read waits for both and prints `profile:loaded 3`. There is no need to erase different result types into a common wrapper just to collect them. [SE-0317 — async let](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md)

### A task group fits an input-driven number of children

**Setup:** Standalone runnable nonthrowing example.

```swift
let names = ["profile", "notifications", "recommendations"]
let completions = await withTaskGroup(of: String.self) { group in
    for name in names {
        group.addTask {
            await Task.yield()
            return "\(name):loaded"
        }
    }
    var values: [String] = []
    for await value in group {
        values.append(value)
    }
    return values
}
precondition(Set(completions) == Set(names.map { "\($0):loaded" }))
```

**Walkthrough:** `addTask` creates a child for each input. `of: String.self` declares the common result type of those children. Iterating the group collects completed values. The set check verifies membership without pretending that completion order must match input order. For a large input, this unbounded teaching example needs the resource limit developed below.

### Throwing children need an explicit result-consumption policy

**Setup:** Prepend **named requests**. Runnable all-or-nothing example.

```swift
do {
    let values = try await withThrowingTaskGroup(of: String.self) { group in
        group.addTask { try await request("profile") }
        group.addTask { try await request("notifications", failing: "notifications") }
        var values: [String] = []
        for try await value in group {
            values.append(value)
        }
        return values
    }
    print(values)
} catch RequestFailure.unavailable(let name) {
    print("batch failed: \(name)")
}
```

**Walkthrough:** The loop observes child outcomes. When it reaches the failed notification result, the error escapes the group body. The group requests cancellation of remaining children, joins them, then throws to the caller. The output is `batch failed: notifications`. A child might already have succeeded, but no partial array is returned.

Merely using `withThrowingTaskGroup` does not choose this policy. The loop and the decision to let its error escape do. The interview comparison below demonstrates other policies. [Swift — TaskGroup implementation and contracts](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift)

### Discarding groups fit side effects whose returned values are unnecessary

**Setup:** Standalone runnable script. The actor protects a cache's warmed-key set.

```swift
actor Cache {
    private(set) var warmed: Set<String> = []
    func warm(_ key: String) async {
        await Task.yield()
        warmed.insert(key)
    }
}

let cache = Cache()
await withDiscardingTaskGroup { group in
    for key in ["profile", "notifications"] {
        group.addTask { await cache.warm(key) }
    }
}
let warmed = await cache.warmed
precondition(warmed == ["profile", "notifications"])
```

**Walkthrough:** The children warm the cache and return no useful values. The discarding group can release completed child results without requiring a result-draining loop. Its return still guarantees both children finished. A throwing discarding group additionally supplies automatic sibling cancellation on a child error, demonstrated in the interview questions. [SE-0381 — discarding groups](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0381-task-group-discard-results.md)

| Shape of work | Starting choice | Reason |
| --- | --- | --- |
| A later request needs an earlier result | Sequential `await` | Preserve the dependency. |
| Small fixed set, possibly different result types | `async let` | Named roles and simple result assembly. |
| Input-driven set or completion-driven decisions | Task group | Add children and consume individual outcomes. |
| Child values are unnecessary | Discarding group | Keep structured lifetime without retaining undrained results. |
| Work must outlive the operation | Explicitly owned unstructured task | Define a separate owner, error path, and completion policy; see D11. |

## Rules and pitfalls

### Immediately awaiting a child in each loop iteration serializes the loop

**Common pitfall:** An `async let` inside a loop is assumed to start the entire batch together.

**Mistake:** Prepend **named requests**. Runnable, but only one request is in flight per iteration.

```swift
var values: [String] = []
for name in ["profile", "notifications", "recommendations"] {
    async let value = request(name)
    values.append(try await value)
}
print(values)
```

**Consequence:** The next iteration does not start until the current child's value has been read. The loop's request work remains sequential.

**Why:** The child binding belongs to the iteration's scope. The explicit await already waits before advancing; even removing the read would leave scope-exit cancellation and joining before that iteration finishes.

**Correction:** For this small batch, use a group to create the independent children before collecting them. Prepend the same setup; run separately.

```swift
let values = try await withThrowingTaskGroup(of: String.self) { group in
    for name in ["profile", "notifications", "recommendations"] {
        group.addTask { try await request(name) }
    }
    var values: [String] = []
    for try await value in group { values.append(value) }
    return values
}
precondition(values.count == 3)
```

**What changes:** The additions are not interleaved with an immediate await of each child's result, so the children can overlap. Result order is now completion order; retain input association when required. The bounded helper below is the appropriate next step for larger inputs.

### Completion order is not input order

**Common pitfall:** Appending each finished result is expected to preserve the order of the original request array.

**Mistake:** Prepend **named requests**. Runnable; the printed array has no required order.

```swift
let names = ["profile", "notifications", "recommendations"]
let values = try await withThrowingTaskGroup(of: String.self) { group in
    for name in names { group.addTask { try await request(name) } }
    var values: [String] = []
    for try await value in group { values.append(value) }
    return values
}
print(values)
```

**Consequence:** The value at position zero can belong to notifications or recommendations. Pairing the returned array with the input by position can attach data to the wrong request.

**Why:** A group yields a child result when that child completes, not when its input position becomes next in a queue.

**Correction:** Return each input index with its result, and let only the parent write the output array. Prepend **named requests**; run separately.

```swift
let names = ["profile", "notifications", "recommendations"]
let values = try await withThrowingTaskGroup(of: (Int, String).self) { group in
    for (index, name) in names.enumerated() {
        group.addTask { (index, try await request(name)) }
    }
    var ordered = [String?](repeating: nil, count: names.count)
    for try await (index, value) in group { ordered[index] = value }
    return ordered.map { $0! }
}
precondition(values == names.map { "\($0):loaded" })
```

**What changes:** Each completion carries its destination. The parent restores input order independently of scheduling. The unwrap is justified because one child was added for every index and the loop completed without throwing. If failures were swallowed, missing positions would need explicit outcomes instead. The exercise forces out-of-order collection to verify this behavior.

### Bound active work when the input can be large

**Common pitfall:** Putting every input in a task group is expected to provide a resource limit automatically.

**Mistake:** Prepend **named requests**. **Compile-only** helper; do not use this as an unbounded network fan-out recipe.

```swift
func loadEveryRequest(_ names: [String]) async throws -> [String] {
    try await withThrowingTaskGroup(of: String.self) { group in
        for name in names { group.addTask { try await request(name) } }
        var values: [String] = []
        for try await value in group { values.append(value) }
        return values
    }
}
```

**Consequence:** A large array can create a large number of tasks and simultaneous external requests. Scheduling limits on CPU threads are not a limit on open connections, queued operations, or retained results.

**Why:** `addTask` creates a child; it does not wait for a resource slot. The parent needs an explicit policy for how many transforms may be active.

**Correction:** Seed a limited number, then add one replacement for each collected result. This standalone **compile-only** reusable helper is deliberately introduced after the simpler groups above. `Input` and `Output` are generic types; `Sendable` and the `@Sendable` transform require safe cross-isolation transfer.

```swift
enum FanOutError: Error { case invalidLimit }

func parallelMap<Input: Sendable, Output: Sendable>(
    _ inputs: [Input],
    maxConcurrent: Int,
    transform: @escaping @Sendable (Input) async throws -> Output
) async throws -> [Output] {
    guard maxConcurrent > 0 else { throw FanOutError.invalidLimit }
    try Task.checkCancellation()
    return try await withThrowingTaskGroup(of: (Int, Output).self) { group in
        var nextIndex = 0
        var results = [Output?](repeating: nil, count: inputs.count)

        while nextIndex < min(maxConcurrent, inputs.count) {
            let index = nextIndex
            let input = inputs[index]
            guard group.addTaskUnlessCancelled(operation: {
                try Task.checkCancellation()
                return (index, try await transform(input))
            }) else { throw CancellationError() }
            nextIndex += 1
        }

        while let (index, value) = try await group.next() {
            try Task.checkCancellation()
            results[index] = .some(value)
            if nextIndex < inputs.count {
                let index = nextIndex
                let input = inputs[index]
                guard group.addTaskUnlessCancelled(operation: {
                    try Task.checkCancellation()
                    return (index, try await transform(input))
                }) else { throw CancellationError() }
                nextIndex += 1
            }
        }
        try Task.checkCancellation()
        return results.map { $0! }
    }
}
```

**What changes:** The initial loop starts at most `maxConcurrent` transforms. Each collected completion makes room for one replacement. Only the parent writes the array, and indexes preserve association. `.some(value)` also preserves a legitimate nil output when `Output` itself is optional; the outer optional records whether that position was filled.

A positive limit with empty input returns an empty array unless cancellation is observed. A nonpositive limit throws. Errors escape the body, requesting sibling cancellation and joining. `addTaskUnlessCancelled` avoids adding to an already-cancelled group, but cancellation can race with addition; the child checks again before invoking the transform.

The bound applies to child transforms in this invocation, not all app traffic, threads, HTTP connections, or work a transform starts internally. Input/output storage is still O(n), and O(n) children are created over the operation's lifetime. An unbounded stream needs backpressure rather than an ever-growing result array; see D10.

### A regular throwing group does not automatically observe ignored child errors

**Common pitfall:** Choosing the throwing-group API is assumed to make any child failure fail the batch, even if its results are never read.

**Mistake:** Prepend **named requests**. Runnable counterexample.

```swift
await withThrowingTaskGroup(of: String.self) { group in
    group.addTask { try await request("profile", failing: "profile") }
}
print("batch reported success")
```

**Consequence:** The child fails, but the group body returns normally and the message is printed. The group joins the child; it does not deliver this ignored error to the caller.

**Why:** A regular throwing group's child error becomes observable through result consumption, such as `next()`, throwing iteration, or `waitForAll()`. Implicit joining at normal body return is not an all-or-nothing error policy.

**Correction:** Consume the outcomes and let failure escape. Prepend the same setup; run separately.

```swift
do {
    try await withThrowingTaskGroup(of: String.self) { group in
        group.addTask { try await request("profile", failing: "profile") }
        for try await _ in group { }
    }
    preconditionFailure("expected a failed batch")
} catch RequestFailure.unavailable(let name) {
    print("batch failed: \(name)")
}
```

**What changes:** The loop reads the failure and propagates it out of the body. The caller sees `batch failed: profile`. If partial results are intended, collect `Result` values or handle selected errors explicitly rather than silently reporting complete success.

## Worked example

### Assemble three required dashboard values

**Setup:** Prepend **named requests**. Standalone runnable all-or-nothing batch.

**Predict:** Can the caller receive a partly populated dashboard if notifications fail?

```swift
struct Dashboard: Sendable {
    let profile: String
    let notifications: String
    let recommendations: String
}

func dashboard(failing: String? = nil) async throws -> Dashboard {
    async let profile = request("profile", failing: failing)
    async let notifications = request("notifications", failing: failing)
    async let recommendations = request("recommendations", failing: failing)
    let loaded = try await (profile, notifications, recommendations)
    return Dashboard(profile: loaded.0, notifications: loaded.1,
                     recommendations: loaded.2)
}

let loaded = try await dashboard()
precondition(loaded.profile == "profile:loaded")
precondition(loaded.notifications == "notifications:loaded")
precondition(loaded.recommendations == "recommendations:loaded")
do {
    _ = try await dashboard(failing: "notifications")
    preconditionFailure("expected failure")
} catch RequestFailure.unavailable(let name) {
    precondition(name == "notifications")
}
```

**Outcome:** The successful call fills all three named fields. The failing call throws instead of returning a dashboard with invented defaults or omitted data.

**Why:** There are three known independent roles, so named bindings express the operation directly. The tuple read observes values in its evaluation order, not completion order. If an observed child throws, the scope cancels and joins any remaining unawaited children before the error reaches the caller.

A partial dashboard would need a different result model, such as `Result<String, Error>` for each section, and an explicit UI policy for failed sections. Replacing failed data with an empty success value would conceal the distinction.

## Interview questions

### When is `async let` better than a task group, and how can the order of awaiting its values delay error observation?

**Answer:** Prefer `async let` for a small fixed set of independent roles, especially when result types differ. Its reads follow the parent's evaluation order, so a failed child can remain unobserved while the parent awaits an earlier slow child.

**Example:** Standalone **compile-only** helper with injected services. The caller can hold the profile operation open while the message operation fails.

```swift
func loadPair(
    profile: @escaping @Sendable () async throws -> String,
    messages: @escaping @Sendable () async throws -> Int
) async throws -> (String, Int) {
    async let name = profile()
    async let count = messages()
    let loadedName = try await name
    let loadedCount = try await count
    return (loadedName, loadedCount)
}
```

**Walkthrough:** Both children start, but the parent first reads `name`. If `messages` fails while `profile` is still unfinished, that failure is not delivered to the parent until it reaches the `count` read. A task group can consume the failed completion earlier because it is not tied to these named reads.

**Boundary:** Earlier error observation still does not guarantee earlier return: a group also has to cancel and join the slow child. Use a group for dynamic child sets or completion-driven decisions, not merely to make a fixed pair look more sophisticated.

### Compare catching `next()`, using `nextResult()`, calling `waitForAll()`, ignoring child results, and a throwing discarding group.

**Answer:** These choices define different error policies. A regular group lets the body decide how to consume errors; a throwing discarding group automatically cancels siblings on a child failure and propagates the first failure through its scope.

#### Catch `next()` and keep draining

**Example:** Prepend **named requests**. Runnable partial-success policy.

```swift
let successes = await withThrowingTaskGroup(of: String.self) { group in
    group.addTask { try await request("profile") }
    group.addTask { try await request("notifications", failing: "notifications") }
    var successes: [String] = []
    while !group.isEmpty {
        do {
            if let value = try await group.next() { successes.append(value) }
        } catch {
            print("recorded a failed child")
        }
    }
    return successes
}
precondition(successes == ["profile:loaded"])
```

**Walkthrough:** Each `next()` consumes one child's completion, returning a value or throwing its error. Catching that error inside the body allows another `next()` call to consume the remaining completion. The surviving value is returned deliberately as partial success. Production code should retain failure details if callers need them.

#### Use `nextResult()` to keep both outcomes as data

**Example:** Prepend **named requests**. Standalone runnable example.

```swift
let outcomes = await withThrowingTaskGroup(of: String.self) { group in
    group.addTask { try await request("profile") }
    group.addTask { try await request("notifications", failing: "notifications") }
    var outcomes: [Result<String, Error>] = []
    while let outcome = await group.nextResult() { outcomes.append(outcome) }
    return outcomes
}
let successes = outcomes.compactMap { try? $0.get() }
precondition(outcomes.count == 2 && successes == ["profile:loaded"])
```

**Walkthrough:** `nextResult()` does not throw the child's error; it returns `.success` or `.failure`. The returned array retains both outcomes even though this small check also extracts the successful value. Add an input ID to each outcome if the caller needs association.

#### Use `waitForAll()` to drain before throwing

**Example:** Prepend **named requests**. Standalone runnable example.

```swift
actor Completion {
    private(set) var recorded = false
    func record() { recorded = true }
}

let completion = Completion()
do {
    try await withThrowingTaskGroup(of: String.self) { group in
        group.addTask { try await request("profile", failing: "profile") }
        group.addTask {
            await completion.record()
            return "notifications:loaded"
        }
        try await group.waitForAll()
    }
    preconditionFailure("expected failure")
} catch RequestFailure.unavailable {
    let recorded = await completion.recorded
    precondition(recorded)
}
```

**Walkthrough:** Under the current contract, `waitForAll()` keeps draining, remembers the first observed error, and throws it after all children finish. The completion record is therefore present when the caller catches the error. This does not mean that child failure immediately cancelled its sibling. Check the selected SDK contract rather than relying on historical implementations.

#### Ignore regular-group results

**Example:** Prepend **named requests**. Runnable counterexample to automatic error propagation.

```swift
await withThrowingTaskGroup(of: String.self) { group in
    group.addTask { try await request("profile", failing: "profile") }
}
print("normal body return ignored the child error")
```

**Walkthrough:** The body never reads a child result. Normal scope exit waits for the child but does not make this failed child throw to the caller. Omitting `try` here is intentional: the body itself does not throw.

#### Use a throwing discarding group for automatic failure propagation

**Example:** Prepend **named requests**. Runnable; the long sleep represents a cooperative sibling that should be cancelled, not a delay used to order a test.

```swift
do {
    try await withThrowingDiscardingTaskGroup { group in
        group.addTask {
            _ = try await request("profile", failing: "profile")
        }
        group.addTask {
            try await Task.sleep(for: .seconds(3600))
        }
    }
    preconditionFailure("expected failure")
} catch RequestFailure.unavailable(let name) {
    precondition(name == "profile")
}
```

**Walkthrough:** The failed child causes automatic sibling cancellation without a parent `next()` loop. The cancelled sleep throws cooperatively. The scope joins both children and then throws the first failure to its caller. An hour is not actually waited because the sibling responds to cancellation.

**Boundary:** Choose among these policies from the operation's contract. First observed failure need not mean first input, and cancellation still cannot force a noncooperating child to finish. For a discarding group, inspect errors inside a child only when intentionally changing the group's fail-on-error policy.

### Why can a first-result or timeout race remain suspended after choosing its winner?

**Answer:** Winner selection decides the result; it does not finish the remaining children. The owner must request cancellation, and the structured scope still waits for their completion, including slow cleanup.

**Shared setup — test gate:** Prepend this to the race and focused exercise. It suspends until opened and ignores cancellation so the driver can deliberately hold cleanup unfinished. It is a test helper, not a production cancellable wait.

```swift
actor Gate {
    private(set) var opened = false
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
```

**Example:** Prepend **test gate**. Standalone runnable controlled race.

```swift
let loserStarted = Gate(), winnerSelected = Gate()
let cleanupStarted = Gate(), releaseCleanup = Gate(), returned = Gate()
let owner = Task {
    let value = await withTaskGroup(of: String.self) { group in
        group.addTask {
            await loserStarted.open()
            while !Task.isCancelled { await Task.yield() }
            await cleanupStarted.open()
            await releaseCleanup.wait()
            return "loser"
        }
        group.addTask {
            await loserStarted.wait()
            return "winner"
        }
        let winner = await group.next()!
        group.cancelAll()
        await winnerSelected.open()
        return winner
    }
    await returned.open()
    return value
}
await winnerSelected.wait()
await cleanupStarted.wait()
let finishedEarly = await returned.opened
precondition(!finishedEarly)
await releaseCleanup.open()
let winner = await owner.value
let finished = await returned.opened
precondition(winner == "winner" && finished)
```

**Walkthrough:** The loser acknowledges entry, allowing the winner to complete. The parent collects that winner and cancels the loser. The loser observes cancellation but pauses its cleanup at a gate. The driver can prove the owner has not returned, then release cleanup and join it. No sleep establishes this event order.

**Boundary:** A timer child has the same structured-lifetime restriction. A first-completion race, first-success race, and race that ignores selected errors are different policies. Swift 6.4 awaited `defer` can also extend scope exit; cancellation shields can protect necessary cleanup without removing the join requirement. See D1 and D2 for those extensions.

## Focused exercise

**Task:** Run three independent requests in a group, preserve input order, and implement an all-or-nothing failure policy. Force out-of-order result collection and check that all children have finished on success and error.

**Acceptance checks:**

- The successful batch returns profile, notifications, and recommendations in input order even when the parent collects recommendations first.
- A selected notification failure throws instead of returning partial values.
- Every child records its cleanup before the caller observes completion, including on error.
- The driver releases every entered wait and joins its task without using elapsed-time sleeps to order events.

**Setup:** Prepend **test gate** and **named requests** for the `RequestFailure` declaration. `ControlledRequest` groups an input with its start, release, and collection acknowledgments. `BatchLog` records finished children and collection order through an actor. The async `defer` records cleanup on both ordinary return and throwing exit; see D1's Swift 6.4 extension.

**Solution:** Standalone runnable script with those two shared setups prepended.

```swift
struct ControlledRequest: Sendable {
    let name: String
    let started = Gate()
    let release = Gate()
    let collected = Gate()
}

actor BatchLog {
    private(set) var finished: Set<String> = []
    private(set) var collectionOrder: [String] = []
    func finish(_ name: String) { finished.insert(name) }
    func collect(_ name: String) { collectionOrder.append(name) }
}

func batch(_ requests: [ControlledRequest], failing: String?,
           log: BatchLog, failureObserved: Gate) async throws -> [String] {
    try await withThrowingTaskGroup(of: (Int, String).self) { group in
        for (index, item) in requests.enumerated() {
            group.addTask {
                defer { await log.finish(item.name) }
                await item.started.open()
                await item.release.wait()
                try Task.checkCancellation()
                if item.name == failing { throw RequestFailure.unavailable(item.name) }
                return (index, "\(item.name):loaded")
            }
        }
        var ordered = [String?](repeating: nil, count: requests.count)
        do {
            while let (index, value) = try await group.next() {
                ordered[index] = value
                await log.collect(requests[index].name)
                await requests[index].collected.open()
            }
        } catch {
            await failureObserved.open()
            throw error
        }
        return ordered.map { $0! }
    }
}

let failureCases: [String?] = [nil, "notifications"]
for failing in failureCases {
    let inputs = ["profile", "notifications", "recommendations"]
    let requests = inputs.map { ControlledRequest(name: $0) }
    let log = BatchLog(), failureObserved = Gate()
    let task = Task {
        try await batch(requests, failing: failing,
                        log: log, failureObserved: failureObserved)
    }
    for item in requests { await item.started.wait() }

    if failing == nil {
        for index in [2, 0, 1] {
            await requests[index].release.open()
            await requests[index].collected.wait()
        }
        let values = try await task.value
        let order = await log.collectionOrder
        precondition(values == inputs.map { "\($0):loaded" })
        precondition(order == ["recommendations", "profile", "notifications"])
    } else {
        await requests[1].release.open()
        await failureObserved.wait()
        // These gates ignore cancellation, so the driver must release the losers.
        await requests[0].release.open()
        await requests[2].release.open()
        do {
            _ = try await task.value
            preconditionFailure("expected failure")
        } catch RequestFailure.unavailable(let name) {
            precondition(name == "notifications")
        }
    }
    let finished = await log.finished
    precondition(finished == Set(inputs))
}
print("ordered success and joined all-or-nothing failure verified")
```

**Reasoning:** On success, the driver waits for the parent's collection acknowledgment before releasing the next child. This forces actual collection order, rather than guessing it from release times or print statements. The parent places values by input index, so the result remains ordered.

On failure, the driver lets notifications fail first. The body exposes that error and leaves the structured scope. Remaining children may observe cancellation, but the deliberately noncancellable gates still need releasing. After the driver releases them and joins `task.value`, all three cleanup records must be present. Only the selected failure escapes; no partially populated array is returned. The generic bounded helper has separate resource-bound checks in the validation record.

## In practice

**Choose it when:** Select the simplest primitive matching the dependency graph, result types, child count, and failure policy. A group is useful when the parent needs completion-driven decisions; named fixed bindings are often clearer for a dashboard.

**Investigate by:** Separate input order, completion order, error observation, cancellation request, and final joining. For a late race, find the child or deferred cleanup still preventing scope exit.

**Strong evidence:** Demonstrate the chosen all-or-nothing or per-item contract, force out-of-order completions, and verify the active-work limit under success, failure, and cancellation. State what the limit measures and what remains unbounded.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]] — Suspension, children, implicit joining, and awaited cleanup.
- [[D2 — Task Hierarchy, Cancellation, and Priorities]] — Cancellation, clocks, timeout ownership, and shields.
- [[D6 — `Sendable`, `@Sendable`, and `sending`]] — Safe transform captures and result transfer.
- [[D7 — Actor Reentrancy and Logic Races]] — Invariants around suspension and cleanup.
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]] — Streaming bounds instead of materialized batches.
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]] — Work with a separate lifetime owner.
- [[A6 — `defer`, Early Exits, and Cleanup Semantics]] — Synchronous cancellation requests and awaited teardown.
- [[F1 — Testing Strategy in Modern Swift]] — Acknowledgments, controlled completion, and injected time.

### Official sources

- [SE-0304 — Structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md) — Group and child lifetime, cancellation, and result consumption.
- [SE-0317 — async let](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md) — Fixed bindings, result reads, and implicit scope exit.
- [Swift — TaskGroup.swift](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift) — `next`, `nextResult`, `waitForAll`, completion order, and current group implementation.
- [Apple — TaskGroup](https://developer.apple.com/documentation/swift/taskgroup) — Group APIs and contracts.
- [SE-0381 — DiscardingTaskGroups](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0381-task-group-discard-results.md) — Result disposal and automatic throwing-group failure behavior.
- [SE-0493 — Async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md) — Awaited scope-exit cleanup.
- [SE-0504 — Task cancellation shields](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0504-task-cancellation-shields.md) — Shielding boundaries without abandoning structured lifetime.
