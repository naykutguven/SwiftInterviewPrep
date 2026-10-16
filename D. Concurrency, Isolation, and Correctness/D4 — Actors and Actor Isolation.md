---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actors
  - actor-isolation
---

# D4 — Actors and Actor Isolation

**Goal:** Protect one counter invariant across concurrent callers without suspending inside its mutation.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An actor is a reference type with an isolation domain for its instance state. Synchronous isolated operations execute without simultaneous access by other actor operations. An actor is not a dedicated thread, a cross-task FIFO queue, or an automatic transaction around an entire async workflow. Suspension lets other work enter and change state. [Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)

## Rules and pitfalls

### Keep an invariant within one isolated operation

A synchronous check-and-update inside the actor can protect a state invariant. Splitting the operation into separate awaited calls gives other callers an opportunity between them. Return an appropriate Sendable snapshot rather than leaking retained mutable non-Sendable storage.

**Common pitfall:** Awaiting actor.get followed by actor.set is not one atomic increment. Put the read and write together inside increment.

### Give shared producers identity and ownership

An actor can retain an in-flight task for deduplication, but it still needs invalidation, replacement, cancellation, and cleanup policy. An old completion must not repopulate invalidated data or clear a replacement's task.

**Common pitfall:** A defer that unconditionally clears an in-flight entry after await can erase newer work. Match the operation's identity or generation before publishing and clearing. Cancelling one waiter must not automatically cancel a producer shared by others.

## Interview questions

### Which state is isolated, and which invariants can become stale at an await or async defer?

Actor instance state belongs to that actor; explicit global-actor state belongs to its declared domain. Any invariant based on mutable state can become stale at a suspension, including an awaited async defer. Re-read, validate an identity/version, or reserve state before suspension according to the product's policy.

### Can an older refresh repopulate the cache after invalidation, or clear a newer in-flight operation on failure?

Yes, unless invalidation advances an identity/generation checked by the old completion. Both success publication and failure cleanup must verify they still own the current slot. Actor serialization prevents simultaneous memory access but does not infer the intended temporal cache policy.

## Focused exercise

**Task:** Put a shared counter behind an actor. Check concurrent callers and identify which invariant is protected without suspension.

**Acceptance checks:**

- All 100 increments contribute to the final snapshot after the group joins.
- The read-modify-write has no internal suspension.
- The test does not depend on caller order or a particular actor thread.

**Solution:**

Standalone actor and task-group probe:

```swift
actor Counter {
    private var value = 0
    func increment() { value += 1 }
    func snapshot() -> Int { value }
}
@main struct Probe {
    static func main() async {
        let counter = Counter()
        await withTaskGroup(of: Void.self) { group in
            for _ in 0..<100 {
                group.addTask { await counter.increment() }
            }
        }
        let result = await counter.snapshot()
        precondition(result == 100)
    }
}
```

**Reasoning:** increment owns the whole state transition inside the actor. The group provides a completion boundary before the snapshot. This bounded example cannot overflow Int; production counters need their own range policy. An async multi-step operation would need additional reentrancy reasoning.

## In practice

**Choose it when:** State is shared through asynchronous calls and one actor can own its invariants.

**Investigate by:** Mark every suspension and identify which assumptions other actor work can invalidate.

**Strong evidence:** Verify shared-operation invalidation and waiter cancellation separately from simple data-race freedom.

### Shared refresh and invalidation — optional

#### Problem

Review a token-refresh actor and identify a logic race that can still happen despite actor isolation.

#### Bad / naive version

```swift
import Foundation

struct Token: Sendable {
    let rawValue: String
    let expiresAt: Date

    var isValid: Bool {
        expiresAt > .now
    }
}

protocol AuthClient: Sendable {
    func refreshToken() async throws -> Token
}

actor TokenStore {
    private var token: Token?
    private let authClient: AuthClient

    init(authClient: AuthClient) {
        self.authClient = authClient
    }

    func validToken() async throws -> Token {
        if let token, token.isValid {
            return token
        }

        let refreshed = try await authClient.refreshToken()
        token = refreshed
        return refreshed
    }
}
```

#### What is wrong?

> This actor is data-race safe, but it is not logic-race safe.

Possible interleaving:

> Task A calls validToken()
> Task A sees token is nil/expired
> Task A starts refreshToken() and suspends
>
> Task B calls validToken()
> Task B also sees token is nil/expired
> Task B starts another refreshToken() and suspends
>
> Task A receives token A and stores it
> Task B receives token B and stores it

Problems this can cause:

- duplicate refresh requests
- wasted network work
- token rotation bugs
- refresh-token invalidation
- last-write-wins with an older or undesired token
- inconsistent request retry behavior

No actor data race exists. The bug is in the workflow.

#### Improved version: shared refresh without invalidation

```swift
import Foundation

struct Token: Sendable {
    let rawValue: String
    let expiresAt: Date

    var isValid: Bool {
        expiresAt > .now
    }
}

actor TokenStore {
    private var token: Token?
    private var refreshTask: Task<Token, any Error>?

    private let refresh: @Sendable () async throws -> Token

    init(refresh: @escaping @Sendable () async throws -> Token) {
        self.refresh = refresh
    }

    func validToken() async throws -> Token {
        if let token, token.isValid {
            return token
        }

        if let refreshTask {
            return try await refreshTask.value
        }

        let task = Task<Token, any Error> {
            try await refresh()
        }

        refreshTask = task

        do {
            let refreshed = try await task.value
            token = refreshed
            refreshTask = nil
            return refreshed
        } catch {
            refreshTask = nil
            throw error
        }
    }

}
```

#### Why this is better

The actor now models both:

> cached token
> in-flight refresh

That means the state machine has three useful states:

```swift
enum TokenState {
    case empty
    case valid(Token)
    case refreshing(Task<Token, any Error>)
}
```

The actor now stores in-flight work explicitly. This first version has no invalidation API and waits for shared completion even when an individual waiter is cancelled. It is not yet a logout/session-boundary design.

With this shape:

> Task A starts refresh and stores refreshTask
> Task B sees refreshTask and awaits the same task
> Task C sees refreshTask and awaits the same task
> Only one refresh request is active

#### Add invalidation and protect against late completion

A state enum alone does not fix invalidation: an old producer can still finish after the state was reset or a newer refresh began. This version associates each producer with a unique ID. Use the Foundation-backed `Token` declaration above.

```swift
enum RefreshError: Error { case superseded }

actor TokenStore {
    private enum State {
        case empty
        case valid(id: UUID, token: Token)
        case refreshing(id: UUID, task: Task<Token, any Error>)
    }

    private var state: State = .empty
    private let refresh: @Sendable () async throws -> Token

    init(refresh: @escaping @Sendable () async throws -> Token) {
        self.refresh = refresh
    }

    func validToken() async throws -> Token {
        try Task.checkCancellation()
        let id: UUID
        let task: Task<Token, any Error>
        switch state {
        case .valid(_, let token) where token.isValid:
            return token
        case .refreshing(let currentID, let currentTask):
            id = currentID
            task = currentTask
        case .empty, .valid:
            id = UUID()
            task = Task { [refresh] in try await refresh() }
            state = .refreshing(id: id, task: task)
        }

        let refreshed: Token
        do {
            refreshed = try await task.value
        } catch {
            if case .refreshing(let currentID, _) = state, currentID == id {
                state = .empty
            }
            throw error
        }

        switch state {
        case .refreshing(let currentID, _) where currentID == id:
            state = .valid(id: id, token: refreshed)
        case .valid(let currentID, _) where currentID == id:
            break // another waiter already committed this producer's result
        default:
            throw RefreshError.superseded
        }
        // A cancelled waiter does not invalidate a successful shared result.
        try Task.checkCancellation()
        return refreshed
    }

    func invalidate() {
        if case .refreshing(_, let task) = state { task.cancel() }
        state = .empty
    }
}
```

Invalidation clears local state and requests producer cancellation. If an uncancellable producer later succeeds, its ID cannot repopulate the cache, clear newer work, or supply a successful result to an old waiter. A late failure also cannot clear a newer refresh. Invalidation does not revoke tokens already returned, undo server-side rotation, or guarantee the old network request has stopped before a new one starts.

Waiter cancellation is checked before entry and after shared completion; it does not promptly interrupt task.value or cancel the producer. The successful producer may populate the cache even if its original waiter was cancelled. Prompt per-waiter cancellation, expiry validation of server responses, revocation, retry/backoff, refresh-token rotation, and persistence require additional domain policy.

## References

### Related guides

- [[D1 — Structured Concurrency Fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[D13 — Swift 6 Strict Concurrency Migration]]

### Official sources

- [Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)
- [Swift.org. "Concurrency." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [Swift.org. "Calling an Actor-Isolated Method from a Synchronous Nonisolated Context." Swift Compiler Diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/)
- [SE-0493 — Async calls in defer](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0493-defer-async.md)
- [SE-0466 — Control default actor isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
