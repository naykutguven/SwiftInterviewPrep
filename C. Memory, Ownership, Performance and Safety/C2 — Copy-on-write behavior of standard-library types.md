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

# C2 — Copy-on-Write Behavior of Standard-Library Types

**Learning goals:**

- Separate independent collection values from shared implementation storage and shared reference elements.
- Explain write/growth costs, slice indices/retention, and the limits of an owned slice copy.
- Audit every private CoW mutation path and choose copyable snapshots versus exclusive storage deliberately.

**Prerequisites:** [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]/[[A2 — `let`, `var`, `mutating`, and method semantics]] for value versus reference mutation, [[C1 — ARC fundamentals]] for weak lifetime probes, and [[C3 — Exclusivity enforcement and `inout`]] before unsafe/exclusive mutation boundaries.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Saving an Array before editing it gives the program two independent sequence values. **Copy-on-write (CoW)** lets an implementation share storage until mutation requires separation. That is an implementation strategy for preserving the value contract, not permission for one array value to change another.

Passing an array can therefore be cheap, while an element write may need to separate shared storage and an append may need to grow capacity. Either can do O(n) element work. Unique storage alone does not guarantee room for another element. A slice also keeps its original index range and can retain a much larger backing allocation than its visible count. [ArraySlice](https://developer.apple.com/documentation/swift/arrayslice)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Test logical independence before asking about allocation

Standalone value probe. Retaining the earlier array makes the before/after contract explicit.

```swift
let saved = [1, 2, 3]
var edited = saved
edited[0] = 9
edited.append(4)
precondition(saved == [1, 2, 3])
precondition(edited == [9, 2, 3, 4])
```

**Walkthrough:** The saved sequence stays unchanged through both a replacement and growth. These assertions establish values. They do not count buffers, copies, retains, or bytes; optimizer and representation choices need separate evidence.

### An array copies references without cloning their objects

Standalone reference-element example. Compare editing an element object with replacing the array’s element slot.

```swift
final class Item {
    var score: Int
    init(_ score: Int) { self.score = score }
}
let original = [Item(1)]
var copy = original
copy[0].score = 2
precondition(original[0].score == 2 && original[0] === copy[0])
copy[0] = Item(9)
precondition(original[0].score == 2 && copy[0].score == 9)
precondition(original[0] !== copy[0])
```

**Walkthrough:** Both sequences initially contain the same object reference. Changing that object is not an Array element-slot write. Replacing the copy’s slot changes only its sequence value. Deep independence needs a payload-specific copying contract.

### Shared retention setup

RetentionFixture stores either a one-element slice or an owned selected array. WeakItem probes observe objects without retaining them. The noninlined initializer ends the source array’s setup lifetime before observations begin. This is a bounded runtime experiment; it does not prescribe every optimizer’s destruction order or physical memory behavior.

```swift
final class Item {
    let id: Int
    init(_ id: Int) { self.id = id }
}

struct WeakItem {
    weak var value: Item?
    init(_ value: Item) { self.value = value }
}

final class RetentionFixture {
    let probes: [WeakItem]
    var slice: ArraySlice<Item>?
    var owned: [Item]?

    @inline(never)
    init(keepSlice: Bool) {
        let items = (0..<3).map(Item.init)
        probes = items.map(WeakItem.init)
        if keepSlice {
            slice = items[1..<2]
        } else {
            owned = Array(items[1..<2])
        }
    }

    func alive() -> [Bool] { probes.map { $0.value != nil } }
}
```

## Rules and pitfalls

### A tiny visible slice can retain invisible elements

**Common pitfall:** Keeping a one-element ArraySlice as long-lived state and expecting it to keep only one object. Prepend RetentionFixture setup.

**Mistake:** Standalone runnable example.

```swift
let selected = RetentionFixture(keepSlice: true)
precondition(selected.slice?.map(\.id) == [1])
precondition(selected.alive() == [true, true, true])
selected.slice = nil
precondition(selected.alive() == [false, false, false])
```

**Consequence:** Only item 1 is visible, but all three source objects remain live in this checked fixture until the slice is released.

**Why:** The slice can keep the base allocation alive, and that allocation retains its other initialized reference elements.

**Correction:** When retaining only the selected sequence is the intended contract, materialize Array(slice). Prepend RetentionFixture setup.

```swift
let selected = RetentionFixture(keepSlice: false)
precondition(selected.owned?.map(\.id) == [1])
precondition(selected.alive() == [false, true, false])
selected.owned = nil
precondition(selected.alive() == [false, false, false])
```

**What changes:** The owned selected array retains only its visible Item in this observation. That Item can still retain a large reachable graph. Allocation and resident-memory effects need actual measurements; visible count and weak probes do not establish them.

### One missed mutation path breaks a custom value contract

**Common pitfall:** Copying a struct whose private reference storage is mutated without a uniqueness policy.

**Mistake:** Standalone runnable example.

```swift
struct SharedList {
    private final class Storage {
        var values: [Int]
        init(_ values: [Int]) { self.values = values }
    }
    private var storage: Storage
    init(_ values: [Int]) { storage = Storage(values) }
    var values: [Int] { storage.values }
    mutating func append(_ value: Int) { storage.values.append(value) }
}
let original = SharedList([1])
var copy = original
copy.append(2)
precondition(original.values == [1, 2])  // Unexpected for an independent value.
```

**Consequence:** The original wrapper observes the copy’s append, despite being a let value.

**Why:** Both wrappers own the same Storage object. Array’s inner CoW cannot separate two aliases that mutate that very same stored Array variable.

**Correction:** Keep Storage private and establish uniqueness before every mutation. This shared CowList setup demonstrates setter, append, removal, and inout projection paths.

```swift
struct CowList {
    private final class Storage {
        var values: [Int]
        init(_ values: [Int]) { self.values = values }
    }
    private var storage: Storage
    init(_ values: [Int]) { storage = Storage(values) }
    var values: [Int] { storage.values }
    private mutating func ensureUnique() {
        if !isKnownUniquelyReferenced(&storage) {
            storage = Storage(storage.values)
        }
    }
    subscript(index: Int) -> Int {
        get { storage.values[index] }
        set {
            ensureUnique()
            storage.values[index] = newValue
        }
    }
    mutating func append(_ value: Int) {
        ensureUnique()
        storage.values.append(value)
    }
    mutating func removeLast() -> Int {
        ensureUnique()
        return storage.values.removeLast()
    }
    mutating func withElement<R>(at index: Int, _ body: (inout Int) throws -> R) rethrows -> R {
        ensureUnique()
        return try body(&storage.values[index])
    }
}
func checkCowList() {
    let original = CowList([1, 2])
    var edited = original
    edited[0] = 9
    precondition(edited.values == [9, 2] && original.values == [1, 2])
    var appended = original
    appended.append(3)
    precondition(appended.values == [1, 2, 3] && original.values == [1, 2])
    var removed = original
    let last = removed.removeLast()
    precondition(last == 2 && removed.values == [1] && original.values == [1, 2])
    var projected = original
    projected.withElement(at: 1) { $0 += 10 }
    precondition(projected.values == [1, 12] && original.values == [1, 2])
}
checkCowList()
```

**What changes:** The outer storage is separated before mutation; its Array then supplies its own inner value semantics. The getter returns an Array value, never the mutable Storage reference. Preconditions on valid indices/nonempty removal still apply. A real implementation must audit every newly added mutation path.

### A uniqueness check does not serialize concurrent access

**Common pitfall:** Treating isKnownUniquelyReferenced as a lock. Compile-only: the unsynchronized shared object below is not used concurrently.

**Mistake:** Compile-only; the unsafe path is not executed.

```swift
final class SharedStore: @unchecked Sendable {
    private final class Storage { var values: [Int] = [] }
    private var storage = Storage()
    func append(_ value: Int) {
        if !isKnownUniquelyReferenced(&storage) { storage = Storage() }
        storage.values.append(value)
    }
}
```

**Consequence:** Concurrent callers could race on the reference, uniqueness decision, and array mutation. The unchecked Sendable promise does not supply protection.

**Why:** The uniqueness query concerns ownership at that moment. It is not a critical section and does not make an otherwise shared mutable variable safe.

**Correction:** Give shared mutation an explicit owner. This actor keeps its Array private and serializes each append.

```swift
actor Store {
    private var values: [Int] = []
    func append(_ value: Int) { values.append(value) }
    func snapshot() -> [Int] { values }
}
let store = Store()
await withTaskGroup(of: Void.self) { group in
    group.addTask { await store.append(1) }
    group.addTask { await store.append(2) }
}
let snapshot = await store.snapshot()
precondition(snapshot.sorted() == [1, 2])
```

**What changes:** The actor protects shared mutation, while the returned Array remains a value snapshot. The group joins both operations; insertion order is intentionally unspecified. [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks|D12]] supplies the synchronous Mutex alternative.

## Worked example

### An owned slice copy still shares visible object payloads

**Setup:** Standalone example. Array(selected) selects independent sequence storage semantics; it does not clone an Item.

**Predict:** Which edit affects the source, and which edit changes only the selected sequence?

```swift
final class Item {
    var name: String
    init(_ name: String) { self.name = name }
}
let source = [Item("A"), Item("B"), Item("C")]
let selected = source[1..<2]
var owned = Array(selected)
precondition(selected.startIndex == 1 && owned.startIndex == 0)
owned[0].name = "B updated"
precondition(source[1].name == "B updated")
owned[0] = Item("replacement")
precondition(source[1].name == "B updated" && owned[0].name == "replacement")
```

**Outcome:** Object mutation is shared; replacing the owned array’s slot is independent.

**Why:** A sequence value contains element values, which here are references. Changing which objects a sequence references is different from changing those objects. Retention bounds and deep-copy semantics are separate decisions.

## Interview questions

### Why can an element write or append require O(n) work even though array passing can be cheap?

**Answer:** Sharing can make passing/assignment cheap, but a write may have to preserve another array’s old elements and append may require capacity growth or representation conversion. Those paths can copy O(n) elements before the local operation finishes.

**Example:** Standalone runnable example.

```swift
let saved = Array(0..<8)
var edited = saved
edited[0] = -1
edited.append(8)
precondition(saved == Array(0..<8))
precondition(edited == [-1, 1, 2, 3, 4, 5, 6, 7, 8])
```

**Walkthrough:** The old value must remain intact through both mutations. Its contract explains why an implementation may need separation; a buffer with inadequate capacity also needs room to grow. [Array implementation and complexity](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift)

**Boundary:** The assertions do not identify which physical copy happened or how many allocations occurred. Unique-capacity append can have a cheaper path; element copying and destruction costs are additional.

### How can a tiny ArraySlice retain invisible elements, and what does Array(slice) still retain?

**Answer:** The slice can own its entire backing allocation even when it exposes only a subrange. Materializing an Array keeps the selected element values; visible class elements and their reachable object graphs remain retained.

**Example:** Prepend RetentionFixture setup.

```swift
let sliced = RetentionFixture(keepSlice: true)
let owned = RetentionFixture(keepSlice: false)
precondition(sliced.alive() == [true, true, true])
precondition(owned.alive() == [false, true, false])
precondition(sliced.slice?.startIndex == 1 && owned.owned?.startIndex == 0)
sliced.slice = nil
owned.owned = nil
precondition(sliced.alive() == [false, false, false])
precondition(owned.alive() == [false, false, false])
```

**Walkthrough:** Both stored collections expose item 1, but their observed retaining paths differ. Clearing them releases the remaining items in this bounded fixture. No destruction ordering is asserted.

**Boundary:** These states are checked in ordinary and optimized builds. They do not measure resident bytes, autorelease behavior, or other application owners. Rebase indices after materializing a copy.

### Can unique outer storage contain a shared Array buffer, and which mutation paths must preserve independence?

**Answer:** Yes. A fresh outer object can store an Array value also retained in a snapshot. Outer uniqueness says nothing about that Array’s internal buffer. The wrapper must preserve values through every setter, append/remove operation, and mutable projection.

**Example:** Standalone runnable example.

```swift
final class Storage {
    var values: [Int]
    init(_ values: [Int]) { self.values = values }
}
func nestedStorageProbe() {
    var outer = Storage([1, 2])
    let snapshot = outer.values
    precondition(isKnownUniquelyReferenced(&outer))
    outer.values.append(3)
    precondition(snapshot == [1, 2] && outer.values == [1, 2, 3])
}
nestedStorageProbe()
```

**Walkthrough:** The only strong reference to Storage is outer, while snapshot retains a separate Array value. Array’s own mutation preserves that snapshot. CowList’s private ensureUnique is responsible for the additional outer layer.

**Boundary:** Do not use isKnownUniquelyReferenced as an Array-buffer inspection API. Prevent mutable Storage from escaping and synchronize any genuinely shared owner separately.

### Extension: Why is UniqueArray a different contract rather than a drop-in snapshot optimization?

**Answer:** UniqueArray permits exclusive ownership and noncopyable elements instead of Array’s freely copyable snapshot contract. Transfer and element removal can be useful, but callers must accept those ownership costs.

**Example:** Standalone runnable example.

```swift
struct Token: ~Copyable { let id: Int }
func uniqueArrayProbe() {
    var values = UniqueArray<Token>()
    values.append(Token(id: 1))
    values.append(Token(id: 2))
    var ids: [Int] = []
    for token in values { ids.append(token.id) }
    let last = values.removeLast()
    precondition(ids == [1, 2] && last.id == 2 && values.count == 1)
}
uniqueArrayProbe()
```

**Walkthrough:** Iteration borrows elements without consuming the owner; removeLast transfers one Token out. No second independent UniqueArray snapshot is silently created. [UniqueArray design](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0527-rigidarray-uniquearray.md)

**Boundary:** The selected API requires Apple OS 27. Exclusive storage can still grow and allocate. Choose ownership first, then measure representative costs; [[C8 — Copyable, noncopyable types, and `~Copyable`|C8]] covers adoption, payload aliases, and container support.

## Focused exercise

**Task:** Mutate a nonzero-index slice while preserving the source and an earlier owned copy.

**Acceptance checks:**

- Slice indices retain the base range rather than starting at zero.
- A slice element write preserves source and owned-copy values.
- The owned copy has rebased indices; an empty end slice is handled without reading an element.

**Solution:** Standalone value/index exercise:

```swift
let source = [10, 20, 30, 40]
var slice = source[1...2]
precondition(slice.startIndex == 1 && slice.endIndex == 3)
let owned = Array(slice)
slice[slice.startIndex] = 99
precondition(source == [10, 20, 30, 40])
precondition(Array(slice) == [99, 30])
precondition(owned == [20, 30] && owned.startIndex == 0)
let empty = source[source.endIndex..<source.endIndex]
precondition(empty.isEmpty && empty.startIndex == source.endIndex)
```

**Reasoning:** The slice preserves the original index range and independent value semantics. Array(slice) changes the retained sequence and index origin; old slice indices must not be reused against the new array. The separate retention fixture answers the lifetime question, not these integer assertions.

## In practice

**Choose it when:** Use temporary slices for traversal and owned selected arrays when long-lived retention warrants the copy. Require a real invariant before adding a custom CoW layer.

**Investigate by:** Keep an old value for semantic tests, inspect weak retaining paths for lifetime, and use allocation/memory tools for physical costs.

**Strong evidence:** Audit all mutation and storage-exposure paths, synchronization, capacity, and reference payloads; compare Array and UniqueArray under their actual ownership contracts.

## References

### Related guides

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[C1 — ARC fundamentals]]
- [[C3 — Exclusivity enforcement and `inout`]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C7 — Ownership modifiers - `borrowing`, `consuming`, and `consume`]]
- [[C8 — Copyable, noncopyable types, and `~Copyable`]]
- [[C9 — Existential, generic, and allocation performance model]]
- [[C10 — Compiler Optimization Behavior and Debug vs Release Differences]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[D12 — Synchronization Library, Mutexes, and Choosing Between Actors and Locks]]
- [[E2 — Foundation and Objective-C Bridging Behavior]]
- [[F3 — Performance investigation habits]]

### Official sources

- [ArraySlice](https://developer.apple.com/documentation/swift/arrayslice)
- [Swift Array documentation/source](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Array.swift)
- [Swift ArraySlice documentation/source](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/ArraySlice.swift)
- [isKnownUniquelyReferenced documentation/source](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/ManagedBuffer.swift)
- [SE-0377: Borrowing and consuming parameter modifiers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)
- [SE-0527: Unique array proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0527-rigidarray-uniquearray.md)
