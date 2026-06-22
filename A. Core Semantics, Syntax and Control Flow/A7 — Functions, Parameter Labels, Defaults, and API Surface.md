---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - functions
  - api-design
---
## 0. Rubric snapshot

**Rubric expectation**

Understand argument labels, default arguments, variadics, `inout`, function types, and how function signatures contribute to API clarity. The rubric explicitly calls out two caveats: default arguments are baked at call sites, and `inout` is not plain pass-by-reference.

**Caveats**

- Default arguments are not just convenience syntax; they affect source and binary compatibility.
- `inout` gives a function temporary exclusive mutable access to a value. It is not a C pointer, C++ reference, or stored alias.
- Parameter labels are part of API design. A technically valid function can still be a bad Swift API.
- Function types erase argument labels; the labels matter at declaration and call sites, not as part of the function value type.

**You should be able to answer**

- Why are argument labels part of Swift API design rather than syntax decoration?
- What are the semantic pitfalls of changing a default parameter value in a library?
- Why is `inout` not plain pass-by-reference?

**You should be able to do**

- Redesign an overly positional API into a Swifty signature with clear labels and defaults.
- Explain how labels, defaults, variadics, `inout`, and function types affect long-term API surface.

---

## 1. Core mental model

A Swift function signature is both a **type contract** and a **human-facing API surface**.

The type contract says:

```swift
(Input types) async throws -> Output
```

But the API surface also says:

```swift
client.data(for: endpoint, cachePolicy: .reloadIgnoringLocalCacheData)
```

That second part matters. Swift is designed around call-site clarity. A function call should usually read like a small phrase. Argument labels are how Swift encodes the role of arguments, especially when the argument type is weakly informative, such as `Bool`, `Int`, `String`, `Data`, or `URL`. The official API Design Guidelines explicitly prioritize clarity over brevity and recommend labels that describe argument roles rather than merely repeating type information. ([Swift.org](https://swift.org/documentation/api-design-guidelines/?utm_source=chatgpt.com "API Design Guidelines | Swift.org"))

The key idea:

```text
A Swift function signature is a public sentence: labels explain roles, defaults express policy, and inout marks exclusive mutation.
```

Default arguments are not overloads, although they can feel like overloads at the call site. They let one declaration support progressive disclosure: required semantic inputs first, less common customization later. The API Design Guidelines specifically prefer a single method with defaults over a family of nearly identical overloads, and recommend putting parameters with defaults toward the end. ([Swift.org](https://swift.org/documentation/api-design-guidelines/?utm_source=chatgpt.com "API Design Guidelines | Swift.org"))

`inout` is Swift’s controlled mutation escape hatch. The caller must write `&`, which makes mutation visible at the call site. During the call, Swift requires exclusive access to the modified storage; overlapping access is rejected or trapped because exclusivity is part of Swift’s memory-safety model. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety/?utm_source=chatgpt.com "Memory Safety"))

---

## 2. Essential mechanics

### Argument labels vs parameter names

Each function parameter can have two names:

```swift
func move(from source: URL, to destination: URL) {
    print("Move \(source) to \(destination)")
}

move(from: oldURL, to: newURL)
```

- `from` and `to` are **argument labels** used at the call site.
    
- `source` and `destination` are **parameter names** used inside the function body.
    
- By default, Swift uses the parameter name as the argument label. The Swift book describes this as the standard model for function parameters. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/functions/?utm_source=chatgpt.com "Functions | Documentation - Swift Programming Language"))
    

Use `_` only when the label adds no useful information:

```swift
func max(_ lhs: Int, _ rhs: Int) -> Int {
    lhs > rhs ? lhs : rhs
}

max(10, 20)
```

Bad label omission:

```swift
func resize(_ width: Double, _ height: Double, _ animated: Bool) {}

resize(320, 240, true)
```

Better:

```swift
func resize(to size: CGSize, animated: Bool = true) {}

resize(to: CGSize(width: 320, height: 240))
resize(to: CGSize(width: 320, height: 240), animated: false)
```

The second version makes the semantic roles visible.

---

### Default arguments

Default arguments let callers omit parameters when the default behavior is correct:

```swift
struct RetryPolicy {
    static let none = RetryPolicy()
    static func exponential(maxAttempts: Int) -> RetryPolicy {
        RetryPolicy()
    }
}

func loadImage(
    from url: URL,
    cachePolicy: URLRequest.CachePolicy = .useProtocolCachePolicy,
    retryPolicy: RetryPolicy = .none,
    timeout: Duration = .seconds(10)
) async throws -> Data {
    Data()
}
```

Callers can start simple:

```swift
let data = try await loadImage(from: url)
```

And opt into detail:

```swift
let data = try await loadImage(
    from: url,
    cachePolicy: .reloadIgnoringLocalCacheData,
    retryPolicy: .exponential(maxAttempts: 3),
    timeout: .seconds(5)
)
```

Good defaults should represent stable, unsurprising behavior. They should not hide major policy decisions that product, security, privacy, or performance owners may want to audit.

A serious library caveat: default argument expressions are emitted into the caller according to Swift ABI discussions, so changing a default can affect newly compiled clients differently from already compiled clients. ([Swift Forums](https://forums.swift.org/t/swift-abi-stability-manifesto/4987 "Swift ABI Stability Manifesto - Discussion - Swift Forums")) In practice: changing a public default is a semantic API change, even if the function declaration still compiles.

---

### Variadic parameters

Variadics are useful when the API naturally accepts zero or more values of the same conceptual kind:

```swift
func log(_ items: Any..., separator: String = " ") {
    let message = items.map(String.init(describing:)).joined(separator: separator)
    print(message)
}

log("User", 42, "logged in")
log("User", 42, "logged in", separator: " | ")
```

Use variadics when:

```text
The values are homogeneous in role, not necessarily in type.
```

Avoid variadics when each argument has a different semantic role:

```swift
// Bad
func track(_ name: String, _ id: String, _ source: String, _ timestamp: Date) {}
```

Better:

```swift
struct AnalyticsEvent {
    var name: String
    var userID: String
    var source: String
    var timestamp: Date = .now
}

func track(_ event: AnalyticsEvent) {}
```

Variadics are not a substitute for a model type.

---

### `inout`

`inout` allows a function to mutate caller-owned storage:

```swift
func clamp(_ value: inout Double, to range: ClosedRange<Double>) {
    value = min(max(value, range.lowerBound), range.upperBound)
}

var alpha = 1.4
clamp(&alpha, to: 0...1)

print(alpha)
```

Output:

```text
1.0
```

The `&` is important. It makes mutation explicit at the call site.

But `inout` is not “just a reference.” Swift’s semantic model is closer to:

```text
copy value in → mutate through exclusive access → write value back
```

The compiler can optimize this, but the exclusivity rule remains.

This is illegal:

```swift
func overwrite(_ first: inout Int, with second: inout Int) {
    first = second
}

var value = 1
overwrite(&value, with: &value)
```

Compiler error in Swift 6.2.1:

```text
/tmp/a7c.swift:6:25: error: inout arguments are not allowed to alias each other
4 | 
5 | var value = 1
6 | overwrite(&value, with: &value)
  |           |             `- error: inout arguments are not allowed to alias each other
  |           `- note: previous aliasing argument
7 | 

/tmp/a7c.swift:6:11: error: overlapping accesses to 'value', but modification requires exclusive access; consider copying to a local variable
4 | 
5 | var value = 1
6 | overwrite(&value, with: &value)
  |           |             `- note: conflicting access is here
  |           `- error: overlapping accesses to 'value', but modification requires exclusive access; consider copying to a local variable
7 | 
```

This is the same family of rules covered more deeply in [[C3 — Exclusivity and inout]].

---

### Function types

A function declaration can have labels:

```swift
func move(from source: Int, to destination: Int) {}
```

But a function type does not keep those labels:

```swift
let operation: (Int, Int) -> Void = move
operation(1, 2)
```

This fails:

```swift
func move(from source: Int, to destination: Int) {}

let f: (from: Int, to: Int) -> Void = move
f(1, 2)
```

Compiler error in Swift 6.2.1:

```text
/tmp/fntype.swift:2:9: error: function types cannot have argument labels; use '_' before 'from'
1 | func move(from source: Int, to destination: Int) {}
2 | let f: (from: Int, to: Int) -> Void = move
  |         `- error: function types cannot have argument labels; use '_' before 'from'
3 | f(1, 2)
4 | 

/tmp/fntype.swift:2:20: error: function types cannot have argument labels; use '_' before 'to'
1 | func move(from source: Int, to destination: Int) {}
2 | let f: (from: Int, to: Int) -> Void = move
  |                    `- error: function types cannot have argument labels; use '_' before 'to'
3 | f(1, 2)
4 | 
```

Defaults also do not become part of the function value type:

```swift
func greet(_ name: String = "Guest") {
    print(name)
}

let g = greet
g()
```

Compiler error in Swift 6.2.1:

```text
/tmp/default_fnvalue.swift:6:3: error: missing argument for parameter #1 in call
3 | }
4 | 
5 | let g = greet
  |     `- note: 'g' declared here
6 | g()
  |   `- error: missing argument for parameter #1 in call
7 | 
```

The declaration `greet(_:)` has a default at direct call sites, but the function value `g` has type `(String) -> Void`.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking labels are cosmetic

Bad:

```swift
func schedule(_ date: Date, _ repeat: Bool, _ alert: Bool) {}

schedule(.now, true, false)
```

This call is unreadable. The booleans are especially bad because `true` and `false` do not explain their roles.

Better:

```swift
func schedule(
    at date: Date,
    repeats: Bool = false,
    sendsAlert: Bool = true
) {}

schedule(at: .now)
schedule(at: .now, repeats: true, sendsAlert: false)
```

The labels turn primitive values into a readable sentence.

---

### Trap 2: Treating default arguments as harmless library details

Bad:

```swift
// v1
public func fetchProducts(limit: Int = 20) async throws -> [Product]

// v2
public func fetchProducts(limit: Int = 100) async throws -> [Product]
```

This looks source-compatible, but it changes behavior for clients that recompile and may not affect already compiled clients in the same way. That can create split behavior across apps, app extensions, binary frameworks, or cached build artifacts.

Better:

```swift
public struct ProductFetchOptions: Sendable {
    public var limit: Int
    public var includesOutOfStock: Bool

    public static let standard = ProductFetchOptions(
        limit: 20,
        includesOutOfStock: false
    )

    public static let expanded = ProductFetchOptions(
        limit: 100,
        includesOutOfStock: false
    )
}

public func fetchProducts(
    options: ProductFetchOptions = .standard
) async throws -> [Product]
```

This makes policy named and reviewable.

---

### Trap 3: Treating `inout` like a stored reference

Bad:

```swift
func capture(_ value: inout Int) -> () -> Void {
    return {
        value += 1
    }
}
```

Compiler error in Swift 6.2.1:

```text
/tmp/inout_capture.swift:2:12: error: escaping closure captures 'inout' parameter 'value'
1 | func capture(_ value: inout Int) -> () -> Void {
  |                `- note: parameter 'value' is declared 'inout'
2 |     return {
  |            `- error: escaping closure captures 'inout' parameter 'value'
3 |         value += 1
  |         `- note: captured here
4 |     }
5 | }
```

Better:

```swift
final class Counter {
    private(set) var value: Int

    init(value: Int) {
        self.value = value
    }

    func increment() {
        value += 1
    }
}
```

Or, if value semantics are preferred:

```swift
struct Counter {
    var value: Int

    mutating func increment() {
        value += 1
    }
}
```

Use `inout` for short, synchronous mutation. Do not use it to fake long-lived shared mutable state.

---

### Trap 4: Overloading instead of using defaults

Bad:

```swift
func search(_ query: String) async throws -> [Result]
func search(_ query: String, limit: Int) async throws -> [Result]
func search(_ query: String, limit: Int, includesArchived: Bool) async throws -> [Result]
```

Better:

```swift
func search(
    for query: String,
    limit: Int = 20,
    includesArchived: Bool = false
) async throws -> [SearchResult]
```

This gives callers progressive disclosure without forcing them to choose among a method family.

But if the options grow or become policy-heavy, move them into a type:

```swift
struct SearchOptions: Sendable {
    var limit: Int = 20
    var includesArchived: Bool = false
    var ranking: Ranking = .relevance
}

func search(
    for query: String,
    options: SearchOptions = SearchOptions()
) async throws -> [SearchResult]
```

---

## 4. Direct answers to rubric questions

### Q1. Why are argument labels part of Swift API design rather than syntax decoration?

Argument labels are part of the meaning of a Swift call. They describe the role of each argument at the use site, disambiguate weakly typed values, and make the call read naturally. The API Design Guidelines explicitly frame clarity at the point of use as more important than brevity. ([Swift.org](https://swift.org/documentation/api-design-guidelines/?utm_source=chatgpt.com "API Design Guidelines | Swift.org"))

Labels are especially important for:

```text
Bool
Int
String
Data
URL
Date
closures
multiple values of the same type
parameters with default values
```

Compare:

```swift
download(url, true, 3, 10)
```

With:

```swift
download(
    from: url,
    allowsCellularAccess: true,
    retryCount: 3,
    timeout: .seconds(10)
)
```

The second call is self-documenting.

Interview version:

> Argument labels are part of Swift’s API design because Swift optimizes for clarity at the call site. Labels encode the role and relationship of arguments, especially when types alone do not communicate meaning. I omit labels only when the argument is naturally part of the base-name phrase or when the arguments cannot be usefully distinguished, like `min(1, 2)`. Otherwise, labels are semantic documentation, not decoration.

---

### Q2. What are the semantic pitfalls of changing a default parameter value in a library?

Changing a default parameter value can silently change behavior for clients that recompile, while already compiled clients may continue using the old default because default argument expressions are emitted into the caller. ([Swift Forums](https://forums.swift.org/t/swift-abi-stability-manifesto/4987 "Swift ABI Stability Manifesto - Discussion - Swift Forums"))

That means this can be dangerous:

```swift
// Old
public func sync(batchSize: Int = 50)

// New
public func sync(batchSize: Int = 500)
```

The signature still looks compatible, but the behavior changed. In production, that could affect memory usage, backend load, battery usage, pagination behavior, or rate limits.

Safer options:

```swift
public enum SyncMode: Sendable {
    case standard
    case highThroughput
}

public func sync(mode: SyncMode = .standard)
```

Or:

```swift
public struct SyncOptions: Sendable {
    public static let standard = SyncOptions(batchSize: 50)
    public static let highThroughput = SyncOptions(batchSize: 500)

    public var batchSize: Int
}

public func sync(options: SyncOptions = .standard)
```

Interview version:

> I treat public defaults as part of the API’s semantic contract. Changing a default may be source-compatible, but it can still change behavior for recompiled clients and may not affect old binaries the same way. For stable libraries, I avoid using defaults for policy that is likely to change. I prefer named options, configuration types, or new API names when changing behavior matters.

---

### Q3. Why is `inout` not plain pass-by-reference?

`inout` gives a function temporary exclusive mutable access to caller-owned storage. The caller must mark the argument with `&`, and Swift enforces exclusivity so the same storage is not simultaneously read or modified in conflicting ways. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety/?utm_source=chatgpt.com "Memory Safety"))

This is valid:

```swift
func reset(_ value: inout Int) {
    value = 0
}

var count = 10
reset(&count)
```

This is not valid:

```swift
func combine(_ lhs: inout Int, _ rhs: inout Int) {
    lhs += rhs
}

var value = 1
combine(&value, &value)
```

The problem is aliasing. The function would have two mutable `inout` accesses to the same storage.

Interview version:

> `inout` is Swift’s explicit short-lived mutation mechanism. Semantically, it behaves like exclusive access to a variable for the duration of the call, not like a stored reference. The compiler may optimize it, but the language rule is that overlapping access is forbidden. That is why `&` is required at the call site and why you cannot let an `inout` parameter escape.

---

## 5. Code probe / focused examples

A7 has no rubric code probe, so use these focused probes instead.

### Probe 1: Labels are not part of function types

Given:

```swift
func move(from source: Int, to destination: Int) {}

let f: (from: Int, to: Int) -> Void = move
f(1, 2)
```

What happens?

```text
/tmp/fntype.swift:2:9: error: function types cannot have argument labels; use '_' before 'from'
1 | func move(from source: Int, to destination: Int) {}
2 | let f: (from: Int, to: Int) -> Void = move
  |         `- error: function types cannot have argument labels; use '_' before 'from'
3 | f(1, 2)
4 | 

/tmp/fntype.swift:2:20: error: function types cannot have argument labels; use '_' before 'to'
1 | func move(from source: Int, to destination: Int) {}
2 | let f: (from: Int, to: Int) -> Void = move
  |                    `- error: function types cannot have argument labels; use '_' before 'to'
3 | f(1, 2)
4 | 
```

Why?

```text
Declaration name: move(from:to:)
Function value type: (Int, Int) -> Void
```

Labels guide declaration lookup and call-site clarity. Once the function is used as a value, the function type contains parameter types and result type, not labels.

Fix:

```swift
func move(from source: Int, to destination: Int) {}

let f: (Int, Int) -> Void = move
f(1, 2)
```

---

### Probe 2: Defaults are not part of function value calls

Given:

```swift
func greet(_ name: String = "Guest") {
    print(name)
}

let g = greet
g()
```

What happens?

```text
/tmp/default_fnvalue.swift:6:3: error: missing argument for parameter #1 in call
3 | }
4 | 
5 | let g = greet
  |     `- note: 'g' declared here
6 | g()
  |   `- error: missing argument for parameter #1 in call
7 | 
```

Why?

The direct declaration call supports default argument insertion:

```swift
greet()
```

But the function value has type:

```swift
(String) -> Void
```

So calling `g()` is missing the required `String`.

Fix:

```swift
let g: (String) -> Void = greet
g("Aykut")
```

Or wrap the default yourself:

```swift
let greetGuest: () -> Void = {
    greet()
}

greetGuest()
```

---

### Probe 3: `inout` requires exclusive access

Given:

```swift
func overwrite(_ first: inout Int, with second: inout Int) {
    first = second
}

var value = 1
overwrite(&value, with: &value)
```

What happens?

```text
/tmp/a7c.swift:6:25: error: inout arguments are not allowed to alias each other
4 | 
5 | var value = 1
6 | overwrite(&value, with: &value)
  |           |             `- error: inout arguments are not allowed to alias each other
  |           `- note: previous aliasing argument
7 | 

/tmp/a7c.swift:6:11: error: overlapping accesses to 'value', but modification requires exclusive access; consider copying to a local variable
4 | 
5 | var value = 1
6 | overwrite(&value, with: &value)
  |           |             `- note: conflicting access is here
  |           `- error: overlapping accesses to 'value', but modification requires exclusive access; consider copying to a local variable
7 | 
```

Why?

Both `&value` arguments request mutable access to the same storage for the duration of the same call.

Correct redesign:

```swift
func overwritten(_ first: Int, with second: Int) -> Int {
    second
}

var value = 1
value = overwritten(value, with: value)
```

Or, if only one mutation is needed:

```swift
func overwrite(_ first: inout Int, with second: Int) {
    first = second
}

var value = 1
overwrite(&value, with: value)
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Return a new value|Prefer value semantics and simple transformations|Requires assignment at call site|
|Use one `inout` plus plain values|Mutate one target using immutable inputs|Still requires exclusivity for the target|
|Use a reference type|Shared identity is genuinely required|Introduces aliasing, ARC, and thread-safety concerns|
|Use an actor|Shared mutable state crosses concurrency domains|Async access and actor reentrancy must be designed carefully|

---

## 6. Exercise

### Problem

Redesign an overly positional API into a Swifty signature with clear labels and defaults.

### Bad / naive version

```swift
func request(
    _ url: URL,
    _ method: String,
    _ body: Data?,
    _ timeout: TimeInterval,
    _ retries: Int,
    _ authenticated: Bool,
    _ cache: Bool
) async throws -> Data {
    Data()
}
```

Call site:

```swift
let data = try await request(
    url,
    "POST",
    body,
    30,
    3,
    true,
    false
)
```

### What is wrong?

```text
- Too many positional arguments.
- Stringly typed HTTP method.
- Boolean parameters hide meaning.
- TimeInterval loses unit clarity.
- Optional body is valid only for some methods, but the API does not express that.
- Retry, auth, cache, and timeout are policy knobs hidden as primitives.
- The call site is unreadable without jumping to the declaration.
```

### Improved version

```swift
enum HTTPMethod: Sendable {
    case get
    case post(Data)
    case put(Data)
    case delete
}

enum Authentication: Sendable {
    case none
    case bearerToken(String)
}

struct RetryPolicy: Sendable {
    var maxAttempts: Int

    static let none = RetryPolicy(maxAttempts: 1)

    static func exponential(maxAttempts: Int) -> RetryPolicy {
        RetryPolicy(maxAttempts: maxAttempts)
    }
}

struct RequestOptions: Sendable {
    var timeout: Duration
    var retryPolicy: RetryPolicy
    var cachePolicy: URLRequest.CachePolicy
    var authentication: Authentication

    static let standard = RequestOptions(
        timeout: .seconds(30),
        retryPolicy: .none,
        cachePolicy: .useProtocolCachePolicy,
        authentication: .none
    )
}

func data(
    for url: URL,
    method: HTTPMethod = .get,
    options: RequestOptions = .standard
) async throws -> Data {
    Data()
}
```

Call site:

```swift
let options = RequestOptions(
    timeout: .seconds(30),
    retryPolicy: .exponential(maxAttempts: 3),
    cachePolicy: .reloadIgnoringLocalCacheData,
    authentication: .bearerToken(token)
)

let data = try await data(
    for: url,
    method: .post(body),
    options: options
)
```

### Why this is better

- `data(for:)` reads naturally.
- `method: .post(body)` couples the body with methods that need a body.
- `Duration` is clearer than raw `TimeInterval`.
- `Authentication` replaces an unclear `Bool`.
- `RequestOptions.standard` gives a named default policy.
- The options type can evolve more safely than a long parameter list.
- The call site communicates intent without inspecting the implementation.

For an app-internal API, the simpler version may be acceptable:

```swift
func data(
    for url: URL,
    method: HTTPMethod = .get,
    timeout: Duration = .seconds(30),
    retryPolicy: RetryPolicy = .none,
    cachePolicy: URLRequest.CachePolicy = .useProtocolCachePolicy,
    authentication: Authentication = .none
) async throws -> Data {
    Data()
}
```

For a public SDK or shared module, prefer the `RequestOptions` version once the parameter list becomes policy-heavy.

---

## 7. Production guidance

Use clear labels when:

```text
- Multiple parameters have the same type.
- Parameters are primitive: Bool, Int, String, Data, URL, Date.
- The parameter expresses a role, policy, direction, source, destination, or condition.
- The call should read naturally without opening the declaration.
```

Use defaults when:

```text
- The default is stable and unsurprising.
- The parameter is less central than the required inputs.
- You want progressive disclosure instead of overload sprawl.
- The default does not hide risky behavior.
```

Use variadics when:

```text
- The arguments are conceptually the same kind of thing.
- The API is naturally list-like.
- A collection parameter would make simple call sites unnecessarily noisy.
```

Use `inout` when:

```text
- The function performs a short, synchronous mutation.
- Mutation should be explicit at the call site.
- Returning a new value would be awkward or less efficient.
- The API has a clear single mutation target.
```

Be careful when:

```text
- Changing public default values.
- Adding new defaulted parameters to public APIs.
- Using Bool defaults that hide important behavior.
- Designing APIs with multiple closure parameters.
- Passing functions as values and expecting labels/defaults to remain available.
- Using inout with collections, computed properties, subscripts, or overlapping storage.
```

Avoid when:

```text
- Positional arguments require the reader to memorize meaning.
- Defaults encode product policy likely to change.
- Variadics are used as an untyped configuration bag.
- inout is used to simulate long-lived shared mutable state.
- An API has many primitive parameters that should be modeled as enums or structs.
```

Debugging checklist:

```text
Can the call site be understood without opening the declaration?
Are Bool, Int, String, URL, Date, or Data parameters labeled clearly?
Are defaults stable semantic choices or hidden policy?
Would changing a default surprise existing clients?
Should a long parameter list become an Options type?
Is inout causing overlapping access?
Did a function value erase labels/defaults?
Would a closure-heavy API read better with named closure labels or a result builder?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Argument labels make functions easier to read. Defaults let callers omit values. `inout` lets functions modify variables.

### Senior answer

> Labels are part of Swift API design because they encode argument roles at the call site. Defaults are useful for progressive disclosure but should be stable, especially in libraries. `inout` expresses explicit, exclusive mutation and is not a general reference mechanism.

### Staff-level answer

> I design Swift function signatures as durable API surfaces. Labels are semantic documentation and affect overload shape; defaults encode policy and can become source or binary compatibility traps; `inout` is a short-lived exclusive mutation boundary. For public APIs, I avoid long primitive parameter lists and unstable defaults. I use domain types, options structs, and named policies when the API needs to evolve. I also consider how function values erase labels/defaults and how concurrency, sendability, and actor isolation affect closure and default-argument design.

Staff-level questions to ask:

```text
Is this function signature stable enough to expose across module boundaries?
Are defaults expressing durable behavior or mutable product policy?
Would an Options type make this API easier to evolve?
Are labels describing roles, or just repeating types?
Could any parameter be modeled as an enum to eliminate invalid states?
Does inout introduce exclusivity or aliasing risks?
Does passing this function as a value erase important call-site clarity?
Does this closure-heavy signature still read clearly with trailing closure syntax?
```

---

## 9. Interview-ready summary

Swift function signatures are API design, not just syntax. Argument labels make roles clear at the call site and should be chosen according to how the call reads. Defaults are good for progressive disclosure, but public defaults are semantic commitments because changing them can split behavior across clients depending on recompilation. Variadics are useful for list-like inputs, not configuration. `inout` means explicit, short-lived, exclusive mutation of caller storage, not a general reference. A strong Swift API minimizes positional ambiguity, avoids primitive obsession, puts stable defaults near the end, and moves policy-heavy parameter sets into named domain types.

---

## 10. Flashcards

Q: Are Swift argument labels just syntax decoration?  
A: No. They are part of API design and declaration naming. They communicate argument roles at the call site.

Q: When should the first argument label be omitted?  
A: When the first argument naturally completes the base-name phrase, when arguments cannot be usefully distinguished, or for value-preserving conversions such as `Int64(someUInt32)`.

Q: Why are default arguments dangerous in public libraries?  
A: They can silently change behavior for recompiled clients, while already compiled clients may retain old default behavior.

Q: Are default arguments part of a function value’s type?  
A: No. A function with a default parameter still becomes a function value requiring that parameter.

Q: Are argument labels part of function types?  
A: No. Function types contain parameter and result types, not argument labels.

Q: What does `inout` mean semantically?  
A: Temporary exclusive mutable access to caller-owned storage for the duration of the call.

Q: Why does Swift require `&` for `inout` arguments?  
A: To make mutation visible at the call site.

Q: Why is passing the same variable to two `inout` parameters invalid?  
A: It creates overlapping mutable access to the same storage, violating exclusivity.

Q: When should you replace several parameters with an options type?  
A: When parameters are policy-heavy, likely to evolve, or make call sites noisy and hard to audit.

Q: What is the smell in `send(url, true, false, 3)`?  
A: Positional primitive parameters hide meaning; labels or domain types should express the roles.

---

## 11. Related sections

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A6 — `defer`, Early Exits, and Cleanup Semantics]]
- [[A8 — Closures, escaping, and capture semantics]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[D6 — `Sendable` and `@Sendable`]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — A7 section and caveats.
- The Swift Programming Language — Functions. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/functions/?utm_source=chatgpt.com "Functions | Documentation - Swift Programming Language"))
- Swift API Design Guidelines — clarity, parameter labels, defaults, and call-site fluency. ([Swift.org](https://swift.org/documentation/api-design-guidelines/?utm_source=chatgpt.com "API Design Guidelines | Swift.org"))
- The Swift Programming Language — Memory Safety. ([Swift Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety/?utm_source=chatgpt.com "Memory Safety"))
- Swift compiler diagnostics — exclusivity violation. ([Swift Belgeleri](https://docs.swift.org/compiler/documentation/diagnostics/exclusivity-violation/?utm_source=chatgpt.com "Overlapping accesses, but operation requires exclusive ..."))
- Swift ABI Stability Manifesto — default argument expressions emitted into the caller. ([Swift Forums](https://forums.swift.org/t/swift-abi-stability-manifesto/4987 "Swift ABI Stability Manifesto - Discussion - Swift Forums"))
- SE-0279 — Multiple Trailing Closures; useful context for closure-heavy API surface design. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0279-multiple-trailing-closures.md "swift-evolution/proposals/0279-multiple-trailing-closures.md at main · swiftlang/swift-evolution · GitHub"))
- SE-0411 — Isolated default value expressions; modern Swift concurrency interaction with default arguments. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0411-isolated-default-values.md "swift-evolution/proposals/0411-isolated-default-values.md at main · swiftlang/swift-evolution · GitHub"))