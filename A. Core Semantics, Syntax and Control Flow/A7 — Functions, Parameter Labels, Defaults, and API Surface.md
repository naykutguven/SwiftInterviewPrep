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

**Learning goals:**

- Design readable calls with labels, domain types, defaults, collections, and trailing closures.
- Distinguish declaration labels/default selection from function values and virtual dispatch.
- Explain inout exclusivity, borrowing/consuming parameters, effects, and isolated defaults.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]/[[A2 — `let`, `var`, `mutating`, and method semantics]] for values and mutation. [[A8 — Closures, Escaping, and Capture Semantics]] explains closures; the extensions use [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]] ownership and [[D4 — Actors and Actor Isolation]]/[[D5 — Global Actors and `@MainActor`]] isolation.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A call such as request("/users", 2, true) makes the reader reconstruct three roles. Labels and domain types put those roles in the API: request(to: endpoint, policy: policy). A default is policy applied when an argument is omitted, not a permanently frozen property of the operation.

A declaration also describes effects (whether a call can suspend or throw), ownership, and isolation. Its function value retains a function-type contract, but invocation uses positional arguments and does not retain omitted defaults. Review both the declaration and realistic calls when designing an API. [API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### Labels identify roles; collections forward a list

A label appears at the call site; the local parameter name is used inside the body. Variadics accept individual arguments, while a collection parameter accepts an existing list.

```swift
struct Endpoint { let path: String }
func request(to endpoint: Endpoint, retryCount: Int = 2) -> String {
    "\(endpoint.path):\(retryCount)"
}
func total(_ values: Int...) -> Int { values.reduce(0, +) }
func total(values: [Int]) -> Int { values.reduce(0, +) }
precondition(request(to: Endpoint(path: "/users")) == "/users:2")
precondition(total(1, 2, 3) == 6)
precondition(total(values: [1, 2, 3]) == 6)
```

**Walkthrough:** to describes the argument’s role while endpoint names it locally. The two total overloads distinguish variadic call syntax from forwarding an array; an array does not automatically expand into variadic arguments.

### Name distinct trailing callbacks

Multiple trailing closures omit the first closure’s label and retain subsequent labels. Matching follows parameter order; adding overloads or defaults can change resolution, so keep representative calls in review.

```swift
enum Failure: Error { case offline }
func fetch(online: Bool, success: (String) -> Void, failure: (Failure) -> Void) {
    if online { success("data") } else { failure(.offline) }
}
var result = ""
fetch(online: true) { value in
    result = value
} failure: { _ in
    result = "failed"
}
precondition(result == "data")
fetch(online: false, success: { result = $0 }, failure: { _ in result = "failed" })
precondition(result == "failed")
```

**Walkthrough:** The trailing form and labeled parenthesized form express the same argument roles. The callee invokes synchronously here; closure syntax does not imply a task or later delivery. [SE-0279](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0279-multiple-trailing-closures.md), [SE-0286](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0286-forward-scan-trailing-closures.md)

## Rules and pitfalls

### An options bag can conceal required input

**Common pitfall:** Making every field optional/defaulted so a request can be constructed without an endpoint.

**Mistake:** Standalone runnable example.

```swift
struct Options {
    var path: String? = nil
    var retries = 2
}
func request(_ options: Options = Options()) -> String { options.path ?? "" }
precondition(request() == "")  // An invalid request silently exists.
```

**Consequence:** The signature permits a request with no destination and silently substitutes empty text.

**Why:** The options type made essential domain data optional instead of merely grouping independent policy choices.

**Correction:** Keep required inputs explicit; use an options type for policy when it earns its complexity.

```swift
struct Endpoint { let path: String }
struct Policy { var retries = 2 }
struct Request {
    let endpoint: Endpoint
    let policy: Policy
}
func request(to endpoint: Endpoint, policy: Policy = Policy()) -> Request {
    Request(endpoint: endpoint, policy: policy)
}
let built = request(to: Endpoint(path: "/users"))
precondition(built.endpoint.path == "/users" && built.policy.retries == 2)
```

**What changes:** Callers must supply a destination. Endpoint validation, policy evolution, and ABI compatibility remain separate design work; the options struct solves none of those automatically.

### A function value does not inherit defaults

**Common pitfall:** Calling an extracted function while omitting an argument that has a declaration default.

**Mistake:** Compile-only; intentionally rejected.

```swift
func retries(_ count: Int = 2) -> Int { count }
let operation = retries
let result = operation()
```

**Consequence:** The compiler requires an Int argument to the function value.

**Why:** The default belongs to calls resolved against the declaration. The extracted value has type (Int) -> Int.

**Correction:** Supply the argument or deliberately wrap the declaration call in a zero-argument closure.

```swift
func retries(_ count: Int = 2) -> Int { count }
let operation = retries
let usingDefault = { retries() }
precondition(operation(4) == 4)
precondition(usingDefault() == 2)
```

**What changes:** The wrapper supplies a new API boundary that chooses when to evaluate the declaration default.

## Worked example

### Default selection and method dispatch are separate

**Setup:** Standalone script. The variable base has static type Base but refers to a Derived instance.

**Predict:** Which declaration supplies each default, and which body executes?

```swift
class Base { func pick(_ value: Int = 1) -> String { "base:\(value)" } }
final class Derived: Base {
    override func pick(_ value: Int = 2) -> String { "derived:\(value)" }
}
let derived = Derived()
let base: Base = derived
let function = base.pick
precondition(
    [derived.pick(), base.pick(), function(3)] == ["derived:2", "derived:1", "derived:3"])
```

**Outcome:** derived:2, derived:1, derived:3.

**Why:** Static declaration lookup chooses 2 or 1 for omitted arguments. Virtual dispatch selects the Derived body for both references. The function value requires an explicit argument.

## Interview questions

### Which declaration parts survive as a function value, and how do labels select equal-typed overloads?

**Answer:** Function values retain typed parameters/results and relevant effects/isolation, but calls use positions and supply the arguments. Select equal-typed overloads by the full declaration name, not just a type annotation.

**Example:** Standalone runnable example.

```swift
func send(to destination: String) -> String { "to:\(destination)" }
func send(from source: String) -> String { "from:\(source)" }
let outbound: (String) -> String = send(to:)
let inbound: (String) -> String = send(from:)
precondition(outbound("Ada") == "to:Ada")
precondition(inbound("Ben") == "from:Ben")
func read(_ value: Int) async throws -> Int { value }
let operation: (Int) async throws -> Int = read
let value = try await operation(7)
precondition(value == 7)
```

**Walkthrough:** Both send declarations have the same function type; to: and from: identify which declaration to extract. The async/throws function value still needs await and try at invocation.

**Boundary:** Function conversions also obey actor isolation, Sendable, and ownership rules. Do not erase those contracts merely because labels disappear.

### Why can an unchanged client retain a literal default but observe a changed named/helper default?

**Answer:** The client emits the omitted argument expression. An emitted literal can stay fixed, while an emitted call to a library getter or helper executes the replacement library’s implementation.

**Example:** Compile-only library source named Defaults.swift. Build module Defaults with `-parse-as-library -module-name Defaults -emit-module -emit-module-path /tmp/Defaults.swiftmodule -c -o /tmp/Defaults.o`. The replacement experiment instead emits a dynamic library from the same source.

```swift
public var currentRetries: Int { 2 }
public func retryPolicy() -> Int { 2 }
public func literal(_ count: Int = 2) -> Int { count }
public func named(_ count: Int = currentRetries) -> Int { count }
public func helper(_ count: Int = retryPolicy()) -> Int { count }
```

**Walkthrough:** In a v1 library, every default resolves to 2. In a v2 build, change all three 2 expressions to 5. Replacing only the library yields 2,5,5 for the unchanged client below; rebuilding the client yields 5,5,5. The validation performs both replacements and records actual outputs.

**Boundary:** This controlled library preserves its public signatures and uses library evolution for the replacement experiment. A general library upgrade can have other ABI constraints; changing a default is also a policy change for rebuilt consumers.

Separate runnable client. For the simple object-linked v1 run, use `-I /tmp /tmp/Defaults.o`. For replacement, link `-L /tmp -lDefaults` to a dynamic library with a stable install name/rpath. Keep the original executable for the second run; rebuild it only for the third.

```swift
import Defaults

let values = [literal(), named(), helper()]
print(values.map(String.init).joined(separator: ","))
precondition(values[1] == values[2])
```

### How do default selection and virtual method dispatch interact?

**Answer:** The statically known declaration selects omitted defaults, while the method body can dispatch to an override. Different defaults on an override can therefore surprise callers of the same instance.

**Example:** Standalone runnable example.

```swift
class Base { func pick(_ value: Int = 1) -> String { "base:\(value)" } }
final class Derived: Base {
    override func pick(_ value: Int = 2) -> String { "derived:\(value)" }
}
let instance = Derived()
let asBase: Base = instance
precondition(instance.pick() == "derived:2")
precondition(asBase.pick() == "derived:1")
precondition(asBase.pick(9) == "derived:9")
```

**Walkthrough:** Both calls dispatch to Derived. The omitted argument differs because one call resolves its default against Base.

**Boundary:** Prefer one explicit policy source when callers should see consistent defaults across static types; that can be a non-defaulted wrapper or a runtime policy accessor.

### Why does one evaluated value plus inout differ from two mutable aliases, and what changes across suspension?

**Answer:** Ordinary argument values are read before exclusive inout access begins. Two simultaneous inout accesses to the same storage overlap. Async inout can operate on an independent local, but direct actor-storage access across suspension is restricted.

**Example:** Standalone runnable example.

```swift
func add(_ value: inout Int, amount: Int) { value += amount }
func incrementAfterYield(_ value: inout Int) async {
    await Task.yield()
    value += 1
}
func localMutation() async -> Int {
    var value = 2
    add(&value, amount: value)
    await incrementAfterYield(&value)
    return value
}
let result = await localMutation()
precondition(result == 5)
```

**Walkthrough:** amount is evaluated as 2 before add mutates value to 4. The async call holds exclusive access to its independent local and returns 5.

**Boundary:** inout is scoped access with writeback semantics, including computed properties; it is not a persistent reference or synchronization. Copying actor state into a local also needs a conflict-aware policy before writing it back.

Separate **compile-only, intentionally rejected** examples: overlapping mutable arguments, escaping inout capture, and actor storage passed to the async function from that answer.

```swift
func overwrite(_ first: inout Int, with second: inout Int) { first = second }
func invalid() {
    var value = 1
    overwrite(&value, with: &value)
}
```

```swift
func capture(_ value: inout Int) -> () -> Void { { value += 1 } }
```

```swift
actor Counter {
    var value = 0
    func invalid() async { await incrementAfterYield(&value) }
}
```

### Extension: how do borrowing, consuming, and isolated defaults change the call contract?

**Answer:** borrowing permits inspection without transferring a noncopyable value; consuming transfers it. An actor-isolated default is evaluated under the callee’s required isolation; an outside caller must make the appropriate actor hop.

**Example:** Standalone runnable example.

```swift
struct Draft: ~Copyable { let text: String }
func preview(_ draft: borrowing Draft) -> String { draft.text }
func finish(_ draft: consuming Draft) -> String { draft.text }
func useDraft() -> [String] {
    let draft = Draft(text: "Notes")
    let before = preview(draft)
    let after = finish(draft)
    return [before, after]
}
@MainActor final class UIContext { let value = 7 }
@MainActor func contextLabel(context: UIContext = UIContext()) -> String {
    "Context \(context.value)"
}
func loadLabel() async -> String { await contextLabel() }
precondition(useDraft() == ["Notes", "Notes"])
let label = await loadLabel()
precondition(label == "Context 7")
```

**Walkthrough:** preview leaves the draft available to finish. loadLabel awaits the MainActor call; the default initializer belongs to that same domain. [SE-0411](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0411-isolated-default-values.md)

**Boundary:** A consumed noncopyable draft cannot then be reused. A consuming parameter on a copyable type does not prohibit other copies. Default expressions are synchronous; arbitrary async work does not belong in them.

## Focused exercise

**Task:** Replace positional request parameters with labeled domain inputs and verify default evaluation.

**Acceptance checks:**

- The endpoint and policy have distinct roles.
- Explicit policy is forwarded by identity; each omitted default constructs a fresh policy.
- Endpoint and policy values are not substituted.

**Solution:** Standalone script; it constructs requests without I/O.

```swift
struct Endpoint { let path: String }
final class Policy { let retries = 2 }
struct Request {
    let endpoint: Endpoint
    let policy: Policy
}
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
precondition(first.policy.retries == 2 && second.policy.retries == 2)
```

**Reasoning:** Object identity makes per-call default evaluation observable without a global counter. The default creates a fresh Policy only for omitted arguments; the explicit path forwards the supplied object.

## In practice

**Choose it when:** Review representative calls before adding overloads or introducing an options type.

**Investigate by:** Separate the declaration used for defaults from the implementation used for dispatch, and trace when argument expressions run.

**Strong evidence:** Compare unchanged and rebuilt clients when evolving defaults; test mutation and isolation boundaries independently.

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
