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

**Learning goals:**

- Reduce diagnostics while preserving the failed generic, builder, ownership or actor relationship.
- Separate stored-state inspection, evaluator execution, precise module tracking and runtime history.
- Repair a retained actor-return boundary and connect task names to request identity without assuming scheduling or freshness.

**Prerequisites:** [[B3 — Generics, Constraints, and Same-Type Relationships]]/[[B14 — Result Builders and DSL Construction]] for generics/builders, [[C1 — ARC fundamentals]] for lifetime, [[D6 — `Sendable`, `@Sendable`, and `sending`]]/[[D7 — Actor Reentrancy and Logic Races]]/[[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]] for transfer/reentrancy/isolation, and [[F5 — Language mode, build settings, and migration flags]] for the exact compiler settings.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A debugger prints a stored field, but evaluating a computed property fails. Those are different operations. Direct inspection can use debug-described storage; an expression may need to import Swift modules, compile code, and execute it in the stopped process.

Likewise, a compiler error, a retained ownership graph, an unsynchronized access, and a stale response require different evidence. Start with a small reproducer preserving the relationship that failed, then choose the tool that answers the actual question. [Compiler diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Make a generic relationship explicit

Standalone generic input. The return type preserves the element’s actual ID rather than guessing its concrete type.

```swift
struct Row: Identifiable { let id: Int }
func firstID<S: Sequence>(_ rows: S) -> S.Element.ID? where S.Element: Identifiable {
    rows.first(where: { _ in true })?.id
}
let rows = [Row(id: 7)]
let identifier: Int? = firstID(rows)
precondition(identifier == 7)
```

**Walkthrough:** The sequence’s element type supplies its ID relationship. Explicit intermediates expose which assumption is wrong without replacing the generic algorithm with unrelated code.

### Keep mutable reference state inside its actor

Shared ImageLoader setup. The actor owns the ImageCache object; callers receive Dictionary/Data values.

```swift
import Foundation

final class ImageCache { var storage: [String: Data] = [:] }
actor ImageLoader {
    private let cache = ImageCache()
    func put(_ data: Data, for key: String) { cache.storage[key] = data }
    func snapshot() -> [String: Data] { cache.storage }
}
```

**Walkthrough:** The mutable class is an implementation detail. put executes on the actor, while snapshot returns value behavior that can cross the boundary. This does not measure buffer allocations.

### A debugger expression can have visible effects

Shared DebugProbe.swift source. Compile as a fresh local executable with -g -Onone -emit-module -module-name DebugProbe and its matching module path. A breakpoint in stopHere makes the distinction observable.

```swift
final class DebugCounter: CustomStringConvertible {
    let stored = 7
    var evaluations = 0
    var computed: Int {
        evaluations += 1
        return stored
    }
    var description: String {
        evaluations += 1
        return "counter=\(stored)"
    }
}
@inline(never) func stopHere(_ counter: DebugCounter) {
    print(counter.stored)  // Stop before this line to inspect the parameter.
}
let counter = DebugCounter()
stopHere(counter)
```

**Walkthrough:** Reading stored/evaluations directly should not call these accessors. Evaluating computed or po counter can run the getter/description and increment evaluations. The commands below give the expected observations at that breakpoint. This revision builds the fixture and runs an ordinary Swift accessor probe, but live LLDB child launch stalled on this host; the counter changes are not a captured debugger transcript.

## Rules and pitfalls

### A different ViewBuilder branch type is not the defect

**Common pitfall:** Treating every heterogeneous builder branch as invalid instead of finding the expression conversion that actually fails.

**Mistake:** Compile-only; intentionally rejected.

```swift
import SwiftUI

@MainActor @ViewBuilder func invalidContent(_ selected: Bool) -> some View {
    if selected { Text("Selected") } else { 42 }
}
```

**Consequence:** The Int expression lacks the required View conversion; the error is not merely that the branches have different concrete types.

**Why:** ViewBuilder can form a structural alternative from supported view expressions. An arbitrary value still needs a supported buildExpression path.

**Correction:** Use appropriate view expressions and retain the different branch types as a valid control.

```swift
import SwiftUI

@MainActor @ViewBuilder func content(_ selected: Bool) -> some View {
    if selected { Text("Selected") } else { Image(systemName: "circle") }
}
```

**What changes:** Text and Image are different types but both satisfy the builder’s view contract. The example is object-compiled; it does not render a UI.

### sending does not disconnect a reference the actor retains

**Common pitfall:** Adding sending to an actor return while keeping the same mutable object in actor storage.

**Mistake:** Compile-only; intentionally rejected.

```swift
final class ImageCache { var count = 0 }
actor Owner {
    private let cache = ImageCache()
    func retained() -> sending ImageCache { cache }
}
func receive(_ owner: Owner) async {
    let value = await owner.retained()
    value.count += 1
}
```

**Consequence:** Ownership analysis rejects the retained reference transfer.

**Why:** The actor still has an alias to mutable state. A keyword cannot make the returned region independent of the retained storage.

**Correction:** Keep the requested mutation isolated and return a Sendable snapshot. Prepend ImageLoader setup.

```swift
let loader = ImageLoader()
await loader.put(Data([1]), for: "avatar")
let old = await loader.snapshot()
await loader.put(Data([2]), for: "avatar")
let current = await loader.snapshot()
precondition(old["avatar"] == Data([1]) && current["avatar"] == Data([2]))
```

**What changes:** The operation stays inside the actor and the old value remains independent after a later update. A fresh disconnected reference can be a different accepted transfer contract; it is not this retained-return contract.

### po is execution, not a passive memory read

**Common pitfall:** Calling a getter or description while investigating state and assuming the target is unchanged.

**Mistake:** With the DebugProbe breakpoint stopped in stopHere, these commands deliberately evaluate code:

```text
expression -- counter.computed
po counter
```

**Consequence:** DebugCounter.evaluations can increase even though the application has not continued past the breakpoint.

**Why:** LLDB’s evaluator compiles an expression and runs accessors or description code. It can allocate, mutate, invoke callbacks, or need compatible modules.

**Correction:** Start with direct stored-field inspection, then evaluate only the operation you intend:

```text
frame variable counter
frame variable counter.evaluations
expression -- counter.computed
frame variable counter.evaluations
```

**What changes:** The first inspection establishes a baseline before evaluator effects. The local fixture checks the actual transcript. A failed expression does not by itself prove corrupt storage; inspect selected frame, modules, optimization and actor context too.

## Worked example

### The actor-return diagnostic survives warning suppression

**Setup:** Independent intentionally rejected source. It preserves the caller that crosses the actor boundary rather than compiling an unused accessor only.

**Predict:** Can suppressing RegionIsolation turn this retained object into a valid cross-actor result?

```swift
import Foundation

final class ImageCache { var storage: [String: Data] = [:] }
actor ImageLoader {
    private let cache = ImageCache()
    func cacheForDebugging() -> ImageCache { cache }
}
@diagnose(RegionIsolation, as: ignored, reason: "Teaching negative: native error remains")
func use(loader: ImageLoader) async {
    let cache = await loader.cacheForDebugging()
    cache.storage["avatar"] = Data()
}
```

**Outcome:** The compiler still rejects the non-Sendable result across isolation.

**Why:** This is a native error, not a warning that the annotation can demote. The earlier snapshot correction changes the boundary itself. Reduction must retain the caller or it may remove the operation that exposes the failure.

## Interview questions

### How do you isolate a generic or builder error, and why are different ViewBuilder branch types not automatically invalid?

**Answer:** Keep the relevant constraint/effect, name intermediate types, and remove unrelated composition. Different branch types can be valid structural alternatives; identify the failed expression or result requirement.

**Example:** Standalone runnable example.

```swift
struct Row: Identifiable { let id: Int }
func label(_ row: Row) -> String { String(row.id) }
let row = Row(id: 7)
let identifier: Int = row.id
let text: String = label(row)
precondition(identifier == 7 && text == "7")
```

**Walkthrough:** The concrete intermediate reveals that ID is Int and text conversion is explicit. The earlier accepted/rejected ViewBuilder pair isolates view conversion rather than blaming heterogeneous branches.

**Boundary:** A reduced example must preserve actor/throws/ownership boundaries if those cause the error. A passing simplified algorithm is not a repair if it changed the intended API.

An independent **intentionally rejected** wrong intermediate type exposes that assumption directly:

```swift
struct Row { let id: Int }
let identifier: String = Row(id: 7).id
```

### Why can stored-field inspection succeed while a computed-property expression fails, and can po change the target?

**Answer:** Stored inspection can use reflection/debug metadata without running a getter. Computed properties and po may need evaluator compilation/module imports and can execute target code. Their failure or success answers a different question from direct storage.

**Example:** Use these commands on the named DebugProbe fixture; expected counter changes follow below:

```sh
lldb /tmp/debug-probe
```

```text
breakpoint set --name stopHere
run
frame variable counter.stored counter.evaluations
expression -- counter.computed
frame variable counter.evaluations
po counter
frame variable counter.evaluations
```

**Walkthrough:** The fresh fixture starts with evaluations zero. The getter and description each increment it when actually evaluated. The stored counter distinguishes inspection from execution. These are expected effects of the supplied accessors; this host did not establish a live LLDB launch.

**Boundary:** Optimized-out variables, missing modules, incompatible SDKs, actor checks and evaluator limitations can all change what commands succeed. A working local -Onone session is not an archived-build relocation or arbitrary-expression guarantee.

### What changed in Swift 6.4 module tracking, and what must custom or archived-build debugging preserve?

**Answer:** Explicitly built modules and debug info track precise module dependencies/paths rather than relying only on names. SwiftPM/Xcode drivers manage that contract; custom frontend builds must preserve matching module artifacts and remapped paths.

**Example:** This is the actual local debug-fixture build shape, using the baseline flags as well:

```sh
swiftc -g -Onone -module-name DebugProbe -emit-module \
  -emit-module-path /tmp/DebugProbe.swiftmodule DebugProbe.swift -o /tmp/debug-probe
lldb /tmp/debug-probe
```

```text
# Run after an evaluator import failure, in the relevant debugger session:
swift-healthcheck
```

**Walkthrough:** Keep the executable/debug information, selected toolchain/SDK and exact referenced modules together. For a custom frontend invoking explicit module builds, -debug-module-path communicates its own module path; remapping must keep those paths resolvable. Darwin dSYMs no longer carry embedded binary Swift modules under the new model. [Precise module tracking](https://www.swift.org/blog/module-tracking-in-debug-info/)

**Boundary:** The command above builds a fresh local debugger fixture. Live launch was not established on this host, and no distributed/archive path-remapping matrix is claimed. A dSYM or matching symbol name alone does not promise every expression can compile. Remove old module-wrapping/add_ast_path assumptions only under the documented explicit-build model.

### What distinguishes a closed ownership cycle, a data race, and a stale result?

**Answer:** A cycle preserves a strong ownership graph; a data race violates concurrent access safety; a stale result violates which operation may publish. Each needs an appropriate ownership, synchronization, or request-history check.

**Example:** Standalone runnable example.

```swift
struct Reply {
    let requestID: Int
    let value: String
}
var currentRequestID = 2
var displayed = "new"
let old = Reply(requestID: 1, value: "old")
if old.requestID == currentRequestID { displayed = old.value }
precondition(displayed == "new")
```

**Walkthrough:** This sequential publication check rejects stale identity without needing a data race. [[C1 — ARC fundamentals|C1]] separately tests an ownership cycle; compiler/sanitizer checking addresses different access defects.

**Boundary:** This small identity example is not a full task owner. [[F1 — Testing Strategy in Modern Swift|F1]] and [[F6 — Observation, property wrappers, and language-adjacent state tools|F6]] use controlled asynchronous request arrival/completion, and memory/sanitizer tools retain their own limits.

The ownership case has a different observable failure. This independent runnable probe creates a closed strong cycle, observes it through a weak reference, then breaks the edge so the example cleans up:

```swift
final class Owner {
    var action: (() -> Void)?
}
func cycleProbe() {
    weak var surviving: Owner?
    do {
        let owner = Owner()
        surviving = owner
        owner.action = { [owner] in withExtendedLifetime(owner) {} }
    }
    precondition(surviving != nil)
    surviving?.action = nil
    precondition(surviving == nil)
}
cycleProbe()
```

The closure retains its owner, which retains the closure. Clearing action breaks that cycle without changing any request identity. A memory graph can reveal the ownership edges; it does not decide whether a result is current.

This separate source is **intentionally rejected**: it attempts to capture the same mutable class into a Sendable detached operation and also mutate it in the caller. No race is executed.

```swift
final class UnsafeCounter { var count = 0 }
func unsafelyShare(_ counter: UnsafeCounter) {
    Task.detached { @Sendable in counter.count += 1 }
    counter.count += 1
}
```

Here the capture violates the static sharing contract. Dynamic race/memory tools inspect executed paths and cannot prove unexecuted paths safe; neither a passing sanitizer run nor fixing this capture establishes result freshness.

### Extension: What does naming a task establish about its owner or result freshness?

**Answer:** Only diagnostic identity. The caller still owns the unstructured handle, cancellation is cooperative, and publication needs an operation identity distinct from repeated query text.

**Example:** Compile-only shared startSearch setup; callers use its owned handle.

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

**Walkthrough:** The helper checks cancellation before and after the service, propagates domain errors, and returns a named task that its caller must await/cancel. Separate controls distinguish creation, Task.name, and handle.name availability.

**Boundary:** The ordinary named initializer is back-deployable to macOS 10.15/iOS 13 in this SDK; current-task name needs OS 26 and handle name OS 27. Compilation for an older target does not establish runtime name support there. The executorPreference initializer has its own newer availability. Naming does not select an executor, cancel on handle drop, or establish freshness.

A runnable caller keeps the handle and observes both result and name under the OS 27 baseline:

```swift
let task = startSearch(requestID: 7, query: "query") { input in [input.uppercased()] }
let values = try await task.value
precondition(values == ["QUERY"] && task.name == "search-7")
```

A fresh independent result is a separate accepted sending contract, rather than a repair by annotation of retained state:

```swift
final class FreshState { var count = 0 }
actor Factory { func make() -> sending FreshState { FreshState() } }
let state = await Factory().make()
state.count = 1
precondition(state.count == 1)
```

## Focused exercise

**Task:** Reduce a retained non-Sendable actor result, repair it with an isolated operation/snapshot, and verify the actual caller.

**Acceptance checks:**

- The reduced retained-return and annotated variants reject for the intended boundary.
- The caller receives a Dictionary/Data value snapshot.
- A later actor mutation leaves the old snapshot intact.

**Solution:** Prepend ImageLoader setup; this is the runnable repair.

```swift
let loader = ImageLoader()
await loader.put(Data([1]), for: "avatar")
let old = await loader.snapshot()
await loader.put(Data([2]), for: "avatar")
await loader.put(Data([3]), for: "thumbnail")
let current = await loader.snapshot()
precondition(old == ["avatar": Data([1])])
precondition(current == ["avatar": Data([2]), "thumbnail": Data([3])])
```

**Reasoning:** The actor retains and mutates the class; only value snapshots cross. Object compilation includes ownership diagnostics, while execution establishes the chosen value behavior. It is independent of allocation counts or image rendering.

## In practice

**Choose it when:** Reduce the actual failed contract before choosing a compiler, debugger or runtime tool.

**Investigate by:** Record the frame/build/modules and distinguish passive inspection from deliberate evaluation; trace request identities across suspension.

**Strong evidence:** State what compiler, debugger, sanitizer and memory evidence establish, then verify the repaired boundary from an actual caller.

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
