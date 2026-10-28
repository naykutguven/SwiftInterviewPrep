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

**Learning goals:**

- Separate Swift source names from encoded keys, enum values, nesting, and encoder strategies.
- State missing/null/type/version precedence rules and test each input independently.
- Evaluate compatibility in each reader/writer direction and distinguish unknown enum strings from lossless future documents.

**Prerequisites:** [[A4 — Optionals and Nil Modeling]] for absence, [[A13 — Error handling and typed throws]] for errors, [[B7 — Synthesized conformances and semantic correctness]] for synthesis, and [[A9 — Enums, Associated Values, Recursive Enums, and State Modeling]] for enum alternatives. Examples use Foundation JSON encoding/decoding.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Renaming a stored property can accidentally rename a JSON key that an old app still reads. Renaming an enum case can independently change its encoded string. Once data is persisted or shared, keys, cases, nesting, null handling, and encoder strategies form a **schema**, the contract between writers and readers.

Codable combines Encodable and Decodable operations; JSONEncoder and JSONDecoder choose a concrete format. Synthesis follows the type and CodingKeys, not the app’s migration policy. Define compatibility in a direction: “new reader accepts old writer” is different from “old reader accepts new writer,” and both differ from safely rewriting a future document. [SE-0166](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### A source rename can preserve the wire spelling

CodingKeys maps the Swift property to the old key; explicit enum raw values preserve old case strings.

```swift
import Foundation

enum SortMode: String, Codable {
    case byName = "name"
    case byDate = "date"
}
struct Settings: Codable {
    let sortMode: SortMode
    enum CodingKeys: String, CodingKey { case sortMode = "sortOrder" }
}
let oldData = Data(#"{"sortOrder":"name"}"#.utf8)
let settings = try JSONDecoder().decode(Settings.self, from: oldData)
precondition(settings.sortMode == .byName)
let encoded = try JSONEncoder().encode(settings)
let object = try JSONSerialization.jsonObject(with: encoded) as! [String: String]
precondition(object == ["sortOrder": "name"])
```

**Walkthrough:** The new Swift names coexist with the old schema in both encoding and decoding. Changing a source name need not change a wire contract; making a new schema is a separate decision.

### Encoder strategies are part of the format contract

The same model can encode its Date in different representations. A matching decoder strategy is required.

```swift
import Foundation

struct Event: Codable { let createdAt: Date }
let event = Event(createdAt: Date(timeIntervalSince1970: 0))
let encoder = JSONEncoder()
encoder.dateEncodingStrategy = .iso8601
let bytes = try encoder.encode(event)
let object = try JSONSerialization.jsonObject(with: bytes) as! [String: Any]
precondition(object["createdAt"] is String)
let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .iso8601
let decoded = try decoder.decode(Event.self, from: bytes)
precondition(decoded.createdAt == event.createdAt)
```

**Walkthrough:** Date’s type alone does not specify this JSON representation. The explicitly paired strategies establish the format used by this boundary; date, key, and data strategies should be reviewed with schema changes.

## Rules and pitfalls

### decodeIfPresent merges null with missing

**Common pitfall:** Falling back to the legacy key when the preferred current key is explicitly null, despite a policy that null is invalid.

**Mistake:** Standalone runnable example.

```swift
import Foundation

struct Profile: Decodable {
    let name: String
    enum CodingKeys: String, CodingKey { case name, oldName }
    init(from decoder: any Decoder) throws {
        let fields = try decoder.container(keyedBy: CodingKeys.self)
        name =
            try fields.decodeIfPresent(String.self, forKey: .name)
            ?? fields.decodeIfPresent(String.self, forKey: .oldName) ?? "Guest"
    }
}
let data = Data(#"{"name":null,"oldName":"old"}"#.utf8)
let profile = try JSONDecoder().decode(Profile.self, from: data)
precondition(profile.name == "old")
```

**Consequence:** The preferred key’s invalid null is silently replaced by old content.

**Why:** decodeIfPresent returns nil for either missing or null, while a wrong type still throws. Coalescing therefore encodes a specific fallback policy.

**Correction:** Test key presence before strict decoding when presence wins even for invalid input.

```swift
import Foundation

struct Profile: Decodable {
    let name: String
    enum CodingKeys: String, CodingKey { case name, oldName }
    init(from decoder: any Decoder) throws {
        let fields = try decoder.container(keyedBy: CodingKeys.self)
        if fields.contains(.name) {
            name = try fields.decode(String.self, forKey: .name)
        } else if fields.contains(.oldName) {
            name = try fields.decode(String.self, forKey: .oldName)
        } else {
            name = "Guest"
        }
    }
}
for json in [#"{"name":null,"oldName":"old"}"#, #"{"name":7,"oldName":"old"}"#] {
    do {
        _ = try JSONDecoder().decode(Profile.self, from: Data(json.utf8))
        preconditionFailure()
    } catch is DecodingError {}
}
```

**What changes:** Present current input is authoritative and must satisfy the current type. The absence-only default and legacy fallback are now separate from malformed-current recovery.

### Successful decoding does not imply safe rewriting

**Common pitfall:** Reading future data into a known-field model and encoding it again.

**Mistake:** Standalone runnable example.

```swift
import Foundation

struct Document: Codable { let name: String }
let future = Data(#"{"schemaVersion":2,"name":"Ada","futureOnly":true}"#.utf8)
let known = try JSONDecoder().decode(Document.self, from: future)
let rewritten = try JSONEncoder().encode(known)
let fields = try JSONSerialization.jsonObject(with: rewritten) as! [String: Any]
precondition(fields["futureOnly"] == nil && fields["schemaVersion"] == nil)
```

**Consequence:** The round trip drops data the old model never represented.

**Why:** Ignoring unknown keys is a reader policy, not a preservation mechanism.

**Correction:** Refuse unsupported rewrites unless unknown content and its semantics are preserved. This correction accepts only known version-1 fields.

```swift
import Foundation

struct AnyKey: CodingKey {
    let stringValue: String
    var intValue: Int? { nil }
    init?(stringValue: String) { self.stringValue = stringValue }
    init?(intValue: Int) { return nil }
}
enum SchemaError: Error { case unsupportedVersion, unknownFields }
struct Document: Codable {
    let schemaVersion: Int
    let name: String
    enum CodingKeys: String, CodingKey { case schemaVersion, name }
    init(from decoder: any Decoder) throws {
        let fields = try decoder.container(keyedBy: CodingKeys.self)
        schemaVersion =
            fields.contains(.schemaVersion)
            ? try fields.decode(Int.self, forKey: .schemaVersion) : 1
        guard schemaVersion == 1 else { throw SchemaError.unsupportedVersion }
        let all = try decoder.container(keyedBy: AnyKey.self)
        guard Set(all.allKeys.map(\.stringValue)).isSubset(of: ["schemaVersion", "name"])
        else { throw SchemaError.unknownFields }
        name = try fields.decode(String.self, forKey: .name)
    }
}
for json in [
    #"{"schemaVersion":2,"name":"Ada"}"#,
    #"{"schemaVersion":1,"name":"Ada","futureOnly":true}"#,
] {
    do {
        _ = try JSONDecoder().decode(Document.self, from: Data(json.utf8))
        preconditionFailure()
    } catch { precondition(error is SchemaError) }
}
let accepted = try JSONDecoder().decode(Document.self, from: Data(#"{"name":"Ada"}"#.utf8))
let encoded = try JSONEncoder().encode(accepted)
let restored = try JSONDecoder().decode(Document.self, from: encoded)
precondition(restored.name == "Ada" && restored.schemaVersion == 1)
```

**What changes:** Only documents whose supported fields are fully represented become rewritable values. A different design could retain a complete JSON tree, but preserving bytes alone still does not prove new semantics can be safely edited.

## Worked example

### Key rename and enum rename fail independently

**Setup:** Standalone decoder. Assertions classify semantic failures rather than depending on full diagnostic strings.

**Predict:** What happens after fixing only the key?

```swift
import Foundation

enum SortMode: String, Codable { case byName, byDate }
struct Settings: Codable { let sortMode: SortMode }
func outcome(_ json: String) -> String {
    do {
        _ = try JSONDecoder().decode(Settings.self, from: Data(json.utf8))
        return "decoded"
    } catch DecodingError.keyNotFound(let key, _) {
        return "missing:\(key.stringValue)"
    } catch DecodingError.dataCorrupted { return "invalid enum" } catch { return "other failure" }
}
precondition(outcome(#"{"sortOrder":"name"}"#) == "missing:sortMode")
precondition(outcome(#"{"sortMode":"name"}"#) == "invalid enum")
precondition(outcome(#"{"sortMode":"byName"}"#) == "decoded")
```

**Outcome:** The old payload first lacks sortMode; changing only the key then exposes the unsupported name string; the current pair succeeds.

**Why:** The schema has two independent spelling decisions. A reader migration must address both or deliberately preserve their historical encodings.

## Interview questions

### What breaks independently when sortOrder/name becomes sortMode/byName?

**Answer:** A Swift property rename affects source callers, a wire-key rename affects persisted structure, and an enum-spelling rename affects encoded values. CodingKeys and explicit raw-value coding can preserve the old wire contract while Swift names change.

**Example:** Standalone runnable example.

```swift
import Foundation

enum SortMode: String, Codable {
    case byName = "name"
    case byDate = "date"
}
struct Settings: Codable {
    let sortMode: SortMode
    enum CodingKeys: String, CodingKey { case sortMode = "sortOrder" }
}
let old = Data(#"{"sortOrder":"name"}"#.utf8)
let value = try JSONDecoder().decode(Settings.self, from: old)
precondition(value.sortMode == .byName)
let encoded = try JSONEncoder().encode(value)
let object = try JSONSerialization.jsonObject(with: encoded) as! [String: String]
precondition(object == ["sortOrder": "name"])
```

**Walkthrough:** The source API uses sortMode/byName, while the JSON still uses sortOrder/name. The worked example demonstrates the two failures when these mappings are absent.

**Boundary:** This preserves this wire spelling in both directions; it does not preserve an old Swift source property name or prove unrelated schema changes safe.

### When should absence default, and when should null, a wrong type, an unknown enum, or version fail?

**Answer:** Choose a domain policy explicitly. This reader defaults only absent fields, rejects present null/wrong types, rejects unknown enum values, and refuses unsupported versions. Other policies are possible, but they must not arise accidentally from coalescing.

**Example:** Standalone runnable example.

```swift
import Foundation

enum Mode: String, Decodable { case byName, byDate }
enum SchemaError: Error { case unsupported }
struct Settings: Decodable {
    let name: String
    let mode: Mode
    enum CodingKeys: String, CodingKey { case schemaVersion, name, mode }
    init(from decoder: any Decoder) throws {
        let fields = try decoder.container(keyedBy: CodingKeys.self)
        let version =
            fields.contains(.schemaVersion)
            ? try fields.decode(Int.self, forKey: .schemaVersion) : 1
        guard version == 1 else { throw SchemaError.unsupported }
        name = fields.contains(.name) ? try fields.decode(String.self, forKey: .name) : "Guest"
        mode = fields.contains(.mode) ? try fields.decode(Mode.self, forKey: .mode) : .byName
    }
}
func outcome(_ json: String) -> String {
    do {
        _ = try JSONDecoder().decode(Settings.self, from: Data(json.utf8))
        return "accepted"
    } catch DecodingError.valueNotFound { return "null" } catch DecodingError.typeMismatch {
        return "type"
    } catch DecodingError.dataCorrupted { return "enum" } catch is SchemaError {
        return "version"
    } catch { return "other" }
}
precondition(outcome("{}") == "accepted")
precondition(outcome(#"{"name":null}"#) == "null")
precondition(outcome(#"{"name":7}"#) == "type")
precondition(outcome(#"{"mode":"future"}"#) == "enum")
precondition(outcome(#"{"schemaVersion":2}"#) == "version")
```

**Walkthrough:** Each input gets the policy for its category. contains distinguishes absence from a present invalid value; decoding enforces types and known enum strings.

**Boundary:** This is a reading policy, not an unknown-field round-trip guarantee. If null means “clear” in the domain, model that state explicitly rather than silently substituting the absence default.

### Can a reader safely rewrite future data just because it preserves unknown enum strings?

**Answer:** No. Preserving one enum string does not preserve unknown JSON fields or their semantics. A reader must either maintain a documented complete round-trip representation or refuse unsupported rewrites.

**Example:** Standalone runnable example.

```swift
import Foundation

enum Mode: Equatable, Codable {
    case byName
    case unknown(String)
    init(from decoder: any Decoder) throws {
        let raw = try decoder.singleValueContainer().decode(String.self)
        self = raw == "byName" ? .byName : .unknown(raw)
    }
    func encode(to encoder: any Encoder) throws {
        var field = encoder.singleValueContainer()
        switch self {
        case .byName: try field.encode("byName")
        case .unknown(let raw): try field.encode(raw)
        }
    }
}
struct Document: Codable { let mode: Mode }
let data = Data(#"{"mode":"future","serverFlag":true}"#.utf8)
let document = try JSONDecoder().decode(Document.self, from: data)
precondition(document.mode == .unknown("future"))
let encoded = try JSONEncoder().encode(document)
let fields = try JSONSerialization.jsonObject(with: encoded) as! [String: Any]
precondition(fields["mode"] as? String == "future")
precondition(fields["serverFlag"] == nil)
```

**Walkthrough:** The enum string survives while serverFlag disappears. The version/unknown-field gate in the pitfall is a safe refusal alternative for a model that cannot preserve such documents.

**Boundary:** Even retaining all JSON members does not authorize edits whose newer semantics the old reader cannot understand. Define which versions are editable, read-only, or unsupported.

### Extension: why is an associated-value unknown case not a synthesized string catch-all?

**Answer:** Associated-value enum synthesis has a structured case/payload representation. Naming a case unknown does not teach the decoder to accept arbitrary strings; custom single-value decoding is required for that wire schema.

**Example:** Standalone runnable example.

```swift
import Foundation

enum RawMode: String, Codable { case byName, byDate }
enum AssociatedMode: Codable {
    case byName
    case unknown(String)
}
let rawData = try JSONEncoder().encode(RawMode.byName)
let rawString = try JSONDecoder().decode(String.self, from: rawData)
precondition(rawString == "byName")
let data = try JSONEncoder().encode(AssociatedMode.unknown("manual"))
let object = try JSONSerialization.jsonObject(with: data) as! [String: [String: String]]
precondition(object == ["unknown": ["_0": "manual"]])
do {
    _ = try JSONDecoder().decode(AssociatedMode.self, from: Data(#""manual""#.utf8))
    preconditionFailure()
} catch is DecodingError {}
```

**Walkthrough:** RawMode uses a JSON string. AssociatedMode encodes a case object containing the associated payload, so a plain manual string does not match it. [SE-0295](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0295-codable-synthesis-for-enums-with-associated-values.md)

**Boundary:** The previous answer demonstrates a deliberate custom string representation. Synthesis is convenient when its representation is the desired schema, not a substitute for specifying an external format.

### Extension: How can a lossy list skip invalid elements without stalling or hiding the loss?

**Answer:** Make discarding an explicit reading policy, consume one source element per iteration, and report the rejected positions. This example discards only DecodingError from an element; failures obtaining the next decoder and other application errors still propagate.

**Example:** Standalone Foundation example with JSONDecoder. superDecoder supplies the next element's decoder and advances the outer container before Element decoding is attempted.

```swift
import Foundation

struct LossyList<Element: Decodable>: Decodable {
    let values: [Element]
    let rejectedIndices: [Int]
    init(from decoder: any Decoder) throws {
        var container = try decoder.unkeyedContainer()
        var accepted: [Element] = []
        var rejected: [Int] = []
        while !container.isAtEnd {
            let sourceIndex = container.currentIndex
            let elementDecoder = try container.superDecoder()
            do { accepted.append(try Element(from: elementDecoder)) } catch is DecodingError {
                rejected.append(sourceIndex)
            }
        }
        values = accepted
        rejectedIndices = rejected
    }
}
let json = Data(#"[1,"bad",null,2,{},[9],3]"#.utf8)
let result = try JSONDecoder().decode(LossyList<Int>.self, from: json)
precondition(result.values == [1, 2, 3])
precondition(result.rejectedIndices == [1, 2, 4, 5])
let empty = try JSONDecoder().decode(LossyList<Int>.self, from: Data("[]".utf8))
precondition(empty.values.isEmpty && empty.rejectedIndices.isEmpty)
do {
    _ = try JSONDecoder().decode(LossyList<Int>.self, from: Data("{}".utf8))
    preconditionFailure()
} catch DecodingError.typeMismatch {}
```

**Walkthrough:** Each of the seven source positions is consumed once. Integers survive in order; the string, null, object, and nested array produce observable rejections. An empty list succeeds, while a wrong root shape still fails. The [JSONDecoder implementation](https://github.com/swiftlang/swift-foundation/blob/main/Sources/FoundationEssentials/JSON/JSONDecoder.swift) shows why advancing through superDecoder differs from retrying a failed decode at the same index.

**Boundary:** This deliberately loses invalid elements and does not preserve a document for rewriting. Use strict array decoding when partial acceptance is inappropriate. Reporting indices here is a minimal policy; a product may also need structured reasons, limits, and user-visible handling.

## Focused exercise

**Task:** Decode current and historical names with an explicit precedence and malformed-input policy.

**Acceptance checks:**

- Historical oldName is accepted; present name wins when both exist.
- Absence of both names defaults to Guest; present empty text remains present.
- Null/wrong-type current input fails even when a valid oldName exists.

**Solution:** Standalone Foundation decoder. This is a reader migration; it does not claim old-reader compatibility with new writes.

```swift
import Foundation

struct Profile: Decodable {
    let name: String
    enum CodingKeys: String, CodingKey { case name, oldName }
    init(from decoder: any Decoder) throws {
        let fields = try decoder.container(keyedBy: CodingKeys.self)
        if fields.contains(.name) {
            name = try fields.decode(String.self, forKey: .name)
        } else if fields.contains(.oldName) {
            name = try fields.decode(String.self, forKey: .oldName)
        } else {
            name = "Guest"
        }
    }
}
func read(_ json: String) throws -> String {
    try JSONDecoder().decode(Profile.self, from: Data(json.utf8)).name
}
let old = try read(#"{"oldName":"Ada"}"#)
let both = try read(#"{"oldName":"old","name":"new"}"#)
let missing = try read("{}")
let empty = try read(#"{"name":""}"#)
precondition(old == "Ada" && both == "new" && missing == "Guest" && empty == "")
for json in [
    #"{"name":null,"oldName":"old"}"#, #"{"name":7,"oldName":"old"}"#,
    #"{"oldName":null}"#, #"{"oldName":false}"#,
] {
    do {
        _ = try read(json)
        preconditionFailure("Expected rejection")
    } catch is DecodingError {}
}
```

**Reasoning:** Presence is checked before strict decoding, so malformed preferred data cannot silently fall back. Add encoding/version/unknown-data handling only under an explicit rewrite policy; reading historical fixtures establishes just that compatibility direction.

## In practice

**Choose it when:** Treat persisted or shared data as a versioned contract, including app-local preferences.

**Investigate by:** Keep historical, malformed, and future fixtures, and label each reader/writer direction being tested.

**Strong evidence:** State which documents can be safely rewritten and prove both field/value migration and unknown-data handling under that policy.

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
- [Apple: decodeIfPresent](<https://developer.apple.com/documentation/swift/keyeddecodingcontainer/decodeifpresent(_:forkey:)>)
- [Apple: JSONDecoder](https://developer.apple.com/documentation/foundation/jsondecoder)
- [Apple: Key decoding strategies](https://developer.apple.com/documentation/foundation/jsondecoder/keydecodingstrategy-swift.enum)
- [JSONDecoder implementation](https://github.com/swiftlang/swift-foundation/blob/main/Sources/FoundationEssentials/JSON/JSONDecoder.swift)
