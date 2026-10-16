---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - codable
  - schema-evolution
---

# B9 — Codable and Schema Evolution

**Goal:** Decode old and current keys under an explicit precedence and malformed-input policy.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Codable describes encoding and decoding operations; JSONEncoder and JSONDecoder determine the concrete format. Once shared or persisted, key spellings, enum cases, nesting, null behavior, and strategies form a schema. Synthesis follows structure and coding keys, not the application's compatibility or migration policy. [SE-0166](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md)

## Rules and pitfalls

### Decide precedence before writing a migration reader

For a rename, keep the old wire key with CodingKeys when the schema should remain stable. If accepting two keys, state which wins when both are present and whether malformed preferred input may fall back. Test missing, null, wrong-type, and unsupported-version cases separately.

**Common pitfall:** decodeIfPresent merges missing and null but still throws on an incompatible type. A stored default is not a general synthesized missing-key policy. Use contains and decodeNil when the distinction matters.

### Test compatibility in both directions

New-reader/old-writer success says nothing about old-reader/new-writer success. Decoding unknown enum strings and preserving unknown JSON fields are separate capabilities.

**Common pitfall:** Reading a newer document and writing only known fields can destroy future data. Refuse unsupported rewrites or preserve unknown content under a documented round-trip policy; do not infer this from successful decoding.

## Interview questions

### What breaks independently when sortOrder/name becomes sortMode/byName?

Renaming a Swift property can be source API change; renaming its wire key changes schema; renaming an enum case can change its encoded spelling. These can break independently. Stable CodingKeys and explicit enum coding preserve wire compatibility while allowing a deliberate source migration.

### When should absence default, and when should null, a wrong type, an unknown enum value, or an unsupported schema version fail?

Default absence only when the domain permits it. Null may mean clear or invalid, a wrong type should normally fail, and unknown enum values need explicit preservation or rejection. Unsupported schema versions should fail or enter a safe read-only path rather than silently downgrade data. Recovery policy belongs to the boundary.

### Can this reader safely rewrite data from a newer writer without dropping unknown fields?

Only if it preserves the newer data's required unknown information and semantics. A reader that ignores unknown fields can load but lose them on rewrite. Test round trips with future fixtures or prohibit rewriting unsupported versions; retaining one unknown enum string alone is insufficient.

## Focused exercise

**Task:** Decode old and current field names under an explicit precedence rule. Check missing, null, and malformed values separately.

**Acceptance checks:**

- Historical keys decode and the current key wins when both exist.
- Missing fields default according to policy.
- Null and wrong-type current values reject instead of silently using the old key.

**Solution:**

Standalone Foundation decoder. Current name takes precedence whenever present; explicit null and wrong types fail. Missing both names uses Guest.

```swift
import Foundation
struct Profile: Decodable {
    let name: String
    enum CodingKeys: String, CodingKey { case name, oldName }
    init(from decoder: any Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        if c.contains(.name) { name = try c.decode(String.self, forKey: .name) }
        else if c.contains(.oldName) { name = try c.decode(String.self, forKey: .oldName) }
        else { name = "Guest" }
    }
}
func read(_ json: String) throws -> String {
    try JSONDecoder().decode(Profile.self, from: Data(json.utf8)).name
}
func probe() throws {
    let old = try read(#"{"oldName":"Ada"}"#)
    let both = try read(#"{"oldName":"old","name":"new"}"#)
    let missing = try read("{}")
    precondition(old == "Ada" && both == "new" && missing == "Guest")
    for raw in [#"{"name":null}"#, #"{"name":7}"#] {
        do { _ = try read(raw); preconditionFailure("expected decoding failure") }
        catch is DecodingError { }
    }
}
try probe()
```

**Reasoning:** contains distinguishes presence before decode enforces a non-null String. This is a reader migration, not proof of old-reader compatibility or lossless future-document rewriting. Add encoding and version policy only when those are actual requirements.

## In practice

**Choose it when:** Treat persisted data as a versioned contract, even for app-local preferences.

**Investigate by:** Keep historical, malformed, and future fixtures with explicit reader/writer directions.

**Strong evidence:** State which documents can be safely rewritten and verify unknown-data handling.

### Enum representation boundaries — optional

#### Enum representation is part of the contract

```swift
enum RawMode: String, Codable {
    case byName, byDate
}

enum AssociatedMode: Codable {
    case byName
    case unknown(String)
}
```

`RawMode.byName` encodes as the JSON string `"byName"`; unknown strings fail decoding. `AssociatedMode.unknown("manual")` uses a nested object such as `{"unknown":{"_0":"manual"}}`. Adding that case does not synthesize a catch-all for arbitrary strings. Associated-value enum synthesis dates to Swift 5.5; use a custom implementation when an external schema requires a different shape. [SE-0295](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0295-codable-synthesis-for-enums-with-associated-values.md)

## References

### Related guides

- [[B7 — Synthesized conformances and semantic correctness]]
- [[B10 — API Design Guidelines and Swift-Native Surface Design]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[A9 — Enums, Associated Values, Recursive Enums, and State Modeling]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[F1 — Testing Strategy in Modern Swift]]

### Official sources

- [SE-0166](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md)
- [SE-0295](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0295-codable-synthesis-for-enums-with-associated-values.md)
- [Apple: Encoding and decoding custom types](https://developer.apple.com/documentation/foundation/encoding-and-decoding-custom-types)
- [Apple: decodeIfPresent](https://developer.apple.com/documentation/swift/keyeddecodingcontainer/decodeifpresent(_:forkey:)
- [Apple: JSONDecoder](https://developer.apple.com/documentation/foundation/jsondecoder)
- [Apple: Key decoding strategies](https://developer.apple.com/documentation/foundation/jsondecoder/keydecodingstrategy-swift.enum)
