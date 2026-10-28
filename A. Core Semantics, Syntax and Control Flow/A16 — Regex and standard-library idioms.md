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

# A16 — Regex and Standard-Library Idioms

**Learning goals:**

- State an exact grammar and distinguish whole, prefix, and substring matching.
- Compare regex literals and RegexBuilder, separating shape, conversion, domain validity, and matching errors.
- Choose map, compactMap, Sequence flatMap, reduce(into:), and lazy evaluation with complete rejection/overflow accounting.

**Prerequisites:** [[A4 — Optionals and Nil Modeling]] for nil, [[A5 — Pattern Matching and Exhaustive Control Flow]] for matching, and [[A13 — Error handling and typed throws]] for typed errors and Result. [[C4 — String and Unicode correctness]] explains Unicode distinctions.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Parsing a row has several jobs: recognize its grammar, convert captured fields, validate domain values, and explain failures. Regex handles shape. A digit match does not prove a number fits Int, and a date-shaped string does not prove a valid calendar day.

**Whole matching** must consume the entire input. **Prefix matching** starts at the beginning but may leave a suffix. **Search matching** can find a match inside other text. Choose the consumption rule and character classes before choosing literal syntax or RegexBuilder; both can express the same grammar. [Regex](https://developer.apple.com/documentation/swift/regex)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; potentially trapping or deadlocking code is never run.

Extended `#/.../#` regex literals used here need no bare-slash-literal flag. RegexBuilder examples import RegexBuilder. Each block remains an independent script unless it names shared setup.

### Consumption is part of the grammar

A fixed ASCII digit pattern can match a prefix or a substring even when the whole input is invalid.

```swift
let pattern = #/user_id:[0-9]+/#
precondition("user_id:42".wholeMatch(of: pattern) != nil)
precondition("user_id:42 trailing".wholeMatch(of: pattern) == nil)
precondition("user_id:42 trailing".prefixMatch(of: pattern) != nil)
precondition("prefix user_id:42".prefixMatch(of: pattern) == nil)
precondition("prefix user_id:42".firstMatch(of: pattern) != nil)
```

**Walkthrough:** Only wholeMatch enforces no trailing or leading text here. The same pattern with prefixMatch or firstMatch implements a different acceptance policy.

### Typed captures still need conversion policy

This grammar allows exactly user_id:, zero or more ASCII spaces/tabs, then one or more ASCII digits. Leading zeroes are accepted; newlines, signs, leading whitespace, and trailing input are rejected.

```swift
import RegexBuilder

func parseUserIDBuilder(_ input: String) -> Int? {
    let pattern: Regex<(Substring, Int)> = Regex {
        "user_id:"
        ZeroOrMore { CharacterClass.anyOf(" \t") }
        TryCapture {
            OneOrMore { CharacterClass("0"..."9") }
        } transform: {
            Int($0)
        }
    }.matchingSemantics(.unicodeScalar)
    return input.wholeMatch(of: pattern)?.1
}
precondition(parseUserIDBuilder("user_id:42") == 42)
precondition(parseUserIDBuilder("user_id:\t00042") == 42)
precondition(parseUserIDBuilder("user_id:٤٢") == nil)
precondition(parseUserIDBuilder("user_id:" + String(repeating: "9", count: 30)) == nil)
```

**Walkthrough:** TryCapture converts the digit Substring to Int and makes conversion failure fail the match. The typed Int capture does not mean the digits cannot overflow; it means only successful conversion produces that capture. [SE-0351](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0351-regex-builder.md)

### Collection transforms have different accounting contracts

map keeps one output per input; compactMap drops nil; Sequence flatMap concatenates sequences; reduce(into:) mutates one accumulator.

```swift
let rows = ["1", "bad", "2"]
let mapped = rows.map { Int($0) }
let accepted = rows.compactMap { Int($0) }
let repeated = accepted.flatMap { [$0, $0] }
let counts = ["a", "b", "a"].reduce(into: [String: Int]()) { result, key in
    result[key, default: 0] += 1
}
precondition(mapped == [1, nil, 2])
precondition(accepted == [1, 2])
precondition(repeated == [1, 1, 2, 2])
precondition(counts == ["a": 2, "b": 1])
```

**Walkthrough:** The malformed middle row remains visible in mapped but disappears from accepted. flatMap here is the sequence operation, distinct from Optional.flatMap in [[A4 — Optionals and Nil Modeling|A4]]. The accumulator’s inout access is scoped; reduce(into:) is not automatic concurrency protection.

## Rules and pitfalls

### Matched digits can still overflow

**Common pitfall:** Force-unwrapping Int conversion because a regex matched ASCII digits. Compile-only; invoking this function would trap.

**Mistake:** Compile-only. The unsafe path is not executed.

```swift
func unsafeID() -> Int {
    let raw = "999999999999999999999999999999"
    precondition(raw.wholeMatch(of: #/[0-9]+/#) != nil)
    return Int(raw)!
}
```

**Consequence:** The number is syntactically valid but not representable as Int.

**Why:** Regex establishes character shape; conversion enforces the numeric representation’s bounds.

**Correction:** Classify conversion failure instead of asserting it away.

```swift
enum Failure: Error, Equatable { case grammar, overflow }
func parse(_ raw: String) throws(Failure) -> Int {
    guard raw.wholeMatch(of: #/[0-9]+/#) != nil else { throw .grammar }
    guard let value = Int(raw) else { throw .overflow }
    return value
}
do {
    _ = try parse(String(repeating: "9", count: 30))
    preconditionFailure()
} catch { precondition(error == .overflow) }
let valid = try parse("00042")
precondition(valid == 42)
```

**What changes:** Overflow remains a distinct outcome from malformed characters. A domain may impose smaller bounds even on representable Int values.

### compactMap can silently lose rejected rows

**Common pitfall:** Using a nil-returning parser when the importer must account for every input.

**Mistake:** Standalone runnable example.

```swift
let rows = ["1", "bad", "2"]
let values = rows.compactMap { Int($0) }
precondition(values == [1, 2] && values.count < rows.count)
```

**Consequence:** The values no longer identify the rejected row or explain its failure.

**Why:** Dropping nil is compactMap’s intended behavior; the accounting requirement needs a different representation.

**Correction:** Store one classified outcome per row, including its original index.

```swift
enum Failure: Error, Equatable { case invalid(String) }
struct Row {
    let index: Int
    let outcome: Result<Int, Failure>
}
let rows = ["1", "bad", "2"].enumerated().map { index, raw in
    Row(index: index, outcome: Int(raw).map { .success($0) } ?? .failure(.invalid(raw)))
}
precondition(rows.count == 3)
precondition(rows[1].index == 1 && rows[1].outcome == .failure(.invalid("bad")))
precondition(rows[2].outcome == .success(2))
```

**What changes:** The result preserves order and a rejection record. compactMap remains appropriate when deliberate filtering is the complete contract.

### Lazy traversal does not memoize work

**Common pitfall:** Assuming traversing the same lazy map twice reuses the first results.

**Mistake:** Standalone runnable example.

```swift
func probe() {
    var calls = 0
    let lazyValues = [1, 2].lazy.map { value in
        calls += 1
        return value * 2
    }
    precondition(calls == 0)
    let first = Array(lazyValues)
    let second = Array(lazyValues)
    precondition(first == second && calls == 4)
}
probe()
```

**Consequence:** The transformation runs again during the second traversal.

**Why:** A lazy adapter defers work to consumption; it is not a cache.

**Correction:** Materialize once when stable reuse is required.

```swift
func probe() {
    var calls = 0
    let values = [1, 2].map { value in
        calls += 1
        return value * 2
    }
    let first = values
    let second = values
    precondition(first == [2, 4] && second == first && calls == 2)
}
probe()
```

**What changes:** The eager array stores the transformed values once. This trades storage for repeatable reuse; a streaming consumer can still prefer a lazy single pass.

## Worked example

### A broader character class changes acceptance

**Setup:** Standalone RegexBuilder script. This example intentionally allows Unicode whitespace and digit characters, and only asks whether the shape matches.

**Predict:** Does this builder implement the same grammar as the ASCII field above?

```swift
import RegexBuilder

let broad = Regex {
    "user_id:"
    ZeroOrMore(.whitespace)
    OneOrMore(.digit)
}
let strict = #/user_id:[ \t]*[0-9]+/#
precondition("user_id:\n42".wholeMatch(of: broad) != nil)
precondition("user_id:٤٢".wholeMatch(of: broad) != nil)
precondition("user_id:\n42".wholeMatch(of: strict) == nil)
precondition("user_id:٤٢".wholeMatch(of: strict) == nil)
```

**Outcome:** The broader builder accepts newline whitespace and non-ASCII digit characters that the strict pattern rejects.

**Why:** The character classes changed, not merely the notation. Numeric conversion would be another stage and can reject text even when this broad grammar matches.

## Interview questions

### How do the literal and builder differ in whitespace, digits, and trailing input?

**Answer:** They need not differ. These two parsers implement the same declared ASCII grammar and whole-input consumption. Their conversion paths both return nil for overflow; broader classes or prefix matching would change the contract.

**Example:** Standalone runnable example.

```swift
import RegexBuilder

func literal(_ input: String) -> Int? {
    let pattern = #/user_id:[ \t]*([0-9]+)/#.matchingSemantics(.unicodeScalar)
    guard let match = input.wholeMatch(of: pattern) else { return nil }
    return Int(match.1)
}
func builder(_ input: String) -> Int? {
    let pattern: Regex<(Substring, Int)> = Regex {
        "user_id:"
        ZeroOrMore { CharacterClass.anyOf(" \t") }
        TryCapture {
            OneOrMore { CharacterClass("0"..."9") }
        } transform: {
            Int($0)
        }
    }.matchingSemantics(.unicodeScalar)
    return input.wholeMatch(of: pattern)?.1
}
for raw in ["user_id:42", "user_id:\t00042"] {
    precondition(literal(raw) == 42 && builder(raw) == 42)
}
for raw in [
    " user_id:42", "user_id:\n42", "user_id:٤٢", "user_id:42x",
    "user_id:", "user:42", "user_id:" + String(repeating: "9", count: 30),
] {
    precondition(literal(raw) == nil && builder(raw) == nil)
}
```

**Walkthrough:** The literal captures a Substring and converts it afterward. The builder’s TryCapture performs the failable conversion during matching. Both pass the same corpus, including horizontal whitespace and leading zeroes.

**Boundary:** Returning only nil merges grammar failure and conversion failure. Choose a classified error when recovery or import accounting requires that distinction. Typed capture output does not by itself establish Sendable for every Regex value.

### Which failures become nil, which throw, and what does complete row accounting require?

**Answer:** Ordinary shape mismatch returns nil. Dynamic pattern construction or a custom matching transform can throw. String matching conveniences can suppress matching errors into nil; use the throwing Regex API when those failures need classification, and preserve one outcome per input row.

**Example:** Standalone runnable example.

```swift
import RegexBuilder

enum MatchFailure: Error { case blocked }
let pattern = Regex {
    "id:"
    TryCapture {
        OneOrMore(.digit)
    } transform: { (text: Substring) -> Int? in
        if text == "13" { throw MatchFailure.blocked }
        return Int(text)
    }
}
precondition("id:x".wholeMatch(of: pattern) == nil)
precondition("id:13".wholeMatch(of: pattern) == nil)
do {
    _ = try pattern.wholeMatch(in: "id:13")
    preconditionFailure("Expected matching error")
} catch { precondition(error is MatchFailure) }
do {
    _ = try Regex("[")
    preconditionFailure("Expected invalid pattern")
} catch { /* Pattern construction failed; no input row was parsed. */  }
```

**Walkthrough:** The convenience reports both mismatch and a suppressed transform error as nil. The throwing call exposes MatchFailure. Invalid dynamic pattern syntax fails before matching even starts. [SE-0357](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0357-regex-string-processing-algorithms.md)

**Boundary:** The nil-dropping pitfall shows per-row Result storage. A full importer must also define domain validation, size limits, rejection order, and aggregate overflow; the optional importer below provides that model.

## Focused exercise

**Task:** Parse the declared ASCII user-ID grammar while classifying malformed input separately from overflow.

**Acceptance checks:**

- Accept ASCII digits, leading zeroes, and only the declared spaces/tabs after user_id:.
- Reject leading/trailing whitespace, newlines, signs, wrong keys, Unicode digits, and trailing text.
- An oversized digit sequence produces overflow, not grammar failure.

**Solution:** Standalone literal parser. The fixed prefix and capture remove the need to guess offsets.

```swift
enum ParseFailure: Error, Equatable { case grammar, overflow }
func userID(_ raw: String) throws(ParseFailure) -> Int {
    let pattern = #/user_id:[ \t]*([0-9]+)/#.matchingSemantics(.unicodeScalar)
    guard let match = raw.wholeMatch(of: pattern) else { throw .grammar }
    guard let value = Int(match.1) else { throw .overflow }
    return value
}
for raw in ["user_id:42", "user_id: 00042", "user_id:\t42"] {
    let value = try userID(raw)
    precondition(value == 42)
}
for raw in [
    " user_id:42", "user_id:42 ", "user_id:42x", "user_id:\n42",
    "user_id:٤٢", "user_id:+42", "user_id:", "user:42",
] {
    do {
        _ = try userID(raw)
        preconditionFailure("Unexpected acceptance")
    } catch { precondition(error == .grammar) }
}
do {
    _ = try userID("user_id:" + String(repeating: "9", count: 30))
    preconditionFailure("Expected overflow")
} catch { precondition(error == .overflow) }
```

**Reasoning:** Whole matching establishes the exact grammar. Int conversion separately establishes representability. The grammar permits zero; a positive-ID domain would add a separate validation case. Reuse a validated pattern or parser dependency in a hot loop rather than assuming notation guarantees caching or performance.

## In practice

**Choose it when:** Use regex for a concise stable grammar and explicit parsing when richer structure or recovery makes it clearer.

**Investigate by:** Test character classes and consumption boundaries, then account for every batch input and numeric conversion.

**Strong evidence:** Preserve rejection reasons, accepted order, and aggregate overflow policy instead of reporting only a successfully parsed subset.

### Complete importer and aggregate overflow — optional

The following larger model preserves the event-import exercise. Read the small parsers first. Its three blocks assemble in order: Event parser setup, Event report setup, then a runnable probe. The policy below describes values emitted by the parser, not every memberwise-constructed Event.

#### Problem and policy

Parse one event per input String, retain classified rejected rows, and total duration by user. Input has exactly this grammar:

> YYYY-MM-DD user=ASCII_DIGITS action=LOWERCASE_ASCII_LETTERS duration=ASCII_DIGITSms

Each separator is one ASCII space. There is no leading/trailing whitespace or terminal newline; upstream framing removes line terminators. Input is limited to 512 UTF-8 bytes. Dates use the proleptic Gregorian calendar with years 1...9999 and valid month/day combinations. User IDs must be positive Ints; duration is a nonnegative Int. Actions are login, logout, or purchase. Leading zeros in numeric fields are allowed. This is a restricted log grammar, not a general date/time or localization API.

Do not force unwrap matched digits or infer calendar validity from punctuation. Do not discard rejected rows with compactMap when the import requires rejection accounting. Do not use unchecked += for arbitrary totals.

#### Parsing and domain validation

**Event parser setup — compile-only.**

```swift
struct EventDate: Equatable, Sendable {
    let year: Int
    let month: Int
    let day: Int

    init?(year: Int, month: Int, day: Int) {
        guard (1...9999).contains(year), (1...12).contains(month) else { return nil }
        let leap =
            year.isMultiple(of: 400)
            || (year.isMultiple(of: 4) && !year.isMultiple(of: 100))
        let days = [31, leap ? 29 : 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31]
        guard (1...days[month - 1]).contains(day) else { return nil }
        self.year = year
        self.month = month
        self.day = day
    }
}

enum EventAction: String, Sendable {
    case login, logout, purchase
}

struct Event: Equatable, Sendable {
    let date: EventDate
    let userID: Int
    let action: EventAction
    let durationMS: Int
}

enum EventParseFailure: Error, Equatable, Sendable {
    case lineTooLong
    case invalidFormat
    case invalidNumber(field: String)
    case invalidDate
    case invalidUserID
    case unknownAction(String)
}

func parseEvent(_ line: String) -> Result<Event, EventParseFailure> {
    guard line.utf8.prefix(513).count <= 512 else { return .failure(.lineTooLong) }
    let regex =
        #/([0-9]{4})-([0-9]{2})-([0-9]{2}) user=([0-9]+) action=([a-z]+) duration=([0-9]+)ms/#
        .matchingSemantics(.unicodeScalar)
    guard let match = line.wholeMatch(of: regex) else { return .failure(.invalidFormat) }
    guard let year = Int(match.1), let month = Int(match.2), let day = Int(match.3) else {
        return .failure(.invalidDate)
    }
    guard let date = EventDate(year: year, month: month, day: day) else {
        return .failure(.invalidDate)
    }
    guard let userID = Int(match.4) else { return .failure(.invalidNumber(field: "user")) }
    guard userID > 0 else { return .failure(.invalidUserID) }
    guard let action = EventAction(rawValue: String(match.5)) else {
        return .failure(.unknownAction(String(match.5)))
    }
    guard let duration = Int(match.6) else {
        return .failure(.invalidNumber(field: "duration"))
    }
    return .success(Event(date: date, userID: userID, action: action, durationMS: duration))
}
```

The nonnegative duration follows from the ASCII digits-only grammar plus successful Int conversion. The four/two/two digit date fields cannot overflow the checked Apple Int representation; they still go through guarded conversion and domain construction. Event's ordinary memberwise initializer is not a global validation boundary: these guarantees describe values emitted by parseEvent. If other callers must be constrained, hide raw construction behind validated domain types or initializers too.

The fixed pattern has no throwing custom transform. This helper intentionally reports a nil from the String convenience as invalidFormat. It does not expose matching-abort details; a parser needing those details should use the throwing Regex API and design a corresponding failure case. It never promises regex cancellation is a hard CPU-time limit.

#### Batch reporting and overflow-aware aggregation

**Event report setup — compile-only; prepend Event parser setup.**

```swift
struct EventRejection: Equatable, Sendable {
    let lineIndex: Int
    let reason: EventParseFailure
}

struct EventReport: Equatable, Sendable {
    let events: [Event]
    let rejections: [EventRejection]
    let durationByUser: [Int: Int]
}

enum AggregationFailure: Error, Equatable, Sendable {
    case durationOverflow(userID: Int, lineIndex: Int)
}

func makeEventReport(
    _ lines: [String]
) throws(AggregationFailure) -> EventReport {
    var events: [Event] = []
    var rejections: [EventRejection] = []
    var totals: [Int: Int] = [:]
    for (index, line) in lines.enumerated() {
        switch parseEvent(line) {
        case .failure(let reason):
            rejections.append(EventRejection(lineIndex: index, reason: reason))
        case .success(let event):
            let (sum, overflow) = totals[event.userID, default: 0]
                .addingReportingOverflow(event.durationMS)
            guard !overflow else {
                throw .durationOverflow(userID: event.userID, lineIndex: index)
            }
            totals[event.userID] = sum
            events.append(event)
        }
    }
    return EventReport(events: events, rejections: rejections, durationByUser: totals)
}
```

Malformed rows become classified rejections, in input order, with zero-based line indices. An aggregation overflow fails the whole call and returns no partial report. The function has no external side effects to undo. It still materializes accepted events, rejections, and totals; per-line size limits do not bound the number of rows or total memory. A streaming import needs a separate retention/backpressure policy.

**Runnable probe — prepend Event parser setup, then Event report setup.**

```swift
func eventReportProbe() throws {
    let report = try makeEventReport([
        "2026-09-24 user=42 action=login duration=120ms",
        "bad line",
        "2026-09-24 user=7 action=purchase duration=450ms",
        "2026-09-24 user=42 action=logout duration=80ms",
    ])
    print(report.events.map { "\($0.userID) \($0.action.rawValue) \($0.durationMS)" })
    print(report.rejections.map(\.lineIndex))
    print(
        report.durationByUser.keys.sorted().map {
            "user \($0): \(report.durationByUser[$0, default: 0])ms"
        })
}

try eventReportProbe()
```

Output:

> ["42 login 120", "7 purchase 450", "42 logout 80"]
> [1]
> ["user 7: 450ms", "user 42: 200ms"]

Test valid leap days, century exceptions, missing/wrong/reordered labels, terminal newlines, Unicode lookalikes, zero IDs, unknown actions, oversized integers, the byte limit, empty/all-rejected batches, and per-user sum overflow. Verify rejected indices and successful order, not just the number of outputs. A literal and builder can both be good solutions if they implement the same declared grammar and failure policy.

These additional runnable checks prepend Event parser setup, then Event report setup. They verify the rejection policy and aggregate-overflow outcome, including empty and entirely rejected batches.

```swift
func checkImporter() throws {
    func line(
        date: String = "2024-02-29", user: String = "42",
        action: String = "login", duration: String = "0"
    ) -> String {
        "\(date) user=\(user) action=\(action) duration=\(duration)ms"
    }
    func rejected(_ raw: String, as reason: EventParseFailure) {
        guard case .failure(let actual) = parseEvent(raw) else { preconditionFailure() }
        precondition(actual == reason)
    }
    guard case .success = parseEvent(line()) else { preconditionFailure() }
    guard case .success = parseEvent(line(date: "2000-02-29")) else { preconditionFailure() }
    rejected(line(date: "1900-02-29"), as: .invalidDate)
    rejected(line(date: "2023-02-29"), as: .invalidDate)
    rejected(line(date: "0000-01-01"), as: .invalidDate)
    rejected(line(user: "0"), as: .invalidUserID)
    rejected(line(user: "٤٢"), as: .invalidFormat)
    rejected(line(action: "other"), as: .unknownAction("other"))
    rejected(line(user: String(repeating: "9", count: 30)), as: .invalidNumber(field: "user"))
    rejected(
        line(duration: String(repeating: "9", count: 30)), as: .invalidNumber(field: "duration"))
    rejected(line() + "\n", as: .invalidFormat)
    rejected(" " + line(), as: .invalidFormat)
    rejected("2024-02-29 action=login user=42 duration=0ms", as: .invalidFormat)
    rejected(String(repeating: "x", count: 513), as: .lineTooLong)
    rejected(String(repeating: "x", count: 512), as: .invalidFormat)
    let empty = try makeEventReport([])
    precondition(empty.events.isEmpty && empty.rejections.isEmpty && empty.durationByUser.isEmpty)
    let invalid = try makeEventReport(["bad", "also bad"])
    precondition(invalid.events.isEmpty && invalid.rejections.map(\.lineIndex) == [0, 1])
    let mixed = try makeEventReport([
        line(duration: "2"), "bad", line(user: "7", duration: "3"), line(duration: "4"),
    ])
    precondition(mixed.events.map(\.userID) == [42, 7, 42])
    precondition(mixed.rejections.map(\.lineIndex) == [1])
    precondition(mixed.durationByUser == [42: 6, 7: 3])
    do {
        _ = try makeEventReport([line(duration: String(Int.max)), line(duration: "1")])
        preconditionFailure("Expected aggregate overflow")
    } catch { precondition(error == .durationOverflow(userID: 42, lineIndex: 1)) }
}
try checkImporter()
```

**Walkthrough:** Calendar validation distinguishes leap-year exceptions from shape. Numeric overflow is classified by field, while malformed rows retain their original zero-based indices. The final sum test accepts each duration individually but rejects their unrepresentable total without returning a partial report. These tests do not establish a CPU-time bound, streaming memory bound, or a guarantee that cancellation interrupts regex work.

## References

### Related guides

- [[A5 — Pattern Matching and Exhaustive Control Flow]]
- [[A7 — Functions, Parameter Labels, Defaults, and API Surface]]
- [[A13 — Error handling and typed throws]]
- [[A14 — Operators, Literals, and Language Affordances]]
- [[B14 — Result Builders and DSL Construction]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C4 — String and Unicode correctness]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[D6 — `Sendable`, `@Sendable`, and `sending`]]
- [[F3 — Performance investigation habits]]

### Official sources

- [Regex](https://developer.apple.com/documentation/swift/regex)
- [SE-0350: Regex type overview](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0350-regex-type-overview.md)
- [SE-0351: RegexBuilder](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0351-regex-builder.md)
- [SE-0354: Regex literals](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0354-regex-literals.md)
- [SE-0357: Regex-powered string processing algorithms](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0357-regex-string-processing-algorithms.md)
- [SE-0363: Unicode for string processing](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0363-unicode-for-string-processing.md)
- [Sequence.compactMap](<https://developer.apple.com/documentation/swift/sequence/compactmap(_:)>)
- [LazySequenceProtocol](https://developer.apple.com/documentation/swift/lazysequenceprotocol)
