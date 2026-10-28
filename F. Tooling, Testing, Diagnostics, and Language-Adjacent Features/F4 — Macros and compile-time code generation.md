---
tags:
  - swift
  - ios
  - interview-prep
  - macros
  - tooling
---

# F4 — Macros and Compile-Time Code Generation

**Learning goals:**

- Separate the public macro declaration, host syntax plugin, and generated client program.
- Check supported syntax, expansion diagnostics, compiler integration and runtime effects independently.
- Choose a macro role deliberately and preserve generated access, isolation, ownership and availability contracts.

**Prerequisites:** [[B10 — API Design Guidelines and Swift-Native Surface Design]]/[[B11 — Library Evolution and Resilience-Aware API Design]]/[[B13 — Property Wrappers and Generated Storage Semantics]]/[[B14 — Result Builders and DSL Construction]] for API, wrappers and builders; [[D6 — `Sendable`, `@Sendable`, and `sending`]]/[[D14 — Approachable Concurrency and Default Isolation]] for isolation; [[E5 — Modules, packages, targets, products, and resources]] for package targets; [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]/[[F5 — Language mode, build settings, and migration flags]] for compiler reproductions and settings.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A macro can replace repeated source structure, but the expanded program still has ordinary Swift rules. Three components participate: a **public declaration** tells client type checking what macro call is supported, a **host plugin** transforms syntax during compilation, and **generated Swift** is checked and executed in the client.

The plugin sees syntax, not a general database of every inferred type or conformance. A syntax snapshot, a successful client build, and a runtime evaluation count answer different questions. [Expression macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

After saving all six package files, build the syntax tests and ordinary client from the package root. SwiftPM handles the public module and host-plugin loading for these targets.

```sh
cd /tmp/MacroNotes
swift test --filter MacroTests
swift run MacroClient
```

This validation host also needs `--disable-sandbox -Xswiftc -disable-sandbox` to launch compiler plugins in its restricted execution environment. The package dependency remains the pinned public SwiftSyntax release.

### State the client contract before implementing syntax

Create `/tmp/MacroNotes` and save the six complete package files at their stated relative paths, including the MacroClient file in the worked example. The other Swift blocks are independent clients or rejected examples, not extra files in those targets. sourceValue returns the input and its source spelling; RoutePath adds a path member; ReplaceWithZero replaces a function body. These are custom teaching macros, not built-in APIs. The expression’s generic T remains Copyable.

**File:** Sources/InterviewMacros/Macros.swift

```swift
@freestanding(expression)
public macro sourceValue<T>(_ value: T) -> (T, String) =
    #externalMacro(module: "InterviewPlugin", type: "SourceValueMacro")

@attached(member, names: named(path))
public macro RoutePath(_ path: String) =
    #externalMacro(module: "InterviewPlugin", type: "RoutePathMacro")

@attached(body)
public macro ReplaceWithZero() =
    #externalMacro(module: "InterviewPlugin", type: "ReplaceWithZeroMacro")
```

**Walkthrough:** #externalMacro identifies the implementation module/type. Freestanding expression, attached member, and attached body roles transform different surfaces. A member declaration’s names clause constrains what it introduces; it does not automatically add public access or a protocol conformance.

### Read a macro call as a generated expression

After building MacroNotes below, this is a separate ordinary InterviewMacros client. Read the call first, then inspect the implementation that produces it.

```swift
import InterviewMacros

let pair = #sourceValue(7)
precondition(pair.0 == 7 && pair.1 == "7")
```

**Walkthrough:** The caller writes #sourceValue(7). Expansion supplies the tuple (7, "7"); the first element is executable input and the second is its source text. The compiler checks that tuple against the public generic declaration before this client runs.

### Transform syntax, keeping the executable expression once

The complete implementation target follows. SwiftSyntaxBuilder’s literal interpolation escapes source text as data. RoutePath accepts structs and noninterpolated literals whose decoded value begins with /; that is a narrow syntax policy, not comprehensive URL validation. BodyMacro supplies the replacement body rather than adding to the old one.

**File:** Sources/InterviewPlugin/Implementation.swift

```swift
import SwiftParser
import SwiftSyntax
import SwiftSyntaxBuilder
import SwiftSyntaxMacros

public struct SourceValueMacro: ExpressionMacro {
    public static func expansion(
        of node: some FreestandingMacroExpansionSyntax,
        in context: some MacroExpansionContext
    ) throws -> ExprSyntax {
        guard node.arguments.count == 1, let argument = node.arguments.first?.expression else {
            throw MacroExpansionErrorMessage("sourceValue requires one expression")
        }
        return "(\(argument), \(literal: argument.trimmedDescription))"
    }
}

public struct RoutePathMacro: MemberMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingMembersOf declaration: some DeclGroupSyntax,
        conformingTo protocols: [TypeSyntax],
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] {
        guard declaration.is(StructDeclSyntax.self) else {
            throw MacroExpansionErrorMessage("RoutePath requires a struct")
        }
        guard let arguments = node.arguments?.as(LabeledExprListSyntax.self),
            arguments.count == 1,
            let literal = arguments.first?.expression.as(StringLiteralExprSyntax.self),
            let path = literal.representedLiteralValue,
            path.hasPrefix("/")
        else {
            throw MacroExpansionErrorMessage(
                "RoutePath requires a noninterpolated string literal starting with /")
        }
        return ["static var path: Swift.String { \(literal: path) }"]
    }
}

public struct ReplaceWithZeroMacro: BodyMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingBodyFor declaration: some DeclSyntaxProtocol & WithOptionalCodeBlockSyntax,
        in context: some MacroExpansionContext
    ) throws -> [CodeBlockItemSyntax] {
        ["return 0"]
    }
}
```

**Walkthrough:** SourceValueMacro inserts argument once as executable syntax, then its trimmed spelling through literal:. RoutePathMacro checks declaration kind/literal syntax and emits an internal static Swift.String member. ReplaceWithZeroMacro discards the old body and emits return 0, which must still satisfy the declared function type/effects. The compiler checks resulting relationships.

### Register the host plugin and build its dependency graph

The entry lists implementation types that the public declarations name. CompilerPluginSupport’s .macro target is distinct from a build-tool plugin; the implementation runs on the build host even when the generated client targets iOS.

**File:** Sources/InterviewPlugin/Plugin.swift

```swift
import SwiftCompilerPlugin
import SwiftSyntaxMacros

@main
struct InterviewPlugin: CompilerPlugin {
    let providingMacros: [Macro.Type] = [
        SourceValueMacro.self, RoutePathMacro.self, ReplaceWithZeroMacro.self,
    ]
}
```

**File:** Package.swift

```swift
import CompilerPluginSupport
// swift-tools-version: 6.4
import PackageDescription

let package = Package(
    name: "MacroNotes",
    platforms: [.macOS(.v15), .iOS(.v18)],
    products: [.library(name: "InterviewMacros", targets: ["InterviewMacros"])],
    dependencies: [
        .package(url: "https://github.com/swiftlang/swift-syntax.git", exact: "604.0.0")
    ],
    targets: [
        .macro(
            name: "InterviewPlugin",
            dependencies: [
                .product(name: "SwiftCompilerPlugin", package: "swift-syntax"),
                .product(name: "SwiftSyntax", package: "swift-syntax"),
                .product(name: "SwiftSyntaxBuilder", package: "swift-syntax"),
                .product(name: "SwiftSyntaxMacros", package: "swift-syntax"),
                .product(name: "SwiftParser", package: "swift-syntax"),
            ],
            swiftSettings: [
                .defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ]
        ),
        .target(
            name: "InterviewMacros", dependencies: ["InterviewPlugin"],
            swiftSettings: [
                .defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ]),
        .executableTarget(
            name: "MacroClient", dependencies: ["InterviewMacros"],
            swiftSettings: [
                .defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ]),
        .testTarget(
            name: "MacroTests",
            dependencies: [
                "InterviewPlugin",
                .product(name: "SwiftSyntaxMacrosTestSupport", package: "swift-syntax"),
            ],
            swiftSettings: [
                .defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
                .enableUpcomingFeature("InferIsolatedConformances"),
            ]
        ),
    ],
    swiftLanguageModes: [.v6]
)
```

**Walkthrough:** The public target depends on its host macro target, while the ordinary executable imports the public module. The package pins SwiftSyntax 604.0.0, aligned with this Swift 6.4 toolchain, for reproducibility. Use a supported compiler/dependency combination and test upgrades rather than assuming every patch needs a new pin. Validation fetched this exact tag and builds it through a temporary local mirror; no private toolchain-library path substitutes for the package dependency. [SwiftSyntax releases](https://github.com/swiftlang/swift-syntax/blob/604.0.0/README.md)

### Inspect generated syntax with dedicated tests

These three tests assert expression expansion, a member expansion, and the unsupported-route diagnostic. They are the syntax layer, not substitutes for the ordinary client used later.

**File:** Tests/MacroTests/MacroTests.swift

```swift
import InterviewPlugin
import SwiftSyntaxMacros
import SwiftSyntaxMacrosTestSupport
import XCTest

final class MacroTests: XCTestCase {
    func testOnceOnlyExpansion() {
        assertMacroExpansion(
            "#sourceValue(next())",
            expandedSource: "(next(), \"next()\")",
            macros: ["sourceValue": SourceValueMacro.self]
        )
    }

    func testRoute() {
        assertMacroExpansion(
            """
            @RoutePath("/users")
            struct Users {}
            """,
            expandedSource: """
                struct Users {

                    static var path: Swift.String {
                        "/users"
                    }
                }
                """,
            macros: ["RoutePath": RoutePathMacro.self]
        )
    }

    func testDiagnostic() {
        assertMacroExpansion(
            """
            @RoutePath("users")
            struct Users {}
            """,
            expandedSource: "struct Users {}",
            diagnostics: [
                DiagnosticSpec(
                    message: "RoutePath requires a noninterpolated string literal starting with /",
                    line: 1, column: 1
                )
            ],
            macros: ["RoutePath": RoutePathMacro.self]
        )
    }
}
```

**Walkthrough:** A matching expansion string establishes the transformation representation. Diagnostic locations/messages are part of this test contract. A separate client must still check types, access, isolation, deployment and executable effects.

## Rules and pitfalls

### Duplicating executable syntax duplicates effects

**Common pitfall:** Emitting an argument twice because both value and description are wanted. This is the mistaken generated Swift, not a supported sourceValue expansion.

**Mistake:** Standalone runnable example.

```swift
func duplicatedProbe() {
    var evaluations = 0
    func next() -> Int {
        evaluations += 1
        return 7
    }
    let pair = (next(), next())
    precondition(pair.0 == 7 && pair.1 == 7 && evaluations == 2)
}
duplicatedProbe()
```

**Consequence:** The two tuple elements evaluate next twice.

**Why:** Both positions contain executable syntax. Source spelling must be emitted as string data instead of another invocation.

**Correction:** Use the real expression macro, whose implementation emits the argument only once.

```swift
import InterviewMacros

func onceProbe() {
    var evaluations = 0
    func next() -> Int {
        evaluations += 1
        return 7
    }
    let pair = #sourceValue(next())
    precondition(pair.0 == 7 && pair.1 == "next()" && evaluations == 1)
}
onceProbe()
```

**What changes:** The result contains one evaluated Int and one source String. The separate package snapshot verifies that shape, while this executable observes the effect count.

### Raw insertion into a string literal does not escape source text

**Common pitfall:** Generating a source-description literal by concatenating quotes around syntax. This bad expansion is intentionally rejected.

**Mistake:** Compile-only; intentionally rejected.

```swift
let pair = ("x", "call("x")")
```

**Consequence:** The embedded quotes break the generated Swift expression.

**Why:** Source spelling can contain quotes, backslashes and raw-string delimiters. Text concatenation does not encode them as literal data.

**Correction:** Use the real macro’s literal interpolation and check an input that contains a quote.

```swift
import InterviewMacros

let pair = #sourceValue(#"a"b"#)
precondition(pair.0 == #"a"b"#)
precondition(pair.1 == ##"#"a"b"#"##)
```

**What changes:** The input value and its exact source spelling survive as separate values. Expansion and compiled execution both matter; an aesthetically plausible string snapshot can still be invalid Swift.

### A public decorated type does not make a generated member public

Shared PrivateRoutes module for the negative client. It is a separately compiled module, not prepended to its consumer. The sample macro intentionally emits internal path access.

```swift
import InterviewMacros

@RoutePath("/users") public struct UsersRoute {}
```

**Common pitfall:** Expecting the macro-generated member to inherit public visibility from the struct.

**Mistake:** Ordinary external client; intentionally rejected:

```swift
import PrivateRoutes

let path = UsersRoute.path
```

**Consequence:** Clients can name the public type but not its internal generated path.

**Why:** Generated code has the access/isolation the macro emitted or the client inferred. Access modifiers do not automatically propagate from the decorated declaration.

**Correction:** Publish an explicit supported API or deliberately change the macro’s generation contract. This PublicRoutes module exposes a real public facade over the internal generated detail:

```swift
import InterviewMacros

@RoutePath("/users") public struct UsersRoute {}
public func usersPath() -> String { UsersRoute.path }
```

**What changes:** The ordinary client below calls the intended public function. A macro that promises a public/nonisolated member would need to emit those modifiers deliberately and validate that contract across modules/default-isolation policies.

```swift
import PublicRoutes

precondition(usersPath() == "/users")
```

## Worked example

### Expand, compile and execute three distinct roles

**Setup:** Complete ordinary MacroClient in the shared package. It imports public declarations; the package loads the registered host plugin.

**Predict:** Will the unresolved name in the replaced body prevent this valid generated program from compiling?

```swift
import InterviewMacros

@RoutePath("/users")
struct UsersRoute {}

@ReplaceWithZero
func replaced() -> Int {
    nameThatDoesNotExist
}

func probe() {
    var evaluations = 0
    func next() -> Int {
        evaluations += 1
        return 7
    }
    let (value, source) = #sourceValue(next())
    precondition(value == 7 && source == "next()" && evaluations == 1)
    precondition(UsersRoute.path == "/users")
    precondition(replaced() == 0)
    print(value, source, evaluations, UsersRoute.path, replaced())
}

probe()
```

**Outcome:** The client prints 7 next() 1 /users 0.

**Why:** The expression evaluates once, the member supplies the path, and the body replacement removes the unresolved original. What is checked is the generated return 0, not the discarded body.

## Interview questions

### What can a macro validate from syntax, and what must the generated contract leave to the compiler?

**Answer:** It can inspect declaration kind, argument syntax and supported literal content. General inferred type/conformance, actor access and generated-program validity remain compiler responsibilities.

**Example:** Standalone runnable example.

```swift
import InterviewMacros

@RoutePath("/users") struct UsersRoute {}
precondition(UsersRoute.path == "/users")
```

**Walkthrough:** The plugin accepts this struct and slash-prefixed literal and emits a path. The variable-form negative below is rejected even if its runtime string would be identical.

**Boundary:** This is a prefix/literal policy, not complete URL validation or a general semantic-query API. A production macro must define conflicts, names and access policy.

An independent **intentionally rejected** syntactic input uses a variable rather than a literal:

```swift
import InterviewMacros

let path = "/users"
@RoutePath(path) struct UsersRoute {}
```

### Why can a macro snapshot pass while a client fails to compile or behaves incorrectly?

**Answer:** The snapshot checks syntax representation; type checking and behavior can still fail because of generated names, effects, access, isolation, availability or duplicated evaluation. Keep separate integration/runtime layers.

**Example:** Independent intentionally rejected client.

```swift
import InterviewMacros

@RoutePath("/users") struct Conflict {
    static var path: String { "/existing" }
}
```

**Walkthrough:** The syntactically sensible generated member conflicts with an existing member, so the compiler rejects the actual client. The plugin does not discover every declaration in other files/extensions.

**Boundary:** Snapshot tests need useful negative diagnostics too, but cannot replace ordinary compiler clients. A unique-name facility helps local generated temporaries; it does not excuse an undefined public naming contract.

### How can a body macro replace an unresolved original body, and what constraints still apply to its replacement?

**Answer:** The role supplies a new body, so the discarded original does not need to be type-correct as the implementation. The replacement still must satisfy return type, effects, isolation, ownership and availability.

**Example:** Standalone runnable example.

```swift
import InterviewMacros

@ReplaceWithZero func replaced() -> Int { nameThatDoesNotExist }
precondition(replaced() == 0)
```

**Walkthrough:** The unresolved original is replaced by return 0 and the Int result executes correctly. The independent String variant below fails on the generated type relationship.

**Boundary:** A body macro is not universally additive. This teaching macro discards behavior and is not a proposed production abstraction.

```swift
import InterviewMacros

@ReplaceWithZero func wrong() -> String { "original" }
```

### Which costs and compatibility obligations justify a macro instead of ordinary code, a protocol or a wrapper?

**Answer:** Use generation when removing repeated syntax is worth maintaining diagnostics, plugin builds, SwiftSyntax/toolchain support and generated API compatibility. Prefer an ordinary abstraction when it already expresses the contract clearly.

**Example:** Standalone runnable example.

```swift
func withSource<T>(_ value: T, source: String) -> (T, String) { (value, source) }
func ordinaryProbe() {
    var evaluations = 0
    func next() -> Int {
        evaluations += 1
        return 7
    }
    let pair = withSource(next(), source: "next()")
    precondition(pair.0 == 7 && pair.1 == "next()" && evaluations == 1)
}
ordinaryProbe()
```

**Walkthrough:** An ordinary helper preserves once-only evaluation, but the caller must keep its source label synchronized manually. The macro buys automatic syntax capture; a protocol or wrapper can address different repeated behavior without a host plugin.

**Boundary:** Build time, diagnostics, generated-member evolution and host/client platform support are real costs. The macro’s deployment requirements come from generated client APIs as well as compiler support, not from the host’s ability to launch the plugin.

### Extension: Does a syntactically accepted declaration prove a conformance or ownership relationship?

**Answer:** No. Generated declarations are ordinary Swift; the compiler must check their constraints and the public macro’s generic input defaults.

**Example:** Independent intentionally rejected compiler-conformance client.

```swift
import InterviewMacros

protocol Endpoint { associatedtype Response: Decodable }
struct NotDecodable {}
@RoutePath("/users") struct InvalidEndpoint: Endpoint { typealias Response = NotDecodable }
```

**Walkthrough:** The route syntax is accepted, but Endpoint conformance fails because the chosen response does not satisfy Decodable. The macro does not claim to validate inferred conformances.

**Boundary:** The expression macro’s T is also Copyable in this sample; suppressing ownership defaults would require a new supported contract, not a claim that syntax substitution handles every noncopyable value.

The independent **intentionally rejected** expression input exposes that Copyable limit:

```swift
import InterviewMacros

struct Token: ~Copyable { let id: Int }
func invalid() {
    let token = Token(id: 1)
    _ = #sourceValue(token)
}
```

## Focused exercise

**Task:** Generate an expression wrapper that evaluates its argument once; verify expansion, compiler integration and behavior separately.

**Acceptance checks:**

- The emitted tuple contains the expression once as executable syntax and escaped spelling as String data.
- A separate client compiles and observes exactly one evaluation, including a throwing input.
- Unsupported syntax, incompatible generated types and missing plugin loading reject independently.

**Solution:** Use the complete package files above and run its three syntax tests and ordinary MacroClient. This separate client demonstrates the throwing expression boundary too:

```swift
import InterviewMacros

enum Failure: Error { case expected }
func throwingProbe() throws {
    var calls = 0
    func value(_ succeeds: Bool) throws -> Int {
        calls += 1
        if !succeeds { throw Failure.expected }
        return 7
    }
    let first = try #sourceValue(value(true))
    precondition(first.0 == 7 && first.1 == "value(true)" && calls == 1)
    do {
        _ = try #sourceValue(value(false))
        preconditionFailure()
    } catch Failure.expected {}
    precondition(calls == 2)
}
try throwingProbe()
```

```sh
# Run from the complete temporary MacroNotes package.
swift test --filter MacroTests
swift run MacroClient
# For a separately compiled client, include the public module and host plugin.
# Discover paths from this build rather than assuming a historical .build/debug layout.
swift build --show-bin-path
# Add -Xfrontend -dump-macro-expansions to the actual client compiler command
# to inspect generated syntax while retaining imports and plugin configuration.
```

**Reasoning:** Runtime checks observe each argument once on success/throw. Syntax snapshots establish the representation; compiled clients check relationships and loaded-plugin integration. The host plugin is built from pinned package sources, while macOS/iOS client compilation and MainActor-default controls are separate. Macro diagnostics, ordinary compiler failures and missing-plugin failures should not be conflated.

## In practice

**Choose it when:** Use generation only for a maintained syntax contract whose benefits exceed an ordinary abstraction’s simplicity.

**Investigate by:** Inspect expansion, compile cross-module/default-isolation clients, and execute side-effect/error cases with a supported toolchain/dependency.

**Strong evidence:** Preserve host/plugin loading and generated access/isolation/ownership/availability contracts, not merely matching expansion text.

## References

### Related guides

- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[B13 — Property Wrappers and Generated Storage Semantics]]
- [[B14 — Result Builders and DSL Construction]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F1 — Testing Strategy in Modern Swift]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]
- [[F5 — Language mode, build settings, and migration flags]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [Expression macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md)
- [SwiftSyntax versioning](https://github.com/swiftlang/swift-syntax/blob/604.0.0/README.md)
- [604.0.0 release](https://github.com/swiftlang/swift-syntax/releases/tag/604.0.0)
- [SwiftSyntax macro documentation](https://github.com/swiftlang/swift-syntax/blob/604.0.0/Sources/SwiftSyntaxMacros/SwiftSyntaxMacros.docc/SwiftSyntaxMacros.md)
- [SE-0389: Attached macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0389-attached-macros.md)
- [SE-0394: Package manager support for custom macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0394-swiftpm-expression-macros.md)
- [SE-0397: Freestanding declaration macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0397-freestanding-declaration-macros.md)
- [SE-0415: Function body macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0415-function-body-macros.md)
