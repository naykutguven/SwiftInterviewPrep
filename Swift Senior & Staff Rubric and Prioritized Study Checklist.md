---
tags:
  - interview-prep
---
---

Verified against modern Swift 6-era sources, including:
- Swift.org Documentation
- The Swift Programming Language
- Swift API Design Guidelines
- "Announcing Swift 6" (September 17, 2024)
- "Swift 6.2 Released" (September 15, 2025)
- Apple docs for `Sendable`, macros, Swift Package Manager settings, and strict concurrency adoption

This document is aimed at a seasoned iOS/macOS engineer. It focuses on Swift language knowledge, the caveats that matter in real codebases, and the judgment signals that distinguish senior from staff-level engineers.
## How to Use This

Use the rubric in two ways:
- As an interview rubric: probe specific concepts with the questions and exercises below.
- As a study checklist: work through the priority tiers at the end.

### Scoring Scale

Use a 0-4 scale per topic:
- `0`: No working understanding.
- `1`: Can repeat definitions, but cannot reason through edge cases.
- `2`: Can use the feature in normal code, but misses caveats and failure modes.
- `3`: Can explain semantics, tradeoffs, and common bugs; applies it correctly in production code.
- `4`: Can teach the topic, design APIs around it, predict interactions with other features, and debug subtle failures.

## Interview Use: Suggested Weighting

For a senior iOS/macOS engineer:
- Must be strong in `A1-A16`, `B1-B11`, `B13-B14`, `C1-C10`, `D1-D13`, `E1-E3`, `E5`, `F1-F3`.
- Should have at least working familiarity with `B12`, `C11-C12`, `D14`, `E4`, `E6-E8`, `F4-F6`.

For a staff-level or principal candidate:
- Expect strong reasoning across all senior areas.
- Expect clear judgment on API design, migration strategy, module boundaries, resilience, and concurrency tradeoffs.
- Expect them to explain not just "what Swift does" but "how to structure a codebase so the language works for you instead of against you."

## Prioritized Study Checklist

### Tier 1: Non-Negotiable for Senior iOS/macOS Work
  
- `A1` Value vs reference semantics, identity, CoW
- `A2` `let`/`var`/`mutating`
- `A3` Initialization model
- `A4` Optionals and nil modeling
- `A5` Pattern matching and exhaustiveness
- `A6` `defer` and cleanup semantics
- `A7` Functions, labels, defaults, and `inout`
- `A8` Closures, escaping, and captures
- `A9` Enum-based state modeling
- `A10` Extensions and retroactive modeling
- `A11` Access control
- `A12` Availability and conditional compilation
- `A13` Error handling and typed throws
- `A14` Operators and literal affordances
- `A15` Type choices: `struct`/`class`/`enum`/`protocol`/`actor`
- `A16` Regex and standard-library idioms
- `B1` Protocols, associated types, `Self`
- `B2` `any` vs `some`
- `B3` Generics and same-type constraints
- `B4` Dispatch model and protocol extension dispatch
- `B5` Key paths
- `B6` `Any`, metatypes, dynamic casting
- `B7` Synthesized conformances
- `B8` `Equatable`/`Hashable`
- `B9` `Codable` evolution
- `B10` Swift API design
- `B11` Resilience-aware public API thinking
- `B13` Property wrappers
- `B14` Result builders
- `C1` ARC fundamentals
- `C2` CoW behavior of stdlib types
- `C3` Exclusivity and `inout`
- `C4` String/Unicode correctness
- `C5` Collection complexity
- `C6` Unsafe memory basics
- `C9` Generic vs existential performance model
- `C10` Debug vs release behavior
- `D1` Structured concurrency fundamentals
- `D2` Cancellation and task hierarchy
- `D3` `async let` and task groups
- `D4` Actors
- `D5` `@MainActor`
- `D6` `Sendable` and `@Sendable`
- `D7` Actor reentrancy
- `D8` Isolation control
- `D9` Continuations
- `D10` `AsyncSequence` and streams
- `D11` Detached tasks
- `D13` Swift 6 strict concurrency migration
- `E1` Objective-C interop
- `E2` Foundation bridging
- `E3` C interop boundaries
- `E5` SwiftPM targets, products, resources
- `F1` Testing async and concurrent code
- `F2` Diagnostics and LLDB
- `F3` Performance investigation habits

### Tier 2: Strong Senior / Emerging Staff Depth

- `B12` Variadic generics and parameter packs
- `C7` Ownership modifiers
- `C8` `Copyable` and noncopyable types
- `C11` Strict memory safety mode
- `C12` `Span` and `InlineArray`
- `D12` Actors vs locks vs other synchronization
- `D14` Swift 6.2 approachable concurrency behavior
- `E6` Import visibility and dependency leakage
- `E7` ABI, source compatibility, module stability
- `E8` `@inlinable` and `@usableFromInline`
- `F4` Macros
- `F5` Language/build settings literacy
- `F6` Observation and generated semantics

### Tier 3: Staff-Level Differentiators and Specialized Mastery

- `E4` C++ interoperability
- `C12` Deep low-level use of `Span`/`InlineArray`
- `C7-C8` Ownership-driven API design for libraries
- `B11`, `E7-E8` Public SDK design under SemVer and binary-compatibility constraints
- `D14`, `F5` Migration strategy across multiple targets, modules, or client-facing packages
- `F4` Macro architecture decisions in library or infrastructure code

## Study Plan Order

Work in this order if building toward staff-level competence:
1. Learn semantic correctness first:
	- `A1-A16`, `B1-B6`, `C1-C6`
2. Build design judgment:
	- `B7-B11`, `B13-B14`, `E1-E3`, `E5`, `F1-F3`
3. Build concurrency mastery:
	- `D1-D13`
4. Add performance and low-level ownership depth:
	- `C7-C12`, `B12`
5. Add library and ecosystem depth:
	- `E4`, `E6-E8`, `F4-F6`, `D14`

## High-Signal Practice Formats

Use these to measure real understanding instead of passive recognition:
- Whiteboard or talk through a bug caused by protocol-extension dispatch.
- Refactor a completion-handler API to `async/await` and explain cancellation and isolation.
- Diagnose a retain cycle with closures and tasks.
- Evolve a `Codable` model without breaking old persisted data.
- Review an API and reduce its access surface while preserving testability.
- Explain how one `await` inside an actor method can introduce a logic bug.
- Compare an existential-based design to a generic one for a hot path.
- Wrap a C API behind a safe Swift interface with minimal unsafe surface area.
- Outline a migration plan from Swift 5.x to Swift 6 strict concurrency.
- Explain whether a library feature should be implemented with a macro, wrapper, protocol, or plain code.

## Minimum Interview Loop for a Senior/Staff Candidate

If you only have time for a tight interview loop, probe these:
- `A1`, `A3`, `A8`
- `B1`, `B2`, `B4`, `B10`
- `C1`, `C3`, `C4`, `C6`
- `D1`, `D4`, `D6`, `D7`, `D9`, `D13`
- `E1`, `E2`, `E5`
- `F1`, `F2`
