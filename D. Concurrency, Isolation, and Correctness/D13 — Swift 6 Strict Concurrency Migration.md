---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - swift6
  - strict-concurrency
---

# D13 — Swift 6 Strict Concurrency Migration

**Learning goals:**

- Reproduce a staged Swift 5 warning and the corresponding Swift 6 error under the same compiler/SDK.
- Repair sharing or isolation contracts and compile their clients rather than hiding diagnostics.
- Plan migration per target, track compatibility exceptions, and distinguish compiler updates from language-mode migration.

**Prerequisites:** [[D5 — Global Actors and `@MainActor`]]–[[D6 — `Sendable`, `@Sendable`, and `sending`]] for isolation and Sendable contracts; [[F5 — Language mode, build settings, and migration flags]] for compiler, language mode, SDK, deployment target, and warning policy.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Concurrency migration makes assumptions about state ownership visible to the compiler. Installing Swift 6.4 and choosing Swift 6 language mode are separate decisions. Swift 5 complete strict-concurrency checking can stage many problems as warnings; Swift 6 mode makes full checking part of the language contract.

Record compiler version, language mode, isolation defaults, upcoming features, SDK annotations, deployment targets, and warning policy separately. A target already using Swift 6 mode under Swift 6.2/6.3 is updating its compiler and SDK, not beginning the Swift 5 warning-staging step again. [Swift migration guide](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Compare modes without changing the source

**Setup:** Standalone **compile-only, expected warning** source. Use the D1 command with `-swift-version 5 -strict-concurrency=complete` replacing the language-mode option; retain the other baseline flags and do not promote the intended warning to an error.

```swift
final class Service { var count = 0 }
func operation(_ service: Service) -> @Sendable () -> Int {
    { service.count += 1; return service.count }
}
```

**Walkthrough:** The explicit Sendable closure captures a mutable ordinary class. Swift 5 complete checking reports the conflict while retaining language compatibility. With the same source and SDK, this separate Swift 6 case is **intentionally rejected**:

```swift
final class Service { var count = 0 }
func operation(_ service: Service) -> @Sendable () -> Int {
    { service.count += 1; return service.count }
}
```

The error identifies a missing sharing contract, not a request to add an arbitrary annotation. Choose a value snapshot when independent data is sufficient, or actor/synchronized ownership when mutable state must actually be shared.

## Rules and pitfalls

### Do not replace a missing contract with an unchecked promise

**Common pitfall:** Marking the class unchecked is treated as implementation of thread safety.

**Mistake:** Standalone **compile-only; do not concurrently invoke**. This is accepted but has no synchronization.

```swift
final class Service: @unchecked Sendable { var count = 0 }
func operation(_ service: Service) -> @Sendable () -> Int {
    { service.count += 1; return service.count }
}
```

**Consequence:** The diagnostic disappears while concurrent callbacks can still race on count.

**Why:** The author has asserted a guarantee the implementation does not provide. Blanket MainActor would be a different policy change: it could serialize unrelated service work onto the UI actor.

**Correction:** **Shared setup — actor operation.** Standalone **compile-only** declarations; prepend them where named.

```swift
actor CounterService {
    private var count = 0
    func increment() -> Int { count += 1; return count }
}
func operation(for service: CounterService) -> @Sendable () async -> Int {
    { await service.increment() }
}
```

**What changes:** The service now owns mutable state behind an explicit async boundary. Consumers must await it. A checked Mutex owner would be another contract for synchronous consumers; a snapshot would not preserve the behavior of a genuinely shared counter.

### Distinguish a new compiler warning from a new language-mode error

**Common pitfall:** Every diagnostic in a Swift 6.4 upgrade is treated as if turning Swift 5 complete checking back on is the repair.

**Mistake:** Standalone **compile-only, expected Swift 6.4 warning**. The target can already be in Swift 6 mode.

```swift
enum Failure: Error { case offline }
func start() { Task<Int, Error> { throw Failure.offline } }
```

**Consequence:** The unobserved throwing-task result needs an owner; changing mode or suppressing the warning does not observe it.

**Why:** Compiler-release diagnostics and language-mode transitions are separate axes. This warning is a prompt to fix ownership, not an automatically thrown service error.

**Correction:** Standalone runnable owner.

```swift
enum Failure: Error { case offline }
let handle = Task<Int, Error> { throw Failure.offline }
do { _ = try await handle.value; preconditionFailure("expected failure") }
catch Failure.offline { }
```

**What changes:** The owner joins and observes failure. Check new warnings, feature flags, and SDK availability for an already-Swift-6 target rather than reclassifying its entire migration.

## Worked example

### Repair the boundary and exercise its consumer

**Setup:** Prepend **actor operation**. Runnable.

**Predict:** Does the repair still represent a shared incrementing service?

```swift
let request = operation(for: CounterService())
let first = await request()
let second = await request()
precondition(first == 1 && second == 2)
```

**Outcome:** The consumer sees one then two.

**Why:** The actor repair preserved shared state instead of replacing it with per-call snapshots. Source compatibility changed deliberately: the callback is now async, so the consumer is updated and checked too.

## Interview questions

### What did complete strict concurrency checking provide before Swift 6 language mode?

**Answer:** It exposed many isolation and sendability problems as staged warnings while keeping Swift 5 compatibility, allowing incremental source repair before enabling Swift 6 errors.

**Example:** Independent **compile-only, expected warning** source using Swift 5 mode plus complete checking.

```swift
final class Cache { var names: [String] = [] }
func read(_ cache: Cache) -> @Sendable () -> Int { { cache.names.count } }
```

**Walkthrough:** Even a read-only closure does not make the reachable mutable cache safe; another alias could write it. The staged warning lets the team identify and repair that contract before changing the target's language mode.

**Boundary:** This uses the Swift 6.4 compiler in Swift 5 language mode, not an older compiler/runtime. Complete checking does not validate every external synchronization, unsafe pointer, or legacy callback assumption.

### When is @preconcurrency acceptable, and why is it rarely the final design?

**Answer:** It can be a narrow compatibility bridge when a legacy module has a documented safe contract that its declarations do not express. It changes specified checking behavior; it adds no lock or actor hop. Record an owner, evidence, and a removal condition.

**Example — legacy module:** Compile this standalone library as `LegacyKit` in Swift 5 mode, then compile the clients separately. The final class has only an immutable Int, but its public interface omits Sendable. In this fixture those immutable-storage facts are the external audit.

```swift
public final class LegacyRecord {
    public let id: Int
    public init(id: Int) { self.id = id }
}
```

The essential library command uses the D1 flags with `-swift-version 5 -parse-as-library -emit-module -module-name LegacyKit -emit-module-path LegacyKit.swiftmodule -c`. Add `-I` pointing to that module directory for each client.

This separate Swift 6 client is **intentionally rejected** without a conformance or compatibility annotation:

```swift
import LegacyKit
func callback(_ record: LegacyRecord) -> @Sendable () -> Int { { record.id } }
```

This **compile-only compatibility client** scopes the import annotation to that dependency:

```swift
@preconcurrency import LegacyKit
func callback(_ record: LegacyRecord) -> @Sendable () -> Int { { record.id } }
```

**Walkthrough:** The annotation changes imported-type diagnostic treatment. It does not change the object or insert synchronization. This particular module is immutable by construction, which is the reason a temporary audit can be justified. A mutable legacy class would need a different contract.

An independent **compile-only** snapshot client avoids sharing the reference at all:

```swift
import LegacyKit
func callback(_ record: LegacyRecord) -> @Sendable () -> Int {
    let id = record.id
    return { id }
}
```

**Boundary:** Prefer an updated checked conformance or a safe adapter. Compatibility annotations on imports, protocol conformances, and declarations have distinct rules; do not treat the spelling as a universal suppression. Remove an exception when the dependency's public contract becomes expressible. [SE-0337](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)

## Focused exercise

**Task:** Reproduce a Swift 5 complete-checking capture warning, observe Swift 6 rejection, then repair the ownership model and compile its client.

**Acceptance checks:**

- The same original source warns in Swift 5 and rejects in Swift 6.
- The repaired consumer preserves shared increment behavior.
- No compatibility annotation is needed for the owned implementation.

**Solution:** Use the two essential-mechanics cases as negative controls, then prepend **actor operation** to this runnable consumer.

```swift
let service = CounterService()
let request = operation(for: service)
let first = await request()
let second = await request()
precondition(first == 1 && second == 2)
```

**Reasoning:** The compiler diagnostic led to an explicit state owner. The updated async consumer proves the repaired API is usable. A green library build without its clients would be incomplete evidence.

## In practice

**Choose it when:** Migrate target by target: app, extension, package, tests, generated code, and public clients can have different effective settings.

**Investigate by:** Record compiler/SDK, language mode, isolation defaults, both opt-ins, deployment target, and warning policy. Classify each failure as UI ownership, shared state, capture, transfer, or imported contract.

**Strong evidence:** Keep an exception ledger with the dependency, external guarantee, responsible owner, verification, and removal condition. Recheck callers and justified exceptions after supported SDK updates; enable focused build gates after contracts are coherent. D14 demonstrates cross-module execution policies.

## References

### Related guides

- [[D4 — Actors and Actor Isolation]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [Migration guide](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)
- [Swift.org. "Announcing Swift 6." Swift.org Blog](https://swift.org/blog/announcing-swift-6/)
- [Apple Developer. "Migrate Your App to Swift 6." WWDC24](https://developer.apple.com/videos/play/wwdc2024/10169/)
- [GitHub. "Support Incremental Migration to Concurrency Checking." Swift Evolution SE-0337](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)
