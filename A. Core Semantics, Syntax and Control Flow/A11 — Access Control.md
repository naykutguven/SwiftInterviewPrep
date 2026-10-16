---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - access-control
---

# A11 — Access Control

**Goal:** Expose a constructible public value and verify its access permissions from an ordinary client.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Access control grants specific source permissions: naming a type, constructing it, reading or writing members, conforming, subclassing, and overriding are separate operations. Public visibility of a struct does not expose a public synthesized initializer. Access control neither synchronizes state nor protects secrets in a shipped binary. [Access Control](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/)

## Rules and pitfalls

### Test the actual boundary

Private means the enclosing declaration and its same-file extensions; fileprivate includes unrelated declarations in the file. Internal covers a module; package covers modules with the same package identity. Public permits external use. Open additionally permits supported external subclassing or overriding.

**Common pitfall:** Testing only inside the module, or with @testable, conceals missing public initializers and members. Compile an ordinary client in a separate module and check each needed permission.

### Separate setter access and emitted implementation

Private(set) narrows assignment to the property through its setter, not mutation inside arbitrary referenced objects. @usableFromInline exposes ABI-callable internal implementation; @inlinable exposes a body under visibility constraints. Neither creates public source access to an internal helper.

**Common pitfall:** Treating export/interface controls as access modifiers confuses source naming with implementation distribution. Inspect both client source use and emitted interfaces; see [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]].

## Interview questions

### Why can a sibling package module name a struct but fail to construct it, and why does Swift 6.4 not fix that automatically?

The struct's visibility and its initializer's access differ. Memberwise synthesis remains at most internal, including after Swift 6.4 changes which initialized private properties participate. Declare a package or public initializer explicitly for the intended clients rather than assuming the type modifier publishes construction.

### How do ordinary external, same-package, and testable clients differ when reading members or subclassing/overriding?

Ordinary external clients see public/open surfaces. Same-package clients additionally see package declarations under matching package identity. Testable import with an enable-testing build permits extra internal access and testing customization; it does not prove ordinary source compatibility or expose private state. Open is the supported external customization promise.

### Why can a same-file type extension read private state while an unrelated helper or another-file extension cannot?

Swift permits the owning type's same-file extensions to participate in its private implementation. An unrelated helper is outside that private context; another-file extension loses the same-file permission. Use fileprivate only when unrelated declarations in the file deliberately share the implementation.

### How do private(set), usableFromInline, inlinable, and export(interface) constrain different aspects of an API?

Private(set) constrains setter access. @usableFromInline commits a source-internal declaration to ABI use. @inlinable exposes an eligible implementation body to clients. @export(interface) controls exported interface implementation policy under its rules; it does not let ordinary clients name private/internal declarations. Review each axis separately.

## Focused exercise

**Task:** Expose a small public value from a module. Compile an ordinary client that constructs and reads it, and reject private member access.

**Acceptance checks:**

- An ordinary import can construct and read Reading.
- Private state and the restricted setter reject external access.
- The check uses no @testable import.

**Solution:**

Two source files in separate modules; compile Core first, then Client using its module and library search paths.

```swift
// Core.swift, module Core
public struct Reading {
    public private(set) var value: Int
    private let source: String
    public init(value: Int) {
        self.value = value
        self.source = "sensor"
    }
}
```

```swift
// Client.swift
import Core
let reading = Reading(value: 7)
precondition(reading.value == 7)
```

Build Core with swiftc -emit-library -emit-module -module-name Core; pass F5's baseline flags. Compile the client with -I and -L pointing at the output directory and -lCore. In a separate rejected client, read reading.source or assign reading.value.

**Reasoning:** The explicit initializer publishes construction; public getter visibility publishes observation. These permissions do not expose the private source or mutation API. A successful same-file compile would not test this boundary.

## In practice

**Choose it when:** Start with the smallest source contract needed by real clients; reserve open for supported customization.

**Investigate by:** Compile ordinary, package, and testable consumers separately.

**Strong evidence:** Preserve legitimate clients while reducing package/public commitments and checking emitted-body visibility.

### Memberwise access compatibility — optional

#### Swift 6.4 memberwise synthesis and compatibility

This standalone probe is shared with the rubric:

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

func initializerAccessProbe() -> [Int] {
    let fresh = Draft(title: "New")
    let restored = Draft.restored(title: "Old", revision: 7)
    return [fresh.revisionNumber, restored.revisionNumber]
}
```

It returns `[0, 7]`. The new initializer omits the initialized less-visible revision; the same-file extension can still use the private compatibility overload that includes it. No experimental flag is needed for this checked Swift 6.4 source. [SE-0502](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0502-exclude-private-from-memberwise-init.md)

This is not “all private properties disappear.” The rule compares memberwise-initializable properties against the maximum relevant access, capped at internal, and excludes less-visible properties only when they have an initial value (including a default-nil optional). An uninitialized private field still constrains initialization. A type whose relevant fields are all private does not gain a broadly accessible memberwise initializer. A separately synthesized no-argument default initializer can still allow construction from those defaults; do not confuse it with the private memberwise overload. A private setter alone is not the same as a private property for this rule.

The synthesized initializer remains at most internal. This public type still cannot be constructed by an ordinary external client through synthesis:

```swift
public struct PublicLabel {
    public var text: String
    private var revision = 0
}
```

Add an explicit public initializer to offer public construction. When moving from 6.2/6.3, inspect unbound references such as Draft.init and extension initializers whose signatures collide with the newly synthesized form; A3 validates those compatibility cases. Do not remove explicit API initializers just because the same-module spelling now compiles.

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
