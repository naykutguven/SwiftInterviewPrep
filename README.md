# SwiftInterviewPrep

For people who Swift, or want to Swift.

This repo is a growing set of Swift interview-prep and deep-dive notes aimed mostly at senior and staff-level iOS/macOS engineers. It is opinionated, practical, and focused on the stuff that tends to matter in real code reviews, real production bugs, and real interview loops.

By no means is this an always-correct, always-complete document set. I try to keep it up to date with modern Swift changes, but it may be imperfect, incomplete, or a little behind the language in places. If something here disagrees with Swift.org, Apple docs, evolution proposals, or compiler behavior, trust the official source and the compiler.

## What's in here 📚

- 70 topic notes across Swift semantics, API design, memory, concurrency, interop, tooling, and testing.
- A top-level rubric/checklist for prioritizing what to study for senior/staff interview prep.
- Consistent note structure so you can move through topics without re-learning the format each time.
- Source-backed notes with cross-links between related topics.

Think of this repo as a study map, not a sacred text.

## Repo layout 🗂️

- [Swift Senior & Staff Rubric and Prioritized Study Checklist](<./Swift Senior & Staff Rubric and Prioritized Study Checklist.md>)  
  Start here if you want the high-level map.

- `A. Core Semantics, Syntax and Control Flow`  
  16 notes on value/reference semantics, initialization, optionals, pattern matching, error handling, type choices, and more.

- `B. Type System, Abstraction, and API Design`  
  14 notes on protocols, generics, existentials, dispatch, `Codable`, API design, property wrappers, and result builders.

- `C. Memory, Ownership, Performance and Safety`  
  12 notes on ARC, copy-on-write, exclusivity, Unicode correctness, unsafe memory, ownership features, and performance tradeoffs.

- `D. Concurrency, Isolation, and Correctness`  
  14 notes on structured concurrency, actors, `Sendable`, reentrancy, continuations, `AsyncSequence`, synchronization, and Swift 6.x concurrency migration.

- `E. Interop, Modules, and Distribution`  
  8 notes on Objective-C, C, C++, SwiftPM, module boundaries, resilience, ABI, and binary distribution concerns.

- `F. Tooling, Testing, Diagnostics, and Language-Adjacent Features`  
  6 notes on testing, LLDB, performance investigation, macros, build settings, and observation-related topics.

## What each note usually includes 🔍

Most notes follow the same rough shape:

- rubric snapshot
- core mental model
- essential mechanics
- common traps and misconceptions
- direct interview-style answers
- code probes and exercises
- production guidance
- flashcards
- sources

That structure is intentional. The goal is not just "know the feature exists," but "explain it, reason about it, and use it without hurting yourself or your teammates."

## How to use this repo 🧭

- If you want a starting point, begin with the [study checklist](<./Swift Senior & Staff Rubric and Prioritized Study Checklist.md>).
- If you are cramming for interviews, hit Tier 1 first, then use Tier 2 and Tier 3 to deepen weak spots.
- If you already write Swift every day, use the notes to pressure-test your mental models, especially around dispatch, ownership, concurrency, and module design.
- If a topic is version-sensitive, verify it against current Swift release notes, proposals, and docs before treating it as gospel.

This repo also works better in an editor or note tool that understands wiki-style links like `[[Related Topic]]`. GitHub still works fine, but the cross-linking feels nicer in tools like Obsidian.

## Good places to start 🚀

- [Swift Senior & Staff Rubric and Prioritized Study Checklist](<./Swift Senior & Staff Rubric and Prioritized Study Checklist.md>)
- [A1 — Value semantics, reference semantics, identity, and copy-on-write](<./A. Core Semantics, Syntax and Control Flow/A1 — Value semantics, reference semantics, identity, and copy-on-write.md>)
- [B10 — API Design Guidelines and Swift-Native Surface Design](<./B. Type System, Abstraction, and API Design/B10 — API Design Guidelines and Swift-Native Surface Design.md>)
- [D14 — Swift 6.2 Approachable Concurrency and New Defaults](<./D. Concurrency, Isolation, and Correctness/D14 — Swift 6.2 Approachable Concurrency and New Defaults.md>)
- [E5 — Modules, packages, targets, products, and resources](<./E. Interop, Modules, and Distribution/E5 — Modules, packages, targets, products, and resources.md>)
- [F1 — Testing Strategy in Modern Swift](<./F. Tooling, Testing, Diagnostics, and Language-Adjacent Features/F1 — Testing Strategy in Modern Swift.md>)

## If you spot something off 🔧

That is normal.

Swift changes. Apple docs change. My understanding changes. Some notes will age better than others.

If you notice something incorrect, unclear, outdated, or badly explained, fix it, rewrite it, or at least leave yourself a note to come back. The best version of this repo is the one that keeps getting sharpened.

## License 📄

See [LICENSE](./LICENSE).
