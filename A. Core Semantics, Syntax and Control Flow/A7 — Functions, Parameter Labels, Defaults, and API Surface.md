---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - functions
  - api-design
---

# A7 — Functions, Parameter Labels, Defaults, and API Surface

**Goal:** Replace ambiguous request arguments with a signature whose labels and default evaluation are testable.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A function signature describes argument roles, ownership, effects, and isolation. Labels help callers distinguish same-typed inputs; defaults express policy chosen when arguments are omitted. A function value preserves parameter and result types plus relevant effects, but its invocation does not carry declaration labels or omitted defaults. [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)

## Rules and pitfalls

### Design calls around domain roles

Use labels and small domain types when primitive arguments can be confused. A variadic parameter models a list of one role; a collection is better when forwarding an existing list. Multiple trailing closures should retain clear labels for distinct callbacks.

**Common pitfall:** Replacing every argument with an options bag can hide required inputs and changing policy. Keep essential domain data explicit and test what defaults actually forward.

### Treat defaults and mutation as contracts

An omitted default expression is evaluated at the call site for each call. Its referenced library helper may execute newer library code. Inout requests scoped exclusive access; it is not a stored reference or automatic concurrency protection.

**Common pitfall:** Assuming changing a literal default updates unchanged binaries overlooks client-emitted arguments. Compare an unchanged client with a rebuilt client; use a runtime policy accessor when that is the intended evolution contract.

## Interview questions

### Which parts of a declaration survive as a function value, and how do you select between equal-typed overloads with different labels?

Function values retain their typed inputs, result, ownership/effect and isolation contract as represented by their function type, but calls use positional arguments and supply every required parameter. To select equal-typed overloads with different labels, reference the full declaration name such as send(to:) instead of expecting a type annotation to disambiguate identical types.

### Why can an unchanged client retain a literal default but observe a changed named/helper-based default?

The original client can retain a compiled literal argument. If that compiled expression calls a library getter or helper, replacing the library can change the helper's result. “Defaults are at the caller” therefore does not imply every observable default is frozen. State which part is emitted and which part calls the library.

### How do default selection and virtual method dispatch interact?

Default selection follows the declaration known at the call site, while an overridden method body can dispatch dynamically. A base-typed call can use the base declaration's default and then execute the subclass override. Avoid different defaults on overrides when that split would surprise callers.

### Why does passing one evaluated value plus one inout differ from two mutable aliases, and what changes across suspension?

An evaluated input value is read before the inout access begins; two overlapping mutable accesses conflict. Async inout can work on an independent local, but passing actor-isolated storage across suspension can violate isolation or reentrancy requirements. Copy to a local and define a conflict-aware writeback policy when appropriate.

## Focused exercise

**Task:** Replace positional request parameters with labeled domain inputs. Check call-site clarity, forwarded values, and default evaluation.

**Acceptance checks:**

- Labels distinguish the endpoint from policy.
- Explicit arguments preserve the supplied object; each omitted default creates a fresh policy.
- The endpoint and policy values are forwarded without substitution.

**Solution:**

Standalone; this records arguments rather than performing I/O:

```swift
struct Endpoint { let path: String }
final class Policy { let retries = 2 }
struct Request { let endpoint: Endpoint; let policy: Policy }
func makeRequest(to endpoint: Endpoint, policy: Policy = Policy()) -> Request {
    Request(endpoint: endpoint, policy: policy)
}
let endpoint = Endpoint(path: "/users")
let supplied = Policy()
let explicit = makeRequest(to: endpoint, policy: supplied)
let first = makeRequest(to: endpoint)
let second = makeRequest(to: endpoint)
precondition(explicit.endpoint.path == "/users")
precondition(explicit.policy === supplied)
precondition(first.policy !== second.policy)
precondition(first.policy.retries == 2)
```

**Reasoning:** The endpoint and policy name distinct input roles. Object identity makes per-call evaluation observable without global mutable instrumentation: two omitted defaults create two objects, while the explicit policy is forwarded unchanged. This checks construction, not network behavior.

## In practice

**Choose it when:** Review representative call sites before expanding overloads or introducing options.

**Investigate by:** Compare the declaration used for default selection with the implementation used for dispatch.

**Strong evidence:** Test old and rebuilt consumers when evolving defaults and validate mutation boundaries across suspension.

### Ownership and isolated defaults — optional

#### Ownership is a parameter contract

```swift
struct Draft: ~Copyable {
    let text: String
}

func preview(_ draft: borrowing Draft) -> String { draft.text }
func finish(_ draft: consuming Draft) -> String { draft.text }

func draftExample() -> [String] {
    let draft = Draft(text: "Notes")
    let before = preview(draft)
    let after = finish(draft)
    return [before, after]
}
```

The borrowed preview leaves the draft available for final consumption. Reusing that noncopyable draft after finish is rejected. This is not the same as ~Sendable, and a consuming parameter on a copyable type does not generally prohibit the caller from retaining a copy. C7/C8 cover accessors, references, and the broader ownership model.

#### Inout is exclusive mutation, not a stored alias

```swift
func clamp(_ value: inout Int, to range: ClosedRange<Int>) {
    value = min(max(value, range.lowerBound), range.upperBound)
}

func add(_ value: inout Int, amount: Int) {
    value += amount
}
```

Clamping 14 to 0...10 writes back 10. `add(&value, amount: value)` is valid: the ordinary value argument is read before exclusive mutation starts. Two inout arguments to the same storage are different. This complete example is intentionally rejected:

```swift
func overwrite(_ first: inout Int, with second: inout Int) {
    first = second
}

func aliasedArguments() {
    var value = 1
    overwrite(&value, with: &value)
}
```

The compiler diagnoses aliasing/overlapping mutable accesses. This escaping capture is also rejected:

```swift
func capture(_ value: inout Int) -> () -> Void {
    { value += 1 }
}
```

The semantic writeback model also works with computed properties; no stable pointer or persistent reference is promised. Exclusivity does not synchronize arbitrary concurrent accesses. C3 retains the deeper static/dynamic access tests.

Short synchronous mutation is often the clearest API, but Swift does not categorically prohibit async inout functions:

```swift
func incrementAfterYield(_ value: inout Int) async {
    await Task.yield()
    value += 1
}

func localMutation() async -> Int {
    var value = 1
    await incrementAfterYield(&value)
    return value
}
```

The local example returns 2. Its exclusive access spans the call, including any suspension. Passing an actor's stored property directly to that async inout function is rejected; suspension may allow actor reentrancy. Prefer returning a result when that makes ownership and state transitions clearer.

#### Isolated defaults obey the callee's actor contract

```swift
@MainActor
final class UIContext {
    let value: Int
    init(value: Int = 7) { self.value = value }
}

@MainActor
func contextLabel(context: UIContext = UIContext()) -> String {
    "Context \(context.value)"
}

func loadContextLabel() async -> String {
    await contextLabel()
}
```

The default initializer is valid in the same MainActor domain as contextLabel. An outside caller awaits the call; a MainActor caller can call it synchronously. A nonisolated declaration cannot simply use that isolated initializer as its own synchronous default. Default evaluation itself is synchronous within the required isolation; `await` is not a way to put arbitrary asynchronous work into a default expression. [SE-0411](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0411-isolated-default-values.md)

## References

### Related guides

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A4 — Optionals and Nil Modeling]]
- [[A8 — Closures, Escaping, and Capture Semantics]]
- [[A13 — Error handling and typed throws]]
- [[B2 — Existentials (`any`) vs Opaque Types (`some`)]]
- [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[B12 — Variadic generics and parameter packs]]
- [[B14 — Result Builders and DSL Construction]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]

### Official sources

- [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [SE-0411](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0411-isolated-default-values.md)
- [The Swift Programming Language: Functions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/functions/)
- [SE-0279: Multiple trailing closures](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0279-multiple-trailing-closures.md)
- [SE-0286: Forward matching of trailing closures](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0286-forward-scan-trailing-closures.md)
- [SE-0508: Array-expression trailing closures](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0508-array-expression-trailing-closures.md)
- [SE-0521: Improved optional opaque/existential syntax](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0521-improved-optional-opaque-and-any.md)
