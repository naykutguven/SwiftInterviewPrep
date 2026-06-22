---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - dispatch
  - protocols
  - api-design
---
## 0. Rubric snapshot

**Rubric expectation**

Understand the difference between static dispatch, class virtual dispatch, Objective-C dynamic dispatch when opted into, witness-table dispatch for protocol requirements, and non-requirement methods added in protocol extensions.

**Caveats**

Many engineers incorrectly think all protocol extension methods are dynamically dispatched. They are not. Choosing `dynamic`, `@objc`, or protocol-based dispatch changes both behavior and performance characteristics.

**You should be able to answer**

- How do dispatch rules differ between a struct method, a class method, a protocol requirement, and a non-requirement method in a protocol extension?
- Why can calling a method through a protocol-typed value produce different behavior than calling it on a concrete type?
- When does Objective-C dynamic dispatch enter the picture?

**You should be able to do**

- Given a protocol, an extension, and a conforming type, predict which implementation is called in different call sites.
- Explain which calls use witness-table dispatch and which do not.

---

## 1. Core mental model

Dispatch is the mechanism Swift uses to decide **which implementation of a method call runs**. The important distinction is whether the compiler knows the target implementation at compile time, or whether the target must be chosen at runtime.

Swift strongly prefers compile-time knowledge when it can get it. A concrete `struct` method call is usually statically dispatched. The compiler knows the concrete type and can often inline or specialize the call. This is one reason value-oriented and generic Swift can be fast.

Classes introduce identity and inheritance, so non-`final` class methods may need runtime dispatch. If a subclass can override a method, the compiler cannot always hard-code the exact function. Native Swift classes typically use a virtual method table model for overridable methods; Swift’s ABI notes describe non-final class method calls as runtime-resolved through a vtable. ([GitHub](https://github.com/apple/swift/blob/main/docs/ABIStabilityManifesto.md "swift/docs/ABIStabilityManifesto.md at main · swiftlang/swift · GitHub"))

Protocols introduce another dispatch mechanism: **witness-table dispatch**. A protocol requirement is a slot in a conformance table. When you call a requirement through an existential such as `any P`, Swift uses the value’s witness table to find the implementation for that concrete conforming type. Swift’s ABI notes describe witness tables as tables of function pointers implementing a protocol conformance, used when the runtime type of an existential is not statically known. ([GitHub](https://github.com/apple/swift/blob/main/docs/ABIStabilityManifesto.md "swift/docs/ABIStabilityManifesto.md at main · swiftlang/swift · GitHub"))

Protocol extension methods are the trap. If a method is a **protocol requirement**, an extension can provide a default implementation, and conforming types can provide their own witness. If a method exists **only in a protocol extension** and is not declared as a requirement, it is just extension sugar. It does not get a witness-table slot. When called through a protocol-typed value, Swift calls the extension implementation based on the static type.

The key idea:

```text
Concrete type call → compiler-chosen implementation
Class override call → vtable/runtime-chosen implementation
Protocol requirement call → witness-table-chosen implementation
Protocol extension non-requirement call → statically chosen extension implementation
```

---

## 2. Essential mechanics

### Static dispatch: concrete value types and non-overridable calls

For a concrete `struct`, `enum`, or `final` class method, Swift usually knows the target implementation directly.

```swift
struct Logger {
    func log(_ message: String) {
        print(message)
    }
}

let logger = Logger()
logger.log("Loaded")
```

Here, `Logger.log` is a concrete method. There is no subclass override and no existential indirection.

This enables:

```text
direct call
inlining
generic specialization
dead-code elimination
```

That does not mean the compiler always inlines it. It means the language model gives the optimizer more information.

---

### Class virtual dispatch: overridable class methods

Non-`final` class instance methods may be dynamically dispatched because subclasses can override them.

```swift
class Screen {
    func render() {
        print("Screen.render")
    }
}

final class HomeScreen: Screen {
    override func render() {
        print("HomeScreen.render")
    }
}

let screen: Screen = HomeScreen()
screen.render()
```

Output:

```text
HomeScreen.render
```

The static type is `Screen`, but the runtime object is `HomeScreen`. Swift must respect overriding.

Marking a class or method `final` removes that override possibility:

```swift
final class AnalyticsTracker {
    func track(_ event: String) {
        print(event)
    }
}
```

Now the compiler can treat the call more like a direct call.

---

### Protocol requirement dispatch: witness tables

A protocol requirement creates a polymorphic customization point.

```swift
protocol Renderer {
    func render()
}

extension Renderer {
    func render() {
        print("Default render")
    }
}

struct HomeRenderer: Renderer {
    func render() {
        print("HomeRenderer.render")
    }
}

let renderer: any Renderer = HomeRenderer()
renderer.render()
```

Output:

```text
HomeRenderer.render
```

`render()` is a protocol requirement. The extension provides a default, but `HomeRenderer` provides its own implementation. When called through `any Renderer`, Swift uses the conformance’s witness table.

The protocol extension default is used only if the conforming type does not provide its own implementation:

```swift
struct DefaultRenderer: Renderer {}

let renderer: any Renderer = DefaultRenderer()
renderer.render()
```

Output:

```text
Default render
```

The Swift book documents that protocols define requirements and that protocol extensions can provide implementations of requirements or add additional functionality. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/?utm_source=chatgpt.com "Protocols - Documentation | Swift.org"))

---

### Protocol extension non-requirement dispatch: static extension lookup

This is the classic trap.

```swift
protocol Renderer {}

extension Renderer {
    func render() {
        print("Extension render")
    }
}

struct HomeRenderer: Renderer {
    func render() {
        print("HomeRenderer.render")
    }
}

let concrete = HomeRenderer()
let erased: any Renderer = HomeRenderer()

concrete.render()
erased.render()
```

Output:

```text
HomeRenderer.render
Extension render
```

Why?

`render()` is not a protocol requirement. It has no witness-table slot. On `concrete`, the compiler sees `HomeRenderer`, so it calls `HomeRenderer.render()`. On `erased`, the static type is `any Renderer`, so the only known `render()` is the extension method.

If you need polymorphism, make it a requirement:

```swift
protocol Renderer {
    func render()
}

extension Renderer {
    func render() {
        print("Default render")
    }
}
```

---

### Objective-C dynamic dispatch: `@objc` and `dynamic`

Objective-C dynamic dispatch enters when you opt Swift declarations into Objective-C runtime behavior, usually for Objective-C interop, selectors, KVO/KVC, UIKit/AppKit target-action, or method replacement patterns. Swift’s documentation says `dynamic` makes access always dynamically dispatched through the Objective-C runtime and prevents inlining/devirtualization; declarations marked `dynamic` must also be `@objc`. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations - Documentation | Swift.org"))

```swift
import Foundation

final class Player: NSObject {
    @objc dynamic var status: String = "idle"
}
```

Use this for interop-driven needs, not as a default abstraction tool.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking protocol extension methods are always polymorphic

Bad:

```swift
protocol AnalyticsEvent {}

extension AnalyticsEvent {
    func name() -> String {
        "default"
    }
}

struct PurchaseEvent: AnalyticsEvent {
    func name() -> String {
        "purchase"
    }
}

func track(_ event: any AnalyticsEvent) {
    print(event.name())
}

track(PurchaseEvent())
```

Output:

```text
default
```

Better:

```swift
protocol AnalyticsEvent {
    func name() -> String
}

extension AnalyticsEvent {
    func name() -> String {
        "default"
    }
}

struct PurchaseEvent: AnalyticsEvent {
    func name() -> String {
        "purchase"
    }
}

func track(_ event: any AnalyticsEvent) {
    print(event.name())
}

track(PurchaseEvent())
```

Output:

```text
purchase
```

The fix is not cosmetic. You moved `name()` from “extension helper” to “required customization point.”

---

### Trap 2: Assuming generic and existential calls behave the same

```swift
protocol Describable {}

extension Describable {
    func describe() {
        print("default")
    }
}

struct User: Describable {
    func describe() {
        print("user")
    }
}

func concrete<T: Describable>(_ value: T) {
    value.describe()
}

func existential(_ value: any Describable) {
    value.describe()
}

concrete(User())
existential(User())
```

Output:

```text
default
default
```

Because `describe()` is not a requirement.

Now make it a requirement:

```swift
protocol Describable {
    func describe()
}
```

Then both calls can dispatch to the conforming type’s witness:

```text
user
user
```

The point: generics preserve concrete type information, but protocol extension non-requirements still are not witness-table customization points.

---

### Trap 3: Using `@objc dynamic` to paper over bad Swift API design

Bad:

```swift
class Service {
    @objc dynamic func load() {
        print("base")
    }
}
```

This forces a dispatch model but does not necessarily create a better Swift abstraction.

Better:

```swift
protocol LoadingService {
    func load() async throws -> [Item]
}

struct RemoteLoadingService: LoadingService {
    func load() async throws -> [Item] {
        []
    }
}
```

Use `@objc dynamic` for Objective-C runtime needs. Use protocols, generics, classes, or closures for Swift-native abstraction.

---

### Trap 4: Adding a method to a protocol extension later and expecting conformers to override it

Suppose v1 ships:

```swift
protocol Cache {}

extension Cache {
    func clear() {
        print("default clear")
    }
}
```

A client adds:

```swift
struct ImageCache: Cache {
    func clear() {
        print("image clear")
    }
}
```

If your framework accepts `any Cache` and calls `clear()`, it still calls the extension method, because `clear()` is not a requirement.

This is a library-design bug. If clients are expected to customize behavior, the method belongs in the protocol requirement list from the start.

---

## 4. Direct answers to rubric questions

### Q1. How do dispatch rules differ between a struct method, a class method, a protocol requirement, and a non-requirement method in a protocol extension?

A `struct` method call on a concrete value is usually statically dispatched. A non-`final` class method may use virtual dispatch because subclasses can override it. A protocol requirement called through an existential uses witness-table dispatch. A protocol extension method that is not a requirement is statically dispatched based on the static type.

Interview version:

> Swift has several dispatch models. Concrete value-type calls are usually statically dispatched. Overridable class methods use virtual dispatch. Protocol requirements use witness-table dispatch when called through an existential or generic constraint. But protocol extension methods that are not requirements are not dynamically dispatched; they are selected from the static type. That distinction is essential when designing protocol customization points.

---

### Q2. Why can calling a method through a protocol-typed value produce different behavior than calling it on a concrete type?

Because the protocol-typed value exposes only the protocol’s requirement surface plus extension methods available on that protocol. If the method is not a protocol requirement, Swift does not look inside the erased concrete type to find a same-named method.

```swift
let concrete = S()
let erased: any P = S()

concrete.g() // S.g
erased.g()   // extension g, if g is not a requirement
```

The concrete call sees `S`. The existential call sees `any P`.

Interview version:

> Calling through a protocol existential changes the static interface. Requirement calls go through the conformance’s witness table, so the concrete implementation can be selected. Non-requirement extension methods do not have witness entries, so Swift resolves them from the protocol extension based on the static type. That is why the same-looking call can produce different behavior.

---

### Q3. When does Objective-C dynamic dispatch enter the picture?

Objective-C dynamic dispatch enters when a declaration is exposed to and dispatched through the Objective-C runtime, commonly via `@objc dynamic`, or through Objective-C-based frameworks and selector mechanisms.

Use it for:

```text
KVO/KVC
target-action
selectors
Objective-C overriding/interposition needs
some UIKit/AppKit runtime integration
```

Avoid using it as a general Swift polymorphism mechanism.

Interview version:

> Objective-C dynamic dispatch is opt-in for Swift declarations that need Objective-C runtime behavior, such as selectors, KVO, or framework interop. It changes both semantics and optimization potential because the compiler can no longer treat the call as a normal Swift direct, virtual, or witness-table call.

---

## 5. Code probe

Given:

```swift
protocol P {
    func f()
}

extension P {
    func f() { print("default f") }
    func g() { print("extension g") }
}

struct S: P {
    func f() { print("S.f") }
    func g() { print("S.g") }
}

let x: P = S()
x.f()
x.g()
```

In modern Swift style, write the existential explicitly:

```swift
let x: any P = S()
```

The dispatch behavior is the same.

### What happens?

```text
S.f
extension g
```

### Why?

`f()` is declared in the protocol:

```swift
protocol P {
    func f()
}
```

That makes `f()` a protocol requirement. `S` provides its own implementation:

```swift
struct S: P {
    func f() { print("S.f") }
}
```

So the conformance’s witness table for `P` points `f` to `S.f`.

`g()` is different:

```swift
extension P {
    func g() { print("extension g") }
}
```

`g()` exists only in the protocol extension. It is not a requirement. Therefore, no witness-table slot exists for `g`. When the static type is `P` / `any P`, Swift resolves `g()` to the extension method.

```text
x: any P
 ├─ stored concrete value: S
 ├─ witness table for P
 │   └─ f → S.f
 └─ extension-only method lookup
     └─ g → P extension g
```

So:

```swift
x.f()
```

uses witness-table dispatch:

```text
S.f
```

And:

```swift
x.g()
```

uses statically resolved protocol-extension dispatch:

```text
extension g
```

### Concrete-type contrast

```swift
let y = S()
y.f()
y.g()
```

Output:

```text
S.f
S.g
```

Now the static type is `S`, so the compiler sees `S.g()` directly.

### Fix or redesign

If `g()` needs polymorphic behavior, make it a protocol requirement:

```swift
protocol P {
    func f()
    func g()
}

extension P {
    func f() { print("default f") }
    func g() { print("extension g") }
}

struct S: P {
    func f() { print("S.f") }
    func g() { print("S.g") }
}

let x: any P = S()
x.f()
x.g()
```

Output:

```text
S.f
S.g
```

### Why this fix is correct

Now `g()` has a witness-table slot. The default implementation remains useful, but it is a real customization point. `S.g()` becomes the witness for `S`’s conformance to `P`.

```text
P witness table for S
 ├─ f → S.f
 └─ g → S.g
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Make `g()` a protocol requirement|`g()` is part of the polymorphic contract|Source-breaking if added to a public protocol without a default|
|Keep `g()` as extension-only helper|`g()` is convenience behavior derived only from requirements|Conforming types cannot override behavior through `any P`|
|Use generics over concrete types|You want specialization and type preservation|Still does not make non-requirement extension methods polymorphic|
|Use class inheritance|You truly need shared identity and override-based behavior|Couples design to reference semantics and subclassing|
|Use `@objc dynamic`|You need Objective-C runtime behavior|Slower, less optimizable, and not Swift-native abstraction|

---

## 6. Exercise

### Problem

Given a protocol, an extension, and a conforming type, predict which implementation is called in three different call sites.

### Bad / naive version

```swift
protocol Displayable {}

extension Displayable {
    func title() -> String {
        "Default title"
    }
}

struct Product: Displayable {
    func title() -> String {
        "Product title"
    }
}

let concrete = Product()
let existential: any Displayable = Product()

func render<T: Displayable>(_ value: T) {
    print(value.title())
}

print(concrete.title())
print(existential.title())
render(Product())
```

### What prints?

```text
Product title
Default title
Default title
```

### What is wrong?

```text
title() looks like a customization point, but it is not a protocol requirement.
The concrete call sees Product.title().
The existential call sees Displayable extension title().
The generic call is constrained only by Displayable, whose known title() is the extension method.
```

The generic case is especially unintuitive. Even though `T` is `Product` at runtime, the generic function body is type-checked against the generic constraint. Since `title()` is not a protocol requirement, Swift resolves the extension method.

### Improved version

```swift
protocol Displayable {
    func title() -> String
}

extension Displayable {
    func title() -> String {
        "Default title"
    }
}

struct Product: Displayable {
    func title() -> String {
        "Product title"
    }
}

let concrete = Product()
let existential: any Displayable = Product()

func render<T: Displayable>(_ value: T) {
    print(value.title())
}

print(concrete.title())
print(existential.title())
render(Product())
```

### What prints now?

```text
Product title
Product title
Product title
```

### Why this is better

`title()` is now part of the protocol contract. The default implementation is still available for conforming types that do not need customization, but conforming types that do customize it are respected through concrete, existential, and generic call sites.

This is the design you want when a method represents semantic behavior of conforming types.

### Alternative: extension helper derived from requirements

If the behavior should not be customized, keep it as an extension method and derive it from requirements:

```swift
protocol Displayable {
    var displayName: String { get }
}

extension Displayable {
    func decoratedTitle() -> String {
        "• \(displayName)"
    }
}

struct Product: Displayable {
    let displayName: String
}

let value: any Displayable = Product(displayName: "Keyboard")
print(value.decoratedTitle())
```

Output:

```text
• Keyboard
```

Here, `decoratedTitle()` is intentionally not customizable. The customization point is `displayName`.

---

## 7. Production guidance

Use protocol requirements when:

```text
A conforming type must be able to customize behavior.
The method is part of the semantic contract.
You will call the method through any P, some P, or T: P.
You are designing SDK or module-boundary APIs.
Tests, mocks, or adapters need polymorphic behavior.
```

Use protocol extension non-requirements when:

```text
The method is convenience behavior.
The method is fully derived from actual requirements.
You do not want conforming types to customize it polymorphically.
You are adding helper methods like map/filter-style utilities.
```

Use classes / virtual dispatch when:

```text
Identity matters.
Shared mutable state matters.
Subclass customization is an intentional design axis.
You are working inside UIKit/AppKit inheritance models.
```

Use `@objc dynamic` when:

```text
Selectors, KVO/KVC, Objective-C runtime replacement, or framework interop require it.
```

Be careful when:

```text
A protocol extension method has the same name as a concrete method.
A public protocol extension adds behavior that clients may assume is overridable.
A mock works in concrete tests but fails when erased to any Protocol.
A generic function unexpectedly calls default behavior.
A performance-sensitive path stores many values as existentials.
```

Avoid when:

```text
You use protocol extensions as hidden inheritance.
You use @objc dynamic just to force runtime behavior.
You add default implementations that silently hide missing conformer behavior.
You erase to any Protocol before you need to.
```

Debugging checklist:

```text
Is the method declared in the protocol body?
Is the call made on a concrete type, a generic T: P, or any P?
Is the implementation a witness or only an extension helper?
Does the conforming type accidentally shadow instead of customize?
Is the class method final, overridable, @objc, or dynamic?
Did type erasure change the visible API surface?
Would making the method a requirement be source-compatible?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Protocol extension methods can have default implementations, and types can override them.

This is incomplete and often wrong because “override” is not the right model for value types or protocol extension non-requirements.

### Senior answer

> Protocol requirements use witness-table dispatch. Protocol extension methods that are not requirements are statically dispatched based on the static type. Therefore, calling the same method on a concrete value versus an existential can produce different behavior.

This is correct and useful.

### Staff-level answer

> I treat protocol requirements as explicit customization points and protocol extension non-requirements as convenience helpers. At API boundaries, especially public SDKs, I avoid adding extension-only methods that look polymorphic. If behavior must vary by conforming type, it belongs in the protocol requirement list, possibly with a default implementation. If behavior is derived from requirements and should be consistent, it belongs in an extension. I also consider performance: concrete generics preserve more type information, existentials use witness tables and may box values, class methods may use vtables, and `@objc dynamic` should be reserved for Objective-C runtime integration.

Staff-level questions to ask:

```text
Is this method a semantic customization point or a derived helper?
Will this API be called through a concrete type, generic constraint, or existential?
Can adding this requirement later break clients?
Does the abstraction need value semantics, existential storage, subclassing, or Objective-C runtime behavior?
Is type erasure happening too early and hiding useful compile-time guarantees?
```

---

## 9. Interview-ready summary

Swift has multiple dispatch models. Concrete value-type and `final` calls are usually statically dispatched. Non-`final` class methods can use virtual dispatch because subclasses may override them. Protocol requirements use witness-table dispatch, so calls through `any P` can reach the conforming type’s implementation. But protocol extension methods that are not requirements are not witness-table dispatched; they are resolved from the static type. Therefore, if a protocol method needs polymorphic behavior, it must be declared as a protocol requirement, even if a protocol extension provides the default implementation.

---

## 10. Flashcards

Q: What is static dispatch?

A: The compiler selects the target implementation at compile time, commonly for concrete value-type methods, `final` class methods, and other non-overridable calls.

Q: What is class virtual dispatch?

A: Runtime method selection for overridable class methods, usually through a vtable-like mechanism, so subclass implementations can be called through a superclass reference.

Q: What is witness-table dispatch?

A: Protocol requirement dispatch through a conformance table that maps each requirement to the conforming type’s implementation.

Q: Are all protocol extension methods dynamically dispatched?

A: No. Only protocol requirements have witness-table entries. Non-requirement protocol extension methods are statically resolved based on the static type.

Q: Why does `let x: any P = S(); x.g()` call the protocol extension `g()` even if `S` has `g()`?

A: Because `g()` is not a protocol requirement, so the existential’s witness table has no `g` slot. Swift resolves `g()` from the protocol extension.

Q: How do you make a protocol extension default implementation customizable?

A: Declare the method in the protocol body as a requirement, then provide a default implementation in an extension.

Q: When should you use `@objc dynamic`?

A: When Objective-C runtime behavior is required, such as selectors, KVO/KVC, or specific framework interop. Not as a default Swift polymorphism tool.

Q: What is the API design rule for protocol extensions?

A: Requirements are customization points. Extension-only methods are convenience helpers.

---

## 11. Related sections

- [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]
- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B3 — Generics, constraints, and same-type relationships]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[E1 — Objective-C interoperability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]

---

## 12. Sources

- Swift Senior/Staff Rubric — B4 dispatch model, caveats, exercise, and code probe.
- Swift Book — Protocols: protocols define requirements that conforming types implement. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/?utm_source=chatgpt.com "Protocols - Documentation | Swift.org"))
- Swift Book — Extensions: protocol extensions can provide implementations of requirements or add functionality. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/?utm_source=chatgpt.com "Extensions - Documentation | Swift.org"))
- Swift ABI Stability Manifesto — class method dispatch via vtables and protocol dispatch via witness tables. ([GitHub](https://github.com/apple/swift/blob/main/docs/ABIStabilityManifesto.md "swift/docs/ABIStabilityManifesto.md at main · swiftlang/swift · GitHub"))
- Swift Book — Declarations: `dynamic` dispatch and Objective-C runtime behavior. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations - Documentation | Swift.org"))