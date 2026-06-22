---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - ownership-modifiers
---
## 0. Rubric snapshot

**Rubric expectation**

Understand Swift’s direction toward explicit ownership for performance-sensitive and move-aware code.

**Caveats**

Ownership annotations are semantically meaningful. They are not micro-optimizations to sprinkle randomly.

**You should be able to answer**

- What problem do `borrowing` and `consuming` solve?
- Why is explicit ownership more than a performance feature for low-level APIs?

**You should be able to do**

- Explain how a resource-holding type should expose an operation that consumes the resource exactly once.

This note follows the attached Swift Section Answer Template.

---

## 1. Core mental model

Swift normally hides ownership traffic from you. You pass values around, and the compiler inserts retains, releases, copies, destroys, and copy-on-write uniqueness checks as needed. That is usually exactly what you want.

Ownership modifiers let an API say **how a value is received**:

```text
borrowing T  = callee may temporarily use T; caller keeps ownership
consuming T  = callee takes ownership of T
inout T      = callee gets exclusive mutable access and must leave T valid
```

`borrowing` means the callee gets shared, temporary access. It should not take the value apart, store it as its own owned value, or end its lifetime. The caller can continue using the value after the call.

`consuming` means the callee receives ownership. It is responsible for storing, forwarding, or destroying the value. For ordinary copyable types, this mostly gives the compiler and API author more control over copies and ARC traffic. For noncopyable/resource types, it becomes part of correctness: after a value is consumed, it must not be used again.

The `consume` operator is caller-side ownership transfer. It explicitly ends the lifetime of a local binding and makes later use a compiler error. SE-0366 describes `consume` as ending the lifetime of a local `let`, local `var`, or function parameter, and emitting diagnostics for use after consume. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0366-move-function.md "raw.githubusercontent.com"))

The key idea:

```text
Ownership modifiers make lifetime transfer explicit enough for the compiler, API readers, and resource-safe designs.
```

---

## 2. Essential mechanics

### 2.1 `borrowing` means temporary shared use

A `borrowing` parameter lets the callee inspect or use a value without taking ownership of it.

```swift
func log(_ message: borrowing String) {
    print(message)
}

func demo() {
    let message = "Saved"
    log(message)
    print(message) // OK: still usable
}
```

For copyable types, Swift often defaults ordinary function parameters to borrowing-like behavior when that is efficient. SE-0377 says most functions default to borrowing, while initializers and setters are more likely to consume because they often store or forward values. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))

The important part is not syntax. It is the contract:

```text
The callee does not own the value.
The caller must keep it alive during the call.
The callee should not require an independent copy unless it explicitly asks for one.
```

---

### 2.2 `consuming` means ownership transfer into the callee

A `consuming` parameter means the callee receives an owned value.

```swift
func appendTerminator(_ bytes: consuming [UInt8]) -> [UInt8] {
    var result = consume bytes
    result.append(0)
    return result
}

func demo() {
    let bytes: [UInt8] = [1, 2, 3]
    let finalized = appendTerminator(consume bytes)
    print(finalized)
}
```

Exact output:

```text
[1, 2, 3, 0]
```

SE-0377 defines the two conventions this way: a callee can borrow a parameter, where the caller guarantees the object stays alive during the call, or consume it, where the callee becomes responsible for releasing, destroying, or forwarding ownership. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))

For value types, “retain” generalizes to an independent copy, and “release” generalizes to destruction of that copy. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))

---

### 2.3 `consume x` ends the lifetime of a binding

`consume` is not just “pass this efficiently.” It invalidates the local binding.

```swift
func sink(_ value: consuming [Int]) -> Int {
    var local = consume value
    local.append(4)
    return local.count
}

func demo() {
    var numbers = [1, 2, 3]
    print(sink(consume numbers))
    print(numbers)
}
```

Exact Swift 6.2.1 compiler error:

```text
/tmp/c7.swift:8:9: warning: variable 'numbers' was never mutated; consider changing to 'let' constant
 6 | 
 7 | func demo() {
 8 |     var numbers = [1, 2, 3]
   |         `- warning: variable 'numbers' was never mutated; consider changing to 'let' constant
 9 |     print(sink(consume numbers))
10 |     print(numbers)

/tmp/c7.swift:8:9: error: 'numbers' used after consume
 6 | 
 7 | func demo() {
 8 |     var numbers = [1, 2, 3]
   |         `- error: 'numbers' used after consume
 9 |     print(sink(consume numbers))
   |                        `- note: consumed here
10 |     print(numbers)
   |           `- note: used here
```

SE-0366 states that `consume` forces ownership transfer out of the binding at that point; any later reachable use of that binding is an error. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0366-move-function.md "raw.githubusercontent.com"))

---

### 2.4 `borrowing` and `consuming` bindings are not implicitly copyable

This is the part many engineers miss.

```swift
func duplicate(_ x: borrowing String) -> (String, String) {
    (x, x)
}
```

Exact Swift 6.2.1 compiler error:

```text
/tmp/c7_borrow.swift:1:18: error: 'x' consumed more than once
1 | func duplicate(_ x: borrowing String) -> (String, String) {
  |                  `- error: 'x' consumed more than once
2 |     (x, x)
3 | }
  | `- note: multiple consumes here
```

Correct version:

```swift
func duplicate(_ x: borrowing String) -> (String, String) {
    (copy x, copy x)
}

print(duplicate("abc"))
```

Exact output:

```text
("abc", "abc")
```

SE-0377 says `borrowing` and `consuming` parameter bindings are not implicitly copyable inside the function body; use `copy x` when you intentionally need an independent copy. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))

---

### 2.5 For noncopyable values, ownership is API correctness

For noncopyable/resource types, a parameter must say whether it borrows, consumes, or mutates the value. SE-0390 states that noncopyable function parameters must explicitly declare `borrowing`, `consuming`, or `inout`, because the ownership convention is part of the API contract. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0390-noncopyable-structs-and-enums.md "raw.githubusercontent.com"))

```swift
struct FileToken: ~Copyable {
    private var descriptor: Int32
    private var isClosed = false

    init(descriptor: Int32) {
        self.descriptor = descriptor
    }

    borrowing func read() {
        print("read", descriptor)
    }

    consuming func close() {
        if !isClosed {
            print("close", descriptor)
            isClosed = true
        }
    }

    deinit {
        if !isClosed {
            print("auto-close", descriptor)
        }
    }
}

func demo() {
    let token = FileToken(descriptor: 42)
    token.read()
    token.close()
}

demo()
```

Exact output:

```text
read 42
close 42
```

Trying to use the token after `close()` fails:

```swift
func demo() {
    let token = FileToken(descriptor: 42)
    token.read()
    token.close()
    token.read()
}
```

Exact Swift 6.2.1 compiler error:

```text
/tmp/c7_handle.swift:28:9: error: 'token' used after consume
26 | 
27 | func demo() {
28 |     let token = FileToken(descriptor: 42)
   |         `- error: 'token' used after consume
29 |     token.read()
30 |     token.close()
   |     `- note: consumed here
31 |     token.read()
   |     `- note: used here
```

This is the real payoff: resource protocols become compile-time enforceable instead of convention-based.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `consuming` means “always faster”

Bad:

```swift
func render(_ model: consuming ViewModel) {
    // No ownership-sensitive need here.
}
```

Better:

```swift
func render(_ model: borrowing ViewModel) {
    // Rendering observes the model; it does not take ownership.
}
```

`consuming` can reduce copies or ARC traffic in specific designs, but it also changes the ownership contract. For ABI-stable public APIs, SE-0377 notes that ownership convention affects the ABI-level calling convention and cannot freely change later. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))

Use it when the API really owns, stores, destroys, forwards, or finalizes the value.

---

### Trap 2: Thinking `consume` consumes the object globally

`consume` consumes a **binding**, not “all aliases to the same object.”

```swift
final class Box {}

func demo() {
    let a = Box()
    let b = a

    _ = consume a

    print(b) // OK: b is a separate binding to the same object
}
```

For classes, references are copyable. Consuming one binding does not prove the object has no other references. For true “exactly once” resource semantics, prefer a noncopyable type when possible.

---

### Trap 3: Using a copyable class for single-owner resources

Bad:

```swift
final class FileHandleBox {
    let fd: Int32
    var isClosed = false

    init(fd: Int32) {
        self.fd = fd
    }

    func close() {
        guard !isClosed else { return }
        isClosed = true
        print("close", fd)
    }
}

func bug() {
    let a = FileHandleBox(fd: 42)
    let b = a

    a.close()
    b.close() // Runtime convention required.
}
```

Better:

```swift
struct FileToken: ~Copyable {
    private var fd: Int32
    private var isClosed = false

    init(fd: Int32) {
        self.fd = fd
    }

    consuming func close() {
        if !isClosed {
            print("close", fd)
            isClosed = true
        }
    }

    deinit {
        if !isClosed {
            print("auto-close", fd)
        }
    }
}
```

The class version shares ownership through references. The noncopyable version models unique ownership directly.

---

### Trap 4: Treating `borrowing` as “just `let`”

`let` is about whether a local binding can be reassigned. `borrowing` is about whether the callee owns the value.

```swift
func inspect(_ value: borrowing String) {
    print(value)
}
```

A borrowed parameter can be used, passed to other borrowing operations, and inspected. But it cannot be implicitly copied into owned storage unless you write `copy`.

---

### Trap 5: Assuming this replaces `inout`

These are different contracts:

```text
borrowing  = shared temporary access
consuming  = take ownership; old binding ends
inout      = exclusive mutable access; caller's variable remains valid after mutation
```

Use `inout` when the caller expects the same variable to be updated. Use `consuming` when the old value should no longer be usable.

---

## 4. Direct answers to rubric questions

### Q1. What problem do `borrowing` and `consuming` solve?

They let APIs explicitly state whether a parameter is temporarily borrowed or ownership is transferred into the callee.

For normal copyable Swift, this gives API authors control over copies, ARC traffic, and ABI-level calling conventions in performance-sensitive or library-level code. SE-0377 says the optimizer can sometimes infer better conventions, but not always, especially across public ABI boundaries, non-final class methods, and protocol requirements. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))

For noncopyable/resource types, the problem is deeper: Swift must know whether an operation leaves the value usable. A `read` operation should borrow a file handle. A `close` operation should consume it. SE-0377 uses exactly this distinction: borrowing a noncopyable file handle leaves it valid; consuming it prevents further use. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))

Interview version:

> `borrowing` and `consuming` make parameter ownership explicit. `borrowing` means the callee temporarily uses a value while the caller keeps ownership. `consuming` means the callee takes ownership and is responsible for storing, forwarding, or destroying the value. For copyable types this can control copies and ARC traffic; for noncopyable or resource-holding types it becomes part of correctness because it determines whether the caller may use the value again.

---

### Q2. Why is explicit ownership more than a performance feature for low-level APIs?

Because ownership is often a semantic property of the domain.

A file descriptor, lock token, transaction, audio buffer lease, GPU resource, or one-shot continuation-like token may have a lifecycle rule: use it while valid, then consume it exactly once. If this is represented with a copyable class or struct, correctness depends on comments, runtime flags, and discipline. If it is represented with noncopyability plus `borrowing`/`consuming`, misuse becomes a compiler error.

SE-0390 explains that noncopyable values always have unique ownership and are useful for resources where normal copyable structs and enums are not a good model. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0390-noncopyable-structs-and-enums.md "raw.githubusercontent.com"))

Interview version:

> Explicit ownership is not only about removing retains. It lets the type system encode resource lifetime. A low-level API can say “this operation only borrows the handle” or “this operation consumes and closes the handle.” That prevents double close, use after close, and accidental shared ownership. Performance improves as a side effect, but the bigger win is that the API states the ownership truth.

---

## 5. Code probe

The rubric has no dedicated C7 code probe. Use these probes instead.

### Probe A — `consume` creates use-after-consume diagnostics

Given:

```swift
func sink(_ value: consuming [Int]) -> Int {
    var local = consume value
    local.append(4)
    return local.count
}

func demo() {
    var numbers = [1, 2, 3]
    print(sink(consume numbers))
    print(numbers)
}
```

### What happens?

```text
error: 'numbers' used after consume
```

Full compiler diagnostic:

```text
/tmp/c7.swift:8:9: error: 'numbers' used after consume
 6 | 
 7 | func demo() {
 8 |     var numbers = [1, 2, 3]
   |         `- error: 'numbers' used after consume
 9 |     print(sink(consume numbers))
   |                        `- note: consumed here
10 |     print(numbers)
   |           `- note: used here
```

### Why?

`consume numbers` ends the lifetime of the local binding `numbers`. The callee receives ownership. After that, the local binding is invalid unless it is a `var` and is reinitialized before use.

```text
numbers owns [1, 2, 3]
        |
consume numbers
        v
sink owns [1, 2, 3]
        |
numbers is invalid
```

### Fix or redesign

```swift
func demo() {
    let numbers = [1, 2, 3]
    print(sink(consume numbers))
}
```

Exact output:

```text
4
```

### Why this fix is correct

The old binding is not used after ownership transfer. The code now reflects the real lifetime: create buffer, transfer it, stop using original binding.

---

### Probe B — borrowed parameters are not implicitly copyable

Given:

```swift
func duplicate(_ x: borrowing String) -> (String, String) {
    (x, x)
}
```

### What happens?

```text
error: 'x' consumed more than once
```

### Fix

```swift
func duplicate(_ x: borrowing String) -> (String, String) {
    (copy x, copy x)
}
```

### Why this fix is correct

`copy x` explicitly requests independent owned values. SE-0377 says the no-implicit-copy rule is attached to `borrowing` and `consuming` bindings, and `copy x` is the way to allow copies intentionally. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Keep `borrowing`, use `copy x`|You want to make copies explicit|More syntax, but clearer ownership|
|Remove `borrowing`|Ordinary app code with no ownership-sensitive need|Loses the no-implicit-copy guard|
|Redesign return type|Duplication is semantically suspicious|May require a better domain model|

---

## 6. Exercise

### Problem

Explain how a resource-holding type should expose an operation that consumes the resource exactly once.

### Bad / naive version

```swift
final class LockToken {
    private var isUnlocked = false

    func unlock() {
        guard !isUnlocked else {
            assertionFailure("Double unlock")
            return
        }

        isUnlocked = true
        print("unlock")
    }
}

func demo() {
    let token = LockToken()
    let alias = token

    token.unlock()
    alias.unlock() // Runtime failure path.
}
```

### What is wrong?

```text
The class reference is copyable.
Multiple aliases can call unlock().
The type relies on runtime state and discipline.
The compiler cannot prove single ownership.
```

This design may be acceptable when shared identity is intended. It is a poor model for a linear, one-shot ownership token.

### Improved version

```swift
struct LockToken: ~Copyable {
    private var isUnlocked = false

    init() {}

    borrowing func assertHeld() {
        print("lock is held")
    }

    consuming func unlock() {
        if !isUnlocked {
            print("unlock")
            isUnlocked = true
        }
    }

    deinit {
        if !isUnlocked {
            print("auto-unlock")
        }
    }
}

func demo() {
    let token = LockToken()
    token.assertHeld()
    token.unlock()
}
```

Exact output:

```text
lock is held
unlock
```

Invalid use:

```swift
func demo() {
    let token = LockToken()
    token.unlock()
    token.assertHeld()
}
```

Compiler behavior:

```text
error: 'token' used after consume
```

### Why this is better

The type models ownership truth:

```text
assertHeld() borrows the token
unlock() consumes the token
after unlock(), the token cannot be used
deinit handles cleanup if ownership ends without explicit unlock
```

This is stronger than “remember not to call unlock twice.” The API makes invalid lifecycle states unrepresentable.

---

## 7. Production guidance

Use this in production when:

```text
You are designing low-level/resource-owning APIs.
You need to model one-shot operations: close, finish, unlock, commit, cancel, consume.
You are writing performance-sensitive generic/library code where copies or ARC traffic matter.
You are designing APIs intended to work with noncopyable values.
You want public protocol requirements to express whether conformers borrow or consume values.
```

Be careful when:

```text
The type is ordinary app-layer data.
The team is not yet comfortable reading ownership diagnostics.
The ownership annotation would be added only because it “seems faster.”
The API is public/ABI-stable and changing ownership later would be costly.
You are working with classes where consuming one reference binding does not prove unique object ownership.
```

Avoid when:

```text
The API merely observes data and normal parameters are enough.
The resource is intentionally shared.
A simpler struct/class/actor design expresses the lifecycle better.
You cannot explain what is being borrowed, consumed, or invalidated.
```

Debugging checklist:

```text
Is the operation observing, mutating, storing, or ending the value?
Should the caller be able to use the value after the call?
Is this a copyable value where aliases still exist?
Is this a unique resource that should be ~Copyable?
Does the implementation accidentally copy a borrowing/consuming parameter?
Should an explicit copy x appear?
Should the operation be borrowing, consuming, mutating, or inout?
Is this public API where ownership convention affects ABI?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `borrowing` avoids copying and `consuming` takes a value.

This is directionally true but incomplete.

### Senior answer

> `borrowing` gives the callee temporary access while the caller keeps ownership. `consuming` transfers ownership to the callee. `consume` at the call site ends the local binding’s lifetime and catches use-after-consume. These tools matter for copy control, CoW uniqueness, ARC traffic, and noncopyable resource APIs.

### Staff-level answer

> Ownership annotations are part of API design. For ordinary app code, I would avoid scattering them unless the ownership contract is meaningful. For low-level libraries, protocol requirements, noncopyable types, and resource wrappers, I would use `borrowing` for read-like operations, `consuming` for terminal operations, and `inout` for replacement/mutation. I would also consider source and ABI stability, team readability, diagnostics quality, and whether the type should be noncopyable so the compiler can enforce exactly-once lifecycle rules.

Staff-level questions to ask:

```text
Is this operation logically read-only, mutating, or terminal?
Should the value remain usable after this call?
Is the type actually unique ownership, or merely a copyable reference?
Would noncopyability make invalid lifecycle states unrepresentable?
Is this annotation local implementation detail or public API contract?
Could changing this convention later break ABI or source for clients?
Will this improve measured performance or just add complexity?
```

---

## 9. Interview-ready summary

`borrowing`, `consuming`, and `consume` make ownership explicit in Swift. `borrowing` means the callee temporarily uses a value and the caller keeps ownership. `consuming` means ownership is transferred to the callee, which must store, forward, or destroy it. `consume x` ends the lifetime of a local binding and makes later use a compiler error. For copyable types this helps control copies, CoW uniqueness, ARC traffic, and ABI-level calling conventions. For noncopyable resource types it is a correctness feature: APIs can encode that `read` borrows a handle while `close` consumes it, preventing use-after-close and double-close at compile time.

---

## 10. Flashcards

Q: What does `borrowing` mean for a parameter?  
A: The callee gets temporary shared access. The caller keeps ownership and can use the value after the call.

Q: What does `consuming` mean for a parameter?  
A: The callee takes ownership and is responsible for storing, forwarding, or destroying the value.

Q: What does `consume x` do?  
A: It ends the lifetime of the local binding `x`; later reachable use of `x` is a compiler error.

Q: Does `consume` destroy all aliases to a class object?  
A: No. It consumes one binding. Other bindings or references may still exist.

Q: Why are `borrowing` and `consuming` useful for noncopyable types?  
A: They tell the compiler whether an operation leaves a value usable or consumes it permanently.

Q: What should a terminal operation like `close()` on a unique file handle usually be?  
A: A `consuming` method.

Q: What should a read-like operation on a unique resource usually be?  
A: A `borrowing` method.

Q: Are `borrowing` and `consuming` just performance hints?  
A: No. They are semantic ownership contracts and can affect correctness, source design, and ABI.

Q: When do you use `copy x`?  
A: When a `borrowing` or `consuming` binding needs an explicit independent owned copy.

Q: How does `consuming` differ from `inout`?  
A: `consuming` transfers ownership and invalidates the old value; `inout` gives exclusive mutable access and must leave the caller’s variable valid.

---

## 11. Related sections

- [[C1 — ARC fundamentals]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C8 — Copyable, noncopyable types, and ~Copyable]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[E7 — Resilience, ABI, and module stability]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist.
- SE-0377: `borrowing` and `consuming` parameter ownership modifiers. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0377-parameter-ownership-modifiers.md "raw.githubusercontent.com"))
- SE-0366: `consume` operator to end the lifetime of a variable binding. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0366-move-function.md "raw.githubusercontent.com"))
- SE-0390: Noncopyable structs and enums. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-evolution/main/proposals/0390-noncopyable-structs-and-enums.md "raw.githubusercontent.com"))