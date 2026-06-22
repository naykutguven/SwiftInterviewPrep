---
tags:
  - swift
  - ios
  - interview-prep
  - core-semantics
  - regex
  - standard-library
  - collections
---
## 0. Rubric snapshot

**Rubric expectation**

Know how to use Swift regex literals, RegexBuilder DSL, standard-library algorithms, and collection APIs before inventing custom parsing, looping, or transformation helpers.

**Caveats**

Regexes and fluent chains can become unreadable if they encode too much logic at once. A simple loop is sometimes clearer than a clever pipeline.

**You should be able to answer**

- When is Swift Regex a better fit than `NSRegularExpression` or ad hoc string splitting?
- When do `map`, `compactMap`, `flatMap`, `reduce(into:)`, `lazy`, and collection helpers improve code?
- When do those APIs make code harder to read?

**You should be able to do**

- Refactor imperative text-processing or collection-transformation code into clearer standard-library and regex-based code without making the logic more opaque.

---

## 1. Core mental model

Swift gives you two major tools for everyday data transformation:

1. **Regex/string-processing APIs** for matching structured text.
2. **Sequence and Collection algorithms** for transforming, filtering, grouping, reducing, and querying values.

The point is not to replace all loops. The point is to avoid writing custom control flow when the standard library already has a name for the operation you mean. `map` means “transform every element.” `compactMap` means “transform and discard failures.” `reduce(into:)` means “accumulate into mutable state.” `first(where:)` means “find the first matching element.” These names encode intent better than low-level loop mechanics.

Swift Regex is a first-class Swift value. You can create it from regex syntax, including literals, or with `RegexBuilder` for a more structured DSL. Apple’s `Regex` documentation describes regular expressions as concise patterns that match or extract portions of strings, and shows construction from both literals and strings. ([Apple Developer](https://developer.apple.com/documentation/swift/regex?utm_source=chatgpt.com "Regex | Apple Developer Documentation")) `RegexBuilder` is a DSL for building regexes in a more expressive, composable Swift style. ([Apple Developer](https://developer.apple.com/documentation/regexbuilder?utm_source=chatgpt.com "RegexBuilder | Apple Developer Documentation"))

The key idea:

```text
Use the highest-level API that still makes the intent obvious.
Do not trade a readable loop for a cryptic pipeline or a cryptic regex.
```

Swift does **not** guarantee that fluent chains are automatically faster or clearer. `map.filter.reduce` can allocate intermediate arrays unless you use `lazy`, and a complex regex can become a write-only mini-language. Staff-level judgment is knowing when to stop abstracting.

---

## 2. Essential mechanics

### Regex literals, `Regex`, and `RegexBuilder`

In Swift 6.x, regex literals use slash delimiters:

```swift
let regex = /user_id:\s*(\d+)/
```

The Swift language reference defines regex literals as regular expression patterns surrounded by slashes. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/lexicalstructure/?utm_source=chatgpt.com "Lexical Structure - Documentation")) For older Swift language modes, you may still see extended delimiters like `#/.../#`.

Use a regex literal when the pattern is short and familiar:

```swift
let input = "name: Aykut, user_id: 42"

if let match = input.firstMatch(of: /user_id:\s*(\d+)/) {
    print(match.1)
}
```

Use `RegexBuilder` when you want named structure, typed transforms, composition, or readability:

```swift
import RegexBuilder

let userIDRegex = Regex {
    "user_id:"
    OneOrMore(.whitespace)
    Capture {
        OneOrMore(.digit)
    } transform: {
        Int($0)!
    }
}

if let match = "user_id: 42".wholeMatch(of: userIDRegex) {
    print(match.1)
}
```

Output:

```text
42
```

`RegexBuilder` matters because it lets you build complex regexes using normal Swift composition. Apple’s WWDC material describes Swift Regex as including the `Regex` type, regex literal syntax, and a result-builder API called `RegexBuilder`. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2022/110358/ "Swift Regex: Beyond the basics - WWDC22 - Videos - Apple Developer"))

### `map`, `compactMap`, and `flatMap`

Use `map` when every input produces exactly one output:

```swift
let names = ["aykut", "emma"]
let displayNames = names.map { $0.capitalized }

print(displayNames)
```

Output:

```text
["Aykut", "Emma"]
```

Use `compactMap` when transformation can fail and you want only successful values:

```swift
let rows = ["1", "", "2", "x", "3"]
let numbers = rows.compactMap(Int.init)

print(numbers)
```

Output:

```text
[1, 2, 3]
```

Apple’s `compactMap(_:)` docs describe it as returning an array containing the non-`nil` results of transforming each sequence element. ([Apple Developer](https://developer.apple.com/documentation/swift/sequence/compactmap%28_%3A%29?changes=latest_minor&utm_source=chatgpt.com "compactMap(_:) | Apple Developer Documentation"))

Use `flatMap` when each input produces a sequence and you want one flattened sequence:

```swift
let groups = [[1, 2], [3], [], [4, 5]]
let values = groups.flatMap { $0 }

print(values)
```

Output:

```text
[1, 2, 3, 4, 5]
```

Do **not** use `flatMap` just because it sounds fancy. If you are removing `nil`, use `compactMap`.

### `reduce(into:)`

Use `reduce(into:)` when accumulating into mutable storage such as a dictionary, set, or array:

```swift
let words = ["warning", "error", "warning"]

let counts = words.reduce(into: [:]) { counts, word in
    counts[word, default: 0] += 1
}

print(counts.sorted { $0.key < $1.key }.map { "\($0.key)=\($0.value)" })
```

Output:

```text
["error=1", "warning=2"]
```

Prefer `reduce(into:)` over `reduce` when the accumulator is mutated repeatedly. It avoids the awkwardness and potential cost of repeatedly returning new accumulator values.

### `lazy`

Use `lazy` when you want to avoid intermediate collections, especially when a later operation short-circuits:

```swift
let firstLargeSquare = (1...)
    .lazy
    .map { $0 * $0 }
    .first { $0 > 10_000 }

print(firstLargeSquare!)
```

Output:

```text
10201
```

Apple’s `LazySequenceProtocol` docs describe lazy sequences as avoiding needless storage allocation and computation by using the underlying sequence for storage and computing elements on demand. ([Apple Developer](https://developer.apple.com/documentation/swift/lazysequenceprotocol?utm_source=chatgpt.com "LazySequenceProtocol | Apple Developer Documentation"))

Without `lazy`, this kind of infinite sequence example would not be valid because eager `map` would try to build an unbounded result.

### Collection helpers

Reach for named helpers before writing loops:

```swift
let values = [2, 4, 6, 8]

values.allSatisfy { $0.isMultiple(of: 2) } // true
values.contains { $0 > 5 }                 // true
values.first { $0 > 5 }                    // 6
values.dropFirst(2)                        // [6, 8]
values.prefix(while: { $0 < 7 })           // [2, 4, 6]
```

These are often clearer than manual `for` loops because the method name tells the reader what kind of operation is happening.

---

## 3. Common traps and misconceptions

### Trap 1: Using string splitting as a parser

Bad:

```swift
func parse(_ line: String) -> Int? {
    let parts = line.split(separator: ":")
    guard parts.count == 2 else { return nil }
    return Int(parts[1].trimmingCharacters(in: .whitespaces))
}

parse("user_id: 42")     // works
parse("user_id: 42:bad") // ambiguous behavior
parse("user_id 42")      // fails, but not expressively
```

This is fragile because the structure is implicit. The code says “split on colon,” not “match a user ID field.”

Better:

```swift
let regex = /user_id:\s*(\d+)/

func parse(_ line: String) -> Int? {
    guard let match = line.wholeMatch(of: regex) else { return nil }
    return Int(match.1)
}
```

Now the expected format is explicit.

### Trap 2: Writing a regex nobody can maintain

Bad:

```swift
let regex = /^([a-z0-9_.+-]+)@([a-z0-9-]+)\.([a-z]{2,})(\?ref=(\d+))?$/
```

This may be acceptable for a tiny private utility, but in production it is easy to break and hard to review.

Better:

```swift
import RegexBuilder

let userReference = Regex {
    Anchor.startOfLine

    Capture {
        OneOrMore {
            CharacterClass(
                .word,
                "_",
                ".",
                "+",
                "-"
            )
        }
    }

    "@"

    Capture {
        OneOrMore {
            CharacterClass(.word, "-")
        }
    }

    "."

    Capture {
        Repeat(.word, 2...)
    }

    Optionally {
        "?ref="
        Capture {
            OneOrMore(.digit)
        } transform: {
            Int($0)!
        }
    }

    Anchor.endOfLine
}
```

This is longer, but reviewable. It exposes structure.

### Trap 3: Replacing clear loops with clever chains

Bad:

```swift
let result = users
    .filter { !$0.isBlocked }
    .flatMap { $0.orders }
    .filter { $0.isPaid }
    .map { $0.total }
    .reduce(0, +)
```

This may be fine if the domain is obvious. But if business rules are nontrivial, a named loop or named intermediate values can be clearer.

Better:

```swift
let activeUsers = users.filter { !$0.isBlocked }
let paidOrders = activeUsers.flatMap(\.orders).filter(\.isPaid)
let totalRevenue = paidOrders.map(\.total).reduce(0, +)
```

Even better when rules deserve names:

```swift
func isBillable(_ order: Order) -> Bool {
    order.isPaid && !order.isRefunded
}

let totalRevenue = users
    .filter { !$0.isBlocked }
    .flatMap(\.orders)
    .filter(isBillable)
    .map(\.total)
    .reduce(0, +)
```

### Trap 4: Using `reduce` for everything

Bad:

```swift
let names = users.reduce(into: [String]()) { result, user in
    result.append(user.name)
}
```

Better:

```swift
let names = users.map(\.name)
```

Use the most specific operation. `map` says more than `reduce`.

### Trap 5: Forgetting Unicode and `String` semantics

Swift `String` is not a byte buffer. A regex over `String` works with Swift’s string model; ad hoc indexing by byte or UTF-16 offset can corrupt user-visible text. This connects directly to [[C4 — String and Unicode correctness]].

---

## 4. Direct answers to rubric questions

### Q1. When is Swift Regex a better fit than `NSRegularExpression` or ad hoc string splitting?

Swift Regex is better when you are working in Swift-native code, the pattern is known at compile time, you want typed captures, you want to use Swift string-processing algorithms like `firstMatch(of:)`, `wholeMatch(of:)`, `replacing(_:with:)`, or you want the readability/composability of `RegexBuilder`.

It is better than ad hoc splitting when the input has real structure: optional fields, whitespace rules, repeated segments, anchors, captures, alternatives, or validation rules.

It is usually better than `NSRegularExpression` for new Swift code because `Regex` integrates with Swift’s type system and string APIs. `NSRegularExpression` still appears when you need older Foundation-based APIs, Objective-C interoperability, or existing regex infrastructure.

Interview version:

> Swift Regex is my default for Swift-native structured text parsing when the pattern is more than a trivial delimiter split. It gives me a first-class `Regex` value, regex literals, RegexBuilder, and typed captures. I still use simple string APIs for genuinely simple cases, and I would only reach for `NSRegularExpression` for legacy Foundation interop, existing code, or APIs that specifically require it.

### Q2. When do `map`, `compactMap`, `flatMap`, `reduce(into:)`, `lazy`, and collection helpers improve code, and when do they make code harder to read?

They improve code when the operation has a standard name:

```text
map          -> transform every element
compactMap   -> transform and discard nil
flatMap      -> transform to sequences and flatten
filter       -> keep matching values
reduce(into:) -> accumulate into mutable state
lazy         -> defer computation and avoid intermediate storage
first(where:) -> find first match
allSatisfy   -> validate every element
```

They make code worse when the chain hides business logic, creates type-inference noise, repeats expensive work, or forces readers to mentally simulate the pipeline.

Interview version:

> I use standard-library algorithms when their names communicate the operation better than a manual loop. But I avoid pipeline cleverness when the business rules are complex, when debugging intermediate state matters, or when the chain creates avoidable allocations. In those cases, named intermediate values or a plain loop can be more maintainable.

---

## 5. Code probe / examples

The rubric has no dedicated A16 code probe. Following the study template’s code-probe style, here are a minimal example, a counterexample, and a production-style example. The template expects exact output or compiler behavior for code probes and examples.

### Minimal example

Given:

```swift
import RegexBuilder

let input = "user_id: 42"

let idRegex = Regex {
    "user_id:"
    OneOrMore(.whitespace)
    Capture {
        OneOrMore(.digit)
    } transform: {
        Int($0)!
    }
}

if let match = input.wholeMatch(of: idRegex) {
    print(match.1)
}

let rows = ["1", "", "2", "x", "3"]
let numbers = rows.compactMap(Int.init)
print(numbers)

let counts = ["warning", "error", "warning"].reduce(into: [:]) { counts, word in
    counts[word, default: 0] += 1
}

print(counts.sorted { $0.key < $1.key }.map { "\($0.key)=\($0.value)" })
```

### What happens?

```text
42
[1, 2, 3]
["error=1", "warning=2"]
```

### Why?

```text
"user_id: 42"
   |
   wholeMatch(of:)
   |
   RegexBuilder pattern:
   - literal "user_id:"
   - whitespace
   - captured digits transformed into Int
   |
   match.1 == 42

["1", "", "2", "x", "3"]
   |
   compactMap(Int.init)
   |
   ["1" -> 1, "" -> nil, "2" -> 2, "x" -> nil, "3" -> 3]
   |
   [1, 2, 3]

["warning", "error", "warning"]
   |
   reduce(into: Dictionary)
   |
   ["warning": 2, "error": 1]
```

### Counterexample: pipeline that is too clever

```swift
let result = lines
    .compactMap { $0.wholeMatch(of: /(\d+),([^,]+),(\d+)/) }
    .map { (Int($0.1)!, String($0.2), Int($0.3)!) }
    .filter { $0.2 != "internal" && $0.3 > 0 }
    .reduce(into: [:]) { $0[$1.1, default: 0] += $1.2 }
```

This is compact, but it is poor production code. The tuple fields are anonymous, force unwraps hide parsing failure, and business rules are mixed into parsing.

Better:

```swift
struct UsageRecord {
    let userID: Int
    let category: String
    let amount: Int
}

func parseUsageRecord(_ line: String) -> UsageRecord? {
    guard let match = line.wholeMatch(of: /(\d+),([^,]+),(\d+)/),
          let userID = Int(match.1),
          let amount = Int(match.3)
    else {
        return nil
    }

    return UsageRecord(
        userID: userID,
        category: String(match.2),
        amount: amount
    )
}

func isBillable(_ record: UsageRecord) -> Bool {
    record.category != "internal" && record.amount > 0
}

let result = lines
    .compactMap(parseUsageRecord)
    .filter(isBillable)
    .reduce(into: [:]) { totals, record in
        totals[record.category, default: 0] += record.amount
    }
```

### Why this fix is correct

The improved version separates:

```text
parsing rule      -> parseUsageRecord
domain rule       -> isBillable
aggregation rule  -> reduce(into:)
```

That separation is the main production value. The code is not just “more functional”; it is more reviewable.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|Regex literal|Short, stable, familiar pattern|Can become unreadable as complexity grows|
|`RegexBuilder`|Complex pattern, typed captures, reviewability|More verbose|
|Plain loop|Stateful logic, early exits, debugging clarity|More boilerplate|
|`NSRegularExpression`|Legacy Foundation/Objective-C interop|Stringly typed, less Swift-native|
|Parser type|Complex grammar or long-lived parsing logic|More design work|

---

## 6. Exercise

### Problem

Refactor imperative text-processing or collection-transformation code into clearer standard-library and regex-based code without making the logic more opaque.

### Bad / naive version

```swift
struct Event {
    let userID: Int
    let action: String
    let durationMS: Int
}

func parseEvents(_ lines: [String]) -> [Event] {
    var events: [Event] = []

    for line in lines {
        let parts = line.split(separator: " ")

        if parts.count != 4 {
            continue
        }

        let userPart = parts[1]
        let actionPart = parts[2]
        let durationPart = parts[3]

        let userPieces = userPart.split(separator: "=")
        let actionPieces = actionPart.split(separator: "=")
        let durationPieces = durationPart.split(separator: "=")

        if userPieces.count != 2 ||
            actionPieces.count != 2 ||
            durationPieces.count != 2 {
            continue
        }

        guard let userID = Int(userPieces[1]) else {
            continue
        }

        let action = String(actionPieces[1])

        let durationString = durationPieces[1].replacingOccurrences(of: "ms", with: "")

        guard let durationMS = Int(durationString) else {
            continue
        }

        events.append(
            Event(
                userID: userID,
                action: action,
                durationMS: durationMS
            )
        )
    }

    return events
}
```

### What is wrong?

```text
The expected line format is implicit.
The parsing logic is scattered across repeated split/count checks.
The code accepts malformed input accidentally.
The domain object is created far away from the format definition.
The loop is doing parsing, validation, and accumulation all at once.
```

### Improved version

```swift
import RegexBuilder

struct Event: CustomStringConvertible {
    let userID: Int
    let action: String
    let durationMS: Int

    var description: String {
        "Event(userID: \(userID), action: \(action), durationMS: \(durationMS))"
    }
}

let lines = [
    "2026-06-19 user=42 action=login duration=120ms",
    "bad line",
    "2026-06-19 user=7 action=purchase duration=450ms",
    "2026-06-19 user=42 action=logout duration=80ms"
]

let lineRegex = Regex {
    Repeat(.digit, count: 4)
    "-"
    Repeat(.digit, count: 2)
    "-"
    Repeat(.digit, count: 2)

    OneOrMore(.whitespace)

    "user="
    Capture {
        OneOrMore(.digit)
    } transform: {
        Int($0)!
    }

    OneOrMore(.whitespace)

    "action="
    Capture {
        OneOrMore(.word)
    } transform: {
        String($0)
    }

    OneOrMore(.whitespace)

    "duration="
    Capture {
        OneOrMore(.digit)
    } transform: {
        Int($0)!
    }
    "ms"
}

let events = lines.compactMap { line -> Event? in
    guard let match = line.wholeMatch(of: lineRegex) else {
        return nil
    }

    return Event(
        userID: match.1,
        action: match.2,
        durationMS: match.3
    )
}

let durationByUser = events.reduce(into: [:]) { result, event in
    result[event.userID, default: 0] += event.durationMS
}

print(events)
print(durationByUser.sorted { $0.key < $1.key }.map { "user \($0.key): \($0.value)ms" })
```

Output:

```text
[Event(userID: 42, action: login, durationMS: 120), Event(userID: 7, action: purchase, durationMS: 450), Event(userID: 42, action: logout, durationMS: 80)]
["user 7: 450ms", "user 42: 200ms"]
```

### Why this is better

The regex defines the accepted input shape in one place. `compactMap` expresses “parse valid events, discard invalid lines.” `reduce(into:)` expresses “aggregate duration by user.” The result is more declarative, but not excessively clever.

A senior/staff-level review should still ask whether `Int($0)!` is acceptable. In this specific regex, the capture is digits only, so the force unwrap is locally justified. In a public parser or untrusted boundary, prefer a throwing parser or explicit guarded transform.

---

## 7. Production guidance

Use this in production when:

```text
The operation has a standard-library name.
The regex pattern is short enough to review, or RegexBuilder makes it readable.
The transformation pipeline expresses business intent better than a manual loop.
You want typed captures instead of stringly typed capture groups.
You want Unicode-aware Swift string processing instead of byte-offset manipulation.
```

Be careful when:

```text
The regex becomes a full parser.
The fluent chain mixes parsing, business rules, networking, mutation, and aggregation.
Intermediate arrays are large and accidental.
Type inference errors become hard to diagnose.
The pipeline is harder to breakpoint than a loop.
```

Avoid when:

```text
A plain loop is clearer.
The input is a real grammar and needs a parser.
The business rules need named concepts.
The regex is copied from another language without validating Swift behavior.
The pipeline exists only to look clever.
```

Debugging checklist:

```text
Can I name each operation in the pipeline?
Are captures typed and validated?
Is wholeMatch(of:) more correct than firstMatch(of:)?
Am I accidentally accepting partial input?
Would RegexBuilder make the pattern easier to review?
Am I allocating intermediate arrays unnecessarily?
Should this be lazy?
Would named intermediate values make this easier to debug?
Is a simple loop clearer?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> I use `map`, `filter`, and regex because they make code shorter.

### Senior answer

> I use standard-library algorithms when they express the operation more directly than a loop. I use Swift Regex for structured text because it integrates with Swift’s string APIs and can give me typed captures. But I avoid clever chains and unreadable regexes.

### Staff-level answer

> I treat regexes and collection pipelines as API design tools, not just syntax shortcuts. I choose the abstraction that makes invariants visible: regex or RegexBuilder for text shape, named functions for domain rules, `compactMap` for failable parsing, `reduce(into:)` for accumulation, and plain loops where stateful control flow is clearer. I also care about reviewability, intermediate allocation, Unicode correctness, and whether the parsing boundary should become a dedicated type.

Staff-level questions to ask:

```text
Is this text format stable enough for regex, or does it need a parser?
Should invalid input be ignored, returned as nil, or reported with a typed error?
Is the regex anchored with wholeMatch(of:) when full validation is required?
Are captures named or mapped into a domain type quickly?
Does this pipeline allocate large intermediate collections?
Would lazy help, or would it obscure evaluation timing?
Are we hiding business rules inside anonymous closures?
Would a loop be more maintainable for debugging and early exits?
```

---

## 9. Interview-ready summary

Swift Regex and standard-library algorithms are about expressing intent at the right level. I use Swift Regex or RegexBuilder when text has structure that simple splitting cannot safely represent, especially when I want Swift-native matching and typed captures. I use `map`, `compactMap`, `flatMap`, `reduce(into:)`, `lazy`, and collection helpers when their names make the transformation clearer than manual loops. But I do not treat fluent code as automatically better: a complex regex or long pipeline can be worse than a plain loop. The senior-level skill is knowing the APIs; the staff-level skill is preserving readability, correctness, performance, and reviewability.

---

## 10. Flashcards

Q: When is Swift Regex better than ad hoc `split`?

A: When the input has meaningful structure: anchors, whitespace rules, optional fields, captures, alternatives, or validation requirements.

Q: When should you use `RegexBuilder` instead of a regex literal?

A: When the regex is complex enough that named structure, typed captures, composition, or reviewability matters more than concision.

Q: What does `compactMap` communicate?

A: “Transform each element, discard failed or missing results.”

Q: What does `flatMap` communicate on sequences?

A: “Transform each element into a sequence, then flatten one level.”

Q: When is `reduce(into:)` preferable to `reduce`?

A: When accumulating into mutable storage such as a dictionary, set, or array.

Q: What problem does `lazy` solve?

A: It defers computation and avoids intermediate collections, especially useful when later operations short-circuit.

Q: When is a loop better than a pipeline?

A: When logic is stateful, business-heavy, needs early exits, needs clear debugging, or the chain hides intent.

Q: What is a common regex correctness bug?

A: Using `firstMatch(of:)` when full validation requires `wholeMatch(of:)`.

---

## 11. Related sections

- [[A5 — Pattern matching and exhaustive control flow]]
- [[A7 — Functions, parameter labels, defaults, and API surface]]
- [[A14 — Operators, literals, and language affordances]]
- [[C4 — String and Unicode correctness]]
- [[C5 — Collection protocols, complexity, and index invalidation]]
- [[F2 — Debugging Swift code with LLDB and compiler diagnostics]]

---

## 12. Sources

- Swift Senior/Staff Rubric and Prioritized Study Checklist — A16 section and senior/staff expectations.
- Apple Developer Documentation — `Regex`. ([Apple Developer](https://developer.apple.com/documentation/swift/regex?utm_source=chatgpt.com "Regex | Apple Developer Documentation"))
- Apple Developer Documentation — `RegexBuilder`. ([Apple Developer](https://developer.apple.com/documentation/regexbuilder?utm_source=chatgpt.com "RegexBuilder | Apple Developer Documentation"))
- The Swift Programming Language — lexical structure for regex literals. ([Swift Belgeleri](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/lexicalstructure/?utm_source=chatgpt.com "Lexical Structure - Documentation"))
- Swift Evolution SE-0357 — regex-powered string-processing algorithms. ([GitHub](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0357-regex-string-processing-algorithms.md "swift-evolution/proposals/0357-regex-string-processing-algorithms.md at main · swiftlang/swift-evolution · GitHub"))
- Apple WWDC22 — Swift Regex: Beyond the basics. ([Apple Developer](https://developer.apple.com/videos/play/wwdc2022/110358/ "Swift Regex: Beyond the basics - WWDC22 - Videos - Apple Developer"))
- Swift.org — Announcing Swift Algorithms. ([swift.org](https://swift.org/blog/swift-algorithms/ "Announcing Swift Algorithms | Swift.org"))