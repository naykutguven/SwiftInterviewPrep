---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - type-choices
---

# A15 — Type Choices - Struct, Class, Enum, Protocol, and Actor

**Learning goals:**

- Choose types from value behavior, identity, alternatives, capabilities, ownership, and isolation.
- Distinguish Copyable, Sendable, default isolation, and synchronization without inferring layout or cost.
- Verify independent drafts, shared services, and freshness at every publication owner.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]/[[A2 — `let`, `var`, `mutating`, and method semantics]] for values and references, [[A9 — Enums, Associated Values, Recursive Enums, and State Modeling]] for state alternatives, and [[C1 — ARC fundamentals]] for shared lifetime. The concurrency comparisons use [[D4 — Actors and Actor Isolation]]/[[D5 — Global Actors and `@MainActor`]]/[[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]].

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An editable draft and a session holding the current draft have different jobs. The draft should change independently. The session may intentionally have one identity observed by several clients. Start with that sharing contract, then choose declarations.

A struct fits independently replaceable values when its fields support that behavior. A class fits reference identity and shared lifetime. An enum expresses alternatives; a protocol expresses a capability or relationship. An actor owns isolated mutable state for concurrent callers. Copyable describes duplication, Sendable describes a concurrency-transfer contract, and isolation describes permitted access. None of these keywords alone proves deep immutability, freshness across suspension, observation, or allocation cost. [Structures and classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Choose independent values and shared identity deliberately

The profile copy changes independently; the session reference remains shared.

```swift
struct Profile { var name: String }
final class Session {
    var token: String
    init(token: String) { self.token = token }
}
struct FeatureSnapshot {
    var profile: Profile
    let session: Session
}
let saved = FeatureSnapshot(profile: Profile(name: "A"), session: Session(token: "old"))
var edited = saved
edited.profile.name = "B"
edited.session.token = "new"
precondition(saved.profile.name == "A" && edited.profile.name == "B")
precondition(saved.session.token == "new" && edited.session.token == "new")
precondition(saved.session === edited.session)
```

**Walkthrough:** FeatureSnapshot copies its stored values, including the session reference. Its name does not make the session an immutable snapshot. [[A1 — Value semantics, reference semantics, identity, and copy-on-write|A1]] explains the aliasing; the first interview answer separately checks Sendable.

### Use an enum for alternatives and a protocol for a capability

These choices solve different modeling problems and can be combined with structs or classes.

```swift
enum SaveState {
    case editing(String)
    case saved(revision: Int)
    case failed(String)
}
protocol TitleProviding { var title: String { get } }
struct Draft: TitleProviding { var title: String }
func heading<T: TitleProviding>(_ value: T) -> String { value.title }
let state = SaveState.editing("Notes")
if case .editing(let text) = state { precondition(text == "Notes") }
precondition(heading(Draft(title: "Notes")) == "Notes")
```

**Walkthrough:** SaveState prevents simultaneous case tags. TitleProviding lets heading depend on a capability rather than a specific storage representation. A protocol can be a useful boundary with only one production implementation.

## Rules and pitfalls

### Cleanup alone does not require shared ownership

**Common pitfall:** Using an aliasable class for a resource whose contract requires one exclusive finish. This is a synthetic release counter, not an OS handle.

**Mistake:** Standalone runnable example.

```swift
final class Log { var releases = 0 }
final class Handle {
    let log: Log
    init(_ log: Log) { self.log = log }
    func finish() { log.releases += 1 }
}
let log = Log()
let handle = Handle(log)
let alias = handle
handle.finish()
alias.finish()
precondition(log.releases == 2)
```

**Consequence:** Two aliases can independently finish the same logical resource.

**Why:** The class supplies shared reference identity, while this requirement called for exclusive ownership. deinit is also supported by noncopyable values.

**Correction:** Use a noncopyable value when exclusive ownership is the intended contract. Its destruction performs the one synthetic release.

```swift
final class Log { var releases = 0 }
struct Handle: ~Copyable {
    let log: Log
    consuming func finish() {}
    deinit { log.releases += 1 }
}
func useHandle(_ log: Log) {
    let handle = Handle(log: log)
    handle.finish()
}
let log = Log()
useHandle(log)
precondition(log.releases == 1)
```

**What changes:** The wrapper cannot be duplicated and finish consumes it. The referenced Log can still have aliases; noncopyability does not establish a deeply unique graph. A shared class with idempotent close can also be correct when sharing is actually required.

### The required call context matters

**Common pitfall:** Choosing an actor for a counter whose clients require a short synchronous method, then calling it without an actor hop.

**Mistake:** Compile-only; intentionally rejected.

```swift
actor Counter {
    private var count = 0
    func value() -> Int { count }
}
func synchronousRead(_ counter: Counter) -> Int { counter.value() }
```

**Consequence:** The compiler rejects the actor-isolated call from a synchronous outside context.

**Why:** An actor’s isolated API usually requires awaiting from outside. Shared state alone does not decide whether that call contract fits the clients.

**Correction:** For a small synchronous critical section, a Mutex-backed type can supply checked shared state. This script requires the SDK/runtime support for Synchronization.Mutex (macOS 15 or equivalent; the baseline host is newer).

```swift
import Synchronization

final class Counter: Sendable {
    private let count = Mutex(0)
    func increment() { count.withLock { $0 += 1 } }
    func value() -> Int { count.withLock { $0 } }
}
let counter = Counter()
counter.increment()
precondition(counter.value() == 1)
```

**What changes:** The lock protects a bounded synchronous access and never spans await or calls arbitrary callbacks. Use an actor when the service’s asynchronous isolation contract fits; [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks|D12]] covers the tradeoffs.

### A display token does not also protect a cache

**Common pitfall:** Allowing the repository to store every completion while only the UI rejects old results. The calls below deliberately deliver completion 2 before completion 1.

**Mistake:** Standalone runnable example.

```swift
actor Cache {
    private var value: Int?
    func store(_ value: Int) { self.value = value }
    func read() -> Int? { value }
}
@MainActor final class Screen {
    let currentRequest = 2
    var visible: Int?
    func complete(_ value: Int, request: Int, cache: Cache) async {
        await cache.store(value)
        guard request == currentRequest else { return }
        visible = value
    }
}
let cache = Cache()
let screen = Screen()
await screen.complete(20, request: 2, cache: cache)
await screen.complete(10, request: 1, cache: cache)
let cached = await cache.read()
precondition(screen.visible == 20 && cached == 10)
```

**Consequence:** The display is current but the shared cache has regressed.

**Why:** MainActor and Cache serialize their own state accesses. They do not define a shared freshness policy, and the cache writes happen before the display’s guard.

**Correction:** Enforce the chosen latest-request policy at the cache too. This separate script registers the two starts before delivering completions.

```swift
actor Cache {
    private var current = 0
    private var value: Int?
    func begin(_ request: Int) { current = request }
    func complete(_ value: Int, request: Int) {
        guard request == current else { return }
        self.value = value
    }
    func read() -> Int? { value }
}
@MainActor final class Screen {
    let currentRequest = 2
    var visible: Int?
    func complete(_ value: Int, request: Int, cache: Cache) async {
        await cache.complete(value, request: request)
        guard request == currentRequest else { return }
        visible = value
    }
}
let cache = Cache()
let screen = Screen()
await cache.begin(1)
await cache.begin(2)
await screen.complete(20, request: 2, cache: cache)
await screen.complete(10, request: 1, cache: cache)
let cached = await cache.read()
precondition(screen.visible == 20 && cached == 20)
```

**What changes:** Both publication owners now check authority. The fixture supplies IDs in a controlled order; production needs a defined token/revision scope, a key/user boundary, and a policy for which callers may replace that authority.

## Worked example

### Commit a value into a shared session

**Setup:** Standalone synchronous script. Concurrency is not involved in this first ownership example.

**Predict:** Does a later draft edit also change the committed session?

```swift
struct Draft {
    var title: String
    var items: [String]
}
final class Session {
    var current: Draft
    init(_ current: Draft) { self.current = current }
}
let initial = Draft(title: "Original", items: ["one"])
let session = Session(initial)
let alias = session
var draft = session.current
draft.items.append("two")
precondition(session.current.items == ["one"])
session.current = draft
draft.items.removeAll()
precondition(alias.current.items == ["one", "two"])
precondition(initial.items == ["one"])
```

**Outcome:** Both session references observe the committed value; later draft mutation remains independent.

**Why:** Session supplies shared identity, and Draft supplies value behavior. Neither choice specifies how a stale concurrent draft should be reconciled.

## Interview questions

### What does copying a struct containing a class or actor share, and is it Sendable?

**Answer:** It copies the stored reference, preserving that service’s identity. Checked Sendable depends on the fields’ contracts, not on struct spelling. An actor reference is safely shareable; an ordinary mutable class generally prevents the wrapper’s checked conformance.

**Example:** Standalone runnable example.

```swift
actor Catalog {
    private var count = 0
    func increment() { count += 1 }
    func value() -> Int { count }
}
struct Services: Sendable { let catalog: Catalog }
let first = Services(catalog: Catalog())
let second = first
precondition(first.catalog === second.catalog)
await second.catalog.increment()
let observed = await first.catalog.value()
precondition(observed == 1)
```

**Walkthrough:** Both wrappers share one actor and observe its updated state. The actor’s isolation makes the shared reference safe under Sendable; it does not create independent snapshots.

**Boundary:** A class can also be Sendable when its immutable state or audited synchronization satisfies the rules. Do not use unchecked conformance to substitute for a missing design.

This separate wrapper is **intentionally rejected**:

```swift
final class Session { var token = "old" }
struct Snapshot: Sendable { let session: Session }
```

### Which requirements justify a class, noncopyable struct, MainActor owner, actor, or Mutex-backed type?

**Answer:** Use a class for shared identity or framework/weak-reference integration, a noncopyable value for exclusive resource lifetime, MainActor for UI-owned state, an actor for asynchronous shared-state access, and Mutex for bounded synchronous protected operations.

**Example:** Standalone runnable example.

```swift
import Synchronization

final class Session { var token = "old" }  // Shared identity, confined by its owner.
struct Ticket: ~Copyable {
    let number: Int
    consuming func finish() -> Int { number }
}
@MainActor final class Screen { var title = "Draft" }
actor Repository {
    private var count = 0
    func increment() { count += 1 }
}
final class Counter: Sendable {
    private let value = Mutex(0)
    func increment() -> Int {
        value.withLock {
            $0 += 1
            return $0
        }
    }
}
func finishTicket() -> Int {
    let ticket = Ticket(number: 7)
    return ticket.finish()
}
let session = Session()
let alias = session
precondition(session === alias)
precondition(finishTicket() == 7)
let screen = Screen()
screen.title = "Edited"
precondition(screen.title == "Edited")
await Repository().increment()
precondition(Counter().increment() == 1)
```

**Walkthrough:** Each declaration serves a different requirement: identity, exclusive consumption, UI isolation, asynchronous shared access, or synchronous protected mutation. The examples compile under the declared nonisolated default; UI ownership is explicit.

**Boundary:** These are design criteria, not a performance ranking. An actor does not automatically add observation, deduplication, task ownership, or freshness. A noncopyable struct can also have deinit, as shown in the pitfall.

### Why can latest-request-wins display coexist with older cache data?

**Answer:** The UI and cache have independent publication policies. A display token can reject an old UI update after the repository has already stored it. Enforce freshness at every state owner that requires it.

**Example:** Standalone runnable example.

```swift
actor Cache {
    private var value: Int?
    func insert(_ value: Int) { self.value = value }
    func read() -> Int? { value }
}
@MainActor final class Display {
    let current = 2
    var value: Int?
    func completed(_ result: Int, request: Int) {
        guard request == current else { return }
        value = result
    }
}
let cache = Cache()
let display = Display()
await cache.insert(20)
display.completed(20, request: 2)
await cache.insert(10)
display.completed(10, request: 1)
let cached = await cache.read()
precondition(display.value == 20 && cached == 10)
```

**Walkthrough:** The ordered calls simulate a newer completion followed by an older one. The display rejects request 1, but the cache accepts its write. The corrected pitfall adds a cache request guard.

**Boundary:** A cache may intentionally use a different rule, such as server revision or per-key freshness across multiple screens. Define that rule explicitly rather than sharing a UI token without a scope.

## Focused exercise

**Task:** Choose an independent draft and a shared session, then verify copy and commit behavior.

**Acceptance checks:**

- Draft edits preserve the session until commit.
- Both session references observe the commit.
- The original snapshot and later draft edits remain independent.

**Solution:** Standalone synchronous model. Concurrent use would require an explicit owner/isolation contract.

```swift
struct Draft {
    var title: String
    var items: [String]
}
final class Session {
    var current: Draft
    init(current: Draft) { self.current = current }
}
let initial = Draft(title: "Original", items: ["one"])
let session = Session(current: initial)
let alias = session
var draft = session.current
draft.title = "Edited"
draft.items.append("two")
precondition(session.current.title == "Original")
precondition(session.current.items == ["one"])
session.current = draft
precondition(alias.current.title == "Edited" && alias.current.items == ["one", "two"])
draft.items.removeAll()
precondition(session.current.items == ["one", "two"])
precondition(initial.title == "Original" && initial.items == ["one"])
```

**Reasoning:** Session owns current state with shared identity. Draft contains independent value data. The class choice is justified by ownership, not an allocation claim. A UI session can be MainActor-owned; a cross-task service needs a checked boundary and a stale-commit policy.

## In practice

**Choose it when:** Write down independent values, shared identities, exclusive resources, capabilities, and required call contexts before choosing types.

**Investigate by:** Test copies and aliases first, then suspension, deduplication, task ownership, and publication separately.

**Strong evidence:** Explain why each type exists and what its isolation leaves to the workflow; verify UI and cache policies independently.

### Service and cache publication boundaries — optional

This compile-only shared setup preserves a real URLSession adapter and a controlled stub. The protocol is useful even with one production implementation because it names the dependency contract and permits deterministic tests. No request is sent by compiling this block or by the stub tests below.

```swift
import Foundation

#if canImport(FoundationNetworking)
    import FoundationNetworking
#endif

protocol ImageFetching: Sendable {
    func imageData(for url: URL) async throws -> Data
}

enum ImageFetchError: Error, Equatable {
    case nonHTTPResponse
    case status(Int)
}

struct URLSessionImageFetcher: ImageFetching {
    let session: URLSession

    func imageData(for url: URL) async throws -> Data {
        let (data, response) = try await session.data(from: url)
        guard let response = response as? HTTPURLResponse else {
            throw ImageFetchError.nonHTTPResponse
        }
        guard (200..<300).contains(response.statusCode) else {
            throw ImageFetchError.status(response.statusCode)
        }
        return data
    }
}

struct StubImageFetcher: ImageFetching {
    let result: Result<Data, ImageFetchError>

    func imageData(for url: URL) async throws -> Data {
        try Task.checkCancellation()
        return try result.get()
    }
}
```

Prepend the service setup. This runnable generic/existential client checks bytes and errors through StubImageFetcher. It does not validate transport, HTTP status handling, or image decoding; the production adapter’s code validates HTTP response/status and returns raw bytes.

```swift
func bytes<Fetcher: ImageFetching>(using fetcher: Fetcher, url: URL) async throws -> Data {
    try await fetcher.imageData(for: url)
}
let url = URL(string: "https://example.invalid/image")!
let stub = StubImageFetcher(result: .success(Data([1, 2])))
let concrete = try await bytes(using: stub, url: url)
let erased: any ImageFetching = stub
let existential = try await erased.imageData(for: url)
precondition(concrete == Data([1, 2]) && existential == concrete)
do {
    _ = try await StubImageFetcher(result: .failure(.status(404))).imageData(for: url)
    preconditionFailure()
} catch { precondition((error as? ImageFetchError) == .status(404)) }
```

**Walkthrough:** The generic call retains a concrete dependency type; the existential supports runtime storage. Neither form alone proves dispatch or allocation cost. These async signatures permit suspension and do not promise a dedicated background thread; [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`|D8]]/[[D14 — Approachable Concurrency and Default Isolation|D14]]/[[F5 — Language mode, build settings, and migration flags|F5]] explain how default isolation and execution settings affect that contract. [SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)

The next compile-only shared setup is ImageCache, followed by a runnable client that prepends it.

```swift
import Foundation

actor ImageCache {
    private var storage: [URL: Data] = [:]

    func data(for url: URL) -> Data? { storage[url] }
    func insert(_ data: Data, for url: URL) { storage[url] = data }
    func remove(for url: URL) { storage[url] = nil }
    func removeAll() { storage.removeAll() }
    var count: Int { storage.count }
}
```

```swift
let url = URL(string: "https://example.invalid/image")!
let cache = ImageCache()
let firstMiss = await cache.data(for: url)
let secondMiss = await cache.data(for: url)
precondition(firstMiss == nil && secondMiss == nil)
await cache.insert(Data([1]), for: url)
let stored = await cache.data(for: url)
let count = await cache.count
precondition(stored == Data([1]) && count == 1)
await cache.remove(for: url)
await cache.insert(Data([2]), for: url)
await cache.removeAll()
let empty = await cache.count
precondition(empty == 0)
```

**Walkthrough:** Separate lookups can both miss before either caller inserts, so actor isolation alone does not coalesce a lookup/fetch/insert workflow. Each synchronous actor method here has no internal suspension. An outside caller awaits actor access; code already in that isolation domain can call synchronously. [Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)

## References

### Related guides

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A4 — Optionals and Nil Modeling]]
- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A9 — Enums, Associated Values, Recursive Enums, and State Modeling]]
- [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]
- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D4 — Actors and Actor Isolation]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [Structures and classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)
- [Default isolation, SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
- [Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [SE-0390 — Noncopyable structs and enums](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [URLSession.data(from:)](<https://developer.apple.com/documentation/foundation/urlsession/data(from:)>)
- [URLProtocol](https://developer.apple.com/documentation/foundation/urlprotocol)
