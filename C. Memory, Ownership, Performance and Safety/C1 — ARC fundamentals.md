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

**Goal:** Remove a concrete strong cycle and verify release through an independent weak observation.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

ARC preserves class instances while strong ownership is needed. A cycle of strong references can keep objects alive after application roots stop reaching them. Closures and framework registrations can supply those edges. Releasing one reference, executing deinit, completing isolated cleanup, and returning physical memory to the OS are different events. [Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)

## Rules and pitfalls

### Trace the whole ownership graph

An owner storing a callback that captures it strongly forms a cycle. Removing that callback or making the back-reference appropriately weak breaks the edge. Deinit cannot break a cycle that prevents deinitialization from beginning.

**Common pitfall:** Weakening one visible reference can leave a timer, registration, bound method, or outer closure retaining the object. Find every strong root and define explicit unregister/invalidate behavior.

### Prove unowned access, not just parent ownership

Checked unowned access requires its referent to remain alive at every access. A child may escape its parent, so a strong parent-to-child edge does not establish the reverse lifetime requirement.

**Common pitfall:** Using unowned to avoid optional handling without proving lifetime can trap. Use weak when absence is permitted, or retain the dependency strongly when its lifetime is required.

### Separate lifetime from work cancellation

Weak capture does not cancel ongoing work or invalidate a run-loop timer. An isolated deinitializer may require executor scheduling; a nil weak observation is not a completion acknowledgment for all teardown.

## Interview questions

### What proves an escaped child cannot outlive its unowned referent?

A documented and enforced lifetime relation must prevent every access after the referent dies. Merely saying the parent owns the child is insufficient if another owner can retain the child. Check escaping callbacks, returned children, and external registrations; otherwise use weak or a strong lifetime dependency.

### How can weak let observe nil, and why is ImmutableWeakCaptures a separate setting?

Weak let fixes a weak reference's initial target while allowing it to observe nil after destruction. ImmutableWeakCaptures controls whether explicit weak closure captures are immutable; it is a separate feature setting from acceptance of weak let declarations. Neither changes the referent's isolation or Sendable requirements.

### Why is a weak reference alone insufficient evidence that actor-isolated cleanup completed?

Weak observation concerns object lifetime, while isolated cleanup can have executor-scheduled work and external resources with their own completion. Provide an explicit shutdown or acknowledgment when callers must know teardown finished; do not infer it from a weak pointer or deinit log.

## Focused exercise

**Task:** Trace one strong ownership cycle, remove the callback, and verify release. State the behavior expected after the owner disappears.

**Acceptance checks:**

- The installed callback works while the owner exists.
- Dropping the external owner leaves the cycle alive.
- Removing the callback releases the object; the weak trigger then does nothing.

**Solution:**

Standalone synchronous ownership probe. The weak trigger is safe after release:

```swift
final class Owner {
    var count = 0
    var callback: (() -> Void)?
    func install() { callback = { self.count += 1 } }
}
func probe() {
    var owner: Owner? = Owner()
    weak var observed = owner
    owner!.install()
    let trigger = { [weak owner] in owner?.callback?() }
    trigger()
    precondition(owner!.count == 1)
    owner = nil
    precondition(observed != nil)
    observed?.callback = nil
    precondition(observed == nil)
    trigger()
}
probe()
```

**Reasoning:** The callback and owner retain each other until the stored edge is cleared. The independent weak observation reveals that specific cycle. Retaining the original strongly capturing callback externally would create another owner and invalidate this release expectation.

## In practice

**Choose it when:** Specify resource ownership and explicit teardown before depending on deinit.

**Investigate by:** Use ownership graphs, Memory Graph, and scoped weak probes; distinguish retained objects from allocator memory.

**Strong evidence:** Verify external registrations and cleanup completion without relying on exact lexical destruction timing.

### Timer ownership and actor cleanup — optional

**Problem:** A screen owns a repeating timer; the run loop owns the registered timer; its callback captures the screen. Draw the graph and identify both the reference cycle and the work that should stop when the screen disappears.

**Worked owner:** The following timer is private and registered only on the main run loop. The callback uses a checked MainActor assumption based on that scheduling contract, because the callback's imported function type does not itself supply static MainActor isolation.

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

start first invalidates any old registration. The timer callback weakly references the owner, stop invalidates/removes ownership, and isolated deinit supplies a final cleanup backstop on the owning actor. A run loop retains its registered timers; invalidation is part of ending that registration. [Timer](https://developer.apple.com/documentation/foundation/timer), [invalidate](https://developer.apple.com/documentation/foundation/timer/invalidate())

The MainActor assumption is specific to this private main-run-loop timer. It neither schedules work nor makes an arbitrary callback safe. If a callback can arrive elsewhere, hop to the actor and account for queued work/cancellation rather than copying this assertion. The temporary test calls fire only from MainActor. It does not expose the private Timer to production callers who might fire it on another executor.

For UIKit, combine the owner with this adapter:

```swift
#if canImport(UIKit)
import UIKit

@MainActor
final class FeedViewController: UIViewController {
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
#endif
```

The adapter compiles against the iOS SDK. The owner is also usable from a macOS UI lifecycle with corresponding explicit start/stop calls. Actual screen presentation/navigation is outside the model fixture. Choose the lifecycle boundary that matches product requirements: not every disappearing view should stop the same work.

**Validation:** The temporary harness manually fires the actual registered Timer on MainActor, checks replacement invalidates the previous timer, checks repeated stop is harmless and invalid timers stop refreshing, and drops the owner while retaining a timer handle to observe weak-owner release and deinit invalidation. A long timer interval keeps the test independent of elapsed-time scheduling. It does not claim periodic accuracy, background execution, or physical UI behavior.

**Task alternative:** A task-based loop still needs ownership and cancellation design. Do not swallow sleep cancellation and then perform one more refresh without checking it. Promote weak ownership only for the required work, specify replacement/freshness policy, and cancel/join as appropriate. Reuse the complete checked patterns in A8/F1 instead of treating a bare Task loop as automatic lifecycle management.

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
- [invalidate](https://developer.apple.com/documentation/foundation/timer/invalidate()
- [SE-0481: weak let](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0481-weak-let.md)
- [SE-0371: Isolated synchronous deinitialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0371-isolated-synchronous-deinit.md)
- [SE-0390: Noncopyable structs and enums](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [SE-0517: UniqueBox](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0517-uniquebox.md)
