---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - value-semantics
  - reference-semantics
  - copy-on-write
---
# A1 — Value semantics, reference semantics, identity, and copy-on-write

Tags: #swift #ios #interview-prep #core-semantics #value-semantics #reference-semantics #copy-on-write

## 0. Rubric snapshot

**Rubric expectation**

Understand the semantic difference between copying a value and sharing identity; know that many standard-library value types use copy-on-write. The rubric explicitly treats A1 as a core senior/staff topic.

**Caveats**

- `let` on a class freezes the reference binding, not the object’s internal mutable state.
- Copy-on-write is an implementation/performance strategy, not the definition of value semantics.
- `Array`, `Dictionary`, `Set`, and `String` usually behave like values while sharing storage internally until mutation. Apple’s `Array` documentation describes this directly: multiple copies can share storage until one copy is modified. ([Apple Developer, "Array"](https://developer.apple.com/documentation/swift/array))

**You should be able to answer**

- What is the difference between `===` and `==`, and when is using `===` a design smell?
- Why can `Array` usually behave like a value even though copying it may be cheap until mutation?

**You should be able to do**

- Given a `struct` containing an `Array` and a `class` containing an `Array`, predict which mutations are visible across copies.
- Explain exactly where value semantics, reference semantics, and copy-on-write each enter the behavior.

---

## 1. Core mental model

Swift has two major semantic models for ordinary domain types:

```text
Value semantics: assignment creates an independent logical value.
Reference semantics: assignment copies a reference to the same identity.
```

A `struct` or `enum` instance is a value. When you assign it to another variable, pass it to a function, or return it, Swift gives you a separate logical value. This does **not** mean Swift eagerly performs a deep physical copy of every byte. It means mutations through one variable should not be visible through another independent variable unless you deliberately store shared reference state inside the value. Swift’s language guide describes structures and enumerations as value types, copied when assigned or passed. ([Swift.org, "Structures and Classes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/))

A `class` instance has identity. Assignment copies the reference, not the object. Two variables can refer to the same instance, and mutating through either name mutates the same object. Swift provides `===` and `!==` specifically to test whether two class references point to the exact same instance. ([Swift.org, "Structures and Classes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/))

Copy-on-write is how Swift standard-library value types often make value semantics efficient. An `Array` variable may share a buffer with another `Array` variable after assignment. As long as nobody mutates, sharing is fine. When one copy mutates, the array checks whether the buffer is uniquely referenced; if not, it copies the buffer and then mutates the new buffer. The observable semantics stay value-like. ([Apple Developer, "Array"](https://developer.apple.com/documentation/swift/array))

The key idea:

```text
Value semantics is about observable independence.
Copy-on-write is one way to implement that independence efficiently.
Reference semantics is about shared identity.
```

Swift guarantees the semantic behavior of value types. It does **not** guarantee that every assignment performs an eager deep copy, nor should production code depend on the exact storage-sharing strategy of standard-library types.

---

## 2. Essential mechanics

### Value assignment copies the logical value

```swift
struct Profile {
    var name: String
}

var first = Profile(name: "Aykut")
var second = first

second.name = "Taylor"

print(first.name)   // Aykut
print(second.name)  // Taylor
```

`first` and `second` are independent logical values. Mutating `second.name` does not mutate `first.name`.

This is the model you usually want for domain models, request payloads, UI state snapshots, reducer state, settings, and immutable data transfer.

---

### Class assignment shares identity

```swift
final class ProfileRef {
    var name: String

    init(name: String) {
        self.name = name
    }
}

let first = ProfileRef(name: "Aykut")
let second = first

second.name = "Taylor"

print(first.name)   // Taylor
print(second.name)  // Taylor
print(first === second) // true
```

`first` and `second` refer to the same object. `let first` prevents rebinding `first` to another object, but it does **not** make the object immutable.

```swift
let user = ProfileRef(name: "Aykut")
user.name = "Taylor"        // allowed
// user = ProfileRef(...)   // not allowed
```

That distinction is a common interview trap.

---

### Copy-on-write preserves value behavior while avoiding unnecessary copies

```swift
var a = [1, 2, 3]
var b = a        // storage may be shared

b.append(4)      // b gets unique storage before mutation

print(a)         // [1, 2, 3]
print(b)         // [1, 2, 3, 4]
```

From your code’s point of view, `Array` behaves like a value. Internally, it may share storage until mutation. Apple documents this copy-on-write strategy for `Array`. ([Apple Developer, "Array"](https://developer.apple.com/documentation/swift/array))

This distinction matters for performance conversations. Saying “arrays copy on assignment” is semantically true but mechanically incomplete. A senior answer says: assignment creates an independent logical value; the actual buffer copy is commonly deferred until mutation.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking `let` makes a class instance immutable

Bad:

```swift
final class Session {
    var token: String

    init(token: String) {
        self.token = token
    }
}

let session = Session(token: "old")
session.token = "new" // allowed
```

Better:

```swift
struct Session {
    let token: String
}

let session = Session(token: "old")
// session.token = "new" // compiler error
```

Or, if identity is required:

```swift
final class Session {
    let token: String

    init(token: String) {
        self.token = token
    }
}
```

`let` on a class reference means “this variable always points to the same object.” It does not mean “this object is frozen.”

---

### Trap 2: Treating copy-on-write as the same thing as value semantics

Copy-on-write is an implementation technique. Value semantics is an observable contract.

You can build a `struct` that accidentally violates value-like expectations by storing a mutable class inside it:

```swift
struct BadValue {
    final class Storage {
        var items: [Int] = []
    }

    var storage = Storage()
}

var a = BadValue()
var b = a

b.storage.items.append(1)

print(a.storage.items) // [1] — surprising
```

This is a `struct`, but it does not provide clean value semantics because its stored reference is shared.

Better:

```swift
struct GoodValue {
    var items: [Int] = []
}

var a = GoodValue()
var b = a

b.items.append(1)

print(a.items) // []
print(b.items) // [1]
```

A `struct` is not magically “deep value semantic” if you hide shared mutable reference storage inside it.

---

### Trap 3: Using `===` as ordinary domain equality

Bad:

```swift
if lhs === rhs {
    print("same user")
}
```

This answers “are these the exact same object?” not “do these represent the same user?”

Better:

```swift
struct User: Equatable {
    let id: UserID
    var name: String
}

if lhs.id == rhs.id {
    print("same user identity in the domain")
}
```

Using `===` is fine for identity-sensitive infrastructure: object graphs, cache entries, view-controller lifecycle debugging, reference-cycle diagnosis, or checking whether two references are intentionally the same instance. It is a smell when business logic depends on accidental object identity instead of explicit domain identity.

---

## 4. Direct answers to rubric questions

### Q1. What is the difference between `===` and `==`, and when is using `===` a design smell?

`==` means equality according to the type’s `Equatable` semantics. `===` means two class references point to the exact same object instance.

`==` is domain-defined. For a `User`, it might mean same `id`. For a `Point`, it might mean same coordinates. For a `String`, it means same textual value. Not every type gets meaningful equality for free; types define or synthesize it according to their structure and semantics.

`===` bypasses domain equality and asks about object identity. Swift only supports it for class instances. The Swift language guide documents identity operators for checking whether two constants or variables refer to exactly the same class instance. ([Swift.org, "Structures and Classes"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/))

Using `===` is a design smell when the domain question is actually “are these equivalent?” or “do these represent the same entity?” It ties correctness to allocation and object lifetime rather than explicit model semantics.

Interview version:

> `==` is semantic equality defined by the type, usually through `Equatable`. `===` is reference identity for class instances: are these two references pointing to the exact same object? I’d use `===` for identity-sensitive infrastructure or debugging, but it’s a smell in domain logic because it often means we forgot to model identity explicitly, usually with an ID or value-based equality.

---

### Q2. Why can `Array` usually behave like a value even though copying it may be cheap until mutation?

Because `Array` uses copy-on-write. Assigning an array to another variable creates an independent logical value, but the underlying storage may be shared until one variable mutates. When mutation happens, the mutating array ensures it has unique storage before applying the change. Apple documents this exact behavior for `Array`: multiple copies share storage until one copy is modified. ([Apple Developer, "Array"](https://developer.apple.com/documentation/swift/array))

So this is safe and value-like:

```swift
var a = [1]
var b = a

b.append(2)

print(a) // [1]
print(b) // [1, 2]
```

The cheap copy is a performance optimization. The semantic guarantee is that mutating `b` does not unexpectedly mutate `a`.

Interview version:

> `Array` has value semantics, but it is implemented with copy-on-write. Assignment gives me an independent logical array, but the buffer can be shared internally. On mutation, Swift checks uniqueness and copies storage if needed. That gives value-like behavior without paying for eager deep copies on every assignment.

---

## 5. Code probe

Given:

```swift
struct ValueBox {
    var items = [1]
}

final class RefBox {
    var items = [1]
}

var a = ValueBox()
var b = a

b.items.append(2)

let c = RefBox()
let d = c

d.items.append(2)

print(a.items, b.items, c.items, d.items)
```

### What happens?

```text
[1] [1, 2] [1, 2] [1, 2]
```

### Why?

Step by step:

```text
1. var a = ValueBox()
   a.items == [1]

2. var b = a
   ValueBox is a struct.
   b is a separate logical ValueBox value.

3. b.items.append(2)
   Array uses CoW.
   b.items mutates independently.
   a.items remains [1].
   b.items becomes [1, 2].

4. let c = RefBox()
   c points to one RefBox instance.

5. let d = c
   d points to the same RefBox instance.

6. d.items.append(2)
   The shared RefBox instance's items are mutated.
   c.items and d.items observe the same array property.

7. print(...)
   a.items == [1]
   b.items == [1, 2]
   c.items == [1, 2]
   d.items == [1, 2]
```

Ownership / identity diagram:

```text
ValueBox path:

a ── ValueBox(items: [1])
b ── ValueBox(items: [1])  -- logically independent
                  │
                  └─ append(2) on b only

After:
a.items = [1]
b.items = [1, 2]


RefBox path:

c ─┐
   ├── RefBox instance { items: [1] }
d ─┘

append(2) through d mutates the shared instance.

After:
c.items = [1, 2]
d.items = [1, 2]
```

The important separation:

```text
Value semantics:
- ValueBox assignment creates independent logical values.
- Mutation through b is not visible through a.

Reference semantics:
- RefBox assignment copies the reference.
- c and d share object identity.
- Mutation through d is visible through c.

Copy-on-write:
- Applies inside Array.
- It makes b.items mutation efficient while preserving Array's value behavior.
- It does not make RefBox itself a value type.
```

### Fix or redesign

The original code is not broken; it is a probe. But if shared mutation is not intended, prefer value modeling:

```swift
struct Box {
    var items = [1]
}

var c = Box()
var d = c

d.items.append(2)

print(c.items) // [1]
print(d.items) // [1, 2]
```

If shared identity is intended, make it explicit in the name and API:

```swift
final class SharedBox {
    private(set) var items = [1]

    func append(_ item: Int) {
        items.append(item)
    }
}

let c = SharedBox()
let d = c

d.append(2)

print(c.items) // [1, 2]
print(d.items) // [1, 2]
```

### Why this fix is correct

The fix is not about syntax. It is about choosing the semantic contract:

```text
Use struct when copies should be independent.
Use class when shared identity is part of the model.
Use private mutation APIs when shared mutable state must be controlled.
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`struct` with value-typed stored properties|Domain data, UI state, request/response models, snapshots|Large values may need careful performance thinking, though CoW helps for stdlib collections|
|`final class` with explicit identity|Shared lifecycle, delegates, object graphs, reference caches, UIKit/AppKit objects|Shared mutation and aliasing bugs become possible|
|`struct` with private CoW storage|Custom large value type that needs efficient copying|More complex implementation; must preserve value semantics carefully|
|`actor` wrapping mutable state|Shared mutable state accessed across concurrency boundaries|Async access, actor reentrancy, and isolation design must be handled deliberately|

---

## 6. Exercise

### Problem

Given a `struct` containing an `Array` and a `class` containing an `Array`, predict which mutations are visible across copies.

### Bad / naive version

```swift
struct Cart {
    var items: [String]
}

final class CartController {
    var cart: Cart

    init(cart: Cart) {
        self.cart = cart
    }
}

let original = CartController(cart: Cart(items: ["Book"]))
let alias = original

alias.cart.items.append("Pen")

print(original.cart.items)
print(alias.cart.items)
```

### What is wrong?

```text
The Cart itself is a value, but CartController is a class.
original and alias point to the same CartController instance.
Mutating alias.cart.items mutates the cart property of the shared controller.
So both print ["Book", "Pen"].
```

A common mistake is to look only at the inner `struct` and conclude that the whole object graph has value semantics. It does not. The outer reference type controls aliasing.

### Improved version

If you want independent carts:

```swift
struct Cart {
    var items: [String]
}

var original = Cart(items: ["Book"])
var copy = original

copy.items.append("Pen")

print(original.items) // ["Book"]
print(copy.items)     // ["Book", "Pen"]
```

If you want shared cart state, make sharing explicit:

```swift
final class SharedCart {
    private(set) var items: [String]

    init(items: [String]) {
        self.items = items
    }

    func addItem(_ item: String) {
        items.append(item)
    }
}

let original = SharedCart(items: ["Book"])
let alias = original

alias.addItem("Pen")

print(original.items) // ["Book", "Pen"]
print(alias.items)    // ["Book", "Pen"]
```

### Why this is better

The improved versions make the semantic contract visible:

```text
Cart as struct:
- Copies are independent.
- Great for UI state, persistence snapshots, reducers, value transformations.

SharedCart as class:
- Copies of references share identity.
- Mutation is centralized through an explicit method.
- Better when identity and shared lifecycle are intentional.
```

At senior/staff level, the answer is not “struct good, class bad.” The answer is: choose the semantic model that matches the domain, then design the API so accidental aliasing is hard.

---

## 7. Production guidance

Use value semantics in production when:

```text
- Modeling domain data.
- Modeling UI state snapshots.
- Passing data across layers.
- Reducing hidden coupling.
- Making tests deterministic.
- Supporting easier reasoning under Swift Concurrency.
```

Use reference semantics in production when:

```text
- Identity is part of the model.
- You need shared mutable state behind a controlled API.
- You are modeling lifecycle-bound objects.
- You are interoperating with UIKit, AppKit, Objective-C, Core Data, delegates, or reference-based frameworks.
- You need polymorphism through class inheritance, though this should be rare in modern Swift domain code.
```

Be careful when:

```text
- A struct stores a mutable class.
- A class exposes public mutable properties.
- Equality is accidentally based on object identity.
- A value type claims Sendable but hides non-thread-safe reference storage.
- CoW performance is assumed instead of measured.
- A small ArraySlice or slice-like view accidentally retains large backing storage.
```

Avoid when:

```text
- Using classes by default for simple data models.
- Using === for business equality.
- Assuming let classReference means deep immutability.
- Hiding shared mutable storage inside a value type without preserving value semantics.
- Depending on exact CoW storage behavior as API contract.
```

Debugging checklist:

```text
1. Is this type a struct, enum, class, or actor?
2. Does assignment create independent logical state or shared identity?
3. Are any stored properties reference types?
4. Are mutations visible through another variable?
5. Is equality domain equality or object identity?
6. Is CoW involved, and is it preserving observable value behavior?
7. Could a reference hidden inside a value type break Sendable, Equatable, or Hashable expectations?
8. Is shared mutation intentional, controlled, and documented?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Structs are copied and classes are shared. Arrays are value types. Use `===` for classes and `==` for equality.

This is directionally right but too shallow.

### Senior answer

> Structs and enums have value semantics, so assignment creates independent logical values. Classes have reference semantics, so assignment copies a reference to the same identity. Standard-library collections like `Array` use copy-on-write, so assignment is often cheap until mutation while preserving value-like behavior. `==` is semantic equality; `===` is class identity.

This is a strong working answer.

### Staff-level answer

> I separate semantic guarantees from implementation strategy. Value semantics means independent observable state, not necessarily eager deep copies. CoW is an optimization used by many standard-library values to preserve that contract efficiently. Reference semantics means shared identity and aliasing, which is sometimes required but must be explicit. In production API design, I avoid hiding mutable reference storage inside value types unless I implement real CoW or document the sharing. I also avoid `===` in domain logic because domain identity should usually be explicit, stable, testable, and independent of allocation.

Staff-level questions to ask:

```text
Is identity actually part of the domain, or are we using a class by habit?
Does this value type contain reference storage, and does that break value expectations?
Should equality mean full structural equality, stable domain identity, or object identity?
Could this design create hidden shared mutation across async or actor boundaries?
Is CoW needed for performance, and can we prove the implementation preserves value semantics?
```

---

## 9. Interview-ready summary

Swift value semantics means assignment and passing create independent logical values; structs and enums usually give you this model. Reference semantics means variables share the same object identity; classes give you that model, and `===` checks whether two class references point to the exact same instance. `==` is domain equality, while `===` is object identity. `Array` behaves like a value even though it often shares storage internally because it uses copy-on-write: storage can be shared until mutation, then the mutating copy gets unique storage. The production judgment is to use value types for independent state and reference types only when identity, lifecycle, or controlled sharing is actually part of the design.

---

## 10. Flashcards

Q: What is the difference between value semantics and reference semantics?  
A: Value semantics means assignment creates independent logical values. Reference semantics means assignment copies a reference to the same object identity.

Q: What does `==` mean?  
A: `==` means semantic equality as defined by the type’s `Equatable` conformance.

Q: What does `===` mean?  
A: `===` means two class references point to the exact same object instance.

Q: Why is `===` often a design smell in domain logic?  
A: Because domain identity should usually be modeled explicitly with stable IDs or value equality, not accidental object allocation identity.

Q: Why can `Array` copying be cheap?  
A: `Array` uses copy-on-write, so multiple arrays can share storage until one is mutated.

Q: Is copy-on-write the same as value semantics?  
A: No. Value semantics is the observable contract; copy-on-write is an implementation strategy for preserving that contract efficiently.

Q: Does `let` make a class instance immutable?  
A: No. It freezes the reference binding, not the object’s internal mutable state.

Q: Can a `struct` accidentally have reference-like behavior?  
A: Yes, if it stores shared mutable reference-type storage without implementing proper value semantics.

---

## 11. Related sections

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A15 — Type choices: struct, class, enum, protocol, and actor]]
- [[B8 — Equatable, Hashable, and collection correctness]]
- [[C2 — Copy-on-write behavior of standard-library types]]

---

## 12. Sources

- "Swift Senior/Staff Rubric." A1 section and code probe.
- Swift.org. "Structures and Classes." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/
- Apple Developer. "Array." Apple Developer Documentation. https://developer.apple.com/documentation/swift/array
- Swift.org. "Declarations." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/
