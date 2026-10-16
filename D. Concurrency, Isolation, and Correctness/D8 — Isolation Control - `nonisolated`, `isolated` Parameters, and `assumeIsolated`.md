---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actor-isolation
  - swift6
---

# D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`

**Goal:** Choose an isolated operation or snapshot-based nonisolated helper without bypassing state protection.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Isolation describes access permission. A nonisolated member opts out of its enclosing actor/global-actor protection and cannot freely read that protected mutable state. An isolated parameter borrows the supplied actor's isolation for the operation. AssumeIsolated asserts an existing executor context; it does not schedule a hop. [Isolated parameters](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md)

## Rules and pitfalls

### Prefer a compiler-checked boundary

Keep state-reading behavior isolated, or pass an independent Sendable snapshot to a pure helper. Synchronous nonisolated code runs on its caller's execution context and can still occupy MainActor with expensive work.

**Common pitfall:** Removing isolation to satisfy a protocol or suppress a diagnostic can expose protected state. Change the witness/consumer contract or use a snapshot instead of reading through the boundary.

### Use runtime assertions only for established context

AssumeIsolated is appropriate at a documented synchronous callback boundary whose executor guarantee is external to the compiler. It checks that assumption and traps when wrong.

**Common pitfall:** Main-thread observation alone is not a general proof of arbitrary actor isolation. Use an async actor call or an explicit actor-bound callback when possible. Under this baseline plain nonisolated async follows caller isolation; @concurrent intentionally switches off it.

## Interview questions

### What does `nonisolated` mean on a member of an actor or globally isolated type?

It removes that member from the enclosing actor/global-actor isolation contract. A synchronous call can occur without an actor hop but cannot directly use protected mutable state. The annotation is meaningful on that member even though ordinary top-level declarations already default nonisolated.

### When is `assumeIsolated` appropriate, and what makes it dangerous?

Use it only when an existing callback/executor contract guarantees the required actor but the type system cannot express it. A wrong assumption traps; it does not make cross-actor access safe or dispatch to the actor. Prefer a compiler-expressed isolation requirement whenever available.

### Why does a pure synchronous nonisolated helper still occupy its MainActor caller?

A synchronous helper executes inline in the caller's context. Removing declaration isolation changes access permission, not work placement. If the work is expensive and must leave MainActor, design an async @concurrent boundary with safely transferable inputs/results.

### Why may a function have only one isolated parameter even when a caller passes one actor twice?

A function has one statically defined isolation domain, so the language permits only one isolated parameter. It cannot validate a declaration by relying on a particular call passing the same actor twice; other callers might pass different instances. Express the operation around one isolated owner.

### What does Hashable UnownedTaskExecutor identify, and why does it neither retain the owner nor prove actor isolation? Check OS 27 availability.

It identifies an unowned executor reference under its equality/hash contract, available with the checked OS 27 support. It neither retains the actor/executor owner nor proves that current execution is actor-isolated. Use it for scoped diagnostics, not a lifetime owner or replacement for isolation checking.

## Focused exercise

**Task:** Review an actor helper that reads stored state. Choose an isolated operation or a nonisolated snapshot-based helper and explain an invalid alternative.

**Acceptance checks:**

- The state-reading operation crosses the actor boundary.
- The nonisolated helper receives independent Int data.
- A nonisolated member that directly reads count rejects.

**Solution:**

Standalone actor snapshot and explicit opt-out:

```swift
actor Store {
    private var count = 3
    func description() -> String { "count: \(count)" }
    func snapshot() -> Int { count }
    nonisolated func describe(snapshot: Int) -> String { "count: \(snapshot)" }
}
@main struct Probe {
    static func main() async {
        let store = Store()
        let value = await store.snapshot()
        let first = await store.description()
        precondition(store.describe(snapshot: value) == first)
    }
}
```

A separate nonisolated func invalid() -> Int { count } inside Store intentionally rejects direct access to its isolated mutable property.

**Reasoning:** Both valid operations can format the same snapshot, but the snapshot may become stale after a later mutation. The helper has no privileged access to live state and makes no offloading or synchronization promise.

## In practice

**Choose it when:** Expose the least isolation permission that preserves the operation's invariant.

**Investigate by:** Identify the protected state, call context, and any external executor guarantee.

**Strong evidence:** Keep boundaries compiler-checked and justify any runtime assertion with documented integration evidence.

### Executor identity diagnostics — optional

#### Swift 6.4 executor identity at specialist depth

UnownedTaskExecutor is Hashable in the checked OS 27 library, so it can key a set or dictionary:

```swift
@available(anyAppleOS 27.0, *)
func distinctExecutors(_ executors: [UnownedTaskExecutor]) -> Int {
    Set(executors).count
}
```

Hashing follows task-executor identity. Keep the owning executors alive while using their unowned handles; keys do not establish ownership or actor isolation. A task executor is not interchangeable with an actor's serial executor, and equality is not permission to call assumeIsolated. Do not persist hashValue as an identifier. This is a scheduling/resource-indexing tool for specialized infrastructure, not a normal UI isolation technique. [SE-0523](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0523-hashable-unownedtask-executor.md)

## References

### Related guides

- [[D4 — Actors and Actor Isolation]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[D13 — Swift 6 Strict Concurrency Migration]]
- [[D14 — Approachable Concurrency and Default Isolation]]

### Official sources

- [Isolated parameters](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md)
- [SE-0523](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0523-hashable-unownedtask-executor.md)
- [Swift.org. "Declarations." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/)
- [Apple Developer. "assumeIsolated(_:file:line:)." Apple Developer Documentation](https://developer.apple.com/documentation/swift/actor/assumeisolated%28_%3afile%3aline%3a%29)
- [GitHub. "Allow nonisolated to Prevent Global Actor Inference." Swift Evolution SE-0449](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md)
- [SE-0461 — Async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)
