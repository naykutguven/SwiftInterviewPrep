---
tags:
  - swift
  - ios
  - interview-prep
  - macros
  - tooling
---

# F4 — Macros and Compile-Time Code Generation

**Goal:** Generate an expression wrapper that evaluates its argument exactly once and validate the expanded client.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A macro has a public declaration contract, a host-side syntax implementation, and generated Swift code compiled in the client. The plugin transforms syntax; it does not evaluate arbitrary arguments as runtime values or query every inferred type/conformance. Generated code still obeys access, ownership, isolation, availability, and ordinary evaluation rules. [Expression macros](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0382-expression-macros.md)

## Rules and pitfalls

### Validate expansion and compiled behavior separately

Use syntax tests for supported input, emitted structure, escaping, and diagnostics. Compile an ordinary client to check generated types, names, access, isolation, and availability; run it to check evaluation count and behavior.

**Common pitfall:** A snapshot can match expected text that does not compile or evaluates a side effect twice. Keep each expression once in executable output and encode source text as a string literal safely.

### Choose generation only for a useful contract

Expression, declaration, member, accessor, conformance/extension, and body roles transform different surfaces. A body macro can replace an original implementation, so macros are not universally additive-only. Host plugin/toolchain dependencies differ from the deployment requirements of generated client code.

**Common pitfall:** A plugin accepting literal syntax does not prove comprehensive route validation or discover every name conflict in other files. State the syntactic domain and let the compiler check resulting Swift relationships.

## Interview questions

### What can a macro validate from syntax, and what must the generated Swift contract leave to the compiler?

It can validate syntax shape, literal spelling/decoded content, declaration kinds, and supported structural arguments. General inferred type relationships, conformance, actor access, and generated-code validity belong to compiler checking. Macro declarations participate in type checking, but SwiftSyntax is not a general semantic-query API.

### Why can a macro snapshot pass while a client fails to compile or behaves incorrectly?

Text can have wrong names, types, access, availability, isolation, or duplicated effects. A snapshot verifies one expansion representation, while a real client verifies integration and runtime behavior. Test all three rather than treating generated text as already checked application code.

### How can a body macro replace an unresolved original body, and what constraints still apply to its replacement?

A body role supplies replacement body syntax, so the discarded unresolved original need not be type-correct as an implementation. The replacement must still satisfy the declaration's result, effects, ownership, isolation, and available APIs. The fixture returning zero fails when its declared result is changed to String.

### Which costs and compatibility obligations justify using a macro rather than a protocol, property wrapper, or ordinary code?

Generation is justified when it removes meaningful repeated structure while preserving a clear contract that ordinary code cannot express as well. Weigh diagnostics, build time, host/plugin distribution, SwiftSyntax compatibility, generated API evolution, and testing burden. A protocol, wrapper, or helper is simpler when it already expresses the behavior.

## Focused exercise

**Task:** Generate one expression wrapper that evaluates its argument once. Check expansion, client compilation, and runtime behavior separately.

**Acceptance checks:**

- Expansion contains the input expression once as executable syntax and its escaped source as data.
- A separate client compiles and observes one evaluation.
- Unsupported syntax and incompatible generated result types reject separately; runtime output is checked independently of snapshots.

**Solution:** The public expression declaration names a host implementation. The implementation emits the argument once as executable syntax and its source spelling as escaped string data. Put these declarations in the separate public/implementation targets of the complete package below; SwiftSyntax 604.0.0 and plugin registration are dependencies.

```swift
// Public declaration target
@freestanding(expression)
public macro sourceValue<T>(_ value: T) -> (T, String) =
    #externalMacro(module: "InterviewPlugin", type: "SourceValueMacro")
```

```swift
import SwiftSyntax
import SwiftSyntaxBuilder
import SwiftSyntaxMacros
import SwiftParser

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
```

For next() returning 7 and incrementing a counter, the generated tuple must yield 7, "next()", and one evaluation. Check the emitted expansion, compile that client, and run its counter assertions separately. The full package supplies registration, ordinary client calls, syntax tests, and independent negative cases.

**Reasoning:** The expression wrapper preserves the original expression's runtime evaluation once, while literal interpolation safely encodes its source spelling. Introducing a closure or duplicated argument would change lifetime/effects and must not be an accidental implementation detail. This example's generic input remains Copyable.


### Complete package and additional macro roles — optional

Temporary package source follows. The focused expression macro is sourceValue; the attached macros retain distinct role and body-replacement lessons. SwiftSyntax 604.0.0 and a supported compiler/plugin build are dependencies, not built-in APIs.

These are files for a **temporary** package, not additions to this Markdown repository. `sourceValue`, `RoutePath`, and `ReplaceWithZero` are custom macros defined here. They are not built-in Swift APIs. `ReplaceWithZero` is a teaching probe, not a suggested production abstraction.

#### Public declaration target

Place this in `Sources/InterviewMacros/Macros.swift`:

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

The expression's generic parameter is implicitly Copyable. This example does not promise support for a noncopyable argument. The route macro adds a named member; it does not add an endpoint conformance or response type.

#### Implementation target

Place this in `Sources/InterviewPlugin/Implementation.swift`:

```swift
import SwiftSyntax
import SwiftSyntaxBuilder
import SwiftSyntaxMacros
import SwiftParser

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
              path.hasPrefix("/") else {
            throw MacroExpansionErrorMessage("RoutePath requires a noninterpolated string literal starting with /")
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

`sourceValue` inserts the argument once as executable syntax and separately encodes its trimmed source spelling as a string literal. SwiftSyntaxBuilder's `literal:` interpolation escapes data for a Swift literal; raw concatenation would mishandle quotes and backslashes.

`RoutePath` deliberately accepts only structs and a noninterpolated literal whose decoded value starts with `/`. `representedLiteralValue` comes from SwiftParser. A variable containing an identical string is outside this syntax contract. This is a prefix rule, not comprehensive URL validation: it does not encode components, reject every invalid route, or implement parameter substitution.

Under this baseline, the generated `path` is nonisolated and **internal**, including on a public struct. A macro promising a nonisolated member across MainActor-default clients would need to emit that annotation explicitly. Access levels do not automatically propagate from the decorated declaration. A production macro must define its intended access policy and test it across modules. The compiler diagnoses duplicate `path` declarations; this sample does not attempt to discover all conflicts across files or extensions.

For a macro that introduces local temporaries, use the expansion context's unique-name facilities and check name lookup. Multiple roles must not rely on mutable plugin state or expansion ordering to coordinate generated identifiers. Keep output reproducible from supported inputs and context.

#### Plugin registration

Place this in `Sources/InterviewPlugin/Plugin.swift`:

```swift
import SwiftCompilerPlugin
import SwiftSyntaxMacros

@main
struct InterviewPlugin: CompilerPlugin {
    let providingMacros: [Macro.Type] = [
        SourceValueMacro.self, RoutePathMacro.self, ReplaceWithZeroMacro.self
    ]
}
```

The module/type strings in `#externalMacro` must identify registered implementation types. Importing the public API module is not enough if the compiler cannot load its plugin when expansion is needed.

#### Package manifest

Place this in `Package.swift`:

```swift
// swift-tools-version: 6.4
import PackageDescription
import CompilerPluginSupport

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
                .product(name: "SwiftParser", package: "swift-syntax")
            ],
            swiftSettings: [.defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"), .enableUpcomingFeature("InferIsolatedConformances")]
        ),
        .target(name: "InterviewMacros", dependencies: ["InterviewPlugin"],
                swiftSettings: [.defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"), .enableUpcomingFeature("InferIsolatedConformances")]),
        .executableTarget(name: "MacroClient", dependencies: ["InterviewMacros"],
                          swiftSettings: [.defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"), .enableUpcomingFeature("InferIsolatedConformances")]),
        .testTarget(
            name: "MacroTests",
            dependencies: [
                "InterviewPlugin",
                .product(name: "SwiftSyntaxMacrosTestSupport", package: "swift-syntax")
            ],
            swiftSettings: [.defaultIsolation(nil), .enableUpcomingFeature("NonisolatedNonsendingByDefault"), .enableUpcomingFeature("InferIsolatedConformances")]
        )
    ],
    swiftLanguageModes: [.v6]
)
```

`CompilerPluginSupport` provides `.macro`; it is distinct from a SwiftPM build-tool plugin. The library target exposes the declarations, while its macro dependency supplies the host executable. The test target imports implementation types to inspect expansions. All files named by the target layout are supplied in this chapter.

SwiftSyntax 604.0.0 was released alongside Swift 6.4. Its release families track Swift tooling; use a version supported by the selected compiler and verify actual package builds. This example pins one checked version for reproducibility. That is not a claim that every compiler patch needs a new package pin, nor a universal dependency-range policy for libraries. Preserve the resolved dependency graph in applications and exercise supported dependency ranges in library CI. [SwiftSyntax versioning](https://github.com/swiftlang/swift-syntax/blob/604.0.0/README.md), [604.0.0 release](https://github.com/swiftlang/swift-syntax/releases/tag/604.0.0)

Swift tools version 6.4 and Swift language mode 6 are different settings. The manifest's `.defaultIsolation(nil)` makes its nonisolated default explicit. The checked build uses SwiftPM's Swift Build default; F5 discusses build-system selection. Do not add private toolchain library search paths to application targets as a substitute for a maintained macro package dependency.

Place this in `Sources/MacroClient/main.swift`:

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
    func next() -> Int { evaluations += 1; return 7 }
    let (value, source) = #sourceValue(next())
    precondition(value == 7 && source == "next()" && evaluations == 1)
    precondition(UsersRoute.path == "/users")
    precondition(replaced() == 0)
    print(value, source, evaluations, UsersRoute.path, replaced())
}

probe()
```

It prints `7 next() 1 /users 0`. The counter proves one runtime evaluation in this case. The source spelling is a string, not a second invocation. The deliberately unresolved name in `replaced` disappears with the old body; `return 0` is what the compiler checks. Changing that function's return type to String fails because the generated Int is incompatible.

This is an **intentionally rejected** client, separate from the package's passing client:

```swift
import InterviewMacros
@RoutePath("users") struct Users {}
```

Its diagnostic asks for a noninterpolated literal starting with `/`. Variables, interpolation, and application to a class are also rejected by the implementation. These are syntax-policy diagnostics, not a runtime route validator.

This second client is **intentionally rejected** by type checking the generated body:

```swift
import InterviewMacros
@ReplaceWithZero func wrong() -> String { "original" }
```

The checked negative cases also cover a duplicate member, an incompatible expression result type, a MainActor call from a nonisolated function, a response type that violates the endpoint protocol, inaccessible generated members across modules, and a missing plugin. The error's origin matters: distinguish a plugin diagnostic from a compiler error in the generated program or a plugin-loading failure.

## In practice

**Choose it when:** Repeated syntax warrants a maintained generation boundary with explicit supported inputs.

**Investigate by:** Inspect macro expansion, compile clients across boundaries, and run side-effect-count tests.

**Strong evidence:** Justify the macro against ordinary code and maintain supported host, client, and SwiftSyntax/toolchain configurations.

### Macro test layers and debugging — optional

Use three complementary layers:

1. **Expansion and diagnostic tests:** inspect the syntax produced for supported input, failure messages, and source locations.
2. **Compiler integration tests:** compile real clients in the relevant modules, language/isolation settings, SDKs, and deployment targets. Include intentional failures.
3. **Behavior tests:** verify evaluation count, error propagation, generated state/ownership behavior, and the domain contract at runtime.

Place this in `Tests/MacroTests/MacroTests.swift`:

```swift
import XCTest
import SwiftSyntaxMacros
import SwiftSyntaxMacrosTestSupport
import InterviewPlugin

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
            diagnostics: [DiagnosticSpec(
                message: "RoutePath requires a noninterpolated string literal starting with /",
                line: 1, column: 1
            )],
            macros: ["RoutePath": RoutePathMacro.self]
        )
    }
}
```

Run `swift test --filter MacroTests` from the temporary package. A passing syntax snapshot does not type-check its output as an app. The separate client checks are what establish the sample's compiled behavior. Diagnostic wording is part of the test contract here; maintain useful errors when upgrading syntax libraries.

In Xcode, inspect the macro expansion at the invocation. For the checked compiler, `swiftc -Xfrontend -dump-macro-expansions` also shows expansions when the command includes the client imports and host plugin configuration. It is a compiler diagnostic option, not a permanent production build setting. F2 covers reducing compiler failures and recording exact invocations.

This chapter's package passes three expansion/diagnostic tests. Additional temporary checks compile the public API and three client sources for macOS 15 and iOS 18 deployment, and for macOS with MainActor default plus NonisolatedNonsendingByDefault. The clients execute on the macOS 27 host. Extra cases exercise escaped/raw literals, nested/generic structs, and a throwing argument evaluated once. The validation record separates these checks from untested older-OS, iOS-runtime, and cross-platform behavior.

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
