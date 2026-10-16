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

**Goal:** Repair a staged concurrency warning and verify the chosen sharing contract in Swift 6 mode.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Migration turns implicit ownership and isolation assumptions into checked contracts. Swift 5 complete checking stages many data-race diagnostics as warnings; Swift 6 mode makes full checking part of the language contract. Compiler release, language mode, Approachable Concurrency, default isolation, SDK annotations, and warning policy must be recorded separately. [Migration guide](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/enabledataracesafety/)

## Rules and pitfalls

### Migrate by boundary and target

Inventory targets and public callers, then classify diagnostics by UI ownership, shared service state, capture, transfer, and imported legacy contracts. Use this collection's nonisolated defaults with both opt-ins; apply MainActor explicitly where UI state requires it.

**Common pitfall:** Blanket MainActor or unchecked Sendable can make a build green while changing execution or claiming nonexistent synchronization. Repair the ownership boundary and compile its actual clients.

### Track compatibility exceptions

Use @preconcurrency only for a justified legacy boundary with documented external guarantees and a removal condition. It changes checking in specified compatibility cases; it adds no lock, actor hop, or runtime proof.

**Common pitfall:** Treating Swift 6.2/6.3 code already in Swift 6 mode as if it still had Swift 5 warning staging confuses two migrations. Audit changed flags, SDK availability, and new diagnostics such as discarded throwing tasks separately.

## Interview questions

### What did complete strict concurrency checking provide before Swift 6 language mode?

It exposed many isolation and sendability problems as warnings while retaining Swift 5 language compatibility. That made incremental source repair possible before opting into Swift 6 errors. It was not equivalent to a complete proof that all dependencies and external synchronization contracts were safe.

### When is `@preconcurrency` acceptable, and why is it rarely the final design?

It can be acceptable when a legacy dependency's unannotated API has a documented safe contract that the current compiler cannot express or infer correctly. Scope it narrowly, record owner and removal condition, and validate the real integration. Prefer an updated dependency or safe adapter as the lasting design.

## Focused exercise

**Task:** Repair one Swift 5 complete-checking warning through ownership or isolation. Compile in Swift 6 mode and explain any remaining compatibility annotation.

**Acceptance checks:**

- The original capture warns in Swift 5 complete checking and rejects in Swift 6.
- The actor-based repair compiles and preserves sequential shared-counter behavior.
- No compatibility annotation remains necessary for this owned implementation.

**Solution:**

First reproduce F5's non-Sendable capture in Swift 5 complete checking; keep the same compiler/SDK and both opt-ins when comparing Swift 6 rejection. This standalone repaired executable uses an actor for a service that genuinely shares state:

```swift
actor CounterService {
    private var count = 0
    func increment() -> Int { count += 1; return count }
}
func operation(for service: CounterService) -> @Sendable () async -> Int {
    { await service.increment() }
}
@main struct Probe {
    static func main() async {
        let operation = operation(for: CounterService())
        let first = await operation()
        let second = await operation()
        precondition(first == 1 && second == 2)
    }
}
```

**Reasoning:** The service still represents shared mutable state, so a snapshot-only replacement would change behavior. Actor ownership makes that state safe and introduces an explicit async access boundary. Update consumers accordingly rather than marking the original class unchecked.

## In practice

**Choose it when:** Migrate modules incrementally with an explicit baseline and caller checks.

**Investigate by:** Reproduce each diagnostic under recorded settings and test its real ownership boundary.

**Strong evidence:** Maintain a target/exception ledger, validate public clients, and add focused failure gates after migration.

### Migration rollout — optional

Start by recording emitted settings for app, extension, package, test, and generated-code targets. Stage legacy targets with complete checking, fix one ownership category at a time, and verify UI/service boundaries with ordinary clients. Enable Swift 6 per target only after its contracts are coherent. Keep both Approachable Concurrency opt-ins explicit and leave default isolation nonisolated unless a documented comparison overrides it. Revisit dependency adapters and compatibility annotations on each supported SDK/toolchain upgrade. Use [[F5 — Language mode, build settings, and migration flags]] for reproducible commands and [[D14 — Approachable Concurrency and Default Isolation]] for execution-policy comparisons.

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
