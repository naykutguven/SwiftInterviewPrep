---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - protocols
  - generics
  - existentials
  - api-design
---

# B1 — Protocols, Associated Types, `Self` Requirements, and Compositions

**Goal:** Preserve a lookup implementation’s key/value relationship at its consumer boundary.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A protocol declares capabilities and relationships. An associated type connects positions in one conformer's API; a generic parameter retains those relationships for a caller-selected type. An existential can retain selected associated-type constraints while allowing runtime implementation choice. Whether an existential can be formed is different from whether a particular member can be called through it. [Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)

## Rules and pitfalls

### Preserve the relationship the operation consumes

A returned associated value can sometimes be erased to its upper bound. An unknown associated input cannot safely accept an arbitrary value. Use a named generic, a constrained existential, or a conforming eraser that preserves the key and value types.

**Common pitfall:** Replacing typed storage with Any plus casts moves an invariant to runtime without defining missing-key or mismatch behavior. Keep erasure private and validate once at registration or adaptation.

### Treat capabilities as contracts, not promises of semantics

A get-only property requirement permits a computed or otherwise mutable implementation. Composition P & Q requires both existing capabilities; a protocol refining P and Q creates another nominal conformance. Marker protocols cannot enforce arbitrary business rules.

**Common pitfall:** Assuming a value's Sendable conformance makes every conformance usable anywhere overlooks isolated conformances. Preserve the witness domain at generic boundaries; see [[D14 — Approachable Concurrency and Default Isolation]].

## Interview questions

### Why does any Parser permit the demonstrated parse operation while an unconstrained lookup cannot accept a String key?

The parser's output can be returned with safe erasure when its associated result is only consumed at the known upper bound. Lookup takes an associated key as input; an unconstrained existential does not establish that its hidden key type is String. Constrain that relationship before calling it.

### Why can a Self result be erased while two independent existential Self inputs cannot be combined? How can a generic helper double one opened value?

A Self result can erase to a protocol existential. Two independent existential values may hide different concrete types, so a Self-typed input cannot combine them safely. Opening one value in a generic helper gives one T; the helper can combine that same value with itself because both positions are T.

### Does satisfying P and Q automatically establish conformance to a new protocol refining them? Does a get-only requirement prove immutable storage?

No. A nominal refining protocol needs an explicit conformance, even if its inherited requirements are already satisfied. A get-only requirement promises readable access, not immutable storage or purity; getters can compute, access shared state, and incur cost.

### Why can a value be Sendable while one of its protocol conformances is MainActor-isolated? What must a generic consumer preserve?

Sendable describes safe transfer/sharing of the value. An isolated conformance separately limits where its witnesses may be used. A generic consumer must preserve the conformance's actor boundary rather than assuming Sendable implies a nonisolated witness path.

### What must be suppressed for a noncopyable factory payload, and why does noncopyable Hashable support not imply Set or CustomStringConvertible support?

Suppress the relevant implicit Copyable constraints on the protocol, associated payload, and generic uses that must accept it. Then check each operation separately: noncopyable Hashable support does not remove Set's Copyable requirement or make CustomStringConvertible accept that domain. SDK constraints remain decisive.

## Focused exercise

**Task:** Define a keyed lookup protocol and a typed consumer. Check one accepted lookup and one rejected key-type mismatch.

**Acceptance checks:**

- Known and absent String keys preserve Int? results.
- An explicitly Int key rejects at compilation.
- No unchecked cast supplies the key relationship.

**Solution:**

Standalone synchronous lookup; absent keys return nil:

```swift
protocol KeyedLookup<Key, Value> {
    associatedtype Key: Hashable
    associatedtype Value
    func value(for key: Key) -> Value?
}
struct Names: KeyedLookup {
    let storage: [String: Int]
    func value(for key: String) -> Int? { storage[key] }
}
func read<L: KeyedLookup>(_ lookup: L, key: L.Key) -> L.Value? {
    lookup.value(for: key)
}
let names = Names(storage: ["Ada": 1])
precondition(read(names, key: "Ada") == 1)
precondition(read(names, key: "missing") == nil)
```

In a separate rejected variant, call read(names, key: 7). The consumer requires Names.Key, which is String.

**Reasoning:** The associated types express a relationship, and the generic consumer carries it through both argument and result. Runtime implementation selection would justify any KeyedLookup<String, Int>; a concrete generic remains simpler when callers already know the implementation.

## In practice

**Choose it when:** Use protocols for real capability boundaries, including a single implementation when testing or ownership separation benefits.

**Investigate by:** Identify each unknown type position and whether it is consumed or returned.

**Strong evidence:** Compare generic, constrained existential, and erased consumers against actual caller operations and isolated witnesses.

### Ownership-aware protocol constraints — optional

#### Swift 6.4 ownership-aware protocols

Standalone capability and generalized equality example:

```swift
protocol IdentifiedResource: ~Copyable {
    var id: Int { get }
}

struct Ticket: ~Copyable, IdentifiedResource, Hashable {
    let id: Int
    static func == (left: borrowing Ticket, right: borrowing Ticket) -> Bool {
        left.id == right.id
    }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}

func resourceID<R: IdentifiedResource & ~Copyable>(
    _ resource: borrowing R
) -> Int { resource.id }

func equal<T: Equatable & ~Copyable>(
    _ left: borrowing T, _ right: borrowing T
) -> Bool { left == right }

func erasedResourceID(
    _ resource: borrowing any IdentifiedResource & ~Copyable
) -> Int { resource.id }
```

Suppressing the protocol's default permits noncopyable conformers. Generic uses and existential storage also need the intended constraints. Here equality and hashing are explicitly implemented using a borrowed identity; no automatic conformance synthesis or container compatibility is assumed.

Expected failure, reusing `Ticket`:

```swift
func cannotStoreTicketsInSet() {
    _ = Set<Ticket>()
}
```

`Hashable` support alone does not remove the set's element copyability requirement. Check the whole API, not just one conformance. [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)

Standalone SDK-limit probe, expected to fail:

```swift
struct PrintedTicket: ~Copyable, CustomStringConvertible {
    var description: String { "ticket" }
}
```

The tested Swift 6.4 SDK still requires `Copyable` for `CustomStringConvertible`. Proposal/release generalization does not prove every listed protocol or operation is generalized in this distribution. C8 records the interface checks and related limits.

#### Suppressed associated-type defaults

Standalone example:

```swift
struct Packet: ~Copyable { let id: Int }

protocol PacketFactory<Output>: ~Copyable {
    associatedtype Output: ~Copyable = Packet
    func make() -> Output
}

struct DefaultFactory: PacketFactory {
    func make() -> Output { Packet(id: 7) }
}

func make<F: PacketFactory & ~Copyable>(
    using factory: borrowing F
) -> F.Output where F.Output: ~Copyable {
    factory.make()
}
```

The default associated-type witness is noncopyable. The generic helper suppresses defaults for both its conformer and primary associated type. A default witness is a fallback type choice, not a rule that all conformers must use `Packet`. SE-0503 is implemented in Swift 6.4; this checked source needs no experimental flag. [SE-0503](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md)

Expected failure, reusing those declarations:

```swift
func ordinaryMake<F: PacketFactory>(_ factory: F) -> F.Output {
    factory.make()
}
func cannotUseImplicitOutputDefault() {
    _ = ordinaryMake(DefaultFactory())
}
```

This generic use reintroduces a `Copyable` default for the primary `Output` and rejects the packet. B3 covers primary versus non-primary defaults, extensions, `~Escapable`, and borrowed iteration. These distinctions are separate from cache key/value relationships.

## References

### Related guides

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[D4 — Actors and Actor Isolation]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D7 — Actor Reentrancy and Logic Races]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]

### Official sources

- [Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)
- [SE-0503](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md)
- [SE-0309: Unlock existentials](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)
- [SE-0346: Primary associated-type constraints](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)
- [SE-0353: Constrained existentials](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0353-constrained-existential-types.md)
- [SE-0521: Optional opaque/existential syntax](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0521-improved-optional-opaque-and-any.md)
