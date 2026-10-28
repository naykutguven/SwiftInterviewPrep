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

**Learning goals:**

- Choose actor-isolated access, a snapshot-based nonisolated helper, or an isolated parameter without bypassing protected state.
- Distinguish access permission, work placement, and runtime isolation assertions.
- Explain executor identity diagnostics and their ownership/availability limits.

**Prerequisites:** [[D4 — Actors and Actor Isolation]]–[[D5 — Global Actors and `@MainActor`]] for actor and global-actor ownership. Executor identity is an optional infrastructure topic.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

`nonisolated` removes an enclosing actor requirement from a member; it does not grant that member permission to touch the actor's mutable state. An `isolated` parameter instead lets a function operate with the supplied actor's permission. These are access contracts, not requests for particular threads.

`assumeIsolated` asserts that execution already has the required actor context. It does not hop to that actor. A false assertion traps. Prefer compiler-expressed access unless a documented external synchronous callback contract supplies a guarantee the compiler cannot express. [SE-0313](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### A nonisolated helper works on independent data

**Shared setup — store:** Prepend it where named; the formatting helper receives a value snapshot.

```swift
actor Store {
    private var count = 3
    func snapshot() -> Int { count }
    func description() -> String { "count: \(count)" }
    nonisolated func describe(snapshot: Int) -> String { "count: \(snapshot)" }
}
```

**Setup:** Prepend **store**. Runnable.

```swift
let store = Store()
let snapshot = await store.snapshot()
let live = await store.description()
precondition(store.describe(snapshot: snapshot) == live)
```

**Walkthrough:** Reading live state needs the actor boundary. Formatting an Int does not. A later mutation could make the snapshot stale without making it unsafe to format.

### One isolated parameter supplies one domain

**Setup:** Standalone runnable example.

```swift
actor Account {
    var balance = 10
    func debit(_ amount: Int) { balance -= amount }
}
func charge(_ amount: Int, to account: isolated Account) -> Int {
    account.debit(amount)
    return account.balance
}
let account = Account()
let remaining = await charge(3, to: account)
precondition(remaining == 7)
```

**Walkthrough:** The outer call obtains Account permission. Inside `charge`, the read and debit use that permission without separate awaits. Real payment code also needs amount validation; this fixture demonstrates isolation, not a complete accounting contract.

## Rules and pitfalls

### Opting out does not permit protected-state access

**Common pitfall:** Marking a state-reading method nonisolated is expected to repair its call-site diagnostic.

**Mistake:** Standalone **intentionally rejected** declaration.

```swift
actor Store {
    private var count = 3
    nonisolated func invalid() -> Int { count }
}
```

**Consequence:** The mutable actor property is inaccessible from this member.

**Why:** The annotation removed the permission that the body needs.

**Correction:** Prepend **store**. Runnable; obtain a snapshot through the actor, then format it.

```swift
let store = Store()
let value = await store.snapshot()
let description = store.describe(snapshot: value)
precondition(description == "count: 3")
```

**What changes:** The data crosses through a compiler-checked actor call. The helper has no access to live protected storage.

### An assertion is not scheduling

**Common pitfall:** `assumeIsolated` is used as if it dispatches a callback onto MainActor.

**Mistake:** Standalone **compile-only; do not call from an unknown context**. This may trap.

```swift
@MainActor final class Model { var count = 3 }
func callback(_ model: Model) -> Int {
    MainActor.assumeIsolated { model.count }
}
```

**Consequence:** If the callback lacks MainActor isolation, the assertion fails. A thread observation is not a general proof of arbitrary actor permission.

**Why:** The function asserts an existing condition; it does not arrange for that condition to become true.

**Correction:** Standalone **compile-only** async boundary.

```swift
@MainActor final class Model { var count = 3 }
func callback(_ model: Model) async -> Int { await model.count }
```

**What changes:** The compiler establishes the required actor call. A synchronous integration that truly cannot await needs a documented callback guarantee, or a redesigned actor-bound callback type.

## Worked example

### Synchronous nonisolated does not offload CPU work

**Setup:** Standalone runnable script.

**Predict:** Does a nonisolated synchronous formatter leave its MainActor caller?

```swift
struct Formatter {
    func format(_ values: [Int]) -> String { values.sorted().map(String.init).joined() }
}
@MainActor func render() -> String { Formatter().format([3, 1, 2]) }
precondition(render() == "123")
```

**Outcome:** The helper runs synchronously in the caller's execution context.

**Why:** Nonisolated describes permission, not a work-placement mechanism. An expensive version could still hold up MainActor. An async `@concurrent` boundary with transferable inputs is the explicit off-actor option under this baseline; D14 verifies that contract.

## Interview questions

### What does nonisolated mean on an actor or globally isolated member?

**Answer:** The member can be used without the enclosing actor's permission, so it must not directly use protected mutable state.

**Example:** Standalone runnable declaration with immutable identity.

```swift
actor IdentifiedStore {
    nonisolated let name: String
    private var count = 0
    init(name: String) { self.name = name }
    nonisolated func label() -> String { "store: \(name)" }
}
let store = IdentifiedStore(name: "profiles")
precondition(store.label() == "store: profiles")
```

**Walkthrough:** The constant String supports independent access. The counter remains protected; the earlier rejected example cannot read it from a nonisolated method.

**Boundary:** Ordinary top-level declarations already default nonisolated here. On an actor member the spelling has a specific opt-out effect; it supplies neither locks nor offloading.

### When is assumeIsolated appropriate, and what makes it dangerous?

**Answer:** It is appropriate only when an existing synchronous integration contract guarantees the actor context. It traps if the assumption is wrong.

**Example:** Standalone runnable known-safe context. An explicit MainActor function provides the guarantee here; an ordinary async integration should simply use checked calls.

```swift
@MainActor final class Model { var count = 3 }
@MainActor func knownContext(_ model: Model) -> Int {
    MainActor.assumeIsolated { model.count }
}
precondition(knownContext(Model()) == 3)
```

**Walkthrough:** The annotation already establishes the context, so the assertion succeeds. This is a demonstration of what is asserted, not a reason to add redundant assertions to normal actor code.

**Boundary:** Do not run a failing assertion as an isolation repair. Framework callbacks need documented executor guarantees, not guesses based on a main-thread check.

### Why does a pure synchronous nonisolated helper still occupy its MainActor caller?

**Answer:** It executes inline. Changing access permission does not introduce suspension or a new execution boundary.

**Example:** Standalone runnable explicit alternative.

```swift
@concurrent func prepare(_ values: [Int]) async -> [Int] {
    precondition(#isolation == nil)
    return values.sorted()
}
@MainActor func render() async -> [Int] { await prepare([2, 1]) }
let values = await render()
precondition(values == [1, 2])
```

**Walkthrough:** `@concurrent` leaves the caller's actor, whereas the worked example's synchronous helper did not. The await resumes the actor caller with its permission when preparation finishes.

**Boundary:** Neither form promises a physical thread or automatic internal cancellation checks. A plain nonisolated async helper follows caller isolation with the baseline feature enabled.

### Why may a function have only one isolated parameter even if a caller passes one actor twice?

**Answer:** The declaration must have one statically valid isolation domain for all callers. It cannot depend on a particular caller making two actor arguments identical.

**Example:** Standalone **intentionally rejected** declaration.

```swift
actor Account { var balance = 0 }
func transfer(from source: isolated Account, to target: isolated Account) {
    target.balance += source.balance
}
```

**Walkthrough:** Two different accounts could be passed. The language cannot grant simultaneous arbitrary actor permission through two isolated parameters. The single-owner `charge` example above is permitted.

**Boundary:** Cross-account workflows need an explicit coordinator, reservation, or compensation protocol. Removing the keyword would not make unsynchronized access safe.

### Extension — What does Hashable UnownedTaskExecutor identify, and why does it neither retain the owner nor prove actor isolation? Check OS 27 availability.

**Answer:** It is an unowned task-executor identity usable under its equality/hash contract. Hashability requires OS 27 in the selected SDK; identity is neither ownership nor actor permission.

**Example:** Standalone **compile-only** specialist helper, with the deployment requirement explicit.

```swift
@available(anyAppleOS 27.0, *)
func distinctExecutors(_ executors: [UnownedTaskExecutor]) -> Int {
    Set(executors).count
}
```

**Walkthrough:** A set can deduplicate handles while their owners remain alive. The function does not construct an owner or authorize an actor call. A task executor is not interchangeable with an actor's serial executor.

**Boundary:** Keep owners alive externally and do not persist hashValue as an identifier. Executor equality cannot justify `assumeIsolated`. [SE-0523](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0523-hashable-unownedtask-executor.md)

## Focused exercise

**Task:** Choose a live isolated description or a pure snapshot-based description, then verify the invalid opt-out.

**Acceptance checks:**

- Live state is read through the actor boundary.
- The nonisolated helper only receives independent Int data.
- Direct protected-state access from a nonisolated member rejects.

**Solution:** Prepend **store**. Runnable. The first rejected pitfall is the negative control.

```swift
let store = Store()
let snapshot = await store.snapshot()
let live = await store.description()
let copied = store.describe(snapshot: snapshot)
precondition(live == copied && copied == "count: 3")
```

**Reasoning:** Both paths format the same captured value, but only the live description has actor permission. The helper makes no promise that the snapshot remains current after future mutations.

## In practice

**Choose it when:** Expose the least permission that preserves the invariant; keep work-placement policy explicit.

**Investigate by:** Identify the protected state and the exact caller contract before choosing an opt-out or assertion.

**Strong evidence:** Compile invalid clients separately and explain every runtime assertion with external integration evidence.

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
