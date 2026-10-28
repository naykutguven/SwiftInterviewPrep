---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - runtime-type-information
---

# B6 — Any, AnyObject, Metatypes, and Runtime Type Information

**Learning goals:**

- Distinguish Any, AnyObject, concrete/generalized/protocol metatypes, and dynamic factory selection.
- Maintain a typed registry invariant through private erasure, actor ownership, duplicate/missing policy, and instance lifetime.
- Use reflection, module selectors, and demangling for diagnostics while keeping durable IDs and availability explicit.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] for class identity, [[A3 — Initialization model]] for required initializers, [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]/[[B2 — Existentials (`any`) vs Opaque Types (`some`)]] for protocols and existential opening, and [[D4 — Actors and Actor Isolation]] for the registry’s actor boundary.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A factory can receive a class type as a value. That **metatype value** may identify a subclass even when the generic parameter is the base type. Calling T() and calling the received type.init() therefore answer different construction requests.

Any erases a value’s static type; AnyObject describes an object reference, including values explicitly bridged to Foundation objects. T.Type retains a generic relationship, while Any.Type erases it within its supported type domain. ObjectIdentifier supplies in-process identity for objects or type metadata. Reflection and demangling produce diagnostic views, not durable domain identifiers. [Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/types/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Erased values need a checked operation at the boundary

Foundation bridging is explicit here. A conditional cast returns nil for a mismatch instead of asserting a type that was erased.

```swift
import Foundation

let value: Any = "Ada"
precondition((value as? String) == "Ada")
precondition((value as? Int) == nil)
let number = 42
let bridged: NSNumber = number as NSNumber
let object: AnyObject = bridged
precondition((object as? NSNumber)?.intValue == 42)
```

**Walkthrough:** Any retains a value whose static String relationship is no longer available without checking. NSNumber is an object representation of the integer; it does not imply that every native value originally had class identity. A forced mismatched cast would trap and is not used.

### Use the received metatype for runtime selection

The explicitly base-typed variable prevents inference from choosing Child as the generic T.

```swift
class Base { required init() {} }
final class Child: Base {}
func staticMake<T: Base>(_ selected: T.Type) -> T { T() }
func dynamicMake<T: Base>(_ selected: T.Type) -> T { selected.init() }
let selected: Base.Type = Child.self
precondition(type(of: staticMake(selected)) == Base.self)
precondition(type(of: dynamicMake(selected)) == Child.self)
```

**Walkthrough:** T is Base in both calls. The metatype value still identifies Child. required init ensures construction through that value is supported throughout the subclass hierarchy.

### Shared registry setup

The following **compile-only** setup is named ServiceRegistry. Its public entry points establish the type key/result relationship before storing an erased closure. register creates fresh instances through the received metatype; registerShared deliberately retains one instance, keyed by its actual dynamic type. Duplicates reject, missing entries throw, and all products are Sendable so they can cross the actor boundary. The private cast checks an invariant established by those entry points; callers cannot modify the erased dictionary.

```swift
protocol Service: AnyObject, Sendable {
    init()
    var label: String { get }
}
final class FileService: Service {
    let label = "file"
    init() {}
}
final class MemoryService: Service {
    let label = "memory"
    init() {}
}
enum RegistryError: Error, Equatable { case duplicate, missing }
actor ServiceRegistry {
    private var factories: [ObjectIdentifier: @Sendable () -> any Service] = [:]
    private func install(
        key: ObjectIdentifier, factory: @escaping @Sendable () -> any Service
    ) throws(RegistryError) {
        guard factories[key] == nil else { throw .duplicate }
        factories[key] = factory
    }
    func register<S: Service>(_ selected: S.Type) throws(RegistryError) {
        try install(key: ObjectIdentifier(selected)) { selected.init() }
    }
    func registerShared<S: Service>(_ instance: S) throws(RegistryError) {
        try install(key: ObjectIdentifier(type(of: instance))) { instance }
    }
    func resolve<S: Service>(_ selected: S.Type) throws(RegistryError) -> S {
        guard let factory = factories[ObjectIdentifier(selected)] else { throw .missing }
        guard let value = factory() as? S else {
            preconditionFailure("Private registry key/result invariant violated")
        }
        return value
    }
}
```

The closure for a fresh registration captures the same metatype used as its key. A shared registration captures the instance whose dynamic type supplies its key. Those paired choices prevent a valid registration from putting an unrelated product under another type’s key. SendableMetatype, inherited by Sendable, concerns metadata transfer; it does not by itself make an arbitrary mutable instance Sendable.

## Rules and pitfalls

### A type argument can be accidentally ignored

**Common pitfall:** Accepting a selected subclass metatype but constructing T() instead of using that value.

**Mistake:** Standalone runnable example.

```swift
class Animal { required init() {} }
final class Dog: Animal {}
func make<T: Animal>(_ selected: T.Type) -> T { T() }
let selected: Animal.Type = Dog.self
precondition(type(of: make(selected)) == Animal.self)
```

**Consequence:** The factory constructs Animal, losing the caller’s runtime choice of Dog.

**Why:** Generic inference fixes T as Animal, while the argument value still identifies Dog. T() uses the former.

**Correction:** Invoke initialization through the received metatype.

```swift
class Animal { required init() {} }
final class Dog: Animal {}
func make<T: Animal>(_ selected: T.Type) -> T { selected.init() }
let selected: Animal.Type = Dog.self
precondition(type(of: make(selected)) == Dog.self)
```

**What changes:** The factory honors runtime selection, with required initialization preserving the hierarchy’s construction contract.

### A reflected name is not a durable ID

**Common pitfall:** Using implementation type spellings as persistence keys when a renamed implementation should remain the same product capability.

**Mistake:** Standalone runnable example.

```swift
struct ImageLoaderV1 {}
struct ImageFetcherV2 {}
let oldKey = String(reflecting: ImageLoaderV1.self)
let newKey = String(reflecting: ImageFetcherV2.self)
precondition(oldKey != newKey)
```

**Consequence:** The two source implementations produce different keys even if the app intends one persistent image-fetching capability.

**Why:** Reflected names describe implementation/module spelling. Their format and names are not the application’s compatibility contract.

**Correction:** Choose an explicit versioned domain identifier and keep implementation metadata diagnostic.

```swift
import Foundation

struct PluginID: Hashable, Codable { let value: String }
let identity = PluginID(value: "image-fetching.v1")
let bytes = try JSONEncoder().encode(identity)
let restored = try JSONDecoder().decode(PluginID.self, from: bytes)
precondition(restored == identity)
```

**What changes:** The persisted key can remain stable while the implementation changes. Its uniqueness scope and future migrations are still application responsibilities; ObjectIdentifier is also not a serialized replacement.

## Worked example

### Fresh instances and shared instances are different registry policies

**Setup:** Prepend ServiceRegistry setup. Calls are serialized by the actor, and returned immutable services are Sendable.

**Predict:** Does actor protection imply singleton lifetime?

```swift
let freshRegistry = ServiceRegistry()
try await freshRegistry.register(FileService.self)
let first = try await freshRegistry.resolve(FileService.self)
let second = try await freshRegistry.resolve(FileService.self)
precondition(first !== second)
let shared = MemoryService()
let sharedRegistry = ServiceRegistry()
try await sharedRegistry.registerShared(shared)
let sharedFirst = try await sharedRegistry.resolve(MemoryService.self)
let sharedSecond = try await sharedRegistry.resolve(MemoryService.self)
precondition(sharedFirst === shared && sharedSecond === shared)
```

**Outcome:** The fresh registration returns distinct objects; the shared registration returns its retained instance.

**Why:** The registration operation defines lifetime. Actor isolation only protects registry state; it does not select that policy or make an otherwise unsafe product transferable.

## Interview questions

### What do the metatype forms preserve, and when can generic type(of:) report the existential container?

**Answer:** T.Type retains T. any P.Type holds a conforming type’s metatype; (any P).Type is the existential container type itself. Any.Type erases the particular type. Generic type(of:) can report the container when its generic T is that existential rather than an opened conformer.

**Example:** Standalone runnable example.

```swift
protocol Named {}
struct User: Named {}
let concrete: User.Type = User.self
let conforming: any Named.Type = User.self
let container: (any Named).Type = (any Named).self
let erased: Any.Type = concrete
precondition(ObjectIdentifier(conforming) == ObjectIdentifier(User.self))
precondition(ObjectIdentifier(container) != ObjectIdentifier(User.self))
precondition(ObjectIdentifier(erased) == ObjectIdentifier(User.self))
func elementMetadata<T>(_ values: [T]) -> Any.Type { type(of: values[0]) }
let value: any Named = User()
precondition(type(of: value) == User.self)
precondition(elementMetadata([value] as [any Named]) == (any Named).self)
```

**Walkthrough:** The direct existential operation can observe User’s dynamic type. The array fixes the helper’s T as any Named, so its generic operation reports that container. These are different static contexts, not contradictory runtime identities.

**Boundary:** The sample array is nonempty by construction. Any/Any.Type do not cover every noncopyable type in the selected compiler; the ownership extension below checks that domain explicitly.

### Why can T() and selected.init() construct different classes for one argument?

**Answer:** The generic T is determined statically after inference/conversions. The received metatype value can identify a subclass. T() constructs the former; selected.init() honors the latter.

**Example:** Standalone runnable example.

```swift
class Animal { required init() {} }
final class Dog: Animal {}
func compare<T: Animal>(_ selected: T.Type) -> (T, T) { (T(), selected.init()) }
let selected: Animal.Type = Dog.self
let (staticResult, selectedResult) = compare(selected)
precondition(type(of: staticResult) == Animal.self)
precondition(type(of: selectedResult) == Dog.self)
```

**Walkthrough:** The argument’s static type establishes T == Animal, but its value is Dog.self. Required initialization makes the dynamic construction legal.

**Boundary:** Writing compare(Dog.self) directly can infer T == Dog and hide this distinction. Fix the argument’s static type when testing the runtime-selection contract.

### How do actor ownership, typed registration, missing/duplicate policy, and instance lifetime interact?

**Answer:** They solve separate invariants. Actor ownership serializes storage access; typed registration pairs the key with its product; errors define missing/duplicate behavior; fresh versus shared registration defines lifetime. Products still need a safe boundary when returned across actors.

**Example:** Prepend ServiceRegistry setup.

```swift
let registry = ServiceRegistry()
do {
    _ = try await registry.resolve(FileService.self)
    preconditionFailure("Expected missing registration")
} catch { precondition(error == .missing) }
try await registry.register(FileService.self)
do {
    try await registry.register(FileService.self)
    preconditionFailure("Expected duplicate rejection")
} catch { precondition(error == .duplicate) }
let first = try await registry.resolve(FileService.self)
let second = try await registry.resolve(FileService.self)
precondition(first.label == "file" && first !== second)
let singleton = MemoryService()
let shared = ServiceRegistry()
try await shared.registerShared(singleton)
let resolved = try await shared.resolve(MemoryService.self)
precondition(resolved === singleton)
```

**Walkthrough:** The first resolution fails, duplicate registration leaves the original policy intact, and fresh/shared products follow their explicit registration methods. The private erasure cannot be mutated by a caller.

**Boundary:** The registry intentionally supports Sendable service objects and synchronous construction. Async factories, reset/invalidation, dependency cycles, or actor-confined non-Sendable products require their own lifecycle APIs.

### Why are reflected/demangled names poor persisted IDs, and how do module presence and OS availability differ?

**Answer:** Names are diagnostic presentation and can change with implementation or compiler releases. A module must exist in the selected SDK before it can be imported; an API inside it can separately require a newer runtime. Use both guards for optional tooling.

**Example:** Standalone runnable example.

```swift
#if canImport(Runtime)
    import Runtime
    @available(anyAppleOS 27.0, *)
    func symbolLabel(_ symbol: String) throws(DemanglingError) -> String {
        try Runtime::demangle(symbol)
    }
#endif
func portableSymbolLabel(_ symbol: String) -> String {
    #if canImport(Runtime)
        if #available(anyAppleOS 27.0, *) { return (try? symbolLabel(symbol)) ?? symbol }
    #endif
    return symbol
}
precondition(portableSymbolLabel("not a mangled symbol") == "not a mangled symbol")
#if canImport(Runtime)
    if #available(anyAppleOS 27.0, *) {
        let label = try symbolLabel("$sSi")
        precondition(label == "Swift.Int")
        do {
            _ = try symbolLabel("invalid")
            preconditionFailure()
        } catch {
            if case .invalidSymbol = error {
            } else {
                preconditionFailure("Unexpected demangling error")
            }
        }
    }
#endif
```

**Walkthrough:** The selected macOS SDK supplies Runtime and the host supports the OS 27 API. A valid symbol demangles; invalid input throws, while the portable helper intentionally preserves it as fallback text. The checked iOS SDK inventory lacks Runtime. [SE-0498](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0498-runtime-demangle.md)

**Boundary:** The exact Swift.Int spelling is a toolchain observation for this symbol, not a persisted format. The API accepts a mangled symbol, not an arbitrary address. [[A12 — Availability and Conditional Compilation|A12]] explains source guards versus availability contexts.

### Extension: does SendableMetatype make instances Sendable, and can Any erase a noncopyable value?

**Answer:** No. SendableMetatype concerns type metadata, while instance transfer needs its own contract. The selected Any and Any.Type domain also does not admit the noncopyable Token examples below.

**Example:** Standalone runnable example.

```swift
protocol MetadataOnly: SendableMetatype {}
final class Mutable: MetadataOnly { var count = 0 }
func acceptMetadata<T: MetadataOnly>(_ selected: T.Type) -> ObjectIdentifier {
    ObjectIdentifier(selected)
}
precondition(acceptMetadata(Mutable.self) == ObjectIdentifier(Mutable.self))
let instance = Mutable()
instance.count = 7
precondition(instance.count == 7)
```

**Walkthrough:** Mutable’s metatype has the declared metadata contract, but the mutable object is still not Sendable. [SE-0470](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md)

**Boundary:** Do not conflate metadata, object lifetime, copyability, and instance safety. Ownership-aware generic APIs can preserve a noncopyable value without converting it to Any.

Prepend that Mutable setup. This **intentionally rejected** instance transfer lacks Sendable:

```swift
func requiresSendable<T: Sendable>(_ value: T) {}
requiresSendable(instance)
```

Two independent **compile-only, intentionally rejected** erasures:

```swift
struct Token: ~Copyable { let id: Int }
func invalid() {
    let erased: Any = Token(id: 7)
    _ = erased
}
```

```swift
struct Token: ~Copyable { let id: Int }
let invalid: Any.Type = Token.self
```

## Focused exercise

**Task:** Compare generic construction with the supplied metatype and check a typed registration relationship.

**Acceptance checks:**

- A base-typed metatype holding Child distinguishes T() from selected.init().
- Registration with Child.self returns the promised Child.
- A factory returning an unrelated class is rejected statically.

**Solution:** Standalone synchronous exercise. The larger actor registry above adds actual storage, duplicate/missing, and lifetime policy.

```swift
class Base { required init() {} }
final class Child: Base {}
func make<T: Base>(_ selected: T.Type) -> (T, T) { (T(), selected.init()) }
func register<T: Base>(_ selected: T.Type, factory: () -> T) -> T { factory() }
let selected: Base.Type = Child.self
let (staticResult, dynamicResult) = make(selected)
precondition(type(of: staticResult) == Base.self)
precondition(type(of: dynamicResult) == Child.self)
let registered = register(Child.self) { Child() }
precondition(type(of: registered) == Child.self)
```

**Reasoning:** The explicit Base.Type prevents the test from accidentally inferring Child as T. The small registration relates its static T to the closure result; ServiceRegistry separately honors runtime metatype selection with selected.init().

Prepend the exercise setup; this unrelated registration is **intentionally rejected**:

```swift
final class Unrelated {}
let invalid = register(Child.self) { Unrelated() }
```

## In practice

**Choose it when:** Use metatypes for genuine runtime construction and typed keys for registries. Keep persistent identity explicit.

**Investigate by:** Compare the static generic argument, supplied metatype value, result’s dynamic type, and actual registry key.

**Strong evidence:** Validate private erasure through typed entry points, exercise lifetime/error policy, and check module availability independently of runtime API availability.

### Reflection, module selectors, and Runtime — optional

These independent examples retain the diagnostic tooling details. A custom mirror supplies a view rather than exhaustive storage metadata; a module selector disambiguates source lookup rather than loading a type dynamically.

```swift
struct Sample: CustomReflectable {
    let actual: Int
    var customMirror: Mirror {
        Mirror(self, children: ["display": "summary"])
    }
}

func reflectedLabels() -> [String] {
    Mirror(reflecting: Sample(actual: 42)).children.compactMap(\.label)
}

precondition(reflectedLabels() == ["display"])
```

The mirror reports display instead of actual. CustomReflectable does not erase the stored data; Mirror is diagnostic, not a stable serializer, arbitrary property editor, exhaustive metadata source, or security boundary. [CustomReflectable](https://developer.apple.com/documentation/swift/customreflectable)

```swift
enum Swift {
    struct Int {}
}

func selectedTypes() -> [Bool] {
    let standard: Swift::Int.Type = Swift::Int.self
    let local: Swift.Int.Type = Swift.Int.self
    return [
        ObjectIdentifier(standard) == ObjectIdentifier(Int.self),
        ObjectIdentifier(local) == ObjectIdentifier(Int.self),
    ]
}

precondition(selectedTypes() == [true, false])
```

Swift::Int selects the module’s Int despite a local declaration named Swift. Swift.Int selects the shadowing declaration in this example. Module selectors were introduced in Swift 6.3. [SE-0491](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0491-module-selectors.md)

The next compile-only shared setup is named BoundedLabel. The temporary-allocation helper lends bounded output storage; demangling fills it or reports the required size. Only initialized output bytes are validated as UTF-8 and copied into a String. No borrowed view escapes. Negative capacity violates the caller precondition and is never tested.

```swift
#if canImport(Runtime)
    import Runtime

    enum BoundedLabel: Equatable {
        case text(String)
        case invalidSymbol
        case truncated(requiredBytes: Int)
        case invalidUTF8
        case otherFailure
    }

    @available(anyAppleOS 27.0, *)
    func boundedSymbolLabel(_ symbol: String, capacity: Int) -> BoundedLabel {
        precondition(capacity >= 0)
        return withTemporaryAllocation(of: UInt8.self, capacity: capacity) { output in
            do {
                try Runtime::demangle(symbol.utf8Span, into: &output)
            } catch DemanglingError.invalidSymbol {
                return .invalidSymbol
            } catch DemanglingError.truncated(let required) {
                return .truncated(requiredBytes: required)
            } catch {
                return .otherFailure
            }
            do {
                let utf8 = try UTF8Span(validating: output.span)
                return .text(String(copying: utf8))
            } catch {
                return .invalidUTF8
            }
        }
    }
#endif
```

Prepend BoundedLabel setup. This runnable probe checks the selected macOS toolchain’s output and truncation observations:

```swift
#if canImport(Runtime)
    precondition(boundedSymbolLabel("$sSi", capacity: 0) == .truncated(requiredBytes: 9))
    precondition(boundedSymbolLabel("$sSi", capacity: 3) == .truncated(requiredBytes: 9))
    precondition(boundedSymbolLabel("$sSi", capacity: 9) == .text("Swift.Int"))
    precondition(boundedSymbolLabel("invalid", capacity: 32) == .invalidSymbol)
#endif
```

The helper discards a truncated prefix instead of presenting incomplete text. Nine is an observation for this symbol/build, not a production sizing constant or C-string terminator count. A diagnostic service still needs input/output limits; neither overload promises async-signal safety or cheap processing of arbitrary input.

## References

### Related guides

- [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]
- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B5 — Key paths and strongly typed indirection]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[E1 — Objective-C Interoperability]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

### Official sources

- [Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/types/)
- [CustomReflectable documentation](https://developer.apple.com/documentation/swift/customreflectable)
- [SE-0491](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0491-module-selectors.md)
- [SE-0498](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0498-runtime-demangle.md)
- [Swift type casting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/typecasting/)
- [Mirror](https://developer.apple.com/documentation/swift/mirror)
- [SE-0470: Isolated conformances and SendableMetatype](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md)
