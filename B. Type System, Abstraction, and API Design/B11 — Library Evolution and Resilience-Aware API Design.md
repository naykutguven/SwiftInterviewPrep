---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
  - library-evolution
  - resilience
---
## 0. Rubric snapshot

**Rubric expectation**

Know what becomes hard to change once a symbol is `public`, and how resilience affects enums, structs, and inlining choices.

**Caveats**

Public API is a promise. Convenience today can become ABI, source-compatibility, or semantic-compatibility debt later.

**You should be able to answer**

- Why is making something `public` in a library more than an access-control decision?
- When would `@frozen` or `@inlinable` be a mistake?

**You should be able to do**

- Evaluate whether a public enum in an SDK should be frozen and how clients should switch over it.

This follows the project’s Swift section answer structure.

---

## 1. Core mental model

Library evolution is about designing APIs so a library can change internally without breaking clients.

The moment a declaration becomes `public`, client code can start depending on it. That dependency is not only about calling the symbol. Clients may depend on its name, signature, generic constraints, enum cases, stored-property layout, protocol requirements, actor isolation, default behavior, and sometimes even implementation body if you mark it `@inlinable`.

Swift’s resilience model creates a boundary between the library and its clients. Across that boundary, the compiler avoids assuming details that the library should be allowed to change later. For example, resilient structs and enums may be manipulated indirectly across module boundaries so the library can evolve representation later. Swift.org describes this as a “resilience boundary” where client compilation must only rely on assumptions guaranteed to remain valid in future library versions. ([Swift.org, "Library Evolution in Swift"](https://swift.org/blog/library-evolution/))

The key idea:

```text
public API = source contract
public ABI = binary contract
@frozen / @inlinable = voluntarily exposing more implementation detail for performance or ergonomics
```

Resilience gives library authors room to evolve. `@frozen` and `@inlinable` intentionally reduce that room.

That tradeoff is the whole topic.

---

## 2. Essential mechanics

### 2.1 `public` exposes more than visibility

`public` means code in another module can name and use the declaration.

```swift
public struct User {
    public let id: UUID
    public var displayName: String

    public init(id: UUID, displayName: String) {
        self.id = id
        self.displayName = displayName
    }
}
```

Once shipped, these become hard to change:

```text
User                    // type name
id                      // public property name and type
displayName             // public property name and type
init(id:displayName:)    // initializer signature and labels
```

Breaking examples:

```swift
// Source-breaking for clients:
public var name: String // renamed from displayName

// Source-breaking for clients:
public init(id: UUID, name: String)

// Source-breaking and potentially ABI-breaking:
public let id: String // changed from UUID
```

Even if the implementation is simple, the public surface is now part of the client’s source code.

---

### 2.2 Resilience hides representation across module boundaries

A resilient public struct can preserve implementation flexibility.

```swift
public struct Token {
    private var rawValue: String

    public init(_ rawValue: String) {
        self.rawValue = rawValue
    }

    public var description: String {
        rawValue
    }
}
```

If the library is built with library evolution support, clients do not need to know the exact stored-property layout. The library can later change private storage:

```swift
public struct Token {
    private var storage: Storage

    public init(_ rawValue: String) {
        self.storage = Storage(rawValue)
    }

    public var description: String {
        storage.rawValue
    }
}
```

That is the point of resilience.

Swift.org notes that across a resilience boundary, property access generally goes through accessors, which allows the property’s underlying implementation to change. The major exception is stored properties of `@frozen` structs, where the compiler can directly access layout. ([Swift.org, "Library Evolution in Swift"](https://swift.org/blog/library-evolution/))

---

### 2.3 `@frozen` opts out of some resilience

`@frozen` says: “Clients may rely on this type’s shape.”

For structs:

```swift
@frozen
public struct Point {
    public let x: Double
    public let y: Double
}
```

This promises the stored-property layout will not change in a binary-incompatible way. Adding, removing, or reordering stored properties of an `@frozen` public struct is binary-incompatible. Swift.org explicitly describes `@frozen` structs as publishing stored-property layout to clients. ([Swift.org, "Library Evolution in Swift"](https://swift.org/blog/library-evolution/))

Bad use:

```swift
@frozen
public struct UserProfile {
    public let id: UUID
    public let name: String

    // You are probably going to want avatarURL, locale,
    // verification state, privacy settings, etc. later.
}
```

Better:

```swift
public struct UserProfile {
    public let id: UUID
    public let name: String

    public init(id: UUID, name: String) {
        self.id = id
        self.name = name
    }
}
```

Leave it resilient unless you have a strong reason to freeze it.

---

### 2.4 `@frozen` enums promise no future cases

A frozen enum says clients may switch exhaustively without future-proofing.

```swift
@frozen
public enum SortDirection {
    case ascending
    case descending
}
```

This is reasonable if the domain is genuinely closed.

But this is dangerous:

```swift
@frozen
public enum PaymentStatus {
    case authorized
    case captured
    case failed
}
```

Payments evolve. Providers add review states, fraud states, regulatory states, partial-capture states, chargeback states, and async settlement states. Freezing this enum is probably wrong.

A non-frozen enum in a resilient library requires clients to handle future cases, usually with `@unknown default`. Swift.org states that a switch over a frozen enum is exhaustive when all cases are covered, while a switch over a non-frozen enum must include `default` or `@unknown default`. ([Swift.org, "Library Evolution in Swift"](https://swift.org/blog/library-evolution/))

Client-side:

```swift
switch status {
case .authorized:
    showAuthorized()
case .captured:
    showCaptured()
case .failed:
    showFailure()
@unknown default:
    showGenericPendingOrUnknownState()
}
```

Use `@unknown default`, not plain `default`, because it preserves compiler warnings when new known cases appear.

---

### 2.5 Swift 6.2-era note: `@nonexhaustive`

For non-resilient Swift packages, enum evolution used to be especially awkward: public enums were effectively exhaustive to clients, so adding a case was source-breaking.

SE-0487 introduces `@nonexhaustive` for public enums in non-resilient libraries. The proposal is marked implemented in Swift 6.2.3, and its goal is to let package authors mark public enums as extensible without requiring full library-evolution mode. ([GitHub, "Nonexhaustive Enums"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md))

Example:

```swift
@nonexhaustive
public enum PaymentStatus {
    case authorized
    case captured
    case failed
}
```

Client code in another module must handle future cases:

```swift
switch status {
case .authorized:
    showAuthorized()
case .captured:
    showCaptured()
case .failed:
    showFailure()
@unknown default:
    showGenericFallback()
}
```

Important nuance: inside the same module or package, exhaustive switching over an `@nonexhaustive` enum does not require `@unknown default`, because the package can be treated as a co-developed unit. ([GitHub, "Nonexhaustive Enums"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md))

---

### 2.6 `@inlinable` exposes implementation body

`@inlinable` lets client modules see a function body for cross-module optimization.

```swift
@inlinable
public func clamped(_ value: Int, to range: ClosedRange<Int>) -> Int {
    min(max(value, range.lowerBound), range.upperBound)
}
```

This is not just a performance annotation. It is a promise that the current body remains valid for future versions of the library. Swift.org says `@inlinable` allows the compiler to look at the body when building client code, but it does not guarantee actual inlining. ([Swift.org, "Library Evolution in Swift"](https://swift.org/blog/library-evolution/))

The body can only reference ABI-public declarations: either `public` or `@usableFromInline`.

```swift
@usableFromInline
func normalize(_ value: Int) -> Int {
    max(0, value)
}

@inlinable
public func score(_ value: Int) -> Int {
    normalize(value)
}
```

`@usableFromInline` is not API-public, but it is ABI-public. Swift.org explicitly says that once published, a `@usableFromInline` declaration must not be removed or incompatibly changed. ([Swift.org, "Library Evolution in Swift"](https://swift.org/blog/library-evolution/))

So this is a mistake:

```swift
@usableFromInline
func temporaryHack(_ input: String) -> String {
    input.trimmingCharacters(in: .whitespacesAndNewlines)
}

@inlinable
public func parse(_ input: String) -> ParsedValue {
    ParsedValue(temporaryHack(input))
}
```

You just turned a temporary helper into binary-interface debt.

---

## 3. Common traps and misconceptions

### Trap 1: “`public` just means other modules can call it”

Bad mental model:

```text
public = visible
```

Better mental model:

```text
public = clients can build source code around it
```

Bad:

```swift
public struct SDKConfig {
    public var baseURL: URL
    public var timeout: TimeInterval
    public var retries: Int
}
```

This exposes every field directly. You have little room to validate, derive, migrate, or change representation.

Better:

```swift
public struct SDKConfig: Sendable {
    public let baseURL: URL
    public let timeout: Duration
    public let retryPolicy: RetryPolicy

    public init(
        baseURL: URL,
        timeout: Duration = .seconds(30),
        retryPolicy: RetryPolicy = .default
    ) {
        self.baseURL = baseURL
        self.timeout = timeout
        self.retryPolicy = retryPolicy
    }
}
```

This still exposes a useful API, but the semantic model is clearer and harder to misuse.

---

### Trap 2: Freezing enums that represent evolving domains

Bad:

```swift
@frozen
public enum SubscriptionState {
    case active
    case expired
    case cancelled
}
```

This looks closed today, but product and billing domains rarely stay closed.

Better:

```swift
@nonexhaustive
public enum SubscriptionState {
    case active
    case expired
    case cancelled
}
```

Or, for a truly open-ended server-driven domain:

```swift
public struct SubscriptionState: RawRepresentable, Hashable, Sendable {
    public let rawValue: String

    public init(rawValue: String) {
        self.rawValue = rawValue
    }

    public static let active = Self(rawValue: "active")
    public static let expired = Self(rawValue: "expired")
    public static let cancelled = Self(rawValue: "cancelled")
}
```

Use an enum when you want pattern matching and a known set of cases. Use a value wrapper when the domain is externally extensible.

---

### Trap 3: Using `@inlinable` as a default optimization habit

Bad:

```swift
@inlinable
public func makeRequest(_ endpoint: Endpoint) async throws -> Response {
    try await transport.send(
        endpoint,
        headers: defaultHeaders(),
        retryPolicy: retryPolicy()
    )
}
```

This exposes too much implementation shape. If `transport`, headers, retry policy, or request construction change, you may have created ABI-public constraints.

Better:

```swift
public func makeRequest(_ endpoint: Endpoint) async throws -> Response {
    try await client.send(endpoint)
}
```

Only reach for `@inlinable` after profiling shows a meaningful cross-module optimization problem.

---

### Trap 4: Adding enum cases casually in a source package

Bad:

```swift
public enum ImageSource {
    case remote(URL)
    case asset(String)
}
```

A client may write:

```swift
switch source {
case .remote(let url):
    loadRemote(url)
case .asset(let name):
    loadAsset(name)
}
```

If your package later adds:

```swift
case file(URL)
```

that client’s source may fail to compile when they update.

Better, if future cases are likely:

```swift
@nonexhaustive
public enum ImageSource {
    case remote(URL)
    case asset(String)
}
```

Then clients must include future handling.

---

## 4. Direct answers to rubric questions

### Q1. Why is making something `public` in a library more than an access-control decision?

Because `public` creates a compatibility contract with clients.

Clients can depend on names, signatures, labels, generic constraints, enum cases, protocol requirements, class inheritance behavior, actor isolation, thrown error shapes, and documented semantics. In binary-distributed libraries, clients may also depend on ABI details. If you add `@frozen`, clients can depend on stored layout or enum exhaustiveness. If you add `@inlinable`, clients can depend on implementation body.

Access control answers “who can see this?” Library evolution asks “what must remain compatible after clients depend on it?”

Interview version:

> Making a symbol public is not just visibility. It is a promise to external clients. The symbol’s name, type, labels, semantics, and sometimes ABI shape become part of the library contract. Resilience lets Swift hide some implementation details across module boundaries, but annotations like `@frozen` and `@inlinable` deliberately expose more detail. A senior engineer treats every public declaration as long-term maintenance debt unless there is a clear reason to expose it.

---

### Q2. When would `@frozen` be a mistake?

`@frozen` is a mistake when the type’s representation or case set may need to evolve.

For structs, `@frozen` is risky if you might add, remove, reorder, or change stored properties. This includes types with private storage, property wrappers, lazy properties, caches, optimization fields, or platform-specific representation.

For enums, `@frozen` is risky if future cases are plausible. SDK error enums, server statuses, payment states, subscription states, permission states, experiment variants, and vendor-driven states usually should not be frozen.

Interview version:

> I would freeze only genuinely closed, stable domains where the performance or ergonomics benefit is worth permanently giving up evolution flexibility. A small enum like sort direction may be fine. An SDK enum representing server, OS, payment, or product states should usually stay non-frozen or be marked nonexhaustive so clients handle future cases.

---

### Q3. When would `@inlinable` be a mistake?

`@inlinable` is a mistake when the function body contains implementation details you want freedom to change, or when there is no measured cross-module performance need.

It is especially risky for high-level SDK APIs, networking APIs, persistence APIs, feature-flag logic, analytics, authentication, and anything with evolving policy. `@inlinable` is more appropriate for tiny, stable, performance-sensitive generic helpers where the body is essentially part of the intended contract.

Interview version:

> I avoid `@inlinable` by default. It is not a magic speed button; it exposes the function body across module boundaries and restricts what the body can reference. I would use it only for small, stable, performance-sensitive APIs after measurement, especially generic utilities where cross-module specialization matters. Otherwise, I keep the implementation resilient.

---

## 5. No rubric code probe

This section has no rubric code probe, so use these examples instead.

### Minimal example: public enum evolution

Version 1:

```swift
public enum LoginState {
    case loggedOut
    case loggedIn(User)
}
```

Client:

```swift
switch state {
case .loggedOut:
    showLogin()
case .loggedIn(let user):
    showHome(user)
}
```

Version 2:

```swift
public enum LoginState {
    case loggedOut
    case loggedIn(User)
    case requiresTwoFactor(User)
}
```

For a normal public enum in a non-resilient package, adding this case can break client source because exhaustive switches must now handle the new case.

Better in Swift 6.2-era toolchains with SE-0487 support:

```swift
@nonexhaustive
public enum LoginState {
    case loggedOut
    case loggedIn(User)
}
```

Client:

```swift
switch state {
case .loggedOut:
    showLogin()
case .loggedIn(let user):
    showHome(user)
@unknown default:
    showSafeFallback()
}
```

Expected behavior:

```text
Clients must handle future unknown cases.
When a new case appears, clients still compile if they have @unknown default,
but the compiler can still help surface missing known cases.
```

---

### Counterexample: incorrectly frozen enum

```swift
@frozen
public enum LoginState {
    case loggedOut
    case loggedIn(User)
}
```

This says the enum will not gain future cases.

That is a bad promise if your authentication system may later add:

```swift
case requiresTwoFactor(User)
case lockedOut(until: Date)
case passwordExpired(User)
case pendingDeviceApproval(User)
```

The bug is not syntax. The bug is that the API made an evolution promise the domain cannot keep.

---

### Production example: SDK status modeling

```swift
@nonexhaustive
public enum PaymentStatus: Sendable {
    case requiresPaymentMethod
    case requiresConfirmation
    case processing
    case succeeded
    case failed(PaymentFailure)
}
```

Client:

```swift
func render(_ status: PaymentStatus) {
    switch status {
    case .requiresPaymentMethod:
        showPaymentMethodForm()

    case .requiresConfirmation:
        showConfirmation()

    case .processing:
        showProcessing()

    case .succeeded:
        showSuccess()

    case .failed(let failure):
        showFailure(failure)

    @unknown default:
        showGenericPendingState()
        logUnknownStatus(status)
    }
}
```

This is the right shape if the SDK owner expects future statuses.

But if the domain is genuinely closed:

```swift
@frozen
public enum Axis: Sendable {
    case horizontal
    case vertical
}
```

Client:

```swift
switch axis {
case .horizontal:
    layoutHorizontally()
case .vertical:
    layoutVertically()
}
```

No `@unknown default` is needed because the API intentionally promises the case set is closed.

---

## 6. Exercise

### Problem

Evaluate whether a public enum in an SDK should be frozen and how clients should switch over it.

### Bad / naive version

```swift
@frozen
public enum DeliveryStatus: Sendable {
    case pending
    case assignedCourier
    case pickedUp
    case delivered
    case cancelled
}
```

Client:

```swift
switch status {
case .pending:
    showPending()
case .assignedCourier:
    showCourierAssigned()
case .pickedUp:
    showPickedUp()
case .delivered:
    showDelivered()
case .cancelled:
    showCancelled()
}
```

### What is wrong?

```text
Delivery status is not a closed domain.
The provider may add new states:
- delayed
- returned
- partiallyDelivered
- failedAddressValidation
- awaitingCustomerAction
- handoffToPartnerCarrier

@frozen promises no future cases.
The client switch has no fallback.
The SDK has traded short-term exhaustiveness for long-term API debt.
```

### Improved version

```swift
@nonexhaustive
public enum DeliveryStatus: Sendable {
    case pending
    case assignedCourier
    case pickedUp
    case delivered
    case cancelled
}
```

Client:

```swift
func render(_ status: DeliveryStatus) {
    switch status {
    case .pending:
        showPending()

    case .assignedCourier:
        showCourierAssigned()

    case .pickedUp:
        showPickedUp()

    case .delivered:
        showDelivered()

    case .cancelled:
        showCancelled()

    @unknown default:
        showGenericInProgressState()
        logUnknownDeliveryStatus(status)
    }
}
```

### Why this is better

The SDK keeps the freedom to add future cases. Clients still handle all known cases explicitly, but they also define safe behavior for unknown future states.

That is usually the correct production behavior for SDKs backed by server-side, business, regulatory, or third-party domains.

### Alternative: use a raw-value struct for highly extensible domains

If statuses are truly server-defined and open-ended, an enum may still be too rigid.

```swift
public struct DeliveryStatus: RawRepresentable, Hashable, Sendable {
    public let rawValue: String

    public init(rawValue: String) {
        self.rawValue = rawValue
    }

    public static let pending = Self(rawValue: "pending")
    public static let assignedCourier = Self(rawValue: "assigned_courier")
    public static let pickedUp = Self(rawValue: "picked_up")
    public static let delivered = Self(rawValue: "delivered")
    public static let cancelled = Self(rawValue: "cancelled")
}
```

Client:

```swift
switch status {
case .pending:
    showPending()
case .assignedCourier:
    showCourierAssigned()
case .pickedUp:
    showPickedUp()
case .delivered:
    showDelivered()
case .cancelled:
    showCancelled()
default:
    showGenericInProgressState()
}
```

Tradeoff:

```text
enum:
- better pattern matching
- clearer known cases
- harder to evolve unless nonexhaustive

raw-value struct:
- easier open-ended evolution
- less exhaustive compiler help
- better for server-defined code systems
```

---

## 7. Production guidance

Use resilience-aware public API design when:

```text
You publish Swift packages, SDKs, frameworks, binary frameworks, or shared internal platform libraries.
You expect clients to update independently from the library.
You care about semantic versioning.
You expose enums, protocols, structs, generic APIs, or performance-sensitive helpers.
You want to avoid accidental API debt.
```

Be careful when:

```text
Adding public enum cases.
Adding public protocol requirements.
Changing generic constraints.
Changing actor isolation on public APIs.
Changing Sendable conformance or closure sendability.
Changing public initializer labels.
Changing default argument values in libraries.
Adding @frozen.
Adding @inlinable.
Adding @usableFromInline.
```

Avoid when:

```text
Freezing domain models that are controlled by servers, OS versions, vendors, regulations, or product strategy.
Inlining high-level APIs whose implementation is likely to change.
Exposing third-party dependency types in public signatures.
Making storage public because it is convenient.
Adding protocols before you know the abstraction boundary.
```

Debugging checklist:

```text
Did this change break source compatibility?
Did this change break binary compatibility?
Did this public API expose implementation detail?
Are clients switching exhaustively over an enum we need to extend?
Did we add a public protocol requirement without a default implementation?
Did @inlinable accidentally expose an internal helper through @usableFromInline?
Did @frozen prevent a representation change?
Is the type resilient in the module configuration we actually ship?
Are we dealing with a source package, binary framework, or Apple-platform framework?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Public means another module can use it. `@frozen` means the enum or struct cannot change. `@inlinable` helps performance.

### Senior answer

> Public APIs are compatibility contracts. I avoid exposing storage and avoid freezing types unless the domain is truly closed. For public enums, I decide whether clients should switch exhaustively or include `@unknown default`. I use `@inlinable` only for small, stable, performance-sensitive functions because it exposes implementation details across module boundaries.

### Staff-level answer

> I separate source compatibility, ABI compatibility, and semantic compatibility. For source packages, I think about SemVer and client recompilation. For binary frameworks, I think about resilience boundaries, layout exposure, dispatch, and cross-module optimization. I avoid freezing or inlining unless the API is intentionally stable and the performance win is measured. For public enums, I classify the domain: closed domains can be frozen; evolving SDK/server domains should be non-frozen or `@nonexhaustive`; highly open domains may be better modeled as raw-value structs. I also design migration paths, deprecation windows, fallback behavior, and API-breaking checks before shipping the public surface.

Staff-level questions to ask:

```text
Is this API source-stable, ABI-stable, or both?
Will clients update source and library together, or independently?
Is this enum domain mathematically closed, product-closed, or externally extensible?
Do clients benefit more from exhaustive switching or forward compatibility?
Is @inlinable justified by profiling, or are we exposing implementation detail prematurely?
Can we add this protocol requirement later without breaking conformers?
Are default arguments, generic constraints, actor isolation, and Sendable annotations part of our compatibility review?
Should this be public, package, internal, or test-only?
```

---

## 9. Interview-ready summary

Library evolution is about preserving the ability to change a library after clients depend on it. `public` is a compatibility promise, not just visibility. Swift resilience hides implementation details across module boundaries, but annotations like `@frozen` and `@inlinable` intentionally expose more detail for exhaustiveness or optimization. I freeze only genuinely closed structs or enums, and I use `@inlinable` only for small, stable, measured hot paths. For SDK enums, I usually prefer non-frozen or `@nonexhaustive` designs with client-side `@unknown default`, unless the case set is truly permanent.

---

## 10. Flashcards

Q: What is the core purpose of Swift resilience?  
A: To let a library change implementation details without breaking clients compiled against its public API.

Q: Why is `public` more than visibility?  
A: It creates a source compatibility contract. Clients can depend on names, signatures, labels, types, enum cases, protocol requirements, and documented semantics.

Q: What does `@frozen` do for a public struct?  
A: It publishes stored-property layout to clients, making adding, removing, or reordering stored properties binary-incompatible.

Q: What does `@frozen` do for a public enum?  
A: It promises the enum will not gain new cases, allowing clients to switch exhaustively.

Q: When is `@frozen` a mistake?  
A: When the type’s stored representation or enum case set may need to evolve.

Q: What does `@inlinable` expose?  
A: The function body across module boundaries for optimization, making the body part of the library’s ABI-relevant contract.

Q: Does `@inlinable` guarantee inlining?  
A: No. It allows the client compiler to see the body; the compiler may inline, specialize, emit a copy, or still call the framework implementation.

Q: What is `@usableFromInline`?  
A: An internal declaration that is not API-public but is ABI-public so `@inlinable` code can reference it.

Q: Why is `@unknown default` better than plain `default` for non-frozen SDK enums?  
A: It handles future cases while still letting the compiler warn when new known cases should be reviewed.

Q: When might a raw-value struct be better than an enum?  
A: When the domain is highly open-ended, server-defined, or vendor-defined, and new values should not require enum evolution.

---

## 11. Related sections

- [[A11 — Access control]]
- [[A12 — Availability and conditional compilation]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B13 — Property wrappers and generated storage semantics]]
- [[E6 — Import visibility and dependency leakage]]
- [[E7 — Resilience, ABI, and module stability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F5 — Language mode, build settings, and migration flags]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>)
- Swift.org. "Library Evolution in Swift." Swift.org Blog. https://swift.org/blog/library-evolution/
- GitHub. "Nonexhaustive Enums." Swift Evolution SE-0487. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0487-extensible-enums.md
- Swift Forums. "SE-0193: Cross-Module Inlining and Specialization." https://forums.swift.org/t/se-0193-cross-module-inlining-and-specialization/7310?page=8
