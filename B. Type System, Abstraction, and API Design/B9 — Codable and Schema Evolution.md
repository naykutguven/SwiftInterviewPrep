---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - codable
  - schema-evolution
---
## 0. Rubric snapshot

**Rubric expectation**

Know `Codable` synthesis, custom coding keys, lossy decoding tradeoffs, backward compatibility, and versioning concerns. The rubric specifically probes property renames, custom `init(from:)`, and evolving a persisted settings model without breaking older data.

**Caveats**

`Codable` makes simple serialization easy. It does **not** make schema evolution trivial. Once data is persisted, your encoded representation becomes a compatibility contract.

**You should be able to answer**

- What breaks when you rename a property in a persisted `Codable` model?
- When is it worth writing a custom `init(from:)`?
- When does synthesis stop being enough?

**You should be able to do**

- Evolve a persisted settings model by renaming a field.
- Add a new enum case without breaking old stored data.
- Decode legacy payloads safely.

---

## 1. Core mental model

`Codable` is Swift’s protocol-based serialization model. `Encodable` means a value can write itself into an external representation. `Decodable` means a value can initialize itself from an external representation. `Codable` is just `Encodable & Decodable`. The original Swift proposal defines `encode(to:)`, `init(from:)`, `CodingKey`, `Encoder`, and `Decoder` as the abstraction layer between Swift types and external formats. ([GitHub, "Swift Archival & Serialization"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md))

The important part: `Codable` does **not** persist your Swift type. It persists a representation of your type. By default, that representation is structurally derived from property names and enum raw values.

The key idea:

```text
Swift model != persisted schema

Your Swift type can evolve freely only if the encoded schema remains compatible,
or if you explicitly migrate old representations.
```

Automatic synthesis is convenient when the Swift shape and persisted shape match. SE-0166 says synthesis works when properties are encodable/decodable and when the `CodingKeys` map one-to-one to properties. If your model no longer maps one-to-one to stored data, you own the decoding logic. ([GitHub, "Swift Archival & Serialization"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md))

For persisted app settings, disk caches, user defaults, JSON files, app group storage, widgets, or synced blobs, treat the encoded keys and values as a schema. Renaming `sortOrder` to `sortMode` is not just a refactor. It changes what the decoder looks for.

---

## 2. Essential mechanics

### Synthesis uses property names unless you override `CodingKeys`

```swift
struct User: Codable {
    var displayName: String
}
```

By default, this expects encoded data shaped like:

```json
{
  "displayName": "Aykut"
}
```

If older data used `"name"` instead, synthesis fails unless you preserve the old key:

```swift
struct User: Codable {
    var displayName: String

    enum CodingKeys: String, CodingKey {
        case displayName = "name"
    }
}
```

Apple’s Codable documentation also describes the special nested `CodingKeys` enum as the customization point for choosing encoded and decoded properties. ([Apple Developer, "Encoding and Decoding Custom Types"](https://developer.apple.com/documentation/foundation/encoding-and-decoding-custom-types))

### `decode` means required; `decodeIfPresent` means optional/migratable

```swift
let value = try container.decode(String.self, forKey: .name)
```

This is strict. Missing key, `null`, or incompatible type can fail.

```swift
let value = try container.decodeIfPresent(String.self, forKey: .name) ?? "Default"
```

This is useful for migrations where old data may not contain a field.

Use this distinction deliberately:

```swift
struct Settings: Decodable {
    var showHints: Bool

    enum CodingKeys: String, CodingKey {
        case showHints
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        showHints = try container.decodeIfPresent(Bool.self, forKey: .showHints) ?? true
    }
}
```

### Raw-value enums are easy until unknown values appear

```swift
enum SortMode: String, Codable {
    case byName
    case byDate
}
```

This decodes `"byName"` and `"byDate"`. It fails on `"priority"`, `"manual"`, or any future value. SE-0166 explicitly supports trivial `Codable` conformance for primitive-backed `RawRepresentable` types, which is why this works. ([GitHub, "Swift Archival & Serialization"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md))

For remote data or persisted data that may be written by future app versions, consider a custom enum decoder:

```swift
enum SortMode: Codable, Equatable {
    case byName
    case byDate
    case byPriority
    case unknown(String)

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let rawValue = try container.decode(String.self)

        switch rawValue {
        case "byName", "name":
            self = .byName
        case "byDate", "date":
            self = .byDate
        case "byPriority", "priority":
            self = .byPriority
        default:
            self = .unknown(rawValue)
        }
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()

        switch self {
        case .byName:
            try container.encode("byName")
        case .byDate:
            try container.encode("byDate")
        case .byPriority:
            try container.encode("byPriority")
        case .unknown(let rawValue):
            try container.encode(rawValue)
        }
    }
}
```

---

## 3. Common traps and misconceptions

### Trap 1: Treating a property rename as a harmless refactor

Bad:

```swift
// v1 persisted this:
struct Settings: Codable {
    var sortOrder: SortMode
}

// v2 changes it:
struct Settings: Codable {
    var sortMode: SortMode
}
```

Old payload:

```json
{
  "sortOrder": "name"
}
```

The v2 synthesized decoder expects `sortMode`, so old data breaks.

Better:

```swift
struct Settings: Codable {
    var sortMode: SortMode

    private enum CodingKeys: String, CodingKey {
        case sortMode
        case sortOrder
    }

    init(sortMode: SortMode = .byName) {
        self.sortMode = sortMode
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)

        if let sortMode = try container.decodeIfPresent(SortMode.self, forKey: .sortMode) {
            self.sortMode = sortMode
        } else if let legacySortOrder = try container.decodeIfPresent(LegacySortOrder.self, forKey: .sortOrder) {
            self.sortMode = SortMode(legacySortOrder)
        } else {
            self.sortMode = .byName
        }
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(sortMode, forKey: .sortMode)
    }
}
```

### Trap 2: Adding non-optional stored properties and expecting old data to decode

Bad:

```swift
struct Settings: Codable {
    var sortMode: SortMode
    var theme: Theme
}
```

If old data lacks `theme`, synthesis fails.

Better:

```swift
struct Settings: Codable {
    var sortMode: SortMode
    var theme: Theme

    private enum CodingKeys: String, CodingKey {
        case sortMode
        case theme
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        sortMode = try container.decodeIfPresent(SortMode.self, forKey: .sortMode) ?? .byName
        theme = try container.decodeIfPresent(Theme.self, forKey: .theme) ?? .system
    }
}
```

### Trap 3: Lossy decoding without a product decision

Lossy decoding means “decode what you can, discard or default the rest.”

That is acceptable for:

```text
Search results
Telemetry
Feature flags with safe defaults
Remote content where partial display is better than failure
```

It is dangerous for:

```text
Persisted user-generated data
Financial data
Security-sensitive settings
Migration-critical local storage
```

A lossy migration can silently corrupt user state. Staff-level judgment is knowing when to fail loudly, when to repair, and when to preserve unknown data for round-tripping.

---

## 4. Direct answers to rubric questions

### Q1. What breaks when you rename a property in a persisted `Codable` model?

The synthesized decoder looks for the new property name, not the old persisted key. If the renamed property is non-optional and no custom key or migration path exists, decoding old data fails with `keyNotFound`.

Interview version:

> `Codable` synthesis derives keys from property names unless I define `CodingKeys`. Once data is persisted, those keys are part of the schema. Renaming a Swift property from `sortOrder` to `sortMode` makes the synthesized decoder look for `sortMode`, so older payloads containing `sortOrder` fail. For persisted data, I either keep the encoded key stable with `CodingKeys`, or write a custom decoder that accepts both old and new representations.

### Q2. When is it worth writing a custom `init(from:)`?

Write a custom `init(from:)` when the stored schema and Swift model are no longer a simple one-to-one match.

Common reasons:

```text
Legacy key names
Renamed properties
Added non-optional properties with defaults
Type changes
Enum raw-value migrations
Unknown future enum cases
Validation during decoding
Fallbacks for corrupted or partial data
Decoding multiple possible schema versions
Flattening or unflattening nested payloads
```

Interview version:

> I keep synthesis for simple stable shapes. I write `init(from:)` when I need schema evolution logic: old keys, defaults for added fields, changed enum raw values, validation, or multiple representations. The moment decoding becomes a compatibility policy rather than a structural mapping, synthesis is no longer the right abstraction.

### Q3. When does synthesis stop being enough?

Synthesis stops being enough when correctness depends on domain rules rather than structure.

Examples:

```swift
// Structural decoding is not enough if old data used "name".
case byName

// Structural decoding is not enough if future data may contain unknown cases.
case unknown(String)

// Structural decoding is not enough if missing data should be repaired.
sortMode = decodedValue ?? .byName
```

Interview version:

> Synthesis is structural. It says “decode these properties with these keys.” It does not know migration rules, defaults, data repair policy, or compatibility direction. For app storage, I treat custom decoding as a migration boundary.

---

## 5. Code probe

Given:

```swift
struct Settings: Codable {
    var sortMode: SortMode
}

enum SortMode: String, Codable {
    case byName
    case byDate
}
```

### What happens?

The definitions alone:

```text
Compiles successfully and prints nothing.
```

But with the older persisted payload from the probe:

```swift
import Foundation

struct Settings: Codable {
    var sortMode: SortMode
}

enum SortMode: String, Codable {
    case byName
    case byDate
}

let data = #"{"sortOrder":"name"}"#.data(using: .utf8)!

do {
    let settings = try JSONDecoder().decode(Settings.self, from: data)
    print(settings)
} catch {
    print(error)
}
```

Exact output tested with Swift 6.2.1:

```text
keyNotFound(CodingKeys(stringValue: "sortMode", intValue: nil), Swift.DecodingError.Context(codingPath: [], debugDescription: "No value associated with key CodingKeys(stringValue: \"sortMode\", intValue: nil) (\"sortMode\").", underlyingError: nil))
```

### Why?

The synthesized `Settings.init(from:)` behaves conceptually like this:

```swift
init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    sortMode = try container.decode(SortMode.self, forKey: .sortMode)
}
```

But the old data contains:

```json
{
  "sortOrder": "name"
}
```

The synthesized decoder expects:

```json
{
  "sortMode": "byName"
}
```

Two incompatibilities exist:

```text
Old key:   sortOrder
New key:   sortMode

Old value: name
New value: byName
```

So even if you accepted the old key, `SortMode: String, Codable` would still fail on `"name"` unless the enum explicitly handles that legacy raw value.

### Fix or redesign

```swift
import Foundation

struct Settings: Codable, Equatable {
    var sortMode: SortMode

    init(sortMode: SortMode = .byName) {
        self.sortMode = sortMode
    }

    private enum CodingKeys: String, CodingKey {
        case sortMode      // new key
        case sortOrder     // legacy key
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)

        if let sortMode = try container.decodeIfPresent(SortMode.self, forKey: .sortMode) {
            self.sortMode = sortMode
        } else if let legacySortOrder = try container.decodeIfPresent(LegacySortOrder.self, forKey: .sortOrder) {
            self.sortMode = SortMode(legacySortOrder)
        } else {
            self.sortMode = .byName
        }
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)

        // Write only the new schema.
        try container.encode(sortMode, forKey: .sortMode)
    }
}

enum SortMode: Equatable, Codable, CustomStringConvertible {
    case byName
    case byDate
    case byPriority
    case unknown(String)

    var description: String {
        switch self {
        case .byName:
            "byName"
        case .byDate:
            "byDate"
        case .byPriority:
            "byPriority"
        case .unknown(let rawValue):
            "unknown(\(rawValue))"
        }
    }

    init(_ legacy: LegacySortOrder) {
        switch legacy {
        case .name:
            self = .byName
        case .date:
            self = .byDate
        }
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let rawValue = try container.decode(String.self)

        switch rawValue {
        case "byName", "name":
            self = .byName
        case "byDate", "date":
            self = .byDate
        case "byPriority", "priority":
            self = .byPriority
        default:
            self = .unknown(rawValue)
        }
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()

        switch self {
        case .byName:
            try container.encode("byName")
        case .byDate:
            try container.encode("byDate")
        case .byPriority:
            try container.encode("byPriority")
        case .unknown(let rawValue):
            try container.encode(rawValue)
        }
    }
}

enum LegacySortOrder: String, Codable {
    case name
    case date
}
```

Test harness:

```swift
let payloads = [
    #"{"sortOrder":"name"}"#,
    #"{"sortMode":"byDate"}"#,
    #"{"sortMode":"priority"}"#,
    #"{"sortMode":"futureServerMode"}"#,
    #"{}"#
]

for payload in payloads {
    let data = payload.data(using: .utf8)!
    let settings = try JSONDecoder().decode(Settings.self, from: data)
    print(settings.sortMode)
}
```

Exact output:

```text
byName
byDate
byPriority
unknown(futureServerMode)
byName
```

### Why this fix is correct

It separates three concerns:

```text
Settings migration:
- Accept old key: sortOrder
- Accept new key: sortMode
- Encode only new key: sortMode

SortMode migration:
- Accept old raw values: name/date
- Accept new raw values: byName/byDate/byPriority
- Preserve unknown raw values when possible

Default policy:
- Missing setting falls back to .byName
```

That is the important production pattern: **be liberal when reading old data, conservative and stable when writing new data.**

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Keep encoded key stable with `case sortMode = "sortOrder"`|Pure Swift rename, no external schema change intended|New schema name never appears on disk|
|Custom `init(from:)` accepting both keys|Real schema migration|More code, needs tests|
|Add `.unknown(String)`|Need forward compatibility or round-trip preservation|More complex switching and UI handling|
|Default unknown values to `.byName`|Settings where safe fallback is acceptable|Loses information|
|Explicit schema version field|Complex migrations across many versions|Versioning logic can become heavy|

---

## 6. Exercise

### Problem

Evolve a persisted settings model by renaming a field and adding a new enum case without breaking older data.

### Bad / naive version

```swift
struct Settings: Codable {
    var sortMode: SortMode
}

enum SortMode: String, Codable {
    case byName
    case byDate
    case byPriority
}
```

### What is wrong?

```text
Old data with {"sortOrder":"name"} fails because the key changed.
Old value "name" does not match new raw value "byName".
Future unknown values still fail.
Adding non-optional fields later will break old payloads unless defaults are handled.
```

### Improved version

Use the implementation from the code probe:

```swift
struct Settings: Codable {
    var sortMode: SortMode

    private enum CodingKeys: String, CodingKey {
        case sortMode
        case sortOrder
    }

    init(sortMode: SortMode = .byName) {
        self.sortMode = sortMode
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)

        if let sortMode = try container.decodeIfPresent(SortMode.self, forKey: .sortMode) {
            self.sortMode = sortMode
        } else if let legacySortOrder = try container.decodeIfPresent(LegacySortOrder.self, forKey: .sortOrder) {
            self.sortMode = SortMode(legacySortOrder)
        } else {
            self.sortMode = .byName
        }
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(sortMode, forKey: .sortMode)
    }
}
```

### Why this is better

This gives you compatibility in both directions that matter for production:

```text
Old app data -> new app can decode it.
New app data -> uses the new schema going forward.
Unknown values -> do not necessarily destroy the whole settings payload.
Missing values -> get explicit defaults instead of accidental crashes.
```

The migration policy is local, testable, and documented by code.

---

## 7. Production guidance

Use custom Codable migrations in production when:

```text
Persisted data outlives a single app launch
UserDefaults stores Codable blobs
Settings are shared with widgets/extensions
A server schema evolves independently from the app
An app may downgrade or sync data between versions
You rename properties or enum cases
You add non-optional fields
```

Be careful when:

```text
Using lossy decoding
Defaulting missing values silently
Ignoring unknown enum cases
Changing raw values of persisted enums
Using JSONDecoder key strategies globally
Mixing server DTOs with domain models
```

Avoid when:

```text
The type is purely internal and never persisted
The data is disposable cache and can be invalidated safely
A full migration layer would be clearer than overloading one Codable type
The encoded format must preserve security/audit-critical data exactly
```

Debugging checklist:

```text
What exact payload is failing?
Which key is missing?
Did a Swift property rename change the encoded key?
Did an enum raw value change?
Was a non-optional field added?
Should missing data default, fail, or trigger migration?
Should unknown enum values be preserved?
Are encode and decode intentionally asymmetric?
Do tests cover old v1/v2/v3 payload fixtures?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `Codable` can encode and decode structs automatically. If it fails, I can add `CodingKeys`.

### Senior answer

> `Codable` synthesis is structural. For persisted data, property names and enum raw values are schema. I use `CodingKeys` and custom `init(from:)` when renames, defaults, old values, or unknown values need explicit compatibility handling.

### Staff-level answer

> I treat every persisted `Codable` representation as a versioned schema. I distinguish domain models from storage DTOs when needed, keep read compatibility for older payloads, write a stable current schema, add fixture-based migration tests, and make lossy decoding an explicit product decision. I do not rely on synthesis once schema evolution becomes part of correctness.

Staff-level questions to ask:

```text
Is this type a domain model, storage model, or network DTO?
Who writes this data, and who reads it?
Can older app versions see data written by newer versions?
Is unknown data safe to drop, or must it be preserved?
Do we have fixture tests for every persisted schema version?
```

---

## 9. Interview-ready summary

`Codable` synthesis is great when the Swift type and encoded schema match exactly. But persisted data turns property names, coding keys, enum raw values, and required fields into compatibility contracts. Renaming a property from `sortOrder` to `sortMode` breaks old data because the synthesized decoder looks for the new key. I use `CodingKeys` and custom `init(from:)` when I need to accept legacy keys, provide defaults for added fields, migrate enum raw values, validate data, or handle unknown future values. For production, I read old schemas, write the current schema, and cover migrations with fixture tests.

---

## 10. Flashcards

Q: What is `Codable`?

A: A type alias for `Encodable & Decodable`; it means a type can encode itself to and initialize itself from an external representation.

Q: Why does renaming a `Codable` property break persisted data?

A: Synthesis uses property names as coding keys unless overridden. Old data has the old key, while the new synthesized decoder expects the new key.

Q: When should you write a custom `init(from:)`?

A: When decoding needs migration, defaults, validation, legacy keys, type changes, alternate representations, lossy behavior, or unknown enum handling.

Q: Why are raw-value `Codable` enums fragile?

A: They fail when stored data contains a raw value not represented by a case, including legacy values or values introduced by future versions.

Q: What is the safest migration pattern for persisted settings?

A: Decode old and new representations, normalize into the current model, and encode only the current schema.

Q: When is lossy decoding acceptable?

A: When partial data is better than total failure and lost values are non-critical, such as remote content lists or telemetry.

Q: When is lossy decoding dangerous?

A: When it hides corruption or loses important user, financial, security, or migration-critical data.

---

## 11. Related sections

- [[B7 — Synthesized conformances and semantic correctness]]
- [[B10 — API design guidelines and Swift-native surface design]]
- [[B11 — Library evolution and resilience-aware API design]]
- [[A9 — Enums, associated values, recursive enums, and state modeling]]
- [[F1 — Testing strategy in modern Swift]]
    

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — B9 rubric entry.
- GitHub. "Swift Archival & Serialization." Swift Evolution SE-0166. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md
- Apple Developer. "Encoding and Decoding Custom Types." Apple Developer Documentation. https://developer.apple.com/documentation/foundation/encoding-and-decoding-custom-types
- Apple Developer. "JSONDecoder." Apple Developer Documentation. https://developer.apple.com/documentation/foundation/jsondecoder
