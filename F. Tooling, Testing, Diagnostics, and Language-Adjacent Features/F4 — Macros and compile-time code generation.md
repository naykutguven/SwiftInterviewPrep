---
tags:
  - swift
  - ios
  - interview-prep
  - tooling
  - macros
  - code-generation
---
## 0. Rubric snapshot

**Rubric expectation**

Know freestanding vs attached macros, Swift’s additive expansion model, package support, and macro tradeoffs. The rubric explicitly treats F4 as “Strong Senior / Emerging Staff Depth,” and macro architecture decisions as a staff-level differentiator.

**Caveats**

Macros move complexity to compile time; they do **not** eliminate complexity.

**You should be able to answer**

- What kinds of boilerplate are good candidates for macros, and what kinds are better handled with normal code?
- Why can macro-heavy codebases become harder to build, debug, or reason about?

**You should be able to do**

- Evaluate whether a request/response endpoint definition should use a macro, a property wrapper, a protocol, or plain code.

---

## 1. Core mental model

Swift macros are **compile-time source transformations**. You write a macro use site, the compiler asks a macro implementation to expand it, and the generated Swift source is compiled as if it had been written there. The important point is that macros are not runtime reflection, not dynamic dispatch, and not magic metadata. They are source-level generation integrated into the compiler pipeline.

Swift has two broad macro families: **freestanding** macros and **attached** macros. Freestanding macros appear independently, using `#`, for example `#stringify(x + y)`. Attached macros are written with attribute syntax, using `@`, and attach to a declaration, for example `@Observable`, `@attached(member)`, or a custom `@Endpoint`. The Swift book describes this top-level distinction directly: freestanding macros stand on their own, while attached macros modify the declaration they are attached to. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/macros/?utm_source=chatgpt.com "Macros - Documentation | Swift.org"))

Swift’s macro model is intentionally **additive**. Apple’s documentation states that calling a macro adds new code alongside your code and does not modify or delete code already in the project. That matters because a macro should make generated behavior inspectable and predictable, not secretly rewrite arbitrary program semantics. ([Apple Developer](https://developer.apple.com/documentation/Swift/applying-macros?utm_source=chatgpt.com "Applying Macros | Apple Developer Documentation"))

Macros are best understood as an API-design tool for **repetitive, mechanical, syntactic boilerplate**. They are a bad fit for business rules, state machines, networking behavior, concurrency semantics, and domain decisions that need to remain obvious in review.

The key idea:

```text
Macro = compile-time source generation. Use it to remove mechanical repetition, not to hide design.
```

Swift guarantees that macro uses participate in normal compilation: macro arguments and generated output are type-checked according to macro declarations and context. SE-0382 describes expression macros as source-to-source syntax transformations whose expanded syntax tree is type-checked against the macro result type. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md "swift-evolution/proposals/0382-expression-macros.md at main · swiftlang/swift-evolution · GitHub"))

Swift does **not** guarantee that a macro-heavy codebase is easier to understand, faster to build, easier to debug, or safer by default. Macros can emit diagnostics and reduce boilerplate, but they can also hide generated API surface, introduce confusing compile errors, and make builds depend on compiler-plugin infrastructure.

---

## 2. Essential mechanics

### Freestanding macros

Freestanding macros are invoked with `#`. They are not attached to a declaration. They are used where an expression or declaration is valid, depending on the macro role.

Expression macro example:

```swift
let result = #stringify(user.id + 1)

// Conceptual expansion:
let result = (user.id + 1, "user.id + 1")
```

SE-0382 introduced expression macros as `#`-prefixed expressions that expand into expressions. The proposal also emphasizes that the expansion is syntactic source generation, then type-checked. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md "swift-evolution/proposals/0382-expression-macros.md at main · swiftlang/swift-evolution · GitHub"))

Freestanding declaration macro example:

```swift
#warning("This API is temporary")
```

SE-0397 generalized freestanding macros so `#`-prefixed macros can also generate declarations rather than values. Declaration macros can be used where a declaration is permitted and can produce zero or more declarations. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0397-freestanding-declaration-macros.md "swift-evolution/proposals/0397-freestanding-declaration-macros.md at main · swiftlang/swift-evolution · GitHub"))

Use freestanding macros when the generated code belongs **at the use site** rather than to an existing declaration.

---

### Attached macros

Attached macros are invoked with `@` and attach to a declaration. They can add members, peers, accessors, conformances, or attributes depending on their role.

Conceptual example:

```swift
@Endpoint(method: .get, path: "/users/{id}", response: UserDTO.self)
struct GetUserRequest {
    let id: UserID
}
```

Possible conceptual expansion:

```swift
struct GetUserRequest {
    let id: UserID
}

extension GetUserRequest: Endpoint {
    typealias Response = UserDTO

    var method: HTTPMethod { .get }

    var path: String {
        "/users/\(id.rawValue)"
    }
}
```

Attached macros were introduced by SE-0389. The proposal describes them as a way to create and extend declarations through syntactic transformations, including generating members, accessors, wrapper functions, attributes, and conformances. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0389-attached-macros.md "swift-evolution/proposals/0389-attached-macros.md at main · swiftlang/swift-evolution · GitHub"))

Use attached macros when the generated code is naturally derived from a declaration’s shape.

---

### Additive expansion model

Macros should be thought of as adding explicit generated Swift code beside existing code.

Bad mental model:

```text
The macro changes what my code means behind the compiler's back.
```

Better mental model:

```text
The macro expands into additional Swift source that I should be able to inspect and reason about.
```

This is why good macros usually have small, predictable expansions. A macro that generates 20 lines of obvious conformance boilerplate is easier to justify than a macro that generates an entire networking stack, persistence layer, retry policy, and analytics pipeline.

---

### Macro package support

Custom macros are distributed through Swift packages. SE-0394 added package manager support for custom macros and introduced a macro target type. Macro implementations are built as host executables, and the compiler runs them on demand during compilation. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0394-swiftpm-expression-macros.md "swift-evolution/proposals/0394-swiftpm-expression-macros.md at main · swiftlang/swift-evolution · GitHub"))

Typical package shape:

```swift
// Package.swift sketch

import PackageDescription
import CompilerPluginSupport

let package = Package(
    name: "EndpointMacros",
    platforms: [
        .iOS(.v18),
        .macOS(.v15)
    ],
    products: [
        .library(name: "EndpointMacros", targets: ["EndpointMacros"])
    ],
    dependencies: [
        .package(url: "https://github.com/swiftlang/swift-syntax.git", from: "603.0.0")
    ],
    targets: [
        .macro(
            name: "EndpointMacrosImplementation",
            dependencies: [
                .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
                .product(name: "SwiftCompilerPlugin", package: "swift-syntax")
            ]
        ),
        .target(
            name: "EndpointMacros",
            dependencies: ["EndpointMacrosImplementation"]
        )
    ]
)
```

The public macro declaration usually lives in a normal library target:

```swift
@attached(extension, conformances: Endpoint)
@attached(member, names: named(Response), named(method), named(path))
public macro Endpoint(
    method: HTTPMethod,
    path: StaticString,
    response: Any.Type
) = #externalMacro(
    module: "EndpointMacrosImplementation",
    type: "EndpointMacro"
)
```

The exact APIs evolve with SwiftSyntax and toolchain versions, so staff-level usage means treating macro packages as compiler-adjacent infrastructure, not casual helper code.

---

## 3. Common traps and misconceptions

### Trap 1: “Macros remove complexity”

They do not. They move complexity from handwritten source into compile-time generation.

Bad:

```swift
@Feature(
    endpoint: "/users/{id}",
    cache: .memoryAndDisk,
    retry: .exponentialBackoff,
    analytics: .screen("profile"),
    auth: .required,
    stalePolicy: .serveStaleThenRefresh
)
struct UserProfileFeature {
    let id: UserID
}
```

This hides networking, caching, retry, analytics, auth, and staleness behavior behind one attribute. The source becomes short, but the system becomes harder to reason about.

Better:

```swift
struct GetUserProfile: Endpoint {
    typealias Response = UserProfileDTO

    let id: UserID

    var method: HTTPMethod { .get }
    var path: String { "/users/\(id.rawValue)" }
}

struct UserProfileRepository {
    let client: HTTPClient
    let cache: UserProfileCache
    let retryPolicy: RetryPolicy

    func profile(id: UserID) async throws -> UserProfile {
        try await cache.value(for: id) {
            try await retryPolicy.run {
                try await client.send(GetUserProfile(id: id))
            }
        }
    }
}
```

Here the endpoint shape is explicit, and behavior remains testable.

---

### Trap 2: Using a macro because syntax looks elegant

A macro is justified when the generated code is mechanical, repeated, and reviewable. It is not justified merely because the call site looks cute.

Bad:

```swift
@AutoEverything
struct Checkout {}
```

Better:

```swift
struct CheckoutStateMachine {
    mutating func apply(_ event: CheckoutEvent) {
        // Explicit domain logic belongs here.
    }
}
```

Domain behavior should usually be plain Swift. Code reviewers need to see it.

---

### Trap 3: Treating macros like type-system features

Macros can generate code, but they are not a replacement for generics, protocols, associated types, or clear domain models.

Bad:

```swift
@GenerateRepository(entity: User.self)
struct UserRepository {}
```

Better:

```swift
protocol Repository<Entity> {
    associatedtype Entity

    func fetch(id: Entity.ID) async throws -> Entity
    func save(_ entity: Entity) async throws
}
```

Use the type system first. Reach for macros when the remaining boilerplate is mechanical.

---

### Trap 4: Hiding generated API surface from clients

Attached macros can add members and conformances. That generated surface becomes part of what users call, autocomplete, debug, and rely on.

Bad:

```swift
@PublicSDKModel
public struct Payment {}
```

If this secretly generates public initializers, public nested types, protocol conformances, and coding behavior, the public API contract becomes unclear.

Better:

```swift
public struct Payment: Sendable, Codable, Equatable {
    public let id: PaymentID
    public let amount: Money
    public let status: PaymentStatus

    public init(id: PaymentID, amount: Money, status: PaymentStatus) {
        self.id = id
        self.amount = amount
        self.status = status
    }
}
```

For public SDKs, generated API is still API. It must be documented and versioned.

---

## 4. Direct answers to rubric questions

### Q1. What kinds of boilerplate are good candidates for macros, and what kinds are better handled with normal code?

Good macro candidates are repetitive, mechanical, local, syntax-derived, and have predictable expansion.

Examples:

```text
Good candidates:
- Repetitive protocol conformance scaffolding
- Boilerplate initializers
- Repetitive coding-key or schema glue
- Lightweight endpoint metadata
- Compile-time checked string/path literals
- Repeated wrapper/trampoline declarations
- Generated declarations from local syntax
```

Poor macro candidates:

```text
Bad candidates:
- Business rules
- State transitions
- Retry, cache, or cancellation behavior
- Concurrency isolation design
- Networking control flow
- Error recovery policy
- Anything reviewers must inspect line-by-line
```

Interview version:

> I use macros when the expansion is mechanical and predictable: the macro should generate code I would otherwise write by hand in the same shape every time. I avoid macros for domain behavior, concurrency semantics, or business logic because those need to stay explicit, testable, and reviewable. A good macro removes repetition without hiding architectural decisions.

---

### Q2. Why can macro-heavy codebases become harder to build, debug, or reason about?

Macro-heavy codebases can become harder because the source you read is not the full source the compiler sees. Generated declarations affect overload resolution, conformances, diagnostics, and autocomplete. When an error appears inside generated code, the developer often has to inspect an expansion rather than the original source.

Builds can also become heavier because macro implementations are compiler plugins. SE-0394 describes macro implementations as external programs built for the host and run by the compiler during compilation. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0394-swiftpm-expression-macros.md "swift-evolution/proposals/0394-swiftpm-expression-macros.md at main · swiftlang/swift-evolution · GitHub")) SE-0382 also notes that syntactic expansions are re-parsed and re-type-checked, which adds compile-time overhead. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md "swift-evolution/proposals/0382-expression-macros.md at main · swiftlang/swift-evolution · GitHub"))

Reasoning cost increases when macros generate large API surfaces, hide side effects, or encode conventions that are not obvious at the use site.

Interview version:

> Macro-heavy code can be harder because there are two programs to understand: the source code and the generated source. The compiler plugin has its own dependencies, build cost, diagnostics, and versioning concerns. I’m comfortable using macros, but I want the generated code to be inspectable, small, documented, and covered by macro expansion tests.

---

## 5. Code probe

The rubric does not include a dedicated code probe for F4, so use these as practical probes instead.

### Minimal example: freestanding expression macro

Given:

```swift
let a = 10
let b = 20

let value = #stringify(a + b)
```

Conceptual expansion:

```swift
let value = (a + b, "a + b")
```

What happens:

```text
value.0 == 30
value.1 == "a + b"
```

Why:

```text
Source:
#stringify(a + b)

Expansion:
(a + b, "a + b")

Then normal Swift type-checking and execution happen.
```

This is a good macro shape because the expansion is small, local, and easy to understand. SE-0382 uses this kind of example to describe expression macros. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md "swift-evolution/proposals/0382-expression-macros.md at main · swiftlang/swift-evolution · GitHub"))

---

### Counterexample: macro hiding behavior

Bad:

```swift
@Endpoint(
    method: .get,
    path: "/users/{id}",
    response: UserDTO.self,
    retry: .automatic,
    cache: .automatic,
    auth: .automatic
)
struct GetUser {
    let id: UserID
}
```

Problem:

```text
This macro is no longer just endpoint boilerplate. It encodes transport policy,
auth policy, retry policy, cache policy, and possibly threading/cancellation behavior.
That is too much hidden behavior for one attribute.
```

Better:

```swift
struct GetUser: Endpoint {
    typealias Response = UserDTO

    let id: UserID

    var method: HTTPMethod { .get }
    var path: String { "/users/\(id.rawValue)" }
}

struct UserService {
    let client: HTTPClient
    let retryPolicy: RetryPolicy

    func user(id: UserID) async throws -> UserDTO {
        try await retryPolicy.run {
            try await client.send(GetUser(id: id))
        }
    }
}
```

Why this fix is correct:

```text
Endpoint metadata remains close to the request type.
Runtime behavior remains explicit, injectable, and testable.
The type system still preserves the request/response relationship.
```

---

### Production example: small attached macro for repetitive endpoint metadata

A reasonable macro target:

```swift
@HTTPRoute(.get, "/users/{id}", response: UserDTO.self)
struct GetUser {
    let id: UserID
}
```

Generated shape:

```swift
extension GetUser: Endpoint {
    typealias Response = UserDTO

    var method: HTTPMethod { .get }

    var path: String {
        "/users/\(id.rawValue)"
    }
}
```

This is acceptable only if the macro does **not** hide runtime behavior. It should synthesize endpoint metadata and diagnostics, not decide retries, caching, cancellation, auth, or analytics.

Alternative fixes and tradeoffs:

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Plain struct + protocol|Small or medium endpoint surface; behavior should stay explicit|More boilerplate|
|Attached macro|Many endpoints with repeated mechanical conformance|Build/debug complexity|
|Freestanding declaration macro|Generated declarations from schema-like local input|Can create large invisible API surface|
|Property wrapper|Per-property storage/access behavior|Wrong abstraction for whole endpoint definitions|
|Result builder|Hierarchical route tables or DSL-like server routing|Can hurt type-checking and diagnostics|
|External codegen|OpenAPI/GraphQL schema source of truth|Extra generation step and drift risk|

---

## 6. Exercise

### Problem

Evaluate whether a request/response endpoint definition should use a macro, a property wrapper, a protocol, or plain code.

### Bad / naive version

```swift
@Endpoint(
    method: "GET",
    path: "/users/{id}",
    response: "UserDTO",
    requiresAuth: true,
    cache: true,
    retry: true,
    analytics: true
)
struct GetUser {
    let id: String
}
```

### What is wrong?

```text
- Stringly typed method.
- Stringly typed response.
- Macro hides too many runtime policies.
- Auth/cache/retry/analytics are not endpoint shape; they are behavior.
- Harder to unit-test because policy decisions are generated.
- Harder to review because a small annotation implies a large system.
- Harder to migrate because changing macro expansion can affect many call sites at once.
```

The main smell is that the macro is being used as an architectural compression tool. That is exactly how macro-based systems become opaque.

### Improved version: protocol-first design

```swift
enum HTTPMethod: Sendable {
    case get
    case post
    case put
    case patch
    case delete
}

protocol Endpoint<Response>: Sendable {
    associatedtype Response: Decodable & Sendable

    var method: HTTPMethod { get }
    var path: String { get }
    var queryItems: [URLQueryItem] { get }
}

struct GetUser: Endpoint {
    typealias Response = UserDTO

    let id: UserID

    var method: HTTPMethod { .get }
    var path: String { "/users/\(id.rawValue)" }
    var queryItems: [URLQueryItem] { [] }
}

struct HTTPClient: Sendable {
    func send<E: Endpoint>(_ endpoint: E) async throws -> E.Response {
        // Build URLRequest.
        // Execute request.
        // Decode E.Response.
        fatalError("Implementation omitted")
    }
}
```

### Why this is better

```text
- Request/response relationship is expressed by associated types.
- The endpoint is ordinary Swift and easy to test.
- Behavior remains outside the endpoint metadata.
- The compiler preserves the response type at the call site.
- No generated code is needed until repetition proves painful.
```

Call site:

```swift
let user: UserDTO = try await client.send(GetUser(id: userID))
```

This should be the default design. It is explicit, type-safe, and debuggable.

---

### When a macro becomes justified

Once you have dozens or hundreds of endpoints with identical mechanical shape, a small attached macro can become reasonable.

```swift
@HTTPRoute(.get, "/users/{id}", response: UserDTO.self)
struct GetUser {
    let id: UserID
}
```

The macro may generate:

```swift
extension GetUser: Endpoint {
    typealias Response = UserDTO

    var method: HTTPMethod { .get }

    var path: String {
        "/users/\(id.rawValue)"
    }

    var queryItems: [URLQueryItem] {
        []
    }
}
```

The macro may also emit diagnostics:

```text
- Path placeholder `{id}` has no matching stored property.
- Stored property `page` is unused by the route.
- Response type must conform to Decodable & Sendable.
- Path must be a static string literal.
```

That is a good macro use: it removes repetitive conformance code and adds compile-time validation, while leaving runtime behavior explicit.

---

### Why a property wrapper is usually wrong here

Property wrappers are about property storage/access semantics.

Bad fit:

```swift
struct GetUser {
    @PathComponent("id")
    var id: UserID
}
```

This can help if you are building a route DSL, but it does not naturally model the endpoint itself. It also scatters endpoint structure across properties.

Property wrappers are better for things like:

```swift
@propertyWrapper
struct Trimmed {
    private var value = ""

    var wrappedValue: String {
        get { value }
        set { value = newValue.trimmingCharacters(in: .whitespacesAndNewlines) }
    }
}
```

Endpoint definitions are usually type-level declarations, not property storage behavior.

---

### Why plain code may still win

For fewer than roughly 10–20 endpoints, plain code is often better:

```swift
struct SearchUsers: Endpoint {
    typealias Response = [UserDTO]

    let query: String
    let page: Int

    var method: HTTPMethod { .get }
    var path: String { "/users/search" }

    var queryItems: [URLQueryItem] {
        [
            URLQueryItem(name: "q", value: query),
            URLQueryItem(name: "page", value: String(page))
        ]
    }
}
```

The cost of a macro package, macro tests, generated-source inspection, SwiftSyntax dependency churn, and onboarding may exceed the boilerplate cost.

---

## 7. Production guidance

Use macros in production when:

```text
- The generated code is mechanical and predictable.
- The expansion is small enough to inspect.
- The macro can emit better compile-time diagnostics than normal code.
- The use site remains honest about what behavior exists.
- You have enough repetition to justify compiler-plugin complexity.
- You test macro expansions directly.
- The generated API is documented if it is public or semi-public.
```

Be careful when:

```text
- The macro generates public API.
- The macro generates protocol conformances.
- The macro depends heavily on SwiftSyntax internals.
- The macro is used across many targets.
- The macro participates in concurrency or isolation-sensitive code.
- The macro hides runtime behavior.
- Errors become harder to understand than the original boilerplate.
```

Avoid macros when:

```text
- Plain protocols or generics solve the problem.
- You are hiding business logic.
- The generated code is large or surprising.
- The macro mostly saves a few lines.
- The team cannot maintain macro implementation code.
- The expansion would need semantic knowledge that the macro system does not provide cleanly.
```

Debugging checklist:

```text
Can I inspect the expanded code?
Is the generated API part of the public contract?
Did the macro generate a conformance, member, accessor, or peer declaration?
Is the error in user-written code or generated code?
Would plain Swift make the behavior clearer?
Is the macro expansion deterministic?
Is the macro tested with representative input syntax?
Does the macro produce actionable diagnostics?
Is the macro increasing build time or invalidating too much work?
Does the macro hide runtime behavior that should be injected and tested?
```

Staff-level rule:

```text
A macro is justified when it improves correctness, consistency, or diagnostics enough to pay for its hidden build and reasoning cost.
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Macros generate code at compile time and reduce boilerplate.

This is true but too shallow.

### Senior answer

> Swift macros are compile-time source transformations. Freestanding macros use `#`; attached macros use `@` and attach to declarations. I would use them for repetitive mechanical code like conformances or route metadata, but I would avoid them for business logic because generated behavior is harder to review and debug.

This is solid senior-level understanding.

### Staff-level answer

> I treat macros as compiler-adjacent infrastructure. Before adding one, I ask whether the type system, protocols, generics, property wrappers, result builders, or plain code solve the problem with less hidden cost. If a macro is justified, I keep the expansion additive, small, inspectable, deterministic, and well-tested. I also think about build impact, public API stability, diagnostics, SwiftSyntax versioning, target boundaries, and whether the generated API creates long-term coupling.

That is the staff-level signal: not just knowing macro syntax, but owning the architecture cost.

Staff-level questions to ask:

```text
What exact code will this macro generate?
Can users inspect and understand the expansion?
Is this macro removing boilerplate or hiding behavior?
Does generated code become public API?
How will we test expansion and diagnostics?
How does this affect incremental builds and CI?
What happens when SwiftSyntax or the compiler version changes?
Can this be done with a protocol, generic type, result builder, or plain code?
Does this macro improve correctness or just reduce typing?
What is the migration path if we remove the macro later?
```

---

## 9. Interview-ready summary

Swift macros are compile-time source generation integrated into the compiler. Freestanding macros use `#` and stand alone; attached macros use `@` and add code related to a declaration. The model is additive: macros add generated Swift source, they do not secretly rewrite or delete existing code. I would use macros for repetitive, mechanical, inspectable boilerplate where compile-time diagnostics add value. I would avoid them for business logic, concurrency policy, networking behavior, or anything that needs to remain explicit in review. Staff-level judgment is knowing when the macro’s consistency and diagnostic benefits outweigh its build, debugging, API, and maintenance costs.

---

## 10. Flashcards

Q: What is the difference between freestanding and attached macros in Swift?  
A: Freestanding macros use `#` and stand on their own. Attached macros use `@` and attach to a declaration, adding members, peers, accessors, conformances, or attributes depending on role.

Q: What does Swift’s additive macro model mean?  
A: A macro adds generated code alongside existing code. It should not be understood as secretly mutating or deleting user-written source.

Q: What makes boilerplate a good macro candidate?  
A: It is repetitive, mechanical, local, syntax-derived, predictable, and produces generated code that developers can inspect and understand.

Q: What should usually not be implemented with macros?  
A: Business logic, state transitions, retry policies, caching behavior, auth decisions, concurrency isolation, and complex runtime behavior.

Q: Why can macros hurt build performance?  
A: Macro implementations are compiler plugins run during compilation, and macro expansions are parsed and type-checked as generated Swift source.

Q: Why are macro-generated public members risky?  
A: They become part of the API surface clients can use and rely on, even if the source declaration looks small.

Q: Why is a protocol often better than a macro for endpoint definitions?  
A: A protocol can express the request/response relationship directly with associated types while keeping behavior explicit and testable.

Q: When might an endpoint macro be justified?  
A: When many endpoints share repetitive mechanical conformance code and the macro only generates metadata while providing useful compile-time diagnostics.

---

## 11. Related sections

- [[B10 — API design guidelines and Swift-native surface design]]
- [[B13 — Property wrappers and generated storage semantics]]
- [[B14 — Result builders and DSL construction]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F5 — Language mode, build settings, and migration flags]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

---

## 12. Sources

- Swift Senior/Staff Rubric — F4 macro expectations, questions, and exercise.
- Swift Book — Macros chapter: freestanding vs attached macros. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/macros/?utm_source=chatgpt.com "Macros - Documentation | Swift.org"))
- Apple Developer Documentation — Applying Macros and additive expansion model. ([Apple Developer](https://developer.apple.com/documentation/Swift/applying-macros?utm_source=chatgpt.com "Applying Macros | Apple Developer Documentation"))
- SE-0382 — Expression Macros. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md "swift-evolution/proposals/0382-expression-macros.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0389 — Attached Macros. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0389-attached-macros.md "swift-evolution/proposals/0389-attached-macros.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0394 — Package Manager Support for Custom Macros. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0394-swiftpm-expression-macros.md "swift-evolution/proposals/0394-swiftpm-expression-macros.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0397 — Freestanding Declaration Macros. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0397-freestanding-declaration-macros.md "swift-evolution/proposals/0397-freestanding-declaration-macros.md at main · swiftlang/swift-evolution · GitHub"))