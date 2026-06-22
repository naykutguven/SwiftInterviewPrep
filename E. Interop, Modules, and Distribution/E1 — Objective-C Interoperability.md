---
tags:
  - swift
  - ios
  - interview-prep
  - interop
  - objc
  - runtime
  - api-design
---
## 0. Rubric snapshot

**Rubric expectation**

Understand `@objc`, `dynamic`, selector-based APIs, `NSObject` constraints, `NSError` bridging, and which Swift features do not map cleanly to Objective-C. The rubric frames this as a senior-level interop, modules, and distribution topic.

**Caveats**

Opting into Objective-C runtime behavior affects:

- dispatch
- reflection
- selectors
- KVC/KVO
- runtime names
- API shape
- binary size and optimization opportunities

**You should be able to answer**

- When do you need `@objc` or `dynamic`, and what are the tradeoffs?
- Why do some Swift language features not export cleanly to Objective-C?

**You should be able to do**

- Expose a Swift API to Objective-C.
- Identify what must change about its signature and types.

---

## 1. Core mental model

Objective-C interoperability is not “Swift with a different syntax.” It is Swift crossing into a different runtime model.

Swift’s native model is strongly typed, generic, value-semantics-friendly, statically optimized, and increasingly concurrency-aware. Objective-C’s model is class/object based, selector based, dynamically dispatched through the Objective-C runtime, nullable by convention, and centered around `NSObject`, `id`, `SEL`, blocks, `NSError`, KVC/KVO, and runtime metadata.

The key distinction:

```text
@objc = expose Swift declaration to Objective-C/runtime metadata
dynamic = force Objective-C runtime dispatch for that declaration
```

`@objc` makes a declaration visible to Objective-C when it can be represented there. Swift’s reference manual says `@objc` applies only to declarations representable in Objective-C, such as nonnested classes, protocols, nongeneric integer-backed enums, class/protocol methods and properties, initializers, and subscripts. It also notes that the compiler can infer `@objc` in cases such as overriding Objective-C declarations, satisfying `@objc` protocol requirements, and Interface Builder annotations like `IBAction`/`IBOutlet`. ([Swift.org, "Attributes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/))

`dynamic` goes further: it forces access through Objective-C runtime dispatch, meaning the compiler must not inline or devirtualize the access. Swift’s reference manual explicitly says `dynamic` members are always dynamically dispatched through the Objective-C runtime and therefore must also be `@objc`. ([Swift.org, "Declarations"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/))

Use Objective-C interop deliberately. It is necessary for UIKit/AppKit target-action, Interface Builder, some delegate patterns, KVC/KVO, runtime reflection, old SDK surfaces, mixed Swift/Objective-C codebases, and SDKs that must serve Objective-C clients. It is not a general-purpose API design style for new Swift-only code.

---

## 2. Essential mechanics

### `@objc`: Objective-C visibility and runtime name

`@objc` tells the compiler to expose a compatible Swift declaration to Objective-C.

```swift
import UIKit

final class SearchButtonController: NSObject {
    @objc private func didTapSearchButton(_ sender: UIButton) {
        print("Search tapped")
    }

    func connect(_ button: UIButton) {
        button.addTarget(
            self,
            action: #selector(didTapSearchButton(_:)),
            for: .touchUpInside
        )
    }
}
```

Why `@objc` is needed here:

```text
UIButton target-action stores a selector.
A selector names an Objective-C runtime method.
Swift-only private methods are not selector-addressable unless exposed to ObjC.
```

`@objc(SomeName)` can also control the Objective-C name and runtime name. Swift’s reference manual notes that the `@objc` attribute accepts a name argument, and that this name is used both in Objective-C code and as the runtime name for APIs like `NSClassFromString`. ([Swift.org, "Attributes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/))

Example:

```swift
@objc(ACMUserSession)
public final class UserSessionObjC: NSObject {
    @objc public let identifier: String

    @objc(initWithIdentifier:)
    public init(identifier: String) {
        self.identifier = identifier
    }
}
```

Objective-C sees roughly:

```objc
@interface ACMUserSession : NSObject
- (instancetype)initWithIdentifier:(NSString *)identifier;
@property (nonatomic, readonly) NSString *identifier;
@end
```

### `dynamic`: force Objective-C dispatch

`@objc` exposes the symbol. It does not necessarily mean every Swift call site must use `objc_msgSend`.

`dynamic` forces dynamic runtime dispatch:

```swift
final class ObservableDownload: NSObject {
    @objc dynamic var progress: Double = 0
}
```

This is common for KVO-style observation because KVO depends on Objective-C runtime behavior.

```text
@objc dynamic var progress

means:

- expose property to Objective-C
- use Objective-C runtime dispatch
- do not inline/devirtualize access
- make it compatible with runtime observation mechanisms
```

Tradeoff: you give up some Swift compiler optimization freedom. Dynamic dispatch is also less explicit in call graphs and can make behavior depend on runtime features such as swizzling, KVO, or `responds(to:)`.

### `@objcMembers`: broad exposure

`@objcMembers` implicitly applies `@objc` to compatible members of a class, its extensions, subclasses, and their extensions.

```swift
@objcMembers
public final class LegacyUser: NSObject {
    public let id: String
    public var displayName: String

    public init(id: String, displayName: String) {
        self.id = id
        self.displayName = displayName
    }
}
```

Use this sparingly. Swift’s reference manual says most code should use `@objc` directly and expose only what is needed; `@objcMembers` is mainly a convenience for libraries that heavily use Objective-C runtime introspection, and unnecessary `@objc` exposure can increase binary size and hurt performance. ([Swift.org, "Attributes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/))

Better default:

```swift
public final class LegacyUser: NSObject {
    @objc public let id: String
    @objc public var displayName: String

    public init(id: String, displayName: String) {
        self.id = id
        self.displayName = displayName
    }
}
```

### `@nonobjc`: suppress accidental exposure

If a class or extension has implicit Objective-C exposure, use `@nonobjc` to keep Swift-only overloads or helpers out of Objective-C.

```swift
@objcMembers
final class FormatterBridge: NSObject {
    func format(_ value: NSNumber) -> NSString {
        "\(value)" as NSString
    }

    @nonobjc
    func format(_ value: Int) -> String {
        "\(value)"
    }
}
```

Swift’s reference manual describes `@nonobjc` as a way to suppress an implicit `@objc` attribute and make a declaration unavailable to Objective-C even if it could otherwise be represented there. It is also useful for resolving overload/circularity issues in `@objc` classes. ([Swift.org, "Attributes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/))

### `NSError` bridging

Objective-C conventionally reports errors through `NSError **` out parameters or completion-handler parameters. Swift maps many of these APIs to `throws`.

Objective-C:

```objc
- (nullable NSURL *)replaceItemAtURL:(NSURL *)url
                             options:(NSFileVersionReplacingOptions)options
                               error:(NSError **)error;
```

Swift import:

```swift
func replaceItem(at url: URL, options: ReplacingOptions = []) throws -> URL
```

Swift Evolution SE-0112 describes this direct interoperability: Objective-C methods with `NSError **` parameters import as throwing Swift methods, and Swift `Error`-conforming values bridge to `NSError`. ([GitHub, "Improved NSError Bridging"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0112-nserror-bridging.md))

For Swift errors that must cross Objective-C boundaries, prefer:

```swift
enum PaymentError: Int, Error, CustomNSError {
    case declined
    case networkUnavailable

    static var errorDomain: String {
        "com.acme.payment"
    }

    var errorCode: Int {
        rawValue
    }

    var errorUserInfo: [String: Any] {
        switch self {
        case .declined:
            [NSLocalizedDescriptionKey: "The payment was declined."]
        case .networkUnavailable:
            [NSLocalizedDescriptionKey: "The network is unavailable."]
        }
    }
}
```

This preserves an Objective-C-friendly domain/code/userInfo shape while keeping a Swift-native error model internally.

### Objective-C completion handlers and Swift concurrency

Objective-C does not have Swift’s native `async`/`await` language model, but Swift has interop rules for completion-handler APIs. SE-0297 says Objective-C completion-handler methods can be translated into Swift `async` methods, and Swift `async @objc` methods can be exported to Objective-C as completion-handler methods. ([GitHub, "Concurrency Interoperability with Objective-C"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md))

Swift:

```swift
@objc
func perform(operation: String) async -> Int {
    operation.count
}
```

Objective-C sees a completion-handler-shaped method roughly like:

```objc
- (void)performWithOperation:(NSString *)operation
           completionHandler:(void (^)(NSInteger result))completionHandler;
```

For `async throws`, Objective-C completion handlers receive an `NSError *` parameter. SE-0297 also states that the synthesized Objective-C implementation creates a detached task to call the Swift async method and forward the result to the completion handler. ([GitHub, "Concurrency Interoperability with Objective-C"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md))

Production caveat: this bridge is useful, but it is not a full Objective-C model of Swift structured concurrency. Be careful with cancellation, actor isolation, Sendable assumptions, and callback threading.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `@objc` means dynamic dispatch

Bad mental model:

```text
@objc means this method is always dynamically dispatched.
```

Better mental model:

```text
@objc exposes a declaration to Objective-C.
dynamic forces Objective-C runtime dispatch.
```

Bad:

```swift
final class Model: NSObject {
    @objc var title: String = ""
}
```

If the reason is KVO or runtime interception, this is usually insufficient.

Better:

```swift
final class Model: NSObject {
    @objc dynamic var title: String = ""
}
```

### Trap 2: Sprinkling `@objcMembers` everywhere

Bad:

```swift
@objcMembers
final class AccountManager: NSObject {
    var cache: [String: Account] = [:]
    var retryPolicy: RetryPolicy = .default
    var debugDescription: String { "..." }

    func loadAccount(id: String) async throws -> Account {
        // ...
    }

    func resetInternalStateForTests() {
        // ...
    }
}
```

Problems:

```text
- Exposes too much implementation detail.
- Some members cannot be represented anyway.
- Increases runtime metadata surface.
- Can create accidental Objective-C API compatibility promises.
- Can make refactoring harder.
```

Better:

```swift
final class AccountManagerObjC: NSObject {
    private let manager: AccountManager

    init(manager: AccountManager) {
        self.manager = manager
    }

    @objc(loadAccountWithID:completionHandler:)
    func loadAccount(
        id: String,
        completionHandler: @escaping (AccountObjC?, NSError?) -> Void
    ) {
        Task {
            do {
                let account = try await manager.loadAccount(id: id)
                completionHandler(AccountObjC(account), nil)
            } catch {
                completionHandler(nil, error as NSError)
            }
        }
    }
}
```

### Trap 3: Exposing Swift-native domain models directly

Bad:

```swift
public struct Account: Sendable {
    public let id: UUID
    public let status: AccountStatus
}

public enum AccountStatus: Sendable {
    case active
    case suspended(reason: String)
    case deleted(at: Date)
}
```

This is good Swift, but poor Objective-C API surface. Objective-C cannot naturally represent Swift structs, associated-value enums, `Sendable`, or strong generic relationships.

Better:

```swift
@objc(ACMAccountStatus)
public enum AccountStatusObjC: Int {
    case active
    case suspended
    case deleted
}

@objc(ACMAccount)
public final class AccountObjC: NSObject {
    @objc public let identifier: String
    @objc public let status: AccountStatusObjC
    @objc public let suspensionReason: String?
    @objc public let deletionDate: Date?

    init(_ account: Account) {
        identifier = account.id.uuidString

        switch account.status {
        case .active:
            status = .active
            suspensionReason = nil
            deletionDate = nil

        case .suspended(let reason):
            status = .suspended
            suspensionReason = reason
            deletionDate = nil

        case .deleted(let date):
            status = .deleted
            suspensionReason = nil
            deletionDate = date
        }
    }
}
```

This is less elegant than the Swift model, but more honest for Objective-C.

### Trap 4: Forgetting that Objective-C names are API

Swift overloading and labels often do not survive as-is across Objective-C.

Bad:

```swift
@objcMembers
final class ImageLoader: NSObject {
    func load(_ url: URL) { }
    func load(_ url: URL, cachePolicy: CachePolicy) { } // CachePolicy is Swift-only
}
```

Better:

```swift
final class ImageLoaderObjC: NSObject {
    @objc(loadImageWithURL:)
    func loadImage(url: URL) { }

    @objc(loadImageWithURL:cachePolicy:)
    func loadImage(url: URL, cachePolicy: Int) { }
}
```

When designing Objective-C-visible Swift APIs, explicitly choose the selector name. Do not rely on accidental generated names for public SDK boundaries.

---

## 4. Direct answers to rubric questions

### Q1. When do you need `@objc` or `dynamic`, and what are the tradeoffs?

You need `@objc` when Swift declarations must be visible to Objective-C code or Objective-C runtime mechanisms.

Typical cases:

```text
- UIKit/AppKit target-action
- #selector(...)
- Interface Builder: IBOutlet, IBAction, IBInspectable, IBDesignable
- Objective-C delegates or protocols
- KVC/KVO
- NSNotification selector observers
- Timer selector APIs
- Objective-C clients of a mixed SDK
- Runtime lookup with NSClassFromString / responds(to:) / perform(...)
```

Example:

```swift
final class LoginViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        navigationItem.rightBarButtonItem = UIBarButtonItem(
            title: "Login",
            style: .done,
            target: self,
            action: #selector(loginButtonTapped)
        )
    }

    @objc private func loginButtonTapped() {
        print("Login")
    }
}
```

You need `dynamic` when you specifically require Objective-C runtime dispatch, not just Objective-C visibility.

Typical cases:

```text
- KVO-observable properties
- method swizzling/interception
- Objective-C runtime replacement behavior
- cases where Swift devirtualization would break runtime expectations
```

Example:

```swift
final class PlayerState: NSObject {
    @objc dynamic var isPlaying = false
}
```

Tradeoffs:

|Choice|Meaning|Tradeoff|
|---|---|---|
|`@objc`|Expose compatible Swift declaration to ObjC/runtime|More runtime metadata, more API surface, ObjC-compatible type restrictions|
|`dynamic`|Force ObjC runtime dispatch|Prevents inlining/devirtualization; slower and more dynamic|
|`@objcMembers`|Broadly expose compatible members|Convenient but easy to overexpose; can increase binary size/perf cost|
|`@nonobjc`|Hide a declaration from ObjC|Useful for overloads/helpers but creates split Swift/ObjC API behavior|

Interview version:

> I use `@objc` when a Swift declaration must participate in Objective-C runtime mechanisms: selectors, target-action, KVC/KVO, Interface Builder, Objective-C protocols, or Objective-C clients. I use `dynamic` only when runtime dispatch itself is required, for example KVO or swizzling. `@objc` is visibility; `dynamic` is dispatch. The tradeoff is that Objective-C exposure constrains the API to Objective-C-representable types and adds runtime metadata, while `dynamic` also prevents Swift from optimizing the call through inlining or devirtualization.

### Q2. Why do some Swift language features not export cleanly to Objective-C?

Because Objective-C and Swift have different type systems and runtime models.

Objective-C understands:

```text
- classes and objects
- selectors
- protocols
- blocks
- pointers and nullability annotations
- NSObject / id / NSError
- integer-backed NS_ENUM-like enums
- NSArray / NSDictionary / NSString / NSNumber / NSData-style object graphs
```

Swift has features Objective-C cannot directly express:

```text
- structs with value semantics
- enums with associated values
- generic functions and generic value types
- protocols with associated types or Self requirements
- opaque result types: some P
- existential spelling and type relationships
- tuples
- nested types
- async/await as a language-level effect
- actors and actor isolation
- Sendable
- typed throws
- property wrappers / result builders / macros as Swift source features
```

Some of these can be adapted. For example:

```swift
enum Status {
    case active
    case suspended(reason: String)
}
```

can become:

```swift
@objc enum StatusObjC: Int {
    case active
    case suspended
}

final class StatusPayloadObjC: NSObject {
    @objc let status: StatusObjC
    @objc let reason: String?
}
```

But this is a lossy mapping. You have moved from a Swift enum that makes impossible states unrepresentable to an Objective-C-compatible object where invalid combinations can exist unless you enforce invariants manually.

Interview version:

> Some Swift features do not export cleanly because Objective-C does not have equivalent concepts. Swift structs, associated-value enums, generics, protocols with associated types, actors, `Sendable`, tuples, and opaque result types encode static type relationships that Objective-C cannot represent. When exposing Swift to Objective-C, I usually keep the Swift-native model internally and build a narrow Objective-C adapter layer using `NSObject` classes, `@objc` integer enums, `NSString`/`NSNumber`/Foundation types, blocks, and `NSError`.

---

## 5. Code examples

E1 has no rubric code probe, so here are a minimal example, a counterexample, and a production-style example.

### Minimal example: selector-based API

```swift
import UIKit

final class SearchController: NSObject {
    @objc private func didTapSearch(_ sender: UIButton) {
        print("Search tapped")
    }

    func wire(_ button: UIButton) {
        button.addTarget(
            self,
            action: #selector(didTapSearch(_:)),
            for: .touchUpInside
        )
    }
}
```

What happens:

```text
No output immediately.

When the button is tapped:
Search tapped
```

Why:

```text
UIButton stores a target and Objective-C selector.
#selector type-checks that didTapSearch(_:) exists and is ObjC-visible.
@objc exposes the Swift method to the Objective-C runtime.
```

### Counterexample: Swift-only method used as selector

```swift
import UIKit

final class SearchController {
    private func didTapSearch(_ sender: UIButton) {
        print("Search tapped")
    }

    func wire(_ button: UIButton) {
        button.addTarget(
            self,
            action: #selector(didTapSearch(_:)),
            for: .touchUpInside
        )
    }
}
```

Problem:

```text
The selector cannot refer to a pure Swift private method.
The method must be exposed to the Objective-C runtime.
```

Fix:

```swift
final class SearchController: NSObject {
    @objc private func didTapSearch(_ sender: UIButton) {
        print("Search tapped")
    }
}
```

### Production example: Swift-first API with Objective-C adapter

Swift-native core:

```swift
public struct PaymentRequest: Sendable {
    public let amount: Decimal
    public let currencyCode: String
}

public struct PaymentReceipt: Sendable {
    public let id: UUID
    public let amount: Decimal
    public let currencyCode: String
}

public enum PaymentProcessorError: Error, Sendable {
    case declined(reason: String)
    case networkUnavailable
}

public protocol PaymentProcessing: Sendable {
    func charge(_ request: PaymentRequest) async throws -> PaymentReceipt
}
```

Objective-C-compatible DTO:

```swift
@objc(ACMPaymentReceipt)
public final class PaymentReceiptObjC: NSObject {
    @objc public let identifier: String
    @objc public let amount: NSDecimalNumber
    @objc public let currencyCode: String

    init(_ receipt: PaymentReceipt) {
        self.identifier = receipt.id.uuidString
        self.amount = NSDecimalNumber(decimal: receipt.amount)
        self.currencyCode = receipt.currencyCode
    }
}
```

Objective-C-compatible wrapper:

```swift
@objc(ACMPaymentProcessor)
public final class PaymentProcessorObjC: NSObject {
    private let processor: PaymentProcessing

    public init(processor: PaymentProcessing) {
        self.processor = processor
    }

    @objc(chargeAmount:currencyCode:completionHandler:)
    public func charge(
        amount: NSDecimalNumber,
        currencyCode: String,
        completionHandler: @escaping (PaymentReceiptObjC?, NSError?) -> Void
    ) {
        let request = PaymentRequest(
            amount: amount.decimalValue,
            currencyCode: currencyCode
        )

        Task {
            do {
                let receipt = try await processor.charge(request)
                completionHandler(PaymentReceiptObjC(receipt), nil)
            } catch {
                completionHandler(nil, error as NSError)
            }
        }
    }
}
```

Objective-C sees a conventional API:

```objc
ACMPaymentProcessor *processor = ...;

[processor chargeAmount:amount
           currencyCode:@"EUR"
      completionHandler:^(ACMPaymentReceipt *receipt, NSError *error) {
          if (error != nil) {
              // Handle error.
              return;
          }

          NSLog(@"%@", receipt.identifier);
      }];
```

Important design choice:

```text
Swift API remains Swift-native.
Objective-C API is a narrow adapter.
The adapter maps:
- Swift struct -> NSObject DTO
- Swift Decimal -> NSDecimalNumber
- Swift UUID -> NSString
- Swift async throws -> completion handler with NSError
- Swift associated-value error -> NSError-compatible representation
```

---

## 6. Exercise

### Problem

Expose a Swift API to Objective-C and identify what must change about its signature and types.

### Bad / naive version

```swift
public struct Profile: Sendable {
    public let id: UUID
    public let name: String
    public let avatarURL: URL?
}

public enum ProfileState: Sendable {
    case loaded(Profile)
    case failed(ProfileLoadError)
}

public enum ProfileLoadError: Error, Sendable {
    case missingUserID
    case transport(Error)
    case decodingFailed(reason: String)
}

public final class ProfileService {
    public func loadProfile(userID: UUID) async throws -> Profile {
        fatalError("Implementation omitted")
    }
}
```

### What is wrong for Objective-C?

```text
Profile is a Swift struct.
UUID is not the natural Objective-C surface; use NSString or NSUUID.
ProfileState has associated values.
ProfileLoadError has associated values and nested Error payloads.
async/await is not native Objective-C syntax.
throws needs NSError bridging.
Sendable is Swift-only.
The class has no ObjC-facing name or NSObject-style surface.
```

### Improved Objective-C-facing version

Keep the Swift-native implementation:

```swift
public struct Profile: Sendable {
    public let id: UUID
    public let name: String
    public let avatarURL: URL?
}

public protocol ProfileLoading: Sendable {
    func loadProfile(userID: UUID) async throws -> Profile
}
```

Add an Objective-C DTO:

```swift
@objc(ACMProfile)
public final class ProfileObjC: NSObject {
    @objc public let identifier: String
    @objc public let name: String
    @objc public let avatarURL: URL?

    init(_ profile: Profile) {
        self.identifier = profile.id.uuidString
        self.name = profile.name
        self.avatarURL = profile.avatarURL
    }
}
```

Add an Objective-C error shape:

```swift
@objc(ACMProfileErrorCode)
public enum ProfileErrorCodeObjC: Int {
    case missingUserID = 1
    case transport = 2
    case decodingFailed = 3
}

enum ProfileLoadError: Error, CustomNSError {
    case missingUserID
    case transport(Error)
    case decodingFailed(reason: String)

    static let errorDomain = "com.acme.profile"

    var errorCode: Int {
        switch self {
        case .missingUserID:
            ProfileErrorCodeObjC.missingUserID.rawValue
        case .transport:
            ProfileErrorCodeObjC.transport.rawValue
        case .decodingFailed:
            ProfileErrorCodeObjC.decodingFailed.rawValue
        }
    }

    var errorUserInfo: [String: Any] {
        switch self {
        case .missingUserID:
            [
                NSLocalizedDescriptionKey: "The user ID is missing."
            ]

        case .transport(let underlying):
            [
                NSLocalizedDescriptionKey: "The profile request failed.",
                NSUnderlyingErrorKey: underlying
            ]

        case .decodingFailed(let reason):
            [
                NSLocalizedDescriptionKey: "The profile response could not be decoded.",
                "reason": reason
            ]
        }
    }
}
```

Add the Objective-C adapter:

```swift
@objc(ACMProfileService)
public final class ProfileServiceObjC: NSObject {
    private let loader: ProfileLoading

    public init(loader: ProfileLoading) {
        self.loader = loader
    }

    @objc(loadProfileWithUserID:completionHandler:)
    public func loadProfile(
        userID: String,
        completionHandler: @escaping (ProfileObjC?, NSError?) -> Void
    ) {
        guard let uuid = UUID(uuidString: userID) else {
            completionHandler(
                nil,
                ProfileLoadError.missingUserID as NSError
            )
            return
        }

        Task {
            do {
                let profile = try await loader.loadProfile(userID: uuid)
                completionHandler(ProfileObjC(profile), nil)
            } catch {
                completionHandler(nil, error as NSError)
            }
        }
    }
}
```

### Why this is better

```text
The Swift API stays expressive and type-safe.
The ObjC API is explicit and stable.
Swift-only concepts do not leak into Objective-C.
Error information survives through NSError.
The selector name is chosen deliberately.
The adapter localizes interop ugliness instead of contaminating the domain model.
```

### What changed?

|Swift-native API|Objective-C-facing API|Reason|
|---|---|---|
|`struct Profile`|`NSObject` DTO|ObjC consumes objects, not Swift value types|
|`UUID`|`String` or `NSUUID`|Better ObjC representation|
|associated-value enum|integer `@objc enum` + fields/userInfo|ObjC cannot model associated values|
|`async throws`|completion handler with `(Result?, NSError?)`|ObjC has no native async/throws syntax|
|Swift `Error` enum|`CustomNSError`|Preserve domain/code/userInfo|
|Swift protocol abstraction|adapter stores Swift dependency|Avoid leaking generic/Sendable/protocol details|
|inferred names|explicit `@objc(...)` selector|Stable public ObjC API|

---

## 7. Production guidance

Use Objective-C interop in production when:

```text
- You are integrating with existing Objective-C app code.
- You are exposing a Swift SDK to Objective-C clients.
- You need UIKit/AppKit target-action or Interface Builder.
- You need Objective-C delegates/protocols.
- You need KVC/KVO or runtime lookup.
- You are gradually migrating a legacy Objective-C codebase to Swift.
```

Be careful when:

```text
- Marking large classes @objcMembers.
- Exposing Swift domain models directly.
- Using dynamic dispatch in hot paths.
- Relying on KVC/KVO instead of explicit Swift state flow.
- Bridging Swift concurrency to Objective-C completion handlers.
- Exposing Objective-C APIs from multiple modules or binary frameworks.
```

Avoid when:

```text
- The API is Swift-only.
- You can use closures, protocols, async/await, or observation directly.
- You are only adding @objc to silence compiler errors without understanding why.
- You are exposing internal implementation details to Objective-C clients.
- You need Swift-only type guarantees like associated values, Sendable, actors, or generic constraints.
```

Debugging checklist:

```text
1. Is the member actually visible to Objective-C?
2. Is the selector name what you think it is?
3. Is the method private/fileprivate/internal/public in a way that affects exposure?
4. Is the type representable in Objective-C?
5. Does the class need NSObject or NSObjectProtocol conformance for this API?
6. Is KVO involved, requiring @objc dynamic?
7. Is the Swift name different from the Objective-C selector?
8. Are errors being bridged as NSError with useful domain/code/userInfo?
9. Are nullability and optionality represented correctly?
10. Is @objcMembers exposing more than intended?
11. Does Swift concurrency introduce actor/thread/cancellation assumptions that ObjC callers do not know about?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `@objc` lets Objective-C call Swift code, and `dynamic` makes dispatch dynamic.

### Senior answer

> `@objc` exposes compatible Swift declarations to Objective-C runtime metadata, which is required for selectors, target-action, KVC/KVO, Interface Builder, Objective-C protocols, and Objective-C clients. `dynamic` is separate: it forces Objective-C runtime dispatch and prevents inlining/devirtualization. I avoid broad `@objcMembers` exposure and prefer narrow adapter layers.

### Staff-level answer

> I treat Objective-C interop as a boundary design problem, not an annotation problem. The Swift side should remain Swift-native: value types, associated-value enums, async/await, actors, and strong type relationships where appropriate. The Objective-C boundary should be explicit: `NSObject` DTOs, integer `@objc` enums, Foundation types, blocks, and `NSError`. I design selectors deliberately, preserve error information through `CustomNSError`, minimize runtime exposure, and isolate migration risk so Objective-C compatibility does not pollute the core architecture.

Staff-level questions to ask:

```text
Who are the Objective-C clients, and how long must we support them?
Is this a public SDK boundary or only an internal migration bridge?
Can we keep Swift-native models internally and expose ObjC adapters externally?
Are selector names stable and documented?
Are errors localizable and inspectable from Objective-C?
Are we overusing @objcMembers?
Does this API need dynamic dispatch, or only Objective-C visibility?
Does KVO/KVC actually belong here, or should state flow be explicit?
How does this interop boundary interact with Swift 6 strict concurrency?
Will this Objective-C surface become binary/source compatibility debt?
```

---

## 9. Interview-ready summary

Objective-C interop is a boundary between Swift’s static, generic, value-oriented type system and Objective-C’s dynamic, selector-based object runtime. `@objc` exposes compatible Swift declarations to Objective-C; `dynamic` forces Objective-C runtime dispatch and prevents some Swift optimizations. I use `@objc` for selectors, Interface Builder, KVC/KVO, Objective-C protocols, and mixed SDKs, and I reserve `dynamic` for cases that truly require runtime dispatch. Swift features like structs, associated-value enums, generics, actors, `Sendable`, tuples, and opaque types do not map cleanly to Objective-C, so the best production design is usually a Swift-native core plus a narrow Objective-C adapter using `NSObject`, Foundation types, integer `@objc` enums, blocks, and `NSError`.

---

## 10. Flashcards

Q: What does `@objc` do?

A: It exposes a compatible Swift declaration to Objective-C code and Objective-C runtime metadata.

Q: What does `dynamic` do?

A: It forces Objective-C runtime dispatch and prevents the compiler from inlining or devirtualizing that member access.

Q: Is `@objc` the same as dynamic dispatch?

A: No. `@objc` is visibility/runtime metadata. `dynamic` is dispatch behavior.

Q: When do you commonly need `@objc` in UIKit?

A: Target-action, `#selector`, `IBAction`, `IBOutlet`, Objective-C delegates, KVC/KVO, timers, and selector-based notification observers.

Q: What is the risk of `@objcMembers`?

A: It exposes many compatible members automatically, increasing runtime/API surface and potentially hurting binary size and performance.

Q: Why do Swift associated-value enums not export cleanly to Objective-C?

A: Objective-C enums are integer constants; they cannot carry per-case payloads.

Q: How should a Swift struct be exposed to Objective-C?

A: Usually through an `NSObject` DTO/wrapper class.

Q: How does `NSError **` commonly import into Swift?

A: As a throwing Swift method.

Q: How should Swift errors be made useful to Objective-C clients?

A: Use `CustomNSError`, `LocalizedError`, stable domains/codes, and meaningful `userInfo`.

Q: What is the production pattern for mixed Swift/Objective-C APIs?

A: Keep a Swift-native core and expose a narrow Objective-C adapter layer.

---

## 11. Related sections

- [[B4 — Dispatch model]]
- [[B6 — Any, AnyObject, metatypes, and runtime type information]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[C1 — ARC fundamentals]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D13 — Swift 6 strict concurrency migration]]
- [[E2 — Foundation and Objective-C bridging behavior]]
- [[E3 — C interoperability and unsafe boundaries]]
- [[E7 — Resilience, ABI, and module stability]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — E1 — Objective-C interoperability.
- Swift.org. "Attributes." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/
- Swift.org. "Declarations." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/
- GitHub. "Improved NSError Bridging." Swift Evolution SE-0112. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0112-nserror-bridging.md
- GitHub. "Concurrency Interoperability with Objective-C." Swift Evolution SE-0297. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md
