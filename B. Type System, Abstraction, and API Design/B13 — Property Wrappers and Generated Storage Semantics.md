---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - property-wrappers
---

# B13 — Property Wrappers and Generated Storage Semantics

**Goal:** Implement clamping for both initialization and later writes through a property wrapper.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A property wrapper stores a wrapper instance and exposes the declared property through wrappedValue. The generated backing storage and optional projectedValue have their own access and mutation behavior. Declarative syntax does not itself supply validation, observation, persistence, or synchronization; those are operations implemented by the wrapper. [SE-0258](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md)

## Rules and pitfalls

### Apply one policy on every path

Initialization must enforce the same invariant as later assignment. The backing _property holds the wrapper, property accesses wrappedValue, and $property accesses projectedValue when supplied. Composition nests wrappers and therefore composes their behavior in a particular order.

**Common pitfall:** Clamping only in didSet misses the initial stored value. Normalize in init(wrappedValue:...) and every setter, or centralize the operation in a shared helper.

### Audit effects beyond individual accessors

Separate locked get and set operations do not make read-modify-write atomic. In Swift 6 mode, a wrapper's isolation does not automatically infer outward isolation onto its owner. Declare MainActor on a UI owner when that is the actual state contract.

**Common pitfall:** A wrapper named Atomic can still lose count += 1 updates if reading and writing lock independently. Protect the whole increment in one critical section or expose a transaction operation; see [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]].

## Interview questions

### What code does a property wrapper effectively synthesize around a property?

The compiler stores an instance such as _volume: Clamped and routes volume through its wrappedValue accessor. If projectedValue exists, $volume exposes it under the generated property's rules. The wrapper, rather than an additional independent Int field, owns this storage policy.

### How can a wrapper accidentally hide thread-safety or actor-isolation problems?

The simple property spelling can conceal shared reference storage, effectful getters, nonmutating setters, or separately locked operations. None automatically makes compound updates atomic or actor-safe. Inspect expanded access paths and require the owner to state its isolation contract.

### Why do private initialized wrapped properties and required private properties produce different initializer access?

A private initialized wrapped property can already be omitted from the more accessible memberwise surface under established wrapper rules. Required private storage can still restrict construction. SE-0502 broadens initialized ordinary-property exclusion in Swift 6.4; it did not invent the wrapper behavior or publish a public initializer.

### Why must a synchronized increment protect the whole operation, and when must the wrapper owner declare MainActor explicitly?

The read, decision, and write must share one lock to prevent another update between them. Individual accessor locks protect only each access. A UI owner needs explicit MainActor when its state is UI-owned, because Swift 6 removed outward isolation inference from property wrappers; the default here remains nonisolated.

## Focused exercise

**Task:** Implement a clamping wrapper. Check initialization, both boundaries, and later mutation; explain its generated storage.

**Acceptance checks:**

- Out-of-range initialization clamps immediately.
- Writes below, within, and at the upper boundary preserve the invariant.
- Copying Meter gives ordinary value-backed storage, without a synchronization claim.

**Solution:**

Standalone wrapper; the accepted domain is the closed range 0...10:

```swift
@propertyWrapper
struct Clamped {
    private var value: Int
    private let range: ClosedRange<Int>
    init(wrappedValue: Int, _ range: ClosedRange<Int>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
    var wrappedValue: Int {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}
struct Meter { @Clamped(0...10) var value = 99 }
var meter = Meter()
precondition(meter.value == 10)
meter.value = -1
precondition(meter.value == 0)
meter.value = 6
precondition(meter.value == 6)
meter.value = 10
precondition(meter.value == 10)
```

**Reasoning:** Both initialization and assignment normalize against the same range. This wrapper deliberately changes invalid inputs into bounds; a domain that must report invalid input should use a throwing/failable operation instead of hiding that decision behind assignment.

## In practice

**Choose it when:** Reuse a small storage policy whose call-site syntax remains honest.

**Investigate by:** Inspect backing and projected values, synthesized initialization, and compound operations.

**Strong evidence:** Test initialization and mutation while auditing the owner's access, isolation, and concurrency requirements.

### Borrow/mutate wrapper and construction history — optional

##### Private initialized properties: keep the history straight

Property wrappers already had special memberwise behavior before Swift 6.4: a private initialized wrapped property can be omitted, and its synthesized private backing storage does not by itself make the owner's initializer private. This checked example prints `SEED`:

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
print(Profile(name: "Ada").cached)
```

Swift 6.4's SE-0502 extends exclusion rules to qualifying ordinary initialized properties; it does not introduce the existing wrapper behavior. Removing cache's initial value in this example leaves a required private parameter and prevents the external Profile(name:) call. A cache argument is not accepted even from a static factory inside this wrapper example; do not assume the new ordinary-property compatibility overload applies to a parameter that was already omitted. Implicit memberwise initializers remain at most internal; public construction still needs an explicit public initializer. [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)

##### Borrow/mutate accessors are supported for this wrapper

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
print(settings.count)
```

This prints `1` without feature flags. The synthesized property still requires the appropriate mutable/exclusive access; a borrow accessor is not a concurrency policy. A noncopyable generic wrapper variant crashed the exact compiler build used here. That is a toolchain limitation, not an intended language diagnostic; it is excluded from baseline exercises. C7's plain noncopyable property projection is independently checked. Do not infer support for every combination merely because the simple copyable wrapper compiles.

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
