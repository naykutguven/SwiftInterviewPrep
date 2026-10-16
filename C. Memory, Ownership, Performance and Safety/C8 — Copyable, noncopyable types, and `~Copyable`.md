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

**Goal:** Transfer an exclusively owned resource token and verify one cleanup event.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Noncopyable values have one owner at a time; transfer is allowed and duplication is not. Copyable is an implicit capability of ordinary generic parameters unless suppressed. A uniquely owned wrapper can still contain a class reference shared elsewhere, so noncopyability does not imply a deeply unique graph, Sendable conformance, or thread-affine destruction. [Noncopyable structs and enums](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)

## Rules and pitfalls

### Suppress only genuinely unnecessary copying

Use ~Copyable on the resource type and relevant generic/associated-type positions. Every consumer must also avoid implicitly requiring a copy. Borrow inspection and consume handoff; use explicit shutdown when cleanup timing or executor completion is a caller requirement.

**Common pitfall:** Declaring a protocol's Item noncopyable-capable does not generalize Holder<Item> or all extension methods. Check each implicit constraint and operation.

### Separate unique storage from payload semantics

UniqueBox and UniqueArray provide different ownership/storage contracts from shared CoW values and require Apple OS 27 in this SDK. clone of an Array payload preserves Array value behavior; clone of a class payload can copy the reference while sharing the object.

**Common pitfall:** Noncopyable Hashable support does not make Set accept noncopyable elements. Check protocol support, synthesized witnesses, container constraints, and OS availability independently. Proposed Optional reference accessors and a public RigidArray are absent from the selected surface.

## Interview questions

### Where must Copyable defaults be suppressed in a generic Holder with an associated Item?

Suppress copying on the holder itself when storing a noncopyable item, on the item's generic parameter, on an associated type that permits such payloads, and on generic uses that otherwise add their own Copyable requirement. Swift 6.4 primary associated-type defaults have specific propagation rules; verify non-primary positions explicitly.

### How does UniqueBox.clone behave with an Array payload versus a class payload?

Cloning a Copyable Array payload produces another logical array value whose mutations remain independent through Array's own semantics. Cloning a class payload can duplicate the reference, preserving object identity and shared mutation. Uniqueness of the box ownership does not turn that reference copy into a deep clone.

### Why can a noncopyable value conform to Hashable yet still fail as a Set element?

The Hashable protocols can support a broader ownership domain than a particular container. Set in this SDK still requires a Copyable element. A valid conformance and even accepted synthesized witnesses do not remove that separate generic requirement.

## Focused exercise

**Task:** Model an exclusively owned resource token. Check transfer and cleanup, then explain what remains shared if its payload is a class.

**Acceptance checks:**

- A consuming call transfers the single token.
- Cleanup occurs once by the completion of this synchronous transfer.
- The original value cannot be used after transfer.

**Solution:**

Standalone noncopyable token with observable cleanup:

```swift
struct Resource: ~Copyable {
    let onClose: () -> Void
    deinit { onClose() }
}
func transfer(_ resource: consuming Resource) {}
func probe() {
    var closed = 0
    let resource = Resource(onClose: { closed += 1 })
    transfer(consume resource)
    precondition(closed == 1)
}
probe()
```

Compile a second use of resource after transfer separately; it intentionally rejects. This fixture has synchronous nonisolated cleanup and no real file, lock, or thread affinity.

**Reasoning:** The resource's noncopyability prevents accidental duplicate owners and cleanup. A class-valued payload could still be referenced elsewhere; the token's unique lifetime does not change that graph. Real native handles also need explicit error, release-thread, and cancellation contracts.

## In practice

**Choose it when:** Exclusive ownership expresses a resource invariant callers must respect.

**Investigate by:** Check ownership paths through generic wrappers and actual SDK container declarations.

**Strong evidence:** Define adoption, deployment, release context, and fallback requirements before introducing newer unique-storage APIs.

### Unique storage and protocol boundaries — optional

#### UniqueBox and UniqueArray in Swift 6.4

UniqueBox owns one heap allocation and is noncopyable even when its contents are copyable. Access `.value` by borrowing or mutation; `consume()` transfers the payload out and ends the box. `clone()` requires a Copyable payload. [SE-0517](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0517-uniquebox.md)

```swift
struct Packet: ~Copyable {
    var count: Int
    deinit { print("destroy", count) }
}
func demo() {
    var box = UniqueBox(Packet(count: 1))
    box.value.count += 1
    let packet = box.consume()
    print(packet.count)
}
demo()
```

Output is `2` then `destroy 2`. A later `box.value` access is rejected. Cloning a box containing a noncopyable Packet is also rejected.

Clone uses the payload's ordinary copy semantics. With an Array payload, later value mutation produces independent array values. With a class payload, both allocations contain references to the same object; cloning the box is not a deep clone. UniqueBox can be useful for stable heap storage, but it adds allocation and indirection and does not automatically improve performance.

UniqueArray owns a growable contiguous buffer and supports noncopyable elements:

```swift
struct Token: ~Copyable { let id: Int }

func demoUniqueArray() {
    var values = UniqueArray<Token>()
    values.append(Token(id: 1))
    values.append(Token(id: 2))
    for value in values { print(value.id) } // borrowing iteration
    print(values.count)                    // owner remains usable
    let last = values.removeLast()         // transfers one element out
    print(last.id, values.count)
}
demoUniqueArray()
```

Output is `1`, `2`, `2`, then `2 1`. See C5/C12 for the iteration and storage comparisons. The tested SDK exposes no public RigidArray; do not substitute the proposal-only name for a usable baseline API. Both unique types above require Apple OS 27 in this SDK.

#### Generalized protocols: verify the boundary

The installed Equatable, Comparable, and Hashable protocols accept noncopyable/nonescapable conformers. Manual noncopyable Hashable conformance works:

```swift
struct Ticket: ~Copyable, Hashable {
    let id: Int
    static func == (lhs: borrowing Self, rhs: borrowing Self) -> Bool {
        lhs.id == rhs.id
    }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
    var description: String { "Ticket(\(id))" }
}
func demo() {
    let first = Ticket(id: 4)
    let second = Ticket(id: 4)
    print(first == second)
    print(first.description)
}
demo()
```

Output is `true` and `Ticket(4)`. The description is an ordinary property. Although [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md) describes wider string/stream protocol generalization, this installed SDK still requires Copyable for CustomStringConvertible and the related string/stream protocols inspected. A noncopyable CustomStringConvertible conformance fails. Also, `Set<Ticket>` still fails: the container requires Copyable elements. Do not equate protocol support with container support, synthesis, or unrestricted print/interpolation.

B7 now checks the synthesis boundary separately: the noncopyable Ticket struct derives Hashable, while the tested TicketState enum fails in generated equality and works with manual borrowing witnesses. See [[B7 — Synthesized conformances and semantic correctness]] for the exact declarations and compiler limitation.

#### Noncopyable continuations and Optional boundaries

D9's `Continuation` is a concrete noncopyable capability: resume consumes the handle, and dropping an unresumed handle traps. Unique ownership still does not prove a callback fires eventually or coordinate cancellation races.

Optional already supports noncopyable payloads and `take()`. The proposed Optional `ref`, `mutableRef`, and `put` operations are absent from the tested SDK; keep those SE-0532 examples outside this baseline. The validation record explains the status conflict.

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
