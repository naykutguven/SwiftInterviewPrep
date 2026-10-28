---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - property-wrappers
---

# B13 — Property Wrappers and Generated Storage Semantics

**Learning goals:**

- Trace a wrapped property through backing storage, wrappedValue, projectedValue, initialization, and composition.
- Enforce one invariant on initialization and assignment, and distinguish value-backed copies from shared reference-backed wrappers.
- Check memberwise construction, compound-operation synchronization, explicit actor ownership, and the copyable borrow/mutate form.

**Prerequisites:** [[A2 — `let`, `var`, `mutating`, and method semantics]]/[[A3 — Initialization model]] for mutation and initialization, [[A11 — Access Control]] for access control, and [[D5 — Global Actors and `@MainActor`]]/[[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]] for actor isolation and Mutex.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A volume property should accept only 0...10. A wrapper can store the value and apply that policy every time it is initialized or assigned. For `@Clamped var volume`, Swift stores a **wrapper instance** in `_volume`; reads and writes of volume go through that instance’s wrappedValue. If the wrapper provides projectedValue, `$volume` exposes that separate API.

There is no second independent Int field hiding behind the wrapper. Its actual storage and accessor implementation determine validation, sharing, and effects. The @ syntax does not itself add observation, persistence, or synchronization. [Property wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Keep backing, wrapped, and projected values distinct

Shared Clamped setup. Initialization and the setter use the same normalization; projectedValue reports the configured range. Meter also demonstrates direct initialization of its backing wrapper.

```swift
@propertyWrapper
struct Clamped {
    private var value: Int
    private let range: ClosedRange<Int>
    init(wrappedValue: Int, _ range: ClosedRange<Int>) {
        self.range = range
        value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
    var wrappedValue: Int {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
    var projectedValue: ClosedRange<Int> { range }
}
struct Meter {
    @Clamped(0...10) var volume = 99
    init(initial: Int = 99) {
        _volume = Clamped(wrappedValue: initial, 0...10)
    }
    var limits: ClosedRange<Int> { $volume }
}
var meter = Meter()
precondition(meter.volume == 10 && meter.limits == 0...10)
meter.volume = -3
precondition(meter.volume == 0)
```

**Walkthrough:** Meter initializes _volume with one Clamped instance. Reading volume invokes its getter; assigning -3 invokes its setter. limits reads the projection rather than another stored value. The default and explicit initialization paths both enforce 0...10.

### Composition nests wrapper instances

Prepend Clamped setup. Box is an outer, value-backed wrapper whose projection returns a copy of its wrapped value.

```swift
@propertyWrapper
struct Box<Value> {
    var wrappedValue: Value
    var projectedValue: Value { wrappedValue }
}
struct Score {
    @Box @Clamped(0...10) var points = 99
    func projectedInner() -> Clamped { $points }
}
var score = Score()
precondition(score.points == 10)
var innerCopy = score.projectedInner()
innerCopy.wrappedValue = 3
precondition(innerCopy.wrappedValue == 3 && score.points == 10)
score.points = -1
precondition(score.points == 0)
```

**Walkthrough:** The backing type is Box<Clamped>. A points access follows Box.wrappedValue then Clamped.wrappedValue. The outer projection returns a Clamped value copy, so changing innerCopy does not write back to score. Reversing wrappers can change both the accepted types and effects.

## Rules and pitfalls

### A property observer does not validate its initial value

**Common pitfall:** Clamping only in didSet and assuming initialization passes through that observer.

**Mistake:** Standalone runnable example.

```swift
@propertyWrapper
struct LateClamped {
    var wrappedValue: Int {
        didSet { wrappedValue = min(max(wrappedValue, 0), 10) }
    }
    init(wrappedValue: Int) { self.wrappedValue = wrappedValue }
}
struct Meter { @LateClamped var value = 99 }
var meter = Meter()
precondition(meter.value == 99)
meter.value = 20
precondition(meter.value == 10)
```

**Consequence:** The first read is 99, outside the intended range; only the later assignment is normalized.

**Why:** Assigning the initial stored property during initialization does not run this didSet path. A later setter policy cannot retroactively establish the initial invariant.

**Correction:** Normalize in init and every assignment. Prepend Clamped setup and use its explicit initializer.

```swift
var corrected = Meter(initial: 99)
precondition(corrected.volume == 10)
corrected.volume = -1
precondition(corrected.volume == 0)
```

**What changes:** Both paths now use the same bounds. Clamping intentionally changes invalid input into a boundary value; use a failable or throwing construction API when callers must learn that input was invalid.

### Shared synchronized-wrapper setup

LockedCount holds a Sendable reference whose Mutex protects an Int. The projection exposes an increment operation that locks the entire read/modify/write. Each ordinary getter and setter is also locked, but their separation is significant. This uses the installed Synchronization module under the chapter’s macOS 27 target; use [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks|D12]]’s availability guidance for deployment.

```swift
import Synchronization

@propertyWrapper
struct LockedCount: Sendable {
    private final class Storage: Sendable {
        let value: Mutex<Int>
        init(_ value: Int) { self.value = Mutex(value) }
    }
    private let storage: Storage
    init(wrappedValue: Int) { storage = Storage(wrappedValue) }
    var wrappedValue: Int {
        get { storage.value.withLock { $0 } }
        nonmutating set { storage.value.withLock { $0 = newValue } }
    }
    var projectedValue: Self { self }
    func increment() { storage.value.withLock { $0 += 1 } }
}
struct Counter: Sendable { @LockedCount var count = 0 }
```

### Individually locked accesses can still lose an update

**Common pitfall:** Assuming separately protected reads and writes make a compound increment atomic. Prepend LockedCount setup. This deterministic interleaving models two callers; it does not run a data race.

**Mistake:** Standalone runnable example.

```swift
let counter = Counter()
let firstRead = counter.count
let secondRead = counter.count
counter.count = firstRead + 1
counter.count = secondRead + 1
precondition(counter.count == 1)
```

**Consequence:** Two intended increments leave 1 rather than 2, although every access held the lock.

**Why:** Both reads observed 0 before either write. The lock did not span the whole operation. A property spelling such as count += 1 must not be assumed to call a transactional API.

**Correction:** Use the projection’s whole-operation increment. Prepend LockedCount setup:

```swift
let counter = Counter()
counter.$count.increment()
counter.$count.increment()
precondition(counter.count == 2)
```

**What changes:** Each increment reads and writes inside one withLock call. Other operations cannot interleave within that critical section. This does not make an arbitrary sequence of separate wrapper operations transactional.

## Worked example

### Copying a wrapper can preserve a shared storage reference

**Setup:** Prepend LockedCount setup. Counter’s wrapper has a nonmutating setter and owns a reference-backed Storage.

**Predict:** Does changing a copy’s count also affect the original?

```swift
let original = Counter()
let copy = original
copy.count = 4
precondition(original.count == 4 && copy.count == 4)
original.$count.increment()
precondition(copy.count == 5)
```

**Outcome:** Both observe 4 and then 5.

**Why:** The struct copy copies the wrapper’s reference to the same Storage. Its setter changes synchronized referenced state, so a let Counter binding can still use it. This is an intentional shared-counter contract, unlike Clamped’s independent value storage.

## Interview questions

### What code does a property wrapper effectively synthesize around a property?

**Answer:** It stores the wrapper in a backing property and forwards ordinary access through wrappedValue. A projected property forwards to projectedValue. The following manual form illustrates the access paths; it is not a claim about every generated compiler detail.

**Example:** Prepend Clamped setup.

```swift
struct ManualMeter {
    private var _volume = Clamped(wrappedValue: 99, 0...10)
    var volume: Int {
        get { _volume.wrappedValue }
        set { _volume.wrappedValue = newValue }
    }
    var limits: ClosedRange<Int> { _volume.projectedValue }
}
var manual = ManualMeter()
precondition(manual.volume == 10 && manual.limits == 0...10)
manual.volume = 4
precondition(manual.volume == 4)
```

**Walkthrough:** The explicit _volume owns the value and range. volume forwards reads/writes; limits shows the projection. The actual wrapped Meter spells the projection $volume and may initialize _volume directly inside an initializer.

**Boundary:** Generated initializer signatures and accessor ownership depend on the wrapper declaration. Inspect those separately instead of assuming another stored Int or a public memberwise initializer.

### How can a wrapper accidentally hide thread-safety or actor-isolation problems?

**Answer:** The property spelling can hide shared references, effectful getters, and nonmutating setters. Inspect the actual backing/accessor path and state the owner’s isolation explicitly. A thread-safe individual access does not establish a larger operation’s policy.

**Example:** Prepend LockedCount setup.

```swift
let first = Counter()
let second = first
second.count = 8
precondition(first.count == 8)
first.$count.increment()
precondition(second.count == 9)
```

**Walkthrough:** The copied Counter shares Storage and the let binding still permits its nonmutating setter. Mutex protects these operations, but it does not imply value independence or UI actor ownership.

**Boundary:** Clamped behaves differently because its storage is a value. Naming a wrapper Atomic or State does not establish a synchronization or actor contract; the implementation and owner annotations must do that.

### Why do private initialized wrapped properties and required private properties produce different initializer access?

**Answer:** An initialized private wrapped property can be omitted from an accessible memberwise initializer because its value is already supplied. Required private wrapped storage still needs construction and can restrict that initializer. This wrapper behavior predates Swift 6.4.

**Example:** Standalone runnable example.

```swift
@propertyWrapper
struct Uppercased {
    private var value: String
    init(wrappedValue: String) { value = wrappedValue.uppercased() }
    var wrappedValue: String {
        get { value }
        set { value = newValue.uppercased() }
    }
}
struct Profile {
    var name: String
    @Uppercased private var cache = "seed"
    var cached: String { cache }
}
precondition(Profile(name: "Ada").cached == "SEED")
```

**Walkthrough:** Profile(name:) can omit cache, whose initialized wrapper supplies SEED. Its synthesized private backing storage does not by itself make that initializer private. SE-0502 extends initialized-property exclusion to qualifying ordinary properties. [Memberwise exclusion](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)

**Boundary:** Memberwise initializers remain at most internal. A public type still needs an explicit public initializer for ordinary clients. Do not assume SE-0502 adds a compatibility cache parameter to this already-omitted wrapped property.

Prepend Uppercased/Profile setup. Removing the cache initial value creates required private storage, so this external construction is **intentionally rejected**:

```swift
struct RequiredProfile {
    var name: String
    @Uppercased private var cache: String
}
let invalid = RequiredProfile(name: "Ada", cache: "seed")
```

Prepend Uppercased/Profile setup. Even this factory inside a differently named owner cannot pass a parameter already omitted for its initialized private wrapper; the extra argument is **intentionally rejected**:

```swift
struct CachedProfile {
    var name: String
    @Uppercased private var cache = "seed"
    static func make() -> Self { Self(name: "Ada", cache: "custom") }
}
```

### Why must a synchronized increment protect the whole operation, and when must the wrapper owner declare MainActor explicitly?

**Answer:** Another caller must not write between the increment’s read and write, so they belong in one critical section. Separately, UI ownership requires an explicit MainActor owner in this baseline: Swift 6 no longer infers its outward isolation merely from a wrapped property.

**Example:** Prepend LockedCount setup; no elapsed-time ordering assumption is used.

```swift
let counter = Counter()
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 {
        group.addTask { counter.$count.increment() }
    }
}
precondition(counter.count == 100)
```

**Walkthrough:** All child tasks share the deliberately Sendable Counter storage. increment locks the whole update, and exiting the group joins every child before the assertion. The UI owner below demonstrates the separate actor decision.

**Boundary:** A lock does not make synchronous UI access legal off the main actor, and an actor annotation does not automatically make multi-await operations atomic. [Removed wrapper isolation inference](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0401-remove-property-wrapper-isolation.md)

Standalone **intentionally rejected** example. A main-actor wrappedValue does not infer MainActor onto Owner or its read method in Swift 6:

```swift
@propertyWrapper
struct UIValue<Value> {
    @MainActor var wrappedValue: Value
    @MainActor init(wrappedValue: Value) { self.wrappedValue = wrappedValue }
}
struct Owner {
    @UIValue var count: Int
    func read() -> Int { count }
}
```

Standalone correction: the owner explicitly states the UI isolation contract. Its method and wrapped access are legal on that actor:

```swift
@propertyWrapper
struct UIValue<Value> {
    @MainActor var wrappedValue: Value
    @MainActor init(wrappedValue: Value) { self.wrappedValue = wrappedValue }
}
@MainActor
struct Screen {
    @UIValue var count = 0
    func read() -> Int { count }
}
precondition(Screen().read() == 0)
```

### Extension: What do borrow/mutate accessors change for a copyable wrapper?

**Answer:** They expose borrowed read access and mutable access to stored value state rather than requiring a separate get/set copy sequence. They still require the correct exclusive mutable access and do not add synchronization.

**Example:** Standalone runnable example.

```swift
@propertyWrapper
struct Borrowed<Value> {
    private var storage: Value
    init(wrappedValue: Value) { storage = wrappedValue }
    var wrappedValue: Value {
        borrow { storage }
        mutate { &storage }
    }
}
struct Settings { @Borrowed var count = 0 }
var settings = Settings()
settings.count += 1
precondition(settings.count == 1)
```

**Walkthrough:** The compound mutation uses the wrapper’s mutable access and leaves count equal to 1. No experimental feature flag is needed in this checked compiler. [Borrow/mutate accessors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)

**Boundary:** Value retains the generic Copyable default. A historical noncopyable generic-wrapper variant crashed this exact compiler build, so it is recorded as a toolchain limitation rather than an expected diagnostic or supported example. [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`|C7]]’s ordinary noncopyable property projection is checked independently.

Prepend Borrowed/Settings setup. A let owner cannot provide the required mutable access; this is **intentionally rejected**:

```swift
let fixed = Settings()
fixed.count += 1
```

## Focused exercise

**Task:** Implement one clamping policy for initialization and assignment; explain its generated storage and value-copy behavior.

**Acceptance checks:**

- High and low out-of-range initialization clamp immediately.
- Writes below, within, and at the upper boundary preserve 0...10.
- Copying a value-backed Meter permits independent later writes.

**Solution:** Prepend Clamped/Meter setup. The mechanics show the full wrapper implementation; these checks exercise its supported paths:

```swift
precondition(Meter(initial: 99).volume == 10)
precondition(Meter(initial: -9).volume == 0)
precondition(Meter(initial: 4).volume == 4)
var original = Meter(initial: 99)
original.volume = -1
precondition(original.volume == 0)
original.volume = 6
precondition(original.volume == 6)
original.volume = 10
precondition(original.volume == 10)
var copy = original
copy.volume = 2
precondition(copy.volume == 2 && original.volume == 10)
precondition(copy.limits == 0...10)
```

**Reasoning:** The backing wrapper applies normalization on both entry paths. Copying it copies its Int and range values; it does not share a reference like LockedCount. The projected range reports configuration, and no observation, persistence, or thread-safety contract is implied.

## In practice

**Choose it when:** Use a wrapper for a small, reusable storage policy whose property syntax makes its behavior understandable.

**Investigate by:** Trace backing storage, construction, wrapped/projected access, composition, ownership, and any shared references before judging the visible property.

**Strong evidence:** Verify initialization and mutation separately, then justify whole-operation synchronization and explicit actor ownership with the actual implementation.

## References

### Related guides

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A3 — Initialization model]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[A11 — Access Control]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [SE-0258](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md)
- [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)
- [SE-0401: Remove outward actor-isolation inference from property wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0401-remove-property-wrapper-isolation.md)
- [SE-0507: Borrow/mutate accessors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0507-borrow-accessors.md)
- [Swift.org. "Properties." The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/)
- [Swift.org. "Property Wrapper Implementation Requirements." Swift Compiler Diagnostics](https://docs.swift.org/compiler/documentation/diagnostics/property-wrapper-requirements/)
