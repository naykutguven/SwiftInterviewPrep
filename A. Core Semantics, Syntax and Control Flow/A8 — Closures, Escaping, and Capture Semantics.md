---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - closures
---

# A8 — Closures, Escaping, and Capture Semantics

**Goal:** Break a stored-callback cycle and verify invocation both before and after the owner is released.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A closure combines executable behavior and a captured environment. Copies can share mutable captured storage. A capture list can snapshot a value, retain an object, or observe it weakly; these choices are separate from whether the callee may retain the closure and where it may execute. Shared capture behavior does not guarantee a particular heap-box allocation. [Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)

## Rules and pitfalls

### Define callback ownership and delivery

A directly declared function parameter is nonescaping unless marked escaping. Optional function values are ordinary stored-capable values and can escape without that parameter annotation. Neither form promises asynchronous or concurrent invocation.

**Common pitfall:** Assuming escaping means “later” misses callbacks invoked immediately and retained afterward. State delivery isolation, invocation count, storage owner, and removal behavior independently.

### Trace every strong edge

An owner storing a closure that captures the owner creates a cycle. Bound instance methods and outer closures can supply the strong edge even when an inner closure uses weak self. Unowned requires a proven lifetime; weak allows absence.

**Common pitfall:** Adding weak self only to an inner task does not remove the enclosing stored callback's strong capture. Inspect the whole ownership graph and test teardown.

### Account for suspension

Promoting weak self to a strong local before await can retain the owner throughout suspension. Capture a dependency separately and promote the owner after await when post-release work should be discarded. Cancellation requests and deinit are not interchangeable lifecycle events.

## Interview questions

### How do direct nonescaping parameters, optional function values, and escaping parameters differ, and does any spelling promise later/concurrent execution?

Nonescaping directly typed parameters cannot be retained beyond the call without an explicit escape. Escaping parameters and optional function values can be retained. None implies when or how many times the callback runs, or whether it is Sendable. Actor and transfer requirements remain separate parts of the signature.

### What do a closure copy, [value], [object], [weak self], and [unowned self] retain or observe?

A copied closure can share its environment. [value] captures that value at creation; [object] retains the same object rather than deep-copying it. Weak self observes optional lifetime without retaining it, while unowned self assumes the object remains alive and can trap if that assumption fails. Referenced mutable state still needs isolation.

### When is implicit self valid after a capture list, and why does a weak inner closure not remove a strong outer cycle?

Explicit strong self capture, and supported weak-self promotion forms, can allow implicit member access because the retention decision is visible. A weak inner closure cannot undo a strong self captured by an outer stored closure. Remove the actual cycle edge, not merely a nearby occurrence of self.

### How do early and late promotion of weak self affect retention across await, cancellation, and deinit?

Early promotion keeps self strong across await; late promotion can let it deinitialize while dependency work runs. Cancellation does not necessarily end that work or prevent stale publication. Define an owner, cancellation, joining, and freshness policy, then test release while the operation is suspended.

## Focused exercise

**Task:** Trace an owner, stored callback, and captured reference. Break the cycle and check both live-owner behavior and post-release invocation.

**Acceptance checks:**

- Invocation updates a live owner.
- The externally retained callback does not keep the owner alive.
- Invocation after release safely does nothing.

**Solution:**

Standalone synchronous lifetime probe; the callback is deliberately non-Sendable:

```swift
final class Owner {
    var count = 0
    var callback: (() -> Void)?
    func install() {
        callback = { [weak self] in self?.count += 1 }
    }
}
func probe() {
    var owner: Owner? = Owner()
    weak var observed = owner
    owner!.install()
    let callback = owner!.callback!
    callback()
    precondition(owner!.count == 1)
    owner = nil
    precondition(observed == nil)
    callback()
}
probe()
```

**Reasoning:** The owner's callback storage is a strong edge, but the capture back to the owner is weak. Retaining the callback externally therefore does not close a cycle. This is a synchronous retention test; it does not establish thread safety or prompt cancellation of async work.

## In practice

**Choose it when:** Use explicit callback lifecycle contracts for subscriptions and separately owned tasks for asynchronous work.

**Investigate by:** Follow strong edges, then release the owner at a controlled suspension point.

**Strong evidence:** Validate delivery, cancellation, cleanup, and late publication separately instead of treating weak capture as a complete design.

### Capture and autoclosure details — optional

#### Weak immutable references and weak-capture mutability are separate

```swift
final class SharedOwner: Sendable {
    let id: Int
    init(id: Int) { self.id = id }
}

final class WeakLink: Sendable {
    weak let owner: SharedOwner?
    init(_ owner: SharedOwner) { self.owner = owner }
}

func weakReader(_ owner: SharedOwner) -> @Sendable () -> Int? {
    { [weak owner] in owner?.id }
}
```

weak let prevents reassignment by program code; it can still observe nil after the referent is released. Checked sendability here also depends on the referenced SharedOwner being Sendable. A weak capture of an arbitrary mutable non-Sendable class does not make a Sendable closure valid. [SE-0481](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)

This next example deliberately exposes the separate configuration boundary:

```swift
func resettableReader(_ owner: SharedOwner) -> () -> Int? {
    { [weak owner] in
        let value = owner?.id
        owner = nil
        return value
    }
}
```

In the checked baseline it returns the ID once and then nil. With `-enable-upcoming-feature ImmutableWeakCaptures`, assignment to the explicit weak capture is rejected. The read-only reader above works with either setting. Do not label every weak capture immutable merely because the compiler version is 6.4. A2/D6 record the same distinction for binding and transfer design.

#### Autoclosure defers an expression; it does not memoize it

```swift
func message(
    enabled: Bool, _ value: @autoclosure () -> String
) -> String? {
    enabled ? value() : nil
}

func twice(_ value: @autoclosure () -> String) -> [String] {
    [value(), value()]
}

func deferred(
    _ value: @autoclosure @escaping () -> String
) -> () -> String {
    value
}
```

For a side-effecting message expression, disabled evaluates it zero times, enabled once, and twice evaluates it twice. The returned deferred closure can evaluate it later and more than once. Captured mutable storage can therefore be read at invocation time; use an explicit snapshot when that is the intended contract. Keep this invisible deferral limited to APIs whose names and behavior make it understandable.

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
