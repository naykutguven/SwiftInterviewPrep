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

# C7 — Ownership Modifiers - `borrowing`, `consuming`, and `consume`

**Goal:** Borrow, transfer, and reinitialize a noncopyable token under object compilation.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Borrowing gives temporary shared access while the caller retains ownership. Consuming gives ownership to the callee, which may forward or destroy the value. Explicit consume ends the current binding's value. For Copyable arguments, a consuming call may receive an implicit copy; for noncopyable values the transfer is part of correctness. [SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)

## Rules and pitfalls

### Make transfer visible where correctness depends on it

Use borrowing for inspection and consuming for exclusive handoff or teardown. A mutable binding can be reinitialized after its old value is consumed, then used as a new initialized value. Ownership of a wrapper does not erase other aliases to a class payload.

**Common pitfall:** Assuming consume destroys an object globally confuses one owned reference/value with every alias in the graph. Review referenced state separately; ownership does not establish Sendable or thread affinity.

### Distinguish projections from owned results

Swift 6.4 borrow/mutate accessors expose existing storage with lifetime/exclusivity constraints. Ref and MutableRef carry dependent access rather than extending ownership arbitrarily.

**Common pitfall:** A Ref formed from an ordinary copyable getter result can refer to a temporary copy rather than original element storage. Inspect the accessor contract and keep mutable access exclusive. Returning a new owned value is a different lifetime promise.

## Interview questions

### When does consuming end the caller's current value, and when can a mutable binding be reused?

A noncopyable consuming transfer, or explicit consume of the current value, prevents later use of that value. A var can receive a new value before reuse; all paths must reestablish initialization. A consuming parameter alone need not end a Copyable caller value because the compiler can pass a copy.

### How do borrow/mutate accessors differ from returning an owned value?

They expose an existing stored value without the ordinary owned get result, and their dependent access constrains owner mutation and lifetime. An owned result can live independently according to its ownership contract. Borrow access cannot return a local temporary whose storage disappears at accessor exit.

### What does Ref(values[0]) refer to when the subscript returns a copy?

When that subscript performs an ordinary copyable get, Ref can borrow the produced temporary rather than the array's original slot. It does not retroactively change the getter into a storage projection. Use a supported borrowing accessor when access to original storage is required, and validate with the selected SDK.

## Focused exercise

**Task:** Borrow and consume a noncopyable token. Show accepted borrowing, rejected use after consume, and valid reinitialization.

**Acceptance checks:**

- Repeated borrowing preserves ownership.
- Use after consume rejects during object compilation.
- Reinitialization creates a new valid value for the mutable binding.

**Solution:**

Standalone ownership probe; compile to an object, not only with -typecheck:

```swift
struct Token: ~Copyable { let id: Int }
func inspect(_ token: borrowing Token) -> Int { token.id }
func finish(_ token: consuming Token) -> Int { token.id }
func probe() {
    var token = Token(id: 1)
    precondition(inspect(token) == 1)
    let completed = finish(consume token)
    precondition(completed == 1)
    token = Token(id: 2)
    precondition(inspect(token) == 2)
    _ = finish(consume token)
}
probe()
```

In a separate rejected variant, call inspect(token) after the first consume and before reinitialization.

**Reasoning:** Inspection needs no ownership transfer. finish consumes the current Token; assignment then restores initialized storage with a new Token. Assertions observe IDs rather than physical copies, destruction timing, or performance.

## In practice

**Choose it when:** Make resource lifetime and caller costs explicit at an ownership boundary.

**Investigate by:** Compile accepted and rejected ownership paths through object generation.

**Strong evidence:** Explain the difference between logical ownership, dependent borrowing, shared payloads, and measured copy cost.

### Accessor and reference lifetimes — optional

#### Borrow and mutate accessors in Swift 6.4

`borrow` exposes stable storage for shared access; `mutate` exposes exclusive mutable access. Unlike an ordinary getter, the access need not produce an owned copy of a noncopyable property. A simple stored-value projection is:

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
func demo() {
    var slot = Slot(7)
    print(slot.token.id)
    slot.token.id += 1
    print(slot.token.id)
}
demo()
```

Output is `7` then `8`. `&storage` is the mutable projection; it does not transfer ownership out of Slot. Access to the containing value stays subject to exclusivity. These accessors provide no post-access cleanup hook: do not treat them as yielding coroutines. The tested compiler rejects a borrow accessor returning a local temporary or a stored property of a class. [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

This is a compiler feature: the same minimal projection compiles for macOS 13. That does not back-deploy new library types such as Ref or UniqueBox.

#### Ref and MutableRef are lifetime-bound references

`Ref<T>` is copyable and nonescapable; `MutableRef<T>` is noncopyable and nonescapable. Both can refer to noncopyable values. The former reads through `.value`; the latter mutates through it while holding exclusive access. They do not own the referent. [SE-0519](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md)

Standalone example on Apple OS 27:

```swift
struct Payload: ~Copyable { var count: Int }

func demoReferences() {
    var payload = Payload(count: 10)
    do {
        let first = Ref(payload)
        let same = first // copies a reference, not Payload
        print(first.value.count, same.value.count)
    }
    do {
        var mutable = MutableRef(&payload)
        mutable.value.count += 2
        print(mutable.value.count)
    }
    print(payload.count)
}
demoReferences()
```

Output is `10 10`, then `12`, then `12`. Invalid overlapping access, checked separately:

```swift
func conflict() {
    var value = 1
    var reference = MutableRef(&value)
    print(value) // error: overlapping accesses
    reference.value += 1 // keeps the exclusive access live across the read
}
```

Moving MutableRef into a new binding also prevents later use of the old binding. Returning a reference to a local variable cannot provide a valid escaping result. Lifetime-aware return APIs require their own supported lifetime contract; these examples do not use experimental lifetime annotations.

A copyable getter can introduce a temporary. In the tested compiler, `Ref(values[0])` for `[Int]` followed by mutation of the array keeps the old element value; it is not a live alias to the array slot. Use the explicit mutable projection `MutableRef(&values[0])` when exclusive in-place mutation is the contract. Never infer aliasing solely from a Ref variable’s name.

## References

### Related guides

- [[C1 — ARC fundamentals]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[E7 — Resilience, ABI, and Module Stability]]

### Official sources

- [SE-0390](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [SE-0507](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)
- [SE-0519](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md)
- [GitHub. "borrowing and consuming Parameter Ownership Modifiers." Swift Evolution SE-0377](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)
- [GitHub. "consume Operator to End the Lifetime of a Variable Binding." Swift Evolution SE-0366](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0366-move-function.md)
