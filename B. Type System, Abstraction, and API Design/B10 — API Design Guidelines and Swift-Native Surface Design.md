---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
---

# B10 — API Design Guidelines and Swift-Native Surface Design

**Goal:** Expose a clear public operation with validated domain inputs and explicit outcomes.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An API contract combines call-site meaning, construction, effects, ownership, and isolation. Labels explain roles; distinct domain types reject categories of misuse; validation rejects invalid values inside those categories. Naming should describe actual mutation and cost. Established pairs such as sort/sorted help callers predict behavior. [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)

## Rules and pitfalls

### Design a usable boundary

Read realistic client calls, including object construction. A public property or type does not publish a public memberwise initializer. Return a named outcome when callers need different success/failure states, and avoid forcing them to infer policy from primitive values.

**Common pitfall:** Better labels do not prevent two same-typed IDs being swapped. Use distinct ID types where that mistake matters and validate their domain on construction.

### Make ownership and execution honest

Consuming transfers a noncopyable value into the callee; for a copyable value, the caller may preserve it by copying. Async permits suspension and throws describes an error path; neither promises offloading, rollback, or cancellation of external effects.

**Common pitfall:** Declaring async throws while silently starting unowned work leaves lifetime and publication undefined. State task ownership and use @concurrent only when leaving the caller's actor is part of the contract.

## Interview questions

### Which misuse does a label reveal, which does a distinct ID type reject, and which still needs validation?

A label reveals an argument's role to a reader. Distinct types can reject swapped user and document IDs at compile time. Both can still contain an empty or malformed identifier unless construction validates it. These are complementary layers, not substitutes.

### What does consuming mean for a noncopyable draft compared with a copyable value?

A consuming noncopyable draft transfers ownership and prevents later use of that consumed value. A consuming copyable parameter can receive a copy while the caller retains its own value. Neither says the operation commits atomically or makes referenced payloads deeply unique.

### How would a separate client expose missing public constructors or accidental isolation requirements?

An ordinary separate client must import, construct inputs, and call from its intended isolation context. That exposes internal-only synthesized initializers and accidental MainActor requirements that same-module or testable code can hide. Compile a rejected invalid-input case separately from access failures.

## Focused exercise

**Task:** Rewrite one ambiguous operation with typed inputs and explicit outcomes. Compile an ordinary client and check an invalid-input boundary.

**Acceptance checks:**

- An ordinary client can construct and call the public surface.
- Empty identifiers reject; self-follow has an explicit outcome.
- Argument labels communicate the two user roles.

**Solution:**

Two-module fixture. Core.swift exports validated IDs and a small operation; the client needs an ordinary import, not @testable.

```swift
// Core.swift, module Core
public struct UserID: Equatable {
    public let rawValue: String
    public init?(_ raw: String) {
        guard !raw.isEmpty else { return nil }
        rawValue = raw
    }
}
public enum FollowOutcome: Equatable { case followed(UserID); case selfFollow }
public func follow(user target: UserID, as viewer: UserID) -> FollowOutcome {
    target == viewer ? .selfFollow : .followed(target)
}
```

```swift
// Client.swift
import Core
let viewer = UserID("viewer")!
let target = UserID("target")!
precondition(follow(user: target, as: viewer) == .followed(target))
precondition(follow(user: viewer, as: viewer) == .selfFollow)
precondition(UserID("") == nil)
```

Use A11's separate-module build process. This fixture returns a domain decision; it performs no remote follow request.

**Reasoning:** Both inputs are UserID because they share one domain, while labels distinguish their roles. If the domain has different ID categories, introduce different types. A real network operation would additionally define authentication, cancellation, idempotency, and publication policy.

## In practice

**Choose it when:** Review representative consumer tasks before adding abstraction or overloads.

**Investigate by:** Compile ordinary clients and test the boundary's invalid values and effects.

**Strong evidence:** Weigh caller clarity, ownership, isolation, and migration while demonstrating usable public construction.

### Intentional non-sendability and ownership — optional

- **Counting words instead of examining use.** More labels are not always clearer, and fewer are not always better.
- **Adding a protocol for every concrete type.** Introduce an abstraction for a real substitution or ownership boundary, not merely to make the declaration look architectural.
- **Turning every flag into a wrapper with one Boolean.** That wrapper is useful only if it adds a meaningful concept, invariant, or future contract.
- **Using an options value as a miscellaneous container.** Group related choices, keep required inputs visible, and document defaults as behavior.
- **Claiming an implementation resizes an image when it only returns the input.** A signature sketch belongs in a protocol or explicitly labeled pseudocode, not under a “working implementation” claim.
- **Assuming `public` properties provide a public initializer.** A public struct's synthesized memberwise initializer is not automatically public. Exercise the API from another module.
- **Using a UI singleton in a neutral request's default argument.** Obtain environment-specific scale from the actual UI context and pass the resulting value explicitly; UI access and portable request construction have different isolation requirements.

#### Intentional non-sendability in 6.4

```swift
struct LocalOptions: ~Sendable {
    var retryLimit: Int
}

func copyLocalOptions() -> Int {
    let original = LocalOptions(retryLimit: 1)
    var copy = original
    copy.retryLimit = 3
    return original.retryLimit + copy.retryLimit
}
```

`~Sendable` suppresses inference even though this stored `Int` could otherwise support it. The type remains copyable: the function returns 4. It does not acquire thread affinity, locking, or an actor owner. The following addition to those declarations intentionally fails:

```swift
func requireSendable<T: Sendable>(_ value: T) {}

func rejectedOptions() {
    requireSendable(LocalOptions(retryLimit: 1))
}
```

Use suppression to record an audited API decision, rather than as decoration on every local value. It does not categorically forbid a checked `sending` transfer of a disconnected value. Conditional conformance and subclass rules have additional details in D6. [SE-0518](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0518-tilde-sendable.md)

#### Ownership belongs in the surface when it matters

```swift
struct Document: Sendable, Equatable {
    let title: String
}

struct Draft: ~Copyable {
    var title: String

    borrowing func preview() -> String { title }
    consuming func finalized() -> Document { Document(title: title) }
}

func finalizeDraft() -> Document {
    let draft = Draft(title: "Notes")
    _ = draft.preview()
    return draft.finalized()
}
```

Preview borrows; finalization consumes this noncopyable draft. This models an in-memory transition, not publishing or writing a file. Reuse after consumption is rejected. Add this separately to the declarations above:

```swift
func reuseDraft() -> String {
    let draft = Draft(title: "Notes")
    _ = draft.finalized()
    return draft.preview() // Expected use-after-consume rejection.
}
```

The ownership modifiers predate 6.4. A consuming parameter of a copyable type does not automatically impose the same last-use restriction because a copy may satisfy ownership. Use `~Copyable` only when the domain or resource model justifies it. [SE-0377](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)

Swift 6.4's `Ref` and `MutableRef` provide lifetime-bound shared/exclusive access to one value. Prefer ordinary borrowing parameters or `inout` when they express the operation; references become useful when the view itself is part of a generic API. They do not own the referent or make a valid asynchronous escape. In the checked Apple SDK their operations require OS 27; C7/C12 carry the detailed lifetime, exclusivity, and availability probes. [SE-0519](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md)

## References

### Related guides

- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A10 — Extensions and Retroactive Modeling]]
- [[A11 — Access Control]]
- [[A13 — Error handling and typed throws]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[E6 — Import Visibility and Dependency Leakage]]

### Official sources

- [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [SE-0518](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0518-tilde-sendable.md)
- [SE-0377](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)
- [SE-0519](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md)
