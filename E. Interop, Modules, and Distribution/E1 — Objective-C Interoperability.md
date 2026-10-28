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

# E1 — Objective-C Interoperability

**Learning goals:**

- Separate Objective-C exposure, runtime dispatch, native Swift dynamic replacement, and observation lifetime.
- Export representable selectors/types/errors and verify the generated header with a real Objective-C client.
- Specify callback context, terminal delivery, cancellation, and token ownership around a Swift async core.

**Prerequisites:** [[A8 — Closures, Escaping, and Capture Semantics]] for closure ownership, [[A13 — Error handling and typed throws]] for errors, [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]] for dispatch, and [[C1 — ARC fundamentals]] for lifetime. [[D1 — Structured Concurrency Fundamentals]]/[[D2 — Task Hierarchy, Cancellation, and Priorities]]/[[D5 — Global Actors and `@MainActor`]]/[[D9 — Continuations and Bridging Callback APIs]] explain the async facade’s tasks, cancellation, MainActor, and callbacks.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An Objective-C caller does not see every Swift declaration. It sees an exported class and compatible selectors, object types, nullable values, blocks, and NSError conventions. Keep Swift-only generics and value models behind an explicit facade that those callers can construct and call.

A **selector** is a method’s Objective-C runtime name; greet: has one colon for its argument. `@objc` exposes a supported declaration to Objective-C; `@objc dynamic` also selects its runtime dispatch path. NSObject provides framework integration but does not expose every Swift method or synchronize mutable state. A generated header is the concrete surface to inspect, and a native consumer verifies whether it is usable. [Using Swift from Objective-C](https://developer.apple.com/documentation/swift/importing-swift-into-objective-c)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Export a representable synchronous operation

Shared GreetingSDK source. It has public construction and a stateless String operation; no UI/actor state needs a special calling context. Compile-only library source; add -parse-as-library -module-name GreetingSDK -emit-module -c -emit-objc-header-path /tmp/GreetingSDK-Swift.h.

```swift
// Greeting.swift, module GreetingSDK
import Foundation

@objc(Greeting) public final class Greeting: NSObject {
    @objc public override init() { super.init() }
    @objc public func greet(_ name: String) -> String { "Hello, \(name)" }
}
```

**Walkthrough:** The public class exports as Greeting and its method as greet: with NSString input/result. The header, native client, and linked runtime execution are checked separately below. A Swift String bridge says nothing about callback or KVO policy.

### A private Swift method can remain selector-visible

Standalone synchronous NotificationCenter fixture. It posts only on its current context and removes the registration explicitly.

```swift
import Foundation

final class SelectorSink: NSObject {
    private(set) var deliveries = 0
    @objc private func receive(_ notification: Notification) { deliveries += 1 }
    var action: Selector { #selector(receive(_:)) }
}
func selectorProbe() -> [String] {
    let center = NotificationCenter()
    let sink = SelectorSink()
    let name = Notification.Name("E1.probe")
    center.addObserver(sink, selector: sink.action, name: name, object: nil)
    center.post(name: name, object: nil)
    center.removeObserver(sink)
    center.post(name: name, object: nil)
    return [
        NSStringFromSelector(sink.action), String(sink.responds(to: sink.action)),
        String(sink.deliveries),
    ]
}
precondition(selectorProbe() == ["receive:", "true", "1"])
```

**Walkthrough:** Private restricts Swift source access, while @objc still exposes the selector. Removal prevents the second delivery. Selector visibility is not access to an arbitrary private Swift method, nor proof that the observer is safe for concurrent posts.

### One asynchronous profile contract, separate from its header

Shared Profile facade setup, used by the cancellation question. A Swift composition root supplies the Swift-only loader protocol; Objective-C receives an already-constructed service rather than an initializer taking that protocol.

The entry and cancel operations require MainActor-compatible execution. The entry returns a token synchronously; exactly one terminal completion is delivered later on MainActor, including invalid UUID errors. Success has a profile and no error; failure has an error and no profile. cancel requests cooperation: cancellation observed before loading avoids the load, and cancellation observed afterward rejects its result. A loader that never finishes can delay delivery indefinitely. Dropping the token does not cancel.

The immutable DTO bridges String/URL to readonly NSString/NSURL properties. NSError has a stable domain/code mapping, and an unexpected loader error is kept as the underlying error. These contracts are not all represented in the generated Objective-C header.

```swift
import Foundation

public struct Profile: Sendable {
    public let id: UUID
    public let name: String
    public let avatarURL: URL?
    public init(id: UUID, name: String, avatarURL: URL?) {
        self.id = id
        self.name = name
        self.avatarURL = avatarURL
    }
}

public protocol ProfileLoading: Sendable {
    func loadProfile(userID: UUID) async throws -> Profile
}

@objc(ACMProfile)
public final class ProfileObjC: NSObject, Sendable {
    @objc public let identifier: String
    @objc public let name: String
    @objc public let avatarURL: URL?

    public init(_ profile: Profile) {
        identifier = profile.id.uuidString
        name = profile.name
        avatarURL = profile.avatarURL
        super.init()
    }
}

@objc(ACMProfileErrorCode)
public enum ProfileErrorCodeObjC: Int {
    case invalidUserID = 1
    case cancelled = 2
    case failed = 3
}

public enum ProfileLoadError: Error, CustomNSError {
    case invalidUserID
    case cancelled
    case failed(any Error)

    public static var errorDomain: String { "com.acme.profile" }
    public var errorCode: Int {
        switch self {
        case .invalidUserID: ProfileErrorCodeObjC.invalidUserID.rawValue
        case .cancelled: ProfileErrorCodeObjC.cancelled.rawValue
        case .failed: ProfileErrorCodeObjC.failed.rawValue
        }
    }
    public var errorUserInfo: [String: Any] {
        switch self {
        case .invalidUserID: [NSLocalizedDescriptionKey: "The user ID is invalid."]
        case .cancelled: [NSLocalizedDescriptionKey: "The request was cancelled."]
        case .failed(let underlying):
            [
                NSLocalizedDescriptionKey: "The profile request failed.",
                NSUnderlyingErrorKey: underlying as NSError,
            ]
        }
    }
}

@MainActor
@objc(ACMProfileRequest)
public final class ProfileRequestObjC: NSObject {
    private var task: Task<Void, Never>?
    fileprivate override init() { super.init() }
    fileprivate func attach(_ task: Task<Void, Never>) { self.task = task }
    fileprivate func finish() { task = nil }
    @objc public func cancel() { task?.cancel() }
    public func waitForCompletion() async { await task?.value }
}

@MainActor
@objc(ACMProfileService)
public final class ProfileServiceObjC: NSObject {
    private let loader: any ProfileLoading
    public init(loader: any ProfileLoading) {
        self.loader = loader
        super.init()
    }

    @objc(loadProfileWithUserID:completionHandler:)
    public func loadProfile(
        userID: String,
        completionHandler: @escaping @MainActor @Sendable (ProfileObjC?, NSError?) -> Void
    ) -> ProfileRequestObjC {
        let request = ProfileRequestObjC()
        let task = Task { [loader, weak request] in
            defer { request?.finish() }
            do {
                try Task.checkCancellation()
                guard let id = UUID(uuidString: userID) else {
                    throw ProfileLoadError.invalidUserID
                }
                let profile = try await loader.loadProfile(userID: id)
                try Task.checkCancellation()
                completionHandler(ProfileObjC(profile), nil)
            } catch {
                let failure: ProfileLoadError
                if Task.isCancelled || error is CancellationError {
                    failure = .cancelled
                } else if let known = error as? ProfileLoadError {
                    failure = known
                } else {
                    failure = .failed(error)
                }
                completionHandler(nil, failure as NSError)
            }
        }
        request.attach(task)
        return request
    }
}
```

**Walkthrough:** The inherited MainActor Task starts after the synchronous entry yields, so attach and the caller’s token assignment happen before delivery. The task retains loader/completion but only weakly observes request, avoiding a token → task → token cycle. defer clears the task slot when done. Cancellation does not roll back external effects or choose which request is freshest. The checked Sendable DTO is immutable; do not generalize that property to mutable NSObject subclasses.

### Small drivers make callback timing observable

Shared facade test setup; prepend Profile facade. FixedLoader returns a known profile and counts calls. Delivery records one terminal callback and lets the caller await it, including when the token is dropped. No sleeps establish event order.

```swift
actor FixedLoader: ProfileLoading {
    private(set) var calls = 0
    func loadProfile(userID: UUID) async throws -> Profile {
        calls += 1
        return Profile(id: userID, name: "Ada", avatarURL: nil)
    }
}
@MainActor final class Delivery {
    private(set) var calls = 0
    private(set) var profile: ProfileObjC?
    private(set) var error: NSError?
    private var finished = false
    private var waiter: CheckedContinuation<Void, Never>?
    func complete(_ profile: ProfileObjC?, _ error: NSError?) {
        MainActor.preconditionIsolated()
        precondition(!finished && (profile == nil) != (error == nil))
        calls += 1
        self.profile = profile
        self.error = error
        finished = true
        waiter?.resume()
        waiter = nil
    }
    func wait() async {
        if finished { return }
        await withCheckedContinuation { waiter = $0 }
    }
}
```

## Rules and pitfalls

### NSObject inheritance does not expose every Swift method

**Common pitfall:** Forming a selector for an ordinary Swift method merely because its owner inherits NSObject.

**Mistake:** Compile-only; intentionally rejected.

```swift
import Foundation

final class Sink: NSObject {
    func receive(_ notification: Notification) {}
    var action: Selector { #selector(receive(_:)) }
}
```

**Consequence:** The compiler rejects #selector because receive has not been exposed to Objective-C.

**Why:** A superclass does not turn every supported Swift member into an exported selector.

**Correction:** Choose explicit exposure for the member that participates in the selector contract.

```swift
import Foundation

final class Sink: NSObject {
    @objc func receive(_ notification: Notification) {}
    var action: Selector { #selector(receive(_:)) }
}
let sink = Sink()
precondition(NSStringFromSelector(sink.action) == "receive:")
precondition(sink.responds(to: sink.action))
```

**What changes:** The selector exists under its actual Objective-C spelling. The earlier NotificationCenter example additionally verifies native delivery and removal.

### A Swift-only value type cannot simply appear in an Objective-C signature

**Common pitfall:** Adding @objc to a method returning an ordinary Swift struct.

**Mistake:** Compile-only; intentionally rejected.

```swift
import Foundation

struct ProfileValue { let name: String }
final class Service: NSObject {
    @objc func profile() -> ProfileValue { ProfileValue(name: "Ada") }
}
```

**Consequence:** The unsupported return type is rejected.

**Why:** The Objective-C header needs a representable class/value convention. It cannot express an arbitrary Swift struct.

**Correction:** Keep the value model internal to Swift and expose an immutable NSObject DTO.

```swift
import Foundation

struct ProfileValue { let name: String }
@objc(ExampleProfile) final class ProfileDTO: NSObject {
    @objc let name: String
    init(_ value: ProfileValue) {
        name = value.name
        super.init()
    }
}
final class Service: NSObject {
    @objc func profile() -> ProfileDTO { ProfileDTO(ProfileValue(name: "Ada")) }
}
precondition(Service().profile().name == "Ada")
```

**What changes:** The facade exposes a readonly NSString-compatible property while retaining the Swift value representation behind the boundary. This is an explicit snapshot, not automatic recursive conversion.

### Header generation does not remove the actor calling requirement

**Common pitfall:** Calling the profile token from a synchronous nonisolated function. Prepend Profile facade.

**Mistake:** Compile-only; intentionally rejected.

```swift
func cancelFromWrongContext(_ request: ProfileRequestObjC) {
    request.cancel()  // Expected error: MainActor-isolated call in a synchronous nonisolated context.
}
```

**Consequence:** Swift rejects the call because cancel belongs to MainActor.

**Why:** An Objective-C header cannot enforce all of the Swift actor/Sendable contract. A native caller must follow the documented context even when its compiler accepts a message send.

**Correction:** Make the caller actor-owned, or await an explicit actor hop when that is the API’s contract. Prepend Profile facade and facade test setups.

```swift
@MainActor func cancelFromOwner(_ request: ProfileRequestObjC) { request.cancel() }
let loader = FixedLoader()
let service = ProfileServiceObjC(loader: loader)
let delivery = Delivery()
let request = service.loadProfile(userID: UUID().uuidString) { delivery.complete($0, $1) }
precondition(delivery.calls == 0)
cancelFromOwner(request)
await request.waitForCompletion()
precondition(delivery.calls == 1 && delivery.error?.code == ProfileErrorCodeObjC.cancelled.rawValue)
let loads = await loader.calls
precondition(loads == 0)
```

**What changes:** Immediate cancellation occurs before the queued MainActor task starts loading. Delivery still arrives once and later on the promised actor. Native adapters must arrange the same context or schedule a real hop, rather than assert an unknown callback is isolated.

## Worked example

### A native client invokes the actual generated selector

**Setup:** Use the GreetingSDK library source from Essential mechanics. Compile the Objective-C client after emitting GreetingSDK-Swift.h; link the Swift library/object and runtime.

**Predict:** Will Swift-only source visibility or a guessed selector be enough to prove this call works?

```swift
import Foundation

let service = Greeting()
precondition(service.greet("Ada") == "Hello, Ada")
```

**Outcome:** The Swift check returns Hello, Ada; the native client below independently sends greet: and checks the same result.

**Why:** The actual generated header supplies construction, selector and NSString types. The native compiler/linker/executable establish a different boundary from a Swift call alone.

```objective-c
// Client.m; imports the emitted header from GreetingSDK
#import <Foundation/Foundation.h>
#import "GreetingSDK-Swift.h"
int main(void) {
    @autoreleasepool {
        Greeting *service = [[Greeting alloc] init];
        NSCAssert([[service greet:@"Ada"] isEqualToString:@"Hello, Ada"], @"bridge failed");
    }
    return 0;
}
```

Validation compiles the client with ARC/modules and warnings as errors, then links with the Swift driver. The exercise below gives the complete build sequence. This fixture establishes the stateless synchronous operation; it does not imply a profile callback can run on any context.

## Interview questions

### Why can a Swift struct have a dynamic method while KVO uses @objc dynamic?

**Answer:** Native Swift dynamic replacement is a different mechanism from Objective-C runtime interception. A struct can use native dynamic; KVO observes a supported NSObject property through @objc dynamic.

**Example:** Standalone runnable example.

```swift
import Foundation

struct NativeValue { dynamic func number() -> Int { 7 } }
final class Meter: NSObject { @objc dynamic var value = 0 }
let meter = Meter()
var observed: [Int] = []
let token = meter.observe(\.value, options: [.new]) { _, change in
    if let value = change.newValue { observed.append(value) }
}
meter.value = 1
token.invalidate()
meter.value = 2
precondition(NativeValue().number() == 7 && observed == [1])
```

**Walkthrough:** The KVO token sees the first write, is invalidated, and sees no second write. The struct call is accepted without becoming an Objective-C object.

**Boundary:** This local synchronous KVO fixture is not thread-safe observation storage. Neither dynamic spelling promises synchronization, Sendable, or correct observer lifetime. [Native dynamic explanation](https://forums.swift.org/t/how-does-the-hot-reloading-work-in-xcode11/25312/5)

### Which Swift types/signatures disappear or change in the exported header, and what proves the native client can call them?

**Answer:** Only supported exposed signatures appear; Swift-only models/generics need a facade. Async/throwing functions can export completion/error conventions. Inspect the actual header and compile an ordinary Objective-C consumer.

**Example:** Compile-only E1Bridge library declaration. Combine with Profile facade to inspect one generated header.

```swift
import Foundation

@objc(ACMAsyncNames)
public final class AsyncNamesObjC: NSObject, Sendable {
    @objc public func name(for identifier: String) async throws -> String {
        guard identifier == "known" else {
            throw NSError(
                domain: "com.acme.names", code: 7,
                userInfo: [NSLocalizedDescriptionKey: "Unknown identifier."])
        }
        return "Ada"
    }
}
```

**Walkthrough:** The emitted E1Bridge-Swift.h exposes nameFor:completionHandler: with nullable NSString result and NSError. A native shim calls success and failure in the additional validation; the Swift-only ProfileLoading initializer remains behind the composition root.

**Boundary:** The compiler-generated async export has no custom cancellation token and this class makes no MainActor completion promise. Check the explicit facade contract separately. [Async Objective-C interop](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md)

### When does this profile facade complete, where does it deliver, and what happens when the token is cancelled or dropped?

**Answer:** It returns its token synchronously and delivers one terminal completion later on MainActor. Cancellation is cooperative; dropping the token does not cancel the task or release its retained loader/completion dependencies.

**Example:** Prepend Profile facade and facade test setups.

```swift
let loader = FixedLoader()
let service = ProfileServiceObjC(loader: loader)
let delivery = Delivery()
var request: ProfileRequestObjC? = service.loadProfile(userID: UUID().uuidString) {
    delivery.complete($0, $1)
}
weak let observed = request
precondition(delivery.calls == 0)
request = nil
precondition(observed == nil)
await delivery.wait()
precondition(delivery.calls == 1 && delivery.profile?.name == "Ada" && delivery.error == nil)
let loads = await loader.calls
precondition(loads == 1)
```

**Walkthrough:** The token disappears before MainActor yields to loading, yet the task still finishes and calls the retained completion. The earlier pitfall checks immediate cancellation; a controlled loader separately checks cancellation after starting.

**Boundary:** A loader that never returns can delay terminal delivery. Immediate terminal cancellation would need a separate one-shot race policy; request freshness needs an owner identity policy. The emitted header omits some actor/Sendable guarantees.

### Extension: How do imported NSError-out and completion APIs become Swift throws/async calls?

**Answer:** Applicable Objective-C naming/result/error conventions can import a synchronous method as throws and a one-shot completion method as async throws. The native implementation must still satisfy valid result/error combinations and one terminal completion.

**Example:** Compile-only Swift client of E1Legacy, whose complete native header/implementation follows. Link its native object for execution.

```swift
import E1Legacy
import Foundation

func importedSyncName(_ identifier: String) throws -> String {
    try E1LegacyNames().readName(for: identifier)
}

func importedAsyncName(_ identifier: String) async throws -> String {
    try await E1LegacyNames().loadName(for: identifier)
}
```

**Walkthrough:** With the native fixture below, both calls return Ada for known and throw NSError domain com.acme.legacy/code 7 for another identifier. CustomNSError in ProfileLoadError deliberately controls its exported domain/code/userInfo rather than exposing arbitrary Swift error cases.

**Boundary:** Not every block-taking method imports as async. Objective-C exceptions are not Swift Error values, and generated naming is not a substitute for the native lifecycle contract.

The E1Legacy header and module map describe real native declarations. The implementation imports the emitted E1Bridge-Swift.h after the Profile facade and AsyncNames declarations have been compiled into that module. E1BeginProfile accepts the concrete facade supplied by this fixture; id is not validation of an arbitrary object.

```objc
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN
@interface E1LegacyNames : NSObject
- (nullable NSString *)readNameForIdentifier:(NSString *)identifier
                                     error:(NSError * _Nullable * _Nullable)error
    NS_SWIFT_NAME(readName(for:));
- (void)loadNameForIdentifier:(NSString *)identifier
           completionHandler:(void (^)(NSString * _Nullable, NSError * _Nullable))completionHandler
    NS_SWIFT_NAME(loadName(for:completionHandler:));
@end

FOUNDATION_EXPORT id E1BeginProfile(id service, NSString *identifier,
    void (^completion)(id _Nullable, NSError * _Nullable));
FOUNDATION_EXPORT void E1CallAsyncNames(id service, NSString *identifier,
    void (^completion)(NSString * _Nullable, NSError * _Nullable));
NS_ASSUME_NONNULL_END
```

```text
module E1Legacy { header "E1Legacy.h" export * }
```

```objc
#import "E1Legacy.h"
#import "E1Bridge-Swift.h"

@implementation E1LegacyNames
- (NSString *)readNameForIdentifier:(NSString *)identifier error:(NSError **)error {
    if ([identifier isEqualToString:@"known"]) { return @"Ada"; }
    if (error != NULL) {
        *error = [NSError errorWithDomain:@"com.acme.legacy" code:7
            userInfo:@{NSLocalizedDescriptionKey: @"Unknown identifier."}];
    }
    return nil;
}
- (void)loadNameForIdentifier:(NSString *)identifier
           completionHandler:(void (^)(NSString *, NSError *))completionHandler {
    NSError *error = nil;
    NSString *name = [self readNameForIdentifier:identifier error:&error];
    completionHandler(name, error);
}
@end

id E1BeginProfile(id service, NSString *identifier,
                 void (^completion)(id, NSError *)) {
    ACMProfileService *typed = service;
    return [typed loadProfileWithUserID:identifier
                    completionHandler:^(ACMProfile *profile, NSError *error) {
        completion(profile, error);
    }];
}

void E1CallAsyncNames(id service, NSString *identifier,
                     void (^completion)(NSString *, NSError *)) {
    ACMAsyncNames *typed = service;
    [typed nameFor:identifier completionHandler:completion];
}
```

Build E1Legacy.m with ARC, modules, Wall/Wextra/Werror and the matching SDK/target, then link it with the Swift E1Bridge object for native/Swift clients. The native callback bridge re-enters MainActor only under the explicit ProfileService contract; an arbitrary callback would require an actual hop.

### Extension: How should a native async callback return to actor-owned Swift code?

**Answer:** Respect the actual callback context rather than inheriting a promise the native API never made. This compiler-generated async export does not promise MainActor delivery. A Sendable callback can resume a continuation without accessing actor-owned state; the awaiting caller then resumes under its own execution contract.

**Example:** Separate runnable Swift client of the earlier E1Bridge/E1Legacy native fixture. Import the emitted E1Bridge module and link its Swift object plus E1Legacy.o. The callback captures only the checked continuation, whose resume operation can be called from the native callback context.

```swift
import E1Bridge
import E1Legacy
import Foundation

func nativeAsyncName(_ identifier: String) async -> (String?, NSError?) {
    await withCheckedContinuation { continuation in
        E1CallAsyncNames(AsyncNamesObjC(), identifier) { @Sendable name, error in
            continuation.resume(returning: (name, error as NSError?))
        }
    }
}
let success = await nativeAsyncName("known")
precondition(success.0 == "Ada" && success.1 == nil)
let failure = await nativeAsyncName("missing")
precondition(failure.0 == nil && failure.1?.domain == "com.acme.names" && failure.1?.code == 7)
```

**Walkthrough:** Success returns Ada; failure returns the exported NSError domain/code. The explicit @Sendable closure avoids inheriting a caller’s actor isolation. Without that distinction, an actor-inheriting Swift closure passed to this unannotated native block parameter can trap when the async export invokes it elsewhere. The separate ProfileService callback has an explicit MainActor promise and follows that different contract.

**Boundary:** This is a one-shot completion bridge, so every path must resume exactly once. It does not add cancellation, guarantee a physical thread, or justify assumeIsolated in an arbitrary callback. UI work belongs after the await in an actor-owned caller or behind a deliberate actor hop.

## Focused exercise

**Task:** Expose a Swift operation to an Objective-C client, inspect its selector/types, invoke it, and document its required context.

**Acceptance checks:**

- The generated header exposes Greeting, public init, greet:, and NSString values.
- The ordinary Objective-C executable successfully invokes the implementation.
- The method is stateless and synchronous; the profile facade has a separate MainActor contract.

**Solution:** Use the earlier GreetingSDK source and Objective-C Client.m. These are actual build commands; add the shared baseline flags from Example setup.

```swift
import Foundation

let greeting = Greeting()
precondition(greeting.greet("Grace") == "Hello, Grace")
```

**Reasoning:** The Swift call is a small behavior check; the following native build/run completes the requested interop proof. The command uses a header generated from this exact library source, rather than handwritten declarations.

```sh
# Run in the temporary directory containing Greeting.swift and Client.m.
xcrun --sdk macosx swiftc -swift-version 6 -default-isolation nonisolated \
  -enable-upcoming-feature NonisolatedNonsendingByDefault \
  -enable-upcoming-feature InferIsolatedConformances \
  -target arm64-apple-macos27.0 -parse-as-library -module-name GreetingSDK \
  -emit-module -c Greeting.swift -emit-objc-header-path GreetingSDK-Swift.h -o Greeting.o
xcrun --sdk macosx clang -fobjc-arc -fmodules -Wall -Wextra -Werror \
  -target arm64-apple-macos27.0 -I . -c Client.m -o Client.o
xcrun --sdk macosx swiftc Greeting.o Client.o -o greeting-client
./greeting-client
```

## In practice

**Choose it when:** Expose a supported native facade while preserving a richer Swift core and explicit callback policy.

**Investigate by:** Read generated selectors, nullability and errors, then exercise real native clients and controlled cancellation/ownership cases.

**Strong evidence:** Keep header representability, native invocation, execution context, delivery count and resource lifetime as distinct evidence.

## References

### Related guides

- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A12 — Availability and Conditional Compilation]]
- [[A13 — Error handling and typed throws]]
- [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]]
- [[B6 — Any, AnyObject, Metatypes, and Runtime Type Information]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C1 — ARC fundamentals]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[D13 — Swift 6 Strict Concurrency Migration]]
- [[E2 — Foundation and Objective-C Bridging Behavior]]
- [[E3 — C Interoperability and Unsafe Boundaries]]
- [[E4 — C++ interoperability]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [Using Swift from Objective-C](https://developer.apple.com/documentation/swift/importing-swift-into-objective-c)
- [SE-0297](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md)
- [SE-0112](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0112-nserror-bridging.md)
- [Swift Evolution SE-0160: limiting Objective-C inference](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0160-objc-inference.md)
- [Compiler-team explanation of native Swift dynamic support](https://forums.swift.org/t/how-does-the-hot-reloading-work-in-xcode11/25312/5)
