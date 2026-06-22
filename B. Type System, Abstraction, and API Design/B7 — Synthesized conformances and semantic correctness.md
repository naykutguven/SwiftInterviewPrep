---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
  - synthesized-conformances
---
## 0. Rubric snapshot

**Rubric expectation**

Know when Swift synthesizes `Equatable`, `Hashable`, `Codable`, `Sendable`, and `CaseIterable`, and when synthesis is semantically wrong even if legal. The rubric’s key caveat is that synthesized correctness is **structural, not domain-aware**.

**You should be able to answer**

- Why might synthesized `Equatable` be technically valid but semantically wrong for a model?
- When should you hand-write `Hashable` instead of using synthesis?

**You should be able to do**

- Given an entity type with stable identity and frequently changing fields, decide what equality should mean and implement it.

---

## 1. Core mental model

Synthesized conformances are compiler-generated implementations of protocol requirements. Swift can generate repetitive, mechanically correct implementations when the shape of a type makes the implementation obvious: compare all stored properties, hash all stored properties, encode/decode all codable properties, expose all enum cases, or check that data can safely cross concurrency boundaries.

The compiler sees **structure**. It does not understand your domain. For a value type like `Point(x:y:)`, structural equality is probably correct. For an entity like `User(id:name:avatarURL:lastSeenAt:)`, structural equality may be wrong because the domain identity is usually the stable `id`, not every mutable field.

The senior/staff-level question is not “Can Swift synthesize this?” It is:

```text
Does the synthesized conformance express the semantic contract this type promises to callers?
```

This matters because conformances are not harmless implementation details. `Equatable` changes comparison semantics. `Hashable` controls `Set` and `Dictionary` correctness. `Codable` becomes a persistence or wire-format contract. `Sendable` becomes a concurrency-safety contract. `CaseIterable` exposes a finite case list, usually in declaration order.

Swift documentation and Swift Evolution describe synthesis as reducing boilerplate in cases where implementation is mechanically derivable, not as proof that the generated behavior matches business identity or long-term API compatibility. SE-0185 explicitly frames synthesized `Equatable`/`Hashable` as boilerplate reduction for memberwise implementations. ([GitHub, "Synthesizing Equatable and Hashable Conformance"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0185-synthesize-equatable-hashable.md))

The key idea:

```text
Synthesis answers: "Can the compiler derive this from stored shape?"
Semantic correctness asks: "Is this the contract users of the type should rely on?"
```

---

## 2. Essential mechanics

### 2.1 `Equatable` synthesis is memberwise for structs and case/payload-based for enums

For structs, synthesized `==` compares stored instance properties. Static and computed properties are not part of synthesis. For enums, equality checks that both values are the same case and that associated values are equal. SE-0185 describes these rules for structs and enums. ([GitHub, "Synthesizing Equatable and Hashable Conformance"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0185-synthesize-equatable-hashable.md))

```swift
struct Point: Equatable {
    var x: Int
    var y: Int
}

print(Point(x: 1, y: 2) == Point(x: 1, y: 2))
```

Exact output:

```text
true
```

This is a good use of synthesis because the domain meaning of a `Point` is exactly its coordinates.

Counterexample:

```swift
struct SynthesizedUser: Equatable, Hashable {
    let id: Int
    var name: String
}

let original = SynthesizedUser(id: 1, name: "A")
let updated = SynthesizedUser(id: 1, name: "B")

print(original == updated)
```

Exact output:

```text
false
```

This is mechanically correct but may be semantically wrong. If `id == 1` means “same persisted user,” then changing `name` should not make it a different entity.

---

### 2.2 `Hashable` synthesis follows the same structural model

A synthesized `Hashable` implementation hashes the same structural state used for equality. Apple’s `Hashable` documentation states that `Hashable` inherits from `Equatable` and that the compiler can synthesize requirements when a custom type declares conformance and meets the criteria. ([Apple Developer, "Hashable"](https://developer.apple.com/documentation/swift/hashable))

Good:

```swift
struct Coordinate: Hashable {
    let latitude: Double
    let longitude: Double
}
```

Risky:

```swift
struct UserEntity: Hashable {
    let id: Int
    var displayName: String
    var avatarURL: URL?
}
```

The synthesized implementation hashes `id`, `displayName`, and `avatarURL`. That can be wrong if the type is used as a stable entity key. A renamed user would hash differently.

Better:

```swift
struct UserEntity: Identifiable, Hashable {
    let id: Int
    var displayName: String
    var avatarURL: URL?

    static func == (lhs: UserEntity, rhs: UserEntity) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

Now equality and hashing are based on stable identity.

---

### 2.3 `Codable` synthesis is convenient, but schema is a contract

`Codable` synthesis works when stored properties are codable and the default property-to-key mapping matches the encoded representation. SE-0166 introduced Swift archival and serialization around `Encodable` and `Decodable`, including compiler-generated implementations when all properties are codable. ([GitHub, "Swift Archival & Serialization"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md))

Good for local DTOs:

```swift
struct LoginResponse: Codable {
    let accessToken: String
    let expiresIn: TimeInterval
}
```

But synthesis becomes fragile when the encoded form is persisted or externally owned:

```swift
struct Settings: Codable {
    var selectedSortMode: SortMode
}
```

If an older app persisted this:

```json
{
  "sortMode": "byName"
}
```

and the new model expects this:

```json
{
  "selectedSortMode": "byName"
}
```

synthesis is no longer enough. You need custom `CodingKeys` or `init(from:)`.

---

### 2.4 `CaseIterable` synthesis is for simple finite enums

`CaseIterable` gives a type a static `allCases` collection. SE-0194 introduced it to represent finite, enumerable sets of values and proposed derived implementation for simple enums. ([GitHub, "Derived Collection of Enum Cases"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0194-derived-collection-of-enum-cases.md)) Apple’s documentation notes that synthesized `allCases` provides cases in declaration order. ([Apple Developer, "CaseIterable"](https://developer.apple.com/documentation/swift/caseiterable))

```swift
enum FeedFilter: CaseIterable {
    case all
    case unread
    case favorites
}

print(FeedFilter.allCases)
```

Typical output:

```text
[all, unread, favorites]
```

Be careful: declaration order may leak into UI order, analytics order, tests, or persistence assumptions.

Better for UI:

```swift
enum FeedFilter: CaseIterable {
    case all
    case unread
    case favorites

    var title: String {
        switch self {
        case .all: "All"
        case .unread: "Unread"
        case .favorites: "Favorites"
        }
    }

    static let displayOrder: [FeedFilter] = [.all, .favorites, .unread]
}
```

Use `allCases` for enumeration. Use explicit arrays for domain order.

---

### 2.5 `Sendable` synthesis/inference is about cross-isolation safety, not general thread safety

`Sendable` marks values that can be safely transferred across concurrency domains. Swift’s concurrency documentation describes a sendable type as one that can be shared from one concurrency domain to another. ([Swift.org, "Concurrency"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)) SE-0302 introduced `Sendable` and `@Sendable` to type-check value passing between structured concurrency tasks and actor messages. ([GitHub, "Sendable and @Sendable Closures"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md))

```swift
struct SearchQuery: Sendable {
    let text: String
    let limit: Int
}
```

This is fine because the stored properties are sendable value types.

Counterexample:

```swift
final class MutableBox {
    var value = 0
}

struct Payload: Sendable {
    let box: MutableBox
}
```

Exact Swift 6.2 compiler error:

```text
error: stored property 'box' of 'Sendable'-conforming struct 'Payload' has non-Sendable type 'MutableBox'
```

The compiler is not saying “classes are always bad.” It is saying this specific payload contains shared mutable reference state without a proven concurrency-safety story.

---

## 3. Common traps and misconceptions

### Trap 1: “Synthesized `Equatable` means correct equality”

Bad:

```swift
struct Article: Equatable {
    let id: UUID
    var title: String
    var body: String
    var updatedAt: Date
}
```

This says two articles are unequal if `updatedAt` changes. That might be correct for a **snapshot**, but wrong for an **entity**.

Better:

```swift
struct Article: Identifiable, Equatable {
    let id: UUID
    var title: String
    var body: String
    var updatedAt: Date

    static func == (lhs: Article, rhs: Article) -> Bool {
        lhs.id == rhs.id
    }
}

extension Article {
    var contentFingerprint: ContentFingerprint {
        ContentFingerprint(title: title, body: body, updatedAt: updatedAt)
    }
}

struct ContentFingerprint: Equatable {
    let title: String
    let body: String
    let updatedAt: Date
}
```

Separate “same entity” from “same content snapshot.”

---

### Trap 2: Hashing mutable fields used in identity collections

Bad:

```swift
struct User: Hashable {
    let id: Int
    var name: String
}
```

If the type is used as a `Set` element or dictionary key, synthesized hashing includes `name`. If your semantic key is `id`, this is the wrong contract.

Better:

```swift
struct User: Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

This aligns equality and hashing with stable identity.

---

### Trap 3: Treating `Codable` synthesis as schema evolution

Synthesis handles the current shape. It does not automatically handle renamed keys, deleted fields, unknown enum cases, lossy migration, old app data, backend inconsistencies, or versioned payloads.

Bad:

```swift
struct Settings: Codable {
    var sortOrder: SortOrder
}
```

Then later:

```swift
struct Settings: Codable {
    var selectedSortOrder: SortOrder
}
```

This silently changes the expected key unless you preserve the old mapping.

Better:

```swift
struct Settings: Codable {
    var selectedSortOrder: SortOrder

    enum CodingKeys: String, CodingKey {
        case selectedSortOrder = "sortOrder"
    }
}
```

For more complex migrations, write `init(from:)`.

---

### Trap 4: Using `CaseIterable.allCases` as a permanent business contract

Bad:

```swift
enum SubscriptionTier: CaseIterable {
    case free
    case pro
    case enterprise
}

// Used for billing order, UI order, analytics index, and tests.
let tierIndex = SubscriptionTier.allCases.firstIndex(of: .pro)
```

Better:

```swift
enum SubscriptionTier: CaseIterable {
    case free
    case pro
    case enterprise

    static let billingOrder: [SubscriptionTier] = [.free, .pro, .enterprise]

    var analyticsName: String {
        switch self {
        case .free: "free"
        case .pro: "pro"
        case .enterprise: "enterprise"
        }
    }
}
```

Declaration order is fine for simple iteration. It should not become hidden product logic.

---

### Trap 5: Thinking `Sendable` synthesis proves deep thread safety

`Sendable` is about safe transfer across isolation domains. It does not mean every operation on the type is atomic, ordered, or logically race-free.

Bad:

```swift
final class Counter: @unchecked Sendable {
    var value = 0
}
```

Better:

```swift
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }

    func currentValue() -> Int {
        value
    }
}
```

Use `@unchecked Sendable` only when you can prove the internal synchronization or immutability invariant.

---

## 4. Direct answers to rubric questions

### Q1. Why might synthesized `Equatable` be technically valid but semantically wrong for a model?

Because synthesized `Equatable` compares structural state, not domain identity.

For a pure value like `Money(amount:currency)` or `Point(x:y:)`, structural equality is likely correct. For an entity like `User`, `Order`, `Conversation`, or `Article`, equality often means “same stable identity,” not “every current field has the same value.”

```swift
struct Conversation: Equatable {
    let id: UUID
    var title: String
    var unreadCount: Int
}
```

Synthesized equality says:

```text
same id + same title + same unreadCount
```

But the domain may require:

```text
same id
```

Interview version:

> Synthesized `Equatable` is memberwise. That is correct for many value objects, but not automatically correct for entities. If a model has stable identity and mutable fields, memberwise equality may treat an updated version of the same entity as a different value. I decide whether the type represents a value snapshot or a domain entity. For snapshots, synthesis is fine. For entities, I usually implement equality explicitly around the stable identifier and use a separate content comparison when needed.

---

### Q2. When should you hand-write `Hashable` instead of using synthesis?

Hand-write `Hashable` when the hash key should be a semantic subset of the stored state, usually a stable identifier.

You should hand-write it when:

```text
The type has stable identity.
Some stored properties are mutable.
Some stored properties are caches, formatting helpers, closures, dependencies, or derived data.
The type is used in Set or Dictionary.
The public API must promise stable key semantics.
```

Example:

```swift
struct Product: Identifiable, Hashable {
    let id: String
    var title: String
    var price: Decimal
    var isFavorite: Bool

    static func == (lhs: Product, rhs: Product) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

The equality/hash invariant must still hold:

```text
If a == b, then a.hash(into:) and b.hash(into:) must feed equivalent data.
```

Interview version:

> I hand-write `Hashable` when structural hashing is not the semantic key. The classic case is an entity with a stable `id` and mutable display fields. Synthesized hashing would include those mutable fields, which is wrong for set/dictionary semantics if the entity’s identity is stable. I make `==` and `hash(into:)` use the same stable identity and keep content comparison as a separate operation if the UI or diffing layer needs it.

---

## 5. Code probe

B7 has no rubric code probe, so use these focused probes.

### Probe A — Valid synthesis

Given:

```swift
struct Point: Equatable {
    var x: Int
    var y: Int
}

print(Point(x: 1, y: 2) == Point(x: 1, y: 2))
```

### What happens?

```text
true
```

### Why?

Swift synthesizes `==` because all stored properties are `Equatable`.

Equivalent hand-written implementation:

```swift
extension Point {
    static func == (lhs: Point, rhs: Point) -> Bool {
        lhs.x == rhs.x && lhs.y == rhs.y
    }
}
```

This is semantically correct because the value of a `Point` is its coordinates.

---

### Probe B — Legal but semantically suspicious synthesis

Given:

```swift
struct SynthesizedUser: Equatable, Hashable {
    let id: Int
    var name: String
}

let original = SynthesizedUser(id: 1, name: "A")
let updated = SynthesizedUser(id: 1, name: "B")

print(original == updated)
```

### What happens?

```text
false
```

### Why?

Synthesized equality compares both `id` and `name`.

```text
original.id == updated.id      true
original.name == updated.name  false
overall equality               false
```

That may be wrong if the domain says both values represent the same user entity.

### Fix or redesign

```swift
struct User: Identifiable, Hashable {
    let id: Int
    var name: String

    static func == (lhs: User, rhs: User) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

### Why this fix is correct

Equality and hashing now express stable identity. `name` can change without changing what entity the value represents.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Synthesized `Equatable`/`Hashable`|Pure value objects where all stored fields define the value|Wrong for stable entities|
|Hand-written equality/hash by `id`|Persisted/domain entities, cache keys, set/dictionary keys|Does not detect content changes|
|Separate identity and snapshot types|You need both stable identity and content-diff semantics|More types, but cleaner contracts|

---

### Probe C — Synthesis fails when a stored property cannot conform

Given:

```swift
struct ScreenModel: Equatable {
    let title: String
    let action: () -> Void
}
```

### What happens?

Swift 6.2 compiler error:

```text
error: type 'ScreenModel' does not conform to protocol 'Equatable'
note: stored property type '() -> Void' does not conform to protocol 'Equatable',
preventing synthesized conformance of 'ScreenModel' to 'Equatable'
```

### Why?

A closure cannot be compared for value equality. The compiler cannot synthesize a meaningful `==`.

### Fix or redesign

If equality is identity-based:

```swift
struct ScreenModel: Equatable, Identifiable {
    let id: UUID
    let title: String
    let action: () -> Void

    static func == (lhs: ScreenModel, rhs: ScreenModel) -> Bool {
        lhs.id == rhs.id
    }
}
```

If equality should compare display state only:

```swift
struct ScreenModel: Equatable {
    let title: String
    let action: () -> Void

    static func == (lhs: ScreenModel, rhs: ScreenModel) -> Bool {
        lhs.title == rhs.title
    }
}
```

The second option is common in UI models, but be explicit: the closure is intentionally excluded.

---

### Probe D — `Sendable` checks stored state

Given:

```swift
final class MutableBox {
    var value = 0
}

struct Payload: Sendable {
    let box: MutableBox
}
```

### What happens?

Swift 6.2 compiler error:

```text
error: stored property 'box' of 'Sendable'-conforming struct 'Payload' has non-Sendable type 'MutableBox'
note: class 'MutableBox' does not conform to the 'Sendable' protocol
```

### Why?

`Payload` claims it is safe to transfer across concurrency domains, but it contains a mutable reference type with no synchronization guarantee.

### Fix or redesign

Prefer value semantics:

```swift
struct Payload: Sendable {
    let value: Int
}
```

Or actor isolation:

```swift
actor Box {
    private var value = 0

    func setValue(_ newValue: Int) {
        value = newValue
    }

    func currentValue() -> Int {
        value
    }
}
```

Use `@unchecked Sendable` only when the type internally enforces thread safety.

---

## 6. Exercise

### Problem

Given an entity type with stable identity and frequently changing fields, decide what equality should mean and implement it.

### Bad / naive version

```swift
struct UserProfile: Identifiable, Codable, Hashable, Sendable {
    let id: UUID
    var displayName: String
    var avatarURL: URL?
    var bio: String
    var followerCount: Int
    var lastSeenAt: Date?
}
```

### What is wrong?

```text
Synthesized Equatable/Hashable includes every stored property.

That means two values with the same id but different displayName, avatarURL, bio,
followerCount, or lastSeenAt are considered unequal.

For a stable entity, this often confuses:
- Set membership
- Dictionary keys
- cache identity
- selection state
- navigation identity
- de-duplication
```

### Improved version

```swift
import Foundation

struct UserProfile: Identifiable, Codable, Sendable {
    let id: UUID
    var displayName: String
    var avatarURL: URL?
    var bio: String
    var followerCount: Int
    var lastSeenAt: Date?
}

extension UserProfile: Equatable {
    static func == (lhs: UserProfile, rhs: UserProfile) -> Bool {
        lhs.id == rhs.id
    }
}

extension UserProfile: Hashable {
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}

extension UserProfile {
    struct ContentSnapshot: Equatable, Sendable {
        let displayName: String
        let avatarURL: URL?
        let bio: String
        let followerCount: Int
        let lastSeenAt: Date?
    }

    var contentSnapshot: ContentSnapshot {
        ContentSnapshot(
            displayName: displayName,
            avatarURL: avatarURL,
            bio: bio,
            followerCount: followerCount,
            lastSeenAt: lastSeenAt
        )
    }
}
```

### Why this is better

The type has two explicit semantics:

```text
UserProfile equality/hash:
same persisted user identity

UserProfile.ContentSnapshot equality:
same visible/content state
```

This prevents one conformance from being overloaded for two different jobs.

For example:

```swift
let id = UUID()

let old = UserProfile(
    id: id,
    displayName: "Aykut",
    avatarURL: nil,
    bio: "iOS Engineer",
    followerCount: 10,
    lastSeenAt: nil
)

let new = UserProfile(
    id: id,
    displayName: "Aykut",
    avatarURL: nil,
    bio: "Senior iOS Engineer",
    followerCount: 11,
    lastSeenAt: Date()
)

print(old == new)
print(old.contentSnapshot == new.contentSnapshot)
```

Expected output:

```text
true
false
```

That is exactly what you want if identity and content-diffing are separate concerns.

---

## 7. Production guidance

Use synthesized conformances in production when:

```text
The type is a pure value object.
Every stored property participates in the semantic value.
The encoded schema exactly matches the stored shape.
The enum is simple and all cases should be exposed.
The Sendable story follows directly from sendable stored values.
The conformance is internal and easy to change.
```

Be careful when:

```text
The type is a domain entity with stable identity.
Stored properties are mutable.
The type is used as a Set element or Dictionary key.
The model is persisted or sent over the network.
The type appears in public SDK API.
The type has actor/global-actor isolation.
The type contains reference storage, closures, caches, delegates, formatters, or dependencies.
```

Avoid synthesis when:

```text
Equality should ignore some fields.
Hashing should use a stable key.
The coding schema is versioned or externally owned.
The enum case order has product meaning.
Sendable requires manual proof through synchronization.
The synthesized behavior would become a misleading public contract.
```

Debugging checklist:

```text
Is this type a value object, entity, DTO, snapshot, or view model?
Does == mean same identity or same content?
Does hash(into:) use exactly the same semantic fields as ==?
Can any hashed field mutate while the value is used as a key?
Is Codable encoding a storage/network contract?
Are CodingKeys stable across app versions?
Is CaseIterable declaration order leaking into UI or persistence?
Does Sendable rely on value semantics, actor isolation, immutability, or unchecked synchronization?
Would adding/removing a stored property silently change conformance behavior?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift can synthesize `Equatable`, `Hashable`, `Codable`, `Sendable`, and `CaseIterable`, so you do not need to write boilerplate when the stored properties conform.

### Senior answer

> Synthesis is useful for pure value types, but it is structural. For entities, I decide whether equality means identity or full content. If the type is used in sets or dictionaries, I make sure `==` and `hash(into:)` are based on stable, consistent fields. For `Codable`, I check whether the encoded shape is a real schema contract. For `Sendable`, I check whether stored state is safe to cross isolation boundaries.

### Staff-level answer

> I treat conformances as API contracts. Synthesis is great when the structural shape is the semantic contract, but dangerous when it accidentally exposes implementation detail. For public libraries and app architecture, I separate identity, content snapshots, persistence schemas, and concurrency transferability. I also consider migration cost: adding a stored property can silently change synthesized equality, hashing, coding, and sendability. In a large codebase, I prefer explicit conformances around domain entities and synthesized conformances around small immutable value objects and DTOs.

Staff-level questions to ask:

```text
Is this type a value object or an entity?
Will clients rely on this conformance across module boundaries?
Could adding a stored property silently change behavior?
Is equality being used for identity, UI diffing, testing, or cache invalidation?
Is this Codable shape persisted, versioned, or backend-owned?
Does Sendable reflect real safety or just silence diagnostics?
Should identity and content comparison be separate APIs?
```

---

## 9. Interview-ready summary

Synthesized conformances remove boilerplate, but they are structural. Swift can derive `Equatable` and `Hashable` from stored properties or enum payloads, `Codable` from codable stored state and coding keys, `CaseIterable` for simple finite enums, and `Sendable` when stored state is safe to transfer across concurrency domains. The important judgment is whether the generated behavior matches the type’s semantic contract. For pure values, synthesis is often ideal. For stable entities, persisted schemas, public APIs, mutable keys, and concurrency-sensitive models, I usually make the contract explicit.

---

## 10. Flashcards

Q: What is the main risk of synthesized `Equatable`?

A: It compares structural stored state, which may not match domain identity.

Q: When is synthesized `Equatable` ideal?

A: For pure value objects where every stored property defines the value.

Q: When should `Hashable` be hand-written?

A: When the hash key should be a stable semantic subset, usually an entity `id`.

Q: What invariant must `Hashable` preserve?

A: If `a == b`, then `a` and `b` must feed equivalent data into `hash(into:)`.

Q: Why can synthesized `Codable` be dangerous for persisted data?

A: Renaming, adding, deleting, or changing stored properties can change the expected encoded schema.

Q: What does `CaseIterable` synthesis provide?

A: A static `allCases` collection for simple finite enums, typically in declaration order.

Q: What does `Sendable` communicate?

A: A value can be safely transferred across concurrency domains.

Q: Does `Sendable` mean all operations are atomic?

A: No. It is about safe transfer across isolation boundaries, not full logical synchronization.

Q: Why can adding a stored property be a breaking semantic change?

A: It can silently affect synthesized equality, hashing, coding, and sendability.

Q: What is a strong pattern for entities that need content diffing?

A: Use identity-based `Equatable`/`Hashable` on the entity and expose a separate `ContentSnapshot: Equatable`.

---

## 11. Related sections

- [[B8 — Equatable, Hashable, and collection correctness]]
- [[B9 — Codable and schema evolution]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — B7 rubric snapshot.
- Swift.org. "Protocols." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/
- Apple Developer. "Hashable." Apple Developer Documentation. https://developer.apple.com/documentation/swift/hashable
- Apple Developer. "Sendable." Apple Developer Documentation. https://developer.apple.com/documentation/swift/sendable
- Apple Developer. "CaseIterable." Apple Developer Documentation. https://developer.apple.com/documentation/swift/caseiterable
- GitHub. "Synthesizing Equatable and Hashable Conformance." Swift Evolution SE-0185. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0185-synthesize-equatable-hashable.md
- GitHub. "Swift Archival & Serialization." Swift Evolution SE-0166. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md
- GitHub. "Derived Collection of Enum Cases." Swift Evolution SE-0194. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0194-derived-collection-of-enum-cases.md
- GitHub. "Sendable and @Sendable Closures." Swift Evolution SE-0302. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md
- GitHub. "Isolated Conformances." Swift Evolution SE-0470. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md
