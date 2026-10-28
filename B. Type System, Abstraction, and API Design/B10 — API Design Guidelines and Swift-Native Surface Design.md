---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
---

# B10 — API Design Guidelines and Swift-Native Surface Design

**Learning goals:**

- Design calls with honest roles, validated domain inputs, mutation behavior, and explicit outcomes.
- Verify ordinary external construction and the intended isolation/execution contract.
- Distinguish copyable consumption, exclusive noncopyable ownership, ~Sendable, and lifetime-bound reference APIs.

**Prerequisites:** [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for labels/defaults/inout, [[A11 — Access Control]] for access, [[A13 — Error handling and typed throws]] for outcomes, and [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]] for capability boundaries. Ownership/execution extensions use [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]/[[C8 — Copyable, noncopyable types, and `~Copyable`]] and [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]].

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

open(7, 42, true) makes a reader infer argument roles, units, and effects. Labels can explain the roles; distinct ID types can reject category swaps; validated construction can reject invalid values. Those layers solve different problems.

An API also includes usable construction, mutation, result shape, ownership, and isolation. A clear name must match the actual effect: sorted returns a new value, while sort changes its receiver. async permits suspension and throws exposes failure; neither promises offloading, atomic rollback, or cancellation of external work. [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Make names match mutation

NameList illustrates the established sort/sorted distinction. Use Array directly when a wrapper supplies no additional domain contract.

```swift
struct NameList: Equatable {
    private(set) var names: [String]
    init(_ names: [String]) { self.names = names }
    mutating func sort() { names.sort() }
    func sorted() -> NameList {
        var result = self
        result.sort()
        return result
    }
}
var original = NameList(["Zoe", "Ada"])
let ordered = original.sorted()
precondition(original.names == ["Zoe", "Ada"])
precondition(ordered.names == ["Ada", "Zoe"])
original.sort()
precondition(original == ordered)
```

**Walkthrough:** The nonmutating operation preserves original; the mutating one changes it. A let binding permits sorted but cannot supply the write access sort needs.

Prepend NameList setup. This **intentionally rejected** call violates that mutation contract:

```swift
let immutable = NameList(["Zoe", "Ada"])
immutable.sort()
```

### Shared public API setup: FollowCore

Compile the next block as a separate module FollowCore, adding `-parse-as-library -module-name FollowCore -emit-module -emit-module-path /tmp/FollowCore.swiftmodule -c -o /tmp/FollowCore.o` to the baseline command. Ordinary clients add `-I /tmp` and link `/tmp/FollowCore.o`; validation uses its isolated temporary directory. This operation returns a domain decision and sends no remote request.

```swift
public struct UserID: Equatable, Sendable {
    public let rawValue: String
    public init?(_ raw: String) {
        guard !raw.isEmpty else { return nil }
        rawValue = raw
    }
}
public enum FollowOutcome: Equatable, Sendable {
    case followed(UserID)
    case selfFollow
}
public func follow(user target: UserID, as viewer: UserID) -> FollowOutcome {
    target == viewer ? .selfFollow : .followed(target)
}
```

## Rules and pitfalls

### Labels expose roles but do not make distinct ID categories

**Common pitfall:** Swapping two Int values despite descriptive labels.

**Mistake:** Standalone runnable example.

```swift
func open(documentID: Int, asUser userID: Int) -> String { "document:\(documentID),user:\(userID)" }
let userID = 7
let documentID = 42
let wrong = open(documentID: userID, asUser: documentID)
precondition(wrong == "document:7,user:42")
```

**Consequence:** The wrong categories compile and are forwarded unchanged.

**Why:** Labels help readers notice a swap; both arguments still have the same static Int type.

**Correction:** Use distinct domain types when the categories are distinct, and validate values within each category.

```swift
struct UserID {
    let value: Int
    init?(_ value: Int) {
        guard value > 0 else { return nil }
        self.value = value
    }
}
struct DocumentID {
    let value: Int
    init?(_ value: Int) {
        guard value > 0 else { return nil }
        self.value = value
    }
}
func openDocument(document: DocumentID, for user: UserID) -> String {
    "\(document.value):\(user.value)"
}
let user = UserID(7)!
let document = DocumentID(42)!
precondition(openDocument(document: document, for: user) == "42:7")
precondition(UserID(0) == nil && DocumentID(-1) == nil)
```

**What changes:** The type system separates categories and the constructors reject invalid members of each category. Two same-domain users can still use one UserID type with labels for viewer/target roles.

Prepend the distinct-ID correction. Passing a user where a document is required is **intentionally rejected**:

```swift
let invalid = openDocument(document: user, for: user)
```

### async does not define ownership of a discarded task

**Common pitfall:** Returning after starting unowned work, even though callers may read the signature as completion of publication. Compile-only; this unjoined task is never started.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
actor Store {
    private var value: Int?
    func publish(_ value: Int) { self.value = value }
}
func publish(_ value: Int, to store: Store) async throws {
    Task { await store.publish(value) }
}
```

**Consequence:** Awaiting this function need not wait for the separately created task to publish. The unnecessary throws also communicates a failure path the body does not implement.

**Why:** Task creates unstructured work with its own lifetime. The outer async signature alone does not join it, cancel it, or define late-publication policy.

**Correction:** Await the actual operation when that is the completion contract; expose task ownership explicitly only when work must outlive the call.

```swift
actor Store {
    private var value: Int?
    func publish(_ value: Int) { self.value = value }
    func read() -> Int? { value }
}
func publish(_ value: Int, to store: Store) async { await store.publish(value) }
let store = Store()
await publish(7, to: store)
let observed = await store.read()
precondition(observed == 7)
```

**What changes:** Return now means the modeled publication completed. A real service additionally needs error, cancellation, idempotency, and freshness contracts; @concurrent is an execution choice, not a task-ownership repair.

For the following construction pitfall, compile this separate **compile-only** module ConstructorOnly with the shared module command:

```swift
public struct Label { public let text: String }
```

### Public properties do not publish an initializer

**Common pitfall:** Testing only inside a module and overlooking ordinary client construction. The mistake is a separate ConstructorOnly client.

**Mistake:** Compile-only; intentionally rejected.

```swift
import ConstructorOnly

let label = Label(text: "Ada")
```

**Consequence:** The ordinary client cannot call the internal synthesized initializer.

**Why:** Construction permission is separate from type/property visibility. @testable tests could hide this missing public surface.

**Correction:** Offer an explicit initializer for supported external construction. FollowCore already publishes a validated one.

```swift
import FollowCore

let user = UserID("Ada")!
precondition(user.rawValue == "Ada")
precondition(UserID("") == nil)
```

**What changes:** The ordinary import can construct and observe the value while invalid construction remains explicit.

## Worked example

### A clear boundary can return a domain outcome without I/O

**Setup:** Separate runnable FollowCore client.

**Predict:** How does a self-follow differ from a normal request?

```swift
import FollowCore

let viewer = UserID("viewer")!
let target = UserID("target")!
precondition(follow(user: target, as: viewer) == .followed(target))
precondition(follow(user: viewer, as: viewer) == .selfFollow)
```

**Outcome:** The operation returns followed(target) or selfFollow without a magic Bool or an actual remote effect.

**Why:** Both arguments are users, so one ID category is appropriate. Labels distinguish their roles and the outcome names the domain decision.

## Interview questions

### Which misuse does a label reveal, which does a distinct ID type reject, and which still needs validation?

**Answer:** Labels reveal roles to readers. Distinct domain types reject category swaps. A constructor must still validate values inside a category. Use each layer for the problem it actually solves.

**Example:** Standalone runnable example.

```swift
struct UserID {
    let value: Int
    init?(_ value: Int) {
        guard value > 0 else { return nil }
        self.value = value
    }
}
struct DocumentID {
    let value: Int
    init?(_ value: Int) {
        guard value > 0 else { return nil }
        self.value = value
    }
}
func canOpen(document: DocumentID, for user: UserID) -> Bool {
    document.value > 0 && user.value > 0
}
let user = UserID(7)!
let document = DocumentID(42)!
precondition(canOpen(document: document, for: user))
precondition(UserID(0) == nil && DocumentID(-1) == nil)
```

**Walkthrough:** The labels describe a document and its user. Their types forbid swapping those categories; their failable initializers reject zero/negative IDs. The pitfall includes a rejected call that passes a user in the document position.

**Boundary:** This positive-number policy is a teaching domain. Real IDs need their actual format and uniqueness scope; adding wrappers around arbitrary primitives is useful only when they express a meaningful invariant.

### What does consuming mean for noncopyable versus copyable drafts?

**Answer:** It transfers a noncopyable value and prevents later use of that consumed value. A copyable argument can supply a copy while the caller retains its own value. Neither contract promises atomic commit or a deeply unique graph.

**Example:** Standalone runnable example.

```swift
struct Draft: ~Copyable {
    let title: String
    borrowing func preview() -> String { title }
    consuming func finalized() -> String { title }
}
struct CopyableDraft { var title: String }
func finish(_ draft: consuming CopyableDraft) -> String { draft.title }
func checkDrafts() {
    let exclusive = Draft(title: "Notes")
    precondition(exclusive.preview() == "Notes")
    let result = exclusive.finalized()
    precondition(result == "Notes")
    let copyable = CopyableDraft(title: "Copy")
    precondition(finish(copyable) == "Copy")
    precondition(copyable.title == "Copy")
}
checkDrafts()
```

**Walkthrough:** Preview borrows the exclusive draft; finalization consumes it. The copyable caller still reads its retained value after a consuming call. [SE-0377](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)

**Boundary:** Use ~Copyable for a justified ownership/resource contract, not to decorate every value. This in-memory finalization neither saves a file nor publishes a remote change.

Prepend that Draft setup. This **compile-only, intentionally rejected** reuse occurs after consuming finalization:

```swift
func invalidReuse() -> String {
    let draft = Draft(title: "Notes")
    _ = draft.finalized()
    return draft.preview()
}
```

### How does a separate client expose missing constructors or accidental isolation?

**Answer:** Make an ordinary import construct inputs and call from the context real clients need. A same-file or @testable test can hide internal construction; a top-level MainActor-only call can hide an unwanted UI isolation requirement.

**Example:** Separate runnable FollowCore client, compiled with the chapter baseline and no testable import.

```swift
import FollowCore

func nonisolatedClient() -> FollowOutcome {
    guard let viewer = UserID("viewer"), let target = UserID("target") else {
        preconditionFailure()
    }
    return follow(user: target, as: viewer)
}
precondition(nonisolatedClient() == .followed(UserID("target")!))
```

**Walkthrough:** This function uses the baseline nonisolated declaration default, so the ordinary client checks both construction and the intended neutral call context. The construction pitfall supplies the rejected missing-public-initializer case.

**Boundary:** The validation also rebuilds the API under a different declaration default and checks the client boundary. Imported modules retain their own compiled isolation contracts; changing the caller’s settings does not rewrite them.

### Extension: how does ~Sendable differ from ~Copyable or actor ownership?

**Answer:** ~Sendable suppresses sendability inference while retaining copyability. It does not add a lock, thread affinity, or actor owner. A checked sending transfer of disconnected state is a separate operation from satisfying a generic Sendable requirement.

**Example:** Standalone runnable example.

```swift
struct LocalOptions: ~Sendable { var retryLimit: Int }
func copyProbe() -> Int {
    let original = LocalOptions(retryLimit: 1)
    var copy = original
    copy.retryLimit = 3
    return original.retryLimit + copy.retryLimit
}
actor Worker {
    private var retries = 0
    func install(_ options: sending LocalOptions) { retries = options.retryLimit }
    func value() -> Int { retries }
}
func transferProbe() async -> Int {
    let worker = Worker()
    let options = LocalOptions(retryLimit: 2)
    await worker.install(options)
    return await worker.value()
}
precondition(copyProbe() == 4)
let transferred = await transferProbe()
precondition(transferred == 2)
```

**Walkthrough:** The local options copy remains independent, but the type does not satisfy Sendable by inference. The second operation transfers a disconnected local value through sending and does not reuse its old region. [SE-0518](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0518-tilde-sendable.md)

**Boundary:** Suppression should record an audited API choice, not be applied indiscriminately. [[D6 — `Sendable`, `@Sendable`, and `sending`|D6]] explains sending regions and conformance/subclass rules.

Prepend LocalOptions setup. This **intentionally rejected** generic use actually requires Sendable:

```swift
func requireSendable<T: Sendable>(_ value: T) {}
requireSendable(LocalOptions(retryLimit: 1))
```

### Extension: when should Ref/MutableRef appear instead of ordinary borrowing or inout?

**Answer:** Use ordinary parameter ownership when it already expresses the operation. A reference type is useful when the dependent access view itself is part of a generic API. Ref reads shared access; MutableRef holds exclusive mutable access, and neither owns an indefinitely escaping referent.

**Example:** Standalone runnable example.

```swift
@available(anyAppleOS 27.0, *)
func increment(_ value: inout Int) { value += 1 }
@available(anyAppleOS 27.0, *)
func referenceProbe() -> Int {
    var value = 10
    do {
        let reference = Ref(value)
        precondition(reference.value == 10)
    }
    do {
        var reference = MutableRef(&value)
        reference.value += 2
    }
    increment(&value)
    return value
}
precondition(referenceProbe() == 13)
```

**Walkthrough:** The read reference finishes before mutable access begins. The mutable view finishes before the final inout call. These OS 27 APIs expose lifetime/exclusivity constraints, not a new owner or synchronization primitive. [SE-0519](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md)

**Boundary:** Do not hold an exclusive view across an overlapping read or assume a copyable getter projects original storage. [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`|C7]]/[[C12 — `Span`, `InlineArray`, and modern low-level abstractions|C12]] cover those lifetime and accessor limits; this fixture makes no async escape.

## Focused exercise

**Task:** Expose one clear public operation with validated inputs and explicit outcomes, then compile an ordinary consumer.

**Acceptance checks:**

- The client can construct and call the public surface without testable access.
- Empty IDs reject; normal and self-follow outcomes differ.
- Labels communicate the target/viewer roles and the function performs only its stated domain decision.

**Solution:** Separate runnable FollowCore client. IDs are nonempty strings under this fixture’s explicit policy.

```swift
import FollowCore

let viewer = UserID("viewer")!
let target = UserID("target")!
precondition(UserID("") == nil)
precondition(viewer.rawValue == "viewer" && target.rawValue == "target")
precondition(follow(user: target, as: viewer) == .followed(target))
precondition(follow(user: viewer, as: viewer) == .selfFollow)
```

**Reasoning:** The explicit public initializer publishes usable construction; labels explain two roles within the same UserID domain; the named outcome states the result. A real remote-follow API needs its own authentication, failure, cancellation, idempotency, and publication contract.

## In practice

**Choose it when:** Review representative consumer tasks before adding protocols, options bags, wrappers, overloads, or a DSL.

**Investigate by:** Compile ordinary clients, test invalid construction and mutation, and trace actual work ownership and actor requirements.

**Strong evidence:** Balance clarity, ownership, isolation, and migration with evidence from usable public construction; name only behavior the implementation actually performs.

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
