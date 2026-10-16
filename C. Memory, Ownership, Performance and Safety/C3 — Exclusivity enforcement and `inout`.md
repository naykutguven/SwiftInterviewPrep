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

**Goal:** Identify overlapping mutable access and replace it with independent scoped values.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Inout temporarily grants exclusive mutable access to caller storage. The copy-in/copy-out model explains observable semantics without promising a physical temporary or stable pointer. The access lasts for the call, and mutating methods similarly require exclusive access to self. Conflicting reads or writes can be diagnosed statically or trap dynamically. [Exclusivity violation](https://docs.swift.org/compiler/documentation/diagnostics/exclusivity-violation/)

## Rules and pitfalls

### Identify storage and access duration

Ordinary value arguments are evaluated before inout access begins. Two inout parameters can nevertheless overlap the same variable or aggregate storage throughout the call. Avoid multiple mutable parameters when callers cannot establish independence.

**Common pitfall:** Passing two array elements with & can overlap access to the whole array even if the integer indices differ. Use an operation designed for that storage, such as swapAt, or stage independent locals with deliberate writeback.

### Preserve lifetime through projections

Borrow/mutate accessors and MutableRef do not bypass exclusivity. A reference-like projection can keep an access alive, preventing mutation of the owner until that access ends.

**Common pitfall:** Treating & as C address-of suggests a stored pointer that can outlive the call. Keep scoped access within its contract; use an explicitly audited unsafe API only when a genuine native boundary requires it.

## Interview questions

### Why is a by-value array snapshot accepted alongside one inout element while two inout elements conflict?

The by-value argument can capture an independent array value before the inout element access begins. Two inout element arguments request simultaneous mutable access through the same array storage. Distinguish argument evaluation from the access duration rather than treating every read near & as an overlap.

### Why is `inout` not the same as C++ references or pointers?

It is a scoped access contract, not a freely storable alias with independent lifetime. Accessors can implement it through read/write semantics, and the language need not expose a stable address. C++ references or pointers have different lifetime and aliasing obligations.

### What kind of overlapping accesses does Swift try to forbid?

Swift forbids conflicting accesses to the same storage while a write or mutable access is active. This includes reading the same variable through another path during an inout call and overlapping mutation of self. Some cases need runtime checks; successful compilation alone does not establish nonoverlap.

## Focused exercise

**Task:** Explain why passing one variable to two inout parameters fails. Redesign the call with independent values and check the result.

**Acceptance checks:**

- Independent locals exchange correctly and write back to the chosen slots.
- The same variable passed twice rejects.
- The standard array operation expresses an in-place swap without caller-supplied overlapping accesses.

**Solution:**

Standalone independent-value redesign:

```swift
func exchange(_ first: inout Int, _ second: inout Int) {
    let saved = first
    first = second
    second = saved
}
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

Compile exchange(&first, &first) separately: it intentionally rejects overlapping inout access. The staged version assumes distinct destination slots; equal indices need an explicit policy.

**Reasoning:** The helper's two mutable accesses refer to separate local storage. Staging is not an automatic transaction across concurrency or reentrancy; no suspension occurs here. Prefer swapAt when its semantics already match the required operation.

## In practice

**Choose it when:** Expose a mutation boundary whose storage and duration callers can understand.

**Investigate by:** Minimize aliases and record access start/end, including accessor projections.

**Strong evidence:** Verify static and dynamic overlap cases separately; run deliberate exclusivity traps in isolated processes with timeouts.

### Accessor and reference access lifetimes — optional

#### Swift 6.4 borrow/mutate accessors preserve access contracts

A mutate accessor projects exclusive access to existing storage; it is not an owned-value getter/setter pair. It can support inout mutation of a noncopyable member without making an intermediate copy:

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
func demo() {
    var slot = Slot(5)
    increment(&slot.token)
    print(slot.token.id)
}
demo()
```

Output is `6`. The property still participates in exclusivity; accessors do not authorize overlapping mutation. C7 covers the projection restrictions. These accessors are separate from proposed yielding-accessor syntax. [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

#### A reference can keep an access live

```swift
struct Token: ~Copyable { var id: Int }
@available(anyAppleOS 27.0, *)
func demo() {
    var token = Token(id: 1)
    do {
        var reference = MutableRef(&token)
        reference.value.id += 1
    }
    print(token.id)
}
if #available(anyAppleOS 27.0, *) { demo() }
```

Output is `2`. The explicit scope ends the exclusive reference before the owner is read. This conflicting version is rejected during object compilation:

```swift
struct Token: ~Copyable { var id: Int }
@available(anyAppleOS 27.0, *)
func invalid() {
    var token = Token(id: 1)
    var reference = MutableRef(&token)
    print(token.id)
    reference.value.id += 1
}
```

The later use keeps reference's borrow active across the direct owner read. Track lifetime through uses rather than assuming that the initializer's expression is the entire access. Shared Ref and Span have analogous lifetime constraints; copyable getter temporaries may have different ownership, as C7 demonstrates. Exclusivity is not a mutex or a complete cross-thread synchronization strategy.

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
