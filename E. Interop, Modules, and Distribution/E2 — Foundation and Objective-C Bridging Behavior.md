---
tags:
  - swift
  - ios
  - interview-prep
  - interop
  - foundation-bridging
  - objc
---

# E2 — Foundation and Objective-C Bridging Behavior

**Goal:** Distinguish a bridged collection’s membership snapshot from its shared reference elements.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Bridging converts representations; it does not promise shared identity, identical mutability, or a fixed conversion cost. A Swift array can snapshot Foundation collection membership while retaining the same reference elements. String/NSString indexing, numeric validation, opaque boxes, and null sentinels introduce other boundaries that need explicit contracts. [Foundation value types](https://developer.apple.com/documentation/foundation)

## Rules and pitfalls

### Test membership and payload changes separately

Changing NSMutableArray membership does not necessarily change a previously converted Swift array's element sequence. Mutating an object already retained by both representations can remain visible through both.

**Common pitfall:** Calling the bridge a deep copy hides shared element state. Copy domain values explicitly when an independent snapshot is required; see [[A1 — Value semantics, reference semantics, identity, and copy-on-write]].

### Validate semantics beyond representability

An NSNumber can represent a Boolean, fractional, out-of-range, or otherwise invalid quantity. A boxed Swift value can round-trip through AnyObject without being valid JSON. NSNull is an explicit collection sentinel, not ordinary missing-key absence.

**Common pitfall:** A successful cast or JSON-compatible container does not establish the domain schema. Validate exactness, range, field meaning, and null policy before constructing domain values.

## Interview questions

### Which mutations remain visible after converting NSMutableArray to a Swift array of reference objects?

The Swift array retains its converted element membership even if the NSMutableArray later appends or removes entries. Its reference elements can still observe object mutation through shared aliases. Test those two operations independently rather than treating the entire graph as one snapshot.

### Why can a boxed Swift payload round-trip but fail JSON validation, and why is NSNumber conversion insufficient for a quantity schema?

Foundation can carry an opaque box whose private payload is recoverable by Swift but unsupported by JSONSerialization's schema. NSNumber conversion establishes a representation, not an integer quantity policy: reject Boolean, fractional, nonfinite, and out-of-range values as the schema requires.

### What does the prepared renderer's conversion counter establish, and how do invalidation, retention, isolation, and actual profiling affect the decision?

The counter proves how many instrumented conversion calls that fixture made. It does not measure total allocations or rendering speed. A useful cache also needs correct invalidation, bounded retention, and isolation. Canonically equal Strings can have different UTF-16 units, so a cached NSRange mapping may need representation-sensitive identity and actual profiling.

## Focused exercise

**Task:** Bridge a mutable Foundation collection into a Swift array of reference elements. Check membership after append and payload state after mutation.

**Acceptance checks:**

- Appending and removing Foundation members do not change the snapshot count.
- Shared object mutation remains visible through the Swift array.
- The retained element identity is checked separately from collection identity.

**Solution:**

Standalone Foundation membership/reference probe:

```swift
import Foundation
final class Box: NSObject {
    var value: Int
    init(_ value: Int) { self.value = value }
}
let first = Box(1)
let source = NSMutableArray(array: [first])
let snapshot = source as! [Box]
source.add(Box(2))
precondition(source.count == 2 && snapshot.count == 1)
first.value = 9
precondition(snapshot[0].value == 9)
source.removeAllObjects()
precondition(snapshot.count == 1 && snapshot[0] === first)
```

**Reasoning:** The conversion forms an Array value of Box references. Value behavior covers that sequence, not recursive object cloning. The forced cast is justified by this fixture's construction; production heterogeneous input should use conditional validation and report invalid elements.

## In practice

**Choose it when:** Put conversion at an explicit boundary rather than repeatedly bridging in a hot path.

**Investigate by:** Count conversions, then profile allocations/cost and test cache invalidation against exact text representations.

**Strong evidence:** Define snapshot depth, numeric/null schema, lifetime, and isolation before claiming an adapter improves performance.

### Boxing, numeric validation, and text units — optional

#### Opaque boxes, optional payloads, and null sentinels

boxesAndNulls returns (true, false, true, true). The Swift struct round-trips through NSArray for Swift callers but is not valid JSON. The optional string array bridges, and nil becomes NSNull in the object collection, which this JSON validity check accepts. NSNull is an object sentinel, not Objective-C nil; APIs that require NSString may reject it. Nested optional states need their own explicit schema.

```swift
import Foundation

struct OpaquePayload: Equatable { let id: Int }
func boxesAndNulls() -> (Bool, Bool, Bool, Bool) {
    let payloads = [OpaquePayload(id: 7)] as NSArray
    let optionals: [String?] = ["A", nil]
    let objects = optionals as NSArray
    return ((payloads as? [OpaquePayload]) == [OpaquePayload(id: 7)],
            JSONSerialization.isValidJSONObject(payloads),
            objects[1] is NSNull,
            JSONSerialization.isValidJSONObject(objects))
}
```

Some older Array source documentation still shows [String?] bridging as rejected. The checked current compiler accepts this exact example, consistent with the older optional-bridging design. Preserve the observed/API contract instead of copying that historical diagnostic. Do not depend on a private opaque-box class name. [SE-0140](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0140-bridge-optional-to-nsnull.md), [SE-0116](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0116-id-as-any.md)

#### Validate meaning as well as representability

This boundary accepts an exact integral NSNumber value from 0 through 1000, rejects Foundation Booleans even though they are NSNumber objects, and rejects fractional, nonfinite, negative, and out-of-range quantities. A numeric String is not accepted. It deliberately accepts NSNumber(value: 1.0) as the exact quantity 1; original numeric construction type is not the schema.

```swift
import Foundation
import CoreFoundation

enum QuantityError: Error { case notNumber, boolean, outOfRange }
func quantity(_ value: Any) throws(QuantityError) -> Int {
    guard let number = value as? NSNumber else { throw .notNumber }
    guard CFGetTypeID(number) != CFBooleanGetTypeID() else { throw .boolean }
    guard let result = Int(exactly: number), (0...1000).contains(result) else {
        throw .outOfRange
    }
    return result
}
```

The Boolean check uses CoreFoundation's public type identity rather than a private NSNumber subclass name. Int(exactly:) supplies the checked numeric conversion. A more general parser may need decimal policy, units, or wider ranges; this function's contract is intentionally narrow. [SE-0139](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0139-bridge-nsnumber-and-nsvalue.md)

#### UTF-16 offsets and whole Characters

For "🇺🇸e\u{301}", String.count is 2 and NSString.length is 6. The helper accepts the flag's {0,4} and the composed character's {4,2}; it rejects {4,1}, {5,0}, and a surrogate-interior {1,0}. It permits {6,0}, the empty range at the end, and checks arithmetic without overflowing location + length. It is a whole-Character policy for this source value, not a universal requirement of every NSString API.

```swift
import Foundation

func wholeCharacterSlice(_ range: NSRange, in text: String) -> String? {
    let units = text.utf16.count
    guard range.location >= 0, range.length >= 0,
          range.location <= units, range.length <= units - range.location,
          let converted = Range(range, in: text) else { return nil }
    func boundary(_ index: String.Index) -> Bool {
        index == text.endIndex || text.indices.contains(index)
    }
    guard boundary(converted.lowerBound), boundary(converted.upperBound) else { return nil }
    return String(text[converted])
}
```

The Character-boundary scan is linear; choose a suitable indexed representation if repeated conversions are performance-critical. The caller must still ensure the NSRange refers to this exact source snapshot. Do not execute an invalid slice to demonstrate an error.

## References

### Related guides

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A4 — Optionals and Nil Modeling]]
- [[A12 — Availability and Conditional Compilation]]
- [[B6 — Any, AnyObject, Metatypes, and Runtime Type Information]]
- [[B8 — Equatable, Hashable, and Collection Correctness]]
- [[B9 — Codable and Schema Evolution]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C4 — String and Unicode correctness]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[E1 — Objective-C Interoperability]]
- [[E3 — C Interoperability and Unsafe Boundaries]]
- [[E5 — Modules, packages, targets, products, and resources]]
- [[E6 — Import Visibility and Dependency Leakage]]
- [[F3 — Performance investigation habits]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

### Official sources

- [Foundation value types](https://developer.apple.com/documentation/foundation)
- [SE-0140](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0140-bridge-optional-to-nsnull.md)
- [SE-0116](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0116-id-as-any.md)
- [SE-0139](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0139-bridge-nsnumber-and-nsvalue.md)
- [SE-0072: remove implicit bridging conversions](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0072-eliminate-implicit-bridging-conversions.md)
- [Standard-library Array source](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift)
- [SF-0023: ProgressManager](https://github.com/swiftlang/swift-foundation/blob/main/Proposals/0023-progress-manager.md)
- [Swift 6.4 release](https://www.swift.org/blog/swift-6.4-released/)
- [Subprocess 1.0.0 manifest](https://github.com/swiftlang/swift-subprocess/blob/1.0.0/Package.swift)
- [tagged documentation](https://github.com/swiftlang/swift-subprocess/blob/1.0.0/README.md)
