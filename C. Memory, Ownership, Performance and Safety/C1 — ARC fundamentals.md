---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - safety
  - arc
---

# C1 — ARC Fundamentals

**Learning goals:**

- Trace strong ownership, weak observations, unowned lifetime obligations, and external registrations through concrete graphs.
- Separate callback lifetime, task/registration cancellation, deinitialization, cleanup completion, and physical memory.
- Choose explicit teardown and verify its contract without relying on exact lexical destruction timing.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] for reference identity, [[A8 — Closures, Escaping, and Capture Semantics]] for captured references, and [[A6 — `defer`, Early Exits, and Cleanup Semantics]] for explicit cleanup. The timer extension uses [[D5 — Global Actors and `@MainActor`]]’s MainActor contract; [[C8 — Copyable, noncopyable types, and `~Copyable`]] explains unique owners.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A screen stores a callback, and the callback captures the screen. Even after navigation removes the last application reference, those two strong edges can keep each other alive. **ARC**, automatic reference counting, preserves class instances while strong ownership remains; it does not discover and collect unreachable strong cycles.

Draw the graph from every owner, including closures, tasks, and framework registrations. A **weak** reference observes an object without keeping it alive and can later read nil. A checked **unowned** reference also avoids ownership, but reading it requires its referent to remain alive. It traps if that lifetime promise is broken. [Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)

Dropping one strong reference, destroying an object, completing actor-isolated teardown, and returning memory to the operating system are separate events. Use explicit shutdown when a caller needs a completion guarantee.

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### A second strong owner outlives the first binding

Copying a class reference gives another binding access to the same object. Clearing one binding does not clear the other. Start with this standalone example:

```swift
final class Profile { var name = "Ada" }
var first: Profile? = Profile()
let second = first!
first?.name = "Grace"
precondition(second.name == "Grace")
first = nil
precondition(second.name == "Grace")
```

**Walkthrough:** first and second initially refer to one Profile, so the name change is visible through second. Setting first to nil removes that reference; second still provides the object needed for the final read. This establishes the ownership behavior without trying to predict an exact destruction point.

### Observe destruction with a bounded probe

The next examples also check release. Reading a weak reference temporarily supplies a strong reference, and the compiler can release a strong value after its last use. The small observation helpers below keep those effects out of the result being checked.

Shared ARC setup: ARCLog records destruction without retaining the observed object; Observed stores only a weak reference. Loading a weak reference creates a temporary strong reference when the object exists. The no-inline inspect helper confines that temporary to a separate call: these probes return only scalars or Void from it, never the object. This synchronous fixture has no isolated deinitializer or native resource.

```swift
final class ARCLog { var events: [String] = [] }
struct Observed<Value: AnyObject> {
    weak var value: Value?
    init(_ value: Value) { self.value = value }
    @inline(never)
    func inspect<Result>(_ body: (Value?) -> Result) -> Result { body(value) }
}
final class Sample {
    let id: Int
    let log: ARCLog
    init(_ id: Int, log: ARCLog) {
        self.id = id
        self.log = log
    }
    deinit { log.events.append("Sample destroyed") }
}
@inline(never)
func releaseAliases(log: ARCLog) -> Observed<Sample> {
    var first: Sample? = Sample(7, log: log)
    var second = first
    let observed = Observed(first!)
    first = nil
    withExtendedLifetime(second) {
        precondition(second?.id == 7 && observed.inspect { $0 != nil })
        precondition(log.events.isEmpty)
    }
    second = nil
    return observed
}
func strongAliasProbe() {
    let log = ARCLog()
    let observed = releaseAliases(log: log)
    precondition(observed.inspect { $0 == nil })
    precondition(log.events == ["Sample destroyed"])
}
strongAliasProbe()
```

**Walkthrough:** Clearing first removes one edge. withExtendedLifetime keeps second’s strong ownership through the observations; merely leaving a binding in lexical scope would not promise that timing. Clearing second removes the remaining stored owner. The separate releaseAliases call then ends any temporary strong reads before the caller checks destruction. The log records one destruction; it measures neither retain-count instructions nor physical memory.

### A saved bound method is another owner

Prepend ARC setup. A bound method value keeps its receiver alive, just as a closure that needs that receiver can.

```swift
final class Counter {
    var value = 0
    func increment() { value += 1 }
}
func boundMethodProbe() {
    var counter: Counter? = Counter()
    let observed = Observed(counter!)
    var action: (() -> Void)? = counter!.increment
    counter = nil
    action?()
    precondition(observed.inspect { $0?.value == 1 })
    action = nil
    precondition(observed.inspect { $0 == nil })
}
boundMethodProbe()
```

**Walkthrough:** The saved action is a root even after counter becomes nil. Removing a callback from its original owner would not remove an independently retained copy of that callback. Follow every retaining edge.

## Rules and pitfalls

### A deinitializer cannot remove the cycle preventing its entry

**Common pitfall:** Expecting deinit to clear a stored closure that strongly captures the same owner. Prepend ARC setup.

**Mistake:** Standalone runnable example.

```swift
final class Screen {
    let log: ARCLog
    var callback: (() -> Void)?
    init(log: ARCLog) { self.log = log }
    func install() { callback = { self.render() } }
    func render() {}
    deinit {
        callback = nil
        log.events.append("Screen destroyed")
    }
}
func cycleProbe() {
    let log = ARCLog()
    var screen: Screen? = Screen(log: log)
    let observed = Observed(screen!)
    screen!.install()
    screen = nil
    precondition(observed.inspect { $0 != nil } && log.events.isEmpty)
    observed.inspect { $0?.callback = nil }  // Explicitly break the test's cycle.
    precondition(observed.inspect { $0 == nil } && log.events == ["Screen destroyed"])
}
cycleProbe()
```

**Consequence:** Removing the external screen leaves Screen → callback → Screen alive. The deinit cleanup cannot run yet.

**Why:** ARC still sees strong ownership through the closure. Cleanup inside an unreachable deinitializer does not remove that ownership first.

**Correction:** When a callback may legitimately outlive its owner and should do nothing afterward, capture weakly. Prepend ARC setup.

```swift
final class Screen {
    private(set) var renders = 0
    var callback: (() -> Void)?
    func install() { callback = { [weak self] in self?.renders += 1 } }
}
func weakCallbackProbe() {
    var screen: Screen? = Screen()
    let observed = Observed(screen!)
    screen!.install()
    let saved = screen!.callback!
    saved()
    precondition(screen!.renders == 1)
    screen = nil
    precondition(observed.inspect { $0 == nil })
    saved()  // The chosen post-owner behavior is no operation.
}
weakCallbackProbe()
```

**What changes:** The callback no longer supplies a strong return edge. Its later invocation tolerates absence. Weakening ownership is a behavior decision: work that must keep its dependency alive needs a different contract.

### A parent-to-child edge does not prove the reverse lifetime

**Common pitfall:** Letting a child escape while its parent is referenced unowned. Compile-only; the post-parent access would trap.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
final class Account { let name = "Ada" }
final class Receipt {
    unowned let account: Account
    init(account: Account) { self.account = account }
    func heading() -> String { account.name }
}
func invalidLifetime() -> String {
    var account: Account? = Account()
    let receipt = Receipt(account: account!)
    account = nil
    return receipt.heading()
}
```

**Consequence:** receipt remains usable as an object, but its heading operation would read a destroyed unowned referent.

**Why:** An independent caller owns the child. Nothing in this type prevents that caller from outliving Account.

**Correction:** If absence is allowed, make it part of the result. This standalone correction uses a weak reference.

```swift
final class Account { let name = "Ada" }
final class Receipt {
    weak let account: Account?
    init(account: Account) { self.account = account }
    func heading() -> String? { account?.name }
}
var account: Account? = Account()
let receipt = Receipt(account: account!)
precondition(receipt.heading() == "Ada")
account = nil
precondition(receipt.heading() == nil)
```

**What changes:** The escaped child can represent its missing parent. Alternatively retain Account strongly when the dependency must remain available; review whether the reverse strong edge would create a cycle.

### Weak capture does not end an external registration

**Common pitfall:** Registering a repeating timer, then relying only on the weak owner disappearing. Compile-only; this forgotten registration is not started.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
import Foundation

@MainActor final class Refresh {
    private var timer: Timer?
    private(set) var ticks = 0
    func start() {
        let next = Timer(timeInterval: 60, repeats: true) { [weak self] _ in
            MainActor.assumeIsolated { self?.ticks += 1 }
        }
        timer = next
        RunLoop.main.add(next, forMode: .common)
    }
}
```

**Consequence:** The weak callback does not retain Refresh, but the run loop can still own the repeating timer and its scheduled work.

**Why:** Object ownership and registration lifetime are separate. A weak callback can stop affecting its owner while leaving a useless registration active.

**Correction:** Define start/stop and invalidate replacement registrations explicitly. This shared PeriodicRefresh setup is compile-only; its private timer is registered only on the main run loop.

```swift
import Foundation

@MainActor
final class PeriodicRefresh {
    private var timer: Timer?
    private(set) var ticks = 0

    func start(interval: TimeInterval = 1) {
        precondition(interval > 0)
        stop()
        let next = Timer(timeInterval: interval, repeats: true) { [weak self] _ in
            // This timer is private and scheduled only on the main run loop.
            MainActor.assumeIsolated { self?.ticks += 1 }
        }
        timer = next
        RunLoop.main.add(next, forMode: .common)
    }

    func stop() {
        timer?.invalidate()
        timer = nil
    }

    isolated deinit { timer?.invalidate() }
}
```

**What changes:** start first stops any previous registration. stop is idempotent and invalidates the timer. Isolated deinit is a cleanup backstop on the owning actor, while callers needing timely cleanup call stop. The checked MainActor assumption relies on this private scheduling contract; it does not schedule or validate arbitrary callbacks. [Timer invalidation](https://developer.apple.com/documentation/foundation/timer/invalidate())

## Worked example

### Two objects survive through one callback edge

**Setup:** Prepend ARC setup. Owner strongly stores Child; Child’s closure strongly captures Owner. makeCycle returns only weak observations, so the retaining graph is visible without adding an owning probe.

**Predict:** Which edge must stop remove, and must the two deinitializers run in a particular order?

```swift
final class Owner {
    let log: ARCLog
    var child: Child?
    private(set) var completions = 0
    init(log: ARCLog) { self.log = log }
    func finish() { completions += 1 }
    func stop() {
        child?.onDone = nil
        child = nil
    }
    deinit { log.events.append("Owner destroyed") }
}
final class Child {
    let log: ARCLog
    var onDone: (() -> Void)?
    init(log: ARCLog) { self.log = log }
    deinit { log.events.append("Child destroyed") }
}
@inline(never)
func makeCycle(log: ARCLog) -> (Observed<Owner>, Observed<Child>) {
    let owner = Owner(log: log)
    let child = Child(log: log)
    owner.child = child
    child.onDone = { owner.finish() }
    return (Observed(owner), Observed(child))
}
func cycleStates() {
    let log = ARCLog()
    let (owner, child) = makeCycle(log: log)
    precondition(owner.inspect { $0 != nil } && child.inspect { $0 != nil } && log.events.isEmpty)
    child.inspect { $0?.onDone?() }
    precondition(owner.inspect { $0?.completions == 1 })
    owner.inspect { $0?.stop() }
    precondition(owner.inspect { $0 == nil } && child.inspect { $0 == nil })
    precondition(log.events.sorted() == ["Child destroyed", "Owner destroyed"])
}
cycleStates()
```

**Outcome:** Both survive after makeCycle returns. stop breaks the callback’s return edge and releases the child slot; both weak observations then become nil in this synchronous fixture.

**Why:** The test checks one event per object without prescribing destruction order. inspect releases its temporary strong observation before returning, so an earlier weak read does not keep the graph alive during the later check. Bounded calls and explicit edge removal make this a useful ownership probe, not a universal lexical-scope or resident-memory rule.

## Interview questions

### What proves an escaped child cannot outlive its unowned referent?

**Answer:** Only an enforced lifetime relationship that makes every unowned access valid. Parent ownership alone is insufficient when another caller can retain the child. If a child must always have its dependency, a strong dependency can express that contract directly.

**Example:** Standalone runnable example.

```swift
final class Account { let name = "Ada" }
final class Receipt {
    let account: Account
    init(account: Account) { self.account = account }
    func heading() -> String { account.name }
}
var external: Account? = Account()
weak let observed = external
var receipt: Receipt? = Receipt(account: external!)
external = nil
precondition(receipt?.heading() == "Ada" && observed != nil)
receipt = nil
precondition(observed == nil)
final class UnownedReceipt {
    unowned let account: Account
    init(account: Account) { self.account = account }
    func heading() -> String { account.name }
}
func boundedUnownedUse() -> String {
    let account = Account()
    let receipt = UnownedReceipt(account: account)
    return withExtendedLifetime(account) { receipt.heading() }
}
precondition(boundedUnownedUse() == "Ada")
```

**Walkthrough:** Receipt deliberately extends Account’s lifetime. Once the last Receipt disappears, this fixture has no remaining owner. In boundedUnownedUse, the child does not escape and withExtendedLifetime keeps Account alive through the unowned access. The weak alternative in the pitfall instead permits an absent account.

**Boundary:** This changes lifetime policy. If Account also strongly owns Receipt, introduce an explicit teardown or a justified weak edge; do not replace it with unowned merely to avoid optional handling.

### How can weak let observe nil, and why is ImmutableWeakCaptures a separate setting?

**Answer:** weak let fixes the weak reference’s initial target without requiring that target to stay alive. ARC can still make reads observe nil. The separate ImmutableWeakCaptures opt-in controls reassignment of explicit weak closure captures; accepting weak let alone does not enable that policy.

**Example:** Standalone runnable example.

```swift
final class Target { let id = 7 }
var target: Target? = Target()
weak let observed = target
precondition(observed?.id == 7)
target = nil
precondition(observed == nil)
func clearingReader(_ target: Target) -> () -> Int? {
    { [weak target] in
        target = nil
        return target?.id
    }
}
let retained = Target()
precondition(clearingReader(retained)() == nil)
precondition(retained.id == 7)
```

**Walkthrough:** The weak observation becomes nil when its last strong owner disappears. The baseline also permits reassignment of the closure’s weak capture, which does not mutate retained itself. [weak let and capture policy](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)

**Boundary:** Weakness changes ownership, not the referent’s actor/Sendable contract. Adding ImmutableWeakCaptures rejects clearingReader’s capture assignment; the declaration’s weak let assignment is independently prohibited.

Prepend that Target setup. Reassigning the weak let is **intentionally rejected**:

```swift
observed = nil
```

This standalone capture mutation is **intentionally rejected** with the additional `-enable-upcoming-feature ImmutableWeakCaptures` flag:

```swift
final class Target {}
func reader(_ target: Target) -> () -> Void {
    { [weak target] in target = nil }
}
```

### Why is a weak reference alone insufficient evidence that actor-isolated cleanup completed?

**Answer:** A weak observation is a lifetime observation, not a completion acknowledgment. An isolated deinitializer may need executor scheduling, and external resources have their own cleanup. Await an explicit shutdown operation when completion matters.

**Example:** Standalone runnable example.

```swift
actor Resource {
    private var closed = false
    func close() { closed = true }
    func isClosed() -> Bool { closed }
}
@MainActor final class Session {
    let resource: Resource
    init(resource: Resource) { self.resource = resource }
    func shutdown() async { await resource.close() }
}
let resource = Resource()
var session: Session? = Session(resource: resource)
weak let observed = session
await session!.shutdown()
let closed = await resource.isClosed()
precondition(closed)
session = nil
precondition(observed == nil)
```

**Walkthrough:** The awaited method establishes resource closure before the caller continues; nil observation is checked separately. This example deliberately does not use deinit as its completion signal. PeriodicRefresh shows a synchronous isolated-deinit backstop. [Isolated synchronous deinitialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0371-isolated-synchronous-deinit.md)

**Boundary:** The fixture models an idempotent close with no network or OS handle. A real shutdown must specify failure, cancellation, concurrent users, and whether completion includes all downstream cleanup.

### Extension: How should the timer owner connect to a UI lifecycle?

**Answer:** Choose an explicit product lifecycle event for starting and stopping work. The timer owner remains responsible for replacement and invalidation; the view controller only calls that policy.

**Example:** Compile-only iOS example. Prepend PeriodicRefresh setup; use the iOS SDK and an arm64-apple-ios27.0 target in place of the macOS SDK/target.

```swift
import UIKit

@MainActor final class FeedViewController: UIViewController {
    private let refresh = PeriodicRefresh()
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        refresh.start()
    }
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        refresh.stop()
    }
}
```

**Walkthrough:** This adapter calls the complete PeriodicRefresh implementation from the corrected registration pitfall. It is object-compiled against the iOS 27 SDK; it does not present a real screen. Separate macOS fixture controls manually fire the actual private timer on MainActor and inspect replacement/stop behavior.

**Boundary:** Not every disappearing view should stop the same work. Weak capture alone also does not cancel a Task loop; use [[A8 — Closures, Escaping, and Capture Semantics|A8]]/[[D2 — Task Hierarchy, Cancellation, and Priorities|D2]]/[[F1 — Testing Strategy in Modern Swift|F1]]’s explicit cancellation and joining contracts for that alternative.

### Extension: Does a unique owner make its class payload uniquely referenced?

**Answer:** No. Uniqueness of a noncopyable owner does not erase references stored elsewhere. Cloning a copyable class reference still shares its object.

**Example:** Standalone runnable example.

```swift
final class Payload { var value = 1 }
func uniqueOwnerProbe() {
    let payload = Payload()
    let first = UniqueBox(payload)
    let second = first.clone()
    second.value.value = 9
    precondition(first.value === payload && second.value === payload)
    precondition(payload.value == 9)
}
uniqueOwnerProbe()
```

**Walkthrough:** The boxes have separate ownership, but payload, first.value, and second.value identify one object. [[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] contrasts this with an Array payload’s independent value mutations. [UniqueBox](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0517-uniquebox.md)

**Boundary:** These APIs require Apple OS 27 in this SDK. No deep-copy, thread-safety, thread-affine destruction, or allocation improvement follows from the wrapper’s ownership.

## Focused exercise

**Task:** Trace a strong cycle, invoke its callback, remove it, and define later trigger behavior.

**Acceptance checks:**

- The installed callback increments while both objects exist.
- Only weak observations remain outside the cycle, yet both objects stay alive before stop.
- stop releases both objects once; a trigger holding only weak observation then does nothing.

**Solution:** Prepend ARC setup, then the worked example’s Owner/Child/makeCycle declarations. The helper’s earlier probe is independent of this exercise’s log.

```swift
let exerciseLog = ARCLog()
let (owner, child) = makeCycle(log: exerciseLog)
let trigger = { child.inspect { $0?.onDone?() } }
trigger()
precondition(owner.inspect { $0?.completions == 1 })
precondition(
    owner.inspect { $0 != nil } && child.inspect { $0 != nil } && exerciseLog.events.isEmpty)
owner.inspect { $0?.stop() }
precondition(owner.inspect { $0 == nil } && child.inspect { $0 == nil })
precondition(exerciseLog.events.sorted() == ["Child destroyed", "Owner destroyed"])
trigger()
precondition(exerciseLog.events.count == 2)
```

**Reasoning:** Observed stores no strong edge; each inspect call temporarily retains a live referent only while that call runs. Clearing onDone breaks the callback’s strong owner capture and clearing child removes the other stored edge. The saved trigger captures only a weak observation; retaining the original strong callback instead would create a different graph.

## In practice

**Choose it when:** Design resource ownership and explicit teardown before relying on deinitialization.

**Investigate by:** Draw every strong edge, then use bounded weak probes, cleanup acknowledgments, and memory tools to answer separate lifetime and allocation questions.

**Strong evidence:** Verify the post-owner callback contract, external unregister/invalidate behavior, and cleanup completion without assuming one lexical release order or memory returned to the OS.

## References

### Related guides

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A3 — Initialization model]]
- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[E2 — Foundation and Objective-C Bridging Behavior]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

### Official sources

- [Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- [Timer](https://developer.apple.com/documentation/foundation/timer)
- [invalidate](<https://developer.apple.com/documentation/foundation/timer/invalidate()>)
- [SE-0481: weak let](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)
- [SE-0371: Isolated synchronous deinitialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0371-isolated-synchronous-deinit.md)
- [SE-0390: Noncopyable structs and enums](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [SE-0517: UniqueBox](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0517-uniquebox.md)
