---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - api-design
  - result-builders
  - dsl
---
## 0. Rubric snapshot

**Rubric expectation**

Understand how result builders transform syntax and why they are useful for declarative APIs. The rubric calls out B14 as a senior-level topic in Type System, Abstraction, and API Design.

**Caveats**

Result builders can create cryptic type-checking failures and hide control-flow semantics from readers.

**You should be able to answer**

- What problem does a result builder solve better than plain closures?
- What are the readability and compile-time costs of overusing builders?

**You should be able to do**

- Design a tiny builder-based DSL and explain whether it is actually better than an array of closures or plain structs.

---

## 1. Core mental model

A result builder is a compile-time transformation for a closure or function body. It lets a sequence of expressions inside a block be collected into a single result value. The feature was formalized by SE-0289 and implemented in Swift 5.4. The proposal describes result builders as a way for specially annotated functions or closures to “implicitly build up a result value from a sequence of components.” ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md "swift-evolution/proposals/0289-result-builders.md at main · swiftlang/swift-evolution · GitHub"))

The important part: the syntax still looks like Swift. A builder does not create a new runtime language. It rewrites selected statements into calls like `buildExpression`, `buildBlock`, `buildOptional`, `buildEither`, and `buildArray`. SE-0289 explicitly frames this as a limited transformation that preserves the dynamic semantics of the original code. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md "swift-evolution/proposals/0289-result-builders.md at main · swiftlang/swift-evolution · GitHub"))

In normal Swift, a closure has one return value. A result builder lets many expression statements contribute to that return value. That is why SwiftUI can let you write:

```swift
VStack {
    Text("Title")
    Text("Subtitle")
}
```

instead of manually constructing a tuple, array, or nested tree. Apple’s `ViewBuilder` is itself a `@resultBuilder`, and Apple describes it as a custom parameter attribute that constructs views from closures. ([Apple Developer](https://developer.apple.com/documentation/SwiftUI/ViewBuilder?changes=l_8_3_8&utm_source=chatgpt.com "ViewBuilder | Apple Developer Documentation"))

The key idea:

```text
Builder block syntax -> compiler transform -> builder method calls -> one final result value
```

Result builders are best when the domain is naturally declarative and structural: UI trees, HTML/XML trees, routes, menus, command lists, validation rules, layout descriptions, test scenarios, or query definitions. Apple’s WWDC21 DSL session describes result builders as one of the main Swift features for embedded DSLs, especially when a DSL gathers values and builds a tree or structure from them. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10253/ "Write a DSL in Swift using result builders - WWDC21 - Videos - Apple Developer"))

They are bad when they merely hide ordinary imperative code. If the reader has to reverse-engineer what `if`, `for`, variables, side effects, and overloads mean in your builder, the DSL is probably too clever.

---

## 2. Essential mechanics

### 2.1 A result builder is a type with static builder methods

At minimum, a result builder type needs enough methods for the syntax you want to support. The Swift language reference says a result builder must implement either `buildBlock(_:)` or both `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)`. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/?utm_source=chatgpt.com "Attributes | Documentation - Swift Programming Language"))

```swift
@resultBuilder
enum StringListBuilder {
    static func buildExpression(_ expression: String) -> [String] {
        [expression]
    }

    static func buildBlock(_ components: [String]...) -> [String] {
        components.flatMap { $0 }
    }
}

func makeList(@StringListBuilder _ content: () -> [String]) -> [String] {
    content()
}

let values = makeList {
    "A"
    "B"
    "C"
}

print(values)
```

Output:

```text
["A", "B", "C"]
```

Conceptually, the compiler rewrites the builder body roughly like this:

```swift
let v0 = StringListBuilder.buildExpression("A")
let v1 = StringListBuilder.buildExpression("B")
let v2 = StringListBuilder.buildExpression("C")
return StringListBuilder.buildBlock(v0, v1, v2)
```

Not exactly source-generated Swift, but this is the right mental model.

---

### 2.2 Control flow only works if the builder supports it

Builder syntax support is opt-in through static methods:

```swift
@resultBuilder
enum StringListBuilder {
    static func buildExpression(_ expression: String) -> [String] {
        [expression]
    }

    static func buildBlock(_ components: [String]...) -> [String] {
        components.flatMap { $0 }
    }

    static func buildOptional(_ component: [String]?) -> [String] {
        component ?? []
    }

    static func buildEither(first component: [String]) -> [String] {
        component
    }

    static func buildEither(second component: [String]) -> [String] {
        component
    }

    static func buildArray(_ components: [[String]]) -> [String] {
        components.flatMap { $0 }
    }
}
```

These unlock common syntax:

```swift
let isAdmin = true
let names = ["Aykut", "Emma"]

let result = makeList {
    "Users"

    if isAdmin {
        "Admin"
    }

    for name in names {
        name
    }
}

print(result)
```

Output:

```text
["Users", "Admin", "Aykut", "Emma"]
```

The `if` still runs like an `if`. The `for` still loops normally. The builder does not virtualize control flow. It collects produced values.

---

### 2.3 Builder blocks are not arbitrary imperative closures

You cannot always write whatever you would write in a normal closure. The builder controls what statements are valid.

Example without `buildArray(_:)`:

```swift
@resultBuilder
enum StringListBuilder {
    static func buildExpression(_ expression: String) -> [String] { [expression] }
    static func buildBlock(_ components: [String]...) -> [String] { components.flatMap { $0 } }
}

func make(@StringListBuilder _ content: () -> [String]) -> [String] {
    content()
}

let values = ["A", "B"]

let result = make {
    for value in values {
        value
    }
}
```

Compiler error with Swift 6.2.1:

```text
no_build_array.swift:13:5: error: closure containing control flow statement cannot be used with result builder 'StringListBuilder'
 1 | @resultBuilder
 2 | enum StringListBuilder {
   |      |- note: add 'buildArray(_:)' to the result builder 'StringListBuilder' to add support for 'for'..'in' loops
   |      `- note: enum 'StringListBuilder' declared here
 3 |     static func buildExpression(_ expression: String) -> [String] { [expression] }
 4 |     static func buildBlock(_ components: [String]...) -> [String] { components.flatMap { $0 } }
   :
11 | let values = ["A", "B"]
12 | let result = make {
13 |     for value in values {
   |     `- error: closure containing control flow statement cannot be used with result builder 'StringListBuilder'
14 |         value
15 |     }
```

Explicit `return` is also usually wrong inside builder bodies:

```swift
let result = make {
    return "A"
}
```

Compiler error with Swift 6.2.1:

```text
return_builder.swift:12:5: error: cannot use explicit 'return' statement in the body of result builder 'StringListBuilder'
10 | 
11 | let result = make {
12 |     return "A"
   |     |- error: cannot use explicit 'return' statement in the body of result builder 'StringListBuilder'
   |     `- note: remove 'return' statements to apply the result builder
13 | }
14 | print(result)
```

---

### 2.4 Modern Swift removed older local-variable limitations

Older Swift result builders had annoying restrictions around local variables. SE-0373 lifted those limitations so local declarations in result-builder-transformed functions are treated more like ordinary local declarations. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0373-vars-without-limits-in-result-builders.md?utm_source=chatgpt.com "0373-vars-without-limits-in-result-builders.md"))

Modern Swift lets you write clearer builder bodies:

```swift
let result = makeList {
    let title = "Users"
    title

    let subtitle: String? = "Active"
    if let subtitle {
        subtitle
    }
}
```

This matters in production because you should not contort a builder body into unreadable expression soup just to satisfy older compiler behavior.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking result builders are runtime magic

Bad mental model:

```text
SwiftUI somehow changes how if/for/closures execute at runtime.
```

Better mental model:

```text
The compiler rewrites expression statements into builder method calls.
Runtime control flow still behaves like Swift control flow.
```

Apple’s WWDC explanation is explicit: result builders gather values produced by code and combine them into a single result; the feature is compile-time and works independently of OS runtime support. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10253/ "Write a DSL in Swift using result builders - WWDC21 - Videos - Apple Developer"))

---

### Trap 2: Using a builder when plain data is clearer

Bad:

```swift
let rules = validate {
    required("email")
    minLength("password", 8)
    matches("email", emailRegex)
}
```

This may look nice, but if the rules are just data, this is often clearer:

```swift
let rules: [ValidationRule] = [
    .required("email"),
    .minLength("password", 8),
    .matches("email", emailRegex)
]
```

A result builder earns its keep when it improves structural expression, nesting, conditional inclusion, or domain readability. It is not automatically better because it removes commas.

---

### Trap 3: Hiding expensive work or side effects inside declarative syntax

Bad:

```swift
screen {
    Header()

    loadUserFromNetwork() // This looks declarative but does real work.

    Footer()
}
```

Better:

```swift
let user = try await userService.loadUser()

screen {
    Header()
    UserSection(user)
    Footer()
}
```

Builder bodies should usually describe structure. Side effects inside them make evaluation order and lifecycle harder to reason about.

---

### Trap 4: Fighting the type checker with huge builder expressions

This happens often in SwiftUI:

```swift
var body: some View {
    VStack {
        // 300 lines of nested conditionals, generic modifiers, overloads, and closures
    }
}
```

Better:

```swift
var body: some View {
    VStack {
        header
        content
        footer
    }
}

private var header: some View { ... }
private var content: some View { ... }
private var footer: some View { ... }
```

Result builders amplify generic type inference. Large builder bodies can produce poor diagnostics and slow type-checking. Swift 5.8 improved result-builder type-checking and diagnostics, but the design issue still exists in large DSL-heavy code. ([SwiftGG](https://swift.swiftgg.team/blog/swift-5.8-released/?utm_source=chatgpt.com "Swift 5.8 Released!"))

---

## 4. Direct answers to rubric questions

### Q1. What problem does a result builder solve better than plain closures?

A result builder solves the problem of turning a block of multiple expressions into one structured result without forcing the caller to manually assemble arrays, tuples, trees, or nested containers.

Plain closure:

```swift
func makePage(_ content: () -> [Node]) -> Node {
    .element("body", content())
}

let page = makePage {
    [
        .element("h1", [.text("Title")]),
        .element("p", [.text("Hello")])
    ]
}
```

Builder closure:

```swift
let page = element("body") {
    element("h1") {
        "Title"
    }

    element("p") {
        "Hello"
    }
}
```

The builder version is better when the shape of the code mirrors the shape of the domain.

Interview version:

> A result builder is useful when the API wants a structured result from many child expressions. A plain closure gives me one explicit return value, so I usually need array literals, commas, or manual tree construction. A builder lets the compiler collect the expression statements and combine them through builder methods. That is valuable for declarative APIs like SwiftUI, HTML builders, routes, menus, and validation rules where the syntax should resemble the domain structure.

---

### Q2. What are the readability and compile-time costs of overusing builders?

Overusing builders makes code harder to debug because the visible code is not the code the type checker is actually checking. Errors often point at the outer builder block instead of the real invalid child expression. Large builder bodies can also stress type inference, especially with heavily generic APIs like SwiftUI.

The readability cost is also real: a DSL requires readers to learn what expressions mean, how `if`/`for` are handled, whether expressions are evaluated immediately or stored, and what side effects are allowed.

Interview version:

> The cost is that result builders hide a transformation. That can be good for declarative structure, but bad when the builder becomes a private language. Diagnostics can become indirect, type-checking can slow down, and readers may not know whether they are looking at data construction, lazy work, side effects, or control flow. I would use builders for APIs whose structure is obvious from the syntax, and avoid them for ordinary procedural logic.

---

## 5. Code probe

The rubric has no dedicated code probe for B14. This section uses a minimal probe-style example instead.

Given:

```swift
enum Node: CustomStringConvertible {
    case text(String)
    case element(String, [Node])

    var description: String {
        switch self {
        case .text(let value):
            value
        case .element(let name, let children):
            "<\(name)>" + children.map(\.description).joined(separator: ",") + "</\(name)>"
        }
    }
}

@resultBuilder
enum NodeBuilder {
    static func buildExpression(_ expression: Node) -> [Node] {
        [expression]
    }

    static func buildExpression(_ expression: String) -> [Node] {
        [.text(expression)]
    }

    static func buildBlock(_ components: [Node]...) -> [Node] {
        components.flatMap { $0 }
    }

    static func buildOptional(_ component: [Node]?) -> [Node] {
        component ?? []
    }

    static func buildEither(first component: [Node]) -> [Node] {
        component
    }

    static func buildEither(second component: [Node]) -> [Node] {
        component
    }

    static func buildArray(_ components: [[Node]]) -> [Node] {
        components.flatMap { $0 }
    }
}

func element(_ name: String, @NodeBuilder children: () -> [Node] = { [] }) -> Node {
    .element(name, children())
}

let loggedIn = true
let names = ["Aykut", "Emma"]

let page = element("ul") {
    "Users"

    if loggedIn {
        for name in names {
            element("li") { name }
        }
    }
}

print(page)
```

### What happens?

Output with Swift 6.2.1:

```text
<ul>Users,<li>Aykut</li>,<li>Emma</li></ul>
```

### Why?

The block passed to `element("ul")` is transformed by `NodeBuilder`.

Roughly:

```text
"Users"
    -> buildExpression("Users")
    -> [.text("Users")]

if loggedIn { ... }
    -> buildOptional(...)
    -> list of child nodes or []

for name in names { ... }
    -> buildArray(...)
    -> flattened list of li nodes

whole block
    -> buildBlock(component0, component1)
    -> [Node]
```

The important semantic point: `loggedIn` is checked normally, and the `for` loop iterates normally. The builder only decides how produced values are collected.

### Equivalent non-builder version

```swift
var children: [Node] = []
children.append(.text("Users"))

if loggedIn {
    for name in names {
        children.append(.element("li", [.text(name)]))
    }
}

let page = Node.element("ul", children)
print(page)
```

The builder version is nicer only because the domain is structural and nested.

---

## 6. Exercise

### Problem

Design a tiny builder-based DSL and explain whether it is actually better than an array of closures or plain structs.

### Bad / naive version

Suppose we want to define request routes:

```swift
struct Route {
    var method: String
    var path: String
    var requiresAuth: Bool
}

let routes = [
    Route(method: "GET", path: "/profile", requiresAuth: true),
    Route(method: "POST", path: "/logout", requiresAuth: true),
    Route(method: "GET", path: "/health", requiresAuth: false)
]
```

This is already clear. A builder here may be overkill if all routes are flat data.

A worse builder would be:

```swift
let routes = routes {
    route("GET", "/profile", true)
    route("POST", "/logout", true)
    route("GET", "/health", false)
}
```

### What is wrong?

```text
The builder adds ceremony but does not improve the model.
The boolean argument is still unclear.
The DSL does not express meaningful structure.
Plain data was already readable.
```

### Improved version

A builder starts to make sense if the domain has grouping, defaults, and conditional inclusion:

```swift
struct Route: CustomStringConvertible {
    enum Method: String {
        case get = "GET"
        case post = "POST"
    }

    var method: Method
    var path: String
    var requiresAuth: Bool

    var description: String {
        "\(method.rawValue) \(path) auth=\(requiresAuth)"
    }
}

struct RouteGroup {
    var prefix: String
    var requiresAuth: Bool
    var routes: [Route]
}

@resultBuilder
enum RouteBuilder {
    static func buildExpression(_ expression: Route) -> [Route] {
        [expression]
    }

    static func buildBlock(_ components: [Route]...) -> [Route] {
        components.flatMap { $0 }
    }

    static func buildOptional(_ component: [Route]?) -> [Route] {
        component ?? []
    }

    static func buildArray(_ components: [[Route]]) -> [Route] {
        components.flatMap { $0 }
    }
}

func route(_ method: Route.Method, _ path: String, requiresAuth: Bool) -> Route {
    Route(method: method, path: path, requiresAuth: requiresAuth)
}

func group(
    _ prefix: String,
    requiresAuth: Bool,
    @RouteBuilder routes: () -> [Route]
) -> [Route] {
    routes().map {
        Route(
            method: $0.method,
            path: prefix + $0.path,
            requiresAuth: $0.requiresAuth || requiresAuth
        )
    }
}

func routes(@RouteBuilder _ content: () -> [Route]) -> [Route] {
    content()
}

let includeDebugRoutes = false

let apiRoutes = routes {
    group("/user", requiresAuth: true) {
        route(.get, "/profile", requiresAuth: false)
        route(.post, "/logout", requiresAuth: false)
    }

    if includeDebugRoutes {
        route(.get, "/debug", requiresAuth: true)
    }

    route(.get, "/health", requiresAuth: false)
}

for route in apiRoutes {
    print(route)
}
```

Output:

```text
GET /user/profile auth=true
POST /user/logout auth=true
GET /health auth=false
```

### Why this is better

This builder is defensible because it expresses actual domain structure:

```text
route list
  group /user
    inherited auth requirement
    nested routes
  optional debug route
  standalone health route
```

The builder removes noise only after the domain has hierarchy and conditional inclusion. It is not just replacing commas.

### Is it better than an array?

For a flat list, no:

```swift
let routes: [Route] = [
    route(.get, "/health", requiresAuth: false)
]
```

For grouped route declarations with defaults and conditional inclusion, yes:

```swift
let apiRoutes = routes {
    group("/user", requiresAuth: true) {
        route(.get, "/profile", requiresAuth: false)
        route(.post, "/logout", requiresAuth: false)
    }

    if includeDebugRoutes {
        route(.get, "/debug", requiresAuth: true)
    }
}
```

The staff-level judgment: introduce the builder only when it makes invalid or noisy construction patterns disappear without hiding important behavior.

---

## 7. Production guidance

Use this in production when:

```text
The domain is naturally declarative.
The syntax mirrors a tree, list, pipeline, or structured declaration.
Call sites become meaningfully clearer.
The builder has a small, documented set of supported statement forms.
The result type is explicit and testable outside the DSL.
```

Good candidates:

```text
SwiftUI-style view trees
Menus and command declarations
HTML/XML/Markdown builders
Navigation route declarations
Validation rule groups
Test scenario setup
Query/filter descriptions
Feature flag or experiment configuration
```

Be careful when:

```text
The builder body contains side effects.
The builder body becomes very large.
The DSL relies on overload-heavy generic inference.
The result type is hidden behind too many opaque types.
The builder supports complex control flow that readers cannot predict.
The builder appears in public SDK API and becomes hard to change later.
```

Avoid when:

```text
An array literal is clearer.
Plain structs communicate the model better.
The DSL only saves commas.
The builder hides networking, persistence, locks, tasks, or mutation.
The team cannot easily debug the generated type errors.
```

Debugging checklist:

```text
What is the builder's Component type?
What is the final Result type?
Which build methods are being used?
Is the failing expression accepted by buildExpression?
Does if require buildOptional or buildEither?
Does for require buildArray?
Can I extract subexpressions into local variables?
Can I split the builder into smaller computed properties/functions?
Is the error caused by the DSL or by an overloaded child expression?
Would plain structs make this easier to inspect?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Result builders let SwiftUI-like APIs return multiple values from a closure.

### Senior answer

> Result builders are compile-time transformations. The compiler rewrites a block of expression statements into calls to static builder methods such as `buildExpression`, `buildBlock`, `buildOptional`, `buildEither`, and `buildArray`. They are useful for declarative APIs where syntax should mirror structure, but they can hurt diagnostics, compile time, and readability if overused.

### Staff-level answer

> I treat result builders as API surface design tools, not syntax candy. They are justified when the domain is structural and the builder removes real accidental complexity, such as manual tree assembly, conditional child inclusion, or grouped declarations. I avoid them when plain data is clearer or when the DSL hides side effects. In public APIs, I also consider source compatibility: once clients write DSL bodies, changing supported expression types or control-flow behavior can become a breaking change. For large SwiftUI or DSL-heavy code, I design escape hatches: plain model types, smaller sub-builders, testable intermediate values, and diagnostics-friendly overloads.

Staff-level questions to ask:

```text
What domain structure does this builder make clearer?
What exact statement forms should the builder support?
What is the concrete Component type, and can it be tested directly?
What diagnostics will users see when they make common mistakes?
Can this be expressed with plain structs or arrays more clearly?
Will this builder become public API, and what changes become breaking?
Are side effects allowed inside the builder body?
How will this behave under strict concurrency and actor isolation?
```

---

## 9. Interview-ready summary

Result builders are a compile-time Swift feature for building one result from a block of expressions. A builder type marked with `@resultBuilder` defines static methods that tell the compiler how to collect expressions, optional branches, either branches, and loops into a final value. They are powerful for declarative, structural APIs like SwiftUI, menus, routes, validation rules, or HTML trees because the call-site syntax mirrors the domain. The tradeoff is that builders hide a transformation, so diagnostics, compile time, and readability can suffer. I would use them when they remove real structural noise, not just to avoid commas or make ordinary imperative code look fancy.

---

## 10. Flashcards

Q: What is a result builder?

A: A compile-time transformation that turns a block of expressions into calls to static builder methods, producing one final result.

Q: What method is commonly required in a basic result builder?

A: `buildBlock(_:)`, or the newer partial-block pair `buildPartialBlock(first:)` and `buildPartialBlock(accumulated:next:)`.

Q: What method supports `if` without `else`?

A: `buildOptional(_:)`.

Q: What methods support `if/else` and `switch`-like branching?

A: `buildEither(first:)` and `buildEither(second:)`.

Q: What method supports `for` loops?

A: `buildArray(_:)`.

Q: Does a result builder change runtime control-flow semantics?

A: No. `if` still branches normally, and `for` still loops normally. The builder collects the produced values.

Q: Why can result-builder errors be cryptic?

A: The compiler type-checks the transformed form, so the reported error may involve builder methods, opaque result types, generic overloads, or an outer block rather than the real invalid expression.

Q: When is a result builder worse than an array literal?

A: When the data is flat, simple, and already readable as plain structs or arrays.

Q: What is a good production rule for custom builders?

A: Use them when the syntax mirrors real domain structure and the intermediate result type remains explicit, testable, and understandable.

---

## 11. Related sections

- [[A14 — Operators, literals, and language affordances]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[B13 — Property wrappers and generated storage semantics]]
- [[B12 — Variadic generics and parameter packs]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F4 — Macros and compile-time code generation]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — B14 rubric and exercise.
- The Swift Programming Language — Attributes / `@resultBuilder`. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/?utm_source=chatgpt.com "Attributes | Documentation - Swift Programming Language"))
- Swift Evolution SE-0289 — Result Builders. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md "swift-evolution/proposals/0289-result-builders.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Evolution SE-0373 — Lift all limitations on variables in result builders. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0373-vars-without-limits-in-result-builders.md?utm_source=chatgpt.com "0373-vars-without-limits-in-result-builders.md"))
- Apple Developer — `ViewBuilder`. ([Apple Developer](https://developer.apple.com/documentation/SwiftUI/ViewBuilder?changes=l_8_3_8&utm_source=chatgpt.com "ViewBuilder | Apple Developer Documentation"))
- Apple WWDC21 — Write a DSL in Swift using result builders. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10253/ "Write a DSL in Swift using result builders - WWDC21 - Videos - Apple Developer"))