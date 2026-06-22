---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actor-isolation
  - swift6
---
## 0. Rubric snapshot

**Rubric expectation**

Understand when behavior should or should not be isolated, and how to express that safely.

**Caveats**

Removing isolation just to satisfy the compiler can reintroduce races or make the concurrency model misleading. D8 specifically asks you to reason about `nonisolated`, `isolated` parameters, and `assumeIsolated`.

**You should be able to answer**

- What does `nonisolated` mean on a member of an actor or globally isolated type?
- When is `assumeIsolated` appropriate, and what makes it dangerous?

**You should be able to do**

- Given an actor API, identify which helpers can be `nonisolated` because they do not touch isolated state.

---

## 1. Core mental model

Actor isolation is Swift’s way of saying:

> This mutable state may only be accessed while executing in the actor’s isolation domain.

For an `actor`, instance methods and mutable stored properties are actor-isolated by default. That means external callers must cross the actor boundary with `await`, and Swift serializes access to the actor-isolated state.

`nonisolated` does the opposite: it explicitly says a declaration is **not protected by that actor/global actor isolation**. A `nonisolated` member behaves like ordinary code outside the actor. Because it can be called from anywhere synchronously, it cannot touch isolated mutable state. Swift’s own documentation describes nonisolated actor members as executing like code outside the actor, without access to isolated state. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations - Documentation | Swift.org"))

`isolated` parameters let a free function, closure, or helper temporarily run inside a specific actor’s isolation domain. This is useful when you want to batch multiple operations on the same actor without repeated `await`s. SE-0313 introduced this as part of improved control over actor isolation, generalizing actor isolation beyond only `self`. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md "swift-evolution/proposals/0313-actor-isolation-control.md at main · swiftlang/swift-evolution · GitHub"))

`assumeIsolated` is the escape hatch. It says: “I know this synchronous code is already running on this actor’s executor; let me access isolated state synchronously.” Apple documents it as assuming and verifying that the current synchronous execution is on the actor’s serial executor. If that assumption is false, execution stops. ([Apple Developer](https://developer.apple.com/documentation/swift/actor/assumeisolated%28_%3Afile%3Aline%3A%29?utm_source=chatgpt.com "assumeIsolated(_:file:line:) | Apple Developer Documentation"))

The key idea:

```text
isolated     = protected by an actor/global actor executor
nonisolated  = outside that protection; cannot touch isolated state
isolated T   = this parameter gives synchronous access to that actor's isolated state
assumeIsolated = runtime-checked escape hatch for legacy synchronous boundaries
```

Swift guarantees data-race prevention around actor-isolated state. It does **not** guarantee your business logic is correct, and it does not make `nonisolated` code safe if you smuggle mutable shared state through it.

---

## 2. Essential mechanics

### 2.1 `nonisolated` on actors

A `nonisolated` actor member can be called synchronously without `await`, but it cannot access actor-isolated state.

```swift
actor ImageCache {
    let namespace: String
    private var storage: [String: Int] = [:]

    init(namespace: String) {
        self.namespace = namespace
    }

    nonisolated var displayName: String {
        "cache:\(namespace)"
    }

    func store(_ value: Int, for key: String) {
        storage[key] = value
    }

    func value(for key: String) -> Int? {
        storage[key]
    }
}

let cache = ImageCache(namespace: "avatars")

// No await needed:
print(cache.displayName)

// Await needed:
await cache.store(1, for: "user-1")
```

`displayName` is safe as `nonisolated` because it only depends on immutable state. `store` and `value(for:)` must remain isolated because they access `storage`.

Bad:

```swift
actor ImageCache {
    private var storage: [String: Int] = [:]

    nonisolated var count: Int {
        storage.count
    }
}
```

Swift 6.2 compiler error:

```text
error: actor-isolated property 'storage' can not be referenced from a nonisolated context
```

That error is the point. `count` would be callable from anywhere synchronously, so reading isolated mutable state there would break actor isolation.

---

### 2.2 `nonisolated` on globally isolated types

The same idea applies to globally isolated types:

```swift
import Foundation

@MainActor
final class ProfileViewModel {
    private var name = "Aykut"

    nonisolated static func normalize(_ raw: String) -> String {
        raw.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
    }

    func updateName(_ raw: String) {
        name = Self.normalize(raw)
    }
}
```

`ProfileViewModel` is `@MainActor`, but `normalize(_:)` does not need UI state or main-actor state. Keeping it `nonisolated` avoids forcing pure string processing onto the main actor.

This matters more in Swift 6.2-era projects because default actor isolation settings can make more code implicitly main-actor isolated. SE-0449 expanded `nonisolated` as a way to cut off global actor inference in more places, implemented in Swift 6.1. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md?utm_source=chatgpt.com "swift-evolution/proposals/0449-nonisolated-for-global-actor ..."))

Use this carefully. Marking a helper `nonisolated` is good when the helper is genuinely pure or uses only safe immutable data. It is bad when you are trying to silence a diagnostic without fixing the underlying isolation model.

---

### 2.3 `isolated` parameters

An `isolated` actor parameter lets a helper execute in that actor’s isolation domain.

```swift
actor Ledger {
    private var balance = 0

    func deposit(_ amount: Int) {
        balance += amount
    }

    func currentBalance() -> Int {
        balance
    }
}

func applyBonus(to ledger: isolated Ledger) {
    ledger.deposit(10)
}
```

Calling it from outside the actor still requires `await`:

```swift
let ledger = Ledger()

await applyBonus(to: ledger)
print(await ledger.currentBalance())
```

Output:

```text
10
```

Inside `applyBonus(to:)`, `ledger` is isolated, so you can call actor-isolated synchronous members without additional `await`.

This is useful for batching:

```swift
func applyMonthlyInterest(to ledger: isolated Ledger) {
    ledger.deposit(10)
    ledger.deposit(20)
    ledger.deposit(30)
}
```

Without `isolated`, each cross-actor operation would need to be async and could introduce more suspension/interleaving points.

Important limitation: a function can have only one `isolated` parameter. Swift cannot generally execute on two different actor executors at once. SE-0313 explicitly discusses this restriction. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md "swift-evolution/proposals/0313-actor-isolation-control.md at main · swiftlang/swift-evolution · GitHub"))

---

### 2.4 `assumeIsolated`

`assumeIsolated` is for synchronous code that is already dynamically executing on an actor’s executor, but where the compiler cannot prove that statically.

Example that works:

```swift
actor Store {
    private var value = 0

    func runLegacyCallbackWhileOnExecutor() {
        legacySynchronousCallback()
    }

    nonisolated func legacySynchronousCallback() {
        assumeIsolated { isolatedSelf in
            isolatedSelf.value += 1
        }
    }

    func get() -> Int {
        value
    }
}

let store = Store()
await store.runLegacyCallbackWhileOnExecutor()
print(await store.get())
```

Output:

```text
1
```

This works because `legacySynchronousCallback()` is called while already executing on `Store`’s executor.

Bad:

```swift
let store = Store()
store.legacySynchronousCallback()
```

Runtime result:

```text
Incorrect actor executor assumption; expected 'Store' executor.
```

The danger is exactly in the name: you are making an assumption. If that assumption is wrong, your program traps. If you overuse it, you are fighting Swift’s isolation model instead of designing with it.

---

## 3. Common traps and misconceptions

### Trap 1: “`nonisolated` means thread-safe”

No. `nonisolated` means **not isolated to this actor**.

Bad:

```swift
actor Cache {
    private var storage: [String: Int] = [:]

    nonisolated func unsafeCount() -> Int {
        storage.count
    }
}
```

Better:

```swift
actor Cache {
    private var storage: [String: Int] = [:]

    func count() -> Int {
        storage.count
    }
}
```

If the value depends on actor-isolated mutable state, keep it isolated.

---

### Trap 2: Marking helpers `@MainActor` because they are used by UI

This is unnecessary isolation:

```swift
@MainActor
func normalizedSearchQuery(_ raw: String) -> String {
    raw.lowercased().trimmingCharacters(in: .whitespacesAndNewlines)
}
```

Better:

```swift
nonisolated func normalizedSearchQuery(_ raw: String) -> String {
    raw.lowercased().trimmingCharacters(in: .whitespacesAndNewlines)
}
```

A pure helper should not require the main actor merely because a view model calls it.

---

### Trap 3: Using `assumeIsolated` as a compiler silencer

Bad smell:

```swift
nonisolated func updateFromLegacyCallback() {
    MainActor.assumeIsolated {
        // mutate UI state
    }
}
```

This is only correct if the callback is **guaranteed** to run on the main actor’s executor. A legacy API saying “called on main thread” is not always the same as Swift’s actor executor model, especially around older Objective-C and callback-based APIs.

Better default:

```swift
func updateFromCallback() {
    Task { @MainActor in
        // mutate UI state
    }
}
```

Use `assumeIsolated` only when you have a narrow, audited synchronous boundary and can prove the executor assumption.

---

### Trap 4: Making protocol conformance work by removing isolation incorrectly

Example:

```swift
protocol Displayable {
    var displayName: String { get }
}

actor UserStore: Displayable {
    private var users: [String] = []

    var displayName: String {
        "Users: \(users.count)"
    }
}
```

This fails because a synchronous protocol requirement cannot safely read actor-isolated mutable state.

Bad “fix”:

```swift
nonisolated var displayName: String {
    "Users: \(users.count)" // illegal
}
```

Correct options:

```swift
protocol AsyncDisplayable {
    var displayName: String { get async }
}
```

or use immutable nonisolated data:

```swift
actor UserStore: Displayable {
    nonisolated let displayName: String

    private var users: [String] = []

    init(displayName: String) {
        self.displayName = displayName
    }
}
```

The design decision is semantic: does the property represent stable identity/metadata, or live isolated state?

---

## 4. Direct answers to rubric questions

### Q1. What does `nonisolated` mean on a member of an actor or globally isolated type?

`nonisolated` means the declaration is not part of the actor/global actor isolation domain. It can be called without `await`, but it cannot access isolated mutable state.

For an actor:

```swift
actor SessionStore {
    nonisolated let namespace: String
    private var token: String?

    init(namespace: String) {
        self.namespace = namespace
    }

    nonisolated var debugName: String {
        "SessionStore(\(namespace))"
    }

    func updateToken(_ token: String) {
        self.token = token
    }
}
```

`debugName` is safe as `nonisolated`. `updateToken(_:)` is not, because it mutates isolated state.

For a globally isolated type:

```swift
@MainActor
final class SearchViewModel {
    private var query = ""

    nonisolated static func sanitize(_ raw: String) -> String {
        raw.trimmingCharacters(in: .whitespacesAndNewlines)
    }

    func updateQuery(_ raw: String) {
        query = Self.sanitize(raw)
    }
}
```

`sanitize(_:)` does not need main-actor isolation.

Interview version:

> `nonisolated` removes a declaration from the surrounding actor or global actor isolation. That makes it synchronously callable from any context, so Swift prevents it from touching isolated state. I use it for pure helpers, immutable metadata, stable identity, or synchronous protocol requirements that can be satisfied without actor state. I avoid it when the value depends on mutable actor-isolated state.

---

### Q2. When is `assumeIsolated` appropriate, and what makes it dangerous?

`assumeIsolated` is appropriate at narrow legacy or callback boundaries where execution is already on the correct actor executor, but the compiler cannot prove it.

Appropriate shape:

```swift
actor Parser {
    private var events: [String] = []

    func handleLegacyEventSynchronously(_ event: String) {
        legacyCallback(event)
    }

    nonisolated private func legacyCallback(_ event: String) {
        assumeIsolated { isolatedSelf in
            isolatedSelf.events.append(event)
        }
    }
}
```

Dangerous shape:

```swift
nonisolated func calledFromAnywhere() {
    assumeIsolated { isolatedSelf in
        isolatedSelf.mutateState()
    }
}
```

The second version is dangerous because callers can invoke it from outside the actor executor. Then the assumption is false and the program traps. Apple documents `assumeIsolated` as verifying that the current synchronous function is executing on the actor’s serial executor. ([Apple Developer](https://developer.apple.com/documentation/swift/actor/assumeisolated%28_%3Afile%3Aline%3A%29?utm_source=chatgpt.com "assumeIsolated(_:file:line:) | Apple Developer Documentation"))

Interview version:

> `assumeIsolated` is a runtime-checked bridge for synchronous code that is already executing on an actor executor but cannot express that statically. It is appropriate for small audited interop boundaries. It is dangerous because the compiler stops protecting you at that point; if the executor assumption is wrong, you trap, and if the assumption is casually applied, the codebase loses clear isolation semantics.

---

## 5. Code probe

The rubric has no D8 code probe, so use these minimal probes instead.

### Probe A — valid `nonisolated`

Given:

```swift
actor ImageCache {
    let namespace: String

    init(namespace: String) {
        self.namespace = namespace
    }

    nonisolated var displayName: String {
        "cache:\(namespace)"
    }
}

let cache = ImageCache(namespace: "avatars")
print(cache.displayName)
```

What happens?

```text
cache:avatars
```

Why?

`displayName` only reads immutable actor state. It does not touch isolated mutable state, so Swift allows it to be `nonisolated` and callable without `await`.

---

### Probe B — invalid `nonisolated`

Given:

```swift
actor ImageCache {
    private var storage: [String: Int] = [:]

    nonisolated func count() -> Int {
        storage.count
    }
}
```

Swift 6.2 compiler error:

```text
error: actor-isolated property 'storage' can not be referenced from a nonisolated context
```

Why?

`storage` is actor-isolated mutable state. A `nonisolated` function can be called from any context synchronously, so Swift cannot allow it to read `storage`.

Fix:

```swift
actor ImageCache {
    private var storage: [String: Int] = [:]

    func count() -> Int {
        storage.count
    }
}
```

Call site:

```swift
let count = await cache.count()
```

---

### Probe C — `isolated` parameter

Given:

```swift
actor Ledger {
    private var balance = 0

    func deposit(_ amount: Int) {
        balance += amount
    }

    func currentBalance() -> Int {
        balance
    }
}

func applyBonus(to ledger: isolated Ledger) {
    ledger.deposit(10)
}

let ledger = Ledger()
await applyBonus(to: ledger)
print(await ledger.currentBalance())
```

Output:

```text
10
```

Why?

`applyBonus(to:)` executes with `ledger` isolated. Inside the function, `ledger.deposit(10)` is a synchronous actor-isolated call. The caller still crosses into that actor domain with `await`.

---

### Alternative fixes and tradeoffs

|Option|When appropriate|Tradeoff|
|---|---|---|
|Keep member isolated|Member reads/writes actor mutable state|Requires `await` from outside|
|Mark member `nonisolated`|Pure helper, immutable metadata, stable identity|Cannot access isolated state|
|Use `isolated` parameter|Batch multiple operations on one actor|Function becomes tied to that actor isolation|
|Use `assumeIsolated`|Narrow audited synchronous interop boundary|Runtime trap if assumption is wrong|
|Redesign protocol as async|Requirement depends on isolated state|Protocol becomes async and affects callers|

---

## 6. Exercise

### Problem

Given an actor API, identify which helpers can be `nonisolated` because they do not touch isolated state.

### Bad / naive version

```swift
import Foundation

actor AvatarCache {
    private let namespace: String
    private var images: [String: Data] = [:]

    init(namespace: String) {
        self.namespace = namespace
    }

    func cacheKey(for userID: String, size: Int) -> String {
        "\(namespace):\(userID):\(size)"
    }

    func normalizedUserID(_ raw: String) -> String {
        raw.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
    }

    func store(_ data: Data, userID: String, size: Int) {
        let key = cacheKey(for: userID, size: size)
        images[key] = data
    }

    func image(userID: String, size: Int) -> Data? {
        let key = cacheKey(for: userID, size: size)
        return images[key]
    }

    func count() -> Int {
        images.count
    }
}
```

### What is wrong?

```text
cacheKey(for:size:) depends only on immutable namespace.
normalizedUserID(_:) is pure.
Both are unnecessarily actor-isolated.
store(_:userID:size:), image(userID:size:), and count() touch mutable isolated state and should stay isolated.
```

The naive design serializes pure work through the actor executor. That is not a data race, but it is unnecessary coupling and can make call sites awkward.

### Improved version

```swift
import Foundation

actor AvatarCache {
    nonisolated let namespace: String
    private var images: [String: Data] = [:]

    init(namespace: String) {
        self.namespace = namespace
    }

    nonisolated func cacheKey(for userID: String, size: Int) -> String {
        "\(namespace):\(Self.normalizedUserID(userID)):\(size)"
    }

    nonisolated static func normalizedUserID(_ raw: String) -> String {
        raw.trimmingCharacters(in: .whitespacesAndNewlines).lowercased()
    }

    func store(_ data: Data, userID: String, size: Int) {
        let key = cacheKey(for: userID, size: size)
        images[key] = data
    }

    func image(userID: String, size: Int) -> Data? {
        let key = cacheKey(for: userID, size: size)
        return images[key]
    }

    func count() -> Int {
        images.count
    }
}
```

### Why this is better

`namespace`, `cacheKey(for:size:)`, and `normalizedUserID(_:)` are safe to call from anywhere. They do not need actor serialization.

The mutable dictionary remains protected:

```swift
await cache.store(data, userID: " Aykut ", size: 128)
let image = await cache.image(userID: "aykut", size: 128)
let count = await cache.count()
```

This design communicates the real concurrency boundary:

```text
Pure/key-generation logic: nonisolated
Mutable cache storage: actor-isolated
```

That is the production-level habit: isolate state, not every helper that happens to live near the state.

---

## 7. Production guidance

Use `nonisolated` when:

```text
- The member is pure.
- The member reads only immutable Sendable state.
- The member represents stable identity or metadata.
- A synchronous protocol requirement can be satisfied without isolated state.
- A helper is inside a @MainActor type but does not need UI/main-actor state.
```

Be careful when:

```text
- The member reads reference-typed storage.
- The member returns a non-Sendable value.
- The member exists only to satisfy a protocol.
- The type is globally isolated because of Swift 6.2 default isolation settings.
- You are tempted to add nonisolated just because the compiler asked for await.
```

Avoid when:

```text
- The member reads or writes actor mutable state.
- The result must reflect live actor state.
- The method starts unstructured work to get back into the actor.
- You cannot explain why synchronous cross-isolation access is safe.
```

Use `isolated` parameters when:

```text
- A helper needs to perform multiple synchronous operations on one actor.
- You want to reduce repeated await points.
- The helper semantically belongs to one actor’s isolation domain but is better expressed outside the actor type.
```

Use `assumeIsolated` only when:

```text
- You are at a narrow synchronous legacy boundary.
- You can prove the current execution is already on the actor executor.
- The runtime trap is acceptable if the assumption is violated.
- The call is documented and covered by tests or assertions.
```

Debugging checklist:

```text
1. Is this code isolated, nonisolated, or globally isolated?
2. Does this member touch mutable actor state?
3. Is this await crossing an actor boundary or merely calling async work?
4. Is a protocol forcing a synchronous requirement onto actor state?
5. Is nonisolated being used for clarity or as a diagnostic silencer?
6. Could a pure helper be moved outside the actor instead?
7. Is assumeIsolated backed by a real executor guarantee?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> `nonisolated` lets actor methods be called without `await`.

This is incomplete. It misses the safety restriction and design implications.

### Senior answer

> `nonisolated` removes a member from actor isolation. It can be called synchronously, but it cannot access isolated mutable state. I use it for pure helpers, immutable metadata, and protocol requirements that do not depend on actor state. I keep anything that reads or writes mutable actor state isolated.

### Staff-level answer

> Isolation annotations are API design. I want the isolation boundary to match the ownership boundary of mutable state. I avoid actor-isolating pure helpers because it creates unnecessary serialization and awkward call sites, but I also avoid removing isolation to satisfy the compiler. For legacy synchronous APIs, I prefer redesigning toward async or actor-aware APIs; `assumeIsolated` is only for narrow audited bridges where the executor guarantee is real. When reviewing a Swift 6 migration, I look for accidental `@MainActor` spread, protocol conformance mismatches, and `nonisolated` annotations that are hiding shared mutable state rather than clarifying it.

Staff-level questions to ask:

```text
Is this type isolated because of real mutable state, or because isolation leaked through annotations?
Does this helper belong inside the actor, or should it be a pure free function/static function?
Is this synchronous protocol requirement compatible with actor isolation?
Would an async requirement better express the true dependency on actor state?
Are we using assumeIsolated because of a proven executor guarantee or because migration is painful?
```

---

## 9. Interview-ready summary

`nonisolated` means a declaration is outside the actor or global actor isolation domain, so it is synchronously callable but cannot touch isolated mutable state. `isolated` parameters let a helper run in a specific actor’s isolation domain, which is useful for batching operations without repeated `await`s. `assumeIsolated` is a runtime-checked escape hatch for synchronous legacy boundaries where you can prove you are already on the actor executor. The senior-level judgment is to isolate mutable state, keep pure helpers nonisolated, and never remove isolation just to silence Swift 6 diagnostics.

---

## 10. Flashcards

Q: What does `nonisolated` mean on an actor member?  
A: The member is not actor-isolated. It can be called without `await`, but it cannot access actor-isolated mutable state.

Q: Can a `nonisolated` actor method read a `var` stored property?  
A: No, not if that property is actor-isolated. Swift rejects it because the method can run outside the actor executor.

Q: What is a good use of `nonisolated`?  
A: Pure helpers, immutable metadata, stable identity, or synchronous protocol requirements that do not depend on actor-isolated state.

Q: What does an `isolated` parameter do?  
A: It makes the function execute in that actor parameter’s isolation domain, allowing synchronous access to that actor’s isolated members inside the function.

Q: Why can a function have only one `isolated` parameter?  
A: Swift generally cannot execute on two different actor executors at the same time.

Q: What is `assumeIsolated` for?  
A: A narrow runtime-checked bridge where synchronous code is already executing on the actor executor but the compiler cannot prove it.

Q: What happens if `assumeIsolated` is wrong?  
A: The program stops execution because the executor assumption fails.

Q: Why is adding `nonisolated` to silence compiler errors dangerous?  
A: It may remove the isolation that was protecting mutable state, or make the API imply synchronous safety that does not exist.

---

## 11. Related sections

- [[D4 — Actors and actor isolation]]
- [[D5 — Global actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D7 — Actor reentrancy and logic races]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[D13 — Swift 6 strict concurrency migration]]
- [[D14 — Swift 6.2 approachable concurrency and new defaults]]

---

## 12. Sources

- Swift Senior/Staff Rubric, D8 — Isolation control.
- SE-0313 — Improved control over actor isolation. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0313-actor-isolation-control.md "swift-evolution/proposals/0313-actor-isolation-control.md at main · swiftlang/swift-evolution · GitHub"))
- Swift Book documentation snippet on `nonisolated` actor members. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/?utm_source=chatgpt.com "Declarations - Documentation | Swift.org"))
- Apple documentation for `Actor.assumeIsolated`. ([Apple Developer](https://developer.apple.com/documentation/swift/actor/assumeisolated%28_%3Afile%3Aline%3A%29?utm_source=chatgpt.com "assumeIsolated(_:file:line:) | Apple Developer Documentation"))
- SE-0449 — Allow `nonisolated` to prevent global actor inference. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md?utm_source=chatgpt.com "swift-evolution/proposals/0449-nonisolated-for-global-actor ..."))