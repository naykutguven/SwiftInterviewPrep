---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - structured-concurrency
  - task-hierarchy
  - priorities
---
## 0. Rubric snapshot

**Rubric expectation**

Understand cooperative cancellation, task-local propagation, parent-child relationships, and priority inheritance basics.

**Caveat**

Cancellation does **not** kill work automatically. It marks a task as cancelled; the task must observe cancellation through `Task.isCancelled`, `Task.checkCancellation()`, a cancellation-aware API, or a cancellation handler. Apple’s WWDC material states this directly: cancelling a parent cancels child tasks, but cancellation is cooperative and acting on it is your code’s responsibility. ([Apple Developer, "Beyond the Basics of Structured Concurrency"](https://developer.apple.com/videos/play/wwdc2023/10170/))

**You should be able to answer**

- Why is cancellation in Swift cooperative rather than preemptive?
- What does a child task inherit from its parent?

**You should be able to do**

- Design an image-loading pipeline that cancels stale work when cells are reused.

---

## 1. Core mental model

Swift tasks form a **task tree** when you use structured concurrency. `async let` and task groups create child tasks. Those child tasks are owned by the lexical scope that created them; they cannot outlive that scope. Apple describes structured tasks as being created by `async let` and task groups, while `Task {}` and `Task.detached {}` create unstructured tasks. ([Apple Developer, "Beyond the Basics of Structured Concurrency"](https://developer.apple.com/videos/play/wwdc2023/10170/))

The task tree gives Swift three important propagation channels:

```text
parent task
 ├─ cancellation signal flows downward
 ├─ priority flows downward / can be escalated
 └─ task-local values are visible to children
```

Cancellation is a **signal**, not a force-stop. Swift does not preemptively terminate a task because doing so would be unsafe: the task might be holding a lock, mutating shared state, writing a file, updating a database, or halfway through establishing an invariant. Instead, Swift marks the task as cancelled and lets code stop at safe points. The Swift book describes Swift’s cancellation model as cooperative. ([Swift.org, "Concurrency"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/))

Priority is also not a hard scheduling guarantee. It is a signal to the runtime about urgency. Child tasks inherit parent priority by default, and priority can be escalated when a higher-priority task waits on lower-priority work. Apple’s WWDC23 session explains that this helps avoid priority inversion. ([Apple Developer, "Beyond the Basics of Structured Concurrency"](https://developer.apple.com/videos/play/wwdc2023/10170/))

Task-local values are scoped context attached to a task hierarchy. They are useful for tracing, request IDs, logging metadata, correlation IDs, locale overrides, or feature-flag context. Apple describes task-local values as data associated with a task hierarchy, and notes that all tasks except detached tasks inherit task-local values from the current task. ([Apple Developer, "Beyond the Basics of Structured Concurrency"](https://developer.apple.com/videos/play/wwdc2023/10170/))

The key idea:

```text
Structured concurrency is ownership for async work.
Cancellation, priority, and task-local context only become predictable when work has an owner.
```

---

## 2. Essential mechanics

### 2.1 Structured child tasks are created by `async let` and task groups

```swift
func loadProfileScreen() async throws -> ProfileScreenModel {
    async let profile = fetchProfile()
    async let avatar = fetchAvatar()

    return try await ProfileScreenModel(
        profile: profile,
        avatar: avatar
    )
}
```

Here, `profile` and `avatar` are child tasks of the current task. The parent cannot return from `loadProfileScreen()` without resolving those child tasks.

Important consequences:

```text
- Parent cancellation propagates to child tasks.
- Child task lifetime is bounded by parent scope.
- Child tasks inherit priority.
- Child tasks inherit task-local values.
```

Apple’s TaskGroup documentation states that child tasks inherit the parent’s priority and task-local values, and their lifetime never exceeds the parent task’s lifetime. ([Apple Developer, "TaskGroup"](https://developer.apple.com/documentation/swift/taskgroup))

---

### 2.2 Cancellation must be observed

Bad mental model:

```text
task.cancel() == immediately stop execution
```

Correct mental model:

```text
task.cancel() == mark the task cancelled and run registered cancellation handlers
```

Use `Task.checkCancellation()` when cancellation should stop work by throwing:

```swift
func processLargeImageData(_ data: Data) async throws -> ProcessedImage {
    try Task.checkCancellation()

    let decoded = try decode(data)

    try Task.checkCancellation()

    let resized = try resize(decoded)

    try Task.checkCancellation()

    return resized
}
```

Use `Task.isCancelled` when you want to return partial work or silently stop:

```swift
func buildSearchIndex(items: [Item]) async -> SearchIndex {
    var index = SearchIndex()

    for item in items {
        if Task.isCancelled {
            return index
        }

        index.insert(item)
    }

    return index
}
```

---

### 2.3 Cancellation handlers are for suspended work or external resources

Polling is not enough when the task is suspended and no code is running. That is when `withTaskCancellationHandler` matters. Apple specifically calls out cancellation handlers for cases like an `AsyncSequence` waiting for the next value. ([Apple Developer, "Beyond the Basics of Structured Concurrency"](https://developer.apple.com/videos/play/wwdc2023/10170/))

```swift
func nextEvent() async -> Event? {
    await withTaskCancellationHandler {
        await eventSource.next()
    } onCancel: {
        eventSource.cancel()
    }
}
```

But be careful: the cancellation handler can run concurrently with the operation. Apple notes that if the handler and main operation share mutable state, that state must be synchronized. ([Apple Developer, "Beyond the Basics of Structured Concurrency"](https://developer.apple.com/videos/play/wwdc2023/10170/))

---

### 2.4 Priorities are inherited and may be escalated

```swift
Task(priority: .userInitiated) {
    await withTaskGroup(of: Void.self) { group in
        group.addTask {
            await loadVisibleCellImages()
        }

        group.addTask {
            await precomputeLayout()
        }
    }
}
```

The child tasks start with the parent’s priority unless you explicitly choose another priority. Priority is a runtime scheduling hint, not a correctness mechanism.

Apple’s WWDC23 session explains that child tasks inherit priority by default, and if a higher-priority task awaits lower-priority child work, the child work can be escalated to avoid priority inversion. ([Apple Developer, "Beyond the Basics of Structured Concurrency"](https://developer.apple.com/videos/play/wwdc2023/10170/))

---

### 2.5 Task-local values flow through the task hierarchy

```swift
enum TraceContext {
    @TaskLocal static var requestID: UUID?
}

func handleRequest(id: UUID) async throws {
    try await TraceContext.$requestID.withValue(id) {
        try await withThrowingTaskGroup(of: Void.self) { group in
            group.addTask {
                await log("Loading user")
                // TraceContext.requestID is visible here.
            }

            group.addTask {
                await log("Loading permissions")
                // TraceContext.requestID is visible here too.
            }

            try await group.waitForAll()
        }
    }
}
```

This is a good use of task-local values: implicit contextual metadata that should travel through the task tree.

This is a bad use:

```swift
enum Dependencies {
    @TaskLocal static var apiClient: APIClient?
}
```

Task-local values should usually not be your main dependency injection system. They are easy to forget to bind and make dependencies invisible at the call site. Use them for cross-cutting context, not ordinary required dependencies.

---

### 2.6 `Task {}` is not a child task

This is a common trap:

```swift
func load() async throws -> Data {
    let task = Task {
        try await fetchData()
    }

    return try await task.value
}
```

This creates an unstructured task. It is not a child task in the structured-concurrency sense. If the caller of `load()` is cancelled while awaiting `task.value`, the inner task is not automatically cancelled just because the caller was cancelled.

Usually, the better design is:

```swift
func load() async throws -> Data {
    try await fetchData()
}
```

If you genuinely need an unstructured task, you must own its lifecycle explicitly:

```swift
func load() async throws -> Data {
    let task = Task {
        try await fetchData()
    }

    return try await withTaskCancellationHandler {
        try await task.value
    } onCancel: {
        task.cancel()
    }
}
```

---

## 3. Common traps and misconceptions

### Trap 1: Thinking cancellation kills the task

Bad:

```swift
func expensiveLoop() async {
    for item in hugeArray {
        process(item)
    }
}
```

If the task is cancelled, this loop still keeps going.

Better:

```swift
func expensiveLoop() async throws {
    for item in hugeArray {
        try Task.checkCancellation()
        process(item)
    }
}
```

Cancellation is cooperative. The task must reach a cancellation-aware point.

---

### Trap 2: Treating `Task {}` as structured child work

Bad:

```swift
func loadBoth() async throws -> (Data, Data) {
    let a = Task { try await fetchA() }
    let b = Task { try await fetchB() }

    return try await (a.value, b.value)
}
```

Better:

```swift
func loadBoth() async throws -> (Data, Data) {
    async let a = fetchA()
    async let b = fetchB()

    return try await (a, b)
}
```

`async let` gives you structured child tasks. The parent scope owns the lifetime. `Task {}` creates unstructured work.

---

### Trap 3: Forgetting that cell reuse is not task cancellation

Bad:

```swift
final class AvatarCell: UICollectionViewCell {
    func configure(url: URL, pipeline: ImageDataPipeline) {
        Task {
            let data = try? await pipeline.data(for: url)
            imageView.image = data.flatMap(UIImage.init(data:))
        }
    }
}
```

Problems:

```text
- The task is not cancelled when the cell is reused.
- The old request can complete later and update the wrong reused cell.
- The task strongly captures the cell.
- There is no cancellation check before UI mutation.
```

Better: store a task handle and cancel it from `prepareForReuse()`.

---

### Trap 4: Using priority as a correctness tool

Bad:

```swift
Task(priority: .background) {
    await mutateSharedState()
}

Task(priority: .userInitiated) {
    await readSharedState()
}
```

Priority does not create ordering. It does not guarantee which task runs first. Use actors, locks, structured awaiting, or explicit state machines for correctness.

---

### Trap 5: Cancelling shared work too aggressively

If multiple cells request the same URL, cancelling one cell’s task should not necessarily cancel the shared network fetch for all other cells. A mature image pipeline separates:

```text
consumer cancellation
vs
producer cancellation
```

A cell disappearing should stop that cell from awaiting and updating UI. Whether the underlying fetch is cancelled depends on whether anyone else still needs it.

This distinction is staff-level API design territory.

---

## 4. Direct answers to rubric questions

### Q1. Why is cancellation in Swift cooperative rather than preemptive?

Cancellation is cooperative because Swift cannot safely stop arbitrary code at an arbitrary instruction. A task might be holding a lock, updating shared state, writing to disk, mutating actor-isolated state, or maintaining an invariant. Preemptively killing it could leave the program corrupted.

Swift cancellation instead marks the task as cancelled. Code observes that signal at safe points using `Task.isCancelled`, `Task.checkCancellation()`, cancellation-aware APIs, or `withTaskCancellationHandler`. Apple’s documentation and WWDC material describe cancellation as cooperative: cancellation marks the task, and the code must respond. ([Swift.org, "Concurrency"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/))

Interview version:

> Swift cancellation is cooperative because the runtime cannot safely terminate arbitrary code without breaking invariants. Cancellation marks the task as cancelled and may run cancellation handlers, but the task must observe that state at safe points. In production, that means checking cancellation before expensive work, after suspension points, before side effects, and when bridging to callback or resource-based APIs.

---

### Q2. What does a child task inherit from its parent?

A structured child task inherits:

```text
- cancellation relationship
- priority
- task-local values
- bounded lifetime under the parent scope
```

A parent task’s cancellation propagates downward to its child tasks. Child tasks inherit the parent’s priority by default. Task-local values are visible through the task hierarchy. The child task’s lifetime is structurally bounded: the parent scope must wait for child tasks to complete before the scope exits. Apple’s TaskGroup documentation describes this lifetime and inheritance relationship. ([Apple Developer, "TaskGroup"](https://developer.apple.com/documentation/swift/taskgroup))

Important nuance:

```text
Cancellation flows downward, not upward by default.
A child being cancelled does not automatically cancel the parent.
A child throwing may cancel siblings depending on the primitive, for example throwing task groups.
```

Also:

```text
Task {} is not a structured child task.
Task.detached {} is even more independent and does not inherit task-local values.
```

Apple states that detached tasks do not inherit the parent task’s priority or task-local storage and require manual handling for cancellation-like behavior. ([Apple Developer, "detached(name:priority:operation:)"](https://developer.apple.com/documentation/swift/task/detached%28name%3apriority%3aoperation%3a%29-795w1))

Interview version:

> A structured child task inherits the parent’s priority and task-local values, and it participates in the parent-child cancellation relationship. Its lifetime is bounded by the parent scope. Parent cancellation propagates downward, but cancellation does not preempt execution; the child still has to observe cancellation. I would not describe `Task {}` as a child task; it is unstructured and needs explicit lifecycle management.

---

## 5. Code probe

D2 has no rubric code probe.

### Minimal example

```swift
func loadScreen() async throws -> ScreenModel {
    async let profile = fetchProfile()
    async let recommendations = fetchRecommendations()

    return try await ScreenModel(
        profile: profile,
        recommendations: recommendations
    )
}
```

What happens semantically:

```text
- fetchProfile() and fetchRecommendations() run as child tasks.
- If loadScreen() is cancelled, both child tasks are marked cancelled.
- The function cannot return until both child tasks are resolved.
- If either child observes cancellation and throws, the error propagates through the await.
```

### Counterexample

```swift
func loadScreen() async throws -> ScreenModel {
    let profileTask = Task {
        try await fetchProfile()
    }

    let recommendationsTask = Task {
        try await fetchRecommendations()
    }

    return try await ScreenModel(
        profile: profileTask.value,
        recommendations: recommendationsTask.value
    )
}
```

What is wrong:

```text
- These are unstructured tasks.
- They are not child tasks of loadScreen().
- Cancellation of the caller does not automatically cancel them.
- Their lifetime is harder to reason about.
```

### Production example

```swift
func loadVisibleContent() async throws -> VisibleContent {
    try Task.checkCancellation()

    async let user = userService.currentUser()
    async let feed = feedService.visibleFeed()
    async let notifications = notificationService.unreadCount()

    let model = try await VisibleContent(
        user: user,
        feed: feed,
        unreadNotifications: notifications
    )

    try Task.checkCancellation()

    return model
}
```

Why this is correct:

```text
- Independent async operations are parallelized.
- The parent scope owns the child tasks.
- Cancellation propagates naturally.
- Cancellation is checked before returning data that may already be stale.
```

---

## 6. Exercise

### Problem

Design an image-loading pipeline that cancels stale work when cells are reused.

---

### Bad / naive version

```swift
final class AvatarCell: UICollectionViewCell {
    private let imageView = UIImageView()

    func configure(url: URL, pipeline: ImageDataPipeline) {
        imageView.image = nil

        Task {
            let data = try? await pipeline.data(for: url)
            imageView.image = data.flatMap(UIImage.init(data:))
        }
    }
}
```

### What is wrong?

```text
1. The task is unstructured and not stored.
2. The cell cannot cancel the task in prepareForReuse().
3. A reused cell can be updated by an old request.
4. The task strongly captures the cell.
5. There is no cancellation check before UI mutation.
6. Failures and cancellation are collapsed into nil.
```

The practical bug:

```text
Cell shows user A.
Cell starts loading A's avatar.
Cell is reused for user B.
A's request finishes late.
Cell now incorrectly displays A's avatar while representing B.
```

---

### Improved version

```swift
import UIKit

actor ImageDataPipeline {
    enum ImageError: Error {
        case invalidResponse
    }

    private var cache: [URL: Data] = [:]

    func data(for url: URL) async throws -> Data {
        if let cached = cache[url] {
            return cached
        }

        try Task.checkCancellation()

        let (data, response) = try await URLSession.shared.data(from: url)

        try Task.checkCancellation()

        guard let httpResponse = response as? HTTPURLResponse,
              (200..<300).contains(httpResponse.statusCode)
        else {
            throw ImageError.invalidResponse
        }

        cache[url] = data
        return data
    }
}

@MainActor
final class AvatarCell: UICollectionViewCell {
    private let imageView = UIImageView()

    private var representedID: UUID?
    private var imageTask: Task<Void, Never>?

    override func prepareForReuse() {
        super.prepareForReuse()

        representedID = nil
        imageTask?.cancel()
        imageTask = nil
        imageView.image = nil
    }

    func configure(
        representedID: UUID,
        imageURL: URL,
        pipeline: ImageDataPipeline
    ) {
        self.representedID = representedID
        imageView.image = nil

        imageTask?.cancel()

        imageTask = Task { [weak self] in
            do {
                let data = try await pipeline.data(for: imageURL)

                try Task.checkCancellation()

                guard let image = UIImage(data: data) else {
                    return
                }

                try Task.checkCancellation()

                guard let self, self.representedID == representedID else {
                    return
                }

                self.imageView.image = image
            } catch is CancellationError {
                // Expected path for reuse, fast scrolling, or disappearing cells.
            } catch {
                guard let self, self.representedID == representedID else {
                    return
                }

                self.imageView.image = UIImage(systemName: "person.crop.circle.badge.exclamationmark")
            }
        }
    }
}
```

### Why this is better

```text
- The cell owns the UI-level task.
- prepareForReuse() cancels stale work.
- The representedID check prevents stale completion from updating a reused cell.
- The pipeline is actor-isolated, so its cache is protected.
- The pipeline returns Data instead of UIImage, avoiding cross-actor UIKit sendability issues.
- Cancellation is checked before expensive/meaningful steps and before UI mutation.
```

Important nuance: the cell’s `Task {}` is still unstructured relative to Swift’s task tree. That is acceptable here because the cell explicitly owns and cancels the task handle. The lifecycle owner is the cell, not a parent async function.

---

### More production-grade version

For a serious image pipeline, add these:

```text
- memory cache
- disk cache
- request coalescing for identical URLs
- priority based on visibility
- cancellation-aware decoding
- placeholder policy
- retry policy
- stale-result suppression
- metrics and logging
```

But be careful with request coalescing:

```text
If 5 cells request the same URL and 1 cell is reused,
cancelling that cell should cancel that cell's await,
not necessarily the shared underlying network request.
```

That distinction separates a decent implementation from a production-grade one.

---

## 7. Production guidance

Use this in production when:

```text
- Work has a clear owner.
- Child work should not outlive the parent operation.
- UI work should be cancelled when the view/cell/task disappears.
- Parallel work should share cancellation, priority, and tracing context.
- You want request IDs, trace IDs, or logging metadata to flow through async calls.
```

Be careful when:

```text
- You introduce Task {} inside async functions.
- You bridge callback APIs.
- You use cancellation handlers with shared mutable state.
- You use task-local values for required dependencies.
- You share one underlying producer among many cancellable consumers.
- You use priority to imply ordering.
```

Avoid when:

```text
- You need deterministic ordering; use explicit awaiting/state machines.
- You need synchronization; use actors, locks, or immutable state.
- You are tempted to use Task.detached just to “move work off the main thread.”
- You are hiding unstructured work inside APIs that appear structured.
```

Debugging checklist:

```text
Cancellation:
- Who owns this task?
- Is it structured or unstructured?
- Who calls cancel()?
- Does the task check cancellation?
- Is the task suspended inside a cancellation-aware API?
- Is there a cancellation handler for external resources?

Stale UI:
- Can this task outlive the cell/view/controller?
- Is there an identity check before applying the result?
- Is cancellation handled as an expected path?

Priority:
- Is priority being used as a hint or incorrectly as ordering?
- Could a high-priority task be waiting on low-priority work?
- Is background prefetch accidentally running as userInitiated?

Task-local values:
- Is this tracing/logging context or a hidden dependency?
- Does detached work need explicit context propagation?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Cancellation means you call `cancel()` on a task. Then the task stops.

This is wrong or at least incomplete.

### Senior answer

> Swift cancellation is cooperative. Cancelling a task marks it as cancelled and child tasks receive that cancellation signal, but code must observe it. I use structured concurrency when the child work should be owned by the parent, and I check cancellation before expensive work, after suspension points, and before committing results to UI or storage.

### Staff-level answer

> I treat cancellation as part of API design, not cleanup syntax. The key question is ownership: who owns the work, and what result becomes invalid when that owner disappears? For structured work, I use `async let` or task groups so cancellation, priority, and task-local context flow naturally. For UI lifecycle work like cell image loading, I store task handles and cancel explicitly because cells are not task-tree parents. For shared producers, I separate consumer cancellation from producer cancellation so one disappearing view does not accidentally cancel work still needed elsewhere.

Staff-level questions to ask:

```text
Who owns this async work?
Should this task be structured or explicitly unstructured?
What should happen if the parent disappears?
Is cancellation supposed to stop the producer, the consumer, or both?
Can stale results still commit after cancellation?
Are we using task-local values for observability or hiding dependencies?
Could priority inversion happen here?
Do we need request coalescing, and how does it interact with cancellation?
```

---

## 9. Interview-ready summary

Swift’s task hierarchy is the ownership model behind structured concurrency. Child tasks created with `async let` or task groups are bounded by the parent scope and inherit cancellation, priority, and task-local values. Cancellation is cooperative: cancelling a task marks it as cancelled, but the task must check cancellation or use cancellation-aware APIs to actually stop. In production, I use structured concurrency when child work belongs to a parent operation, and explicit task handles when lifecycle ownership is external, such as reusable cells or view models. Priorities are scheduling hints, not correctness tools, and task-local values are best for tracing/logging context rather than hidden dependencies.

---

## 10. Flashcards

Q: What creates structured child tasks in Swift?  
A: `async let` and task groups.

Q: Is `Task {}` a child task?  
A: No. It creates an unstructured task. It may inherit some context, but it is not structurally owned by the surrounding scope.

Q: Is `Task.detached {}` a child task?  
A: No. It is detached from the current task hierarchy and does not inherit task-local values or priority from the current task.

Q: What does cancellation do?  
A: It marks a task as cancelled and runs cancellation handlers. It does not forcibly stop execution.

Q: How should throwing code observe cancellation?  
A: Use `try Task.checkCancellation()`.

Q: How should non-throwing code observe cancellation?  
A: Use `Task.isCancelled` and return early or produce a partial result.

Q: What does a child task inherit from its parent?  
A: Priority, task-local values, cancellation relationship, and bounded lifetime.

Q: Does cancellation propagate upward?  
A: Not by default. Parent cancellation propagates downward. Child failure may affect siblings depending on the primitive, such as throwing task groups.

Q: Why is cancellation cooperative?  
A: Preemptively killing arbitrary code could break invariants, leak resources, or leave shared state inconsistent.

Q: What is priority inversion?  
A: A high-priority task is blocked waiting for lower-priority work.

Q: Are task priorities correctness guarantees?  
A: No. They are scheduling hints.

Q: What are task-local values good for?  
A: Request IDs, trace IDs, logging metadata, and other cross-cutting context.

Q: What are task-local values bad for?  
A: Required dependencies or hidden mutable application state.

Q: Why do image-loading cells need explicit task cancellation?  
A: Cell reuse is not part of Swift’s task hierarchy, so the cell must cancel stale work manually.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D3 — `async let`, task groups, and choosing the right concurrency primitive]]
- [[D10 — `AsyncSequence`, streams, and backpressure-aware design]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D12 — Synchronization library, mutexes, and choosing between actors and locks]]
- [[F1 — Testing async and concurrent code]]
- [[F2 — Diagnostics and LLDB]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — D2 task hierarchy, cancellation, and priorities.
- Swift.org. "Concurrency." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- Apple Developer. "TaskGroup." Apple Developer Documentation. https://developer.apple.com/documentation/swift/taskgroup
- Apple Developer. "detached(name:priority:operation:)." Apple Developer Documentation. https://developer.apple.com/documentation/swift/task/detached%28name%3apriority%3aoperation%3a%29-795w1
- Apple Developer. "Beyond the Basics of Structured Concurrency." WWDC23. https://developer.apple.com/videos/play/wwdc2023/10170/
- GitHub. "Structured Concurrency." Swift Evolution SE-0304. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md
- GitHub. "Task-Local Values." Swift Evolution SE-0311. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0311-task-locals.md
