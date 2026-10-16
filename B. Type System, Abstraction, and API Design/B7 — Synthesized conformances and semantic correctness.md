---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
  - synthesized-conformances
---

# B7 — Synthesized Conformances and Semantic Correctness

**Goal:** Make customized equality and hashing describe the same semantic components.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Synthesis derives behavior from a declaration; it cannot choose domain identity, persistence policy, or which content changes matter. Two snapshots can share an entity ID and differ as values. Identifiable neither requires Equatable nor makes equality compare only ID. Keep stable selection keys separate from content snapshots. [SE-0185: equality and hashing synthesis](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0185-synthesize-equatable-hashable.md)

## Rules and pitfalls

### Choose structural or identity equality deliberately

Structural synthesis compares stored state whose types conform. ID-only equality can be valid for a particular abstraction, but hashing must use matching components. Compare content separately when the UI or tests need to detect updates.

**Common pitfall:** Writing == using only id does not teach synthesized hash(into:) to omit other fields. Equal instances can then hash differently, violating Hashable. Implement both consistently or retain structural synthesis and use a separate ID key.

### Verify the actual synthesized surface

Codable derives a representation, not missing-key defaults or schema evolution. CaseIterable exposes cases, not a durable persisted ordinal. Sendable checks or infers a transfer-safety marker; it generates no locking or copying.

**Common pitfall:** Protocol support for noncopyable types does not prove synthesis or consumer support. Check conformance, generated witnesses, and concrete containers independently under the selected toolchain.

## Interview questions

### Why can snapshots with the same ID legitimately compare unequal, and how should selection keys differ from content comparison?

They represent different content for the same entity, so structural inequality can correctly trigger refresh or test failure. Use the ID as a selection/dictionary key when entity continuity matters; compare snapshots when content matters. ID-only equality is an explicit alternative, not a consequence of Identifiable.

### What goes wrong when == compares only id but hash(into:) is still synthesized?

Synthesized hashing can still combine all stored properties, allowing two ID-equal values with different content to violate the equal-values-have-equal-hashes requirement. Write hash(into:) with the same identity components or remove the customized equality. A single observed hash collision cannot validate a broken contract.

### Does a default-valued decoded var handle a missing key? What does Sendable actually generate?

No. A default-valued var does not generally make synthesized decoding tolerate a missing required key; optional decoding and explicit decodeIfPresent/default policy differ. Sendable does not generate operations or synchronization. It checks or infers whether the stored representation satisfies its safety contract.

## Focused exercise

**Task:** Compare snapshots with the same ID and changed content. Customize equality, then make hashing use matching components and check both.

**Acceptance checks:**

- Changed content produces unequal snapshots with equal domain IDs.
- ID-equal keys behave as one Set element.
- Equality laws and a different-ID key are checked independently of exact hash integers.

**Solution:**

Standalone structural snapshots and deliberate ID-only keys:

```swift
struct Snapshot: Equatable { let id: Int; var name: String }
struct EntityKey: Hashable {
    let id: Int
    let label: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
let old = Snapshot(id: 1, name: "Ada")
let new = Snapshot(id: 1, name: "Updated")
precondition(old != new && old.id == new.id)
let first = EntityKey(id: 1, label: old.name)
let second = EntityKey(id: 1, label: new.name)
precondition(first == second)
precondition(Set([first, second]).count == 1)
precondition(Set([first]).contains(second))
```

**Reasoning:** The snapshot captures content while EntityKey deliberately ignores label for equality and hashing. The Set observations check useful behavior but are not a proof for every value. Keep keys stable during collection membership, especially when reference state can mutate.

## In practice

**Choose it when:** Accept synthesis only after its derived semantics match the caller's contract.

**Investigate by:** Compare every customized equality component against hashing and persisted fields.

**Strong evidence:** Keep selection, content changes, decoding evolution, and transfer safety independently testable.

### Noncopyable conformance and synthesis — optional

#### Swift 6.4: test protocol support, synthesis, and consumers separately

[SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md) generalizes simple protocols. In the installed Swift 6.4 compiler/SDK, this noncopyable struct synthesizes Equatable and Hashable successfully:

```swift
struct Ticket: ~Copyable, Hashable {
    let id: Int
}

func ticketsEqual(
    _ lhs: borrowing Ticket, _ rhs: borrowing Ticket
) -> Bool {
    lhs == rhs
}
```

Two separately constructed Ticket values with ID 7 compare equal without consuming either. Hashable support does not make Ticket copyable. `Set<Ticket>` still fails because Set requires copyable elements. Codable also still requires Copyable in this SDK.

The analogous enum synthesis has an implementation limit in this toolchain:

```swift
enum TicketState: ~Copyable, Hashable {
    case empty
    case issued(Int)
}
```

Compilation rejects it with `type '(TicketState, TicketState)' containing noncopyable element is not supported`. This is a checked synthesis limitation, not a claim that all noncopyable enums can never conform. A manual implementation avoids the problematic generated comparison:

```swift
enum TicketState: ~Copyable, Hashable {
    case empty
    case issued(Int)

    static func == (
        lhs: borrowing Self, rhs: borrowing Self
    ) -> Bool {
        switch lhs {
        case .empty:
            switch rhs {
            case .empty: true
            case .issued: false
            }
        case .issued(let leftID):
            switch rhs {
            case .empty: false
            case .issued(let rightID): leftID == rightID
            }
        }
    }

    func hash(into hasher: inout Hasher) {
        switch self {
        case .empty:
            hasher.combine(0)
        case .issued(let id):
            hasher.combine(1)
            hasher.combine(id)
        }
    }
}
```

The case tag and payload reflect equality's essential components. This example covers copyable Int payloads within a noncopyable enum; it is not a guarantee for every payload or synthesis shape. C8 covers ownership, generalized protocols, and other SDK limits.

## References

### Related guides

- [[B8 — Equatable, Hashable, and Collection Correctness]]
- [[B9 — Codable and Schema Evolution]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D14 — Approachable Concurrency and Default Isolation]]

### Official sources

- [SE-0185: equality and hashing synthesis](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0185-synthesize-equatable-hashable.md)
- [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)
- [Hashable](https://developer.apple.com/documentation/swift/hashable)
- [CaseIterable](https://developer.apple.com/documentation/swift/caseiterable)
- [SE-0166: Swift archival and serialization](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md)
- [SE-0302: Sendable and @Sendable](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
