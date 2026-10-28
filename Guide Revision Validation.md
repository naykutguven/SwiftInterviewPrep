---
tags:
  - swift
  - interview-prep
  - validation
  - swift6-4
---

# Guide Revision Validation

The October 16 record below describes that historical revision. The [initial October 28 teaching revision](#teaching-revision--2026-10-28) updates the template, README, and D1–D3. The [remaining D-guide revision](#remaining-d-guides--2026-10-28) extends the same explanation and example requirements to D4–D14. The [A-guide revision](#a-guides--2026-10-28) applies that standard to A1–A16, the [B-guide revision](#b-guides--2026-10-28) applies it to B1–B14, and the [C-guide revision](#c-guides--2026-10-28) applies it to C1–C12. The other 14 guides in E and F are still awaiting review against the new standard.

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

## B guides — 2026-10-28

### Scope and teaching review

Revised **B1–B14** against the strengthened template and the D/A teaching standard. README now records **A1–A16, B1–B14, and D1–D14** as complete; the **26 guides in C, E, and F** still await this revision. The template and rubric did not need another change. Topic IDs, filenames, frontmatter, tags, and aliases are unchanged. The earlier October 16, D, and A evidence above retains its original scope.

The chapters now introduce a concrete problem and define the terms used to explain it. Small mechanics precede combinations, worked examples precede assessment, and extensions retain their labels. All **31 pitfall pairs** include mistaken code, consequence, cause, correction, and an explanation of the restored behavior. All **72 interview answers**, including extensions, include a direct answer, local code, an outcome walkthrough, and a material boundary. Named shared declarations have assembly instructions; separate-module examples identify the library, ordinary client, and required settings.

The review mapped the rubric to teaching material and checked the explanation of cause and effect separately from code/heading counts:

| Guide | Essential expectations, questions, pitfalls, and exercise evidence |
| --- | --- |
| B1 | Associated input/output and Self relationships; existential formation versus callable members; generic, constrained-existential, and conforming-erased consumers; composition versus explicit nominal refinement; readable versus immutable state; isolated witnesses versus Sendable. The lookup checks present/missing String keys and rejects an Int key. The extension separates noncopyable protocol/payload/use-site defaults from Hashable, Set, and CustomStringConvertible support. |
| B2 | Caller-selected generics and parameter some; result opaque identity and generic families; runtime existential choice; opening, invariant results, independent inputs, and dynamic subclasses; Sequence element constraints; optional spelling/inference; AnyView, any View, and builder structure. The exercise demonstrates generic, existential, and one-enum opaque alternatives without a performance claim. |
| B3 | Capability and same-element proofs; distinct source/destination containers; conditional conformance; equal types versus runtime values, lengths, and indices; Copyable/Escapable defaults; primary/non-primary associated types; generalized extensions; literal inference; Iterable versus Sequence. Array, slice, empty, and fixed String/Int mismatch checks cover the append exercise. |
| B4 | Concrete/generic/existential/superclass call surfaces; requirement witnesses versus extension-only helpers; inherited defaults versus overridable base methods; constrained lookup versus unconditional/conditional conformance; isolation permission; selector exposure, KVO interception/invalidation, and native dynamic. Both versions of the f/g exercise run; SIL and optimized controls are scoped separately. |
| B5 | Root/Value relationships; read, writable, and reference-writable capability; struct-root reference writes; literal-to-function conversion versus stored paths; partial/full erasure; mutable getters and stable/canonical sorting; subscript captures and Sendable; appending overload losses; actor and noncopyable limits. Projection/write exercises include read-only and immutable-root rejections. |
| B6 | Any/AnyObject and explicit bridging; concrete/generalized/protocol metatypes; generic container versus opened type(of:); supplied metatype construction; private registry erasure, actor ownership, missing/duplicate handling, fresh/shared lifetime, and metadata versus instance sendability. Reflection, module selectors, guarded demangling, bounded output, and durable identity remain distinct. The exercise fixes the base-typed metatype and rejects an unrelated factory result. |
| B7 | Structural equality and hashing versus entity identity; conditional synthesis; CodingKeys and excluded/default/computed fields; CaseIterable eligibility/order; Sendable checking and nonpublic value inference. The mismatched synthesized hash is never used at runtime. Noncopyable struct synthesis, rejected enum synthesis, manual borrowed enum witnesses, and consumer constraints are checked separately. The exercise tests equality laws, content changes, and matching hash components. |
| B8 | Equality/hash laws and lawful collisions; unstable hashValue versus durable IDs; independent value copies versus reference-observed key mutation; stable membership versus retained content; insert/update/dictionary replacement. New Dictionary.Keys/CollectionOfOne/EmptyCollection conformances have explicit OS boundaries. The exercise inspects both membership and the retained snapshot. |
| B9 | Stable source/wire mappings and encoder strategies; independent key/enum renames; missing/null/type/enum/version policies; current-key precedence; safe rewriting versus dropped future fields; raw versus associated-value enum synthesis. The migration exercise checks both names, absence, null, and malformed input. A lossy-list extension consumes each source element once, preserves accepted order, reports rejected indices, and rejects a wrong root shape. |
| B10 | Labels, distinct domain types, validation, mutation names, honest task ownership/effects, explicit public construction, and outcome shape. Noncopyable versus copyable consuming, ordinary client isolation, ~Sendable versus sending, and scoped Ref/MutableRef access are demonstrated. The FollowCore client checks valid/empty inputs and normal/self-follow decisions without pretending to perform network I/O. |
| B11 | Source/ABI/module-interface/behavior/schema compatibility; resilient versus frozen representation; five enum configurations across module/package boundaries; callable symbols, client-visible bodies, mandatory inlining, and library specialization. Separate unchanged/rebuilt clients establish the stated replacement outcomes. The exercise includes an explicitly maintained v2 consumer. Current export-helper and warning-promotion limits remain qualified. |
| B12 | each/repeat, typed arity, projection, same-shape relationships, iteration, zero/scalar/heterogeneous results, eager arguments versus skipped comparisons, runtime zip truncation, and explicit task scheduling. Collections and meaningful overloads are compared. Nominal versus function/alias availability, overload ambiguity, Copyable defaults, and nominal restrictions are checked. The equality exercise rejects explicit type and arity mismatches. |
| B13 | Backing/wrapped/projected storage; direct backing initialization; composition order and projected value copies; initialization versus didSet; value-backed versus shared reference-backed copies; locked access versus whole-operation increment; memberwise access history; explicit UI actor ownership; copyable borrow/mutate accessors. The exercise checks high/low/interior initialization, mutation boundaries, and copy independence. |
| B14 | Expression/component/final-result types; block, branch, optional, loop, array-helper, and final hooks; explicit-return opt-out; eager component effects versus stored actions; partial-block ordering; typed pack construction; custom Array/Dictionary APIs versus parser support; closure-after-literal migration and public-hook compatibility. Undefined/mismatched helpers were corrected. Empty/optional/order checks match plain array construction. |

The final reading pass checked that core rules are explained before ownership/availability exceptions, that examples establish what their prose claims, and that broad comparisons show their meaningful cases. Invalid examples test the intended constraint rather than an unrelated parse/access failure. Incoming and outgoing wiki links and heading destinations are valid. Inline official sources have consolidated, deduplicated reference lists; two malformed inherited Apple documentation links were repaired.

### Compiler and runtime checks

Apple Swift **6.4**, `swiftlang-6.4.0.34.1` (`clang-2100.3.34.1`), from `/Applications/Xcode_27.app/Contents/Developer`. The SDK is macOS **27.0** and the arm64 host is macOS **27.0.1**, build **26A434**. Ordinary examples use:

```sh
-swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -target arm64-apple-macos27.0 -warnings-as-errors
```

Every published Swift fence was extracted from the final Markdown and assembled with only its named dependencies. Compiler paths, complete arguments, source SHA-256, statuses, diagnostics, and runtime output are retained per case. Function-only, intentionally unsafe, and rejected examples were **object-compiled** with `-c`, including ownership/SIL diagnostics. Runnable examples were linked and executed separately with 15-second timeouts. Invalid comparators, broken Hashable use, and stored-key invariant violations were not executed. Compiler output passed through xcsift with raw output and pipeline status retained.

| Guide | Published Swift blocks checked | Runnable examples passed | Intended compiler rejections |
| --- | --- | --- | --- |
| B1 | 20 | 12 | 8 |
| B2 | 23 | 12 | 11 |
| B3 | 22 | 13 | 9 |
| B4 | 18 | 17 | 1 |
| B5 | 27 | 14 | 12 |
| B6 | 22 | 16 | 4 |
| B7 | 19 | 13 | 5 |
| B8 | 13 | 11 | 0 |
| B9 | 13 | 13 | 0 |
| B10 | 20 | 12 | 5 |
| B11 | 23 | 8 | 5 |
| B12 | 25 | 16 | 9 |
| B13 | 19 | 14 | 4 |
| B14 | 20 | 15 | 5 |
| B1–B14 total | **284** | **186** | **78** |

The other **20 published blocks** are accepted object-compilation cases: shared declarations, separate library modules, and deliberately unexecuted examples. B14's explicit-return example retains its intended disabled-builder warning by compiling without -warnings-as-errors. B3's heterogeneous-literal inference probe also omits that flag; this compiler emits no warning for the checked call. Ordinary accepted examples keep warnings as errors. The final full extraction pass passed all 284 blocks.

### Additional configuration and boundary controls

**64 additional compiler cases** and three runs of unchanged executables add **18 successful runtime cases** and **13 intended rejections**. The B revision therefore records **348 compiler cases**, **204 runtime cases**, and **91 intended compiler rejections** overall. Running an already compiled executable is not counted as another compiler case.

- **Dispatch and derived behavior:** B4's SILGen shows witness_method for the generic requirement, a function_ref for its extension helper, and class_method for the superclass call. Optimized scripts preserve helper, inherited-witness, and constrained-default behavior. The concrete/generic/existential mechanics also run under MainActor declaration defaults as a separate configuration. B7's generated hash SIL includes both id and name despite custom ID-only equality; the invalid hash collection is never executed.
- **Runtime and collection availability:** B6's guarded demangling source compiles for macOS 26; an unguarded call rejects. The installed iOS 27 and simulator 27 SDKs accept the guarded fallback source and reject direct Runtime import because the module is absent. B8's unguarded new collection conformances reject for macOS 26, while Set(dictionary.keys) compiles for that target and runs on the OS 27 host.
- **API isolation:** FollowCore is rebuilt under MainActor declaration defaults. Its ordinary neutral client rejects the imported actor-isolated construction/call surface, confirming that the client's nonisolated defaults do not rewrite the library contract.
- **Enum matrix:** Five separately emitted DeliveryAPI variants compare plain nonresilient, plain resilient, frozen resilient, explicit nonexhaustive nonresilient, and nonexhaustive(warn) nonresilient enums. Separate Consumer versus DeliveryKit package identities establish the published external/same-package table. A warn enum still rejects a missing known case without fallback. An omitted known case plus @unknown default warns and executes fallback even with -warnings-as-errors in this compiler; ordinary default emits no such warning. This promotion limitation is not generalized to every warning.
- **Resilient enum replacement:** StatusSDK v1 and an external optimized client initially print ready. A compatible dylib replacement adds/returns paused; the byte-for-byte unchanged client prints future. Rebuilding unchanged source warns and prints future. The maintained consumer explicitly handles paused and prints paused.
- **Representation replacement:** RepresentationAPI changes nonfrozen Token's private String storage to private reference storage plus a generation value while preserving its construction/getter contract. The unchanged and rebuilt clients both print abc. Its interface hides nonfrozen private fields and exposes the frozen Point layout fields. No incompatible frozen-layout edit was attempted.
- **Definition visibility:** RevisionAPI v1 prints `1,1,1,1`. Replacing four literal bodies with 2 yields `2,1,1,1` from the unchanged optimized executable and `2,2,2,2` after rebuilding. The interface withholds libraryRevision's body and exposes clientRevision/visibleRevision/forcedRevision bodies. Demangled public symbols include libraryRevision, visibleRevision, and forcedRevision, with no library clientRevision entry. Client emission is therefore distinct from inlining.
- **Specialization and textual imports:** Optimized containsZero SIL compares the generic metatype with Array<Int>, branches to a concrete path, and retains the generic witness-based fallback. The object exports one generic containsZero entry; no separate specialized public ABI or speedup is claimed. Partial specialization and the misspelled attribute reject. Four isolated textual interfaces import successfully with this compiler after withholding their binary modules.
- **Pack deployment:** Unguarded nominal Route rejects for macOS 13 and iOS 16; its availability-protected declaration compiles, and unguarded declarations compile at macOS 14/iOS 17. The separate function/alias source compiles for the older targets. This is selective compilation, not execution on those systems.
- **Builder diagnostics:** A separately typed ordinary closure with the wrong [String] result rejects. The published direct TupleBuilder function exposes the missing optional-hook diagnostic. The fragile newline-after-array closure example preserves its original layout rather than allowing the formatter to conceal the parsing edge.

All dynamic replacements use -enable-library-evolution, -emit-library/-emit-module, stable @rpath install names, and client rpaths. The three original executable hashes match after replacement. The replacement sources, exact commands, original interfaces, symbol output, SIL, and observations are retained separately from ordinary object-linked examples.

### Evidence and limits

Fixtures, manifests, exact commands, per-block hashes/results, raw/xcsift logs, configuration controls, dylibs/clients, interfaces, symbol/SIL observations, formatting results, and editorial checks remain under `/private/tmp/swift-study-b-guides-20261028`. Only Markdown is changed in the repository. Final checks preserve original frontmatter, require all template sections and local pitfall/answer examples, compare published source hashes with compiler records, check internal guide/heading links, and check whitespace. The formatter preserves the deliberately invalid B2 composition syntax and B14's diagnostic-sensitive newline example.

Historical compiler limitations remain explicit: inferred noncopyable key-path roots and one noncopyable generic wrapper caused compiler failures; those crash probes were not rerun or counted as intended rejections. Some invalid direct same-type/generic-builder expressions failed to yield useful diagnostics in the earlier validation; fixed function-type/ordinary-closure/direct-builder probes expose the intended errors. Current @export(interface) helper eligibility still does not replace the working @usableFromInline pattern for an exposed @inlinable caller. Proposal text alone is not treated as SDK/compiler support.

Runtime evidence is **arm64 macOS 27.0.1 only**. iOS/device execution, older-runtime behavior, rendered SwiftUI, live services, production registry lifecycle, performance/allocation measurements, scheduling fairness, an older-compiler textual-interface matrix, and full binary-framework distribution remain outside these checks. The SwiftUI examples test construction/type behavior; the KVO example tests a synchronous local observation and invalidation. Library replacement demonstrates the specific preserved contracts, not arbitrary API/layout compatibility. Previously revised A/D examples and unrelated C/E/F guides were not rerun for these B-only edits.

## C guides — 2026-10-28

### Scope and teaching review

Revised **C1–C12** against the strengthened template and the D/A/B teaching standard. README now records **A1–A16, B1–B14, C1–C12, and D1–D14** as complete; **14 guides in E and F** remain. The template and rubric did not need another change. Topic IDs, filenames, frontmatter, tags, and aliases are unchanged. Earlier validation sections retain their historical scope and counts.

Concrete problems and definitions now precede the advanced ownership, lifetime, and compiler combinations. Every chapter includes learning goals, prerequisites, small mechanics, a worked example, and a focused exercise with observable acceptance checks. All **27 pitfalls** include a mistake, consequence, cause, correction, and explanation of the restored behavior. All **49 interview answers**, including extensions, include a direct answer, local code, a walkthrough, and a meaningful boundary. C9's inspection pitfall uses actual SIL commands as its correction, as the template permits for tooling. Shared declarations identify their assembly order, and deliberately rejected or unexecuted code is labelled at its use site.

The coverage review maps the essential rubric expectations and questions to teaching material, and each exercise requirement to its solution and verification:

| Guide | Essential expectations, questions, pitfalls, and exercise evidence |
| --- | --- |
| C1 | Strong/weak/unowned graphs, bound methods, external registrations, explicit shutdown, isolated-deinit completion limits, and unique-owner class aliases. The cycle exercise invokes its callback, explicitly stops the graph, records each destruction once without ordering them, and defines later weak-trigger behavior. Bounded observations and withExtendedLifetime avoid assumptions about temporary strong reads or lexical scope. |
| C2 | Logical copies versus buffer/capacity costs, class payload mutation versus slot replacement, slice indices and invisible retention, outer CoW uniqueness versus an inner Array snapshot, and synchronization limits. Every custom mutation path starts from a fresh shared copy. The exercise checks nonzero/empty slice indices, independent source/owned-copy values, and rebasing. UniqueArray's different ownership contract follows the foundations. |
| C3 | Evaluated snapshots before inout access, computed get/set writeback, exclusive whole-variable access, static rejection versus dynamic conflict, pointer conversion limits, and noncopyable borrow/mutate/MutableRef access. The exercise rejects one variable passed twice and verifies independent-value and swapAt repairs. |
| C4 | Character/scalar/UTF-16/UTF-8 counts, canonical equality versus bytes, same-snapshot indices, whole-grapheme boundary policy, overflow-safe Foundation range validation, selection revisions, Character/byte budgets, and UTF8Span validation. Truncation checks include empty/exact boundaries, combining marks, flags, CRLF, joined emoji, and ellipsis accounting; rendered width remains separate. |
| C5 | Single-pass Sequence versus Collection, custom noninteger indices, instrumented forward traversal versus random-access index work, mutation versus identity, sorted binary search, and borrowing Iterable with typed failure. The search exercise covers nonzero slice indices, first duplicates, boundaries, absence, and empty input. Generic failure checking is distinguished from the concrete Never runtime examples. |
| C6 | Allocation, binding/assuming/rebinding, initialized prefixes, alignment, bounds, access lifetime, and scoped versus retained native addresses. Invalid cleanup, incompatible binding, and escaped reads remain unexecuted. Corrections verify destruction after success/throw, owned empty/nonempty checksums, Darwin's supported sockaddr prefix rebinding, and initialized temporary raw bytes. |
| C7 | Borrowing versus consuming, Copyable caller copies, explicit consume, all-path reinitialization, class aliases after wrapper transfer, owned getters versus borrowed/mutable projections, and Ref/MutableRef scope. The function-local getter reference observes its copied temporary while MutableRef updates the actual slot. The token exercise checks repeated borrowing, rejected later use, transfer, and valid reinitialization. |
| C8 | Suppression of owner/associated-type/use-site Copyable defaults, actual noncopyable protocol support, unique-owner transfer and clone behavior, Set/CustomStringConvertible limitations, and the installed Optional surface. The resource exercise transfers and closes exactly once while its independent class alias remains usable; it makes no native-descriptor adoption claim. Missing proposed reference accessors and RigidArray are explicit rejections. |
| C9 | Type relationships versus specialization, witness dispatch, representation, allocation, and measurement; concrete/opaque/generic/existential/enum/closure contracts; equivalent traversal and observable effect order; actual library/client boundaries; and reachable SIL inspection. The pipeline exercise tests empty/edge/all-UInt8 inputs and five clamp bounds through four contracts, checks original input, and states what a later benchmark must measure. |
| C10 | Effective optimization/assertion/source-condition settings, required work outside assert, recoverable preparation failure, algorithmic comparisons versus timings, mandatory inlining versus body visibility/specialization, and callback pointer-escape limits. The repair exercise verifies both preparation outcomes across six settings; no required work disappears with assertions. |
| C11 | Checker versus warning policy and actual correctness, narrow unsafe acknowledgment, unsafe declaration obligations, unverified @safe claims, and a complete C checksum boundary. The audit exercises initialized input, bounds, empty input, non-retention, synchronous lifetime, and the absence of native allocation/cleanup. Compile acceptance of an invalid count or escaped pointer is never treated as runtime safety evidence. |
| C12 | Owned dynamic/inline/unique storage versus borrowed Span/Ref views, fixed-count layout versus stack placement, initialized count versus capacity, borrowing iteration, cleanup on throw, wire fields versus Swift struct layout, and operation-specific availability. The parser exercise checks an unaligned final valid field, empty/short input, negative/Int.min/Int.max offsets, explicit endian order, and nil before invalid access. |

A final reading pass checked explanation order, named setup, broad comparisons, exercise boundaries, and cross-guide consistency. Invalid examples must expose the intended rule rather than an unrelated missing argument or access error. C1's optimized checks exposed both early release after a last use and temporary strong ownership from a weak read; its final probes now bound those effects explicitly. Official references are consolidated and deduplicated, including repaired inherited Apple links.

### Compiler and runtime checks

Apple Swift **6.4**, `swiftlang-6.4.0.34.1` (`clang-2100.3.34.1`), from `/Applications/Xcode_27.app/Contents/Developer`. The SDK is macOS **27.0**; the arm64 host is macOS **27.0.1**, build **26A434**. Ordinary examples use:

```sh
-swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -target arm64-apple-macos27.0 -warnings-as-errors
```

Every published Swift fence was extracted from the final Markdown and assembled with only its named dependencies. Exact compiler/SDK paths, complete arguments, source SHA-256, statuses, diagnostics, and runtime output are retained per case. Function-only, intentionally invalid-to-run, and rejected examples were **object-compiled** with -c, including ownership/SIL diagnostics. Runnable examples were linked and executed separately with 15-second timeouts. Compiler output passed through xcsift with raw output and the pipeline status retained. Diagnostic checks inspect error/warning lines so a matching word in an argument or source note cannot establish the intended rejection.

| Guide | Published Swift blocks checked | Runnable examples passed | Intended compiler rejections |
| --- | --- | --- | --- |
| C1 | 18 | 12 | 2 |
| C2 | 15 | 13 | 0 |
| C3 | 16 | 11 | 3 |
| C4 | 15 | 14 | 0 |
| C5 | 15 | 11 | 4 |
| C6 | 15 | 11 | 1 |
| C7 | 19 | 12 | 7 |
| C8 | 20 | 10 | 9 |
| C9 | 16 | 11 | 1 |
| C10 | 17 | 11 | 3 |
| C11 | 12 | 9 | 0 |
| C12 | 16 | 11 | 4 |
| C1–C12 total | **194** | **136** | **34** |

The other **24 published blocks** are accepted object-compilation cases, including shared declarations, the separate PixelCore module, and deliberately unexecuted examples. C11's unacknowledged native-call example uses strict checking without warnings-as-errors to demonstrate its expected warning; ordinary accepted cases retain warnings as errors. All 194 final published blocks pass their stated dispositions.

### Additional configuration and boundary controls

**74 additional Swift compiler cases** add **50 successful runtime cases** and **8 intended rejections**. Three separately compiled specified-trap cases execute in isolated child processes and terminate as expected. Overall, the C revision records **268 Swift compiler cases**, **186 successful runtime cases**, **42 intended compiler rejections**, and **3 expected runtime traps**. The native dependency adds **one C compilation**, counted separately.

- **Ownership and registration:** Every runnable C1 block also passes with -O. C2's mutation/retention, C7's getter temporary, C8's resource cleanup, C9's captured-object lifetime, and C12's temporary cleanup pass optimized controls. The actual PeriodicRefresh timer is manually fired on MainActor in Onone/O: replacement invalidates the old timer, stop is idempotent, invalidated timers cannot increment, and isolated owner destruction invalidates a retained registration. No sleeps prove event order. The read-only ImmutableWeakCaptures control accepts and observes later absence.
- **Availability and scope:** Unguarded UniqueBox/UniqueArray/Ref use rejects for macOS 26 and iOS 26; guarded declarations compile. The UIKit lifecycle adapter object-compiles with the iOS 27 SDK. Noncopyable borrow/mutate accessors compile for macOS 13. InlineArray indexing rejects for macOS 25 and compiles for 26. Basic Span indexing and temporary output spans compile under strict checking for macOS 13 and iOS 18; Array.span separately rejects for macOS 25. Unguarded endian loads reject for macOS 26, while the guarded parser compiles there and both its modern and explicit fallback operations run on the OS 27 host. Function-local getter Ref accepts, while the top-level getter-temporary lifetime rejects.
- **Unicode:** Optimized range/selection and display-truncation checks preserve the same snapshot, overflow, whole-grapheme, and budget policies. UTF8Span accepts valid decomposed UTF-8 and rejects a truncated sequence through its validating initializer. The Foundation conversion inside a combining Character is recorded as this target's observation rather than a portable selection-validation guarantee.
- **Equivalent work and SIL:** C9 emits SILGen/Onone, mandatory SIL/Onone, and optimized SIL/O from the published assembled source. Reachability follows function_ref targets: each initial/mandatory known or unknown path has one transform witness call; optimized known generic/known existential paths have none and contain the bounded subtraction, while the unknown path retains one. The separately emitted resilient PixelCore library retains a witness call in both its generic and existential paths. Its public interface omits those bodies, and the ordinary separately compiled client supplies its own conformer and checks equal output. These observations measure neither time nor allocation.
- **Assertions and inlining:** Onone/O/Osize/Ounchecked and explicit Release/Debug assertion configurations produce the documented assert/precondition evaluation counts. Required preparation and recoverable failure remain observable under all six. Onone/O alone leave DEBUG undefined; -D DEBUG enables the source condition. C10's complement call is present in SILGen and removed from the wrapper by mandatory SIL/Onone, O, and Osize, where subtraction is visible. B11's historical body-export/replacement evidence is referenced with its original scope rather than rerun or inferred from this wrapper.
- **Native audit:** The exact published C source/header/modulemap build with Apple Clang 21.0.0, clang-2100.3.34.2, C11, -Wall -Wextra -Werror, and matching explicit SDK/target. Swift clients link that object. The ordinary unacknowledged call runs; strict checking plus warning promotion rejects it. Unacknowledged broad-wrapper calls and unsafe declaration bodies also reject. Bounds/empty-input corrections execute; acknowledged out-of-bounds reads and @safe pointer escapes remain compile-only.
- **Specified traps:** Dynamic exclusivity and debug assertion traps report the expected failure; an optimized precondition traps too. These three child executions have ten-second timeouts and core dumps disabled. The optimized assertion case returns normally because its condition is omitted. Invalid pointers, incompatible bindings, uninitialized destruction, races, and unchecked overflow are never executed.

### Evidence and limits

Fixtures, extraction manifests, source hashes/results, exact commands, raw/xcsift output, native files/object, module/interface/client, SIL/reachability observations, configuration controls, formatting results, and editorial checks remain under `/private/tmp/swift-study-c-guides-20261028`. Only Markdown is changed in the repository. Final checks preserve original frontmatter, require every template section and local pitfall/answer example, compare published source hashes with compiler records, validate incoming/outgoing guide and heading links, and check whitespace. The native source/header/modulemap hashes also match their published blocks.

Sources reviewed include the Swift language book, installed SDK declarations, Apple API documentation, and relevant proposals for weak let, isolated deinitialization, ownership/noncopyability, borrowing accessors, Ref/MutableRef, UniqueBox/UniqueArray, Iterable, UTF8Span, strict memory safety, temporary output spans, safe raw loading, and mandatory inlining. A proposal's acceptance is not substituted for support in the selected SDK.

Execution evidence is **arm64 macOS 27.0.1 only**. Older deployment targets and iOS evidence are compilation checks. There is no new device/UI run, older-runtime execution, live native service, sanitizer/Instruments campaign, resident-memory/allocation measurement, throughput benchmark, custom failing Iterable runtime, or older-compiler matrix. The timer fixture tests synchronous manual firing on its declared executor, not real run-loop scheduling latency. The resource token models a cleanup contract rather than a real descriptor. Layout, weak-lifetime, traversal-count, and SIL observations retain their stated limits. Previously revised A/B/D examples and the remaining E/F guides were not rerun for these C-only edits.
