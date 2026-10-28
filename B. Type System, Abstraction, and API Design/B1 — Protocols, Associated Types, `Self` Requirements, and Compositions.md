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

**Learning goals:**

- Preserve associated-type and Self relationships through generic, constrained existential, and erased consumers.
- Distinguish readable capabilities, protocol composition, nominal refinement, and semantic validation.
- Keep isolated conformances and noncopyable ownership constraints explicit at their consumer boundaries.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]/[[A2 — `let`, `var`, `mutating`, and method semantics]] for values and mutation, [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for function signatures, and [[A11 — Access Control]] for access control. The extensions use [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]/[[C8 — Copyable, noncopyable types, and `~Copyable`]] ownership and [[D5 — Global Actors and `@MainActor`]]/[[D6 — `Sendable`, `@Sendable`, and `sending`]] isolation.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A cache lookup connects an input key to an output value. If one implementation uses String keys and Int values, a caller must retain that relationship to make a valid lookup. A protocol can express the capability, and its **associated types** name the implementation-specific types used in that capability.

A generic parameter gives one conforming type a name, so its related types remain connected. An **existential**, spelled any P, stores a conformer chosen at runtime behind a protocol interface. You can form that interface even when some operations cannot be called through it: reading a safely erased result and supplying an unknown input are different problems. A **witness** is the implementation that fulfills a protocol requirement. [Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Name the key/value relationship

Shared setup named KeyedLookup. The names in angle brackets are primary associated types, making constraints such as any KeyedLookup<String, Int> expressible; they do not make the protocol an ordinary generic struct.

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

**Walkthrough:** Names supplies String and Int as its witnesses for Key and Value. read carries those types through its argument and result, so a missing key is still an Int? nil rather than a type mismatch.

### Erase the implementation while retaining its useful types

Prepend KeyedLookup setup. A constrained existential and a conforming eraser are two ways to keep the key/value contract. The eraser stores its closure privately.

```swift
struct AnyKeyedLookup<Key: Hashable, Value>: KeyedLookup {
    private let getValue: (Key) -> Value?
    init<L: KeyedLookup>(_ lookup: L) where L.Key == Key, L.Value == Value {
        getValue = lookup.value(for:)
    }
    func value(for key: Key) -> Value? { getValue(key) }
}
let selected: any KeyedLookup<String, Int> = Names(storage: ["Ada": 1])
let wrapped = AnyKeyedLookup(Names(storage: ["Ada": 1]))
precondition(selected.value(for: "Ada") == 1)
precondition(read(wrapped, key: "Ada") == 1)
```

**Walkthrough:** Both callers know that inputs are String and outputs are Int?. The concrete wrapper itself conforms to KeyedLookup; an existential does not gain general self-conformance merely by being stored.

## Rules and pitfalls

### Any plus a cast can merge absence with invalid data

**Common pitfall:** Erasing both keys and values before establishing a typed registration invariant.

**Mistake:** Standalone runnable example.

```swift
struct LooseLookup {
    let storage: [AnyHashable: Any]
    func integer(for key: AnyHashable) -> Int? { storage[key] as? Int }
}
let lookup = LooseLookup(storage: ["Ada": "wrong type"])
precondition(lookup.integer(for: "Ada") == nil)
precondition(lookup.integer(for: "missing") == nil)
```

**Consequence:** A present String value looks exactly like an absent key to the integer caller.

**Why:** The cast became the type contract, and its failure was silently mapped to nil.

**Correction:** Keep the String/Int relationship in storage and in the consumer. Prepend KeyedLookup setup.

```swift
let lookup: any KeyedLookup<String, Int> = Names(storage: ["Ada": 7])
precondition(lookup.value(for: "Ada") == 7)
precondition(lookup.value(for: "missing") == nil)
```

**What changes:** The dictionary cannot be initialized with a String value under this contract. When erasure is required, a typed wrapper can validate adaptation once and hide the erased storage.

### Sendable does not make every witness nonisolated

**Common pitfall:** Using a MainActor-isolated conformance from a nonisolated generic caller. Sendable permits the value’s transfer/sharing; conformance isolation separately governs witness use.

**Mistake:** Compile-only; intentionally rejected.

```swift
protocol Titled { func title() -> String }
struct Label: Sendable, @MainActor Titled {
    let text: String
    @MainActor func title() -> String { text }
}
func render<T: Titled>(_ value: T) -> String { value.title() }
func outside(_ value: Label) -> String { render(value) }
```

**Consequence:** The compiler rejects use of Label’s isolated conformance from outside its required actor domain.

**Why:** Knowing that Label is Sendable does not authorize invoking its witnesses on an arbitrary actor.

**Correction:** Preserve the conformance’s domain in the generic operation. This UI-label example uses MainActor explicitly.

```swift
protocol Titled { func title() -> String }
struct Label: Sendable, @MainActor Titled {
    let text: String
    @MainActor func title() -> String { text }
}
@MainActor func render<T: Titled>(_ value: T) -> String { value.title() }
let label = Label(text: "Ada")
precondition(render(label) == "Ada")
```

**What changes:** The consumer now uses the witness in its authorized domain. [[D5 — Global Actors and `@MainActor`|D5]]/[[D14 — Approachable Concurrency and Default Isolation|D14]] cover broader inference and caller-isolation rules.

## Worked example

### The same lookup through three consumer shapes

**Setup:** Prepend KeyedLookup setup. No I/O or concurrent state is involved.

**Predict:** Which boundary preserves the key and result types while permitting implementation variation?

```swift
func generic<L: KeyedLookup>(_ lookup: L) -> Int? where L.Key == String, L.Value == Int {
    lookup.value(for: "Ada")
}
func runtime(_ lookup: any KeyedLookup<String, Int>) -> Int? {
    lookup.value(for: "Ada")
}
struct SavedLookup: KeyedLookup {
    let get: (String) -> Int?
    func value(for key: String) -> Int? { get(key) }
}
let concrete = Names(storage: ["Ada": 1])
let erased = SavedLookup(get: concrete.value(for:))
precondition(generic(concrete) == 1)
precondition(runtime(concrete) == 1)
precondition(generic(erased) == 1)
```

**Outcome:** All three return Int? and accept only String keys.

**Why:** The generic keeps the caller’s implementation type; the existential permits runtime selection; the wrapper supplies its own conformance while privately forwarding behavior. Choose the shape from caller operations, not a presumed allocation cost.

## Interview questions

### Why can any Parser return a parsed value while an unconstrained lookup cannot accept a String key?

**Answer:** A returned associated value can be safely erased to its known upper bound. An associated input needs a known type relationship: an arbitrary String cannot stand in for every hidden Key.

**Example:** Standalone runnable example.

```swift
protocol Parser {
    associatedtype Output
    func parse() -> Output
}
struct CountParser: Parser { func parse() -> Int { 7 } }
let parser: any Parser = CountParser()
let output: Any = parser.parse()
precondition((output as? Int) == 7)
protocol Lookup<Key, Value> {
    associatedtype Key: Hashable
    associatedtype Value
    func get(_ key: Key) -> Value?
}
struct Counts: Lookup {
    func get(_ key: String) -> Int? { key == "Ada" ? 7 : nil }
}
let typed: any Lookup<String, Int> = Counts()
precondition(typed.get("Ada") == 7)
```

**Walkthrough:** parse’s Int result can become Any without accepting an invalid input. typed retains String as its key type, so its call is valid. Erasing Counts to unconstrained any Lookup would lose that input relationship. [SE-0309](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)

**Boundary:** The unconstrained parser result has lost its concrete static type. A caller needing integer arithmetic should preserve Output == Int rather than immediately casting every result.

Prepend that answer’s declarations. This independent **compile-only, intentionally rejected** function does not know the lookup’s Key:

```swift
func invalid(_ lookup: any Lookup) { _ = lookup.get("Ada") }
```

### Why can a Self result be erased while independent existential Self inputs cannot combine? How does a generic helper double one value?

**Answer:** Self in a result can erase back to the protocol. Self in an input must match the receiver’s hidden type. Opening one existential in a generic helper gives a single T, allowing that same value to combine with itself.

**Example:** Standalone runnable example.

```swift
protocol Combining {
    var amount: Int { get }
    func fresh() -> Self
    func combined(with other: Self) -> Self
}
struct Count: Combining {
    let amount: Int
    func fresh() -> Self { self }
    func combined(with other: Self) -> Self { Count(amount: amount + other.amount) }
}
func doubled<T: Combining>(_ value: T) -> T { value.combined(with: value) }
let value: any Combining = Count(amount: 7)
let fresh: any Combining = value.fresh()
let twice: any Combining = doubled(value)
precondition(fresh.amount == 7 && twice.amount == 14)
```

**Walkthrough:** fresh preserves behavior while erasing its result. doubled operates on one opened type inside the generic body, then its result erases again. Two independent existential arguments do not establish that common type.

**Boundary:** This permits a single-value generic operation, not unrestricted mixing of erased values. Keep a named generic relationship before erasure when two inputs must have the same type.

With that Combining setup, this **compile-only, intentionally rejected** function supplies an independently erased argument:

```swift
func invalid(_ first: any Combining, _ second: any Combining) -> any Combining {
    first.combined(with: second)
}
```

### Does P & Q imply conformance to a protocol refining both? Does get-only prove immutable storage?

**Answer:** No. Composition requires two existing conformances; a newly declared refining protocol requires its own nominal conformance. A get-only requirement promises readable access, not immutable storage, purity, or constant cost.

**Example:** Standalone runnable example.

```swift
protocol Named { var name: String { get } }
protocol Identified { var id: Int { get } }
protocol Record: Named, Identified {}
final class User: Named, Identified {
    var name = "Ada"
    let id = 7
}
func composed(_ value: any Named & Identified) -> String { value.name }
extension User: Record {}  // Explicit conformance to the refining protocol.
func refined<R: Record>(_ value: R) -> Int { value.id }
let user = User()
let readable: any Named = user
user.name = "Changed"
precondition(readable.name == "Changed")
precondition(composed(user) == "Changed" && refined(user) == 7)
```

**Walkthrough:** The explicit extension establishes Record conformance. The read-only interface still observes mutation through the original class reference.

**Boundary:** A marker protocol with no enforceable requirements cannot prove a business invariant such as a validated email. Validate domain data at construction and expose operations that preserve the invariant.

Prepend that answer’s declarations. This separate type has both capabilities but no Record conformance, so the call is **intentionally rejected**:

```swift
struct Separate: Named, Identified {
    let name = "Ada"
    let id = 7
}
let invalid = refined(Separate())
```

### Why can a Sendable value have a MainActor-isolated conformance, and what must the consumer preserve?

**Answer:** Sendable concerns the value’s safe transfer/sharing. Isolated conformance concerns where its protocol witnesses may be used. A generic consumer must preserve the latter boundary even when its argument is Sendable.

**Example:** Standalone runnable example.

```swift
protocol Titled { func title() -> String }
struct Label: Sendable, @MainActor Titled {
    let text: String
    @MainActor func title() -> String { text }
}
func forward<T: Sendable>(_ value: T) -> T { value }
func transfer(_ label: Label) -> Label { forward(label) }
@MainActor func render<T: Titled>(_ value: T) -> String { value.title() }
let label = transfer(Label(text: "Ada"))
precondition(label.text == "Ada")
precondition(render(label) == "Ada")
```

**Walkthrough:** transfer is nonisolated and uses only Sendable. render invokes Titled’s witness on MainActor. The pitfall shows the rejected nonisolated use of that isolated conformance.

**Boundary:** Neither an immutable binding nor a generic Sendable requirement makes every protocol witness nonisolated. Imported modules and conformance inference retain their own declared contracts.

### Extension: what must be suppressed for a noncopyable factory payload, and why does Hashable not imply Set support?

**Answer:** Suppress the relevant implicit Copyable defaults at the protocol, associated type, and generic use sites. Then check each consumer independently: a generalized conformance does not generalize every container or unrelated protocol.

**Example:** Standalone runnable example.

```swift
struct Ticket: ~Copyable, Hashable {
    let id: Int
    static func == (left: borrowing Ticket, right: borrowing Ticket) -> Bool { left.id == right.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
protocol TicketFactory<Output>: ~Copyable {
    associatedtype Output: ~Copyable = Ticket
    func make() -> Output
}
struct DefaultFactory: TicketFactory {
    func make() -> Output { Ticket(id: 7) }
}
func make<F: TicketFactory & ~Copyable>(_ factory: borrowing F) -> F.Output
where F.Output: ~Copyable { factory.make() }
func probe() {
    let ticket = make(DefaultFactory())
    let same = Ticket(id: 7)
    precondition(ticket == same)
    var hasher = Hasher()
    ticket.hash(into: &hasher)
}
probe()
```

**Walkthrough:** The factory uses a noncopyable default output, and make explicitly permits both its conformer and Output to be noncopyable. Ticket supplies borrowed equality/hash witnesses. [SE-0503](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md), [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md)

**Boundary:** The selected SDK still requires Copyable for Set elements and CustomStringConvertible conformers. Suppression does not promise deep uniqueness, synthesis for every type, or a container implementation that never copies. [[B3 — Generics, Constraints, and Same-Type Relationships|B3]]/[[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] develop the separate defaults and SDK limits.

Two independent **compile-only, intentionally rejected** consumers prepend the TicketFactory answer’s setup:

```swift
func invalidSet() { _ = Set<Ticket>() }
```

```swift
func ordinary<F: TicketFactory>(_ factory: F) -> F.Output { factory.make() }
func invalidDefault() { _ = ordinary(DefaultFactory()) }
```

Separate **intentionally rejected** SDK protocol limit:

```swift
struct PrintedTicket: ~Copyable, CustomStringConvertible {
    var description: String { "ticket" }
}
```

## Focused exercise

**Task:** Define a keyed lookup and a typed consumer without unchecked casts.

**Acceptance checks:**

- Known and absent String keys yield Int? results.
- An explicitly Int key is rejected at compilation.
- The same relationship is retained through a constrained existential.

**Solution:** Prepend KeyedLookup setup. The rejected call immediately afterward is a separate compilation.

```swift
let directory = Names(storage: ["Ada": 1, "Ben": 2])
precondition(read(directory, key: "Ada") == 1)
precondition(read(directory, key: "missing") == nil)
let selected: any KeyedLookup<String, Int> = directory
precondition(selected.value(for: "Ben") == 2)
```

**Reasoning:** The key and result share one implementation’s associated-type witnesses. Runtime implementation choice can erase Names without erasing String and Int. A plain generic remains sufficient when that variation is unnecessary.

Prepend KeyedLookup setup; this **intentionally rejected** key is Int rather than Names.Key == String:

```swift
let invalid = read(Names(storage: ["Ada": 1]), key: 7)
```

## In practice

**Choose it when:** Use protocols for actual capability boundaries, including one production implementation when testing or ownership separation benefits.

**Investigate by:** Identify every unknown type position and whether callers supply it or receive it. Check the witness’s isolation separately.

**Strong evidence:** Compare a generic, a constrained existential, and an owned eraser using the caller’s actual operations; verify ownership defaults and SDK consumers independently.

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
