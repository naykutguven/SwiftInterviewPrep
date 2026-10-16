---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - type-choices
---

# A15 — Type Choices - Struct, Class, Enum, Protocol, and Actor

**Goal:** Choose an independently editable draft and a shared session with explicit ownership.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Choose a type from the sharing contract. Structs fit independently replaceable values when their fields support that behavior. Classes fit reference identity and shared lifetime. Enums express alternatives; protocols express capabilities and relationships. Actors protect their own mutable state across concurrency boundaries. None of these choices alone proves cost, deep immutability, or correct asynchronous ordering. [Structures and classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)

## Rules and pitfalls

### Choose identity and isolation separately

A class can be immutable, MainActor-owned, actor-confined, or protected by audited synchronization. A uniquely owned resource can instead use a noncopyable struct with deinit. An actor fits asynchronous shared-state access; Mutex fits short synchronous critical sections.

**Common pitfall:** Selecting a class only because cleanup is needed ignores noncopyable ownership. Selecting an actor only because state is shared ignores whether callers need a synchronous operation and whether locks can safely bound it. See [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]].

### Preserve the caller's value contract

An independent draft should store independent domain values. A struct containing a class or actor can intentionally share a service without being a snapshot of that service. Sendable is checked separately from copyability.

**Common pitfall:** MainActor or actor serialization does not establish latest-request-wins. A suspended operation can publish late, and separate UI/cache owners can disagree. Place freshness policy at every state publication boundary that requires it.

## Interview questions

### What does copying a struct containing a class or actor actually share, and does the struct satisfy Sendable?

It copies the stored reference, sharing the class or actor identity. Whether the struct satisfies Sendable depends on each field and its isolation/transfer contract. An actor reference is safely shareable, but a mutable ordinary class field generally prevents checked Sendable. Sharing a safe service still does not snapshot its state.

### Which requirements justify a class, a noncopyable struct, a MainActor owner, an actor, or a Mutex-backed type?

Use a class for shared identity or required framework/weak-reference integration; a noncopyable struct for exclusive resource ownership; MainActor for UI-owned state; an actor for asynchronous shared-state serialization; and a Mutex-backed type for small synchronous protected state. Choose from call requirements, lifetime, and invariants rather than keyword preference.

### Why can a model's latest-request-wins display still coexist with an older value in a shared cache?

The display and cache enforce separate policies. A UI request token can prevent old display updates while the repository still accepts an older cache write. If cache freshness matters, enforce a revision or request rule at the cache too. Cancellation alone does not authorize or forbid either publication.

## Focused exercise

**Task:** Choose a representation for an independent draft and a shared session. Check copy behavior and explain who owns mutable state.

**Acceptance checks:**

- Draft edits preserve the current session value until commit.
- Both session references observe the committed value.
- A captured original and further draft edits remain independent.

**Solution:**

Standalone synchronous model. The session deliberately has shared reference identity; concurrent use would require a stated isolation model.

```swift
struct Draft { var title: String; var items: [String] }
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
precondition(initial.items == ["one"])
session.current = draft
precondition(alias.current.items == ["one", "two"])
draft.items.removeAll()
precondition(session.current.items == ["one", "two"])
```

**Reasoning:** The session owns mutable current state; the draft owns independent value data. The class is justified by shared identity, not an allocation claim. A UI session can be explicitly MainActor-isolated; a cross-task service needs an actor or another checked boundary. Stale commits need a product-level conflict policy.

## In practice

**Choose it when:** Write down independent state, shared services, lifecycle ownership, and required call contexts before choosing declarations.

**Investigate by:** Test copies and aliases, then test suspension and publication separately.

**Strong evidence:** Explain why each type exists, identify what its isolation does not guarantee, and verify both UI and cache freshness rules.

### Service and cache publication boundaries — optional

#### A real service boundary can have one production implementation

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

The URLSession adapter returns bytes for successful HTTP responses and propagates transport errors; it does not decode or verify an image. Injecting the session permits controlled tests and application configuration. The protocol defines a useful dependency boundary even with one production implementation. A generic caller can retain Fetcher's concrete type; `any ImageFetching` is useful when runtime storage needs implementation variation. Neither form alone proves allocation or dispatch costs.

These async signatures permit suspension. They do not promise a dedicated background thread. The default nonisolated async execution policy can differ by build settings; choose explicit execution/isolation annotations when the API requires them. Existing fundamentals and 6.2/6.3 migration details belong in D8/D14/F5. [Default isolation, SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)

#### An actor protects its own state, not every surrounding workflow

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

Each synchronous actor operation has no suspension inside it. A caller outside the actor generally awaits its isolated operations. Calls already on the same isolation domain can be synchronous; nonisolated members have a different contract. “Every actor read is async” is too broad.

A sequence of lookup, fetch, and insert is not one atomic actor operation. Two callers can both miss and fetch. Making fetch a method of the actor would not remove that issue if the method suspends without coordinating in-flight requests. Actors also have identity and can be shared through otherwise value-shaped services. [Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)

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
- [URLSession.data(from:)](https://developer.apple.com/documentation/foundation/urlsession/data(from:)
- [URLProtocol](https://developer.apple.com/documentation/foundation/urlprotocol)
