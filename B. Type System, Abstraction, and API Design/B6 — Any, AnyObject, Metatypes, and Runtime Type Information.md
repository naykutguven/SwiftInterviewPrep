---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - runtime-type-information
---

# B6 — Any, AnyObject, Metatypes, and Runtime Type Information

**Goal:** Initialize through a supplied class metatype and verify the selected dynamic class.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Runtime metadata can recover relationships hidden by erasure, but the API must still preserve them deliberately. T.Type retains a generic type relationship; a class metatype can hold a subclass. Any.Type erases that relationship within its supported type domain. ObjectIdentifier provides lifetime-scoped identity, not a durable serialized type name. [Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/types/)

## Rules and pitfalls

### Use the supplied metatype value

T() constructs through the generic type parameter. type.init() uses the supplied metatype value and can select a subclass initializer when a base metatype is the argument. A required initializer preserves the construction obligation through subclasses.

**Common pitfall:** Accepting a type argument but calling T() can ignore the runtime selection. Test the resulting dynamic class with an explicitly base-typed metatype.

### Keep registry erasure private

A typed registration method should relate the key type to its factory result before storing an erased closure. Define duplicates, missing entries, instance lifetime, and actor ownership as separate policies.

**Common pitfall:** Names obtained by reflection or demangling are presentation details, not stable persisted IDs. Register an explicit domain key when identity must survive renaming, compiler changes, or process boundaries. SendableMetatype concerns type metadata, not the safety of its instances.

## Interview questions

### What information does each metatype form preserve, and when can type(of:) in generic code report the existential container?

T.Type preserves T; any P.Type holds a conforming type's metatype, while (any P).Type names the existential container type itself. Any.Type erases the specific type in Any's domain. Generic type(of:) can reflect the existential container in contexts where T is that existential rather than an opened concrete type; test the precise generic use.

### Why can T() and type.init() return different dynamic classes for one argument?

T is fixed by generic inference after conversions, while the metatype value can still contain a subclass. T() uses the former; type.init() dispatches construction through the latter. Required initialization makes that subclass construction valid across the hierarchy.

### How do actor ownership, typed registration, missing entries, duplicates, and instance lifetime interact in a factory registry?

Actor ownership serializes registry mutations; typed registration establishes the key/factory relationship. Missing entries need a stated error or optional result, duplicates need rejection or replacement rules, and factories must define new versus shared instances. Actor protection does not decide lifecycle or guarantee a safe non-Sendable return across its boundary.

### Why are reflected or demangled names unsuitable persisted type IDs, and how do module presence and OS availability differ?

Reflected and demangled spellings can change with modules, implementation details, or compiler releases and can collide as user-facing identifiers. Check module presence with canImport separately from runtime availability. In this SDK Runtime is absent on iOS; the demonstrated macOS demangling API needs OS 27. Keep it optional presentation tooling.

## Focused exercise

**Task:** Compare T() with initialization through a supplied class metatype. Check the resulting dynamic types and reject an unrelated registration.

**Acceptance checks:**

- A base-typed metatype containing Child exposes the T()/type.init() distinction.
- Typed registration returns the promised subclass.
- An unrelated factory result rejects without a runtime cast.

**Solution:**

Standalone synchronous metatype comparison:

```swift
class Base { required init() {} }
final class Child: Base { required init() { super.init() } }
func make<T: Base>(_ type: T.Type) -> (T, T) {
    (T(), type.init())
}
let selected: Base.Type = Child.self
let (generic, dynamic) = make(selected)
precondition(type(of: generic) == Base.self)
precondition(type(of: dynamic) == Child.self)
func register<T: Base>(_ type: T.Type, factory: () -> T) -> T {
    factory()
}
let registered = register(Child.self) { Child() }
precondition(type(of: registered) == Child.self)
```

A registration of Child.self with a factory returning an unrelated type intentionally rejects in a separate compile-only variant.

**Reasoning:** Using an explicit Base.Type prevents inference from selecting Child as T and hiding the distinction. The miniature registration checks one type relationship; a real registry additionally needs storage, duplicate, missing-entry, isolation, and instance-lifetime policies.

## In practice

**Choose it when:** Use metatype factories for genuine runtime type selection and typed keys for registry APIs.

**Investigate by:** Compare static generic type, supplied metatype, and resulting dynamic class.

**Strong evidence:** Validate erased registry storage through typed entry points and test its lifecycle and isolation rules.

### Reflection, module selectors, and Runtime — optional

#### Reflection is a view supplied by the type

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
```

Result: `["display"]`, not an exhaustive list of stored properties. CustomReflectable can change the view. Mirror is useful for diagnostics, not stable serialization, arbitrary property mutation, method invocation, exhaustive metadata, or security enforcement. A custom mirror does not erase the underlying data. [CustomReflectable documentation](https://developer.apple.com/documentation/swift/customreflectable)

#### Module selectors disambiguate source lookup

```swift
enum Swift {
    struct Int {}
}

func selectedTypes() -> [Bool] {
    let standard: Swift::Int.Type = Swift::Int.self
    let local: Swift.Int.Type = Swift.Int.self
    return [ObjectIdentifier(standard) == ObjectIdentifier(Int.self),
            ObjectIdentifier(local) == ObjectIdentifier(Int.self)]
}
```

Result: `[true, false]`. The `::` selector identifies the module even when a declaration shadows its name. It does not load a plugin, perform a runtime cast, or identify a persisted type. This syntax was implemented in Swift 6.3 and remains part of the 6.4 baseline. [SE-0491](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0491-module-selectors.md)

#### Demangle a symbol for presentation

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
    if #available(anyAppleOS 27.0, *) {
        return (try? symbolLabel(symbol)) ?? symbol
    }
    #endif
    return symbol
}
```

The checked macOS SDK provides Runtime; the iOS device/simulator SDK inventory does not. The import guard keeps unsupported builds valid, and portableSymbolLabel retains the raw input when the module/API is unavailable or the symbol is invalid. On the macOS 27 host, `symbolLabel("$sSi")` returns `Swift.Int`; an invalid input throws `.invalidSymbol`. The API takes a mangled symbol, not an arbitrary memory address or a source type name. Demangled output is diagnostic text whose format can change, not a parseable stable identity. [SE-0498](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0498-runtime-demangle.md)

For a bounded output buffer, keep truncation explicit and validate bytes before creating a String:

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

For `$sSi`, capacities 0 and 3 report 9 required bytes in this toolchain; capacity 9 returns `.text("Swift.Int")`. The function discards a truncated prefix, allowing a caller to choose whether to retry within its own size limit. The byte count is not a C-string terminator count. No unsafe pointer or span escapes its storage lifetime. The required size and exact presentation are observations of this symbol/toolchain, not constants to hard-code in production.

The String and span overloads have separate allocation/handling tradeoffs; neither promises that it is appropriate inside an async-signal handler or that an arbitrary amount of untrusted input is cheap. Choose resource limits when building a diagnostic service. F2 covers the broader debugging pipeline.

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
