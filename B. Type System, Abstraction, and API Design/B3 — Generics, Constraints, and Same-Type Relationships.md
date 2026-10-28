---
tags:
  - swift
  - ios
  - interview-prep
  - generics
  - type-system
---

# B3 — Generics, Constraints, and Same-Type Relationships

**Learning goals:**

- State the capability and same-type relationships a generic implementation actually uses.
- Distinguish equal types from equal values, lengths, indices, or order, including literal inference.
- Check Copyable/Escapable and associated-type defaults at protocol, generic, and extension boundaries.

**Prerequisites:** [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]] for protocols and associated types; [[A7 — Functions, Parameter Labels, Defaults, and API Surface]] for function parameters and inout. Ownership extensions use [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]/[[C8 — Copyable, noncopyable types, and `~Copyable`]] and the Span foundations in [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]].

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Appending an array slice to an array does not require the two containers to have the same type. It requires the source to produce elements and the destination to accept that element type. Generic constraints state those proofs before the body is checked.

A **capability constraint**, such as S: Sequence, supplies operations. A **same-type constraint**, such as S.Element == C.Element, connects type positions. It does not compare runtime values or establish equal lengths, indices, or order. A generic body is checked against these declared requirements, even if every current caller happens to use Int arrays. [Generics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Connect element types without equating container types

Shared setup named appendAll. Sequence supplies traversal; RangeReplaceableCollection supplies element insertion. The where clause connects their elements.

```swift
func appendAll<S: Sequence, C: RangeReplaceableCollection>(_ source: S, to target: inout C)
where S.Element == C.Element {
    for element in source { target.append(element) }
}
var target = [1]
let values = [2, 3]
appendAll(values[1...], to: &target)
precondition(target == [1, 3])
var contiguous = ContiguousArray<Int>()
appendAll(values, to: &contiguous)
precondition(Array(contiguous) == [2, 3])
```

**Walkthrough:** ArraySlice<Int>, Array<Int>, and ContiguousArray<Int> are different containers with compatible element types. The loop uses traversal order and never assumes a slice starts at index zero.

### Conditional conformance adds a capability only where supported

The generic wrapper can hold a non-Equatable payload, but its equality conformance exists only for Equatable payloads.

```swift
struct Box<Value> { let value: Value }
extension Box: Equatable where Value: Equatable {}
func equal<T: Equatable>(_ first: T, _ second: T) -> Bool { first == second }
precondition(equal(Box(value: 7), Box(value: 7)))
precondition(!equal(Box(value: 7), Box(value: 8)))
let behavior = Box(value: { 7 })
precondition(behavior.value() == 7)
```

**Walkthrough:** Box<Int> has the equality capability; Box<() -> Int> is still a usable wrapper without it. Conditional conformance does not make arbitrary closures equatable. [SE-0143](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0143-conditional-conformances.md)

### Ordinary generic defaults are constraints too

An ordinary generic T implicitly requires Copyable and Escapable. Copyable permits duplication of the value. Escapable permits values that are not tied to a borrowed lifetime. A noncopyable token and a nonescapable Span can fail different defaults. Suppression removes a default requirement; the implementation must still avoid operations that need copying or an escaping lifetime. The ownership examples below are separate from the copyable-element append helper.

## Rules and pitfalls

### Traversal alone does not prove compatible element types

**Common pitfall:** Appending S.Element into C without stating that it is C.Element.

**Mistake:** Compile-only; intentionally rejected.

```swift
func invalid<S: Sequence, C: RangeReplaceableCollection>(_ source: S, to target: inout C) {
    for element in source { target.append(element) }
}
```

**Consequence:** The generic body fails because the destination expects C.Element, which can differ from S.Element.

**Why:** Two independent capability constraints do not connect their associated types.

**Correction:** State the equality used by insertion. This standalone correction includes its own helper and driver.

```swift
func appendAll<S: Sequence, C: RangeReplaceableCollection>(_ source: S, to target: inout C)
where S.Element == C.Element {
    for element in source { target.append(element) }
}
var target = [1]
appendAll([2, 3], to: &target)
precondition(target == [1, 2, 3])
```

**What changes:** The body now has the proof needed to insert each source element; it still assumes no relationship between container indices or lengths.

### Requiring identical containers excludes valid callers

**Common pitfall:** Using one container parameter C for both source and destination when only their elements must agree.

**Mistake:** Compile-only; intentionally rejected.

```swift
func appendSame<C: RangeReplaceableCollection>(_ source: C, to target: inout C) {
    target.append(contentsOf: source)
}
func invalid() {
    let values = [1, 2, 3]
    var target: [Int] = []
    appendSame(values[1...], to: &target)
}
```

**Consequence:** The slice and array cannot satisfy this single concrete C contract.

**Why:** ArraySlice and Array have the same element type, but are not the same container type.

**Correction:** Use separate source and destination types and equate only the elements. Prepend appendAll setup.

```swift
let sourceValues = [1, 2, 3]
var destination: [Int] = []
appendAll(sourceValues[1...], to: &destination)
precondition(destination == [2, 3])
```

**What changes:** The slice works because the signature states the relationship actually used by the operation.

### borrowing alone does not remove Copyable defaults

**Common pitfall:** Passing a noncopyable token to an ordinary generic even though its parameter is borrowed.

**Mistake:** Compile-only; intentionally rejected.

```swift
struct Token: ~Copyable { let id: Int }
func inspect<T>(_ value: borrowing T) -> Int { 1 }
func invalid() -> Int {
    let token = Token(id: 7)
    return inspect(token)
}
```

**Consequence:** The compiler rejects Token’s failure to satisfy the implicit Copyable constraint.

**Why:** borrowing describes this parameter’s access. The separate T constraint still requires a copyable type.

**Correction:** Suppress the default when the implementation only borrows. This helper lets a callback inspect the actual payload.

```swift
struct Token: ~Copyable { let id: Int }
func inspect<T: ~Copyable, R>(_ value: borrowing T, using body: (borrowing T) -> R) -> R {
    body(value)
}
func probe() -> Int {
    let token = Token(id: 7)
    return inspect(token) { $0.id }
}
precondition(probe() == 7)
```

**What changes:** The helper supports noncopyable input without consuming it. R still has ordinary defaults, and Escapable remains a separate input requirement.

## Worked example

### Same elements, different indices and lengths

**Setup:** Prepend appendAll setup. An array slice retains its original collection indices.

**Predict:** Does appending need index zero, equal lengths, or identical container types?

```swift
let source = [10, 20, 30]
let slice = source[1...]
precondition(slice.startIndex == 1)
var destination = [99]
appendAll(slice, to: &destination)
precondition(destination == [99, 20, 30])
precondition(source == [10, 20, 30])
```

**Outcome:** The slice’s elements append in order without changing the source.

**Why:** The loop asks Sequence for successive elements. A shared Element type says nothing about valid container indices, equality of contents, or traversal count.

## Interview questions

### What does A.Element == B.Element prove beyond two Sequence conformances? What do equality and hashing need?

**Answer:** It proves a shared element type, enabling typed transfer. It does not supply value equality or hashing. Comparing contents additionally needs Equatable; hash-based operations need Hashable and the container’s other constraints.

**Example:** Standalone runnable example.

```swift
func sameValues<A: Sequence, B: Sequence>(_ first: A, _ second: B) -> Bool
where A.Element == B.Element, A.Element: Equatable {
    first.elementsEqual(second)
}
func unique<S: Sequence>(_ values: S) -> Set<S.Element> where S.Element: Hashable {
    Set(values)
}
let values = [1, 2, 3]
precondition(sameValues(values[1...], [2, 3]))
precondition(!sameValues(values, [1]))
precondition(unique([1, 1, 2]) == [1, 2])
```

**Walkthrough:** The two different containers can compare because they share Equatable Int elements. The shorter sequence is unequal despite having the same element type. unique states its extra hashing requirement.

**Boundary:** These helpers do not establish a canonical order, equal indices, or specialization. Generalizing equality protocols also does not remove Set’s independent Copyable element requirement.

### Why can a protocol permit a noncopyable payload while its generic use still requires Copyable? How do primary and non-primary types differ?

**Answer:** The consumer has its own defaults. Swift 6.4 permits suppressed/defaulted associated types, but a use of a primary associated-type protocol can introduce an implicit Copyable requirement again. Non-primary associated types do not gain that same extra use-site default merely by being associated types.

**Example:** Standalone runnable example.

```swift
struct PayloadToken: ~Copyable { let id: Int }
protocol PayloadSource<Payload>: ~Copyable {
    associatedtype Payload: ~Copyable = PayloadToken
    func makePayload() -> Payload
}
protocol SecondarySource: ~Copyable {
    associatedtype Payload: ~Copyable
    func makePayload() -> Payload
}
struct TokenSource: PayloadSource, SecondarySource {
    typealias Payload = PayloadToken
    func makePayload() -> Payload { PayloadToken(id: 7) }
}
func produce<S: PayloadSource & ~Copyable>(_ source: borrowing S) -> S.Payload
where S.Payload: ~Copyable { source.makePayload() }
func secondary<S: SecondarySource>(_ source: S) -> S.Payload { source.makePayload() }
func checkPayload() {
    let first = produce(TokenSource())
    let second = secondary(TokenSource())
    precondition(first.id == 7 && second.id == 7)
}
checkPayload()
```

**Walkthrough:** The protocol permits PayloadToken, and produce explicitly preserves that domain at its generic use. SecondarySource’s non-primary Payload also remains supported by secondary. TokenSource itself is copyable here; conformer and payload requirements are independent. [SE-0503](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0503-suppressed-associated-types.md)

**Boundary:** A default associated-type witness is a fallback choice, not a mandate that every conformer use it. The implementation must also obey lifetime rules when suppressing Escapable.

These independent **compile-only, intentionally rejected** uses prepend the PayloadSource setup:

```swift
func ordinary<S: PayloadSource>(_ source: S) -> S.Payload { source.makePayload() }
func invalid() { _ = ordinary(TokenSource()) }
```

```swift
extension PayloadSource { func ordinaryPayload() -> Payload { makePayload() } }
func invalidExtension() { _ = TokenSource().ordinaryPayload() }
```

A broadened extension can instead state the intended defaults explicitly. This runnable block prepends PayloadSource setup:

```swift
extension PayloadSource where Self: ~Copyable, Payload: ~Copyable {
    func withPayload<R>(_ body: (consuming Payload) -> R) -> R { body(makePayload()) }
}
precondition(TokenSource().withPayload { $0.id } == 7)
```

### Why can Token fail an ordinary generic while Span fails a different implicit constraint?

**Answer:** A noncopyable Token violates Copyable. Span is a lifetime-dependent view and violates an ordinary Escapable default. Removing one requirement does not remove the other; a fully borrowing helper can suppress both when its implementation supports that domain.

**Example:** Standalone runnable example.

```swift
struct Token: ~Copyable { let id: Int }
func withBorrowed<T: ~Copyable & ~Escapable, R>(
    _ value: borrowing T, _ body: (borrowing T) -> R
) -> R { body(value) }
func probe() {
    let token = Token(id: 7)
    precondition(withBorrowed(token) { $0.id } == 7)
    let values = [1, 2, 3]
    let view = values.span
    precondition(withBorrowed(view) { $0.count } == 3)
}
probe()
```

**Walkthrough:** Both input values are borrowed, and the returned R values are ordinary owned Ints. The view remains within the array’s lifetime. [SE-0427](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0427-noncopyable-generics.md), [SE-0446](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0446-non-escapable.md)

**Boundary:** Suppressing defaults does not permit storing a borrowed Span indefinitely or copying a noncopyable value. [[C12 — `Span`, `InlineArray`, and modern low-level abstractions|C12]] covers deployment and lifetime requirements of the selected view APIs.

This standalone **compile-only, intentionally rejected** call suppresses Copyable but still requires Escapable:

```swift
func onlyNoncopyable<T: ~Copyable>(_ value: borrowing T) -> Int { 1 }
func invalid() -> Int {
    let values = [1, 2, 3]
    return onlyNoncopyable(values.span)
}
```

### Why can mixed array literals satisfy same-element constraints, and why use explicit types in negative probes?

**Answer:** Inference can select Any as the shared element type. The constraint is then satisfied even though the payload values differ in type. A negative test should fix [Int] and [String] explicitly so it tests the intended mismatch.

**Example:** Standalone runnable inference probe. Compile without -warnings-as-errors to retain any heterogeneous-literal diagnostics.

```swift
func sharedElement<A: Sequence, B: Sequence>(_ first: A, _ second: B) -> Any.Type
where A.Element == B.Element { A.Element.self }
let inferred = sharedElement([1, "a"], [2, false])
precondition(inferred == Any.self)
let explicit: [Int] = [1, 2]
precondition(sharedElement(explicit, explicit[1...]) == Int.self)
```

**Walkthrough:** The mixed literals can become [Any], making both Element witnesses Any. The explicit arrays/slice retain Int. A compiler diagnostic about heterogeneous inference does not make this a same-type violation.

**Boundary:** Types are checked after permitted inference and conversions. Use a function-type specialization for a diagnostic-sensitive negative probe rather than relying on literal intent. The historical validation found direct invalid same-type calls for which this compiler failed to produce a useful diagnostic; the fixed specialization below exposes the intended type mismatch.

Prepend appendAll setup. This **intentionally rejected** specialization fixes both containers and therefore exposes the incompatible elements:

```swift
let invalid: ([String], inout [Int]) -> Void = appendAll
```

### Extension: why does borrowing iteration need a different generic contract?

**Answer:** It can borrow elements from a noncopyable or nonescapable iterable and preserve a typed iteration failure. Sequence and Iterable are different protocols; supporting one does not imply conformance to the other.

**Example:** Standalone runnable example.

```swift
@available(anyAppleOS 27.0, *)
func visit<I: Iterable & ~Copyable & ~Escapable>(
    _ values: borrowing I, _ body: (borrowing I.Element) -> Void
) throws(I.Failure) where I.Element: ~Copyable {
    for try value in values { body(value) }
}
func checkVisit() {
    let values = [1, 2, 3]
    var sum = 0
    visit(values.span) { sum += $0 }
    precondition(sum == 6)
}
checkVisit()
```

**Walkthrough:** The Span supplies Iterable in this SDK; the visit body borrows each Int, and the concrete failure type is Never, so this call needs no try. [SE-0516](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md)

**Boundary:** The selected API requires OS 27. The array itself does not satisfy Iterable here. This is synchronous traversal, not indexing or work scheduling; [[C5 — Collection Protocols, Complexity, and Index Invalidation|C5]] contains the larger protocol matrix.

Prepend visit from that answer. This **intentionally rejected** call passes the Array itself rather than a conforming view:

```swift
func invalidVisit() { visit([1, 2, 3]) { _ in } }
```

## Focused exercise

**Task:** Repair appendAll using the source/destination element relationship, with no unnecessary source-container constraint.

**Acceptance checks:**

- Array and ArraySlice inputs append in traversal order.
- Empty input preserves the target.
- Explicit String elements reject for an Int target.

**Solution:** Standalone array-target helper. Its parameter directly expresses the required equality; the negative specialization below is compiled separately.

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

**Reasoning:** The destination is an array of S.Element, while S itself need only be a Sequence. A general RangeReplaceableCollection destination can use the earlier where-clause version. Both signatures establish a type relationship, not runtime equality or specialization.

Prepend only the focused exercise’s appendAll setup. This fixed mismatch is **intentionally rejected**:

```swift
let invalid: ([String], inout [Int]) -> Void = appendAll
```

## In practice

**Choose it when:** Generalize over only the operations and relationships the algorithm needs.

**Investigate by:** Write explicit input types and inspect capability, same-type, Copyable, and Escapable constraints separately.

**Strong evidence:** Explain accepted heterogeneous container types and rejected mismatched elements, then verify supported consumers instead of assuming generalized protocols automatically generalize every API.

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
