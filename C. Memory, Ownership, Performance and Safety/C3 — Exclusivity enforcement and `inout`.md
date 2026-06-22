---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - inout
  - exclusivity
---
## 0. Rubric snapshot

**Rubric expectation**

Understand exclusive access rules, dynamic exclusivity traps, and the borrow-like nature of `inout`. The rubric specifically expects you to explain why passing the same variable to two `inout` parameters is invalid and how to redesign such APIs.

**Caveats**

`inout` is not a stable alias, not a C pointer, and not a C++ reference. Some overlapping-access problems are caught statically; others can compile and trap dynamically at runtime. Swift 5 and later enforce exclusive access dynamically in both Debug and Release builds by default. ([Swift.org, "Swift 5 Exclusivity Enforcement"](https://swift.org/blog/swift-5-exclusivity/))

**You should be able to answer**

- Why is `inout` not the same as C++ references or pointers?
- What kind of overlapping accesses does Swift try to forbid?

**You should be able to do**

- Explain why passing the same variable to two `inout` parameters is invalid.
- Redesign an API to avoid overlapping access.
- Predict the compiler error for the code probe.

---

## 1. Core mental model

Swift’s exclusivity rule is:

```text
For one memory location, Swift allows either many reads or one mutation, but not both at the same time.
```

This is stricter than basic memory safety. Some code could be “memory-safe” in the narrow sense and still violate Swift’s exclusivity model. Swift uses exclusivity to make mutation predictable, preserve value semantics, and enable compiler optimizations. The Swift compiler documentation describes exclusivity checks as important for memory safety and optimization. ([Swift.org, "Exclusivity Violation"](https://docs.swift.org/compiler/documentation/diagnostics/exclusivity-violation/))

`inout` means: “temporarily give this function exclusive mutable access to the caller’s storage.” The function may read and write through the parameter, and when the function returns, the caller regains normal access. The Swift book describes `inout` with a copy-in/copy-out model: the argument’s value is copied into the function, modified there, and written back to the original when the function returns. ([Swift.org, "Declarations"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/))

However, you should not treat `inout` as literally “copying” or literally “passing an address.” The copy-in/copy-out model defines the semantics. The compiler is free to optimize the implementation, commonly by passing direct access to storage, but your code cannot depend on address identity or aliasing behavior.

The key idea:

```text
inout = temporary exclusive mutable access, not a pointer alias.
```

This is why `swapValues(&x, &x)` is invalid. Both parameters would require exclusive mutable access to the same storage for the same call duration. Swift refuses because there is no safe, deterministic way to let two independent mutable parameters alias the same variable.

---

## 2. Essential mechanics

### Accesses have a kind and a duration

Swift distinguishes reads from mutations. A simple read is usually instantaneous. A normal assignment is a mutation. An `inout` argument creates a longer access that lasts across the function call.

```swift
var x = 1

let y = x
// Instantaneous read of x.

x += 1
// Mutation of x.

func increment(_ value: inout Int) {
    value += 1
}

increment(&x)
// Exclusive mutable access to x for the duration of increment(_:).
```

The Swift book states that the write access for an `inout` parameter starts after non-`inout` parameters are evaluated and lasts for the whole function call. ([Swift.org, "Memory Safety"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety/))

That explains why this is valid:

```swift
func add(_ lhs: inout Int, _ rhs: Int) {
    lhs += rhs
}

var x = 1
add(&x, x)

print(x) // 2
```

The second argument `x` is evaluated before the long-term `inout` access begins. The function effectively receives:

```swift
add(&x, 1)
```

But this is invalid:

```swift
func addBoth(_ lhs: inout Int, _ rhs: inout Int) {
    lhs += rhs
}

var x = 1
addBoth(&x, &x)
```

Both parameters would require long-term mutable access to the same storage.

---

### `inout` access excludes other access to the same storage

Inside a function, do not access the original variable through another name while it is passed as `inout`. The Swift book explicitly warns that accessing the original value while it is passed as an in-out argument violates memory exclusivity. ([Swift.org, "Declarations"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/))

```swift
var stepSize = 1

func increment(_ number: inout Int) {
    number += stepSize
}

increment(&stepSize)
```

This can compile, but it traps at runtime:

```text
Simultaneous accesses to ..., but modification requires exclusive access.
Fatal access conflict detected.
```

Why?

```text
increment(&stepSize)
    starts exclusive write access to stepSize

inside increment:
    number += stepSize
              ^ reads stepSize through another name

same storage:
    number    -> stepSize
    stepSize  -> stepSize

write + read overlap => exclusivity violation
```

Fix it by separating the read from the write access:

```swift
var stepSize = 1

func increment(_ number: inout Int, by amount: Int) {
    number += amount
}

let amount = stepSize
increment(&stepSize, by: amount)

print(stepSize) // 2
```

---

### Mutating methods also create exclusive access to `self`

A `mutating` method on a value type gets exclusive mutable access to `self` during the call.

```swift
struct Counter {
    var value = 0

    mutating func increment() {
        value += 1
    }
}

var counter = Counter()
counter.increment()
```

Conceptually:

```text
counter.increment()
≈ Counter.increment(&counter)
```

This matters when a mutating method tries to read or mutate the same storage through another path. It is not only explicit `inout` parameters that participate in exclusivity. `self` in a mutating method does too. Swift.org’s exclusivity post describes the same principle: a variable cannot be accessed through a different name while it is being modified as an `inout` argument or as `self` in a mutating method. ([Swift.org, "Swift 5 Exclusivity Enforcement"](https://swift.org/blog/swift-5-exclusivity/))

---

## 3. Common traps and misconceptions

### Trap 1: Treating `&` as “address-of”

Bad mental model:

```text
&x means “give the function a pointer to x.”
```

Better mental model:

```text
&x means “I allow this call to temporarily mutate x.”
```

Bad:

```swift
func storePointer(_ value: inout Int) {
    // Wrong mental model: thinking `value` is a stable address-like alias.
}
```

Better:

```swift
func increment(_ value: inout Int) {
    value += 1
}
```

Use `inout` for scoped mutation, not for storing identity, keeping aliases, or simulating pointer-based APIs.

---

### Trap 2: Designing APIs with multiple `inout` parameters that may alias

Bad:

```swift
func update(_ lhs: inout Int, _ rhs: inout Int) {
    lhs += 1
    rhs += 1
}

var x = 0
update(&x, &x) // Invalid
```

Better:

```swift
func updated(_ lhs: Int, _ rhs: Int) -> (Int, Int) {
    (lhs + 1, rhs + 1)
}

var x = 0
let result = updated(x, x)
```

Or model the mutation as one aggregate value:

```swift
struct Pair {
    var lhs: Int
    var rhs: Int

    mutating func incrementBoth() {
        lhs += 1
        rhs += 1
    }
}
```

The aggregate model gives Swift one exclusive access to one value, instead of two potentially overlapping accesses.

---

### Trap 3: Mutating collection elements through overlapping access

This shape is a classic source of exclusivity problems:

```swift
var numbers = [1, 2, 3]

func modify(_ value: inout Int, using collection: [Int]) {
    value += collection.count
}

modify(&numbers[0], using: numbers)
```

The problem is that `numbers[0]` requires mutable access into `numbers`, while passing `numbers` also reads the collection.

Better:

```swift
var numbers = [1, 2, 3]
let count = numbers.count

numbers[0] += count
```

For swapping collection elements, do not write this:

```swift
swap(&numbers[0], &numbers[1])
```

Prefer the collection API designed for this:

```swift
numbers.swapAt(0, 1)
```

This avoids exposing two independently aliased `inout` accesses into the same collection.

---

## 4. Direct answers to rubric questions

### Q1. Why is `inout` not the same as C++ references or pointers?

`inout` is a scoped mutation mechanism with exclusive-access rules. It does not give you a stable alias or address that can be stored, escaped, or reasoned about like a pointer/reference.

Swift defines `inout` semantically as copy-in/copy-out: a value is made available to the function, the function mutates it, and the final value is written back to the original storage. The compiler may optimize this, but source code must not depend on the optimization. ([Swift.org, "Declarations"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/))

Interview version:

> `inout` is Swift’s way to express temporary caller-visible mutation. It is not an address-of operation. The callee gets exclusive mutable access for the duration of the call, and the caller cannot access the same storage through another path during that access. The implementation may be optimized, but semantically it behaves like copy-in/copy-out with exclusive writeback.

---

### Q2. What kind of overlapping accesses does Swift try to forbid?

Swift forbids overlapping accesses where at least one access is a mutation and both accesses refer to the same storage.

Examples:

```swift
swapValues(&x, &x)
// two simultaneous inout write accesses to x
```

```swift
var stepSize = 1

func increment(_ number: inout Int) {
    number += stepSize
}

increment(&stepSize)
// inout write access to stepSize overlaps with read of stepSize
```

```swift
var array = [1, 2, 3]

func update(_ value: inout Int, using array: [Int]) {
    value += array.count
}

update(&array[0], using: array)
// mutation of part of array overlaps with read of whole array
```

Interview version:

> Swift tries to prevent simultaneous access to the same memory when one of the accesses is a write. Multiple reads are fine. A write must be exclusive. This matters for `inout`, `mutating` methods, collection element mutation, computed properties, and captured storage. Some conflicts are statically rejected; others are dynamically checked and trap.

---

## 5. Code probe

Given:

```swift
func swapValues(_ a: inout Int, _ b: inout Int) {
    let temp = a
    a = b
    b = temp
}

var x = 1
swapValues(&x, &x)
```

### What happens?

Swift 6.2.1 produces these compiler errors:

```text
/tmp/c3.swift:7:16: error: inout arguments are not allowed to alias each other
5 | }
6 | var x = 1
7 | swapValues(&x, &x)
  |            |   `- error: inout arguments are not allowed to alias each other
  |            `- note: previous aliasing argument
8 |

/tmp/c3.swift:7:12: error: overlapping accesses to 'x', but modification requires exclusive access; consider copying to a local variable
5 | }
6 | var x = 1
7 | swapValues(&x, &x)
  |            |   `- note: conflicting access is here
  |            `- error: overlapping accesses to 'x', but modification requires exclusive access; consider copying to a local variable
8 |
```

### Why?

The call requires two simultaneous exclusive mutable accesses to the same variable.

```text
swapValues(&x, &x)

Parameter a:
    needs exclusive mutable access to x
    duration: whole function call

Parameter b:
    needs exclusive mutable access to x
    duration: whole function call

Same memory location:
    x

Conflict:
    write access through a overlaps write access through b
```

If Swift allowed this, the semantics would be ambiguous and optimization-hostile. Should `a = b` observe the old value, the partially updated value, or the same aliased storage? In a pointer-based language, this kind of aliasing often becomes a source of subtle bugs. Swift chooses to make the invalid access explicit.

### Fix or redesign

For two distinct variables, the original API is fine:

```swift
func swapValues(_ a: inout Int, _ b: inout Int) {
    let temp = a
    a = b
    b = temp
}

var x = 1
var y = 2

swapValues(&x, &y)

print(x, y) // 2 1
```

For one variable, swapping it with itself should be a no-op, so do not call a two-`inout` API:

```swift
var x = 1

// No-op. Nothing to swap.
print(x) // 1
```

For indexed collection mutation, use a collection-level API:

```swift
var values = [1, 2, 3]

values.swapAt(0, 2)

print(values) // [3, 2, 1]
```

For domain logic, prefer returning a new value or mutating a single aggregate:

```swift
struct Bounds {
    var lower: Int
    var upper: Int

    mutating func normalize() {
        if lower > upper {
            swap(&lower, &upper)
        }
    }
}

var bounds = Bounds(lower: 10, upper: 2)
bounds.normalize()

print(bounds) // Bounds(lower: 2, upper: 10)
```

### Why this fix is correct

`Bounds.normalize()` creates one exclusive mutable access to `bounds`. Inside that access, Swift can reason about mutation of distinct stored properties. The API no longer asks callers to pass two independent mutable references that might accidentally point to the same storage.

```text
Bad API:
    normalize(&a, &b)
    caller must guarantee a and b do not alias

Better API:
    bounds.normalize()
    one aggregate owns the invariant
```

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Keep two `inout` parameters|Low-level utilities where arguments are obviously distinct, such as swapping two local variables|Caller must avoid aliasing|
|Return a new value or tuple|Transformations, normalization, parsing, value-oriented APIs|May be less ergonomic for large in-place algorithms|
|Mutate a single aggregate|Domain invariants belong together, such as ranges, bounds, geometry, request state|Requires better modeling upfront|
|Use collection-specific APIs like `swapAt`|Mutating two elements of the same collection|Less general than arbitrary `inout`, but safer|
|Use unsafe pointers|Interop or performance-critical low-level code|Requires manual lifetime, aliasing, and safety reasoning|

---

## 6. Exercise

### Problem

Explain why passing the same variable to two `inout` parameters is invalid and redesign the API.

### Bad / naive version

```swift
func clamp(_ value: inout Int, min lowerBound: inout Int, max upperBound: inout Int) {
    if lowerBound > upperBound {
        swap(&lowerBound, &upperBound)
    }

    if value < lowerBound {
        value = lowerBound
    } else if value > upperBound {
        value = upperBound
    }
}

var x = 10

clamp(&x, min: &x, max: &x)
```

### What is wrong?

```text
The API accepts three independent mutable parameters.

The caller can pass the same variable more than once.

Each inout parameter requires exclusive mutable access for the duration of the call.

Passing &x three times creates overlapping mutable accesses to x.

Swift rejects this because the API shape allows aliasing that would make mutation order and writeback semantics ambiguous.
```

Even if the function body “looks harmless,” the signature itself is dangerous. It asks the caller to uphold a non-aliasing invariant that the type system cannot express through plain `inout`.

### Improved version

Use immutable bounds and one mutable value:

```swift
func clamped(_ value: Int, min lowerBound: Int, max upperBound: Int) -> Int {
    let lower = Swift.min(lowerBound, upperBound)
    let upper = Swift.max(lowerBound, upperBound)

    return Swift.min(Swift.max(value, lower), upper)
}

var x = 10
x = clamped(x, min: x, max: x)

print(x) // 10
```

Or model the bounds explicitly:

```swift
struct ClosedIntegerBounds {
    private(set) var lower: Int
    private(set) var upper: Int

    init(_ a: Int, _ b: Int) {
        lower = Swift.min(a, b)
        upper = Swift.max(a, b)
    }

    func clamping(_ value: Int) -> Int {
        Swift.min(Swift.max(value, lower), upper)
    }
}

let bounds = ClosedIntegerBounds(0, 100)

var score = 150
score = bounds.clamping(score)

print(score) // 100
```

For a truly in-place API:

```swift
extension Int {
    mutating func clamp(to bounds: ClosedIntegerBounds) {
        self = bounds.clamping(self)
    }
}

var score = 150
score.clamp(to: ClosedIntegerBounds(0, 100))

print(score) // 100
```

### Why this is better

The improved design separates:

```text
value being mutated:
    score

configuration being read:
    bounds
```

The bounds are immutable input. The value is the only mutable target. That removes overlapping mutable access and makes the API easier to reason about.

This is the production lesson: if an API has multiple `inout` parameters, ask whether those parameters are truly independent mutable locations. If not, the domain model is probably wrong.

---

## 7. Production guidance

Use `inout` in production when:

```text
- You need scoped, caller-visible mutation.
- The mutation is local, obvious, and does not escape.
- The API has one clear mutation target.
- You are implementing low-level algorithms where returning a new value would be awkward or inefficient.
- You are writing operators or collection-style mutation APIs.
```

Be careful when:

```text
- A function has multiple inout parameters.
- The caller may pass properties, subscripts, or collection elements.
- The function also reads from global state or captured state.
- The mutation target is part of a larger aggregate that is also being read.
- You are trying to optimize before measuring.
```

Avoid when:

```text
- A return value would express the transformation more clearly.
- The API requires caller knowledge that two arguments must not alias.
- You are trying to store or escape access to the parameter.
- You are using inout as a substitute for domain modeling.
- You are simulating C pointer-style APIs in ordinary Swift code.
```

Debugging checklist:

```text
1. Which exact storage location is being mutated?
2. Does the same storage appear through another name?
3. Is an inout argument active during the conflicting read/write?
4. Is self in a mutating method part of the conflict?
5. Is a collection element being mutated while the collection is also read?
6. Is the conflict compile-time or runtime?
7. Can the API be redesigned around a single mutation target?
8. Can immutable inputs be copied before beginning mutation?
9. Would returning a new value be clearer?
10. Is this actually a domain-modeling problem?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `inout` lets a function change a variable outside the function.

This is true but incomplete.

### Senior answer

> `inout` gives a function temporary exclusive mutable access to caller-owned storage. Swift prevents overlapping access where one access is a mutation. Passing the same variable to two `inout` parameters is invalid because both parameters would require exclusive access to the same memory for the same call duration.

### Staff-level answer

> `inout` is an API design tool for scoped mutation, not a pointer mechanism. The important judgment is whether the API has one clear mutation target or whether it forces callers to reason about aliasing. Multiple `inout` parameters, collection element mutation, and mutating methods that also read through another path are red flags. In production, I would prefer immutable inputs plus a return value, a single aggregate with a mutating method, or a collection-specific operation like `swapAt` rather than exposing alias-prone mutation.

Staff-level questions to ask:

```text
Does this API have one mutation target or multiple alias-prone targets?
Can the invariant be modeled as one aggregate value?
Would returning a transformed value be clearer?
Is the inout parameter crossing abstraction boundaries where aliasing becomes hard to reason about?
Could a collection-specific or domain-specific API avoid exposing raw mutation?
```

---

## 9. Interview-ready summary

`inout` in Swift means temporary exclusive mutable access to caller-owned storage. It is not a C pointer or C++ reference, and `&` is not a general address-of operator. Swift’s exclusivity rule allows either multiple reads or one mutation to a memory location at a time. An `inout` access lasts for the duration of the function call, so passing the same variable to two `inout` parameters would create two simultaneous mutable accesses to the same storage. Swift rejects that statically when it can, and dynamically traps for cases it can only detect at runtime. Good Swift API design keeps mutation targets obvious, avoids alias-prone multiple-`inout` signatures, and often prefers returning a value or mutating a single aggregate.

---

## 10. Flashcards

Q: What does `inout` mean in Swift?

A: Temporary exclusive mutable access to caller-owned storage for the duration of a function call.

Q: Is `&x` Swift’s address-of operator?

A: No. At a call site, `&x` marks that `x` is passed as an `inout` argument and may be mutated.

Q: Why is `swapValues(&x, &x)` invalid?

A: Both parameters need exclusive mutable access to the same storage at the same time.

Q: What does Swift’s exclusivity rule allow?

A: Either multiple reads or one mutation to the same memory location, but not overlapping read/write or write/write access.

Q: Are exclusivity violations always compile-time errors?

A: No. Some are statically rejected; others compile and trap dynamically at runtime.

Q: Why is `inout` not a C++ reference?

A: It does not provide a stable alias. Its semantics are scoped mutation with copy-in/copy-out behavior and exclusivity enforcement.

Q: What is a common API smell involving `inout`?

A: Multiple `inout` parameters where callers can accidentally pass overlapping storage.

Q: How can you redesign an alias-prone `inout` API?

A: Use immutable inputs plus a return value, mutate a single aggregate, or provide a domain-specific method.

---

## 11. Related sections

- [[A7 — Functions, parameter labels, defaults, and API surface]]
- [[A15 — Type choices: struct, class, enum, protocol, and actor]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C6 — Unsafe pointers, buffers, and raw memory]]
- [[C10 — Compiler optimization behavior and debug vs release differences]]
- [[D12 — Synchronization library, mutexes, and choosing between actors and locks]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — C3 — Exclusivity enforcement and `inout`.
- Swift.org. "Memory Safety." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/memorysafety/
- Swift.org. "Declarations." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/
- Swift.org. "Swift 5 Exclusivity Enforcement." Swift.org Blog. https://swift.org/blog/swift-5-exclusivity/
- GitHub. "Enforce Exclusive Access to Memory." Swift Evolution SE-0176. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0176-enforce-exclusive-access-to-memory.md
