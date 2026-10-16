---
tags:
  - swift
  - ios
  - interview-prep
  - interop-modules-distribution
  - import-visibility
  - dependency-leakage
---

# E6 — Import Visibility and Dependency Leakage

**Goal:** Hide vendor types and failures behind a validated SDK-owned public contract.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Import access, member lookup, downstream module loading, and executable linkage are distinct boundaries. Internal import can permit implementation use while preventing a vendor type from appearing in public signatures or visible bodies. A scalar result does not by itself hide dependencies embedded in typed errors, layouts, conformances, or serialized implementation. [SE-0409](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md)

## Rules and pitfalls

### Map behavior, not just names

Own the public value and error types and translate actual vendor fields/failures into them. Validate domain invariants and size limits before exposing an outcome. Explicit import access communicates the intended ceiling.

**Common pitfall:** Changing internal import to public only to satisfy a public vendor return commits clients to that dependency. Prefer an owned adapter when vendor exposure is not the SDK's contract.

### Verify loading separately from linkage

Resilient ordinary clients may compile without a hidden dependency module in their search path, while still needing its executable code at link/runtime. Nonresilient layouts, package access, and @testable can change module-loading requirements.

**Common pitfall:** A transitive extension lookup succeeding does not prove public import re-exports every vendor name. InternalImportsByDefault and MemberImportVisibility control different rules; test them explicitly.

## Interview questions

### Why does an internal import reject a public vendor error/result while still permitting ordinary implementation use?

The import's access ceiling permits ordinary implementation calls but not a more visible public signature that names the imported result or typed error. Both success and throws types are API dependencies. Map them to owned types or intentionally publish the dependency with a supported compatibility promise.

### Why can a transitive extension compile without proving that public import re-exports all vendor names?

Member lookup can find visible extension members through imported types under the compiler's lookup rules without making every module-level vendor name directly available. Public import is an exposure policy, not a blanket namespace re-export. Test source naming and extension visibility separately, including MemberImportVisibility.

### Which clients can omit the vendor module, and why does that not remove the vendor link requirement?

The checked resilient ordinary binary/textual-interface consumer can omit the hidden module when no other public path requires it. Nonresilient, testing, and package contexts can differ. The SDK implementation still calls vendor code, so final linkage/deployment must supply that code even when source import is unnecessary.

## Focused exercise

**Task:** Hide a vendor result behind an SDK-owned value and error. Verify the public signature and a separate consumer under declared build settings.

**Acceptance checks:**

- Valid and malformed inputs map to SDK-owned fields/errors.
- The public interface names no hidden vendor types or typed errors.
- An ordinary resilient client compiles without the vendor module but still requires the vendor object at link time.

**Solution:**

Use this real temporary vendor module and its intentional leaking control, then the SDK-owned adapter/client. The fixtures state their search paths and separate compile from link checks.

ThirdParty.swift is a complete local vendor fixture. Its small text grammar is exactly two pipe-separated fields; it is not a networking client or a third-party package download.

```swift
public enum VendorFailure: Error { case malformed }
public struct VendorRecord: Sendable {
    public let key: String
    public let label: String
    public init(key: String, label: String) { self.key = key; self.label = label }
}
public struct VendorDecoder: Sendable {
    public init() {}
    public func decode(_ raw: String) throws(VendorFailure) -> VendorRecord {
        let fields = raw.split(separator: "|", omittingEmptySubsequences: false)
        guard fields.count == 2 else { throw .malformed }
        return VendorRecord(key: String(fields[0]), label: String(fields[1]))
    }
}
extension String {
    public var vendorByteCount: Int { utf8.count }
}
```

LeakySDK.swift compiles because the vendor is intentionally public. Both the result type and typed thrown error belong to ThirdParty:

```swift
public import ThirdParty

public struct RawParser {
    public init() {}
    public func parse(_ raw: String) throws(VendorFailure) -> VendorRecord {
        try VendorDecoder().decode(raw)
    }
}
```

BadSDK.swift is an independent expected failure:

```swift
internal import ThirdParty

public func expose(_ raw: String) throws(VendorFailure) -> VendorRecord {
    try VendorDecoder().decode(raw)
}
```

The checked diagnostic identifies the typed thrown error as an internal-imported type in a public declaration. A separate result-only variant likewise rejects public vendor types. This is current Swift 6.4 evidence; the old chapter's Swift 6.2.1/Linux output has been replaced.

InlineBad.swift is another independent expected failure:

```swift
internal import ThirdParty

@inlinable public func byteCount(_ raw: String) -> Int {
    raw.vendorByteCount
}
```

Even though its signature uses only String and Int, the inlinable body references a property imported as internal. Changing the import to public makes the probe compile, but deliberately makes the dependency available to the serialized body. Do not make that change automatically just to silence the error; E8 covers definition visibility and inlining tradeoffs.

Further controls reject a public vendor typealias, a vendor payload in a public enum, and a private vendor field in a frozen public struct through an internal import. The ordinary nonfrozen ProfileParser below can store its private vendor decoder; its consumer-loading behavior still depends on library evolution.

Redesign RawParser so ordinary callers receive domain-owned values and errors. Preserve actual input-dependent behavior, validate the domain rules, and prove where the vendor module and vendor code remain necessary. The goal is a useful semantic boundary rather than a wrapper that merely renames VendorRecord.

ProfileSDK.swift:

```swift
internal import ThirdParty

public enum ProfileError: Error, Equatable {
    case oversizedPayload, malformedPayload, invalidID, invalidName
}
public struct Profile: Sendable, Equatable {
    public let id: String
    public let displayName: String
    public init(id: String, displayName: String) throws(ProfileError) {
        guard !id.isEmpty else { throw .invalidID }
        guard !displayName.isEmpty else { throw .invalidName }
        self.id = id
        self.displayName = displayName
    }
}
public struct ProfileParser: Sendable {
    private let decoder = VendorDecoder()
    public init() {}
    public func parse(_ raw: String) throws(ProfileError) -> Profile {
        guard raw.utf8.count <= 4096 else { throw .oversizedPayload }
        let record: VendorRecord
        do { record = try decoder.decode(raw) }
        catch { throw .malformedPayload }
        return try Profile(id: record.key, displayName: record.label)
    }
}
```

This adapter checks a UTF-8 size limit, translates malformed vendor input, maps actual fields, and lets Profile enforce nonempty ID/name. It does not trim whitespace, interpret escaping, or claim a full production profile schema. A larger application should specify those policies explicitly. The limit constrains accepted payloads; the input String already exists, and counting its UTF-8 bytes is work.

Client.swift is an ordinary external consumer. It uses no @testable import and names no vendor type:

```swift
import ProfileSDK

@main struct Client {
    static func main() throws {
        let parser = ProfileParser()
        let ada = try parser.parse("ada|Ada")
        let expectedAda = try Profile(id: "ada", displayName: "Ada")
        precondition(ada == expectedAda)
        let emoji = try parser.parse("emoji|🦊")
        precondition(emoji.displayName == "🦊")
        for (input, expected) in [
            ("missing-separator", ProfileError.malformedPayload),
            ("a|b|c", .malformedPayload), ("|Ada", .invalidID), ("ada|", .invalidName),
            (String(repeating: "x", count: 4097), .oversizedPayload)
        ] {
            do { _ = try parser.parse(input); preconditionFailure("Expected failure") }
            catch { precondition(error == expected) }
        }
        let limit = "a|" + String(repeating: "x", count: 4094)
        let atLimit = try parser.parse(limit)
        precondition(atLimit.displayName.utf8.count == 4094)
        print("ProfileSDK public consumer passed")
    }
}
```

All files belong in a temporary directory. A minimal host-build recipe with the selected Swift 6.4 toolchain is:

```sh
mkdir -p deps public objects
swiftc -swift-version 6 -enable-upcoming-feature NonisolatedNonsendingByDefault -enable-upcoming-feature InferIsolatedConformances -default-isolation nonisolated -parse-as-library -enable-library-evolution -emit-module -c -module-name ThirdParty ThirdParty.swift -emit-module-path deps/ThirdParty.swiftmodule -o objects/ThirdParty.o
swiftc -swift-version 6 -enable-upcoming-feature NonisolatedNonsendingByDefault -enable-upcoming-feature InferIsolatedConformances -default-isolation nonisolated -parse-as-library -enable-library-evolution -I deps -emit-module -c -module-name ProfileSDK ProfileSDK.swift -emit-module-path public/ProfileSDK.swiftmodule -emit-module-interface-path public/ProfileSDK.swiftinterface -o objects/ProfileSDK.o
swiftc -swift-version 6 -enable-upcoming-feature NonisolatedNonsendingByDefault -enable-upcoming-feature InferIsolatedConformances -default-isolation nonisolated -parse-as-library -I public -c Client.swift -o objects/Client.o
swiftc objects/Client.o objects/ProfileSDK.o objects/ThirdParty.o -o client
./client
```

The public directory contains only ProfileSDK artifacts, so the client compile cannot resolve ThirdParty from that search path. The final link deliberately supplies all three objects. For the checked macOS 13/iOS 16 configuration, add the explicit SDK, target, and module-cache options recorded in the validation record. Enable testing on the SDK only when reproducing the separate @testable control.

Acceptance criteria:

1. Valid records and Unicode names map to the actual SDK-owned Profile fields.
2. Missing/extra separators, empty ID/name, and oversized input map to the documented errors; exactly 4096 UTF-8 bytes are accepted by the tested input.
3. ProfileSDK's emitted public interface contains neither ThirdParty imports nor vendor type names. LeakySDK's interface contains both.
4. Ordinary clients compile with only the resilient SDK binary module or textual interface available; the non-resilient and @testable controls fail without the vendor module.
5. Omitting the vendor object causes the intended undefined-symbol link failure; supplying it runs the consumer successfully.
6. Keep import defaults, member-visibility settings, and module selectors distinct when explaining the results.

The earlier validation record checked explicitly annotated source variants for macOS 13, iOS 16, and a MainActor-default comparison. The revised unannotated source here uses the shared nonisolated baseline. The two macOS configurations run the consumer on macOS 27. The focused access/loading/feature controls are identified separately in the validation record; iOS evidence is compilation only.

**Reasoning:** A semantic adapter preserves useful behavior while localizing dependency evolution. Hidden source types do not remove implementation code dependencies. Check the exact module-wide import policy and release model rather than generalizing one successful client compile.

## In practice

**Choose it when:** Keep a vendor replaceable behind a domain contract callers can understand.

**Investigate by:** Inspect textual interfaces and compile ordinary, nonresilient, package, and testable controls independently.

**Strong evidence:** Distinguish access, module loading, member visibility, and linking with positive and intentional negative fixtures.

## References

### Related guides

- [[A11 — Access Control]]
- [[A13 — Error handling and typed throws]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[D9 — Continuations and Bridging Callback APIs]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E7 — Resilience, ABI, and Module Stability]]
- [[E8 — Binary frameworks, inlineability, and public implementation detail leakage]]
- [[F5 — Language mode, build settings, and migration flags]]

### Official sources

- [SE-0409](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md)
- [SE-0444](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0444-member-import-visibility.md)
- [SE-0491](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0491-module-selectors.md)
- [Implementation-only import diagnostic](https://docs.swift.org/latest/documentation/diagnostics/implementation-only-deprecated/)
