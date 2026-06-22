---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - closures
---
## 0. Rubric snapshot

**Rubric expectation**

Know escaping vs nonescaping closures, capture behavior for value and reference types, capture lists, and `@autoclosure`. The rubric specifically calls out that capturing a mutable local boxes storage, and that capturing `self` strongly in long-lived async work frequently leaks.

**Caveats**

- A closure is not just syntax for “inline function.”
- A closure can carry captured state.
- Closures are reference types.
- Storing a closure can extend the lifetime of everything it strongly captures.
- Escaping closures that capture `self` are a major source of retain cycles.

**You should be able to answer**

- What changes when a closure becomes `@escaping`?
- How does `[weak self]` differ from `[unowned self]`, and when is each justified?

**You should be able to do**

- Diagnose a retain cycle involving a view controller, a service callback, and a stored closure.
- Predict the output of the rubric code probe.

---

## 1. Core mental model

A closure is a callable reference that may carry an environment with it. That environment contains the values or variables the closure needs from the scope where it was created. Swift calls this “capturing values”: the closure can refer to constants and variables from the surrounding context, and it can keep those captured values alive after the original scope has ended. The Swift book explicitly describes closures as self-contained blocks that can capture and store references to surrounding constants and variables. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/Closures.md "raw.githubusercontent.com"))

The dangerous part is not the `{ ... }` syntax. The dangerous part is lifetime. A nonescaping closure is guaranteed not to outlive the function call that receives it. An escaping closure may be stored, executed later, passed into async work, or retained by another object. Once a closure escapes, its captures can escape too.

The key idea:

```text
closure = callable code + captured environment + lifetime
```

Closures are reference types. Assigning a closure to two variables does not copy independent closure state; both variables can refer to the same closure and the same captured storage. Swift’s closure documentation shows this with incrementer closures that share their captured running total. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/Closures.md "raw.githubusercontent.com"))

For value types, capture semantics depend on whether the closure captures a value snapshot or mutable storage. A closure that captures a mutable local variable captures storage so later calls observe mutations. The Swift book describes this as capturing a reference to local variables such as `runningTotal`, ensuring the variable remains available after the creating function returns. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/Closures.md "raw.githubusercontent.com"))

For class instances, capturing `self` strongly increments the reference count of the instance. If the instance also stores the closure, you get the classic cycle:

```text
object -> closure property -> closure -> object
```

Swift’s ARC documentation explicitly describes strong reference cycles between a class instance and a closure assigned to one of its properties. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/AutomaticReferenceCounting.md "raw.githubusercontent.com"))

---

## 2. Essential mechanics

### Nonescaping is the default for closure parameters

Function parameters of function type are nonescaping by default. That means the callee can call the closure during the function call, but cannot store it for later.

```swift
func performNow(_ operation: () -> Void) {
    operation()
}

performNow {
    print("Runs before performNow returns")
}
```

This gives the compiler stronger lifetime guarantees. It also means a nonescaping closure can implicitly refer to `self` in places where an escaping closure would require explicit `self`.

### `@escaping` means the closure may outlive the function call

A closure escapes when it is passed into a function but may be called after that function returns. Common causes:

```text
stored in a property
stored in an array
passed to async work
used as a completion handler
returned from a function
```

Example:

```swift
final class Loader {
    private var completion: (() -> Void)?

    func load(completion: @escaping () -> Void) {
        self.completion = completion
    }
}
```

The Swift book describes escaping closures as closures that are passed as arguments but called after the receiving function returns; storing a closure outside the function is the canonical example. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/Closures.md "raw.githubusercontent.com"))

### Escaping closures make `self` capture explicit

```swift
final class ViewModel {
    var title = ""

    func load(completion: @escaping () -> Void) {
        completion()
    }

    func start() {
        load {
            self.title = "Loaded"
        }
    }
}
```

In an escaping closure, Swift requires explicit `self` when capturing a class instance. This is not cosmetic. It forces you to notice that the closure may keep the object alive. Swift’s documentation says explicit `self` expresses intent and reminds you to check for reference cycles. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/Closures.md "raw.githubusercontent.com"))

### Capturing mutable locals captures storage

```swift
func makeCounter() -> () -> Int {
    var count = 0

    return {
        count += 1
        return count
    }
}

let counter = makeCounter()
print(counter()) // 1
print(counter()) // 2
```

`count` does not disappear when `makeCounter()` returns. The returned closure keeps captured storage alive.

This is why saying “closures capture values” is sometimes too imprecise. For mutable locals, the important mental model is:

```text
the closure captures the variable's storage, not just its initial value
```

### Capture lists let you override default capture behavior

Capture lists are written before the closure body:

```swift
final class Controller {
    var onTap: (() -> Void)?

    func bind() {
        onTap = { [weak self] in
            self?.handleTap()
        }
    }

    private func handleTap() {}
}
```

A capture list can capture references as `weak` or `unowned`, or capture a snapshot of a value:

```swift
var count = 0

let printSnapshot = { [count] in
    print(count)
}

count = 10
printSnapshot() // 0
```

Without `[count]`, the closure would read the captured variable storage and print the later value.

### `@autoclosure` delays expression evaluation

`@autoclosure` automatically wraps an expression in a closure:

```swift
func logIfNeeded(_ message: @autoclosure () -> String, enabled: Bool) {
    guard enabled else { return }
    print(message())
}

logIfNeeded(expensiveMessage(), enabled: false)
```

`expensiveMessage()` is not evaluated unless `message()` is called.

Swift’s documentation describes `@autoclosure` as automatically creating a no-argument closure around an expression, allowing delayed evaluation. It also warns that overuse can make code harder to understand because evaluation is deferred invisibly. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/Closures.md "raw.githubusercontent.com"))

Use `@autoclosure` mainly for assertion-like APIs, lazy logging, and APIs where the function name makes deferred evaluation obvious.

---

## 3. Common traps and misconceptions

### Trap 1: “Escaping just means async”

Escaping is about lifetime, not necessarily threading.

Bad:

```swift
final class Store {
    private var callbacks: [() -> Void] = []

    func add(_ callback: () -> Void) {
        callbacks.append(callback)
    }
}
```

This does not compile because `callback` is nonescaping by default.

Better:

```swift
final class Store {
    private var callbacks: [() -> Void] = []

    func add(_ callback: @escaping () -> Void) {
        callbacks.append(callback)
    }
}
```

A stored closure escapes even if it is eventually called synchronously.

### Trap 2: “Use `[weak self]` everywhere”

`[weak self]` is safe but not always semantically correct. It turns `self` into an optional and allows the closure to silently do nothing after the object is gone.

That is usually correct for UI callbacks:

```swift
service.onUpdate = { [weak self] value in
    self?.render(value)
}
```

But it can hide bugs in domain code where the object must exist for the operation to be meaningful.

### Trap 3: “Use `[unowned self]` because it avoids optionals”

`unowned` is not “weak without optional syntax.” It is a promise that the captured object outlives the closure. If that promise is wrong, accessing the reference after deallocation traps.

Use it only when the lifetime relationship is structurally guaranteed.

Good candidate:

```swift
final class HTMLElement {
    let name: String

    lazy var asHTML: () -> String = { [unowned self] in
        "<\(self.name)>"
    }

    init(name: String) {
        self.name = name
    }
}
```

Here, the closure is owned by the instance and is not expected to outlive it. Swift’s ARC documentation uses this kind of example for `unowned` closure captures. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/AutomaticReferenceCounting.md "raw.githubusercontent.com"))

Risky candidate:

```swift
service.onComplete = { [unowned self] result in
    self.render(result)
}
```

If the service outlives the view controller, this can crash.

### Trap 4: “A closure property is like a method”

A method does not usually create a stored object that strongly captures `self`. A closure property can.

```swift
final class Formatter {
    var prefix = "ID"

    lazy var format: (Int) -> String = {
        "\(self.prefix)-\($0)"
    }
}
```

This creates:

```text
Formatter -> format closure -> Formatter
```

Unless you break the cycle, `Formatter` may never deallocate.

### Trap 5: Capturing `self` inside nested escaping closures

This can happen even when the inner closure uses `[weak self]`.

```swift
outerEscapingClosure {
    innerEscapingClosure { [weak self] in
        self?.doSomething()
    }

    self.doSomethingElse()
}
```

The outer closure may still strongly capture `self`. Swift has diagnostics around implicit strong captures of weak capture list items in nested escaping closures because this pattern can unexpectedly extend lifetimes or create cycles. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/implicit-strong-capture/?utm_source=chatgpt.com "Implicit strong captures of weak capture list items ..."))

---

## 4. Direct answers to rubric questions

### Q1. What changes when a closure becomes `@escaping`?

An `@escaping` closure is allowed to outlive the function call that receives it. That means it can be stored, called later, passed into asynchronous work, or otherwise retained beyond the current call stack.

This changes the compiler’s lifetime model:

- the closure may retain captured objects longer;
- captured `self` in class instances must be explicit;
- storing the closure is allowed;
- retain cycles become a design concern;
- in concurrency contexts, `@escaping @Sendable` closures also impose sendability restrictions on captures.

Example:

```swift
var handlers: [() -> Void] = []

func store(_ handler: @escaping () -> Void) {
    handlers.append(handler)
}
```

Without `@escaping`, this is illegal because `handler` would be nonescaping.

Interview version:

> `@escaping` means the closure can survive past the function call that received it. That changes lifetime and ownership: the closure may be stored or run later, so its captures may be retained longer. For class instances, capturing `self` becomes explicit because a strong capture can create a retain cycle. It is not just an async marker; storing a closure is enough to make it escaping.

### Q2. How does `[weak self]` differ from `[unowned self]`, and when is each justified?

`[weak self]` captures `self` without keeping it alive and turns it into an optional. If `self` is deallocated, the reference becomes `nil`.

```swift
callback = { [weak self] in
    guard let self else { return }
    self.updateUI()
}
```

Use `[weak self]` when the closure may outlive `self`, and it is acceptable for the closure to do nothing after `self` is gone. This is common for UI callbacks, service completions, notification handlers, timers, and long-lived async work.

`[unowned self]` captures `self` without keeping it alive, but it assumes `self` will still exist whenever the closure runs. If not, the program traps.

```swift
lazy var descriptionProvider: () -> String = { [unowned self] in
    self.description
}
```

Use `[unowned self]` only when the closure and object have tightly coupled lifetimes and the closure cannot be invoked after the object is gone. Swift’s ARC documentation says weak references are appropriate when the reference may become `nil`, while unowned references are appropriate when the closure and captured instance are expected to be deallocated together. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/AutomaticReferenceCounting.md "raw.githubusercontent.com"))

Interview version:

> `weak` is a non-owning optional reference that becomes `nil` after deallocation. It is the safe default for closures that may outlive the object, especially UI and async callbacks. `unowned` is a non-owning reference that assumes the object is still alive; accessing it after deallocation traps. I use `unowned` only when the lifetime relationship is structurally guaranteed, not just to avoid optional unwrapping.

---

## 5. Code probe

Given:

```swift
final class Screen {
    var onTap: (() -> Void)?

    deinit {
        print("Screen deinit")
    }

    func bind() {
        onTap = {
            print(self)
        }
    }
}

var screen: Screen? = Screen()
screen?.bind()
screen = nil
```

### What happens?

```text
No output.
```

### Why?

`screen?.bind()` assigns a closure to the `onTap` property.

Inside the closure:

```swift
print(self)
```

Because `self` is a class instance and the closure is stored on the instance, the closure strongly captures `self`.

The object graph becomes:

```text
screen variable
  |
  v
Screen instance
  |
  | strong
  v
onTap closure
  |
  | strong capture
  v
Screen instance
```

Then:

```swift
screen = nil
```

removes the external strong reference from the variable, but the internal cycle remains:

```text
Screen instance -> onTap closure -> Screen instance
```

So `Screen` is not deallocated. Therefore:

```swift
deinit {
    print("Screen deinit")
}
```

does not run.

The closure is also never called, so:

```swift
print(self)
```

does not run either.

### Fix or redesign

```swift
final class Screen {
    var onTap: (() -> Void)?

    deinit {
        print("Screen deinit")
    }

    func bind() {
        onTap = { [weak self] in
            guard let self else { return }
            print(self)
        }
    }
}

var screen: Screen? = Screen()
screen?.bind()
screen = nil
```

Output:

```text
Screen deinit
```

### Why this fix is correct

`[weak self]` breaks the ownership cycle:

```text
Screen instance -> onTap closure - - weak - -> Screen instance
```

The closure no longer keeps the `Screen` alive. When `screen = nil` removes the last strong external reference, the instance can deallocate.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`[weak self]`|Closure may outlive the object; UI callbacks; service completions; async work|Requires optional handling; closure may silently no-op|
|`[unowned self]`|Closure cannot outlive the object by construction|Crashes if lifetime assumption is wrong|
|Explicit teardown|You have a lifecycle event where callback storage should be cleared|Easy to forget; lifecycle coupling|
|Avoid stored closure|The callback should not be stateful or long-lived|Requires redesign|
|Use async/await|The operation is a one-shot async result|Better structure, cancellation, and ownership; needs migration|

Example explicit teardown:

```swift
final class Screen {
    var onTap: (() -> Void)?

    func bind() {
        onTap = { [weak self] in
            self?.handleTap()
        }
    }

    func unbind() {
        onTap = nil
    }

    private func handleTap() {}
}
```

---

## 6. Exercise

### Problem

Diagnose a retain cycle involving a view controller, a service callback, and a stored closure.

### Bad / naive version

```swift
import UIKit

struct Profile {
    let name: String
}

final class ProfileService {
    var onProfileLoaded: ((Profile) -> Void)?

    func loadProfile() {
        // Imagine a network request finishes later.
        onProfileLoaded?(Profile(name: "Blob"))
    }
}

@MainActor
final class ProfileViewController: UIViewController {
    private let service = ProfileService()

    override func viewDidLoad() {
        super.viewDidLoad()

        service.onProfileLoaded = { profile in
            self.render(profile)
        }

        service.loadProfile()
    }

    private func render(_ profile: Profile) {
        title = profile.name
    }

    deinit {
        print("ProfileViewController deinit")
    }
}
```

### What is wrong?

```text
ProfileViewController strongly owns ProfileService.
ProfileService strongly owns onProfileLoaded closure.
onProfileLoaded closure strongly captures ProfileViewController.
```

Ownership graph:

```text
ProfileViewController
    |
    | strong
    v
ProfileService
    |
    | strong
    v
closure
    |
    | strong capture
    v
ProfileViewController
```

The view controller cannot deallocate.

### Improved version

```swift
import UIKit

struct Profile {
    let name: String
}

final class ProfileService {
    var onProfileLoaded: ((Profile) -> Void)?

    func loadProfile() {
        onProfileLoaded?(Profile(name: "Blob"))
    }

    func cancel() {
        onProfileLoaded = nil
    }
}

@MainActor
final class ProfileViewController: UIViewController {
    private let service = ProfileService()

    override func viewDidLoad() {
        super.viewDidLoad()

        service.onProfileLoaded = { [weak self] profile in
            guard let self else { return }
            self.render(profile)
        }

        service.loadProfile()
    }

    private func render(_ profile: Profile) {
        title = profile.name
    }

    deinit {
        service.cancel()
        print("ProfileViewController deinit")
    }
}
```

### Why this is better

`[weak self]` breaks the ownership cycle. `service.cancel()` also clears the callback during teardown, which is useful when the service owns resources or could emit later.

However, this is still callback-based. A more modern Swift design for a one-shot request is usually async/await:

```swift
import UIKit

struct Profile {
    let name: String
}

protocol ProfileLoading {
    func loadProfile() async throws -> Profile
}

final class LiveProfileService: ProfileLoading {
    func loadProfile() async throws -> Profile {
        Profile(name: "Blob")
    }
}

@MainActor
final class ProfileViewController: UIViewController {
    private let service: ProfileLoading
    private var loadTask: Task<Void, Never>?

    init(service: ProfileLoading) {
        self.service = service
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        loadTask = Task { [weak self] in
            do {
                let profile = try await service.loadProfile()
                guard let self else { return }
                self.render(profile)
            } catch {
                guard let self else { return }
                self.renderError(error)
            }
        }
    }

    private func render(_ profile: Profile) {
        title = profile.name
    }

    private func renderError(_ error: Error) {
        title = "Failed"
    }

    deinit {
        loadTask?.cancel()
    }
}
```

This avoids a permanently stored service callback and makes the operation lifecycle explicit.

---

## 7. Production guidance

Use closures in production when:

```text
You need a small behavior parameter.
You need a callback for a short-lived operation.
You need lazy/deferred evaluation.
You need dependency injection without creating a protocol.
You need to customize an algorithm, such as sorting, mapping, filtering, or event handling.
```

Be careful when:

```text
The closure is stored.
The closure is retained by a long-lived service.
The closure captures self.
The closure starts or participates in async work.
The closure is @Sendable and captures mutable or non-Sendable state.
The closure hides deferred evaluation through @autoclosure.
```

Avoid when:

```text
The closure becomes a hidden lifecycle mechanism.
The closure stores complex state that should be modeled as a type.
The closure is used as an implicit delegate replacement without clear ownership.
The API needs multiple related callbacks that should be an enum, AsyncSequence, delegate, or async function.
```

Debugging checklist:

```text
Who owns the closure?
Does the closure escape?
Does it capture self?
Is self captured strongly, weakly, or unowned?
Can the closure outlive self?
Is there a cycle: object -> property -> closure -> object?
Is the closure ever cleared?
Would async/await model this better?
Would AsyncSequence model this better?
Does @Sendable change what captures are legal?
```

Modern Swift concurrency caveat:

```swift
func run(_ operation: @escaping @Sendable () async -> Void) {
    Task {
        await operation()
    }
}
```

`@Sendable` is separate from `@escaping`, but they often appear together in async APIs. `@Sendable` closures can be called concurrently, so captured values must be safe to access across concurrency domains; Swift compiler diagnostics explicitly warn about unsafe captures in `@Sendable` closures. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/sendable-closure-captures/?utm_source=chatgpt.com "Captures in a `@Sendable` closure ..."))

---

## 8. Senior vs staff-level signal

### Basic answer

> Closures are blocks of code. Escaping means they run later. Use weak self to avoid memory leaks.

### Senior answer

> A closure is a reference type that can capture surrounding state. Nonescaping closures are bounded by the callee’s call frame, while escaping closures may be stored or executed later. Once a closure escapes, captured objects can have extended lifetimes, and a stored closure that strongly captures `self` can create a retain cycle. I usually use `[weak self]` for UI and service callbacks that may outlive the owner, and `[unowned self]` only when lifetime is guaranteed.

### Staff-level answer

> I treat closure APIs as ownership APIs. The important questions are: who stores the closure, who owns whom, when is it cleared, and what lifetime does the closure imply for its captures? For one-shot async operations, I prefer `async throws` over stored callbacks. For streams of events, I consider `AsyncSequence` or a delegate with explicit ownership. For UI code, `[weak self]` is usually the correct default because the view controller may disappear. For library code, I avoid forcing clients into hidden retain cycles by documenting callback lifetime, clearing callbacks, and designing APIs that make ownership visible.

Staff-level questions to ask:

```text
Is this closure a short-lived behavior parameter or a long-lived lifecycle hook?
Can the closure outlive the object it captures?
Who clears the closure, and when?
Would async/await or AsyncSequence express this lifecycle better?
Does the closure need @Sendable, and are its captures safe under Swift 6 strict concurrency?
```

---

## 9. Interview-ready summary

Closures in Swift are reference types that can capture surrounding state. A nonescaping closure is limited to the duration of the function call that receives it, while an `@escaping` closure may be stored or called later, which extends the lifetime of its captures. Mutable locals are captured as storage, so a closure can preserve and mutate state after the original scope returns. Class instances are strongly captured by default, so a stored closure that references `self` can create a cycle like `object -> closure -> object`. Use `[weak self]` when the closure may outlive the object, and `[unowned self]` only when the object and closure have a guaranteed shared lifetime. `@autoclosure` is a separate feature for delayed expression evaluation and should be used sparingly because it hides when work happens.

---

## 10. Flashcards

Q: What is a closure in Swift?

A: A reference type representing callable code, optionally bundled with captured state from its surrounding scope.

Q: What does `@escaping` mean?

A: The closure may outlive the function call that receives it, usually because it is stored, returned, or executed asynchronously.

Q: Are closure parameters escaping by default?

A: No. Closure parameters are nonescaping by default unless marked `@escaping`.

Q: Why does an escaping closure require explicit `self` for class instances?

A: Because it may retain `self` beyond the function call, and explicit `self` forces you to notice possible reference cycles.

Q: What ownership does a closure use when capturing a class instance by default?

A: Strong capture.

Q: What is the classic closure retain cycle?

A: An object strongly owns a closure property, and the closure strongly captures the object.

Q: How does `[weak self]` behave?

A: It captures `self` without retaining it and exposes it as an optional that becomes `nil` after deallocation.

Q: How does `[unowned self]` behave?

A: It captures `self` without retaining it but assumes `self` is alive when accessed; if not, the program traps.

Q: When is `[weak self]` usually correct?

A: UI callbacks, service completions, timers, notifications, and async work where the closure may outlive the object.

Q: When is `[unowned self]` justified?

A: When the closure and object have a structurally guaranteed shared lifetime and the closure cannot run after the object deallocates.

Q: What does capturing a mutable local variable mean?

A: The closure captures variable storage, allowing the value to persist and change across closure calls.

Q: What does `@autoclosure` do?

A: It automatically wraps an expression in a no-argument closure, delaying evaluation until the closure is called.

---

## 11. Related sections

- [[A1 — Value semantics, reference semantics, identity, and copy-on-write]]
- [[A2 — `let`, `var`, `mutating`, and method semantics]]
- [[C1 — ARC fundamentals]]
- [[D6 — `Sendable` and `@Sendable`]]
- [[D9 — Continuations and bridging callback APIs]]
- [[D10 — `AsyncSequence`, Streams, and Backpressure-Aware Design]]

---

## 12. Sources

- Swift Senior/Staff Rubric — A8 closure expectations, caveats, exercise, and code probe.

- Swift.org Documentation — Closures: capture behavior, escaping closures, explicit `self`, and `@autoclosure`. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/Closures.md "raw.githubusercontent.com"))
- Swift.org Documentation — Automatic Reference Counting: closure reference cycles, capture lists, weak vs unowned. ([GitHub](https://raw.githubusercontent.com/swiftlang/swift-book/main/TSPL.docc/LanguageGuide/AutomaticReferenceCounting.md "raw.githubusercontent.com"))
- Swift.org Compiler Diagnostics — `@Sendable` closure captures. ([docs.swift.org](https://docs.swift.org/compiler/documentation/diagnostics/sendable-closure-captures/?utm_source=chatgpt.com "Captures in a `@Sendable` closure ..."))