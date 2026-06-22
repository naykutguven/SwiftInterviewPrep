---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - type-choices
---
## 0. Rubric snapshot

**Rubric expectation**

Choose between `struct`, `class`, `enum`, `protocol`, and `actor` based on value semantics, identity, finite-state modeling, abstraction boundaries, and isolated mutable state.

**Caveats**

Reaching for classes or protocols by default usually adds incidental complexity. Actors are for isolated mutable state; they are not a generic replacement for all model types.

**You should be able to answer**

- When is a class genuinely required instead of a struct?
- When should mutable shared state live behind an actor rather than a class plus manual synchronization?

**You should be able to do**

- Take a small feature with domain models, shared cache, request state, and service abstraction, then choose `struct`, `class`, `enum`, `protocol`, or `actor` for each piece with justification.

---

## 1. Core mental model

Swift type choice is not about syntax preference. It is about choosing the semantic contract that matches the problem.

A `struct` says: “this is a value.” Copies should behave independently, equality is usually structural or domain-defined, and mutation should affect only the value being mutated. Swift’s official documentation defines structures and enumerations as value types, while classes are reference types that are not copied when assigned or passed around. ([Swift.org, "Structures and Classes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/))

An `enum` says: “this value is one of a closed set of states.” It is best when the domain has mutually exclusive cases, especially when each case needs different associated data. Swift enums model a group of related values in a type-safe way, and associated values can vary by case. ([Swift.org, "Enumerations"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/))

A `class` says: “this object has identity and shared lifetime.” Two references can point to the same instance. Mutation through one reference is visible through another. That is powerful, but it creates aliasing, ARC lifetime, retain-cycle, and concurrency questions.

A `protocol` says: “this is a capability boundary.” It should model behavior that multiple concrete types can provide, not become a default substitute for every type. Swift protocols define a blueprint of methods, properties, and requirements for a task or piece of functionality. ([Swift.org, "Protocols"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/))

An `actor` says: “this is shared mutable state protected by actor isolation.” Swift guarantees that only code running on an actor can access that actor’s local state directly; crossing the actor boundary requires asynchronous access. ([Swift.org, "Concurrency"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/))

The key idea:

```text
struct = value
enum = closed state
class = identity / shared lifetime
protocol = behavior boundary
actor = isolated shared mutable state
```

---

## 2. Essential mechanics

### 2.1 Prefer `struct` for domain data

Use `struct` when the value should be copied, transformed, compared, serialized, sent across layers, or passed between tasks without hidden shared mutation.

```swift
struct Product: Identifiable, Equatable, Sendable {
    let id: Product.ID
    var name: String
    var price: Decimal

    struct ID: Hashable, Sendable {
        let rawValue: String
    }
}
```

This is a good domain model because:

```text
- Product has no meaningful object identity beyond its domain ID.
- Mutating a copy should not mutate another copy.
- It is easy to test, serialize, compare, and pass around.
```

A class would be suspicious here unless the product object itself has shared mutable identity inside the app.

---

### 2.2 Prefer `enum` for finite state

Use `enum` when only one state is valid at a time.

Bad:

```swift
struct RequestStatus {
    var isLoading = false
    var data: Data?
    var error: Error?
}
```

This allows invalid combinations:

```text
isLoading = true + data != nil
isLoading = true + error != nil
data != nil + error != nil
isLoading = false + data == nil + error == nil
```

Better:

```swift
enum RequestState<Value: Sendable>: Sendable {
    case idle
    case loading
    case loaded(Value)
    case failed(String)
}
```

Now the impossible states are unrepresentable.

---

### 2.3 Use `class` only when identity or reference lifetime is real

Use `class` when the thing being modeled must be shared by reference.

```swift
final class DownloadTask {
    let id: UUID
    private var isCancelled = false

    init(id: UUID = UUID()) {
        self.id = id
    }

    func cancel() {
        isCancelled = true
    }
}
```

This can justify a class because cancellation is identity-based. You want every holder of the task reference to talk about the same task.

A `final class` is usually the default class shape unless subclassing is part of the design. Subclassing is a public customization mechanism, not just “code reuse.”

---

### 2.4 Use `protocol` for real abstraction boundaries

Use protocols when callers need to depend on capability, not implementation.

```swift
protocol ImageFetching: Sendable {
    func imageData(for url: URL) async throws -> Data
}

struct URLSessionImageFetcher: ImageFetching {
    func imageData(for url: URL) async throws -> Data {
        let (data, _) = try await URLSession.shared.data(from: url)
        return data
    }
}
```

This is useful because production and test implementations can vary:

```swift
struct StubImageFetcher: ImageFetching {
    let result: Result<Data, Error>

    func imageData(for url: URL) async throws -> Data {
        try result.get()
    }
}
```

Do not create protocols just because a type exists. Create protocols when there is a real substitution boundary.

---

### 2.5 Use `actor` for shared mutable state across concurrency domains

Use actors when state is mutable, shared, and accessed from concurrent tasks.

```swift
actor ImageCache {
    private var storage: [URL: Data] = [:]

    func data(for url: URL) -> Data? {
        storage[url]
    }

    func insert(_ data: Data, for url: URL) {
        storage[url] = data
    }
}
```

The actor owns the synchronization policy. Callers cannot directly mutate `storage`, so the cache’s invariants stay centralized.

```swift
let cache = ImageCache()

if let data = await cache.data(for: url) {
    // Use cached data.
} else {
    let data = try await fetcher.imageData(for: url)
    await cache.insert(data, for: url)
}
```

Actors are not “async classes.” They are isolation boundaries. They protect actor-isolated state from data races, but they do not automatically fix stale reads, reentrancy bugs, or poor workflow design.

---

## 3. Common traps and misconceptions

### Trap 1: Using classes for ordinary data

Bad:

```swift
final class User {
    var id: String
    var name: String

    init(id: String, name: String) {
        self.id = id
        self.name = name
    }
}
```

This creates reference aliasing for no clear reason.

Better:

```swift
struct User: Identifiable, Equatable, Sendable {
    let id: String
    var name: String
}
```

Use a class only if shared identity is part of the domain or framework contract.

---

### Trap 2: Creating protocols for every concrete type

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

This protocol adds no useful abstraction. It just duplicates the concrete model.

Better:

```swift
struct User: Identifiable, Equatable, Sendable {
    let id: String
    let name: String
}

protocol UserLoading: Sendable {
    func user(id: User.ID) async throws -> User
}
```

The protocol belongs at the behavior boundary, not around plain data.

---

### Trap 3: Treating actors as a replacement for all mutable types

Bad:

```swift
actor UserProfile {
    var name: String
    var avatarURL: URL?

    init(name: String, avatarURL: URL?) {
        self.name = name
        self.avatarURL = avatarURL
    }
}
```

This makes every read asynchronous for no good reason.

Better:

```swift
struct UserProfile: Equatable, Sendable {
    var name: String
    var avatarURL: URL?
}
```

Use an actor for shared mutable state, not for every model that has `var`.

---

### Trap 4: Hiding synchronization inside a mutable class without a clear invariant

Bad:

```swift
final class ImageCache {
    var storage: [URL: Data] = [:]
}
```

This is unsafe if accessed from multiple tasks or threads.

Better:

```swift
actor ImageCache {
    private var storage: [URL: Data] = [:]

    func data(for url: URL) -> Data? {
        storage[url]
    }

    func insert(_ data: Data, for url: URL) {
        storage[url] = data
    }
}
```

A class plus a lock can be valid, but the locking invariant must be explicit, small, and consistently enforced. Otherwise, use an actor.

---

## 4. Direct answers to rubric questions

### Q1. When is a class genuinely required instead of a struct?

A class is genuinely required when reference identity, shared mutable lifetime, inheritance, Objective-C interoperability, weak references, or deinitialization-based resource management is part of the design.

Common valid reasons:

```text
- You need identity separate from value equality.
- Multiple owners must observe or mutate the same instance.
- You need weak references to break ownership cycles.
- You need inheritance or NSObject / Objective-C runtime behavior.
- You need reference-based framework integration.
- You need deinit to close, cancel, invalidate, or release something.
```

Do not use a class just because the model has many properties or methods. A large model can still be a value.

Interview version:

> I use a struct for ordinary domain data and a class only when identity or shared lifetime is part of the semantics. If copying the value should create an independent value, it should be a struct. If two references must intentionally observe the same mutable object, or if I need inheritance, weak references, Objective-C interop, or deinit-driven lifetime behavior, then a class is justified. I usually make classes final unless subclassing is an explicit API contract.

---

### Q2. When should mutable shared state live behind an actor rather than a class plus manual synchronization?

Mutable shared state should live behind an actor when the state is accessed across concurrent tasks or isolation domains and correctness depends on serialized access to that state.

Good actor candidates:

```text
- shared caches
- token refresh coordinators
- session state
- deduplication / single-flight registries
- mutable stores used from multiple tasks
- services with internal mutable state and async APIs
```

A class plus a lock can be better when:

```text
- the critical section is tiny and synchronous
- performance overhead of actor hops is measurable and relevant
- the code cannot suspend inside the protected operation
- the locking policy is very local and auditable
```

Interview version:

> I choose an actor when shared mutable state is naturally part of an async system and I want the compiler to enforce isolation. A class with manual synchronization is still valid for small synchronous hot paths, but it relies on discipline. An actor gives a stronger architectural boundary: state access is centralized, cross-boundary calls are explicit, and Swift can diagnose direct unsafe access. That said, actors solve data races, not every logic race; I still need to design around reentrancy and stale state.

---

## 5. Code probe

A15 has no rubric code probe. Instead, use these probes.

### 5.1 Minimal example: value vs reference behavior

Given:

```swift
struct UserProfile {
    var name: String
}

final class Session {
    var token: String

    init(token: String) {
        self.token = token
    }
}

var profileA = UserProfile(name: "A")
var profileB = profileA
profileB.name = "B"

let sessionA = Session(token: "old")
let sessionB = sessionA
sessionB.token = "new"

print(profileA.name, profileB.name)
print(sessionA.token, sessionB.token)
```

### What happens?

```text
A B
new new
```

### Why?

```text
profileA ── value copy ──> profileB
  name: "A"               name: "B"

sessionA ─┐
          ├──> same Session instance
sessionB ─┘
          token: "new"
```

`UserProfile` is a struct, so assigning it creates an independent value. Mutating `profileB` does not mutate `profileA`.

`Session` is a class, so assigning it copies the reference, not the object. `sessionA` and `sessionB` point to the same instance.

---

### 5.2 Counterexample: actor state cannot be mutated directly from outside

Given:

```swift
import Foundation

actor ImageCache {
    var storage: [URL: Data] = [:]
}

let cache = ImageCache()
let url = URL(string: "https://example.com/image.png")!
cache.storage[url] = Data()
```

### What happens?

```text
/tmp/a15_counter.swift:9:7: error: actor-isolated property 'storage' can not be mutated from the main actor
 2 | 
 3 | actor ImageCache {
 4 |     var storage: [URL: Data] = [:]
   |         `- note: mutation of this property is only permitted within the actor
 5 | }
 6 | 
 7 | let cache = ImageCache()
 8 | let url = URL(string: "https://example.com/image.png")!
 9 | cache.storage[url] = Data()
   |       `- error: actor-isolated property 'storage' can not be mutated from the main actor
10 | 
```

### Why?

Actor-isolated state can only be accessed directly from inside the actor. External code must use actor methods and cross the isolation boundary.

Correct:

```swift
import Foundation

actor ImageCache {
    private var storage: [URL: Data] = [:]

    func insert(_ data: Data, for url: URL) {
        storage[url] = data
    }

    func data(for url: URL) -> Data? {
        storage[url]
    }
}

let cache = ImageCache()
let url = URL(string: "https://example.com/image.png")!

await cache.insert(Data(), for: url)
let data = await cache.data(for: url)
```

### Why this fix is correct

The actor keeps `storage` private and exposes operations that preserve its invariants. External code cannot mutate the dictionary directly.

---

### 5.3 Production example: feature-level type choices

```swift
import Foundation

struct ImageID: Hashable, Sendable {
    let rawValue: String
}

struct FeedItem: Identifiable, Equatable, Sendable {
    let id: String
    let title: String
    let imageURL: URL
}

enum ImageRequestState: Sendable {
    case idle
    case loading
    case loaded(Data)
    case failed(String)
}

protocol ImageFetching: Sendable {
    func imageData(for url: URL) async throws -> Data
}

struct URLSessionImageFetcher: ImageFetching {
    func imageData(for url: URL) async throws -> Data {
        let (data, _) = try await URLSession.shared.data(from: url)
        return data
    }
}

actor ImageCache {
    private var storage: [URL: Data] = [:]

    func data(for url: URL) -> Data? {
        storage[url]
    }

    func insert(_ data: Data, for url: URL) {
        storage[url] = data
    }
}

struct FeedImageLoadingFeature<Fetcher: ImageFetching>: Sendable {
    let fetcher: Fetcher
    let cache: ImageCache

    func loadImage(for item: FeedItem) async -> ImageRequestState {
        if let cached = await cache.data(for: item.imageURL) {
            return .loaded(cached)
        }

        do {
            let data = try await fetcher.imageData(for: item.imageURL)
            await cache.insert(data, for: item.imageURL)
            return .loaded(data)
        } catch {
            return .failed(error.localizedDescription)
        }
    }
}
```

Type choices:

|Piece|Type choice|Reason|
|---|---|---|
|`FeedItem`|`struct`|Domain data with value semantics|
|`ImageID`|`struct`|Strongly typed ID, avoids raw-string confusion|
|`ImageRequestState`|`enum`|Mutually exclusive finite request states|
|`ImageFetching`|`protocol`|Real service boundary for production/test substitution|
|`URLSessionImageFetcher`|`struct`|Stateless concrete implementation|
|`ImageCache`|`actor`|Shared mutable state used by concurrent tasks|
|`FeedImageLoadingFeature`|`struct` generic over `Fetcher`|Lightweight composition without existential erasure|

---

## 6. Exercise

### Problem

Take a small feature with domain models, shared cache, request state, and service abstraction, and choose `struct`, `class`, `enum`, `protocol`, or `actor` for each piece with justification.

### Bad / naive version

```swift
import Foundation

final class FeedItem {
    var id: String
    var title: String
    var imageURL: URL

    init(id: String, title: String, imageURL: URL) {
        self.id = id
        self.title = title
        self.imageURL = imageURL
    }
}

protocol FeedItemProtocol {
    var id: String { get }
    var title: String { get }
}

final class ImageCache {
    var storage: [URL: Data] = [:]
}

final class ImageLoader {
    var isLoading = false
    var data: Data?
    var errorMessage: String?

    let cache = ImageCache()

    func load(_ item: FeedItem) async {
        isLoading = true

        if let cached = cache.storage[item.imageURL] {
            data = cached
            isLoading = false
            return
        }

        do {
            let (newData, _) = try await URLSession.shared.data(from: item.imageURL)
            cache.storage[item.imageURL] = newData
            data = newData
            isLoading = false
        } catch {
            errorMessage = error.localizedDescription
            isLoading = false
        }
    }
}
```

### What is wrong?

```text
FeedItem is a class without a clear identity requirement.
FeedItemProtocol is a useless abstraction around plain data.
ImageCache exposes mutable shared dictionary state.
ImageLoader mixes UI/request state, networking, and caching.
isLoading/data/errorMessage allow invalid combinations.
The cache is unsafe if accessed concurrently.
The service implementation is hard to replace in tests.
```

### Improved version

```swift
import Foundation

struct FeedItem: Identifiable, Equatable, Sendable {
    let id: String
    let title: String
    let imageURL: URL
}

enum ImageRequestState: Sendable {
    case idle
    case loading
    case loaded(Data)
    case failed(String)
}

protocol ImageFetching: Sendable {
    func imageData(for url: URL) async throws -> Data
}

struct URLSessionImageFetcher: ImageFetching {
    func imageData(for url: URL) async throws -> Data {
        let (data, _) = try await URLSession.shared.data(from: url)
        return data
    }
}

actor ImageCache {
    private var storage: [URL: Data] = [:]

    func data(for url: URL) -> Data? {
        storage[url]
    }

    func insert(_ data: Data, for url: URL) {
        storage[url] = data
    }
}

@MainActor
final class FeedImageViewModel<Fetcher: ImageFetching> {
    private let item: FeedItem
    private let fetcher: Fetcher
    private let cache: ImageCache

    private(set) var state: ImageRequestState = .idle

    init(item: FeedItem, fetcher: Fetcher, cache: ImageCache) {
        self.item = item
        self.fetcher = fetcher
        self.cache = cache
    }

    func load() async {
        state = .loading

        if let cached = await cache.data(for: item.imageURL) {
            state = .loaded(cached)
            return
        }

        do {
            let data = try await fetcher.imageData(for: item.imageURL)
            await cache.insert(data, for: item.imageURL)
            state = .loaded(data)
        } catch {
            state = .failed(error.localizedDescription)
        }
    }
}
```

### Why this is better

```text
FeedItem is value data.
ImageRequestState makes invalid states unrepresentable.
ImageFetching is a real service abstraction.
URLSessionImageFetcher is a concrete implementation, not forced into a class.
ImageCache is an actor because cache state is shared and mutable.
FeedImageViewModel is a class because UI view models often need identity and observable lifetime.
@MainActor keeps UI-facing state mutation on the UI isolation domain.
```

The view model is the one place where a class is defensible: UI state often has identity, lifecycle, and observation concerns. The domain model and request state should still be values.

---

## 7. Production guidance

Use this in production when:

```text
Use struct for domain values, DTOs, IDs, configuration, snapshots, and stateless services.
Use enum for finite state, commands, events, result-like domain outcomes, and mutually exclusive modes.
Use final class for identity, shared lifetime, framework interop, weak references, and reference-based UI objects.
Use protocol for real behavior boundaries, especially dependency injection and substitutable services.
Use actor for shared mutable state accessed from async/concurrent code.
```

Be careful when:

```text
A class has mutable state and is accessed from multiple tasks.
A protocol has only one conforming type and no test or module boundary.
An actor method reads state, awaits, then assumes the state is unchanged.
A value type contains reference-typed mutable storage.
A public protocol freezes more API surface than intended.
```

Avoid when:

```text
Avoid class for plain data.
Avoid protocol for every noun.
Avoid actor for immutable values.
Avoid enum when clients must add new cases externally.
Avoid subclassing unless customization by inheritance is explicitly required.
```

Debugging checklist:

```text
Is this bug caused by shared reference aliasing?
Should this model be a value instead of a class?
Are multiple booleans/optionals encoding one enum state?
Is this protocol a real abstraction or ceremony?
Is this mutable state accessed across tasks?
Should this be actor-isolated?
Is actor reentrancy causing stale state after an await?
Would a generic parameter preserve type information better than any Protocol?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Structs are for data, classes are for objects, enums are for choices, protocols are for interfaces, and actors are for concurrency.

### Senior answer

> I choose based on semantics. Structs are for values, enums are for closed state, classes are for identity and shared lifetime, protocols are for behavioral boundaries, and actors are for isolated mutable state. I avoid defaulting to classes or protocols because they add aliasing, dynamic dispatch, mocking ceremony, and API surface.

### Staff-level answer

> I treat type choice as architecture. I want most domain state to be values, invalid states represented as enums, identity isolated to the few places that truly need reference semantics, protocols placed at module or dependency boundaries, and shared mutable state protected by actors or deliberately scoped synchronization. I also consider public API resilience, Sendable, testability, performance, and whether an abstraction preserves or erases useful type information.

Staff-level questions to ask:

```text
Is this type modeling data, identity, state, behavior, or synchronization?
Who owns mutation?
Can invalid states be made unrepresentable?
Will this abstraction leak into public API?
Does this protocol preserve useful type relationships or erase them too early?
Is shared mutable state protected by language-enforced isolation or manual discipline?
Will this design survive Swift 6 strict concurrency?
Can this be tested without over-mocking?
```

---

## 9. Interview-ready summary

In Swift, I choose the type that matches the semantic contract. `struct` is my default for domain values because copies are independent and the model is easier to reason about. `enum` is for closed finite states and associated data, especially to make invalid states unrepresentable. `class` is for real identity, shared lifetime, weak references, framework interop, inheritance, or deinitialization behavior. `protocol` is for capability boundaries, not for wrapping every concrete type. `actor` is for shared mutable state that must be isolated across concurrent tasks. The senior-level judgment is knowing not only what each type does, but what complexity it introduces.

---

## 10. Flashcards

Q: What is the default type choice for ordinary domain data in Swift?  
A: `struct`, because domain data usually wants value semantics, independent copies, easier testing, and safer concurrency.

Q: When is a class justified?  
A: When identity, shared mutable lifetime, weak references, inheritance, Objective-C/framework interop, or deinit-based resource management is part of the design.

Q: Why are enums better than several booleans for request state?  
A: Enums encode mutually exclusive states directly and make invalid state combinations unrepresentable.

Q: When should you introduce a protocol?  
A: When there is a real behavior boundary with multiple implementations, test substitution, module separation, or generic capability modeling.

Q: When is an actor better than a class with mutable state?  
A: When state is shared across concurrent tasks and you want compiler-enforced isolation rather than relying on manual synchronization discipline.

Q: What do actors not solve?  
A: Actors prevent direct data races on actor-isolated state, but they do not automatically prevent stale reads, reentrancy bugs, poor workflow design, or all logical races.

Q: Why is “protocol for every model” a smell?  
A: It adds abstraction without substitution, increases API surface, can erase useful type information, and makes code harder to navigate.

Q: Why is “class for every model” a smell?  
A: It introduces reference aliasing, ARC lifetime issues, retain-cycle risk, and harder concurrency reasoning without necessarily modeling real identity.

---

## 11. Related sections

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A9 — Enums, associated values, recursive enums, and state modeling]]
- [[B1 — Protocols, associated types, Self requirements, and compositions]]
- [[B2 — Existentials (any) vs opaque types (some)]]
- [[D4 — Actors and actor isolation]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]

---

## 12. Sources

- "Swift Senior/Staff Rubric and Prioritized Study Checklist." A15 type choices.
- Swift.org. "Structures and Classes." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/
- Swift.org. "Enumerations." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/
- Swift.org. "Protocols." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/
- Swift.org. "Concurrency." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- Swift.org. "Actor-Isolated Call." Swift Compiler Diagnostics. https://docs.swift.org/compiler/documentation/diagnostics/actor-isolated-call/
- Apple Developer. "Sendable." Apple Developer Documentation. https://developer.apple.com/documentation/Swift/Sendable
- Swift.org. "API Design Guidelines." Swift.org Documentation. https://swift.org/documentation/api-design-guidelines/
