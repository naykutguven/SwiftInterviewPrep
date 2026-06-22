---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - debug-release
  - compiler-optimization
---
## 0. Rubric snapshot

**Rubric expectation**

Know that debug builds can mislead performance investigations; understand specialization, inlining, optimization levels, assertion behavior, and when inspecting lowered/generated output such as SIL is warranted for ownership or performance questions.

**Caveats**

Measuring debug builds and reasoning from them is often wrong.

**You should be able to answer**

- Why can a data structure or algorithm seem slow in debug and fine in release?
- When would you inspect SIL or compiler-generated output rather than keep guessing about a performance or ownership issue?
- What is the difference in intent between `assert`, `precondition`, and `fatalError`?

**You should be able to do**

- Given a crash that appears only in release or only in debug, outline a debugging plan.

---

## 1. Core mental model

Swift source code is not the thing that runs. Swift source is type-checked, lowered into intermediate forms, optimized, and eventually emitted as machine code. A debug build and a release build can have very different generated code, even when the source is identical.

Debug builds prioritize fast compilation, debuggability, predictable stepping, symbol quality, and testability. SwiftPM debug builds use `-Onone`, `-g`, and `-enable-testing`, meaning no optimization, debug information, and testability support. ([Swift Docs](https://docs.swift.org/swiftpm/documentation/packagemanagerdocs/usingbuildconfigurations/?utm_source=chatgpt.com "Using build configurations")) This makes debug builds excellent for development, but often bad evidence for performance conclusions.

Release builds prioritize runtime performance and/or code size. Optimized builds enable transformations such as inlining, generic specialization, dead-code elimination, ARC traffic reduction, devirtualization, loop optimization, and removal of debug-only checks. The Swift compiler team’s whole-module optimization write-up gives function specialization, cross-file inlining, and redundant reference-count elimination as examples of optimizations enabled when the compiler has more visibility. ([Swift.org](https://swift.org/blog/whole-module-optimizations/?utm_source=chatgpt.com "Whole-Module Optimization in Swift 3"))

The key idea:

```text
Debug answers: “Can I inspect and step through this?”
Release answers: “What code should actually ship and run fast?”
```

Swift guarantees language semantics, not that debug and release will have the same performance profile, call shape, ARC traffic, stack frames, or assertion behavior. If a bug only appears under one build configuration, that is not automatically “a compiler bug.” It may be undefined behavior, an invalid unsafe-memory assumption, an over-optimized-away debug assumption, an assertion/precondition difference, a race exposed by timing, or a genuine optimizer/codegen issue.

---

## 2. Essential mechanics

### Optimization levels change the generated program shape

Common Swift compiler optimization modes:

```text
-Onone      No optimization. Debug-friendly. Slow runtime.
-O          Optimize for speed. Typical release behavior.
-Osize      Optimize for code size. Often good for app binary size.
-Ounchecked Optimize aggressively and remove some safety checks. Rarely appropriate.
```

`-Osize` is not just “slower release”; Swift’s code-size optimization mode is designed to reduce binary size and can be appropriate when size matters, though performance-sensitive code may still prefer `-O`. ([Swift.org](https://swift.org/blog/osize/?utm_source=chatgpt.com "Code Size Optimization Mode in Swift 4.1"))

Example:

```swift
func sum(_ values: [Int]) -> Int {
    var result = 0

    for value in values {
        result += value
    }

    return result
}
```

In debug, this may preserve more obvious loop structure and runtime overhead so LLDB can inspect locals. In release, the optimizer may inline calls, remove redundant retains/releases, specialize generic paths, and simplify bounds checks when it can prove safety.

The source-level algorithm is the same. The generated machine code may be materially different.

---

### Specialization makes generic Swift fast

Generics are often zero-cost in optimized Swift when the compiler can specialize them for concrete types.

```swift
func total<C: Collection>(_ values: C) -> Int where C.Element == Int {
    values.reduce(0, +)
}

let array = [1, 2, 3]
print(total(array))
```

In optimized builds, the compiler may specialize `total` for `Array<Int>`, inline `reduce`, and remove abstraction overhead. In debug, it often keeps the generic machinery more explicit to preserve debuggability and compilation speed.

This is why a generic collection algorithm can look unacceptably slow in debug but perform well in release.

---

### Inlining is an optimization, not an API-design crutch

Inlining replaces a function call with the function body at the call site. That can remove call overhead and unlock more optimization, but it can also increase code size.

```swift
func clamp(_ value: Int, lower: Int, upper: Int) -> Int {
    min(max(value, lower), upper)
}
```

For small hot functions, release optimization may inline automatically. In most app code, do not reach for `@inline(__always)` casually. If you are designing public library APIs, `@inlinable` is even more serious: it exposes the function body as part of the module interface so clients can optimize across module boundaries. The Swift Evolution proposal for cross-module inlining says the Swift compiler performs aggressive optimization within a module, including specialization and inlining, but cross-module boundaries limit what the optimizer can see unless APIs expose more implementation. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md?utm_source=chatgpt.com "Cross-module inlining and specialization - swift-evolution"))

Senior-level rule:

```text
Prefer measuring first. Use inlining annotations only when evidence shows the boundary matters.
```

---

### Assertions are configuration-sensitive

Verified with Swift 6.2.1:

```swift
var checks = 0

func isValid(_ value: Int) -> Bool {
    checks += 1
    return value >= 0
}

for i in 0..<3 {
    assert(isValid(i))
    print(i)
}

print("checks", checks)
```

Compiled with:

```bash
swiftc -Onone assert.swift -o debug
swiftc -O assert.swift -o release
```

Debug output:

```text
0
1
2
checks 3
```

Release output:

```text
0
1
2
checks 0
```

Why: `assert` is for debug-time internal consistency checks. In optimized release builds, the assertion condition is not evaluated. Do not put required side effects inside `assert`.

---

### SIL is useful when source-level reasoning stops being enough

SIL is Swift Intermediate Language. It is useful when you need to inspect ownership, ARC, generic specialization, closure captures, dynamic dispatch, or optimizer behavior. The Swift compiler debugging guide documents commands such as `swiftc -emit-silgen`, `swiftc -emit-sil -Onone`, and `swiftc -emit-sil -O`; it also notes that `-emit-sil -O` prints SIL after the complete SIL optimization pipeline. ([GitHub](https://github.com/swiftlang/swift/blob/main/docs/DebuggingTheCompiler.md "swift/docs/DebuggingTheCompiler.md at main · swiftlang/swift · GitHub"))

Useful commands:

```bash
# Raw SIL after SILGen
swiftc -emit-silgen -Onone File.swift

# SIL after mandatory passes, debug-like
swiftc -emit-sil -Onone File.swift

# Optimized SIL
swiftc -emit-sil -O File.swift

# LLVM IR
swiftc -emit-ir -O File.swift

# Assembly
swiftc -S -O File.swift
```

Use SIL when you need to answer questions like:

```text
Was this generic function specialized?
Was this closure allocation removed?
Is this retain/release pair still present?
Did this protocol call become a direct call?
Did the optimizer remove the branch I thought existed?
```

Do not use SIL as your first performance tool. Start with Instruments, benchmarks, and measurements. Apple’s Instruments documentation positions Instruments as the tool for analyzing app performance, resource usage, responsiveness, and memory behavior. ([Apple Developer](https://developer.apple.com/tutorials/instruments?utm_source=chatgpt.com "Instruments Tutorials | Apple Developer Documentation"))

---

## 3. Common traps and misconceptions

### Trap 1: Benchmarking debug builds

Bad:

```swift
let start = ContinuousClock.now
_ = (0..<1_000_000).map { $0 * 2 }.filter { $0.isMultiple(of: 3) }
print(start.duration(to: .now))
```

Then concluding:

```text
“map/filter is slow in Swift.”
```

Better:

```swift
import Testing

@Test
func benchmarkTransform() {
    // Use a proper benchmark harness or Instruments.
    // Build with release-like optimization.
}
```

Debug builds preserve abstraction and avoid expensive optimization passes. Optimized builds may fuse, specialize, inline, and remove overhead.

---

### Trap 2: Putting required work inside `assert`

Bad:

```swift
assert(database.connect())
```

If `connect()` performs required work, release builds may skip it.

Better:

```swift
let didConnect = database.connect()
precondition(didConnect, "Database must be connected before continuing")
```

Better still for recoverable production failure:

```swift
guard database.connect() else {
    throw StartupError.databaseUnavailable
}
```

Use `assert` for programmer assumptions during development, not required runtime behavior.

---

### Trap 3: Treating release-only crashes as “impossible”

A release-only crash often means one of these:

```text
Unsafe pointer lifetime bug
Race condition exposed by timing
Incorrect precondition
Assumption removed with assert
Undefined behavior hidden by debug layout
Optimizer-sensitive compiler bug
Different resource/configuration in release scheme
```

The wrong reaction:

```text
“It works in debug, so release must be wrong.”
```

The right reaction:

```text
“What assumption exists only because debug builds are slower, less optimized, or more checked?”
```

---

### Trap 4: Overusing `@inline(__always)`

Bad:

```swift
@inline(__always)
func parseUserProfile(_ data: Data) -> UserProfile {
    // large implementation
}
```

This can increase binary size and reduce instruction-cache locality. It may also make debugging worse.

Better:

```swift
func parseUserProfile(_ data: Data) -> UserProfile {
    // let the optimizer decide first
}
```

Only force inlining after measurement shows the call boundary is the problem.

---

### Trap 5: Confusing `@inlinable` with “make this faster”

`@inlinable` is a library-evolution decision. It exposes implementation to clients for optimization. That can freeze implementation details and create compatibility debt.

Bad:

```swift
@inlinable
public func normalize(_ value: Double) -> Double {
    // implementation you may want to change later
}
```

Better:

```swift
public func normalize(_ value: Double) -> Double {
    // keep implementation resilient until evidence says otherwise
}
```

Use `@inlinable` mainly for small, stable, performance-critical public APIs where cross-module specialization matters.

---

## 4. Direct answers to rubric questions

### Q1. Why can a data structure or algorithm seem slow in debug and fine in release?

Because debug builds disable most optimizations. Generic specialization, inlining, ARC elimination, devirtualization, loop optimization, and dead-code elimination may not happen, or may happen much less aggressively.

A generic data structure is the canonical example:

```swift
struct Queue<Element> {
    private var storage: [Element] = []

    mutating func enqueue(_ element: Element) {
        storage.append(element)
    }

    mutating func dequeue() -> Element? {
        storage.isEmpty ? nil : storage.removeFirst()
    }
}
```

In debug, abstraction costs, bounds checks, ARC traffic, and non-specialized generic paths can dominate. In release, the optimizer may see concrete `Element` types, inline methods, and remove redundant work.

Interview version:

> A debug Swift build is built for debuggability, not performance. With `-Onone`, generic code may remain unspecialized, small functions may not inline, ARC traffic may remain visible, and abstraction overhead can dominate. A release build changes the generated code substantially. So I don’t use debug timings to judge Swift performance. I measure optimized builds with Instruments or a benchmark harness, then inspect SIL only if I need to understand specialization, ownership, or dispatch behavior.

---

### Q2. When would you inspect SIL or compiler-generated output rather than keep guessing about a performance or ownership issue?

Inspect SIL when the question is specifically about what the compiler generated.

Good reasons:

```text
You suspect a generic function is not specialized.
You suspect an existential or protocol call is preventing optimization.
You need to inspect retain/release traffic.
You need to understand closure captures or heap boxes.
You need to confirm whether an inlineable public API is optimized across a module boundary.
You are debugging an optimizer-sensitive release-only issue.
```

Bad reasons:

```text
You have not profiled yet.
You do not know whether the code is on a hot path.
You are trying to justify clever code without evidence.
```

SIL is not a replacement for profiling. It is a microscope for compiler behavior.

Interview version:

> I inspect SIL after measurement has narrowed the issue to a compiler-relevant question: specialization, inlining, ARC, closure allocation, dispatch, or ownership. For normal app performance issues, I start with Instruments and targeted benchmarks. If the profile says a generic abstraction is hot, then SIL can tell me whether the optimizer specialized it or whether an existential, dynamic dispatch, resilience boundary, or closure capture is blocking optimization.

---

### Q3. What is the difference in intent between `assert`, `precondition`, and `fatalError`?

`assert` is for debug-only internal correctness checks. It helps catch programmer mistakes during development and should not be required for production behavior.

```swift
assert(index >= 0)
```

`precondition` is for conditions that must be true for the program to safely continue, including in optimized shipping code. Apple’s `precondition` documentation describes it as detecting conditions that must prevent the program from proceeding even in shipping code. ([Apple Developer](https://developer.apple.com/documentation/swift/precondition%28_%3A_%3Afile%3Aline%3A%29?utm_source=chatgpt.com "precondition(_:_:file:line:)"))

```swift
precondition(index >= 0, "Index must be non-negative")
```

`fatalError` unconditionally stops execution and returns `Never`. Apple documents `fatalError` as unconditionally printing a message and stopping execution. ([Apple Developer](https://developer.apple.com/documentation/swift/fatalerror%28_%3Afile%3Aline%3A%29?utm_source=chatgpt.com "fatalError(_:file:line:)"))

```swift
fatalError("Subclass must override this method")
```

Mental model:

```text
assert       = “This should catch my bug while developing.”
precondition = “Continuing would violate the API contract or corrupt correctness.”
fatalError   = “There is no valid continuation path.”
```

Interview version:

> I use `assert` for debug-only assumptions, `precondition` for non-recoverable contract violations that must still be checked in optimized builds, and `fatalError` when execution cannot continue unconditionally. I never put required side effects inside `assert`, because optimized builds can remove it. For user-facing or environmental failures, I prefer throwing, returning a result, or modeling failure explicitly instead of crashing.

---

## 5. Code probe

The rubric has no code probe for C10. Use these instead.

### Minimal example: `assert` behavior differs by optimization level

Given:

```swift
var checks = 0

func isValid(_ value: Int) -> Bool {
    checks += 1
    return value >= 0
}

for i in 0..<3 {
    assert(isValid(i))
    print(i)
}

print("checks", checks)
```

Compiled with `-Onone`:

```text
0
1
2
checks 3
```

Compiled with `-O`:

```text
0
1
2
checks 0
```

Why:

```text
-Onone:
assert condition is evaluated
checks increments three times

-O:
assert condition is not evaluated
checks remains zero
```

State diagram:

```text
Debug:
loop -> assert(isValid(i)) -> checks += 1 -> print(i)

Release:
loop -> assert removed / not evaluated -> print(i)
```

---

### Counterexample: using `assert` for required work

Bad:

```swift
final class Cache {
    private(set) var isPrepared = false

    func prepare() -> Bool {
        isPrepared = true
        return true
    }
}

let cache = Cache()

assert(cache.prepare())

print(cache.isPrepared)
```

Debug output:

```text
true
```

Release output:

```text
false
```

Why: `prepare()` is called only as the `assert` condition. In optimized release builds, that condition is not evaluated.

Better:

```swift
let didPrepare = cache.prepare()
precondition(didPrepare, "Cache preparation must succeed before use")

print(cache.isPrepared)
```

---

### Production example: release-only crash around unsafe memory

Bad:

```swift
func makePointer() -> UnsafePointer<UInt8> {
    let bytes: [UInt8] = [1, 2, 3]

    return bytes.withUnsafeBufferPointer { buffer in
        buffer.baseAddress!
    }
}

let pointer = makePointer()
print(pointer[0])
```

This may appear to work in debug and fail in release. The pointer escapes the lifetime guaranteed by `withUnsafeBufferPointer`.

Better:

```swift
func withBytes<R>(_ body: (UnsafeBufferPointer<UInt8>) throws -> R) rethrows -> R {
    let bytes: [UInt8] = [1, 2, 3]

    return try bytes.withUnsafeBufferPointer { buffer in
        try body(buffer)
    }
}

let first = withBytes { buffer in
    buffer[0]
}

print(first)
```

Output:

```text
1
```

Why this fix is correct: the unsafe buffer pointer is used only inside the closure where Swift guarantees the buffer lifetime.

---

## 6. Exercise

### Problem

Given a crash that appears only in release or only in debug, outline a debugging plan.

### Bad / naive version

```text
1. Assume the compiler is broken.
2. Add random print statements.
3. Change optimization level until the crash disappears.
4. Ship the configuration that seems to work.
```

### What is wrong?

```text
This treats the symptom, not the cause.
It can hide unsafe-memory bugs, race conditions, assertion misuse, and configuration drift.
It provides no reproducible evidence.
It risks shipping a fragile workaround.
```

### Improved version

#### Step 1: Reproduce with a controlled matrix

```text
Configuration:
- Debug / -Onone
- Release / -O
- Release / -Osize if used
- Sanitizers enabled where possible
- Same device/simulator, OS version, account, data set, feature flags

Record:
- Exact scheme
- Swift optimization level
- Compilation conditions
- Swift language mode
- Whole-module / incremental settings
- Testability settings
- Active feature flags
```

Do not compare “debug simulator” with “release physical device” and conclude the build mode is the only difference.

---

#### Step 2: Classify the failure

```text
Crash type:
- precondition/fatalError/assertion?
- EXC_BAD_ACCESS?
- index out of range?
- force unwrap nil?
- actor/concurrency runtime trap?
- data race / Thread Sanitizer signal?
- memory graph / retain cycle issue?
- watchdog / main-thread hang?
```

A release-only `EXC_BAD_ACCESS` points toward unsafe memory, object lifetime, data races, or optimizer-exposed undefined behavior.

A debug-only assertion failure may mean the code violates an invariant but release removes the check.

---

#### Step 3: Check assertion and precondition usage

Search for:

```swift
assert(...)
assertionFailure(...)
precondition(...)
preconditionFailure(...)
fatalError(...)
```

Look for side effects:

```swift
assert(service.start())
assert(cache.populate())
assert(lock.acquire())
```

Refactor required work out of assertions:

```swift
let didStart = service.start()
precondition(didStart, "Service must start before use")
```

---

#### Step 4: Enable diagnostics

Use relevant tools:

```text
Address Sanitizer:
- catches many memory errors

Thread Sanitizer:
- catches many data races

Undefined Behavior Sanitizer:
- useful for low-level interop and numeric assumptions

Malloc Scribble / Guard Malloc / Zombies:
- useful for lifetime and over-release style bugs

Instruments:
- Time Profiler, Allocations, Leaks, Hangs, System Trace
```

For memory allocation investigations, Apple documents the Allocations instrument as tracking heap and anonymous VM allocations by size and category. ([Apple Developer](https://developer.apple.com/documentation/xcode/gathering-information-about-memory-use?utm_source=chatgpt.com "Gathering information about memory use"))

---

#### Step 5: Reduce the repro

Create the smallest failing case:

```text
Same function?
Same input?
Same optimization level?
Same module boundary?
Same dependency version?
Same concurrency timing?
Same target settings?
```

If a release-only crash disappears when code is moved into one file, suspect inlining/specialization/module-boundary behavior or undefined behavior exposed by layout changes.

---

#### Step 6: Inspect generated output only after narrowing

Use SIL when the narrowed question is compiler-shaped:

```bash
swiftc -emit-sil -Onone File.swift > debug.sil
swiftc -emit-sil -O File.swift > release.sil
```

Compare:

```text
Was a branch removed?
Was a generic function specialized?
Was a retain/release eliminated?
Did an existential box appear?
Was a closure heap-allocated?
Did a precondition remain?
```

The compiler debugging guide explicitly documents dumping SIL at different phases and optimization levels, including `-emit-silgen`, `-emit-sil -Onone`, and `-emit-sil -O`. ([GitHub](https://github.com/swiftlang/swift/blob/main/docs/DebuggingTheCompiler.md "swift/docs/DebuggingTheCompiler.md at main · swiftlang/swift · GitHub"))

---

#### Step 7: Fix the semantic issue, not just the build setting

Examples:

|Symptom|Likely issue|Better fix|
|---|---|---|
|Release-only `EXC_BAD_ACCESS`|Escaped unsafe pointer|Keep pointer use inside lifetime closure|
|Debug-only crash in `assert`|Invariant violated|Fix invariant; do not remove assertion blindly|
|Release-only stale UI state|Race/timing difference|Add isolation/cancellation/stale-result handling|
|Release-only missing behavior|Side effect inside `assert`|Move side effect into normal code|
|Release-only performance regression|Missed specialization/inlining|Measure, then adjust API shape or module boundary|
|Debug-only slowness|`-Onone` overhead|Measure optimized build before rewriting|

---

## 7. Production guidance

Use this knowledge in production when:

```text
Investigating performance regressions
Reviewing benchmark claims
Debugging release-only crashes
Designing hot-path generic APIs
Evaluating @inlinable or @inline annotations
Investigating ARC, ownership, or closure allocation issues
Deciding whether a bug is semantic, configuration-related, or compiler-related
```

Be careful when:

```text
A benchmark was run in Debug
A fix only changes optimization settings
A function has required side effects inside assert
A public library exposes @inlinable APIs
A release crash appears in unsafe or concurrent code
A Swift package target has different build settings from the app target
```

Avoid when:

```text
Using SIL before profiling
Adding @inline(__always) because code “feels slow”
Using -Ounchecked in app production builds
Assuming debug call stacks match release call stacks
Treating assert/precondition/fatalError as ordinary error handling
```

Debugging checklist:

```text
Can I reproduce under both -Onone and -O?
Am I measuring the same device, OS, data, and feature flags?
Is required work hidden inside assert?
Is this a crash, hang, race, memory bug, or logic bug?
Do sanitizers reveal anything?
Does Instruments identify the hot path?
Is a generic/protocol abstraction blocking specialization?
Is unsafe memory escaping its valid lifetime?
Do app and package targets use different Swift settings?
Is SIL needed to answer a specific compiler-behavior question?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Debug is slower than release because release is optimized.

This is true but too shallow.

### Senior answer

> Debug uses little or no optimization, so generic specialization, inlining, ARC cleanup, and devirtualization may not happen. I would not trust debug timings. I would reproduce with a release-like configuration, profile with Instruments, and check assertion/precondition differences if behavior changes between configurations.

### Staff-level answer

> Debug and release are different compiler products with different goals. For performance, I establish a reproducible optimized benchmark or Instruments trace before changing code. For release-only crashes, I first suspect unsafe lifetime, data races, side effects hidden in debug-only checks, or configuration drift before blaming the compiler. If profiling points to a compiler-shaped question, I inspect optimized SIL to verify specialization, dispatch, ARC, closure allocation, and module-boundary behavior. For libraries, I treat `@inlinable` and forced inlining as API/resilience decisions, not local performance tweaks.

Staff-level questions to ask:

```text
Are we measuring optimized code or debug code?
Is the hot path actually hot in production traces?
Did the abstraction cross a resilience or module boundary?
Are we relying on assert for required behavior?
Is this crash caused by invalid Swift semantics, unsafe memory, or timing-sensitive concurrency?
Do package targets, app targets, tests, and CI use consistent Swift build settings?
Would changing the API shape unlock specialization without exposing implementation detail?
Is @inlinable worth the public contract it creates?
Can a benchmark prove this fix is worth the added complexity?
```

---

## 9. Interview-ready summary

Debug and release Swift builds should be treated as different generated programs with the same source-level semantics but different optimization goals. Debug builds preserve debuggability and often skip specialization, inlining, and ARC cleanup, so they can badly mislead performance analysis. Release builds optimize aggressively and can expose unsafe-memory bugs, races, assertion misuse, and invalid assumptions. I use `assert` for debug-only programmer checks, `precondition` for non-recoverable contract violations that must still be checked in shipping code, and `fatalError` when there is no valid continuation. For performance, I measure optimized builds first; for compiler-shaped questions, I inspect SIL to verify specialization, dispatch, ownership, and generated code behavior.

---

## 10. Flashcards

Q: Why are debug Swift builds often misleading for performance?

A: They usually compile with `-Onone`, preserving debuggability and skipping most optimizations such as specialization, inlining, ARC cleanup, and devirtualization.

Q: What is generic specialization?

A: The optimizer creates concrete implementations of generic code for specific types, removing much of the generic abstraction overhead.

Q: When should you inspect SIL?

A: When measurement has narrowed the issue to compiler behavior: specialization, ARC, dispatch, closure allocation, ownership, or module-boundary optimization.

Q: What is the intent of `assert`?

A: Debug-time internal consistency checking. It must not contain required production side effects.

Q: What is the intent of `precondition`?

A: Enforce a non-recoverable API or program contract that must hold for execution to safely continue, including optimized builds.

Q: What is the intent of `fatalError`?

A: Unconditionally terminate execution because there is no valid continuation path.

Q: Why is `@inlinable` risky in a public library?

A: It exposes the function body to clients for optimization, making implementation details part of the public optimization/resilience contract.

Q: What should you do first for a release-only crash?

A: Reproduce with a controlled configuration matrix, classify the crash, check assertion/precondition usage, enable diagnostics/sanitizers, then reduce the repro.

Q: What is the main risk of `-Ounchecked`?

A: It removes some safety checks and lets the optimizer assume conditions that may not actually hold, which can turn bugs into undefined or corrupt behavior.

Q: Why can a debug-only crash still indicate a real bug?

A: Debug assertions may expose an invariant violation that release removes. Removing the assertion does not fix the violated invariant.

---

## 11. Related sections

- [[C9 — Existential, generic, and allocation performance model]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — C10 rubric expectation, caveats, questions, and exercise.
- SwiftPM documentation: debug builds use `-Onone`, `-g`, and `-enable-testing`. ([Swift.org](https://docs.swift.org/swiftpm/documentation/packagemanagerdocs/usingbuildconfigurations/?utm_source=chatgpt.com "Using build configurations")
- Swift.org compiler debugging guide: dumping SIL, optimized SIL, LLVM IR, and assembly. ([GitHub](https://github.com/swiftlang/swift/blob/main/docs/DebuggingTheCompiler.md "swift/docs/DebuggingTheCompiler.md at main · swiftlang/swift · GitHub"))
    
- Swift.org whole-module optimization article: specialization, inlining, and redundant ARC elimination. ([Swift.org](https://swift.org/blog/whole-module-optimizations/?utm_source=chatgpt.com "Whole-Module Optimization in Swift 3"))
    
- Swift Evolution SE-0193: cross-module inlining and specialization context. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md?utm_source=chatgpt.com "Cross-module inlining and specialization - swift-evolution"))
    
- Swift.org code-size optimization mode: `-Osize` behavior and tradeoffs. ([Swift.org](https://swift.org/blog/osize/?utm_source=chatgpt.com "Code Size Optimization Mode in Swift 4.1"))
    
- Apple Developer Documentation: `precondition` and `fatalError`. ([Apple Developer](https://developer.apple.com/documentation/swift/precondition%28_%3A_%3Afile%3Aline%3A%29?utm_source=chatgpt.com "precondition(_:_:file:line:)"))
    
- Apple Instruments documentation: performance/resource analysis and allocation tracking.