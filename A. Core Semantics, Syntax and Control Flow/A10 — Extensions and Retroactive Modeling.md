---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - extensions
  - retroactive-modeling
---

# A10 — Extensions and Retroactive Modeling

**Goal:** Replace an external conformance with an owned adapter whose mapping and semantics are explicit.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

An extension contributes members or a conformance to a type; it does not create a private local version of that type. A typealias also preserves type identity. When your module owns neither a type nor a protocol, an added conformance competes with upstream and other dependencies across the process. Prefer an owned adapter when you need different semantics. [SE-0364](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md)

## Rules and pitfalls

### Decide who owns the contract

Owned-type extensions can organize helpers and witnesses. A constrained helper exists only for matching generic arguments; a conditional conformance makes the type satisfy a protocol under constraints. These are different promises.

**Common pitfall:** Adding @retroactive only acknowledges collision risk. It does not namespace a conformance, make it private, or supply missing witnesses. Use an adapter with a documented equality, identity, or serialization rule when your policy is local.

### Verify file placement and client construction

Empty synthesized conformance extensions can depend on the source file containing the stored-property declaration. Moving them to another file can break synthesis. Manual witnesses need accessible state and a valid construction path.

**Common pitfall:** An imported Codable extension can remain invalid after @retroactive because synthesis cannot inspect and initialize the external type as required. Map through the dependency's public API into an owned Codable value.

## Interview questions

### Who owns the type/protocol, and what does the same-package exception mean for warning behavior?

If you own the type or protocol, you can coordinate its semantics and evolution. If neither is owned, duplicate conformances are a process-level risk. Coordinating modules in the same package are exempt from the retroactive warning in the specified cases; that exception does not make contradictory duplicate conformances a sound design.

### Why can an imported-type Codable extension still fail after adding retroactive, and what does a manual witness require?

The annotation handles the warning, not synthesis. Manual decoding must obtain fields through an accessible schema and invoke a valid public initializer or factory, with required validation. Encoding must deliberately choose fields and format. A wrapper often avoids both inaccessible construction and an unwanted global serialization contract.

### How do static generic use, runtime conformance discovery, and member-name ambiguity differ across modules?

Static generic use depends on the conformance visible during compilation. Runtime casts can discover loaded conformances beyond an apparent source-import boundary. Ordinary extension members have distinct symbols but can still collide during source lookup. An adapter prevents a local semantic choice from becoming an external type's global conformance.

## Focused exercise

**Task:** Replace an imported-type conformance with an owned adapter. Check construction, mapping, and the adapter's protocol semantics.

**Acceptance checks:**

- The adapter maps both coordinates through accessible construction.
- Out-of-domain and nonfinite coordinates reject.
- Equality compares owned validated data without changing VendorPoint.

**Solution:**

Standalone adapter probe; VendorPoint stands in for an imported module's public surface. A genuine cross-module fixture is retained below.

```swift
struct VendorPoint { let latitude: Double; let longitude: Double }
struct Location: Equatable {
    let latitude: Double
    let longitude: Double
    init?(_ point: VendorPoint) {
        guard point.latitude.isFinite, point.longitude.isFinite,
              (-90...90).contains(point.latitude),
              (-180...180).contains(point.longitude) else { return nil }
        latitude = point.latitude
        longitude = point.longitude
    }
}
let point = VendorPoint(latitude: 52, longitude: 13)
precondition(Location(point) == Location(point))
precondition(Location(VendorPoint(latitude: 91, longitude: 0)) == nil)
precondition(Location(VendorPoint(latitude: .nan, longitude: 0)) == nil)
```

**Reasoning:** The owned type makes validation and equality local contracts. If persistence is required, define a versioned schema separately instead of deriving it accidentally from a vendor type's current stored properties.

## In practice

**Choose it when:** Use adapters for local identity, coding, and compatibility decisions; use extensions for behavior that truly belongs to the owned contract.

**Investigate by:** Compile ordinary clients with realistic imports and inspect which module owns each conformance.

**Strong evidence:** Test construction and mapping across an actual dependency boundary, including a compatible upstream addition.

### Separate-module adapter and schema — optional

Review an SDK that wants to persist a vendor coordinate. Compile the modules separately, establish the rejected synthesis case, and replace the retroactive schema with an owned DTO.

#### MapVendor.swift — a separate dependency module

```swift
public struct Coordinate: Sendable {
    public let latitude: Double
    public let longitude: Double

    public init(latitude: Double, longitude: Double) {
        self.latitude = latitude
        self.longitude = longitude
    }
}
```

This teaching vendor permits any Double pair. The explicit public initializer makes outside mapping possible. It does not claim to validate geographic coordinates or provide persistence.

#### TrackingSDK.swift — rejected synthesis attempt

This is an **expected rejection**, even after acknowledging both Codable conformances:

```swift
import MapVendor

extension Coordinate: @retroactive Codable {}
```

The adapter cannot synthesize these imported-type witnesses as if it owned Coordinate's source file. Manual encoding/decoding could use the public fields/initializer and compile, but it would still impose a retroactive schema. A typealias to Coordinate would not remove that ownership issue.

#### TrackingSDK.swift — owned schema and explicit validation

```swift
import MapVendor

public enum CoordinateValidationError: Error { case outOfRange }

public struct CoordinateDTO: Codable, Hashable, Sendable {
    public let latitude: Double
    public let longitude: Double

    private enum CodingKeys: String, CodingKey {
        case latitude, longitude, lat, lng
    }

    public init(latitude: Double, longitude: Double) throws {
        guard latitude.isFinite, longitude.isFinite,
              (-90...90).contains(latitude), (-180...180).contains(longitude) else {
            throw CoordinateValidationError.outOfRange
        }
        self.latitude = latitude
        self.longitude = longitude
    }

    public init(_ coordinate: Coordinate) throws {
        try self.init(latitude: coordinate.latitude, longitude: coordinate.longitude)
    }

    public var vendorCoordinate: Coordinate {
        Coordinate(latitude: latitude, longitude: longitude)
    }

    public init(from decoder: any Decoder) throws {
        let values = try decoder.container(keyedBy: CodingKeys.self)
        let hasCurrent = values.contains(.latitude) || values.contains(.longitude)
        let hasLegacy = values.contains(.lat) || values.contains(.lng)
        guard !(hasCurrent && hasLegacy) else {
            throw DecodingError.dataCorrupted(.init(codingPath: decoder.codingPath,
                                                    debugDescription: "Mixed coordinate key formats"))
        }
        let latitude = try values.decode(Double.self, forKey: hasCurrent ? .latitude : .lat)
        let longitude = try values.decode(Double.self, forKey: hasCurrent ? .longitude : .lng)
        do {
            try self.init(latitude: latitude, longitude: longitude)
        } catch {
            throw DecodingError.dataCorrupted(.init(codingPath: decoder.codingPath,
                                                    debugDescription: "Coordinate out of range",
                                                    underlyingError: error))
        }
    }

    public func encode(to encoder: any Encoder) throws {
        var values = encoder.container(keyedBy: CodingKeys.self)
        try values.encode(latitude, forKey: .latitude)
        try values.encode(longitude, forKey: .longitude)
    }
}
```

The policy is explicit: finite latitude in -90...90 and longitude in -180...180; both current keys or both legacy keys; reject mixed formats rather than choosing one silently. Missing, null, or wrong-typed values fail decoding. Successful encoding always writes latitude/longitude. Unknown unrelated keys are ignored and not preserved by this representation. The model does not normalize longitudes or validate a richer geospatial reference system.

The current reader accepting legacy keys proves one migration direction; it does not prove an old reader understands newly written data. Adding a future upstream Coordinate conformance does not replace CoordinateDTO's distinct conformance, but upstream construction/member changes can still require mapping changes. B9 contains broader schema-evolution cases.

#### Required checks and tradeoffs

Validate current and legacy fixtures, both range boundaries, conversion to/from the vendor type, a round trip, and encoding only the current keys. Reject partial key pairs, mixed formats, null/wrong types, out-of-range values, and non-finite programmatic inputs. Confirm the public DTO works from an ordinary external client.

Compare alternatives deliberately: a named conversion function avoids adding conformance at all; an owned protocol describes a distinct capability; a wrapper/DTO gives a distinct type and schema; an unavoidable @retroactive bridge still needs an owner, compatibility monitoring, and a migration plan. None guarantees zero allocation or eliminates all future dependency work.

## References

### Related guides

- [[A3 — Initialization model]]
- [[A11 — Access Control]]
- [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]
- [[B3 — Generics, Constraints, and Same-Type Relationships]]
- [[B4 — Dispatch Model - Static, Class Virtual, Witness-Table, and Protocol Extension Dispatch]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[B9 — Codable and Schema Evolution]]
- [[B11 — Library Evolution and Resilience-Aware API Design]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D14 — Approachable Concurrency and Default Isolation]]
- [[E6 — Import Visibility and Dependency Leakage]]

### Official sources

- [SE-0364](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md)
- [Extensions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/extensions/)
- [Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [Access Control](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/)
