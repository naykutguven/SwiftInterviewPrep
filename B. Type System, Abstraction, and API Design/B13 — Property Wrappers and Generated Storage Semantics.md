---
tags:
  - swift
  - ios
  - interview-prep
  - type-system
  - property-wrappers
---
## 0. Rubric snapshot

**Rubric expectation**

Know `wrappedValue`, `projectedValue`, initialization rules, access-control effects, and wrapper composition.

**Caveats**

Property wrappers affect storage, mutability, isolation, and API shape in non-obvious ways. The rubric specifically asks you to explain synthesized storage, thread-safety / actor-isolation risks, and implement a small validating or logging wrapper.

**You should be able to answer**

- What code does a property wrapper effectively synthesize around a property?
- How can a wrapper accidentally hide thread-safety or actor-isolation problems?

**You should be able to do**

- Implement a small wrapper that validates inputs or logs mutations, and explain its initialization behavior.
- Predict the output of the `@Clamped` code probe.

---

## 1. Core mental model

A property wrapper is not magic storage attached to a normal stored property. It is a source-level transformation: Swift stores an instance of the wrapper type, then exposes the declared property as computed access through the wrapper’s `wrappedValue`.

The declared property:

```swift
@Clamped var value = 150
```

is best understood as:

```swift
private var _value = Clamped(wrappedValue: 150)

var value: Int {
    get { _value.wrappedValue }
    set { _value.wrappedValue = newValue }
}
```

The wrapper owns the real storage. The wrapped property is the public-facing API. The optional projected property, accessed with `$value`, exposes extra wrapper-defined API through `projectedValue`.

The key idea:

```text
@Wrapper var x: T  ≈  private var _x: Wrapper<T> + computed var x: T via _x.wrappedValue
```

This is why wrappers are powerful: they let a library author reuse storage policies such as validation, persistence, dependency lookup, binding, lazy initialization, observation, or logging without hand-writing the same getter/setter shape everywhere. SE-0258 describes property wrappers as a way for a property declaration to state which wrapper implements it; the wrapper provides storage, `wrappedValue` implements access, and `init(wrappedValue:)` initializes wrapper storage from the property’s value. ([GitHub, "Property Wrappers"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md))

What Swift guarantees is the access transformation and initialization rules. What Swift does **not** guarantee is that the wrapper’s internal policy is semantically safe. A wrapper can hide locks, reference storage, global state, `UserDefaults`, unsafe pointers, non-`Sendable` services, or main-actor-only assumptions behind a harmless-looking property.

---

## 2. Essential mechanics

### 2.1 `wrappedValue` defines the property’s apparent value

Every property wrapper type must be marked with `@propertyWrapper` and must provide a non-static `wrappedValue` property. Swift compiler diagnostics document that `wrappedValue` is required, must be non-static, and must have compatible access with the wrapper type; wrapper initializers also cannot be failable. ([Swift.org, "Property Wrapper Implementation Requirements"](https://docs.swift.org/compiler/documentation/diagnostics/property-wrapper-requirements/))

```swift
@propertyWrapper
struct Uppercased {
    private var storage: String

    init(wrappedValue: String) {
        self.storage = wrappedValue.uppercased()
    }

    var wrappedValue: String {
        get { storage }
        set { storage = newValue.uppercased() }
    }
}

struct User {
    @Uppercased var username: String = "aykut"
}

print(User().username)
```

Output:

```text
AYKUT
```

`username` looks like a `String`, but the stored property is effectively `_username: Uppercased`.

---

### 2.2 `projectedValue` defines `$property`

A wrapper can expose a second API through `projectedValue`. This is accessed using `$property`.

```swift
@propertyWrapper
struct Validated<Value> {
    private var storage: Value
    private let isValid: (Value) -> Bool

    init(wrappedValue: Value, _ isValid: @escaping (Value) -> Bool) {
        self.isValid = isValid
        self.storage = wrappedValue
    }

    var wrappedValue: Value {
        get { storage }
        set {
            if isValid(newValue) {
                storage = newValue
            }
        }
    }

    var projectedValue: Bool {
        isValid(storage)
    }
}

struct Form {
    @Validated({ !$0.isEmpty }) var name = ""
}

let form = Form()
print(form.name)
print(form.$name)
```

Output:

```text

false
```

`form.name` is the wrapped value. `form.$name` is the projection. The Swift Book notes that projected values can expose any type, including a boolean, another object, or the wrapper itself. ([Swift.org, "Properties"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/))

---

### 2.3 Initialization has several shapes

The most common form uses `init(wrappedValue:)`:

```swift
@Clamped var value = 150
```

This becomes approximately:

```swift
private var _value = Clamped(wrappedValue: 150)
```

You can also pass wrapper-specific arguments:

```swift
@propertyWrapper
struct ClampedToRange {
    private var value: Int
    private let range: ClosedRange<Int>

    init(wrappedValue: Int, _ range: ClosedRange<Int>) {
        self.range = range
        self.value = range.clamp(wrappedValue)
    }

    var wrappedValue: Int {
        get { value }
        set { value = range.clamp(newValue) }
    }
}

private extension ClosedRange where Bound == Int {
    func clamp(_ value: Int) -> Int {
        min(max(value, lowerBound), upperBound)
    }
}

struct Score {
    @ClampedToRange(0...100) var value = 150
}
```

This becomes approximately:

```swift
private var _value = ClampedToRange(wrappedValue: 150, 0...100)
```

SE-0258 describes three storage-initialization routes: from the original property value through `init(wrappedValue:)`, from explicit wrapper initializer arguments, or from a no-argument `init()` when available. ([GitHub, "Property Wrappers"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md))

---

### 2.4 Memberwise initializers are affected

For a struct, the stored property is the wrapper storage, but the memberwise initializer may expose either the original property type or the wrapper type.

For this:

```swift
struct Score {
    @Clamped var value = 150
}
```

Swift effectively provides something like:

```swift
init(value: Int = 150) {
    self._value = Clamped(wrappedValue: value)
}
```

So this also clamps:

```swift
print(Score(value: 200).value)
print(Score(value: -20).value)
```

Output:

```text
100
0
```

SE-0258 says the memberwise initializer parameter uses the original property type when the wrapped property has an initial value or when the wrapper has `init(wrappedValue:)`; otherwise, it can use the wrapper type directly. ([GitHub, "Property Wrappers"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md))

---

### 2.5 Mutability follows the wrapper’s accessors

A wrapper can make a property’s getter or setter mutating / nonmutating in surprising ways.

```swift
@propertyWrapper
struct ReferenceBacked<Value> {
    final class Storage {
        var value: Value

        init(_ value: Value) {
            self.value = value
        }
    }

    private let storage: Storage

    init(wrappedValue: Value) {
        self.storage = Storage(wrappedValue)
    }

    var wrappedValue: Value {
        get { storage.value }
        nonmutating set { storage.value = newValue }
    }
}

struct Settings {
    @ReferenceBacked var count = 0
}

let settings = Settings()
settings.count = 10
print(settings.count)
```

Output:

```text
10
```

This compiles even though `settings` is a `let`, because the setter is `nonmutating` and mutates reference-backed storage. SE-0258 explicitly covers how mutating getters and nonmutating setters on wrapper types affect the synthesized property. ([GitHub, "Property Wrappers"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md))

That is useful in some cases, but dangerous if the API appears value-semantic while secretly sharing mutable reference state.

---

### 2.6 Wrapper composition nests wrappers

Multiple wrappers compose by nesting:

```swift
@A @B var value: Int
```

is conceptually:

```swift
private var _value: A<B<Int>>
```

The getter/setter goes through multiple `wrappedValue` layers.

```swift
var value: Int {
    get { _value.wrappedValue.wrappedValue }
    set { _value.wrappedValue.wrappedValue = newValue }
}
```

SE-0258 describes wrapper composition as nesting later wrapper types inside earlier wrapper types; for example, `@DelayedMutable @Copying var path` uses backing storage like `DelayedMutable<Copying<UIBezierPath>>`. ([GitHub, "Property Wrappers"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md))

Composition order matters. `@A @B` and `@B @A` are not generally equivalent.

---

## 3. Common traps and misconceptions

### Trap 1: Thinking the wrapped property is stored directly

Bad mental model:

```swift
struct Score {
    @Clamped var value = 150
}

// Mistaken assumption:
// Score stores Int directly.
```

Better mental model:

```swift
struct Score {
    private var _value = Clamped(wrappedValue: 150)

    var value: Int {
        get { _value.wrappedValue }
        set { _value.wrappedValue = newValue }
    }
}
```

The stored property is the wrapper. This matters for initialization, synthesized conformances, memory layout, access control, and concurrency.

---

### Trap 2: Treating wrappers as thread-safe because they look declarative

Bad:

```swift
@propertyWrapper
final class Locked<Value> {
    private var value: Value

    init(wrappedValue: Value) {
        self.value = wrappedValue
    }

    var wrappedValue: Value {
        get { value }
        set { value = newValue }
    }
}
```

The name says `Locked`, but there is no lock. Worse, even a wrapper that uses a lock can still expose unsafe compound operations:

```swift
@Locked var count = 0

count += 1
```

This is usually read-modify-write:

```text
read count
compute count + 1
write count
```

If the lock only protects each getter and setter separately, the whole increment is not atomic.

Better:

```swift
final class Counter {
    private let lock = NSLock()
    private var value = 0

    func increment() {
        lock.withLock {
            value += 1
        }
    }

    func read() -> Int {
        lock.withLock { value }
    }
}

private extension NSLock {
    func withLock<T>(_ operation: () throws -> T) rethrows -> T {
        lock()
        defer { unlock() }
        return try operation()
    }
}
```

Use a wrapper for simple access policy. Use a real synchronization abstraction for compound invariants.

---

### Trap 3: Hiding actor-isolation problems behind a wrapper

Bad:

```swift
@MainActor
final class UIStore {
    var title = ""
}

@propertyWrapper
struct UIBacked {
    private let store: UIStore

    init(wrappedValue: String) {
        self.store = UIStore()
        self.store.title = wrappedValue
    }

    var wrappedValue: String {
        get { store.title }
        set { store.title = newValue }
    }
}
```

This wrapper has a main-actor dependency. In Swift 6, accessing `store.title` from non-main-actor code is an isolation issue. A wrapper does not erase actor isolation. It can only hide the shape until the compiler or runtime forces you to face it.

Better:

```swift
@MainActor
@propertyWrapper
struct MainActorBacked {
    private var value: String

    init(wrappedValue: String) {
        self.value = wrappedValue
    }

    var wrappedValue: String {
        get { value }
        set { value = newValue }
    }
}
```

Now the wrapper’s isolation contract is explicit.

---

### Trap 4: Overusing wrappers for business logic

Bad:

```swift
struct Payment {
    @ValidCreditCard var cardNumber: String
    @ValidCurrency var amount: Decimal
    @BusinessRuleChecked var merchantID: String
}
```

This can hide domain rules inside local property access. Validation often depends on multiple fields, server configuration, jurisdiction, feature flags, or payment method.

Better:

```swift
struct PaymentDraft {
    var cardNumber: String
    var amount: Decimal
    var merchantID: String
}

struct PaymentValidator {
    func validate(_ draft: PaymentDraft) -> [PaymentValidationError] {
        // Cross-field, domain-level validation.
    }
}
```

Wrappers are good for local property behavior. They are a poor place for complex domain workflows.

---

## 4. Direct answers to rubric questions

### Q1. What code does a property wrapper effectively synthesize around a property?

A property wrapper synthesizes a private backing storage property of the wrapper type and a computed property that gets and sets through `wrappedValue`.

Example:

```swift
struct Score {
    @Clamped var value = 150
}
```

Approximately becomes:

```swift
struct Score {
    private var _value: Clamped = Clamped(wrappedValue: 150)

    var value: Int {
        get { _value.wrappedValue }
        set { _value.wrappedValue = newValue }
    }
}
```

If the wrapper defines `projectedValue`, Swift also synthesizes `$value`:

```swift
var $value: Projection {
    _value.projectedValue
}
```

Interview version:

> A property wrapper turns the declared property into computed access over synthesized wrapper storage. The backing property is usually named with an underscore, like `_value`, and stores the wrapper. The user-facing property delegates get and set to `_value.wrappedValue`. If the wrapper has `projectedValue`, Swift also exposes `$value`. The important senior-level point is that storage, initialization, mutability, access control, and synthesis all follow the wrapper, not the apparent property type.

---

### Q2. How can a wrapper accidentally hide thread-safety or actor-isolation problems?

A wrapper can make unsafe or isolated state look like an ordinary property. That is dangerous because call sites may assume simple field access, while the wrapper is actually touching shared mutable state, reference storage, global state, `UserDefaults`, a lock, an actor-isolated object, or a non-`Sendable` dependency.

Example problem:

```swift
@propertyWrapper
struct Shared<Value> {
    static var storage: [String: Any] = [:]

    let key: String

    var wrappedValue: Value {
        get { Self.storage[key] as! Value }
        set { Self.storage[key] = newValue }
    }
}
```

This wrapper hides global mutable state. In Swift 6 strict concurrency, static mutable state and cross-isolation access are exactly the kind of design that should be made explicit, not disguised.

Interview version:

> Wrappers can hide the real storage and execution assumptions. A property might look like a plain `Int`, but the wrapper could be touching shared reference state, a lock, `UserDefaults`, or main-actor-isolated UI state. That can mislead readers and sometimes the API design. In Swift 6, I would not use wrappers to silence isolation issues. I would make the isolation explicit with actors, global actors, immutable values, or a narrow synchronized abstraction.

---

## 5. Code probe

Given:

```swift
@propertyWrapper
struct Clamped {
    private var value: Int

    init(wrappedValue: Int) {
        value = min(max(wrappedValue, 0), 100)
    }

    var wrappedValue: Int {
        get { value }
        set { value = min(max(newValue, 0), 100) }
    }
}

struct Score {
    @Clamped var value = 150
}

print(Score().value)
```

### What happens?

```text
100
```

### Why?

`Score` does not store `value` as an `Int` directly. Swift synthesizes wrapper storage initialized through `Clamped.init(wrappedValue:)`.

Approximate expansion:

```swift
struct Score {
    private var _value: Clamped = Clamped(wrappedValue: 150)

    var value: Int {
        get { _value.wrappedValue }
        set { _value.wrappedValue = newValue }
    }

    init() {}
}
```

Initialization flow:

```text
Score()
  -> initializes _value with Clamped(wrappedValue: 150)
  -> Clamped.init clamps 150 into 0...100
  -> stored wrapper value becomes 100
  -> Score().value calls _value.wrappedValue getter
  -> prints 100
```

State diagram:

```text
Declared:
@Clamped var value = 150

Effective storage:
_value: Clamped(value: 100)

Public access:
value -> _value.wrappedValue -> 100
```

### Custom memberwise initialization interaction

Because `value` has an initial value and `Clamped` has `init(wrappedValue:)`, the synthesized memberwise initializer is approximately:

```swift
init(value: Int = 150) {
    self._value = Clamped(wrappedValue: value)
}
```

So these also clamp:

```swift
print(Score(value: 250).value)
print(Score(value: -10).value)
print(Score(value: 50).value)
```

Output:

```text
100
0
50
```

### Fix or redesign

The wrapper is correct for a simple invariant, but a more reusable production version should avoid hard-coding `0...100`:

```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    private let range: ClosedRange<Value>

    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = range.clamp(wrappedValue)
    }

    var wrappedValue: Value {
        get { value }
        set { value = range.clamp(newValue) }
    }
}

private extension ClosedRange {
    func clamp(_ value: Bound) -> Bound {
        min(max(value, lowerBound), upperBound)
    }
}

struct Score {
    @Clamped(0...100) var value = 150
}

print(Score().value)
```

Output:

```text
100
```

### Why this fix is correct

The invariant belongs to the wrapper: every initialization and assignment passes through the same clamping rule. The range is explicit at the declaration site, so the reader does not need to know hidden constants inside `Clamped`.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Property wrapper|Local invariant applies to one property independently|Can hide storage and access semantics|
|Custom type, e.g. `ScoreValue`|The value has domain meaning across the system|More explicit, better for APIs, slightly more ceremony|
|Validation service|Validation depends on multiple fields or external rules|Not as lightweight, but better for real domain logic|
|Setter validation|One-off type-local invariant|Less reusable, but more obvious|

---

## 6. Exercise

### Problem

Implement a small wrapper that validates inputs or logs mutations, and explain its initialization behavior.

### Bad / naive version

```swift
struct Profile {
    var age: Int = 0 {
        didSet {
            if age < 0 {
                age = 0
            }
            if age > 150 {
                age = 150
            }
            print("age changed to \(age)")
        }
    }
}
```

### What is wrong?

```text
The logic is tied to one property.
The same validation will be duplicated elsewhere.
didSet does not run during initialization.
The logging and validation policy is not reusable.
The allowed range is not explicit in the property's type or declaration.
```

### Improved version

```swift
@propertyWrapper
struct LoggedClamped<Value: Comparable> {
    private var value: Value
    private let range: ClosedRange<Value>
    private let name: String

    init(
        wrappedValue: Value,
        _ range: ClosedRange<Value>,
        name: String
    ) {
        self.range = range
        self.name = name

        let clamped = range.clamp(wrappedValue)
        self.value = clamped

        print("\(name) initialized with \(clamped)")
    }

    var wrappedValue: Value {
        get { value }
        set {
            let oldValue = value
            let newClampedValue = range.clamp(newValue)
            value = newClampedValue
            print("\(name) changed from \(oldValue) to \(newClampedValue)")
        }
    }

    var projectedValue: ClosedRange<Value> {
        range
    }
}

private extension ClosedRange {
    func clamp(_ value: Bound) -> Bound {
        min(max(value, lowerBound), upperBound)
    }
}

struct Profile {
    @LoggedClamped(0...150, name: "age") var age = 200
}

var profile = Profile()
print(profile.age)
profile.age = -10
print(profile.age)
print(profile.$age)
```

Output:

```text
age initialized with 150
150
age changed from 150 to 0
0
0...150
```

### Why this is better

The validation policy is reusable. Initialization and assignment both go through the same clamping rule. The declaration site shows the allowed range. The projection exposes metadata about the wrapper without exposing private storage.

The effective storage is:

```swift
private var _age = LoggedClamped(wrappedValue: 200, 0...150, name: "age")
```

The public property is:

```swift
var age: Int {
    get { _age.wrappedValue }
    set { _age.wrappedValue = newValue }
}
```

The projected property is:

```swift
var $age: ClosedRange<Int> {
    _age.projectedValue
}
```

---

## 7. Production guidance

Use this in production when:

```text
The behavior is local to a single property.
The wrapper makes a repeated storage/access pattern clearer.
The invariant is simple and independent.
The wrapper has obvious semantics from its name and initializer.
The wrapper does not hide surprising global, thread, actor, or lifecycle behavior.
```

Good examples:

```swift
@Clamped(0...100) var progress = 0
@Trimmed var displayName = ""
@UserDefault("hasSeenOnboarding") var hasSeenOnboarding = false
@MainActorBacked var title = ""
```

Be careful when:

```text
The wrapper uses reference storage.
The wrapper has a nonmutating setter.
The wrapper accesses global mutable state.
The wrapper touches UserDefaults, Keychain, files, database, or network.
The wrapper claims synchronization.
The wrapper is used inside Sendable types.
The wrapper is used on actor-isolated or global-actor-isolated types.
The wrapper affects Codable, Hashable, Equatable, or Sendable synthesis.
```

Avoid when:

```text
The logic depends on multiple fields.
The logic is domain workflow, not property access.
The wrapper is used to hide dependency injection.
The wrapper hides async work behind synchronous property access.
The wrapper silences Swift 6 isolation warnings instead of fixing the design.
The wrapper makes simple code harder to read.
```

Debugging checklist:

```text
What is the actual synthesized backing storage type?
Does the wrapper use value storage or reference storage?
Is the setter mutating or nonmutating?
Does the wrapper define projectedValue?
Does $property expose safe API or unsafe internals?
Does the wrapper touch global/shared state?
Is the wrapper Sendable?
Is the wrapper actor-isolated or global-actor-isolated?
Does it affect synthesized Codable/Hashable/Equatable?
Is wrapper composition order changing behavior?
Would an explicit type or service be clearer?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> A property wrapper lets you add behavior to a property using `wrappedValue`.

### Senior answer

> A property wrapper stores a wrapper instance and exposes the declared property as computed access through `wrappedValue`. I pay attention to initialization, projection, mutability, access control, and whether the wrapper hides reference storage or side effects.

### Staff-level answer

> Property wrappers are API-shaping tools, not just syntax sugar. They are useful when they make repeated property storage policies explicit and reusable, but they can become dangerous when they hide synchronization, actor isolation, persistence, dependency lookup, or domain logic. In a Swift 6 codebase, I would review wrappers for sendability, global actor isolation, synthesized conformances, and whether the apparent property access still communicates the real cost and correctness model.

Staff-level questions to ask:

```text
Does this wrapper make the code clearer at the call site and declaration site?
Is the wrapper hiding storage, synchronization, or isolation that should be explicit?
What are the synthesized backing storage and memberwise initializer shapes?
Does this wrapper affect Codable, Hashable, Equatable, or Sendable synthesis?
Would this be better as a domain type, actor, service, macro, or plain property?
```

---

## 9. Interview-ready summary

A property wrapper turns a property declaration into computed access over synthesized wrapper storage. The wrapper stores the real state, `wrappedValue` defines the apparent property value, and `projectedValue` optionally defines `$property`. Initialization usually flows through `init(wrappedValue:)`, and memberwise initializers can expose either the wrapped type or wrapper type depending on the declaration. The senior-level caveat is that wrappers can hide mutability, reference storage, side effects, synchronization, actor isolation, and synthesized-conformance behavior. I use them for simple, local, reusable property policies, not for complex domain logic or to paper over concurrency problems.

---

## 10. Flashcards

Q: What does `@Wrapper var x: T` effectively synthesize?

A: A private backing property like `_x: Wrapper` plus a computed `x: T` that gets and sets `_x.wrappedValue`.

Q: What is `wrappedValue`?

A: The property on the wrapper type that defines the apparent value exposed by the wrapped property.

Q: What is `projectedValue`?

A: Optional wrapper-defined API exposed through `$property`.

Q: What does `@Clamped var value = 150` do during initialization?

A: It initializes backing storage with `Clamped(wrappedValue: 150)`, so the wrapper can transform or validate the initial value.

Q: What does the B13 code probe print?

A: `100`, because `150` is clamped during `Clamped.init(wrappedValue:)`.

Q: Why can wrappers be dangerous in concurrent code?

A: They can hide shared mutable state, reference storage, locks, global state, non-`Sendable` dependencies, or actor-isolated state behind normal property syntax.

Q: How does wrapper composition work?

A: Wrappers nest. `@A @B var x: Int` is conceptually backed by something like `A<B<Int>>`.

Q: When is a property wrapper the wrong tool?

A: When the behavior is cross-field, workflow-level, async, actor-dependent, synchronization-heavy, or domain-specific enough to deserve an explicit type or service.

---

## 11. Related sections

- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[A11 — Access control]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[D5 — Global Actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[F6 — Observation, property wrappers, and language-adjacent state tools]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — B13.
- Swift.org. "Properties." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/
- Swift.org. "Property Wrapper Implementation Requirements." Swift Compiler Diagnostics. https://docs.swift.org/compiler/documentation/diagnostics/property-wrapper-requirements/
- GitHub. "Property Wrappers." Swift Evolution SE-0258. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md
