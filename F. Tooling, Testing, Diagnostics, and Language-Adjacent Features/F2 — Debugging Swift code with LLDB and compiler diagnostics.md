---
tags:
  - swift
  - ios
  - interview-prep
  - tooling
  - diagnostics
  - lldb
  - concurrency
  - debugging
---
## 0. Rubric snapshot

**Rubric expectation**

Understand how to debug Swift code using compiler diagnostics, fix-its, LLDB, async backtraces, task context, named-task debugging, memory graph clues, and basic debugger workflows. The rubric explicitly calls out that senior engineers should treat diagnostics as part of Swift’s type/isolation model, not as noise.

**Caveats**

Compiler errors in Swift are often not merely syntax problems. They are frequently telling you that one of these models has been violated:

```text
type system
ownership/lifetime model
actor isolation model
Sendable transfer model
generic constraints
result-builder transformation
```

**You should be able to answer**

- How do you approach a 20-line generic or result-builder compiler error without thrashing?
- How do async backtraces, task context, and task names help debug a concurrent hang or stale-state bug?
- What debugging signals suggest a retain cycle versus a data race versus a stale-state bug?

**You should be able to do**

- Given an isolation diagnostic about crossing actor boundaries with a non-sendable type, explain how to narrow the real issue.

---

## 1. Core mental model

Swift debugging is not just “pause and inspect variables.” Swift gives you several layers of evidence, and each layer answers a different question.

The compiler answers:

```text
Is this program legal according to Swift’s type, ownership, and isolation rules?
```

LLDB answers:

```text
What is the runtime doing at this stopped point?
```

The memory graph answers:

```text
Why is this object still alive?
```

Concurrency debugging answers:

```text
Which task is running, suspended, blocked, cancelled, or producing stale work?
```

The key staff-level habit is to avoid treating these tools as separate tricks. A good Swift debugger maps every symptom back to the model it violates: type relationship, lifetime, isolation, ordering, cancellation, or state ownership.

The key idea:

```text
Diagnostic first, model second, runtime evidence third, fix last.
```

Swift compiler diagnostics are often better understood as “proof failures.” When the compiler says a value is non-`Sendable`, or a result-builder expression cannot infer a type, or an actor-isolated member cannot be accessed, the compiler is saying: “I cannot prove the invariant you need here.” Your job is not to silence it. Your job is to identify which invariant the program is failing to express.

---

## 2. Essential mechanics

### 2.1 Read diagnostics from the bottom of the model, not the top of the error

Long Swift errors often contain multiple symptoms. The first highlighted line is not always the root cause.

For generics, look for:

```text
Which type parameter failed?
Which associated type relationship is missing?
Was type information erased too early?
Is the compiler trying to infer too much from one expression?
```

Example:

```swift
protocol Repository {
    associatedtype Entity
    func fetch() async throws -> [Entity]
}

func load<R: Repository>(_ repository: R) async throws {
    let values: [User] = try await repository.fetch()
}
```

This fails unless the compiler knows:

```swift
R.Entity == User
```

Better:

```swift
func load<R: Repository>(_ repository: R) async throws where R.Entity == User {
    let values = try await repository.fetch()
}
```

The diagnostic is not “Swift is bad at generics.” The real issue is that the API did not express the same-type relationship.

---

### 2.2 Result-builder errors: shrink the builder until the real type mismatch appears

Result builders transform syntax. A SwiftUI `body` error may be reported on a huge `VStack`, but the real problem is often one branch returning a different view shape, a missing `return` in helper code, an availability branch, or an expression that is not a `View`.

Bad debugging pattern:

```swift
var body: some View {
    VStack {
        header

        if isLoading {
            ProgressView()
        } else {
            users.map { user in
                Text(user.name)
            }
        }
    }
}
```

The `else` branch returns `[Text]`, not a view.

Better:

```swift
var body: some View {
    VStack {
        header

        if isLoading {
            ProgressView()
        } else {
            ForEach(users) { user in
                Text(user.name)
            }
        }
    }
}
```

Debugging approach:

```text
1. Extract suspicious branches into small computed properties.
2. Add explicit return types.
3. Replace complex expressions with placeholders.
4. Reintroduce lines one by one.
5. Find the first expression whose type is not what the builder expects.
```

---

### 2.3 LLDB basics: inspect state without changing the program first

LLDB is Xcode’s debugger engine. The LLDB docs describe it as the debugger used by Xcode on macOS and Apple platform simulators/devices, and Apple’s LLDB guide shows the standard workflow of inspecting threads, backtraces, frames, variables, and expressions. ([LLDB, "LLDB"](https://lldb.llvm.org/))

Core commands:

```lldb
po object
p value
expr someExpression
frame variable
thread list
thread backtrace
thread backtrace all
breakpoint set --name functionName
breakpoint set --file File.swift --line 42
watchpoint set variable someVariable
```

Use `frame variable` before `expr` when possible:

```lldb
(lldb) frame variable self
(lldb) frame variable viewModel.state
```

`frame variable` is safer for inspection because it does not run arbitrary Swift code. `expr` can call methods, trigger computed properties, mutate state, and sometimes change the bug.

Use `expr` when you intentionally need evaluation:

```lldb
(lldb) expr viewModel.debugDescription()
(lldb) expr await someActor.currentState()
```

---

### 2.4 Async debugging: follow tasks, not just threads

In Swift concurrency, “which thread am I on?” is often the wrong first question. Tasks can suspend and resume on different threads. Xcode 26 improves Swift concurrency debugging by following execution into asynchronous functions even across thread switches, showing task IDs, and displaying easier-to-read representations of tasks, task groups, and actors. ([Apple Developer, "What’s new in Xcode 26"](https://developer.apple.com/videos/play/wwdc2025/247/))

Modern Swift also supports named tasks. `Task.init(name:priority:operation:)` exists for creating named tasks, and Swift Evolution SE-0469 introduced task names specifically to help tools such as debuggers, profilers, and task dumpers identify asynchronous work. ([Apple Developer, "Task"](https://developer.apple.com/documentation/swift/task))

Example:

```swift
final class SearchViewModel {
    private var searchTask: Task<Void, Never>?

    func search(query: String) {
        searchTask?.cancel()

        searchTask = Task(name: "Search query: \(query)") {
            try? await Task.sleep(for: .milliseconds(300))

            guard !Task.isCancelled else { return }

            let results = try? await performSearch(query)
            await MainActor.run {
                self.apply(results, for: query)
            }
        }
    }
}
```

Task names help answer:

```text
Which logical operation is this suspended task?
Is this an old search task or the latest one?
Is this task still running after the screen disappeared?
Is this task blocked waiting for an actor, network response, or cancellation?
```

---

### 2.5 Runtime tools map to bug categories

Use the right tool for the suspected bug.

|Bug category|Best first evidence|Common signal|
|---|---|---|
|Type-system bug|Compiler diagnostic|Constraint, associated type, generic, `some`/`any` mismatch|
|Isolation bug|Swift 6 diagnostic|`actor-isolated`, `non-Sendable`, `@Sendable`, crossing actor boundary|
|Retain cycle|Memory graph / `deinit` not called|Object still alive, closure context retaining `self`|
|Data race|Thread Sanitizer / strict concurrency|Unsynchronized mutable state from multiple threads/tasks|
|Stale-state bug|Logs, task IDs, breakpoints, sequence numbers|Old async result overwrites newer UI state|
|Hang/deadlock|Backtraces, task context, locks/actors|Waiting cycle, blocked executor, actor reentrancy issue|

Apple’s memory debugging material highlights retain cycles as a common Swift leak pattern and explains that closure captures can create strong reference cycles. Apple’s Xcode runtime diagnostics also include Thread Sanitizer for detecting race conditions. ([Apple Developer, "Detect and diagnose memory issues"](https://developer.apple.com/videos/play/wwdc2021/10180/))

---

## 3. Common traps and misconceptions

### Trap 1: Treating fix-its as design advice

Fix-its are often syntactic repairs, not architectural recommendations.

Bad:

```swift
final class Cache {
    var storage: [String: Int] = [:]
}

// Bad if this is just silencing a Swift 6 concurrency diagnostic.
extension Cache: @unchecked Sendable {}
```

Better:

```swift
actor Cache {
    private var storage: [String: Int] = [:]

    func value(for key: String) -> Int? {
        storage[key]
    }

    func setValue(_ value: Int, for key: String) {
        storage[key] = value
    }
}
```

`@unchecked Sendable` says: “I have manually proven this is safe.” It does not make mutable shared state safe.

---

### Trap 2: Debugging SwiftUI builder errors at full size

If a `body` has 150 lines, the compiler has too much expression context. Shrink the expression.

Bad:

```swift
var body: some View {
    VStack {
        // 150 lines of nested conditional UI
    }
}
```

Better:

```swift
var body: some View {
    content
}

@ViewBuilder
private var content: some View {
    if isLoading {
        loadingView
    } else {
        loadedView
    }
}
```

Then add explicit types where useful:

```swift
private var title: some View {
    Text(viewModel.title)
}
```

---

### Trap 3: Confusing data races with stale-state bugs

A data race is unsynchronized concurrent access to mutable memory.

A stale-state bug is logically outdated work applying successfully.

This can be stale-state-safe but still not data-race-safe:

```swift
final class SearchState {
    var latestQueryID = UUID()
}
```

This can be data-race-safe but still stale-state-buggy:

```swift
actor SearchStore {
    private var results: [ResultItem] = []

    func apply(_ results: [ResultItem]) {
        self.results = results
    }
}
```

The actor serializes mutation, but it does not know whether the result belongs to the latest request. You still need logical ordering:

```swift
actor SearchStore {
    private var latestRequestID: UUID?

    func startRequest() -> UUID {
        let id = UUID()
        latestRequestID = id
        return id
    }

    func apply(_ results: [ResultItem], requestID: UUID) {
        guard requestID == latestRequestID else { return }
        self.results = results
    }
}
```

---

### Trap 4: Using `po` and `expr` without realizing they can execute code

This can run code:

```lldb
(lldb) po viewModel.title
```

If `title` is computed, it may execute arbitrary Swift.

Safer first pass:

```lldb
(lldb) frame variable viewModel
```

Then use `expr` intentionally.

---

### Trap 5: Looking only at threads in Swift concurrency

Threads are implementation detail. Tasks are the logical units of work.

Bad mental model:

```text
This bug happened because code resumed on another thread.
```

Better mental model:

```text
This task suspended. While it was suspended, another task changed the relevant state. When the first task resumed, its earlier assumptions were stale.
```

---

## 4. Direct answers to rubric questions

### Q1. How do you approach a 20-line generic or result-builder compiler error without thrashing?

First, reduce the expression until the real type relationship appears. Do not randomly apply fix-its.

For generic errors:

```text
1. Identify the generic parameter involved.
2. Identify the protocol requirement or associated type involved.
3. Add explicit intermediate types.
4. Add missing where clauses or same-type constraints.
5. Avoid erasing to any too early.
```

For result-builder errors:

```text
1. Replace branches with placeholders.
2. Extract subviews/building blocks.
3. Add explicit return types.
4. Check each branch returns a builder-compatible type.
5. Reintroduce complexity gradually.
```

Interview version:

> I don’t try to fix a giant Swift error at the reported line immediately. I first reduce the expression and make the compiler’s hidden assumptions explicit. For generics, I look for the missing same-type relationship or associated type constraint. For result builders, I isolate branches and add explicit return types until the actual non-builder expression appears. The goal is to find the model mismatch, not to chase surface syntax.

---

### Q2. How do async backtraces, task context, and task names help debug a concurrent hang or stale-state bug?

They let you debug the logical unit of work instead of only the physical thread.

Async code can suspend and resume later. A thread backtrace alone may not explain which user action, request, or task produced the current state. Task IDs and task names connect the runtime frame to the logical operation.

Example:

```swift
Task(name: "Avatar load userID=\(userID)") {
    let image = try await imageService.loadAvatar(userID: userID)
    await MainActor.run {
        self.avatar = image
    }
}
```

This helps you distinguish:

```text
Avatar load userID=123
Avatar load userID=456
Search query: "swift"
Search query: "swift concurrency"
```

That matters when old work is still alive and overwriting new state.

Interview version:

> In Swift concurrency I care about tasks, not just threads. A task can suspend and resume on a different thread, so a traditional thread-only view can hide the logical operation. Async backtraces, task IDs, and task names let me identify which request or workflow is currently running, suspended, or applying state. For stale-state bugs, I usually pair that with cancellation checks, request IDs, and breakpoints at the state mutation site.

---

### Q3. What debugging signals suggest a retain cycle versus a data race versus a stale-state bug?

A retain cycle usually means an object never deinitializes after its owner is gone. The memory graph shows a strong reference path, often through a closure context, task, timer, delegate, or callback.

A data race usually means multiple threads/tasks access the same mutable memory without synchronization. Signals include Thread Sanitizer reports, nondeterministic crashes, corrupted collections, or Swift 6 strict-concurrency diagnostics around non-`Sendable` shared mutable state.

A stale-state bug usually has no memory corruption and no leak. Instead, an older async operation completes after a newer operation and applies outdated state.

Interview version:

> I separate the symptom categories. If `deinit` never runs and the memory graph shows a strong path through a closure or task, I suspect a retain cycle. If Thread Sanitizer reports unsynchronized access or mutable state is touched across isolation domains, I suspect a data race. If everything is memory-safe but an old request overwrites newer UI state, I suspect stale-state logic and look for missing cancellation, sequence IDs, or actor reentrancy issues.

---

## 5. Code probe

The rubric has no explicit code probe for F2, so use this diagnostic probe instead.

Given:

```swift
import Foundation

final class ImageCache {
    var storage: [String: Data] = [:]
}

actor ImageLoader {
    private let cache = ImageCache()

    func cacheForDebugging() -> ImageCache {
        cache
    }
}

func use(loader: ImageLoader) async {
    let cache = await loader.cacheForDebugging()
    cache.storage["avatar"] = Data()
}
```

### What happens?

With Swift 6 language mode and complete strict concurrency checking, Swift 6.2.1 reports:

```text
/tmp/F2.swift:16:30: error: non-Sendable 'ImageCache'-typed result can not be returned from actor-isolated instance method 'cacheForDebugging()' to nonisolated context
 1 | import Foundation
 2 | 
 3 | final class ImageCache {
   |             `- note: class 'ImageCache' does not conform to the 'Sendable' protocol
 4 |     var storage: [String: Data] = [:]
 5 | }
   :
14 | 
15 | func use(loader: ImageLoader) async {
16 |     let cache = await loader.cacheForDebugging()
   |                              `- error: non-Sendable 'ImageCache'-typed result can not be returned from actor-isolated instance method 'cacheForDebugging()' to nonisolated context
17 |     cache.storage["avatar"] = Data()
18 | }
```

### Why?

`ImageCache` is a mutable reference type:

```swift
final class ImageCache {
    var storage: [String: Data] = [:]
}
```

The actor owns it:

```swift
actor ImageLoader {
    private let cache = ImageCache()
}
```

Returning it across the actor boundary would let nonisolated code mutate the same reference outside the actor:

```swift
let cache = await loader.cacheForDebugging()
cache.storage["avatar"] = Data()
```

That defeats the actor’s isolation.

```text
ImageLoader actor
 └── isolated cache: ImageCache
      └── mutable storage

await loader.cacheForDebugging()
        ↓
nonisolated caller receives same mutable reference
        ↓
actor no longer controls all access to cache.storage
```

The diagnostic is not saying “classes are forbidden.” It is saying:

```text
A mutable, non-Sendable reference cannot safely cross this isolation boundary.
```

Swift’s concurrency diagnostics document this general category: accessing actor-isolated state from outside the actor can create data races, and non-`Sendable` values generally cannot be safely shared across concurrency domains. ([Swift.org, "Calling an Actor-Isolated Method from a Synchronous Nonisolated Context"](https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/))

### Fix or redesign

#### Option A — Keep the mutable cache inside the actor

```swift
import Foundation

actor ImageLoader {
    private var storage: [String: Data] = [:]

    func data(for key: String) -> Data? {
        storage[key]
    }

    func setData(_ data: Data, for key: String) {
        storage[key] = data
    }

    func snapshotForDebugging() -> [String: Data] {
        storage
    }
}

func use(loader: ImageLoader) async {
    await loader.setData(Data(), for: "avatar")
}
```

### Why this fix is correct

The mutable state remains actor-isolated. Callers can interact with it only through actor methods.

The debug snapshot is a value copy of the dictionary’s value-semantic interface. The actor does not leak the mutable reference that owns the state.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Keep cache inside the actor|Shared mutable cache with async access|Requires `await`; actor can become a bottleneck if overused|
|Return immutable value snapshots|Debugging, UI display, metrics|Snapshot may become stale immediately|
|Make the type a value type|Cache state is small or naturally copyable|Copies may be expensive for large state|
|Use a lock-protected `final class` with `@unchecked Sendable`|Low-level performance-sensitive shared state with a proven synchronization invariant|You own the proof; misuse can reintroduce data races|
|Mark everything `@MainActor`|UI-only mutable state|Can accidentally serialize non-UI work on the main actor|

---

## 6. Exercise

### Problem

Given an isolation diagnostic about crossing actor boundaries with a non-sendable type, explain how you would narrow the real issue.

### Bad / naive version

```swift
import Foundation

final class Session {
    var token: String?
}

actor AuthStore {
    private let session = Session()

    func currentSession() -> Session {
        session
    }
}

func refresh(authStore: AuthStore) async {
    let session = await authStore.currentSession()
    session.token = "new-token"
}
```

### What is wrong?

```text
The actor protects the Session reference only while it stays inside the actor.

Returning Session leaks the same mutable reference to nonisolated code.

The caller can mutate session.token without going through the actor.

The compiler cannot prove this transfer is safe because Session is a mutable non-Sendable class.
```

The narrowing process:

```text
1. Identify the boundary:
   await authStore.currentSession()

2. Identify the transferred type:
   Session

3. Ask whether the transferred value is Sendable:
   No. It is a mutable class.

4. Ask who owns mutation:
   Supposedly AuthStore, but currentSession leaks mutation capability.

5. Ask what the caller really needs:
   token value, debug snapshot, or mutation operation?

6. Redesign the API around that need.
```

### Improved version

```swift
actor AuthStore {
    private var token: String?

    func currentToken() -> String? {
        token
    }

    func updateToken(_ token: String?) {
        self.token = token
    }
}

func refresh(authStore: AuthStore) async {
    await authStore.updateToken("new-token")
}
```

### Why this is better

The actor owns the mutable state. The caller gets operations, not raw mutable storage.

The public actor API expresses the synchronization boundary:

```text
outside code cannot directly mutate token
outside code must await actor-isolated operations
non-Sendable mutable references do not escape
```

### Production example

For a real image pipeline:

```swift
actor ImageRepository {
    private var cache: [URL: Data] = [:]
    private let client: HTTPClient

    init(client: HTTPClient) {
        self.client = client
    }

    func imageData(for url: URL) async throws -> Data {
        if let cached = cache[url] {
            return cached
        }

        let data = try await client.data(from: url)
        cache[url] = data
        return data
    }

    func clearCache() {
        cache.removeAll()
    }
}
```

This avoids exposing the cache object. The actor exposes domain operations.

---

## 7. Production guidance

Use this in production when:

```text
You hit a complex compiler diagnostic.
You are debugging SwiftUI/result-builder type failures.
You are migrating to Swift 6 strict concurrency.
You are investigating async hangs, stale UI updates, or cancelled work still running.
You are diagnosing memory leaks or retain cycles.
You are investigating crashes that differ between debug and release.
```

Be careful when:

```text
A fix-it adds @MainActor, @unchecked Sendable, any, force casts, or type erasure.
A debugger expression calls computed properties or methods.
A memory graph shows framework objects; not every strong edge is your bug.
A concurrency bug disappears when you add logging or breakpoints.
A task is unstructured and outlives the owner that created it.
```

Avoid when:

```text
You are using sleep-based debugging for async ordering.
You are silencing diagnostics without changing ownership/isolation.
You are debugging SwiftUI builder errors without shrinking the expression.
You are relying only on thread identity in task-based concurrency.
You are assuming actors prevent stale-state bugs.
```

Debugging checklist:

```text
Compiler diagnostics:
- What exact model is violated: type, isolation, ownership, sendability, availability, or builder transformation?
- Is the reported line the cause or just where inference failed?
- Can I add explicit types or constraints to reveal the real issue?
- Did a fix-it change API semantics?

LLDB:
- Which thread and frame am I stopped in?
- What does thread backtrace all show?
- Can frame variable inspect this without running code?
- Would expr or po execute code and perturb the bug?

Concurrency:
- Which task is this?
- Is it named?
- Was it cancelled?
- Is it old work applying to new state?
- Did an await create an interleaving window?
- Did a non-Sendable reference cross an isolation boundary?

Memory:
- Does deinit run?
- Is there a closure context retaining self?
- Is a task, timer, delegate, or callback retaining the owner?
- Is the object truly leaked or still legitimately reachable?

Data races:
- Is mutable state accessed outside its actor/lock/main actor?
- Does Thread Sanitizer report unsynchronized access?
- Is @unchecked Sendable hiding a missing invariant?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> I read the error, try the fix-it, and use breakpoints or print statements if it still does not work.

### Senior answer

> I reduce the failing expression, add explicit types, and identify whether the compiler is complaining about generic constraints, result-builder transformation, Sendable, or actor isolation. At runtime, I use LLDB, memory graph, Thread Sanitizer, and async backtraces depending on the symptom.

### Staff-level answer

> I classify the bug by violated model before fixing it. For a Swift 6 concurrency diagnostic, I identify the isolation boundary, the transferred type, the mutation owner, and whether the API is leaking non-Sendable mutable state. For async runtime bugs, I name tasks, inspect task context, prove cancellation and stale-result suppression, and redesign ownership boundaries if the code is hard to reason about. I treat diagnostics as architectural feedback, not compiler noise.

Staff-level questions to ask:

```text
What invariant is the compiler unable to prove?
Is this diagnostic exposing a real API ownership problem?
Are we leaking mutable implementation detail across an isolation boundary?
Would the fix still be correct under cancellation, reentrancy, and repeated calls?
Can we make this bug impossible through API shape instead of relying on review discipline?
```

---

## 9. Interview-ready summary

Swift debugging starts by identifying which model is failing: type inference, generic constraints, result-builder transformation, ownership, actor isolation, sendability, lifetime, or async ordering. For long compiler errors, I shrink the expression and add explicit types rather than chasing fix-its. For LLDB, I inspect frames, variables, threads, and backtraces, using `expr` carefully because it can execute code. For Swift concurrency, I debug tasks rather than only threads: task IDs, task names, async backtraces, cancellation, and stale-result checks are central. For memory and race bugs, I separate retain cycles, data races, and stale-state bugs because they need different evidence and different fixes.

---

## 10. Flashcards

Q: Why should Swift compiler diagnostics not be treated as noise?

A: They encode Swift’s type, ownership, and isolation rules. A diagnostic often means the compiler cannot prove a safety or type invariant.

Q: What is the first move when facing a huge SwiftUI result-builder error?

A: Shrink the builder. Extract branches, add explicit return types, replace complex expressions with placeholders, and find the first branch that does not return the expected builder-compatible type.

Q: Why can `po` or `expr` be dangerous in LLDB?

A: They can execute Swift code, including computed properties and methods, which can mutate state or perturb the bug.

Q: Why are task names useful in Swift concurrency debugging?

A: They connect runtime task state to logical work, such as a specific search query, image load, or sync operation.

Q: What is the difference between a data race and a stale-state bug?

A: A data race is unsynchronized concurrent memory access. A stale-state bug is outdated async work applying successfully after newer work.

Q: What does a non-`Sendable` actor-boundary diagnostic usually ask you to inspect?

A: Which value crosses the boundary, who owns its mutation, whether it is a mutable reference, and whether the API should expose operations or snapshots instead.

Q: What is the safest fix when an actor returns a mutable non-`Sendable` class?

A: Do not return the class. Keep the mutable state inside the actor and expose actor-isolated operations or immutable snapshots.

Q: What debugging evidence suggests a retain cycle?

A: `deinit` does not run, the memory graph shows a strong reference path, often through a closure context, task, timer, callback, or delegate.

---

## 11. Related sections

- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and assumeIsolatedUntitled]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[C1 — ARC fundamentals]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[F1 — Testing strategy in modern Swift]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — F2 section.
- LLDB. "LLDB." LLDB Documentation. https://lldb.llvm.org/
- Apple Developer. "Using LLDB as a Standalone Debugger." LLDB Transition Guide. https://developer.apple.com/library/archive/documentation/IDEs/Conceptual/gdb_to_lldb_transition_guide/document/lldb-terminal-workflow-tutorial.html
- Apple Developer. "What’s new in Xcode 26." WWDC25. https://developer.apple.com/videos/play/wwdc2025/247/
- Apple Developer. "Task." Apple Developer Documentation. https://developer.apple.com/documentation/swift/task
- Swift Evolution SE-0469 task names. TODO: verify source formatting
- Swift.org. "Calling an Actor-Isolated Method from a Synchronous Nonisolated Context." Swift Compiler Diagnostics. https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/
- Swift 6 concurrency migration guidance for non-`Sendable` data. TODO: verify source formatting
- Apple Developer. "Detect and diagnose memory issues." WWDC21. https://developer.apple.com/videos/play/wwdc2021/10180/
