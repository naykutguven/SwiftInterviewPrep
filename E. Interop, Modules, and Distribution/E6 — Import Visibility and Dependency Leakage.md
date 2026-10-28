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

**Learning goals:**

- Separate source import access, member lookup, downstream module loading, and final executable linkage.
- Map actual vendor fields/errors into an SDK-owned public contract and inspect its serialized interface.
- Check ordinary, textual-interface, nonresilient, testing, and import-feature clients under their own settings.

**Prerequisites:** [[A11 — Access Control]] for access, [[A13 — Error handling and typed throws]] for typed throws, [[B11 — Library Evolution and Resilience-Aware API Design]] for library evolution, and [[E5 — Modules, packages, targets, products, and resources]] for module/package identity. [[E7 — Resilience, ABI, and Module Stability]]/[[E8 — Binary frameworks, inlineability, and public implementation detail leakage]] explain distribution and client-visible bodies.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Your SDK uses a vendor decoder. Callers should receive a Profile and ProfileError owned by your SDK. Returning VendorRecord or throwing VendorFailure instead makes that vendor part of the public contract, even when the function looks like a small wrapper.

An import’s access level limits where imported declarations may be exposed. That is separate from which extension members lookup finds, whether a consumer must load a dependency module, and which object/library code the final executable needs. Keep those four questions distinct. [Import access levels](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0409-access-level-on-imports.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Compile a real vendor rather than inventing its types

Shared ThirdParty module. Its grammar is exactly two pipe-separated fields; the SDK will add nonempty/domain/size rules. This local fixture has no network or downloaded dependency.

Each library block below is compiled independently with the baseline plus -parse-as-library -O -enable-library-evolution -emit-module -c. Give it the stated module name, separate output directory, and emit a .swiftinterface. Add -I deps for modules importing ThirdParty. The ordinary ProfileSDK consumer uses only -I public; all link objects are supplied separately. Complete commands appear in the exercise.

```swift
public enum VendorFailure: Error { case malformed }
public struct VendorRecord: Sendable {
    public let key: String
    public let label: String
    public init(key: String, label: String) {
        self.key = key
        self.label = label
    }
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

**Walkthrough:** VendorDecoder checks field count and returns actual input-dependent fields. String.vendorByteCount is a public extension used later to distinguish member lookup from top-level names. The vendor’s public types are not automatically your SDK’s public contract.

### Implementation use can remain internal

Compile-only ImplementationSDK source against ThirdParty. An internal import permits calling the decoder inside an ordinary body whose public result/error do not expose vendor types.

```swift
internal import ThirdParty

public func key(in raw: String) -> String? {
    do { return try VendorDecoder().decode(raw).key } catch { return nil }
}
```

**Walkthrough:** The returned String? is SDK-owned syntax, and the vendor error is handled locally. This simple optional intentionally merges malformed input with absence; the full domain adapter below retains more useful error distinctions.

## Rules and pitfalls

### A public typed error is a dependency too

**Common pitfall:** Treating only the return value as the public dependency while exposing a vendor typed error.

**Mistake:** Compile-only; intentionally rejected.

```swift
internal import ThirdParty

public func expose(_ raw: String) throws(VendorFailure) -> VendorRecord {
    try VendorDecoder().decode(raw)
}
```

**Consequence:** The public signature rejects the vendor error/result imported with internal visibility.

**Why:** Both success and throws types appear in the declaration clients compile against. An import access ceiling applies to those uses.

**Correction:** Own the public value/error and map real vendor behavior. Shared ProfileSDK source, built against ThirdParty but emitted into the public directory.

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
        do { record = try decoder.decode(raw) } catch { throw .malformedPayload }
        return try Profile(id: record.key, displayName: record.label)
    }
}
```

**What changes:** The adapter limits UTF-8 input to 4096 bytes, translates malformed grammar, maps actual fields, and validates nonempty ID/name. Its ordinary public interface hides vendor types. It does not trim whitespace or define escaping; those would be explicit schema choices.

### A scalar signature can still leak a client-visible dependency

**Common pitfall:** Using an internal-imported property from an @inlinable public body.

**Mistake:** Compile-only; intentionally rejected.

```swift
internal import ThirdParty

@inlinable public func byteCount(_ raw: String) -> Int {
    raw.vendorByteCount
}
```

**Consequence:** The compiler rejects the body’s vendor member despite String/Int signature types.

**Why:** Clients can compile an inlinable body, so its referenced declarations need appropriate visibility too.

**Correction:** Keep the implementation opaque when client optimization is not justified.

```swift
internal import ThirdParty

public func byteCount(_ raw: String) -> Int { raw.vendorByteCount }
```

**What changes:** The vendor call stays in the library implementation. A deliberate public import/body exposure is another supported policy, but it publishes that dependency and needs compatibility review.

### Hiding a module does not remove its code from the executable

**Common pitfall:** Treating a successfully compiled ordinary client without the vendor module as proof that the vendor is unnecessary at link time.

**Mistake:** After compiling Client.o and the two library objects from the exercise, this real link intentionally fails:

```sh
swiftc objects/Client.o objects/ProfileSDK.o -o client-missing-vendor
```

**Consequence:** Undefined vendor symbols prevent the executable from linking.

**Why:** ProfileSDK’s implementation still calls ThirdParty. Hiding source names/module-loading requirements does not implement or erase that code.

**Correction:** Supply the implementation code while keeping the client’s source search path limited:

```sh
swiftc objects/Client.o objects/ProfileSDK.o objects/ThirdParty.o -o client
./client
```

**What changes:** The ordinary client can compile without vendor declarations and link with vendor code. The validation checks the missing-object failure separately from the successful complete link; a dynamic library also needs runtime delivery/search-path checks.

## Worked example

### An ordinary client receives domain outcomes

**Setup:** Run against the compiled resilient ProfileSDK and ThirdParty objects. The compiler’s module search path contains public only, with no ThirdParty module.

**Predict:** Does rejecting malformed input require exposing VendorFailure?

```swift
import ProfileSDK

let parser = ProfileParser()
let ada = try parser.parse("ada|Ada")
precondition(ada.id == "ada" && ada.displayName == "Ada")
do {
    _ = try parser.parse("no separator")
    preconditionFailure()
} catch ProfileError.malformedPayload {}
do {
    _ = try parser.parse("|Ada")
    preconditionFailure()
} catch ProfileError.invalidID {}
```

**Outcome:** Success retains the actual fields and failures use ProfileError.

**Why:** The SDK supplies the semantic translation. Inspecting its interface and withholding the vendor module verifies the source boundary; supplying both objects verifies a different link boundary.

## Interview questions

### Why does an internal import reject a public vendor error/result while permitting ordinary implementation use?

**Answer:** Because the import’s access ceiling allows ordinary bodies but forbids more visible declarations from naming the imported contract. Map public values/errors or publish the dependency deliberately.

**Example:** Compile-only LeakySDK module against ThirdParty; emit its interface separately from ProfileSDK.

```swift
public import ThirdParty

public struct RawParser {
    public init() {}
    public func parse(_ raw: String) throws(VendorFailure) -> VendorRecord {
        try VendorDecoder().decode(raw)
    }
}
```

**Walkthrough:** This LeakySDK compiles with public import and explicitly exposes VendorRecord/VendorFailure in its interface. The ProfileSDK correction makes a different contract by translating both.

**Boundary:** Public import is appropriate when clients are intended to depend on the vendor. It is not a neutral fix for an SDK that promises vendor independence.

Independent **intentionally rejected** result-only, typealias, enum payload, and frozen-layout dependencies through an internal import:

```swift
internal import ThirdParty

public func record() -> VendorRecord { VendorRecord(key: "a", label: "A") }
```

```swift
internal import ThirdParty

public typealias Record = VendorRecord
```

```swift
internal import ThirdParty

public enum Outcome { case record(VendorRecord) }
```

```swift
internal import ThirdParty

@frozen public struct Frozen { private let decoder: VendorDecoder }
```

Shared LookupSDK source for the next question. This intentionally public import supports an exposure policy, not automatic re-export of every name:

```swift
public import ThirdParty

public func lookupMarker() -> VendorRecord { VendorRecord(key: "a", label: "A") }
```

### Why can a transitive extension compile without proving that public import re-exports all vendor names?

**Answer:** Member lookup and module-level names follow distinct historical rules. With this baseline, a transitive extension can be found without bringing all vendor names into scope; MemberImportVisibility makes the member import requirement explicit.

**Example:** Separate LookupSDK client; both module search directories and implementation objects are present. No direct ThirdParty import in this control.

```swift
import LookupSDK

precondition(lookupMarker().label == "A")
precondition("é".vendorByteCount == 2)
```

**Walkthrough:** This ordinary client calls the transitively loaded String extension under baseline lookup. A separate control adds MemberImportVisibility and rejects that extension call; directly importing ThirdParty restores it.

**Boundary:** Do not infer a blanket re-export or silently enable a module-wide flag. Protocol default-witness lookup has its own documented exception. [Member visibility](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0444-member-import-visibility.md)

The reliable direct-import/module-selector client is local to this answer:

```swift
import LookupSDK
import ThirdParty

let decoder = ThirdParty::VendorDecoder()
let record = try decoder.decode("a|Ada")
precondition(record.label == "Ada" && "é".vendorByteCount == 2)
```

### Which clients can omit the vendor module, and why does that not remove its link requirement?

**Answer:** This ordinary resilient binary/textual-interface client can omit the hidden module because no public surface needs it. Nonresilient layout, package, and @testable contexts can require more. Vendor code is still called by the SDK implementation.

**Example:** Ordinary resilient client with no testable import and no ThirdParty module search path.

```swift
import ProfileSDK

let profile = try ProfileParser().parse("emoji|🦊")
precondition(profile.id == "emoji" && profile.displayName == "🦊")
```

**Walkthrough:** The client imports only ProfileSDK, with its module path limited to public. Additional controls withhold the binary module to force textual import, compare nonresilient/testable loading, and deliberately omit the vendor object at link time.

**Boundary:** One successful ordinary client is not every serialization context. A compiler feature flag, package identity, library-evolution policy, or public conformance can change requirements.

### Extension: How does InternalImportsByDefault differ from MemberImportVisibility?

**Answer:** The first changes the access default of an unannotated import; the second changes what imported members are visible in a source file. Neither is a substitute for an explicit public dependency policy.

**Example:** Compile-only baseline declaration against ThirdParty; compare the flag as a separate control.

```swift
import ThirdParty

public func exposedRecord() -> VendorRecord { VendorRecord(key: "a", label: "A") }
```

**Walkthrough:** The baseline accepts this declaration under the ordinary import default. A separate same-source control enables InternalImportsByDefault and rejects the public vendor type. Explicit public import is the intentional exposure alternative.

**Boundary:** Declare module-wide migration settings and test both source access and member lookup. The explicit internal/public examples in this chapter make their policy independent of the default.

## Focused exercise

**Task:** Hide vendor fields/failures behind an SDK-owned value/error, then verify an ordinary external consumer.

**Acceptance checks:**

- Valid and Unicode records retain their actual fields; domain errors distinguish grammar, ID/name, and size failures.
- Public ProfileSDK interface contains no ThirdParty import or vendor type name.
- The client compiles without the vendor module; missing-object linkage fails and complete linkage runs.

**Solution:** Client.swift is an ordinary external client of the earlier ThirdParty/ProfileSDK sources. Add -parse-as-library for this @main form.

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
            (String(repeating: "x", count: 4097), .oversizedPayload),
        ] {
            do {
                _ = try parser.parse(input)
                preconditionFailure("Expected failure")
            } catch { precondition(error == expected) }
        }
        let limit = "a|" + String(repeating: "x", count: 4094)
        let atLimit = try parser.parse(limit)
        precondition(atLimit.displayName.utf8.count == 4094)
        print("ProfileSDK public consumer passed")
    }
}
```

**Reasoning:** The 4096-byte boundary is tested exactly, and malformed/extra/missing/empty fields map to declared domain errors. These checks verify behavior rather than a wrapper that merely renames types. The code remains a small text grammar, not a complete production profile format.

```sh
# Save the stated sources as ThirdParty.swift, ProfileSDK.swift, Client.swift.
mkdir -p deps public objects
# Add the shared baseline, SDK, target and module-cache flags to each compile.
swiftc -parse-as-library -enable-library-evolution -emit-module -c \
  -module-name ThirdParty ThirdParty.swift -emit-module-path deps/ThirdParty.swiftmodule \
  -o objects/ThirdParty.o
swiftc -parse-as-library -enable-library-evolution -I deps -emit-module -c \
  -module-name ProfileSDK ProfileSDK.swift -emit-module-path public/ProfileSDK.swiftmodule \
  -emit-module-interface-path public/ProfileSDK.swiftinterface -o objects/ProfileSDK.o
swiftc -parse-as-library -I public -c Client.swift -o objects/Client.o
swiftc objects/Client.o objects/ProfileSDK.o objects/ThirdParty.o -o client
./client
```

## In practice

**Choose it when:** Own the public semantic contract when vendor exposure is not a supported promise.

**Investigate by:** Inspect interfaces and compare ordinary, interface-only, nonresilient, package/testing, and feature-flag clients independently.

**Strong evidence:** Validate actual field/error mapping and separate source access, module loading and executable linkage.

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
