---
tags:
  - swift
  - ios
  - interview-prep
  - concurrency
  - actors
  - actor-reentrancy
  - logic-races
---
## 0. Rubric snapshot

**Rubric expectation**

Understand that actor methods can interleave at suspension points, and that actor reentrancy is a central design concern for correctness.

**Caveats**

Actors prevent low-level data races on actor-isolated mutable state, but they do **not** make a whole async method atomic. Swift Evolution SE-0306 states that actor-isolated functions are reentrant: when such a function suspends, other work may execute on the same actor before the original function resumes. It also explicitly warns that actor state can change across an `await`. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

**You should be able to answer**

- What does actor reentrancy mean in practice?
- How can an `await` in the middle of an actor method invalidate assumptions made earlier in the method?

**You should be able to do**

- Debug an actor method that checks a cache, awaits network work, then writes stale data.
- Predict whether the `Counter.incrementIfZero()` code probe can produce `2`.

---

## 1. Core mental model

An actor gives you **serialized access to actor-isolated state while code is actively executing on the actor**. It does not guarantee that an entire `async` method runs as one uninterrupted transaction.

The important boundary is `await`.

Before an `await`, your actor method has exclusive access to its isolated state. At the `await`, the method may suspend. While it is suspended, the actor is free to process other queued work. That other work can mutate the same actor-isolated state. When the original method resumes, all assumptions it made before the `await` may be stale.

Swift Evolution describes this as **interleaving**: no two pieces of actor-isolated code execute at the exact same time on the same actor, but different partial executions may interleave at suspension points. This preserves the “single-threaded illusion” at the low level while still allowing high-level logic races. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

The key idea:

```text
Actor isolation protects memory safety.
Actor reentrancy can still break logical invariants across await.
```

A good senior-level shorthand:

```text
Inside an actor:
- synchronous region before await = critical section
- await = critical section ends
- code after await = must revalidate assumptions
```

Apple’s WWDC actor guidance uses the cache/download example: checking a cache, awaiting a download, and then writing into the cache can be wrong because another task may have populated the cache while the first task was suspended. Apple’s suggested principle is to check assumptions after `await` and keep actor-state mutation in synchronous actor code where possible. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10133/?utm_source=chatgpt.com "Protect mutable state with Swift actors - WWDC21 - Videos"))

---

## 2. Essential mechanics

### 2.1 Actor-isolated synchronous code runs uninterrupted

```swift
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }

    func resetAndIncrement() {
        value = 0
        increment()
    }
}
```

`resetAndIncrement()` contains no suspension point. While it is executing on the actor, another task cannot interleave and mutate `value` halfway through this synchronous sequence.

This is the safe mental model:

```text
No await inside actor-isolated method body
→ no reentrancy inside that method body
→ local actor-state reasoning is sequential
```

### 2.2 `await` splits the method into separate actor turns

```swift
actor TokenStore {
    private var token: String?

    func refreshIfNeeded() async throws -> String {
        if let token {
            return token
        }

        let newToken = try await fetchTokenFromNetwork()

        token = newToken
        return newToken
    }

    private func fetchTokenFromNetwork() async throws -> String {
        "fresh-token"
    }
}
```

The method looks sequential, but actor execution is effectively split:

```text
Turn 1:
- check token
- observe nil
- start await fetchTokenFromNetwork()
- suspend

Other actor work may run here.

Turn 2:
- resume
- write token
- return
```

If two tasks call `refreshIfNeeded()` concurrently, both can observe `token == nil`, both can fetch, and the later resumed task can overwrite the earlier result.

No memory race occurs. The bug is a **logic race**.

### 2.3 Reentrancy is intentional, not an accident

Non-reentrant actors sound simpler, but they can cause deadlocks and unnecessary blocking. SE-0306 explains that reentrancy avoids deadlocks where actors wait on each other, improves scheduling, and prevents one suspended call from blocking unrelated actor work. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))

So the answer is not “Swift actors are broken.” The answer is:

```text
Actors are safe for isolated mutable state.
You still design async actor methods around suspension points.
```

---

## 3. Common traps and misconceptions

### Trap 1: “Actors make async methods atomic”

Bad:

```swift
actor ImageCache {
    private var cache: [URL: Data] = [:]

    func data(for url: URL) async throws -> Data {
        if let cached = cache[url] {
            return cached
        }

        let downloaded = try await download(url)

        cache[url] = downloaded
        return downloaded
    }
}
```

Problem:

```text
The cache check happened before await.
The cache write happened after await.
Another task may have filled cache[url] during the await.
```

Better:

```swift
actor ImageCache {
    private var cache: [URL: Data] = [:]

    func data(for url: URL) async throws -> Data {
        if let cached = cache[url] {
            return cached
        }

        let downloaded = try await download(url)

        if let cached = cache[url] {
            return cached
        }

        cache[url] = downloaded
        return downloaded
    }
}
```

This is the minimal “revalidate after await” fix.

### Trap 2: “No data race means no race”

Actors prevent simultaneous unsynchronized mutation of actor-isolated state. They do **not** prevent stale reads, duplicated work, check-then-act bugs, or overwrites caused by stale assumptions.

```text
Data race:
Two threads/tasks access the same memory concurrently, at least one writes.

Logic race:
The program is memory-safe, but the result depends on timing/interleaving.
```

Actor reentrancy bugs are usually logic races.

### Trap 3: “Just move the await outside the actor”

Sometimes moving work outside the actor helps, but doing it blindly can make the invariant worse.

Bad instinct:

```swift
let downloaded = try await download(url)
await cache.store(downloaded, for: url)
```

This removes the suspension from the actor, but it does not solve duplicate downloads or stale writes by itself. You still need an explicit policy:

```text
- first writer wins
- last writer wins
- reuse in-flight work
- version-check before write
- invalidate old results
```

### Trap 4: “Use Task.detached to avoid actor blocking”

`Task.detached` loses task hierarchy, actor context, priority inheritance, and task-local assumptions. It is usually the wrong fix for actor reentrancy. Reentrancy is a design problem around invariants, not a reason to escape structured concurrency.

---

## 4. Direct answers to rubric questions

### Q1. What does actor reentrancy mean in practice?

Actor reentrancy means an actor-isolated async method can suspend at an `await`, and while it is suspended, the same actor can run other queued actor-isolated work. When the suspended method resumes, actor state may no longer match what it observed before suspension.

Interview version:

> Actor reentrancy means an actor does not lock itself for the full duration of an async method. It serializes access only while code is actively running on the actor. At an `await`, the method can suspend, and another task can enter the actor and mutate its state. So actors prevent data races, but they do not make async methods transactional.

### Q2. How can an `await` in the middle of an actor method invalidate assumptions made earlier in the method?

Any state read before `await` may be stale after `await`.

```swift
actor Cart {
    private var items: [String] = []

    func checkout() async throws {
        guard !items.isEmpty else { return }

        try await authorizePayment()

        // Wrong assumption:
        // items may have changed while authorizePayment() was suspended.
        items.removeAll()
    }
}
```

The method assumes that the cart after authorization is the same cart that was checked before authorization. That assumption is not guaranteed.

Interview version:

> `await` is a boundary where the actor can process other messages. If I check a condition before `await`, I either need to commit the relevant state before suspending, snapshot the state, or revalidate after the await. Otherwise, I have a classic check-then-act race, just in actor form.

---

## 5. Code probe

Given:

```swift
actor Counter {
    var value = 0

    func incrementIfZero() async {
        if value == 0 {
            try? await Task.sleep(nanoseconds: 10_000_000)
            value += 1
        }
    }
}
```

### What happens?

As written, this code only declares an actor and method.

```text
No output.
No compiler error.
```

If two tasks call `incrementIfZero()` concurrently, then yes, `value` can become `2`.

Example driver:

```swift
let counter = Counter()

async let first: Void = counter.incrementIfZero()
async let second: Void = counter.incrementIfZero()

await first
await second

print(await counter.value)
```

Possible output:

```text
2
```

But the final value is timing-dependent. It can also be:

```text
1
```

### Why?

The interleaving that produces `2`:

```text
Initial:
value = 0

Task A enters incrementIfZero()
A checks value == 0       ✅
A reaches await sleep
A suspends

Task B enters incrementIfZero()
B checks value == 0       ✅
B reaches await sleep
B suspends

A resumes
A executes value += 1
value = 1

B resumes
B executes value += 1
value = 2
```

The bug is here:

```swift
if value == 0 {
    try? await Task.sleep(nanoseconds: 10_000_000)
    value += 1
}
```

The condition `value == 0` was checked before suspension. The mutation happens after suspension. The method assumes the condition is still true after `await`, but actor reentrancy makes that assumption invalid.

### Fix or redesign

#### Fix 1: Revalidate after suspension

Use this when the semantics are “increment only if the value is still zero after the async work.”

```swift
actor Counter {
    var value = 0

    func incrementIfZero() async {
        guard value == 0 else { return }

        try? await Task.sleep(nanoseconds: 10_000_000)

        guard value == 0 else { return }

        value += 1
    }
}
```

Now the interleaving becomes safe:

```text
A checks 0, suspends.
B checks 0, suspends.
A resumes, checks 0, increments to 1.
B resumes, checks again, sees 1, returns.
```

Final value:

```text
1
```

#### Fix 2: Commit state before suspension

Use this when the semantics are “claim the right to increment before doing async work.”

```swift
actor Counter {
    var value = 0

    func incrementIfZero() async {
        guard value == 0 else { return }

        value = 1

        try? await Task.sleep(nanoseconds: 10_000_000)
    }
}
```

This turns the check-and-mutation into one synchronous actor turn.

### Why this fix is correct

Both fixes make the invariant explicit.

The invariant is:

```text
value must only transition from 0 to 1 once.
```

The original code splits the invariant across an `await`:

```text
check value == 0
await
mutate value
```

That is unsafe.

The corrected versions either:

```text
check → await → recheck → mutate
```

or:

```text
check → mutate → await
```

Both avoid relying on a pre-`await` fact after suspension.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Revalidate after `await`|Async work result should only apply if state is still compatible|May waste duplicated async work|
|Commit/reserve before `await`|You can represent “in progress” synchronously|Requires a state model beyond simple values|
|Store in-flight `Task`|Multiple callers should share one async operation|More complex cancellation/error handling|
|Version/check token|You need stale-result suppression|Requires careful version ownership|
|Separate pure async work from actor mutation|Network/CPU work should not be actor-isolated|Still requires a safe write policy|

---

## 6. Exercise

### Problem

Debug an actor method that checks a cache, awaits network work, then writes stale data.

### Bad / naive version

```swift
import Foundation

actor DataCache {
    private var cache: [URL: Data] = [:]

    func data(for url: URL) async throws -> Data {
        if let cached = cache[url] {
            return cached
        }

        let downloaded = try await download(url)

        cache[url] = downloaded
        return downloaded
    }

    private func download(_ url: URL) async throws -> Data {
        try await URLSession.shared.data(from: url).0
    }
}
```

### What is wrong?

```text
1. Task A checks cache[url] and sees nil.
2. Task A starts download and suspends.
3. Task B checks cache[url] and also sees nil.
4. Task B starts another download and suspends.
5. Task A resumes and writes cache[url].
6. Task B resumes later and overwrites cache[url].
```

This code is memory-safe but logically unstable.

The stale assumption:

```text
"cache[url] was nil before await, therefore I should write after await."
```

That does not hold.

### Improved version: first cached result wins

```swift
import Foundation

actor DataCache {
    private var cache: [URL: Data] = [:]

    func data(for url: URL) async throws -> Data {
        if let cached = cache[url] {
            return cached
        }

        let downloaded = try await download(url)

        if let cached = cache[url] {
            return cached
        }

        cache[url] = downloaded
        return downloaded
    }

    private func download(_ url: URL) async throws -> Data {
        try await URLSession.shared.data(from: url).0
    }
}
```

### Why this is better

After the `await`, it rechecks the actor state before writing. That makes the post-suspension decision depend on current actor state, not stale pre-suspension state.

However, it still allows duplicate downloads.

### Improved version: single-flight cache

Use this when duplicate downloads are wasteful or semantically wrong.

```swift
import Foundation

actor DataCache {
    private enum Entry {
        case ready(Data)
        case inFlight(Task<Data, Error>)
    }

    private var entries: [URL: Entry] = [:]

    func data(
        for url: URL,
        fetch: @Sendable @escaping (URL) async throws -> Data
    ) async throws -> Data {
        if let entry = entries[url] {
            switch entry {
            case .ready(let data):
                return data

            case .inFlight(let task):
                return try await task.value
            }
        }

        let task = Task {
            try await fetch(url)
        }

        entries[url] = .inFlight(task)

        do {
            let data = try await task.value

            if case .ready(let cached)? = entries[url] {
                return cached
            }

            entries[url] = .ready(data)
            return data
        } catch {
            entries[url] = nil
            throw error
        }
    }
}
```

Call site:

```swift
let cache = DataCache()
let url = URL(string: "https://example.com/image.png")!

let data = try await cache.data(for: url) { url in
    try await URLSession.shared.data(from: url).0
}
```

### Why this is better

This models the real state machine:

```text
missing → inFlight(task) → ready(data)
```

That prevents impossible or ambiguous states. Multiple callers for the same URL share the same in-flight task instead of starting duplicate downloads.

Production caveat: cancellation policy needs thought. If one caller is cancelled, should the shared download continue for other callers or be cancelled when nobody is waiting? That is a product/API decision, not just a syntax detail.

---

## 7. Production guidance

Use actor reentrancy awareness in production when:

```text
- actor methods contain await between reading and writing actor state
- caches perform check → fetch → write
- token refresh logic performs check → refresh → store
- UI models suppress stale search results
- download managers deduplicate in-flight work
- payment/order/checkout flows depend on state remaining stable
```

Be careful when:

```text
- an actor method has more than one await
- state is read before await and used after await
- you use "if missing, fetch, then store" patterns
- multiple callers may request the same resource concurrently
- cancellation can leave "in progress" state behind
- actor methods mutate several related properties across suspension points
```

Avoid:

```text
- assuming actors make async methods atomic
- storing partial invariants across await without a state machine
- fixing reentrancy by sprinkling Task.detached
- hiding reentrancy bugs behind @MainActor
- using @unchecked Sendable to silence a logic race
```

Debugging checklist:

```text
1. Where are the await points inside the actor method?
2. Which actor-isolated values are read before each await?
3. Which of those values are used after the await?
4. Can another actor method mutate those values while this method is suspended?
5. Should the code revalidate, reserve state, snapshot, version-check, or share in-flight work?
6. What is the intended conflict policy: first writer wins, last writer wins, latest request wins, or single-flight?
7. Does cancellation clean up in-progress state?
8. Do tests force interleaving, or do they merely hope timing exposes the bug?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Actors prevent race conditions because they serialize access to state.

This is incomplete and dangerous.

### Senior answer

> Actors prevent low-level data races on isolated state, but actor methods are reentrant at suspension points. If I read state before an `await`, I cannot assume it is still true afterward. I need to revalidate, snapshot, or update state synchronously before suspending.

### Staff-level answer

> Actor reentrancy is an API design concern. I avoid encoding long-lived invariants across suspension points. For cache/token/download flows, I usually model explicit states like missing, in-flight, ready, failed, or stale. I also define conflict and cancellation policy. Strict concurrency can catch unsafe cross-isolation transfers, but it will not prove that my actor’s business logic is transactionally correct.

Staff-level questions to ask:

```text
What invariant is expected to survive across await?
Should this operation be idempotent?
Should concurrent callers share in-flight work?
What happens if the first caller is cancelled?
Should stale results be discarded, merged, or allowed to overwrite?
Can the actor state machine make invalid states unrepresentable?
Do tests deterministically force the dangerous interleaving?
```

---

## 9. Interview-ready summary

Actor reentrancy means an actor-isolated async method may suspend at `await`, allowing other work to run on the same actor before the original method resumes. Actors still prevent low-level data races because isolated state is not accessed concurrently, but they do not make async methods atomic. Any actor state read before `await` may be stale after `await`, so production actor code must revalidate assumptions, commit state before suspension, snapshot intentionally, or model in-flight operations explicitly. The classic bug is a cache method that checks for a missing value, awaits a download, then overwrites a value that another task already cached.

---

## 10. Flashcards

Q: What does actor reentrancy mean?

A: An actor-isolated async method can suspend at `await`, and while suspended, other work can execute on the same actor before the original method resumes.

Q: Do actors make async methods atomic?

A: No. Actors serialize access while code is executing on the actor, but `await` breaks the method into separate execution turns.

Q: What kind of bug does actor reentrancy commonly cause?

A: Logic races: stale reads, duplicated work, check-then-act bugs, and stale result overwrites.

Q: What is the safest rule for actor state across `await`?

A: Do not rely on actor state observed before `await` unless you revalidate it, snapshot it intentionally, or commit/reserve state before suspending.

Q: Can two concurrent calls to `incrementIfZero()` make the value `2`?

A: Yes. Both calls can check `value == 0`, suspend, then resume and increment.

Q: What is the difference between a data race and an actor reentrancy bug?

A: A data race is unsafe concurrent memory access. An actor reentrancy bug is memory-safe but logically wrong because state changes between suspension and resumption.

Q: What is a single-flight pattern?

A: A design where concurrent callers share one in-flight async operation instead of starting duplicate work.

Q: Does Swift 6 strict concurrency catch actor reentrancy logic races?

A: Not generally. Strict concurrency catches many data-race and isolation violations, but high-level stale-state logic remains a design and testing responsibility.

---

## 11. Related sections

- [[D1 — Structured concurrency fundamentals]]
- [[D2 — Task hierarchy, cancellation, and priorities]]
- [[D3 — `async let`, task groups, and choosing the right concurrency primitive]]
- [[D4 — Actors and actor isolation]]
- [[D5 — Global actors and `@MainActor`]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D8 — Isolation control: `nonisolated`, isolated parameters, and `assumeIsolated`]]
- [[D10 — `AsyncSequence`, streams, and backpressure-aware design]]
- [[D11 — Detached tasks, task inheritance, and isolation loss]]
- [[F1 — Testing async and concurrent code]]

---

## 12. Sources

- Swift Senior/Staff Rubric: D7 Actor reentrancy and logic races.
- Swift Evolution SE-0306 — Actors: actor-isolated functions are reentrant, interleaving can occur at suspension points, and actor state can change across `await`. ([GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md "swift-evolution/proposals/0306-actors.md at main · swiftlang/swift-evolution · GitHub"))
- Apple WWDC21 — “Protect mutable state with Swift actors”: cache/download example, rechecking assumptions after `await`, and designing actor-state mutation carefully. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10133/?utm_source=chatgpt.com "Protect mutable state with Swift actors - WWDC21 - Videos"))
- The Swift Programming Language — Concurrency: asynchronous code can suspend and resume later. ([docs.swift.org](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/?utm_source=chatgpt.com "Concurrency - Documentation | Swift.org"))