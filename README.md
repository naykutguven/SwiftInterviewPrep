# SwiftInterviewPrep

For people who Swift, or want to Swift.

This repo is a growing set of Swift interview-prep and deep-dive notes aimed mostly at senior and staff-level iOS/macOS engineers. It is opinionated, practical, and focused on the stuff that tends to matter in real code reviews, real production bugs, and real interview loops.

By no means is this an always-correct, always-complete document set. I try to keep it up to date with modern Swift changes, but it may be imperfect, incomplete, or a little behind the language in places. Check claims against official Swift and Apple documentation; if compiler behavior differs, record the exact environment and investigate. An accepted evolution proposal alone does not establish that an API is available in the selected SDK.

## Swift 6.4 update

All 70 topic guides were revised on 2026-10-16 using the approved template and revised senior/staff rubric. The [guide revision validation](<./Guide Revision Validation.md>) records settings, compiler/runtime evidence, and remaining limits.

Unless an example explicitly compares other configurations, the shared baseline is:

| Setting | Baseline |
| --- | --- |
| Compiler | Swift 6.4 |
| Language mode | Swift 6 |
| Approachable Concurrency | Enabled |
| Default Actor Isolation | `nonisolated` |

In Swift 6 mode, the additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See the rubric's [shared example baseline](<./Swift Senior & Staff Rubric and Prioritized Study Checklist.md#shared-example-baseline>) for equivalent flags. Approachable Concurrency and default actor isolation are separate settings; imported modules retain their own isolation contracts. Read each example's setup for dependencies, availability, and overrides.

iOS/macOS remain central; other platforms receive brief architectural coverage. Validation used temporary fixtures, with runtime checks on arm64 macOS 27 and selected Apple deployment-target compilation. The current validation record identifies unchecked configurations and unresolved SwiftSyntax package integration. This repository remains Markdown-only.

For Swift 6.4 feature orientation, use the [migration guide](<./Swift 6.4 What's New and Migration Guide.md>). The [update roadmap](<./Swift 6.4 Update Roadmap.md>) and [September validation record](<./Swift 6.4 Validation Record.md>) retain the earlier review's scope and settings; those results do not establish behavior under the revised baseline.

## What's in here 📚

- 70 topic notes across Swift semantics, API design, memory, concurrency, interop, tooling, and testing.
- A top-level rubric/checklist for prioritizing what to study for senior/staff interview prep.
- Consistent note structure so you can move through topics without re-learning the format each time.
- Source-backed notes with cross-links between related topics.

Think of this repo as a study map, not a sacred text.

## Repo layout 🗂️

- [Swift Senior & Staff Rubric and Prioritized Study Checklist](<./Swift Senior & Staff Rubric and Prioritized Study Checklist.md>)
  Start here if you want the high-level map.

- [A. Core Semantics, Syntax and Control Flow](<./A. Core Semantics, Syntax and Control Flow/>)
  16 notes on value/reference semantics, initialization, optionals, pattern matching, error handling, type choices, and more.

- [B. Type System, Abstraction, and API Design](<./B. Type System, Abstraction, and API Design/>)
  14 notes on protocols, generics, existentials, dispatch, `Codable`, API design, property wrappers, and result builders.

- [C. Memory, Ownership, Performance and Safety](<./C. Memory, Ownership, Performance and Safety/>)
  12 notes on ARC, copy-on-write, exclusivity, Unicode correctness, unsafe memory, ownership features, and performance tradeoffs.

- [D. Concurrency, Isolation, and Correctness](<./D. Concurrency, Isolation, and Correctness/>)
  14 notes on structured concurrency, actors, `Sendable`, reentrancy, continuations, `AsyncSequence`, synchronization, and Swift 6.x concurrency migration.

- [E. Interop, Modules, and Distribution](<./E. Interop, Modules, and Distribution/>)
  8 notes on Objective-C, C, C++, SwiftPM, module boundaries, resilience, ABI, and binary distribution concerns.

- [F. Tooling, Testing, Diagnostics, and Language-Adjacent Features](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/>)
  6 notes on testing, LLDB, performance investigation, macros, build settings, and observation-related topics.

## What each guide includes 🔍

The [approved answer template](<./Swift Section Answer Template Revised.md>) gives each guide a consistent structure:

- One observable learning goal and explicit example settings.
- A mental model explaining the mechanism, consequences, and limitations.
- Rules with meaningful pitfalls and the correct approach.
- Direct answers to the topic's essential rubric questions.
- A focused rubric exercise with acceptance checks, a solution, and reasoning.
- Practical decisions, investigation methods, and evidence of deeper judgment.
- Related guides and official sources, also cited beside relevant claims.

Worked examples appear when they add a distinct lesson. Substantial specialist material lives in clearly marked optional subsections.

That structure is intentional. The goal is not just "know the feature exists," but "explain it, reason about it, and use it without hurting yourself or your teammates."

## How to use this repo 🧭

- If you want a starting point, begin with the [study checklist](<./Swift Senior & Staff Rubric and Prioritized Study Checklist.md>).
- Establish the settings with F5 and D14, then work through Tier 1's stated foundations. Choose a role track for Tier 2 and add Tier 3 specialist topics when the work requires them. Tiers describe study order; assessment depth depends on the role and selected task.
- Mark a topic complete when you can explain its rule, solve its focused exercise, and check a relevant failure case without a substantive hint. Use the rubric's observable score anchors to assess depth.
- If you already write Swift every day, use the notes to pressure-test your mental models, especially around dispatch, ownership, concurrency, and module design.
- If a topic is version-sensitive, verify it against current Swift release notes, proposals, and docs before treating it as gospel.

The guides use wiki-style links such as `[[D14 — Approachable Concurrency and Default Isolation]]`. Obsidian resolves these links, including heading references. On GitHub, use this README's Markdown links and the topic folders to navigate.

## Good places to start 🚀

- [Swift Senior & Staff Rubric and Prioritized Study Checklist](<./Swift Senior & Staff Rubric and Prioritized Study Checklist.md>)
- [F5 — Language mode, build settings, and migration flags](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F5 — Language mode, build settings, and migration flags.md>)
- [D14 — Approachable Concurrency and Default Isolation](<./D. Concurrency, Isolation, and Correctness/D14 — Approachable Concurrency and Default Isolation.md>)
- [A1 — Value semantics, reference semantics, identity, and copy-on-write](<./A. Core Semantics, Syntax and Control Flow/A1 — Value semantics, reference semantics, identity, and copy-on-write.md>)
- [B10 — API Design Guidelines and Swift-Native Surface Design](<./B. Type System, Abstraction, and API Design/B10 — API Design Guidelines and Swift-Native Surface Design.md>)
- [E5 — Modules, packages, targets, products, and resources](<./E. Interop, Modules, and Distribution/E5 — Modules, packages, targets, products, and resources.md>)
- [F1 — Testing Strategy in Modern Swift](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F1 — Testing Strategy in Modern Swift.md>)

## If you spot something off 🔧

That is normal.

Swift changes. Apple docs change. My understanding changes. Some notes will age better than others.

If you notice something incorrect, unclear, outdated, or badly explained, fix it, rewrite it, or at least leave yourself a note to come back. The best version of this repo is the one that keeps getting sharpened.

## License 📄

See [LICENSE](./LICENSE).
