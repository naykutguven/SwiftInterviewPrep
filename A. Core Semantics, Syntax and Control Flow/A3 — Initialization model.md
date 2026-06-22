---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - initialization
---
## 0. Rubric snapshot

**Rubric expectation**

Know designated vs convenience initializers, two-phase initialization, memberwise initialization, failable initialization, `required`, and initialization safety rules. The rubric explicitly flags property observers, restricted use of `self`, and subclass ordering as the traps to understand.

**Caveats**

- Property observers generally do not run when a property is being initialized.
- You cannot freely use `self` before all stored properties for the current initialization phase are initialized.
- In class hierarchies, subclass stored properties must be initialized before delegating upward to `super.init`.
- Convenience initializers must delegate before touching instance state.

**You should be able to answer**

- Why can’t you access `self` freely before all stored properties are initialized?
- When does Swift synthesize a memberwise initializer, and when does it disappear?

**You should be able to do**

- Given a class hierarchy with stored properties, identify which initializer must call `super.init` first and why.

---

## 1. Core mental model

Initialization is Swift’s proof that an instance starts life in a valid state. Before an instance can be used, every stored property must have a value, class inheritance must initialize each layer correctly, and no method or property access may observe a partially formed object. Swift’s safety model depends on “initialized before use,” which Apple also advertises as one of Swift’s safety properties. ([Apple Developer](https://developer.apple.com/swift/?utm_source=chatgpt.com "Swift - Apple Developer"))

For structs and enums, initialization is mostly about assigning all stored properties before use. For classes, initialization is more complex because an instance is built across an inheritance chain. Swift class initialization is two-phase: first, every stored property in every class layer gets initialized; second, each layer may customize the fully initialized instance. Swift’s compiler performs safety checks for this model, including the rule that a designated initializer initializes its own stored properties before delegating to a superclass initializer. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/?utm_source=chatgpt.com "Initialization - Documentation | Swift.org"))

Designated initializers are the primary initialization paths for a class. Convenience initializers are secondary paths that must eventually delegate to a designated initializer in the same class. Convenience initializers cannot call superclass initializers directly; their job is to normalize arguments and funnel construction into the designated path. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations | Documentation - Swift Programming Language"))

The key idea:

```text
Initialization is not “running init code”; it is Swift proving that no code can observe invalid storage.
```

Swift guarantees that stored properties are initialized before normal instance use. Swift does **not** guarantee that observers, computed properties, overridden methods, or arbitrary instance methods are safe to use before initialization completes.

---

## 2. Essential mechanics

### Designated initializers initialize the layer they introduce

A class designated initializer must initialize all stored properties introduced by that class before calling `super.init`.

Bad:

```swift
import Foundation

class Resource {
    let id: String

    init(id: String) {
        self.id = id
    }
}

final class ImageResource: Resource {
    let url: URL

    init(id: String, url: URL) {
        super.init(id: id)
        self.url = url
    }
}
```

Swift 6.2 compiler error:

```text
hierarchy_bad.swift:16:18: error: immutable value 'self.url' may only be initialized once
10 | 
11 | final class ImageResource: Resource {
12 |     let url: URL
   |     `- note: change 'let' to 'var' to make it mutable
13 | 
14 |     init(id: String, url: URL) {
15 |         super.init(id: id)
16 |         self.url = url
   |                  `- error: immutable value 'self.url' may only be initialized once
17 |     }
18 | }

hierarchy_bad.swift:15:15: error: property 'self.url' not initialized at super.init call
13 | 
14 |     init(id: String, url: URL) {
15 |         super.init(id: id)
   |               `- error: property 'self.url' not initialized at super.init call
16 |         self.url = url
17 |     }
```

Better:

```swift
import Foundation

class Resource {
    let id: String

    init(id: String) {
        self.id = id
    }
}

final class ImageResource: Resource {
    let url: URL

    init(id: String, url: URL) {
        self.url = url       // initialize subclass storage first
        super.init(id: id)   // then initialize superclass storage
    }
}
```

The subclass must initialize `url` first because `url` belongs to `ImageResource`. Then `Resource` initializes `id`.

### Convenience initializers delegate sideways before touching state

A convenience initializer is not allowed to partially initialize the class itself. It must call another initializer in the same class first, and that chain must eventually reach a designated initializer. Swift’s declarations documentation states that convenience initializers delegate to another convenience initializer or a designated initializer, and the process must end at a designated initializer. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations | Documentation - Swift Programming Language"))

```swift
final class Endpoint {
    let scheme: String
    let host: String
    let path: String

    init(scheme: String, host: String, path: String) {
        self.scheme = scheme
        self.host = host
        self.path = path
    }

    convenience init(host: String, path: String) {
        self.init(scheme: "https", host: host, path: path)
    }
}
```

The convenience initializer does not initialize `scheme`, `host`, or `path` directly. It converts a simpler API into the designated initializer.

### `self` is restricted until initialization is safe

Before all stored properties are initialized, using `self` as a value, reading instance properties, or calling instance methods is unsafe. The reason is dynamic behavior: an instance method may depend on state that has not been assigned yet.

Bad:

```swift
import Foundation

struct User {
    let id: Int
    let displayName: String

    init(rawName: String) {
        self.displayName = normalize(rawName)
        self.id = 1
    }

    func normalize(_ name: String) -> String {
        name.trimmingCharacters(in: .whitespacesAndNewlines)
    }
}
```

Swift 6.2 compiler error:

```text
self_before_init.swift:8:28: error: 'self' used before all stored properties are initialized
 2 | 
 3 | struct User {
 4 |     let id: Int
   |         `- note: 'self.id' not initialized
 5 |     let displayName: String
   |         `- note: 'self.displayName' not initialized
 6 | 
 7 |     init(rawName: String) {
 8 |         self.displayName = normalize(rawName)
   |                            `- error: 'self' used before all stored properties are initialized
 9 |         self.id = 1
10 |     }
```

Better:

```swift
import Foundation

struct User {
    let id: Int
    let displayName: String

    init(rawName: String) {
        self.displayName = Self.normalize(rawName)
        self.id = 1
    }

    private static func normalize(_ name: String) -> String {
        name.trimmingCharacters(in: .whitespacesAndNewlines)
    }
}
```

A static helper does not need an initialized instance, so it is safe.

### Memberwise initialization is synthesized for structs, with caveats

Swift gives structures a memberwise initializer by default. Stored properties with default values can be omitted from the memberwise call. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/?utm_source=chatgpt.com "Initialization - Documentation | Swift.org"))

```swift
struct Size {
    var width = 0.0
    var height = 0.0
}

let explicit = Size(width: 100, height: 200)
let defaulted = Size(height: 200)
```

The synthesized memberwise initializer disappears when you declare a custom initializer inside the main struct declaration:

```swift
struct User {
    let id: Int
    let name: String

    init(id: Int) {
        self.id = id
        self.name = "Guest"
    }
}

let user = User(id: 1, name: "Aykut")
```

Swift 6.2 compiler error:

```text
memberwise.swift:11:30: error: extra argument 'name' in call
 9 | }
10 | 
11 | let user = User(id: 1, name: "Aykut")
   |                              `- error: extra argument 'name' in call
12 | print(user)
13 | 
```

To keep the synthesized memberwise initializer, put custom initializers in an extension:

```swift
struct User {
    let id: Int
    let name: String
}

extension User {
    init(id: Int) {
        self.init(id: id, name: "Guest")
    }
}

let named = User(id: 1, name: "Aykut")
let guest = User(id: 2)
```

Also, access control matters. The synthesized memberwise initializer is `internal` by default unless private/file-private stored properties reduce its visibility; it is not automatically public for a public struct. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/?utm_source=chatgpt.com "Access Control | Documentation - Swift Programming Language"))

### Failable initializers model invalid construction

Use `init?` when construction can fail and the failure is an expected part of the domain.

```swift
struct NonEmptyString {
    let value: String

    init?(_ value: String) {
        guard !value.isEmpty else {
            return nil
        }

        self.value = value
    }
}

let valid = NonEmptyString("Aykut") // NonEmptyString?
let invalid = NonEmptyString("")    // nil
```

Use this for local validation where `nil` is enough information. Use `throws` when callers need a reason.

```swift
enum UserIDError: Error {
    case empty
    case containsWhitespace
}

struct UserID {
    let value: String

    init(_ value: String) throws {
        guard !value.isEmpty else { throw UserIDError.empty }
        guard !value.contains(where: \.isWhitespace) else {
            throw UserIDError.containsWhitespace
        }

        self.value = value
    }
}
```

### `required` propagates initializer obligations through subclasses

A `required` initializer means every subclass must provide or inherit that initializer. This is common when a protocol requires an initializer on a non-final class. The Swift protocols documentation notes that a class implementation of a protocol initializer requirement is marked `required` so subclasses also satisfy the protocol. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/?utm_source=chatgpt.com "Protocols - Documentation | Swift.org"))

```swift
protocol DecodableModel {
    init(payload: [String: Any])
}

class Model: DecodableModel {
    required init(payload: [String: Any]) {
        // initialize base model
    }
}

final class FinalModel: DecodableModel {
    init(payload: [String: Any]) {
        // final classes have no subclasses to force
    }
}
```

`required` is about subclassing obligations, not about “this initializer is important.”

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `super.init` always comes first

In Swift, a subclass designated initializer initializes its own stored properties before calling `super.init`.

Bad:

```swift
final class AvatarImage: Resource {
    let url: URL

    init(id: String, url: URL) {
        super.init(id: id)
        self.url = url
    }
}
```

Better:

```swift
final class AvatarImage: Resource {
    let url: URL

    init(id: String, url: URL) {
        self.url = url
        super.init(id: id)
    }
}
```

This differs from many Objective-C habits. In Swift, each class layer proves its own storage is initialized before handing off to the superclass.

### Trap 2: Expecting `didSet` during initialization

Property observers are not called when a variable or property is first initialized; they run when the value is set outside the initialization context. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations | Documentation - Swift Programming Language"))

```swift
final class UserProfile {
    var name: String {
        didSet { print("didSet", name) }
    }

    init(name: String) {
        self.name = name
    }

    func rename(to name: String) {
        self.name = name
    }
}

let profile = UserProfile(name: "Aykut")
profile.rename(to: "Ada")
```

Output:

```text
didSet Ada
```

No observer fires for `"Aykut"` because that assignment establishes initial storage.

Nuance: superclass property observers can run when an inherited property is assigned in a subclass initializer **after** `super.init` has returned. That is not the same as a type observing its own initial stored-property assignment.

### Trap 3: Hiding construction failure with optionals everywhere

Bad:

```swift
struct PaymentRequest {
    var amount: Decimal?
    var currencyCode: String?
}
```

This allows invalid intermediate states to leak into the rest of the codebase.

Better:

```swift
struct PaymentRequest {
    let amount: Decimal
    let currencyCode: String

    init?(amount: Decimal, currencyCode: String) {
        guard amount > 0 else { return nil }
        guard currencyCode.count == 3 else { return nil }

        self.amount = amount
        self.currencyCode = currencyCode
    }
}
```

Even better if callers need diagnostics:

```swift
struct PaymentRequest {
    let amount: Decimal
    let currencyCode: String

    init(amount: Decimal, currencyCode: String) throws {
        guard amount > 0 else { throw ValidationError.invalidAmount }
        guard currencyCode.count == 3 else { throw ValidationError.invalidCurrency }

        self.amount = amount
        self.currencyCode = currencyCode
    }
}

enum ValidationError: Error {
    case invalidAmount
    case invalidCurrency
}
```

### Trap 4: Treating memberwise initializers as stable API

A synthesized memberwise initializer is convenient for internal models, but it is often a poor public API. Adding, removing, reordering, renaming, or changing access of stored properties changes the initializer surface.

Bad public SDK shape:

```swift
public struct AnalyticsEvent {
    public var name: String
    public var timestamp: Date
    public var metadata: [String: String]
}
```

Better:

```swift
public struct AnalyticsEvent {
    public let name: String
    public let metadata: [String: String]
    public let timestamp: Date

    public init(
        name: String,
        metadata: [String: String] = [:],
        timestamp: Date = Date()
    ) {
        self.name = name
        self.metadata = metadata
        self.timestamp = timestamp
    }
}
```

Explicit initializers protect the public API from storage layout churn.

---

## 4. Direct answers to rubric questions

### Q1. Why can’t you access `self` freely before all stored properties are initialized?

Because `self` represents the whole instance, and before stored properties are initialized, the whole instance does not yet exist in a valid state. Calling an instance method, reading a computed property, passing `self` to another function, or triggering dynamic dispatch could observe uninitialized storage.

Interview version:

> Swift restricts `self` before initialization completes because an instance method or property access can observe the object as a whole. Until all stored properties for the current phase are initialized, the object is only partially formed. Swift allows direct assignment to stored properties, but it does not allow general instance use because that would break initialized-before-use safety.

### Q2. When does Swift synthesize a memberwise initializer, and when does it disappear?

Swift synthesizes a memberwise initializer for structs that do not define their own initializer in the main struct declaration. Stored properties become initializer parameters, and properties with defaults can be omitted. It disappears if you define a custom initializer inside the struct body. It remains available if you define custom initializers in an extension. Access control can also make it less visible than you expect. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/?utm_source=chatgpt.com "Initialization - Documentation | Swift.org"))

Interview version:

> Structs get a synthesized memberwise initializer as long as the primary type declaration does not define a custom initializer. If I add a custom initializer inside the struct, Swift assumes I’m taking control of construction and stops synthesizing the memberwise initializer. If I want both, I keep the stored-property declaration simple and put convenience initializers in extensions. For public types, I usually write explicit initializers because synthesized memberwise init is not a stable API contract.

### Q3. Which initializer must call `super.init` first in a class hierarchy?

A subclass **designated** initializer must initialize its own stored properties first, then call a superclass designated initializer. A subclass **convenience** initializer must call another initializer in the same class first, not `super.init`. After `super.init`, the subclass initializer may customize inherited properties.

Interview version:

> In Swift, the subclass designated initializer initializes the subclass’s own stored properties before calling `super.init`. That proves the subclass layer is safe before the superclass initializer runs. Convenience initializers do not call `super.init` directly; they delegate sideways to another initializer in the same class, eventually reaching a designated initializer.

---

## 5. Code probe

The rubric has no A3 code probe. Here are three focused probes.

### Probe 1: Property observers during initialization

Given:

```swift
final class UserProfile {
    var name: String {
        didSet { print("didSet", name) }
    }

    init(name: String) {
        self.name = name
    }

    func rename(to name: String) {
        self.name = name
    }
}

let profile = UserProfile(name: "Aykut")
profile.rename(to: "Ada")
```

Output:

```text
didSet Ada
```

Why:

```text
init assignment:
self.name = "Aykut"   -> initializes storage directly; no didSet

post-init mutation:
self.name = "Ada"     -> normal property assignment; didSet runs
```

### Probe 2: Using `self` too early

Given:

```swift
import Foundation

struct User {
    let id: Int
    let displayName: String

    init(rawName: String) {
        self.displayName = normalize(rawName)
        self.id = 1
    }

    func normalize(_ name: String) -> String {
        name.trimmingCharacters(in: .whitespacesAndNewlines)
    }
}
```

Compiler error:

```text
self_before_init.swift:8:28: error: 'self' used before all stored properties are initialized
 2 | 
 3 | struct User {
 4 |     let id: Int
   |         `- note: 'self.id' not initialized
 5 |     let displayName: String
   |         `- note: 'self.displayName' not initialized
 6 | 
 7 |     init(rawName: String) {
 8 |         self.displayName = normalize(rawName)
   |                            `- error: 'self' used before all stored properties are initialized
 9 |         self.id = 1
10 |     }
```

Fix:

```swift
import Foundation

struct User {
    let id: Int
    let displayName: String

    init(rawName: String) {
        self.displayName = Self.normalize(rawName)
        self.id = 1
    }

    private static func normalize(_ name: String) -> String {
        name.trimmingCharacters(in: .whitespacesAndNewlines)
    }
}
```

### Probe 3: Subclass initializer ordering

Given:

```swift
import Foundation

class Resource {
    let id: String

    init(id: String) {
        self.id = id
    }
}

final class ImageResource: Resource {
    let url: URL

    init(id: String, url: URL) {
        super.init(id: id)
        self.url = url
    }
}
```

Compiler error:

```text
hierarchy_bad.swift:16:18: error: immutable value 'self.url' may only be initialized once
10 | 
11 | final class ImageResource: Resource {
12 |     let url: URL
   |     `- note: change 'let' to 'var' to make it mutable
13 | 
14 |     init(id: String, url: URL) {
15 |         super.init(id: id)
16 |         self.url = url
   |                  `- error: immutable value 'self.url' may only be initialized once
17 |     }
18 | }

hierarchy_bad.swift:15:15: error: property 'self.url' not initialized at super.init call
13 | 
14 |     init(id: String, url: URL) {
15 |         super.init(id: id)
   |               `- error: property 'self.url' not initialized at super.init call
16 |         self.url = url
17 |     }
```

Corrected:

```swift
import Foundation

final class ImageResource: Resource {
    let url: URL

    init(id: String, url: URL) {
        self.url = url
        super.init(id: id)
    }
}
```

---

## 6. Exercise

### Problem

Given a class hierarchy with stored properties, identify which initializer must call `super.init` first and why.

### Bad / naive version

```swift
import Foundation

class MediaItem {
    let id: UUID
    var title: String

    init(id: UUID, title: String) {
        self.id = id
        self.title = title
    }
}

final class PodcastEpisode: MediaItem {
    let audioURL: URL
    let duration: TimeInterval

    init(id: UUID, title: String, audioURL: URL, duration: TimeInterval) {
        super.init(id: id, title: title)
        self.audioURL = audioURL
        self.duration = duration
    }

    convenience init(title: String, audioURL: URL) {
        self.audioURL = audioURL
        self.init(
            id: UUID(),
            title: title,
            audioURL: audioURL,
            duration: 0
        )
    }
}
```

### What is wrong?

```text
1. PodcastEpisode's designated initializer calls super.init before initializing audioURL and duration.
2. The convenience initializer tries to assign stored property audioURL directly before delegating.
3. Convenience initializers must delegate to another initializer in the same class before touching instance state.
4. The designated initializer owns initialization of PodcastEpisode's stored properties.
5. MediaItem owns initialization of MediaItem's stored properties.
```

### Improved version

```swift
import Foundation

class MediaItem {
    let id: UUID
    var title: String

    init(id: UUID, title: String) {
        self.id = id
        self.title = title
    }
}

final class PodcastEpisode: MediaItem {
    let audioURL: URL
    let duration: TimeInterval

    init(
        id: UUID,
        title: String,
        audioURL: URL,
        duration: TimeInterval
    ) {
        self.audioURL = audioURL
        self.duration = duration
        super.init(id: id, title: title)
    }

    convenience init(title: String, audioURL: URL) {
        self.init(
            id: UUID(),
            title: title,
            audioURL: audioURL,
            duration: 0
        )
    }
}
```

### Why this is better

`PodcastEpisode.init(id:title:audioURL:duration:)` is the designated initializer. It initializes the stored properties introduced by `PodcastEpisode`, then delegates upward to `MediaItem.init`. The convenience initializer does not touch storage directly; it normalizes missing arguments and delegates to the designated initializer.

```text
PodcastEpisode designated init
├─ initialize PodcastEpisode.audioURL
├─ initialize PodcastEpisode.duration
└─ call MediaItem designated init
   ├─ initialize MediaItem.id
   └─ initialize MediaItem.title
```

This makes ownership of initialization clear and prevents partially initialized objects from being observed.

---

## 7. Production guidance

Use this in production when:

```text
- Creating domain models that must never exist in invalid states.
- Designing SDK/public initializers where construction is part of the API contract.
- Building UIKit/AppKit subclasses where superclass initialization order matters.
- Modeling validation with init?, init throws, or explicit factories.
- Keeping struct memberwise initialization internal while exposing stable public initializers.
```

Be careful when:

```text
- Adding custom initializers to structs that previously relied on synthesized memberwise init.
- Using property wrappers, because generated storage can change initializer shape.
- Writing subclass initializers that assign inherited properties.
- Using property observers for setup logic.
- Exposing memberwise-like public initializers for types whose storage may evolve.
```

Avoid when:

```text
- Using optionals just to bypass initialization rules.
- Calling instance methods from init before all stored properties are assigned.
- Calling overridable methods from initializers.
- Using implicitly unwrapped optionals as a routine way to escape initialization design.
- Treating convenience initializers as partial designated initializers.
```

Debugging checklist:

```text
- Which stored properties belong to this type?
- Which stored properties belong to the superclass?
- Is this initializer designated or convenience?
- Is self being used as a whole instance before all storage is initialized?
- Is a property observer expected to run during initialization?
- Did a custom struct initializer remove synthesized memberwise initialization?
- Is construction failure better modeled as init?, throws, or a factory?
- Is the initializer public API or merely internal construction convenience?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift initializers set all properties. Classes call `super.init`. Structs get memberwise initializers.

### Senior answer

> Swift initialization is a safety system. A designated initializer must initialize the properties introduced by its type, then delegate up. Convenience initializers delegate sideways first. You cannot use `self` freely before full initialization because methods and computed properties could observe incomplete state. For structs, memberwise initialization is synthesized unless you take control with a custom initializer in the type body.

### Staff-level answer

> Initialization is part of API and invariant design. I distinguish storage initialization from semantic validation, choose `init?`, `throws`, factories, or builders based on caller needs, and avoid exposing storage-shaped public initializers. In class hierarchies, I keep designated paths minimal and explicit, avoid overridable behavior in initialization, and treat memberwise initializers as an internal convenience rather than a public contract. When reviewing code, I look for optionals or IUOs used to dodge initialization rules because those usually move invalid state from compile time to runtime.

Staff-level questions to ask:

```text
1. Is this initializer enforcing a real invariant or merely assigning storage?
2. Should invalid input produce nil, a typed error, or a separate validation result?
3. Is this public initializer stable if stored properties change?
4. Are we relying on property observers for initial setup?
5. Does subclassing add value here, or would composition avoid initialization complexity?
```

---

## 9. Interview-ready summary

Swift initialization proves that instances start in a valid state. Structs get synthesized memberwise initializers unless custom initializers in the main declaration take over. Classes use designated initializers for primary construction and convenience initializers for delegation. In a subclass designated initializer, Swift requires initializing the subclass’s stored properties before calling `super.init`, then the superclass initializes its own layer. Swift restricts `self` before full initialization because methods, computed properties, or dynamic dispatch could observe uninitialized storage. In production, good initializer design is about preserving invariants, choosing the right failure model, and avoiding public APIs that expose storage layout.

---

## 10. Flashcards

Q: What is Swift initialization really proving?  
A: That every stored property has a valid initial value before the instance can be used normally.

Q: In a subclass designated initializer, what comes first: subclass stored properties or `super.init`?  
A: Subclass stored properties come first, then `super.init`.

Q: Why can’t an initializer call instance methods before all stored properties are initialized?  
A: Instance methods use `self`, and `self` represents the whole instance. The method could observe uninitialized state.

Q: When does a struct lose its synthesized memberwise initializer?  
A: When it declares a custom initializer inside the main struct declaration.

Q: How can you keep memberwise initialization while adding custom initializers?  
A: Put the custom initializers in an extension.

Q: Do property observers run when a property receives its initial value?  
A: No. Initial assignment sets storage directly; observers run for later assignments outside that initialization context.

Q: What is a convenience initializer allowed to do before delegating?  
A: It can compute arguments, but it cannot assign instance properties or use `self` as an initialized instance before delegating.

Q: When should you use `init?` instead of `throws`?  
A: Use `init?` when failure is expected and `nil` is enough information; use `throws` when the caller needs a reason.

Q: What does `required init` mean?  
A: Subclasses must provide or inherit that initializer, commonly to preserve protocol conformance across a class hierarchy.

Q: Why are synthesized memberwise initializers risky in public APIs?  
A: They mirror stored properties, so storage changes can become source-breaking API changes.

---

## 11. Related sections

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A4 — Optionals and nil modeling]]
- [[A15 — Type choices: struct, class, enum, protocol, and actor]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B13 — Property wrappers and generated storage semantics]]
- [[C1 — ARC fundamentals]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist.
- Swift Programming Language — Initialization. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/?utm_source=chatgpt.com "Initialization - Documentation | Swift.org"))
- Swift Programming Language — Declarations. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations | Documentation - Swift Programming Language"))
- Swift Programming Language — Access Control. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/?utm_source=chatgpt.com "Access Control | Documentation - Swift Programming Language"))
- Swift Programming Language — Protocols. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/?utm_source=chatgpt.com "Protocols - Documentation | Swift.org"))
- Swift Programming Language — Property observers in declarations. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations | Documentation - Swift Programming Language"))