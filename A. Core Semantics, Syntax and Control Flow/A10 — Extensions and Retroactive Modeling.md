---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - extensions
  - retroactive-modeling
---
## 0. Rubric snapshot

**Rubric expectation**

Know how extensions organize behavior, add protocol conformances, and change discoverability. The rubric specifically flags A10 as a senior-level topic because retroactive conformances are global and can create semantic collisions across modules.

**Caveats**

Retroactive conformances are not local decoration. Once a module adds a conformance, that conformance becomes part of the program’s type system behavior for clients that import the module. Swift Evolution SE-0364 states that protocol conformances are globally unique within a process in the Swift runtime, and duplicate conformances can cause major problems for clients and library evolution. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md "swift-evolution/proposals/0364-retroactive-conformance-warning.md at main · swiftlang/swift-evolution · GitHub"))

**You should be able to answer**

- When is adding a conformance in an extension a good idea, and when is it dangerous?
- Why can an extension-based conformance in one module affect code elsewhere?

**You should be able to do**

- Review a package that adds `Codable` to an external type.
- Explain the compatibility, ownership, and semantic risks.

---

## 1. Core mental model

An extension lets you attach behavior to an existing nominal type after the type’s original declaration. This is useful for organizing code by responsibility, adding protocol conformances, separating platform-specific behavior, and keeping public models readable. Swift’s language guide describes extensions as a way to add new functionality to existing types, including types whose original source you do not control. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/?utm_source=chatgpt.com "Extensions - Documentation | Swift.org"))

The important point: extensions do **not** create a local “view” of a type. They modify what members or conformances are visible through normal Swift lookup in the module/import context. A method added through an extension is still a method on that type. A protocol conformance added through an extension is still a conformance of that type.

Protocol conformance is the sharp edge. If you write:

```swift
extension ExternalType: Codable {}
```

you are not merely helping your own file compile. You are declaring that `ExternalType` conforms to `Codable` for the whole module graph that sees this module. If this is a library, every app or package that imports it can inherit that modeling decision.

Retroactive modeling means: “I am modeling a type after the fact.” This is often powerful when you own either the type or the protocol. It becomes dangerous when you own neither.

The key idea:

```text
extension member = added behavior
extension conformance = global semantic claim about a type
retroactive conformance = global semantic claim made by a non-owner
```

Swift guarantees that extension members and conformances are checked by the compiler. Swift does **not** guarantee that a conformance you add for someone else’s type to someone else’s protocol will remain compatible with future versions of those modules. SE-0364 exists because if the owning module later adds the same conformance, it is indeterminate which conformance “wins” before the duplicate is removed, and clients can observe broken behavior. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md "swift-evolution/proposals/0364-retroactive-conformance-warning.md at main · swiftlang/swift-evolution · GitHub"))

---

## 2. Essential mechanics

### Extensions add behavior, not stored instance state

Extensions can add computed properties, methods, initializers, subscripts, nested types, and protocol conformances. They cannot add stored instance properties because that would change the stored layout of an already-declared type.

```swift
struct User {
    let id: UUID
    let name: String
}

extension User {
    var displayName: String {
        name.isEmpty ? "Unknown User" : name
    }

    func isSameUser(as other: User) -> Bool {
        id == other.id
    }
}
```

This is a good use of extensions: the behavior belongs naturally to `User`, and the type is owned by your module.

Bad:

```swift
extension User {
    // Not allowed: extensions can't add stored instance properties.
    var cachedAvatarURL: URL?
}
```

Better:

```swift
struct UserViewState {
    let user: User
    var cachedAvatarURL: URL?
}
```

The better version makes the additional state explicit and avoids pretending that every `User` globally has UI cache state.

---

### Extensions can add protocol conformances

You can add protocol conformance in an extension. Swift’s protocol documentation explicitly supports extending an existing type to adopt and conform to a new protocol, even when you do not have access to the original type’s source. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/?utm_source=chatgpt.com "Protocols | Documentation - Swift Programming Language"))

```swift
struct User {
    let id: UUID
    let name: String
}

extension User: Identifiable {}

extension User: Codable {}

extension User: Hashable {}
```

This is usually good when you own `User`. The extension can group conformance-specific code and improve readability:

```swift
extension User: CustomStringConvertible {
    var description: String {
        "\(name) (\(id))"
    }
}
```

This is especially useful for separating concerns:

```swift
// User.swift
struct User {
    let id: UUID
    let name: String
}

// User+Codable.swift
extension User: Codable {}

// User+ViewModels.swift
extension User {
    var title: String { name }
}
```

This organization improves discoverability when done consistently. It hurts discoverability when behavior is scattered across unrelated files, targets, or dependencies.

---

### Ownership matrix for protocol conformances

The conformance risk depends on who owns the type and who owns the protocol.

|Type owner|Protocol owner|Example|Risk|
|---|--:|---|---|
|You|You|`extension User: AppSerializable`|Low|
|You|External|`extension User: Codable`|Usually fine|
|External|You|`extension URLRequest: EndpointConvertible`|Usually acceptable|
|External|External|`extension Date: Identifiable`|Dangerous retroactive conformance|

The dangerous case is the last one: you own neither the type nor the protocol.

```swift
import Foundation

extension Date: Identifiable {
    public var id: TimeInterval { timeIntervalSince1970 }
}
```

This compiles with a Swift 6 warning, because both `Date` and `Identifiable` come from external modules relative to your module.

SE-0364 says the warning appears when the extended type and the protocol both come from different modules than the extension, with exceptions such as same-package conformances and Swift overlays. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md "swift-evolution/proposals/0364-retroactive-conformance-warning.md at main · swiftlang/swift-evolution · GitHub"))

---

### `@retroactive` acknowledges risk; it does not remove risk

Swift 6 added a warning for retroactive conformances of external types. SE-0364 marks this proposal as implemented in Swift 6.0. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md "swift-evolution/proposals/0364-retroactive-conformance-warning.md at main · swiftlang/swift-evolution · GitHub"))

```swift
import Foundation

extension Date: @retroactive Identifiable {
    public var id: TimeInterval { timeIntervalSince1970 }
}
```

This silences the warning, but it means:

```text
I know this conformance is retroactive.
I accept responsibility for future collisions.
```

It is not a magic namespace. It does not make the conformance private. Swift does not support private protocol conformances; this is also why conformance extensions cannot have explicit access modifiers in the same way normal member-only extensions can. The Swift access-control documentation notes that an extension adding protocol conformance cannot provide an explicit access-level modifier; the protocol’s access level controls the default access level for protocol requirement implementations. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/?utm_source=chatgpt.com "Access Control | Documentation - Swift Programming Language"))

---

## 3. Common traps and misconceptions

### Trap 1: “It is just an extension, so it is local”

Bad:

```swift
// In MyAnalyticsSDK

import Foundation

extension Date: Identifiable {
    public var id: TimeInterval { timeIntervalSince1970 }
}
```

This is not local to `MyAnalyticsSDK` in practice. Any client importing `MyAnalyticsSDK` can now see `Date` as `Identifiable`, and generic APIs constrained to `Identifiable` may start accepting `Date`.

Better:

```swift
public struct AnalyticsDateID: Hashable, Codable, Sendable {
    public let rawValue: TimeInterval

    public init(_ date: Date) {
        self.rawValue = date.timeIntervalSince1970
    }
}
```

This models your SDK’s identity rule without claiming that `Date` globally has that identity.

---

### Trap 2: Adding `Codable` to external types because it is convenient

Bad:

```swift
import ExternalPaymentsSDK

extension PaymentCard: Codable {}
```

This is risky if `PaymentCard` is external and `Codable` is external. Your module now defines the serialization semantics for a type you do not own.

Better:

```swift
import ExternalPaymentsSDK

public struct PaymentCardDTO: Codable, Sendable {
    public let id: String
    public let brand: String
    public let last4: String

    public init(card: PaymentCard) {
        self.id = card.id
        self.brand = card.brand.rawValue
        self.last4 = card.lastFourDigits
    }
}
```

The DTO is owned by your module, so its schema evolution is your responsibility.

---

### Trap 3: Splitting extensions until behavior becomes undiscoverable

Bad:

```swift
// User.swift
struct User {
    let id: UUID
    let name: String
}

// User+Formatting.swift
extension User {
    var title: String { name }
}

// SomeRandomFile.swift
extension User {
    var isPremiumDisplayEligible: Bool { true }
}

// DebugHelpers.swift
extension User: CustomDebugStringConvertible {
    var debugDescription: String { "\(id): \(name)" }
}
```

This becomes hard to audit. A developer reading `User.swift` cannot tell what the type can do.

Better:

```swift
// User.swift
struct User {
    let id: UUID
    let name: String
    let subscription: Subscription?
}

// MARK: - Display

extension User {
    var displayTitle: String {
        name
    }

    var isPremiumDisplayEligible: Bool {
        subscription?.isActive == true
    }
}

// MARK: - CustomDebugStringConvertible

extension User: CustomDebugStringConvertible {
    var debugDescription: String {
        "\(id): \(name)"
    }
}
```

Use extensions to group behavior by responsibility, not to hide complexity.

---

### Trap 4: Assuming protocol extension methods are always dynamically dispatched

This topic overlaps with [[B4 — Dispatch model: static, class virtual, witness-table, and protocol extension dispatch]].

```swift
protocol Trackable {}

extension Trackable {
    func track() {
        print("default")
    }
}

struct Screen: Trackable {
    func track() {
        print("screen")
    }
}

let value: any Trackable = Screen()
value.track()
```

If `track()` is not a protocol requirement, calls through the existential use the extension member, not the concrete method. Extension-based organization can therefore change dispatch expectations if you confuse protocol requirements with protocol-extension helpers.

Better:

```swift
protocol Trackable {
    func track()
}

extension Trackable {
    func track() {
        print("default")
    }
}
```

Now `track()` is a requirement, and conforming types can participate in witness-table dispatch.

---

## 4. Direct answers to rubric questions

### Q1. When is adding a conformance in an extension a good idea, and when is it dangerous?

Adding a conformance in an extension is a good idea when you own the type, or when you own the protocol and intentionally adapt an external type into your domain. It is dangerous when you own neither the type nor the protocol.

Good:

```swift
struct FeedItem {
    let id: UUID
    let title: String
}

extension FeedItem: Identifiable {}
extension FeedItem: Codable {}
extension FeedItem: Sendable {}
```

You own `FeedItem`, so you own the semantics of identity, serialization, and sendability.

Also reasonable:

```swift
protocol AnalyticsValue {
    var analyticsDescription: String { get }
}

extension URL: AnalyticsValue {
    var analyticsDescription: String {
        absoluteString
    }
}
```

Here, `URL` is external, but `AnalyticsValue` is your protocol. The conformance expresses your module’s domain-specific interpretation.

Dangerous:

```swift
import Foundation

extension Date: Identifiable {
    public var id: TimeInterval { timeIntervalSince1970 }
}
```

You own neither `Date` nor `Identifiable`. If Foundation later adds `Date: Identifiable` with a different `id`, your code and persisted data can break. SE-0364 uses this exact class of problem as motivation: a library adding `Date: Identifiable` can conflict with a future Foundation conformance and propagate the issue to every client importing that library. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md "swift-evolution/proposals/0364-retroactive-conformance-warning.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> Adding a conformance in an extension is clean when the module owns the semantic decision. If I own the type, conforming it to `Codable`, `Hashable`, or `Sendable` in extensions is usually just organization. If I own the protocol, adapting external types can also be valid. The risky case is conforming someone else’s type to someone else’s protocol. That conformance is globally visible and can collide with future conformances from the owning modules, so in a library I would usually avoid it and use a wrapper or DTO instead.

---

### Q2. Why can an extension-based conformance in one module affect code elsewhere?

Because protocol conformance is not scoped to the file where the extension appears. Once the module is imported, the conformance participates in type checking, overload resolution, generic constraints, encoding/decoding, collection behavior, and runtime conformance lookup.

Example:

```swift
// Module: AnalyticsSDK

import Foundation

extension Date: @retroactive Identifiable {
    public var id: TimeInterval { timeIntervalSince1970 }
}
```

Now another module can do this:

```swift
import AnalyticsSDK
import SwiftUI

let dates: [Date] = [Date()]

// This becomes possible because Date is now Identifiable
// in the importing program context.
List(dates) { date in
    Text(date.formatted())
}
```

That might look helpful, but the SDK has now made a UI identity decision for every client. If a client expected a different identity rule, or if Foundation later provides a different rule, the SDK has polluted the client’s model.

Interview version:

> An extension-based conformance affects other code because Swift conformances are global semantic facts, not file-local helpers. Importing a module can make new generic constraints succeed, change overload choices, and expose new serialization or identity behavior. That is useful when the conformance owner owns the semantic decision, but dangerous when a dependency silently makes a modeling choice for types it does not own.

---

## 5. Code probe

The rubric does not include a code probe for A10, so use this Swift 6.x probe instead.

Given:

```swift
import Foundation

extension Date: Identifiable {
    public var id: TimeInterval { timeIntervalSince1970 }
}
```

### What happens?

Using Swift 6.2.1, this produces a warning. Exact output from `swiftc -swift-version 6` on Linux:

```text
retro.swift:3:1: warning: extension declares a conformance of imported type 'Date' to imported protocol 'Identifiable'; this will not behave correctly if the owners of 'FoundationEssentials' introduce this conformance in the future
1 | import Foundation
2 | 
3 | extension Date: Identifiable {
  | |- warning: extension declares a conformance of imported type 'Date' to imported protocol 'Identifiable'; this will not behave correctly if the owners of 'FoundationEssentials' introduce this conformance in the future
  | `- note: add '@retroactive' to silence this warning
4 |     public var id: TimeInterval { timeIntervalSince1970 }
5 | }
```

On Apple platforms, the module name shown in the diagnostic may differ, but the semantic issue is the same.

### Why?

```text
Date          -> external type
Identifiable  -> external protocol
Your module   -> declares the conformance

Result:
Your module declares a semantic relationship it does not own.
```

SE-0364 says this warning is emitted when the type being extended and the protocol being conformed to are both declared in modules different from the extension’s module. It also says conformances of external types to protocols defined in the current module, and extensions of external types that do not introduce conformances, are still valid and allowed. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md "swift-evolution/proposals/0364-retroactive-conformance-warning.md at main · swiftlang/swift-evolution · GitHub"))

### Fix or redesign

Preferred library-safe design:

```swift
import Foundation

public struct IdentifiedDate: Identifiable, Hashable, Codable, Sendable {
    public let date: Date

    public var id: TimeInterval {
        date.timeIntervalSince1970
    }

    public init(_ date: Date) {
        self.date = date
    }
}
```

Usage:

```swift
let dates: [IdentifiedDate] = rawDates.map(IdentifiedDate.init)
```

### Why this fix is correct

The wrapper makes the semantic decision explicit:

```text
Date itself is not globally Identifiable.
IdentifiedDate is your domain's identified representation of Date.
```

You own `IdentifiedDate`, so you own its `Identifiable`, `Hashable`, `Codable`, and `Sendable` semantics.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Wrapper type|Public libraries, SDKs, reusable packages|Slight call-site overhead, but safest ownership model|
|DTO type|Serialization boundaries, API payloads, persistence|Requires mapping layer|
|Own protocol|Domain-specific adaptation of external types|Clients must use your protocol, not standard-library generic APIs|
|`@retroactive` conformance|App target or unavoidable compatibility bridge|Acknowledges risk; does not make conformance safe or private|
|Fully qualified conformance|Source compatibility across compiler versions|Still retroactive; only silences/avoids syntax dependency|

---

## 6. Exercise

### Problem

Review a package that adds `Codable` to an external type. Explain the compatibility and ownership risks.

### Bad / naive version

Assume an external package defines this:

```swift
// Module: MapVendor

public struct Coordinate {
    public let latitude: Double
    public let longitude: Double
}
```

Your SDK does this:

```swift
// Module: TrackingSDK

import MapVendor

extension Coordinate: Codable {}
```

### What is wrong?

```text
1. TrackingSDK does not own Coordinate.
2. TrackingSDK does not own Codable.
3. The conformance is globally visible to clients importing TrackingSDK.
4. MapVendor might add its own Codable conformance later.
5. Another dependency might also add Coordinate: Codable.
6. The serialized schema is now chosen by a non-owner.
7. If clients persist this format, changing it becomes a compatibility problem.
```

The most dangerous part is schema ownership. `Codable` is not just a compiler convenience; it defines an encoding and decoding contract. If `TrackingSDK` serializes `Coordinate` as:

```json
{
  "latitude": 52.52,
  "longitude": 13.405
}
```

but `MapVendor` later decides the official schema should be:

```json
{
  "lat": 52.52,
  "lng": 13.405
}
```

then clients can face migration bugs, decoding failures, or inconsistent persisted data.

### Improved version

Use a DTO owned by your module:

```swift
import MapVendor

public struct CoordinateDTO: Codable, Hashable, Sendable {
    public let latitude: Double
    public let longitude: Double

    public init(latitude: Double, longitude: Double) {
        self.latitude = latitude
        self.longitude = longitude
    }

    public init(_ coordinate: Coordinate) {
        self.latitude = coordinate.latitude
        self.longitude = coordinate.longitude
    }

    public var vendorCoordinate: Coordinate {
        Coordinate(latitude: latitude, longitude: longitude)
    }
}
```

If you want schema evolution:

```swift
public struct CoordinateDTO: Codable, Hashable, Sendable {
    public let latitude: Double
    public let longitude: Double

    private enum CodingKeys: String, CodingKey {
        case latitude
        case longitude
        case lat
        case lng
    }

    public init(latitude: Double, longitude: Double) {
        self.latitude = latitude
        self.longitude = longitude
    }

    public init(_ coordinate: Coordinate) {
        self.latitude = coordinate.latitude
        self.longitude = coordinate.longitude
    }

    public init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)

        if let latitude = try container.decodeIfPresent(Double.self, forKey: .latitude),
           let longitude = try container.decodeIfPresent(Double.self, forKey: .longitude) {
            self.latitude = latitude
            self.longitude = longitude
            return
        }

        self.latitude = try container.decode(Double.self, forKey: .lat)
        self.longitude = try container.decode(Double.self, forKey: .lng)
    }

    public func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(latitude, forKey: .latitude)
        try container.encode(longitude, forKey: .longitude)
    }

    public var vendorCoordinate: Coordinate {
        Coordinate(latitude: latitude, longitude: longitude)
    }
}
```

### Why this is better

The DTO makes ownership clear:

```text
MapVendor owns Coordinate.
Swift owns Codable.
TrackingSDK owns CoordinateDTO.
TrackingSDK owns CoordinateDTO's serialized schema.
```

This is safer for public packages because future changes to `MapVendor.Coordinate` do not automatically break your persisted schema. You can also version, migrate, deprecate, and document the DTO independently.

---

## 7. Production guidance

Use extensions in production when:

```text
You own the type and want to group behavior by responsibility.
You want to separate protocol conformances into clear sections.
You want to add domain-specific helpers to types in the same module.
You own the protocol and intentionally adapt external types to your abstraction.
You are creating constrained generic helpers, such as Collection helpers.
```

Be careful when:

```text
The extension is public.
The extension is in a reusable library.
The extension adds conformance to a type you do not own.
The extension adds conformance to a protocol you do not own.
The conformance affects identity, equality, hashing, serialization, ordering, sendability, or UI diffing.
The extension lives in a dependency imported by many targets.
```

Avoid when:

```text
You own neither the type nor the protocol and this is a public package.
The conformance is merely for convenience.
The conformance defines persistence or wire-format semantics for an external type.
The extension hides important behavior far away from the type.
The extension makes a general-purpose type carry feature-specific behavior.
```

Debugging checklist:

```text
Who owns the type?
Who owns the protocol?
Is this conformance visible outside the module?
Can an upstream library add this conformance later?
Could another dependency add the same conformance?
Does this conformance affect Codable, Hashable, Equatable, Identifiable, Sendable, Comparable, or Collection behavior?
Does importing this module change overload resolution or generic API availability?
Is the extension in an app target or a reusable package?
Would a wrapper or DTO make the semantic ownership clearer?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Extensions let you add methods and protocol conformances to existing types.

This is true but incomplete.

### Senior answer

> Extensions are useful for organizing behavior and separating conformances, but a protocol conformance added in an extension is still a real conformance of the type. I avoid retroactively conforming external types to external protocols, especially in libraries, because it can collide with future conformances and create semantic ambiguity.

### Staff-level answer

> I treat extension-based conformances as API and ownership decisions, not syntax. If my module owns the type or the protocol, extensions are a clean way to organize behavior. If I own neither, I assume the conformance is a compatibility liability. For public packages, I prefer wrappers, DTOs, adapter types, or owned protocols. I only use `@retroactive` when the blast radius is contained, the risk is documented, and there is a migration plan if the upstream owner later adds the conformance.

Staff-level questions to ask:

```text
Is this conformance part of the public API surface?
Would importing this package change behavior in unrelated client code?
Does this conformance define identity, equality, hashing, serialization, or concurrency safety?
What happens if the type owner adds the same conformance next year?
Can we model this with a wrapper, DTO, or owned protocol instead?
```

---

## 9. Interview-ready summary

Extensions are a powerful organization and modeling tool in Swift. They let you add behavior and protocol conformances outside the original type declaration, which is clean when you own the semantic decision. The main caveat is retroactive conformance: if you conform an external type to an external protocol, that conformance becomes globally visible to clients that import your module and can collide with future conformances from the owning modules. In production, especially in libraries, I avoid using extensions to make ownership claims I do not own; I prefer wrappers, DTOs, or owned protocols unless the retroactive conformance is explicitly justified and contained.

---

## 10. Flashcards

Q: What is the difference between adding a method in an extension and adding a protocol conformance in an extension?  
A: A method adds callable behavior. A protocol conformance makes a global semantic claim that the type satisfies a protocol, affecting generic constraints, overloads, runtime conformance lookup, and client behavior.

Q: When is extension-based conformance usually safe?  
A: When you own the type, or when you own the protocol and intentionally adapt an external type to your domain.

Q: What is the dangerous retroactive conformance case?  
A: Conforming an external type to an external protocol, such as `extension Date: Identifiable`.

Q: Why is adding `Codable` to an external type risky?  
A: It defines serialization semantics for a type you do not own, can conflict with future owner-provided conformances, and can break persisted or wire-format compatibility.

Q: What does `@retroactive` do?  
A: It acknowledges and silences Swift’s retroactive conformance warning. It does not make the conformance private or remove future collision risk.

Q: What is the safest alternative to retroactively adding `Codable` to an external type?  
A: Define an owned DTO or wrapper type that conforms to `Codable`, and map to/from the external type.

---

## 11. Related sections

- [[A11 — Access control]]
- [[B4 — Dispatch model: static, class virtual, witness-table, and protocol extension dispatch]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[B9 — Codable and schema evolution]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[E6 — Import visibility and dependency leakage]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — A10 rubric entry.
- Swift.org Documentation — Extensions. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/?utm_source=chatgpt.com "Extensions - Documentation | Swift.org"))
- Swift.org Documentation — Protocols: adding protocol conformance with an extension. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/?utm_source=chatgpt.com "Protocols | Documentation - Swift Programming Language"))
- Swift.org Documentation — Access Control: protocol-conformance extensions and access-level rules. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/?utm_source=chatgpt.com "Access Control | Documentation - Swift Programming Language"))
- Swift Evolution SE-0364 — Warning for Retroactive Conformances of External Types. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md "swift-evolution/proposals/0364-retroactive-conformance-warning.md at main · swiftlang/swift-evolution · GitHub"))