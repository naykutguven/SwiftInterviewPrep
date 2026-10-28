---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - closures
---

# A8 — Closures, Escaping, and Capture Semantics

**Learning goals:**

- Explain captured storage, value snapshots, reference captures, and closure lifetime annotations.
- Find strong ownership edges through stored callbacks, bound methods, and nested closures.
- Verify weak promotion across suspension and distinguish autoclosure evaluation, isolation, and sendability.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] for reference sharing, [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for function values, and [[C1 — ARC fundamentals]] for strong/weak/unowned ARC references. The async answer uses [[D1 — Structured Concurrency Fundamentals]]/[[D2 — Task Hierarchy, Cancellation, and Priorities]].

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A button handler contains both instructions and the values it needs later. That combination is a **closure**: executable behavior plus a captured environment. Copying the closure can keep access to the same mutable captured variable. A capture list chooses how particular values enter that environment.

Keep three questions separate: may the callee retain the closure, what does it retain, and where may it execute? @escaping addresses lifetime permission, [weak self] addresses a reference’s ownership, and actor/Sendable annotations address isolation and transfer. None alone defines delivery timing or invocation count. [Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### A value capture snapshots a value, not an object graph

The ordinary closure observes the later variable; the capture-list closure stores its value at creation.

```swift
func captureProbe() {
    var count = 1
    let live = { count }
    let snapshot = { [count] in count }
    let copied = live
    count = 2
    precondition(live() == 2 && copied() == 2)
    precondition(snapshot() == 1)
    final class Box { var count = 1 }
    let box = Box()
    let objectCapture = { [box] in box.count }
    box.count = 9
    precondition(objectCapture() == 9)
}
captureProbe()
```

**Walkthrough:** live and its copy observe one captured variable. snapshot retains the earlier Int. [box] retains the same reference, so it observes later object mutation. This behavior does not promise a particular heap-box allocation.

### Retaining permission and delivery timing are independent

The escaping callback runs once immediately and is then returned for later use. An optional function parameter is also a stored-capable value.

```swift
func immediate(_ body: () -> Void) { body() }
func retain(_ body: @escaping () -> Void) -> () -> Void {
    body()
    return body
}
func retainOptional(_ body: (() -> Void)?) -> (() -> Void)? { body }
var calls = 0
immediate { calls += 1 }
let later = retain { calls += 1 }
precondition(calls == 2)
later()
let optional = retainOptional(later)
optional?()
precondition(calls == 4)
```

**Walkthrough:** The callback marked escaping is called synchronously before retain returns and may then run again. Optional closure parameters do not need @escaping on the parameter to be retained.

## Rules and pitfalls

### Escaping does not promise later execution

**Common pitfall:** Setting readiness only after registering a callback because its parameter is @escaping.

**Mistake:** Standalone runnable example.

```swift
func register(_ callback: @escaping () -> Void) { callback() }
func probe() -> Bool {
    var ready = false
    var observed = false
    register { observed = ready }
    ready = true
    return observed
}
precondition(!probe())
```

**Consequence:** The callback observes false because register invokes it immediately.

**Why:** The annotation allows retention beyond the call; it does not require deferral.

**Correction:** Establish the state before registration, and document delivery timing separately.

```swift
func register(_ callback: @escaping () -> Void) { callback() }
func probe() -> Bool {
    let ready = true
    var observed = false
    register { observed = ready }
    return observed
}
precondition(probe())
```

**What changes:** Both immediate and later delivery now see established state. Real subscriptions also need invocation-count, actor-delivery, removal, and lifetime contracts.

### A weak inner closure cannot undo an outer strong edge

**Common pitfall:** A stored outer callback explicitly captures self to build an inner weak closure. This synchronous model exposes the ownership graph without introducing task scheduling.

**Mistake:** Standalone runnable example.

```swift
final class Owner {
    var count = 0
    var callback: (() -> (() -> Void))?
    func install() {
        callback = { [self] in { [weak self] in self?.count += 1 } }
    }
}
func probe() {
    var owner: Owner? = Owner()
    weak let observed = owner
    owner?.install()
    owner = nil
    precondition(observed != nil)  // Outer stored callback retains its owner.
    observed?.callback = nil  // Break the intentional test cycle.
    precondition(observed == nil)
}
probe()
```

**Consequence:** The outer stored callback keeps Owner alive. Swift 6.4 also diagnoses this mismatch when the outer strong capture is implicit; writing [self] acknowledges that edge without removing it. Weak capture in the closure it later creates does not remove that edge.

**Why:** Producing the inner weak capture requires the outer closure to have self. The cycle is Owner → outer closure → Owner.

**Correction:** Make the actual outer back-reference weak. A Task nested inside a stored callback needs the same graph analysis.

```swift
final class Owner {
    var count = 0
    var callback: (() -> (() -> Void))?
    func install() {
        callback = { [weak self] in { [weak self] in self?.count += 1 } }
    }
}
func probe() {
    var owner: Owner? = Owner()
    weak let observed = owner
    owner?.install()
    let outer = owner!.callback!
    let inner = outer()
    inner()
    precondition(owner?.count == 1)
    owner = nil
    precondition(observed == nil)
    inner()
}
probe()
```

**What changes:** Both closure layers observe optional lifetime. Test code retains the callbacks after release and safely invokes one without retaining the owner.

## Worked example

### A bound instance method is also a strong edge

**Setup:** Standalone synchronous probe. The intentional cycle is explicitly removed before returning.

**Predict:** Does storing handle retain its instance even without a written self capture?

```swift
final class Screen {
    var callback: (() -> Void)?
    var taps = 0
    func handle() { taps += 1 }
    func install() { callback = handle }
}
func probe() {
    var screen: Screen? = Screen()
    weak let observed = screen
    screen?.install()
    screen?.callback?()
    precondition(screen?.taps == 1)
    screen = nil
    precondition(observed != nil)
    observed?.callback = nil
    precondition(observed == nil)
}
probe()
```

**Outcome:** The screen remains alive until callback storage is cleared.

**Why:** A bound method pairs the function with its instance. Storing that method on its own instance closes the same cycle as a closure with a strong self capture.

## Interview questions

### How do direct nonescaping parameters, optional function values, and escaping parameters differ?

**Answer:** A direct function parameter is nonescaping by default and cannot be retained beyond the call. An escaping parameter or optional function value can be retained. None of these spellings promises later or concurrent execution.

**Example:** Standalone runnable example.

```swift
func callNow(_ body: () -> Int) -> Int { body() }
func save(_ body: @escaping () -> Int) -> () -> Int { body }
func saveOptional(_ body: (() -> Int)?) -> (() -> Int)? { body }
precondition(callNow { 1 } == 1)
let saved = save { 2 }
let optional = saveOptional { 3 }
precondition(saved() == 2 && optional?() == 3)
```

**Walkthrough:** callNow uses its callback within the call. The other functions return callbacks, which outlive the parameter’s call frame.

**Boundary:** Sendable and actor delivery remain separate contracts. A nonescaping closure can still have side effects; an escaping closure need never be retained by a particular implementation.

This independent **compile-only, intentionally rejected** example returns a nonescaping parameter:

```swift
func invalid(_ body: () -> Int) -> () -> Int { body }
```

### What do closure copies, [value], [object], weak, and unowned retain or observe?

**Answer:** Copies can share captured mutable storage. A value capture copies that value; an object capture retains the same instance. weak observes optional lifetime. unowned requires the object to outlive every access and traps if that assumption fails.

**Example:** Standalone runnable example.

```swift
final class Owner { var count = 1 }
func probe() {
    var number = 1
    let live = { number }
    let copied = live
    let value = { [number] in number }
    number = 2
    precondition(copied() == 2 && value() == 1)
    var owner: Owner? = Owner()
    let weakRead = { [weak object = owner] in object?.count }
    do {
        let strong = owner!
        let objectRead = { [strong] in strong.count }
        let unownedRead = { [unowned strong] in strong.count }
        strong.count = 3
        precondition(objectRead() == 3 && unownedRead() == 3)
    }  // Both closures are finished while the object is still alive.
    owner = nil
    precondition(weakRead() == nil)
}
probe()
```

**Walkthrough:** The live closure copy reads the shared variable, while [number] reads 1. Both object captures observe the same object’s edited count. After all strong references are released, the weak reader returns nil.

**Boundary:** Do not run an unowned callback after releasing its referent. Weak does not synchronize object access or make a mutable class Sendable.

### When is implicit self valid after capture, and why is a weak inner closure insufficient?

**Answer:** An explicit [self] capture can make retention intent clear enough for implicit member access. Unwrapping [weak self] with guard let self also permits it. The outer closure must itself avoid a strong ownership cycle.

**Example:** Standalone runnable example.

```swift
final class Owner {
    var count = 0
    func strongReader() -> () -> Int { { [self] in count } }
    func weakHandler() -> () -> Void {
        { [weak self] in
            guard let self else { return }
            count += 1
        }
    }
}
var owner: Owner? = Owner()
let handler = owner!.weakHandler()
handler()
precondition(owner?.count == 1)
let reader = owner!.strongReader()
owner = nil
precondition(reader() == 1)
handler()  // Owner remains alive because reader deliberately retains it.
precondition(reader() == 2)
```

**Walkthrough:** The weak handler promotes self only during invocation. reader intentionally retains the owner, so clearing owner alone does not destroy it. The nested-closure pitfall shows the independent outer edge. [SE-0269](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md), [SE-0365](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0365-implicit-self-weak-capture.md)

**Boundary:** Clear spelling does not make retention desirable. An owner storing strongReader() itself would create a cycle; an externally owned reader can have a legitimate strong lifetime contract.

### Shared async-lifetime setup

**Compile-only shared setup:** Gate is a one-shot test signal. wait suspends until open is called, and an already open gate returns immediately. It intentionally ignores task cancellation, letting the next answer isolate retention from cancellation. It is not a production cancellation-aware primitive.

```swift
actor Gate {
    private var opened = false
    private var waiters: [CheckedContinuation<Void, Never>] = []
    func wait() async {
        if opened { return }
        await withCheckedContinuation { waiters.append($0) }
    }
    func open() {
        opened = true
        let pending = waiters
        waiters.removeAll()
        for waiter in pending { waiter.resume() }
    }
}
```

### How do early and late weak promotion affect retention across await, cancellation, and deinit?

**Answer:** Early promotion keeps an owner strong across suspension if it is needed afterward. Late promotion permits release while a separately captured dependency runs. Cancellation is only a request and does not itself end that work.

**Example:** Prepend Shared async-lifetime setup. MainActor models screen state; the actor gates establish order without timing assumptions.

```swift
@MainActor final class Screen { var updates = 0 }
@MainActor func checkRetention(early: Bool) async {
    let started = Gate()
    let release = Gate()
    var screen: Screen? = Screen()
    weak let observed = screen
    let task: Task<Void, Never>
    if early {
        task = Task { [weak screen] in
            guard let screen else { return }
            await started.open()
            await release.wait()
            screen.updates += 1
        }
    } else {
        task = Task { [weak screen] in
            await started.open()
            await release.wait()
            guard let screen else { return }
            screen.updates += 1
        }
    }
    await started.wait()
    screen = nil
    task.cancel()
    if early { precondition(observed != nil) } else { precondition(observed == nil) }
    await release.open()
    await task.value
    precondition(observed == nil)
}
await checkRetention(early: true)
await checkRetention(early: false)
```

**Walkthrough:** The started gate proves the early branch has promoted screen before release. During suspension it retains the object even after cancellation. The late branch can release it before the dependency finishes. Opening release and joining the task ends both tests without sleeps.

**Boundary:** Strong async retention can be temporary rather than a permanent leak. Late promotion alone does not stop dependency work or reject stale publication; owners still need cancellation, joining, and freshness policies ([[D2 — Task Hierarchy, Cancellation, and Priorities|D2]]/[[D7 — Actor Reentrancy and Logic Races|D7]]).

### Extension: how do weak let and ImmutableWeakCaptures differ?

**Answer:** weak let prevents program reassignment while allowing ARC to zero the reference. The separate ImmutableWeakCaptures opt-in makes explicit weak closure captures immutable; it is not implied merely by Swift 6.4.

**Example:** Standalone runnable example.

```swift
final class SharedOwner: Sendable { let id = 7 }
final class WeakLink: Sendable {
    weak let owner: SharedOwner?
    init(_ owner: SharedOwner) { self.owner = owner }
}
func resettable(_ owner: SharedOwner) -> () -> Int? {
    { [weak owner] in
        let id = owner?.id
        owner = nil
        return id
    }
}
var owner: SharedOwner? = SharedOwner()
let link = WeakLink(owner!)
let reader = resettable(owner!)
precondition(reader() == 7 && reader() == nil)
owner = nil
precondition(link.owner == nil)
```

**Walkthrough:** The baseline permits resetting the weak capture, so the reader returns 7 then nil. The weak-let link also becomes nil after the last strong owner is released. [SE-0481](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)

**Boundary:** Adding `-enable-upcoming-feature ImmutableWeakCaptures` rejects the assignment inside the closure. Checked Sendable on WeakLink also depends on SharedOwner’s conformance; weak capture of arbitrary mutable state is insufficient.

With the additional ImmutableWeakCaptures opt-in, this standalone variant is **intentionally rejected**:

```swift
final class SharedOwner: Sendable { let id = 7 }
func invalidReset(_ owner: SharedOwner) -> () -> Int? {
    { [weak owner] in
        owner = nil
        return owner?.id
    }
}
```

The preceding block is also checked separately with `-enable-upcoming-feature ImmutableWeakCaptures`; that configuration is **intentionally rejected** for the assignment to owner. A read-only weak reader works under both settings:

```swift
final class SharedOwner: Sendable { let id = 7 }
func readOnly(_ owner: SharedOwner) -> @Sendable () -> Int? {
    { [weak owner] in owner?.id }
}
let owner = SharedOwner()
precondition(readOnly(owner)() == 7)
```

### Extension: does autoclosure memoize or merely defer an expression?

**Answer:** It wraps an expression in a closure. Evaluation occurs each time the body invokes that closure; there is no automatic caching. An escaping autoclosure can retain the expression for later invocation.

**Example:** Standalone runnable example.

```swift
func message(enabled: Bool, _ value: @autoclosure () -> String) -> String? {
    enabled ? value() : nil
}
func twice(_ value: @autoclosure () -> String) -> [String] { [value(), value()] }
func deferred(_ value: @autoclosure @escaping () -> String) -> () -> String { value }
func checkEvaluation() {
    var calls = 0
    func next() -> String {
        calls += 1
        return String(calls)
    }
    precondition(message(enabled: false, next()) == nil && calls == 0)
    precondition(message(enabled: true, next()) == "1" && calls == 1)
    precondition(twice(next()) == ["2", "3"])
    let later = deferred(next())
    precondition(calls == 3)
    precondition(later() == "4" && later() == "5")
}
checkEvaluation()
```

**Walkthrough:** Disabled evaluates zero times, enabled once, twice twice, and the returned closure once per invocation. Captured variables can be read at invocation time rather than closure creation.

**Boundary:** Use this hidden deferral sparingly, with names and behavior that make it understandable. Capture an explicit value snapshot when later reads should not observe changing state.

## Focused exercise

**Task:** Break a stored-callback cycle and verify both live-owner behavior and invocation after release.

**Acceptance checks:**

- The callback updates a live owner.
- An externally retained callback does not retain its owner.
- Calling it after release safely does nothing.

**Solution:** Standalone synchronous test. The callback deliberately has no concurrent delivery contract.

```swift
final class Owner {
    var count = 0
    var callback: (() -> Void)?
    func install() { callback = { [weak self] in self?.count += 1 } }
}
func probe() {
    var owner: Owner? = Owner()
    weak let observed = owner
    owner!.install()
    let callback = owner!.callback!
    callback()
    precondition(owner?.count == 1)
    owner = nil
    precondition(observed == nil)
    callback()
    precondition(observed == nil)
}
probe()
```

**Reasoning:** Owner strongly stores the closure, but the back-reference is weak. Retaining the closure externally no longer closes a cycle. The test establishes lifetime behavior; it does not establish thread safety or cancellation of asynchronous dependencies.

## In practice

**Choose it when:** Define callback ownership, delivery isolation, invocation count, and removal separately.

**Investigate by:** Follow every strong edge, including bound methods and outer closures; release the owner at a controlled suspension point.

**Strong evidence:** Verify teardown, cancellation, joining, and late publication independently. Weak capture is one ownership choice within that larger lifecycle design.

## References

### Related guides

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[B14 — Result Builders and DSL Construction]]
- [[C1 — ARC fundamentals]]
- [[D2 — Task Hierarchy, Cancellation, and Priorities]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]

### Official sources

- [Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)
- [SE-0481](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)
- [Swift: Automatic reference counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- [SE-0269: Implicit self after explicit capture](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0269-implicit-self-explicit-capture.md)
- [SE-0365: Implicit self after weak capture unwrapping](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0365-implicit-self-weak-capture.md)
- [Swift compiler: ImplicitStrongCapture](https://docs.swift.org/compiler/documentation/diagnostics/implicit-strong-capture/)
- [Swift compiler: Sendable captures](https://docs.swift.org/compiler/documentation/diagnostics/sendable-closure-captures/)
