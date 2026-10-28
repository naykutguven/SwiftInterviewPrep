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

**Learning goals:**

- Explain how Equatable, Hashable, Codable, and CaseIterable derive behavior, and how Sendable checking differs.
- Keep entity identity, snapshot content equality, matching hash components, and wire schema deliberate.
- Check noncopyable protocol support, synthesis, and concrete consumers as three separate questions.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] for equality versus identity, [[A4 — Optionals and Nil Modeling]] for absence, and [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]/[[B3 — Generics, Constraints, and Same-Type Relationships]] for conformance and conditional constraints. [[B8 — Equatable, Hashable, and Collection Correctness]]/[[B9 — Codable and Schema Evolution]] develop collection and persistence contracts.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Two snapshots can describe the same user before and after a name change. Their IDs match, while their contents differ. Synthesized equality can correctly notice that change. Identifiable does not instruct Equatable to compare only the ID.

**Synthesis** derives code from a declaration and eligible stored state. It cannot choose the application’s identity, caching, or migration policy. Decide whether you are comparing content or identifying an entity, then verify that equality, hashing, encoding, and selection each implement their intended contract. [SE-0185](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0185-synthesize-equatable-hashable.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Structural synthesis compares content, not merely an ID

Both stored fields participate in the synthesized equality/hash contract.

```swift
struct Snapshot: Identifiable, Hashable {
    let id: Int
    var name: String
}
let before = Snapshot(id: 1, name: "Ada")
let after = Snapshot(id: 1, name: "Updated")
precondition(before.id == after.id)
precondition(before != after)
precondition(Set([before, after]).count == 2)
var selectedByID = [before.id: before]
selectedByID[after.id] = after
precondition(selectedByID.count == 1 && selectedByID[1]?.name == "Updated")
```

**Walkthrough:** The Set sees two different content values. The dictionary uses one stable entity key and replaces its snapshot. These are two valid but different operations.

### Coding and case iteration have their own derived surfaces

CodingKeys chooses persisted fields; computed properties are not automatically stored in the encoded representation. CaseIterable synthesis enumerates cases that have no associated values and no case-specific availability annotations. A payload-bearing case would need an explicit policy for which payload values to enumerate. Neither coding nor case iteration follows an equality customization. [CaseIterable synthesis](https://developer.apple.com/documentation/swift/caseiterable)

```swift
import Foundation

struct Record: Codable {
    let name: String
    var cache = 0
    var uppercase: String { name.uppercased() }
    enum CodingKeys: String, CodingKey { case name }
}
enum Mode: String, CaseIterable { case compact, detailed }
let record = Record(name: "Ada", cache: 7)
let data = try JSONEncoder().encode(record)
let fields = try JSONSerialization.jsonObject(with: data) as! [String: Any]
precondition(Set(fields.keys) == ["name"])
let decoded = try JSONDecoder().decode(Record.self, from: data)
precondition(decoded.name == "Ada" && decoded.cache == 0)
precondition(decoded.uppercase == "ADA")
precondition(Mode.allCases.map(\.rawValue) == ["compact", "detailed"])
```

**Walkthrough:** Only name is encoded. The excluded cache has its own initial value after decoding; the computed property is recomputed. Case iteration follows this declaration’s cases, but its positional index is not a durable persisted identifier.

### Conditional synthesis follows the payload's capabilities

A generic wrapper does not need to restrict all payloads at declaration time. State each conditional conformance where its derived operation is available. This standalone example separates holding a closure from comparing or coding it:

```swift
import Foundation

struct Box<Value> { let value: Value }
extension Box: Equatable where Value: Equatable {}
extension Box: Hashable where Value: Hashable {}
extension Box: Codable where Value: Codable {}
let first = Box(value: 7)
precondition(first == Box(value: 7))
precondition(Set([first, Box(value: 7)]).count == 1)
let bytes = try JSONEncoder().encode(first)
let restored = try JSONDecoder().decode(Box<Int>.self, from: bytes)
precondition(restored == first)
let behavior = Box(value: { 7 })
precondition(behavior.value() == 7)
```

**Walkthrough:** Box<Int> receives all three conformances. Box<() -> Int> is still a usable wrapper, but the closure does not meet these equality/hash/coding constraints. A conforming payload supplies the capability; it does not decide whether the generated content semantics are correct for the application. [[B3 — Generics, Constraints, and Same-Type Relationships|B3]] teaches the generic constraint mechanics.

## Rules and pitfalls

### Custom equality does not customize synthesized hashing

**Common pitfall:** Comparing only id while leaving Hashable to synthesize from all stored fields. Compile-only: this broken contract is never used in a hash collection.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
struct BadKey: Hashable {
    let id: Int
    let name: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
}
func invalidUse() {
    let old = BadKey(id: 1, name: "old")
    let new = BadKey(id: 1, name: "new")
    _ = Set([old, new])
}
```

**Consequence:** Equal values can feed different essential components into the generated hash implementation, violating Hashable’s contract.

**Why:** The equality body is not analyzed to derive a matching hash policy. Synthesized hashing still follows eligible stored properties.

**Correction:** Implement both operations with the same identity components, or keep structural equality and use a separate ID key.

```swift
struct EntityKey: Hashable {
    let id: Int
    let label: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
let old = EntityKey(id: 1, label: "old")
let new = EntityKey(id: 1, label: "new")
precondition(old == new)
precondition(Set([old, new]).count == 1)
```

**What changes:** Equality and hashing intentionally ignore label together. One observed hash collision cannot validate a wrong implementation; reason about the components for all valid values.

### A stored var default is not a synthesized missing-key policy

**Common pitfall:** Expecting a default-valued nonoptional property to absorb a missing JSON key.

**Mistake:** Standalone runnable example.

```swift
import Foundation

struct Settings: Decodable { var attempts = 3 }
do {
    _ = try JSONDecoder().decode(Settings.self, from: Data("{}".utf8))
    preconditionFailure("Expected missing-key failure")
} catch DecodingError.keyNotFound(let key, _) {
    precondition(key.stringValue == "attempts")
}
```

**Consequence:** Synthesized decoding throws keyNotFound for attempts.

**Why:** The stored initializer expression does not generally replace decoding of a required var key.

**Correction:** Write the actual boundary policy. Here absence defaults to 3, while present null or a wrong type remains invalid.

```swift
import Foundation

struct Settings: Decodable {
    let attempts: Int
    enum CodingKeys: String, CodingKey { case attempts }
    init(from decoder: any Decoder) throws {
        let fields = try decoder.container(keyedBy: CodingKeys.self)
        attempts = fields.contains(.attempts) ? try fields.decode(Int.self, forKey: .attempts) : 3
    }
}
let missing = try JSONDecoder().decode(Settings.self, from: Data("{}".utf8))
precondition(missing.attempts == 3)
for json in [#"{"attempts":null}"#, #"{"attempts":"three"}"#] {
    do {
        _ = try JSONDecoder().decode(Settings.self, from: Data(json.utf8))
        preconditionFailure()
    } catch is DecodingError {}
}
```

**What changes:** The decoder now states which input earns a default instead of hiding malformed input. [[B9 — Codable and Schema Evolution|B9]] expands versioning and migration direction.

## Worked example

### Selection identity and content comparison can coexist

**Setup:** Standalone script. Selection stores only the entity ID; the snapshot retains content equality.

**Predict:** Should a name update change selection or signal changed content?

```swift
struct Snapshot: Identifiable, Equatable {
    let id: Int
    var name: String
}
let old = Snapshot(id: 7, name: "Ada")
let updated = Snapshot(id: 7, name: "Ada L.")
let selection: Set<Int> = [old.id]
precondition(selection.contains(updated.id))
precondition(old != updated)
```

**Outcome:** Selection remains on user 7 while content comparison observes the update.

**Why:** The two operations ask different questions. Making all snapshots ID-equal would discard content differences that a renderer or test may need.

## Interview questions

### Why can same-ID snapshots compare unequal, and how should selection keys differ?

**Answer:** They can represent different content for the same entity. Structural inequality is useful for detecting those changes. Use the stable ID for selection or lookup and compare snapshots when content matters.

**Example:** Standalone runnable example.

```swift
struct Snapshot: Identifiable, Hashable {
    let id: Int
    var name: String
}
let first = Snapshot(id: 1, name: "A")
let second = Snapshot(id: 1, name: "B")
let selectedIDs: Set<Int> = [first.id]
precondition(first.id == second.id && first != second)
precondition(selectedIDs.contains(second.id))
```

**Walkthrough:** Identifiable exposes an ID, while synthesized Hashable/Equatable still describes both stored fields. The selected entity survives its content change.

**Boundary:** ID-only equality can be correct for a deliberately identity-based abstraction, but it should not silently become the comparison used for every content update.

### What goes wrong when == uses only id but hash(into:) is still synthesized?

**Answer:** The hash can include stored content that equality ignores. Equal values must hash from matching essential components, so customize both operations or retain both synthesized operations.

**Example:** Standalone runnable example.

```swift
struct Key: Hashable {
    let id: Int
    let displayName: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
let first = Key(id: 1, displayName: "old")
let second = Key(id: 1, displayName: "new")
precondition(first == second)
precondition(Set([first]).contains(second))
precondition(Set([first, second]).count == 1)
```

**Walkthrough:** Both witnesses use id, so the Set treats the two keys as equal despite their different labels. The compile-only pitfall shows the mismatched synthesized alternative.

**Boundary:** Do not assert particular hash integers or prove correctness from one run. Matching components are the contract; collisions between unequal values are allowed.

### Does a decoded var default handle missing keys, and what does Sendable generate?

**Answer:** A default-valued var does not generally default a missing required key under synthesized decoding. Sendable separately checks or infers transfer safety; it generates no lock, deep copy, or mutation API.

**Example:** Standalone runnable example.

```swift
import Foundation

struct Options: Decodable, Sendable { var attempts = 3 }
do {
    _ = try JSONDecoder().decode(Options.self, from: Data("{}".utf8))
    preconditionFailure()
} catch DecodingError.keyNotFound(let key, _) { precondition(key.stringValue == "attempts") }
let saved = Options(attempts: 3)
var copy = saved
copy.attempts = 7
precondition(saved.attempts == 3 && copy.attempts == 7)
func accept<T: Sendable>(_ value: T) {}
accept(saved)
struct LocalSnapshot { let id: Int }
accept(LocalSnapshot(id: 1))
```

**Walkthrough:** The decoding path still rejects absence. The ordinary value copy has independent integer state; Sendable checking does not supply that behavior or synchronize a shared object. LocalSnapshot is a nonpublic value type whose stored Int permits inferred Sendable, while Options explicitly declares the conformance. [SE-0302](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)

**Boundary:** Explicit decoding is needed for the chosen missing/null policy. A mutable reference field can block checked Sendable even in a struct with let fields.

Two separate **compile-only, intentionally rejected** synthesis/marker examples illustrate those limits:

```swift
struct Action: Equatable { let perform: () -> Void }
```

```swift
final class Storage { var value = 0 }
struct Shared: Sendable { let storage: Storage }
```

### Extension: how do noncopyable protocol support, synthesis, and consumer support differ?

**Answer:** They are separate checks. This Swift 6.4 compiler synthesizes Hashable for the shown noncopyable struct, while the analogous enum synthesis has a limitation. A manual enum witness can work; Set and Codable still have independent Copyable requirements.

**Example:** Standalone runnable example.

```swift
struct Ticket: ~Copyable, Hashable { let id: Int }
func checkTickets() {
    let first = Ticket(id: 7)
    let same = Ticket(id: 7)
    let different = Ticket(id: 8)
    precondition(first == same && first != different)
    var hasher = Hasher()
    first.hash(into: &hasher)
    precondition(first.id == 7)  // Hashing did not consume it.
}
checkTickets()
```

**Walkthrough:** The generated equality/hash witnesses borrow Ticket. Its noncopyability remains intact. [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)

**Boundary:** Do not generalize one successful synthesis to every enum/payload or consumer. The following cases check the current compiler/SDK’s actual boundary; [[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] gives the ownership foundations.

Standalone **intentionally rejected** enum synthesis in this toolchain:

```swift
enum TicketState: ~Copyable, Hashable {
    case empty
    case issued(Int)
}
```

Separate runnable manual correction. Nested borrowed switches avoid the unsupported generated tuple comparison. The tag and payload both participate in equality/hashing:

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

func checkStates() {
    let empty = TicketState.empty
    let first = TicketState.issued(7)
    let same = TicketState.issued(7)
    let different = TicketState.issued(8)
    precondition(empty == empty)
    precondition(first == same && first != different && first != empty)
    var hasher = Hasher()
    first.hash(into: &hasher)
}
checkStates()
```

Prepend the Ticket answer’s setup for the Set rejection; the Codable declaration is standalone. Both are **intentionally rejected** Copyable consumer limits:

```swift
func invalidSet() { _ = Set<Ticket>() }
```

```swift
struct StoredTicket: ~Copyable, Codable { let id: Int }
```

## Focused exercise

**Task:** Compare changed snapshots, then customize identity-key equality and matching hashing.

**Acceptance checks:**

- Same ID and changed content produce unequal snapshots.
- Identity-equal keys behave as one Set member.
- Reflexivity, symmetry, transitivity, and a distinct-ID case hold for the test domain.

**Solution:** Standalone script; no exact hash values are asserted.

```swift
struct Snapshot: Equatable {
    let id: Int
    var name: String
}
struct EntityKey: Hashable {
    let id: Int
    let label: String
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.id == rhs.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
let old = Snapshot(id: 1, name: "Ada")
let new = Snapshot(id: 1, name: "Updated")
precondition(old.id == new.id && old != new)
let first = EntityKey(id: 1, label: old.name)
let second = EntityKey(id: 1, label: new.name)
let third = EntityKey(id: 1, label: "another")
precondition(first == first)
precondition(first == second && second == first)
precondition(first == second && second == third && first == third)
precondition(first != EntityKey(id: 2, label: old.name))
precondition(Set([first, second, third]).count == 1)
precondition(Set([first]).contains(second))
```

**Reasoning:** The snapshot tracks content while EntityKey deliberately ignores label in both witnesses. These assertions test meaningful cases; they do not prove an arbitrary equality implementation for every possible value. Keep collection-observed key state stable.

## In practice

**Choose it when:** Accept synthesis after verifying its derived semantics match the operation’s contract.

**Investigate by:** Compare customized equality components, hash inputs, encoded fields, case iteration, and Sendable checking independently.

**Strong evidence:** Keep selection, content updates, decoding policy, ownership support, and concrete consumer constraints independently testable.

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
- [SE-0194: CaseIterable synthesis](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0194-derived-collection-of-enum-cases.md)
