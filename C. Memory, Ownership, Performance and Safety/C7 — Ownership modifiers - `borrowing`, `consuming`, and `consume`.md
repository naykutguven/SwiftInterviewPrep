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

**Learning goals:**

- Trace borrowing, consuming parameters, explicit consume, later use, and reinitialization across copyable and noncopyable values.
- Distinguish owned get results from stored borrow/mutate projections.
- Use Ref/MutableRef with their actual referent, lifetime, and exclusivity contracts rather than assuming deep uniqueness or thread affinity.

**Prerequisites:** [[A2 — `let`, `var`, `mutating`, and method semantics]]/[[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for mutation and parameters, [[C1 — ARC fundamentals]] for object aliases, and [[C3 — Exclusivity enforcement and `inout`]] for exclusive access. [[C8 — Copyable, noncopyable types, and `~Copyable`]] explains ~Copyable adoption and generic defaults.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Inspecting a resource token should not transfer it; handing it to its final consumer should. **borrowing** gives temporary shared access while ownership stays with the caller. **consuming** gives the callee an owned value that it may forward or destroy. Explicit **consume** ends the binding’s current value and transfers it.

For a noncopyable value, transfer prevents a second owner from using the same value. A Copyable argument can instead supply an implicit copy to a consuming parameter. Neither kind of ownership promises a fixed physical copy count, deep uniqueness of class payloads, Sendable conformance, or a destruction thread. [Parameter ownership](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Borrow repeatedly, then transfer the current token

Shared Token ownership setup. IDs make valid use visible without depending on an address or retain count.

```swift
struct Token: ~Copyable { var id: Int }
func inspect(_ token: borrowing Token) -> Int { token.id }
func finish(_ token: consuming Token) -> Int { token.id }
func transferProbe() {
    let token = Token(id: 7)
    precondition(inspect(token) == 7 && inspect(token) == 7)
    let result = finish(consume token)
    precondition(result == 7)
}
transferProbe()
```

**Walkthrough:** Inspection borrows the same token twice. finish receives its ownership; the current token value is unavailable afterward. The explicit operator makes the transfer visible, although a consuming noncopyable call can also transfer without that spelling.

### A consuming parameter can receive a copyable caller’s retained value

Standalone String example. The test checks source-level usability rather than physical copying.

```swift
func count(_ text: consuming String) -> Int { text.count }
let text = "Ada"
precondition(count(text) == 3)
precondition(text == "Ada")
```

**Walkthrough:** String is Copyable, so the caller can keep text while satisfying the callee’s owned parameter. Do not infer that every consuming call invalidates a Copyable binding or that the optimizer must allocate a new String buffer.

## Rules and pitfalls

### Consuming a wrapper does not destroy every class alias

**Common pitfall:** Expecting transfer of one noncopyable holder to remove an independent strong reference to its class payload.

**Mistake:** Standalone runnable example.

```swift
final class Payload { var value = 1 }
struct Holder: ~Copyable { let payload: Payload }
func finish(_ holder: consuming Holder) {}
func aliasProbe() {
    let payload = Payload()
    weak let observed = payload
    let holder = Holder(payload: payload)
    finish(consume holder)
    precondition(observed != nil)
    payload.value = 2
    precondition(observed?.value == 2)
}
aliasProbe()
```

**Consequence:** The object remains alive and mutable through payload after Holder is consumed.

**Why:** The holder owned one reference. Consuming it does not remove another owner’s reference or clone the graph into unique storage.

**Correction:** When release is required, account for and end every owning edge. This independent correction releases the final external alias explicitly.

```swift
final class Payload {}
struct Holder: ~Copyable { let payload: Payload }
func finish(_ holder: consuming Holder) {}
func releaseProbe() {
    var external: Payload? = Payload()
    weak let observed = external
    let holder = Holder(payload: external!)
    finish(consume holder)
    precondition(observed != nil)
    external = nil
    precondition(observed == nil)
}
releaseProbe()
```

**What changes:** The graph now has no remaining strong owner in this synchronous fixture. A resource API needing exclusive payload access must prevent or deliberately model aliases at adoption; ownership annotations alone cannot retroactively erase them.

### Ref of a getter result need not refer to the original slot

**Common pitfall:** Treating Ref(values[0]) as a live alias to the array element after an ordinary copyable get.

**Mistake:** Standalone runnable example.

```swift
func getterProbe() {
    var values = [10]
    let reference = Ref(values[0])
    values[0] = 20
    precondition(reference.value == 10 && values[0] == 20)
}
getterProbe()
```

**Consequence:** The reference still reads 10 in this checked compiler/SDK fixture.

**Why:** The ordinary getter produced a copyable temporary, and Ref borrowed that value. The reference constructor does not turn an owned getter into a storage projection.

**Correction:** Use an explicit mutable projection when exclusive in-place mutation is the intended contract, and end it before accessing the owner.

```swift
var values = [10]
do {
    var reference = MutableRef(&values[0])
    reference.value += 2
}
precondition(values == [12])
```

**What changes:** MutableRef holds exclusive access through the subscript’s supported mutable path. The scope ends that access before the array assertion. These Ref APIs require Apple OS 27 and do not supply synchronization.

## Worked example

### Reinitialization creates a new value for a consumed var

**Setup:** Prepend Token ownership setup. The mutable storage is reused only after a new Token is assigned.

**Predict:** Does reusing the variable name resurrect the consumed token?

```swift
func reinitializeProbe() {
    var current = Token(id: 1)
    let oldID = finish(consume current)
    current = Token(id: 2)
    precondition(oldID == 1 && inspect(current) == 2)
    _ = finish(consume current)
}
reinitializeProbe()
```

**Outcome:** The first transfer returns 1; the reinitialized value reads 2 and can be transferred independently.

**Why:** The binding is storage that can receive another initialized value. It is not permission to read its consumed old value or skip reinitialization on one control-flow path.

## Interview questions

### When does consuming end the caller’s current value, and when can a mutable binding be reused?

**Answer:** A noncopyable consuming transfer or explicit consume ends the current value. A var can be assigned a new value before reuse, on every path that reaches the later use. A consuming Copyable parameter can receive a copy without ending the caller’s binding.

**Example:** Prepend Token ownership setup.

```swift
func useTokens(_ replace: Bool) -> Int {
    var token = Token(id: 1)
    _ = finish(consume token)
    if replace { token = Token(id: 2) } else { token = Token(id: 3) }
    return inspect(token)
}
precondition(useTokens(true) == 2 && useTokens(false) == 3)
```

**Walkthrough:** Both branches restore initialization before inspect. The previous token remains consumed. The following negatives separate an unconditional use-after-consume from a missing reinitialization path. [Explicit consume](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0366-move-function.md)

**Boundary:** Ownership checks must run through object compilation; -typecheck alone can miss these diagnostics. Reinitializing a wrapper does not undo external effects performed by the old value’s consumer.

Each **intentionally rejected** function prepends Token ownership setup independently:

```swift
func invalid() -> Int {
    let token = Token(id: 1)
    _ = finish(consume token)
    return inspect(token)
}
```

```swift
func invalid(_ replace: Bool) -> Int {
    var token = Token(id: 1)
    _ = finish(consume token)
    if replace { token = Token(id: 2) }
    return inspect(token)
}
```

### How do borrow/mutate accessors differ from returning an owned value?

**Answer:** They project stored state with dependent shared/exclusive access, rather than returning an independently owned get result. An owned getter can construct a new value whose later lifetime and mutation are separate.

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
    var newToken: Token { Token(id: storage.id) }
}
func projectionProbe() {
    var slot = Slot(7)
    let owned = slot.newToken
    slot.token.id += 1
    precondition(owned.id == 7 && slot.token.id == 8)
}
projectionProbe()
```

**Walkthrough:** token exposes existing storage without copying the noncopyable Token. newToken constructs a separate owned token from its copyable ID, so it retains 7 after the slot changes. [Borrow/mutate accessors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

**Boundary:** The projection needs stable supported storage and has no post-access cleanup hook. It is not a yielding coroutine. This compiler rejects the local-temporary and shared class-field projections shown below; a compiler feature’s availability is separate from Ref runtime availability.

Two separate **intentionally rejected** storage projections:

```swift
struct Invalid {
    var number: Int {
        borrow {
            let local = 1
            return local
        }
    }
}
```

```swift
final class Storage { var value = 1 }
struct Invalid {
    let storage = Storage()
    var value: Int { borrow { storage.value } }
}
```

### What does Ref(values[0]) refer to when the subscript returns a copy?

**Answer:** It can borrow the temporary produced by the getter rather than the original element storage. Inspect the accessor contract; a Ref variable’s name does not establish live aliasing.

**Example:** Standalone runnable example.

```swift
func getterReferences() {
    var values = [10]
    let oldValue = Ref(values[0])
    let sameReference = oldValue
    values[0] = 20
    precondition(oldValue.value == 10 && sameReference.value == 10)
    precondition(values[0] == 20)
}
getterReferences()
```

**Walkthrough:** Copying Ref copies its dependent reference, not an independently owned original array slot. Both references read the getter temporary in the selected toolchain.

**Boundary:** This is a concrete SDK/accessor observation, not a claim that every Ref always snapshots. A supported borrowing storage projection can have another referent; explicit MutableRef(&values[0]) holds exclusive access as the corrected pitfall shows.

### Extension: What lifetime and ownership do Ref and MutableRef carry?

**Answer:** Ref is copyable and nonescapable; MutableRef is noncopyable and nonescapable. Both can access noncopyable values without owning them, and the mutable reference holds exclusive access through its uses.

**Example:** Standalone runnable example.

```swift
struct Payload: ~Copyable { var count: Int }
func references() {
    var payload = Payload(count: 10)
    do {
        let first = Ref(payload)
        let same = first
        precondition(first.value.count == 10 && same.value.count == 10)
    }
    do {
        var mutable = MutableRef(&payload)
        mutable.value.count += 2
        precondition(mutable.value.count == 12)
    }
    precondition(payload.count == 12)
}
references()
```

**Walkthrough:** The shared references finish before exclusive access begins, and exclusive access finishes before the final owner read. The owner remains responsible for the payload lifetime. [Ref and MutableRef](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md)

**Boundary:** The checked APIs require Apple OS 27. Returning a dependent view needs a valid supported lifetime contract; these examples use no experimental lifetime annotations and cannot return a reference to expired local storage.

Three standalone **intentionally rejected** ownership/lifetime cases:

```swift
func conflictingRead() {
    var value = 1
    var reference = MutableRef(&value)
    print(value)
    reference.value += 1
}
```

```swift
func movedReference() {
    var value = 1
    let first = MutableRef(&value)
    var second = consume first
    second.value += 1
    print(first.value)
}
```

```swift
func escapedLocal() -> Ref<Int> {
    let local = 1
    return Ref(local)
}
```

## Focused exercise

**Task:** Borrow and consume one noncopyable token, then reinitialize a mutable binding before reuse.

**Acceptance checks:**

- Repeated inspections borrow without transferring the token.
- A consumed value rejects later use before reinitialization.
- Both the original transfer and new initialized token produce their own IDs.

**Solution:** Prepend Token ownership setup. The rejected use-after-consume function above is the separate negative acceptance check.

```swift
func ownershipExercise() {
    var token = Token(id: 1)
    precondition(inspect(token) == 1 && inspect(token) == 1)
    let oldID = finish(consume token)
    precondition(oldID == 1)
    token = Token(id: 2)
    precondition(inspect(token) == 2)
    let newID = finish(consume token)
    precondition(newID == 2)
}
ownershipExercise()
```

**Reasoning:** Borrowing preserves ownership; consuming transfers the current Token. Assignment restores initialization before the next use. IDs establish the logical behavior without claiming physical moves, unique object graphs, or performance.

## In practice

**Choose it when:** Use borrowing for inspection and consuming for a justified ownership handoff or teardown.

**Investigate by:** Track each initialized value, its last transfer, every reinitialization path, and any borrowed projection’s owner and last use through object generation.

**Strong evidence:** Explain the difference between value ownership, dependent access, shared reference payloads, and measured copy costs; keep actor and Sendable requirements separate.

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
