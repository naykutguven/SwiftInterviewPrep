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
## 0. Rubric snapshot

**Rubric expectation**

Understand when to use `async let`, throwing task groups, discarding task groups, and plain sequential `await`.

**Caveats**

Not all async-looking work should be parallelized. Different primitives have different overhead, cancellation behavior, result-collection behavior, and failure semantics. The rubric specifically asks you to explain when `async let` is better than a task group, what happens when a throwing task group child fails, and how to parallelize three independent network calls.

**You should be able to answer**

- When is `async let` better than a task group?
- What are the failure and cancellation semantics of a throwing task group?
- When is plain sequential `await` actually the right design?

**You should be able to do**

- Parallelize three independent async operations.
- Choose between `async let`, `withTaskGroup`, `withThrowingTaskGroup`, `withDiscardingTaskGroup`, and sequential code.
- Explain how one failure should affect sibling work.

---

## 1. Core mental model

Swift gives you several structured concurrency primitives, but they are not interchangeable syntax variants. They model different **shapes of child work**.

The simplest case is sequential `await`: start operation A, wait for it, then use its result to decide or perform operation B. This is correct when B depends on A, when ordering matters, or when parallelism would waste resources.

`async let` creates a fixed number of child tasks whose results are needed later in the same lexical scope. The child task starts when the `async let` declaration is reached, and reading the value later is an `await` suspension point. SE-0317 describes `async let` as local constants whose initializer expression runs in a separate concurrently executing child task; the child starts as soon as the declaration is encountered. ([GitHub, "async let"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md))

Task groups are for a dynamic number of child tasks, completion-order result collection, races, fan-out/fan-in, and custom cancellation behavior. SE-0317 explicitly calls out that dynamic-size parallel map and “whichever completes first” APIs require a task group rather than `async let`. ([GitHub, "async let"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md))

The key idea:

```text
sequential await = ordered dependency
async let        = fixed small fan-out
task group       = dynamic fan-out / completion-order collection / custom cancellation
discarding group = dynamic fan-out where child results are irrelevant
```

Swift’s structured concurrency guarantee matters here: child tasks do not outlive the scope that created them. Task groups and `async let` both wait for their child tasks before the scope returns. ([GitHub, "TaskGroup.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift))

---

## 2. Essential mechanics

### 2.1 Plain sequential `await`

Use sequential `await` when operation B depends on operation A.

```swift
func loadUserProfile() async throws -> ProfileViewModel {
    let user = try await api.fetchUser()
    let avatar = try await api.fetchAvatar(for: user.id)
    return ProfileViewModel(user: user, avatar: avatar)
}
```

This is not “less advanced.” It is the correct model when dependency order is real.

Bad parallelization here would either duplicate work, require awkward optional state, or introduce cancellation behavior that does not match the domain.

---

### 2.2 `async let`

Use `async let` when you have a small, fixed number of independent operations.

```swift
func loadHomeData() async throws -> HomeData {
    async let profile = api.fetchProfile()
    async let recommendations = api.fetchRecommendations()
    async let unreadCount = api.fetchUnreadCount()

    return try await HomeData(
        profile: profile,
        recommendations: recommendations,
        unreadCount: unreadCount
    )
}
```

`async let` is ideal here because:

- the number of operations is known at compile time,
- the results can have different types,
- the parent needs all results before returning,
- the child tasks are scoped to this function.

You do not write `try await` on the right-hand side for single-expression async-let initializers; the `try` / `await` effects are enforced when reading the async-let value. SE-0317 documents this behavior. ([GitHub, "async let"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md))

---

### 2.3 `withTaskGroup`

Use a non-throwing task group for dynamic child work where child tasks cannot throw.

```swift
func loadThumbnails(ids: [ImageID]) async -> [ImageID: Image] {
    await withTaskGroup(of: (ImageID, Image?).self) { group in
        for id in ids {
            group.addTask {
                let image = await imageCache.thumbnail(for: id)
                return (id, image)
            }
        }

        var result: [ImageID: Image] = [:]

        for await (id, image) in group {
            if let image {
                result[id] = image
            }
        }

        return result
    }
}
```

Task-group results are produced in **completion order**, not in the order tasks were added. The standard library documentation for `TaskGroup.next()` states that successive values appear in completion order. ([GitHub, "TaskGroup.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift))

---

### 2.4 `withThrowingTaskGroup`

Use a throwing task group when each child can fail and the parent needs to decide how failures affect siblings.

```swift
func fetchAll(ids: [Int]) async throws -> [Int] {
    try await withThrowingTaskGroup(of: Int.self) { group in
        for id in ids {
            group.addTask {
                try await fetch(id)
            }
        }

        var values: [Int] = []

        while let value = try await group.next() {
            values.append(value)
        }

        return values
    }
}
```

Important nuance: a throwing task group is not automatically “fail fast” merely because one child throws. The error is observed when you call `try await group.next()`, iterate with `for try await`, or wait for all. If that error is propagated out of the group body, remaining child tasks are implicitly canceled. ([GitHub, "TaskGroup.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift))

The group still waits for child tasks to finish; cancellation is cooperative, not preemptive. The standard library documentation states that task groups always wait for child tasks before returning and that `cancelAll()` only signals cancellation; children must cooperate and return early. ([GitHub, "TaskGroup.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift))

---

### 2.5 Discarding task groups

Use a discarding task group when you want structured dynamic child work but do not care about each child’s returned value.

```swift
func warmCaches(keys: [CacheKey]) async {
    await withDiscardingTaskGroup { group in
        for key in keys {
            group.addTask {
                await cache.preload(key)
            }
        }
    }
}
```

Use `withThrowingDiscardingTaskGroup` when the children do not produce useful results but can fail. Apple’s documentation describes throwing discarding task groups as canceling themselves when a child throws, with the first error propagated. ([Apple Developer, "withThrowingDiscardingTaskGroup(returning:isolation:body:)"](https://developer.apple.com/documentation/swift/withthrowingdiscardingtaskgroup%28returning%3aisolation%3abody%3a%29))

This is a good fit for “do N independent side-effecting jobs, wait until all complete, fail if any fail.”

---

## 3. Common traps and misconceptions

### Trap 1: `async let` in a loop does not automatically make the whole loop parallel

Bad:

```swift
for id in ids {
    async let image = fetchImage(id)
    images.append(await image)
}
```

This starts one child task and immediately waits for it before the next iteration. That is effectively sequential.

Better:

```swift
let images = await withTaskGroup(of: Image.self) { group in
    for id in ids {
        group.addTask {
            await fetchImage(id)
        }
    }

    var images: [Image] = []
    for await image in group {
        images.append(image)
    }
    return images
}
```

Use a task group for dynamic fan-out.

---

### Trap 2: Treating task groups as fire-and-forget

This is wrong:

```swift
await withTaskGroup(of: Void.self) { group in
    for event in events {
        group.addTask {
            await analytics.send(event)
        }
    }

    return
}
```

It may look like it returns immediately, but it does not. Structured task groups wait for all child tasks before returning. ([GitHub, "TaskGroup.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift))

If you truly need work to outlive the current scope, that is unstructured concurrency territory, but that should be a deliberate lifecycle decision, not a casual escape hatch.

---

### Trap 3: Assuming cancellation stops work instantly

Cancellation is a signal. It does not kill a child task.

Bad:

```swift
group.cancelAll()
// Assuming all children are now stopped immediately.
```

Better:

```swift
group.addTask {
    try Task.checkCancellation()
    let data = try await api.fetchData()
    try Task.checkCancellation()
    return data
}
```

Children must check cancellation or call cancellation-aware APIs.

---

### Trap 4: Parallelizing work that should stay sequential

Bad:

```swift
async let token = auth.refreshToken()
async let profile = api.fetchProfile(using: token)
```

`profile` depends on `token`; this does not model the dependency correctly.

Better:

```swift
let token = try await auth.refreshToken()
let profile = try await api.fetchProfile(using: token)
```

Parallelism is not a goal by itself. Correct dependency modeling comes first.

---

## 4. Direct answers to rubric questions

### Q1. When is `async let` better than a task group?

`async let` is better when you have a fixed, small number of independent child operations, especially when they return different types.

Example:

```swift
async let profile = fetchProfile()
async let settings = fetchSettings()
async let flags = fetchFeatureFlags()

let model = try await HomeModel(
    profile: profile,
    settings: settings,
    flags: flags
)
```

A task group would be more verbose here, especially if the results have heterogeneous types. SE-0317 calls this exact kind of heterogeneous result initialization pattern a weakness of task groups and the motivation for `async let`. ([GitHub, "async let"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md))

Interview version:

> I use `async let` for fixed, lexical fan-out: a few independent operations that I know statically and whose results I need before leaving the scope. It is cleaner than a task group for heterogeneous results. I switch to a task group when the number of tasks is dynamic, when I need completion-order processing, bounded fan-out, racing, or custom cancellation behavior.

---

### Q2. What are the failure and cancellation semantics of a throwing task group?

A throwing task group lets child tasks throw, but errors are observed through `try await group.next()`, `for try await`, or `waitForAll()`. If a child error is propagated out of the group body, the remaining child tasks are implicitly canceled. ([GitHub, "TaskGroup.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift))

However, cancellation is cooperative. The group still waits for remaining child tasks to finish. If those tasks ignore cancellation, the parent still waits. ([GitHub, "TaskGroup.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift))

Also, `waitForAll()` captures and rethrows the first child error, but the documentation notes that this does not cancel the group automatically when the error is captured by `waitForAll()` itself. ([GitHub, "TaskGroup.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift))

Interview version:

> A throwing task group does not magically kill sibling tasks the moment one child throws. The parent observes the error while collecting results. If that error escapes the group body, Swift cancels the remaining children and waits for them to complete. Since cancellation is cooperative, children need to check cancellation or use cancellation-aware APIs. For fail-fast semantics, I usually collect with `group.next()` and let the first meaningful error escape, or explicitly call `cancelAll()` if I catch and handle inside the group.

---

### Q3. When is plain sequential `await` the right primitive?

Use sequential `await` when operations have true data dependency, required ordering, side effects that must not overlap, or when the work is cheap enough that concurrency overhead is not justified.

```swift
let session = try await auth.startSession()
let dashboard = try await api.fetchDashboard(sessionID: session.id)
```

Interview version:

> I do not parallelize just because code is async. If the second operation depends on the first, or if ordering is part of the business logic, sequential `await` is clearer and safer. Concurrency is a correctness and performance tool, not decoration.

---

## 5. Code probe

The D3 rubric has no dedicated code probe, so here are supplemental probes that expose the common mistakes.

### Probe A — `async let` requires an async context

Given:

```swift
func work() async -> Int { 1 }

func bad() {
    async let value = work()
    print(value)
}
```

### What happens?

Compiled with Swift 6.2.1 in Swift 6 language mode:

```text
/tmp/d3_error.swift:4:11: error: 'async let' in a function that does not support concurrency
1 | func work() async -> Int { 1 }
2 | 
3 | func bad() {
  |      `- note: add 'async' to function 'bad()' to make it asynchronous
4 |     async let value = work()
  |           `- error: 'async let' in a function that does not support concurrency
5 |     print(value)
6 | }

/tmp/d3_error.swift:5:11: error: 'async let' in a function that does not support concurrency
1 | func work() async -> Int { 1 }
2 | 
3 | func bad() {
  |      `- note: add 'async' to function 'bad()' to make it asynchronous
4 |     async let value = work()
5 |     print(value)
  |           `- error: 'async let' in a function that does not support concurrency
6 | }
```

### Why?

An `async let` child must be awaited before leaving its scope. Therefore the enclosing context must support suspension. SE-0317 states that `async let` declarations can only occur where `await` is legal, such as async functions or async closures. ([GitHub, "async let"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md))

### Fix

```swift
func good() async {
    async let value = work()
    print(await value)
}
```

---

### Probe B — `async let` in a loop, awaited immediately

Given:

```swift
import Foundation

func fetch(_ id: Int) async -> Int {
    print("start \(id)")
    try? await Task.sleep(for: .milliseconds(5))
    print("end \(id)")
    return id
}

@main
struct Main {
    static func main() async {
        var values: [Int] = []

        for id in 1...3 {
            async let value = fetch(id)
            values.append(await value)
        }

        print(values)
    }
}
```

### What happens?

```text
start 1
end 1
start 2
end 2
start 3
end 3
[1, 2, 3]
```

### Why?

Each iteration creates one child task and immediately awaits it. The next iteration does not begin until the current child result is available. The code uses `async let`, but the control flow is sequential.

### Fix or redesign

```swift
import Foundation

func fetch(_ id: Int) async -> Int {
    id * 10
}

@main
struct Main {
    static func main() async {
        let ids = [1, 2, 3]

        let values = await withTaskGroup(of: Int.self) { group in
            for id in ids {
                group.addTask {
                    await fetch(id)
                }
            }

            var collected: [Int] = []
            for await value in group {
                collected.append(value)
            }
            return collected.sorted()
        }

        print(values)
    }
}
```

Exact output:

```text
[10, 20, 30]
```

### Why this fix is correct

The task group creates all child tasks first, then collects results as they complete. Since result order is completion-based and not insertion-based, the example sorts before printing to make output deterministic.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`async let`|Fixed small number of known operations|Not good for dynamic counts|
|`withTaskGroup`|Dynamic count, completion-order collection|More boilerplate|
|`withThrowingTaskGroup`|Dynamic count where children can throw|Requires clear failure policy|
|`withDiscardingTaskGroup`|Dynamic side-effecting work with no useful result|Cannot collect child values|
|Sequential `await`|Real dependency or required ordering|No parallel speedup|

---

## 6. Exercise

### Problem

Parallelize three independent network calls and explain how failure in one should affect the others.

### Bad / naive version

```swift
func loadHomeData() async throws -> HomeData {
    let profile = try await fetchProfile()
    let recommendations = try await fetchRecommendations()
    let unreadCount = try await fetchUnreadCount()

    return HomeData(
        profile: profile,
        recommendations: recommendations,
        unreadCount: unreadCount
    )
}
```

### What is wrong?

```text
The calls are independent, but the code waits for each one before starting the next.
Total latency becomes roughly:
profile latency + recommendations latency + unread count latency
```

This is correct but unnecessarily slow if the three calls are truly independent.

### Improved version with `async let`

```swift
struct HomeData: CustomStringConvertible {
    let profile: String
    let recommendations: [String]
    let unreadCount: Int

    var description: String {
        "HomeData(profile: \(profile), recommendations: \(recommendations), unreadCount: \(unreadCount))"
    }
}

func fetchProfile() async throws -> String {
    "Aykut"
}

func fetchRecommendations() async throws -> [String] {
    ["Swift", "Concurrency"]
}

func fetchUnreadCount() async throws -> Int {
    3
}

func loadHomeData() async throws -> HomeData {
    async let profile = fetchProfile()
    async let recommendations = fetchRecommendations()
    async let unreadCount = fetchUnreadCount()

    return try await HomeData(
        profile: profile,
        recommendations: recommendations,
        unreadCount: unreadCount
    )
}
```

Test harness:

```swift
@main
struct Main {
    static func main() async throws {
        print(try await loadHomeData())
    }
}
```

Exact output:

```text
HomeData(profile: Aykut, recommendations: ["Swift", "Concurrency"], unreadCount: 3)
```

### Why this is better

All three child tasks start before the parent awaits their values. The parent still cannot return until all children complete, so the structure remains safe.

Failure policy here is **all-or-nothing**:

```swift
enum APIError: Error {
    case profileFailed
}

func fetchProfile() async throws -> String {
    throw APIError.profileFailed
}

func fetchRecommendations() async throws -> [String] {
    do {
        try await Task.sleep(for: .seconds(1))
        return ["Swift"]
    } catch {
        print("recommendations cancelled")
        throw error
    }
}

func load() async throws -> (String, [String]) {
    async let profile = fetchProfile()
    async let recommendations = fetchRecommendations()
    return try await (profile, recommendations)
}
```

Exact output when `fetchProfile()` throws:

```text
recommendations cancelled
profileFailed
```

Why: once the error causes the scope to exit, the sibling async-let child is canceled and awaited before the parent finishes. SE-0317 states that async-let child tasks cannot outlive their scope, are implicitly awaited by scope exit, and are canceled before awaiting when the scope exits by throwing. ([GitHub, "async let"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md))

### Alternative: partial-data policy

Sometimes one failed call should not fail the whole screen.

```swift
struct PartialHomeData {
    let profile: String
    let recommendations: [String]
    let unreadCount: Int
}

func loadPartialHomeData() async throws -> PartialHomeData {
    async let profile = fetchProfile()

    async let recommendations: [String] = {
        do {
            return try await fetchRecommendations()
        } catch {
            return []
        }
    }()

    async let unreadCount: Int = {
        do {
            return try await fetchUnreadCount()
        } catch {
            return 0
        }
    }()

    return try await PartialHomeData(
        profile: profile,
        recommendations: recommendations,
        unreadCount: unreadCount
    )
}
```

This says:

```text
profile failure = fail the screen
recommendations failure = show empty recommendations
unread count failure = show zero
```

That is a product decision encoded in the concurrency structure.

---

## 7. Production guidance

Use `async let` in production when:

```text
- You have a fixed, small number of independent operations.
- Results can have different types.
- The parent needs all results before continuing.
- Failure should usually cancel sibling work.
- The lexical scope clearly owns the child tasks.
```

Use task groups when:

```text
- The number of child tasks depends on runtime data.
- You need completion-order processing.
- You need race / first-success / first-result behavior.
- You need to limit concurrency width manually.
- You need custom cancellation or partial failure handling.
```

Use discarding task groups when:

```text
- You need dynamic structured child work.
- Child tasks are side-effecting.
- Individual child return values do not matter.
```

Use sequential `await` when:

```text
- Operation B depends on operation A.
- Side effects must happen in order.
- Parallelism would increase server load without improving UX.
- The work is cheap and concurrency overhead is not worth it.
```

Be careful when:

```text
- Capturing non-Sendable mutable state inside group child tasks.
- Mutating shared dictionaries or arrays from child tasks.
- Assuming result order from a task group.
- Ignoring cancellation in long-running child tasks.
- Creating unbounded task groups over large collections.
```

Avoid when:

```text
- You are parallelizing just to look sophisticated.
- You are using a task group as fire-and-forget.
- You are hiding business failure policy inside generic catch blocks.
- You are using Task.detached because task group syntax feels verbose.
```

Debugging checklist:

```text
1. Are these operations truly independent?
2. Is the task count fixed or dynamic?
3. Do I need all results, first result, or no results?
4. What should happen when one child fails?
5. Do sibling tasks observe cancellation?
6. Is result order meaningful?
7. Am I accidentally mutating shared state from children?
8. Is the concurrency width bounded enough for production load?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `async let` and task groups both run async work in parallel. Use `async let` for simple cases and task groups for many tasks.

### Senior answer

> `async let` is for fixed lexical fan-out where I know the child tasks statically and need their results before leaving scope. Task groups are for dynamic fan-out, completion-order collection, custom cancellation, and patterns like parallel map or race. I also consider whether the work should be parallel at all, because sequential `await` is correct when there is a dependency.

### Staff-level answer

> I choose the primitive based on workload shape, failure semantics, cancellation behavior, ordering, resource pressure, and API readability. For UI aggregation, `async let` is often the best expression of “load these three independent pieces or fail the whole model.” For dynamic collections, I use task groups, often with bounded concurrency and explicit result ordering. I avoid mutating shared state from child tasks and make failure policy visible: all-or-nothing, partial data, first-success, or best-effort. I also think about backend pressure, cancellation from view lifecycle, testability, and whether concurrency improves user-perceived latency enough to justify the complexity.

Staff-level questions to ask:

```text
What is the failure policy: all-or-nothing, partial, best-effort, first-success?
Can the child tasks overwhelm a backend or local resource if unbounded?
Do results need source order, completion order, or no order?
Do child tasks capture only Sendable state?
How will cancellation from UI lifecycle propagate into this work?
```

---

## 9. Interview-ready summary

`async let` is the right tool for a fixed, small number of independent child tasks whose results are needed in the same scope, especially when result types differ. Task groups are better for dynamic fan-out, completion-order result collection, racing, bounded parallelism, and custom cancellation. Throwing task groups surface child errors when results are collected; if an error escapes the group body, remaining children are canceled, but cancellation is cooperative and the group still waits for children to finish. Sequential `await` remains the best design when operations are ordered or dependent.

---

## 10. Flashcards

Q: When should you use `async let`?  
A: For a fixed, small number of independent child operations whose results are needed before leaving the current scope.

Q: When should you use a task group instead of `async let`?  
A: When the number of tasks is dynamic, results should be collected in completion order, concurrency needs bounding, or the pattern is race / first-success / dynamic fan-out.

Q: Does a throwing task group immediately cancel siblings when one child throws?  
A: Not merely because a child throws. The error must be observed and propagated, or cancellation must be explicitly requested. If the error escapes the group body, remaining children are canceled.

Q: Does cancellation stop child tasks immediately?  
A: No. Cancellation is cooperative. Child tasks must check cancellation or call cancellation-aware APIs.

Q: Do task-group results arrive in insertion order?  
A: No. They arrive in completion order.

Q: Is sequential `await` less correct than parallel code?  
A: No. Sequential `await` is correct when operations are dependent, ordered, or not worth parallelizing.

Q: Why can `async let` be cleaner than a task group for three network calls?  
A: It avoids boilerplate and handles heterogeneous result types naturally.

Q: Why is an unbounded task group risky?  
A: It can create too many concurrent child tasks, overload networking, exhaust memory, or create server-side pressure.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D10 — `AsyncSequence`, streams, and backpressure-aware design]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[F1 — Testing async and concurrent code]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — D3 — `async let`, task groups, and choosing the right concurrency primitive.
- GitHub. "async let." Swift Evolution SE-0317. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0317-async-let.md
- GitHub. "Structured Concurrency." Swift Evolution SE-0304. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md
- GitHub. "TaskGroup.swift." swiftlang/swift. https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskGroup.swift
- Apple Developer. "TaskGroup." Apple Developer Documentation. https://developer.apple.com/documentation/swift/taskgroup
