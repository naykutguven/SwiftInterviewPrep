---
tags:
  - interview-prep
  - swift
  - structured-concurrency
  - async-await
  - concurrency
---
## 0. Rubric snapshot

**Rubric expectation**

Understand:

- `async` / `await`
- suspension points
- task trees
- the difference between concurrency and parallelism

The rubric’s caveat is the key interview signal: `await` is a suspension point, not syntax noise; every suspension can change interleaving assumptions.

**You should be able to answer**

- What does `await` actually mean semantically?
- Why is structured concurrency safer than ad hoc callback trees or detached work?

**You should be able to do**

- Rewrite a callback-based API to `async` / `await`.
- Explain what correctness properties improve.

---

## 1. Core mental model

Swift concurrency is not “callbacks with nicer syntax.” The model is: asynchronous functions can suspend without blocking a thread, and structured concurrency gives concurrent child work a lexical lifetime.

The key idea:

```text
async function = ordinary-looking control flow that may suspend at explicit await points
structured concurrency = child work cannot outlive the scope that created it
```

An `async` function can start, reach an `await`, give up its thread, and later resume from the same logical point. Swift does **not** guarantee that it resumes on the same physical thread, unless actor isolation gives you an executor guarantee such as `@MainActor` or another actor. SE-0296 describes asynchronous functions as functions with the special ability to give up their thread and resume later; thread identity is intentionally not part of the high-level model. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md "swift-evolution/proposals/0296-async-await.md at main · swiftlang/swift-evolution · GitHub"))

`await` marks a **potential suspension point**. It does not mean the function definitely suspends every time. It means the operation is allowed to suspend, and the caller must acknowledge that suspension could interrupt atomic-looking reasoning. SE-0296 explicitly says potential suspension points must be marked with `await` because suspension can break atomicity and allow other work to interleave. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md "swift-evolution/proposals/0296-async-await.md at main · swiftlang/swift-evolution · GitHub"))

Structured concurrency is about task shape. Child tasks created with `async let` or task groups form a task tree. Parent tasks cannot complete until their structured child tasks complete. Apple’s WWDC material describes this as a real hierarchy that affects cancellation, priority, and task-local values, not merely an implementation detail. ([Apple Developer](https://developer.apple.com/wwdc21/10134 "Explore structured concurrency in Swift - WWDC21 - Videos - Apple Developer"))

Concurrency is not the same as parallelism:

```text
Concurrency = multiple pieces of work are in progress over overlapping time.
Parallelism = multiple pieces of work execute at the exact same time, usually on different cores.
```

You can have concurrency on one thread through suspension and interleaving. You can have parallelism when independent tasks are scheduled simultaneously. `async` / `await` enables non-blocking asynchronous composition; it does not automatically mean “run this in parallel.”

---

## 2. Essential mechanics

### 2.1 `async` changes the function type

An `async` function can only be called from an asynchronous context, or from a new task created at a synchronous boundary.

```swift
func fetchUser() async throws -> User {
    // May suspend internally.
}

func loadScreen() async throws {
    let user = try await fetchUser()
    render(user)
}
```

`async` is part of the function’s type, similar to `throws`. A synchronous function cannot simply call an async function because the async function may need to abandon its thread, while the synchronous caller has no suspension machinery. SE-0296 explains that blocking the entire thread to wait would defeat the purpose of async functions. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md "swift-evolution/proposals/0296-async-await.md at main · swiftlang/swift-evolution · GitHub"))

Bad:

```swift
func buttonTapped() {
    let user = await fetchUser()
    // error: 'async' call in a function that does not support concurrency
}
```

Better at a synchronous boundary:

```swift
func buttonTapped() {
    Task {
        let user = try await fetchUser()
        render(user)
    }
}
```

But this `Task {}` is an **unstructured task**. It is useful at boundaries such as UI callbacks, but it does not create a structured child task of `buttonTapped()`.

---

### 2.2 `await` is a possible suspension point

```swift
let user = try await api.fetchUser(id: id)
```

This means:

```text
The current task may suspend here.
The current thread is not blocked just because this task is waiting.
Other work may run before this task resumes.
The task resumes from this logical point when the awaited operation completes.
```

It does **not** mean:

```text
Start this function on a background thread.
Always suspend.
Run the next line in parallel.
Keep the same thread after resumption.
```

This is the biggest D1 trap. `await` is a semantic warning label: “your assumptions before this line may be stale after this line.”

---

### 2.3 Sequential `await` is still sequential

```swift
func loadProfile() async throws -> Profile {
    let user = try await api.fetchUser()
    let avatar = try await api.fetchAvatar(for: user.id)

    return Profile(user: user, avatar: avatar)
}
```

This is asynchronous but sequential:

```text
fetchUser starts
wait for fetchUser
fetchAvatar starts
wait for fetchAvatar
build Profile
```

It is still valuable because it does not block a thread while waiting. But it is not parallel.

Use sequential `await` when the second operation depends on the first.

---

### 2.4 `async let` creates structured child tasks

```swift
func loadProfile(id: User.ID) async throws -> Profile {
    async let user = api.fetchUser(id: id)
    async let avatar = api.fetchAvatar(userID: id)

    let (loadedUser, loadedAvatar) = try await (user, avatar)
    return Profile(user: loadedUser, avatar: loadedAvatar)
}
```

Here, both operations can start concurrently because they are independent.

The child tasks are scoped to `loadProfile`. The parent cannot return while these child tasks are still running. If exiting the scope early, Swift cancels unawaited structured child work and waits for it to finish. Apple describes this as a fundamental guarantee that prevents accidentally leaking tasks. ([Apple Developer](https://developer.apple.com/wwdc21/10134 "Explore structured concurrency in Swift - WWDC21 - Videos - Apple Developer"))

Use `async let` when:

```text
The number of child tasks is known statically.
The child work is independent.
The child work should not outlive the current scope.
```

Use a task group when the number of child tasks is dynamic. That is D3.

---

### 2.5 Task trees make lifetime explicit

Structured tasks form a parent-child tree:

```text
Parent task: loadFeed()
├── child task: fetchArticles()
├── child task: fetchRecommendations()
└── child task: fetchUserPreferences()
```

This tree gives Swift a place to attach important behavior:

```text
lifetime
cancellation
priority propagation
task-local values
debugging / tracing
```

WWDC23 summarizes the model well: structured concurrency lets you reason about where execution branches off and rejoins, similar to how `if` and `for` structure synchronous control flow; structured tasks are created with `async let` and task groups, while `Task` and `Task.detached` are unstructured. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2023/10170/ "Beyond the basics of structured concurrency - WWDC23 - Videos - Apple Developer"))

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `await` means “run in background”

Bad:

```swift
func load() async throws {
    let a = try await api.fetchA()
    let b = try await api.fetchB()
    let c = try await api.fetchC()

    render(a, b, c)
}
```

This is asynchronous, but not concurrent among `fetchA`, `fetchB`, and `fetchC`.

Better when independent:

```swift
func load() async throws {
    async let a = api.fetchA()
    async let b = api.fetchB()
    async let c = api.fetchC()

    try await render(a, b, c)
}
```

Nuance: `try await render(a, b, c)` is valid only if `render` accepts the resolved values and the `async let` values are awaited at that use site. For clarity, many teams prefer this:

```swift
let (loadedA, loadedB, loadedC) = try await (a, b, c)
render(loadedA, loadedB, loadedC)
```

---

### Trap 2: Assuming state before `await` is still valid after `await`

Bad:

```swift
@MainActor
final class SearchViewModel {
    var query = ""
    var results: [ResultItem] = []

    func search() async {
        let currentQuery = query
        let loaded = try? await api.search(currentQuery)

        // Bug risk: query may have changed while this task was suspended.
        results = loaded ?? []
    }
}
```

Better:

```swift
@MainActor
final class SearchViewModel {
    var query = ""
    var results: [ResultItem] = []

    func search() async {
        let currentQuery = query
        let loaded = try? await api.search(currentQuery)

        guard query == currentQuery else {
            return // stale result
        }

        results = loaded ?? []
    }
}
```

This trap becomes central with actors and UI state. `await` can create a gap where another task changes the world.

---

### Trap 3: Wrapping everything in `Task {}`

Bad:

```swift
func load() {
    Task {
        await fetchA()
    }

    Task {
        await fetchB()
    }

    Task {
        await fetchC()
    }
}
```

This creates unstructured work. The surrounding function does not naturally wait for these tasks, cancellation is not scoped to the function, and failures can become hard to reason about.

Better:

```swift
func load() async throws {
    async let a: Void = fetchA()
    async let b: Void = fetchB()
    async let c: Void = fetchC()

    try await (a, b, c)
}
```

Use `Task {}` at real async boundaries, such as:

```text
UIKit delegate callback
SwiftUI button action
App lifecycle callback
fire-and-forget telemetry with explicit ownership
```

Do not use it to avoid designing an async API.

---

### Trap 4: Blocking inside async code

Bad:

```swift
func load() async {
    Thread.sleep(forTimeInterval: 2)
}
```

Better:

```swift
func load() async throws {
    try await Task.sleep(for: .seconds(2))
}
```

Blocking occupies a thread. Suspension frees the thread for other work. Swift concurrency is designed around suspension, not around hiding blocking calls behind `async`.

---

## 4. Direct answers to rubric questions

### Q1. What does `await` actually mean semantically?

`await` marks a potential suspension point. The current task may pause, give up its thread, and resume later from the same logical point. The operation may not actually suspend at runtime, but the call site must be written as if it could.

Interview version:

> `await` is not just syntax for waiting. It marks a potential suspension point where the current task may give up its thread and later resume. That matters because suspension breaks atomic-looking reasoning: after an `await`, other work may have interleaved, state may have changed, and actor-isolated code may have processed other messages. It does not automatically start parallel work and it does not guarantee resumption on the same thread.

---

### Q2. Why is structured concurrency safer than ad hoc callback trees or detached work?

Structured concurrency gives child work a lexical scope and a parent-child relationship. That lets Swift manage lifetime, cancellation, priority, task-local values, error propagation, and cleanup in a predictable way.

Callback trees and detached tasks often hide ownership:

```text
Who owns this work?
When does it stop?
Who observes errors?
What cancels it?
Can it outlive the screen/service/request that started it?
```

Structured concurrency makes those questions visible in the code.

Interview version:

> Structured concurrency is safer because child tasks are tied to the scope that creates them. The parent cannot finish until structured children finish, and cancellation/error behavior has a defined path through the task tree. With callbacks or detached tasks, work can easily outlive its owner, drop errors, ignore cancellation, or mutate stale state after the user has moved on.

---

## 5. Code probe / examples

D1 has no rubric code probe, so use these three examples instead.

### 5.1 Minimal example: async does not automatically mean parallel

Given:

```swift
func first() async -> Int {
    print("first")
    return 1
}

func second() async -> Int {
    print("second")
    return 2
}

@main
struct App {
    static func main() async {
        let a = await first()
        let b = await second()
        print(a + b)
    }
}
```

Exact output:

```text
first
second
3
```

Why:

```text
main starts
await first()
first completes
await second()
second completes
print sum
```

There is no child task here. The same task performs the calls sequentially.

---

### 5.2 Counterexample: async call from sync context

Given:

```swift
func fetch() async -> Int {
    1
}

func tapped() {
    let data = await fetch()
    print(data)
}
```

Swift 6.2 compiler error:

```text
/tmp/test.swift:4:22: error: 'async' call in a function that does not support concurrency
1 | func fetch() async -> Int { 1 }
2 | 
3 | func tapped() {
  |      `- note: add 'async' to function 'tapped()' to make it asynchronous
4 |     let data = await fetch()
  |                      `- error: 'async' call in a function that does not support concurrency
5 |     print(data)
6 | }
```

Fix at an async boundary:

```swift
func tapped() {
    Task {
        let data = await fetch()
        print(data)
    }
}
```

Better architectural fix when possible:

```swift
func tapped() {
    viewModel.load()
}

@MainActor
final class ViewModel {
    func load() {
        Task {
            let data = await fetch()
            print(data)
        }
    }
}
```

The `Task` should be owned by the boundary that can reason about lifetime. In UIKit or SwiftUI, that is usually the view model, view task modifier, or coordinator.

---

### 5.3 Production example: independent requests

Naive sequential version:

```swift
func makeDashboard(userID: User.ID) async throws -> Dashboard {
    let user = try await api.fetchUser(id: userID)
    let notifications = try await api.fetchNotifications(userID: userID)
    let recommendations = try await api.fetchRecommendations(userID: userID)

    return Dashboard(
        user: user,
        notifications: notifications,
        recommendations: recommendations
    )
}
```

Improved structured version:

```swift
func makeDashboard(userID: User.ID) async throws -> Dashboard {
    async let user = api.fetchUser(id: userID)
    async let notifications = api.fetchNotifications(userID: userID)
    async let recommendations = api.fetchRecommendations(userID: userID)

    let loaded = try await (
        user,
        notifications,
        recommendations
    )

    return Dashboard(
        user: loaded.0,
        notifications: loaded.1,
        recommendations: loaded.2
    )
}
```

Why this is correct:

```text
The three requests are independent.
The child tasks are scoped to makeDashboard(userID:).
The function cannot return while those children are still running.
If the parent task is cancelled, cancellation can propagate through the task tree.
If one child throws, the structured scope has defined cleanup behavior.
```

Tradeoff:

|Option|Use when|Tradeoff|
|---|---|---|
|Sequential `await`|Later work depends on earlier results|Simpler, but no overlap|
|`async let`|Small fixed number of independent operations|Great readability, less flexible|
|Task group|Dynamic number of child operations|More code, more control|
|`Task {}`|Synchronous boundary into async world|Unstructured lifetime|
|`Task.detached`|Rarely; independent work that must not inherit current context|Easy to lose cancellation, priority, actor isolation, and ownership|

---

## 6. Exercise

### Problem

Rewrite a callback-based API to `async` / `await` and explain what correctness properties improve.

### Bad / naive callback version

```swift
struct UserDTO: Decodable {
    let id: String
    let name: String
}

protocol LegacyUserClient {
    func fetchUser(
        id: String,
        completion: @escaping (Result<UserDTO, Error>) -> Void
    )
}

final class ProfileViewModel {
    private let client: LegacyUserClient

    var name: String = ""
    var errorMessage: String?

    init(client: LegacyUserClient) {
        self.client = client
    }

    func load(id: String) {
        client.fetchUser(id: id) { result in
            switch result {
            case .success(let user):
                self.name = user.name

            case .failure(let error):
                self.errorMessage = error.localizedDescription
            }
        }
    }
}
```

### What is wrong?

```text
The completion can run on an unclear executor/thread.
The lifetime of the callback is not obvious.
self is captured strongly.
Cancellation is not represented.
Error flow is separated from normal control flow.
Composition with other async operations becomes nested and fragile.
Testing requires callback expectations instead of direct async tests.
```

This code might be acceptable in older codebases, but it does not express the operation’s semantics clearly.

---

### Improved async wrapper

```swift
struct UserClient {
    private let legacy: LegacyUserClient

    init(legacy: LegacyUserClient) {
        self.legacy = legacy
    }

    func fetchUser(id: String) async throws -> UserDTO {
        try await withCheckedThrowingContinuation { continuation in
            legacy.fetchUser(id: id) { result in
                continuation.resume(with: result)
            }
        }
    }
}
```

Important caveat:

```text
This wrapper is correct only for a one-shot callback that always completes exactly once.
It does not automatically add cancellation to the underlying legacy operation.
Continuation correctness is D9 territory.
```

### Improved view model

```swift
@MainActor
final class ProfileViewModel {
    private let client: UserClient

    private(set) var name: String = ""
    private(set) var errorMessage: String?

    init(client: UserClient) {
        self.client = client
    }

    func load(id: String) async {
        do {
            let user = try await client.fetchUser(id: id)
            name = user.name
            errorMessage = nil
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

Call from a SwiftUI view:

```swift
struct ProfileView: View {
    @State private var viewModel: ProfileViewModel

    var body: some View {
        Text(viewModel.name)
            .task {
                await viewModel.load(id: "123")
            }
    }
}
```

### Why this is better

```text
The operation has a normal return value.
Errors use throwing control flow.
UI mutation is main-actor isolated.
The call site can use structured task lifetime through SwiftUI .task.
Tests can await the operation directly.
Composition with async let or task groups becomes straightforward.
```

For example:

```swift
func makeProfileScreen(id: String) async throws -> ProfileScreenModel {
    async let user = userClient.fetchUser(id: id)
    async let permissions = permissionsClient.fetchPermissions(userID: id)

    let (loadedUser, loadedPermissions) = try await (user, permissions)

    return ProfileScreenModel(
        user: loadedUser,
        permissions: loadedPermissions
    )
}
```

The improvement is not just readability. The async version makes ownership, isolation, error flow, and composition easier to reason about.

---

## 7. Production guidance

Use structured concurrency in production when:

```text
You are composing asynchronous operations.
The work has a clear owner and lifetime.
The caller should observe success or failure.
Cancellation should flow from parent to child.
The work should be testable with async tests.
You are replacing callback pyramids or DispatchGroup-style coordination.
```

Be careful when:

```text
You cross from synchronous APIs into async code.
You use Task {} from UI callbacks.
You bridge legacy completion handlers.
You mutate state after await.
You assume sequential awaits are concurrent.
You add async let without proving independence.
```

Avoid:

```text
Task.detached as a default escape hatch.
Blocking waits inside async functions.
Semaphores to force async code back into sync code.
Fire-and-forget business logic without explicit ownership.
Large async functions that mutate shared state across many await points.
```

Debugging checklist:

```text
Where are the suspension points?
Can state change between this await and the next line?
Is this work structured or unstructured?
Who cancels it?
Who observes errors?
Can the task outlive the screen/request/service that started it?
Are independent operations actually run concurrently?
Is UI mutation main-actor isolated?
Is legacy callback bridging exactly-once and cancellation-aware?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `async` / `await` makes asynchronous code easier to write. You put `await` before async calls.

This is not enough for senior interviews.

### Senior answer

> `await` marks a potential suspension point. The current task may suspend without blocking a thread and resume later. After an `await`, other work may have interleaved, so I need to revalidate state. Structured concurrency gives child work scoped lifetimes through `async let` and task groups, which improves cancellation, error propagation, and reasoning compared to callbacks or detached tasks.

### Staff-level answer

> I treat Swift concurrency as an ownership and correctness model, not just syntax. Async APIs should express operation lifetime, isolation, cancellation, and error flow. I prefer structured child tasks for bounded work, keep unstructured tasks at explicit boundaries, avoid detached tasks unless I can prove context independence, and review every `await` as a possible interleaving point. In migrations, I wrap callback APIs at module boundaries, preserve cancellation where possible, isolate UI mutation with `@MainActor`, and avoid spreading continuations or `Task {}` throughout the codebase.

Staff-level questions to ask:

```text
What owns this asynchronous work?
Can the task outlive its owner?
What happens if the parent is cancelled?
Where can state become stale across await?
Is this concurrency actually needed, or is sequential await clearer?
Should this be async let, a task group, AsyncSequence, actor isolation, or plain sequential code?
What is the migration boundary between legacy callbacks and async APIs?
```

---

## 9. Interview-ready summary

Structured concurrency in Swift gives asynchronous work a shape. `async` functions can suspend without blocking a thread, and `await` marks a potential suspension point where interleaving can happen. Sequential `await`s are not parallel; concurrency comes from creating child tasks with tools like `async let` or task groups. The safety benefit is that structured child tasks have scoped lifetimes: the parent cannot finish while its children are still running, and cancellation, priority, and task-local information can flow through the task tree. In production, I use structured concurrency to make ownership, cancellation, error flow, and state isolation explicit, while keeping `Task {}` and especially `Task.detached` limited to deliberate boundaries.

---

## 10. Flashcards

Q: What does `await` mean in Swift?

A: It marks a potential suspension point. The current task may give up its thread and resume later.

Q: Does `await` start work in parallel?

A: No. `await` waits for an async operation. Parallel/concurrent child work requires constructs like `async let` or task groups.

Q: Can an async function resume on a different thread after `await`?

A: Yes. Swift does not guarantee the same thread after suspension. Actor isolation can guarantee resumption on the actor’s executor, not on an arbitrary same thread.

Q: What is structured concurrency?

A: A model where child tasks are scoped to the parent task, forming a task tree with defined lifetime, cancellation, priority, and task-local behavior.

Q: Which Swift constructs create structured child tasks?

A: `async let` and task groups.

Q: Is `Task {}` structured?

A: No. `Task {}` creates unstructured work. It may inherit context, but it is not a structured child with lexical lifetime.

Q: Why is `Task.detached` risky?

A: It loses structured lifetime and does not inherit the same context in the way ordinary structured child tasks do. It is easy to lose cancellation, priority, isolation, and ownership reasoning.

Q: What should you re-check after an `await`?

A: Any assumption that could have changed while suspended: user input, selected item, current query, authorization state, cache contents, actor state, view lifecycle.

Q: When should you use sequential `await`?

A: When operations depend on each other or when concurrency would add unnecessary complexity.

Q: When should you use `async let`?

A: For a small, fixed number of independent child operations that should complete within the current scope.

---

## 11. Related sections

- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D3 — `async let`, task groups, and choosing the right concurrency primitive]]
- [[D4 — Actors and actor isolation]]
- [[D5 — Global actors and `@MainActor`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D9 — Continuations and bridging callback APIs]]
- [[D10 — `AsyncSequence`, streams, and backpressure-aware design]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D13 — Swift 6 strict concurrency migration]]
- [[D14 — Swift 6.2 approachable concurrency and new defaults]]
- [[F1 — Testing async and concurrent code]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

---

## 12. Sources

- Swift Senior/Staff Rubric — D1 Structured concurrency fundamentals.
- Swift Evolution SE-0296 — `async` / `await`, suspension points, and why `await` marks possible suspension. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md "swift-evolution/proposals/0296-async-await.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Evolution SE-0304 — structured concurrency, task hierarchy, and async/await not introducing concurrency by itself. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md "swift-evolution/proposals/0304-structured-concurrency.md at main · swiftlang/swift-evolution · GitHub"))
- Apple WWDC21 — task trees, child task lifetime, cancellation, and structured task cleanup. ([Apple Developer](https://developer.apple.com/wwdc21/10134 "Explore structured concurrency in Swift - WWDC21 - Videos - Apple Developer"))
- Apple WWDC23 — task tree model, structured vs unstructured tasks, and why to prefer structured tasks. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2023/10170/ "Beyond the basics of structured concurrency - WWDC23 - Videos - Apple Developer"))