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

**Learning goals:**

- Distinguish representation conversion, collection membership, and shared mutable payloads.
- Validate boxed/null/numeric/text input against the domain contract rather than trusting a cast.
- Place conversion and caching behind explicit invalidation, retention, isolation, and deployment policies.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]] for value/reference snapshots, [[A4 — Optionals and Nil Modeling]] for absence, [[C4 — String and Unicode correctness]] for Unicode units and ranges, and [[E1 — Objective-C Interoperability]] for the Objective-C boundary. [[B9 — Codable and Schema Evolution]] explains schema policies; [[F3 — Performance investigation habits]] separates counters from measured performance.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

A Foundation collection arrives from an Objective-C API. Converting it to a Swift Array answers “which element values are in this sequence?” It does not recursively clone objects stored in those elements. A successful bridge also does not prove that a number is a valid quantity or a payload can be encoded as JSON.

**Bridging** converts between supported Swift and Foundation representations. Keep four decisions separate: which representation an API needs, which objects remain shared, what input the application accepts, and how often conversion happens. There is no general fixed allocation or conversion-cost guarantee. [Foundation](https://developer.apple.com/documentation/foundation)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Bridge membership while retaining the same objects

Shared MutablePerson setup. This synchronous fixture owns its Foundation collection locally and never mutates it concurrently.

```swift
import Foundation

final class MutablePerson: NSObject {
    var value: Int
    init(_ value: Int) { self.value = value }
}
func membershipProbe() {
    let first = MutablePerson(1)
    let source = NSMutableArray(object: first)
    let snapshot = source as! [MutablePerson]
    source.add(MutablePerson(2))
    precondition(source.count == 2 && snapshot.count == 1)
    first.value = 9
    precondition(snapshot[0].value == 9)
    source.removeAllObjects()
    precondition(snapshot.count == 1 && snapshot[0] === first)
}
membershipProbe()
```

**Walkthrough:** Appending changes source membership, while snapshot keeps one element. Changing first.value is visible through that shared object. Removing all source members still leaves the snapshot owning first. The forced cast is justified by the fixture’s known element types; validate unknown input conditionally.

### Choose the text API’s indexing unit

Standalone conversion. NSString lengths and NSRange offsets use UTF-16 units, while String.count counts extended grapheme clusters, called Characters.

```swift
import Foundation

let text = "🇺🇸e\u{301}"
let native = text as NSString
precondition(text.count == 2 && native.length == 6)
precondition(native.substring(with: NSRange(location: 0, length: 4)) == "🇺🇸")
```

**Walkthrough:** The flag occupies one Character and four UTF-16 units. Bridging does not make a Character offset into an NSString offset. The supplied range belongs to this exact text; later examples also validate boundaries and arithmetic.

### Validate a numeric quantity before constructing it

Shared quantity setup. Its schema accepts exact integral numbers from 0 through 1000, including a floating representation of an exact integer; it rejects Booleans and strings.

```swift
import CoreFoundation
import Foundation

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

**Walkthrough:** NSNumber is a representation that can also hold Boolean and fractional values. The public CoreFoundation type identity distinguishes Boolean; Int(exactly:) and the range check establish the numeric rule. The error describes why a representable value fails this schema.

### Validate a Foundation range against one snapshot

Shared wholeCharacterSlice setup. This policy permits only complete Characters, including an empty range at a Character boundary.

```swift
import Foundation

func wholeCharacterSlice(_ range: NSRange, in text: String) -> String? {
    let units = text.utf16.count
    guard range.location >= 0, range.length >= 0,
        range.location <= units, range.length <= units - range.location,
        let converted = Range(range, in: text)
    else { return nil }
    func boundary(_ index: String.Index) -> Bool {
        index == text.endIndex || text.indices.contains(index)
    }
    guard boundary(converted.lowerBound), boundary(converted.upperBound) else { return nil }
    return String(text[converted])
}
```

**Walkthrough:** The subtraction-based check cannot overflow location + length. Conversion establishes positions in this snapshot; the boundary scan adds the application’s whole-Character policy. A different NSString API can legitimately use another policy.

## Rules and pitfalls

### A membership snapshot is not a deep snapshot

**Common pitfall:** Treating the bridged array as an independent object graph. Prepend MutablePerson setup.

**Mistake:** Standalone runnable example.

```swift
let person = MutablePerson(1)
let original = NSMutableArray(object: person)
let snapshot = original as! [MutablePerson]
person.value = 9
precondition(snapshot[0].value == 9)
```

**Consequence:** The saved array sees the changed payload even though its membership is independent.

**Why:** Each element is still a reference to the same object; Array value semantics do not recursively clone it.

**Correction:** Copy the needed domain fields into a value when the contract requires an independent snapshot. Prepend MutablePerson setup.

```swift
struct PersonSnapshot: Equatable { let value: Int }
let person = MutablePerson(1)
let original = NSMutableArray(object: person)
let snapshot = (original as! [MutablePerson]).map { PersonSnapshot(value: $0.value) }
person.value = 9
precondition(snapshot == [PersonSnapshot(value: 1)])
```

**What changes:** The value fields capture the intended state. Referenced fields added later would require the same depth review; this local copy does not synchronize concurrent writers.

### A lossy conversion can silently accept the wrong schema

**Common pitfall:** Reading intValue and accepting a fractional quantity.

**Mistake:** Standalone runnable example.

```swift
import Foundation

let supplied = NSNumber(value: 1.5)
let accepted = supplied.intValue
precondition(accepted == 1)
```

**Consequence:** The original fraction disappears and an apparently valid integer remains.

**Why:** A conversion that produces an Int is not a check that the source was an exact allowed quantity.

**Correction:** Use the shared quantity validator and keep its error observable. Prepend quantity setup.

```swift
let exact = try quantity(NSNumber(value: 1.0))
precondition(exact == 1)
for invalid: Any in [
    NSNumber(value: true), NSNumber(value: 1.5), NSNumber(value: Double.nan), NSNumber(value: -1),
    NSNumber(value: 1001), "1",
] {
    do {
        _ = try quantity(invalid)
        preconditionFailure("Expected schema rejection")
    } catch {}
}
```

**What changes:** The boundary rejects meaning that the schema does not support. A different domain may need decimal precision, units, or a larger range.

### Canonical text equality is too weak for a UTF-16 cache key

**Common pitfall:** Reusing representation-sensitive results because String equality says the text is equal.

**Mistake:** Standalone runnable example.

```swift
import Foundation

let composed = "é"
let decomposed = "e\u{301}"
precondition(composed == decomposed)
let cachedLength = (composed as NSString).length
let incorrectlyReused = composed == decomposed ? cachedLength : (decomposed as NSString).length
precondition(incorrectlyReused == 1 && (decomposed as NSString).length == 2)
```

**Consequence:** The reused UTF-16 length is wrong for the new representation.

**Why:** String equality compares canonical text. NSString offsets and lengths describe a particular sequence of UTF-16 units.

**Correction:** Key this representation cache by exact UTF-16 units and invalidate explicitly. Shared PreparedRenderer setup; MainActor owns its mutable cache.

```swift
import Foundation

@MainActor final class PreparedRenderer {
    private var units: [UInt16]?
    private var native: NSString?
    private(set) var conversions = 0
    func prepare(_ text: String) -> NSString {
        let key = Array(text.utf16)
        if key != units {
            native = text as NSString
            units = key
            conversions += 1
        }
        return native!
    }
    func invalidate() {
        units = nil
        native = nil
    }
}
let renderer = PreparedRenderer()
precondition(renderer.prepare("é").length == 1)
precondition(renderer.prepare("e\u{301}").length == 2)
precondition(renderer.conversions == 2)
```

**What changes:** The second representation rebuilds the cached NSString. This one-entry cache bounds entry count, not byte size; every key construction still scans and retains text units. The actor policy controls access rather than relying on NSString immutability to protect cache variables.

## Worked example

### An opaque Foundation box can round-trip without being JSON

**Setup:** Standalone Foundation example.

**Predict:** Does success converting back to the Swift type imply JSONSerialization can encode the object collection?

```swift
import Foundation

struct OpaquePayload: Equatable { let id: Int }
func boxesAndNulls() -> (Bool, Bool, Bool, Bool) {
    let payloads = [OpaquePayload(id: 7)] as NSArray
    let optionals: [String?] = ["A", nil]
    let objects = optionals as NSArray
    return (
        (payloads as? [OpaquePayload]) == [OpaquePayload(id: 7)],
        JSONSerialization.isValidJSONObject(payloads),
        objects[1] is NSNull,
        JSONSerialization.isValidJSONObject(objects)
    )
}

let result = boxesAndNulls()
precondition(result.0 && !result.1 && result.2 && result.3)
```

**Outcome:** The Swift payload round-trips; its collection is not valid JSON. A bridged optional nil is represented by NSNull, and the string/null collection is valid JSON.

**Why:** Foundation can carry an opaque Swift box that JSON has no schema for. NSNull is an actual object sentinel; a missing dictionary key and a present null still describe different application states. Avoid private box/subclass names. [Optional bridging](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0140-bridge-optional-to-nsnull.md)

## Interview questions

### Which mutations remain visible after converting NSMutableArray to a Swift array of reference objects?

**Answer:** Membership changes remain local to the collection value, while payload mutation remains visible through shared object references.

**Example:** Prepend MutablePerson setup.

```swift
let person = MutablePerson(1)
let native = NSMutableArray(object: person)
let saved = native as! [MutablePerson]
native.add(MutablePerson(2))
person.value = 7
precondition(native.count == 2 && saved.count == 1)
precondition(saved[0] === person && saved[0].value == 7)
native.removeAllObjects()
precondition(saved.count == 1 && saved[0].value == 7)
```

**Walkthrough:** Append and removal change the Foundation collection; neither changes saved’s sequence. The retained element still identifies person and exposes its new value.

**Boundary:** This is a local synchronous observation, not concurrent collection safety or a recursive immutable graph.

### Why can a boxed Swift payload round-trip but fail JSON validation, and why is NSNumber conversion insufficient for a quantity schema?

**Answer:** Opaque transport, JSON representation, and numeric domain validity are three different contracts. A recoverable Swift box can be unsupported by JSON; a valid JSON number can still be an invalid quantity.

**Example:** Prepend quantity setup.

```swift
import Foundation

struct Payload { let id: Int }
let objects = [Payload(id: 7)] as NSArray
precondition((objects as? [Payload])?.first?.id == 7)
precondition(!JSONSerialization.isValidJSONObject(objects))
let fields: [String: Any] = ["quantity": NSNumber(value: true), "note": NSNull()]
precondition(fields["missing"] == nil && fields["note"] is NSNull)
do {
    _ = try quantity(fields["quantity"]!)
    preconditionFailure()
} catch QuantityError.boolean {}
```

**Walkthrough:** The opaque object can be recovered only by a consumer that understands its Swift payload. The dictionary distinguishes absence/null, and the quantity validator rejects a Boolean despite NSNumber representation.

**Boundary:** JSON validity does not validate field names, units, decimal rules, or missing/null semantics. Decode and validate a declared schema.

### What does the prepared renderer’s conversion counter establish, and how do invalidation, retention, isolation, and profiling affect the decision?

**Answer:** It counts this fixture’s explicit NSString conversions. Reuse can reduce those calls, but a performance decision also needs the cost of key construction, retained text, invalidation, and actual rendering.

**Example:** Prepend PreparedRenderer setup.

```swift
let cached = PreparedRenderer()
for _ in 0..<10 { precondition(cached.prepare("Ada").length == 3) }
precondition(cached.conversions == 1)
cached.invalidate()
precondition(cached.prepare("Ada").length == 3 && cached.conversions == 2)
precondition(cached.prepare("Grace").length == 5 && cached.conversions == 3)
```

**Walkthrough:** Ten identical inputs use one instrumented conversion. Explicit invalidation and changed content each cause one more. The earlier pitfall checks canonically equal but differently encoded content too.

**Boundary:** The counter is not an allocation or timing benchmark. MainActor owns this example’s cache; a non-UI adapter can use a different explicit owner. Old mapped ranges still need snapshot identity.

### Extension: What does a range conversion still need to validate?

**Answer:** Bounds, arithmetic, the exact source snapshot, and the application’s boundary policy. A convertible NSRange can end inside a Character.

**Example:** Prepend wholeCharacterSlice setup.

```swift
let text = "🇺🇸e\u{301}"
precondition(wholeCharacterSlice(NSRange(location: 0, length: 4), in: text) == "🇺🇸")
precondition(wholeCharacterSlice(NSRange(location: 4, length: 2), in: text) == "e\u{301}")
for range in [
    NSRange(location: 4, length: 1), NSRange(location: 5, length: 0),
    NSRange(location: 1, length: 0), NSRange(location: Int.max, length: Int.max),
    NSRange(location: NSNotFound, length: 0),
] {
    precondition(wholeCharacterSlice(range, in: text) == nil)
}
precondition(wholeCharacterSlice(NSRange(location: 6, length: 0), in: text) == "")
```

**Walkthrough:** The flag and accented Character succeed, interior boundaries fail this policy, and huge values reject before overflowing arithmetic. The empty end range succeeds.

**Boundary:** Repeated boundary scans are linear. [[C4 — String and Unicode correctness|C4]] adds revisioned selections; do not reuse ranges from another equal-looking snapshot.

### Extension: Why do compiler, Foundation runtime, and package-platform availability require separate checks?

**Answer:** The compiler’s ability to parse a name does not establish runtime support or a package’s supported platforms. Check the selected SDK declaration and the pinned package manifest independently.

**Example:** Standalone runnable example.

```swift
import Foundation

@available(anyAppleOS 27.0, *)
func progressType() -> Any.Type { ProgressManager.self }
func foundationAvailable() -> Bool {
    if #available(anyAppleOS 27.0, *) {
        _ = progressType()
        return true
    }
    return false
}
precondition(foundationAvailable())
```

**Walkthrough:** This guarded SDK name check runs on the OS 27 host and is compiled separately for an older target. It does not exercise a progress tree. Subprocess 1.0.0 separately declares tools 6.2, macOS 13 and an iOS 99 exclusion in its manifest; it is a package dependency rather than an automatic Foundation import. [Pinned manifest](https://github.com/swiftlang/swift-subprocess/blob/1.0.0/Package.swift)

**Boundary:** No Subprocess dependency is resolved or process launched by this example. A production package must validate its pinned release, dependencies, destination, and execution policy.

## Focused exercise

**Task:** Bridge a mutable Foundation collection into a Swift array of reference elements and check sequence and payload behavior separately.

**Acceptance checks:**

- Append/removal leave saved membership intact.
- Shared payload edits remain visible and retained identity is checked.
- An independently copied domain value stays unchanged.

**Solution:** Prepend MutablePerson setup.

```swift
struct SavedPerson { let value: Int }
let person = MutablePerson(1)
let source = NSMutableArray(object: person)
let saved = source as! [MutablePerson]
let independent = saved.map { SavedPerson(value: $0.value) }
source.add(MutablePerson(2))
person.value = 9
precondition(source.count == 2 && saved.count == 1)
precondition(saved[0] === person && saved[0].value == 9)
precondition(independent[0].value == 1)
source.removeAllObjects()
precondition(saved.count == 1 && saved[0].value == 9)
```

**Reasoning:** The sequence snapshot and the domain-value snapshot answer different questions. Reconstructing value fields changes the snapshot depth; it does not make an unknown Foundation object safe to read concurrently.

## In practice

**Choose it when:** Use a declared representation boundary and an explicit schema, then introduce caching for an observed need.

**Investigate by:** Check membership, object identity, exact text units, invalidation and numeric/null errors separately.

**Strong evidence:** Count instrumented conversions and profile representative optimized work before claiming a faster renderer or fewer allocations.

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
