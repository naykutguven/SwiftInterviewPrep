---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - detached-tasks
  - task-inheritance
  - main-actor
---
## 0. Rubric snapshot

**Rubric expectation**

Understand the difference between `Task {}` and `Task.detached {}`. The rubric explicitly calls out D11 as “Detached tasks, task inheritance, and isolation loss.”

**Caveats**

Detached tasks lose actor context, task-local values, priority inheritance, and structured-parent guarantees; many uses are wrong.

**You should be able to answer**

- Why is `Task.detached` often a footgun?
- What does `Task {}` inherit that a detached task does not?

**You should be able to do**

- Review code that uses detached tasks from a view model and explain why it may violate isolation or lifecycle expectations.
- Diagnose the rubric code probe involving a `@MainActor` view model and `Task.detached`.

---

## 1. Core mental model

`Task.detached` creates a new top-level unstructured task that intentionally does **not** inherit the surrounding concurrency context. That is the whole point of it. It says: “Start independent async work that is not part of the current actor, not part of the current task’s metadata, and not automatically governed by the current task’s lifecycle.”

`Task {}` is also an **unstructured** task, not a child task. This is a common trap. It is not structured like `async let` or `withTaskGroup`. However, `Task {}` does inherit important context from where it is created: priority, task-local values, and actor isolation. Swift Evolution SE-0304 says unstructured tasks created with the `Task` initializer inherit priority, task-local values, and actor isolation from the creation context; detached tasks do not inherit priority, task-local values, or actor context. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md "swift-evolution/proposals/0304-structured-concurrency.md at main · swiftlang/swift-evolution · GitHub"))

Structured child tasks are different. `async let` and task groups create child tasks whose lifetime is scoped to the parent; Swift automatically cancels and awaits unfinished child tasks when leaving the scope. Apple’s WWDC structured concurrency session describes the task tree as fundamental because it propagates cancellation, priority, and task-local variables, and prevents accidentally leaking child tasks beyond their scope. ([Apple Developer](https://developer.apple.com/wwdc21/10134 "Explore structured concurrency in Swift - WWDC21 - Videos - Apple Developer"))

The key idea:

```text
Task {}          = unstructured, but context-inheriting
Task.detached {} = unstructured, context-detached
async let/group  = structured child task
```

For iOS code, especially `@MainActor` view models, `Task.detached` is usually the wrong instinct. You usually want UI orchestration to stay main-actor isolated, while expensive or shared mutable work lives behind an async service, an actor, a task group, or a deliberately nonisolated/concurrent boundary.

---

## 2. Essential mechanics

### 2.1 `Task {}` inherits actor isolation

Inside a `@MainActor` type, `Task {}` inherits the main actor context. That means the closure can access main-actor-isolated state directly.

```swift
@MainActor
final class ViewModel {
    var title = ""

    func load() {
        Task {
            title = "Done" // OK: closure inherits MainActor isolation
        }
    }
}
```

This does **not** mean the work is structured. If you discard the task handle, the task can still run to completion. SE-0304 says task handles are used to await or cancel unstructured tasks, but tasks run to completion even if there are no remaining uses of the handle. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md "swift-evolution/proposals/0304-structured-concurrency.md at main · swiftlang/swift-evolution · GitHub"))

### 2.2 `Task.detached {}` loses actor isolation

A detached task starts outside the current actor context. In a `@MainActor` view model, this means the closure is **not** main-actor isolated.

```swift
@MainActor
final class ViewModel {
    var title = ""

    func load() {
        Task.detached {
            title = "Done" // Error
        }
    }
}
```

The detached closure cannot synchronously mutate `title`, because `title` is isolated to `MainActor`. Global actors make annotated declarations actor-isolated; declarations isolated to a global actor can only be synchronously accessed from that same actor, and cross-actor access must be asynchronous. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md "swift-evolution/proposals/0316-global-actors.md at main · swiftlang/swift-evolution · GitHub"))

### 2.3 `Task.detached` does not inherit priority or task-local values

This matters in production more than people expect.

```swift
enum RequestID {
    @TaskLocal static var current: String?
}

func handleRequest() async {
    await RequestID.$current.withValue("abc-123") {
        Task {
            print(RequestID.current as Any) // Optional("abc-123")
        }

        Task.detached {
            print(RequestID.current as Any) // nil
        }
    }
}
```

That means detached tasks can silently lose tracing IDs, logging context, locale/request metadata, priority hints, and other task-local context.

### 2.4 `Task.detached` does not give you lifecycle management

This is a view-model bug pattern:

```swift
@MainActor
final class SearchViewModel {
    var results: [String] = []

    func search() {
        Task.detached {
            let values = await fetchResults()
            await MainActor.run {
                self.results = values
            }
        }
    }
}
```

Even if you fix the actor hop with `MainActor.run`, the task is still detached from the view model’s lifecycle. If the screen disappears, the task does not automatically cancel. If a newer search starts, the old detached task can still race and publish stale results.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `Task.detached` means “background thread”

Bad:

```swift
@MainActor
func refresh() {
    Task.detached {
        let data = await service.fetch()
        await MainActor.run {
            self.data = data
        }
    }
}
```

This is not a good general “background work” API. It creates lifecycle, priority, task-local, cancellation, and isolation problems.

Better:

```swift
@MainActor
func refresh() {
    refreshTask?.cancel()

    refreshTask = Task {
        do {
            let data = try await service.fetch()
            try Task.checkCancellation()
            self.data = data
        } catch is CancellationError {
            // Ignore stale work.
        } catch {
            self.errorMessage = error.localizedDescription
        }
    }
}
```

The view model keeps a handle, can cancel stale work, and UI mutation remains main-actor isolated.

### Trap 2: Thinking `Task {}` is structured

`Task {}` is **not** a child task. It does not have the parent-scope guarantee of `async let` or task groups.

```swift
func load() {
    Task {
        await doWork()
    }
}
```

This task may outlive `load()`. If you need scoped lifetime, prefer:

```swift
async let user = service.user()
async let settings = service.settings()

let model = try await Model(user: user, settings: settings)
```

or:

```swift
try await withThrowingTaskGroup(of: Item.self) { group in
    // Child tasks scoped to this block.
}
```

### Trap 3: Using detached tasks to silence isolation diagnostics

Bad:

```swift
@MainActor
final class ViewModel {
    var title = ""

    func load() {
        Task.detached {
            await MainActor.run {
                self.title = "Done"
            }
        }
    }
}
```

This may compile, but it dodges the design question: why is this work detached from the view model’s actor and lifecycle? Usually, the answer is “because the author wanted the diagnostic to go away.”

### Trap 4: Forgetting stale-result suppression

Bad:

```swift
@MainActor
func search(_ query: String) {
    Task.detached {
        let results = await service.search(query)
        await MainActor.run {
            self.results = results
        }
    }
}
```

If the user types quickly, an older search can finish after a newer one and overwrite the UI.

Better:

```swift
@MainActor
func search(_ query: String) {
    searchTask?.cancel()

    searchTask = Task { [service] in
        do {
            let results = try await service.search(query)
            try Task.checkCancellation()
            self.results = results
        } catch is CancellationError {
            // Expected for stale searches.
        } catch {
            self.errorMessage = error.localizedDescription
        }
    }
}
```

---

## 4. Direct answers to rubric questions

### Q1. Why is `Task.detached` often a footgun?

Because it throws away the surrounding execution context. It does not inherit actor isolation, priority, task-local values, or structured-parent guarantees. That makes it easy to accidentally update actor-isolated state from the wrong context, lose logging/tracing metadata, ignore cancellation, create stale-result races, and keep work alive after the screen or owner has gone away.

Interview version:

> `Task.detached` is dangerous because it creates independent unstructured work. It does not inherit the current actor, task-local values, or priority, and it is not tied to the current task’s lifecycle. In UI code, that often means losing `@MainActor` isolation and cancellation behavior. I only use it when I intentionally want independent work and I can explicitly manage priority, cancellation, Sendable captures, and actor hops.

### Q2. What does `Task {}` inherit that a detached task does not?

`Task {}` inherits important context from where it is created: priority, task-local values, and actor isolation. `Task.detached {}` does not inherit those. SE-0304 explicitly distinguishes context-inheriting unstructured tasks from detached tasks. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md "swift-evolution/proposals/0304-structured-concurrency.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> `Task {}` is unstructured, but context-inheriting. If I create it inside a `@MainActor` view model, the closure is main-actor isolated and can mutate UI state. It also inherits task-local values and priority. `Task.detached` is deliberately independent: no actor context, no task-local values, and no inherited priority.

### Q3. Is `Task {}` structured?

No. `Task {}` is unstructured. It inherits context, but it is not a child task scoped to the current function. If you discard its handle, it still runs. If you need structured lifetime, use `async let` or a task group.

Interview version:

> I separate two axes: structured lifetime and context inheritance. `async let` and task groups are structured child tasks. `Task {}` is unstructured but inherits context. `Task.detached {}` is unstructured and does not inherit context.

---

## 5. Code probe

Given:

```swift
@MainActor
final class ViewModel {
    var title = ""

    func load() {
        Task.detached {
            self.title = "Done"
        }
    }
}
```

### What happens?

With Swift 6.2.1 using `-swift-version 6`, this does **not** compile:

```text
/tmp/d11.swift:7:18: error: main actor-isolated property 'title' can not be mutated from a nonisolated context
 1 | @MainActor
 2 | final class ViewModel {
 3 |     var title = ""
   |         `- note: mutation of this property is only permitted within the actor
 4 |
 5 |     func load() {
 6 |         Task.detached {
 7 |             self.title = "Done"
   |                  |- error: main actor-isolated property 'title' can not be mutated from a nonisolated context
   |                  `- note: consider declaring an isolated method on 'MainActor' to perform the mutation
 8 |         }
 9 |     }
```

There is no runtime output because compilation fails.

### Why?

`ViewModel` is annotated `@MainActor`, so its instance members are isolated to the main actor. `title` can only be mutated synchronously from main-actor-isolated code.

`Task.detached` creates a detached task. The closure is not main-actor isolated. Therefore, `self.title = "Done"` is a cross-actor mutation attempted from a nonisolated context.

```text
@MainActor ViewModel
 └── title: MainActor-isolated state

load()
 └── Task.detached
      └── closure is nonisolated
           └── self.title = "Done" ❌ cross-actor mutation
```

### Fix or redesign

For this simple case, use `Task {}`:

```swift
@MainActor
final class ViewModel {
    var title = ""

    func load() {
        Task {
            self.title = "Done"
        }
    }
}
```

This compiles because the `Task {}` closure inherits the surrounding main actor isolation.

### Why this fix is correct

The mutation is UI/view-model state mutation. It belongs on the main actor. `Task {}` preserves the actor context, so the closure can mutate `title` safely.

However, this is still an unstructured task. In real view models, store the task handle if the work can outlive the current call or needs cancellation.

```swift
@MainActor
final class ViewModel {
    private var loadTask: Task<Void, Never>?
    var title = ""

    func load() {
        loadTask?.cancel()

        loadTask = Task {
            do {
                let value = try await fetchTitle()
                try Task.checkCancellation()
                self.title = value
            } catch is CancellationError {
                // Expected when newer work replaces older work.
            } catch {
                self.title = "Failed"
            }
        }
    }

    func cancelLoading() {
        loadTask?.cancel()
        loadTask = nil
    }

    private func fetchTitle() async throws -> String {
        "Done"
    }
}
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`Task {}` inside `@MainActor`|UI orchestration and view-model state updates|Still unstructured; keep a handle for cancellation|
|`await MainActor.run { ... }` from detached task|Rare bridge from genuinely independent work back to UI|Does not fix detached lifecycle, priority, or task-local loss|
|Service actor + `Task {}`|Shared mutable service state or serialized backend/client state|More explicit architecture; requires actor-aware API design|
|`async let` / task group|Work is scoped to an async function and must finish before returning|Requires an async context; not for fire-and-forget UI event entry points|
|`Task.detached`|Truly independent work that must not inherit actor context|You must explicitly manage cancellation, priority, Sendable captures, and actor hops|

---

## 6. Exercise

### Problem

Review code that uses detached tasks from a view model and explain why it may violate isolation or lifecycle expectations.

### Bad / naive version

```swift
struct Article: Sendable {
    let id: UUID
    let title: String
}

protocol ArticleService: Sendable {
    func fetchArticles() async throws -> [Article]
}

@MainActor
final class FeedViewModel {
    private let service: ArticleService

    var articles: [Article] = []
    var isLoading = false
    var errorMessage: String?

    init(service: ArticleService) {
        self.service = service
    }

    func refresh() {
        isLoading = true

        Task.detached {
            do {
                let articles = try await self.service.fetchArticles()

                await MainActor.run {
                    self.articles = articles
                    self.isLoading = false
                }
            } catch {
                await MainActor.run {
                    self.errorMessage = error.localizedDescription
                    self.isLoading = false
                }
            }
        }
    }
}
```

### What is wrong?

```text
1. The task is detached from the view model's actor context.
2. The task is detached from the view model's lifecycle.
3. There is no handle, so refresh work cannot be cancelled.
4. Repeated refreshes can publish stale results.
5. Task-local values such as request IDs or logging context are lost.
6. Priority is not inherited from the user action.
7. Strongly capturing self can keep the view model alive longer than expected.
8. MainActor.run fixes only the final UI mutation, not the overall design.
```

### Improved version

```swift
struct Article: Sendable {
    let id: UUID
    let title: String
}

protocol ArticleService: Sendable {
    func fetchArticles() async throws -> [Article]
}

@MainActor
final class FeedViewModel {
    private let service: ArticleService
    private var refreshTask: Task<Void, Never>?

    var articles: [Article] = []
    var isLoading = false
    var errorMessage: String?

    init(service: ArticleService) {
        self.service = service
    }

    func refresh() {
        refreshTask?.cancel()

        refreshTask = Task { [service] in
            isLoading = true
            errorMessage = nil

            do {
                let articles = try await service.fetchArticles()
                try Task.checkCancellation()

                self.articles = articles
                self.isLoading = false
            } catch is CancellationError {
                self.isLoading = false
            } catch {
                self.errorMessage = error.localizedDescription
                self.isLoading = false
            }
        }
    }

    func cancelRefresh() {
        refreshTask?.cancel()
        refreshTask = nil
        isLoading = false
    }
}
```

### Why this is better

The task is still unstructured, but it inherits `MainActor` isolation, task-local values, and priority from the calling context. The view model stores the task handle, so it can cancel stale work. UI state mutation remains on the main actor. The service remains the abstraction boundary for async work.

This is the pattern you usually want in UIKit/SwiftUI view models:

```text
ViewModel: @MainActor orchestration and UI state
Service/actor: async work, networking, persistence, shared mutable state
Task handle: cancellation and stale-work suppression
```

If fetching involves CPU-heavy work, do not solve that by blindly detaching the whole view-model operation. Move the CPU-heavy part behind a deliberate non-main isolation boundary, such as a service actor, a nonisolated async function with appropriate Swift 6.2 behavior, or an explicitly concurrent API where appropriate.

---

## 7. Production guidance

Use `Task.detached` in production when:

```text
- The work is genuinely independent from the caller's actor and lifecycle.
- You explicitly do not want actor-context inheritance.
- Captures are Sendable and safe to transfer across isolation domains.
- You keep the returned Task handle if cancellation or observation matters.
- You explicitly choose priority when priority matters.
- You explicitly restore any needed task-local/logging/tracing context.
```

Be careful when:

```text
- Calling from @MainActor types.
- Updating UI state after detached work.
- Capturing self.
- Starting work from view lifecycle methods.
- Starting work from cell reuse, search, scrolling, or navigation flows.
- Relying on task-local values for logging, tracing, auth, or request metadata.
```

Avoid when:

```text
- You only want to call an async function from synchronous code.
- You only want to avoid a compiler isolation diagnostic.
- You need cancellation tied to a parent task.
- You need task-local values or inherited priority.
- The work belongs to a view model's lifecycle.
- Structured concurrency can express the operation.
```

Debugging checklist:

```text
- Is this task detached because it truly must be independent?
- Who owns the Task handle?
- How is cancellation triggered?
- Can an older task publish stale results after a newer task?
- Are UI mutations main-actor isolated?
- Are all captured values Sendable or actor-isolated safely?
- Did we lose task-local request/logging context?
- Did we lose priority from the user action?
- Would async let, a task group, an actor, or Task {} be a better fit?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `Task.detached` runs code in the background. Use `MainActor.run` when you need to update UI.

This is incomplete and risky.

### Senior answer

> `Task.detached` creates independent unstructured work. It does not inherit actor isolation, priority, or task-local values. In a `@MainActor` view model, I would usually use `Task {}` and store the task handle so I can cancel stale work. I would only hop to `MainActor` from detached work if the detached work is truly independent.

### Staff-level answer

> I treat `Task.detached` as an explicit architecture boundary. It means this work is not part of the current actor, not part of structured lifetime, and not inheriting task metadata. Before accepting it, I want to know who owns cancellation, what priority it should run at, whether captures are Sendable, whether task-local tracing matters, and how stale results are suppressed. In app code, detached tasks should be rare; most UI workflows should use `@MainActor` orchestration plus services, actors, task groups, or explicit concurrency boundaries.

Staff-level questions to ask:

```text
What lifecycle owns this task?
What cancellation path stops it?
Does it need inherited task-local metadata?
Does it need user-initiated priority?
Is this escaping @MainActor for a valid reason or just silencing diagnostics?
Could a stale detached task update UI after newer state exists?
Would an actor or structured task group model the ownership better?
```

---

## 9. Interview-ready summary

`Task {}` and `Task.detached {}` are both unstructured tasks, but `Task {}` inherits context while `Task.detached` deliberately does not. `Task {}` inherits actor isolation, priority, and task-local values from where it is created, so it is usually appropriate for starting async UI work from a `@MainActor` view model, especially if you store the task handle for cancellation. `Task.detached` creates independent work outside the current actor and task context, so it is a footgun unless you explicitly manage cancellation, priority, task-local metadata, Sendable captures, and actor hops back to UI state.

---

## 10. Flashcards

Q: Is `Task {}` structured concurrency?  
A: No. `Task {}` creates an unstructured task, but it inherits context such as actor isolation, priority, and task-local values.

Q: What does `Task.detached {}` not inherit?  
A: Actor context, task-local values, and priority. It also does not participate in structured parent-child lifetime guarantees.

Q: Why does `Task.detached` fail inside a `@MainActor` view model when mutating a property?  
A: The detached closure is nonisolated, while the property is main-actor isolated. Cross-actor mutation must happen through an actor hop, not direct synchronous mutation.

Q: Does `MainActor.run` make detached tasks safe?  
A: It only makes the specific UI mutation happen on the main actor. It does not fix detached lifecycle, cancellation, priority, task-local, or stale-result issues.

Q: When is `Task.detached` appropriate?  
A: Rarely: when the work is truly independent from the current actor and lifecycle, captures are safe, and cancellation/priority/context are explicitly managed.

Q: What is the usual view-model pattern?  
A: Use `@MainActor` for UI state, start work with `Task {}`, store the task handle, cancel stale work, and put non-UI async work behind a service or actor.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D5 — Global actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D13 — Swift 6 strict concurrency migration]]
- [[D14 — Swift 6.2 approachable concurrency and new defaults]]
- [[F1 — Testing async and concurrent code]]

---

## 12. Sources

- Swift Senior/Staff Rubric, D11: detached tasks, task inheritance, and isolation loss.
- SE-0304: Structured Concurrency — task tree, unstructured tasks, context inheritance, detached tasks, task handles, cancellation. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md "swift-evolution/proposals/0304-structured-concurrency.md at main · swiftlang/swift-evolution · GitHub"))
- Apple WWDC21: “Explore structured concurrency in Swift” — task tree, child-task lifetime, cancellation, and structured concurrency guarantees. ([Apple Developer](https://developer.apple.com/wwdc21/10134 "Explore structured concurrency in Swift - WWDC21 - Videos - Apple Developer"))
- SE-0316: Global Actors — `MainActor`, global actor isolation, and main-thread UI state. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md "swift-evolution/proposals/0316-global-actors.md at main · swiftlang/swift-evolution · GitHub"))