---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actors
  - mutex
  - synchronization
  - swift6
---

# D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks

**Learning goals:**

- Choose actor ownership, synchronous mutual exclusion, or synchronized immutable snapshots from caller requirements.
- Protect the complete invariant and avoid suspension, reentrant callbacks, lock-order cycles, and mutable-reference escapes.
- Explain try-lock outcomes, nested optional results, and the established no-spurious-failure guarantee.

**Prerequisites:** [[D4 — Actors and Actor Isolation]] for actors, [[D6 — `Sendable`, `@Sendable`, and `sending`]] for Sendable owners, and [[D1 — Structured Concurrency Fundamentals]] for cooperative forward progress. Mutex requires macOS 15/iOS 18 in this SDK.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A **mutex** grants one execution context exclusive synchronous access to its protected value. The code executed while holding it is a **critical section**. An actor instead owns state behind an isolation boundary that callers may need to await. Choose from the access contract: a synchronous callback may need a short mutex operation; an asynchronous domain service often fits an actor.

Immutable data avoids concurrent mutation, but replacing a shared snapshot still needs safe publication. A lock should protect a complete state transition, not obscure who owns a mutable graph. [SE-0433](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Shared setup — checked counter

This final class has immutable Sendable Mutex storage and checked conformance. No mutable reference escapes.

```swift
import Synchronization
final class Counter: Sendable {
    private let state = Mutex(0)
    func increment() { state.withLock { $0 += 1 } }
    func snapshot() -> Int { state.withLock { $0 } }
}
```

**Setup:** Prepend **checked counter**. Runnable.

```swift
let counter = Counter()
counter.increment()
precondition(counter.snapshot() == 1)
```

**Walkthrough:** withLock acquires, lends protected mutable access to a synchronous closure, and releases on normal or throwing exit. The type alone does not combine separate calls into one compound transaction.

## Rules and pitfalls

### A critical section cannot suspend

**Common pitfall:** Await is moved inside a lock so an async workflow appears atomic.

**Mistake:** Standalone **intentionally rejected** async closure.

```swift
import Synchronization
func invalid() async {
    let state = Mutex(0)
    await state.withLock { value in
        await Task.yield()
        value += 1
    }
}
```

**Consequence:** The closure cannot satisfy withLock's synchronous contract. Blocking on future task work instead would introduce a forward-progress/deadlock risk.

**Why:** Another task must not be needed to resume code that holds the resource preventing that other task from progressing.

**Correction:** Prepend **checked counter**. Runnable synchronous transition followed by independent async work.

```swift
let counter = Counter()
counter.increment()
let snapshot = counter.snapshot()
await Task.yield()
let prepared = "count: \(snapshot)"
precondition(prepared == "count: 1")
```

**What changes:** The lock is already released before the await. If a later commit depends on this snapshot still being current, revalidate a version under the lock; this read-only preparation makes no live-state guarantee.

### A protected reference can still escape

**Common pitfall:** A Mutex around a class is assumed to protect the object after a method returns its retained reference.

**Mistake:** Standalone runnable alias probe on the selected compiler; it does not execute concurrent unsafe mutation.

```swift
import Synchronization
final class Payload { var count = 0 }
let state = Mutex(Payload())
let first = state.withLock { $0 }
let second = state.withLock { $0 }
precondition(first === second)
```

**Consequence:** Both callers receive the same mutable object. A caller can access it without holding the lock.

**Why:** The wrapper's local lock discipline does not by itself enforce encapsulation of every returned reference. Compiler acceptance is narrower than a whole-type ownership proof.

**Correction:** Standalone runnable owner returning an independent scalar.

```swift
import Synchronization
final class ReferenceCounter: Sendable {
    private final class State { var value = 0 }
    private let state = Mutex(State())
    func next() -> Int {
        state.withLock { $0.value += 1; return $0.value }
    }
}
let counter = ReferenceCounter()
precondition(counter.next() == 1 && counter.next() == 2)
```

**What changes:** The mutable reference remains private, and every mutation stays under the mutex. Callers receive only Int snapshots; the protected value itself need not be Sendable.

### Separately locked accessors do not protect read-modify-write

**Common pitfall:** A locked getter plus a locked setter is expected to preserve every increment.

**Mistake:** Standalone **compile-only** owner.

```swift
import Synchronization
final class SplitCounter: Sendable {
    private let state = Mutex(0)
    func get() -> Int { state.withLock { $0 } }
    func set(_ value: Int) { state.withLock { $0 = value } }
    func increment() { set(get() + 1) }
}
```

**Consequence:** Two callers can read zero then both write one.

**Why:** The invariant spans both calls, but neither lock acquisition covers the whole transition.

**Correction:** Prepend **checked counter**. Runnable concurrent callers.

```swift
let counter = Counter()
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 { group.addTask { counter.increment() } }
}
precondition(counter.snapshot() == 100)
```

**What changes:** Each increment's read and write share one acquisition. The group joins every caller before the snapshot.

## Worked example

### Invoke unknown callbacks after unlocking

**Setup:** Standalone runnable model.

**Predict:** Can a notification callback call back into the same owner?

```swift
import Synchronization
final class Owner: Sendable {
    private let state = Mutex(0)
    func snapshot() -> Int { state.withLock { $0 } }
    func update(notify: (Int) -> Void) {
        let value = state.withLock { $0 += 1; return $0 }
        notify(value)
    }
}
let owner = Owner()
owner.update { value in precondition(owner.snapshot() == value) }
```

**Outcome:** The reentrant read succeeds because notify runs after the critical section.

**Why:** Calling it inside withLock would try to reacquire the same nonrecursive mutex. Cancellation actions and final resource destruction can also invoke unknown code. Remove the selected value/resource under the lock and act after unlocking, as D9's adapter does.

## Interview questions

### When is an actor better than a mutex, and when is a mutex appropriate?

**Answer:** Prefer an actor for asynchronous domain ownership; use a mutex for short synchronous invariants required by synchronous callers. Immutable snapshots can simplify reads when replacement is synchronized.

**Example:** Standalone runnable synchronized publication.

```swift
import Synchronization
struct Snapshot: Sendable { let revision: Int; let names: [String] }
final class Published: Sendable {
    private let state = Mutex(Snapshot(revision: 0, names: []))
    func replace(_ value: Snapshot) { state.withLock { $0 = value } }
    func read() -> Snapshot { state.withLock { $0 } }
}
let published = Published()
published.replace(Snapshot(revision: 1, names: ["Lea"]))
precondition(published.read().names == ["Lea"])
```

**Walkthrough:** Readers receive immutable independent values, while the changing shared slot is protected. An actor could expose equivalent state through awaited methods if that fits the callers.

**Boundary:** A large copy or expensive transformation still belongs outside a short critical section. Do not choose locks simply to avoid reasoning about state ownership.

### Why can wrapping everything in a lock be worse than modeling ownership explicitly?

**Answer:** Broad locking can hide reference escapes, serialize unrelated work, and introduce contention or lock-order cycles. A defined owner and a small invariant are easier to audit.

**Example:** Standalone **compile-only; do not concurrently call the two paths**. They acquire the same locks in opposite orders.

```swift
import Synchronization
final class OppositeOrder: Sendable {
    private let first = Mutex(0), second = Mutex(0)
    func pathA() { first.withLock { _ in second.withLock { $0 += 1 } } }
    func pathB() { second.withLock { _ in first.withLock { $0 += 1 } } }
}
```

**Walkthrough:** A can hold first while waiting for second, as B holds second while waiting for first. Neither can finish. One combined state/lock or a consistently enforced acquisition order removes that particular cycle.

**Boundary:** Do not execute the deadlock as a normal study test. A universal lock-order policy still needs auditing of callbacks and resource destruction.

### Why is a short synchronous critical section appropriate while waiting on a semaphore for future task work is unsafe? How do reentrant callbacks create deadlocks?

**Answer:** A short lock waits for an existing holder to finish bounded synchronous work. A semaphore may wait for queued task work that cannot run because all workers are blocked. Reentrancy adds a separate cycle when a callback tries to acquire a lock its caller still holds.

**Example:** Standalone **compile-only; do not call with a reentrant callback**.

```swift
import Synchronization
final class UnsafeNotifier: Sendable {
    private let state = Mutex(0)
    func read() -> Int { state.withLock { $0 } }
    func update(notify: () -> Void) { state.withLock { _ in notify() } }
}
```

**Walkthrough:** If notify calls read, it attempts the same nonrecursive lock before update releases it. The worked example instead obtains a snapshot under lock and invokes the callback afterward.

**Boundary:** A short section must actually be bounded and avoid unknown code. D1 explains why wrapping a blocking wait in a synchronous helper does not restore cooperative forward progress.

### Extension — Can withLockIfAvailable fail spuriously? Distinguish outer nil from a successful closure returning nil.

**Answer:** The Swift Mutex try-lock contract does not permit spurious failure; this guarantee dates to Swift 6.0. Outer nil means acquisition failed, while some(nil) can mean the acquired closure legitimately returned nil.

**Example:** Standalone runnable uncontended case.

```swift
import Synchronization
let state = Mutex(0)
let attempt: Int?? = state.withLockIfAvailable { _ -> Int? in nil }
switch attempt {
case .none: preconditionFailure("uncontended acquisition failed")
case .some(.none): break
case .some(.some): preconditionFailure("unexpected closure result")
}
enum Failure: Error { case expected }
do {
    try state.withLock { _ in throw Failure.expected }
} catch Failure.expected { }
let afterThrow = state.withLockIfAvailable { $0 }
precondition(afterThrow == 0)
```

**Walkthrough:** The first attempt acquires the lock and wraps the closure's nil in an outer some. Throwing from a later closure still releases the lock, allowing the final uncontended acquisition.

**Boundary:** The contract promises neither fairness nor a successful future retry. Do not turn it into a busy-spin replacement for a blocking design or call it recursively on the same held mutex. SE-0512 clarifies the established guarantee rather than introducing a new 6.4 API.

## Focused exercise

**Task:** Implement a synchronous, checked Sendable counter and verify 100 concurrent increments.

**Acceptance checks:**

- One acquisition covers every read-modify-write.
- No closure suspends or returns retained mutable state.
- The final snapshot follows joining all callers.

**Solution:** Prepend **checked counter**. Runnable.

```swift
let counter = Counter()
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 { group.addTask { counter.increment() } }
}
precondition(counter.snapshot() == 100)
```

**Reasoning:** The mutex protects the transition, while the group defines completion. This small value range cannot overflow. The separately locked getter/setter counter remains incorrect despite each individual access being protected.

## In practice

**Choose it when:** Synchronous callers need a short complete state transition and ownership can stay encapsulated.

**Investigate by:** Audit every accessor, escape, callback, destructor, and multi-lock path.

**Strong evidence:** Compare actor and mutex designs under actual caller constraints, and distinguish language acceptance from a whole-type encapsulation argument.

## References

### Related guides

- [[D4 — Actors and Actor Isolation]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D13 — Swift 6 Strict Concurrency Migration]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [Synchronization.Mutex](https://developer.apple.com/documentation/synchronization/mutex)
- [SE-0512](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0512-withlockifavailable-cannot-spuriously-fail.md)
- [Swift.org. "Announcing Swift 6." Swift.org Blog](https://swift.org/blog/announcing-swift-6/)
- [GitHub. "Synchronous Mutual Exclusion Lock." Swift Evolution SE-0433](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md)
- [Swift.org. "Concurrency." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
