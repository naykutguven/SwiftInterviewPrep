---
tags:
  - "#swift"
  - interview-prep
  - core-semantics
  - defer
  - clean-up
  - control-flow
---
## 0. Rubric snapshot

**Rubric expectation**

Understand `defer` ordering, especially around `throw`, `return`, and nested scopes. The key caveat is that `defer` runs when leaving the **current scope**, not “at function end” in some abstract sense.

**You should be able to answer**

- In what order do multiple `defer` blocks execute?
- Where does `defer` become clearer than duplicating cleanup code in `catch` and `return` paths?

**You should be able to do**

- Use `defer` to guarantee cleanup of a file descriptor or lock in a function with multiple exit points.

---

## 1. Core mental model

`defer` registers synchronous cleanup work to run when execution leaves the **lexical scope** where the `defer` statement was reached. Swift’s language guide describes `defer` as code executed when leaving the current scope, and its error-handling section specifies that deferred actions execute in reverse source order. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/?utm_source=chatgpt.com "Control Flow - Documentation | Swift.org"))

The important word is **reached**. A `defer` statement does nothing until execution passes over it. If you return before reaching a `defer`, that cleanup is never registered.

The second important word is **scope**. If a `defer` is inside a `do`, `if`, `for`, `while`, or `switch` case body, it runs when that inner scope exits, not necessarily when the whole function exits. Swift forum discussion clarifies this as Swift’s static lexical scope, not a dynamic “current function” scope like Go’s `defer`. ([Swift Forums](https://forums.swift.org/t/details-of-defer-statement-in-swift/4483?utm_source=chatgpt.com "Details of defer statement in Swift"))

The key idea:

```text
defer = register cleanup now; execute it synchronously when this exact scope exits, in LIFO order.
```

`defer` is not error handling. It does not catch errors. It does not change the returned value after the return expression is evaluated. It is a control-flow safety tool for cleanup, rollback, unlocking, logging completion, and restoring temporary state.

---

## 2. Essential mechanics

### 2.1 Multiple `defer` blocks run in reverse order

Multiple `defer` blocks in the same scope execute like a stack: last registered, first executed. Swift’s documentation explicitly states that deferred actions run in reverse order. ([docs.swift.org](https://docs.swift.org/swift-book/LanguageGuide/ErrorHandling.html?utm_source=chatgpt.com "Error Handling - Documentation | Swift.org"))

```swift
func example() {
    defer { print("first defer") }
    defer { print("second defer") }

    print("body")
}

example()
```

Output:

```text
body
second defer
first defer
```

This is useful because resources are usually acquired in one order and should be released in the opposite order.

```swift
lockA.lock()
defer { lockA.unlock() }

lockB.lock()
defer { lockB.unlock() }

// lockB unlocks first, then lockA
```

---

### 2.2 `defer` runs on normal return, early return, and `throw`

Once a `defer` is reached, it runs when the scope exits, regardless of whether the scope exits normally, through `return`, or by throwing an error. The official docs describe this behavior in the context of cleanup actions. ([docs.swift.org](https://docs.swift.org/swift-book/LanguageGuide/ErrorHandling.html?utm_source=chatgpt.com "Error Handling - Documentation | Swift.org"))

```swift
enum LoadError: Error {
    case missingFile
}

func load(shouldThrow: Bool) throws -> String {
    print("open")
    defer { print("close") }

    if shouldThrow {
        print("throw")
        throw LoadError.missingFile
    }

    print("return")
    return "data"
}
```

For success:

```text
open
return
close
```

For failure:

```text
open
throw
close
```

---

### 2.3 `defer` belongs to the current lexical scope

This is the most common interview trap.

```swift
func nestedScope() {
    print("start")

    defer { print("outer") }

    do {
        defer { print("inner") }
        print("inside do")
    }

    print("after do")
}

nestedScope()
```

Output:

```text
start
inside do
inner
after do
outer
```

The inner `defer` does **not** wait until the function ends. It runs when the `do` block exits.

---

### 2.4 `defer` must be reached before it can run

Bad assumption:

```swift
func example(_ shouldExit: Bool) {
    if shouldExit {
        return
    }

    defer { print("cleanup") }

    print("work")
}
```

Calling `example(true)` prints nothing. The `defer` was never reached, so no cleanup was registered.

This matters for resource handling:

```swift
let fd = open(path, O_RDONLY)
guard fd >= 0 else {
    throw FileError.openFailed
}

defer {
    close(fd)
}
```

Place `defer` **immediately after successful acquisition**, not before acquisition and not after logic that may exit early.

---

### 2.5 `defer` is synchronous and cannot transfer control out

A `defer` body cannot contain `return`, `break`, `continue`, or throw an error out of the `defer` body. Swift’s docs state that deferred statements may not contain code that transfers control out of the statements, such as `break`, `return`, or throwing an error. ([docs.swift.org](https://docs.swift.org/swift-book/LanguageGuide/ErrorHandling.html?utm_source=chatgpt.com "Error Handling - Documentation | Swift.org"))

Compiler probe:

```swift
func invalid() {
    defer {
        return
    }
}
```

Compiler output with Swift 6.2.1:

```text
/tmp/defer_error.swift:2:5: warning: 'defer' statement at end of scope always executes immediately; replace with 'do' statement to silence this warning
/tmp/defer_error.swift:3:9: error: 'return' cannot transfer control out of a defer statement
```

In an async function, this is also invalid:

```swift
func g() async {}

func f() async {
    defer {
        await g()
    }
}
```

Compiler output with Swift 6.2.1:

```text
error: 'async' call cannot occur in a defer body
```

So `defer` is good for synchronous cleanup. It is not a place to perform async teardown.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `defer` means “at function end”

Bad:

```swift
func process(_ items: [Int]) {
    for item in items {
        defer {
            print("finished item", item)
        }

        print("processing", item)
    }

    print("all done")
}
```

The `defer` runs at the end of each loop iteration, not after the whole loop.

Better:

```swift
func process(_ items: [Int]) {
    defer {
        print("all done")
    }

    for item in items {
        print("processing", item)
    }
}
```

Use `defer` at the scope whose exit you actually care about.

---

### Trap 2: Registering cleanup too late

Bad:

```swift
func readConfig(path: String) throws -> Data {
    let fd = open(path, O_RDONLY)

    guard fd >= 0 else {
        throw FileError.openFailed(errno: errno)
    }

    guard isReadable(fd) else {
        close(fd)
        throw FileError.notReadable
    }

    defer {
        close(fd)
    }

    return try readAll(fd)
}
```

If `isReadable(fd)` fails, cleanup is manually duplicated. As the function grows, someone will eventually miss one path.

Better:

```swift
func readConfig(path: String) throws -> Data {
    let fd = open(path, O_RDONLY)

    guard fd >= 0 else {
        throw FileError.openFailed(errno: errno)
    }

    defer {
        close(fd)
    }

    guard isReadable(fd) else {
        throw FileError.notReadable
    }

    return try readAll(fd)
}
```

The correct pattern is:

```text
acquire resource
check acquisition succeeded
immediately defer cleanup
do work with many possible exits
```

---

### Trap 3: Hiding business logic in `defer`

Bad:

```swift
func save() throws {
    defer {
        analytics.track("save_finished")
        user.hasCompletedOnboarding = true
        cache.invalidateEverything()
    }

    try database.save(user)
}
```

This is too much hidden behavior. A reviewer scanning the main flow may miss important side effects.

Better:

```swift
func save() throws {
    defer {
        analytics.track("save_finished")
    }

    try database.save(user)
    user.hasCompletedOnboarding = true
    cache.invalidateUser(user.id)
}
```

Use `defer` for cleanup, symmetric state restoration, and small completion bookkeeping. Do not use it to hide the feature’s main behavior.

---

### Trap 4: Expecting `defer` to mutate the returned value

Probe:

```swift
func returnValueProbe() -> Int {
    var value = 1

    defer {
        value = 2
        print("defer value:", value)
    }

    return value
}

print("returned:", returnValueProbe())
```

Output:

```text
defer value: 2
returned: 1
```

Why: the `return value` expression is evaluated before the deferred block runs. The `defer` can still mutate the local variable storage, but the returned `Int` has already been produced.

For reference types, this can be more surprising because the returned value may point to the same object that `defer` mutates.

---

## 4. Direct answers to rubric questions

### Q1. In what order do multiple `defer` blocks execute?

Multiple `defer` blocks in the same scope execute in reverse order of registration: last in, first out. This mirrors resource cleanup: if you acquire A then B, B should usually be released before A. ([docs.swift.org](https://docs.swift.org/swift-book/LanguageGuide/ErrorHandling.html?utm_source=chatgpt.com "Error Handling - Documentation | Swift.org"))

Interview version:

> Multiple `defer` blocks execute in LIFO order. The last `defer` reached in a scope runs first when that scope exits. This is useful because cleanup usually needs to happen in the reverse order of acquisition.

---

### Q2. Where does `defer` become clearer than duplicating cleanup code in `catch` and `return` paths?

`defer` is clearer when a function has one resource or state transition that must be undone regardless of whether the function succeeds, returns early, or throws.

Good examples:

```swift
lock.lock()
defer { lock.unlock() }

isLoading = true
defer { isLoading = false }

let fd = open(path, O_RDONLY)
guard fd >= 0 else { throw FileError.openFailed(errno: errno) }
defer { close(fd) }
```

Without `defer`, every `return`, `throw`, and `catch` path needs a copy of the cleanup. That is noisy and fragile.

Interview version:

> I use `defer` when cleanup is semantically tied to a successful acquisition or temporary state change. It keeps the main control flow readable and prevents missed cleanup on early returns or thrown errors. I avoid it for branch-specific business logic because that can hide important side effects.

---

## 5. Code probes

The rubric has no supplied code probe for A6, so use these probes.

### Probe 1 — ordering and nested scope

Given:

```swift
func orderingProbe() {
    print("start")
    defer { print("outer 1") }
    defer { print("outer 2") }

    do {
        defer { print("inner") }
        print("inside")
    }

    print("after inner")
}

orderingProbe()
```

Output:

```text
start
inside
inner
after inner
outer 2
outer 1
```

Why:

```text
function scope:
  defer outer 1 registered
  defer outer 2 registered

do scope:
  defer inner registered
  leaving do scope -> inner runs

leaving function scope:
  outer 2 runs
  outer 1 runs
```

The inner `defer` belongs to the `do` scope. The outer defers belong to the function scope.

---

### Probe 2 — return and throw

Given:

```swift
enum ProbeError: Error {
    case failed
}

func exitProbe(_ shouldThrow: Bool) throws -> String {
    print("open")
    defer { print("close") }

    guard !shouldThrow else {
        print("about to throw")
        throw ProbeError.failed
    }

    print("about to return")
    return "success"
}

print(try exitProbe(false))

do {
    _ = try exitProbe(true)
} catch {
    print("caught")
}
```

Output:

```text
open
about to return
close
success
open
about to throw
close
caught
```

Why:

`defer { print("close") }` is reached before both possible exits. Therefore it runs before the successful return completes and before the thrown error propagates to the `catch`.

---

### Probe 3 — return value evaluation

Given:

```swift
func returnValueProbe() -> Int {
    var value = 1

    defer {
        value = 2
        print("defer value:", value)
    }

    return value
}

print("returned:", returnValueProbe())
```

Output:

```text
defer value: 2
returned: 1
```

Why:

```text
1. return value is evaluated -> 1
2. defer runs -> local value becomes 2
3. already-evaluated return value is delivered -> 1
```

Do not use `defer` as a sneaky way to modify the returned value. Use it for cleanup or observation.

---

## 6. Exercise

### Problem

Use `defer` to guarantee cleanup of a file descriptor or lock in a function with multiple exit points.

### Bad / naive version

```swift
import Darwin
import Foundation

enum FileError: Error {
    case openFailed(errno: Int32)
    case readFailed(errno: Int32)
}

func firstByte(at path: String) throws -> UInt8? {
    let fd = open(path, O_RDONLY)

    guard fd >= 0 else {
        throw FileError.openFailed(errno: errno)
    }

    var byte: UInt8 = 0
    let count = read(fd, &byte, 1)

    if count < 0 {
        close(fd)
        throw FileError.readFailed(errno: errno)
    }

    if count == 0 {
        close(fd)
        return nil
    }

    close(fd)
    return byte
}
```

### What is wrong?

```text
The file descriptor cleanup is duplicated across every exit path.
A future branch can easily forget close(fd).
The cleanup logic distracts from the actual read logic.
```

There is also a subtle bug: if you call `close(fd)` before reading `errno`, you might accidentally observe a changed `errno`. Error values should be constructed before cleanup can disturb diagnostic state.

### Improved version

```swift
import Darwin
import Foundation

enum FileError: Error {
    case openFailed(errno: Int32)
    case readFailed(errno: Int32)
}

func firstByte(at path: String) throws -> UInt8? {
    let fd = open(path, O_RDONLY)

    guard fd >= 0 else {
        throw FileError.openFailed(errno: errno)
    }

    defer {
        close(fd)
    }

    var byte: UInt8 = 0
    let count = read(fd, &byte, 1)

    if count < 0 {
        throw FileError.readFailed(errno: errno)
    }

    if count == 0 {
        return nil
    }

    return byte
}
```

### Why this is better

The cleanup is registered immediately after successful acquisition. Every later exit path is safe:

```text
read fails       -> throw -> defer closes fd
empty file       -> return nil -> defer closes fd
read succeeds    -> return byte -> defer closes fd
future new guard -> return/throw -> defer closes fd
```

This is the production-grade pattern.

---

## 7. Production guidance

Use `defer` in production when:

```text
A resource is acquired and must be released.
A lock is manually acquired and must be unlocked.
A temporary state must be restored.
A loading flag must be reset on all exits.
Telemetry must record completion of a scoped operation.
```

Good iOS example:

```swift
@MainActor
final class ProfileViewModel {
    private(set) var isLoading = false

    func loadProfile() async {
        isLoading = true
        defer { isLoading = false }

        do {
            try Task.checkCancellation()
            // await service.loadProfile()
        } catch is CancellationError {
            // cancellation-specific handling
        } catch {
            // error handling
        }
    }
}
```

Important nuance: the `defer` body itself cannot `await`. It runs synchronously when the async function’s scope exits.

Be careful when:

```text
The defer is inside an inner scope.
The defer is after code that can exit early.
The defer body has meaningful business side effects.
The defer body depends on mutable local state that changes later.
The cleanup itself can fail and you need to report that failure.
```

Avoid when:

```text
A scoped API already exists, such as withLock.
The cleanup needs async work.
The operation is clearer as explicit do/catch/finalization.
The deferred code hides important domain behavior.
```

For locks, prefer scoped APIs when available. Apple’s `Synchronization.Mutex` provides exclusive access to protected state, and `withLock(_:)` calls a closure after acquiring the lock and then releases ownership. ([Apple Developer](https://developer.apple.com/documentation/Synchronization/Mutex?utm_source=chatgpt.com "Mutex | Apple Developer Documentation"))

Manual lock version:

```swift
import Foundation

final class LegacyCache {
    private let lock = NSLock()
    private var storage: [String: Data] = [:]

    func value(for key: String) -> Data? {
        lock.lock()
        defer { lock.unlock() }

        return storage[key]
    }
}
```

Modern scoped version:

```swift
import Synchronization
import Foundation

final class Cache: Sendable {
    private let storage = Mutex<[String: Data]>([:])

    func value(for key: String) -> Data? {
        storage.withLock { values in
            values[key]
        }
    }
}
```

The scoped version is harder to misuse because the API itself owns the unlock behavior.

Debugging checklist:

```text
Was the defer statement actually reached?
Which lexical scope does this defer belong to?
Are there multiple defers, and is their LIFO order intentional?
Is cleanup registered immediately after successful acquisition?
Does the defer hide business logic?
Does the defer rely on mutable state that changes before scope exit?
Is this async code trying to do async cleanup in defer?
Is a lock held across an await? If yes, redesign.
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `defer` runs later, usually at the end of a function. It is useful for cleanup.

This is incomplete because it misses scope, ordering, and early-exit mechanics.

### Senior answer

> `defer` runs when the current lexical scope exits, not necessarily when the function exits. Multiple defers run in reverse order. I use it immediately after successful resource acquisition or temporary state changes so cleanup happens on return, throw, and early exit.

### Staff-level answer

> `defer` is a scoped cleanup mechanism. I use it to make ownership and cleanup symmetric: acquire, immediately register release, then proceed with the real logic. In reviews, I check that the defer is in the correct lexical scope, is reached on all relevant paths, is small and synchronous, and does not hide business side effects. For locks, I prefer scoped APIs like `withLock` when available because they encode the cleanup discipline into the API.

Staff-level questions to ask:

```text
Is this cleanup tied to a successfully acquired resource?
Is the defer placed immediately after acquisition?
Is the defer in the intended scope?
Would a scoped API make this impossible to misuse?
Could this deferred cleanup fail, and if so, where should that failure be handled?
```

---

## 9. Interview-ready summary

`defer` registers synchronous code to run when the current lexical scope exits. It runs on normal exit, early `return`, and `throw`, as long as the `defer` statement was reached. Multiple defers execute in reverse order, which matches resource cleanup patterns. I use it for cleanup, unlocking, restoring temporary state, and small completion bookkeeping. I avoid using it for hidden business logic, async cleanup, or code whose scope is unclear.

---

## 10. Flashcards

Q: When does a Swift `defer` block run?  
A: When execution leaves the lexical scope where the `defer` statement was reached.

Q: Do multiple `defer` blocks run in source order?  
A: No. They run in reverse order: last registered, first executed.

Q: Does `defer` run if a function throws?  
A: Yes, if the `defer` statement was reached before the throw.

Q: Does `defer` run if execution returns before reaching the `defer` statement?  
A: No. A `defer` must be reached before it is registered.

Q: Why is `defer` useful with locks?  
A: It guarantees `unlock()` runs on every exit path after `lock()` succeeds.

Q: Can a `defer` body `return`?  
A: No. Swift forbids transferring control out of a `defer` body.

Q: Can a `defer` body call `await`?  
A: No. `async` calls cannot occur inside a `defer` body.

Q: Why is `defer` inside a loop body often surprising?  
A: It runs at the end of each iteration’s scope, not after the whole loop.

Q: Does mutating a local variable in `defer` change an already evaluated return value?  
A: No for value returns. The return expression is evaluated before defers execute.

Q: What is the best placement for cleanup `defer`?  
A: Immediately after successful resource acquisition.

---

## 11. Related sections

- [[A3 — Initialization model]]
- [[A7 — Functions, parameter labels, defaults, and API surface]]
- [[A13 — Error handling and typed throws]]
- [[C1 — ARC fundamentals]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D12 — Synchronization library, mutexes, and choosing between actors and locks]]

---

## 12. Sources

- Swift Senior/Staff Rubric, A6 — `defer`, early exits, and cleanup semantics.
- The Swift Programming Language — Control Flow / Deferred Actions. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/?utm_source=chatgpt.com "Control Flow - Documentation | Swift.org"))
- The Swift Programming Language — Error Handling / Specifying Cleanup Actions. ([docs.swift.org](https://docs.swift.org/swift-book/LanguageGuide/ErrorHandling.html?utm_source=chatgpt.com "Error Handling - Documentation | Swift.org"))
- Swift Forums discussion clarifying static lexical scope behavior. ([Swift Forums](https://forums.swift.org/t/details-of-defer-statement-in-swift/4483?utm_source=chatgpt.com "Details of defer statement in Swift"))
- Apple Developer Documentation — `Synchronization.Mutex` and `withLock(_:)`. ([Apple Developer](https://developer.apple.com/documentation/Synchronization/Mutex?utm_source=chatgpt.com "Mutex | Apple Developer Documentation"))