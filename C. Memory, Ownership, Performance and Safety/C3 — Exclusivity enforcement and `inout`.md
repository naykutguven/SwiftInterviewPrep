---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - inout
  - exclusivity
---

# C3 — Exclusivity Enforcement and `inout`

**Learning goals:**

- Locate storage and the duration of conflicting reads/writes, including array elements and mutating self.
- Distinguish argument evaluation, scoped inout access, unsafe addresses, and dynamic enforcement.
- Use borrow/mutate accessors and MutableRef without extending an exclusive access across conflicting owner uses.

**Prerequisites:** [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for parameter passing, [[A2 — `let`, `var`, `mutating`, and method semantics]] for mutation, and [[C2 — Copy-on-write behavior of standard-library types]] for array snapshots. [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]] develops noncopyable ownership and dependent references.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A swap helper needs permission to change two values. Passing the same variable twice asks it to hold two simultaneous mutable accesses to one piece of storage. Swift’s **exclusivity** rule prevents conflicting access while mutation is active; some conflicts are caught by compilation and others by runtime checks.

An inout parameter grants scoped exclusive mutable access for the call. The **copy-in/copy-out model** explains the observable result: read the value, let the function modify it, then write it back. It does not promise an actual temporary or stable address. Mutating methods similarly need exclusive access to self. [Memory safety](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Identify the two access scopes

Shared exchange setup. first and second below are independent local storage.

```swift
func exchange(_ first: inout Int, _ second: inout Int) {
    let saved = first
    first = second
    second = saved
}
func independentProbe() {
    var first = 1
    var second = 2
    exchange(&first, &second)
    precondition(first == 2 && second == 1)
}
independentProbe()
```

**Walkthrough:** Both mutable accesses remain active for exchange, but they target different variables. The helper’s correctness requires that distinction; & is a request for scoped access, not an independent alias a caller can retain.

### Computed properties can provide inout through accessors

Standalone log. The property deliberately has an ordinary get/set pair so its access path is visible.

```swift
final class Log { var events: [String] = [] }
struct Meter {
    private var stored = 0
    let log: Log
    init(log: Log) { self.log = log }
    var value: Int {
        get {
            log.events.append("get")
            return stored
        }
        set {
            log.events.append("set")
            stored = newValue
        }
    }
}
func addTwo(_ value: inout Int) { value += 2 }
let log = Log()
var meter = Meter(log: log)
addTwo(&meter.value)
precondition(log.events == ["get", "set"])
precondition(meter.value == 2)
```

**Walkthrough:** The call reads and writes through the computed property. The logged get/set pair is checked before the final value read adds another get. This is why an inout contract must not be described as a permanently stored pointer to a field.

## Rules and pitfalls

### Different element indices can still request overlapping array access

**Common pitfall:** Passing two array elements to two inout parameters. Prepend exchange setup.

**Mistake:** Compile-only; intentionally rejected.

```swift
func invalidArraySwap() {
    var values = [1, 2]
    exchange(&values[0], &values[1])
}
```

**Consequence:** The compiler rejects the overlapping mutable accesses through values.

**Why:** Each element argument needs mutable access through the array value for the call. Distinct integer indices do not make those caller-side access scopes independent.

**Correction:** Use the array operation designed to express this change. This standalone correction also checks the same-index policy.

```swift
var values = [1, 2]
values.swapAt(0, 1)
precondition(values == [2, 1])
values.swapAt(0, 0)
precondition(values == [2, 1])
```

**What changes:** swapAt encapsulates the storage mutation instead of requesting two overlapping caller accesses. Staging independent locals is another option when writeback and equal-index behavior are specified explicitly.

### An implicit pointer argument is not a lasting alias

**Common pitfall:** Storing a pointer produced by & and using it after the pointer-taking call. Compile-only; the escaped-address access is never run.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
final class PointerSink {
    var saved: UnsafeMutablePointer<Int>?
    func remember(_ pointer: UnsafeMutablePointer<Int>) { saved = pointer }
}
func invalidAddress() {
    var value = 1
    let sink = PointerSink()
    sink.remember(&value)
    sink.saved!.pointee += 1
}
```

**Consequence:** The stored address has no supported lifetime for the later access, even if a particular stack arrangement appears to work.

**Why:** Swift’s implicit pointer conversion grants access for the call, not a retained pointer contract. Inout itself also does not grant a freely escaping alias.

**Correction:** Keep ordinary mutation in an inout operation. If a native pointer is required, finish its documented use synchronously within that call.

```swift
func increment(_ value: inout Int) { value += 1 }
var value = 1
increment(&value)
precondition(value == 2)
```

**What changes:** The caller regains ordinary access after the mutation finishes. A native API that retains memory needs explicit ownership until completion; [[C6 — Unsafe pointers, buffers, and raw memory|C6]] explains that different boundary.

## Worked example

### A value snapshot is evaluated before mutable access begins

**Setup:** Standalone by-value/inout example. The returned value is the pre-write snapshot.

**Predict:** Does reading the array as a value conflict with changing its first element in the same call?

```swift
func writeFirst(_ snapshot: [Int], element: inout Int) -> [Int] {
    element = 9
    return snapshot
}
var values = [1, 2]
let before = writeFirst(values, element: &values[0])
precondition(before == [1, 2] && values == [9, 2])
```

**Outcome:** The snapshot remains [1, 2], and values becomes [9, 2].

**Why:** Ordinary value arguments are evaluated before inout access starts. The body reads its independent Array value, not values through another active alias. This does not make two inout element arguments legal.

## Interview questions

### Why is inout different from C++ references or pointers?

**Answer:** It is a scoped access contract, potentially supplied by computed accessors, rather than a freely storable address. The callee must leave the caller’s storage initialized, and conflicting accesses remain forbidden during the call.

**Example:** Standalone runnable example.

```swift
struct Counter {
    var stored: Int
    var doubled: Int {
        get { stored * 2 }
        set { stored = newValue / 2 }
    }
}
func replace(_ value: inout Int, with replacement: Int) { value = replacement }
var counter = Counter(stored: 3)
replace(&counter.doubled, with: 10)
precondition(counter.stored == 5 && counter.doubled == 10)
```

**Walkthrough:** doubled has no separately stored Int field. Its accessors still allow an inout update whose final write changes stored. No retained address is part of the contract.

**Boundary:** The example uses small representable integers. Unsafe pointer interoperability has additional lifetime, initialization, and aliasing obligations; & syntax alone does not satisfy them.

### What overlapping accesses does Swift try to forbid, and why can a conflict survive compilation?

**Answer:** A conflicting read/write or write/write through the same storage is invalid while a mutable access is active. The compiler diagnoses conflicts it can establish; runtime enforcement covers paths it cannot resolve statically.

**Example:** Compile-only, deliberately dynamically conflicting example.

```swift
var step = 1
@MainActor func increase(_ value: inout Int) { value += step }
increase(&step)
```

**Walkthrough:** The global read inside increase overlaps the caller’s mutable access to step. This source object-compiles, but its separately executed debug fixture stops with an exclusivity violation. The chapter does not continue execution after that trap. [Exclusivity enforcement](https://swift.org/blog/swift-5-exclusivity/)

**Boundary:** Compile-only review example; do not run it in a shared process. The validation runs the specified runtime trap in an isolated child with a timeout. Exclusivity is not a mutex or a complete cross-thread synchronization strategy.

The minimal correction snapshots the unrelated read before the mutable call, so the body no longer reaches back into step:

```swift
func increase(_ value: inout Int, by amount: Int) { value += amount }
var step = 1
let amount = step
increase(&step, by: amount)
precondition(step == 2)
```

### Extension: Why is a by-value array snapshot accepted alongside one inout element while two inout elements conflict?

**Answer:** The ordinary value argument is evaluated before mutable access begins. Two inout arguments instead ask for overlapping mutable access scopes throughout the call. Distinguish evaluation order from access duration.

**Example:** Standalone runnable example.

```swift
func replace(_ element: inout Int, using snapshot: [Int]) -> [Int] {
    element = snapshot.last!
    return snapshot
}
var values = [1, 7]
let before = replace(&values[0], using: values)
precondition(before == [1, 7] && values == [7, 7])
```

**Walkthrough:** Even with the inout argument written first, the ordinary snapshot argument is evaluated before that mutable access starts. The nonempty fixture makes last valid. The separate two-inout pitfall rejects.

**Boundary:** A snapshot can preserve logical values without guaranteeing a particular copy count. A helper that accesses the original array through another captured/global path during mutation would have a different conflict.

### Extension: How do borrow/mutate accessors preserve exclusivity for a noncopyable member?

**Answer:** They project existing storage with shared or exclusive access rather than requiring an owned get result. Projecting a member still holds the owner’s required access; it does not authorize a second conflicting use.

**Example:** Standalone runnable example.

```swift
struct Token: ~Copyable { var id: Int }
struct Slot: ~Copyable {
    private var storage: Token
    init(_ id: Int) { storage = Token(id: id) }
    var token: Token {
        borrow { storage }
        mutate { &storage }
    }
}
func increment(_ token: inout Token) { token.id += 1 }
func projectionProbe() {
    var slot = Slot(5)
    increment(&slot.token)
    precondition(slot.token.id == 6)
}
projectionProbe()
```

**Walkthrough:** mutate supplies exclusive access to storage without copying Token; borrow allows the later read. The increment finishes before that read begins. [Borrow/mutate accessors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

**Boundary:** This is distinct from a yielding accessor with post-access cleanup. [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`|C7]] checks projection restrictions, and separate deployment controls show that this compiler feature does not require the newer Ref runtime APIs.

### Extension: How can MutableRef keep an exclusive access live beyond its initializer?

**Answer:** Its dependent lifetime extends through its uses. The owner must not be read or mutated through a conflicting path while that exclusive reference remains active.

**Example:** Standalone runnable example.

```swift
struct Token: ~Copyable { var id: Int }
func referenceProbe() {
    var token = Token(id: 1)
    do {
        var reference = MutableRef(&token)
        reference.value.id += 1
    }
    precondition(token.id == 2)
}
referenceProbe()
```

**Walkthrough:** The explicit scope ends reference’s exclusive access before token is read. Both the token and reference remain subject to their ownership contracts. [Ref and MutableRef](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md)

**Boundary:** The selected APIs require Apple OS 27. Do not infer access duration from the initializer expression alone, and do not treat a reference wrapper as synchronization.

Prepend Token from the reference answer. This later use keeps the exclusive reference live across the direct read; object compilation **intentionally rejects** it:

```swift
func conflict() {
    var token = Token(id: 1)
    var reference = MutableRef(&token)
    print(token.id)
    reference.value.id += 1
}
```

## Focused exercise

**Task:** Replace a same-variable two-inout call with independent values and deliberate writeback.

**Acceptance checks:**

- Independent locals exchange and update the intended distinct array slots.
- Passing one variable twice rejects independently.
- swapAt expresses the equivalent array operation and specifies same-index behavior.

**Solution:** Prepend exchange setup. The two destination slots in this staged example are distinct:

```swift
var values = [1, 2]
var first = values[0]
var second = values[1]
exchange(&first, &second)
values[0] = first
values[1] = second
precondition(values == [2, 1])
values.swapAt(0, 1)
precondition(values == [1, 2])
```

**Reasoning:** The helper receives separate local storage and finishes before writeback. Staging is not an automatic transaction across reentrancy or concurrency. Prefer swapAt for this actual array operation.

Prepend only exchange setup. The original same-storage call is **intentionally rejected**:

```swift
func invalid() {
    var value = 1
    exchange(&value, &value)
}
```

## In practice

**Choose it when:** Expose a mutation boundary whose storage and access duration callers can understand.

**Investigate by:** Identify each access’s start, last use, and storage path; check both static and dynamic enforcement without executing undefined pointer behavior.

**Strong evidence:** Use an encapsulated storage operation or deliberate independent-value staging, then verify accessor and reference lifetimes separately from concurrency protection.

## References

### Related guides

- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]

### Official sources

- [Exclusivity violation](https://docs.swift.org/compiler/documentation/diagnostics/exclusivity-violation/)
- [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)
- [SE-0519: Ref and MutableRef](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md)
- [Swift.org. "Memory Safety." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety/)
- [Swift.org. "Declarations." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/)
- [Swift.org. "Swift 5 Exclusivity Enforcement." Swift.org Blog](https://swift.org/blog/swift-5-exclusivity/)
- [GitHub. "Enforce Exclusive Access to Memory." Swift Evolution SE-0176](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0176-enforce-exclusive-access-to-memory.md)
