---
tags:
  - swift
  - ios
  - interview-prep
  - interop-modules-distribution
  - import-visibility
  - dependency-leakage
---
## 0. Rubric snapshot

**Rubric expectation**

Understand access-controlled imports and the general problem of leaking implementation-only dependencies into public APIs. The rubric calls out E6 as a Tier 2 “Strong Senior / Emerging Staff Depth” topic.

**Caveats**

Public API surfaces that mention dependency types effectively lock in that dependency.

**You should be able to answer**

- Why is import visibility part of API design and not just a build detail?
- What happens if a public type signature exposes a type from an implementation dependency?

**You should be able to do**

- Review a public SDK API that returns a concrete type from a third-party library.
- Redesign it to avoid dependency leakage.

This follows the attached section-answer structure: rubric snapshot, mental model, mechanics, traps, direct answers, code probe, exercise, production guidance, senior/staff signal, summary, flashcards, related sections, and sources.

---

## 1. Core mental model

An `import` is not just “make names available in this file.” In a library or SDK, an import can become part of the module’s public contract if public declarations mention types from that imported module.

Swift 6 introduced access-level modifiers on imports through SE-0409. The feature lets you say whether an imported dependency is visible to all clients, only the package, only the module, or only the source file. SE-0409 says this is meant to enforce which declarations can reference an imported module, hide implementation details, and manage dependency creep. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))

The key idea:

```text
A dependency becomes part of your API the moment its types appear in your public signatures.
```

For example:

```swift
public import Alamofire

public protocol ImageService {
    func requestImage(_ url: URL) -> DataRequest
}
```

This is not merely an implementation detail anymore. `Alamofire.DataRequest` is now part of your SDK’s public API. Clients can depend on it, test against it, extend around it, and become coupled to it. Removing Alamofire later is no longer an internal refactor; it is a source-breaking API change.

Access-controlled imports let the compiler help you enforce your architectural intent:

```swift
internal import Alamofire

public protocol ImageService {
    func imageData(from url: URL) async throws -> Data
}

internal struct AlamofireImageService: ImageService {
    func imageData(from url: URL) async throws -> Data {
        // Use Alamofire internally.
    }
}
```

Now Alamofire remains an implementation choice. Your clients see your domain API, not your transport implementation.

Swift guarantees that declarations imported through a lower-visibility import cannot appear in more-visible declaration signatures. For example, an `internal import` cannot be used in a `public` method return type. SE-0409 extends normal access-control checking so imported declarations have an upper-bound visibility inside the file. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))

Swift does **not** guarantee that import visibility magically removes all runtime or linker dependency concerns. This is source/API visibility control, not a complete dependency-vendoring or binary-packaging model. SE-0409 explicitly describes ABI impact as a compile-time type-checking change. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))

---

## 2. Essential mechanics

### Mechanic 1: Imports have access levels in Swift 6

Swift supports access modifiers on imports:

```swift
public import PublicDependency
package import PackageDependency
internal import InternalDependency
fileprivate import FilePrivateDependency
private import PrivateDependency
```

SE-0409 lists `public`, `package`, `internal`, `fileprivate`, and `private` as valid import access levels; `open` is rejected on import declarations. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))

The visibility means:

|Import|Imported types can appear in signatures of…|
|---|---|
|`public import M`|public and lower declarations|
|`package import M`|package, internal, fileprivate, private declarations|
|`internal import M`|internal, fileprivate, private declarations|
|`fileprivate import M`|fileprivate, private declarations in the file|
|`private import M`|fileprivate/private-level use in the file|

Important nuance: lower-visibility imports can still be used in normal implementation bodies. The restriction is mainly about declaration signatures and inlinable code.

```swift
internal import ThirdPartyNetworking

public struct SDKClient {
    public init() {}

    public func loadProfile() async throws -> UserProfile {
        // OK: ThirdPartyNetworking is used inside the body.
        let response = try await ThirdPartyNetworkingClient().get("/profile")
        return UserProfile(response)
    }
}
```

Bad:

```swift
internal import ThirdPartyNetworking

public struct SDKClient {
    // Error: public API exposes an internal-imported type.
    public func rawRequest() -> ThirdPartyRequest {
        ThirdPartyRequest()
    }
}
```

---

### Mechanic 2: Unannotated imports are still public by default in Swift 6.x

In current Swift 6 language modes, plain `import Foo` preserves the older behavior and is treated as public by default. SE-0409 says this is for source compatibility; a future language mode is expected to make unannotated imports internal by default. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))

So this:

```swift
import ThirdParty
```

currently behaves like this in Swift 6.x:

```swift
public import ThirdParty
```

For library code, that means staff-level code should usually be explicit:

```swift
internal import Alamofire
internal import GRDB
public import Foundation
```

This makes the public dependency contract obvious during review.

---

### Mechanic 3: `@_implementationOnly` is legacy; prefer access-level imports

Before SE-0409, many Swift codebases used:

```swift
@_implementationOnly import SomeDependency
```

That underscored attribute is now deprecated in favor of access-level imports. Swift’s compiler diagnostic documentation says `@_implementationOnly` became deprecated when SE-0409 introduced support for access levels on import declarations. ([Swift Belgeleri](https://docs.swift.org/compiler/documentation/diagnostics/implementation-only-deprecated/?utm_source=chatgpt.com "Deprecated implementation-only imports ..."))

Modern replacement:

```swift
internal import SomeDependency
```

or, if the dependency is only needed in one file:

```swift
private import SomeDependency
```

---

### Mechanic 4: Public imports are not the same as re-exporting

`public import Foo` means declarations from `Foo` may be referenced in your public API.

It does **not** mean your clients can use `Foo` as if they had imported it directly in every context. That stronger “umbrella module” behavior historically comes from underscored re-export patterns such as `@_exported import`. SE-0409 says `@_exported` is a step above `public import` and is orthogonal to access-level restrictions. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))

Use `@_exported` very cautiously. It makes dependency creep worse unless you are intentionally building an umbrella module or preserving source compatibility after moving declarations between modules.

---

## 3. Common traps and misconceptions

### Trap 1: “The dependency is only used by my SDK, so it is internal”

Not if your public API mentions it.

Bad:

```swift
public import Lottie

public struct AnimationViewFactory {
    public init() {}

    public func makeLoadingAnimation() -> LottieAnimationView {
        LottieAnimationView(name: "loading")
    }
}
```

This exposes Lottie as part of your SDK contract.

Better:

```swift
internal import Lottie
public import UIKit

public struct LoadingAnimationView: UIViewRepresentable {
    public init() {}

    public func makeUIView(context: Context) -> UIView {
        LottieAnimationView(name: "loading")
    }

    public func updateUIView(_ uiView: UIView, context: Context) {}
}
```

Or for UIKit-only SDKs:

```swift
internal import Lottie
public import UIKit

public final class LoadingAnimationView: UIView {
    private let animationView = LottieAnimationView(name: "loading")

    public override init(frame: CGRect) {
        super.init(frame: frame)
        addSubview(animationView)
    }

    public required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
```

Now clients depend on your stable type, not Lottie’s type.

---

### Trap 2: Thinking `private import` makes a dependency disappear from build/linking

`private import` controls source-level visibility in that file. It does not mean the dependency is not linked, packaged, dynamically loaded, or required at runtime.

This matters for SDK distribution. You still need to answer:

```text
Is the dependency static or dynamic?
Is it embedded in the app?
Does the binary framework require the dependency at runtime?
Does the .swiftinterface expose the dependency?
Does the public API mention dependency symbols?
```

Import visibility helps with the public API and compiler interface. It is not a packaging strategy by itself.

---

### Trap 3: Returning third-party types because it is “more flexible”

Returning third-party types often feels flexible but is usually coupling disguised as flexibility.

Bad:

```swift
public import GRDB

public protocol UserStore {
    func userPublisher(id: User.ID) -> ValueObservation<User>
}
```

This locks the public abstraction to GRDB.

Better:

```swift
internal import GRDB

public protocol UserStore {
    func userStream(id: User.ID) -> AsyncThrowingStream<User, Error>
}
```

This exposes a Swift-native abstraction. Internally, GRDB can still drive the stream.

---

### Trap 4: Hiding a dependency too aggressively

Sometimes a dependency is legitimately part of the public model. For example, an SDK that extends SwiftUI may intentionally expose `SwiftUI.View`, `Binding`, `EnvironmentValues`, or `ViewModifier`.

This is fine:

```swift
public import SwiftUI

public struct ProfileBadge: View {
    public var body: some View {
        Text("Pro")
    }
}
```

The rule is not “never expose dependencies.” The rule is:

```text
Expose dependencies only when they are intentionally part of your public abstraction.
```

---

## 4. Direct answers to rubric questions

### Q1. Why is import visibility part of API design and not just a build detail?

Because imported types can appear in your public signatures, and public signatures define what clients can depend on. Once a dependency’s types appear in your public API, that dependency becomes part of your source-compatibility contract.

SE-0409 frames this as an API-design problem: library authors often intend some dependencies to be client-visible and others to remain implementation details; without import-level enforcement, it is easy to accidentally expose an implementation dependency in a public declaration. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))

Interview version:

> Import visibility is API design because imports control which external module types are allowed to appear in my public surface. If I return `Alamofire.DataRequest` from a public SDK method, Alamofire is no longer an implementation detail. Clients now compile against that choice. Access-level imports let me express and enforce whether a dependency is public, package-only, module-internal, or file-local.

---

### Q2. What happens if a public type signature exposes a type from an implementation dependency?

The dependency becomes part of your public API. Clients may need that dependency to compile, may write code against that type, and may become source-coupled to its semantics. Replacing the dependency later becomes a breaking API change.

Example:

```swift
public import Alamofire

public protocol APIClient {
    func perform(_ endpoint: Endpoint) -> DataRequest
}
```

Now `Alamofire.DataRequest` is part of your SDK contract.

If you later want to move to `URLSession`, you cannot simply change this:

```swift
public protocol APIClient {
    func perform(_ endpoint: Endpoint) async throws -> Data
}
```

That is a public API break.

Interview version:

> If a public signature exposes a type from an implementation dependency, that dependency is no longer private. It leaks into the client’s compile-time world and into the SDK’s semantic versioning contract. Removing or replacing it later is not an internal refactor; it is a source-breaking API change unless you preserve the old API.

---

## 5. Code probe

The rubric does not include a specific E6 code probe, so here is a minimal Swift 6.2 compiler probe.

Given module `ThirdParty`:

```swift
// ThirdParty.swift
public struct VendorResponse {
    public init() {}
}
```

Given module `SDK`:

```swift
// SDK.swift
internal import ThirdParty

public struct Client {
    public init() {}

    public func fetch() -> VendorResponse {
        VendorResponse()
    }
}
```

Compiled with Swift 6.2.1:

```text
Swift version 6.2.1 (swift-6.2.1-RELEASE)
Target: x86_64-unknown-linux-gnu
```

### What happens?

Exact compiler error:

```text
error: emit-module command failed with exit code 1 (use -v to see invocation)
SDK.swift:5:17: error: method cannot be declared public because its result uses an internal type
1 | internal import ThirdParty
  |          `- note: struct 'VendorResponse' imported as 'internal' from 'ThirdParty' here
2 | 
3 | public struct Client {
4 |     public init() {}
5 |     public func fetch() -> VendorResponse {
  |                 |- error: method cannot be declared public because its result uses an internal type
  |                 `- note: struct 'VendorResponse' is imported by this file as 'internal' from 'ThirdParty'
6 |         VendorResponse()
7 |     }

/tmp/.../ThirdParty.swift:1:15: note: type declared here
1 | public struct VendorResponse {
  |               `- note: type declared here
2 |     public init() {}
3 | }
```

### Why?

`VendorResponse` is public **inside the ThirdParty module**, but `SDK.swift` imported `ThirdParty` as `internal`.

Inside `SDK.swift`, the compiler treats `ThirdParty.VendorResponse` as having at most `internal` visibility. Therefore, this public method is illegal:

```swift
public func fetch() -> VendorResponse
```

because a public declaration cannot expose an internal type.

```text
ThirdParty.VendorResponse
        │
        │ imported through
        ▼
internal import ThirdParty
        │
        ▼
visible to SDK.swift as internal
        │
        ▼
cannot appear in public SDK API
```

This mirrors SE-0409’s rule: imported declarations get an upper-bound access level based on the import, and the compiler diagnoses more-visible declarations that reference less-visible imported types. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))

### Fix or redesign

Option A — dependency is intentionally public:

```swift
public import ThirdParty

public struct Client {
    public init() {}

    public func fetch() -> VendorResponse {
        VendorResponse()
    }
}
```

This compiles, but it intentionally exposes `ThirdParty`.

Option B — keep dependency internal and expose your own type:

```swift
internal import ThirdParty

public struct Response: Sendable {
    public let statusCode: Int
    public let body: Data

    public init(statusCode: Int, body: Data) {
        self.statusCode = statusCode
        self.body = body
    }
}

public struct Client {
    public init() {}

    public func fetch() -> Response {
        let vendor = VendorResponse()

        // Map vendor-specific response into SDK-owned model.
        return Response(statusCode: 200, body: Data())
    }
}
```

### Why this fix is correct

The second design preserves the dependency boundary:

```text
Public SDK API:
Client.fetch() -> Response

Implementation:
VendorResponse -> Response mapping

Client sees:
SDK.Response

Client does not see:
ThirdParty.VendorResponse
```

Now you can replace `ThirdParty` later without changing the public method signature.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`public import ThirdParty`|The dependency is intentionally part of your SDK’s public model.|Clients are coupled to that dependency.|
|SDK-owned DTOs|You want a stable public contract independent of implementation choices.|Requires mapping code and careful schema evolution.|
|Swift-native abstractions|You can express behavior with `AsyncSequence`, `Sendable`, protocols, closures, or value types.|May hide useful lower-level features of the dependency.|
|Adapter/proxy type|You need to expose capability without exposing the concrete dependency type.|Can become a leaky wrapper if it mirrors the dependency too closely.|
|Generic abstraction|You want clients to plug in their own implementations.|Can complicate API surface and type inference.|

---

## 6. Exercise

### Problem

Review a public SDK API that returns a concrete type from a third-party library; redesign it to avoid dependency leakage.

### Bad / naive version

```swift
public import Alamofire
public import Foundation

public struct SDKClient {
    public init() {}

    public func fetchUser(id: String) -> DataRequest {
        AF.request("https://api.example.com/users/\(id)")
    }
}
```

### What is wrong?

```text
1. DataRequest is an Alamofire type exposed in public SDK API.
2. Clients now compile against Alamofire.
3. Replacing Alamofire with URLSession becomes source-breaking.
4. Tests, mocks, and app architecture are forced to know about Alamofire.
5. The SDK is exporting transport implementation instead of domain behavior.
```

This is the central E6 smell: public API mentions an implementation dependency.

### Improved version

```swift
internal import Alamofire
public import Foundation

public struct User: Decodable, Sendable {
    public let id: String
    public let name: String

    public init(id: String, name: String) {
        self.id = id
        self.name = name
    }
}

public enum SDKError: Error, Sendable {
    case invalidResponse
    case transportFailed
    case decodingFailed
}

public struct SDKClient: Sendable {
    private let baseURL: URL

    public init(baseURL: URL) {
        self.baseURL = baseURL
    }

    public func fetchUser(id: String) async throws(SDKError) -> User {
        let url = baseURL.appending(path: "users/\(id)")

        do {
            let data = try await requestData(from: url)
            return try decodeUser(from: data)
        } catch let error as SDKError {
            throw error
        } catch {
            throw .transportFailed
        }
    }

    private func requestData(from url: URL) async throws -> Data {
        try await withCheckedThrowingContinuation { continuation in
            AF.request(url).responseData { response in
                switch response.result {
                case .success(let data):
                    continuation.resume(returning: data)
                case .failure:
                    continuation.resume(throwing: SDKError.transportFailed)
                }
            }
        }
    }

    private func decodeUser(from data: Data) throws(SDKError) -> User {
        do {
            return try JSONDecoder().decode(User.self, from: data)
        } catch {
            throw .decodingFailed
        }
    }
}
```

### Why this is better

The public API is now:

```swift
public func fetchUser(id: String) async throws(SDKError) -> User
```

That API expresses domain behavior: “fetch a user.” It does not expose “make an Alamofire request.”

The SDK owns the public model:

```swift
User
SDKError
SDKClient
```

Alamofire remains an internal implementation detail:

```swift
internal import Alamofire
```

Later, the implementation can move from Alamofire to `URLSession`:

```swift
private func requestData(from url: URL) async throws -> Data {
    let (data, _) = try await URLSession.shared.data(from: url)
    return data
}
```

No public API change required.

### Production example: adapter boundary

A production SDK often benefits from an internal adapter:

```swift
internal protocol HTTPTransport: Sendable {
    func data(from url: URL) async throws -> Data
}

internal struct AlamofireTransport: HTTPTransport {
    func data(from url: URL) async throws -> Data {
        try await withCheckedThrowingContinuation { continuation in
            AF.request(url).responseData { response in
                continuation.resume(with: response.result.mapError { $0 })
            }
        }
    }
}

public struct SDKClient: Sendable {
    private let transport: any HTTPTransport
    private let baseURL: URL

    internal init(baseURL: URL, transport: any HTTPTransport) {
        self.baseURL = baseURL
        self.transport = transport
    }

    public init(baseURL: URL) {
        self.init(baseURL: baseURL, transport: AlamofireTransport())
    }

    public func fetchUser(id: String) async throws -> User {
        let data = try await transport.data(from: baseURL.appending(path: "users/\(id)"))
        return try JSONDecoder().decode(User.self, from: data)
    }
}
```

Here, even the transport abstraction is internal. Public clients get a stable SDK surface; tests inside the package can inject fake transports.

---

## 7. Production guidance

Use this in production when:

```text
You are building SDKs, Swift packages, shared app modules, design-system modules, feature modules, or binary frameworks.
You want the compiler to enforce dependency boundaries.
You want to prevent implementation dependencies from appearing in public APIs.
You want smaller, cleaner module interfaces.
You want to preserve the ability to replace third-party libraries later.
```

Be careful when:

```text
A dependency type appears in a public protocol, public struct property, public enum associated value, public function parameter, public return type, public typealias, or @inlinable code.
A dependency seems harmless because it is “just Foundation-like.”
A wrapper type mirrors the third-party API so closely that it becomes a fake abstraction.
A public import is used because it “fixes the compiler error” rather than because the dependency is intentionally public.
```

Avoid when:

```text
You are hiding a dependency that is genuinely part of the public abstraction.
You are using access-level imports as a substitute for proper packaging, linking, or binary distribution decisions.
You are creating excessive wrapper layers that add no semantic stability.
```

Debugging checklist:

```text
Does the public .swiftinterface mention third-party modules?
Do public declarations mention third-party types?
Can clients compile without directly knowing the implementation dependency?
Could we replace the dependency without changing public signatures?
Is the dependency exposed intentionally or accidentally?
Should the import be public, package, internal, or private?
Is @inlinable forcing implementation details into the public contract?
Are tests relying on @testable public import instead of package/internal test-support APIs?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Import visibility controls whether a module can be used publicly or internally.

This is too shallow.

### Senior answer

> Import visibility prevents implementation dependencies from leaking into public APIs. If I mark a dependency as `internal import`, the compiler prevents me from using its types in public signatures, which helps keep the SDK’s public surface stable.

Good senior answer.

### Staff-level answer

> Import visibility is a dependency-boundary and API-evolution tool. I use `public import` only when a dependency is intentionally part of the public abstraction. Otherwise I prefer `internal`, `package`, or `private` imports so the compiler catches accidental leakage. For SDKs, this matters for SemVer, source compatibility, module interfaces, build performance, test-support boundaries, and future implementation swaps. I also review `@inlinable`, `@usableFromInline`, and public typealiases because they can leak implementation details even when the normal API surface looks clean.

That is the signal.

Staff-level questions to ask:

```text
Is this dependency part of our product contract or merely one implementation choice?
Could we replace this dependency in six months without breaking clients?
Does our public API expose behavior, domain models, or third-party mechanics?
Are public imports reviewed as carefully as public declarations?
Does our binary/module distribution strategy match our source-level dependency visibility?
```

---

## 9. Interview-ready summary

Import visibility is API design because imports define which dependency types are allowed to appear in your module’s public surface. In Swift 6, SE-0409 lets us write `public import`, `package import`, `internal import`, `fileprivate import`, or `private import`, and the compiler uses that access level as an upper bound when type-checking signatures. If a public API returns a third-party concrete type, that dependency is no longer an implementation detail; it becomes part of the client’s source-compatibility contract. A strong design keeps implementation dependencies behind adapters, SDK-owned DTOs, protocols, or Swift-native abstractions unless the dependency is intentionally part of the public model.

---

## 10. Flashcards

Q: What is dependency leakage in Swift module design?  
A: It is when an implementation dependency becomes visible in a public API, usually because a public signature mentions a type from that dependency.

Q: What does `internal import Foo` prevent?  
A: It prevents declarations from `Foo` from appearing in public or package-level signatures in that file. They can still be used internally.

Q: Is plain `import Foo` internal by default in Swift 6.2?  
A: No. In current Swift 6.x language modes, unannotated imports are still public by default for source compatibility. A future language mode is expected to change that default.

Q: When should you use `public import`?  
A: When the dependency is intentionally part of your module’s public abstraction and clients are expected to know about it.

Q: Why is returning `Alamofire.DataRequest` from a public SDK API risky?  
A: It couples clients to Alamofire and makes replacing Alamofire a breaking API change.

Q: What is the modern replacement for `@_implementationOnly import`?  
A: Use access-level imports such as `internal import`, `package import`, or `private import`, depending on the intended visibility.

Q: Does `private import` solve runtime packaging or linking problems?  
A: No. It controls source-level visibility in the file; packaging and runtime dependencies still need separate handling.

Q: What is a good public replacement for a third-party networking return type?  
A: A domain result type, `Data`, `AsyncSequence`, an SDK-owned DTO, or an SDK-owned protocol/adapter abstraction.

---

## 11. Related sections

- [[A11 — Access control]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E7 — Resilience, ABI, and module stability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist, E6 section.
- Swift Evolution SE-0409: Access-level modifiers on import declarations. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md "swift-evolution/proposals/0409-access-level-on-imports.md at main · swiftlang/swift-evolution · GitHub"))
- Swift compiler diagnostics: deprecated `@_implementationOnly import`. ([Swift Belgeleri](https://docs.swift.org/compiler/documentation/diagnostics/implementation-only-deprecated/?utm_source=chatgpt.com "Deprecated implementation-only imports ..."))