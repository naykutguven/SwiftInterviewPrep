---
tags:
  - swift
  - ios
  - interview-prep
  - interop-modules-distribution
  - binary-frameworks
  - inlinable
  - usableFromInline
---
## 0. Rubric snapshot

**Rubric expectation**

Understand the cost of `@inlinable`, `@usableFromInline`, and exposing internals for performance. The rubric frames E8 as a binary-framework and public-implementation-leakage topic, not just a micro-optimization topic.

**Caveats**

Performance annotations can freeze implementation detail into a public contract.

**You should be able to answer**

- Why is `@inlinable` not just a free performance win?
- When is `@usableFromInline` the right compromise?

**You should be able to do**

- Review a public utility package and decide which, if any, helpers should become inlineable.

---

## 1. Core mental model

Swift normally hides implementation bodies across module boundaries. A client that imports a binary framework sees the framework’s public interface, but not arbitrary private/internal implementation bodies. This is essential for resilience: the framework author can change internals without forcing clients to recompile.

`@inlinable` deliberately punches a hole through that boundary. It makes the function body available as part of the module interface so the client compiler can optimize calls from another module. That can enable inlining, specialization, constant folding, and removal of abstraction overhead. But it also means the implementation body becomes something clients can compile against. SE-0193 states that `@inlinable` exports the body as part of the module interface, while `@usableFromInline` marks an internal declaration as part of the binary interface without making it part of the source-level public API. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md "swift-evolution/proposals/0193-cross-module-inlining-and-specialization.md at main · swiftlang/swift-evolution · GitHub"))

The dangerous part: client binaries may contain optimized copies of an old implementation. If you later change an `@inlinable` function, old clients may still run old inlined logic until recompiled. SE-0193 explicitly warns that changing an `@inlinable` body can leave binaries using the old definition, the new definition, or a mix of both depending on call sites. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md "swift-evolution/proposals/0193-cross-module-inlining-and-specialization.md at main · swiftlang/swift-evolution · GitHub"))

The key idea:

```text
public API exposes names;
@inlinable exposes bodies;
@usableFromInline exposes internal ABI symbols needed by exposed bodies.
```

For staff-level API design, the question is not “can this be faster?” The question is: “Is this body stable, boring, pure, dependency-light, and worth exposing to every client compiler?”

---

## 2. Essential mechanics

### 2.1 Binary distribution changes the optimization boundary

When a framework is built and shipped separately from its clients, the client compiler cannot freely optimize through the framework’s implementation. Library evolution support exists so a binary framework can be updated without recompiling clients. Swift’s library-evolution documentation says module stability lets modules built with different compiler versions work together, while library evolution lets binary frameworks make additive API changes while remaining binary compatible. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))

For Apple-platform binary distribution, Xcode’s `BUILD_LIBRARY_FOR_DISTRIBUTION` turns on library evolution and module stability. The Swift.org guidance says this setting should be used for binary frameworks intended for distribution, and that enabling library evolution changes performance characteristics. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))

```swift
// Public framework API:
public struct Score {
    private let rawValue: Int

    public init(_ rawValue: Int) {
        self.rawValue = rawValue
    }

    public var normalized: Int {
        max(0, min(rawValue, 100))
    }
}
```

Without `@inlinable`, client code calls `Score.normalized` through the framework boundary. The framework author keeps maximum freedom to change the body.

### 2.2 `@inlinable` exposes the body across modules

`@inlinable` does **not** mean “force inline.” It means “make this body visible to clients so the optimizer may use it.” SE-0193 says the optimizer may inline, specialize, ignore the body, or continue referencing the public entry point. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md "swift-evolution/proposals/0193-cross-module-inlining-and-specialization.md at main · swiftlang/swift-evolution · GitHub"))

Bad:

```swift
private func clampScore(_ value: Int) -> Int {
    max(0, min(value, 100))
}

@inlinable
public func clampedScore(_ value: Int) -> Int {
    clampScore(value)
}
```

With Swift 6.2.1, this fails:

```text
error: global function 'clampScore' is private and cannot be referenced from an '@inlinable' function
note: global function 'clampScore' is not '@usableFromInline' or public
```

Why? An `@inlinable` body can be emitted into client code, so everything it references must also be visible at the ABI level.

Better:

```swift
@usableFromInline
internal func clampScore(_ value: Int) -> Int {
    max(0, min(value, 100))
}

@inlinable
public func clampedScore(_ value: Int) -> Int {
    clampScore(value)
}
```

This keeps `clampScore` out of the source-level public API while making it available to inlinable code.

### 2.3 `@usableFromInline` is “ABI-public, source-internal”

`@usableFromInline` is the compromise attribute. It says: “This declaration is not public API for source users, but it is part of the binary interface because inlinable public code needs to reference it.”

Swift.org’s library-evolution documentation describes `@usableFromInline` declarations as ABI-public but not source-public. They can be referenced from `@inlinable` code but not directly from client source. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))

```swift
public struct Score {
    @usableFromInline
    internal let rawValue: Int

    public init(_ rawValue: Int) {
        self.rawValue = rawValue
    }

    @inlinable
    public var normalized: Int {
        max(0, min(rawValue, 100))
    }
}
```

This compiles with library evolution enabled. But `rawValue` is now ABI-public. That means changing or removing it is no longer “just an internal refactor.”

### 2.4 `@inline(__always)` is not the same concept

`@inlinable` is about **cross-module body visibility**.

`@inline(__always)` is an optimizer directive/hint about **inlining behavior**.

Do not use them interchangeably:

```swift
@inline(__always)
public func fastThing(_ x: Int) -> Int {
    x &* 31
}
```

This does not necessarily make the body available across a binary framework boundary in the same way `@inlinable` does. Conversely:

```swift
@inlinable
public func fastThing(_ x: Int) -> Int {
    x &* 31
}
```

This makes the body available to clients, but still does not guarantee the optimizer will inline it.

---

## 3. Common traps and misconceptions

### Trap 1: “`@inlinable` is a free performance win”

It is not free. It exports implementation detail. Old client binaries may keep old logic. Future semantic changes become risky.

Bad:

```swift
@inlinable
public func normalizeUsername(_ username: String) -> String {
    username
        .trimmingCharacters(in: .whitespacesAndNewlines)
        .lowercased()
}
```

This looks harmless, but username normalization is business logic. Product requirements may change: case sensitivity, Unicode normalization, locale behavior, server compatibility, migration behavior. Making it `@inlinable` increases the chance that old clients keep old behavior.

Better:

```swift
public func normalizeUsername(_ username: String) -> String {
    username
        .trimmingCharacters(in: .whitespacesAndNewlines)
        .lowercased()
}
```

Keep behavior-changing logic behind the binary boundary unless you have a measured reason and a stable semantic contract.

### Trap 2: Making helpers public just to satisfy `@inlinable`

Bad:

```swift
public func clampScore(_ value: Int) -> Int {
    max(0, min(value, 100))
}

@inlinable
public func normalizedScore(_ value: Int) -> Int {
    clampScore(value)
}
```

This pollutes the public source API. Now clients can call `clampScore` directly, documentation has to explain it, and removing it becomes a source-breaking change.

Better:

```swift
@usableFromInline
internal func clampScore(_ value: Int) -> Int {
    max(0, min(value, 100))
}

@inlinable
public func normalizedScore(_ value: Int) -> Int {
    clampScore(value)
}
```

This is the intended compromise: ABI-visible, source-hidden.

### Trap 3: Inlining behavior that depends on private dependencies

An inlinable body can leak more than Swift symbols. It can leak algorithm choices, dependency usage, data representation, and semantic assumptions.

Bad:

```swift
@inlinable
public func makeCacheKey(_ input: String) -> String {
    // Pretend this calls an internal hashing/versioning strategy.
    "v1:" + input.lowercased()
}
```

If the cache-key format later changes to `"v2:"`, old clients may still produce `"v1:"` keys until rebuilt.

Better:

```swift
public func makeCacheKey(_ input: String) -> String {
    "v1:" + input.lowercased()
}
```

Cache keys, hashing, persistence formats, crypto choices, server protocol behavior, and analytics schemas should usually **not** be inlinable.

---

## 4. Direct answers to rubric questions

### Q1. Why is `@inlinable` not just a free performance win?

Because `@inlinable` exposes the function body across module boundaries. That may allow client-side optimization, but it also turns implementation into part of the module interface. Clients may inline or specialize old versions of the body, so future changes can produce mixed behavior across rebuilt and non-rebuilt clients. SE-0193 explicitly warns that changes to `@inlinable` bodies should be considered carefully and are best suited to “obviously correct” algorithms whose future changes are optimizations rather than semantic changes. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md "swift-evolution/proposals/0193-cross-module-inlining-and-specialization.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> `@inlinable` is not just an inline hint. It exports the implementation body to client modules so their optimizer can use it. That can help performance for tiny generic algorithms, but it also leaks implementation detail and makes future changes risky because old clients may have compiled copies of the old body. I would only use it for measured hot paths with stable semantics, not for business logic or code whose behavior may evolve.

### Q2. When is `@usableFromInline` the right compromise?

Use `@usableFromInline` when an `@inlinable` public or package-facing implementation needs an internal helper, type, property, or initializer, but you do not want that symbol to become source-level public API.

It is the right compromise when:

```text
- the public API genuinely benefits from cross-module optimization;
- the helper is stable enough to become ABI-public;
- you want to avoid polluting the source-level public API;
- the helper does not encode volatile business/product behavior.
```

Interview version:

> `@usableFromInline` is appropriate when I have a measured reason to make a public API `@inlinable`, and that body needs internal support code. It keeps the helper source-internal, so clients cannot call it directly, but it makes the helper ABI-public so inlinable client code can reference it. I still treat it as a compatibility commitment, not as ordinary internal code.

---

## 5. Code probe replacement

The rubric has no code probe for E8, so use these probes instead.

### Probe A — private helper from `@inlinable`

Given:

```swift
private func normalize(_ value: Int) -> Int {
    max(0, min(value, 100))
}

@inlinable
public func clampedScore(_ value: Int) -> Int {
    normalize(value)
}
```

### What happens?

Swift 6.2.1 compiler error:

```text
error: global function 'normalize' is private and cannot be referenced from an '@inlinable' function
note: global function 'normalize' is not '@usableFromInline' or public
```

### Why?

```text
@inlinable public function body
        │
        ├── may be emitted into client module
        │
        └── therefore every referenced declaration must be ABI-public
                    │
                    └── private normalize is not ABI-public
```

### Fix or redesign

```swift
@usableFromInline
internal func normalize(_ value: Int) -> Int {
    max(0, min(value, 100))
}

@inlinable
public func clampedScore(_ value: Int) -> Int {
    normalize(value)
}
```

### Why this fix is correct

`normalize` remains hidden from the source-level public API, but it becomes visible enough at the ABI level for inlinable client code.

### Probe B — `@usableFromInline` on `private`

Given:

```swift
@usableFromInline
private func normalize(_ value: Int) -> Int {
    max(0, min(value, 100))
}
```

### What happens?

Swift 6.2.1 compiler error:

```text
error: '@usableFromInline' attribute can only be applied to internal or package declarations, but global function 'normalize' is private
```

### Why?

A `private` declaration is intentionally file/local implementation detail. `@usableFromInline` means the declaration can be referenced from code emitted into clients. Those goals conflict.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Keep the API non-`@inlinable`|Default choice for most SDK/business logic|Less cross-module optimization|
|Mark a small helper `@usableFromInline internal`|Measured hot path; helper has stable semantics|Helper becomes ABI-public|
|Make the helper `public`|Helper is genuinely part of the public API|Larger source API and documentation burden|
|Inline the helper body directly|Tiny, obvious expression|Duplicates logic if used elsewhere|
|Avoid binary distribution; ship source|Internal packages built with the app|Clients must build source; not always possible for SDKs|

---

## 6. Exercise

### Problem

Review a public utility package and decide which, if any, helpers should become inlineable.

### Bad / naive version

```swift
public struct UtilityKit {
    @inlinable
    public static func normalizedUsername(_ username: String) -> String {
        username
            .trimmingCharacters(in: .whitespacesAndNewlines)
            .lowercased()
    }

    @inlinable
    public static func analyticsEventName(_ rawName: String) -> String {
        "ios_v2_" + rawName.lowercased()
    }

    @inlinable
    public static func clampedPercentage(_ value: Int) -> Int {
        clamp(value, lowerBound: 0, upperBound: 100)
    }
}

@usableFromInline
internal func clamp(_ value: Int, lowerBound: Int, upperBound: Int) -> Int {
    max(lowerBound, min(value, upperBound))
}
```

### What is wrong?

```text
normalizedUsername:
- Product/server semantics may change.
- Unicode, locale, and compatibility behavior may evolve.
- Do not inline unless the behavior is permanently frozen.

analyticsEventName:
- Analytics schema/versioning is volatile.
- Old clients could keep emitting old event names.
- Strong candidate to keep behind framework boundary.

clampedPercentage:
- Pure arithmetic.
- Stable semantics.
- Tiny helper.
- Plausible candidate for @inlinable if measured hot enough.
```

### Improved version

```swift
public struct UtilityKit {
    public static func normalizedUsername(_ username: String) -> String {
        username
            .trimmingCharacters(in: .whitespacesAndNewlines)
            .lowercased()
    }

    public static func analyticsEventName(_ rawName: String) -> String {
        "ios_v2_" + rawName.lowercased()
    }

    @inlinable
    public static func clampedPercentage(_ value: Int) -> Int {
        clamp(value, lowerBound: 0, upperBound: 100)
    }
}

@usableFromInline
internal func clamp(_ value: Int, lowerBound: Int, upperBound: Int) -> Int {
    max(lowerBound, min(value, upperBound))
}
```

### Why this is better

The revised design only exposes a tiny, pure, stable arithmetic operation for cross-module optimization. It keeps product behavior, server compatibility, analytics schema decisions, and Unicode/lifecycle policy behind the framework boundary.

The staff-level decision is not “mark small things inlineable.” It is:

```text
Inline stable mechanics.
Hide evolving policy.
Measure before exposing.
Treat ABI-public internals as compatibility commitments.
```

### Production example

A reasonable use case:

```swift
public struct RingBuffer<Element> {
    @usableFromInline
    internal var storage: [Element?]

    @usableFromInline
    internal var head: Int

    @usableFromInline
    internal var capacity: Int

    public init(capacity: Int) {
        self.capacity = capacity
        self.storage = Array(repeating: nil, count: capacity)
        self.head = 0
    }

    @inlinable
    public var isFull: Bool {
        storage.allSatisfy { $0 != nil }
    }

    @inlinable
    public func wrappedIndex(after index: Int) -> Int {
        (index + 1) % capacity
    }
}
```

Even here, be careful. `wrappedIndex(after:)` is stable arithmetic. `isFull` may be too expensive or representation-dependent to expose unless you are comfortable freezing the storage model. A stronger design might inline only `wrappedIndex(after:)` and keep `isFull` non-inlinable.

---

## 7. Production guidance

Use this in production when:

```text
- You ship a binary framework or SDK.
- You have a measured cross-module performance issue.
- The function is tiny, pure, generic, and mechanically stable.
- The implementation depends only on public or stable ABI-public helpers.
- The behavior is unlikely to change except for equivalent optimization.
```

Be careful when:

```text
- The function encodes business rules.
- The function touches persistence, hashing, analytics, server protocols, normalization, caching, or crypto.
- The function references internal representation.
- The function depends on third-party implementation details.
- The framework is built with library evolution enabled.
```

Avoid when:

```text
- You are guessing about performance.
- You are trying to silence compiler errors by exposing internals.
- You are using @inlinable as a style preference.
- You may need to change behavior without forcing client recompilation.
- The code is large, branchy, dependency-heavy, or policy-driven.
```

Debugging checklist:

```text
Is this framework source-distributed or binary-distributed?
Is BUILD_LIBRARY_FOR_DISTRIBUTION enabled?
Is this code actually a measured hot path?
Does @inlinable expose behavior that may need to change?
Does the inlinable body reference internal/private symbols?
Should a helper be @usableFromInline or should the API stop being @inlinable?
Could old clients keep running the old implementation body?
Does the function depend on private imports or implementation-only dependencies?
Would changing this body create mixed old/new behavior in the same app?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `@inlinable` makes functions faster by allowing inlining.

This is incomplete. It misses the binary/module boundary and compatibility cost.

### Senior answer

> `@inlinable` exposes a function body across module boundaries so client code can optimize it. It can help with small generic hot paths, but it leaks implementation detail. `@usableFromInline` lets internal helpers be used by inlinable code without making them public.

Good, but still mostly feature-level.

### Staff-level answer

> I treat `@inlinable` as a public implementation contract. For binary frameworks, it can improve performance by letting the client optimizer see stable generic or tiny pure bodies, but old clients may keep compiled copies of that body. I avoid it for business logic, persistence formats, server behavior, analytics, hashing, and anything likely to evolve. If an inlinable public API needs support code, I use `@usableFromInline` only for stable helpers and treat those helpers as ABI-public. I would require measurement, API review, and compatibility review before adding these annotations to an SDK.

Staff-level questions to ask:

```text
Is this source-distributed or binary-distributed?
Is library evolution enabled?
What measured performance problem are we solving?
Can the implementation body change semantically in the future?
Can old clients safely keep the old body?
Are we leaking dependency types, storage layout, or business rules?
Would @usableFromInline be better than making a helper public?
Would removing @inlinable later fix the problem, or have old clients already compiled against it?
```

---

## 9. Interview-ready summary

`@inlinable` is a cross-module optimization tool, not a generic “make it faster” switch. It exposes a function body as part of the module interface so client modules can inline or specialize it, which is useful for tiny, stable, performance-critical generic code. The cost is that implementation detail becomes part of the compatibility story: old clients may keep compiled copies of old logic. `@usableFromInline` is the compromise for internal helpers needed by inlinable code; it keeps them out of the source-level public API but makes them ABI-public. In production SDKs, I use these annotations sparingly, only after measurement, and avoid them for evolving business logic or dependency-heavy implementation details.

---

## 10. Flashcards

Q: What does `@inlinable` actually expose?

A: The function body across module boundaries, making it available to the client optimizer.

Q: Does `@inlinable` force a function to be inlined?

A: No. It allows the optimizer to use the body. The optimizer may inline, specialize, analyze, or ignore it.

Q: Why can `@inlinable` be risky in a binary framework?

A: Old clients may contain optimized copies of the old body, so changing the implementation later may not affect already-built clients.

Q: What does `@usableFromInline` do?

A: It makes an internal/package declaration usable from inlinable code at the ABI level without making it source-public.

Q: Is `@usableFromInline` ordinary internal API?

A: No. It is source-internal but ABI-public, so it carries compatibility obligations.

Q: When is `@inlinable` most justified?

A: Tiny, pure, stable, measured hot paths, especially generic algorithms where cross-module specialization removes abstraction overhead.

Q: What kind of code should usually not be `@inlinable`?

A: Business rules, persistence formats, analytics naming, server compatibility logic, hashing/crypto choices, localization, and behavior likely to evolve.

Q: What compiler error happens if an `@inlinable` function references a private helper?

A: Swift reports that the private helper cannot be referenced from an `@inlinable` function because it is not `@usableFromInline` or public.

---

## 11. Related sections

- [[B11 — Library evolution and resilience-aware API design]]
- [[E6 — Import visibility and dependency leakage]]
- [[E7 — Resilience, ABI, and module stability]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[F3 — Performance investigation habits]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — E8 rubric, questions, and exercise.
- SE-0193: Cross-module inlining and specialization — defines `@inlinable`, `@usableFromInline`, inlinable-context restrictions, and resilience caveats. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md "swift-evolution/proposals/0193-cross-module-inlining-and-specialization.md at main · swiftlang/swift-evolution · GitHub"))
- Swift.org: Library Evolution in Swift — binary frameworks, module stability, library evolution, `BUILD_LIBRARY_FOR_DISTRIBUTION`, and ABI-public declarations. ([Swift.org](https://swift.org/blog/library-evolution/ "Library Evolution in Swift | Swift.org"))
- Swift.org: ABI Stability and More — distinction between ABI stability, module stability, and library evolution. ([Swift.org](https://swift.org/blog/abi-stability-and-more/ "ABI Stability and More | Swift.org"))