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

**Goal:** Export a Swift operation, inspect its generated Objective-C selector, and invoke it from a native client.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Objective-C clients consume representable classes, selectors, blocks, nullable object references, and NSError conventions. Keep richer Swift generic/value/actor implementation behind a deliberate facade. @objc exposes a supported declaration; @objc dynamic selects Objective-C runtime dispatch; NSObject supplies framework integration without exposing every member or synchronizing state. [Using Swift from Objective-C](https://developer.apple.com/documentation/swift/importing-swift-into-objective-c)

## Rules and pitfalls

### Inspect the generated surface

Give external clients public construction and representable methods, then generate the header and compile a real Objective-C client. Swift-only generic or value signatures can disappear or require adaptation; Swift effects can export through completion/error conventions.

**Common pitfall:** Guessing a selector or assuming every NSObject member is exported can leave a facade unusable. Treat the generated header as evidence and test native invocation.

### Document contracts the header cannot express

State calling context, callback delivery, invocation count, cancellation, and token lifetime. The checked Objective-C header does not carry all Swift MainActor/Sendable callback guarantees.

**Common pitfall:** A native caller can violate an actor context the Swift compiler enforces for Swift clients. Require the documented context or implement a deliberate asynchronous hop. Dropping a token and cancelling it are distinct unless the API explicitly ties them.

## Interview questions

### Why can a Swift struct have a dynamic method while KVO uses @objc dynamic?

Native Swift dynamic replacement is a separate mechanism and can apply to a struct method. KVO relies on Objective-C runtime interception, so the observed supported property needs the Objective-C dynamic path. Neither dynamic spelling alone establishes thread safety or observation lifetime.

### Which Swift types/signatures disappear or change in the exported header, and what proves the native client can call it?

Unsupported Swift-only types or generic signatures cannot simply appear as native declarations; async/error forms can be adapted to blocks and NSError conventions. Only inspecting the emitted header and compiling/invoking an ordinary Objective-C client proves that specific exported surface is usable.

### When does this profile facade complete, where does it deliver, and what happens when the token is cancelled or dropped?

It returns a token synchronously and delivers one terminal completion asynchronously on MainActor, including invalid-ID failure. Dropping the token does not cancel. cancel requests cooperation: cancellation observed before loading avoids the load; cancellation observed after loading rejects its result. A loader that never finishes can prevent delivery. The task retains its dependencies, weakly references the token, and clears the token's task reference on completion.

## Focused exercise

**Task:** Expose one Swift operation to an Objective-C client. Inspect its selector and types, invoke it, and document the required calling context.

**Acceptance checks:**

- The header exposes the intended class, initializer, greet: selector, and NSString types.
- The Objective-C executable invokes the Swift implementation successfully.
- Calling context and mutable-state assumptions are documented separately from header types.

**Solution:**

Two-source macOS fixture using Foundation; no UI state or special actor context is required for this stateless synchronous method.

```swift
// Greeting.swift, module GreetingSDK
import Foundation
@objc(Greeting) public final class Greeting: NSObject {
    @objc public override init() { super.init() }
    @objc public func greet(_ name: String) -> String { "Hello, \(name)" }
}
```

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

Emit the Swift library/module with -emit-objc-header-path GreetingSDK-Swift.h, inspect greet:, then compile Client.m against that header and link the Swift library. Use F5's baseline flags.

**Reasoning:** This minimal facade uses representable Foundation values and ordinary synchronous execution. It establishes export and invocation, not KVO, callback cancellation, or UI delivery; the extended facade retains those distinct lessons.

## In practice

**Choose it when:** A native client requires a stable representable facade rather than the entire Swift implementation surface.

**Investigate by:** Inspect generated headers, native calls, and callback lifetime/context.

**Strong evidence:** Verify selector/types and the actual cancellation, error, isolation, and ownership contracts from both languages.

### Native async facade and callback ownership — optional

#### Compile the native declarations, not a guessed Swift signature

Temporary E1Legacy.h defines a synchronous NSError-out API and a one-shot completion API. The two C entry points are validation shims that make real Objective-C calls into the Swift facades.

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

Temporary module.modulemap makes that header importable by Swift:

> module E1Legacy {
>     header "E1Legacy.h"
>     export *
> }

The checked Swift imports are:

```swift
import Foundation
import E1Legacy

func importedSyncName(_ identifier: String) throws -> String {
    try E1LegacyNames().readName(for: identifier)
}

func importedAsyncName(_ identifier: String) async throws -> String {
    try await E1LegacyNames().loadName(for: identifier)
}
```

Both functions return Ada for "known" and throw NSError domain com.acme.legacy/code 7 for another identifier. The completion form imports as async throws because it follows the applicable conventions; not every block-taking method qualifies. A one-shot contract and valid result/error combinations remain the native implementer's responsibility. Swift Error bridging can preserve NSError identity information; CustomNSError lets a Swift error supply deliberate domain, code, and userInfo. [SE-0297](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md), [SE-0112](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0112-nserror-bridging.md)

#### Export async work to an Objective-C completion method

```swift
import Foundation

@objc(ACMAsyncNames)
public final class AsyncNamesObjC: NSObject, Sendable {
    @objc public func name(for identifier: String) async throws -> String {
        guard identifier == "known" else {
            throw NSError(domain: "com.acme.names", code: 7,
                          userInfo: [NSLocalizedDescriptionKey: "Unknown identifier."])
        }
        return "Ada"
    }
}
```

The generated E1Bridge-Swift.h exposes nameFor:completionHandler: with nullable NSString result and nullable NSError. The checked native client invokes it for success and failure. This compiler-generated entry does not expose the custom cancellation token of the next exercise, and this example does not promise MainActor completion delivery. The driver resumes a continuation without touching actor-owned mutable state. [SE-0297](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0297-concurrency-objc.md)

#### The actual Objective-C implementation/client

Temporary E1Legacy.m imports the compiler-generated header. Compile this after emitting E1Bridge-Swift.h from the async class and the complete exercise below. E1BeginProfile receives the concrete facade supplied by the Swift fixture; its id parameters do not constitute runtime validation of an arbitrary foreign object.

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

The header, implementation, module map, imported Swift calls, and exported Swift classes belong to the same temporary mixed-language fixture. The native source compiles with ARC, modules, Wall/Wextra/Werror. The validation record describes SDK/target/module-cache commands and observed header checks.


### Profile facade and cancellation ownership — optional

**Contract:** Expose an asynchronous Swift profile loader to Objective-C while keeping the Swift core value-oriented. A Swift composition root constructs the loader and service, then passes the service object to Objective-C. Objective-C clients are not given an initializer that accepts the Swift-only protocol.

Required contract:

1. A MainActor caller supplies a UUID string and receives a request token synchronously. Invalid syntax is an invalid-ID error, not a missing-profile result.
2. A terminal completion is delivered asynchronously on MainActor, including validation failure; success has a profile and no error, failure has an error and no profile.
3. cancel requests cooperative cancellation. Cancellation observed before loading avoids starting the loader; cancellation observed after loading rejects its result. A loader that never finishes can prevent terminal delivery.
4. Dropping the token does not cancel. Avoid a token/task retain cycle, retain work dependencies until completion, and release the token's task reference when finished.
5. NSError has a stable domain/code contract, and unexpected loader failures retain their underlying error. Cancellation takes precedence when the task is cancelled or the loader throws CancellationError.
6. Exported profile objects are final immutable snapshots with checked Sendable conformance. Do not generalize that property to arbitrary NSObject subclasses.

A complete reference implementation follows. waitForCompletion is a Swift-only validation/lifecycle helper; the generated Objective-C token exposes cancel. The String/URL properties bridge to readonly NSString/NSURL properties. The integer enum describes error codes; the associated-value Swift error stays behind the boundary.

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
        case let .failed(underlying):
            [NSLocalizedDescriptionKey: "The profile request failed.",
             NSUnderlyingErrorKey: underlying as NSError]
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

The inherited MainActor Task begins after this synchronous method yields, so attach completes before its body can finish and the caller can store the returned token before callback delivery. The Task retains loader and completion; weak request prevents the token→task→token cycle. It is unstructured work with an explicit owner handle, not a child automatically cancelled when a caller's task ends. Cancellation is a request, does not roll back effects, and does not by itself choose which of several requests is freshest.

Objective-C must call both service entry and cancel on the main actor's compatible execution context; the generated header does not enforce that contract. Swift rejects the following synchronous nonisolated call when compiled with the reference implementation:

```swift
func cancelFromWrongContext(_ request: ProfileRequestObjC) {
    request.cancel() // Expected error: MainActor-isolated call in a synchronous nonisolated context.
}
```

The fixture uses MainActor-isolated orchestration and checks callback isolation where MainActor.preconditionIsolated is available. Its Objective-C callback re-enters Swift using assumeIsolated only because this specific adapter supplies the established contract. Do not copy that assertion into an arbitrary callback API; it validates a promise rather than scheduling a hop.

**Acceptance cases:** success and readonly DTO fields; invalid UUID; immediate cancellation before load starts; cancellation after start even if the loader returns success; ordinary failure with underlying domain/code/userInfo preserved; loader CancellationError; repeated/late cancellation without duplicate delivery; successful delivery after the token deallocates; actual Objective-C invocation through the generated header. Use a controlled actor loader and continuations to choose event order, not timing sleeps. Compare a negative Swift caller and rejected representability examples with their expected diagnostics.

**Discussion:** If production requires cancellation to complete immediately even when the dependency ignores it, introduce an explicit one-shot terminal state and a cancellation/dependency race policy. If multiple requests update one UI, track request identity in the UI owner. Neither policy can be inferred from this small adapter or from adding @Sendable to a callback.

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
