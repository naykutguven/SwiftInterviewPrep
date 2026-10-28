---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - key-paths
---

# B5 — Key Paths and Strongly Typed Indirection

**Learning goals:**

- Preserve Root, Value, and read/write capability through projection and descriptor APIs.
- Explain computed accessors, captured subscript arguments, erasure, and literal-to-function conversion.
- Keep root isolation and path Sendable contracts separate, and choose stable sorting keys deliberately.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]/[[A2 — `let`, `var`, `mutating`, and method semantics]] for values and reference-backed mutation, [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for inout, and [[B3 — Generics, Constraints, and Same-Type Relationships]] for generic relationships. Actor examples use [[D4 — Actors and Actor Isolation]]/[[D5 — Global Actors and `@MainActor`]]/[[D6 — `Sendable`, `@Sendable`, and `sending`]].

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A table column can store “read this property from a row” instead of storing one row’s current value. A **key path** is that typed access description. KeyPath<Root, Value> records where access starts and what it returns; applying it actually performs the access.

WritableKeyPath adds mutation through a mutable root. ReferenceWritableKeyPath supports mutation without replacing the root binding, including paths that start at a struct and reach reference storage. Accessors can compute or change state. The path neither caches their results nor synchronizes the root. [Key-path expressions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/expressions/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Separate read capability from write capability

Shared setup named Projection. R and V preserve both ends of the path; write explicitly requests mutable root access.

```swift
struct Settings { var volume: Int }
func project<R, V>(_ root: R, through path: KeyPath<R, V>) -> V {
    root[keyPath: path]
}
func write<R, V>(_ value: V, to root: inout R, through path: WritableKeyPath<R, V>) {
    root[keyPath: path] = value
}
var settings = Settings(volume: 1)
write(4, to: &settings, through: \.volume)
precondition(project(settings, through: \.volume) == 4)
```

**Walkthrough:** The getter helper accepts read-only paths. The setter needs WritableKeyPath and inout R. Neither helper erases V to Any or implies that R can be shared between actors.

### Literal conversion to a function does not apply to every stored path

A key-path literal can be used where a projection function is expected. A stored path can always be applied explicitly in a closure.

```swift
struct User { let name: String }
let users = [User(name: "Ada"), User(name: "Ben")]
let literalFunction: (User) -> String = \User.name
let stored: KeyPath<User, String> = \User.name
precondition(users.map(\.name) == ["Ada", "Ben"])
precondition(literalFunction(users[0]) == "Ada")
precondition(users.map { $0[keyPath: stored] } == ["Ada", "Ben"])
```

**Walkthrough:** The literal participates in the compiler’s literal-to-function conversion. stored is already a path value, so its explicit closure applies it rather than assuming a general conversion. [SE-0249](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0249-key-path-literal-function-expressions.md)

Prepend that User/path setup. A direct stored-path-to-function assignment is **intentionally rejected**:

```swift
let invalid: (User) -> String = stored
```

### Erasure removes static information

PartialKeyPath keeps Root but erases Value; AnyKeyPath erases both. Applying these APIs needs their corresponding dynamic result checks.

```swift
struct User { let age: Int }
struct Other { let value: String }
let user = User(age: 42)
let partial: PartialKeyPath<User> = \User.age
let erased: AnyKeyPath = partial
let incompatible: AnyKeyPath = \Other.value
let result: Any = user[keyPath: partial]
precondition((result as? Int) == 42)
precondition((user[keyPath: erased] as? Int) == 42)
precondition(user[keyPath: incompatible] == nil)
```

**Walkthrough:** The partial path can read from User, but the caller no longer statically knows Int. Full erasure also requires checking root compatibility. Prefer typed projection when those checks provide no useful flexibility.

## Rules and pitfalls

### Erasing reference-write capability changes the available operation

**Common pitfall:** Keeping only WritableKeyPath while expecting a write through a let class reference.

**Mistake:** Compile-only; intentionally rejected.

```swift
final class Settings { var volume = 1 }
let settings = Settings()
let path: WritableKeyPath<Settings, Int> = \Settings.volume
settings[keyPath: path] = 4
```

**Consequence:** The write is rejected because the statically selected operation requires mutable root access.

**Why:** The path’s concrete origin can be reference-writable, but the stored static type exposes only WritableKeyPath’s operation.

**Correction:** Keep the stronger capability when reference mutation is the intended API.

```swift
final class Settings { var volume = 1 }
let settings = Settings()
let path: ReferenceWritableKeyPath<Settings, Int> = \Settings.volume
settings[keyPath: path] = 4
precondition(settings.volume == 4)
```

**What changes:** The root reference stays fixed while the object changes. This provides no synchronization for shared Settings instances.

### A descriptor must retain the capability it promises

**Common pitfall:** Accepting a writable path but storing only KeyPath, then trying to implement a setter.

**Mistake:** Compile-only; intentionally rejected.

```swift
struct Descriptor<Root, Value> {
    let path: KeyPath<Root, Value>
    func set(_ value: Value, on root: inout Root) { root[keyPath: path] = value }
}
```

**Consequence:** Compilation rejects the assignment through the read-only path.

**Why:** The descriptor discarded the capability its method needs.

**Correction:** Store WritableKeyPath for a descriptor that intentionally mutates its root.

```swift
struct Descriptor<Root, Value> {
    let path: WritableKeyPath<Root, Value>
    func set(_ value: Value, on root: inout Root) { root[keyPath: path] = value }
}
struct Row { var score: Int }
var row = Row(score: 1)
Descriptor(path: \Row.score).set(7, on: &row)
precondition(row.score == 7)
```

**What changes:** The stored type now matches the operation. A transferable descriptor must separately preserve Sendable on the path and respect root isolation.

### A computed key is not a cached sorting value

**Common pitfall:** Sorting by a getter that changes on every comparison. Compile-only: this inconsistent comparator is never run.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
final class Reading {
    private var next = 0
    var score: Int {
        next += 1
        return next
    }
}
func invalidSort(_ values: [Reading]) -> [Reading] {
    let path = \Reading.score
    return values.sorted { $0[keyPath: path] < $1[keyPath: path] }
}
```

**Consequence:** Repeated comparisons can observe changing keys, violating the comparator’s consistency requirements.

**Why:** Applying a path runs the getter each time. A key-path spelling does not make an accessor pure or stable.

**Correction:** Snapshot each key once under the model’s required isolation, then compare those snapshots.

```swift
final class Reading {
    let id: Int
    private(set) var accesses = 0
    init(id: Int) { self.id = id }
    var score: Int {
        accesses += 1
        return accesses
    }
}
let values = [Reading(id: 2), Reading(id: 1)]
let snapshots = values.map { (value: $0, score: $0.score) }
let ordered = snapshots.sorted {
    $0.score == $1.score ? $0.value.id < $1.value.id : $0.score < $1.score
}
precondition(ordered.map { $0.value.id } == [1, 2])
precondition(values.map(\.accesses) == [1, 1])
```

**What changes:** Comparisons now use fixed Int keys, and an explicit ID tie-breaker makes equal scores deterministic. Capturing consistent keys from concurrently changing state still needs an isolation policy.

## Worked example

### A Sendable path can still invoke a mutable getter

**Setup:** Standalone script. Reading is deliberately a non-Sendable mutable class and is used synchronously in one context.

**Predict:** Does keeping one path reuse the first value?

```swift
final class Reading {
    private(set) var accesses = 0
    var value: Int {
        accesses += 1
        return accesses
    }
}
func probe() -> [Int] {
    let reading = Reading()
    let path: KeyPath<Reading, Int> & Sendable = \Reading.value
    return [reading[keyPath: path], reading[keyPath: path], reading.accesses]
}
precondition(probe() == [1, 2, 2])
```

**Outcome:** The result is [1, 2, 2].

**Why:** The transferable access description is distinct from the root object. Each application performs a new getter call; neither caching nor root thread safety follows from the path’s Sendable type.

## Interview questions

### When is a key path the right contract, and when is a closure clearer?

**Answer:** Use a path when typed property access is the operation. Use a closure for transformations, multiple inputs, throwing/async work, or an explicit actor requirement. Hidden getter effects remain real through a path.

**Example:** Standalone runnable example.

```swift
struct User {
    let name: String
    let age: Int
}
func column<R, V>(_ rows: [R], path: KeyPath<R, V>) -> [V] {
    rows.map { $0[keyPath: path] }
}
let users = [User(name: "Ada", age: 42)]
let ages: [Int] = column(users, path: \.age)
let descriptions = users.map { "\($0.name): \($0.age) years" }
precondition(ages == [42])
precondition(descriptions == ["Ada: 42 years"])
```

**Walkthrough:** The column preserves the projected Int type. The description is a transformation involving two fields and formatting, which a closure states directly.

**Boundary:** A computed property could also expose the description, but that would be a separate model API decision. A path itself does not declare throws, async execution, or a root’s synchronization policy.

### Why can let Screen support a reference-writable path while a let class fails after erasure?

**Answer:** Capability follows the complete path and its static type. A struct can contain a reference whose object is mutable. Conversely, storing a class path as WritableKeyPath loses the no-root-replacement operation.

**Example:** Standalone runnable example.

```swift
final class Settings { var volume = 5 }
struct Screen { let settings: Settings }
let screen = Screen(settings: Settings())
let complete: ReferenceWritableKeyPath<Screen, Int> = \Screen.settings.volume
screen[keyPath: complete] = 8
precondition(screen.settings.volume == 8)
let settings = Settings()
let preserved: ReferenceWritableKeyPath<Settings, Int> = \Settings.volume
settings[keyPath: preserved] = 9
precondition(settings.volume == 9)
```

**Walkthrough:** Neither let binding is replaced. Both paths reach reference storage. The pitfall’s erased WritableKeyPath demands a different static root access and is rejected.

**Boundary:** A nonmutating setter can similarly provide reference-style mutation. ReferenceWritableKeyPath does not require Root to be a class and does not make shared state safe.

### What can a Sendable path capture, and why does it not make its root Sendable?

**Answer:** Captured subscript arguments must satisfy the path’s transfer rules. The path and root are checked independently; a transferable descriptor can still target non-Sendable mutable state and must be applied within that root’s authorized context.

**Example:** Standalone runnable example.

```swift
struct Row: Sendable {
    let names: [String]
    subscript(index: Int) -> String { names[index] }
}
func column(_ index: Int) -> KeyPath<Row, String> & Sendable { \Row.[index] }
actor Reader {
    func read<R: Sendable, V: Sendable>(_ root: R, through path: KeyPath<R, V> & Sendable) -> V {
        root[keyPath: path]
    }
}
let row = Row(names: ["Ada", "Ben"])
let value = await Reader().read(row, through: column(1))
precondition(value == "Ben")
```

**Walkthrough:** The captured index is Int and the Row is an independent Sendable value, so both cross the actor boundary under their declared contracts. The worked example shows that a Sendable path can also target a mutable non-Sendable root locally. [SE-0418](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0418-inferring-sendable-for-methods.md)

**Boundary:** A key path does not validate the subscript’s index or synchronize accesses. Capturing a non-Sendable reference or forming a path to isolated state requires separate checking.

This standalone **compile-only, intentionally rejected** path captures a non-Sendable subscript argument. A read-only immutable class can explicitly adopt Sendable if its stored state satisfies that contract, or the index can be a Sendable value as above.

```swift
final class Index: Hashable {
    let number: Int
    init(_ number: Int) { self.number = number }
    static func == (lhs: Index, rhs: Index) -> Bool { lhs.number == rhs.number }
    func hash(into hasher: inout Hasher) { hasher.combine(number) }
}
struct Row { subscript(index: Index) -> Int { index.number } }
func invalid(_ index: Index) -> KeyPath<Row, Int> & Sendable { \Row.[index] }
```

### Does stable sorting define a canonical tie order, and what if keys change while comparing?

**Answer:** Stable sorting preserves input-relative order for ties. It does not choose an input-independent canonical order. Changing keys can invalidate comparator consistency; snapshot them under the required isolation, then add a tie-breaker when canonical order matters.

**Example:** Standalone runnable example.

```swift
struct Row {
    let id: Int
    let score: Int
}
let first = [Row(id: 2, score: 10), Row(id: 1, score: 10)]
let second = Array(first.reversed())
let stableFirst = first.sorted { $0.score < $1.score }
let stableSecond = second.sorted { $0.score < $1.score }
func canonical(_ rows: [Row]) -> [Int] {
    rows.sorted { $0.score == $1.score ? $0.id < $1.id : $0.score < $1.score }.map(\.id)
}
precondition(stableFirst.map(\.id) == [2, 1])
precondition(stableSecond.map(\.id) == [1, 2])
precondition(canonical(first) == [1, 2] && canonical(second) == [1, 2])
```

**Walkthrough:** The stable results preserve two different input orders. The explicit ID tie-breaker produces the same order for both. [SE-0372](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0372-document-sorting-as-stable.md)

**Boundary:** The chosen tie-breaker must itself be stable and meaningful in the domain. The computed-getter pitfall demonstrates why a path does not make changing comparison keys consistent.

### Extension: why can appending paths lose writable or Sendable capability?

**Answer:** The selected appending overload determines the result’s static type. Intersection types and erased inputs can select a weaker overload, and returned path declarations do not automatically preserve a literal’s Sendable guarantee.

**Example:** Standalone runnable example.

```swift
struct Address: Sendable { var city: String }
struct User: Sendable { var address: Address }
let first: WritableKeyPath<User, Address> & Sendable = \User.address
let second: WritableKeyPath<Address, String> & Sendable = \Address.city
let appended = first.appending(path: second)
let read: KeyPath<User, String> = appended
precondition(User(address: Address(city: "Berlin"))[keyPath: read] == "Berlin")
let complete: WritableKeyPath<User, String> & Sendable = \User.address.city
var user = User(address: Address(city: "Berlin"))
user[keyPath: complete] = "Paris"
precondition(user.address.city == "Paris")
```

**Walkthrough:** The complete literal states the required final capability directly. In the checked appended expression, read access remains available but the following stronger assignments are rejected.

**Boundary:** Do not recover a lost transfer guarantee with an unchecked cast. Inspect the selected SDK overload and require the capability at your descriptor’s construction boundary.

Each **intentionally rejected** assignment prepends that appending setup:

```swift
let invalid: WritableKeyPath<User, String> = appended
```

```swift
let invalid: KeyPath<User, String> & Sendable = appended
```

### Extension: how do isolation and ownership limit path formation?

**Answer:** An access description must respect isolated properties, and this SDK’s explicit KeyPath<Root, Value> signature still requires Copyable types. A path is not a general escape hatch around actor or ownership rules.

**Example:** Standalone runnable example.

```swift
@MainActor final class CounterModel { var count = 0 }
@MainActor func update(_ model: CounterModel) -> Int {
    let path: ReferenceWritableKeyPath<CounterModel, Int> = \CounterModel.count
    model[keyPath: path] += 1
    return model.count
}
precondition(update(CounterModel()) == 1)
```

**Walkthrough:** Formation and application stay within the model’s MainActor domain. The following declarations separately check transfer, actor-instance access, and noncopyable root limits.

**Boundary:** An earlier inferred noncopyable-root experiment crashed the compiler during SIL generation; it is historical failure evidence, not a supported alternative. This revision checks the explicit unsupported signature without rerunning that crash.

The first two **compile-only, intentionally rejected** variants prepend CounterModel setup; the remaining two are standalone:

```swift
func invalid() -> KeyPath<CounterModel, Int> { \CounterModel.count }
```

```swift
@MainActor func invalidTransfer() -> KeyPath<CounterModel, Int> & Sendable { \CounterModel.count }
```

```swift
actor Counter {
    var count = 0
    func invalid() -> KeyPath<Counter, Int> { \Counter.count }
}
```

```swift
struct Token: ~Copyable { var id: Int }
func invalid(_ path: KeyPath<Token, Int>) {}
```

## Focused exercise

**Task:** Write a typed projection helper and mutate through only the capability the operation needs.

**Acceptance checks:**

- Projection retains Int as its result type.
- Writable access changes a mutable value root.
- A read-only erased path and an immutable root cannot be used for that write.

**Solution:** Prepend Projection setup. Both negative cases immediately below are separate compilations.

```swift
var current = Settings(volume: 1)
write(4, to: &current, through: \.volume)
let projected: Int = project(current, through: \.volume)
precondition(projected == 4)
let readOnly: KeyPath<Settings, Int> = \.volume
precondition(project(current, through: readOnly) == 4)
```

**Reasoning:** Root and Value remain connected without a cast. The setter explicitly retains WritableKeyPath and inout root access; the read-only view deliberately exposes less capability.

Prepend Projection setup followed by the exercise block; these writes are **intentionally rejected**:

```swift
write(9, to: &current, through: readOnly)
```

```swift
let immutable = Settings(volume: 1)
write(9, to: &immutable, through: \.volume)
```

## In practice

**Choose it when:** Use key paths for typed access descriptions; use closures when effects, parameters, or actor context need to be explicit.

**Investigate by:** Inspect the static path type after erasure and appending, then check captured arguments and root access independently.

**Strong evidence:** Validate descriptor capabilities, root isolation, stable comparison keys, and the actual supported ownership domain at each boundary.

## References

### Related guides

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B6 — Any, AnyObject, Metatypes, and Runtime Type Information]]
- [[B13 — Property Wrappers and Generated Storage Semantics]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [Key-path expressions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/expressions/)
- [SE-0161: Typed key paths](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0161-key-paths.md)
- [SE-0249: Key-path literals as functions](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0249-key-path-literal-function-expressions.md)
- [SE-0418: Sendable inference for methods and key paths](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0418-inferring-sendable-for-methods.md)
- [SE-0372: Stable sorting](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0372-document-sorting-as-stable.md)
