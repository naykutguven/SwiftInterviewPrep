---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - access-control
---
# A11 — Access Control

Tags: #swift #ios #interview-prep #core-semantics #access-control

## 0. Rubric snapshot

**Rubric expectation**

Understand `private`, `fileprivate`, `internal`, `package`, `public`, and `open`. The A11 rubric specifically probes the difference between `public` and `open`, why `package` matters in multi-target Swift packages, and how to choose minimum access levels across app, core, and test-support targets.

**Caveats**

`public` does **not** mean overridable outside the defining module. `open` has real library-evolution cost. Overexposing symbols turns implementation details into API promises.

**You should be able to answer**

- What is the practical difference between `public` and `open`?
- Why is `package` useful in multi-target Swift packages?

**You should be able to do**

- Given a package with app, core, and test-support targets, choose the minimum access level for several symbols and justify each.

---

## 1. Core mental model

Access control is Swift’s compile-time mechanism for controlling **who is allowed to name, call, instantiate, subclass, override, or mutate** a declaration. It is not primarily about security. It is about API boundaries, encapsulation, build boundaries, and future change. Swift’s official documentation describes access control as a way to restrict access from other source files and modules, hide implementation details, and define a preferred interface. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/?utm_source=chatgpt.com "Access Control | Documentation - Swift Programming Language"))

The important boundaries are:

```text
scope < file < module < package < external clients
```

A **module** is usually a target: an app target, framework target, SwiftPM target, or test target. A **package** can contain multiple modules/targets. This is why `internal` and `package` are not the same: `internal` crosses files inside one module; `package` crosses modules inside one package. SE-0386 introduced `package` because package authors often had to make helper APIs `public` just so sibling targets could use them, accidentally exposing those helpers to external clients. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0386-package-access-modifier.md "swift-evolution/proposals/0386-package-access-modifier.md at main · swiftlang/swift-evolution · GitHub"))

The access-control question is not “can I make this compile?” The senior/staff-level question is:

```text
Who needs this symbol, and what future changes am I willing to make harder?
```

The broader a symbol’s access level, the more people can depend on it. Once external clients depend on it, renaming it, changing its signature, changing subclassing behavior, or removing it becomes a source-compatibility decision. For SDKs, public access is product surface area.

The key idea:

```text
Access control is API debt control: expose the smallest surface that lets the architecture work.
```

---

## 2. Essential mechanics

### Access levels, from narrowest to broadest

```swift
private      // Same declaration / lexical scope, plus same-file extensions of that declaration
fileprivate  // Same source file
internal     // Same module; default
package      // Same package, including sibling package targets/modules
public       // Any importing module can use it
open         // Any importing module can use it AND subclass/override it
```

`open` is only relevant to classes and overridable class members. Structs, enums, actors, protocols, functions, and variables top out at `public`.

```swift
public struct User {
    public let id: UUID

    public init(id: UUID) {
        self.id = id
    }
}

// Invalid:
// open struct User {}
```

For ordinary app code, `internal` is the default and is often enough. For reusable modules, `public` and `package` become design tools.

---

### `public` vs `open`

`public` means “usable outside this module.” It does **not** mean “subclassable or overridable outside this module.”

```swift
// SDK module

public class ImageLoader {
    public init() {}

    public func load() {
        // usable by clients
    }
}
```

External clients can instantiate and call this class, but they cannot subclass it outside the module.

```swift
// App module

import SDK

let loader = ImageLoader()
loader.load()

final class CustomLoader: ImageLoader {
    // Error:
    // cannot inherit from non-open class 'ImageLoader'
    // outside of its defining module
}
```

To allow external subclassing, the class itself must be `open`.

```swift
// SDK module

open class ImageLoader {
    public init() {}

    public func load() {
        // usable, but not overridable outside the SDK module
    }

    open func makeRequest() {
        // usable and overridable outside the SDK module
    }
}
```

A class can be `open` while some of its methods remain merely `public`. This is common and useful: you may want clients to subclass a type but only override specific customization points. SE-0117 introduced `open` to distinguish public use from public overridability. ([GitHub](https://github.com/apple/swift-evolution/blob/master/proposals/0117-non-public-subclassable-by-default.md?utm_source=chatgpt.com "swift-evolution/proposals/0117-non-public-subclassable-by ..."))

---

### `package` solves the “public helper API” problem

Before `package`, multi-target Swift packages had an awkward choice:

```text
internal  -> too narrow: sibling targets cannot use it
public    -> too broad: external clients can use it
```

`package` fills that gap.

```swift
// Package: NetworkingKit
// Target: NetworkingCore

package struct RequestSigner {
    package func sign(_ request: URLRequest) -> URLRequest {
        request
    }
}
```

```swift
// Same package
// Target: NetworkingFeature

import NetworkingCore

func buildRequest() {
    let signer = RequestSigner() // OK: same package
}
```

```swift
// External app package

import NetworkingCore

let signer = RequestSigner()
// Error: unavailable outside the package
```

SE-0386 defines `package` as accessible from outside the defining module, but only from other modules in the same package. It also places `package` below `public`/`open` and above `internal`/`fileprivate`/`private` for signature accessibility rules. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0386-package-access-modifier.md "swift-evolution/proposals/0386-package-access-modifier.md at main · swiftlang/swift-evolution · GitHub"))

---

### Access level of a declaration constrains its signature

A declaration cannot be more visible than the types in its public-facing signature.

Bad:

```swift
package struct CacheKey {
    package let rawValue: String
}

public func lookup(_ key: CacheKey) -> String? {
    nil
}
```

Swift 6.2 compiler error:

```text
error: function cannot be declared public because its parameter uses a package type
```

Why? A `public` function is visible to external clients, but external clients cannot name `CacheKey`. That would create an unusable API.

Better:

```swift
package struct CacheKey {
    package let rawValue: String
}

package func lookup(_ key: CacheKey) -> String? {
    nil
}
```

Or redesign the public API so it uses public types:

```swift
public func lookup(rawKey: String) -> String? {
    nil
}
```

---

### Setter access can be narrower than getter access

A common Swift API pattern is `public private(set)` or `public internal(set)`.

```swift
public struct DownloadState {
    public private(set) var progress: Double = 0

    package mutating func updateProgress(_ progress: Double) {
        self.progress = progress
    }
}
```

External clients can read `progress`, but only the defining scope can mutate it through the property setter. Package-internal code can mutate it through an intentional API.

This is useful when clients need observation but not arbitrary mutation.

---

## 3. Common traps and misconceptions

### Trap 1: Treating `public` as “subclassable”

Bad:

```swift
public class AnalyticsProvider {
    public init() {}

    public func track(_ event: String) {}
}
```

A client can call it, but cannot subclass it outside the defining module.

Better, if subclassing is intentionally supported:

```swift
open class AnalyticsProvider {
    public init() {}

    public func track(_ event: String) {
        prepare(event)
    }

    open func prepare(_ event: String) {
        // documented customization point
    }
}
```

But do not make it `open` casually. `open` means you are designing an inheritance contract.

---

### Trap 2: Making test seams `public`

Bad:

```swift
public final class MockPaymentClient: PaymentClient {
    public init() {}
}
```

If this is only for tests, it is not product API. Making it `public` lets real clients depend on it.

Better:

```swift
// Target: PaymentTestSupport

package final class MockPaymentClient: PaymentClient {
    package init() {}
}
```

Or, if tests live in a separate package and genuinely need it, put it in a clearly named test-support product. Do not smuggle it into the production API.

---

### Trap 3: Using `fileprivate` as a dumping ground

Bad:

```swift
// Huge file with unrelated types just so they can share fileprivate state.

fileprivate final class TokenStorage {
    var token: String?
}

fileprivate final class LoginCoordinator {
    let storage: TokenStorage

    init(storage: TokenStorage) {
        self.storage = storage
    }
}
```

Better:

```swift
final class TokenStorage {
    private var token: String?

    func read() -> String? {
        token
    }

    func write(_ token: String?) {
        self.token = token
    }
}
```

`fileprivate` is useful when multiple declarations in the same file are intentionally co-designed. If the file becomes an access-control workaround, the boundary is probably wrong.

---

### Trap 4: Letting implementation dependencies leak into public API

Bad:

```swift
import SomeInternalJSONLibrary

public struct UserPayload {
    public let raw: SomeInternalJSONLibrary.JSONValue
}
```

Now the dependency is part of your public API. Removing or replacing that library becomes a breaking change.

Better:

```swift
public struct UserPayload {
    public let id: String
    public let name: String
}
```

Keep implementation dependencies behind `internal` or `package` boundaries.

---

## 4. Direct answers to rubric questions

### Q1. What is the practical difference between `public` and `open`?

`public` allows external modules to use a declaration. For classes and class members, `open` additionally allows external modules to subclass the class or override the member.

```swift
public class PublicBase {
    public init() {}
    public func render() {}
}

open class OpenBase {
    public init() {}
    public func render() {}
    open func customize() {}
}
```

From another module:

```swift
final class BadSubclass: PublicBase {}
```

Swift 6.2 compiler error:

```text
error: cannot inherit from non-open class 'PublicBase' outside of its defining module
```

And:

```swift
final class Plugin: OpenBase {
    override func render() {}
    override func customize() {}
}
```

Swift 6.2 compiler error for `render()`:

```text
error: overriding non-open instance method outside of its defining module
```

`customize()` is allowed because it is `open`.

Interview version:

> `public` is for external use; `open` is for external use plus external subclassing or overriding. I treat `open` as a much stronger promise than `public`, because it means clients can build behavior on top of my inheritance points. For SDK code, I only use `open` when I have designed and documented the subclassing contract. Otherwise I prefer `public final`, `public` value types, protocols, or composition.

---

### Q2. Why is `package` useful in multi-target Swift packages?

`package` lets sibling modules inside the same Swift package share implementation APIs without exposing those APIs to external clients.

Without `package`, developers often make helper declarations `public` because `internal` is only module-wide. That creates accidental public API. SE-0386’s motivation explicitly describes this problem: APIs needed by another module in the same package had to be public, which allowed clients outside the package to depend on APIs that were never meant for them. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0386-package-access-modifier.md "swift-evolution/proposals/0386-package-access-modifier.md at main · swiftlang/swift-evolution · GitHub"))

Example:

```swift
// Target: Core

package struct FeatureFlagStore {
    package func isEnabled(_ key: String) -> Bool {
        false
    }
}
```

```swift
// Target: AppFeature, same package

import Core

let flags = FeatureFlagStore() // OK
```

```swift
// External package

import Core

let flags = FeatureFlagStore() // Not visible
```

Interview version:

> `package` is the access level I use when a symbol is shared across targets in the same Swift package but should not become public API. It is especially useful for packages split into Core, UI, Networking, and TestSupport targets. It keeps architecture flexible without forcing helper APIs into the external surface area.

---

## 5. Code probe / examples

The A11 rubric has no dedicated code probe, so use these three probes instead.

### Minimal example: ordinary access boundaries

```swift
public struct User {
    public let id: UUID
    internal let databaseID: Int
    private let checksum: String

    public init(id: UUID) {
        self.id = id
        self.databaseID = 0
        self.checksum = ""
    }
}
```

What external clients see:

```swift
let user = User(id: UUID())
print(user.id)          // OK
print(user.databaseID)  // Error: internal
print(user.checksum)    // Error: private
```

Why:

```text
public id         -> external API
internal databaseID -> same module only
private checksum  -> implementation detail of User
```

---

### Counterexample: invalid public API using a package type

Given:

```swift
package struct CacheKey {
    package let rawValue: String
}

public func lookup(_ key: CacheKey) -> String? {
    nil
}
```

Swift 6.2 compiler error:

```text
error: function cannot be declared public because its parameter uses a package type
```

Why:

```text
public function
└── parameter type: package CacheKey

External clients can see lookup(...)
External clients cannot name CacheKey
=> unusable public API
```

Fix:

```swift
package struct CacheKey {
    package let rawValue: String
}

package func lookup(_ key: CacheKey) -> String? {
    nil
}
```

Or expose a public type:

```swift
public struct PublicCacheKey {
    public let rawValue: String

    public init(rawValue: String) {
        self.rawValue = rawValue
    }
}

public func lookup(_ key: PublicCacheKey) -> String? {
    nil
}
```

---

### Production example: public facade, package internals

```swift
// Target: AnalyticsCore

public struct AnalyticsEvent: Sendable {
    public let name: String
    public let properties: [String: String]

    public init(name: String, properties: [String: String] = [:]) {
        self.name = name
        self.properties = properties
    }
}

public protocol AnalyticsClient: Sendable {
    func track(_ event: AnalyticsEvent) async
}

package struct EventBatcher {
    package func batch(_ events: [AnalyticsEvent]) -> [[AnalyticsEvent]] {
        events.chunked(into: 50)
    }
}

package struct AnalyticsEndpoint {
    package let url: URL
}
```

```swift
// Target: AnalyticsNetworking, same package

import AnalyticsCore

package final class HTTPAnalyticsClient: AnalyticsClient {
    private let endpoint: AnalyticsEndpoint
    private let batcher: EventBatcher

    package init(endpoint: AnalyticsEndpoint, batcher: EventBatcher) {
        self.endpoint = endpoint
        self.batcher = batcher
    }

    public func track(_ event: AnalyticsEvent) async {
        // implementation
    }
}
```

The package exposes:

```text
public:
- AnalyticsEvent
- AnalyticsClient

package:
- EventBatcher
- AnalyticsEndpoint
- HTTPAnalyticsClient
```

This gives clients a stable API while allowing sibling targets to collaborate internally.

---

## 6. Exercise

### Problem

Given a package with app, core, and test-support targets, choose the minimum access level for several symbols and justify each.

Assume this structure:

```text
Package: ShoppingAppPackage

Targets:
- ShoppingApp          // executable/app target
- ShoppingCore         // domain, networking abstractions, use cases
- ShoppingTestSupport  // mocks, fixtures, builders for tests
- ShoppingCoreTests    // tests
```

### Bad / naive version

```swift
// Target: ShoppingCore

public struct CartItem: Equatable {
    public let id: UUID
    public let name: String
    public let price: Decimal
}

public final class CartRepository {
    public init() {}

    public func loadCart() async throws -> [CartItem] {
        []
    }
}

public struct CartDatabaseRow {
    public let id: String
    public let payload: Data
}

public final class MockCartRepository: CartRepository {
    public var items: [CartItem] = []

    public override func loadCart() async throws -> [CartItem] {
        items
    }
}
```

### What is wrong?

```text
CartItem may reasonably be public if it is product API.

CartRepository being a public concrete class forces clients to depend on construction and subclassing behavior.

CartDatabaseRow is persistence implementation detail but public.

MockCartRepository is test infrastructure but public.

The design exposes too much, makes future refactors harder, and confuses production API with test support.
```

### Improved version

```swift
// Target: ShoppingCore

public struct CartItem: Equatable, Sendable {
    public let id: UUID
    public let name: String
    public let price: Decimal

    public init(id: UUID, name: String, price: Decimal) {
        self.id = id
        self.name = name
        self.price = price
    }
}

public protocol CartRepository: Sendable {
    func loadCart() async throws -> [CartItem]
}

package struct CartDatabaseRow: Sendable {
    package let id: String
    package let payload: Data
}

package final class SQLiteCartRepository: CartRepository {
    private let database: CartDatabase

    package init(database: CartDatabase) {
        self.database = database
    }

    public func loadCart() async throws -> [CartItem] {
        []
    }
}

private struct CartDatabase {
    func queryRows() throws -> [CartDatabaseRow] {
        []
    }
}
```

```swift
// Target: ShoppingTestSupport

import ShoppingCore

package final class MockCartRepository: CartRepository, @unchecked Sendable {
    package var result: Result<[CartItem], Error>

    package init(result: Result<[CartItem], Error> = .success([])) {
        self.result = result
    }

    public func loadCart() async throws -> [CartItem] {
        try result.get()
    }
}
```

### Access-level choices

|Symbol|Target|Access level|Justification|
|---|--:|--:|---|
|`CartItem`|`ShoppingCore`|`public`|It is part of the app/package’s intentional API surface.|
|`CartRepository` protocol|`ShoppingCore`|`public`|Feature modules and clients can depend on the abstraction.|
|`SQLiteCartRepository`|`ShoppingCore`|`package`|Sibling targets may assemble it; external clients should not depend on the concrete implementation.|
|`CartDatabaseRow`|`ShoppingCore`|`package` or `internal`|Use `package` only if another target needs it; otherwise `internal`.|
|`CartDatabase`|`ShoppingCore`|`private` or `internal`|Pure implementation detail.|
|`MockCartRepository`|`ShoppingTestSupport`|`package`|Shared by tests inside the package but not production API.|
|Test fixture builders|`ShoppingTestSupport`|`package`|Useful across test targets, not external API.|
|App coordinator|`ShoppingApp`|`internal`|App target implementation detail.|
|View-private state|`ShoppingApp`|`private`|Only the enclosing type should mutate it.|

### Why this is better

This design exposes the model and abstraction that clients need, keeps persistence details hidden, and prevents test infrastructure from becoming production API. It also gives you freedom to replace SQLite with another storage implementation without breaking external callers.

---

## 7. Production guidance

Use this in production when:

```text
private:
- Stored properties that should only be mutated through methods.
- Helper methods used only by one type.
- Invariants that must not be bypassed.

fileprivate:
- A small cluster of tightly related declarations in one file.
- Private helpers shared between a type and same-file helper extensions.

internal:
- Most app-target code.
- Feature implementation that only one target/module needs.

package:
- Shared internals across SwiftPM targets.
- Test support used by multiple test targets in the same package.
- Internal adapters, concrete implementations, factories, and utilities shared across package modules.

public:
- Stable API meant for external modules.
- Public structs/enums/protocols/functions in an SDK.
- App-facing API from a framework target.

open:
- Explicit subclassing/overriding extension points in an SDK.
- Framework base classes with documented override contracts.
```

Be careful when:

```text
- A public API mentions a third-party dependency type.
- A public class is not final.
- A public initializer makes construction promises you may later regret.
- A package symbol leaks into a public signature.
- Tests require public access to production internals.
- You use open without documenting subclassing rules.
```

Avoid when:

```text
- Making something public just because another target needs it: use package.
- Making something open just because public did not allow subclassing.
- Using fileprivate to compensate for oversized files.
- Using @testable import as the main design strategy for testability.
- Exposing mocks, fixtures, or debug helpers as public API.
```

Debugging checklist:

```text
Can the caller and callee see each other’s module?
Is the symbol hidden by private, fileprivate, internal, package, public, or open?
Is the declaration’s signature using less-visible types?
Is the class public but not open?
Is the method public but not open?
Is the initializer visible enough?
Is the symbol in a sibling target or a different package?
Is a test relying on @testable import instead of a real seam?
Did a public API accidentally expose an implementation dependency?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `private` hides things, `public` exposes things, and `open` lets classes be subclassed.

### Senior answer

> Access control is how I preserve encapsulation and reduce API surface. I default to `internal`, use `private` for invariants, use `package` for shared internals across package targets, use `public` for intentional external API, and reserve `open` for designed subclassing points.

### Staff-level answer

> Access control is an architecture and evolution tool. I design module/package boundaries so implementation details stay movable, test support does not become product API, and external clients only see stable contracts. I avoid `open` unless inheritance is a documented extension mechanism, because overriding points become long-term compatibility obligations. For multi-target packages, I use `package` to prevent helper APIs from leaking into the public surface while still keeping targets properly factored.

Staff-level questions to ask:

```text
Is this symbol part of the product contract or only an implementation detail?
Will clients need to name this type in their code?
Does this API expose a third-party dependency?
Can this be package instead of public?
Can this be public final instead of open?
Is subclassing actually the right extension mechanism?
Would changing this symbol require a major version bump?
Are tests forcing us to weaken access control?
Should this target boundary exist, or are we compensating with access modifiers?
```

---

## 9. Interview-ready summary

Swift access control is about preserving API boundaries and future flexibility. `private` is scoped to the declaration, `fileprivate` to the file, `internal` to the module, `package` to modules in the same package, `public` to external clients, and `open` to external clients plus subclassing/overriding. The biggest senior-level distinction is that `public` does not imply overridable; `open` is an explicit inheritance contract. In production, I expose the minimum access needed: `internal` by default, `private` for invariants, `package` for shared package internals, `public` for stable API, and `open` only for documented subclassing points.

---

## 10. Flashcards

Q: What is Swift’s default access level?

A: `internal`.

Q: What is the practical difference between `public` and `open`?

A: `public` allows use from external modules. `open` also allows external subclassing of classes and overriding of class members.

Q: Can a `public` class be subclassed outside its defining module?

A: No. It must be `open`.

Q: Can a `public` method be overridden outside its defining module?

A: No. It must be `open`, and the class must support external subclassing.

Q: What problem does `package` solve?

A: It lets multiple modules in the same Swift package share APIs without exposing those APIs to external clients.

Q: Why can’t a `public` function accept a `package` type as a parameter?

A: External clients could see the function but could not name the parameter type, so the public API would be unusable.

Q: When should you use `fileprivate`?

A: When multiple declarations in the same file are intentionally co-designed and need shared implementation access.

Q: Why is `open` expensive for library evolution?

A: It lets external clients override or subclass your implementation, so future internal changes can break client behavior.

Q: What is a common access-control smell in tests?

A: Making production internals `public` just so tests can reach them.

Q: What is the preferred access level for shared test fixtures inside a Swift package?

A: Usually `package`, if the fixtures are shared across package test targets but should not be public API.

---

## 11. Related sections

- [[A10 — Extensions and retroactive modeling]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E6 — Import visibility and dependency leakage]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — A11 Access Control.
- Swift Documentation — Access Control. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/?utm_source=chatgpt.com "Access Control | Documentation - Swift Programming Language"))
- Swift Evolution SE-0386 — `package` Access Modifier. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0386-package-access-modifier.md "swift-evolution/proposals/0386-package-access-modifier.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Evolution SE-0117 — Distinguishing Public Access and Public Overridability. ([GitHub](https://github.com/apple/swift-evolution/blob/master/proposals/0117-non-public-subclassable-by-default.md?utm_source=chatgpt.com "swift-evolution/proposals/0117-non-public-subclassable-by ..."))