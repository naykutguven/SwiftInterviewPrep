---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - arc
---
## 0. Rubric snapshot

**Rubric expectation**

Know retain/release semantics, strong cycles, `weak`, `unowned`, and deinitialization timing.

**Caveats**

ARC is deterministic at normal scope/lifetime boundaries, but lifetimes can be extended by closure captures, async work, autorelease pools, bridging, timers, run loops, and framework ownership.

**You should be able to answer**

- When is `unowned` correct, and when is it a crash waiting to happen?
- Why can an object survive longer than expected even without an obvious strong reference in your current scope?

**You should be able to do**

- Trace a retain cycle involving a view controller, a timer, and a closure.
- Predict which deinitializers run in the code probe.
- Identify the strong cycle and fix it without changing the outward behavior of the callback.

---

## 1. Core mental model

ARC, or Automatic Reference Counting, is Swift’s memory-management system for class instances. Swift inserts retains and releases around ownership operations so that an object stays alive while it has at least one strong reference and is deallocated when the last strong reference goes away. The Swift book describes ARC as Swift’s mechanism for tracking and managing memory for class instances, and the same ARC chapter covers strong cycles, `weak`, `unowned`, and closure capture lists. ([Swift.org, "Automatic Reference Counting"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/))

ARC is not garbage collection. There is no general tracing collector that later discovers unreachable cycles. ARC only counts references. If object A strongly owns object B, object B strongly owns a closure, and the closure strongly captures object A, all reference counts stay above zero forever. The graph may be unreachable from your code, but ARC still sees every object in the cycle as owned.

`weak` and `unowned` are both non-owning references. They break cycles because they do not increment the referenced object’s strong reference count. The important difference is failure behavior: `weak` must be optional and becomes `nil` when the object deallocates; `unowned` is used when the referenced object is expected to outlive the reference, and accessing it after deallocation is a runtime failure. Swift’s documentation describes capture lists as the way to change how closures capture references, including `weak` and `unowned`. ([Swift.org, "Automatic Reference Counting"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/))

Closures matter because closures are reference types and can capture objects strongly. If a class instance stores a closure property, and that closure refers to the instance, the instance can own the closure while the closure owns the instance. Swift’s closure documentation explicitly calls out this pattern and says capture lists are used to break those cycles. ([Swift.org, "Closures"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/))

The key idea:

```text
ARC destroys objects when strong-reference count reaches zero.
ARC does not destroy closed strong-reference cycles.
```

Swift guarantees:

```text
A class instance remains alive while strongly referenced.
deinit runs when the instance is actually deallocated.
weak references do not keep an object alive.
weak references become nil after deallocation.
```

Swift does not guarantee:

```text
Cycles are automatically collected.
A closure capture is harmless.
An object dies exactly where your human eye sees the last local variable.
unowned is safe just because it avoids optionals.
```

---

## 2. Essential mechanics

### Strong references are the default

Most references to class instances are strong unless marked `weak` or `unowned`.

```swift
final class Service {
    deinit {
        print("Service deinit")
    }
}

var service: Service? = Service()
let alias = service

service = nil
print("still alive")

// alias still strongly references the Service instance.
```

`service = nil` removes one strong reference. It does not deallocate the object while `alias` still owns it.

A strong reference answers:

```text
Keep this object alive for as long as I point to it.
```

### `weak` breaks ownership and models optional lifetime

Use `weak` when the referenced object may disappear independently.

```swift
protocol FlowCoordinatorDelegate: AnyObject {
    func didFinish()
}

final class FlowCoordinator {
    weak var delegate: FlowCoordinatorDelegate?
}
```

This is common for delegate relationships: the coordinator should not keep the delegate alive. The Swift book’s protocol examples also describe delegates as weak references to prevent strong reference cycles. ([Swift.org, "Protocols"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/))

A `weak` reference answers:

```text
I want to refer to this object if it is still alive, but I do not own it.
```

### `unowned` breaks ownership but assumes lifetime

Use `unowned` only when the referenced object must outlive the reference.

```swift
final class Customer {
    let card: CreditCard

    init(number: String) {
        self.card = CreditCard(number: number, customer: self)
    }
}

final class CreditCard {
    let number: String
    unowned let customer: Customer

    init(number: String, customer: Customer) {
        self.number = number
        self.customer = customer
    }
}
```

This shape is valid only if a `CreditCard` can never outlive its `Customer`. If the card can escape into a cache, async task, closure, notification center, or timer, `unowned` becomes dangerous.

A `unowned` reference answers:

```text
I do not own this object, but I require it to still exist whenever I use it.
```

### Closure captures are strong by default

```swift
final class Screen {
    var onTap: (() -> Void)?

    func bind() {
        onTap = {
            print(self)
        }
    }
}
```

This creates:

```text
Screen ─strong─> closure
closure ─strong─> Screen
```

A capture list changes the capture behavior:

```swift
final class Screen {
    var onTap: (() -> Void)?

    func bind() {
        onTap = { [weak self] in
            guard let self else { return }
            print(self)
        }
    }
}
```

### Lifetimes can be extended by frameworks and bridging

ARC can feel “late” when another owner exists outside your local code. Common examples:

```text
A stored escaping closure captures self.
A Task captures self until the task completes or is cancelled.
A Timer is retained by a RunLoop.
Foundation/Objective-C bridging creates autoreleased objects.
A framework stores your object as a delegate, target, callback, or observer.
```

Apple’s autorelease-pool documentation says autorelease pools hold objects until the pool drains; with ARC, Swift uses `@autoreleasepool` blocks rather than manual `NSAutoreleasePool` management. ([Apple Developer, "NSAutoreleasePool"](https://developer.apple.com/documentation/foundation/nsautoreleasepool)) Apple’s Timer and RunLoop docs also matter here: a run loop retains a timer, invalidation removes it, and a timer can repeatedly reschedule until invalidated. ([Apple Developer, "Timer"](https://developer.apple.com/documentation/foundation/timer))

---

## 3. Common traps and misconceptions

### Trap 1: “I used ARC, so memory leaks are impossible”

ARC prevents many manual retain/release bugs. It does not solve cycles.

Bad:

```swift
final class Parent {
    var child: Child?
}

final class Child {
    var parent: Parent?
}
```

Better:

```swift
final class Parent {
    var child: Child?
}

final class Child {
    weak var parent: Parent?
}
```

The parent owns the child. The child observes or refers back to the parent without owning it.

### Trap 2: Using `[weak self]` everywhere without thinking

`[weak self]` is often correct for UI callbacks, timers, long-lived closures, and work that can outlive the object. But using it everywhere can hide bugs where the operation actually requires the owner to stay alive.

Example:

```swift
service.save { [weak self] result in
    self?.state = .saved(result)
}
```

This may be right for a view controller. It may be wrong for a coordinator or operation object whose whole purpose is to complete the save.

Better design question:

```text
Should this callback own the object until completion, or should the object be allowed to disappear?
```

### Trap 3: Using `[unowned self]` because it avoids optional unwrapping

Bad:

```swift
timer = .scheduledTimer(withTimeInterval: 1, repeats: true) { [unowned self] _ in
    refresh()
}
```

This is a crash waiting to happen if the timer fires after `self` deallocates.

Better:

```swift
timer = .scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
    self?.refresh()
}
```

For UI objects, `weak` is usually the default choice because views and controllers have lifetimes controlled by navigation, presentation, reuse, cancellation, and user behavior.

### Trap 4: Thinking `deinit` is a lifecycle API

`deinit` is a cleanup backstop, not the primary lifecycle hook for user-visible behavior.

Bad:

```swift
deinit {
    timer?.invalidate()
}
```

This is not enough if the timer cycle prevents `deinit` from running.

Better:

```swift
override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)
    timer?.invalidate()
    timer = nil
}

deinit {
    timer?.invalidate()
}
```

Use explicit lifecycle cleanup where the resource should stop. Keep `deinit` as a safety net.

---

## 4. Direct answers to rubric questions

### Q1. When is `unowned` correct, and when is it a crash waiting to happen?

`unowned` is correct when the referenced object is guaranteed to outlive the reference. It is dangerous when the referenced object can deallocate before the `unowned` reference is accessed.

Good cases:

```text
Child-like helper that cannot escape its owner.
Back-reference where the owner strictly owns the referencer.
Tightly coupled object graph with a documented lifetime invariant.
```

Dangerous cases:

```text
Escaping closures.
Timers.
Detached or long-lived tasks.
Notification observers.
Caches.
Delegates whose owner is not structurally guaranteed.
Anything crossing UIKit/AppKit lifecycle boundaries.
```

Interview version:

> `unowned` is for non-owning references with a strict lifetime invariant: the referenced object must outlive the reference. I use it only when the object graph structurally proves that invariant. If a closure, timer, async task, cache, or framework callback can outlive the owner, `unowned` is unsafe because access after deallocation traps. In those cases I usually use `weak` or redesign ownership.

### Q2. Why can an object survive longer than expected even without an obvious strong reference in your current scope?

Because the strong reference may be elsewhere in the ownership graph. Local scope is not the whole ownership model.

Common hidden owners:

```text
Stored escaping closures
Tasks
Timers retained by run loops
Framework callback storage
Notification/Combine subscriptions
Objective-C autorelease pools
Bridged Foundation objects
Singletons and caches
```

Example:

```swift
final class ViewModel {
    var task: Task<Void, Never>?

    func load() {
        task = Task {
            await fetch()
            print(self)
        }
    }

    func fetch() async {}
}
```

The task closure captures `self`. Even if the current method returns, `self` can remain alive until the task completes or the task reference/capture graph is broken.

Interview version:

> I do not reason about ARC from local variables alone. I reason from the ownership graph. An object can survive because an escaping closure, task, run loop, timer, autorelease pool, bridge, cache, or framework object still owns it. The absence of an obvious local strong reference does not mean the strong count is zero.

---

## 5. Code probe

Given:

```swift
final class Owner {
    var child: Child?

    deinit {
        print("Owner deinit")
    }
}

final class Child {
    var onDone: (() -> Void)?

    deinit {
        print("Child deinit")
    }
}

func make() {
    let owner = Owner()
    let child = Child()

    owner.child = child
    child.onDone = { print(owner) }
}

make()
```

### What happens?

```text
<no output>
```

No deinitializer prints.

### Why?

After these lines:

```swift
owner.child = child
child.onDone = { print(owner) }
```

The ownership graph is:

```text
owner local ─strong─> Owner
child local ─strong─> Child

Owner.child ─strong─> Child
Child.onDone ─strong─> closure
closure capture ─strong─> Owner
```

When `make()` returns, the local variables are released:

```text
owner local released
child local released
```

But this graph remains:

```text
Owner ─strong─> Child
Child ─strong─> closure
closure ─strong─> Owner
```

That is a closed strong-reference cycle. ARC cannot deallocate any node because every node still has a strong owner inside the cycle.

The closure is never called, so `print(owner)` does not run. The only possible output would come from `deinit`, but neither object deinitializes.

### Fix or redesign

```swift
final class Owner {
    var child: Child?

    deinit {
        print("Owner deinit")
    }
}

final class Child {
    var onDone: (() -> Void)?

    deinit {
        print("Child deinit")
    }
}

func make() {
    let owner = Owner()
    let child = Child()

    owner.child = child

    child.onDone = { [weak owner] in
        guard let owner else { return }
        print(owner)
    }
}

make()
```

Output:

```text
Owner deinit
Child deinit
```

### Why this fix is correct

The closure no longer owns `owner`.

The graph becomes:

```text
Owner ─strong─> Child
Child ─strong─> closure
closure ─weak─> Owner
```

When `make()` returns:

```text
owner local released
Owner has no remaining strong owners
Owner deinitializes
Owner.child releases Child
child local released
Child has no remaining strong owners
Child deinitializes
Child.onDone releases closure
```

The callback behavior while `owner` is alive is unchanged: calling `onDone` still prints the owner. If `owner` is gone, the callback safely becomes a no-op instead of keeping the object graph alive forever.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`[weak owner]`|The callback may outlive `owner`, or lifetime is not strictly guaranteed.|Callback becomes a no-op after `owner` deallocates.|
|`[unowned owner]`|`owner` is guaranteed to outlive the callback.|Runtime trap if that invariant is ever broken.|
|Explicit teardown: `child.onDone = nil`|You have a clear lifecycle boundary, such as `stop()`, `close()`, or `viewWillDisappear`.|Requires discipline; missed teardown leaks.|
|Redesign ownership|The child should not know about the owner directly.|More design work; usually best for reusable components.|

Example with explicit teardown:

```swift
final class Owner {
    var child: Child?

    func stop() {
        child?.onDone = nil
        child = nil
    }

    deinit {
        stop()
        print("Owner deinit")
    }
}
```

This is useful for resource-like lifecycles, but it is not enough if the only cleanup is in `deinit` and the cycle prevents `deinit` from running.

---

## 6. Exercise

### Problem

Trace a retain cycle involving a view controller, a timer, and a closure.

### Bad / naive version

```swift
import UIKit

@MainActor
final class FeedViewController: UIViewController {
    private var timer: Timer?

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        timer = .scheduledTimer(withTimeInterval: 1, repeats: true) { _ in
            self.refresh()
        }
    }

    private func refresh() {
        print("refresh")
    }

    deinit {
        print("FeedViewController deinit")
    }
}
```

### What is wrong?

```text
FeedViewController strongly owns timer.
The run loop strongly owns timer.
Timer strongly owns its closure.
The closure strongly captures FeedViewController.
Therefore FeedViewController cannot deinitialize.
```

Ownership graph:

```text
RunLoop ─strong─> Timer
FeedViewController ─strong─> Timer
Timer ─strong─> closure
closure ─strong─> FeedViewController
```

Even if the view controller disappears, the timer can keep firing because it is scheduled on the run loop. Apple’s documentation notes that a run loop retains a timer and that invalidation removes the timer from the run loop. ([Apple Developer, "add(_:forMode:)"](https://developer.apple.com/documentation/foundation/runloop/add%28_%3aformode%3a%29-392ag))

### Improved version

```swift
import UIKit

@MainActor
final class FeedViewController: UIViewController {
    private var timer: Timer?

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        startRefreshing()
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        stopRefreshing()
    }

    private func startRefreshing() {
        stopRefreshing()

        timer = .scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
            self?.refresh()
        }
    }

    private func stopRefreshing() {
        timer?.invalidate()
        timer = nil
    }

    private func refresh() {
        print("refresh")
    }

    deinit {
        stopRefreshing()
        print("FeedViewController deinit")
    }
}
```

### Why this is better

It fixes both sides of the problem:

```text
[weak self] prevents Timer -> closure -> ViewController from owning the view controller.
invalidate() removes the timer from the run loop.
timer = nil removes the view controller's strong reference to the timer.
viewWillDisappear stops work at the lifecycle boundary, instead of waiting for deinit.
```

This is production-grade because the resource is stopped explicitly when it should stop, and `deinit` is only a backstop.

### Production alternative: avoid Timer ownership entirely where possible

For Swift Concurrency-based periodic UI work, a task can be clearer, but it has the same capture problem if written carelessly.

Bad:

```swift
private var refreshTask: Task<Void, Never>?

func startRefreshing() {
    refreshTask = Task {
        while !Task.isCancelled {
            try? await Task.sleep(for: .seconds(1))
            self.refresh()
        }
    }
}
```

Better:

```swift
private var refreshTask: Task<Void, Never>?

func startRefreshing() {
    stopRefreshing()

    refreshTask = Task { [weak self] in
        while !Task.isCancelled {
            try? await Task.sleep(for: .seconds(1))

            guard let self else { return }
            self.refresh()
        }
    }
}

func stopRefreshing() {
    refreshTask?.cancel()
    refreshTask = nil
}
```

This avoids run-loop timer behavior but not ownership reasoning. A task closure can still retain `self`.

---

## 7. Production guidance

Use strong references when:

```text
The current object owns the dependency.
The child/resource should live at least as long as the owner.
You want lifetime to be explicit and deterministic.
```

Use `weak` when:

```text
The relationship is back-reference-like.
The referenced object may disappear independently.
The callback can outlive the owner.
You are dealing with UI lifecycle, delegates, timers, async work, or reusable views/cells.
```

Use `unowned` when:

```text
The lifetime invariant is structurally guaranteed.
The referencer cannot escape the owner.
The object graph makes it impossible for the reference to outlive the owner.
You would rather crash than silently ignore a violated invariant.
```

Be careful when:

```text
A closure is stored.
A closure is passed to an escaping API.
A task captures self.
A timer, display link, notification token, Combine subscription, or observation token exists.
A delegate is not weak.
Foundation/Objective-C bridging or autorelease pools are involved.
```

Avoid:

```text
Using unowned in UI callbacks by default.
Using weak everywhere without understanding ownership.
Relying only on deinit for stopping timers or streams.
Creating bidirectional strong references between parent and child objects.
Storing closures that capture their owner strongly.
```

Debugging checklist:

```text
Is deinit expected but not called?
Which object owns this object?
Does a stored closure capture self?
Does a task capture self?
Is there a Timer, CADisplayLink, NotificationCenter observer, Combine subscription, or KVO/observation token?
Does a framework retain a target, delegate, data source, callback, or block?
Is cleanup only in deinit even though deinit cannot run because of the cycle?
Can Xcode's Memory Graph show a path from a root object to the leaked instance?
Would temporarily adding deinit logs to related nodes reveal which object stays alive?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> ARC automatically manages memory. Use `weak` to avoid retain cycles and `unowned` when you know the object exists.

### Senior answer

> ARC is deterministic reference counting for class instances, but it cannot collect cycles. I reason from the ownership graph: strong references keep objects alive, closures capture strongly by default, and `weak`/`unowned` are tools for expressing non-ownership. I use `weak` when lifetimes are independent and `unowned` only when the lifetime invariant is structurally guaranteed.

### Staff-level answer

> ARC bugs are design bugs in the ownership graph. I look for hidden owners: escaping closures, tasks, timers, run loops, observers, caches, autorelease pools, and framework callback storage. For production code, I do not just sprinkle `[weak self]`; I decide whether an operation should keep its owner alive, whether the API should expose explicit cancellation/teardown, and whether ownership should be inverted or modeled through value types, actors, or short-lived resource handles.

Staff-level questions to ask:

```text
Who owns whom, and is that ownership part of the domain model or accidental?
Can this callback outlive its owner?
Is deinit being used as primary lifecycle cleanup instead of a safety net?
Would cancellation, invalidation, or explicit teardown make the ownership boundary clearer?
Is weak capture hiding a lost operation that should actually be retained until completion?
Can this reference graph be simplified with value types, dependency inversion, or a narrower API?
```

---

## 9. Interview-ready summary

ARC is Swift’s deterministic reference-counting system for class instances. Strong references keep objects alive; when the last strong reference disappears, `deinit` runs and the instance is deallocated. ARC does not detect strong cycles, so bidirectional object graphs and stored closures can leak. Closures capture references strongly by default, which is why `self -> closure -> self` is a common leak. `weak` is for non-owning references that may become `nil`; `unowned` is for non-owning references with a strict lifetime guarantee and will trap if used after deallocation. In production, I debug ARC issues by drawing the ownership graph and looking for hidden owners like tasks, timers, run loops, observers, subscriptions, caches, autorelease pools, and framework callback storage.

---

## 10. Flashcards

Q: What does ARC count?

A: Strong references to class instances. It keeps an instance alive while the strong-reference count is greater than zero.

Q: Does ARC collect strong reference cycles?

A: No. If objects strongly reference each other in a closed cycle, ARC sees each object as still owned.

Q: What is the difference between `weak` and `unowned`?

A: `weak` does not retain and becomes `nil` after deallocation. `unowned` does not retain and assumes the referenced object still exists; accessing it after deallocation traps.

Q: Why do stored closures often cause retain cycles?

A: Closures are reference types and capture class instances strongly by default. If an object stores a closure that captures the object, they can strongly own each other.

Q: Why is `unowned self` dangerous in a timer callback?

A: The timer or run loop can outlive the object. If the timer fires after the object deallocates, accessing `unowned self` traps.

Q: Why might `deinit` not run even after setting a variable to `nil`?

A: Some other owner still has a strong reference, often through a closure, task, timer, observer, cache, or framework-owned object.

Q: What is the exact output of the C1 code probe?

A: No output. Neither `Owner deinit` nor `Child deinit` prints because `Owner -> Child -> closure -> Owner` forms a strong cycle.

Q: What is the safest default capture for UI callbacks that may outlive the view controller?

A: Usually `[weak self]`, combined with explicit cancellation or invalidation at the right lifecycle boundary.

---

## 11. Related sections

- [[A8 — Closures, escaping, and capture semantics]]
- [[A15 — Type choices: struct, class, enum, protocol, and actor]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[E2 — Foundation and Objective-C bridging behavior]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — C1 ARC fundamentals.
- Swift.org. "Automatic Reference Counting." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/
- Swift.org. "Closures." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/
- Swift.org. "Declarations." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/
- Apple Developer. "NSAutoreleasePool." Apple Developer Documentation. https://developer.apple.com/documentation/foundation/nsautoreleasepool
- Apple Developer. "Timer." Apple Developer Documentation. https://developer.apple.com/documentation/foundation/timer
