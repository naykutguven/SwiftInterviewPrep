# Swift 6.4 What's New and Migration Guide

Baseline: Swift 6.4 / Swift 6 language mode. Collection review: 2026-09-27. This guide prioritizes senior/staff iOS and macOS work, with brief platform context. All 70 chapter reviews are complete. The [roadmap](<./Swift 6.4 Update Roadmap.md>) records scope and feature homes; the [validation record](<./Swift 6.4 Validation Record.md>) records compiler evidence, source-only coverage, and limitations.

## Start with the settings

A compiler release, language mode, package tools version, default actor isolation, upcoming features, and deployment target are separate inputs. Swift 6.4 still uses `-swift-version 6`; `6.4` is not a valid value for that flag. Conversely, `#if swift(>=6.4)` is true in mode 6 on the checked compiler because it compares the effective language version. A tools-version 6.4 manifest does not by itself describe every target's isolation policy.

Before migrating, record the current settings and keep them fixed while comparing diagnostics. Then adopt policy changes deliberately. The notes use explicit nonisolated defaults as the ordinary compiler baseline and name MainActor/NonisolatedNonsendingByDefault comparisons where tested. Synchronous nonisolated functions do not move work to a background executor. Start with [[D14 — Approachable Concurrency and Default Isolation]] and [[F5 — Language mode, build settings, and migration flags]].

## Changes worth learning first

| Change in the 6.4 baseline | Practical consequence | Teaching home |
| --- | --- | --- |
| Async `defer` | Async cleanup is awaited before scope exit. Cleanup can suspend and expose actor reentrancy; it is not automatically shielded from cancellation. | A6, D1, D7 |
| Cancellation shields | Define bounded cleanup and cancellation-observation policy. A shield does not guarantee external I/O succeeds or terminates. | D2, A6, D3 |
| Noncopyable `Continuation` | Compare consuming handles, typed errors, checked/unsafe alternatives, dropped-handle traps, and synchronized callback/cancellation state. Ownership does not guarantee a callback arrives. | D9, C8 |
| Explicit `~Sendable` | Suppress inferred sendability intentionally. A safe `sending` handoff is different from conformance suitable for general sharing. | D6, D13, B10 |
| Throwing-task result diagnostics | Observe errors and own cancellation/join policy. Dropping a task handle neither cancels nor makes its work structured. | D1, D11, A13 |
| Async `Result` initializer | Capture an async outcome without inventing a new task. Keep cancellation visible where the operation's contract requires it. | A13 |
| Optional `some`/`any` spelling | Shorter syntax does not change opaque identity, existential behavior, nested nil, or ownership rules. Group protocol compositions explicitly. | A4, B2, A7 |
| Initialized private properties and memberwise initialization | Revisit synthesis and compatibility overloads. Synthesized construction still has an access ceiling; public APIs need intentional initializers. | A3, A11, B13 |
| Array/dictionary-expression trailing closures | Understand parsing and user-defined initializers; this does not provide a built-in collection result builder. | A7, A14, B14 |
| `anyAppleOS` and source warning control | Separate supported syntax, runtime availability, platform overrides, warning identifiers, and scoped `@diagnose` policy. | A12, F2, F5 |

## Ownership and performance depth

These features matter most when a workload or API contract needs them. They do not replace the ordinary senior foundations of value semantics, ARC, exclusivity, and measurement.

| Area | What to add to the mental model | Teaching home |
| --- | --- | --- |
| Borrow/mutate accessors; `Ref`/`MutableRef` | Access duration, lifetime dependencies, and exclusive mutation matter even without copying. Distinguish these shipped mechanisms from separately proposed yielding accessors. | A2, C3, C7, C12 |
| `Iterable` and borrowing iteration | Borrow noncopyable elements and account for throwing iteration. Retain the distinct `Sequence`/`Collection` complexity and index contracts. | C5, B1, B3, C12 |
| `UniqueBox` and `UniqueArray` | Unique ownership changes copying, allocation, and transfer choices; it does not prove faster code or deep immutability. | C1–C2, C8, C12, F3 |
| Protocol generalization | Check conformance, synthesis, associated-type defaults, and container constraints separately. Noncopyable Hashable support does not make every standard container accept noncopyable elements. | B1, B3, B7–B8, C8 |
| Safe temporary and raw-span operations | Bounds-safe APIs still need valid formats, initialization, lifetimes, and byte-order policy. Prefer owned results at escaping boundaries. | C6, C11–C12, E3 |
| Smaller library changes | Collection/executor hashability has availability constraints. A non-spurious try-lock guarantee is not a fairness guarantee or a promise that a lock is available. | B8, C5, D8, D12 |

Library availability is independent of compiler syntax. In the checked Apple SDK, `Iterable`, `Ref`/`MutableRef`, `UniqueBox`, `UniqueArray`, noncopyable continuations, cancellation shields, and advanced observation require OS 27. `InlineArray`, `Array.span`, and `UTF8Span` have OS 26 gates; core span functionality has different back-deployment coverage. Follow each chapter's exact API/target annotations rather than assigning one minimum OS to the whole curriculum.

## The 6.3 catch-up and 6.2 migration context

| Earlier feature | What this update clarifies | Teaching home |
| --- | --- | --- |
| 6.3 immutable weak references | `weak let` does not mean the referent remains alive. Immutable weak captures require a separate checked feature setting. | A2, A8, C1, D6 |
| 6.3 module selectors | `Module::Name` disambiguates a module; it does not bypass imports or access control. | B6, E5–E6 |
| 6.3 `@export`, `@inline(always)`, `@specialized` | Distinguish source visibility, callable symbols, copied bodies, mandatory inlining, and library-side specialization. Old clients can retain old bodies after a library update. | B11, C9–C10, E8 |
| 6.3 `@c` and global `@implementation` | Check native declarations, generated headers, ownership, and actual C consumers. | E3 |
| 6.2.3 enum extensibility | Separate source exhaustiveness warnings from resilient binary layout and unknown runtime cases. | A5, A9, B11, E7 |
| 6.2 concurrency and memory foundations | Preserve the introduction history of approachable concurrency, isolated synchronous deinit, Span/InlineArray, and strict-memory-safety work. Recheck target settings and lifetime rules under 6.4. | D14, C1, C11–C12 |

When upgrading existing 6.2/6.3 code, first rebuild with matching policies, then review new diagnostics and exposed APIs. Audit callback adapters, cleanup, captures, and actor invariants. Check generated initializers and clients of public protocols. Revalidate a binary SDK with an unchanged client as well as rebuilt source. Raise package tools versions and deployment targets only when intentionally changing the support contract. Older compiler/runtime execution was not performed for this collection; migration comparisons are source and current-toolchain evidence.

## Tooling, testing, and platform boundaries

- **SwiftPM:** E5 checks the default Swift Build backend, real targets/products/resources, external consumers, and platform-conditional SBOM contents. Discover output locations through the tool; do not assume a historic `.build` path.
- **Testing:** F1 checks targeted Swift Testing/XCTest interoperability in both directions, tool-version defaults, reflection, attachments, and per-case repetition. A later passing retry summary does not erase an earlier nonzero process status. Keep deterministic task completion/cancellation checks.
- **Debugging and macros:** F2 separates frame inspection from expression compilation and explains module tracking. F4 validates a real macro package and external clients. A toolchain upgrade does not prove archived module relocation, every LLDB expression, or every macro dependency combination works.
- **Observation and Foundation:** F6 checks token lifetime, event timing, isolation, and the shipped continuous-observation API. E2 checks bridge identity/value distinctions and OS-gated progress APIs. Keep UI freshness and cancellation policy explicit.
- **Distribution:** E6 distinguishes import access from module loading, linkage, and runtime delivery. E7 checks unchanged clients and real device/Simulator/macOS XCFramework slices. E8 checks policy hiding and client-retained implementations. Packaging success alone is not a compatibility test.
- **Brief platform context:** E2 covers Subprocess 1.0 as a separate package, not an ordinary iOS Foundation facility. E5 summarizes Android/Java, Wasm, and Embedded Swift. These are source-reviewed architectural context; no non-Apple runtime execution is claimed.

## Checked limits and release-status corrections

The release overview is a useful starting point, but proposal text, SDK declarations, and observed compiler behavior can differ. Keep these limits visible:

| Boundary | Curriculum treatment |
| --- | --- |
| Optional reference operations and `RigidArray` | The installed 6.4 SDK lacks proposed `Optional.ref`/`mutableRef`/`put` and public `RigidArray`. Do not teach them as available baseline APIs. |
| Generalized protocols | Noncopyable Equatable/Comparable/Hashable support does not imply that `CustomStringConvertible` or `TextOutputStream` dropped Copyable in this SDK. |
| C++ spans | Stable input/owned-copy exercises pass. Default-empty conversion needs the published guard; experimental lifetime-bound returns have a preserved runtime failure and remain compile-only research examples. |
| Observation | The SDK spelling is `withContinuousObservation`. Combined mutation/deinit registration has a documented local callback discrepancy. |
| Runtime module | Demangling is checked on macOS 27; the installed iOS SDK lacks the Runtime module. An OS availability check cannot make a missing module importable. |
| SBOMs | Generated contents were inspected; missing local schema resources prevented schema validation. |
| Binary tooling | The ABI digester can report a frozen-layout violation with status zero. The checked Swift Build CLI consumer needed an explicit relative framework runpath to launch. |
| Future proposals | `withDeadline`, `Disconnected`, `FilePath`, noncopyable `reduce`, file-level defaults, and other accepted work are not adopted merely because a proposal exists. The roadmap records status boundaries. |

The [validation record](<./Swift 6.4 Validation Record.md>) retains compiler crashes, experimental failures, platform restrictions, and the distinction between historical execution records and raw evidence still available at final audit. No production guidance depends on treating a known failing experiment as a working API.

## Interview priorities

Start with correctness and settings literacy: cleanup that can suspend, task ownership, actor reentrancy, safe transfer, source access, and availability. Add one concrete migration prompt to a normal senior interview loop. For staff/library roles, extend it with ownership APIs, ordinary external clients, compatible binary replacement, generated interfaces, and measurement tradeoffs.

Use the [rubric and study checklist](<./Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) for scoring and exercises. Syntax recall alone is weaker evidence than explaining a failure, fixing the contract, and showing how the fix was validated.
