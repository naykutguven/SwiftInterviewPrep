---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - performance
  - ownership
  - safety
  - copyable
  - noncopyable
---
## 0. Rubric snapshot

**Rubric expectation**

Know what `Copyable` means, when noncopyable modeling is useful, and how modern Swift lets generic code work with both copyable and noncopyable types. The uploaded rubric places C8 in the low-level ownership/performance section and asks you to model a file handle or lock token as noncopyable.

**Caveats**

Noncopyable types change API design, generic constraints, control flow, and pattern matching. Do not suppress copyability because it “looks more advanced”; do it when the domain truth is single ownership.

**You should be able to answer**

- What does `Copyable` mean semantically, and when do you need to spell it or suppress it?
- Why would you make a Swift type noncopyable?
- How does noncopyability interact with generics and protocol design, especially when one API should support both copyable and noncopyable inputs?

**You should be able to do**

- Model a file handle or lock token as a noncopyable type and explain what misuse it prevents.

---

## 1. Core mental model

Most Swift values are **copyable**. That does not mean Swift eagerly copies memory every time; it means the language is allowed to produce another logically independent value when needed. For normal structs and enums, assignment, argument passing, tuple formation, generic wrapping, and closure capture can all rely on this copyability assumption.

A **noncopyable** type is different: it represents a value with **unique ownership**. There is exactly one owner of that value at a time. Moving ownership is allowed; duplicating ownership is not. Swift Evolution SE-0390 introduced noncopyable structs and enums, also called move-only types, and describes them as values that always have unique ownership, unlike normal Swift values that can be freely copied. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0390-noncopyable-structs-and-enums.md "raw.githubusercontent.com"))

You opt out of implicit copyability with `~Copyable`:

```swift
struct FileDescriptor: ~Copyable {
    private let rawValue: CInt
}
```

This is not “a protocol conformance to noncopyable.” It is a **suppression** of the default `Copyable` assumption. SE-0427 later generalized this model for generics: `Copyable` became the formal capability, most things require it by default, and `~Copyable` suppresses that default where you want generic code to accept either copyable or noncopyable values. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md "swift-evolution/proposals/0427-noncopyable-generics.md at main · swiftlang/swift-evolution · GitHub"))

The key idea:

```text
Copyable value     => Swift may duplicate it when needed.
~Copyable value    => Swift must track one owner; use must be borrowing, consuming, or mutating.
T                  => implicitly T: Copyable
T: ~Copyable       => maybe copyable; accepts both copyable and noncopyable values
```

Noncopyable types are about **semantic ownership**, not just performance. Good examples: file descriptors, lock tokens, unique buffers, cryptographic secret material, linear capabilities, one-shot resources, and C/C++ handles where double-close or use-after-close is a real bug.

---

## 2. Essential mechanics

### `Copyable` is the default capability

Most Swift structs, enums, classes, generic parameters, protocols, existentials, and associated types are implicitly `Copyable` unless you suppress that default. SE-0427 states that every struct, enum, class, generic parameter, protocol, and associated type conforms to `Copyable` by default, and `~Copyable` suppresses the inferred default requirement. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md "swift-evolution/proposals/0427-noncopyable-generics.md at main · swiftlang/swift-evolution · GitHub"))

```swift
struct UserID {
    let rawValue: String
}

// Conceptually:
struct UserID /* : Copyable */ {
    let rawValue: String
}
```

For ordinary app models, this is exactly what you want. `UserID`, `URL`, `Date`, `CGPoint`, and most DTOs should be copyable.

### `~Copyable` suppresses copyability

```swift
struct Token: ~Copyable {
    let id: Int
}
```

This means `Token` values cannot be implicitly duplicated.

```swift
struct Token: ~Copyable {}

func use(_ token: borrowing Token) {}

func demo() {
    let a = Token()
    let b = a
    use(a)
    use(b)
}
```

Swift 6.2.1 compiler output:

```text
/tmp/c8_copy.swift:6:9: error: 'a' used after consume
 4 | 
 5 | func demo() {
 6 |     let a = Token()
   |         `- error: 'a' used after consume
 7 |     let b = a
   |             `- note: consumed here
 8 |     use(a)
   |         `- note: used here
 9 |     use(b)
10 | }
```

`let b = a` does not copy `a`; it **moves/consumes** ownership from `a` into `b`. After that, `a` is invalid.

### Noncopyable parameters force explicit ownership

For noncopyable values, the ownership convention is part of the API contract. SE-0390 describes `consume` as invalidating the value and `borrow` as shared access that does not take ownership. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0390-noncopyable-structs-and-enums.md "raw.githubusercontent.com"))

```swift
struct Token: ~Copyable {
    let id: Int

    borrowing func inspect() {
        print("inspecting \(id)")
    }

    consuming func finish() {
        print("finished \(id)")
    }

    deinit {
        print("deinit \(id)")
    }
}

func demo() {
    let token = Token(id: 1)
    token.inspect()
    token.finish()
}

demo()
```

Output:

```text
inspecting 1
finished 1
deinit 1
```

After a consuming use, the value cannot be used again:

```swift
struct Token: ~Copyable {
    borrowing func inspect() {}
    consuming func finish() {}
}

func demo() {
    let token = Token()
    token.finish()
    token.inspect()
}
```

Swift 6.2.1 compiler output:

```text
/tmp/c8_after_consume.swift:7:9: error: 'token' used after consume
 5 | 
 6 | func demo() {
 7 |     let token = Token()
   |         `- error: 'token' used after consume
 8 |     token.finish()
   |     `- note: consumed here
 9 |     token.inspect()
   |     `- note: used here
10 | }
11 | 
```

### Generic code is `Copyable` unless you opt out

A plain generic parameter still has an implicit `Copyable` requirement.

```swift
struct Token: ~Copyable {}

func inspect<T>(_ value: borrowing T) {}

func demo() {
    let x = Token()
    inspect(x)
}
```

Swift 6.2.1 compiler output:

```text
/tmp/c8_generic_bad.swift:7:5: error: global function 'inspect' requires that 'Token' conform to 'Copyable'
1 | struct Token: ~Copyable {}
2 | 
3 | func inspect<T>(_ value: borrowing T) {}
  |      `- note: 'where T: Copyable' is implicit here
4 | 
5 | func demo() {
6 |     let x = Token()
7 |     inspect(x)
  |     `- error: global function 'inspect' requires that 'Token' conform to 'Copyable'
8 | }
9 | 
```

To support both copyable and noncopyable values, write `T: ~Copyable`:

```swift
struct Token: ~Copyable {}

func inspect<T: ~Copyable>(_ value: borrowing T) {
    _ = value
}

func take<T: ~Copyable>(_ value: consuming T) {
    _ = value
}

func demo() {
    let x = Token()
    inspect(x)
    take(x)
}
```

This compiles because `T: ~Copyable` means “maybe copyable,” not “must be noncopyable.” The Swift book describes `~Copyable` as a suppressed constraint that allows both copyable and noncopyable types. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/?utm_source=chatgpt.com "Generics | Documentation - Swift Programming Language"))

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `~Copyable` means “noncopyable protocol conformance”

Bad mental model:

```swift
struct Token: ~Copyable {}
// "Token conforms to a NonCopyable protocol."
```

Better mental model:

```swift
struct Token: ~Copyable {}
// "Token suppresses the default Copyable capability."
```

`Copyable` is the positive capability. `~Copyable` removes the implicit requirement that the value can be copied.

### Trap 2: Making types noncopyable for style

Bad:

```swift
struct UserProfile: ~Copyable {
    var name: String
    var avatarURL: URL
}
```

This is usually a mistake. A user profile is ordinary value data. Making it noncopyable damages ergonomics: storing it in arrays, passing it through generic APIs, testing it, logging it, and composing it become harder.

Better:

```swift
struct UserProfile: Sendable, Equatable {
    var name: String
    var avatarURL: URL
}
```

Use noncopyability when the domain has **one owner**: a lock token, unique file descriptor, temporary unsafe memory region, one-time continuation-like capability, or secret that should not be implicitly duplicated.

### Trap 3: Forgetting that classes remain copyable references

SE-0390 states that classes may contain noncopyable stored properties, but class declarations themselves cannot use `~Copyable`; class values remain copyable because references can be retained and released. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0390-noncopyable-structs-and-enums.md "raw.githubusercontent.com"))

```swift
final class SharedBox {
    var token: Token

    init(token: consuming Token) {
        self.token = token
    }
}
```

The `Token` is noncopyable, but `SharedBox` references are copyable. That may reintroduce shared ownership around a unique resource. Sometimes that is intentional; often it defeats the point.

### Trap 4: Assuming noncopyable automatically means thread-safe

`~Copyable` prevents implicit duplication. It does not make interior state synchronized, actor-isolated, or `Sendable`.

```swift
struct UniqueBuffer: ~Copyable {
    // Unique ownership, yes.
    // Thread-safe mutation, not automatically.
}
```

You still need concurrency design: `Sendable`, actor isolation, locks, or clear single-thread confinement.

---

## 4. Direct answers to rubric questions

### Q1. What does `Copyable` mean semantically, and when do you need to spell it or suppress it?

`Copyable` means Swift may create another logically equivalent value when language operations need a copy. Most Swift types are implicitly `Copyable`, so you rarely write it. You suppress it with `~Copyable` when duplication would violate the resource model.

Interview version:

> `Copyable` is Swift’s capability for values that can be duplicated by normal value operations. Most Swift code gets this implicitly. I only spell or suppress it when ownership matters: `T` in a generic API means `T: Copyable` by default, while `T: ~Copyable` means the API can work with both copyable and noncopyable values. For a resource like a file descriptor or lock token, I’d make the type `~Copyable` because copying the value would create two apparent owners of one underlying resource.

### Q2. Why would you make a Swift type noncopyable?

You make a type noncopyable when the value represents a unique resource or capability where duplicate ownership is wrong.

Good candidates:

```text
File descriptor
Lock guard/token
Unique unsafe buffer
One-shot resource
C/C++ handle with manual close/free
Secret data that should not be implicitly duplicated
Linear permission/capability token
```

Bad candidates:

```text
DTOs
View state
API response models
Configuration structs
Most SwiftUI/UIKit view models
Ordinary domain values
```

Interview version:

> I’d make a type noncopyable when copying the value would create a correctness bug, not just because I want fewer copies. A file descriptor is a good example: two Swift values pointing at the same descriptor can both try to close it, or one can use it after the other closes it. A noncopyable wrapper lets the compiler enforce single ownership and makes borrowing versus consuming part of the API.

### Q3. How does noncopyability interact with generics and protocol design?

Plain generic parameters and protocols are copyable by default. If a generic API should accept noncopyable values, it must suppress the default with `~Copyable`.

```swift
func copyableOnly<T>(_ value: T) {
    // implicit T: Copyable
}

func acceptsBoth<T: ~Copyable>(_ value: borrowing T) {
    // accepts copyable and noncopyable values
}
```

For protocols, this matters because a protocol can accidentally exclude noncopyable conformers if its `Self` or associated types implicitly require `Copyable`. SE-0427 was specifically introduced to close the expressivity gap where noncopyable types could not participate in generics, protocols, or existentials. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md "swift-evolution/proposals/0427-noncopyable-generics.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> In modern Swift, generic APIs are still copyable unless you opt out. That means `func f<T>(_ value: T)` is effectively constrained to `T: Copyable`. If I’m designing a low-level API that should accept a unique buffer or file token, I need `T: ~Copyable` and ownership-qualified parameters like `borrowing` or `consuming`. For protocols, I’d be deliberate because protocol requirements, associated types, and existential use can easily reintroduce copyability assumptions.

---

## 5. Code probe

The rubric has no dedicated C8 code probe, so use these three probes instead. The answer format follows the uploaded Swift section template’s structure for rubric snapshot, mechanics, traps, direct answers, exercise, production guidance, senior/staff signal, flashcards, related sections, and sources.

### 5.1 Minimal valid example

Given:

```swift
struct Token: ~Copyable {
    let id: Int

    borrowing func inspect() {
        print("inspecting \(id)")
    }

    consuming func finish() {
        print("finished \(id)")
    }

    deinit {
        print("deinit \(id)")
    }
}

func demo() {
    let token = Token(id: 1)
    token.inspect()
    token.finish()
}

demo()
```

What happens?

```text
inspecting 1
finished 1
deinit 1
```

Why?

```text
token.inspect() borrows token.
token.finish() consumes token.
After finish(), token's lifetime ends.
deinit runs once for the unique value.
```

### 5.2 Counterexample: accidental “copy”

Given:

```swift
struct Token: ~Copyable {}

func use(_ token: borrowing Token) {}

func demo() {
    let a = Token()
    let b = a
    use(a)
    use(b)
}
```

What happens?

```text
/tmp/c8_copy.swift:6:9: error: 'a' used after consume
 4 | 
 5 | func demo() {
 6 |     let a = Token()
   |         `- error: 'a' used after consume
 7 |     let b = a
   |             `- note: consumed here
 8 |     use(a)
   |         `- note: used here
 9 |     use(b)
10 | }
```

Why?

```text
let b = a
```

does not copy `a`. It transfers ownership from `a` into `b`. Since `a` is noncopyable, using `a` afterward is invalid.

### 5.3 Production-style example: lock token

A lock token is a good noncopyable model because copying it would imply multiple owners of one lock acquisition. That can cause double-unlock, unlock-after-transfer, or unclear critical-section ownership.

```swift
import Foundation

public struct LockToken: ~Copyable {
    private let lock: NSLock
    private var isActive = true

    fileprivate init(lock: NSLock) {
        self.lock = lock
        lock.lock()
    }

    public mutating func unlock() {
        guard isActive else { return }
        isActive = false
        lock.unlock()
    }

    deinit {
        if isActive {
            lock.unlock()
        }
    }
}

public final class Gate: @unchecked Sendable {
    private let lock = NSLock()

    public init() {}

    public func acquire() -> LockToken {
        LockToken(lock: lock)
    }
}

func useProtectedResource() {
    let gate = Gate()

    var token = gate.acquire()
    // protected work here
    token.unlock()
}
```

Misuse attempt:

```swift
func misuse() {
    let gate = Gate()
    let first = gate.acquire()
    let second = first
    _ = second
    _ = first
}
```

Swift 6.2.1 compiler output:

```text
/tmp/c8_lock_bad.swift:20:9: error: 'first' used after consume
18 | func misuse() {
19 |     let gate = Gate()
20 |     let first = gate.acquire()
   |         `- error: 'first' used after consume
21 |     let second = first
   |                  `- note: consumed here
22 |     _ = second
23 |     _ = first
   |         `- note: used here
24 | }
25 | 
```

Why this design is correct:

```text
The lock acquisition is represented by exactly one LockToken.
Moving the token transfers the responsibility to unlock.
The token's deinit is a fallback cleanup boundary.
The compiler prevents two token values from both believing they own the same acquisition.
```

Alternative fixes and tradeoffs:

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`defer { lock.unlock() }`|Simple local critical section|Easy, but ownership is not represented as a value|
|Closure API: `withLock { ... }`|Preferred high-level API|Prevents escaping the token, but less flexible for staged workflows|
|Noncopyable `LockToken`|Capability-style API or staged lifetime|More advanced; generic/protocol ergonomics are harder|
|Class-based token|Needs Objective-C/reference interop|Reintroduces shared ownership and ARC overhead|

---

## 6. Exercise

### Problem

Model a file handle or lock token as a noncopyable type and explain what misuse it prevents.

### Bad / naive version

```swift
import Foundation

struct NaiveLockToken {
    private let lock: NSLock

    init(lock: NSLock) {
        self.lock = lock
        lock.lock()
    }

    func unlock() {
        lock.unlock()
    }
}

func misuse(lock: NSLock) {
    let first = NaiveLockToken(lock: lock)
    let second = first

    first.unlock()
    second.unlock()
}
```

### What is wrong?

```text
NaiveLockToken is copyable.
first and second both represent the same lock acquisition.
Both can call unlock().
The type permits double-unlock because ownership is not modeled.
```

### Improved version

```swift
import Foundation

public struct LockToken: ~Copyable {
    private let lock: NSLock
    private var isActive = true

    fileprivate init(lock: NSLock) {
        self.lock = lock
        lock.lock()
    }

    public mutating func unlock() {
        guard isActive else { return }
        isActive = false
        lock.unlock()
    }

    deinit {
        if isActive {
            lock.unlock()
        }
    }
}

public final class Gate: @unchecked Sendable {
    private let lock = NSLock()

    public init() {}

    public func acquire() -> LockToken {
        LockToken(lock: lock)
    }

    public func withLock<R>(_ body: () throws -> R) rethrows -> R {
        var token = acquire()
        defer {
            token.unlock()
        }
        return try body()
    }
}
```

### Why this is better

The noncopyable token makes lock ownership explicit. There can be only one owner responsible for releasing the acquisition. The `withLock` API remains the simpler default for most production code, while `acquire()` exposes a lower-level capability for cases where the critical section must span multiple helper calls.

In production, prefer this shape:

```swift
gate.withLock {
    // critical section
}
```

Expose the noncopyable token only when the caller genuinely needs staged ownership.

---

## 7. Production guidance

Use this in production when:

```text
A value represents exactly one owned resource.
Copying the value would cause double-close, double-free, double-unlock, or use-after-close.
The API boundary benefits from compile-time ownership enforcement.
You are wrapping C, C++, POSIX, crypto, memory, file, socket, or synchronization primitives.
```

Be careful when:

```text
The type must participate heavily in generic containers, existentials, Codable, Equatable, or logging.
The team is not ready to reason about borrowing/consuming APIs.
The value crosses concurrency domains and also needs Sendable reasoning.
The type is public API; adding or removing ~Copyable changes the API contract.
```

Avoid when:

```text
The type is ordinary value data.
You only want performance.
You are trying to prevent mutation rather than copying.
A simple closure API would express the lifetime more safely.
A class/actor is actually needed because shared identity is the domain model.
```

Debugging checklist:

```text
Is this value truly uniquely owned?
Which operation borrows it?
Which operation consumes it?
After consumption, is there any attempted use?
Did a generic function accidentally require Copyable?
Did a protocol or associated type accidentally exclude noncopyable conformers?
Is a class wrapper reintroducing shared ownership?
Does deinit perform exactly-once cleanup?
Would withX { ... } be simpler than exposing a token?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `~Copyable` means a Swift value cannot be copied.

### Senior answer

> `~Copyable` suppresses Swift’s default `Copyable` assumption. It is useful for unique resources where assignment or passing should move ownership instead of duplicate it. APIs using these values need explicit `borrowing`, `consuming`, or `inout` ownership.

### Staff-level answer

> I would treat noncopyability as an API design tool for ownership truth. It belongs at narrow boundaries: file descriptors, locks, unique buffers, secrets, low-level interop handles. I would keep most app-layer models copyable. For public APIs, I’d think carefully before exposing `~Copyable`, because it affects generics, protocols, existentials, source compatibility, testing, and adoption. I’d usually pair a safe high-level closure API with a lower-level noncopyable capability only where staged ownership is required.

Staff-level questions to ask:

```text
Does this type model data, identity, or unique ownership?
Is noncopyability visible in public API, or hidden behind a safe wrapper?
Can clients accidentally regain shared ownership through a class box?
Should the generic API require T: Copyable, or should it support T: ~Copyable?
Is cleanup guaranteed exactly once across success, failure, early return, and deinit?
```

---

## 9. Interview-ready summary

`Copyable` is Swift’s default capability for values that can be duplicated by normal language operations. `~Copyable` suppresses that default and gives you a move-only value with unique ownership. That is useful for resources like file descriptors, lock tokens, unique buffers, or secrets where accidental duplication creates correctness bugs. The important design shift is that APIs must say whether they borrow, consume, or mutate the value. In generic code, plain `T` still implicitly means `T: Copyable`; use `T: ~Copyable` when the API should support both copyable and noncopyable values.

---

## 10. Flashcards

Q: What does `Copyable` mean in Swift?  
A: It means Swift may duplicate a value when language operations require a copy. Most Swift types are implicitly `Copyable`.

Q: What does `~Copyable` mean?  
A: It suppresses the default `Copyable` requirement. It does not mean “conforms to a NonCopyable protocol.”

Q: Does `T: ~Copyable` mean `T` must be noncopyable?  
A: No. It means “maybe copyable”; the generic API can accept both copyable and noncopyable values.

Q: Why make a file descriptor wrapper noncopyable?  
A: To prevent multiple Swift values from believing they each own the same OS handle, which can cause double-close or use-after-close.

Q: What happens when assigning a noncopyable value to another variable?  
A: Ownership is consumed/moved into the new binding; the original binding cannot be used afterward.

Q: Are classes allowed to be `~Copyable`?  
A: No. Class references remain copyable; classes can contain noncopyable stored properties, but the reference itself is still shared.

Q: What are the three main ownership modes for noncopyable APIs?  
A: `borrowing`, `consuming`, and `inout`/`mutating`.

Q: When is `~Copyable` a bad idea?  
A: For ordinary app data, DTOs, view state, and domain models where copying is semantically fine and ergonomic generic use matters.

---

## 11. Related sections

- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[E3 — C interoperability and unsafe boundaries]]
- [[E4 — C++ interoperability]]

---

## 12. Sources

- Uploaded Swift Senior/Staff Rubric, C8 section.
- Swift Evolution SE-0390: Noncopyable structs and enums. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0390-noncopyable-structs-and-enums.md "raw.githubusercontent.com"))
- Swift Evolution SE-0427: Noncopyable generics. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md "swift-evolution/proposals/0427-noncopyable-generics.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Evolution SE-0432: Borrowing and consuming pattern matching for noncopyable types. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0432-noncopyable-switch.md "swift-evolution/proposals/0432-noncopyable-switch.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Book, Generics: `~Copyable` as a suppressed constraint / “maybe copyable.” ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/?utm_source=chatgpt.com "Generics | Documentation - Swift Programming Language"))