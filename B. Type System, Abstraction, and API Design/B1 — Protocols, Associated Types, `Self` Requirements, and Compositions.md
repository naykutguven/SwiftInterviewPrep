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
## 0. Rubric snapshot

**Rubric expectation**

Know when protocols model behavior well, when associated types are necessary, how protocol compositions express capability sets, and why protocols with `Self` or associated types behave differently from simple existential-friendly protocols.

**Caveats**

Over-protocolization can produce unusable APIs and erase important type relationships. Marker protocols should communicate real semantic guarantees, not just empty taxonomy labels.

**You should be able to answer**

- Why can’t every protocol be used directly as a type?
- When should a protocol have an associated type instead of using an existential payload?
- When is a protocol composition like `P & Q` better than creating a new umbrella protocol, and when is a marker protocol justified?

**You should be able to do**

- Design a `Cache` abstraction and decide whether it should use associated types, generics, existentials, or type erasure.

---

## 1. Core mental model

A protocol is not just an “interface” in the Java/C# sense. In Swift, a protocol can be used in multiple roles:

```text
protocol as constraint  -> compile-time abstraction
protocol as any P       -> runtime boxed/type-erased abstraction
protocol as P & Q       -> capability composition
```

The most important distinction is **type-level abstraction vs value-level abstraction**.

A generic constraint preserves the concrete conforming type:

```swift
func use<C: Cache>(_ cache: C) { ... }
```

Inside that function, the compiler still knows there is one specific `C`. That means it can preserve relationships like:

```text
C.Key is the same everywhere
C.Value is the same everywhere
```

An existential value erases the concrete type:

```swift
let cache: any Cache = ...
```

This says: “I have some value whose concrete type conforms to `Cache`, but I am intentionally hiding which one.” That is useful at runtime boundaries, but it loses some static relationships. SE-0335 introduced explicit `any` syntax to make existential types visible because plain protocol names in type position made existential costs and limitations too easy to miss. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md?utm_source=chatgpt.com "swift-evolution/proposals/0335-existential-any.md at main"))

Associated types let a protocol describe a family of types where part of the API is chosen by the conforming type. The Swift book describes an associated type as a placeholder for a type used by a protocol, where the actual type is specified when the protocol is adopted. ([Swift Dokümantasyonu](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/?utm_source=chatgpt.com "Generics | Documentation - Swift Programming Language"))

The key idea:

```text
Use protocols to model capability; use associated types to preserve type relationships; use any P only when type erasure is intentional.
```

---

## 2. Essential mechanics

### 2.1 Simple existential-friendly protocols

A protocol with no associated-type-sensitive or `Self`-sensitive requirements can usually be used comfortably as an existential.

```swift
protocol ImageLoading {
    func loadImage() async throws -> Data
}

func display(loader: any ImageLoading) async throws {
    let data = try await loader.loadImage()
    print(data.count)
}
```

This works because the method does not depend on an unknown associated type or require another value of the exact same conforming type.

```text
any ImageLoading
    -> can hold LocalImageLoader
    -> can hold RemoteImageLoader
    -> caller only needs loadImage() -> Data
```

Use this shape when the protocol truly describes a stable runtime capability.

---

### 2.2 Associated types preserve relationships

Associated types are for APIs where the conforming type decides part of the API surface.

```swift
protocol Parser {
    associatedtype Output

    func parse(_ input: String) -> Output?
}

struct IntParser: Parser {
    func parse(_ input: String) -> Int? {
        Int(input)
    }
}

struct URLParser: Parser {
    func parse(_ input: String) -> URL? {
        URL(string: input)
    }
}
```

The protocol does not say every parser returns `Any`. It says each parser has a specific `Output`.

That difference matters:

```swift
func parse<P: Parser>(_ input: String, using parser: P) -> P.Output? {
    parser.parse(input)
}

let number = parse("42", using: IntParser())
// number is Int?
```

The compiler preserves the relationship:

```text
P == IntParser
P.Output == Int
parse(...) -> Int?
```

If you had used `Any`, every caller would need downcasts and runtime checks.

---

### 2.3 `Self` requirements mean “the concrete conforming type”

`Self` inside a protocol means the eventual conforming type.

```swift
protocol Combinable {
    func combined(with other: Self) -> Self
}

struct Amount: Combinable {
    var value: Int

    func combined(with other: Amount) -> Amount {
        Amount(value: value + other.value)
    }
}
```

This says:

```text
Amount can combine with Amount.
Vector can combine with Vector.
Amount is not required to combine with Vector.
```

That is a strong static guarantee. But it is also why existential use becomes limited:

```swift
let a: any Combinable = Amount(value: 1)
let b: any Combinable = Amount(value: 2)

let c = a.combined(with: b)
```

Compiler error with Swift 6.2.1:

```text
error: member 'combined' cannot be used on value of type 'any Combinable'; consider using a generic constraint instead [#ExistentialMemberAccess]
```

Why? Because `a` and `b` are both “some `Combinable`”, but the compiler cannot prove they are the same concrete type after erasure. Swift’s compiler documentation explicitly calls out member-access limitations for protocol members that reference `Self` or `Self`-rooted associated types. ([Swift Dokümantasyonu](https://docs.swift.org/compiler/documentation/diagnostics/existential-member-access-limitations/?utm_source=chatgpt.com "Using protocol members with references to `Self` or `Self`"))

Use a generic instead:

```swift
func combine<T: Combinable>(_ lhs: T, _ rhs: T) -> T {
    lhs.combined(with: rhs)
}

let result = combine(Amount(value: 1), Amount(value: 2))
```

---

### 2.4 Protocol compositions express local capability sets

A protocol composition means “a type that conforms to all of these protocols.” The Swift reference describes protocol composition types as types that conform to each protocol in a specified list. ([Swift Dokümantasyonu](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/types/?utm_source=chatgpt.com "Types | Documentation - Swift Programming Language"))

```swift
protocol ImageLoading {
    func loadImage() async throws -> Data
}

protocol Cancellable {
    func cancel()
}

func startRequest(_ request: any ImageLoading & Cancellable) async throws {
    defer { request.cancel() }
    _ = try await request.loadImage()
}
```

This is better than inventing a new protocol when the combination is local and mechanical.

Bad:

```swift
protocol ImageLoadingAndCancellable: ImageLoading, Cancellable {}
```

Better:

```swift
func startRequest(_ request: any ImageLoading & Cancellable) async throws {
    defer { request.cancel() }
    _ = try await request.loadImage()
}
```

Create a named protocol only when the combination has a domain meaning.

```swift
protocol ImageRequest: ImageLoading, Cancellable {
    var requestID: UUID { get }
}
```

Now `ImageRequest` is not just “P plus Q”; it names a concept in the system.

---

## 3. Common traps and misconceptions

### Trap 1: Treating protocols as default architecture

Bad:

```swift
protocol UserProtocol {
    var id: String { get }
    var name: String { get }
}

struct User: UserProtocol {
    let id: String
    let name: String
}
```

This protocol adds no useful abstraction. It just creates indirection.

Better:

```swift
struct User: Equatable, Identifiable, Sendable {
    let id: String
    let name: String
}
```

Protocols are useful when you need substitutability, generic constraints, mocking at a boundary, plugin behavior, or a semantic contract shared by multiple conforming types.

They are not useful just because “interfaces are clean.”

---

### Trap 2: Replacing associated types with `Any`

Bad:

```swift
protocol AnyParser {
    func parse(_ input: String) -> Any?
}

let parser: AnyParser = ...
let value = parser.parse("42") as? Int
```

This loses the output relationship. The compiler cannot help you anymore.

Better:

```swift
protocol Parser {
    associatedtype Output

    func parse(_ input: String) -> Output?
}

func parse<P: Parser>(_ input: String, using parser: P) -> P.Output? {
    parser.parse(input)
}
```

Use `Any` only when heterogeneity is a real requirement and the downcast boundary is deliberate.

---

### Trap 3: Using existentials where generics are required

Bad:

```swift
protocol Cache {
    associatedtype Key: Hashable
    associatedtype Value

    func get(_ key: Key) -> Value?
}

func read(_ cache: any Cache) {
    _ = cache.get("user:1")
}
```

Compiler error:

```text
error: member 'get' cannot be used on value of type 'any Cache'; consider using a generic constraint instead [#ExistentialMemberAccess]
```

Better:

```swift
func read<C: Cache>(_ cache: C, key: C.Key) -> C.Value? {
    cache.get(key)
}
```

The generic form preserves:

```text
C.Key
C.Value
```

The existential form erases them.

---

### Trap 4: Empty marker protocols as taxonomy

Bad:

```swift
protocol AppModel {}

struct User: AppModel {}
struct Order: AppModel {}
struct FeatureFlag: AppModel {}
```

This does not communicate behavior or a semantic guarantee.

Better:

```swift
protocol PersistableModel: Codable, Sendable {
    static var storageKey: String { get }
}
```

Now the marker-ish protocol has real meaning: persistence compatibility, sendability, and a storage identity.

A marker protocol is justified when it communicates a meaningful guarantee that other code relies on. `Sendable` is the canonical Swift example: it is marker-like, but it expresses a concurrency-safety contract enforced or checked by the compiler.

---

## 4. Direct answers to rubric questions

### Q1. Why can’t every protocol be used directly as a type?

Because using a protocol as a type creates an existential value, and an existential erases the concrete conforming type. If protocol requirements depend on `Self` or associated types, some members need type information that has been erased.

Modern Swift nuance: many protocols with associated types can now be written as existentials using `any P`, especially after the language work to unlock existentials. But forming the existential is not the same as being able to use every member safely. SE-0309 generalized value-level abstraction for protocols, while compiler diagnostics still restrict member access when the erased type relationship matters. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md?utm_source=chatgpt.com "0309-unlock-existential-types-for-all-protocols.md"))

Example:

```swift
protocol Cache {
    associatedtype Key: Hashable
    associatedtype Value

    func get(_ key: Key) -> Value?
}

let cache: any Cache = ...
```

This can store some cache, but the caller does not know:

```text
What is Key?
What is Value?
```

So this is not safe:

```swift
cache.get("user:1")
```

The erased cache might be:

```text
DiskCache<URL, Data>
MemoryCache<Int, UIImage>
UserCache<String, User>
```

Interview version:

> A protocol value like `any P` is a boxed value whose concrete conforming type is hidden. That works well when the protocol’s requirements do not depend on hidden type relationships. But if the protocol uses associated types or `Self`, the caller often needs information that was erased. In those cases I usually use a generic constraint, an opaque type, constrained existential syntax, or a deliberate type-erased wrapper.

---

### Q2. When should a protocol have an associated type instead of using an existential payload?

Use an associated type when the protocol has a stable type relationship that callers should not lose.

Good associated-type cases:

```text
Parser.Output
Cache.Key / Cache.Value
Repository.Entity
ImageDecoder.OutputImage
Serializer.Model
```

Example:

```swift
protocol Serializer {
    associatedtype Model

    func encode(_ model: Model) throws -> Data
    func decode(_ data: Data) throws -> Model
}
```

This says encode and decode operate on the same `Model`.

Bad existential-payload version:

```swift
protocol AnySerializer {
    func encode(_ model: Any) throws -> Data
    func decode(_ data: Data) throws -> Any
}
```

This loses the relationship. A serializer could accept `User` and return `Order`, and the compiler would not know.

Use existential payloads only when the model is genuinely heterogeneous:

```swift
protocol AnalyticsValue {
    var analyticsDescription: String { get }
}

struct AnalyticsEvent {
    var properties: [String: any AnalyticsValue]
}
```

Here heterogeneity is the point.

Interview version:

> I use an associated type when the conforming type chooses a type that must stay consistent across the protocol’s API. A parser’s output, a cache’s key and value, or a repository’s entity are not arbitrary `Any` payloads; they are part of the abstraction’s contract. I use an existential payload only at a deliberate heterogeneity boundary, usually close to serialization, analytics, plugin systems, or UI composition.

---

### Q3. When is `P & Q` better than creating a new umbrella protocol, and when is a marker protocol justified?

Use `P & Q` when the combination is local and does not deserve a new domain name.

```swift
func configure(_ object: any ObservableObject & Identifiable) {
    print(object.id)
}
```

Creating this is usually noise:

```swift
protocol IdentifiableObservableObject: ObservableObject, Identifiable {}
```

Use a named umbrella protocol when the combined capabilities represent a real domain concept, appear repeatedly, or need extra requirements/default behavior.

```swift
protocol UploadTask: Cancellable, ProgressReporting, Sendable {
    var id: UUID { get }
}
```

Use marker protocols rarely. A marker protocol is justified only when conformance means something that code can trust.

Reasonable:

```swift
protocol RedactableLogValue: Sendable {}
```

This can mean: “Safe to include in privacy-filtered logs.”

Weak:

```swift
protocol AppThing {}
```

That communicates nothing.

Interview version:

> I prefer protocol composition for local capability requirements because it avoids naming fake concepts. I create an umbrella protocol when the composition has domain meaning or needs additional requirements. Marker protocols are justified only when they encode a real semantic guarantee, not when they are just empty taxonomy.

---

## 5. Mini code probes

The rubric does not include a B1 code probe, so use these instead.

### Probe 1: Generic constraint preserves the associated type

Given:

```swift
protocol Parser {
    associatedtype Output

    func parse(_ input: String) -> Output?
}

struct IntParser: Parser {
    func parse(_ input: String) -> Int? {
        Int(input)
    }
}

func parse<P: Parser>(_ input: String, using parser: P) -> P.Output? {
    parser.parse(input)
}

print(parse("42", using: IntParser()) as Any)
```

Output with Swift 6.2.1:

```text
Optional(42)
```

Why:

```text
P == IntParser
P.Output == Int
parse(...) -> Int?
```

The generic function preserves the relationship between the parser and its output.

---

### Probe 2: Existential erases associated-type information

Given:

```swift
protocol Cache {
    associatedtype Key: Hashable
    associatedtype Value

    func get(_ key: Key) -> Value?
}

struct StringIntCache: Cache {
    func get(_ key: String) -> Int? {
        42
    }
}

let cache: any Cache = StringIntCache()
let value = cache.get("user:1")
print(value as Any)
```

Compiler error with Swift 6.2.1:

```text
error: member 'get' cannot be used on value of type 'any Cache'; consider using a generic constraint instead [#ExistentialMemberAccess]
```

Why:

```text
any Cache
    hides concrete type
    hides Key
    hides Value

cache.get("user:1")
    requires caller to know Key == String
    but existential erased that fact
```

Fix with a generic:

```swift
func read<C: Cache>(_ cache: C, key: C.Key) -> C.Value? {
    cache.get(key)
}

let cache = StringIntCache()
print(read(cache, key: "user:1") as Any)
```

Output:

```text
Optional(42)
```

---

### Probe 3: Constrained existential with primary associated types

Modern Swift lets protocols expose primary associated types:

```swift
protocol Cache<Key, Value> {
    associatedtype Key: Hashable
    associatedtype Value

    func get(_ key: Key) -> Value?
}

struct StringIntCache: Cache {
    func get(_ key: String) -> Int? {
        42
    }
}

let cache: any Cache<String, Int> = StringIntCache()
let value = cache.get("user:1")

print(value as Any)
```

Output with Swift 6.2.1:

```text
Optional(42)
```

Why this works:

```text
any Cache<String, Int>
    still erases concrete cache type
    but preserves Key == String
    and Value == Int
```

This is useful at storage/API boundaries where the concrete implementation can vary, but the key/value relationship must stay known.

SE-0346 introduced lightweight same-type syntax for constraining associated types, which is the language direction behind spelling like `Collection<Element>` or custom protocols with primary associated types. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md?utm_source=chatgpt.com "0346-light-weight-same-type-syntax.md"))

---

## 6. Exercise

### Problem

Design a `Cache` abstraction and decide whether it should use associated types, generics, or type erasure.

---

### Bad / naive version

```swift
protocol Cache {
    func value(forKey key: AnyHashable) async -> Any?
    func insert(_ value: Any, forKey key: AnyHashable) async
}

struct UserRepository {
    let cache: any Cache

    func user(id: String) async -> User? {
        await cache.value(forKey: id) as? User
    }

    func save(_ user: User) async {
        await cache.insert(user, forKey: user.id)
    }
}

struct User: Identifiable, Sendable {
    let id: String
    let name: String
}
```

### What is wrong?

```text
The key type is erased.
The value type is erased.
Every read needs a downcast.
Wrong values can be inserted under valid keys.
The compiler cannot prove UserRepository uses a User cache.
The abstraction hides bugs until runtime.
```

This compiles, but it allows nonsense:

```swift
await cache.insert(UIImage(), forKey: "user-123")
await cache.insert(User(id: "1", name: "A"), forKey: 42)
```

The API says “cache,” but it does not preserve the cache’s key/value contract.

---

### Improved version: associated types

```swift
protocol Cache<Key, Value>: Sendable {
    associatedtype Key: Hashable & Sendable
    associatedtype Value: Sendable

    func value(for key: Key) async -> Value?
    func insert(_ value: Value, for key: Key) async
}
```

This protocol says:

```text
A cache has one Key type.
A cache has one Value type.
Reads and writes must use those same types.
```

---

### Concrete implementation

```swift
actor InMemoryCache<Key: Hashable & Sendable, Value: Sendable>: Cache {
    private var storage: [Key: Value] = [:]

    func value(for key: Key) -> Value? {
        storage[key]
    }

    func insert(_ value: Value, for key: Key) {
        storage[key] = value
    }
}
```

Notes:

```text
actor
    protects shared mutable storage

Key: Hashable
    required for Dictionary

Key: Sendable, Value: Sendable
    appropriate because async actor calls cross isolation boundaries
```

---

### Generic consumer: best when the concrete cache type should be preserved

```swift
struct User: Identifiable, Sendable {
    let id: String
    let name: String
}

struct UserRepository<C: Cache>: Sendable where C.Key == String, C.Value == User {
    private let cache: C

    init(cache: C) {
        self.cache = cache
    }

    func user(id: String) async -> User? {
        await cache.value(for: id)
    }

    func save(_ user: User) async {
        await cache.insert(user, for: user.id)
    }
}
```

Why this is strong:

```text
UserRepository cannot accidentally receive an Image cache.
No downcasts.
No Any.
The compiler preserves C.Key == String and C.Value == User.
The concrete cache type remains available for optimization.
```

Use this inside modules and hot paths.

---

### Type-erased wrapper: best at storage or module boundaries

Sometimes you do not want `UserRepository` to be generic. For example:

```text
You want to store repositories in collections.
You want a stable public API.
You want to hide the concrete cache implementation.
You want runtime swapping between memory, disk, and test caches.
```

Use type erasure deliberately:

```swift
struct AnyCache<Key: Hashable & Sendable, Value: Sendable>: Cache {
    private let _value: @Sendable (Key) async -> Value?
    private let _insert: @Sendable (Value, Key) async -> Void

    init<C: Cache>(_ cache: C) where C.Key == Key, C.Value == Value {
        self._value = { key in
            await cache.value(for: key)
        }

        self._insert = { value, key in
            await cache.insert(value, for: key)
        }
    }

    func value(for key: Key) async -> Value? {
        await _value(key)
    }

    func insert(_ value: Value, for key: Key) async {
        await _insert(value, key)
    }
}
```

Now the repository can be non-generic while preserving key/value types:

```swift
struct ErasedUserRepository: Sendable {
    private let cache: AnyCache<String, User>

    init(cache: AnyCache<String, User>) {
        self.cache = cache
    }

    func user(id: String) async -> User? {
        await cache.value(for: id)
    }

    func save(_ user: User) async {
        await cache.insert(user, for: user.id)
    }
}
```

---

### Existential option: useful but still erased

With primary associated types, this is also possible:

```swift
struct ExistentialUserRepository {
    private let cache: any Cache<String, User>

    init(cache: any Cache<String, User>) {
        self.cache = cache
    }

    func user(id: String) async -> User? {
        await cache.value(for: id)
    }

    func save(_ user: User) async {
        await cache.insert(user, for: user.id)
    }
}
```

This preserves:

```text
Key == String
Value == User
```

But erases:

```text
Concrete cache type
Specialization opportunities
Implementation-specific API
```

Use this when runtime flexibility matters more than preserving the concrete type.

---

### Decision table

|Design|Use when|Tradeoff|
|---|---|---|
|`protocol Cache<Key, Value>` with associated types|The abstraction has stable key/value relationships|Generic consumers may become more complex|
|Generic consumer `UserRepository<C: Cache>`|Internal code, hot paths, compile-time guarantees|Concrete type leaks into the consumer’s type|
|`any Cache<String, User>`|You need runtime polymorphism and constrained associated types are enough|Existential dispatch/type erasure|
|`AnyCache<String, User>`|You want a stable non-generic stored property or public API|Boilerplate and closure indirection|
|`AnyHashable` / `Any` cache|Truly heterogeneous cache or serialization boundary|Downcasts, runtime bugs, weaker API|

---

## 7. Production guidance

Use protocols in production when:

```text
Multiple real conforming types exist or are expected.
The abstraction captures behavior, not just data shape.
You need a test boundary around I/O, networking, persistence, or system APIs.
You need generic algorithms over related types.
You need to model semantic capability, for example Cancellable, Retriable, Persistable.
```

Be careful when:

```text
A protocol has only one conforming type.
The protocol just mirrors a struct’s stored properties.
You are adding associated types but all callers immediately erase them.
You need arrays of heterogeneous values.
A public protocol might freeze an API design too early.
The protocol is used to avoid deciding ownership, isolation, or lifetime.
```

Avoid when:

```text
A concrete type is clearer.
A closure parameter would be enough.
An enum models the finite set of variants better.
A generic function is enough and no named protocol is needed.
The protocol is an empty marker with no semantic contract.
```

Debugging checklist:

```text
Am I using a protocol as a generic constraint or as an existential value?
Did I accidentally erase an associated type that the caller needs?
Would a same-type constraint make the API safer?
Would primary associated types make the existential usable?
Is this protocol modeling behavior or just creating indirection?
Does a marker protocol communicate a real guarantee?
Is a protocol composition enough, or is there a real named domain concept?
Is type erasure at the correct boundary, or did it happen too early?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Protocols define methods that types implement. Associated types are like generics for protocols. You use them when the protocol needs a placeholder type.

### Senior answer

> Protocols can be used either as generic constraints or existential values, and those are different tools. Associated types preserve relationships like a parser’s output or a cache’s value type. If I erase the protocol into `any P`, I may lose access to members that depend on `Self` or associated types, so I choose between generics, constrained existentials, opaque types, and type erasure based on the API boundary.

### Staff-level answer

> I treat protocols as semantic contracts, not default architecture. For internal code, I prefer generics when preserving type relationships matters, because they keep the compiler involved and avoid unnecessary erasure. At module boundaries, I may use constrained existentials or type-erased wrappers to stabilize API shape and hide implementation detail. I avoid marker protocols unless they encode a real guarantee, and I prefer protocol composition over fake umbrella protocols unless the combination names a domain concept. For public SDKs, I’m especially cautious because protocols, associated types, and type-erased wrappers become long-term compatibility commitments.

Staff-level questions to ask:

```text
Is this protocol a semantic contract or just indirection?
Do callers need the concrete type relationship?
Should this be generic, existential, opaque, or type-erased?
Where is the correct erasure boundary?
Will this public protocol still be evolvable in two years?
Does this marker protocol encode a guarantee that code can rely on?
Would an enum or concrete type model this better?
Does the abstraction preserve Sendable/isolation requirements?
```

---

## 9. Interview-ready summary

Protocols in Swift are powerful because they can model both behavior and type relationships. A simple protocol can often be used as `any P`, but protocols with associated types or `Self` requirements describe relationships tied to the concrete conforming type. If I erase that concrete type too early, I lose guarantees and may not be able to call important members. My default is to use generics when the caller needs type relationships, constrained existentials or type erasure at runtime boundaries, and protocol composition for local capability sets. I avoid protocols that only mirror data or create taxonomy without a semantic contract.

---

## 10. Flashcards

Q: What is the difference between `T: P` and `any P`?

A: `T: P` is a generic constraint that preserves the concrete type. `any P` is an existential value that erases the concrete type.

Q: Why do associated types often make existential use harder?

A: Because the associated type is chosen by the concrete conforming type. Once the type is erased, the caller may no longer know which associated type is valid.

Q: When should a protocol use an associated type?

A: When the protocol needs to preserve a type relationship, such as `Parser.Output`, `Cache.Key`, `Cache.Value`, or `Repository.Entity`.

Q: What does `Self` mean inside a protocol?

A: It refers to the concrete conforming type, not the protocol existential.

Q: Why is `func combined(with other: Self) -> Self` hard to call on `any Combinable`?

A: Because two `any Combinable` values might hide different concrete types. The compiler cannot prove both sides have the same `Self`.

Q: When is `P & Q` better than an umbrella protocol?

A: When the combination is local and mechanical, and does not represent a reusable domain concept.

Q: When is an umbrella protocol justified?

A: When the combination has a meaningful name, appears repeatedly, or needs additional requirements/default behavior.

Q: When is a marker protocol justified?

A: When conformance communicates a real semantic guarantee that code can rely on.

Q: What is the main problem with `Any`-based protocol APIs?

A: They erase useful type information and move correctness from compile time to runtime downcasts.

Q: What is type erasure for?

A: Hiding concrete implementation types while preserving a useful public or stored API shape.

---

## 11. Related sections

- [[B2 — Existentials (any) vs opaque types (some)]]
- [[B3 — Generics, constraints, and same-type relationships]]
- [[B4 — Dispatch model: static, class virtual, witness-table, and protocol extension dispatch]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[D6 — `Sendable` and `@Sendable`]]

---

## 12. Sources

- Swift Senior/Staff Rubric — B1 expectations, caveats, questions, and exercise.
- Swift.org, _The Swift Programming Language_ — Protocols. ([Swift Dokümantasyonu](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/?utm_source=chatgpt.com "Protocols - Documentation | Swift.org"))
- Swift.org, _The Swift Programming Language_ — Generics / Associated Types. ([Swift Dokümantasyonu](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/?utm_source=chatgpt.com "Generics | Documentation - Swift Programming Language"))
- Swift.org, _The Swift Programming Language_ — Protocol Composition Types. ([Swift Dokümantasyonu](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/types/?utm_source=chatgpt.com "Types | Documentation - Swift Programming Language"))
- Swift compiler diagnostics — existential `any`. ([Swift Dokümantasyonu](https://docs.swift.org/compiler/documentation/diagnostics/existential-any/?utm_source=chatgpt.com "Existential any (ExistentialAny) | Documentation"))
- Swift compiler diagnostics — existential member access limitations for `Self` and associated types. ([Swift Dokümantasyonu](https://docs.swift.org/compiler/documentation/diagnostics/existential-member-access-limitations/?utm_source=chatgpt.com "Using protocol members with references to `Self` or `Self`"))
- Swift Evolution SE-0309 — unlock existential types for all protocols. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md?utm_source=chatgpt.com "0309-unlock-existential-types-for-all-protocols.md"))
- Swift Evolution SE-0335 — introduce existential `any`. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md?utm_source=chatgpt.com "swift-evolution/proposals/0335-existential-any.md at main"))
- Swift Evolution SE-0346 — lightweight same-type requirements for primary associated types. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md?utm_source=chatgpt.com "0346-light-weight-same-type-syntax.md"))