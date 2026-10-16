---
tags:
  - interview-prep
  - swift
  - ios
---

# Swift Senior/Staff Rubric and Prioritized Study Checklist

Swift 6.4 baseline. Rubric revised 2026-10-16; the collection validation record retains its 2026-09-27 review scope. This rubric covers 70 topic notes. The [roadmap](<./Swift 6.4 Update Roadmap.md>) records feature coverage; the [validation record](<./Swift 6.4 Validation Record.md>) separates compiler/runtime evidence, source review, and platform limits. New assessment prompts here may need corresponding additions to the topic notes.

Use this to assess Swift semantics and engineering judgment for senior and staff iOS/macOS work. Combine it with role-specific architecture, delivery, and leadership evidence in a complete hiring process.

Primary references:

- [The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/) and [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [Swift 6.4 release](https://www.swift.org/blog/swift-6.4-released/) and the implemented proposals linked by each topic
- [Global-actor-isolated conformances, SE-0470](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md)
- [Xcode build settings reference](https://developer.apple.com/documentation/xcode/build-settings-reference) for the Approachable Concurrency feature bundle
- [Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/) for suspension, blocking, and forward progress
- [Clock](https://developer.apple.com/documentation/swift/clock) for elapsed-time measurement and testable time dependencies

## How to Use This

For study, start with Tier 1's stated focus, then choose a role track. Mark a topic complete when you can explain its rule, solve its focused exercise, and check a relevant failure case without a substantive hint.

For interviews, choose the topics, depth, and settings before the session. Sample questions from the detailed rubric; its full question bank is for repeated practice. Each topic has a focused exercise, score anchors, and a link to longer chapter work. Extensions are optional unless the role requires them.

### Shared Example Baseline

Unless a probe states otherwise, assume:

| Setting | Baseline |
| --- | --- |
| Compiler / language mode | Swift 6.4 / Swift 6 |
| Approachable Concurrency | Enabled |
| Default Actor Isolation | `nonisolated` |
| Strict concurrency checking | Complete, as required by Swift 6 mode |
| Optimization | Default debug (`-Onone`); performance probes state their configuration |

In Swift 6 mode, the additional Approachable Concurrency opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. The bundle's other features are already enabled by the language mode. Equivalent compiler flags:

```sh
-swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances
```

Approachable Concurrency and Default Actor Isolation are separate settings. Eligible unannotated declarations remain nonisolated; use explicit `@MainActor` when a scenario requires UI or other main-actor ownership. Plain nonisolated async functions follow the caller's actor under this baseline; `@concurrent` selects off-actor execution. Synchronous nonisolated calls continue on the caller's execution context.

Record the SDK and deployment target when availability matters. Imported declarations retain their defining module's isolation contract. Linked notes and the validation record may use other settings; apply this baseline to rubric examples and label comparison configurations explicitly. The earlier validation record does not establish results under this revised baseline.

Label examples as standalone, dependent on linked declarations, compile-only, or intentionally failing. Run deliberate traps separately. For concurrent code, distinguish required behavior from one possible execution order; compiler diagnostic wording can vary.

### Scoring Scale

Score the selected scope of a topic, rather than every feature mentioned in its chapter:

| Score | Observable evidence |
| --- | --- |
| `0` | Cannot form a working explanation or start the task after the problem is clarified. |
| `1` | Recognizes terms but needs step-by-step help to predict behavior. |
| `2` | Handles the routine case but needs a substantive hint to find a relevant failure or repair it. |
| `3` | Independently predicts behavior, explains the rule, fixes or implements the selected case, and checks a relevant boundary. |
| `4` | Meets `3`, then compares viable designs under stated constraints and explains how to validate the contract across callers, versions, or isolation boundaries. |
| `N/A` | Not assessed or outside the role's scope; record which. |

Each topic's `Score anchors` give concrete examples for `3` and `4`. They guide comparable evidence; alternative correct designs can demonstrate the same depth. Record whether the candidate worked independently, needed problem clarification, or received a substantive hint. Clarifying assumptions and checking documentation are useful engineering behaviors.

## Interview Use: Assessment Guidance

For a senior candidate, seek repeated score-3 evidence in the selected foundations and role track: correct reasoning, a workable repair, and a meaningful check. Investigate an unresolved ownership or concurrency error with another example. Record how the candidate responds to evidence and corrects the model.

For a staff candidate, also seek score-4 evidence in relevant decisions: define a contract, weigh costs for callers and maintainers, plan adoption or migration, and identify validation and rollback needs. Advanced syntax, teaching fluency, or one score of `4` does not establish staff scope by itself.

Write a short evidence-based assessment across the sampled areas. Keep untested areas as `N/A`; avoid averaging all 70 topics into a hiring threshold. A strength in specialist APIs does not resolve an observed correctness gap in the selected work.

For study tracking, record: `topic | target depth | score | hints used | evidence | next practice`. Revisit a weakness with a different example before marking it complete.

## Prioritized Study Checklist

Tiers describe study order. Assessment depth depends on the role and selected task. Each topic has one primary tier; deeper questions remain available in any topic. For example, Tier 1 D14 begins with settings literacy, while a cross-module isolation migration is an extension.

### Tier 1: Study the Foundations First

Focus on the behavior stated here. Aim to apply it independently; revisit detailed SDK APIs when needed.

- [ ] `F5` Distinguish compiler release, language mode, target settings, SDK, and deployment target.
- [ ] `D14` Apply Approachable Concurrency with nonisolated defaults; distinguish caller-isolated and concurrent async execution.
- [ ] `A1` Predict value changes, shared references, and logical CoW independence.
- [ ] `A2` Explain binding mutability, stored values, and mutation through references.
- [ ] `A3` Apply initializer delegation and stored-property initialization rules.
- [ ] `A4` Model absence deliberately and explain optional transformations.
- [ ] `A8` Trace captures, retention, and callback lifetime.
- [ ] `A9` Model valid state alternatives and transitions.
- [ ] `A11` Apply source-file, module, and public API access boundaries.
- [ ] `A13` Preserve error information and choose throwing versus explicit results.
- [ ] `B1` Express protocol contracts and associated-type relationships.
- [ ] `B2` Choose generic, opaque, or existential contracts.
- [ ] `B3` State the relationships a generic algorithm requires.
- [ ] `B4` Predict requirement and extension-member dispatch.
- [ ] `B8` Preserve equality, hashing, and stored-key invariants.
- [ ] `B10` Design clear calls with usable construction and explicit effects.
- [ ] `C1` Diagnose ownership cycles and define resource teardown.
- [ ] `C3` Explain exclusive access and repair overlapping `inout` use.
- [ ] `C4` Choose the correct text unit and validate indices against the right string.
- [ ] `C5` Preserve collection traversal, index, and complexity contracts.
- [ ] `D1` Explain suspension, child lifetimes, blocking, and forward progress.
- [ ] `D2` Design cooperative cancellation, task ownership, and elapsed-time policies.
- [ ] `D3` Choose sequential work, `async let`, or groups with explicit failure policy.
- [ ] `D4` Identify actor-owned state and the limits of serialization.
- [ ] `D5` Keep UI ownership explicit and recognize isolated protocol conformances.
- [ ] `D6` Distinguish sendable types, sendable closures, and a specific transfer.
- [ ] `D7` Revalidate assumptions across actor suspension.
- [ ] `D11` Give unstructured tasks an owner, error path, and cancellation policy.
- [ ] `D12` Choose actors or locks and protect a complete synchronous invariant.
- [ ] `F1` Control asynchronous events and time in meaningful regression tests.
- [ ] `F2` Reduce a diagnostic and investigate state with appropriate tools.
- [ ] `F3` Form a performance hypothesis and collect relevant measurements.

### Tier 2: Apply Common App and Maintenance Skills

Learn the rule and recognize the failure mode. Develop score-3 depth in topics used by the role.

- [ ] `A5` Pattern matching, guarded cases, and external enum fallback.
- [ ] `A6` Cleanup registration, exit ordering, and awaited teardown.
- [ ] `A7` Labels, defaults, function values, and deliberate mutation.
- [ ] `A10` Extension boundaries, conditional conformances, and retroactive modeling.
- [ ] `A12` Compile-time capability checks and runtime availability.
- [ ] `A15` Choose values, reference identity, state enums, protocols, and actors.
- [ ] `B5` Typed key paths and writable capabilities.
- [ ] `B6` Dynamic casts, metatypes, and typed factory boundaries.
- [ ] `B7` Synthesized conformances and their semantic limits.
- [ ] `B9` Persisted schema evolution and malformed-input policy.
- [ ] `B13` Property-wrapper storage, initialization, and isolation.
- [ ] `B14` Result-builder transformations and API tradeoffs.
- [ ] `C2` CoW costs, reference elements, and slice retention.
- [ ] `C6` Unsafe-memory lifetime, initialization, and bounds.
- [ ] `C9` Abstraction costs and optimizer evidence.
- [ ] `C10` Optimization, assertions, and configuration-specific failures.
- [ ] `D8` Explicit isolation control and checked assumptions.
- [ ] `D9` Callback adaptation, exactly-once completion, and cancellation.
- [ ] `D10` Streams, buffering, termination, and subscription ownership.
- [ ] `D13` Target-by-target strict concurrency migration.
- [ ] `E1` Objective-C exposure and native caller contracts.
- [ ] `E2` Foundation conversion, snapshots, and shared reference payloads.
- [ ] `E5` Package graphs, target-owned resources, and external consumers.
- [ ] `F6` Observation dependencies, model isolation, and UI state lifetime.

### Tier 3: Choose Specialist Depth for the Role

Study these when their constraints occur in the work. Specialist fluency can support senior or staff performance; it is not a level by itself.

- [ ] `A14` Custom operators, literals, and evaluation semantics.
- [ ] `A16` Regex construction, input grammar, and parsing pipelines.
- [ ] `B11` Public API evolution and resilience policy.
- [ ] `B12` Parameter packs and typed variadic APIs.
- [ ] `C7` Ownership modifiers and borrowed accessors.
- [ ] `C8` Noncopyable contracts and unique owners.
- [ ] `C11` Strict memory safety and audited unsafe boundaries.
- [ ] `C12` Borrowed spans, inline storage, references, and low-level parsing.
- [ ] `E3` C ABI, native ownership, and safe wrappers.
- [ ] `E4` C++ copy/move, lifetime, and build requirements.
- [ ] `E6` Import visibility and dependency exposure.
- [ ] `E7` ABI, module stability, and binary distribution.
- [ ] `E8` Exposed bodies, ABI helpers, inlining, and specialization.
- [ ] `F4` Macro implementation, diagnostics, and compiler integration.

### Role Tracks

Select the relevant track before setting assessment depth. Add a specialist topic when it is part of the actual responsibilities.

| Work | Topics to emphasize after the foundations | Evidence to seek |
| --- | --- | --- |
| App/UI features | `A5-A7`, `A12`, `A15`, `B9`, `B13-B14`, `D10`, `E5`, `F6` | Valid state, observable dependencies, lifecycle ownership, and testable behavior. |
| Legacy app migration | `A10`, `D8-D9`, `D13`, `E1-E2`; `E3` for C boundaries | Compatible caller contracts, cancellation, isolation, and incremental rollout. |
| SDK/infrastructure | `B11`, `E5-E8`; `F4` when generation is part of the role | Usable external APIs, dependency boundaries, compatibility evidence, and adoption plans. |
| Performance/systems | `C2`, `C6-C12`, `F3`; `B12` or `E4` when used | Equivalent behavior, ownership/lifetime safety, representative measurements, and justified complexity. |

## Study Plan Order

1. Establish Approachable Concurrency with nonisolated default isolation using baseline `F5` and `D14`.
2. Work through Tier 1 semantics and ownership, using `F1-F2` to check predictions and investigate failures.
3. Study the Tier 1 concurrency topics together. Practice cancellation, stale results, blocking, and synchronization before adding API detail.
4. Apply a role track through focused Tier 2 exercises; use `F3` when making a cost claim.
5. Select Tier 3 topics for actual constraints, then extend a working solution across module, compatibility, or lifecycle boundaries.

## High-Signal Practice Formats

Use one main objective per exercise: predict behavior, explain the rule, change the code if needed, then check an observable result. Keep multi-feature scenarios as extensions.

- Trace a copied value with a reference payload; state the promised snapshot behavior.
- Predict protocol calls through concrete and generic consumers; repair a witness mismatch.
- Trace one retain cycle and verify release after explicit teardown.
- Force two actor operations to finish out of order; preserve the selected freshness policy.
- Review a semaphore wait inside a task; explain its forward-progress risk and a suitable async boundary.
- Inject a clock or time dependency into a debounce or timeout policy; test it without elapsed-time sleeps.
- Decode old and malformed fixtures under an explicit schema policy.
- Compare equivalent abstraction choices under a representative optimized workload.
- For a relevant role, evolve a public module, wrap a native API, or stage a target migration with explicit acceptance criteria.

## Minimum Interview Loop: 60 Minutes

Use three prepared scenarios. The topic IDs indicate possible evidence; select follow-ups in advance and mark everything else `N/A`.

| Time | Scenario and primary objective | Selected evidence |
| --- | --- | --- |
| 5 min | Establish target settings and explain the tasks. | Baseline `F5`/`D14`; shared assumptions. |
| 15 min | Copy an edit draft containing a reference payload, then inspect a retained callback. | `A1`, `A8`, `C1`: predict aliases, explain retention, and repair one ownership problem. |
| 15 min | Call a small protocol API through concrete and generic consumers. | `B1`, `B2`, `B4`, `B10`: predict dispatch, preserve the needed type relationship, and justify the public contract. |
| 20 min | Complete two asynchronous requests out of order, including one cancelled request. | `D1-D2`, `D4-D7`, `F1`: assign task ownership, preserve current state, and describe a deterministic regression test. |
| 5 min | Review one tradeoff and record remaining uncertainty. | Senior: validate the repair. Staff: extend one decision to callers, adoption, and failure recovery. |

For a 45-minute session, use 5 minutes for setup, 15 for one ownership or API scenario, 20 for concurrency, and 5 for discussion. This provides less breadth; preserve unassessed areas as `N/A`.

For SDK/infrastructure roles, replace the API scenario with an external-client evolution exercise from `B11`/`E6-E8`. For performance roles, replace it with an `F3` measurement investigation. For legacy roles, replace it with a `D9`/`E1` callback-boundary review. Keep the time budget and assess the selected objective.

### Version-Specific Evidence

Teach and assess the documented contract. Label a conflicting compiler or runtime result as an observation with its exact environment and a reproducible source. A candidate can explain the contract and propose a useful check without recalling a particular compiler defect.

The [validation record](<./Swift 6.4 Validation Record.md>) retains the environment and reproduction details. These optional research extensions were recorded during the collection review through 2026-09-27; this rubric revision does not claim they were rerun.

| Topic | Recorded observation | Where to investigate |
| --- | --- | --- |
| `D2` | A nested cancellation-handler fixture suppressed both outer and inner handlers under a shield. Keep this separate from the documented shielding contract. | [D2 checks](<./D. Concurrency, Isolation, and Correctness/D2 — Task Hierarchy, Cancellation, and Priorities.md>) |
| `B13` | One noncopyable generic wrapper variant crashed the tested compiler. | [B13 limitation](<./B. Type System, Abstraction, and API Design/B13 — Property Wrappers and Generated Storage Semantics.md>) |
| `C8`, `C12` | Proposed Optional reference accessors and public RigidArray were absent from the installed interface. | [C8 API checks](<./C. Memory, Ownership, Performance and Safety/C8 — Copyable, noncopyable types, and `~Copyable`.md>) |
| `E4` | Experimental C++ span-return execution failed; an empty stable-input conversion needed a guard. | [E4 native fixture](<./E. Interop, Modules, and Distribution/E4 — C++ interoperability.md>) |
| `E5` | SBOM generation skipped schema validation because the toolchain's schema bundle was missing. | [E5 tooling checks](<./E. Interop, Modules, and Distribution/E5 — Modules, packages, targets, products, and resources.md>) |
| `F6` | Continuous-observation naming and combined-event behavior differed from proposal examples. | [F6 API and runtime checks](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F6 — Observation, property wrappers, and language-adjacent state tools.md>) |

## Detailed Rubric

### A. Core Semantics, Syntax, and Control Flow

- **A1. Value semantics, reference semantics, identity, and copy-on-write**
  - Expect: Distinguish copying an ordinary value, sharing an object reference, and transferring a noncopyable value. Follow reference fields and closure captures; separate domain identity, equality, isolation, and storage cost.
  - Caveats: Array CoW does not clone class elements. A let wrapper can still expose mutable referenced state. Noncopyability is not deep uniqueness or Sendable conformance, and an extracted array value can be independent even when its owner is a shared class.
  - Score anchors: `3` — Predicts independent value edits and shared reference changes. `4` — Defines and checks a snapshot contract for reference-containing models.
  - Ask: "What question do ==, ===, and explicit domain IDs each answer?"
  - Ask: "Why do object mutation, array-element replacement, and extracting an array property have different aliasing effects?"
  - Ask: "How can a copied struct containing a closure share mutable state, and why does noncopyable ownership not guarantee a deeply unique graph?"
  - Exercise: "Copy a draft containing an array and a class payload. Check independent array edits, shared object mutation, and the original snapshot."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A1 — Value semantics, reference semantics, identity, and copy-on-write.md>).
  - Code Probe (call boxProbe):
    ```swift
    struct ValueBox {
        var items = [1]
    }

    final class RefBox {
        var items = [1]
    }

    func boxProbe() {
        let a = ValueBox()
        var b = a
        b.items.append(2)

        let c = RefBox()
        let d = c
        d.items.append(2)
        print(a.items, b.items, c.items, d.items)

        var extracted = c.items
        extracted.append(3)
        print(c.items, extracted)
    }
    ```
  - Probe outcome: Prints [1] [1, 2] [1, 2] [1, 2], then [1, 2] [1, 2, 3]. The copied ValueBox changes independently; c/d share one object's array property; the extracted Array is a separate value. These outputs establish no buffer-allocation count.

- **A2. `let`, `var`, `mutating`, and method semantics**
  - Expect: Understand value/reference mutability, mutating and nonmutating operations, and borrow/mutate accessors for stored-value projections.
  - Caveats: A nonmutating setter or method can still mutate through reference storage; protocol requirements can force awkward mutability shapes; synthesis first requires the stored types to conform and then follows their semantics. Stored-value mutate access still requires a mutable owner and exclusive access; it adds no synchronization.
  - Score anchors: `3` — Distinguishes binding, value, and referenced-object mutation. `4` — Designs accessors whose mutation and isolation costs are explicit.
  - Ask: "Why does a `struct` method that changes stored properties need `mutating`, but a class method does not?"
  - Ask: "What does `let` mean for a `struct` instance versus a `class` instance?"
  - Ask: "Why can weak let observe nil while still rejecting reassignment, and what Sendable constraints remain?"
  - Ask: "How can one mutable reference-typed stored property change a type's `Sendable` story or make a synthesized conformance less meaningful?"
  - Ask: "Why can a reference-backed nonmutating setter work through let while a stored-value mutate accessor cannot?"
  - Exercise: "Compare a stored-value mutation with a reference-backed nonmutating setter through let. Identify which owner must be mutable and what state is shared."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A2 — `let`, `var`, `mutating`, and method semantics.md>).
  - Code Probe:
    ```swift
    struct Counter {
        private final class Storage {
            var value = 0
        }

        private let storage = Storage()

        var value: Int {
            get { storage.value }
            nonmutating set { storage.value = newValue }
        }
    }

    let counter = Counter()
    counter.value = 3
    print(counter.value)
    ```
  - Probe Ask: "Why does this compile even though `counter` is bound with `let`?"
  - Probe Ask: "Why is this legal, why might it still be surprising, and when would this design be justified or dangerous?"

- **A3. Initialization model**
  - Expect: Know designated vs convenience initializers, two-phase initialization, memberwise initialization, failable init, `required`, and initialization safety rules. Include Swift 6.4 initialized-private-property exclusion when explaining memberwise synthesis.
  - Caveats: Direct initial storage assignment does not fire observers; a helper called after stored properties are initialized can. Whole-instance access is restricted before initialization, but a direct read of initialized storage can be valid. These rules do not establish domain validity; subclass ordering matters.
  - Score anchors: `3` — Initializes subclass storage before delegating and customizes inherited mutable properties afterwards. `4` — Preserves construction invariants while evolving a public type.
  - Ask (extension, Swift 6.4): "How does adding a private initialized cache differ from adding a private property that requires a value?"
  - Ask: "Why can a direct stored-field read succeed where a computed-property read fails during initialization? When can an observer run from an initializer?"
  - Ask: "Why can't you access `self` freely before all stored properties are initialized?"
  - Ask: "When does Swift synthesize a memberwise initializer, and when does it disappear?"
  - Ask: "In what order must a designated subclass initializer initialize its own stored properties, call super.init, and modify inherited properties? How does a convenience initializer delegate through self.init?"
  - Exercise: "Initialize a subclass with one new and one inherited property. Show valid delegation order and explain a rejected ordering."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A3 — Initialization model.md>).

- **A4. Optionals and nil modeling**
  - Expect: Explain Optional cases, binding/patterns, chaining, map/flatMap, lazy defaults, try? flattening, and IUO inference. Include Swift 6.4 optional some/any syntax and per-operation noncopyable payload constraints.
  - Caveats: An optional alone cannot distinguish missing, cleared, and failed states without an additional contract. A default can hide absence; empty text is still a present value. A force unwrap asserts presence.
  - Score anchors: `3` — Preserves the intended meaning of nil and optional transformations. `4` — Chooses a representation that preserves distinctions required by callers.
  - Ask: "How do foo?.bar() and foo!.bar() differ, including whether call arguments or assignment work run?"
  - Ask: "When does map preserve a distinction that flatMap merges? Why does try? behave differently for an already optional result?"
  - Ask: "How do dictionary lookup, nil assignment, and .some(nil) distinguish absent keys from stored nil?"
  - Ask: "When does an IUO remain optional through inference, and when is it implicitly forced?"
  - Ask (extension, Swift 6.4): "How do some P? and any P? relate to the parenthesized spellings? Why does a composition still need parentheses?"
  - Ask (extension): "Why can Token? store a noncopyable value while this SDK's public map rejects Token as its input payload?"
  - Exercise: "Model an optional profile name with a lazy default. Check absence, empty text, and whether the default expression runs."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A4 — Optionals and Nil Modeling.md>).

- **A5. Pattern matching and exhaustive control flow**
  - Expect: Explain first-match ordering, payload bindings, guarded subsets, optional filtering, tuple/grouped patterns, fallthrough, and external enum fallback. Separate coverage from business correctness.
  - Caveats: where belongs to its individual comma-separated pattern. unknown default can execute for an omitted known case while producing a warning. Nonexhaustive source checking and binary resilience are separate contracts.
  - Score anchors: `3` — Accounts for every case and the unguarded remainder of a payload. `4` — Chooses an external-enum fallback that supports the API's evolution policy.
  - Ask: "Why does this guarded case need remaining payload coverage?"
  - Ask: "Which external enum omission is an error in Swift 6 mode, and which warns while still reaching a fallback?"
  - Exercise: "Map a response enum using guarded status cases. Check success, unexpected status, failure, and cancellation; explain remaining coverage."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A5 — Pattern Matching and Exhaustive Control Flow.md>).
  - Code Probe:
    ```swift
    enum Response: Sendable {
        case completed(status: Int)
        case failed(message: String)
        case cancelled
    }

    func render(_ response: Response) -> String {
        switch response {
        case let .completed(code) where (200..<300).contains(code):
            "ok \(code)"
        case let .completed(code):
            "unexpected \(code)"
        case let .failed(message):
            "failure \(message)"
        case .cancelled:
            "cancelled"
        }
    }
    ```
  - Probe outcome: completed status 204 yields ok 204; 404 yields unexpected 404. The failed and cancelled shapes have explicit behavior. Adding a new Response case or removing the remaining completed-payload branch is rejected. Follow up with the chapter's separately compiled SDKStates consumer.

- **A6. `defer`, early exits, and cleanup semantics**
  - Expect: Understand `defer` ordering, especially around `throw`, `return`, and nested scopes. Include Swift 6.4 awaited cleanup in async contexts and distinguish waiting from cancellation shielding.
  - Caveats: Defer follows lexical scope and must be reached. Cancellation alone does not force scope exit; fatal traps/process termination do not guarantee cleanup. A return expression is evaluated before cleanup, but a returned reference can still expose mutations to shared objects.
  - Score anchors: `3` — Traces registered cleanup on ordinary scope exits. `4` — Defines failure and cancellation policy for bounded asynchronous teardown.
  - Ask (extension, Swift 6.4): "Can a defer await, and when is its cleanup complete relative to return or catch?"
  - Ask: "In what order do multiple `defer` blocks execute?"
  - Ask: "Where does `defer` become clearer than duplicating cleanup code in `catch` and `return` paths?"
  - Exercise: "Register cleanup in nested scopes with return and throw paths. Trace reached defers and verify reverse registration order in each scope."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A6 — `defer`, Early Exits, and Cleanup Semantics.md>).

- **A7. Functions, parameter labels, defaults, and API surface**
  - Expect: Design clear calls using labels, defaults, domain types, variadics/collections, and deliberate mutation. Explain function values, default expression evaluation, trailing-closure matching, ownership, effects, and isolation.
  - Caveats: Caller code supplies default expressions, but a library getter/helper used by one can change after replacement. Options types do not freeze policy or guarantee ABI compatibility. Inout is scoped exclusive mutation; an async local use can compile while direct actor-storage use is rejected.
  - Score anchors: `3` — Predicts labels, function-value types, and omitted defaults. `4` — Evolves defaults and construction without hiding policy changes from clients.
  - Ask: "Which parts of a declaration survive as a function value, and how do you select between equal-typed overloads with different labels?"
  - Ask: "Why can an unchanged client retain a literal default but observe a changed named/helper-based default?"
  - Ask: "How do default selection and virtual method dispatch interact?"
  - Ask: "Why does passing one evaluated value plus one inout differ from two mutable aliases, and what changes across suspension?"
  - Exercise: "Replace positional request parameters with labeled domain inputs. Check call-site clarity, forwarded values, and default evaluation."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A7 — Functions, Parameter Labels, Defaults, and API Surface.md>).
  - Code Probe:
    ```swift
    class Base {
        func pick(_ value: Int = 1) -> String { "base:\(value)" }
    }

    final class Derived: Base {
        override func pick(_ value: Int = 2) -> String { "derived:\(value)" }
    }

    func dispatchDefaults() -> [String] {
        let derived = Derived()
        let base: Base = derived
        let function = base.pick
        return [derived.pick(), base.pick(), function(3)]
    }
    ```
  - Probe outcome: derived:2, derived:1, derived:3. The static declaration supplies the omitted default, the override supplies the virtual body, and the function value requires an explicit argument.

- **A8. Closures, escaping, and capture semantics**
  - Expect: Explain shared captured storage, value/reference snapshots, escaping versus nonescaping lifetime, optional callback values, explicit-self exceptions, weak/unowned references, autoclosure evaluation, and isolation/transfer constraints.
  - Caveats: Shared mutable-variable behavior does not guarantee a physical heap box. Strong async capture can extend lifetime without a permanent leak. Bound methods and nested closures can retain owners; weak self promoted before await can remain strong across suspension. weak let and ImmutableWeakCaptures are separate controls.
  - Score anchors: `3` — Explains captured storage and repairs a concrete retain cycle. `4` — Chooses a callback lifetime contract and validates teardown across suspension.
  - Ask: "How do direct nonescaping parameters, optional function values, and escaping parameters differ, and does any spelling promise later/concurrent execution?"
  - Ask: "What do a closure copy, [value], [object], [weak self], and [unowned self] retain or observe?"
  - Ask: "When is implicit self valid after a capture list, and why does a weak inner closure not remove a strong outer cycle?"
  - Ask: "How do early and late promotion of weak self affect retention across await, cancellation, and deinit?"
  - Exercise: "Trace an owner, stored callback, and captured reference. Break the cycle and check both live-owner behavior and post-release invocation."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A8 — Closures, Escaping, and Capture Semantics.md>).
  - Code Probe:
    ```swift
    final class LifetimeLog {
        var events: [String] = []
    }

    final class Screen {
        let log: LifetimeLog
        var onTap: (() -> Void)?
        private(set) var tapCount = 0
        init(log: LifetimeLog) { self.log = log }
        deinit { log.events.append("Screen deinit") }

        func bindStrong() { onTap = { self.handleTap() } }
        func bindMethod() { onTap = handleTap }
        func bindWeak() {
            onTap = { [weak self] in
                guard let self else { return }
                handleTap()
            }
        }
        private func handleTap() { tapCount += 1 }
    }

    func cycleProbe() -> [String] {
        let log = LifetimeLog()
        weak var observed: Screen?
        do {
            let screen = Screen(log: log)
            screen.bindStrong()
            observed = screen
        }
        let retained = observed != nil
        observed?.onTap = nil
        return [retained ? "retained" : "released"] + log.events
    }
    ```
  - Probe outcome: cycleProbe returns retained followed by Screen deinit after explicit cycle removal. A stored bound method also retains Screen. A weak handler permits release, and a copied weak callback tolerates later invocation. Explain the graph before generalizing from deinit timing.

- **A9. Enums, associated values, recursive enums, and state modeling**
  - Expect: Model mutually exclusive alternatives while preserving legitimate stale-content refresh. Separate case shape, payload validity, transition validity, and execution of effects. Explain recursive storage, reference payload sharing, and noncopyable payload ownership.
  - Caveats: Separate flags can represent combinations the product forbids. An enum alone does not validate payloads, reject old callbacks, or execute cancellation. Copying a copyable enum does not clone class payloads. Loading with previous data can be a valid refresh state.
  - Score anchors: `3` — Models the required alternatives without contradictory flags. `4` — Separates payload and transition validity from execution of effects.
  - Ask: "Which combinations should this product preserve, and which should the type exclude?"
  - Ask: "How can a retry timer or cancellation acknowledgment arrive after replacement, and what prevents it from changing the current state?"
  - Exercise: "Represent loading, loaded, and failed states while retaining previous data where required. Check allowed transitions and excluded combinations."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A9 — Enums, Associated Values, Recursive Enums, and State Modeling.md>).
  - Code Probe (use the chapter's LoadState declaration) — [setup and shared declarations](<./A. Core Semantics, Syntax and Control Flow/A9 — Enums, Associated Values, Recursive Enums, and State Modeling.md>):
    ```swift
    func snapshotProbe() {
        var state = LoadState<[Int]>.loaded([1, 2])
        let original = state
        state = .loading(previous: state.visibleValue)
        print(state.isLoading, state.visibleValue ?? [])
        state = .failed(message: "offline", previous: state.visibleValue)
        print(state.isLoading, state.errorMessage ?? "none", state.visibleValue ?? [])
        print(original.isLoading, original.errorMessage == nil, original.visibleValue ?? [])
    }
    ```
  - Probe outcome: The loading and failed snapshots preserve [1, 2]; the original remains loaded. Output is true [1, 2], then false offline [1, 2], then false true [1, 2]. Follow up with the chapter's reference-payload and stale-completion probes.

- **A10. Extensions and retroactive modeling**
  - Expect: Distinguish member organization, synthesized/manual witnesses, constrained helpers, conditional conformances, and retroactive ownership. Validate file placement, imports, isolation, accessible construction, and semantic contracts.
  - Caveats: Moving an empty synthesized conformance into another file can fail. A typealias is not a new type. Retroactive acknowledgment neither creates a private conformance nor resolves duplicates; ordinary extension members can also make source lookup ambiguous. Runtime conformance discovery is not contained by a source import boundary.
  - Score anchors: `3` — Separates extension helpers, witnesses, and retroactive ownership. `4` — Chooses an adapter boundary that tolerates upstream API evolution.
  - Ask: "Who owns the type/protocol, and what does the same-package exception mean for warning behavior?"
  - Ask: "Why can an imported-type Codable extension still fail after adding retroactive, and what does a manual witness require?"
  - Ask: "How do static generic use, runtime conformance discovery, and member-name ambiguity differ across modules?"
  - Exercise: "Replace an imported-type conformance with an owned adapter. Check construction, mapping, and the adapter's protocol semantics."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A10 — Extensions and Retroactive Modeling.md>).
  - Code Probe (compile independently from its alternatives):
    ```swift
    import Foundation

    extension Date: Identifiable {
        public var id: TimeInterval { timeIntervalSince1970 }
    }
    ```
  - Probe outcome: Swift 6.4 emits the imported-type/imported-protocol conformance warning. The separate retroactive and fully qualified variants acknowledge it. The timestamp policy still does not establish unique event identity, and acknowledgment does not silence duplicate-conformance or synthesis errors.

- **A11. Access control**
  - Expect: Distinguish private/fileprivate/internal/package/public/open through actual source-file, module, and package boundaries. Check construction, member, setter, witness, subclassing, and override access separately.
  - Caveats: Public/package type visibility does not publish synthesized initializers. SE-0502 changes memberwise participation, not the internal access ceiling. Testable clients have extra permissions. Export/body controls do not make a private/internal name public source API, and access control does not establish isolation or security.
  - Score anchors: `3` — Checks construction and member access across a real module boundary. `4` — Limits package and public contracts while preserving legitimate clients.
  - Ask: "Why can a sibling package module name a struct but fail to construct it, and why does Swift 6.4 not fix that automatically?"
  - Ask: "How do ordinary external, same-package, and testable clients differ when reading members or subclassing/overriding?"
  - Ask: "Why can a same-file type extension read private state while an unrelated helper or another-file extension cannot?"
  - Ask: "How do private(set), usableFromInline, inlinable, and export(interface) constrain different aspects of an API?"
  - Exercise: "Expose a small public value from a module. Compile an ordinary client that constructs and reads it, and reject private member access."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A11 — Access Control.md>).
  - Code Probe:
    ```swift
    struct Draft {
        var title: String
        private var revision = 0
    }

    extension Draft {
        var revisionNumber: Int { revision }
        static func restored(title: String, revision: Int) -> Self {
            Self(title: title, revision: revision)
        }
    }

    func initializerAccessProbe() -> [Int] {
        let fresh = Draft(title: "New")
        let restored = Draft.restored(title: "Old", revision: 7)
        return [fresh.revisionNumber, restored.revisionNumber]
    }
    ```
  - Probe outcome: initializerAccessProbe returns 0,7. The new internal memberwise initializer omits initialized private revision; the same-file type extension can still use the private compatibility overload. A separate file can use Draft(title:), but ordinary external clients do not gain public or package construction from synthesis.

- **A12. Availability and conditional compilation**
  - Expect: Separate compiler, language-mode, SDK/module, and runtime requirements; use anyAppleOS with valid aligned versions and specific-platform overrides, alongside availability/deprecation and source guards.
  - Caveats: Runtime availability and compile-time conditional compilation solve different problems.
  - Score anchors: `3` — Chooses the guard that matches the missing capability. `4` — Maintains one public contract across supported compiler and runtime combinations.
  - Ask: "What is the difference between `#available` and `#if canImport`?"
  - Ask: "Why can a call guarded by anyAppleOS 26 still fail for macOS, and why is that guard not a test for an Apple target?"
  - Ask: "How would you deprecate an API while giving users an actionable migration path?"
  - Exercise: "Guard one newer API behind a fallback. Explain runtime availability versus compiler/module guards and compile for the older deployment target."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A12 — Availability and Conditional Compilation.md>).

- **A13. Error handling and typed throws**
  - Expect: Know `throws`, `rethrows`, `Result`, `try?`, `try!`, and modern typed throws in Swift 6. Include Swift 6.4 async Result initialization and its typed-error and cancellation implications.
  - Caveats: try? collapses information; typed error types constrain API evolution. Async Result can need an explicit closure error annotation and does not clear cancellation. Rethrows cannot describe independent failure; enum-case catch clauses are not equivalent to an exhaustive switch.
  - Score anchors: `3` — Chooses throwing or explicit results without silently erasing failure. `4` — Explains error-type evolution and cancellation policy at a public boundary.
  - Ask (extension, Swift 6.4): "Does awaiting a Result initializer start a new task or handle cancellation for you?"
  - Ask: "Why can separate typed enum-case catches fail exhaustiveness checking while an unconditional catch with a switch works?"
  - Ask: "When should you return `Result` instead of throwing?"
  - Ask: "What does `throws(Never)` mean semantically?"
  - Exercise: "Implement a failing transform with throws, then expose its outcome as Result. Check success and preserved error information."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A13 — Error handling and typed throws.md>).

- **A14. Operators, literals, and language affordances**
  - Expect: Separate operator grouping, overload selection, argument evaluation, and domain semantics. Explain checked/wrapping/reporting overflow, literal inference, explicit conversion, and nonthrowing/nonfailable literal requirements.
  - Caveats: Precedence does not create short-circuiting. Predicate operands can be built eagerly while their later matching short-circuits. Literal preconditions execute at runtime; string-literal conformance does not imply interpolation or conversion from a String variable. SE-0508 adds parsing, not standard builders or callable arrays.
  - Score anchors: `3` — Separates grouping, operand evaluation, and operator behavior. `4` — Justifies a symbolic API against a clearer named alternative.
  - Ask: "When does a symbol improve a domain API, and what failure, unit, effect, or cost does it hide?"
  - Ask: "Which parts of Predicate conjunction are eager, and what actually skips its right-hand matching work?"
  - Ask: "Why does a literal construct a custom value while assigning a String variable fails? Can its literal initializer throw?"
  - Ask: "How can changing parentheses alter an eligibility policy, and how can public precedence/overload changes affect existing clients?"
  - Exercise: "Compare an eager Boolean operator with an autoclosure-based short-circuiting operator. Trace which right-hand expressions execute."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A14 — Operators, Literals, and Language Affordances.md>).
  - Code Probe:
    ```swift
    infix operator &&&: LogicalConjunctionPrecedence
    infix operator &&?: LogicalConjunctionPrecedence

    func &&& (lhs: Bool, rhs: Bool) -> Bool {
        lhs && rhs
    }

    func &&? (lhs: Bool, rhs: @autoclosure () -> Bool) -> Bool {
        lhs && rhs()
    }

    func evaluationProbe() -> [String] {
        var events: [String] = []
        func mark(_ event: String) -> Bool {
            events.append(event)
            return true
        }
        _ = false && mark("standard")
        _ = false &&& mark("eager")
        _ = false &&? mark("lazy")
        _ = true &&? mark("needed")
        return events
    }
    ```
  - Probe outcome: evaluationProbe returns eager then needed. The ordinary Bool operand has already been evaluated; the autoclosure version defers its expression and this implementation invokes it only when lhs is true. Both custom operators use the same precedence group.

- **A15. Type choices: struct, class, enum, protocol, and actor**
  - Expect: Choose by independent value behavior, domain/reference identity, shared versus exclusive ownership, state alternatives, capability boundaries, and isolation. Distinguish Copyable, Sendable, default isolation, and representation.
  - Caveats: Structs can carry shared references; noncopyable values support deinit; loading plus previous data can be valid. Actors and MainActor do not guarantee freshness across await, deduplication, task ownership, or observation. A protocol can be useful with one production implementation.
  - Score anchors: `3` — Chooses types from identity, value behavior, and isolation needs. `4` — Balances caller contracts, lifecycle ownership, and future API changes.
  - Ask: "What does copying a struct containing a class or actor actually share, and does the struct satisfy Sendable?"
  - Ask: "Which requirements justify a class, a noncopyable struct, a MainActor owner, an actor, or a Mutex-backed type?"
  - Ask: "Why can a model's latest-request-wins display still coexist with an older value in a shared cache?"
  - Exercise: "Choose a representation for an independent draft and a shared session. Check copy behavior and explain who owns mutable state."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A15 — Type choices - struct, class, enum, protocol, and actor.md>).
  - Code Probe:
    ```swift
    struct Profile {
        var name: String
    }

    final class Session {
        var token: String
        init(token: String) { self.token = token }
    }

    struct FeatureSnapshot {
        var profile: Profile
        let session: Session
    }

    func representationProbe() -> [String] {
        let original = FeatureSnapshot(profile: Profile(name: "A"), session: Session(token: "old"))
        var copy = original
        copy.profile.name = "B"
        copy.session.token = "new"
        return [original.profile.name, copy.profile.name,
                original.session.token, copy.session.token,
                String(original.session === copy.session)]
    }
    ```
  - Probe outcome: representationProbe returns A, B, new, new, true. The Profile is copied as a value; both snapshots retain the same mutable Session. A Sendable conformance is rejected for that payload unless its representation/contract changes; unchecked conformance would not supply synchronization.

- **A16. Regex and standard-library idioms**
  - Expect: Specify the accepted grammar, distinguish whole/prefix/substring matching, and choose literals or RegexBuilder deliberately. Explain map, nil-dropping compactMap, Sequence flatMap, mutable reduction, and lazy evaluation precisely.
  - Caveats: Digit matching does not establish Int representability; date shape does not establish calendar validity. String matching conveniences can suppress matching errors. Lazy traversal is not memoization, and typed Regex output does not imply Sendable.
  - Score anchors: `3` — Matches the stated grammar and handles conversion failure. `4` — Chooses a parsing and rejection policy that accounts for every input.
  - Ask: "How do this literal and builder differ in accepted whitespace, digits, and trailing input?"
  - Ask: "Which failures become nil, which throw, and what would an importer need to account for every input row?"
  - Exercise: "Specify and parse an ASCII user-ID grammar. Check whitespace, trailing text, invalid digits, and integer overflow."
  - Extension: [Full chapter exercise and deeper checks](<./A. Core Semantics, Syntax and Control Flow/A16 — Regex and standard-library idioms.md>).
  - Code Probe:
    ```swift
    import RegexBuilder

    func parseUserIDBuilder(_ input: String) -> Int? {
        let regex: Regex<(Substring, Int)> = Regex {
            "user_id:"
            ZeroOrMore { CharacterClass.anyOf(" \t") }
            TryCapture {
                OneOrMore { CharacterClass("0"..."9") }
            } transform: { Int($0) }
        }.matchingSemantics(.unicodeScalar)
        return input.wholeMatch(of: regex)?.1
    }
    ```
  - Probe outcome: user_id:42 and user_id: followed by a tab and 00042 return 42. Overflowing decimals, Unicode lookalike digits, newlines, wrong keys, and trailing text return nil. The chapter's literal parser passes the same corpus; the Int capture comes from TryCapture's failable transform.

### B. Type System, Abstraction, and API Design

- **B1. Protocols, associated types, `Self` requirements, and compositions**
  - Expect: Design behavioral contracts and preserve associated-type/Self relationships. Distinguish existential formation from usable members, composition from nominal refinement, and generic/constrained-existential/conforming-erased consumers. Recognize that a conformance can be restricted to a global actor.
  - Caveats: An associated-type result can sometimes erase safely while an unknown input blocks member access. Custom markers do not enforce arbitrary semantic guarantees. Check ownership defaults and actual SDK conformance limits.
  - Score anchors: `3` — Preserves associated-type relationships needed by the consumer. `4` — Chooses generic, existential, or erased contracts without losing required capabilities.
  - Ask: "Why does any Parser permit the demonstrated parse operation while an unconstrained lookup cannot accept a String key?"
  - Ask: "Why can a Self result be erased while two independent existential Self inputs cannot be combined? How can a generic helper double one opened value?"
  - Ask: "Does satisfying P and Q automatically establish conformance to a new protocol refining them? Does a get-only requirement prove immutable storage?"
  - Ask: "Why can a value be Sendable while one of its protocol conformances is MainActor-isolated? What must a generic consumer preserve?"
  - Ask (extension): "What must be suppressed for a noncopyable factory payload, and why does noncopyable Hashable support not imply Set or CustomStringConvertible support?"
  - Exercise: "Define a keyed lookup protocol and a typed consumer. Check one accepted lookup and one rejected key-type mismatch."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B1 — Protocols, Associated Types, `Self` Requirements, and Compositions.md>).
  - Code Probe:
    ```swift
    protocol Lookup<Key, Value> {
        associatedtype Key: Hashable
        associatedtype Value
        func get(_ key: Key) -> Value?
    }

    struct StringIntLookup: Lookup {
        let values: [String: Int]
        func get(_ key: String) -> Int? { values[key] }
    }

    func cannotReadErased(_ lookup: any Lookup) {
        _ = lookup.get("user:1")
    }
    ```
  - Probe outcome: Swift 6.4 rejects get on the unconstrained existential because Key is unknown. A generic reader and any Lookup<String, Int> both support typed lookup; implementation erasure need not erase the key/value relationship.

- **B2. Existentials (`any`) vs opaque types (`some`)**
  - Expect: Choose concrete, generic, opaque, and existential contracts by type ownership and retained relationships. Explain result versus parameter some, generic opaque families, associated-type constraints, existential opening, and Swift 6.4 optional spelling.
  - Caveats: Existentials need not allocate; generics need not specialize. Opening does not give any P general self-conformance. A named generic parameter constrains types after permitted conversions, not necessarily dynamic subclass identity.
  - Score anchors: `3` — Distinguishes caller-selected generics, hidden concrete results, and existential values. `4` — Chooses an abstraction boundary using type relationships and measured costs.
  - Ask: "Why can two calls to one opaque factory compare while two factories returning Int behind some Equatable cannot?"
  - Ask: "Why can a plain existential enter a generic call while an invariant Holder result or two independent existential arguments fail?"
  - Ask: "What does any Sequence<Int> preserve, and what do neither any Sequence<Int> nor some Sequence<Int> promise about ordering, eagerness, or performance?"
  - Ask: "How do some P? and any P? relate to the older parenthesized spellings? Why can bare nil or an optional existential still fail at a generic parameter?"
  - Ask: "How do AnyView, any View, and a ViewBuilder result differ?"
  - Exercise: "Expose two implementations through a generic or existential API. Explain who selects the concrete type and check the intended caller operations."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B2 — Existentials (`any`) vs Opaque Types (`some`).md>).
  - Code Probe:
    ```swift
    protocol ImageSource {}
    struct Remote: ImageSource {}
    struct Local: ImageSource {}

    func makeSource(_ local: Bool) -> some ImageSource {
        if local {
            return Local()
        } else {
            return Remote()
        }
    }
    ```
  - Probe Ask: "Why do these return paths fail? Compare an existential result, one conforming enum wrapper, and a caller-selected generic factory."
  - Probe outcome: Swift 6.4 rejects nonmatching underlying return types. All three redesigned contracts are checked in B2; performance and UI rendering were not benchmarked.

- **B3. Generics, constraints, and same-type relationships**
  - Expect: Express capability and same-type constraints, conditional conformances, generic return relationships, and Copyable/Escapable defaults. Explain Swift 6.4 primary associated-type defaults and their explicit suppression.
  - Caveats: Type equality does not prove equal values, lengths, indices, or ordering. Array literals can infer Any and satisfy a shared-element constraint. Generalizing a protocol does not automatically generalize every generic use or extension. Generics do not guarantee specialization.
  - Score anchors: `3` — States the capability and same-type constraints the implementation needs. `4` — Generalizes only the relationships the API can actually support.
  - Ask: "What does A.Element == B.Element prove beyond A: Sequence and B: Sequence? What additional constraint does equality or hashing need?"
  - Ask: "Why can a protocol permit a noncopyable Payload while a generic use still requires it to be Copyable? How do primary and non-primary associated types differ here?"
  - Ask: "Why can a Token fail an ordinary generic parameter, while a Span fails for a different implicit constraint?"
  - Ask: "Why can mixed array literals satisfy a same-element constraint, and why should a negative interview probe use explicit types?"
  - Exercise: "Repair appendAll with the required element relationship. Check Array and ArraySlice inputs and an explicitly typed mismatch."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B3 — Generics, Constraints, and Same-Type Relationships.md>).
  - Code Probe:
    ```swift
    func appendAll<S: Sequence, C: RangeReplaceableCollection>(
        _ source: S,
        to destination: inout C
    ) {
        for element in source {
            destination.append(element)
        }
    }
    ```
  - Probe outcome: Swift 6.4 rejects passing S.Element to an append operation requiring C.Element. Add where S.Element == C.Element. B3 includes checked Array/ArraySlice usage, negative type specializations, and the compiler's diagnostic limitation for some invalid direct calls.

- **B4. Dispatch model: static, class virtual, witness-table, and protocol extension dispatch**
  - Expect: Predict declaration/witness selection through concrete, generic, existential, superclass, and Objective-C call surfaces. Distinguish extension-only helpers, inherited default witnesses, and constrained defaults from class overrides.
  - Caveats: Optimization preserves semantic choices; it does not rerun overload resolution for a generic caller. A same-named child method does not replace an inherited default witness. @objc exposure, @objc dynamic, and native Swift dynamic are distinct. Isolated conformances restrict their consumers; this rubric infers no MainActor default for ordinary protocol extensions.
  - Score anchors: `3` — Predicts requirement versus extension-only member selection. `4` — Designs an overridable witness path and explains inherited-default limits.
  - Ask: "Why can a child class's concrete render differ from its generic or existential render? How does an overridable base member change the witness path?"
  - Ask: "Why can direct Box<Int> lookup select a constrained default while its unconditional conformance still uses the general witness?"
  - Ask: "Which choices are language semantics, and what do SIL instructions or an optimized run actually establish?"
  - Ask: "Why must a caller satisfy a conformance's isolation boundary as well as select the correct witness?"
  - Ask: "When does a selector need @objc, when does KVO property access need dynamic behavior, and why can a Swift struct have a dynamic method?"
  - Exercise: "Predict f and g through concrete, generic, and existential consumers. Make g a requirement and check the changed witness path."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch.md>).
  - Code Probe:
    ```swift
    protocol P {
        func f()
    }

    extension P {
        func f() { print("default f") }
        func g() { print("extension g") }
    }

    struct S: P {
        func f() { print("S.f") }
        func g() { print("S.g") }
    }

    func probe() {
        let value: any P = S()
        value.f()
        value.g()
    }
    ```
  - Probe outcome: Calling probe prints S.f then extension g. The generic path keeps the same behavior under the checked optimized build. Making g a requirement changes its witness selection; The linked B4 chapter also checks inherited witness exceptions and alternative default-isolation settings.

- **B5. Key paths and strongly typed indirection**
  - Expect: Preserve Root, Value, writable/reference-writable capability, and Sendable requirements through projection and descriptor APIs. Explain partial/full erasure, captured subscript arguments, and actor-isolation boundaries.
  - Caveats: A computed getter can have effects; a path does not cache the result or synchronize the root. ReferenceWritableKeyPath can have a struct root. Literal-to-function conversion does not generally convert stored path variables. The checked appending overload can lose writable/Sendable capabilities; the explicit noncopyable-root signature still requires Copyable.
  - Score anchors: `3` — Preserves root, value, and writable capability. `4` — Chooses key paths or closures with explicit effects and isolation boundaries.
  - Ask: "When does a key path communicate the right access contract, and when is a closure clearer?"
  - Ask: "Why can a let Screen struct support a reference-writable path while a let Settings class fails after erasure to WritableKeyPath?"
  - Ask: "What can a Sendable key path capture, and why does it not make its root safe to share?"
  - Ask: "Does stable sorting define a canonical order for ties, and what if a computed key changes during comparison?"
  - Exercise: "Write a typed projection helper and apply a writable path to a mutable root. Explain a rejected write after capability erasure."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B5 — Key paths and strongly typed indirection.md>).
  - Code Probe:
    ```swift
    final class Reading {
        private(set) var accesses = 0
        var value: Int {
            accesses += 1
            return accesses
        }
    }

    func getterProbe() -> [Int] {
        let reading = Reading()
        let path: KeyPath<Reading, Int> & Sendable = \Reading.value
        return [reading[keyPath: path], reading[keyPath: path], reading.accesses]
    }
    ```
  - Probe outcome: getterProbe returns [1, 2, 2]. Each application runs the getter. The path is Sendable; the mutable Reading object has no Sendable or synchronization guarantee. B5 separately checks a path and immutable User crossing an actor boundary.

- **B6. Any, AnyObject, metatypes, and runtime type information**
  - Expect: Distinguish erasure, bridging/casting, typed/generalized/protocol metatypes, dynamic factory selection, runtime identity, reflection, module selectors, and Runtime demangling. Keep registry erasure private while specifying lifecycle and isolation.
  - Caveats: Any and Any.Type do not cover the checked noncopyable domain. any P.Type differs from (any P).Type. T() can ignore a supplied subclass metatype. SendableMetatype does not make instances Sendable. Runtime is absent from the checked iOS SDK; macOS demangling needs OS 27. Module selectors were introduced in 6.3.
  - Score anchors: `3` — Uses the supplied metatype when runtime factory selection is intended. `4` — Maintains a typed registry invariant across erased storage and lifecycle policy.
  - Ask: "What information does each metatype form preserve, and when can type(of:) in generic code report the existential container?"
  - Ask: "Why can T() and type.init() return different dynamic classes for one argument?"
  - Ask: "How do actor ownership, typed registration, missing entries, duplicates, and instance lifetime interact in a factory registry?"
  - Ask: "Why are reflected or demangled names unsuitable persisted type IDs, and how do module presence and OS availability differ?"
  - Exercise: "Compare T() with initialization through a supplied class metatype. Check the resulting dynamic types and reject an unrelated registration."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B6 — Any, AnyObject, Metatypes, and Runtime Type Information.md>).
  - Code Probe:
    ```swift
    class Animal {
        required init() {}
    }
    final class Dog: Animal {}

    func staticMake<T: Animal>(_ type: T.Type) -> T {
        T()
    }

    func dynamicMake<T: Animal>(_ type: T.Type) -> T {
        type.init()
    }

    func factoryComparison() -> [Bool] {
        let selected: Animal.Type = Dog.self
        let first = staticMake(selected)
        let second = dynamicMake(selected)
        return [type(of: first) == Animal.self, type(of: second) == Dog.self]
    }
    ```
  - Probe outcome: factoryComparison returns [true, true]. T is Animal while the selected metatype is Dog; type.init() honors that runtime selection. The actor registry uses the received metatype, not P(), and keeps its key/result invariant private.

- **B7. Synthesized conformances and semantic correctness**
  - Expect: Explain Equatable/Hashable/Codable/CaseIterable derivation and distinguish it from Sendable marker checking/inference. Verify noncopyable protocol support, synthesis, and consumers separately.
  - Caveats: Identifiable does not make equality ID-only. Entity snapshots can properly use structural equality. A custom == does not make synthesized hashing omit other stored properties. Noncopyable struct Hashable synthesis works in the checked 6.4 compiler; the enum example needs manual borrowing witnesses. Set and Codable still require Copyable.
  - Score anchors: `3` — Distinguishes synthesized behavior from intended identity and equality. `4` — Keeps selection, content comparison, and persistence contracts consistent.
  - Ask: "Why can snapshots with the same ID legitimately compare unequal, and how should selection keys differ from content comparison?"
  - Ask: "What goes wrong when == compares only id but hash(into:) is still synthesized?"
  - Ask: "Does a default-valued decoded var handle a missing key? What does Sendable actually generate?"
  - Exercise: "Compare snapshots with the same ID and changed content. Customize equality, then make hashing use matching components and check both."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B7 — Synthesized conformances and semantic correctness.md>).
  - Code Probe:
    ```swift
    struct UserSnapshot: Identifiable, Hashable {
        let id: Int
        var name: String
    }

    func snapshotComparison() -> [Bool] {
        let before = UserSnapshot(id: 1, name: "A")
        let after = UserSnapshot(id: 1, name: "B")
        return [before.id == after.id, before == after]
    }
    ```
  - Probe outcome: snapshotComparison returns [true, false]. Same identity does not imply equal content. B7 also checks closure synthesis failure, conditional conformances, coding keys, and the noncopyable synthesis boundary.

- **B8. Equatable, Hashable, and collection correctness**
  - Expect: Explain equality laws, matching hashing components, collision handling, stable keys, reference aliases, and insertion versus replacement. Include Swift 6.4's additional collection conformances and their OS availability.
  - Caveats: Independent value-copy mutation is safe; changing collection-observed key state through an alias is not. Hash values and ObjectIdentifier are not durable domain IDs. Generalized Hashable does not remove Set's Copyable requirement.
  - Score anchors: `3` — Maintains equal-value hashing and stable stored-key state. `4` — Chooses identity and replacement semantics that remain valid through aliases.
  - Ask: "Why does equal hash not establish equality, and why are persistent hashValue keys wrong?"
  - Ask: "How can a struct containing a reference have the same mutable-key bug as a class?"
  - Ask: "Why can insert retain stale content after an ID-only equality fix, and when should update(with:) or dictionary assignment be used?"
  - Ask: "What does Dictionary.Keys hashing ignore, and why can EmptyCollection<NotHashable> now be Hashable?"
  - Exercise: "Store snapshots under stable keys, then replace one with changed content. Check membership and the retained value after insertion versus replacement."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B8 — Equatable, Hashable, and Collection Correctness.md>).
  - Code Probe (compile-only; deliberately violates the stored-key invariant):
    ```swift
    final class User: Hashable {
        let id: Int
        var name: String

        init(id: Int, name: String) {
            self.id = id
            self.name = name
        }

        static func == (lhs: User, rhs: User) -> Bool {
            lhs.id == rhs.id && lhs.name == rhs.name
        }

        func hash(into hasher: inout Hasher) {
            hasher.combine(id)
            hasher.combine(name)
        }
    }

    let a = User(id: 1, name: "A")
    let b = User(id: 1, name: "B")
    var users: Set<User> = [a, b]
    a.name = "B" // Violates the stored-key invariant here.

    let values = users.map { "\($0.id)-\($0.name)" }.sorted()
    print(users.count)
    print(values)
    print(Set(values).count)
    ```
  - Probe outcome: The mutation changes equality/hash-relevant state already observed by the Set. No output after that violation is guaranteed. Repair with stable keys, then verify valid insertion/replacement and permitted mutation behavior; B8's replacement exercise distinguishes insert from update(with:).

- **B9. Codable and schema evolution**
  - Expect: Explain stable coding keys, synthesis limits, missing/null/type distinctions, raw versus associated-value enum representations, migration policy, and compatibility in each writer/reader direction.
  - Caveats: Reading old data and writing a current schema does not prove old-reader compatibility. Preserving an unknown enum string does not preserve unknown JSON members. Stored defaults and decodeIfPresent do not repair arbitrary malformed input.
  - Score anchors: `3` — Reads historical fixtures and rejects malformed data under stated policy. `4` — Plans reader/writer compatibility and safe handling of unknown future data.
  - Ask: "What breaks independently when sortOrder/name becomes sortMode/byName?"
  - Ask: "When should absence default, and when should null, a wrong type, an unknown enum value, or an unsupported schema version fail?"
  - Ask: "Can this reader safely rewrite data from a newer writer without dropping unknown fields?"
  - Exercise: "Decode old and current field names under an explicit precedence rule. Check missing, null, and malformed values separately."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B9 — Codable and Schema Evolution.md>).
  - Code Probe:
    ```swift
    import Foundation

    struct Settings: Codable {
        var sortMode: SortMode
    }

    enum SortMode: String, Codable {
        case byName, byDate
    }

    func decodeProbe(_ json: String) -> String {
        do {
            _ = try JSONDecoder().decode(Settings.self, from: Data(json.utf8))
            return "decoded"
        } catch DecodingError.keyNotFound(let key, _) {
            return "missing: \(key.stringValue)"
        } catch DecodingError.dataCorrupted(let context) {
            return "invalid value: \(context.codingPath.map(\.stringValue).joined(separator: "."))"
        } catch {
            return "other error"
        }
    }
    ```
  - Probe outcome: The legacy sortOrder/name payload reports missing: sortMode. Fixing only the key reports invalid value: sortMode. Current sortMode/byName decodes. Semantic assertions avoid full diagnostic-string coupling.
  - Follow-up: Associated-value enum synthesis is not a string catch-all; lossy lists need an explicit discard policy, progress through invalid elements, and observable rejected indices.

- **B10. API design guidelines and Swift-native surface design**
  - Expect: Review call-site clarity, semantic labels, domain types and validation, real mutation behavior, public construction, result shape, ownership, and isolation as one API contract.
  - Caveats: Labels do not create type separation; public properties do not supply a public memberwise initializer. Chaining can be meaningfully ordered. async throws alone guarantees neither offloading nor cancellation of underlying effects. ~Sendable is separate from ~Copyable.
  - Score anchors: `3` — Produces clear calls with usable construction and honest effects. `4` — Weighs caller ergonomics, ownership, isolation, and migration costs.
  - Ask: "Which misuse does a label reveal, which does a distinct ID type reject, and which still needs validation?"
  - Ask: "What does consuming mean for a noncopyable draft compared with a copyable value?"
  - Ask: "How would a separate client expose missing public constructors or accidental isolation requirements?"
  - Exercise: "Rewrite one ambiguous operation with typed inputs and explicit outcomes. Compile an ordinary client and check an invalid-input boundary."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B10 — API Design Guidelines and Swift-Native Surface Design.md>).
  - Code Probe:
    ```swift
    struct NameList: Equatable {
        private(set) var names: [String]

        init(_ names: [String]) { self.names = names }

        mutating func sort() { names.sort() }

        func sorted() -> NameList {
            var result = self
            result.sort()
            return result
        }
    }

    func mutationProbe() -> [Bool] {
        var original = NameList(["Zoe", "Ada"])
        let ordered = original.sorted()
        let preserved = original.names == ["Zoe", "Ada"]
        let returnedSorted = ordered.names == ["Ada", "Zoe"]
        original.sort()
        return [preserved, returnedSorted, original == ordered]
    }
    ```
  - Probe outcome: mutationProbe returns [true, true, true]. sorted leaves the original unchanged; sort changes it. A let binding permits sorted and rejects sort. The wrapper teaches behavior; use Array directly when no extra domain contract is needed.
  - Follow-up: Explain when 6.4 sendability suppression or lifetime-bound Ref/MutableRef belongs in an API, with OS availability and ownership limits. Keep ordinary borrowing/inout when sufficient.

- **B11. Library evolution and resilience-aware API design**
  - Expect: Separate source, ABI, module-interface, behavioral, and persisted-schema contracts. Explain frozen layout, enum boundaries, client-visible bodies, callable symbols, internal specialization, and release configuration.
  - Caveats: nonexhaustive does not enable binary resilience. Old clients may retain old bodies. public inline(always) normally implies body-visibility restrictions; export and optimization controls are distinct. Use checked behavior for the current compiler's export-helper and warning-promotion limits.
  - Score anchors: `3` — Separates source checking from the chosen resilience model. `4` — Plans a compatible release with evidence from unchanged and rebuilt clients.
  - Ask: "Which changes require recompilation, and which must an unchanged client binary tolerate?"
  - Ask: "How do ordinary, resilient, frozen, nonexhaustive, and nonexhaustive(warn) enums differ across a real module/package boundary?"
  - Ask: "Why can replacing a library update an interface-backed function while an old client retains other function bodies?"
  - Ask: "Does specialized expose a new specialized ABI, and is inline(always) just another spelling of inline(__always)?"
  - Exercise: "Add a case to a public enum and rebuild a separate consumer. Check fallback requirements and explain source versus binary compatibility."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B11 — Library Evolution and Resilience-Aware API Design.md>).
  - Code Probe (library module DeliveryAPI):
    ```swift
    @nonexhaustive
    public enum DeliveryStatus: Sendable {
        case pending, delivered
    }

    public func latestStatus() -> DeliveryStatus { .pending }
    ```
  - Code Probe (separate consumer):
    ```swift
    import DeliveryAPI

    func statusLabel(_ status: DeliveryStatus) -> String {
        switch status {
        case .pending: "pending"
        case .delivered: "delivered"
        @unknown default: "unknown"
        }
    }
    ```
  - Probe outcome: An external switch without fallback is rejected. Known cases produce their labels; a new case reaches unknown in the unchanged resilient client. Recompiling against the expanded enum warns about the newly known omitted case. Same-package source checking is a distinct boundary.

- **B12. Variadic generics and parameter packs**
  - Expect: Explain typed arity, each/repeat, pack iteration, same-shape constraints, and zero/single/many-element behavior. Compare mechanical overload families with packs and runtime-length collections.
  - Caveats: A pack is not a runtime array, work scheduler, or unrolling guarantee. Variadic nominal types have a runtime boundary; aliases and functions need separate availability checks. Each unconstrained element still requires Copyable in these examples.
  - Score anchors: `3` — Preserves typed arity and corresponding-element relationships. `4` — Chooses packs over overloads or collections only when the contract benefits.
  - Ask: "Why does one tuple expansion collapse to a scalar, while passing a concrete tuple as one argument does not unpack it?"
  - Ask: "What does same shape mean, and how does it differ from runtime zip truncation?"
  - Ask: "Which evaluations does an early return skip, and which call arguments have already been evaluated?"
  - Ask: "When would a collection or a few meaningful overloads be clearer than a pack API?"
  - Exercise: "Implement a pack-based equality helper. Check zero, one, and heterogeneous inputs, plus an incompatible shape or element type."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B12 — Variadic generics and parameter packs.md>).
  - Code Probe:
    ```swift
    func allEqual<each Element: Equatable>(
        _ lhs: (repeat each Element),
        _ rhs: (repeat each Element)
    ) -> Bool {
        for (left, right) in repeat (each lhs, each rhs) {
            guard left == right else { return false }
        }
        return true
    }
    ```
  - Probe outcome: Empty inputs compare equal; single inputs use scalar equality. Heterogeneous tuples compare position by position and stop after the first mismatch. Different pack shapes or incompatible corresponding types are rejected. This helper does not introduce a tuple protocol conformance.

- **B13. Property wrappers and generated storage semantics**
  - Expect: Know wrapped/projected values, initialization, access, composition, and the checked copyable borrow/mutate wrapper. Distinguish generated storage from isolation and synchronization policy.
  - Caveats: Private initialized wrapper omission predates Swift 6.4; SE-0502 broadens ordinary-property rules. Swift 6 mode removes outward actor-isolation inference from wrappers. Separate locked accessors do not make a compound update atomic.
  - Score anchors: `3` — Explains wrapped storage, construction, and mutation behavior. `4` — Audits wrapper effects on compound operations, access, and isolation.
  - Ask: "What code does a property wrapper effectively synthesize around a property?"
  - Ask: "How can a wrapper accidentally hide thread-safety or actor-isolation problems?"
  - Ask: "Why do private initialized wrapped properties and required private properties produce different initializer access?"
  - Ask: "Why must a synchronized increment protect the whole operation, and when must the wrapper owner declare MainActor explicitly?"
  - Exercise: "Implement a clamping wrapper. Check initialization, both boundaries, and later mutation; explain its generated storage."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B13 — Property Wrappers and Generated Storage Semantics.md>).
  - Code Probe:
    ```swift
    @propertyWrapper
    struct Clamped {
        private var value: Int

        init(wrappedValue: Int) {
            value = min(max(wrappedValue, 0), 100)
        }

        var wrappedValue: Int {
            get { value }
            set { value = min(max(newValue, 0), 100) }
        }
    }

    struct Score {
        @Clamped var value = 150
    }

    print(Score().value)
    ```
  - Probe Ask: "What prints, and why?"
  - Probe Ask: "What storage does Swift effectively synthesize for `@Clamped var value`, and how would custom memberwise initialization interact with the wrapper?"

- **B14. Result builders and DSL construction**
  - Expect: Explain expression/component/final-result types, supported control flow, evaluation, and the tradeoff against plain data construction. Distinguish partial-block and pack-based combination from array-expression trailing-closure parsing.
  - Caveats: Builder syntax does not establish purity, deferred execution, SwiftUI identity, actor isolation, or sendability. buildArray handles loop results; a helper returning several values still needs an accepted expression conversion. Swift 6.4 does not supply a universal ArrayBuilder initializer.
  - Score anchors: `3` — Explains the builder hooks used by the selected control flow. `4` — Justifies the DSL's evaluation and source-compatibility costs.
  - Ask: "Which hooks transform a list expression, an optional branch, a for-in loop, and the final result?"
  - Ask: "Why does an explicit return of the required array compile, and when does the compiler warn that the builder is disabled?"
  - Ask: "What must exist for [String] { ... } to type-check, and what changed from the older .init spelling?"
  - Ask: "When are a plain array or model constructors clearer, and which builder changes could break a client's source?"
  - Exercise: "Build a small list DSL supporting expressions and an optional branch. Check empty input and output order, then compare plain array construction."
  - Extension: [Full chapter exercise and deeper checks](<./B. Type System, Abstraction, and API Design/B14 — Result Builders and DSL Construction.md>).
  - Code Probe:
    ```swift
    @resultBuilder
    enum SumBuilder {
        static func buildExpression(_ value: Int) -> Int { value }
        static func buildBlock() -> Int { 0 }
        static func buildBlock(_ values: Int...) -> Int { -1 }
        static func buildPartialBlock(first: Int) -> Int { first }
        static func buildPartialBlock(accumulated: Int, next: Int) -> Int {
            accumulated + next
        }
        static func buildFinalResult(_ value: Int) -> String { "total=\(value)" }
    }

    func total(@SumBuilder _ content: () -> String) -> String {
        content()
    }
    ```
  - Probe outcome: total {} returns total=0; total { 1; 2; 3 } returns total=6. The available partial-block pair wins over the nonempty buildBlock overload; the empty block still needs buildBlock(). Expression components are evaluated before this fixture's accumulation calls, and buildFinalResult changes Int to String.

### C. Memory, Ownership, Performance, and Safety

- **C1. ARC fundamentals**
  - Expect: Explain ownership graphs, weak nil observations, checked unowned access lifetimes, external registrations, and explicit resource teardown. Distinguish lifetime behavior from retain counts and physical memory measurements.
  - Caveats: Lexical scope is not a universal exact-destruction rule; isolated deinit may need executor scheduling. Weak capture does not cancel work or invalidate a run-loop timer. UniqueBox does not make a class payload uniquely referenced.
  - Score anchors: `3` — Identifies the retaining edges and verifies a valid teardown path. `4` — Defines explicit resource ownership without relying on exact destruction timing.
  - Ask: "What proves an escaped child cannot outlive its unowned referent?"
  - Ask: "How can weak let observe nil, and why is ImmutableWeakCaptures a separate setting?"
  - Ask: "Why is a weak reference alone insufficient evidence that actor-isolated cleanup completed?"
  - Exercise: "Trace one strong ownership cycle, remove the callback, and verify release. State the behavior expected after the owner disappears."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C1 — ARC fundamentals.md>).
  - Code Probe (with ARCLog and Observed from C1) — [setup and shared declarations](<./C. Memory, Ownership, Performance and Safety/C1 — ARC fundamentals.md>):
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
    ```
  - Probe outcome: makeCycle returns two live weak observations and an empty destruction log. The callback increments completions. Calling stop through the owner observation breaks the cycle; both destruction events occur once in the checked fixture, without a prescribed order. A weak callback changes the post-owner lifetime contract and must be chosen deliberately.

- **C2. Copy-on-write behavior of standard-library types**
  - Expect: Distinguish logical copies, storage sharing, capacity/representation costs, reference elements, and slice retention. Review private CoW mutation coverage; compare Array snapshots with the different ownership contract of UniqueArray.
  - Caveats: Uniqueness checks are not locks or Array buffer-inspection APIs. Unique storage can still grow/allocate. Functional updates and consuming parameters do not imply a fixed physical-copy count; test behavior and measure cost separately.
  - Score anchors: `3` — Separates logical independence, reference payloads, and slice retention. `4` — Chooses a retention strategy from ownership evidence and measured cost.
  - Ask: "Why can an element write or append require O(n) work even though array passing can be cheap?"
  - Ask: "How can a tiny ArraySlice retain invisible elements, and what does Array(slice) still retain?"
  - Ask: "Can a unique outer storage object contain a shared Array buffer, and which mutation paths must preserve independence?"
  - Exercise: "Mutate an ArraySlice and compare the source value. Explain its retained base storage and check nonzero indices and an owned slice copy."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C2 — Copy-on-write behavior of standard-library types.md>).
  - Code Probe (call sliceProbe):
    ```swift
    func sliceProbe() {
        let source = Array(10...20)
        var slice = source[3...5]
        print(slice.startIndex, slice.endIndex, slice.count,
              slice[slice.startIndex])
        slice[slice.startIndex] = -1
        print(source[3], slice[slice.startIndex], Array(slice))
    }
    ```
  - Probe outcome: Prints 3 6 3 13, then 13 -1 [-1, 14, 15]. The slice preserves nonzero indices and its mutation leaves the original array value unchanged. The chapter's separate weak-reference fixture investigates retained objects; this output alone measures neither allocation count nor memory bytes.

- **C3. Exclusivity enforcement and `inout`**
  - Expect: Explain argument evaluation before inout access, and how Swift 6.4 borrow/mutate accessors and MutableRef preserve exclusivity. Understand exclusive access rules, dynamic exclusivity traps, and the borrow-like nature of `inout`.
  - Caveats: `inout` is not a stable alias; overlapping access bugs can compile and still trap at runtime.
  - Score anchors: `3` — Identifies overlapping access and preserves exclusive mutation. `4` — Designs a mutation boundary whose aliasing and lifetime rules remain usable.
  - Ask (extension, Swift 6.4): "Why is a by-value array snapshot accepted alongside one inout element while two inout elements conflict?"
  - Ask: "Why is `inout` not the same as C++ references or pointers?"
  - Ask: "What kind of overlapping accesses does Swift try to forbid?"
  - Exercise: "Explain why passing one variable to two inout parameters fails. Redesign the call with independent values and check the result."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C3 — Exclusivity enforcement and `inout`.md>).
  - Code Probe:
    ```swift
    func swapValues(_ a: inout Int, _ b: inout Int) {
        let temp = a
        a = b
        b = temp
    }

    var x = 1
    swapValues(&x, &x)
    ```
  - Probe Ask: "Why is this invalid in Swift?"
  - Probe Ask: "What exclusivity rule is being protected, and what API redesigns avoid the overlapping-access problem?"

- **C4. String and Unicode correctness**
  - Expect: Specify Character/scalar/UTF-8/UTF-16/layout units, distinguish canonical equality from byte identity, and validate ranges against the correct snapshot and boundary policy.
  - Caveats: String.Index can describe positions inside a grapheme; NSRange conversion alone is not a selection validator. Character counts do not measure width. UTF8Span borrows valid UTF-8 and has deployment/lifetime constraints.
  - Score anchors: `3` — Uses the correct text view and valid indices for the task. `4` — Preserves encoding, selection, and normalization contracts across APIs.
  - Ask: "Why can equal String keys have different encoded bytes?"
  - Ask: "What else must be checked after converting a UTF-16 range, and why avoid location + length before validation?"
  - Exercise: "Count and truncate text by a declared unit. Check combining marks, a flag, empty input, and an exact boundary."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C4 — String and Unicode correctness.md>).
  - Code Probe (call unicodeProbe):
    ```swift
    func unicodeCounts(_ text: String) -> (Int, Int, Int, Int) {
        (text.count, text.unicodeScalars.count, text.utf16.count, text.utf8.count)
    }

    func unicodeProbe() {
        let text = "\u{1F1FA}\u{1F1F8}e\u{301}"
        let (characters, scalars, utf16, utf8) = unicodeCounts(text)
        print(characters, scalars, utf16, utf8)
    }
    ```
  - Probe outcome: 2 4 6 11 counts Characters, scalars, UTF-16 code units, and UTF-8 bytes respectively. The flag and accented e are two Characters; neither view count establishes rendered width. Follow up with the chapter's range conversion, validation, and truncation fixtures.

- **C5. Collection protocols, complexity, and index invalidation**
  - Expect: Distinguish Sequence/Collection traversal and complexity contracts from Swift 6.4 Iterable borrowing and typed iteration failure.
  - Caveats: Generic indices need not be Int or start at zero. Structural mutation can invalidate them. Binary search requires sorted input and constant-time index operations for logarithmic index work; comparison cost remains separate.
  - Score anchors: `3` — Preserves traversal, index validity, and the algorithm's complexity assumptions. `4` — Selects a generic contract and verifies costs without assuming integer indices.
  - Ask: "Which protocol preserves this algorithm's semantics and complexity?"
  - Ask: "Why can InlineArray support for-in without Sequence conformance, while Array does not directly satisfy Iterable in this SDK?"
  - Ask: "Which owner, element, and lifetime defaults must a generic borrowing visitor suppress?"
  - Exercise: "Search a sorted ArraySlice while preserving original indices. Check empty, missing, and boundary matches; explain the required collection constraint."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C5 — Collection Protocols, Complexity, and Index Invalidation.md>).

- **C6. Unsafe pointers, buffers, and raw memory**
  - Expect: Separate allocation, binding, initialization, alignment, and access lifetime. Prefer Swift 6.4 temporary/output spans and checked raw operations where appropriate. Know pointer lifetime, initialization, binding, rebinding, alignment, and when to prefer safe wrappers over unsafe memory APIs.
  - Caveats: Most pointer bugs are lifetime or aliasing bugs, not syntax bugs.
  - Score anchors: `3` — Keeps pointer access inside its valid initialized lifetime. `4` — Designs a safe native boundary with explicit retention and cleanup policy.
  - Ask (extension, Swift 6.4): "Why can a closure wrapper still leak an unsafe pointer, and what does a span improve?"
  - Ask: "What is the difference between raw memory and typed memory in Swift?"
  - Ask: "When is `withUnsafeBytes` safer than storing an unsafe pointer for later?"
  - Exercise: "Replace an escaped buffer pointer with scoped processing or owned storage. Check empty input and explain initialization and lifetime obligations."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C6 — Unsafe pointers, buffers, and raw memory.md>).
  - Code Probe:
    ```swift
    // Compile-only review example: using the escaped pointer is invalid.
    let bytes: [UInt8] = [1, 2, 3]
    var saved: UnsafePointer<UInt8>?

    bytes.withUnsafeBufferPointer { buffer in
        saved = buffer.baseAddress
    }

    print(saved![0])
    ```
  - Probe Ask: "Why is this invalid even if it appears to work?"
  - Probe Ask: "What lifetime guarantee does `withUnsafeBufferPointer` actually provide, and how would you redesign this safely?"

- **C7. Ownership modifiers: borrowing, consuming, and consume**
  - Expect: Explain borrowing, consuming, explicit consume, reinitialization, and Swift 6.4 borrow/mutate accessors and Ref/MutableRef lifetimes.
  - Caveats: A consuming parameter can receive an implicit copy of a Copyable argument. A reference to a copyable getter result may observe a temporary; mutable access must remain exclusive. Ownership does not establish thread affinity.
  - Score anchors: `3` — Predicts ownership transfer and subsequent-use rules. `4` — Chooses borrowed or consuming contracts with explicit caller lifetime costs.
  - Ask: "When does consuming end the caller's current value, and when can a mutable binding be reused?"
  - Ask: "How do borrow/mutate accessors differ from returning an owned value?"
  - Ask: "What does Ref(values[0]) refer to when the subscript returns a copy?"
  - Exercise: "Borrow and consume a noncopyable token. Show accepted borrowing, rejected use after consume, and valid reinitialization."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`.md>).

- **C8. Copyable, noncopyable types, and ~Copyable**
  - Expect: Explain single ownership, suppressed generic and associated-type defaults, UniqueBox/UniqueArray, and the actual noncopyable protocol support in the selected SDK.
  - Caveats: Copying a class reference does not clone its object. Noncopyable Hashable conformance does not make Set accept the value. Unique owners require Apple OS 27 here; proposed Optional reference accessors and public RigidArray are absent.
  - Score anchors: `3` — Suppresses copying where required and explains remaining reference aliases. `4` — Defines adoption, release, and container requirements for an ownership API.
  - Ask: "Where must Copyable defaults be suppressed in a generic Holder with an associated Item?"
  - Ask: "How does UniqueBox.clone behave with an Array payload versus a class payload?"
  - Ask: "Why can a noncopyable value conform to Hashable yet still fail as a Set element?"
  - Exercise: "Model an exclusively owned resource token. Check transfer and cleanup, then explain what remains shared if its payload is a class."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C8 — Copyable, noncopyable types, and `~Copyable`.md>).

- **C9. Existential, generic, and allocation performance model**
  - Expect: Distinguish preserved type relationships from actual specialization, dispatch, representation, and allocation. Compare concrete/generic/opaque/existential/enum/closure-erased designs with the same work and behavior.
  - Caveats: Unspecialized generic code can use witnesses, a known existential can devirtualize, and a concrete eraser can still call a stored closure. MemoryLayout does not count allocations. Loop fusion can reorder observable effects even when final bytes match.
  - Score anchors: `3` — Separates type relationships from dispatch and allocation assumptions. `4` — Uses equivalent workloads and optimizer evidence to justify an abstraction choice.
  - Ask: "What does the optimizer know at this call and across the module boundary?"
  - Ask: "Why can [any P] retain representation/dispatch costs, and why does [T] alone not prove their removal?"
  - Exercise: "Compare generic and existential pipelines performing identical work. Check equal results and identify what must be measured before claiming a speedup."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C9 — Existential, generic, and allocation performance model.md>).
  - Code Probe (use the chapter's transform and pipeline declarations) — [setup and shared declarations](<./C. Memory, Ownership, Performance and Safety/C9 — Existential, generic, and allocation performance model.md>):
    ```swift
    func pipelineProbe() -> [[UInt8]] {
        let input: [UInt8] = [0, 127, 255]
        var erased = input
        var opened = input
        var closed = input
        let transforms: [any PixelTransform] = [Invert(), Clamp(maximum: 200)]
        existentialPipeline(transforms, to: &erased)
        openedPipeline(transforms, to: &opened)
        genericPipeline([PixelRule.invert, .clamp(maximum: 200)], to: &closed)
        return [input, erased, opened, closed]
    }
    ```
  - Probe outcome: The original is [0, 127, 255]; the erased, opened, and closed variants each produce [200, 128, 0]. This is semantic equivalence and source-value independence, not a speed or allocation result. The separate orderingProbe demonstrates why changing pass order needs a purity/effect contract.

- **C10. Compiler optimization behavior and debug vs release differences**
  - Expect: Record effective optimization, assertion, source-condition, language, isolation, and target settings. Distinguish compiler transformations from runtime measurements and keep required work outside assertion autoclosures.
  - Caveats: Onone does not define DEBUG; assertion configuration can be overridden. Ordinary O/Osize retains preconditions, while unchecked configuration does not. Mandatory inlining, client body visibility, and explicit specialization are different controls; a scoped callback alone cannot prevent a pointer result escaping.
  - Score anchors: `3` — Explains assertion evaluation and repairs a configuration-dependent defect. `4` — Isolates a release-only failure and validates the semantic repair.
  - Ask: "Why can debug timings misrepresent a shipped workload without fixing algorithmic complexity?"
  - Ask: "When does assert evaluate, how can that be overridden, and how should recoverable preparation failure be modeled?"
  - Ask: "What do the inline(always) diagnostics and mandatory-pass SIL establish, and which runtime/ABI claims need different evidence?"
  - Exercise: "Move required work out of an assert autoclosure. Compare debug and optimized behavior while holding input and relevant settings fixed."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C10 — Compiler Optimization Behavior and Debug vs Release Differences.md>).
  - Code Probe:
    ```swift
    func assertionProbe() -> Int {
        var checks = 0
        func isValid(_ value: Int) -> Bool {
            checks += 1
            return value >= 0
        }
        for value in 0..<3 { assert(isValid(value)) }
        return checks
    }
    ```
  - Probe outcome: Default Onone returns 3; O/Osize/Ounchecked returns 0. Onone plus assert-config Release returns 0; O plus assert-config Debug returns 3. These deliberately visible side effects illustrate why required preparation belongs in ordinary control flow.

- **C11. Strict memory safety mode and safe systems programming**
  - Expect: Choose warning/error policy separately from correctness. Use checked temporary-span/raw-load APIs where possible. Know what Swift 6.4's strict memory safety checking is trying to expose and why explicit unsafe acknowledgments matter.
  - Caveats: The absence of warnings is not proof that unsafe code is correct; the presence of unsafe APIs should be intentional and auditable.
  - Score anchors: `3` — Explains what the safety checker can and cannot establish. `4` — Keeps an auditable unsafe surface with justified assumptions and verification.
  - Ask (extension, Swift 6.4): "What does @safe assert, and why can a scoped callback still escape a pointer?"
  - Ask: "What kinds of constructs does strict memory safety mode want you to acknowledge?"
  - Ask: "Why is making unsafe code explicit valuable even if the code is known-correct?"
  - Exercise: "Audit one native wrapper's pointer access. Identify initialization, bounds, and cleanup obligations and explain where unsafe acknowledgment belongs."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C11 — Strict Memory Safety Mode and Safe Systems Programming.md>).

- **C12. Span, InlineArray, and modern low-level abstractions**
  - Expect: Choose owned inline/heap storage versus borrowed views. Explain borrowing iteration, initialized temporary storage, safe raw loads, and per-operation availability.
  - Caveats: Iterable is not Sequence/Collection. Bounds-checked loads can trap, and Swift memory layout is not a wire format. A fixed-size inline value need not live on the stack.
  - Score anchors: `3` — Separates borrowed lifetime, bounds, and representation from wire format. `4` — Chooses owned storage and borrowed views under deployment and allocation constraints.
  - Ask: "How do Array, InlineArray, UniqueArray, Span, and Ref differ in ownership and lifetime?"
  - Ask: "What must a parser validate before a safe raw load?"
  - Exercise: "Read one fixed-width field from borrowed bytes with explicit byte order. Check the last valid offset, negative input, and an oversized offset."
  - Extension: [Full chapter exercise and deeper checks](<./C. Memory, Ownership, Performance and Safety/C12 — `Span`, `InlineArray`, and modern low-level abstractions.md>).

### D. Concurrency, Isolation, and Correctness

- **D1. Structured concurrency fundamentals**
  - Expect: Explain `async/await`, task trees, concurrency versus parallelism, and suspension versus thread blocking. Include implicitly awaited scope-exit work, ownership of unstructured task errors, and the cooperative executor's need for forward progress.
  - Caveats: Await marks potential suspension; it does not promise a background thread. Blocking a cooperative worker while waiting for future task work can prevent progress. SwiftUI manages .task lifetime, while async let/task groups provide language-level child lifetimes. An async wrapper does not make blocking legacy work nonblocking.
  - Score anchors: `3` — Distinguishes suspension, blocking, and structured child lifetime. `4` — Chooses a task structure that preserves ownership and forward progress.
  - Ask (extension, Swift 6.4): "Can an early return suspend while deferred async cleanup completes?"
  - Ask: "What does `await` actually mean semantically?"
  - Ask: "Why can a semaphore wait inside a task deadlock when its signal depends on work queued to the same cooperative pool?"
  - Ask: "How would you adapt an asynchronous DispatchQueue callback without waiting synchronously, and where should a genuinely blocking legacy operation run?"
  - Ask: "Why is structured concurrency safer than ad hoc callback trees or detached work?"
  - Exercise: "Trace two child operations and the parent's scope exit. Identify potential suspension and explain why an unfinished child must be joined."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D1 — Structured Concurrency Fundamentals.md>).

- **D2. Task hierarchy, cancellation, and priorities**
  - Expect: Explain cooperative cancellation, task-local propagation, parent-child relationships, and priority inheritance. Choose a clock and define timeout cancellation/completion policy. At deeper scope, explain cancellation shielding and its runtime availability.
  - Caveats: A cancellation request does not guarantee prompt completion. Awaiting a shared task is not automatically interrupted by consumer cancellation. A shield masks current-context cancellation and can block automatic propagation to enclosed children; it does not uncancel the task. Regular group failure handling depends on where the error is observed.
  - Score anchors: `3` — Defines cooperative cancellation and who owns unfinished work. `4` — Combines task ownership, cancellation, and elapsed-time policy under failure.
  - Ask (extension, Swift 6.4): "Does a shield uncancel the task, or change how the current context observes cancellation?"
  - Ask: "Why does repeated cell configuration need a unique request identity, and why must stale error handling check cancellation too?"
  - Ask: "Why is cancellation in Swift cooperative rather than preemptive?"
  - Ask: "What does a child task inherit from its parent?"
  - Ask: "When does a policy need elapsed time instead of a wall-clock date? How do ContinuousClock and SuspendingClock differ during system sleep?"
  - Ask: "If a timeout wins a structured race, who cancels and joins the losing work? Why can the function still return after its time budget?"
  - Exercise: "Cancel a gated operation before and during execution. Check its cancellation observation, completion path, and cleanup without elapsed-time sleeps."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D2 — Task Hierarchy, Cancellation, and Priorities.md>).

- **D3. `async let`, task groups, and choosing the right concurrency primitive**
  - Expect: Choose sequential await, fixed async-let children, regular task groups, or discarding groups based on dependencies, result ordering, failure policy, and bounded resource use.
  - Caveats: Unobserved regular-group errors are not automatically propagated; waitForAll drains before throwing. Throwing discarding groups cancel siblings on child failure. All structured scopes still wait for cooperative completion, including under cancellation shields.
  - Score anchors: `3` — Chooses a primitive and explains result and error observation. `4` — Bounds work and preserves ordering, cancellation, and failure semantics.
  - Ask: "When is `async let` better than a task group, and how can the order of awaiting its values delay error observation?"
  - Ask: "Compare catching `next()`, using `nextResult()`, calling `waitForAll()`, ignoring child results, and a throwing discarding group."
  - Ask: "Why can a first-result or timeout race remain suspended after choosing its winner?"
  - Exercise: "Run three independent requests with async let or a group. Preserve result association and check the selected all-or-nothing or partial-result policy."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D3 — `async let`, Task Groups, and Choosing the Right Concurrency Primitive.md>).

- **D4. Actors and actor isolation**
  - Expect: Explain actor ownership, potential suspension, data-race prevention, and logic races. Model shared producers with explicit identity and invalidation policy.
  - Caveats: Actors do not provide whole-workflow atomicity or cross-task FIFO order. Cancelling a waiter does not automatically cancel shared work. A Sendable type can still acquire isolated computed members under target defaults.
  - Score anchors: `3` — Identifies isolated state and serializes its synchronous mutation. `4` — Defines shared-operation ownership, invalidation, and caller cancellation policy.
  - Ask: "Which state is isolated, and which invariants can become stale at an await or async defer?"
  - Ask: "Can an older refresh repopulate the cache after invalidation, or clear a newer in-flight operation on failure?"
  - Exercise: "Put a shared counter behind an actor. Check concurrent callers and identify which invariant is protected without suspension."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D4 — Actors and Actor Isolation.md>).

- **D5. Global actors and `@MainActor`**
  - Expect: Choose whole-type versus member isolation, distinguish instance actors from global actors, and separate UI ownership from expensive CPU execution. Explain explicit global-actor-isolated conformances and the callers they permit.
  - Caveats: MainActor does not prevent stale results. Synchronous nonisolated helpers do not offload work; Sendable alone does not specify async execution or permission to use an isolated conformance. This rubric uses explicit MainActor ownership where required, while retaining nonisolated target defaults.
  - Score anchors: `3` — Distinguishes type, member, and conformance isolation. `4` — Designs a usable UI protocol boundary without unsafe isolation assumptions.
  - Ask: "When should a whole type be MainActor, and when is a member annotation enough?"
  - Ask: "What does MainActor isolation on a conformance permit that a nonisolated witness cannot do? Compare an isolated consumer with a generic consumer that crosses isolation."
  - Ask: "Why does awaiting nonblocking I/O differ from sorting synchronously on MainActor, and what does @concurrent guarantee?"
  - Ask: "Why must a cancelled search guard ordinary errors and cleanup as well as successful results?"
  - Exercise: "Give a MainActor model an explicit MainActor protocol conformance. Call the same generic helper from MainActor and from a nonisolated caller; explain which use is permitted."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D5 — Global Actors and `@MainActor`.md>).

- **D6. `Sendable`, `@Sendable`, and `sending`**
  - Expect: Distinguish checked type conformance, closure capture rules, execution isolation, and a particular parameter/result transfer. Include ~Sendable suppression, conditional conformance, public/@frozen inference, and weak-reference migration.
  - Caveats: sending is not noncopyable ownership, and Sendable values can still be reused. A final owner of let Mutex storage can use checked Sendable; unchecked conformance needs a separate whole-type audit. ImmutableWeakCaptures is distinct from support for weak let.
  - Score anchors: `3` — Separates type sendability, closure captures, and a specific safe handoff. `4` — Chooses a documented sharing contract and audits aliases across boundaries.
  - Ask (extension, Swift 6.4): "How do omitted conformance, ~Sendable, conditional Sendable, and @unchecked Sendable differ? What does -Wwarning ExplicitSendable audit?"
  - Ask: "Why does a direct MainActor class with ~Sendable differ from removing an inherited conformance?"
  - Ask: "Why does let or a capture list fail to protect a mutable referenced object?"
  - Ask: "Why is a Task's sending operation closure not the same API as an explicit @Sendable callback?"
  - Ask: "When can a non-Sendable region be transferred, and which later alias uses invalidate it?"
  - Ask: "How do weak let storage and ImmutableWeakCaptures affect lifetime and sendability?"
  - Exercise: "Repair an unsafe shared service with a value, actor, or checked Mutex-backed owner. Explain capture versus transfer and check the chosen boundary."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D6 — `Sendable`, `@Sendable`, and `sending`.md>).
  - Code Probe:
    ```swift
    final class Cache {
        var store: [String: Int] = [:]
    }

    func run(
        _ operation: @Sendable @escaping () async -> Void
    ) -> Task<Void, Never> {
        Task { await operation() }
    }

    func broken() -> Task<Void, Never> {
        let cache = Cache()
        return run {
            cache.store["a"] = 1
        }
    }
    ```
  - Probe Ask: "Why does Swift 6.4 reject capture of the mutable, non-Sendable Cache under this rubric's settings?"
  - Probe Ask: "Choose an actor, value snapshot, or checked Sendable Mutex owner. Where would a manually audited unchecked conformance still be needed?"
  - Probe Ask: "Why can a sending handoff accept a disconnected mutable object while an explicit @Sendable capture of that type still fails?"

- **D7. Actor reentrancy and logic races**
  - Expect: Know that actor methods can interleave at suspension points and that reentrancy is a central design concern. Include reentrancy during awaited deferred cleanup.
  - Caveats: Actor isolation serializes isolated access but does not make an async workflow atomic. Await may not suspend. First-result caching still duplicates work; shared-task caching needs an explicit consumer cancellation and invalidation policy.
  - Score anchors: `3` — Identifies stale assumptions and repairs the invariant across suspension. `4` — Defines freshness, shared work, cancellation, and invalidation as separate policies.
  - Ask (extension, Swift 6.4): "Can old deferred cleanup overwrite newer actor state while an async function is exiting? Why must that cleanup finish before the function returns?"
  - Ask: "What does actor reentrancy mean in practice?"
  - Ask: "How can an `await` in the middle of an actor method invalidate assumptions made earlier in the method?"
  - Exercise: "Gate two actor operations so they complete out of order. Check that an older completion cannot replace the selected current result."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D7 — Actor Reentrancy and Logic Races.md>).
  - Code Probe:
    ```swift
    actor Counter {
        var value = 0

        func incrementIfZero() async {
            if value == 0 {
                try? await Task.sleep(for: .milliseconds(10))
                value += 1
            }
        }
    }
    ```
  - Probe Ask: "If two tasks call incrementIfZero concurrently, can value become 2? Why does this sleep demonstrate a possible interleaving rather than coordinate a deterministic test?"
  - Probe Ask: "What actor reentrancy behavior creates the bug, and how would you fix the logic rather than just moving code around?"

- **D8. Isolation control: `nonisolated`, isolated parameters, and `assumeIsolated`**
  - Expect: Understand when behavior should or should not be isolated, and how to express that safely.
  - Caveats: Synchronous nonisolated does not offload work. Under this baseline, plain nonisolated async and explicit nonisolated(nonsending) both follow the caller's actor; @concurrent selects off-actor execution. sending describes transfer. assumeIsolated checks an existing executor context and traps on a wrong assumption.
  - Score anchors: `3` — Uses explicit isolation without bypassing protected state. `4` — Chooses compiler-checked boundaries and justifies any runtime isolation assertion.
  - Ask: "What does `nonisolated` mean on a member of an actor or globally isolated type?"
  - Ask: "When is `assumeIsolated` appropriate, and what makes it dangerous?"
  - Ask: "Why does a pure synchronous nonisolated helper still occupy its MainActor caller?"
  - Ask: "Why may a function have only one isolated parameter even when a caller passes one actor twice?"
  - Ask (extension): "What does Hashable UnownedTaskExecutor identify, and why does it neither retain the owner nor prove actor isolation? Check OS 27 availability."
  - Exercise: "Review an actor helper that reads stored state. Choose an isolated operation or a nonisolated snapshot-based helper and explain an invalid alternative."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`.md>).

- **D9. Continuations and bridging callback APIs**
  - Expect: Compare noncopyable Continuation with checked/unsafe handles, preserve one-result versus event semantics, and design exactly-once completion with explicit cancellation and lifetime policy.
  - Caveats: Consuming reuse is a compiler error; checked duplicate resume and dropped new handles trap; retained unfinished handles can still hang. The selected SDK also supports explicitly typed checked throwing factories. Continuation requires OS 27 here.
  - Score anchors: `3` — Completes a continuation once and preserves the callback's result contract. `4` — Handles cancellation and registration races with explicit resource ownership.
  - Ask (extension, Swift 6.4): "Which handle supports noncopyable success, and why are typed failures not exclusive to withContinuation in this SDK?"
  - Ask: "Why can't an arbitrary escaping callback simply consume a captured noncopyable continuation?"
  - Ask: "Can onCancel precede operation entry, or precede installation of the request handle?"
  - Ask: "Why does this adapter select its terminal action under the lock, then resume, cancel, and release resources outside it? Which callbacks or destructors could reenter?"
  - Ask: "Can success still win if cancellation is requested afterwards? What policy does the adapter implement?"
  - Exercise: "Adapt a one-result callback using a synchronized terminal state. Check synchronous completion and duplicate delivery; state the cancellation policy."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D9 — Continuations and Bridging Callback APIs.md>).
  - Code Probe:
    ```swift
    func legacy(
        _ completion: @escaping @Sendable (Result<Int, any Error>) -> Void
    ) {
        completion(.success(1))
        completion(.success(2))
    }

    func adapted() async throws -> Int {
        try await withCheckedThrowingContinuation { continuation in
            legacy { result in
                continuation.resume(with: result)
            }
        }
    }
    ```
  - Probe Ask: "Why does this compile and then trap before returning a normal result?"
  - Probe Ask: "When is a first-result-only gate appropriate, and when should this be an AsyncSequence?"
  - Probe Ask: "Why does switching continuation type not itself fix a missing callback or cancel the external request?"

- **D10. `AsyncSequence`, streams, and backpressure-aware design**
  - Expect: Understand when a single async result should become an async sequence, and how streams model event sources.
  - Caveats: A bare break does not guarantee termination of a retained stream. Use an explicit subscription owner/close path, Sendable payloads, and a buffering policy that matches the domain. Channel handoff is not processing acknowledgment; continuous Observation is not an ordered event log.
  - Score anchors: `3` — Defines event completion, buffering, and subscription lifetime. `4` — Chooses a loss and backpressure policy suitable for the source.
  - Ask: "When should an API return `AsyncSequence` instead of a callback or delegate?"
  - Ask: "What cancellation and resource-lifetime issues arise with `AsyncStream`?"
  - Ask: "What happens to onTermination when the consumer breaks but both stream and continuation remain alive?"
  - Ask: "Can a channel send return without successful delivery? Who releases a producer after early consumer exit?"
  - Exercise: "Return three events and explicitly finish a stream. Check normal completion and define who closes the source after early consumer exit."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design.md>).
  - Code Probe:
    ```swift
    func makeStream() -> AsyncStream<Int> {
        AsyncStream { continuation in
            for i in 0..<3 {
                continuation.yield(i)
            }
        }
    }
    ```
  - Probe Ask: "Why might a consumer hang forever waiting for this stream to finish?"
  - Probe Ask: "What does this teach you about stream lifetime, completion, and resource cleanup?"

- **D11. Detached tasks, task inheritance, and isolation loss**
  - Expect: Know the difference between `Task {}` and `Task.detached {}`. Include Swift 6.4 unused throwing-task warnings and explicit error/lifetime ownership.
  - Caveats: Task {} inherits lexical actor context and current task metadata, but neither initializer automatically propagates creator cancellation. Priority can be escalated later and does not prove inheritance or ordering. A handle and cancel() alone do not prevent stale errors or cleanup.
  - Score anchors: `3` — Explains task inheritance and assigns error and lifetime ownership. `4` — Chooses an unstructured boundary with cancellation and publication guarantees.
  - Ask (extension, Swift 6.4): "Does explicitly discarding a throwing Task handle handle its error?"
  - Ask: "Why is `Task.detached` often a footgun?"
  - Ask: "What does `Task {}` inherit that a detached task does not?"
  - Ask: "Does Task {} in a synchronous nonisolated helper inherit the MainActor of its caller? What if its creator is already cancelled?"
  - Exercise: "Replace an unowned throwing task with an awaited handle or explicit task owner. Check error delivery and state how cancellation reaches the operation."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D11 — Detached Tasks, Task Inheritance, and Isolation Loss.md>).
  - Code Probe:
    ```swift
    @MainActor
    final class ViewModel {
        var title = ""

        func load() {
            Task.detached {
                self.title = "Done"
            }
        }
    }
    ```
  - Probe Ask: "Why is `Task.detached` the wrong tool here?"
  - Probe Ask: "What context is lost relative to `Task {}`, and what is the correct way to structure this work?"

- **D12. Synchronization library, mutexes, and choosing between actors and locks**
  - Expect: Choose actor ownership, a synchronous mutex, or immutable publication from the access contract. Protect complete invariants and explain lock ordering, reentrant callbacks, and the forward-progress risks of waiting for future task work.
  - Caveats: Mutex requires macOS 15 / iOS 18 in the checked SDK. Its synchronous closure releases on throw and cannot suspend. Keep critical sections short; avoid calling unknown or reentrant code under the lock. Compiler acceptance alone does not prove a retained non-Sendable reference cannot escape. Immutable snapshots still need synchronized publication.
  - Score anchors: `3` — Protects a whole invariant with an appropriate actor or lock. `4` — Accounts for lock ordering, callbacks, and cooperative-executor forward progress.
  - Ask: "When is an actor a better fit than a mutex, and when is a mutex still appropriate?"
  - Ask: "Why can wrapping everything in a lock be worse than modeling ownership explicitly?"
  - Ask: "Why can a short synchronous critical section be appropriate while a semaphore wait for future task work is unsafe? How do lock ordering and reentrant callbacks create deadlocks?"
  - Ask (extension): "Can withLockIfAvailable fail spuriously? Distinguish outer nil from a successful closure returning nil; preserve the guarantee's Swift 6.0 status."
  - Exercise: "Protect a read-modify-write counter with one synchronous critical section. Explain why separately locked accessors fail and why the closure must not suspend."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks.md>).

- **D13. Swift 6 strict concurrency migration**
  - Expect: Migrate target by target toward Swift 6 with Approachable Concurrency and nonisolated default isolation. Distinguish Swift 5 warning staging from an existing 6.2/6.3 Swift 6 target. Track isolation flags, new task warnings, SDK availability, and justified compatibility annotations.
  - Caveats: Silencing diagnostics without rethinking ownership/isolation is not a migration.
  - Score anchors: `3` — Resolves a diagnostic with a valid ownership or isolation change. `4` — Plans incremental migration with client checks and tracked exceptions.
  - Ask: "What did complete strict concurrency checking provide before Swift 6 language mode?"
  - Ask: "When is `@preconcurrency` acceptable, and why is it rarely the final design?"
  - Exercise: "Repair one Swift 5 complete-checking warning through ownership or isolation. Compile in Swift 6 mode and explain any remaining compatibility annotation."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D13 — Swift 6 Strict Concurrency Migration.md>).

- **D14. Approachable Concurrency and Default Isolation**
  - Expect: Apply Approachable Concurrency with nonisolated default isolation. Explain caller-isolated nonisolated async execution, explicit @concurrent work, and conformance inference with InferIsolatedConformances enabled. Distinguish the feature bundle from target default isolation.
  - Caveats: An ordinary declaration does not become MainActor-isolated merely because Approachable Concurrency is enabled. Explicit or inferred actor isolation can still apply through other rules. A module's execution and conformance contracts remain relevant at its client boundary.
  - Score anchors: `3` — Separates target defaults from an explicit execution contract. `4` — Plans consistent isolation and conformance policies across module boundaries.
  - Ask: "Why does enabling Approachable Concurrency leave an ordinary type nonisolated in this rubric? Where would explicit MainActor isolation be appropriate?"
  - Ask: "A MainActor caller awaits a plain nonisolated async helper. Which actor does the helper use, and what changes with @concurrent or a nonisolated caller?"
  - Ask: "When should explicit nonisolated(nonsending) or @concurrent replace reliance on target defaults?"
  - Ask: "With InferIsolatedConformances enabled, how can a MainActor type infer an isolated conformance? When can its witnesses instead support a nonisolated conformance?"
  - Ask (extension): "How would disabling caller-isolated defaults or changing default actor isolation affect a separately compiled module? Explain the difference from the rubric baseline."
  - Exercise: "Call plain nonisolated async, explicit nonisolated(nonsending), and @concurrent helpers from MainActor and a nonisolated context. Predict actor isolation without assuming a particular thread."
  - Extension: [Full chapter exercise and deeper checks](<./D. Concurrency, Isolation, and Correctness/D14 — Approachable Concurrency and Default Isolation.md>).

### E. Interop, Modules, and Distribution

- **E1. Objective-C interoperability**
  - Expect: Distinguish Objective-C exposure, @objc dynamic dispatch, and native Swift dynamic. Design compatible selectors/types/errors, inspect the generated header, and compile an Objective-C client.
  - Caveats: NSObject does not expose every member or synchronize state. The checked header omits MainActor/Sendable callback guarantees; Objective-C callers must obey the documented context. Cancellation and callback lifetime are explicit policies.
  - Score anchors: `3` — Checks the actual exported surface and native invocation. `4` — Preserves callback, error, isolation, and ownership contracts for native callers.
  - Ask: "Why can a Swift struct have a dynamic method while KVO uses @objc dynamic?"
  - Ask: "Which Swift types/signatures disappear or change in the exported header, and what proves the native client can call it?"
  - Ask: "When does this profile facade complete, where does it deliver, and what happens when the token is cancelled or dropped?"
  - Exercise: "Expose one Swift operation to an Objective-C client. Inspect its selector and types, invoke it, and document the required calling context."
  - Extension: [Full chapter exercise and deeper checks](<./E. Interop, Modules, and Distribution/E1 — Objective-C Interoperability.md>).
  - Code Probe:
    ```swift
    import Foundation

    final class SelectorSink: NSObject {
        private(set) var deliveries = 0
        @objc private func receive(_ notification: Notification) { deliveries += 1 }
        var action: Selector { #selector(receive(_:)) }
    }

    func selectorProbe() -> [String] {
        let center = NotificationCenter()
        let sink = SelectorSink()
        let name = Notification.Name("E1.probe")
        center.addObserver(sink, selector: sink.action, name: name, object: nil)
        center.post(name: name, object: nil)
        center.removeObserver(sink)
        center.post(name: name, object: nil)
        return [NSStringFromSelector(sink.action), String(sink.responds(to: sink.action)), String(sink.deliveries)]
    }
    ```
  - Probe outcome: ["receive:", "true", "1"]. A private Swift method can remain Objective-C-visible; removal prevents the second delivery. The mutable observer fixture posts on one context and is not a concurrent abstraction.

- **E2. Foundation and Objective-C bridging behavior**
  - Expect: Distinguish representation conversion, membership snapshots, shared mutable elements, opaque boxes, null sentinels, numeric validation, and text index units. Isolate and measure the boundary.
  - Caveats: An adapter alone does not reduce conversion count; canonical String equality can hide UTF-16 changes relevant to a cache. Foundation runtime availability and Subprocess package/platform support are separate from the Swift compiler version.
  - Score anchors: `3` — Distinguishes membership snapshots from shared mutable elements. `4` — Chooses a conversion boundary with valid caching and measured costs.
  - Ask: "Which mutations remain visible after converting NSMutableArray to a Swift array of reference objects?"
  - Ask: "Why can a boxed Swift payload round-trip but fail JSON validation, and why is NSNumber conversion insufficient for a quantity schema?"
  - Ask: "What does the prepared renderer's conversion counter establish, and how do invalidation, retention, isolation, and actual profiling affect the decision?"
  - Exercise: "Bridge a mutable Foundation collection into a Swift array of reference elements. Check membership after append and payload state after mutation."
  - Extension: [Full chapter exercise and deeper checks](<./E. Interop, Modules, and Distribution/E2 — Foundation and Objective-C Bridging Behavior.md>).
  - Code Probe:
    ```swift
    import Foundation

    final class MutablePerson: NSObject {
        var name: String
        init(_ name: String) { self.name = name }
    }

    func membershipAndReferences() -> (Int, Int, Bool, String) {
        let person = MutablePerson("Ada")
        let original = NSMutableArray(object: person)
        let snapshot = original as! [MutablePerson]
        original.add(MutablePerson("Grace"))
        person.name = "Updated"
        return (snapshot.count, original.count, snapshot[0] === person, snapshot[0].name)
    }
    ```
  - Probe outcome: (1, 2, true, "Updated"). Swift membership is a snapshot while the same mutable person is shared. The local fixture does not establish concurrent mutation safety.

- **E3. C interoperability and unsafe boundaries**
  - Expect: Translate real C ownership, count, lifetime, nullability, error, and callback contracts into a small Swift-facing API. Verify generated headers and native consumers.
  - Caveats: @safe, unsafe, and annotations do not verify a C implementation. Generated safe overloads and raw ABI implementations are separate; class aliases share close state, and non-sendability does not synchronize native globals.
  - Score anchors: `3` — Translates native count, ownership, and lifetime obligations. `4` — Designs a small safe interface verified with native consumers and diagnostics.
  - Ask: "Who owns an output returned on failure or with zero length, and where must cleanup be registered?"
  - Ask: "Why does a counted/noescape declaration provide a safe Span overload while its Swift @implementation still takes pointer/count?"
  - Ask: "When is passUnretained valid for callback context, and what changes if callbacks can escape or overlap?"
  - Exercise: "Wrap a synchronous pointer/count C call. Check empty input and failure cleanup; state whether the callee may retain the input."
  - Extension: [Full chapter exercise and deeper checks](<./E. Interop, Modules, and Distribution/E3 — C Interoperability and Unsafe Boundaries.md>).
  - Code Probe — [setup and shared declarations](<./E. Interop, Modules, and Distribution/E3 — C Interoperability and Unsafe Boundaries.md>):
    ```swift
    import E3Native

    private final class VisitBox { var values: [UInt8] = [] }
    private func visitThunk(_ value: UInt8, _ context: UnsafeMutableRawPointer) {
        let box = unsafe Unmanaged<VisitBox>.fromOpaque(context).takeUnretainedValue()
        box.values.append(value)
    }

    func visitBytes(_ values: [UInt8]) -> [UInt8] {
        let box = VisitBox()
        return withExtendedLifetime(box) {
            values.withUnsafeBufferPointer { buffer in
                unsafe e3_visit(buffer.baseAddress, buffer.count, visitThunk,
                                Unmanaged.passUnretained(box).toOpaque())
            }
            return box.values
        }
    }
    ```
  - Probe outcome: With E3's header and implementation, visitBytes returns its input sequence, including empty input and every UInt8. The context is valid because callbacks are synchronous/serial, C retains nothing, and the box stays alive through the entire call; asynchronous registration requires a different ownership protocol.

- **E4. C++ interoperability**
  - Expect: Review native copy/move/destruction, owned versus borrowed data, mutation invalidation, exceptions, concurrency confinement, and consumer build requirements. Select this depth when the role includes C++ interoperability.
  - Caveats: C++ classes usually import as values; copies follow native semantics. Stable span inputs, experimental lifetime-bound return overloads, and C++ exports have different support limits. Validate empty inputs and actual native execution alongside compilation.
  - Score anchors: `3` — Explains imported copy and lifetime semantics from the native API. `4` — Chooses confinement and owned snapshots under native build and failure constraints.
  - Ask: "Which native operations can copy, invalidate a view, or throw, and how would your Swift API encode each contract?"
  - Ask: "Why does a Swift-only snapshot not make the engine Sendable or remove a dependent module's C++ setting?"
  - Ask: "What evidence distinguishes the stable span input support from the conditional return-overload examples?"
  - Exercise: "Copy an imported C++ value, mutate the copy, and observe native copy/destruction behavior. Explain which result depends on the native contract."
  - Extension: [Full chapter exercise and deeper checks](<./E. Interop, Modules, and Distribution/E4 — C++ interoperability.md>).
  - Code Probe — [setup and shared declarations](<./E. Interop, Modules, and Distribution/E4 — C++ interoperability.md>):
    ```swift
    import E4Core

    func valueProbe() -> (CInt, CInt, CInt) {
        e4ResetCopies()
        let original = E4Value()
        var copy = original
        copy.setFirst(9)
        return (original.first(), copy.first(), e4CopyCount())
    }
    func borrowedRead(_ value: borrowing E4Value) -> CInt { value.first() }
    ```
  - Probe outcome: With E4's C++20 header, original remains 1 and copy becomes 9, with at least one observed native copy. Live values return to zero after the probe. The separate borrowing call adds no observed copy in the checked builds; this is not a universal allocation or speed guarantee.

- **E5. Modules, packages, targets, products, and resources**
  - Expect: Distinguish packages/products/targets/modules, apply package access deliberately, own resources in their targets, and verify an acyclic graph with tests and an ordinary external consumer. Include Swift Build's 6.4 default, artifact discovery, and SBOM scope.
  - Caveats: A product is not an access-control firewall. Bundle.main is not a package-wide resource bundle. More targets do not guarantee faster builds. Successful SBOM generation alone does not establish schema validity or complete dependency coverage.
  - Score anchors: `3` — Distinguishes products, targets, modules, access, and resource ownership. `4` — Justifies a package graph using client, build, and distribution evidence.
  - Ask: "Why can a client depend on ProfileKit, import ProfileCore, and still fail to use its package helper?"
  - Ask: "How do process/copy rules and the owning target change resource lookup in tests and consumers?"
  - Ask: "Why do the macOS build, iOS build, and standalone package-graph SBOMs differ?"
  - Exercise: "Place a resource in its owning package target and load it from a separate consumer. Check public construction and resource lookup."
  - Extension: [Full chapter exercise and deeper checks](<./E. Interop, Modules, and Distribution/E5 — Modules, packages, targets, products, and resources.md>).
  - Code Probe (in the separate Consumer package described in E5) — [setup and shared declarations](<./E. Interop, Modules, and Distribution/E5 — Modules, packages, targets, products, and resources.md>):
    ```swift
    import Foundation
    import ProfileCore
    import ProfileStore
    import ProfileUI

    @main struct Consumer {
        @MainActor static func main() async throws {
            let profile = try await BundledProfileStore().profile(id: "ada")
            let title = try ProfilePresenter().title(for: profile)
            guard title == "Hello: Ada", try BundledProfileStore.metadata() == "copied directory" else {
                fatalError("External resource/API contract failed")
            }
            print(title)
            print(ProfilePresenter.platformLabel)
        }
    }
    ```
  - Probe outcome: The ordinary macOS consumer prints Hello: Ada and macOS sample after loading real target-owned resources. Product-name import and package/internal helper access fail separately. The four package tests pass under each of the two checked isolation policies; iOS is compile-only evidence.

- **E6. Import visibility and dependency leakage**
  - Expect: Enforce intentional dependency exposure with explicit imports, inspect typed errors and serialized/layout details, and validate SDK-owned mappings with ordinary external clients.
  - Caveats: Source access, member lookup, transitive module loading, and linkage are separate. Import-default/member-visibility flags change different rules. A resilient client may omit a hidden module while still requiring its executable code; non-resilient, package, and testing cases differ.
  - Score anchors: `3` — Controls dependency exposure in the actual public interface. `4` — Separates source access, module loading, and executable linkage requirements.
  - Ask: "Why does an internal import reject a public vendor error/result while still permitting ordinary implementation use?"
  - Ask: "Why can a transitive extension compile without proving that public import re-exports all vendor names?"
  - Ask: "Which clients can omit the vendor module, and why does that not remove the vendor link requirement?"
  - Exercise: "Hide a vendor result behind an SDK-owned value and error. Verify the public signature and a separate consumer under declared build settings."
  - Extension: [Full chapter exercise and deeper checks](<./E. Interop, Modules, and Distribution/E6 — Import Visibility and Dependency Leakage.md>).
  - Code Probe (compile-only intended failure against E6's real ThirdParty module) — [setup and shared declarations](<./E. Interop, Modules, and Distribution/E6 — Import Visibility and Dependency Leakage.md>):
    ```swift
    internal import ThirdParty

    public func expose(_ raw: String) throws(VendorFailure) -> VendorRecord {
        try VendorDecoder().decode(raw)
    }
    ```
  - Probe outcome: The public signature is rejected because its typed thrown error is imported as internal. Making the import public compiles but exposes vendor contracts. E6's domain adapter instead maps real fields/errors and verifies separate module-loading and link behavior.

- **E7. Resilience, ABI, and module stability**
  - Expect: Separate source, ABI, compiler-import, behavioral, and deployment contracts. Test unchanged consumers, supported interfaces/platform slices, and actual runtime delivery under the chosen library-evolution model.
  - Caveats: Frozen layout includes private storage; a suitable default witness can support protocol evolution, but not every default or behavior change is safe. XCFramework creation and successful tool exit codes do not prove every release gate.
  - Score anchors: `3` — Separates source, ABI, and compiler-import compatibility. `4` — Defines distribution gates across supported compilers, platform slices, and runtimes.
  - Ask: "Why does an unchanged client retain its old default argument while calling a new protocol default witness?"
  - Ask: "What does an interface-only import prove with one installed compiler, and what still needs a supported-Xcode matrix?"
  - Ask: "Why can a successfully built binary package still fail to load its framework? What evidence would distinguish linking, embedding, and runtime search-path failures?"
  - Exercise: "Build a consumer against a resilient library, replace it with a compatible version, and compare that unchanged executable with a rebuilt consumer."
  - Extension: [Full chapter exercise and deeper checks](<./E. Interop, Modules, and Distribution/E7 — Resilience, ABI, and Module Stability.md>).
  - Code Probe (compile against E7SDK v1, then run the identical binary against v2) — [setup and shared declarations](<./E. Interop, Modules, and Distribution/E7 — Resilience, ABI, and Module Stability.md>):
    ```swift
    import E7SDK

    struct ClientProvider: MessageProvider {
        func title() -> String { "hello" }
    }
    func label(_ state: DeliveryState) -> String {
        switch state {
        case .queued: "queued"
        case .finished: "finished"
        @unknown default: "unrecognized"
        }
    }
    @main struct Client {
        @MainActor static func main() {
            let reading = Reading(42)
            let copy = reading
            precondition(copy.value == 42)
            print(copy.value, label(currentState()), render(ClientProvider()), attempts(), separator: " | ")
        }
    }
    ```
  - Probe outcome: v1 prints 42 | queued | hello | 1. With only the library replaced, it prints 42 | unrecognized | hello + default | 1. Rebuilding the same source uses default argument 5 and warns about the new known enum case. This is process-by-process replacement, not live hot-swapping.

- **E8. Binary frameworks, inlineability, and public implementation detail leakage**
  - Expect: Distinguish source access, callable library symbols, exposed bodies, mandatory inlining, and library-side specialization. Recognize @export, @inline(always), and @specialized as 6.3 controls available in the 6.4 baseline.
  - Caveats: Old clients can retain emitted bodies even with @inline(never). Source-internal ABI helpers remain compatibility commitments; stable arithmetic still needs a valid input domain and performance evidence.
  - Score anchors: `3` — Distinguishes public access, ABI symbols, and exposed implementation bodies. `4` — Chooses visibility and specialization policy with explicit compatibility commitments.
  - Ask: "Why does replacing the dylib update an interface-exported function but leave the client-emitted function unchanged?"
  - Ask: "When is @usableFromInline preferable to a public helper, and when should the caller stay opaque?"
  - Ask: "How can prespecialization retain a generic API without exposing its implementation?"
  - Exercise: "Compare a library-called function with a client-visible body across a compatible library replacement. Explain which output an unchanged client may retain."
  - Extension: [Full chapter exercise and deeper checks](<./E. Interop, Modules, and Distribution/E8 — Binary frameworks, inlineability, and public implementation detail leakage.md>).
  - Code Probe (build against E8Visibility v1, then replace only the compatible dylib) — [setup and shared declarations](<./E. Interop, Modules, and Distribution/E8 — Binary frameworks, inlineability, and public implementation detail leakage.md>):
    ```swift
    import E8Visibility

    @main struct Client {
        @MainActor static func main() {
            print(libraryRevision(), clientRevision(),
                  visibleRevision(), forcedRevision())
        }
    }
    ```
  - Probe outcome: the optimized original prints 1 1 1 1 against v1 and 2 1 1 1 against v2. Rebuilding prints 2 2 2 2. Client emission, inlining, and library calls are distinct contracts; this is a checked fixture, not a guarantee that every inlinable call is inlined.

### F. Tooling, Testing, Diagnostics, and Language-Adjacent Features

- **F1. Testing strategy in modern Swift**
  - Expect: Choose unit/integration/UI/performance boundaries, await async results, preserve actor isolation, and use independent parameterized fixtures. Control request arrival, completion, cancellation, and cleanup. Inject a clock or time dependency when elapsed time determines behavior.
  - Caveats: Confirmation scopes do not wait by themselves; resuming a continuation is not task completion. Serialized tests are not a global lock or an ordering contract. Swift 6.4 interoperability depends on direction, mode, runner, and XCTest implementation; test deliberate failures during migration.
  - Score anchors: `3` — Controls the events and time that determine the asserted behavior. `4` — Chooses test boundaries that expose the defect and acknowledge remaining integration risks.
  - Ask: "What does a manual debounce gate prove, and what still needs a virtual-clock or integration test?"
  - Ask: "How would a controllable clock verify debounce or timeout boundaries without sleeping? Which scheduler or external-I/O behavior still needs integration coverage?"
  - Ask: "Why can limited interop leave an XCTest failure inside Swift Testing passing, and why must a runner's exit status survive output formatting?"
  - Ask (extension): "How do warning severity, Test.cancel, CustomTestReflectable, attachments, and per-case repetitions affect evidence without replacing assertions?"
  - Exercise: "Test a debounce policy with an injected clock or controllable time dependency. Advance time, check emission and cancellation, and join the owned work."
  - Extension: [Full chapter exercise and deeper checks](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F1 — Testing Strategy in Modern Swift.md>).
  - Code Probe (intentionally failing; run separately from the passing suite):
    ```swift
    import Testing

    @Test func expectationVersusRequirement() throws {
        let condition = Bool(false)
        #expect(condition)
        print("after expect")
        try #require(condition)
        print("after require")
    }
    ```
  - Probe outcome: The runner records two issues, prints after expect, and does not print after require. Require throws on failure; expect ordinarily continues. F1 separately verifies actual interop failure reporting and warns against treating a later repetition's passing summary as the command's exit status.

- **F2. Debugging Swift code with LLDB and compiler diagnostics**
  - Expect: Reduce generic, builder, and actor-boundary diagnostics under recorded Swift 6.4 settings. Separate stored-state inspection from evaluator execution, track request identity across suspension, and explain precise explicit-module tracking.
  - Caveats: A warning exception cannot repair a native error. Task names do not guarantee cancellation or freshness; their creation/current/handle APIs have distinct availability. A retained reference is not a disconnected sending transfer. Debugger, sanitizer, and memory evidence have scoped limits.
  - Score anchors: `3` — Explains the diagnostic and verifies the chosen boundary. `4` — Combines compiler, debugger, and runtime evidence without overstating any tool.
  - Ask: "How do you isolate a generic or builder error, and why are different ViewBuilder branch types not automatically invalid?"
  - Ask: "Why can stored-field inspection succeed while a computed-property expression fails, and can po change the target?"
  - Ask: "What changed in Swift 6.4 module tracking, and what must custom or archived-build debugging preserve?"
  - Ask: "What distinguishes a closed ownership cycle, a data race, and a stale result?"
  - Exercise: "Reduce a non-Sendable actor-return diagnostic to a small example. Repair it with an isolated operation or snapshot and check the caller."
  - Extension: [Full chapter exercise and deeper checks](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F2 — Debugging Swift code with LLDB and compiler diagnostics.md>).
  - Code Probe (intentionally fails; Swift 6.4 compiler, Swift 6 language mode):
    ```swift
    import Foundation

    final class ImageCache {
        var storage: [String: Data] = [:]
    }

    actor ImageLoader {
        private let cache = ImageCache()

        func cacheForDebugging() -> ImageCache {
            cache
        }
    }

    func use(loader: ImageLoader) async {
        let cache = await loader.cacheForDebugging()
        cache.storage["avatar"] = Data()
    }
    ```
  - Probe outcome: The actor retains a mutable non-Sendable ImageCache, so returning it across the boundary rejects. RegionIsolation warning suppression leaves the native error intact. Adding sending to the retained return also rejects; returning a fresh disconnected value is a separate accepted case. The corrected note keeps mutation actor-isolated and exposes Dictionary/Data snapshots.

- **F3. Performance investigation habits**
  - Expect: Define representative scenarios and matching optimized builds; distinguish wall time, CPU sampling, waits, allocations, and retained memory. Use signposts, Instruments, and targeted benchmarks with explicit measured boundaries.
  - Caveats: New storage and optimization controls do not establish universal speedups. A checksum alone is not a complete optimization barrier; a passing measurement test without a configured baseline is not a regression gate. Offloading must preserve task ownership, cancellation, and publication freshness.
  - Score anchors: `3` — Forms a falsifiable hypothesis and measures the relevant cost. `4` — Connects user-visible gains, implementation costs, and a defensible regression policy.
  - Ask: "What would you measure before rewriting an abstraction-heavy path, and how would you distinguish CPU work, contention, and blocking?"
  - Ask: "How do you compare Array, UniqueArray, and Span without changing the ownership requirement or timing the wrong work?"
  - Ask: "Why can correct output and an accurate timer still produce a misleading benchmark?"
  - Ask: "Which user-visible gain justifies the complexity, memory, and API cost of a proposed optimization?"
  - Exercise: "Define one representative workload, metric, and baseline. Compare one change in matching optimized builds, verify equivalent behavior, and report measurement limits."
  - Extension: [Full chapter exercise and deeper checks](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F3 — Performance investigation habits.md>).
  - Code Probe (combine with F3's StorageWork definitions; Apple OS 27 or newer) — [setup and shared declarations](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F3 — Performance investigation habits.md>):
    ```swift
    @available(anyAppleOS 27.0, *)
    func storageProbe() {
        let original: [UInt64] = [1, 2, 3]
        var ordinary = original
        var unique = UniqueArray<UInt64>()
        for value in original { unique.append(value) }
        let arrayResult = StorageWork.updateArray(&ordinary, salt: 2)
        let uniqueResult = StorageWork.updateUnique(&unique, salt: 2)
        print(original, ordinary, arrayResult, uniqueResult, StorageWork.sumSpan(ordinary.span))
    }
    ```
  - Probe outcome: It prints [1, 2, 3] [3, 4, 5] 12 12 12. This proves the illustrated values and snapshot independence; it supplies no timing, allocation-count, or UI performance evidence. The chapter's separate controlled experiment measures a narrower workload and records its limits.

- **F4. Macros and compile-time code generation**
  - Expect: Explain expression/declaration and attached roles, including body replacement; separate the public macro contract, host plugin, and compiled client code. Validate syntax, compiler integration, and behavior with a supported SwiftSyntax/toolchain combination.
  - Caveats: Generated code still has evaluation, ownership, isolation, access, and availability consequences. Syntax snapshots do not prove type correctness; syntax APIs do not provide general inferred conformance information. Macros are not universally additive-only.
  - Score anchors: `3` — Separates macro syntax transformation from generated-code correctness. `4` — Justifies generation and manages host, client, and toolchain compatibility.
  - Ask: "What can a macro validate from syntax, and what must the generated Swift contract leave to the compiler?"
  - Ask: "Why can a macro snapshot pass while a client fails to compile or behaves incorrectly?"
  - Ask: "How can a body macro replace an unresolved original body, and what constraints still apply to its replacement?"
  - Ask: "Which costs and compatibility obligations justify using a macro rather than a protocol, property wrapper, or ordinary code?"
  - Exercise: "Generate one expression wrapper that evaluates its argument once. Check expansion, client compilation, and runtime behavior separately."
  - Extension: [Full chapter exercise and deeper checks](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F4 — Macros and compile-time code generation.md>).
  - Code Probe (requires F4's temporary InterviewMacros/InterviewPlugin package; separate client file) — [setup and shared declarations](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F4 — Macros and compile-time code generation.md>):
    ```swift
    import InterviewMacros

    @RoutePath("/users")
    struct UsersRoute {}

    @ReplaceWithZero
    func replaced() -> Int {
        nameThatDoesNotExist
    }

    func probe() {
        var evaluations = 0
        func next() -> Int { evaluations += 1; return 7 }
        let (value, source) = #sourceValue(next())
        precondition(value == 7 && source == "next()" && evaluations == 1)
        precondition(UsersRoute.path == "/users")
        precondition(replaced() == 0)
        print(value, source, evaluations, UsersRoute.path, replaced())
    }

    probe()
    ```
  - Probe outcome: Prints 7 next() 1 /users 0. The expression runs once; the member macro supplies the path; the body macro discards the unresolved original body and supplies a checked Int result. Changing the declared result to String fails. These are custom teaching macros, not built-in Swift APIs.

- **F5. Language mode, build settings, and migration flags**
  - Expect: Reproduce the rubric baseline: Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, and nonisolated default isolation. Separate these settings from SDK and deployment target. Use warning policy and @diagnose without confusing suppression with safety; distinguish strict memory safety from concurrency.
  - Caveats: Compiler release, language mode, target defaults, and SDK can change diagnostics independently. Preserve the relevant settings when comparing failures.
  - Score anchors: `3` — Distinguishes compiler, language, isolation, SDK, and deployment settings. `4` — Defines reproducible settings and warning policy across a migration.
  - Ask: "What kinds of issues are surfaced by strict concurrency, default isolation, and strict memory safety, respectively?"
  - Ask: "Why is build-setting literacy important when debugging behavior differences between targets?"
  - Ask: "Can @diagnose suppress a Swift 6 Sendable error, and does importing a module re-infer its isolation under the client’s defaults?"
  - Exercise: "Compile one capture example in Swift 5 complete-checking and Swift 6 modes. Record settings and explain the change in diagnostic severity."
  - Extension: [Full chapter exercise and deeper checks](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F5 — Language mode, build settings, and migration flags.md>).

- **F6. Observation, property wrappers, and language-adjacent state tools**
  - Expect: Explain actual tracked reads, generated storage, State view identity, Bindable projections, ignored fields, and explicit model isolation. Fix cache-hit and external-storage observation gaps. Separate request cancellation from permission to publish a result.
  - Caveats: Bindable can retain its model. Observation does not supply synchronization, persistence, or lossless event delivery. Advanced options/events/tokens require Apple OS 27 in the tested SDK; Observations requires 26. Check the SDK contract and callback lifecycle before adopting newer APIs.
  - Score anchors: `3` — Identifies tracked reads and separates observation from synchronization. `4` — Designs observable state with explicit identity, lifetime, and publication policy.
  - Ask: "Why can an ignored cache work on its first read but fail to invalidate after a cache-hit read?"
  - Ask: "Which actor runs a synchronous change callback versus a continuous callback, and what owns the observation's lifetime?"
  - Ask: "Why can cancelling the older task and comparing query text still publish a stale search result?"
  - Exercise: "Track one observable property. Check unrelated and ignored writes, explain one-shot registration, and state the model's required isolation."
  - Extension: [Full chapter exercise and deeper checks](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F6 — Observation, property wrappers, and language-adjacent state tools.md>).
  - Code Probe (call on MainActor; macOS 14 / iOS 17 or newer):
    ```swift
    import Observation

    @MainActor @Observable
    final class CounterModel {
        var count = 0
        var unrelated = 0
        @ObservationIgnored var bookkeeping = 0
    }

    @MainActor
    func legacyProbe() {
        let model = CounterModel()
        _ = withObservationTracking { model.count } onChange: {
            print("change")
        }
        model.unrelated = 1
        model.bookkeeping = 1
        model.count = 1
        model.count = 2
        print(model.count)
    }
    ```
  - Probe outcome: Swift 6.4 prints change once, then 2. Only count was tracked; unrelated/ignored writes do not trigger this registration, and the original callback is one-shot. It runs before the relevant write, without granting actor-isolated access to its Sendable closure.
