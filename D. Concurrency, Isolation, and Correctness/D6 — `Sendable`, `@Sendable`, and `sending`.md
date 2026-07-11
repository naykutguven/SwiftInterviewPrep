---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - sendable
  - strict-concurrency
---
# D6 — `Sendable`, `@Sendable`, and `sending`

## 0. Rubric snapshot

**Rubric expectation**

Understand `Sendable`, `@Sendable`, generic sendability constraints, and `sending` parameters/results for value types, classes, actors, closures, and generic APIs. The rubric's D6 probe focuses on why capturing a mutable class instance in a `@Sendable` async closure is unsafe under Swift 6 strict concurrency, and why a `sending` handoff requires safe value transfer.

**Caveats**

- `@unchecked Sendable` is a proof obligation, not a compiler silencer.
- Sendability is about safe cross-isolation transfer, not merely “thread use.”
- `@Sendable` closures restrict captured values because they may be transferred to another concurrency domain or invoked concurrently.
- `sending` is value-transfer semantics at a parameter/result boundary, not `Sendable` conformance, not `@Sendable` closure checking, and not move-only ownership.
- A class being `final` is not enough; its stored state and mutation model matter.

**You should be able to answer**

- Why can a final immutable class sometimes be `Sendable`, and why are many mutable classes not?
- What does `@Sendable` require of captured values?
- How is `sending T` different from `T: Sendable` and `@Sendable`?
- Why can `sending` sometimes allow transfer of a non-`Sendable` value?

**You should be able to do**

- Take a non-sendable service type used in async code and propose the safest fix among actorization, value modeling, or unchecked conformance.
- Given an API that accepts a `sending` parameter or returns a `sending` result, explain what ownership/aliasing assumptions the compiler is enforcing and what would make the transfer unsafe.
- Explain the code probe’s Swift 6 compiler error and redesign it safely.

---

## 1. Core mental model

Swift’s concurrency model protects against **data races** by tracking when values cross concurrency domains: tasks, actors, global actors, and `@Sendable` closures. A type is `Sendable` when values of that type can be safely transferred across such boundaries. The Swift Book describes a sendable type as one that can be shared from one concurrency domain to another. ([Swift.org, "Concurrency"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/))

`Sendable` is a **marker protocol**. It does not add methods. It communicates a semantic guarantee to the compiler: “values of this type can safely cross isolation boundaries.” SE-0302 introduced `Sendable` and `@Sendable` to type-check value passing between structured concurrency constructs and actor messages. ([GitHub, "Sendable and @Sendable Closures"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md))

`@Sendable` is not a protocol conformance on a nominal type. It is an attribute on a **function type**. A `@Sendable` closure is safe to pass across concurrency domains, so Swift checks what it captures. The key rule: captured values must be safe to share. SE-0302 states that a `@Sendable` function’s captures must conform to `Sendable`, and mutable captured variables need explicit by-value capture if appropriate. ([GitHub, "Sendable and @Sendable Closures"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md))

`sending T` is different again. It is a **value-level transfer** annotation on a parameter or result at a specific isolation boundary. SE-0430 added `sending` so Swift can allow safe transfer of some non-`Sendable` values when the value is disconnected from other aliases that could keep using it concurrently. ([GitHub, "Transferring Parameters and Results"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md))

That does not mean the type is generally `Sendable`. A mutable class can be passed as `sending Cache` at one boundary if the compiler can prove the value is safely handed off there, but `Cache` has not become safe to share between arbitrary tasks. It also does not mean move-only ownership. Ordinary copyable Swift values can participate in a `sending` transfer; the compiler is reasoning about isolation regions and aliases, not saying the type is `~Copyable`.

`nonsending` is the opposite default: arguments and results are not sent across an isolation boundary. This matters most in Swift 6.2 discussions of `nonisolated(nonsending)`, where a nonisolated async function can run in the caller's isolation context and does not force arguments/results to be transferred away by default.

The key idea:

```text
Sendable = safe value transfer across isolation boundaries
@Sendable = safe function/closure transfer across isolation boundaries
sending T = this value is transferred at this parameter/result boundary
nonsending = arguments/results stay in the caller's isolation context by default
```

`Sendable` does **not** mean “this thing is always immutable.” It means the public behavior of the value is safe to use after being sent across concurrency domains. Value-semantic structs often satisfy this naturally. Actors satisfy it because actor state is isolated. Mutable classes usually do not, because the same reference identity can be accessed concurrently from multiple domains.

---

## 2. Essential mechanics

### Value types are usually the easiest Sendable types

Structs and enums can conform to `Sendable` when their stored properties or associated values are also `Sendable`. Many standard-library value types, such as `Int`, `String`, `Array<Element>` where `Element: Sendable`, and `Dictionary<Key, Value>` where both generic arguments are `Sendable`, are sendable.

```swift
struct UserID: Sendable {
    let rawValue: String
}

struct Profile: Sendable {
    let id: UserID
    let displayName: String
}
```

A struct is not magically sendable if it contains a non-sendable reference:

```swift
final class ImageCache {
    var storage: [String: Data] = [:]
}

struct LoaderState: Sendable {
    let cache: ImageCache
    // Error in Swift 6:
    // stored property 'cache' of 'Sendable'-conforming struct
    // has non-sendable type 'ImageCache'
}
```

The compiler can structurally check structs and enums because it can inspect their stored data. SE-0302 specifies that structs and enums can be checked for `Sendable` conformance based on whether their members are sendable. ([GitHub, "Sendable and @Sendable Closures"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md))

---

### Classes are Sendable only under stricter conditions

A class reference points to shared identity. If two tasks hold the same class instance, they can potentially read and mutate the same storage concurrently.

This can be checked safely in a narrow case:

```swift
final class APIConfiguration: Sendable {
    let baseURL: URL
    let timeout: Duration

    init(baseURL: URL, timeout: Duration) {
        self.baseURL = baseURL
        self.timeout = timeout
    }
}
```

Why this is acceptable:

```text
final class
+ immutable let storage
+ all stored properties are Sendable
= no subclass can add unsafe mutable state
= no exposed mutable shared state to race on
```

This is not acceptable:

```swift
final class Cache: Sendable {
    var store: [String: Int] = [:]
    // Error: mutable stored property in a Sendable class
}
```

A mutable class can still be safely shareable if it uses internal synchronization, but the compiler generally cannot prove that. That is when `@unchecked Sendable` may be justified. SE-0302 explicitly treats `@unchecked Sendable` as the escape hatch for classes whose memory safety is provided by access control and internal synchronization. ([GitHub, "Sendable and @Sendable Closures"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md))

```swift
import Synchronization

final class LockedCache: @unchecked Sendable {
    private let store = Mutex<[String: Int]>([:])

    func set(_ key: String, value: Int) {
        store.withLock { dictionary in
            dictionary[key] = value
        }
    }

    func value(for key: String) -> Int? {
        store.withLock { dictionary in
            dictionary[key]
        }
    }
}
```

The danger is not the syntax. The danger is lying to the compiler.

---

### Actors are implicitly Sendable

Actor references are safe to transfer because their mutable state is protected by actor isolation.

```swift
actor Cache {
    private var store: [String: Int] = [:]

    func set(_ key: String, value: Int) {
        store[key] = value
    }

    func value(for key: String) -> Int? {
        store[key]
    }
}
```

You can send the actor reference across tasks, but you cannot directly touch its isolated state from outside:

```swift
let cache = Cache()

Task {
    await cache.set("a", value: 1)
}
```

Actor references are sendable because access to actor-isolated mutable state requires actor hops. SE-0302 notes that actor types provide their own internal synchronization and implicitly conform to `Sendable`. ([GitHub, "Sendable and @Sendable Closures"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md))

---

### `@Sendable` constrains closure captures

A closure passed to an API like this:

```swift
func run(_ operation: @Sendable @escaping () async -> Void) {}
```

is allowed to escape and be run from another concurrency domain. Therefore, Swift checks its captures.

Safe:

```swift
let id = "user-1"

run {
    print(id)
}
```

`String` is `Sendable`, and `id` is an immutable capture.

Unsafe:

```swift
final class Counter {
    var value = 0
}

let counter = Counter()

run {
    counter.value += 1
}
```

The closure captures a non-sendable reference type and mutates shared state. Swift 6 rejects this because the closure could run concurrently with other code that also has access to the same `counter`.

Mutable local captures are also restricted:

```swift
var total = 0

run {
    total += 1
    // Error: mutation of captured var in concurrently-executing code
}
```

A capture list can create a by-value snapshot, but that only helps if the captured value itself is `Sendable`:

```swift
var suffix = "!"

run { [suffix] in
    print("Done\(suffix)")
}
```

---

### Generic APIs need explicit Sendable constraints when values cross isolation

If a generic value is stored, sent to a task, returned across an actor boundary, or captured by a `@Sendable` closure, the generic parameter often needs a `Sendable` constraint.

Bad:

```swift
func process<T>(_ value: T) {
    Task {
        print(value)
    }
}
```

Better:

```swift
func process<T: Sendable>(_ value: T) {
    Task {
        print(value)
    }
}
```

For a generic container:

```swift
struct Box<Value: Sendable>: Sendable {
    let value: Value
}
```

Do not add `T: Sendable` everywhere by reflex. Add it where the API’s semantics actually require values to cross concurrency domains.

---

### `sending` transfers a value at one boundary

`sending` belongs to a parameter or result, not to the type declaration. It says the value being passed or returned must be safe to transfer at that boundary.

Start with a non-`Sendable` mutable class:

```swift
final class ScratchBuffer {
    var bytes: [UInt8] = []
}
```

This type is not generally safe to share across tasks. But a fresh, unaliased instance can be handed off:

```swift
@MainActor
func consumeOnMain(_ buffer: sending ScratchBuffer) {
    buffer.bytes.append(1)
}

func buildAndSend() async {
    let buffer = ScratchBuffer()
    buffer.bytes.append(0)

    await consumeOnMain(buffer)
}
```

The point is not that `ScratchBuffer` became `Sendable`. The point is that this particular value is transferred away at this call boundary.

### `sending` results must be disconnected from the callee

A function returning `sending T` promises the result can be safely transferred to the caller.

Good:

```swift
@MainActor
func makeBuffer() -> sending ScratchBuffer {
    ScratchBuffer()
}

func useBuffer() async {
    let buffer = await makeBuffer()
    await consumeOnMain(buffer)
}
```

The returned buffer is newly created, so it is not still stored in the main actor's state.

Invalid transfer shape:

```swift
@MainActor
final class BufferStore {
    private let cached = ScratchBuffer()

    func current() -> sending ScratchBuffer {
        cached
        // Not a disconnected result: the store still owns this object.
    }
}
```

Returning `cached` as `sending` would let the caller use the same non-`Sendable` object while the main actor can also keep using it. SE-0430 models this as an invalid `sending` result; current compiler diagnostics can lag on some region-isolation edge cases, so the semantic rule matters more than one exact diagnostic.

### Unsafe aliases make a transfer invalid

Swift may reject a `sending` transfer when another alias can still use the same non-`Sendable` object after the handoff.

```swift
func unsafeAlias() async {
    let buffer = ScratchBuffer()
    let alias = buffer

    await consumeOnMain(buffer)

    alias.bytes.append(2)
    // Unsafe: later use through 'alias' could race with the transferred value.
}
```

The compiler is not being picky about variable names. `buffer` and `alias` point to the same object:

```text
caller alias ─┐
              ├── same ScratchBuffer instance
sending arg ──┘
```

If the callee can keep or merge the value into another isolation region, later mutation through `alias` would be concurrent access to non-`Sendable` shared state.

### `nonsending` keeps values in the caller's isolation context

`nonsending` means arguments and results are not sent across an isolation boundary by default. In Swift 6.2 discussions, this appears most often as `nonisolated(nonsending)` for async functions.

```swift
nonisolated(nonsending) func renderTitle(_ title: String) async -> String {
    title.lowercased()
}
```

The important distinction is conceptual:

```text
nonisolated / isolated = where code executes
sending / nonsending   = whether values transfer across an isolation boundary
```

Do not use `nonsending` as a performance model. It describes value transfer. If the work must leave the caller's actor, that is a D14 `@concurrent` question.

---

## 3. Common traps and misconceptions

### Trap 1: Treating `@unchecked Sendable` as a migration shortcut

Bad:

```swift
final class ImagePipeline: @unchecked Sendable {
    var cache: [URL: UIImage] = [:]
}
```

This compiles, but it is still unsafe. Multiple tasks can mutate `cache` concurrently.

Better:

```swift
actor ImagePipeline {
    private var cache: [URL: UIImage] = [:]

    func image(for url: URL) async throws -> UIImage {
        if let cached = cache[url] {
            return cached
        }

        let image = try await loadImage(from: url)
        cache[url] = image
        return image
    }

    private func loadImage(from url: URL) async throws -> UIImage {
        // Network + decode work.
        UIImage()
    }
}
```

Use `@unchecked Sendable` only when the type has a real synchronization invariant that you can document and test.

---

### Trap 2: Thinking `let` makes a class instance safe

Bad:

```swift
final class SessionState {
    var token: String?
}

let state = SessionState()

run {
    state.token = "abc"
}
```

`let state` freezes the reference, not the object behind the reference. The object’s `var token` is still mutable shared state.

Better:

```swift
struct SessionState: Sendable {
    let token: String?
}
```

or:

```swift
actor SessionStore {
    private var token: String?

    func updateToken(_ token: String?) {
        self.token = token
    }
}
```

---

### Trap 3: Assuming `@Sendable` means “runs on a background thread”

`@Sendable` says something about **type safety of a closure crossing concurrency domains**. It does not say where the closure runs.

```swift
@MainActor
func updateUI(_ body: @Sendable @MainActor () -> Void) {
    body()
}
```

This closure is sendable and main-actor-isolated. Those are different facts:

```text
@Sendable  -> safe to transfer as a closure value
@MainActor -> must execute on the main actor
```

---

### Trap 4: Making protocols `Sendable` without understanding conformers

This can be good:

```swift
protocol AnalyticsEvent: Sendable {
    var name: String { get }
}
```

This says every event value must be safe to cross concurrency domains.

This can be over-constrained:

```swift
protocol ViewRenderable: Sendable {
    func render()
}
```

If conformers are UI objects, delegates, or reference-heavy adapters, `Sendable` might be the wrong semantic promise. In iOS code, UI-related types are often better isolated to `@MainActor` than made `Sendable`.

---

### Trap 5: Treating `sending` as `Sendable`

`sending Cache` does not mean `Cache: Sendable`.

Bad mental model:

```swift
final class Cache {
    var store: [String: Int] = [:]
}

func handOff(_ cache: sending Cache) async {}

func later(cache: Cache) {
    Task.detached {
        cache.store["a"] = 1
    }
}
```

The `handOff` API may accept a one-time safe transfer. It does not make every future use of `Cache` safe inside detached tasks or `@Sendable` closures.

Better mental model:

```text
Sendable Cache    = every Cache value can safely cross isolation boundaries.
sending Cache     = this Cache value is transferred at this one boundary.
@Sendable closure = this function value is safe to transfer; captures are checked.
```

### Trap 6: Treating `sending` as move-only

`sending` is not Swift's move-only ownership system. It does not make the type noncopyable, and it does not mean all values of that type can only be used once.

What it does mean:

```text
At this parameter/result boundary, Swift must be able to prove that
using the transferred region elsewhere cannot race with the destination.
```

If there is an alias that can be used after the transfer, Swift may reject the call or the later use.

Bad:

```swift
func handOff(_ cache: sending Cache) async {}

func invalid() async {
    let cache = Cache()
    let alias = cache

    await handOff(cache)
    alias.store["a"] = 1
}
```

Better:

```swift
func valid() async {
    let cache = Cache()
    cache.store["a"] = 1

    await handOff(cache)
    // Do not keep using this non-Sendable object in the caller.
}
```

---

## 4. Direct answers to rubric questions

### Q1. Why can a final immutable class sometimes be Sendable, and why are many mutable classes not?

A final immutable class can sometimes be `Sendable` because sharing the reference does not expose mutable shared state. `final` prevents subclasses from adding unsafe mutable storage, and immutable `let` stored properties of `Sendable` types can be checked by the compiler.

```swift
final class FeatureFlags: Sendable {
    let isNewCheckoutEnabled: Bool

    init(isNewCheckoutEnabled: Bool) {
        self.isNewCheckoutEnabled = isNewCheckoutEnabled
    }
}
```

Many mutable classes are not `Sendable` because class instances have shared identity. If two tasks hold the same instance, they can concurrently access the same mutable storage.

```swift
final class MutableFeatureFlags {
    var isNewCheckoutEnabled = false
}
```

The compiler cannot assume this is safe. It does not know whether your code will serialize access, use a lock, or accidentally race.

Interview version:

> A final immutable class can be `Sendable` when it contains only immutable `Sendable` state, because sharing the reference cannot cause concurrent mutation, and `final` prevents subclasses from adding unsafe state. Mutable classes usually are not `Sendable` because the same reference identity can be accessed from multiple concurrency domains. If a mutable class is truly safe, it needs actor isolation, value redesign, or a carefully justified `@unchecked Sendable` conformance backed by synchronization.

---

### Q2. What does `@Sendable` require of captured values?

A `@Sendable` closure may be transferred to another concurrency domain, so captured values must be safe to share. Captured values generally need to conform to `Sendable`. Mutable captured variables are especially restricted because they are captured by reference-like storage and could be accessed concurrently.

Safe:

```swift
let prefix = "user"

run {
    print(prefix)
}
```

Unsafe:

```swift
var count = 0

run {
    count += 1
}
```

Also unsafe:

```swift
final class Cache {
    var store: [String: Int] = [:]
}

let cache = Cache()

run {
    cache.store["a"] = 1
}
```

The second example captures mutable local state. The third captures a non-sendable mutable reference type.

Interview version:

> `@Sendable` is part of a function type. It means the function value can safely cross concurrency domains. For closures, Swift checks the capture list: captured values must be `Sendable`, and mutable captures cannot be shared by reference into concurrently executing code. A capture list can make a snapshot, but the captured value still has to be sendable.

---

### Q3. How is `sending T` different from `T: Sendable` and `@Sendable`?

`T: Sendable` is a type constraint: every value of `T` must satisfy the sendability promise. `@Sendable` is a function-type attribute: the closure/function value can be transferred, so its captures are checked. `sending T` is a value-transfer annotation: this particular argument or result must be safely transferable at this boundary.

```swift
func requiresType<Value: Sendable>(_ value: Value) {}
func requiresClosure(_ operation: @Sendable () -> Void) {}
func requiresTransfer(_ cache: sending Cache) async {}
```

They answer different questions:

```text
T: Sendable      -> Is this type generally safe to send?
@Sendable        -> Is this function value safe to send?
sending T        -> Is this particular value safe to transfer here?
```

Interview version:

> `Sendable` is a type-level promise. `@Sendable` is a function-type promise about a closure value and its captures. `sending` is narrower: it applies to one parameter or result and says the value is being transferred at that boundary. A non-`Sendable` class might be passed as `sending` if the compiler can prove there are no live aliases that could race, but the class has not become generally `Sendable`.

---

### Q4. Why can `sending` sometimes allow transfer of a non-`Sendable` value?

Because some non-`Sendable` values are unsafe to **share**, but still safe to **transfer**. If a mutable class instance is newly created and no caller-side alias will keep using it, Swift can allow it to move from one isolation region to another.

```swift
final class Cache {
    var store: [String: Int] = [:]
}

@MainActor
func install(_ cache: sending Cache) {
    cache.store["ready"] = 1
}

func makeAndInstall() async {
    let cache = Cache()
    await install(cache)
}
```

This is safe because the caller is not trying to keep using `cache` after the transfer. It becomes unsafe if an alias remains live:

```swift
func broken() async {
    let cache = Cache()
    let alias = cache

    await install(cache)
    alias.store["late"] = 1
}
```

Swift may diagnose the handoff or the later alias use because the same mutable object could be used from both isolation regions.

Interview version:

> `sending` works for some non-`Sendable` values because Swift is not claiming the type is shareable. It is proving a one-time transfer of a disconnected value. If the source side keeps an alias and uses it after the handoff, the transfer is unsafe and Swift should reject it.

---

## 5. Code probe

Given:

```swift
final class Cache {
    var store: [String: Int] = [:]
}

func run(_ operation: @Sendable @escaping () async -> Void) {}

let cache = Cache()

run {
    cache.store["a"] = 1
}
```

### What happens?

With Swift 6.2.1 and `swiftc -swift-version 6`, as written at top level, this does **not** compile:

```text
error: non-Sendable type 'Cache' of let 'cache' cannot exit main actor-isolated context
note: class 'Cache' does not conform to the 'Sendable' protocol

error: main actor-isolated let 'cache' cannot be accessed from outside of the actor
```

If the same probe is placed inside a normal function, the clearer canonical diagnostic is:

```text
error: capture of 'cache' with non-Sendable type 'Cache' in a '@Sendable' closure [#SendableClosureCaptures]
note: class 'Cache' does not conform to the 'Sendable' protocol
```

There is no runtime output because compilation fails.

A related Swift 6 diagnostic often appears with `Task.detached`:

```text
error: passing closure as a 'sending' parameter risks causing data races
between code in the current task and concurrent execution of the closure
```

That wording means the closure value itself is being transferred into concurrent execution. If the closure captures `self` or another mutable non-`Sendable` reference that remains usable in the current task, Swift sees a possible race between the source task and the detached task.

### Why?

`run` accepts this closure type:

```swift
@Sendable @escaping () async -> Void
```

That means the closure can escape and may be executed from another concurrency domain. Swift therefore checks its captures.

The closure captures:

```swift
let cache = Cache()
```

But `Cache` is a mutable class:

```swift
final class Cache {
    var store: [String: Int] = [:]
}
```

The class has shared identity and mutable state. If this closure runs concurrently with other code that also has `cache`, both domains could read/write `store` at the same time.

```text
Main/task A owns reference ─┐
                            ├── same Cache instance ── mutable store
Task/closure B captures ────┘
```

That is exactly the kind of shared mutable state Swift 6 strict concurrency is designed to reject. Swift 6’s opt-in language mode diagnoses potential data races as compiler errors. ([Swift.org, "Announcing Swift 6"](https://swift.org/blog/announcing-swift-6/))

### Fix or redesign — actor isolation

```swift
actor Cache {
    private var store: [String: Int] = [:]

    func set(_ key: String, value: Int) {
        store[key] = value
    }

    func value(for key: String) -> Int? {
        store[key]
    }
}

func run(_ operation: @Sendable @escaping () async -> Void) {}

let cache = Cache()

run {
    await cache.set("a", value: 1)
}
```

### Why this fix is correct

The mutable dictionary is now actor-isolated. The closure captures a `Cache` actor reference, and actor references are safe to transfer. Mutation goes through the actor:

```text
@Sendable closure
    captures actor reference
        await cache.set(...)
            serializes access to actor-isolated store
```

The mutation is no longer direct shared mutable access.

### Follow-up: `sending` parameter handoff

Given:

```swift
final class Cache {
    var store: [String: Int] = [:]
}

func handOff(_ cache: sending Cache) async {}

func broken() async {
    let cache = Cache()

    await handOff(cache)
    cache.store["a"] = 1
}
```

What happens?

```text
Swift may reject the transfer or the later use because 'cache' is a
non-Sendable value used again after being passed as a sending parameter.
```

Why?

`handOff(_:)` is allowed to send `cache` to another isolation region. If the caller then mutates `cache.store`, the same mutable class instance may be accessed in both places. `sending` only works when the value is disconnected from later source-side uses.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Actor isolation|Shared mutable state accessed from multiple tasks|Async API surface; every isolated access may require `await`|
|Value modeling|State can be copied, snapshotted, or transformed immutably|May require redesigning APIs around returned values instead of in-place mutation|
|`@unchecked Sendable` + lock|Performance-sensitive synchronous API with a small, well-defined invariant|Compiler stops checking the promise; correctness depends on your lock discipline|
|`@MainActor` isolation|UI-owned mutable state that must only be touched on the main actor|Can over-serialize work and accidentally move non-UI work to the main actor|

Value-style redesign:

```swift
struct CacheSnapshot: Sendable {
    var store: [String: Int] = [:]

    func setting(_ key: String, to value: Int) -> Self {
        var copy = self
        copy.store[key] = value
        return copy
    }
}

let cache = CacheSnapshot()

run { [cache] in
    let updated = cache.setting("a", to: 1)
    _ = updated
}
```

Locked class redesign:

```swift
import Synchronization

final class Cache: @unchecked Sendable {
    private let store = Mutex<[String: Int]>([:])

    func set(_ key: String, value: Int) {
        store.withLock { dictionary in
            dictionary[key] = value
        }
    }

    func value(for key: String) -> Int? {
        store.withLock { dictionary in
            dictionary[key]
        }
    }
}

let cache = Cache()

run {
    cache.set("a", value: 1)
}
```

The locked version is acceptable only if all mutable access goes through the lock. One exposed unsafe mutable property invalidates the proof.

---

## 6. Exercise

### Problem

Take a non-sendable service type used in async code and propose the safest fix among actorization, value modeling, or unchecked conformance. Then explain whether a `sending` parameter or `sending` result would be safe for a non-`Sendable` value at a specific API boundary.

### Bad / naive version

```swift
final class TokenService {
    var token: String?
    var refreshCount = 0

    func refreshToken() async throws -> String {
        refreshCount += 1

        let newToken = try await fetchTokenFromServer()
        token = newToken
        return newToken
    }

    private func fetchTokenFromServer() async throws -> String {
        "token-\(UUID().uuidString)"
    }
}

func perform(_ operation: @Sendable @escaping () async -> Void) {}

let service = TokenService()

perform {
    _ = try? await service.refreshToken()
}
```

### What is wrong?

```text
TokenService is a mutable reference type.
The @Sendable closure captures it.
Multiple tasks could call refreshToken() concurrently.
token and refreshCount can be read/written at the same time.
Swift cannot prove this is safe.
```

This service also has a logical race: two simultaneous refreshes can both fetch new tokens, and the slower one can overwrite the newer token.

### Improved version: actorization

```swift
actor TokenService {
    private var token: String?
    private var refreshCount = 0
    private var refreshTask: Task<String, Error>?

    func validToken() async throws -> String {
        if let token {
            return token
        }

        if let refreshTask {
            return try await refreshTask.value
        }

        let task = Task<String, Error> {
            try await Self.fetchTokenFromServer()
        }

        refreshTask = task

        do {
            let newToken = try await task.value
            token = newToken
            refreshCount += 1
            refreshTask = nil
            return newToken
        } catch {
            refreshTask = nil
            throw error
        }
    }

    private static func fetchTokenFromServer() async throws -> String {
        "token-\(UUID().uuidString)"
    }
}
```

Usage:

```swift
func perform(_ operation: @Sendable @escaping () async -> Void) {}

let service = TokenService()

perform {
    _ = try? await service.validToken()
}
```

### Why this is better

The actor owns the mutable state:

```text
token
refreshCount
refreshTask
```

External tasks cannot directly mutate those properties. They must call actor-isolated methods. This solves the sendability problem and gives you one place to reason about token-refresh invariants.

This version also prevents duplicate refresh work by storing an in-flight `Task`.

### Alternative: value modeling

Use value modeling when the service does not truly need identity or shared mutable state.

```swift
struct TokenRequest: Sendable {
    let userID: String
    let deviceID: String
}

struct TokenResponse: Sendable {
    let token: String
    let expirationDate: Date
}

struct TokenClient: Sendable {
    var fetchToken: @Sendable (TokenRequest) async throws -> TokenResponse

    func token(for request: TokenRequest) async throws -> TokenResponse {
        try await fetchToken(request)
    }
}
```

This is better when the “service” is really just immutable configuration plus behavior. State lives elsewhere.

### Alternative: `@unchecked Sendable`

Use this only when you need a synchronous reference type and have a small synchronization boundary.

```swift
import Synchronization

final class TokenStore: @unchecked Sendable {
    private let state = Mutex<State>(State())

    struct State {
        var token: String?
        var refreshCount = 0
    }

    func token() -> String? {
        state.withLock { $0.token }
    }

    func updateToken(_ token: String) {
        state.withLock {
            $0.token = token
            $0.refreshCount += 1
        }
    }
}
```

This is safe only if:

```text
all mutable state is private
all reads/writes use the same synchronization primitive
no method exposes mutable internal references
callbacks are not invoked while holding the lock
the invariant is documented and tested
```

### Sending API check

Use `sending` only when the API really transfers a disconnected value region.

```swift
final class DraftTokenStore {
    private(set) var token: String?

    func updateToken(_ token: String) {
        self.token = token
    }
}
```

Parameter handoff:

```swift
func install(_ store: sending DraftTokenStore) async {}

func makeStore() async {
    let store = DraftTokenStore()
    store.updateToken("abc")

    await install(store)
}
```

This shape is plausible because the caller builds the store, initializes it, and stops using it after the handoff.

Unsafe alias:

```swift
func invalidInstall() async {
    let store = DraftTokenStore()
    let alias = store

    await install(store)
    alias.updateToken("late")
}
```

The alias makes the transfer unsafe. The callee may use the store in another isolation region while the caller continues mutating the same object.

Result:

```swift
func makeDetachedStore() -> sending DraftTokenStore {
    DraftTokenStore()
}
```

This is the right shape for a `sending` result: return a new value that is not still connected to the callee's mutable state. Returning a stored non-`Sendable` singleton or actor-isolated property as `sending` is the wrong shape.

---

## 7. Production guidance

Use `Sendable` in production when:

```text
A value crosses task, actor, or global-actor boundaries.
A public async API accepts generic values that may be used concurrently.
A closure may escape into concurrent execution.
A type is intended to be safely used from multiple concurrency domains.
A parameter or result should require a one-time transfer, in which case use sending at that boundary.
```

Be careful when:

```text
A class has mutable state.
A protocol is marked Sendable and now every conformer must uphold that promise.
A closure captures self.
A dependency is imported from Objective-C or older Swift without concurrency annotations.
A public type gets Sendable conformance as part of its API contract.
Using sending with reference types that may have aliases in the caller.
Returning a sending result that is still stored in actor-isolated or shared state.
```

Avoid when:

```text
You add @unchecked Sendable just to silence Swift 6 errors.
You make UI objects Sendable instead of isolating them to @MainActor.
You add T: Sendable to generic APIs that never cross concurrency boundaries.
You expose mutable reference storage from a Sendable type.
You use sending as a substitute for a real ownership model.
You assume sending means move-only or generally Sendable.
```

Debugging checklist:

```text
What value is crossing the concurrency boundary?
Is it a value type, actor, immutable class, mutable class, closure, or generic?
Does the closure need @Sendable?
What does the closure capture?
Is self captured?
Is the captured object mutable?
Should the state be actor-isolated instead?
Could this be modeled as an immutable value?
Is @unchecked Sendable backed by a real lock or synchronization invariant?
Is the type public, and does Sendable become part of the library contract?
Is this a type-level Sendable problem, a @Sendable closure capture problem, or a sending transfer problem?
If a sending value is non-Sendable, are there aliases that can still be used after transfer?
If a function returns sending T, is the result newly created or otherwise disconnected from callee state?
Does nonsending behavior explain why a value is not crossing a boundary here?
```

For Swift 6 migrations, do not treat diagnostics as noise. Swift’s migration guide notes that complete concurrency checking can surface latent issues even in Swift 5 code that does not directly use concurrency features, and Swift 6 language mode can turn some of those into errors. ([Swift.org, "Common Compiler Errors"](https://swift.org/migration/documentation/swift-6-concurrency-migration-guide/commonproblems/))

---

## 8. Senior vs staff-level signal

### Basic answer

> `Sendable` means a type is safe to use with concurrency. `@Sendable` means a closure can be used in concurrent code. `sending` means a value is sent somewhere.

This is not wrong, but it is too vague.

### Senior answer

> `Sendable` is a marker protocol for values that can safely cross concurrency domains. Structs and enums can usually be checked structurally. Actors are implicitly sendable. A final immutable class can be checked as sendable, but mutable classes usually need actor isolation or carefully synchronized `@unchecked Sendable`. A `@Sendable` closure must not capture non-sendable or mutable shared state. `sending T` is a boundary-specific transfer of a value; it does not make `T` generally `Sendable`.

This is a solid interview answer.

### Staff-level answer

> I treat sendability as an API boundary and ownership design question, not as an annotation problem. For new code, I prefer immutable value models and actors for shared mutable state. I use `@Sendable` on callbacks that may execute concurrently or escape into tasks. I add `T: Sendable` to generic APIs only where the value actually crosses isolation. I use `sending` only when the API really transfers a disconnected value region at a parameter or result boundary. For legacy mutable reference types, I either isolate them behind an actor, redesign them into values, or use `@unchecked Sendable` only when there is a documented synchronization invariant. During Swift 6 migration, I do not silence diagnostics until I understand which value is crossing which isolation boundary.

Staff-level questions to ask:

```text
Is this type meant to be shared, transferred, or isolated?
Who owns the mutable state?
Can this become a value type instead of a shared reference?
Should this be an actor rather than a locked class?
Is Sendable part of the public API contract?
Does @unchecked Sendable have a documented invariant?
Are we over-constraining generic APIs with Sendable?
Are UI types main-actor-isolated instead of incorrectly made Sendable?
Is sending describing a real transfer, or hiding a shared mutable reference?
Can any alias use the transferred non-Sendable value after the boundary?
```

---

## 9. Interview-ready summary

`Sendable` is Swift's marker protocol for values that can safely cross concurrency domains such as tasks and actors. `@Sendable` is a function-type attribute: a `@Sendable` closure can be transferred into concurrent execution, so Swift checks its captures. `sending T` is narrower than both: it is a value-level transfer at one parameter or result boundary, and it can sometimes allow a non-`Sendable` value to move across isolation when the compiler can prove there are no unsafe aliases. The right fix for a non-sendable mutable service is usually actor isolation or value modeling; `@unchecked Sendable` is only acceptable when you can prove safety with internal synchronization.

---

## 10. Flashcards

Q: What does `Sendable` mean?

A: A type whose values can safely cross concurrency domains.

Q: What does `@Sendable` mean?

A: A function or closure value can safely be transferred across concurrency domains; captured values must satisfy sendability rules.

Q: Why is a mutable class usually not `Sendable`?

A: Class instances have shared identity, so multiple tasks can access the same mutable storage concurrently.

Q: Why can a final immutable class be `Sendable`?

A: `final` prevents unsafe subclass storage, and immutable `let` properties of `Sendable` types cannot be concurrently mutated.

Q: Are actors `Sendable`?

A: Yes. Actor references are sendable because access to mutable actor state is protected by actor isolation.

Q: Does `let object = SomeClass()` make the object immutable?

A: No. It makes the reference immutable, not the object’s internal state.

Q: What is the danger of `@unchecked Sendable`?

A: The compiler stops checking the conformance, so the author must prove synchronization and safety manually.

Q: Should every async generic API require `T: Sendable`?

A: No. Add `T: Sendable` only when values actually cross concurrency domains or are captured by sendable concurrent work.

Q: What does `sending T` mean?

A: This argument or result value is transferred at a specific isolation boundary; the type `T` has not necessarily become generally `Sendable`.

Q: Does `sending` mean move-only?

A: No. `sending` is about safe isolation-region transfer, not noncopyable ownership.

Q: Why can `sending` work with some non-`Sendable` values?

A: A non-`Sendable` value can be safe to transfer if it is disconnected from aliases that could keep using it concurrently.

Q: What does `nonsending` mean?

A: Arguments and results are not sent across an isolation boundary by default; they stay in the caller's isolation context.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D4 — Actors and actor isolation]]
- [[D5 — Global actors and `@MainActor`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D8 — Isolation Control - `nonisolated`, `isolated` Parameters, and `assumeIsolated`]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D13 — Swift 6 strict concurrency migration]]
- [[B7 — Synthesized conformances and semantic correctness]]
- [[C1 — ARC fundamentals]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — D6 — `Sendable`, `@Sendable`, and `sending`.
- Swift.org. "Concurrency." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- GitHub. "Sendable and @Sendable Closures." Swift Evolution SE-0302. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md
- GitHub. "Transferring Parameters and Results." Swift Evolution SE-0430. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md
- Swift.org. "Captures in a @Sendable Closure." Swift Compiler Diagnostics. https://docs.swift.org/compiler/documentation/diagnostics/sendable-closure-captures/
- Swift.org. "Common Compiler Errors." Swift 6 Concurrency Migration Guide. https://swift.org/migration/documentation/swift-6-concurrency-migration-guide/commonproblems/
- Swift.org. "Announcing Swift 6." Swift.org Blog. https://swift.org/blog/announcing-swift-6/
