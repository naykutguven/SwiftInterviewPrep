---
tags:
  - swift
  - ios
  - interview-prep
  - interop
  - foundation-bridging
  - objc
---
## 0. Rubric snapshot

**Rubric expectation**

Understand how Swift standard-library value types bridge to Foundation / Objective-C reference types, especially `String` ⇄ `NSString`, `Array` ⇄ `NSArray`, `Dictionary` ⇄ `NSDictionary`, and `Set` ⇄ `NSSet`. The rubric specifically expects you to reason about where costs or semantic shifts occur.

**Caveats**

Bridging can:

- copy data,
- box Swift values into Objective-C objects,
- change mutability assumptions,
- expose reference semantics behind value-looking APIs,
- hide performance costs at call boundaries.

Apple’s Foundation documentation explicitly groups Foundation classes bridged to Swift standard-library value types and says bridged reference types are useful when you need reference semantics or Foundation-specific behavior. ([Apple Developer, "Classes Bridged to Swift Standard Library Value Types"](https://developer.apple.com/documentation/foundation/classes-bridged-to-swift-standard-library-value-types))

**You should be able to answer**

- What happens when Swift `String` or `Array` crosses into Objective-C APIs?
- Why can bridging hide performance costs in hot paths?

**You should be able to do**

- Investigate a performance issue around repeated bridging between Swift collections and Foundation APIs.

---

## 1. Core mental model

Foundation bridging is Swift’s compatibility layer between two different worlds:

```text
Swift value world        Objective-C / Foundation reference world
String             ⇄     NSString
Array<Element>     ⇄     NSArray
Dictionary<K, V>   ⇄     NSDictionary
Set<Element>       ⇄     NSSet
Int / Bool / Double ⇄    NSNumber
Data               ⇄     NSData
Date               ⇄     NSDate
URL                ⇄     NSURL
```

The important part is not the syntax. The important part is that Swift’s standard-library types usually present **value semantics**, while Foundation collection/string types are **reference objects**. Bridging lets APIs interoperate, but it does not erase the semantic mismatch.

The key idea:

```text
Bridging is a boundary conversion, not a free identity-preserving view.
```

Sometimes bridging is cheap. Sometimes it allocates, copies, boxes each element, validates types, or forces Swift to adopt Foundation-backed storage. The code often looks harmless:

```swift
legacyAPI(items as NSArray)
```

But that one line may allocate an `NSArray`, bridge every `String` to `NSString`, box numbers into `NSNumber`, or copy a mutable Foundation collection into Swift value storage.

For `Array`, the Swift standard library documents the important performance distinction: bridging from `Array` to `NSArray` is O(1) time and space when elements are already class instances or `@objc` protocol values; otherwise it takes O(n) time and space. Bridging from `NSArray` to `Array<Int>` also performs a bridging copy into contiguous Swift storage. ([GitHub, "Array.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift))

For `String`, Swift can bridge to `NSString`, and a `String` originating from Objective-C may use `NSString` storage. The standard-library source explicitly warns that arbitrary `NSString` subclasses can back Swift `String`, so Swift gives no representation or efficiency guarantee in that case; the first mutation may require copying into unique contiguous storage. ([GitHub, "String.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/String.swift))

---

## 2. Essential mechanics

### 2.1 Manual bridging uses `as`

Modern Swift does not generally let you silently assign a Swift value type to its Foundation counterpart. You normally bridge explicitly with `as`.

```swift
import Foundation

let names = ["Aykut", "Emma"]

let foundationNames = names as NSArray
let swiftNames = foundationNames as! [String]

print(foundationNames.count)
print(swiftNames)
```

Output:

```text
2
["Aykut", "Emma"]
```

Without the explicit bridge:

```swift
import Foundation

let names = ["Aykut", "Emma"]
let foundationNames: NSArray = names
```

Compiler error:

```text
error: cannot convert value of type '[String]' to specified type 'NSArray'
```

This is intentional. Swift Evolution SE-0072 removed implicit bridging conversions to simplify the type model and make these boundary conversions explicit. ([GitHub, "Fully Eliminate Implicit Bridging Conversions from Swift"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0072-eliminate-implicit-bridging-conversions.md))

---

### 2.2 `String` and `NSString` have different indexing models

Swift `String` is a collection of `Character` values, where a `Character` is an extended grapheme cluster. `NSString.length` is based on UTF-16 code units.

```swift
import Foundation

let s = "\u{1F1FA}\u{1F1F8}e\u{301}" // 🇺🇸 + é
let ns = s as NSString

print(s.count)
print(ns.length)
```

Output:

```text
2
6
```

Why?

```text
Swift String.count:
- 🇺🇸 is one Character
- e + combining acute accent is one Character
=> 2

NSString.length:
- 🇺🇸 is four UTF-16 code units
- e + combining acute accent is two UTF-16 code units
=> 6
```

This is why blindly passing `String` ranges into `NSString` / `NSRange` APIs is dangerous. In production, convert ranges deliberately:

```swift
let string = "Café 🇺🇸"
let nsRange = NSRange(location: 0, length: 4)

if let swiftRange = Range(nsRange, in: string) {
    print(string[swiftRange])
}
```

The trap is assuming:

```text
Swift Character index == NSString UTF-16 offset
```

That is false.

---

### 2.3 `Array` and `NSArray` differ in element storage, mutability, and cost

`Array` is a Swift value type with copy-on-write storage. `NSArray` is an immutable Foundation reference type. `NSMutableArray` is a mutable Foundation reference type. Apple’s `NSArray` documentation describes `NSArray` as static and `NSMutableArray` as dynamic, and also notes that you can use `NSArray` in Swift when you require reference semantics. ([Apple Developer, "NSArray"](https://developer.apple.com/documentation/foundation/nsarray))

```swift
import Foundation

let mutable: NSMutableArray = ["A", "B"]

let snapshot = mutable as! [String]

mutable.add("C")

print(snapshot)
print(mutable.count)

var swift = mutable as! [String]
swift.append("D")

print(swift)
print(mutable.count)
```

Output:

```text
["A", "B"]
3
["A", "B", "C", "D"]
3
```

The Swift array created from the mutable Foundation array is not a live mutable view over the same collection structure. It behaves like a Swift value. Later structural mutations to `NSMutableArray` do not mutate the previously bridged Swift `Array`.

However, if the array contains reference objects, both arrays may still point to the same objects. The collection structure can be copied while the elements remain shared references.

---

### 2.4 Dictionaries and sets have the same boundary problem

For dictionaries, Apple documents that bridging between `Dictionary` and `NSDictionary` is possible when keys and values are classes, `@objc` protocols, or types that bridge to Foundation types. ([Apple Developer, "Dictionary"](https://developer.apple.com/documentation/swift/dictionary))

```swift
import Foundation

let swift: [String: Int] = [
    "unread": 3,
    "archived": 10
]

let ns = swift as NSDictionary
let back = ns as! [String: Int]

print(back["unread"]!)
```

Output:

```text
3
```

This looks simple, but the bridge may need to convert:

```text
String -> NSString
Int    -> NSNumber
Dictionary storage -> NSDictionary storage
```

For one small dictionary, irrelevant. For thousands of dictionaries per frame in a scrolling screen, not irrelevant.

---

## 3. Common traps and misconceptions

### Trap 1: “Bridging is just a cast”

Bad mental model:

```swift
let ns = swiftArray as NSArray
```

```text
“It is just changing the type label.”
```

Better mental model:

```text
This may allocate an Objective-C collection and bridge every element.
```

For arrays, O(1) bridging is possible only in favorable cases, such as class or `@objc` protocol elements. If the element type itself must bridge, the operation can become O(n) time and space. ([GitHub, "Array.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift))

Bad:

```swift
for row in rows {
    legacyRenderer.render(row.items as NSArray)
}
```

Better:

```swift
let bridgedRows: [NSArray] = rows.map { $0.items as NSArray }

for bridged in bridgedRows {
    legacyRenderer.render(bridged)
}
```

Better still, if you own the boundary:

```swift
struct LegacyRendererAdapter {
    private let renderer: LegacyRenderer

    func render(items: [String]) {
        let bridgedItems = items as NSArray
        renderer.render(bridgedItems)
    }
}
```

Put the bridge at the adapter boundary, not randomly throughout app code.

---

### Trap 2: Assuming Foundation mutability maps to Swift `var`

This is wrong:

```text
Swift var Array == NSMutableArray
Swift let Array == NSArray
```

`var` controls whether the Swift variable can be reassigned or mutated through Swift’s value API. It does not mean the storage is an `NSMutableArray`.

Likewise, `NSArray` being immutable does not make all contained objects immutable. It only means the collection’s membership is immutable through `NSArray`.

Bad:

```swift
let users = legacyUsers as! [User]
legacyUsers.add(newUser)

// Assuming `users` now contains `newUser`
```

Better:

```swift
let users = legacyUsers as! [User]

// Treat this as a snapshot of collection membership.
// Re-bridge or redesign the API if live mutation is required.
```

---

### Trap 3: Ignoring `NSString` / `NSRange` semantics

Bad:

```swift
let nsRange = NSRange(location: 0, length: 5)
let start = text.index(text.startIndex, offsetBy: nsRange.location)
let end = text.index(start, offsetBy: nsRange.length)
let range = start..<end
```

This assumes `NSRange` offsets are Swift `Character` offsets. They are not.

Better:

```swift
if let range = Range(nsRange, in: text) {
    let substring = text[range]
    print(substring)
}
```

When interoperating with APIs like `NSRegularExpression`, `NSAttributedString`, or older Objective-C text APIs, treat range conversion as a correctness boundary.

---

### Trap 4: Bridging arbitrary Swift values into Objective-C collections

Swift can sometimes box values so they can sit inside Objective-C containers. That does not mean Objective-C code understands those values.

Bad:

```swift
struct Payload {
    let id: Int
}

let payloads = [Payload(id: 1)]
legacyObjCAPI(payloads as NSArray)
```

This may produce opaque Swift boxes from the Objective-C side. A real Objective-C API expecting `NSString`, `NSNumber`, `NSURL`, or model objects will not magically understand arbitrary Swift structs.

Better:

```swift
struct PayloadDTO {
    let id: Int

    var foundationRepresentation: NSDictionary {
        ["id": id]
    }
}

let payloads = [PayloadDTO(id: 1)]
let bridged = payloads.map(\.foundationRepresentation) as NSArray
legacyObjCAPI(bridged)
```

Or better, expose a typed Swift adapter and hide Foundation representation entirely.

---

## 4. Direct answers to rubric questions

### Q1. What happens when Swift `String` or `Array` crosses into Objective-C APIs?

A Swift `String` crossing into Objective-C is bridged to `NSString`. A Swift `Array` crossing into Objective-C is bridged to `NSArray`, provided its elements can be represented in the Objective-C/Foundation world. The operation may be cheap, but it may also allocate, copy, or bridge each element.

For `String`, the boundary can change storage representation. A Swift `String` that originates from Objective-C may be backed by `NSString` storage, and Swift does not guarantee the same representation or efficiency as a native Swift string. The first mutation may need to copy into unique contiguous storage. ([GitHub, "String.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/String.swift))

For `Array`, if elements are already class instances or `@objc` protocol values, bridging to `NSArray` can be O(1). If the elements need to bridge — for example, `String` to `NSString` or `Int` to `NSNumber` — the bridge can be O(n) in time and space. ([GitHub, "Array.swift"](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift))

Interview version:

> When Swift values cross into Objective-C APIs, Swift bridges them to Foundation reference types. `String` can become `NSString`, and `Array` can become `NSArray`. The important part is that this is not always a free cast. The bridge may allocate, copy, box elements, or change storage representation. I treat it as a boundary conversion and keep it out of hot loops unless I have measured it.

---

### Q2. Why can bridging hide performance costs in hot paths?

Because the source code often looks like a normal call or cast:

```swift
legacy.process(items as NSArray)
```

But the runtime may be doing this:

```text
[String]      -> NSArray
String        -> NSString
Int           -> NSNumber
Swift storage -> Foundation storage
mutable copy  -> immutable copy
```

If this happens repeatedly in a scrolling cell, text layout path, JSON conversion layer, analytics pipeline, or image-processing loop, the cost can show up as allocation churn, ARC traffic, autoreleased Foundation objects, or unexpected O(n) copies.

Interview version:

> Bridging hides cost because the syntax is tiny but the operation can be large. A cast like `as NSArray` can force per-element bridging and allocation. In a hot path, I would inspect allocations and time profiles, then move the bridge to a narrow adapter boundary, bridge once, cache immutable bridged representations when safe, or replace the Objective-C-facing API with a Swift-native one.

---

## 5. Code probes

The rubric does not provide a code probe for E2, so these are added probes.

### Probe 1 — Explicit bridging

Given:

```swift
import Foundation

let names = ["A", "B"]
let nsNames = names as NSArray
let back = nsNames as! [String]

print(nsNames.count)
print(back)
```

Output:

```text
2
["A", "B"]
```

Why?

`names as NSArray` explicitly bridges the Swift array to a Foundation array. `nsNames as! [String]` bridges back to a Swift array with `String` elements.

The key detail is not the output. The key detail is that the operation may or may not be cheap depending on element representation.

---

### Probe 2 — No implicit assignment bridge

Given:

```swift
import Foundation

let names = ["A", "B"]
let nsNames: NSArray = names
```

Compiler error:

```text
error: cannot convert value of type '[String]' to specified type 'NSArray'
```

Why?

Modern Swift requires an explicit bridging coercion in this situation:

```swift
let nsNames: NSArray = names as NSArray
```

This keeps the boundary visible.

---

### Probe 3 — `String` vs `NSString` length

Given:

```swift
import Foundation

let s = "\u{1F1FA}\u{1F1F8}e\u{301}"
let ns = s as NSString

print(s.count)
print(ns.length)
```

Output:

```text
2
6
```

Why?

```text
Swift String.count counts Characters:
🇺🇸 = one Character
é  = one Character

NSString.length counts UTF-16 code units:
🇺🇸 = four UTF-16 code units
é  = two UTF-16 code units
```

This is a correctness issue, not just an API detail.

---

### Probe 4 — `NSMutableArray` does not become a live Swift value view

Given:

```swift
import Foundation

let mutable: NSMutableArray = ["A", "B"]

let snapshot = mutable as! [String]

mutable.add("C")

print(snapshot)
print(mutable.count)

var swift = mutable as! [String]
swift.append("D")

print(swift)
print(mutable.count)
```

Output:

```text
["A", "B"]
3
["A", "B", "C", "D"]
3
```

Why?

The Swift array behaves like a Swift value after bridging. Mutating the original `NSMutableArray` later does not mutate the already-bridged Swift array’s collection structure. Mutating the Swift array later does not mutate the `NSMutableArray`.

---

## 6. Exercise

### Problem

Investigate a performance issue around repeated bridging between Swift collections and Foundation APIs.

### Bad / naive version

```swift
import Foundation

final class LegacyRenderer {
    func render(_ items: NSArray) {
        // Objective-C-era rendering implementation.
    }
}

struct Row {
    let titles: [String]
}

func render(rows: [Row], renderer: LegacyRenderer) {
    for row in rows {
        renderer.render(row.titles as NSArray)
    }
}
```

### What is wrong?

```text
Every iteration may bridge [String] to NSArray.
Every String may bridge to NSString.
The renderer boundary is inside the hot loop.
The cost is invisible at the call site.
The design spreads Objective-C representation into Swift feature code.
```

In a scrolling UI, this can become allocation churn and frame-time instability.

### Improved version: bridge once at the boundary

```swift
import Foundation

final class LegacyRenderer {
    func render(_ items: NSArray) {
        // Objective-C-era rendering implementation.
    }
}

struct Row {
    let titles: [String]
}

struct LegacyRendererAdapter {
    private let renderer: LegacyRenderer

    init(renderer: LegacyRenderer) {
        self.renderer = renderer
    }

    func render(row: Row) {
        let bridgedTitles = row.titles as NSArray
        renderer.render(bridgedTitles)
    }
}
```

This is better because the bridge is localized. Swift app code stays Swift-native.

### Improved version: pre-bridge reused data

```swift
import Foundation

struct Row {
    let titles: [String]
}

struct PreparedRow {
    let titles: NSArray
}

func prepare(rows: [Row]) -> [PreparedRow] {
    rows.map { row in
        PreparedRow(titles: row.titles as NSArray)
    }
}

func render(preparedRows: [PreparedRow], renderer: LegacyRenderer) {
    for row in preparedRows {
        renderer.render(row.titles)
    }
}
```

This is appropriate when:

```text
The rows are reused.
The data is immutable for the rendering lifetime.
The Objective-C API cannot be changed.
Profiling shows bridging cost matters.
```

### Better architectural fix: stop leaking Foundation types

```swift
protocol RowRendering {
    func render(titles: [String])
}

final class LegacyRendererAdapter: RowRendering {
    private let renderer: LegacyRenderer

    init(renderer: LegacyRenderer) {
        self.renderer = renderer
    }

    func render(titles: [String]) {
        renderer.render(titles as NSArray)
    }
}
```

Now the rest of the app depends on:

```swift
func render(titles: [String])
```

not:

```swift
func render(_ items: NSArray)
```

This improves API design and keeps Foundation bridging behind an adapter.

### Investigation plan

Use this checklist:

```text
1. Reproduce in a Release build, not Debug.
2. Add signposts around the suspected hot path.
3. Use Instruments Time Profiler.
4. Use Allocations to look for NSArray, NSDictionary, NSString, NSNumber, _SwiftValue, CFArray, CFString.
5. Search stacks for bridging-related functions or Objective-C retain/copy activity.
6. Count how often the bridge happens per frame, per cell, per request, or per event.
7. Move bridging outward and compare measurements.
8. Keep the simpler design unless profiling proves the bridge matters.
```

---

## 7. Production guidance

Use bridging in production when:

```text
You call Apple/Foundation APIs that still require Objective-C types.
You maintain Objective-C-compatible SDK surfaces.
You isolate legacy Objective-C components behind Swift adapters.
You need Foundation-specific behavior, such as NSString APIs, NSAttributedString, NSRegularExpression, or NSDictionary-based APIs.
```

Be careful when:

```text
The bridge is inside a loop.
The bridge is inside scrolling, layout, rendering, parsing, logging, or analytics hot paths.
The collection contains value types that must become Foundation objects.
The API uses NSMutableArray / NSMutableDictionary and callers expect live mutation.
The API uses NSString ranges, NSRange, or UTF-16 offsets.
The API accepts Any / id and may receive opaque boxed Swift values.
```

Avoid when:

```text
A Swift-native API would be clearer.
You are using NSArray / NSDictionary only out of habit.
You are exposing Foundation types from a public Swift SDK without a real interop reason.
You are hiding an untyped data model behind [String: Any] or NSDictionary.
```

Debugging checklist:

```text
Is this bridge explicit or hidden through an imported Objective-C API?
Is the bridged value created once or repeatedly?
Are elements already Objective-C objects, or must each element bridge?
Is mutability expected to be shared?
Are NSString/NSRange semantics being mixed with Swift String.Index?
Does Instruments show allocation churn from Foundation containers or boxed values?
Can this bridge move to an adapter boundary?
Can the Objective-C API be wrapped with a Swift-native surface?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift bridges types like `String` to `NSString` and `Array` to `NSArray` so Swift can call Objective-C APIs.

### Senior answer

> Swift bridges standard-library value types to Foundation reference types, but the bridge can allocate, copy, or change semantics. I watch for this especially with collections, strings, mutability, and range conversions. In hot paths, I measure rather than assume.

### Staff-level answer

> Bridging is an architectural boundary. I do not let `NSArray`, `NSDictionary`, `NSString`, or `AnyObject` leak through Swift app layers unless there is a deliberate interop reason. I isolate legacy Foundation APIs behind adapters, preserve Swift-native types internally, measure bridging cost in Release builds, and only optimize after evidence. For SDKs, I also treat Foundation exposure as public API debt because it locks clients into Objective-C-era semantics.

Staff-level questions to ask:

```text
Where is the interop boundary supposed to live?
Are we bridging once at the edge or repeatedly in feature code?
Does this public API need Foundation types, or can it expose Swift-native types?
Are we depending on NSMutableArray / NSMutableDictionary shared mutation?
Are String ranges crossing between Swift Character semantics and NSString UTF-16 semantics?
```

---

## 9. Interview-ready summary

Foundation bridging lets Swift interoperate with Objective-C APIs by converting Swift value types like `String`, `Array`, `Dictionary`, and `Set` to Foundation reference types like `NSString`, `NSArray`, `NSDictionary`, and `NSSet`. The dangerous assumption is that this is just a cast. It can allocate, copy, box elements, change storage representation, or change mutability and indexing assumptions. I keep bridging visible, isolate it at adapter boundaries, avoid it in hot loops, and use Instruments to verify whether it matters before complicating the design.

---

## 10. Flashcards

Q: What is Foundation bridging in Swift?  
A: The conversion layer between Swift standard-library value types and Objective-C/Foundation reference types, such as `String` ⇄ `NSString` and `Array` ⇄ `NSArray`.

Q: Is `array as NSArray` always cheap?  
A: No. It can be O(1) when elements are already class or `@objc` protocol values, but it can be O(n) time and space when elements must bridge.

Q: Why is `String` ⇄ `NSString` bridging semantically risky?  
A: Swift `String` works in `Character` / grapheme-cluster terms, while `NSString` APIs commonly use UTF-16 offsets and `NSRange`.

Q: Does bridging `NSMutableArray` to `[T]` create a live Swift view?  
A: No. Treat the bridged Swift array as Swift value-semantic collection membership, not as a live mutable view over the Objective-C collection.

Q: Why can bridging hurt scrolling performance?  
A: Repeated bridges can allocate Foundation containers and bridge every element, causing allocation churn and extra ARC work per frame.

Q: Where should bridging live in a clean Swift codebase?  
A: At narrow interop boundaries, usually inside adapters around legacy Objective-C/Foundation APIs.

Q: What should you look for in Instruments?  
A: Repeated allocations of `NSArray`, `NSDictionary`, `NSString`, `NSNumber`, `_SwiftValue`, `CFArray`, `CFString`, and stacks related to bridging/copying.

Q: What is the senior-level rule for Foundation types in Swift APIs?  
A: Prefer Swift-native types internally. Expose Foundation types only when interop, reference semantics, or Foundation-specific behavior is intentional.

---

## 11. Related sections

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A8 — Closures, escaping, and capture semantics]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C4 — String and Unicode correctness]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[E1 — Objective-C interoperability]]
- [[E3 — C interoperability and unsafe boundaries]]
- [[E6 — Import visibility and dependency leakage]]
- [[F3 — Performance investigation habits]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — E2 Foundation and Objective-C bridging behavior.
- Apple Developer. "Classes Bridged to Swift Standard Library Value Types." Apple Developer Documentation. https://developer.apple.com/documentation/foundation/classes-bridged-to-swift-standard-library-value-types
- GitHub. "Array.swift." swiftlang/swift. https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift
- GitHub. "String.swift." swiftlang/swift. https://github.com/swiftlang/swift/blob/main/stdlib/public/core/String.swift
- Apple Developer. "NSArray." Apple Developer Documentation. https://developer.apple.com/documentation/foundation/nsarray
- Apple Developer. "Dictionary." Apple Developer Documentation. https://developer.apple.com/documentation/swift/dictionary
- GitHub. "Fully Eliminate Implicit Bridging Conversions from Swift." Swift Evolution SE-0072. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0072-eliminate-implicit-bridging-conversions.md
