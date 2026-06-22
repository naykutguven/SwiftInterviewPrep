---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
---
## 0. Rubric snapshot

**Rubric expectation**

Understand naming, argument labels, mutating vs nonmutating APIs, fluent vs explicit style, and minimizing surface area.

**Caveats**

A technically correct API can still be un-Swifty, ambiguous at the call site, and expensive to maintain.

**You should be able to answer**

- Why are naming and labels part of correctness in Swift API design?
- What signs show an API was ported mechanically from another language without adapting to Swift idioms?

**You should be able to do**

- Review a small SDK API and rewrite three signatures to better fit Swift API Design Guidelines.

---

## 1. Core mental model

Swift API design is about **call-site semantics**. A declaration is written once, but every consumer reads the call site repeatedly. The official guidelines make this explicit: clarity at the point of use is the most important goal, and clarity is more important than brevity. ([Swift.org, "API Design Guidelines"](https://swift.org/documentation/api-design-guidelines/))

A Swift-native API should read like a small phrase that explains what is happening:

```swift
cache.removeValue(forKey: userID)
image.resized(to: targetSize)
try await client.fetchUser(id: userID)
```

The name, base name, argument labels, mutability, return type, and error model together form the API contract. In Swift, labels are not decoration. They disambiguate roles, compensate for weak type information, and prevent accidental misuse.

The key idea:

```text
A Swift API is judged primarily by the clarity and correctness of its use site, not by how compact its declaration looks.
```

Swift does **not** guarantee that a legal signature is a good API. This compiles:

```swift
func perform(_ a: String, _ b: Bool, _ c: Int)
```

But the call site is useless:

```swift
perform("abc", true, 3)
```

A good Swift API makes misuse harder:

```swift
func upload(
    fileNamed name: String,
    allowsCellularAccess: Bool,
    retryLimit: Int
)
```

---

## 2. Essential mechanics

### 2.1 Optimize for the use site

The guideline is not “make declarations pretty.” It is “make calls clear in context.” The official guidelines say reading a declaration is seldom sufficient; you should examine actual use cases. ([Swift.org, "API Design Guidelines"](https://swift.org/documentation/api-design-guidelines/))

Bad:

```swift
func resize(_ image: UIImage, _ size: CGSize, _ mode: Int) -> UIImage

resize(image, size, 1)
```

Better:

```swift
enum ImageResizeMode {
    case aspectFit
    case aspectFill
    case stretch
}

func resizedImage(
    from image: UIImage,
    to size: CGSize,
    mode: ImageResizeMode
) -> UIImage

resizedImage(from: image, to: size, mode: .aspectFit)
```

Better as a method when there is a natural receiver:

```swift
extension UIImage {
    func resized(to size: CGSize, mode: ImageResizeMode) -> UIImage {
        // implementation
    }
}

image.resized(to: size, mode: .aspectFit)
```

The receiver `image` now carries part of the meaning. The method name and labels carry the rest.

---

### 2.2 Labels express parameter roles, not just types

The guidelines say variables, parameters, and associated types should be named according to their roles rather than their type constraints. They also say weakly typed parameters such as `String`, `Int`, `Any`, `AnyObject`, or `NSObject` often need extra clarity. ([Swift.org, "API Design Guidelines"](https://swift.org/documentation/api-design-guidelines/))

Bad:

```swift
func track(_ string: String, _ string2: String, _ int: Int)
```

Call site:

```swift
track("checkout", "button", 3)
```

Better:

```swift
func trackEvent(
    named eventName: String,
    source: AnalyticsSource,
    retryLimit: Int
)
```

Call site:

```swift
trackEvent(named: "checkout", source: .button, retryLimit: 3)
```

Even better if the domain has stronger types:

```swift
struct EventName: RawRepresentable, Hashable {
    let rawValue: String
}

enum AnalyticsSource {
    case button
    case deepLink
    case pushNotification
}

func trackEvent(
    named eventName: EventName,
    source: AnalyticsSource,
    retryLimit: Int
)
```

The senior-level move is not just renaming parameters. It is replacing primitive obsession with domain types when labels alone cannot protect correctness.

---

### 2.3 Mutating and nonmutating APIs should be named consistently

The Swift guidelines distinguish side-effecting APIs from value-returning APIs. Side-effecting methods should read as imperative verb phrases, while methods without side effects should generally read as noun phrases. For mutating/nonmutating pairs, Swift convention uses patterns like `sort()` / `sorted()` and `formUnion(_:)` / `union(_:)`. ([Swift.org, "API Design Guidelines"](https://swift.org/documentation/api-design-guidelines/))

Bad:

```swift
extension Array where Element: Comparable {
    mutating func sortedInPlace() {
        sort()
    }

    func sortCopy() -> [Element] {
        sorted()
    }
}
```

Better:

```swift
var values = [3, 1, 2]

values.sort()          // mutates in place
let copy = values.sorted() // returns a new value
```

For noun-like operations:

```swift
let combined = a.union(b) // nonmutating
a.formUnion(b)            // mutating
```

This matters because the name communicates ownership and side effects before the reader inspects the implementation.

---

### 2.4 Prefer methods/properties when there is a natural receiver

The guidelines prefer methods and properties over free functions unless there is no obvious `self`, the function is unconstrained generic, or function syntax is established domain notation. ([Swift.org, "API Design Guidelines"](https://swift.org/documentation/api-design-guidelines/))

Bad:

```swift
func validateEmail(_ email: String) -> Bool
func normalizeURL(_ url: URL) -> URL
```

Better:

```swift
extension EmailAddress {
    var isValid: Bool { /* ... */ }
}

extension URL {
    var normalizedForCaching: URL { /* ... */ }
}
```

But do not force everything into extensions. If the operation belongs to a service boundary, a method on a service can be clearer:

```swift
protocol UserFetching {
    func user(id: User.ID) async throws -> User
}
```

Not every operation on `URL`, `String`, or `Data` should become an extension. Staff-level judgment is knowing where the semantic owner lives.

---

### 2.5 Minimize public surface area

API design is also subtraction. Every public type, overload, default value, behavior, and naming convention becomes something clients learn, depend on, test against, and complain about when changed.

Bad:

```swift
public final class AnalyticsSDK {
    public func track(_ event: String)
    public func track(_ event: String, params: [String: Any])
    public func sendEvent(_ event: String)
    public func logEvent(_ event: String)
    public func uploadEvent(_ event: String)
}
```

Better:

```swift
public protocol AnalyticsTracking: Sendable {
    func track(_ event: AnalyticsEvent) async
}

public struct AnalyticsEvent: Sendable, Hashable {
    public let name: Name
    public let properties: [PropertyKey: PropertyValue]
}
```

One clear abstraction is better than five synonyms.

---

## 3. Common traps and misconceptions

### Trap 1: Treating labels as style instead of semantics

Bad:

```swift
func schedule(_ date: Date, _ date2: Date, _ bool: Bool)
```

This is not just ugly. It is unsafe because the call site does not reveal which date is start, which is end, or what the boolean means.

Better:

```swift
func scheduleMeeting(
    from startDate: Date,
    to endDate: Date,
    sendsReminder: Bool
)
```

Even better:

```swift
struct MeetingReminderPolicy: Sendable, Hashable {
    var sendsReminder: Bool
}

func scheduleMeeting(
    from startDate: Date,
    to endDate: Date,
    reminderPolicy: MeetingReminderPolicy
)
```

### Trap 2: Mechanical Objective-C / Java / Kotlin naming

Bad Swift:

```swift
func getUserById(_ id: String) async throws -> User
func setUserName(_ name: String)
func isUserActive(_ user: User) -> Bool
```

More Swift-native:

```swift
func user(id: User.ID) async throws -> User
var userName: String
func isActive(_ user: User) -> Bool
```

`get` is often redundant in Swift when the method simply returns a value. There are exceptions, such as APIs that write into an `inout` or pointer parameter, or domain-specific cases like HTTP GET, but ordinary value-returning APIs usually do not need it.

### Trap 3: Boolean blindness

Bad:

```swift
client.request("/users", true, false, true)
```

Better:

```swift
client.request(
    .users,
    cachePolicy: .reloadIgnoringLocalCache,
    allowsCellularAccess: false,
    retries: .upTo(3)
)
```

A boolean is fine when it reads clearly:

```swift
view.dismiss(animated: true)
```

It is not fine when several booleans encode hidden modes.

### Trap 4: Overusing fluent chaining

Bad:

```swift
request
    .withUser(id)
    .withCache(true)
    .withRetry(3)
    .withTimeout(10)
    .execute()
```

This can hide required configuration, invalid states, order-dependence, and error boundaries.

Better:

```swift
let request = UserRequest(
    id: id,
    cachePolicy: .returnCacheElseLoad,
    retryPolicy: .upTo(3),
    timeout: .seconds(10)
)

let user = try await client.user(for: request)
```

Fluent APIs are good when each step is truly compositional and order-independent. They are risky when they simulate a mutable builder without modeling required state.

---

## 4. Direct answers to rubric questions

### Q1. Why are naming and labels part of correctness in Swift API design?

Because the call site is where most API correctness is judged. A bad name or missing label can make two valid calls look equally plausible while only one is semantically correct.

Example:

```swift
func transfer(_ amount: Decimal, _ source: Account, _ destination: Account)
```

Call site:

```swift
transfer(100, savings, checking)
```

This compiles, but the direction is easy to misread.

Better:

```swift
func transfer(
    _ amount: Decimal,
    from source: Account,
    to destination: Account
)
```

Call site:

```swift
transfer(100, from: savings, to: checking)
```

The improved API reduces the chance of reversing source and destination.

Interview version:

> Naming and labels are part of correctness in Swift because they carry semantic information at the call site. Swift APIs are read far more often than they are declared, so labels should express roles, side effects, and relationships between arguments. A legal signature can still be a bad API if it makes the correct call hard to distinguish from an incorrect one.

---

### Q2. What signs show an API was ported mechanically from another language without adapting to Swift idioms?

Common signs:

```text
- get/set prefixes for ordinary properties or simple value-returning methods
- unlabeled positional parameters
- primitive strings, ints, and booleans where domain types should exist
- callback-heavy APIs where async/await would be the natural Swift surface
- ObjC-style naming that repeats type information
- method names that do not read naturally at the call site
- large manager classes with many unrelated verbs
- overexposed public surface area
- nullable/optional-heavy models instead of enum-based state
```

Bad:

```swift
final class UserManager {
    func getUserById(_ id: String, completion: @escaping (User?, Error?) -> Void)
    func setUserName(_ name: String)
    func deleteUserWithId(_ id: String, _ hardDelete: Bool)
}
```

Better:

```swift
protocol UserRepository: Sendable {
    func user(id: User.ID) async throws -> User
    func updateName(_ name: User.Name, for userID: User.ID) async throws
    func deleteUser(id: User.ID, mode: UserDeletionMode) async throws
}

enum UserDeletionMode: Sendable {
    case soft
    case permanent
}
```

Interview version:

> A mechanically ported API usually exposes the source language’s habits instead of Swift’s call-site clarity. I look for `get`/`set` noise, unlabeled parameters, boolean flags, primitive obsession, callback shapes where async/await would be clearer, and names that repeat type information instead of expressing roles. A Swift-native rewrite usually strengthens types, improves labels, chooses properties or methods deliberately, and reduces surface area.

---

## 5. Code probe

No rubric code probe is provided for B10. This section uses API review examples instead.

### Minimal example

```swift
struct ImageProcessor {
    func resize(_ image: UIImage, _ size: CGSize) -> UIImage {
        image
    }
}
```

### What happens?

```text
Compiles successfully.
No runtime output.
```

### Why this is still weak API design

The signature is legal, but the call site is not ideal:

```swift
processor.resize(image, size)
```

The first argument can be unlabeled if it forms a clear phrase with the base name, but this use site is borderline: `resize image size` does not say whether `size` is a target size, maximum size, minimum size, canvas size, or crop size.

### Redesign

```swift
struct ImageProcessor {
    func resizedImage(
        from image: UIImage,
        to targetSize: CGSize,
        contentMode: ImageContentMode
    ) -> UIImage {
        image
    }
}

enum ImageContentMode {
    case aspectFit
    case aspectFill
    case stretch
}
```

Call site:

```swift
let thumbnail = processor.resizedImage(
    from: image,
    to: CGSize(width: 120, height: 120),
    contentMode: .aspectFill
)
```

### Why this fix is correct

The improved signature names the semantic role of each argument. It also replaces a likely future boolean or integer mode with an enum.

### Counterexample

Over-labeling can also be bad:

```swift
imageProcessor.resizeImage(
    imageToResize: image,
    toTargetSize: size,
    usingImageContentMode: .aspectFit
)
```

This is noisy. Swift API design is not “label everything with maximum words.” It is “make the use site clear and concise.”

### Production example

```swift
public protocol ImageLoading: Sendable {
    func image(
        for request: ImageRequest
    ) async throws -> UIImage
}

public struct ImageRequest: Sendable, Hashable {
    public var url: URL
    public var targetSize: CGSize?
    public var scale: CGFloat
    public var cachePolicy: ImageCachePolicy

    public init(
        url: URL,
        targetSize: CGSize? = nil,
        scale: CGFloat = UIScreen.main.scale,
        cachePolicy: ImageCachePolicy = .returnCacheElseLoad
    ) {
        self.url = url
        self.targetSize = targetSize
        self.scale = scale
        self.cachePolicy = cachePolicy
    }
}
```

This API avoids parameter explosion and keeps future evolution easier. Instead of adding overloads for every combination of size, scale, cache policy, priority, and retry behavior, it groups request configuration into a named value.

---

## 6. Exercise

### Problem

Review a small SDK API and rewrite three signatures to better fit Swift API Design Guidelines.

### Bad / naive SDK API

```swift
public final class PaymentSDK {
    public func getCustomerById(
        _ id: String,
        completion: @escaping (Customer?, Error?) -> Void
    )

    public func createPayment(
        _ amount: Double,
        _ currency: String,
        _ save: Bool,
        _ metadata: [String: Any]?
    )

    public func deleteCard(
        _ cardId: String,
        _ hardDelete: Bool
    )
}
```

### What is wrong?

```text
1. getCustomerById carries non-Swift naming and callback-era shape.
2. String IDs and currency codes are weakly typed.
3. completion: (Customer?, Error?) allows invalid states:
   - customer and error both nil
   - customer and error both non-nil
4. createPayment has positional parameters and boolean blindness.
5. amount as Double is risky for money.
6. metadata: [String: Any]? weakens type safety and sendability.
7. deleteCard(_:_:)
   hides the meaning of hardDelete at the call site.
8. The SDK exposes one large manager class instead of smaller capability-focused abstractions.
```

### Improved version

```swift
public protocol CustomerFetching: Sendable {
    func customer(id: Customer.ID) async throws -> Customer
}

public protocol PaymentCreating: Sendable {
    func createPayment(
        amount: Money,
        options: PaymentOptions
    ) async throws -> Payment
}

public protocol CardDeleting: Sendable {
    func deleteCard(
        id: Card.ID,
        mode: CardDeletionMode
    ) async throws
}

public struct Money: Sendable, Hashable {
    public let minorUnits: Int64
    public let currency: Currency

    public init(minorUnits: Int64, currency: Currency) {
        self.minorUnits = minorUnits
        self.currency = currency
    }
}

public struct PaymentOptions: Sendable, Hashable {
    public var savesPaymentMethod: Bool
    public var metadata: [PaymentMetadataKey: PaymentMetadataValue]

    public init(
        savesPaymentMethod: Bool = false,
        metadata: [PaymentMetadataKey: PaymentMetadataValue] = [:]
    ) {
        self.savesPaymentMethod = savesPaymentMethod
        self.metadata = metadata
    }
}

public enum CardDeletionMode: Sendable, Hashable {
    case detachFromCustomer
    case permanentlyDelete
}
```

### Three rewritten signatures

#### 1. Customer lookup

Bad:

```swift
func getCustomerById(
    _ id: String,
    completion: @escaping (Customer?, Error?) -> Void
)
```

Better:

```swift
func customer(id: Customer.ID) async throws -> Customer
```

Why:

```text
- Removes redundant get.
- Uses a typed ID.
- Uses async throws instead of invalid optional/error tuple.
- Reads naturally: customer(id:).
```

#### 2. Payment creation

Bad:

```swift
func createPayment(
    _ amount: Double,
    _ currency: String,
    _ save: Bool,
    _ metadata: [String: Any]?
)
```

Better:

```swift
func createPayment(
    amount: Money,
    options: PaymentOptions
) async throws -> Payment
```

Why:

```text
- Avoids Double for money.
- Avoids positional primitive parameters.
- Replaces boolean/metadata sprawl with a named options value.
- Gives the API room to evolve without overload explosion.
```

#### 3. Card deletion

Bad:

```swift
func deleteCard(_ cardId: String, _ hardDelete: Bool)
```

Better:

```swift
func deleteCard(
    id: Card.ID,
    mode: CardDeletionMode
) async throws
```

Why:

```text
- The ID is typed.
- The deletion behavior is explicit.
- The call site is self-documenting.
```

Call site:

```swift
try await cards.deleteCard(
    id: cardID,
    mode: .detachFromCustomer
)
```

This is materially safer than:

```swift
deleteCard("card_123", false)
```

---

## 7. Production guidance

Use this in production when:

```text
- Designing public SDKs, packages, modules, or app-internal service APIs.
- Replacing callback-era APIs with async/await.
- Reviewing PRs that add new public or widely used APIs.
- Creating domain models for money, IDs, request configuration, analytics, routing, caching, or persistence.
- Reducing overloads and boolean flags.
```

Be careful when:

```text
- A fluent API looks nice but hides required state or ordering constraints.
- An options object becomes a junk drawer.
- You add default parameters to public APIs without thinking about source and behavioral compatibility.
- You expose concrete third-party dependency types in your own API.
- You use Any, [String: Any], String IDs, Int modes, or Bool flags in public signatures.
```

Avoid when:

```text
- You are polishing names while the underlying model is wrong.
- You are creating protocols for every type without a real abstraction boundary.
- You are hiding important side effects behind property-looking APIs.
- You are making call sites short at the cost of semantic clarity.
```

Debugging checklist:

```text
At the call site, can I tell what every argument means?
Could two same-typed arguments be accidentally swapped?
Does a Bool need to become an enum?
Does a String/Int need to become a domain type?
Is this operation naturally a method, property, initializer, or free function?
Does the name reveal side effects?
Is mutation obvious?
Are there too many overloads?
Can this public surface be smaller?
Will this API still make sense after adding one more option?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift APIs should have nice names and follow camelCase.

### Senior answer

> Swift API design is about clarity at the call site. I use labels to express parameter roles, avoid boolean blindness, choose mutating and nonmutating names consistently, and prefer domain types over primitive strings and ints.

### Staff-level answer

> Swift API design is a long-term maintenance and correctness problem. I review APIs from the call site, model domain concepts explicitly, reduce invalid states, minimize public surface area, and think about source compatibility before exposing symbols. I also adapt APIs to Swift’s modern idioms: async/await instead of callback tuples, `Sendable` value types where cross-concurrency use is expected, enum modes instead of booleans, and smaller protocols around real capability boundaries.

Staff-level questions to ask:

```text
What misuse does this signature allow?
Is this API clear from the call site alone?
Are we exposing implementation detail or third-party dependency types?
Can we replace primitive parameters with domain types?
Should this be async throws, Result, AsyncSequence, or a synchronous value?
Is this public API small enough to support for years?
Will adding future options force overload sprawl?
Does the name communicate mutation, allocation, caching, network access, or side effects?
```

---

## 9. Interview-ready summary

Swift API design is not superficial naming. In Swift, names and labels carry semantic information at the call site, so they directly affect correctness, readability, and maintainability. A good API reads naturally, makes side effects and mutation clear, uses labels to express argument roles, replaces primitive flags with domain types, and keeps the public surface small. At senior/staff level, I would not just rename methods; I would question whether the abstraction, error model, concurrency shape, mutability, and evolution story are right.

---

## 10. Flashcards

Q: What is the most important goal of Swift API design?

A: Clarity at the point of use.

Q: Why are argument labels part of correctness?

A: They express semantic roles at the call site and reduce misuse, especially when parameters share weak or identical types.

Q: What is boolean blindness?

A: A call-site problem where `true` or `false` does not explain the behavior being selected.

Q: How should mutating and nonmutating pairs usually be named?

A: Use imperative verbs for mutating methods and `-ed` / `-ing` forms for nonmutating variants, such as `sort()` / `sorted()`.

Q: When should a `Bool` parameter become an enum?

A: When the meaning is not obvious at the call site, when more modes may appear, or when the boolean encodes domain behavior.

Q: What is a sign of mechanically ported API design?

A: `get`/`set` noise, unlabeled positional parameters, primitive flags, callback tuples, and names that repeat type information instead of expressing roles.

Q: Why is minimizing surface area part of API design?

A: Every exposed symbol becomes a maintenance, documentation, testing, and compatibility commitment.

Q: What is better than `[String: Any]` in a public Swift API?

A: A domain-specific value type or typed dictionary, ideally with `Sendable` and `Hashable` when appropriate.

---

## 11. Related sections

- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A10 — Extensions and Retroactive Modeling]]
- [[A11 — Access Control]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B7 — Synthesized Conformances and Semantic Correctness]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[E6 — Import Visibility and Dependency Leakage]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — B10: API design guidelines and Swift-native surface design.
- Swift.org. "API Design Guidelines." https://swift.org/documentation/api-design-guidelines/
