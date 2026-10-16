---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - sendable
  - strict-concurrency
---

# D6 — `Sendable`, `@Sendable`, and `sending`

**Goal:** Repair shared mutable service access with an actor and validate sendable callback capture.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Sendable is a checked promise about safe value transfer/sharing, not a method that copies or locks data. @Sendable applies capture requirements to a function value. Sending describes a particular handoff of a region that must no longer be accessed through invalidating aliases. Execution isolation determines where work runs. These contracts interact but answer different questions. [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)

## Rules and pitfalls

### Choose sharing or handoff explicitly

Use independent values, actors, immutable checked classes, or audited synchronized storage for sharing. A final class with let Mutex storage can support checked Sendable under the relevant SDK. Use sending for a genuine disconnected handoff rather than claiming an unsafe type is universally shareable.

**Common pitfall:** Let and capture lists copy a reference, not its mutable object. They do not repair cross-task mutation. @unchecked Sendable also adds no synchronization; audit the whole type and every escape before choosing it.

### Preserve explicit boundary contracts

Caller-isolated async avoids an implicit send under the baseline, but explicit sending still imposes its transfer requirement. A Task operation's sending closure can permit a one-time disconnected handoff; an explicit @Sendable callback promises a different sharing contract.

**Common pitfall:** Removing a diagnostic with an annotation without checking later alias use can leave the model wrong. Compile both the accepted handoff and a later use that must reject.

## Interview questions

### How do omitted conformance, ~Sendable, conditional Sendable, and @unchecked Sendable differ? What does -Wwarning ExplicitSendable audit?

Omitted conformance leaves inference rules applicable; ~Sendable suppresses relevant inference without a negative runtime conformance. Conditional conformance provides the promise only under constraints; unchecked claims a manually audited promise. ExplicitSendable warns about public API decisions under its supported rules, not every possible unsafe access.

### Why does a direct MainActor class with ~Sendable differ from removing an inherited conformance?

Direct explicit ~Sendable can suppress inference associated with that declaration under the current rules. It cannot erase an inherited conformance already established by a superclass. Model direct inference and inherited protocol commitments separately instead of using suppression as an inheritance opt-out.

### Why does let or a capture list fail to protect a mutable referenced object?

Both retain the same reference, so other aliases can still mutate the object. Immutable binding does not establish immutable reachable state. Use independent value data or a defined isolation/synchronization owner; copying a reference into a capture list does not add one.

### Why is a Task's sending operation closure not the same API as an explicit @Sendable callback?

A sending closure can transfer one disconnected captured region into task ownership when later aliases do not invalidate the transfer. @Sendable is a reusable function-value sharing promise with capture checks. Both can be safe under different conditions; neither by itself specifies the actor used for execution.

### When can a non-Sendable region be transferred, and which later alias uses invalidate it?

When the region is disconnected from concurrently accessible state and the transfer contract prevents later conflicting use. Access through a retained alias after sending, or returning mutable state still retained by an actor, can invalidate that proof. Sendable values remain reusable because their type already supports safe sharing.

### How do weak let storage and ImmutableWeakCaptures affect lifetime and sendability?

Weak let permits a fixed weak target to become unobservable after destruction. ImmutableWeakCaptures separately controls weak capture mutability. The referent's type and isolation still determine Sendable safety; weak lifetime observation does not make an unsynchronized mutable service safe.

## Focused exercise

**Task:** Repair an unsafe shared service with a value, actor, or checked Mutex-backed owner. Explain capture versus transfer and check the chosen boundary.

**Acceptance checks:**

- The explicit sendable closure safely captures an actor reference.
- Concurrent calls produce exactly one and two, with no required association to first/second.
- Replacing the actor with an unsynchronized mutable ordinary class must reject the same sendable-capture design.

**Solution:**

Standalone actor-backed service and explicit sendable callback:

```swift
actor Service {
    private var count = 0
    func next() -> Int { count += 1; return count }
}
func callback(for service: Service) -> @Sendable () async -> Int {
    { await service.next() }
}
@main struct Probe {
    static func main() async {
        let service = Service()
        let operation = callback(for: service)
        async let first = operation()
        async let second = operation()
        let results = await [first, second]
        precondition(results.sorted() == [1, 2])
    }
}
```

**Reasoning:** The service owns shared state, while the callback's capture is shareable because actor access remains isolated. No disconnected one-time region transfer is needed. Sorting test results avoids inventing actor FIFO ordering among independent child callers.

## In practice

**Choose it when:** Define whether a boundary shares a value or relinquishes access to one region.

**Investigate by:** Follow aliases before and after transfer and inspect the closure's actual function type.

**Strong evidence:** Compare actor, value, and checked synchronization designs under explicit caller and lifecycle requirements.

### Explicit suppression, synchronized storage, and transfers — optional

#### Swift 6.4: explicit non-sendability

`~Sendable` is a declaration-level audit decision for a struct, enum, or class. It is not a generic negative constraint, an extension annotation, or a synchronization guarantee. Actors do not accept the suppression syntax, and it cannot remove an inherited Sendable conformance, including one supplied by a global-actor base class. A base class using it does not permanently prohibit a separately justified Sendable subclass. [SE-0518](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0518-tilde-sendable.md)

This fails even though the stored `Int` is Sendable:

```swift
struct Local: ~Sendable { let value: Int }
func requireSendable<T: Sendable>(_ value: T) {}

func demo() {
    requireSendable(Local(value: 1)) // Expected conformance error.
}
```

A conditional conformance can still state when a container supports transfer:

```swift
struct Container<Value>: ~Sendable {
    let value: Value
}

extension Container: Sendable where Value: Sendable {}
```

A particular transfer can be valid without conformance:

```swift
struct Local: ~Sendable { let value: Int }

actor Sink {
    func accept(_ value: sending Local) -> Int { value.value }
}

@concurrent
func deliver(to sink: Sink) async -> Int {
    let value = Local(value: 1)
    return await sink.accept(value)
}
```

A direct global-actor annotation is a distinct case: the checked compiler accepts `@MainActor final class Local: ~Sendable {}` and rejects passing it to a generic Sendable requirement. Its members still obey MainActor isolation. Do not generalize the inherited-conformance restriction into a claim that every directly actor-isolated class rejects suppression.

For public API audits, Swift 6.4's opt-in `-Wwarning ExplicitSendable` diagnostic group reports declarations that have not stated their sendability decision. Use checked `Sendable` or deliberate suppression after reviewing the contract; an unchecked annotation is not an appropriate automatic fix. No experimental flag is needed for the `~Sendable` examples here.

#### Checked classes and synchronized storage

A final class can have immutable Sendable storage:

```swift
import Foundation

final class APIConfiguration: Sendable {
    let baseURL: URL
    let timeout: Duration

    init(baseURL: URL, timeout: Duration) {
        self.baseURL = baseURL
        self.timeout = timeout
    }
}
```

An ordinary mutable stored property fails checked class conformance:

```swift
final class Cache: Sendable {
    var store: [String: Int] = [:] // Expected mutable-storage error.
}
```

Mutable behavior can instead be contained in an immutable, Sendable synchronization object. This class uses **checked** conformance:

```swift
import Synchronization

final class LockedCache: Sendable {
    private let store = Mutex<[String: Int]>([:])

    func set(_ key: String, value: Int) {
        store.withLock { $0[key] = value }
    }

    func value(for key: String) -> Int? {
        store.withLock { $0[key] }
    }
}
```

`Mutex` itself supplies a synchronization-backed Sendable contract; its owner does not need `@unchecked` just because its methods mutate protected state. Keep all accesses inside the lock, return safe snapshots, and preserve invariants in one critical section. Checked conformance does not excuse leaking a retained mutable reference from the lock; D12 records a compiler-accepted escape and explains why encapsulation still needs review.

Legacy classes using separately locked mutable stored properties may need `@unchecked Sendable` when the compiler cannot verify their discipline. That author-supplied promise covers every property and access path, not just the method that triggered a diagnostic.

#### `sending` parameters, results, and aliases

The following declarations form one complete example:

```swift
final class ScratchBuffer {
    var bytes: [UInt8] = []
}

@MainActor
func consumeOnMain(_ buffer: sending ScratchBuffer) -> Int {
    buffer.bytes.append(1)
    return buffer.bytes.count
}

@MainActor
func makeBuffer() -> sending ScratchBuffer {
    ScratchBuffer()
}

@concurrent
func buildAndSend() async -> Int {
    let buffer = ScratchBuffer()
    buffer.bytes.append(0)
    return await consumeOnMain(buffer)
}

@concurrent
func useBuffer() async -> Int {
    let buffer = await makeBuffer()
    return await consumeOnMain(buffer)
}
```

`buildAndSend` returns 2; `useBuffer` returns 1. The returned object is disconnected from the factory's state. Existing aliases that are no longer used need not invalidate a transfer; later conflicting uses do.

This additional function fails when compiled with the preceding declarations:

```swift
@concurrent
func unsafeAlias() async {
    let buffer = ScratchBuffer()
    let alias = buffer
    _ = await consumeOnMain(buffer)
    alias.bytes.append(2) // Later source-side use invalidates transfer.
}
```

A result still retained in actor-isolated state is not disconnected. Reusing `ScratchBuffer`:

```swift
@MainActor
final class BufferStore {
    private let cached = ScratchBuffer()

    func current() -> sending ScratchBuffer {
        cached // Expected sending-result data-race diagnostic.
    }
}
```

A Sendable value does not have the same restriction on later use:

```swift
@MainActor
func acceptNumber(_ value: sending Int) -> Int { value }

@concurrent
func reuseNumber() async -> Int {
    let value = 7
    let received = await acceptNumber(value)
    return received + value
}
```

This returns 14. `sending` does not make `Int` or other copyable types single-use. For a non-Sendable region, the relevant question is whether source and destination uses can race.

## References

### Related guides

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[C1 — ARC fundamentals]]
- [[D1 — Structured Concurrency Fundamentals]]
- [[D4 — Actors and Actor Isolation]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[D13 — Swift 6 Strict Concurrency Migration]]
- [[D14 — Approachable Concurrency and Default Isolation]]

### Official sources

- [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
- [SE-0518](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0518-tilde-sendable.md)
- [The Swift Programming Language — Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [SE-0430 — Transferring parameters and results](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md)
- [SE-0461 — Async function isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)
- [SE-0481 — weak let](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)
- [Swift 6 migration guide — Common compiler errors](https://swift.org/migration/documentation/swift-6-concurrency-migration-guide/commonproblems/)
