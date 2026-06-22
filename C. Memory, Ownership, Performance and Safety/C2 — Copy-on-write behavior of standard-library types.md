---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - copy-on-write
---
## 0. Rubric snapshot

**Rubric expectation**

Understand uniqueness checks, shared storage, and when mutation triggers actual copying. The rubric specifically asks you to explain why passing an `Array` around can be cheap until mutation, why `ArraySlice` can create hidden retention or memory-footprint issues, and how to diagnose a small slice keeping a huge buffer alive.

**Caveats**

- Bridging to Objective-C can defeat normal performance expectations.
- Slices often share backing storage.
- CoW is an implementation strategy used by specific types; it is not the same thing as value semantics.

**You should be able to answer**

- Why can passing an `Array` around be cheap until mutation?
- What hidden retention or memory-footprint issues can `ArraySlice` create?

**You should be able to do**

- Diagnose why holding onto a small slice of a huge array keeps the whole buffer alive.

---

## 1. Core mental model

Copy-on-write is how Swift gives large value types value semantics without eagerly copying large storage every time you assign, pass, or return them.

An `Array` is a value. Assigning it to another variable creates another logical value. But the element buffer does not necessarily get copied immediately. Instead, multiple `Array` values may point to the same storage. As long as they are only read, this is fine because nobody can observe a difference.

When one of those array values is mutated, Swift checks whether the storage is uniquely referenced. If it is unique, Swift mutates in place. If it is shared, Swift first creates a separate buffer, then mutates that buffer. The public behavior is still value semantics: mutating `b` does not mutate `a`.

Apple’s standard library documentation describes this directly: arrays are value types, mutating a shared array incurs copying, and a uniquely owned array can mutate in place. The Swift source docs also call out that standard-library arrays use CoW so passing an array argument can drop from an O(n) copy to O(1) storage sharing. ([GitHub](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift "swift/stdlib/public/core/Array.swift at main · swiftlang/swift · GitHub"))

The key idea:

```text
Array value copy ≠ immediate buffer copy.
Read-only sharing is cheap.
Mutation requires unique storage; otherwise Swift copies first.
```

Swift guarantees the semantic behavior: separate array values behave independently after mutation. Swift does **not** guarantee that every assignment is physically cheap, that every mutation is O(1), or that you can rely on storage identity. CoW is an optimization detail with observable performance and memory consequences, not an API contract you should poke with pointer identity tests.

---

## 2. Essential mechanics

### Mechanic 1: Assignment usually shares storage until mutation

```swift
var a = [1, 2, 3]
var b = a

b.append(4)

print(a)
print(b)
```

Output:

```text
[1, 2, 3]
[1, 2, 3, 4]
```

`a` and `b` are separate values. The append to `b` does not affect `a`.

Conceptually:

```text
Before mutation:

a ─┐
   ├── shared buffer [1, 2, 3]
b ─┘

After b.append(4):

a ─── buffer [1, 2, 3]
b ─── buffer [1, 2, 3, 4]
```

The important distinction is semantic vs physical copying. Semantically, `b = a` creates a new value. Physically, the buffer can be shared until mutation.

---

### Mechanic 2: CoW copies the collection storage, not necessarily the elements’ identities

This matters when the array holds reference types.

```swift
final class Box {
    var value: Int

    init(_ value: Int) {
        self.value = value
    }
}

var a = [Box(1)]
var b = a

b[0].value = 2

print(a[0].value, b[0].value)

b.append(Box(3))

print(a.count, b.count)
print(a[0].value, b[0].value)
```

Output:

```text
2 2
1 2
2 2
```

`b[0].value = 2` mutates the `Box` object, not the array’s structure. Both arrays still contain a reference to the same `Box`.

`b.append(Box(3))` mutates `b`’s array storage. At that point, CoW preserves independent array values, so `a.count` remains `1` and `b.count` becomes `2`.

The array has value semantics for its **collection structure**. It does not deep-copy reference-typed elements for you.

---

### Mechanic 3: Mutating shared or bridged storage can be O(n)

Array element reads are O(1). Writes are O(1) only when the array has uniquely owned native storage. The standard library documentation says writing can become O(n) when storage is shared with another array or when the array uses bridged `NSArray` storage. ([GitHub](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift "swift/stdlib/public/core/Array.swift at main · swiftlang/swift · GitHub"))

```swift
var original = Array(0..<1_000_000)
var copy = original

// This may require copying the million-element buffer first.
copy[0] = -1
```

From a production perspective, the problematic case is not “arrays are slow.” The real problem is accidental repeated mutation of non-unique storage in a hot path.

Bad pattern:

```swift
func appendOne(_ array: [Int], value: Int) -> [Int] {
    var copy = array
    copy.append(value)
    return copy
}

var result: [Int] = []
for i in 0..<10_000 {
    result = appendOne(result, value: i)
}
```

Better:

```swift
var result: [Int] = []
result.reserveCapacity(10_000)

 for i in 0..<10_000 {
    result.append(i)
}
```

The better version makes ownership and mutation local. It avoids repeated value reassignment patterns that can interfere with in-place growth and optimization.

---

### Mechanic 4: `ArraySlice` is a view over larger storage

`ArraySlice` is designed to avoid copying. The Swift standard-library docs say `ArraySlice` presents a view onto the storage of a larger array instead of copying slice elements into new storage. ([GitHub](https://github.com/apple/swift/blob/main/stdlib/public/core/ArraySlice.swift "swift/stdlib/public/core/ArraySlice.swift at main · swiftlang/swift · GitHub"))

```swift
let source = Array(10...20)
let slice = source[3...5]

print(slice.startIndex, slice.endIndex, slice.count)
print(slice[slice.startIndex])
```

Output:

```text
3 6 3
13
```

Two important details:

```text
source: [10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
index:    0   1   2   3   4   5   6   7   8   9   10

slice = source[3...5]
slice elements: [13, 14, 15]
slice indices:    3   4   5
```

An `ArraySlice` does not necessarily start at index `0`. Use `slice.startIndex`, `slice.endIndex`, and collection APIs instead of assuming zero-based indexing.

---

## 3. Common traps and misconceptions

### Trap 1: “CoW means all structs copy lazily”

No. CoW is not automatic for all structs. It is a design used by types such as `Array`, `Dictionary`, `Set`, and `String`. Your custom structs do not magically become CoW unless you implement that storage strategy yourself.

Bad mental model:

```swift
struct BigModel {
    var values: (Int, Int, Int, Int)
}

// Do not assume Swift secretly creates reference storage and uniqueness checks
// for arbitrary structs like this.
```

Better mental model:

```swift
struct BigBuffer {
    private final class Storage {
        var values: [Int]

        init(values: [Int]) {
            self.values = values
        }

        func copy() -> Storage {
            Storage(values: values)
        }
    }

    private var storage: Storage

    init(values: [Int]) {
        self.storage = Storage(values: values)
    }

    var values: [Int] {
        storage.values
    }

    mutating func append(_ value: Int) {
        if !isKnownUniquelyReferenced(&storage) {
            storage = storage.copy()
        }

        storage.values.append(value)
    }
}
```

`isKnownUniquelyReferenced(_:)` exists specifically to help implement CoW storage for value types that internally hold reference storage. Apple’s documentation describes it as useful for implementing copy-on-write optimization for deep storage of value types. ([Apple Developer](https://developer.apple.com/documentation/swift/isknownuniquelyreferenced%28_%3A%29-5kvtu?utm_source=chatgpt.com "isKnownUniquelyReferenced(_:)"))

---

### Trap 2: “Array of classes gives deep value semantics”

An array of class instances gives value semantics for the array container, not for the objects inside it.

Bad:

```swift
final class User {
    var name: String

    init(name: String) {
        self.name = name
    }
}

let users = [User(name: "Aykut")]
var copy = users

copy[0].name = "Changed"

// users[0].name is also "Changed"
```

Better, when independent mutation is required:

```swift
struct User {
    var name: String
}

let users = [User(name: "Aykut")]
var copy = users

copy[0].name = "Changed"

// users[0].name remains "Aykut"
```

Or, if identity is intentional, model it explicitly and do not sell the API as deep value-semantic.

---

### Trap 3: Storing `ArraySlice` as long-lived state

`ArraySlice` is efficient for temporary views. It is risky as long-lived storage. The standard library warns that long-term storage of `ArraySlice` is discouraged because a slice holds a reference to the entire storage of the larger array, not just the visible elements, and can prolong the lifetime of otherwise inaccessible elements. ([GitHub](https://github.com/apple/swift/blob/main/stdlib/public/core/ArraySlice.swift "swift/stdlib/public/core/ArraySlice.swift at main · swiftlang/swift · GitHub"))

Bad:

```swift
final class SearchCache {
    private var topResults: ArraySlice<Result> = []

    func update(with results: [Result]) {
        topResults = results.prefix(10)
    }
}
```

Better:

```swift
final class SearchCache {
    private var topResults: [Result] = []

    func update(with results: [Result]) {
        topResults = Array(results.prefix(10))
    }
}
```

The better version intentionally copies the small subset and releases the large original buffer when nothing else needs it.

---

### Trap 4: Testing CoW by relying on pointer identity

You can sometimes observe buffer addresses with unsafe APIs, but this is not a stable semantic contract for application logic.

Bad:

```swift
// Avoid tests that assert exact storage identity or address behavior.
// Optimizer behavior, bridging, capacity, and implementation details can change.
```

Better:

```swift
// Test value semantics:
var a = [1, 2, 3]
var b = a
b.append(4)

assert(a == [1, 2, 3])
assert(b == [1, 2, 3, 4])
```

For performance questions, benchmark and profile. Do not infer too much from one pointer experiment.

---

## 4. Direct answers to rubric questions

### Q1. Why can passing an `Array` around be cheap until mutation?

Because `Array` is a value type with copy-on-write storage. Assigning or passing an array can create a new logical value that shares the same underlying buffer. As long as the array is only read, no element buffer copy is needed. When a mutation happens, Swift checks whether the storage is uniquely owned. If not, it copies before mutating.

Interview version:

> `Array` has value semantics but uses copy-on-write storage. Passing or assigning it usually shares the underlying buffer, so the operation can be cheap. The semantic copy exists immediately, but the physical element copy is deferred until mutation. On mutation, Swift checks storage uniqueness: unique storage mutates in place; shared storage is copied first so the other array values remain unchanged.

---

### Q2. What hidden retention or memory-footprint issues can `ArraySlice` create?

An `ArraySlice` may keep the entire original array storage alive, even if the slice exposes only one or two elements. This is intentional: the slice is a view over a subrange of the original buffer, so creating it can be cheap. The cost is that storing the slice long-term can retain a huge backing buffer and all its elements.

Interview version:

> `ArraySlice` is great as a temporary view, but it is dangerous as stored state. A tiny slice can retain the whole backing array buffer. If I keep `array.prefix(10)` from a huge response in a cache, I may accidentally keep the whole response alive. If the slice needs to outlive the original processing step, I usually convert it to `Array(slice)` to make the copy explicit.

---

## 5. Code probe

The rubric has no dedicated C2 code probe, so use these three probes instead.

### Probe A: Minimal CoW behavior

Given:

```swift
var a = [1, 2, 3]
var b = a

b.append(4)

print(a)
print(b)
```

Output:

```text
[1, 2, 3]
[1, 2, 3, 4]
```

Why:

```text
a and b may initially share storage.
b.append mutates b.
Because storage is shared, b receives unique storage before mutation.
a remains unchanged.
```

---

### Probe B: Counterexample — reference elements are still shared

Given:

```swift
final class Box {
    var value: Int

    init(_ value: Int) {
        self.value = value
    }
}

var a = [Box(1)]
var b = a

b[0].value = 2

print(a[0].value, b[0].value)

b.append(Box(3))

print(a.count, b.count)
print(a[0].value, b[0].value)
```

Output:

```text
2 2
1 2
2 2
```

Why:

```text
b[0].value = 2 mutates the shared Box object.
It does not structurally mutate the array buffer.

b.append(Box(3)) structurally mutates b's array.
That triggers normal CoW behavior for the array storage.
```

Redesign when you need deep value behavior:

```swift
struct Box {
    var value: Int
}

var a = [Box(value: 1)]
var b = a

b[0].value = 2

print(a[0].value, b[0].value)
```

Output:

```text
1 2
```

Why this fix is correct:

The element itself is now a value. The array’s value semantics and the element’s value semantics align.

---

### Probe C: Production bug — tiny slice retains whole storage

Given:

```swift
final class Item {
    let id: Int

    init(_ id: Int) {
        self.id = id
    }

    deinit {
        print("deinit \(id)")
    }
}

var storedSlice: ArraySlice<Item>?

func makeSlice() {
    let items = (0..<3).map(Item.init)
    storedSlice = items[1...1]
    print("slice count", storedSlice!.count)
}

makeSlice()
print("after makeSlice")
storedSlice = nil
print("after nil")
```

Output:

```text
slice count 1
after makeSlice
deinit 0
deinit 1
deinit 2
after nil
```

Why:

```text
storedSlice exposes only item 1.
But it keeps the original array storage alive.
The storage still retains items 0, 1, and 2.
Only after storedSlice is released can all elements deinitialize.
```

Fix:

```swift
var storedItems: [Item]?

func makeArrayCopy() {
    let items = (0..<3).map(Item.init)
    storedItems = Array(items[1...1])
}
```

Why this fix is correct:

`Array(items[1...1])` creates a new array containing only the visible slice elements. It intentionally pays a small copy cost to avoid retaining the larger original buffer.

Alternative fixes and tradeoffs:

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Keep `ArraySlice`|Short-lived algorithmic view inside one function|Can retain large backing storage if it escapes|
|Convert to `Array(slice)`|Long-lived storage, cache, model state, API boundary|Copies visible elements|
|Change API to accept generic `Collection`/`SubSequence`|Library algorithms that should avoid copying|More complex generic constraints|
|Use indices/ranges into known owner|You control the owner lifetime explicitly|More bookkeeping and lifetime coupling|

---

## 6. Exercise

### Problem

Diagnose why holding onto a small slice of a huge array keeps the whole buffer alive.

### Bad / naive version

```swift
struct ImageSearchResult {
    let id: String
    let thumbnailData: [UInt8]
}

@MainActor
final class SearchViewModel {
    private(set) var visibleResults: ArraySlice<ImageSearchResult> = []

    func update(results: [ImageSearchResult]) {
        visibleResults = results.prefix(20)
    }
}
```

### What is wrong?

```text
results.prefix(20) returns an ArraySlice.
The view model stores that slice.
The slice may retain the entire storage of results.
If results contains thousands of large thumbnail payloads, keeping 20 visible elements can accidentally keep the full response buffer alive.
```

The bug can look like:

```text
Network response decoded.
UI stores only first 20 results.
Memory does not drop after leaving the loading scope.
Memory graph shows many ImageSearchResult / Data / buffer objects still retained.
The retaining path goes through ArraySlice storage.
```

### Improved version

```swift
struct ImageSearchResult {
    let id: String
    let thumbnailData: [UInt8]
}

@MainActor
final class SearchViewModel {
    private(set) var visibleResults: [ImageSearchResult] = []

    func update(results: [ImageSearchResult]) {
        visibleResults = Array(results.prefix(20))
    }
}
```

### Why this is better

This makes the lifetime boundary explicit. The view model owns exactly the elements it intends to display. The large response array can be released after `update(results:)` returns, assuming no other references keep it alive.

For performance-sensitive code, this is the right tradeoff: copy 20 elements now to avoid retaining potentially thousands of elements indefinitely.

### Production variant

If the elements themselves hold large reference storage, copying the array copies element values but may not deep-copy all underlying payloads. For example, if each result contains reference-backed image data, you still need to think about whether each visible result intentionally owns that data.

Better model:

```swift
struct ImageSearchResultSummary: Identifiable {
    let id: String
    let title: String
    let thumbnailURL: URL
}

@MainActor
final class SearchViewModel {
    private(set) var visibleResults: [ImageSearchResultSummary] = []

    func update(results: [ImageSearchResultSummary]) {
        visibleResults = Array(results.prefix(20))
    }
}
```

This avoids storing heavyweight payloads in UI state unless the UI truly needs them.

---

## 7. Production guidance

Use CoW-friendly standard-library types in production when:

```text
You want value semantics with efficient passing.
You have large collections that are commonly read and occasionally mutated.
You want local reasoning: mutation of one variable should not mutate another logical value.
You are building APIs where callers should not care about storage ownership.
```

Be careful when:

```text
Mutating arrays inside tight loops after repeated assignment.
Storing ArraySlice in properties, caches, async state, or view models.
Arrays contain reference types whose interior state can be mutated.
Bridging to or from Objective-C/Foundation collection APIs.
Measuring performance in debug builds.
Relying on storage identity or pointer addresses.
```

Avoid when:

```text
You need identity semantics.
You need guaranteed in-place mutation across multiple aliases.
You need deep-copy behavior for reference-typed elements.
You are using ArraySlice as long-lived storage just to avoid a small copy.
```

Debugging checklist:

```text
Is this type actually CoW, or am I assuming it is?
Is mutation structural mutation of the collection, or interior mutation of referenced elements?
Is the array uniquely owned at the mutation point?
Is a small ArraySlice stored in a long-lived object?
Did prefix/drop/suffix create a slice that escaped?
Is Objective-C bridging involved?
Is the performance issue measured in release with Instruments?
Would Array(slice) intentionally reduce retained memory?
Would reserveCapacity or in-place mutation remove repeated reallocations/copies?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift arrays are value types. When you copy them, changes to one do not affect the other.

### Senior answer

> Swift arrays have value semantics implemented with copy-on-write storage. Assignment or argument passing can share a buffer until mutation. Mutation checks uniqueness and copies if storage is shared. I’m careful with arrays of reference types because CoW does not deep-copy the objects inside the array.

### Staff-level answer

> CoW is a storage strategy that lets APIs expose value semantics without eagerly copying large buffers. I design around semantic ownership first, then validate hot paths with Instruments. I avoid long-lived `ArraySlice` state because it can retain entire buffers. I also watch for bridging, repeated mutation of non-unique storage, and arrays of reference types where the container is value-semantic but the elements are identity-semantic. At API boundaries, I choose whether to preserve a slice view, copy to `Array`, or generalize over `Collection` based on lifetime and performance requirements.

Staff-level questions to ask:

```text
Does this API promise value semantics or identity semantics?
Who owns this buffer after this function returns?
Can this slice escape into long-lived state?
Are we optimizing away a copy at the cost of retaining a huge buffer?
Are we measuring release performance, or guessing from debug behavior?
Are reference-typed elements undermining the value-semantic story?
Does Objective-C bridging change allocation or mutation complexity?
```

---

## 9. Interview-ready summary

Swift standard-library collections such as `Array` use copy-on-write to preserve value semantics efficiently. Copying an array value often shares the underlying buffer; mutation checks whether the buffer is uniquely referenced and copies only if needed. This makes passing arrays cheap in many read-heavy paths, but mutation of shared or bridged storage can become O(n). `ArraySlice` is another important caveat: it is a cheap view into a larger buffer and may retain the whole original storage, so it is good for temporary algorithmic work but usually wrong for long-lived state unless converted to `Array`.

---

## 10. Flashcards

Q: What does copy-on-write mean for `Array`?

A: Multiple array values can share the same storage until one is mutated; mutation copies first if the storage is not uniquely owned.

Q: Does CoW mean `Array` assignment has reference semantics?

A: No. The public semantics are still value semantics. Storage sharing is an optimization that must not make mutation visible through another array value.

Q: Does an array of classes deep-copy its elements on mutation?

A: No. CoW applies to the array’s storage. Referenced objects inside the array can still be shared and mutated.

Q: Why can `ArraySlice` cause memory retention issues?

A: A slice can hold a reference to the entire original array storage, not only the visible elements.

Q: When should you convert `ArraySlice` to `Array`?

A: When the slice escapes into long-lived storage, crosses an API boundary where ownership should be independent, or would otherwise retain a large backing buffer.

Q: What does `isKnownUniquelyReferenced(_:)` help with?

A: Implementing custom CoW types that store their data behind a private reference object.

Q: Why can Objective-C bridging affect array performance?

A: Bridged `NSArray` storage can change copy and mutation costs; Swift array writes can become O(n) when storage is shared or bridged.

Q: Is CoW automatic for every Swift struct?

A: No. It must be implemented by the type, usually with reference storage and uniqueness checks.

---

## 11. Related sections

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[C1 — ARC fundamentals]]
- [[C3 — Exclusivity enforcement and inout]]
- [[C5 — Collection protocols, complexity, and index invalidation]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[E2 — Foundation and Objective-C bridging behavior]]

---

## 12. Sources

- Swift Senior/Staff Rubric — C2 Copy-on-write behavior of standard-library types.
- Swift standard library `Array` documentation/source comments on shared storage, unique mutation, bridging, and write complexity. ([GitHub](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift "swift/stdlib/public/core/Array.swift at main · swiftlang/swift · GitHub"))
- Swift standard library `ArraySlice` documentation/source comments on slice views, retained backing storage, and slice indices. ([GitHub](https://github.com/apple/swift/blob/main/stdlib/public/core/ArraySlice.swift "swift/stdlib/public/core/ArraySlice.swift at main · swiftlang/swift · GitHub"))
- Swift optimization tips on using CoW semantics for large values and implementing custom CoW storage. ([GitHub](https://github.com/swiftlang/swift/blob/main/docs/OptimizationTips.rst "swift/docs/OptimizationTips.rst at main · swiftlang/swift · GitHub"))