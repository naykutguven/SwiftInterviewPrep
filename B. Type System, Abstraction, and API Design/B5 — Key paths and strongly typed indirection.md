---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - key-paths
---

# B5 — Key Paths and Strongly Typed Indirection

**Goal:** Preserve typed projection and write capability when passing a key path.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A key path is an access description, not a cached property value. Its static type preserves the root, value, and possibly write capability. Applying it can run accessors and captured subscript arguments; it does not promise a pure getter or synchronize the root. Prefer a closure when the operation's effects or parameters deserve a more explicit contract. [Key-path expressions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/expressions/)

## Rules and pitfalls

### Preserve the capability callers need

KeyPath reads. WritableKeyPath requires mutable root access for writing. ReferenceWritableKeyPath supports writes without replacing the root binding; the complete path determines that capability, so its root can be a struct containing a reference or a nonmutating setter.

**Common pitfall:** Erasing a ReferenceWritableKeyPath to WritableKeyPath loses the statically available reference-write capability. A let root then fails even if its concrete class could otherwise be mutated.

### Check capture, isolation, and dynamic getters

Sendable key paths must satisfy their captured-value constraints. That does not make every root Sendable or grant access to an actor's state. PartialKeyPath erases Value; AnyKeyPath erases both ends. Appending paths can lose capabilities through the chosen overload.

**Common pitfall:** A descriptor that stores an erased path can accidentally discard writable or Sendable requirements. Preserve them in its signature, or use a clearly isolated closure when that better states the operation.

## Interview questions

### When does a key path communicate the right access contract, and when is a closure clearer?

A path fits reusable typed property projection, sorting descriptors, or mutation when access itself is the contract. A closure is clearer for throwing work, multiple inputs, transformations, asynchronous behavior, or an explicit actor requirement. A computed property's hidden effects remain real through a path.

### Why can a let Screen struct support a reference-writable path while a let Settings class fails after erasure to WritableKeyPath?

A Screen value can reach a class or nonmutating setter through the path, giving reference-writable behavior without replacing Screen. Conversely, after erasure to WritableKeyPath, the static operation requests a mutable root. Let Settings then rejects despite the dynamic path's stronger origin.

### What can a Sendable key path capture, and why does it not make its root safe to share?

Its captured subscript arguments must satisfy the path's transfer requirements. The path value and root value are different objects of checking: a Sendable access description can still target non-Sendable mutable state. Perform application within the root's required isolation.

### Does stable sorting define a canonical order for ties, and what if a computed key changes during comparison?

Stable sorting preserves the original relative order of equal keys, not a canonical order independent of input. A changing computed key can invalidate a consistent comparator. Snapshot keys under suitable isolation and include a deterministic tie-breaker when the product needs canonical ordering.

## Focused exercise

**Task:** Write a typed projection helper and apply a writable path to a mutable root. Explain a rejected write after capability erasure.

**Acceptance checks:**

- Projection retains the concrete Int result type.
- Writable access updates a mutable root.
- Erased read-only capability cannot be used to write.

**Solution:**

Standalone typed projection and mutation:

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
let readOnly: KeyPath<Settings, Int> = \.volume
precondition(project(settings, through: readOnly) == 4)
```

Passing readOnly to write is a separate intended rejection: KeyPath does not expose writable capability. Writing through a let Settings value also rejects.

**Reasoning:** The two helpers expose the minimum capability each uses. Retaining Root and Value avoids casts, while retaining WritableKeyPath is necessary for the mutation. This is local synchronous value mutation, with no claim about actor safety.

## In practice

**Choose it when:** Use key paths for typed access descriptions; use closures for richer effects.

**Investigate by:** Inspect the path's static type after every erasure or appending operation.

**Strong evidence:** Verify root isolation, captured values, stable keys, and writable capability at the descriptor boundary.

### Complete-path capability and ownership — optional

#### Mutation depends on the complete path and its static type

This is an independent fixture; its reference-type Settings is separate from the focused exercise's value type. Compile each rejected variant separately.

A value mutation needs a mutable root access:

```swift
struct Counter {
    var value: Int
}

func incrementedValue() -> Int {
    var counter = Counter(value: 1)
    let path: WritableKeyPath<Counter, Int> = \.value
    counter[keyPath: path] += 1
    return counter.value
}
```

Result: 2. Changing counter to a let binding fails; a WritableKeyPath does not bypass the root's immutability.

Reference mutation can occur through a struct root too:

```swift
final class Settings {
    var volume = 5
}

struct Screen {
    let settings: Settings
}

func changeVolume() -> Int {
    let screen = Screen(settings: Settings())
    let path: ReferenceWritableKeyPath<Screen, Int> = \Screen.settings.volume
    screen[keyPath: path] = 8
    return screen.settings.volume
}
```

Result: 8. The path traverses the struct's immutable reference and mutates the referenced object. ReferenceWritableKeyPath does not require Root itself to be a class. Nonmutating setters can likewise provide reference-style mutation; the property contract matters.

With Settings above, erasing that capability changes which assignment operation is available:

```swift
func erasedMutation() {
    let settings = Settings()
    let path: WritableKeyPath<Settings, Int> = \Settings.volume
    settings[keyPath: path] = 8
}
```

This fails because the statically selected operation requires a mutable root access. Preserve ReferenceWritableKeyPath if the API intentionally writes through a let root. It supplies no synchronization for shared Settings instances.

#### Isolation and ownership constraints remain real

Keep a MainActor property path inside its isolation domain:

```swift
@MainActor final class CounterModel {
    var count = 0
}

@MainActor func updateModel(_ model: CounterModel) -> Int {
    let path: ReferenceWritableKeyPath<CounterModel, Int> = \CounterModel.count
    model[keyPath: path] += 1
    return model.count
}
```

The formation and application compile in MainActor code. This nonisolated formation fails:

```swift
func counterPath() -> KeyPath<CounterModel, Int> {
    \CounterModel.count
}
```

The MainActor path also cannot be returned as `KeyPath & Sendable` in the checked source. Do not erase isolation behind a transferable descriptor. An instance-actor property is rejected as a path even inside that actor in the checked example; expose an actor operation instead.

```swift
actor CounterActor {
    var count = 0
    func path() -> KeyPath<CounterActor, Int> { \CounterActor.count }
}
```

KeyPath's Root/Value generics still have Copyable requirements in this SDK. This explicit signature is rejected:

```swift
struct Token: ~Copyable { var id: Int }
func inspect(_ path: KeyPath<Token, Int>) {}
```

An exploratory inferred `\Token.id` declaration instead aborted this compiler during SIL generation. It is not a supported alternative. The validation record retains the reproduction; use an ownership-aware API rather than advertising general key-path support for noncopyable roots.

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
