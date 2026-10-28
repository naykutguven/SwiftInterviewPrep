---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - access-control
---

# A11 — Access Control

**Learning goals:**

- Distinguish private, fileprivate, internal, package, public, and open through real file and module boundaries.
- Check type naming, construction, getters, setters, and subclassing independently.
- Separate source access from inlinable bodies, ABI helpers, and Swift 6.4 export/memberwise rules.

**Prerequisites:** [[A3 — Initialization model]] for initializers and [[A10 — Extensions and Retroactive Modeling]] for extensions. [[E5 — Modules, packages, targets, products, and resources]] explains modules and package identity; [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]] expands implementation export.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Seeing a public struct does not mean you can construct it, read every property, or assign every setter. Each operation requires its own permission. Access control defines those source permissions across declarations, files, modules, and packages; it neither synchronizes state nor keeps secrets out of a shipped binary.

private covers the owning declaration and its same-file extensions. fileprivate additionally permits unrelated code in that file. internal covers a module. package covers modules built with the same package identity. public permits external source use; open additionally promises supported external subclassing or overriding. A testable import gives testing permissions that ordinary clients lack. [Access Control](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Getter and setter permissions differ

The public getter can expose a value while the type keeps writes behind a method. This same-file example is a first step, not an external-client test.

```swift
public struct Reading {
    public private(set) var value: Int
    public init(value: Int) { self.value = value }
    public mutating func increment() { value += 1 }
}
var reading = Reading(value: 7)
reading.increment()
precondition(reading.value == 8)
```

**Walkthrough:** Callers can request the supported increment operation. They cannot directly assign value outside the setter’s private context. A reference-valued property could still expose mutable object state despite private(set).

### Shared module setup: AccessCore

Build the following as its own file/module with the baseline flags plus `-parse-as-library -module-name AccessCore -package-name Study -enable-testing -emit-module -emit-module-path /tmp/AccessCore.swiftmodule -c -o /tmp/AccessCore.o`. Client source is always a different module. Use `-I /tmp` and link AccessCore.o for a runnable client. Validation substitutes its isolated temporary directory.

The ordinary-client cases use `-package-name Outside`; sibling-package cases use `-package-name Study`. @testable import additionally requires this module’s -enable-testing build. These are deliberate independent boundaries.

```swift
public struct Reading {
    public private(set) var value: Int
    private let source: String
    public init(value: Int) {
        self.value = value
        source = "sensor"
    }
}
public struct PublicLabel {
    public var text: String
    private var revision = 0
}
package struct PackageLabel { package var text: String }
package struct PackageReading {
    package let value: Int
    package init(value: Int) { self.value = value }
}
internal struct InternalReading { var value: Int }
public class PublicBase {
    public init() {}
    public func value() -> Int { 1 }
}
open class OpenBase {
    public init() {}
    open func value() -> Int { 1 }
}
```

## Rules and pitfalls

### Public visibility does not publish synthesized construction

**Common pitfall:** Expecting a public struct’s memberwise initializer to be public.

**Mistake:** Compile-only; intentionally rejected.

```swift
import AccessCore

let label = PublicLabel(text: "Ada")
```

**Consequence:** An ordinary client cannot call the internal synthesized initializer.

**Why:** The type’s public visibility and construction access are separate. Swift 6.4 can omit the initialized private revision, but the memberwise access ceiling is still internal.

**Correction:** Offer an explicit public initializer for a public construction contract. AccessCore.Reading already does this.

```swift
import AccessCore

let reading = Reading(value: 7)
precondition(reading.value == 7)
```

**What changes:** The client uses the intended public initializer and getter without testing permissions.

### A public conformance needs public witnesses

**Common pitfall:** Keeping a required property internal on a public type conforming to a public protocol.

**Mistake:** Compile-only; intentionally rejected.

```swift
public protocol ReadingSource { var value: Int { get } }
public struct SensorReading: ReadingSource { let value: Int }
```

**Consequence:** Compilation requires the witness to be public.

**Why:** External clients using the public protocol need the implementation promised by that public conformance.

**Correction:** Make the intended witness public and expose construction deliberately. Compile this correction as module WitnessCore with the shared module command.

```swift
public protocol ReadingSource { var value: Int { get } }
public struct SensorReading: ReadingSource {
    public let value: Int
    public init(value: Int) { self.value = value }
}
```

**What changes:** Public protocol use, property reading, and construction now have explicit source permissions.

Ordinary external client of that correction, linking WitnessCore.o:

```swift
import WitnessCore

func read<T: ReadingSource>(_ source: T) -> Int { source.value }
precondition(read(SensorReading(value: 9)) == 9)
```

### Exporting a body does not grant arbitrary source access

**Common pitfall:** Referencing a private helper from an inlinable public body.

**Mistake:** Compile-only; intentionally rejected.

```swift
private func hidden() -> Int { 7 }
@inlinable public func value() -> Int { hidden() }
```

**Consequence:** Compilation rejects the private dependency in an externally available body.

**Why:** Clients compiling the exposed body need a declaration that is eligible for use through the ABI, the compiled binary calling contract. A private source helper does not satisfy those rules.

**Correction:** Expose the helper for ABI use only when that commitment is intended. This is compile-only module source.

```swift
@usableFromInline internal func helper() -> Int { 7 }
@inlinable public func value() -> Int { helper() }
```

**What changes:** The helper becomes available to emitted client implementations while remaining internal to source callers. Keeping the public body unexposed is another option when optimization does not justify the commitment.

## Worked example

### Same-file extensions participate in private implementation

**Setup:** Standalone file. Draft’s extension belongs to the type’s private context in this file.

**Predict:** Can the extension restore a private initialized revision?

```swift
struct Draft {
    var title: String
    private var revision = 0
}
extension Draft {
    var revisionNumber: Int { revision }
    static func restored(title: String, revision: Int) -> Self {
        Self(title: title, revision: revision)
    }
}
let fresh = Draft(title: "New")
let restored = Draft.restored(title: "Old", revision: 7)
precondition([fresh.revisionNumber, restored.revisionNumber] == [0, 7])
```

**Outcome:** The same-file extension can read revision and use the private compatibility initializer.

**Why:** Swift 6.4 offers an internal form omitting initialized less-visible revision, while preserving the applicable private form. This does not publish either form as public/package API. [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)

## Interview questions

### Why can a sibling package module name a struct but fail to construct it?

**Answer:** package visibility exposes the type, but a synthesized memberwise initializer is still at most internal. Swift 6.4 changes property participation, not that access ceiling. Provide a package initializer when siblings should construct it.

**Example:** Runnable sibling client of AccessCore: use -package-name Study.

```swift
import AccessCore

let visibleType = PackageLabel.self
_ = visibleType
let reading = PackageReading(value: 9)
precondition(reading.value == 9)
```

**Walkthrough:** Matching Study package identity permits naming PackageLabel. PackageReading explicitly exposes construction, so that separate call works.

**Boundary:** PackageLabel(text:) remains inaccessible to a sibling module. Matching package identity is a build setting/manifest relationship, not a folder naming convention. [SE-0386](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0386-package-access-modifier.md)

Separate sibling client, **intentionally rejected** because PackageLabel has no package initializer:

```swift
import AccessCore

let label = PackageLabel(text: "Ada")
```

### How do ordinary external, same-package, and testable clients differ for reading and subclassing?

**Answer:** An ordinary client uses public/open API. A sibling additionally uses package API. @testable import exposes eligible internal API and testing customization from an -enable-testing build. open is the supported external customization contract.

**Example:** Runnable ordinary Outside client of AccessCore.

```swift
import AccessCore

final class Specialized: OpenBase {
    override func value() -> Int { 2 }
}
precondition(Specialized().value() == 2)
precondition(Reading(value: 7).value == 7)
```

**Walkthrough:** An ordinary Outside client can construct Reading and subclass/override OpenBase. It cannot use InternalReading or subclass the non-open PublicBase through this import.

**Boundary:** Same-package visibility does not expose private storage. Testable success does not prove ordinary source compatibility. Public non-open customization remains a different contract from open. [SE-0117](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0117-non-public-subclassable-by-default.md)

Separate runnable sibling client with -package-name Study. It can read package API; non-open PublicBase still does not publish cross-module customization through an ordinary import:

```swift
import AccessCore

precondition(PackageReading(value: 9).value == 9)
```

Separate runnable testing client, using the same compiled AccessCore with -enable-testing:

```swift
@testable import AccessCore

final class TestSubclass: PublicBase {
    override func value() -> Int { 3 }
}
precondition(InternalReading(value: 4).value == 4)
precondition(TestSubclass().value() == 3)
```

Two separate ordinary Outside clients are **intentionally rejected**:

```swift
import AccessCore

let internalValue = InternalReading(value: 4)
```

```swift
import AccessCore

class InvalidSubclass: PublicBase {}
```

### Why can a same-file extension read private state while an unrelated helper or another-file extension cannot?

**Answer:** The type’s same-file extensions share its private implementation context. Unrelated declarations and other-file extensions do not. fileprivate deliberately broadens access to unrelated code in one file.

**Example:** Standalone runnable example.

```swift
struct Secret {
    private var number = 7
    fileprivate var label = "visible in this file"
}
extension Secret { var exposedNumber: Int { number } }
func helper(_ secret: Secret) -> String { secret.label }
precondition(Secret().exposedNumber == 7)
precondition(helper(Secret()) == "visible in this file")
```

**Walkthrough:** The extension can read number; helper can read label because it is fileprivate. helper does not gain permission to read number.

**Boundary:** Moving the extension to another file removes its private access. Do not make storage public merely to repair file organization; expose the smallest needed operation instead.

Separate **intentionally rejected** unrelated-helper example:

```swift
struct Secret { private var number = 7 }
func unrelated(_ secret: Secret) -> Int { secret.number }
```

For the other-file check, save this compile-only declaration as Secret.swift:

```swift
struct Secret { private var number = 7 }
```

Compile this second file together with Secret.swift using `-parse-as-library -whole-module-optimization`; it is **intentionally rejected**:

```swift
extension Secret { var exposedNumber: Int { number } }
```

### How do private(set), usableFromInline, inlinable, and export(interface) constrain different aspects?

**Answer:** private(set) controls source assignment. @usableFromInline makes an internal helper eligible for ABI use by exposed bodies. @inlinable exposes an implementation body. @export(interface) emits a callable symbol while withholding that body from clients; none upgrades a private/internal name to public source API.

**Example:** Compile-only library source. Use baseline flags plus -parse-as-library. [[E8 — Binary frameworks, inlineability, and public implementation detail leakage|E8]] supplies binary-distribution checks.

```swift
public struct Counter {
    public private(set) var count = 0
    public init() {}
    public mutating func increment() { count += 1 }
}
@usableFromInline internal func incremented(_ value: Int) -> Int { value + 1 }
@inlinable public func next(_ value: Int) -> Int { incremented(value) }
private func policy() -> Int { 7 }
@export(interface) public func libraryPolicy() -> Int { policy() }
```

**Walkthrough:** The setter remains private. next’s visible body may call the ABI-visible helper. libraryPolicy’s body can call a private helper while keeping that body on the library side. [SE-0497](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)

**Boundary:** These are distinct commitments, not access synonyms. @export cannot simply be combined with @inlinable or @usableFromInline on the same declaration. [[E8 — Binary frameworks, inlineability, and public implementation detail leakage|E8]] records interface/symbol and unchanged-client experiments; this block only compiles the API shape.

## Focused exercise

**Task:** Expose a small public value and verify ordinary-client construction, reads, and denied writes/private reads.

**Acceptance checks:**

- An ordinary import constructs Reading and reads value.
- Reading.source and direct assignment to Reading.value are rejected.
- No testable import or matching package identity is used.

**Solution:** Runnable ordinary Outside client of AccessCore. The two rejected clients immediately below are independent files.

```swift
import AccessCore

let reading = Reading(value: 7)
precondition(reading.value == 7)
```

**Reasoning:** The explicit public initializer exposes construction; the getter exposes observation. The negative clients test the separate permissions that a successful read cannot establish.

Independent **intentionally rejected** clients, using the same -I and Outside package settings:

```swift
import AccessCore

let reading = Reading(value: 7)
let source = reading.source
```

```swift
import AccessCore

var reading = Reading(value: 7)
reading.value = 9
```

## In practice

**Choose it when:** Start with the smallest source contract needed by real clients. Reserve open for supported customization.

**Investigate by:** Compile ordinary, same-package, and testable clients separately; test construction and each member operation.

**Strong evidence:** Preserve legitimate clients while reducing public/package commitments, and review emitted-body visibility separately from source access.

The memberwise examples do not mean all private properties disappear. The Swift 6.4 rule omits initialized properties that are less visible than the relevant synthesized access. Uninitialized private storage still restricts initialization; all-private storage does not gain a broadly visible memberwise initializer. A separately synthesized no-argument default initializer is a different overload. A private setter alone is not a private property. [[A3 — Initialization model|A3]] covers initializer collisions and unbound init references during migration.

## References

### Related guides

- [[A3 — Initialization model]]
- [[A10 — Extensions and Retroactive Modeling]]
- [[A15 — Type choices - struct, class, enum, protocol, and actor]]
- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[E7 — Resilience, ABI, and Module Stability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [Access Control](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/)
- [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)
- [SE-0386 — Package Access Modifier](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0386-package-access-modifier.md)
- [SE-0117 — Public access and public overridability](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0117-non-public-subclassable-by-default.md)
- [SE-0497 — Definition visibility](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0497-definition-visibility.md)
