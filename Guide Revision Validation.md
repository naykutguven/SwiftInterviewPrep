---
tags:
  - swift
  - interview-prep
  - validation
  - swift6-4
---

# Guide Revision Validation

The October 16 record below describes that historical revision. The [initial October 28 teaching revision](#teaching-revision--2026-10-28) updates the template, README, and D1–D3. The [remaining D-guide revision](#remaining-d-guides--2026-10-28) extends the same explanation and example requirements to D4–D14. The [A-guide revision](#a-guides--2026-10-28) applies that standard to A1–A16. The other 40 guides in B, C, E, and F are still awaiting review against the new standard.

Reviewed 2026-10-16. All **70 topic guides** were updated from the existing working tree: A1–A16, B1–B14, C1–C12, D1–D14, E1–E8, and F1–F6. F5 and D14 were revised first to establish consistent settings.

The approved template and revised rubric are byte-for-byte unchanged. Topic filenames, IDs, YAML, aliases, and existing tags are preserved, including D14's existing renamed filename and legacy alias. Existing README and migration/validation documents were left intact. Repeated level scripts, spoken versions, summaries, and flashcards were removed; selected ownership, native-interface, migration, and measurement details remain in marked optional subsections. Missing dependencies discovered while extracting specialist examples were restored.

## Configuration

Apple Swift **6.4**, `swiftlang-6.4.0.34.1`, from `/Applications/Xcode_27.app/Contents/Developer`. Validation used the arm64 macOS host and installed Apple SDKs. Unless explicitly comparing settings, compiler probes used:

```sh
-swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances
```

Object compilation was used for ownership checks. Compiler module caches, generated sources, native objects, packages, binaries, and logs stayed in temporary fixtures under `/tmp/swift-guide-revision`. Macro-capable compilation used the supported `-disable-sandbox` option because nested plugin sandbox creation failed in the execution environment. Package tests used their corresponding sandbox option and temporary cache paths. Build/test output passed through xcsift with the underlying pipeline failure status preserved.

## Structural and editorial checks

- Confirmed exactly 70 expected topic IDs and the same filenames as the initial working-tree snapshot.
- Parsed all 70 YAML frontmatter blocks with Ruby's YAML parser and compared them with the original blocks.
- Checked required template headings, one goal, rubric questions, focused task, acceptance checks, solution, reasoning, practical guidance, related guides, and consolidated official sources. Worked examples are included where they teach a separate lesson.
- Checked guide wiki links and heading destinations outside code fences. Template illustration links and README's pre-existing illustrative Related Topic link are outside the revised-guide check.
- Checked for unresolved template tokens, authoring comments, old answer scripts, stale numbered-section references, and prose code fences.
- Reviewed default-isolation claims, caller-isolated async behavior, imported contracts, required annotations, availability qualifications, and source/proposal distinctions. Removed redundant nonisolated declarations.
- Reviewed the working-tree diff and whitespace, including restored native headers, conformance examples, and callback dependencies. No topic was deleted or renamed by this revision.

## Compiler and runtime evidence

**57 accepted standalone focused fixtures** were object-compiled, linked, and executed under the baseline. Two additional focused rejection variants were checked with their shared declarations. Separate native/package fixtures covered the remaining exercises; this does not imply every optional snippet or platform combination was rerun.

| Area | Evidence obtained |
| --- | --- |
| Language, types, ownership | Accepted examples plus intended rejections for immutable mutation, initializer ordering, guarded-switch coverage, mismatched keys/elements, erased key-path writes, overlapping inout, use after consume, unsafe sendable capture, and isolated-state access. |
| Settings and imports | Swift 5 complete-checking warning versus Swift 6 error; all four D14 default/execution settings combinations; separately compiled callee modules with caller-isolated defaults off/on and the same client. |
| Structured/concurrent work | Child result association, actor and Mutex counters, gated cancellation and joined async cleanup, out-of-order freshness, allowed/rejected isolated-conformance callers, one-shot callback completion, and explicit stream finishing. |
| Public APIs and dependencies | Ordinary separate clients for A11/B10; private/setter rejections; B11 enum-addition warning; E6 hidden vendor-module compilation, public/serialized leakage rejections, intentional missing-vendor link failure, and successful complete linkage. |
| Native interoperability | Generated Objective-C selector/header and native invocation; profile-facade success, invalid input, cancellation before loading, late success after cancellation, and dropped-token delivery. E3's C allocation/failure modes returned live handle/buffer counts to zero. C++ copy/destruction and borrowed-read probes ran with both -Onone and -O. |
| Binary replacement | E7/E8 original executables ran against v1 and compatible v2 dylibs with unchanged executable hashes, then were rebuilt against v2. Outputs matched the guides' distinct default-argument, witness, and client-body behavior. |
| Testing/resources | The injected-time debounce test passed. A separate expected-failure target recorded two issues, printed after expect, skipped after require, and returned failure. An ordinary external SwiftPM consumer built and loaded its target-owned resource. |
| SDK-sensitive operations | Borrow/mutate accessors and noncopyable borrowing iteration compiled and ran. Optional.ref and noncopyable-input Optional.map rejected as documented. Basic temporary-span operations object-compiled for macOS 13 and iOS 18; the guarded OS 27 raw byte-order load ran on the host. Observation's one-shot tracked/unrelated/ignored-write exercise compiled and ran. |
| Traps and optimization | Debug assertion, optimized precondition, and dynamic exclusivity traps ran in separate processes with eight-second timeouts. Optimized assert skipped its condition. No undefined pointer access or invalid unchecked arithmetic was executed. C10's repair also ran with -O. F3's narrow membership comparison ran seven optimized samples with equivalent results; these are not UI or allocation benchmarks. |
| Macros | The implementation object compiled against installed host SwiftSyntax modules. Direct expansion produced `(next(), "next()")`; compiling and executing that expanded client verified one evaluation. Full pinned-package/plugin integration is limited below. |

Version-sensitive review used official Swift/Apple documentation, the selected SDK/compiler, and relevant proposals. In particular, [SE-0461](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md) and [SE-0470](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md) support the isolation model; [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md) and [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md) cover synthesis/accessors. [SE-0532](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0532-optional-noncopyable-improvements.md) remains accepted rather than evidence that every proposed API is present in this SDK. [ST-0021](https://github.com/swiftlang/swift-evolution/blob/main/proposals/testing/0021-targeted-interoperability-swift-testing-and-xctest.md) documents the interoperability modes; the guides retain the distinction between its formal scope and a particular Xcode XCTest runtime.

## Material limits

The complete F4 package could not resolve pinned SwiftSyntax 604.0.0: the temporary repository cache was unavailable and GitHub DNS resolution was blocked. Its maintained package source, plugin registration, syntax tests, and negative clients are preserved, but a complete package/plugin run was not established. Host implementation/expansion and expanded-client checks are narrower evidence.

No new physical-device/UI, live-server, LLDB session, Instruments/sanitizer campaign, supported-Xcode matrix, full XCFramework distribution matrix, or older-runtime execution is claimed. The focused iOS temporary-span probe is compilation only. Historical specialist measurements and broader platform observations remain scoped to [[Swift 6.4 Validation Record]] and their original settings; they are not silently promoted to results under this revised baseline. Every optional native/macro fixture and deliberate rejection was not exhaustively rerun.

## Teaching revision — 2026-10-28

### Scope and acceptance requirements

Revised the answer template and README, then used D1 as the reference chapter before applying the standard to D2–D3. The rubric, topic IDs, filenames, and YAML frontmatter are unchanged. At that stage, the other 67 topic guides had not been revised or certified against this teaching standard. Earlier evidence above remains historical.

The template now requires topic-wide learning goals, stated prerequisites, essential mechanics before advanced combinations, definitions before jargon, and a worked example before assessment. It removes the word-count target and permits useful explanatory repetition. Every included pitfall must contain mistaken code, its consequence, its explanation, corrected code, and an explanation of the correction. Every interview answer, including extensions, must contain a direct answer, reasoning, local code, an outcome walkthrough, and a meaningful boundary. Named shared declarations may support multiple examples; a distant exercise or link cannot replace the local code. Optional interview follow-ups must be fully answered Extension entries.

Acceptance review maps essential rubric expectations to teaching material, questions to local examples, pitfalls to mistake/correction pairs, and exercise checks to verification. Compiler evidence and editorial review are separate requirements. No fixed number of words, fences, or headings establishes instructional quality by itself.

### Coverage review

| Guide and rubric coverage | Teaching sections and local examples | Verification |
| --- | --- | --- |
| D1: tasks, async/await, suspension versus blocking, concurrency versus parallelism | Mental model defines the terms; sequential and fixed-child mechanics precede the first pitfall; the await answer traces the current task | Sequential/fixed result checks; no claim that a yield proves actual overlap or parallel execution |
| D1: task trees, child scope exit, error observation, unstructured ownership | Scope-exit table and group example; Task pitfall; unawaited-child worked example; structured-error answer; two-child exercise | Normal group return joins; implicit async-let cancellation/cleanup precedes return; error reaches caller; required result retains association |
| D1: forward progress, callback adaptation, blocking legacy work | Legacy-file mistake/correction; semaphore answer; callback and blocking-read adapters | Published adapters compiled; both corrected file bridges read a temporary fixture; dangerous hidden semaphore wait type-checked only; direct async semaphore call rejected separately |
| D1 extension: awaited defer before early return | Final, explicitly labelled Extension answer | Lease cleanup completes before the caller receives the return value |
| D2: cancellation request, observation, handlers, completion and cleanup | Small unchecked/checking examples; handler mechanics; cooperative-cancellation answer; gated exercise | Already-cancelled handler/operation ordering; before-entry and during-work cancellation both throw and join exactly one awaited cleanup |
| D2: priority inheritance/escalation, task-local propagation, task context | Dedicated mechanics for task locals and priorities; priority-order pitfall; child-inheritance answer | Task-local values propagate to ordinary/structured tasks and not detached work; no numeric priority or scheduling order is asserted |
| D2: ownership and shared producers | Cancelled-consumer worked example | Consumer cancellation leaves producer uncancelled and does not interrupt its value wait; driver releases and joins the work |
| D2: request identity and all publication paths | Search-state mistake/correction; A → B → A identity answer | Controlled old success/error with replacement, with/without cancellation; cancelled current success/error without replacement; stale cleanup cannot clear the new slot |
| D2: elapsed-time policies and timeout ownership | Timed-result mistake/correction; clock answer; timeout answer | Constructed-instant budget boundaries; timeout-helper success and service failure; no real system-sleep or hard-latency guarantee |
| D2 extension: shielding and runtime availability | Final Extension answer with static/handle observations and enclosed child | Cancelled state masked inside shields, observable afterward, handle remains cancelled; OS 26 deployment rejected |
| D3: sequential, fixed, dynamic, regular throwing, and discarding work | Five essential mechanics and choice table; fixed dashboard worked example; fixed-pair answer | Runnable examples check dependency, heterogeneous values, dynamic membership, all-or-nothing failure, cache warming, and fixed result roles |
| D3: ordering and bounded resources | Immediate-await loop pitfall; completion-order mistake/correction; bounded generic helper | Exercise forces actual collection order 2 → 0 → 1 while returning input order; helper reaches a peak of two active transforms, preserves nil outputs and order, handles empty/invalid input, and stops admission on failure/cancellation |
| D3: result/error policy | Ignored-error pitfall and five separately demonstrated comparison cases | Catch-next partial success; nextResult retains both outcomes; waitForAll drains before throwing; ignored results return normally; throwing discarding group cancels a long sleeper and propagates failure |
| D3: awaited binding order and race completion | Fixed-pair answer; controlled winner/cleanup race | Error observation waits behind the first binding; selected winner does not permit return while a loser's cleanup gate remains closed |
| D3: exercise success, failure, and joining | Controlled three-request exercise | Input-order success, selected notification failure, all three cleanup acknowledgments before caller completion, and released gates on both paths |

D1's reference review checked all five rubric questions, every essential expectation, three pitfall pairs, and the focused exercise before D2–D3 were adapted. Definitions and small mechanics now precede continuations and gates. Its normal-group-exit rule is distinguished from async-let cancellation, and core lifetime evidence is no longer hidden in an optional subsection. Only the async-defer question is labelled an extension.

The corresponding review of D2–D3 checked all six D2 questions and all three D3 question groups, including each of the five group-policy comparison cases. D2's shielding extension follows the foundations. D3 introduces basic groups before its generic bounded helper. Across the three chapters, all **10 pitfall blocks** and **14 interview answers** include local code and explanations. Shared setups, allowed output orders, compile-only examples, and unsafe-to-run examples are identified at their use sites.

### Compiler and runtime evidence

Apple Swift **6.4**, `swiftlang-6.4.0.34.1`, from `/Applications/Xcode_27.app/Contents/Developer`, with the installed macOS SDK and an explicit `arm64-apple-macos27.0` target. The baseline flags are the same as above. These checks additionally treat warnings as errors and use a temporary module cache. The two `@TaskLocal` macro examples required the supported `-disable-sandbox` compiler option because this execution environment cannot create a nested plugin sandbox; ordinary user builds do not require that environment-specific workaround. Verbose compiler output was passed through xcsift with the compiler/pipeline status preserved and raw logs retained.

Every published Swift fence was extracted and checked with exactly its named shared declarations. Function-only and unsafe examples were type-checked. Runnable snippets were linked and executed in separate processes with a 12-second timeout. No dangerous semaphore wait was executed. Additional drivers exercise published helpers and corrections; their setup and source hashes are recorded separately.

| Guide | Published Swift blocks checked | Accepted compiler cases, including extra drivers | Runnable cases passed |
| --- | --- | --- | --- |
| D1 | 20 | 22 | 16 |
| D2 | 22 | 26 | 17 |
| D3 | 24 | 27 | 22 |
| Total | **66** | **75** | **55** |

Two additional intended rejections establish the selected SDK's direct semaphore-wait prohibition in an async context and cancellation-shield availability when targeting macOS 26. The synchronous semaphore wrapper in D1 still compiles and remains unsafe to call, illustrating the diagnostic's limit. Shielded child/static-state observations pass on the selected OS 27 runtime.

The extra drivers cover two corrected file bridges, index construction, current/stale search publication, timeout-helper success/service failure, bounded optional-result fan-out, admission stopping after failure/cancellation, and delayed async-let error observation. They do not depend on elapsed-time sleeps to prove event order. The long sleeper in the discarding-group example represents cancellable work and exits through cancellation.

Fixtures, commands, source hashes, raw compiler output, xcsift output, runtime output, and JSON dispositions remain under `/private/tmp/swift-study-guide-revision-20261028`. Only Markdown is added or edited in the repository. The documented script commands use top-level async code rather than `@main`/`-parse-as-library`; the chapters explicitly state top-level main-actor isolation and the module's separate nonisolated default.

### Editorial checks and limits

Checked all required sections, local code/explanation coverage for every pitfall and answer, preserved frontmatter, balanced fences, absence of template placeholders, internal guide/heading links, and whitespace. Read the chapters in learning order, checked each exercise requirement against its solution, and compared cancellation/error claims across D1–D3 and the selected SDK. Official sources include SE-0296, SE-0304, SE-0311, SE-0317, SE-0329, SE-0381, SE-0461, SE-0493, SE-0504, current TaskGroup source, and Apple's structured-concurrency/forward-progress sessions.

Execution evidence is limited to arm64 macOS 27. No iOS/device run, older-runtime execution, live network request, real system-sleep/clock adjustment, UI integration, scheduling fairness, hard timeout latency, app-wide resource cap, or performance measurement is established. Numeric priority values and relative task starts remain intentionally unspecified. The earlier nested cancellation-handler/shield observation is retained with its original scope and was not rerun as part of this revision. A real blocking-read adapter still needs admission and legacy cancellation policy. The generic helper bounds its own active transforms and retains O(n) storage; it is not a streaming or app-wide limit.

## Remaining D guides — 2026-10-28

### Scope and editorial coverage

Applied the strengthened template to D4–D14 after the D1 reference review and D2–D3 revision. All **14 concurrency guides** now use topic-wide learning goals, stated prerequisites, small essential-mechanics examples, mistake/consequence/explanation/correction pitfall pairs, worked teaching examples, locally demonstrated interview answers, and focused exercises with observable acceptance checks. D4–D14 add **24 pitfall pairs and 45 interview answers**, including the duplicate-callback rubric probe and separately labelled specialist extensions. The other **56** guides are unchanged by this follow-up.

The rubric, topic IDs, filenames, frontmatter/tags, and D14's legacy alias are preserved. Existing specialist contracts were explained rather than silently dropped: shared token invalidation, explicit suppression and region transfer, async-defer reentrancy, executor identity, cancellation/registration races, external-channel semantics, try-lock guarantees, and cross-module defaults all remain covered. The first D1–D3 revision record above retains its original scope and counts.

| Guide | Essential expectations, questions, pitfalls, and exercise review |
| --- | --- |
| D4 | Actor access, atomic read-modify-write, isolated state versus logic races, snapshots, shared refresh identity, invalidation, late success/failure, waiter cancellation policy. The controlled invalidation driver rejects an old successful token after a newer token is cached. |
| D5 | Whole-type/member/global-actor ownership, explicit isolated conformance, allowed and rejected generic clients, off-actor CPU work, and success/error/cleanup freshness. A custom global actor example shares one domain across declarations. |
| D6 | Values/immutable classes, checked Mutex ownership, explicit Sendable capture rejection, disconnected handoff and later-alias rejection, retained sending-result rejection, reusable Sendable values, weak lifetime, direct versus inherited suppression, conditional/public/frozen inference, and manual unchecked audits. |
| D7 | Reentrant read/await/commit segments, freshness versus deduplication, repeated-input identity, stale success/error, and awaited-cleanup interleaving. Gates force replacement to finish before old work; async defer rechecks identity after its own suspension. |
| D8 | Snapshot-based nonisolated access, isolated parameter access, rejected live-state opt-out and multiple isolated parameters, synchronous placement versus concurrent execution, safe asserted context, and OS 27 executor identity/lifetime limits. |
| D9 | Synchronous completion, synchronized duplicate selection, remembered cancellation before registration, full noncopyable continuation state machine, before-entry cancellation, reentrant native cancellation, first-terminal-event wins, typed checked failure, noncopyable payload, and exhaustive optional value/error normalization. Dangerous checked-double-resume code is compiled but never executed. |
| D10 | Finite and throwing streams, newest/oldest drop policies, retained break versus explicit close, cancellation cleanup, termination callbacks, post-finish yields, and genuine channel dependency examples. Closed-channel send returns without delivery; explicit finish/cancel releases the early-exit producer before group completion. |
| D11 | Observed throwing handles, expected unused-task warning versus explicit discard, ordinary/detached task-local and lexical actor inheritance, cancelled-creator non-propagation, rejected detached UI mutation, current-task concurrent alternatives, and gated explicit cancellation/joining. |
| D12 | Whole synchronous invariants, rejected async lock closures, compiled-but-dangerous reentrancy/opposite lock ordering, callbacks after unlocking, synchronized immutable publication, retained-reference escape versus scalar snapshots, nested optional try-lock results, and unlock-on-throw. |
| D13 | Same-source Swift 5 warning versus Swift 6 rejection, unsafe unchecked promises versus owned actor repair, actual async consumers, new compiler warnings versus language-mode staging, and separately compiled immutable LegacyKit clients with ordinary/preconcurrency/snapshot boundaries. |
| D14 | Separate declaration defaults and async defaults, actor/no-actor calls, explicit execution contracts, inferred isolated versus genuine nonisolated witnesses, separately compiled DefaultsOff/DefaultsOn libraries, allowed/rejected clients, and all four default-isolation/caller-default configurations. |

Every included pitfall was reviewed for mistaken code, an observable consequence, its cause, corrected code, and the restored guarantee. Every interview answer has a direct answer, local code, walkthrough, and boundary; a cross-link alone does not satisfy the example requirement. Worked examples precede assessment. Gates and other support types are introduced before use, and expected nondeterministic results are described without assuming task FIFO order or a physical thread.

### Compiler, runtime, and configuration checks

The compiler remains Apple Swift **6.4**, `swiftlang-6.4.0.34.1`, with the installed Xcode 27 macOS SDK, an explicit `arm64-apple-macos27.0` target, and the baseline flags recorded above. Ordinary accepted examples were checked with warnings as errors. Explicit warning examples retain their intended warnings instead of being treated as accidental failures. TaskLocal macro cases use the same environment-specific supported `-disable-sandbox` compiler option documented for the earlier revision.

Function-only and intentionally rejected examples were **object-compiled**, not merely type-checked: region-transfer and noncopyable ownership diagnostics can be emitted during SIL generation. Runnable examples were linked and executed in separate processes with a 15-second timeout. Published blocks were extracted byte-for-byte with only their explicitly named shared declarations. Potentially deadlocking, trapping, or concurrently unsafe counterexamples were not executed. Compiler output passed through xcsift with raw logs and the pipeline status retained.

| Guide | Published Swift blocks checked | Runnable examples passed | Intended compiler rejections |
| --- | --- | --- | --- |
| D4 | 13 | 5 | 0 |
| D5 | 14 | 6 | 2 |
| D6 | 23 | 10 | 7 |
| D7 | 12 | 3 | 0 |
| D8 | 14 | 8 | 2 |
| D9 | 18 | 10 | 2 |
| D10 | 15 | 12 | 0 |
| D11 | 13 | 9 | 1 |
| D12 | 14 | 9 | 1 |
| D13 | 13 | 3 | 2 |
| D14 | 19 | 12 | 1 |
| D4–D14 total | **168** | **87** | **18** |

Eleven additional configuration controls bring this follow-up to **179 compiler cases**, **91 successful runtime cases**, and **23 intended rejections**. Four runtime controls recompile D14's published exercise under nonisolated/MainActor declaration defaults with caller-isolated async defaults enabled/omitted. Four declaration controls independently check that an ordinary class's mutable property becomes inaccessible to an explicitly nonisolated reader only under MainActor defaults. Three further rejections check ImmutableWeakCaptures reassignment, unguarded OS 27 executor-identity use targeting macOS 26, and Mutex use targeting macOS 14. The weak-capture source compiles under the baseline before being rejected under its separate feature flag.

D13 compares Swift 5 complete-checking warnings and Swift 6 errors with the same compiler, SDK, source, and other baseline options. Its LegacyKit module is separately compiled in Swift 5 mode; Swift 6 clients verify the missing imported conformance, narrowly annotated compatibility import, and independent snapshot alternative. The immutable fixture is a documented audit case, not permission to suppress diagnostics for arbitrary mutable legacy objects.

D14 builds the same public Contracts source into DefaultsOff and DefaultsOn, omitting/enabling NonisolatedNonsendingByDefault respectively. Both ordinary baseline and caller-default-disabled clients keep observing the callee's compiled contract. Explicit caller-bound and concurrent functions remain stable. The conformance examples separately verify inferred MainActor witnesses, rejected off-actor use, and a genuinely nonisolated immutable witness.

### External channel dependency

D10's two AsyncChannel clients were compiled and run against the real **Swift Async Algorithms 1.0.4** sources, commit `042e1c4d9d19748c9c228f8d4ebc97bb1e339b0b`, with **Swift Collections 1.1.4**, commit `671108c96644956dddcd89dd59c203dcdb36cec7`. Both repositories were fetched only into the temporary validation directory. SWIFTCI_USE_LOCAL_DEPS selected the local Collections checkout, avoiding an unrelated documentation-plugin fetch. Module, package, configuration, and security caches were also kept in temporary directories.

The package target was built using its own declared Swift 5 language mode and supported availability settings. That dependency build emitted **five warnings** under this newer compiler; the two guide clients themselves compile in Swift 6 mode with the baseline and warnings as errors, and both runtime checks pass. This is not a claim that the entire pinned package is warning-free or migrated to Swift 6 mode. The published executable-package manifest was object-compiled against the installed PackageDescription API; channel clients were linked against the actual built package modules/objects. A separate full resolution/build of that published remote-dependency client package was not repeated.

### Evidence and limits

Per-guide manifests and result JSON, compiler commands, source hashes, raw/xcsift logs, runtime output, configuration controls, package checkouts/build outputs, and editorial results remain under `/private/tmp/swift-study-guide-revision-20261028`. Only Markdown is changed in the repository; no package, binary, source fixture, permanent test harness, or dependency is added to it.

Final editorial checks cover every required section, all local pitfall/answer examples, shared-setup assembly, original frontmatter and aliases, balanced fences, absent placeholders, incoming/outgoing guide and heading links, and whitespace. Compiler results are matched against the final published block hashes. The official Swift/Apple sources and selected SDK declarations were reviewed for isolation, sending, continuation ownership, channel behavior, and the try-lock guarantee; SE-0512's no-spurious-failure status remains Swift 6.0 rather than being presented as a new 6.4 API.

Runtime evidence is **arm64 macOS 27 only**. Older targets are selective availability compilation controls, not runtime coverage; there is no iOS/device, live HTTP/UI, real system-sleep, performance, scheduling fairness, broad native teardown, or cross-platform validation. A one-shot gate does not repair a source that silently never completes, and first-result races still join cooperative cleanup. Compiler-accepted reference escape from Mutex is demonstrated without concurrent unsafe mutation and remains an encapsulation warning. The optional future/legacy integration policies described in the chapters require their own product-specific evidence.


## A guides — 2026-10-28

### Scope and teaching review

Revised **A1–A16** against the strengthened template, using the D guides' teaching standard. README now records A1–A16 and D1–D14 as complete; the **40 guides in B, C, E, and F** still await this revision. The template and rubric did not need another change. Topic filenames, IDs, frontmatter, tags, and aliases are unchanged. Historical evidence above remains scoped to its original revision.

Each chapter now introduces its concrete problem and terms, teaches small mechanics before combinations, and places a worked example before assessment. All **37 pitfall pairs** include mistaken code, consequence, cause, correction, and the restored guarantee. All **65 interview answers**, including extensions, have a direct answer, local code, walkthrough, and boundary. Broad comparisons are demonstrated through multiple cases; A3 combines the rubric's stored-read, whole-self-access, and observer questions in one answer. Named shared setups have explicit assembly instructions. Exercises check relevant invalid, empty, or boundary inputs rather than only the happy path.

The review checked coverage and the explanation of cause and effect, separately from counting headings or code fences:

| Guide | Essential expectations, questions, pitfalls, and exercise evidence |
| --- | --- |
| A1 | Value/reference assignment; array sequence versus object mutation; extracted array values; equality, object identity, and domain IDs; shared closure storage; noncopyable wrapper aliases; separate isolation/Sendable contract. The draft exercise checks empty/nonempty arrays, shared payload edits, reference replacement, and separate payloads. |
| A2 | let/var receivers; mutating self and reset; reference-backed nonmutating setter versus stored-value mutate access; weak let and ARC zeroing; synthesized equality/coding semantics and checked Sendable constraints; borrow/mutate accessors with noncopyable storage. Immutable writes reject. |
| A3 | Direct initialized-field reads versus premature computed self access; direct initialization versus observer-triggering helper writes; own-storage/super/customization order; designated, convenience, required, and failable paths. The exercise validates subclass construction and invalid input; Swift 6.4 private-cache synthesis is qualified by access and compatibility controls. |
| A4 | Optional cases and binding; skipped chaining arguments/assignment work; lazy defaults; nested map versus flatMap and try? flattening; dictionary absent/stored-nil/present states; IUO inference and forcing. Extensions check optional some/any grouping, missing opaque inference, noncopyable take/output-map, and rejected noncopyable-input map. |
| A5 | First-match order, payload bindings and guarded remainder; optional filtering, tuples and grouped patterns; individual where clauses and unconditional fallthrough. Separate SDKStates clients check future-case rejection, omitted-known-case warning/fallback, and plain default. The exercise tests 199/200/299/300, failure, and cancellation. |
| A6 | Reached registrations, reverse order, lexical nested exits, return-expression values versus returned references, early exit before registration, and cooperative cancellation. Async defer finishes before return/catch; shielding is separate. The exercise checks return, throw/catch, and skipped registration without timing assumptions. |
| A7 | Labels, domain inputs, variadics versus collections, multiple trailing closures, full-name overload selection, function-value effects/default loss, and default selection versus virtual dispatch. Inout tests evaluated input, overlapping aliases, escaping capture, independent async local, and rejected actor storage. Borrowing/consuming and MainActor defaults are explicit. |
| A8 | Shared captures and value/reference capture lists; nonescaping, optional, and escaping callback lifetime; immediate delivery; strong stored/bound-method/nested edges; weak/unowned lifetimes and implicit self. Gates establish early versus late promotion across await and cancellation. Separate weak-capture configurations and autoclosure evaluation are checked; exercise callbacks work before and after owner release. |
| A9 | LoadState preserves initial absence, refresh content, failure content, empty success, and original snapshots. Payload validation and authorized transitions are separate. Request identities reject stale completions/timers/acknowledgments. Recursive layout/overflow and noncopyable enum borrowing/destruction are checked; the retained lifecycle tests retry budget, timer retirement, cancellation reason, reset, and cross-model tokens. |
| A10 | Constrained helpers versus conditional conformance; type/protocol ownership and package exception; typealias identity; same-file synthesis versus manual witnesses; imported Codable synthesis rejection and explicit manual alternative. Real modules demonstrate runtime conformance discovery, rejected static generic use without its import, and ambiguous ordinary extension members. Owned mapping and versioned DTO tests cover boundaries, invalid formats, and ordinary external construction. |
| A11 | Private/fileprivate/internal/package/public/open permissions; type naming versus construction; getter versus setter; public protocol witnesses; same-file versus other-file private access; ordinary/sibling/testable clients and customization. The exercise rejects private reads and setter writes. Memberwise compatibility and source access versus ABI/body export remain distinct. |
| A12 | Compiler, effective language mode, module/SDK visibility, and runtime requirements are separate. macOS 12-target compilation checks guarded ContinuousClock use and rejects canImport-only use. anyAppleOS alignment/platform overrides, Apple source selection, equivalent deprecation adapters, consumer diagnostics, and elapsed-seconds fallback are explicit. |
| A13 | Sequential throws versus retained Result, closure-derived rethrows versus independent failure, loss through try?, and asserted try! behavior; typed catch analysis versus exhaustive error switch; throws(Never). Async Result preserves explicit typed error, captures cancellation without clearing it, can succeed when cancellation is ignored, and carries noncopyable success. The exercise retains malformed/empty input. |
| A14 | Grouping and policy, checked/wrapping/reporting arithmetic, eager operands versus autoclosure short-circuiting, predicate construction versus matching, unit/failure visibility, literal context versus String conversion/interpolation, and nonthrowing/nonfailable witnesses. Swift 6.4 collection trailing closures require actual initializers. The exercise verifies the eager/needed event sequence. |
| A15 | Independent drafts, shared class/actor identity, enum alternatives, and protocol capabilities; exclusive noncopyable cleanup; MainActor, actor, and synchronous Mutex call contracts; Sendable checked separately. Controlled completion order demonstrates independent display/cache freshness policies. The exercise checks snapshot/commit independence; stub service and basic cache clients use no live network. |
| A16 | Whole/prefix/search consumption, exact ASCII/horizontal-whitespace grammar, equivalent literal/builder corpus, conversion overflow, broader Unicode classes, and matching error suppression versus throwing APIs. Map/compactMap/Sequence flatMap/reduce(into:) and repeated lazy traversal have distinct outcomes. The importer checks row order/rejections, leap-year exceptions, invalid fields, size limits, and per-user aggregate overflow. |

Advanced ownership, schema, service, lifecycle, and importer material remains explicitly optional after the core teaching. The A16 literal and builder now implement the rubric's same `user_id:` grammar, including only the stated ASCII spaces/tabs; the stricter and broader examples explain exactly why acceptance differs. Internal guide/heading links were checked in both directions, including the retained A9 lifecycle anchor. Official references remain consolidated; two malformed inherited Apple documentation links were repaired.

### Compiler and runtime checks

Apple Swift **6.4**, `swiftlang-6.4.0.34.1` (`clang-2100.3.34.1`), from `/Applications/Xcode_27.app/Contents/Developer`. The SDK is macOS **27.0**; the arm64 host is macOS **27.0.1**, build **26A434**. Unless an example declares an override, compilation uses:

```sh
-swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -target arm64-apple-macos27.0 -warnings-as-errors
```

The compiler and SDK paths, temporary module cache, complete arguments, source SHA-256, compiler status/diagnostics, and runtime output are recorded per case. Each published Swift fence was extracted from the final Markdown and assembled with only its named dependencies. Function-only, unsafe-to-run, and rejected examples were **object-compiled** with `-c`, including ownership/SIL diagnostics. Executable examples were linked and run in separate processes with a 15-second timeout. Potentially trapping or concurrently unsafe paths were not executed. Compiler output passed through xcsift with the pipeline status and raw output retained.

| Guide | Published Swift blocks checked | Runnable examples passed | Intended compiler rejections |
| --- | --- | --- | --- |
| A1 | 13 | 11 | 2 |
| A2 | 15 | 13 | 2 |
| A3 | 16 | 12 | 3 |
| A4 | 17 | 13 | 3 |
| A5 | 15 | 12 | 2 |
| A6 | 12 | 12 | 0 |
| A7 | 17 | 12 | 4 |
| A8 | 18 | 15 | 2 |
| A9 | 17 | 14 | 2 |
| A10 | 21 | 9 | 4 |
| A11 | 25 | 10 | 10 |
| A12 | 12 | 8 | 2 |
| A13 | 16 | 13 | 3 |
| A14 | 18 | 12 | 5 |
| A15 | 18 | 14 | 2 |
| A16 | 17 | 14 | 0 |
| A1–A16 total | **267** | **194** | **46** |

The other **27 published blocks** are accepted object-compilation cases, such as shared declarations, dependency modules, or deliberately unexecuted trap examples. Five published runnable examples explicitly retain warnings: A5's grouped-pattern and omitted-known-case examples, plus three A12 deprecation comparisons. Ordinary accepted cases use warnings as errors. A5's mistaken grouped pattern retains the single-line form that exposes the selected compiler's where warning; changing its formatting can acknowledge the pattern grouping without correcting the underlying acceptance bug.

### Additional configuration and boundary controls

**22 additional compiler cases** and one run of an unchanged executable produce **9 additional successful runtime cases** and **7 additional intended rejections**. This revision therefore records **289 compiler cases**, **203 runtime cases**, and **53 intended compiler rejections** in total. The run of an already compiled executable is not counted as another compiler case.

- **A7 library replacement:** Built Defaults v1 and a client, replaced only a compatible resilient dylib with v2, then rebuilt the client. The v1 client prints `2,2,2`; the byte-for-byte unchanged client with v2 prints `2,5,5`; the rebuilt client prints `5,5,5`. The library uses `-enable-library-evolution -emit-library -emit-module`, a stable `@rpath/libDefaults.dylib` install name, and a client rpath. The unchanged executable's hash is retained. This distinguishes emitted literal defaults from calls to replacement library getters/helpers.
- **A3/A4/A14 constraints:** An unbound memberwise initializer reference selects the Swift 6.4 shorter signature; a colliding extension initializer rejects. Optional.ref/mutableRef/put remain absent in this SDK. An array value cannot accept a closure without callAsFunction merely because the syntax now parses.
- **A5 warning policy:** The omitted-known-case client still runs its fallback and emits the warning with `-warnings-as-errors` on this compiler. Exit status alone is insufficient for that diagnostic; B11's previously recorded limitation is reconfirmed.
- **A10 ownership:** A separately compiled vendor with package identity Geography and a same-package conformance compiles without the retroactive warning. A different package identity warns; unacknowledged Date conformance warns; the fully qualified acknowledgment runs successfully. The published runtime-discovery client links the bridge metadata without importing its conformance for static checking.
- **A11 source and binary access:** Same-package non-open subclassing and outside-package type access reject. A no-argument default initializer for all-private initialized storage works, while the memberwise argument remains unavailable. An ExportCore interface exposes the inlinable next body and ABI-visible helper declaration, but not the exported libraryPolicy body. An ordinary client calls the public API; direct internal/private helper calls reject.
- **A12 configuration:** With the same Swift 6.4 compiler and source, Swift 5 language mode reports `Swift 6 mode: false`. Compiler version and effective language mode remain independent. Older-target availability checks are compilation evidence, not older-runtime execution.

### Evidence and limits

Fixtures, manifests, per-block source hashes and results, exact commands, raw/xcsift output, control sources and binaries, module interfaces, formatting results, and editorial results remain under `/private/tmp/swift-study-a-guides-20261028`. Only Markdown is changed in the repository. Final checks compare compiler records to the published block hashes, preserve original frontmatter, require all template sections and local pitfall/answer examples, validate incoming/outgoing wiki links and heading destinations, and check whitespace. Code formatting preserves intentionally rejected syntax and the diagnostic-sensitive grouped-pattern form.

Runtime evidence is **arm64 macOS 27.0.1 only**. Selective macOS 12/13 deployment-target compilation does not establish fallback execution on those systems. No iOS/device, live HTTP/UI, performance/allocation, scheduling-fairness, adversarial recursion-depth, or full cross-platform/compiler matrix is claimed. The service tests exercise stubs; cancellation gates establish controlled ordering, not a production cancellation-aware primitive. The lifecycle model checks transitions and commands rather than executing network/timer effects. Historical broader fixtures retain their original scope; the D examples and unrelated topic groups were not rerun for these A-only edits.
