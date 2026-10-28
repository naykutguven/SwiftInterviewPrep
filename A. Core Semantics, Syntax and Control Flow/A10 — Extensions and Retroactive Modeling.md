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

**Learning goals:**

- Distinguish extension helpers, protocol witnesses, conditional conformances, and type ownership.
- Explain retroactive-conformance warnings, file-sensitive synthesis, and public construction requirements.
- Choose an owned adapter and verify its semantics across a real module boundary.

**Prerequisites:** [[A3 — Initialization model]] for initialization, [[A11 — Access Control]] for module access, and [[B1 — Protocols, Associated Types, `Self` Requirements, and Compositions]]/[[B3 — Generics, Constraints, and Same-Type Relationships]] for protocol and generic basics. Read the core examples before the schema extension.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A vendor coordinate might need your app’s equality, identity, or persistence policy. An extension can add members or protocol conformance to that coordinate, but it does not make a new local type. A typealias also preserves the original type’s identity. A wrapper creates a distinct type whose semantics you own.

A **witness** implements a protocol requirement. A **retroactive conformance** adds a relationship when your module owns neither the type nor the protocol. Another dependency or a later vendor version may add the same relationship. @retroactive acknowledges that risk; it does not create a private or namespaced conformance. [SE-0364](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0364-retroactive-conformance-warning.md)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

### A constrained helper and a conditional conformance promise different things

The helper exists only for String payloads. Equatable conformance exists whenever the payload is Equatable.

```swift
struct Box<Value> { let value: Value }
extension Box where Value == String {
    var isEmpty: Bool { value.isEmpty }
}
extension Box: Equatable where Value: Equatable {}
func equal<T: Equatable>(_ a: T, _ b: T) -> Bool { a == b }
precondition(Box(value: "").isEmpty)
precondition(equal(Box(value: 7), Box(value: 7)))
```

**Walkthrough:** isEmpty is a member available on Box<String>. The conditional conformance permits generic Equatable use for Box<Int> too. Extensions can organize these contracts, but cannot add ordinary stored instance properties.

### Shared dependency setup: MapVendor

Compile the following as MapVendor.swift in a separate module. Add `-parse-as-library -module-name MapVendor -emit-module -emit-module-path /tmp/MapVendor.swiftmodule -c -o /tmp/MapVendor.o` to the baseline flags. Clients use `-I /tmp`; executable clients also link `/tmp/MapVendor.o`. Validation uses an isolated temporary directory instead of /tmp.

The vendor accepts any Double pair and offers a public initializer. It promises neither geographic validation nor persistence. Those are choices for the adapter.

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

## Rules and pitfalls

### Acknowledgment does not supply synthesized witnesses

**Common pitfall:** Adding @retroactive to an imported Codable conformance and expecting synthesis to become legal. Compile as a separate client of MapVendor.

**Mistake:** Compile-only; intentionally rejected.

```swift
import MapVendor

extension Coordinate: @retroactive Codable {}
```

**Consequence:** The imported type still cannot receive these synthesized Codable witnesses from your source file.

**Why:** The annotation addresses the ownership warning, not access to the declaration’s synthesis context or initialization rules.

**Correction:** Own the data representation. This separate client creates a distinct Codable type through the vendor’s public getters.

```swift
import Foundation
import MapVendor

struct CoordinateData: Codable, Equatable {
    let latitude: Double
    let longitude: Double
    init(_ point: Coordinate) {
        latitude = point.latitude
        longitude = point.longitude
    }
}
let point = Coordinate(latitude: 52, longitude: 13)
let data = CoordinateData(point)
let bytes = try JSONEncoder().encode(data)
let decoded = try JSONDecoder().decode(CoordinateData.self, from: bytes)
precondition(decoded == data)
```

**What changes:** The schema and conformance now belong to a distinct type. This small example does not validate coordinates; the exercise and schema extension add that requirement.

### Shared same-module file setup: Reading.swift

Save this declaration as its own file. The next pitfall compiles another file together with Reading.swift in the same module, rather than importing Reading from a different module. Use baseline flags plus `-parse-as-library -whole-module-optimization Reading.swift Other.swift -c -o /tmp/Reading.o`.

```swift
struct Reading { let value: Int }
```

### Moving a synthesized conformance can change legality

**Common pitfall:** Moving an empty synthesized Equatable extension out of the type’s defining file.

**Mistake:** Compile-only; intentionally rejected.

```swift
extension Reading: Equatable {}
```

**Consequence:** Synthesis fails in the second file even though the declaration is in the same module.

**Why:** The compiler requires this synthesized conformance to be in the type’s source file. Merely making the property accessible in the module does not move its synthesis context.

**Correction:** Either keep the synthesized conformance beside Reading, or write explicit witnesses with accessible state. This second-file correction is compile-only.

```swift
extension Reading: Equatable {
    static func == (lhs: Reading, rhs: Reading) -> Bool { lhs.value == rhs.value }
}
```

**What changes:** The manual witness uses the internal field available in the same module. A private field in another file would need an appropriate exposed operation instead.

## Worked example

### An owned protocol gives a foreign type a locally owned capability

**Setup:** Separate runnable client of MapVendor. This module owns DescribedPoint, so the conformance is not retroactive.

**Predict:** Can a generic helper use the capability without choosing global equality or serialization?

```swift
import MapVendor

protocol DescribedPoint { var coordinateLabel: String { get } }
extension Coordinate: DescribedPoint {
    var coordinateLabel: String { "\(latitude),\(longitude)" }
}
func label<T: DescribedPoint>(_ value: T) -> String { value.coordinateLabel }
precondition(label(Coordinate(latitude: 52, longitude: 13)) == "52.0,13.0")
```

**Outcome:** The generic helper accepts Coordinate through the owned protocol.

**Why:** Owning the protocol lets this module define its capability. A wrapper is still preferable when callers need a distinct identity, validation boundary, or serialization type.

## Interview questions

### Who owns the type/protocol, and what does the same-package exception mean?

**Answer:** If your module owns the type or protocol, it can coordinate that contract. If it owns neither, the conformance normally warns. Swift also treats coordinated modules in the same package as a warning exception; this does not make conflicting conformances sound.

**Example:** Standalone runnable example.

```swift
import Foundation

extension Date: @retroactive Identifiable {
    public var id: TimeInterval { timeIntervalSince1970 }
}
let first = Date(timeIntervalSince1970: 7)
let second = Date(timeIntervalSince1970: 7)
precondition(first.id == second.id)
```

**Walkthrough:** This module owns neither Date nor Identifiable. The annotation acknowledges the risk. The equal timestamp IDs also demonstrate why this choice does not establish unique event identity. Removing the annotation produces a warning; fully qualifying both imported names can also acknowledge it.

**Boundary:** The same-package exception requires matching package identity, not just directories or a common organization. The validation checks matching and different package identities separately. Duplicate conformance remains a process-level risk; choose one owner.

### Why can imported Codable still fail after @retroactive, and what does a manual witness require?

**Answer:** Acknowledgment does not enable synthesis for the imported type. Manual witnesses must deliberately encode fields, decode a schema, validate input, and construct the type through an accessible initializer or factory.

**Example:** Separate runnable MapVendor client; do not combine it with the rejected conformance or the owned DTO module.

```swift
import Foundation
import MapVendor

extension Coordinate: @retroactive Codable {
    private enum Keys: String, CodingKey { case latitude, longitude }
    public init(from decoder: any Decoder) throws {
        let fields = try decoder.container(keyedBy: Keys.self)
        let latitude = try fields.decode(Double.self, forKey: .latitude)
        let longitude = try fields.decode(Double.self, forKey: .longitude)
        guard latitude.isFinite, longitude.isFinite,
            (-90...90).contains(latitude), (-180...180).contains(longitude)
        else {
            throw DecodingError.dataCorrupted(
                .init(
                    codingPath: decoder.codingPath,
                    debugDescription: "Invalid coordinate"))
        }
        self.init(latitude: latitude, longitude: longitude)
    }
    public func encode(to encoder: any Encoder) throws {
        var fields = encoder.container(keyedBy: Keys.self)
        try fields.encode(latitude, forKey: .latitude)
        try fields.encode(longitude, forKey: .longitude)
    }
}
let json = Data(#"{"latitude":52,"longitude":13}"#.utf8)
let point = try JSONDecoder().decode(Coordinate.self, from: json)
precondition(point.latitude == 52 && point.longitude == 13)
```

**Walkthrough:** Decoding reads a chosen schema, checks the domain, then calls the public vendor initializer. Encoding selects keys explicitly. This is an independently compiled alternative, not linked with another Coordinate Codable conformance.

**Boundary:** It still imposes a global conformance and permits programmatic vendor values outside the decoder’s validation. An owned data transfer object (DTO) makes that invariant and schema local; use this bridge only with an explicit compatibility owner.

### Shared runtime-discovery setup

The next two blocks are separate compile-only modules. Build RuntimeProbe first; it has no import of the bridge. Build CoordinateBridge with `-I /tmp` after MapVendor. Both use the module command from Shared dependency setup with their own module names. A client can then link all three objects while importing only RuntimeProbe and MapVendor.

```swift
public func discover(_ value: Any) -> String? {
    (value as? any CustomStringConvertible)?.description
}
```

```swift
import MapVendor

extension Coordinate: @retroactive CustomStringConvertible {
    public var description: String { "point" }
}
```

### How do static generic use, runtime conformance discovery, and member ambiguity differ across modules?

**Answer:** Static generic calls need a conformance visible to the compiling source. A runtime cast can discover a conformance in a loaded module even when the cast’s module never imported it. Ordinary extension members have separate symbols but can still make source lookup ambiguous.

**Example:** Runnable client of all three shared modules. Link MapVendor.o, RuntimeProbe.o, and CoordinateBridge.o; import only the modules shown.

```swift
import MapVendor
import RuntimeProbe

let point = Coordinate(latitude: 52, longitude: 13)
precondition(discover(point) == "point")
```

**Walkthrough:** RuntimeProbe was compiled without CoordinateBridge. Linking the bridge’s object loads its conformance metadata, so discover can find it at runtime. The client also omits importing CoordinateBridge.

**Boundary:** Source imports do not create private runtime conformances. Conversely, a runtime cast result does not give the source a statically visible conformance. Two imported extensions that add the same member name can make lookup ambiguous even without adding a conformance.

To see the separate member-lookup problem, compile these two independent extension modules with the MapVendor search path. Their symbols have different module owners, but a client importing both cannot choose label automatically:

```swift
import MapVendor

extension Coordinate { public var label: String { "A" } }
```

```swift
import MapVendor

extension Coordinate { public var label: String { "B" } }
```

This separate client is **intentionally rejected** for ambiguous member lookup, even though neither extension introduced a conformance:

```swift
import CoordinateLabelA
import CoordinateLabelB
import MapVendor

let label = Coordinate(latitude: 52, longitude: 13).label
```

Importing only one extension module resolves that source lookup. This runnable client links MapVendor.o and CoordinateLabelA.o:

```swift
import CoordinateLabelA
import MapVendor

precondition(Coordinate(latitude: 52, longitude: 13).label == "A")
```

Using only the MapVendor import, this separate **compile-only, intentionally rejected** client cannot statically prove the generic requirement, even if an eventual executable would link CoordinateBridge:

```swift
import MapVendor

func label<T: CustomStringConvertible>(_ value: T) -> String { value.description }
let text = label(Coordinate(latitude: 52, longitude: 13))
```

## Focused exercise

**Task:** Replace an imported-type conformance with an owned validated adapter.

**Acceptance checks:**

- Map both coordinates through the public vendor API.
- Reject out-of-range and nonfinite input in either coordinate.
- Equality compares owned data; conversion back uses accessible construction.

**Solution:** Separate runnable MapVendor client.

```swift
import MapVendor

struct Location: Equatable {
    let latitude: Double
    let longitude: Double
    init?(_ point: Coordinate) {
        guard point.latitude.isFinite, point.longitude.isFinite,
            (-90...90).contains(point.latitude), (-180...180).contains(point.longitude)
        else { return nil }
        latitude = point.latitude
        longitude = point.longitude
    }
    var vendorCoordinate: Coordinate { Coordinate(latitude: latitude, longitude: longitude) }
}
let point = Coordinate(latitude: 52, longitude: 13)
let location = Location(point)!
precondition(location == Location(point))
precondition(location.vendorCoordinate.latitude == 52)
precondition(location.vendorCoordinate.longitude == 13)
for (latitude, longitude) in [(-90.0, -180.0), (90.0, 180.0)] {
    precondition(Location(Coordinate(latitude: latitude, longitude: longitude)) != nil)
}
for (latitude, longitude) in [
    (91.0, 0.0), (-91.0, 0.0), (0.0, 181.0),
    (0.0, -181.0), (Double.nan, 0.0), (0.0, Double.infinity),
] {
    precondition(Location(Coordinate(latitude: latitude, longitude: longitude)) == nil)
}
```

**Reasoning:** The owned type makes geographic validity and equality explicit without modifying Coordinate’s protocol relationships. A typealias would retain the foreign identity and could not provide this independent conformance boundary.

## In practice

**Choose it when:** Use adapters for local identity, coding, or compatibility policy. Use extensions for behavior that belongs to an owned contract.

**Investigate by:** Compile realistic clients and inspect the owner of each conformance, imported member, and construction path.

**Strong evidence:** Test mapping across a real dependency boundary, including upstream additions and the behavior of loaded conformances.

### Separate-module adapter and schema — optional

This preserves the larger persistence example. Compile the next block as TrackingSDK, importing the already built MapVendor. The schema accepts current latitude/longitude or legacy lat/lng pairs and writes only current keys. Mixed, partial, null, wrong-typed, and invalid coordinates fail. Unknown unrelated keys are ignored rather than retained.

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
            (-90...90).contains(latitude), (-180...180).contains(longitude)
        else {
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
            throw DecodingError.dataCorrupted(
                .init(
                    codingPath: decoder.codingPath,
                    debugDescription: "Mixed coordinate key formats"))
        }
        let latitude = try values.decode(Double.self, forKey: hasCurrent ? .latitude : .lat)
        let longitude = try values.decode(Double.self, forKey: hasCurrent ? .longitude : .lng)
        do {
            try self.init(latitude: latitude, longitude: longitude)
        } catch {
            throw DecodingError.dataCorrupted(
                .init(
                    codingPath: decoder.codingPath,
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

Runnable ordinary external client: add `-I /tmp`, link MapVendor.o and TrackingSDK.o. These checks exercise both schema versions and invalid inputs; accepting legacy data demonstrates only the new-reader/old-data migration direction.

```swift
import Foundation
import MapVendor
import TrackingSDK

func decode(_ json: String) throws -> CoordinateDTO {
    try JSONDecoder().decode(CoordinateDTO.self, from: Data(json.utf8))
}
let current = try decode(#"{"latitude":52,"longitude":13}"#)
let legacy = try decode(#"{"lat":52,"lng":13}"#)
precondition(current == legacy)
let mapped = try CoordinateDTO(current.vendorCoordinate)
precondition(mapped == current)
let encoded = try JSONEncoder().encode(current)
let keys = try JSONSerialization.jsonObject(with: encoded) as! [String: Any]
precondition(Set(keys.keys) == ["latitude", "longitude"])
let roundTrip = try JSONDecoder().decode(CoordinateDTO.self, from: encoded)
precondition(roundTrip == current)
for invalid in [
    #"{"latitude":52}"#, #"{"lat":52}"#,
    #"{"latitude":52,"lng":13}"#, #"{"lat":52,"longitude":13}"#,
    #"{"latitude":null,"longitude":13}"#,
    #"{"latitude":"52","longitude":13}"#,
    #"{"latitude":91,"longitude":13}"#,
] {
    do {
        _ = try decode(invalid)
        preconditionFailure("Expected rejection")
    } catch { precondition(error is DecodingError) }
}
for (latitude, longitude) in [(-90.0, -180.0), (90.0, 180.0)] {
    _ = try CoordinateDTO(latitude: latitude, longitude: longitude)
}
for invalid in [Double.nan, Double.infinity, -Double.infinity] {
    do {
        _ = try CoordinateDTO(latitude: invalid, longitude: 0)
        preconditionFailure()
    } catch { precondition(error is CoordinateValidationError) }
    do {
        _ = try CoordinateDTO(latitude: 0, longitude: invalid)
        preconditionFailure()
    } catch { precondition(error is CoordinateValidationError) }
}
```

**Walkthrough:** Public construction allows ordinary client mapping in both directions. The decoder routes every accepted format through the same validation initializer; encoding writes one current format. The distinct DTO conformance survives a vendor adding its own conformance, though vendor API changes can still require mapping updates. This example does not prove old readers understand new writes or validate a richer geospatial reference system.

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
