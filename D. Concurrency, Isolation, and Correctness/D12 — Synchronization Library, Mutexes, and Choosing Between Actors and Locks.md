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

**Goal:** Protect one complete counter increment in a synchronous mutex critical section.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An actor owns state through an isolation domain with asynchronous access from elsewhere. A mutex provides synchronous mutual exclusion around its protected value. Immutable values avoid mutation conflicts, but publishing a changing snapshot still requires synchronization. Choose from the caller's access and progress requirements rather than treating actors and locks as interchangeable. [Synchronization.Mutex](https://developer.apple.com/documentation/synchronization/mutex)

## Rules and pitfalls

### Protect the whole invariant

A read-modify-write must occur under one lock. Separately locked getters and setters permit another update between them. Mutex.withLock has a synchronous closure, releases on normal or throwing exit, and cannot suspend.

**Common pitfall:** Moving an await into a critical section is invalid, while blocking for work that will run later can deadlock cooperative execution. Copy the necessary data under the lock, perform independent work outside, then revalidate before committing if needed.

### Keep unknown code and lock ordering outside

Keep critical sections short and avoid callbacks, cancellation methods, or destructors that may reenter. When multiple locks are unavoidable, define and enforce one acquisition order.

**Common pitfall:** A lock around a mutable reference does not prevent that reference escaping to an unsynchronized caller. Compiler acceptance of a closure result is not proof of a whole-type ownership contract; review every access and escape.

## Interview questions

### When is an actor a better fit than a mutex, and when is a mutex still appropriate?

An actor fits domain state with asynchronous operations and explicit ownership. A mutex fits small synchronous invariants needed by callbacks or synchronous APIs. Immutable publication can be simpler for read-heavy state if replacement is synchronized. Mutex requires macOS 15 / iOS 18 in this SDK.

### Why can wrapping everything in a lock be worse than modeling ownership explicitly?

It can hide unclear ownership, create contention, increase lock-order complexity, and call unknown code while holding a lock. Model a single owner first and lock a bounded state transition where synchronous access is actually needed. Do not serialize arbitrary I/O or long computation by default.

### Why can a short synchronous critical section be appropriate while a semaphore wait for future task work is unsafe? How do lock ordering and reentrant callbacks create deadlocks?

A short critical section waits for another currently running holder to finish bounded work. Waiting on a semaphore for future queued task work can consume every cooperative worker needed to make that work progress. Opposite lock order and reentrant callbacks add independent circular-wait risks.

### Can withLockIfAvailable fail spuriously? Distinguish outer nil from a successful closure returning nil; preserve the guarantee's Swift 6.0 status.

The Swift Mutex try-lock contract does not permit spurious failure; this guarantee dates to Swift 6.0. Outer nil means acquisition failed, while .some(nil) can mean the lock was acquired and the closure legitimately returned nil. Do not introduce a newer-version claim for the established guarantee.

## Focused exercise

**Task:** Protect a read-modify-write counter with one synchronous critical section. Explain why separately locked accessors fail and why the closure must not suspend.

**Acceptance checks:**

- All increments are retained after child completion.
- Each read-modify-write is protected by one withLock invocation.
- No closure suspends or exposes a mutable reference.

**Solution:**

Standalone checked Sendable owner; macOS 15 / iOS 18 or newer:

```swift
import Synchronization
@available(macOS 15, iOS 18, *)
final class Counter: Sendable {
    private let state = Mutex(0)
    func increment() { state.withLock { $0 += 1 } }
    func snapshot() -> Int { state.withLock { $0 } }
}
@main struct Probe {
    static func main() async {
        if #available(macOS 15, iOS 18, *) {
            let counter = Counter()
            await withTaskGroup(of: Void.self) { group in
                for _ in 0..<100 { group.addTask { counter.increment() } }
            }
            precondition(counter.snapshot() == 100)
        }
    }
}
```

**Reasoning:** The final class has immutable stored Mutex ownership; the mutex governs all mutable counter access, supporting checked Sendable rather than an unchecked assertion. A separate locked read and write would permit lost updates despite each access being individually protected.

## In practice

**Choose it when:** Synchronous callers need a short complete state transition.

**Investigate by:** Audit lock order, callbacks, reference escapes, and forward-progress dependencies.

**Strong evidence:** Compare actor ownership and mutex access under real caller constraints, then test whole invariants and contention.

### Try-lock and non-Sendable protected values — optional

#### Try-lock semantics and Swift 6.4 reading

withLockIfAvailable attempts immediate acquisition and returns nil if the mutex is held by another execution context. SE-0512 clarifies that it does not fail spuriously; Swift Evolution backdates this guarantee to Swift 6.0, so it is not a new 6.4 API. It promises neither fairness nor future availability. Do not replace a blocking design with a busy retry loop. [SE-0512](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0512-withlockifavailable-cannot-spuriously-fail.md)

An optional closure result adds a second optional layer:

```swift
import Synchronization

let state = Mutex(0)
let attempted: Int?? = state.withLockIfAvailable { _ -> Int? in nil }
switch attempted {
case .none: print("busy")
case .some(.none): print("acquired, result nil")
case .some(.some(let value)): print(value)
}
```

This uncontended example prints acquired, result nil. An outer nil means no closure execution; a successful closure may itself return nil or throw. Throwing still releases the lock.

#### The protected value need not itself be Sendable

Mutex is a Sendable owner of its protected state. The sending annotations model transfer into and out of the critical section; they do not grant permission to leak a retained mutable reference. Returning an independent Sendable snapshot is a clear API:

```swift
import Synchronization

final class ReferenceCounter: Sendable {
    private final class State {
        var value = 0
    }
    private let state = Mutex(State())
    func increment() -> Int {
        state.withLock {
            $0.value += 1
            return $0.value
        }
    }
}
```

The nested mutable class has no Sendable conformance. It stays owned by the mutex, and callers receive an Int. In a separate temporary probe, returning the retained reference directly from withLock compiled on the checked toolchain and two calls returned the same object. Compiler acceptance of that shape is not proof of safe encapsulation. Keep the reference private and audit every accessor; D6 covers the general sending rules.

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
