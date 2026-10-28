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

**Learning goals:**

- Distinguish a checked Sendable type, a Sendable closure, an isolation boundary, and a particular sending handoff.
- Follow mutable aliases and choose a value, actor, or synchronized owner for shared state.
- Explain weak-reference migration and explicit/conditional sendability without treating annotations as synchronization.

**Prerequisites:** [[D4 — Actors and Actor Isolation]]–[[D5 — Global Actors and `@MainActor`]] for isolation; value/reference semantics and closure capture from [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] and [[A8 — Closures, Escaping, and Capture Semantics]]. Mutex examples require macOS 15/iOS 18 or later.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A mutable object cannot safely be edited by independent tasks just because both references are declared `let`. The binding is fixed; the referenced object's contents may still change. The boundary needs a contract: share data whose type supports sharing, keep mutation behind an owner, or transfer one disconnected object so the source cannot keep using it unsafely.

`Sendable` is the type's promise that its values can safely cross isolation boundaries. `@Sendable` is a function-value contract that constrains captures. `sending` describes a particular handoff, which can be safe without making the entire type universally shareable. **Isolation** says where an operation may access state; sendability alone does not say which actor executes it. [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Checked values and immutable classes

**Setup:** Standalone runnable script. Every stored value supports safe transfer.

```swift
struct Configuration: Sendable { let endpoint: String; let retries: Int }
final class Settings: Sendable {
    let configuration: Configuration
    init(_ configuration: Configuration) { self.configuration = configuration }
}
let settings = Settings(Configuration(endpoint: "profiles", retries: 2))
let work = Task.detached { settings.configuration.retries }
let retries = await work.value
precondition(retries == 2)
```

**Walkthrough:** The immutable reference exposes only immutable Sendable state. The detached task can read it under checked conformance. Sendable does not insert a lock or copy this class instance.

### Mutable behavior can live behind an immutable synchronization owner

**Setup:** Standalone runnable script. Mutex supplies the synchronization contract.

```swift
import Synchronization
final class LockedCounter: Sendable {
    private let state = Mutex(0)
    func next() -> Int { state.withLock { value in value += 1; return value } }
}
let counter = LockedCounter()
async let first = Task.detached { counter.next() }.value
async let second = Task.detached { counter.next() }.value
let values = await [first, second]
precondition(values.sorted() == [1, 2])
```

**Walkthrough:** Checked class conformance is possible because its stored property is a constant Sendable mutex. Each method changes only protected storage. The conformance needs no `@unchecked` annotation. Returning a retained mutable reference from that storage would still require an ownership audit; D12 explains this limitation.

## Rules and pitfalls

### A constant reference or capture list does not freeze its object

**Common pitfall:** Capturing `[cache]` is expected to make an unsynchronized mutable class safe in an explicit Sendable callback.

**Mistake:** Standalone **intentionally rejected** function, matching the rubric's mutable-cache probe.

```swift
final class Cache { var store: [String: Int] = [:] }
func callback(_ cache: Cache) -> @Sendable () -> Void {
    { [cache] in cache.store["profile"] = 1 }
}
```

**Consequence:** Swift rejects the non-Sendable capture. If the check were bypassed, other references could mutate the same dictionary.

**Why:** The capture list copies a reference, not the object graph. `let` has the same limitation.

**Correction:** Standalone runnable actor-backed callback.

```swift
actor Cache {
    private var store: [String: Int] = [:]
    func set(_ key: String, value: Int) { store[key] = value }
    func get(_ key: String) -> Int? { store[key] }
}
func callback(_ cache: Cache) -> @Sendable () async -> Void {
    { await cache.set("profile", value: 1) }
}
let cache = Cache()
await callback(cache)()
let value = await cache.get("profile")
precondition(value == 1)
```

**What changes:** The callback safely shares an actor reference; the dictionary remains actor-owned. A value snapshot or a fully encapsulated Mutex owner would be different valid contracts.

### A transfer is invalidated by later conflicting alias use

**Shared setup — buffer boundary:** Prepend this to both versions. The class is deliberately not Sendable.

```swift
final class ScratchBuffer { var bytes: [UInt8] = [] }
@MainActor func consume(_ buffer: sending ScratchBuffer) -> Int {
    buffer.bytes.append(1)
    return buffer.bytes.count
}
```

**Common pitfall:** A handoff is considered safe while another source-side alias is still used afterward.

**Mistake:** Prepend **buffer boundary**. **Intentionally rejected**.

```swift
@concurrent func invalidAlias() async {
    let buffer = ScratchBuffer()
    let alias = buffer
    _ = await consume(buffer)
    alias.bytes.append(2)
}
```

**Consequence:** The compiler cannot prove exclusive access across the handoff and rejects it.

**Why:** Both names refer to one region of mutable state. A **region** groups values whose aliases must be considered together for transfer safety.

**Correction:** Prepend **buffer boundary**. Runnable; relinquish source-side access after the handoff.

```swift
@concurrent func transfer() async -> Int {
    let buffer = ScratchBuffer()
    buffer.bytes.append(0)
    return await consume(buffer)
}
let count = await transfer()
precondition(count == 2)
```

**What changes:** The new buffer is disconnected from other accessible state, and no source-side alias is used after sending. `sending` does not make every copyable value single-use; an Int remains reusable. [SE-0430](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md)

## Worked example

### A sending result must be independent of retained actor state

**Setup:** Prepend **buffer boundary**. **Compile-only** factory and consumer.

**Predict:** Does returning a fresh buffer impose the same risk as returning a buffer still cached by the actor?

```swift
@MainActor func makeBuffer() -> sending ScratchBuffer { ScratchBuffer() }
@concurrent func useFreshBuffer() async -> Int {
    let buffer = await makeBuffer()
    return await consume(buffer)
}
```

**Outcome:** The fresh result can transfer; `useFreshBuffer` returns one. The factory retains no alias.

**Why:** A sending result promises the handoff is safe at that boundary. The following **intentionally rejected** factory, with **buffer boundary** prepended, still retains the object:

```swift
@MainActor final class BufferStore {
    private let cached = ScratchBuffer()
    func current() -> sending ScratchBuffer { cached }
}
```

The actor could continue to access `cached`, so the returned object is not disconnected. Return an independent value or keep operations on the actor instead.

## Interview questions

### Why does let or a capture list fail to protect a mutable referenced object?

**Answer:** They preserve or copy the reference, leaving the same mutable object reachable through other aliases.

**Example:** Standalone runnable same-context example; it demonstrates aliasing without executing a data race.

```swift
final class Box { var value = 0 }
let original = Box()
let read = { [original] in original.value }
let alias = original
alias.value = 7
precondition(read() == 7)
```

**Walkthrough:** The capture still points to the object modified through `alias`. The closure sees seven, not a frozen copy of zero.

**Boundary:** A real immutable snapshot copies the required value data. A class with immutable bindings to mutable descendants needs the same whole-graph audit.

### Why is a Task's sending operation closure not the same API as an explicit @Sendable callback?

**Answer:** A sending task operation can take a disconnected region into new ownership; an explicit Sendable callback requires a capture contract suitable for sharing that function value.

**Example:** Standalone runnable handoff from a genuinely nonisolated execution context.

```swift
final class WorkBuffer { var count = 0 }
@concurrent func handOff() async -> Int {
    let buffer = WorkBuffer()
    let task = Task { buffer.count += 1; return buffer.count }
    return await task.value
}
let result = await handOff()
precondition(result == 1)
```

**Walkthrough:** The source does not access the buffer after handing its capture to the task. The handle is joined. There is no claim that WorkBuffer supports general concurrent sharing.

This separate **intentionally rejected** callback shows the stronger capture check:

```swift
final class WorkBuffer { var count = 0 }
func share(_ buffer: WorkBuffer) -> @Sendable () -> Int {
    { buffer.count }
}
```

**Boundary:** Neither `sending` nor `@Sendable` selects an actor by itself. A `Task` also needs explicit lifetime and error ownership; the transfer check does not supply those policies.

### When can a non-Sendable region be transferred, and which later alias uses invalidate it?

**Answer:** A disconnected region can transfer when the source relinquishes conflicting access. Later alias reads or writes, or an actor retaining the returned object, can invalidate that proof.

**Example:** Prepend **buffer boundary**. Runnable safe result transfer plus a reusable Sendable value.

```swift
@MainActor func fresh() -> sending ScratchBuffer { ScratchBuffer() }
@MainActor func acceptNumber(_ value: sending Int) -> Int { value }
@concurrent func runTransfers() async -> (Int, Int) {
    let buffer = await fresh()
    let count = await consume(buffer)
    let number = 7
    let received = await acceptNumber(number)
    return (count, received + number)
}
let result = await runTransfers()
precondition(result.0 == 1 && result.1 == 14)
```

**Walkthrough:** The factory's buffer is new and unretained, so it can be handed to MainActor. The Int can also be used after sending because its type already supports safe sharing.

**Boundary:** An existing alias that is never used again need not invalidate the handoff. The compiler reasons about uses and regions, not merely how many variable names exist.

### How do weak let storage and ImmutableWeakCaptures affect lifetime and sendability?

**Answer:** `weak let` fixes a weak reference's initial target while allowing it to observe nil after destruction. `ImmutableWeakCaptures` separately controls reassignment of explicit weak captures. Neither makes an unsynchronized referent Sendable.

**Example:** Standalone runnable lifetime example on Swift 6.4.

```swift
final class Owner { }
final class Watch {
    weak let owner: Owner?
    init(_ owner: Owner) { self.owner = owner }
}
var owner: Owner? = Owner()
let watch = Watch(owner!)
precondition(watch.owner != nil)
owner = nil
precondition(watch.owner == nil)
```

**Walkthrough:** The weak reference does not keep Owner alive. Its binding is immutable, but the observed value becomes nil. This is lifetime behavior, not synchronization.

The following **intentionally rejected** closure still captures a non-Sendable mutable object, even though its capture is weak:

```swift
final class Owner { var count = 0 }
func callback(_ owner: Owner) -> @Sendable () -> Int {
    { [weak owner] in owner?.count ?? 0 }
}
```

Explicit weak capture reassignment is a separate decision. This standalone **compile-only** callback is accepted under the baseline, without ImmutableWeakCaptures:

```swift
final class CaptureOwner { }
func clearLater(_ owner: CaptureOwner) -> () -> Void {
    { [weak owner] in if owner != nil { owner = nil } }
}
```

Recompile that same source with `-enable-upcoming-feature ImmutableWeakCaptures`: it is **intentionally rejected** because the capture is immutable under that setting. This is independent of accepting weak let storage. The negative control is recorded in validation.

**Boundary:** Code that reassigns a weak capture needs a separate `ImmutableWeakCaptures` migration check. Enabling that feature is distinct from the compiler accepting weak let; it does not grant access to isolated state. [SE-0481](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)

### Why does a direct MainActor class with ~Sendable differ from removing an inherited conformance?

**Answer:** Suppression can prevent inference for the declaration itself, but it cannot erase a superclass's existing Sendable conformance.

**Example:** Standalone **intentionally rejected** generic use. The declaration of the direct class is accepted; the Sendable requirement fails.

```swift
@MainActor final class Local: ~Sendable { }
func requireSendable<T: Sendable>(_ value: T) { }
@MainActor func check() { requireSendable(Local()) }
```

The inherited-conformance case instead rejects the attempted suppression itself:

```swift
@MainActor class Base { }
final class Child: Base, ~Sendable { }
```

**Walkthrough:** Direct global-actor isolation and inherited protocol commitments are different sources of conformance. The inherited commitment remains part of the subclass contract.

**Boundary:** The direct Local members remain MainActor-isolated despite suppression. A disconnected particular handoff can still be safe without a positive Sendable conformance.

### Extension — How do omitted conformance, ~Sendable, conditional Sendable, and @unchecked Sendable differ? What does -Wwarning ExplicitSendable audit?

**Answer:** Omission leaves inference rules applicable; suppression records a deliberate choice against relevant inference; conditional conformance grants the promise under constraints; unchecked conformance makes the author responsible for an audit the compiler cannot establish.

**Example:** Standalone runnable conditional container.

```swift
struct Envelope<Value>: ~Sendable { let value: Value }
extension Envelope: Sendable where Value: Sendable { }
func requireSendable<T: Sendable>(_ value: T) { }
requireSendable(Envelope(value: "profile"))
```

**Walkthrough:** Suppression does not prevent an explicit conditional conformance. String satisfies the constraint, so this Envelope supports sharing. `~Sendable` is not a generic negative constraint, an extension annotation, or valid actor suppression syntax. No experimental flag is needed in this selected compiler.

Public inference also depends on resilience. This standalone **compile-only** example checks internal and frozen public value types:

```swift
struct InternalSnapshot { let count: Int }
@frozen public struct FrozenSnapshot { public let count: Int }
func requireSendable<T: Sendable>(_ value: T) { }
func check() {
    requireSendable(InternalSnapshot(count: 1))
    requireSendable(FrozenSnapshot(count: 1))
}
```

A non-frozen public struct does not receive that same implicit commitment. The opt-in audit reports unstated public sendability decisions. Compile this standalone declaration with `-Wwarning ExplicitSendable` (expected warning, not an intended compiler error):

```swift
public struct PublicSnapshot { public let count: Int }
```

An audited legacy synchronization scheme may require unchecked conformance when the compiler cannot verify its discipline. This standalone **compile-only** example keeps every mutable access behind the same lock:

```swift
import Foundation
final class LegacyCounter: @unchecked Sendable {
    private let lock = NSLock()
    private var count = 0
    func next() -> Int {
        lock.lock()
        defer { lock.unlock() }
        count += 1
        return count
    }
}
```

**Boundary:** The unchecked annotation supplies no lock; the implementation and every future method must preserve the audit. Prefer checked Mutex-backed storage when available. The ExplicitSendable diagnostic group is an API audit, not proof that every access is race-free. [SE-0518](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0518-tilde-sendable.md)

## Focused exercise

**Task:** Repair the rubric's mutable Cache callback using an actor, a value snapshot, or a checked Mutex owner. Explain sharing versus transfer.

**Acceptance checks:**

- An explicit Sendable closure safely captures its owner.
- Two concurrent requests produce one and two, without assuming caller order.
- Substituting an unsynchronized mutable class rejects the capture contract.

**Solution:** Standalone runnable actor-backed service; the rejected cache callback above is the negative control.

```swift
actor Service {
    private var count = 0
    func next() -> Int { count += 1; return count }
}
func callback(for service: Service) -> @Sendable () async -> Int {
    { await service.next() }
}
let operation = callback(for: Service())
async let first = operation()
async let second = operation()
let results = await [first, second]
precondition(results.sorted() == [1, 2])
```

**Reasoning:** The captured actor reference is shareable while its integer remains isolated. Sorting the observations avoids inventing FIFO order. This is ongoing sharing through an owner, not a disconnected one-time buffer handoff.

## In practice

**Choose it when:** State precisely whether the boundary shares a value, captures an owner, or relinquishes a mutable region.

**Investigate by:** Follow every alias and inspect the actual function type. Verify a permitted transfer and a later conflicting use separately.

**Strong evidence:** Compare value, actor, and synchronized designs without using unchecked annotations to hide an unresolved ownership question.

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
