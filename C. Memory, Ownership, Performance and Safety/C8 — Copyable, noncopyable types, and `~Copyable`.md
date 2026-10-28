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

# C8 — Copyable, Noncopyable Types, and `~Copyable`

**Learning goals:**

- Model one owner at a time while accounting for separate class aliases, adoption, and cleanup context.
- Suppress Copyable defaults at the resource, wrapper, protocol payload, generic use, and extension positions that require it.
- Check UniqueBox/UniqueArray, protocol witnesses, actual consumers, Optional operations, and deployment independently.

**Prerequisites:** [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]] for borrowing/consuming, [[B3 — Generics, Constraints, and Same-Type Relationships]] for associated-type defaults, [[B7 — Synthesized conformances and semantic correctness]] for synthesis, and [[C1 — ARC fundamentals]] for retaining graphs and cleanup completion.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Copying a file-handle owner could accidentally arrange two closes of one handle. A **noncopyable value**, declared with ~Copyable, prevents ordinary duplication of that value while allowing a transfer to a new owner. It does not prevent two separate initializers from adopting the same external handle; the adoption API must control that separately.

Copyable is also an implicit requirement of ordinary generic parameters and many consumers. Generalizing one declaration does not automatically generalize every wrapper or container around it. A noncopyable wrapper can still store a class reference that other code owns, so exclusive wrapper ownership is not a deeply unique object graph, a Sendable guarantee, or a destruction-thread promise. [Noncopyable values](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### A wrapper must permit the ownership domain it stores

Shared Token/Holder setup. Both Item and its containing value suppress the ordinary Copyable requirement.

```swift
struct Token: ~Copyable { let id: Int }
struct Holder<Item: ~Copyable>: ~Copyable {
    var item: Item
    init(_ item: consuming Item) { self.item = item }
}
func holderProbe() {
    let holder = Holder(Token(id: 7))
    precondition(holder.item.id == 7)
}
holderProbe()
```

**Walkthrough:** Holder takes ownership of Item rather than copying it. Its own noncopyability prevents copying the stored Token indirectly. Copyable payloads remain allowed; suppression broadens the permitted domain rather than requiring every payload to be noncopyable.

### UniqueBox transfers a payload; UniqueArray can remove an owned element

Standalone Apple OS 27 example. The log observes synchronous Packet destruction, and the primitive array demonstrates removal separately.

```swift
final class Log { var destroyed: [Int] = [] }
struct Packet: ~Copyable {
    var count: Int
    let log: Log
    deinit { log.destroyed.append(count) }
}
func useBox(_ log: Log) -> Int {
    var box = UniqueBox(Packet(count: 1, log: log))
    box.value.count += 1
    let packet = box.consume()
    return packet.count
}
let log = Log()
let result = useBox(log)
precondition(result == 2 && log.destroyed == [2])
func useArray() {
    var values = UniqueArray<Token>()
    values.append(Token(id: 1))
    values.append(Token(id: 2))
    var ids: [Int] = []
    for value in values { ids.append(value.id) }
    let last = values.removeLast()
    precondition(ids == [1, 2] && last.id == 2 && values.count == 1)
}
useArray()
```

**Walkthrough:** consume ends the box and returns its payload. The payload is destroyed once when the synchronous helper finishes. Borrowing array iteration leaves its elements owned by the array; removeLast transfers one out. These types introduce their own storage/allocation contract, not a blanket performance benefit.

## Rules and pitfalls

### A permitted associated payload does not broaden every generic use

**Common pitfall:** Defining a protocol that permits a noncopyable Item but using it through a helper that reinstates the primary-associated-type Copyable default.

**Mistake:** Compile-only; intentionally rejected.

```swift
struct Token: ~Copyable { let id: Int }
protocol Source<Item>: ~Copyable {
    associatedtype Item: ~Copyable
    func make() -> Item
}
struct TokenSource: Source { func make() -> Token { Token(id: 7) } }
func ordinary<S: Source>(_ source: S) -> S.Item { source.make() }
func invalid() { _ = ordinary(TokenSource()) }
```

**Consequence:** The concrete Token payload cannot satisfy the helper’s implicit Copyable requirement.

**Why:** The protocol declaration and consumer are separate generic contracts. Permitting a payload at one boundary does not erase another boundary’s default.

**Correction:** Preserve the intended payload domain at the use and wrapper. This standalone correction spells each suppression it needs.

```swift
struct Token: ~Copyable { let id: Int }
struct Holder<Item: ~Copyable>: ~Copyable { let item: Item }
protocol Source<Item>: ~Copyable {
    associatedtype Item: ~Copyable
    func make() -> Item
}
struct TokenSource: Source { func make() -> Token { Token(id: 7) } }
func hold<S: Source & ~Copyable>(_ source: borrowing S) -> Holder<S.Item>
where S.Item: ~Copyable { Holder(item: source.make()) }
func check() {
    let held = hold(TokenSource())
    precondition(held.item.id == 7)
}
check()
```

**What changes:** The wrapper, associated Item, and generic use now accept the Token. The source itself is copyable here; owner and payload constraints remain independent. Extensions have their own defaults too. [Suppressed associated types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md)

Shared Ticket setup for the next pitfall. These manual witnesses borrow the value and use the same identity components:

```swift
struct Ticket: ~Copyable, Hashable {
    let id: Int
    static func == (lhs: borrowing Self, rhs: borrowing Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
    var description: String { "Ticket(\(id))" }
}
```

### Protocol support does not remove a container’s Copyable requirement

**Common pitfall:** Using a noncopyable Hashable Ticket as a Set element. Prepend Ticket setup.

**Mistake:** Compile-only; intentionally rejected.

```swift
func invalid() { _ = Set<Ticket>() }
```

**Consequence:** The compiler rejects the container’s element constraint even though Ticket’s witnesses are valid.

**Why:** Hashable’s supported ownership domain is broader than this Set declaration. Conformance, synthesis, and consumer support are three separate checks.

**Correction:** Keep the resource in a supported owner and use a copyable domain ID for hashed membership. Prepend Ticket setup.

```swift
func ticketMembership() {
    var tickets = UniqueArray<Ticket>()
    tickets.append(Ticket(id: 4))
    var ids: Set<Int> = []
    for ticket in tickets { ids.insert(ticket.id) }
    precondition(ids.contains(4) && tickets.count == 1)
}
ticketMembership()
```

**What changes:** The hash collection stores IDs; the unique container owns resources. This does not supply an automatic ID-to-resource lookup or a deep-clone contract. The UniqueArray path needs Apple OS 27.

## Worked example

### Cloning a box uses its payload’s ordinary copy semantics

**Setup:** Standalone Apple OS 27 fixture. Compare a value-semantic Array with a reference-semantic class payload.

**Predict:** Will editing each cloned box change the first box’s contents?

```swift
final class Payload { var count = 1 }
func cloneProbe() {
    let first = UniqueBox([1])
    var second = first.clone()
    second.value.append(2)
    precondition(first.value == [1] && second.value == [1, 2])
    let payload = Payload()
    let objectBox = UniqueBox(payload)
    let otherBox = objectBox.clone()
    otherBox.value.count = 9
    precondition(objectBox.value === payload && otherBox.value === payload)
    precondition(objectBox.value.count == 9)
}
cloneProbe()
```

**Outcome:** Array mutation is independent; class mutation is shared.

**Why:** clone requires a Copyable payload and uses that payload’s copy contract. It creates another box, not a universal deep clone. Unique ownership of the boxes and ownership of objects reachable through their payloads are different layers. [UniqueBox](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0517-uniquebox.md)

## Interview questions

### Where must Copyable defaults be suppressed in a Holder with an associated Item?

**Answer:** On the noncopyable-containing holder, its Item parameter, an associated payload that permits that domain, and generic uses/extensions whose defaults would add Copyable again. Borrowing a parameter alone does not remove its type constraints.

**Example:** Prepend the corrected Source/Holder/TokenSource/hold setup.

```swift
extension Source where Self: ~Copyable, Item: ~Copyable {
    func withItem<R>(_ body: (consuming Item) -> R) -> R { body(make()) }
}
precondition(TokenSource().withItem { $0.id } == 7)
func read(_ holder: borrowing Holder<Token>) -> Int { holder.item.id }
func checkHolder() {
    let value = hold(TokenSource())
    precondition(read(value) == 7)
}
checkHolder()
```

**Walkthrough:** The extension states the same ownership domain as the corrected helper. It hands its newly produced Item to the consuming closure and returns an ordinary owned Int. [[B3 — Generics, Constraints, and Same-Type Relationships|B3]] compares primary and non-primary associated-type defaults in detail.

**Boundary:** The implementation must also avoid copying the resource. Suppressing Escapable is a different lifetime decision and is needed only for a supported borrowed/nonescapable domain.

Prepend only the initial Token/Holder setup. An ordinary generic wrapper cannot store Token under its default; this use is **intentionally rejected**:

```swift
struct Ordinary<Item> { let item: Item }
func invalidWrapper() { _ = Ordinary(item: Token(id: 1)) }
```

### How does UniqueBox.clone behave with an Array payload versus a class payload?

**Answer:** It follows the Copyable payload’s ordinary semantics. An Array clone is another logical array value, while a class clone can copy its reference and keep one shared object. clone is unavailable for a noncopyable payload.

**Example:** Standalone runnable example.

```swift
final class State { var count = 0 }
func compareClones() {
    let array = UniqueBox([10])
    var arrayCopy = array.clone()
    arrayCopy.value[0] = 20
    precondition(array.value == [10] && arrayCopy.value == [20])
    let shared = State()
    let state = UniqueBox(shared)
    let stateCopy = state.clone()
    stateCopy.value.count += 1
    precondition(state.value === stateCopy.value && shared.count == 1)
}
compareClones()
```

**Walkthrough:** The array slot edit stays independent; the class edit is observed through both boxes and the original alias. Both calls use the same clone API with different payload contracts.

**Boundary:** UniqueBox adds heap storage/indirection and requires OS 27 here. The following Packet clone rejection is a different question from whether a class-valued clone is deep.

Prepend Packet/Log from the unique-storage mechanics. These two functions are independently **intentionally rejected**:

```swift
func invalidClone() {
    let box = UniqueBox(Packet(count: 1, log: Log()))
    _ = box.clone()
}
```

```swift
func invalidBoxUse() {
    let box = UniqueBox(Packet(count: 1, log: Log()))
    let packet = box.consume()
    print(packet.count)
    print(box.value.count)
}
```

### Why can a noncopyable value conform to Hashable yet still fail as a Set element?

**Answer:** The protocol witnesses can borrow a noncopyable conformer, while Set still requires Copyable elements. That independent requirement is not changed by a successful conformance or synthesized witness.

**Example:** Prepend Ticket setup.

```swift
func equalityProbe() {
    let first = Ticket(id: 4)
    let same = Ticket(id: 4)
    let different = Ticket(id: 5)
    precondition(first == same && first != different)
    var hasher = Hasher()
    first.hash(into: &hasher)
    precondition(first.description == "Ticket(4)")
}
equalityProbe()
```

**Walkthrough:** The manual equality/hash witnesses borrow Ticket and keep it available. description is an ordinary property; no unrestricted printing/interpolation conformance is implied. The Set rejection appears in the pitfall. [Generalized simple protocols](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)

**Boundary:** The selected compiler derives Hashable for a noncopyable struct but has the tested enum-synthesis limitation recorded in [[B7 — Synthesized conformances and semantic correctness|B7]]. CustomStringConvertible still requires Copyable in this SDK, despite broader proposal text; check the actual declaration.

This standalone **intentionally rejected** conformance demonstrates that SDK boundary:

```swift
struct Described: ~Copyable, CustomStringConvertible { var description: String { "token" } }
```

### Extension: Which Optional operations are actually available for a noncopyable payload?

**Answer:** Optional already supports a noncopyable payload and take(), which moves the stored value out and leaves none. The proposed ref/mutableRef/put spellings are absent from this selected SDK.

**Example:** Prepend Token/Holder setup.

```swift
func optionalProbe() {
    var slot: Token? = Token(id: 7)
    guard let moved = slot.take() else { preconditionFailure() }
    precondition(moved.id == 7)
    if case .some = slot { preconditionFailure("Expected empty slot") }
    slot = Token(id: 8)
    guard let replacement = slot.take() else { preconditionFailure() }
    precondition(replacement.id == 8)
}
optionalProbe()
```

**Walkthrough:** Each take transfers one payload out before the slot is reused. No second Token copy is created. [[A4 — Optionals and Nil Modeling|A4]] shows the surrounding optional ownership operations.

**Boundary:** The SE-0532 proposal is not evidence that every named API is installed. [[D9 — Continuations and Bridging Callback APIs|D9]] separately demonstrates a noncopyable continuation whose resume consumes its handle; unique ownership alone does not guarantee eventual completion or cancellation coordination.

The missing Optional operations and public RigidArray name are two separate **intentionally rejected** SDK probes:

```swift
func proposed(_ slot: inout Token?) {
    _ = slot.ref
    _ = slot.mutableRef
    slot.put(Token(id: 1))
}
```

```swift
func proposedRigidArray() { _ = RigidArray<Int>() }
```

## Focused exercise

**Task:** Transfer one exclusively owned resource token, verify its synchronous cleanup, and account for a class payload’s other owner.

**Acceptance checks:**

- One consuming transfer runs this token’s cleanup exactly once by return.
- The old resource value rejects a later use.
- A separately owned class payload remains shared after the token is consumed.

**Solution:** Standalone synchronous model; no real file descriptor, lock, or actor-affine release is claimed:

```swift
final class Payload { var count = 0 }
struct Resource: ~Copyable {
    let payload: Payload
    let onClose: () -> Void
    deinit { onClose() }
}
func transfer(_ resource: consuming Resource) { resource.payload.count += 1 }
func resourceProbe() {
    var closed = 0
    let payload = Payload()
    let resource = Resource(payload: payload, onClose: { closed += 1 })
    transfer(consume resource)
    precondition(closed == 1)
    precondition(payload.count == 1)
    payload.count += 1
    precondition(payload.count == 2)
}
resourceProbe()
```

**Reasoning:** Resource cannot be duplicated accidentally, but its Payload reference was intentionally also held by the caller. Cleanup belongs to this token’s synchronous nonisolated lifetime; native resources need explicit adoption, error, cancellation, and release-context contracts when those matter.

Prepend Resource/Payload/transfer from the exercise. The original value cannot be reused; this is **intentionally rejected**:

```swift
func invalidReuse() {
    let resource = Resource(payload: Payload(), onClose: {})
    transfer(consume resource)
    _ = resource.payload.count
}
```

## In practice

**Choose it when:** Use noncopyability when one-owner transfer expresses an actual resource invariant, and keep ordinary copyable values when snapshots are the requirement.

**Investigate by:** Audit every implicit generic/associated-type default, payload alias, consumer constraint, and selected operation’s availability.

**Strong evidence:** Define who can adopt/release the resource and on which context, verify cleanup and transfer paths, then measure storage costs without equating unique ownership with a unique graph.

## References

### Related guides

- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[E3 — C Interoperability and Unsafe Boundaries]]
- [[E4 — C++ interoperability]]

### Official sources

- [Noncopyable structs and enums](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [SE-0517](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0517-uniquebox.md)
- [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)
- [GitHub. "Noncopyable Generics." Swift Evolution SE-0427](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md)
- [GitHub. "Borrowing and Consuming Pattern Matching for Noncopyable Types." Swift Evolution SE-0432](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0432-noncopyable-switch.md)
- [Swift.org. "Generics." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)
- [SE-0527: UniqueArray design](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0527-rigidarray-uniquearray.md)
- [SE-0503: Suppressed associated-type defaults](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md)
