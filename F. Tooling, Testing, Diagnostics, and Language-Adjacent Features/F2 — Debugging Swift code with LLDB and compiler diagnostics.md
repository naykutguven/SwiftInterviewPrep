---
tags:
  - swift
  - ios
  - interview-prep
  - tooling
  - diagnostics
  - lldb
  - concurrency
  - debugging
---

# F2 — Debugging Swift Code with LLDB and Compiler Diagnostics

**Goal:** Reduce an actor-return diagnostic and verify a value-snapshot repair from its caller.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Compiler diagnostics explain a violated static contract; debugger inspection, expression evaluation, memory tools, and event history answer different runtime questions. Start with the smallest reproducer preserving relevant types, effects, imports, and settings. Accepted code can still contain stale-result logic or unsafe external behavior. [Swift compiler diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/)

## Rules and pitfalls

### Reduce the actual failed relationship

Replace unrelated generic/builder layers with explicit intermediate types and keep the original actor boundary. ViewBuilder can support different branch view types through its transform; a branch containing a non-view expression is a different defect.

**Common pitfall:** Adding sending to a retained mutable actor property does not make its returned reference disconnected. Return an independent snapshot or perform the requested operation inside the actor.

### Inspect before evaluating

Stored-field inspection can work without compiling an expression, while computed properties and po can execute application code. Optimized variables, module artifacts, actor context, and debugger evaluator support affect results.

**Common pitfall:** A failed expression does not prove corrupt state, and a successful po is not necessarily passive. Record the selected frame and prefer direct inspection before deliberate evaluation.

## Interview questions

### How do you isolate a generic or builder error, and why are different ViewBuilder branch types not automatically invalid?

Remove unrelated layers while retaining the type/effect constraint that fails, then give intermediate values explicit types. Builder branches can become different generic structural alternatives, so different view types alone are not invalid. Determine whether expression conversion, a hook, result type, or isolation requirement is missing.

### Why can stored-field inspection succeed while a computed-property expression fails, and can po change the target?

Stored-field inspection can read debug-described memory directly. A computed-property expression needs evaluator compilation and can invoke code, mutate state, allocate, or trigger callbacks. po can therefore change the target. Treat it as execution, especially around actor isolation or unsafe access.

### What changed in Swift 6.4 module tracking, and what must custom or archived-build debugging preserve?

Swift 6.4 completes precise explicit-module dependency tracking instead of ambiguous by-name lookup, reducing embedded binary-module data. Custom and archived debug workflows must preserve the referenced exact module artifacts and suitable paths/toolchain context. A matching symbol name alone cannot replace a missing module or correct dSYM.

### What distinguishes a closed ownership cycle, a data race, and a stale result?

A closed strong cycle prevents ARC release; a data race permits unsynchronized concurrent access; a stale result violates temporal business policy despite possible data-race freedom. Use ownership graphs, concurrency checking/sanitizers, and controlled request history respectively. No one tool proves all three absent.

## Focused exercise

**Task:** Reduce a non-Sendable actor-return diagnostic to a small example. Repair it with an isolated operation or snapshot and check the caller.

**Acceptance checks:**

- The caller receives a Sendable Dictionary/Data snapshot.
- Later actor mutation preserves the earlier snapshot.
- The retained-class-return variant rejects; warning suppression cannot repair its native error.

**Solution:**

Standalone snapshot repair using Foundation Data:

```swift
import Foundation
final class ImageCache { var storage: [String: Data] = [:] }
actor ImageLoader {
    private let cache = ImageCache()
    func put(_ data: Data, for key: String) { cache.storage[key] = data }
    func snapshot() -> [String: Data] { cache.storage }
}
@main struct Probe {
    static func main() async {
        let loader = ImageLoader()
        await loader.put(Data([1]), for: "avatar")
        let old = await loader.snapshot()
        await loader.put(Data([2]), for: "avatar")
        let current = await loader.snapshot()
        precondition(old["avatar"] == Data([1]))
        precondition(current["avatar"] == Data([2]))
    }
}
```

To reproduce the original error separately, add func cacheForDebugging() -> ImageCache { cache } inside ImageLoader and return it to a caller that mutates storage. That retained non-Sendable reference cannot cross the boundary safely.

**Reasoning:** The actor keeps the mutable reference and exposes independent value behavior. This validates isolation and snapshot semantics, not allocation count, image rendering, or a live debugger session. The full diagnostic variants below separate retained returns from fresh disconnected transfer.

## In practice

**Choose it when:** A compiler or runtime observation needs a smaller reproducible boundary.

**Investigate by:** Combine reduced compiler probes, intentional evaluator use, and request/task identities.

**Strong evidence:** State the limits of compiler, debugger, sanitizer, and memory evidence while validating the chosen repair.

### Module tracking and async diagnostic metadata — optional

#### Swift 6.4 module tracking

For explicitly built modules, Swift 6.3 began recording precise dependencies and the current module's path; 6.4 completes the transition. SwiftPM/Xcode handle this through the driver. A custom system invoking the frontend directly must supply `-debug-module-path` and keep paths resolvable, including remapping for distributed builds.

Such build systems can remove Linux/Windows module-wrapping steps and Darwin `-add_ast_path` usage. Binary Swift modules are no longer packed into dSYMs. Complex expressions still need compatible modules at their original or remapped locations. A dSYM alone therefore does not promise arbitrary expression evaluation on an archived build. Reflection-based inspection has different requirements. [Swift.org: Module Tracking in Swift Debug Info](https://www.swift.org/blog/module-tracking-in-debug-info/)

Treat an expression-import failure separately from a source error. Compare the loaded binary, debug information, toolchain, SDK, and module paths; run `swift-healthcheck` after the failure to collect evaluator diagnostics. Do not assume deleting caches repairs the underlying build contract. The module-tracking description above is documented behavior; this chapter's local debugger check is a freshly built macOS executable, not an archive-relocation test.

#### Async debugging: connect the task to the request

Task naming is 6.2 baseline material retained in 6.4. Names help correlate work in tools; they do not establish ownership, cancellation, or result freshness. Use a request identifier that distinguishes repeated requests without placing full query text or credentials in the name. [SE-0469](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0469-task-names.md)

```swift
func startSearch(
    requestID: UInt64,
    query: String,
    search: @escaping @Sendable (String) async throws -> [String]
) -> Task<[String], any Error> {
    Task(name: "search-\(requestID)") {
        try Task.checkCancellation()
        let values = try await search(query)
        try Task.checkCancellation()
        return values
    }
}
```

In the checked SDK, this creation helper compiles for macOS 15 / iOS 18. Reading the current task through `Task.name` requires Apple OS 26; reading a task handle through `handle.name` requires 27. The latter accessor is present in this Swift 6.4 distribution even though the SE-0469 page still labels its amendment unimplemented. Availability-negative checks and the macOS 27 runtime distinguish these APIs; older-runtime name display was not tested.

The caller owns this unstructured task: retain its handle, await `value` and handle errors, and request cancellation when appropriate. A successful service return after cancellation is rejected by the second check. A domain error thrown by the service still propagates; this helper does not convert every error into cancellation. Dropping a handle does not cancel its task. Naming also does not move work to another executor.

For UI publication, the owner must additionally compare the captured request identity with the current generation after suspension. Cancelling an old request is not permission to publish a new one, and equal query text does not mean equal request identity. F1 supplies the full controlled search example and regression tests; A8 covers owner lifetime. This helper intentionally covers task creation and observation, not a second UI controller implementation.

Trace request start, service completion, cancellation request, and the decision to publish or discard using the same identity. Inspect task/async context alongside thread stacks; a suspended task is not necessarily blocked on a currently executing thread. Use the debugger's actual task views and supported stepping behavior for the selected Xcode/runtime. Do not infer actor isolation solely from thread identity. See [[D7 — Actor Reentrancy and Logic Races]] and [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]].

## References

### Related guides

- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B6 — Any, AnyObject, Metatypes, and Runtime Type Information]]
- [[B14 — Result Builders and DSL Construction]]
- [[C1 — ARC fundamentals]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[D11 — Detached Tasks, Task Inheritance, and Isolation Loss]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [Swift compiler diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/)
- [Swift.org: Module Tracking in Swift Debug Info](https://www.swift.org/blog/module-tracking-in-debug-info/)
- [SE-0469](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0469-task-names.md)
- [SE-0522: Source-level warning control](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0522-source-warning-control.md)
- [LLDB tutorial](https://lldb.llvm.org/use/tutorial.html)
- [SE-0498: Runtime demangling](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0498-runtime-demangle.md)
- [Swift 6 migration guide: Common compiler errors](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/commonproblems/)
- [Apple: Diagnosing issues using crash reports and device logs](https://developer.apple.com/documentation/xcode/diagnosing-issues-using-crash-reports-and-device-logs)
- [Apple: Detect and diagnose memory issues](https://developer.apple.com/videos/play/wwdc2021/10180/)
