---
tags:
  - swift
  - ios
  - interview-prep
  - generics
  - type-system
---

# B3 — Generics, Constraints, and Same-Type Relationships

**Goal:** Express the element relationship needed to append a different sequence type safely.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A generic body is checked against its declared requirements, not the behavior of current callers. Capability constraints provide operations; same-type constraints relate positions in the API. Equal element types do not imply equal containers, values, lengths, indices, or ordering. State the weakest relationship the implementation actually needs. [Generics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)

## Rules and pitfalls

### Distinguish capability from equality

Sequence supplies traversal. Equatable supplies value comparison and Hashable supplies hashing; neither follows merely from equal element types. Conditional conformances apply capabilities only when their declared constraints hold.

**Common pitfall:** Requiring two collections to have the same concrete type excludes Array and ArraySlice unnecessarily. Relate their Element types when the implementation only transfers elements.

### Check implicit ownership defaults at every boundary

Ordinary generic parameters implicitly require Copyable and Escapable unless those constraints are suppressed where supported. Generalizing a protocol does not automatically generalize every consumer and extension.

**Common pitfall:** A noncopyable value and a nonescapable Span can both fail an ordinary generic call for different reasons. Read the signature and SDK declarations rather than treating all generic rejection as a missing protocol conformance.

## Interview questions

### What does A.Element == B.Element prove beyond A: Sequence and B: Sequence? What additional constraint does equality or hashing need?

It proves both sequences produce the same element type, permitting one sequence's element to enter the other's typed storage. It proves no equality operation exists on those elements. Comparing them needs Equatable, and hash-based operations need Hashable in addition to the required container capabilities.

### Why can a protocol permit a noncopyable Payload while a generic use still requires it to be Copyable? How do primary and non-primary associated types differ here?

The generic use can impose its own implicit Copyable requirement even if the protocol allows a noncopyable associated payload. Swift 6.4 primary associated-type defaults can inherit relevant suppression, while non-primary associated types and generic uses need explicit treatment. Check protocol, associated type, and consumer signatures independently.

### Why can a Token fail an ordinary generic parameter, while a Span fails for a different implicit constraint?

Token can fail the implicit Copyable constraint; Span can fail the distinct Escapable default because it is a lifetime-dependent view. Suppressing one does not suppress the other. A generalized API must also avoid operations whose implementation genuinely needs copying or escape.

### Why can mixed array literals satisfy a same-element constraint, and why should a negative interview probe use explicit types?

Inference can choose Any as a shared element type for mixed literals, making the supposed negative example valid. Use explicitly typed [Int] and [String] variables to test a real mismatch. The compiler verifies the types after permitted inference and conversions, not the intended interview trap.

## Focused exercise

**Task:** Repair appendAll with the required element relationship. Check Array and ArraySlice inputs and an explicitly typed mismatch.

**Acceptance checks:**

- Array and ArraySlice inputs append in traversal order.
- Empty input preserves the target.
- An explicitly String-element input rejects for an Int target.

**Solution:**

Standalone append helper:

```swift
func appendAll<S: Sequence>(_ source: S, to target: inout [S.Element]) {
    for element in source { target.append(element) }
}
var target = [1]
let source: [Int] = [2, 3]
appendAll(source, to: &target)
appendAll(source[1...], to: &target)
precondition(target == [1, 2, 3, 3])
appendAll([Int](), to: &target)
precondition(target == [1, 2, 3, 3])
```

A separate call with let wrong: [String] = ["x"] and the same Int target intentionally rejects. The target element is S.Element, so the needed equality is expressed directly in this signature.

**Reasoning:** The target fixes the element relationship without requiring the source container to be Array. The loop needs only Sequence traversal, so accepting Collection would add an unnecessary restriction. This ordinary helper remains copyable-element-oriented; borrowing iteration needs a different ownership-aware contract.

## In practice

**Choose it when:** Generalize operations only over the capabilities and relationships they actually use.

**Investigate by:** Write explicit input types and inspect each implicit ownership constraint.

**Strong evidence:** Explain accepted heterogeneous container types and rejected mismatched elements without claiming specialization or runtime equality.

### Suppression defaults and borrowing iteration — optional

#### Swift 6.4 associated-type suppression and defaults

This standalone source uses a noncopyable associated type as the default witness:

```swift
struct PayloadToken: ~Copyable {
    let id: Int
}

protocol PayloadSource<Payload>: ~Copyable {
    associatedtype Payload: ~Copyable = PayloadToken
    func makePayload() -> Payload
}

struct TokenSource: PayloadSource {
    func makePayload() -> Payload { PayloadToken(id: 7) }
}

func produce<S: PayloadSource & ~Copyable>(
    _ source: borrowing S
) -> S.Payload where S.Payload: ~Copyable {
    source.makePayload()
}
```

`TokenSource` uses the default `PayloadToken`; `produce` supports both a noncopyable conformer and a noncopyable primary associated type by suppressing each relevant default. The result is owned by the caller. The proposal also supports `~Escapable`, but that requires separate lifetime reasoning. [SE-0503](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md)

Expected failure, reusing the source declarations:

```swift
func ordinaryProduce<S: PayloadSource>(_ source: S) -> S.Payload {
    source.makePayload()
}
func cannotUseDefaultPayload() {
    _ = ordinaryProduce(TokenSource())
}
```

Although the protocol permits a noncopyable payload, this ordinary generic use defaults the primary associated type back to `Copyable` and rejects `PayloadToken`. Ordinary non-primary associated types do not get this same extra use-site default merely because they are associated types.

The distinction also matters in protocol extensions. Expected failure:

```swift
extension PayloadSource {
    func ordinaryPayload() -> Payload { makePayload() }
}
func cannotUseDefaultExtension() {
    _ = TokenSource().ordinaryPayload()
}
```

Broaden the extension explicitly when its implementation permits it:

```swift
extension PayloadSource where Self: ~Copyable, Payload: ~Copyable {
    func withPayload<R>(_ body: (consuming Payload) -> R) -> R {
        body(makePayload())
    }
}
```

The callback consumes the newly created payload; its result `R` retains ordinary defaults. The checked example requires no experimental feature flag. A proposal's historical implementation flag is not a reason to enable that flag in a released-toolchain project.

#### Borrowing iteration requires its own generic contract

Standalone visitor, used on the OS 27 path:

```swift
@available(anyAppleOS 27.0, *)
func visit<I: Iterable & ~Copyable & ~Escapable>(
    _ values: borrowing I,
    _ body: (borrowing I.Element) -> Void
) throws(I.Failure) where I.Element: ~Copyable {
    for try value in values { body(value) }
}
```

This preserves the iteration failure type and borrows elements. It does not imply collection indexing or async work. In the checked SDK an array itself does not conform to `Iterable`; a suitable `.span` view does. Keep `Sequence`/`Collection` when their contracts are the ones the algorithm needs. C5 provides the full positive/negative matrix and deployment discussion.

## References

### Related guides

- [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]
- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]

### Official sources

- [Generics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)
- [SE-0503](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md)
- [SE-0143: Conditional conformances](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0143-conditional-conformances.md)
- [SE-0427: Noncopyable generics](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md)
- [SE-0446: Nonescapable types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0446-non-escapable.md)
- [SE-0452: Integer generic parameters](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0452-integer-generic-parameters.md)
- [SE-0516: Borrowing iteration](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md)
