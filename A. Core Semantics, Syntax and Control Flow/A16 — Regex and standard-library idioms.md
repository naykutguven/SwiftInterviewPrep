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

**Goal:** Parse an explicitly ASCII user-ID field without accepting trailing input or hiding overflow.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

Parsing has separate stages: recognize grammar, convert fields, validate domain values, and report failures. Regex can establish shape but not Int representability or calendar validity. Whole matching consumes the entire input; prefix and search matching deliberately allow remaining text. Specify the grammar before choosing a literal or RegexBuilder. [Regex](https://developer.apple.com/documentation/swift/regex)

## Rules and pitfalls

### State character and rejection policies

ASCII [0-9] is narrower than Unicode digit matching. Decide whether whitespace, signs, leading zeroes, and trailing input are valid. A typed Regex output describes capture types, not universal Sendable conformance or successful numeric conversion.

**Common pitfall:** A digit match can still overflow Int. Check conversion and preserve the original row in an error report when an importer must account for every input.

### Choose collection transforms deliberately

Map produces one transformed output per input. CompactMap discards nil results; Sequence flatMap concatenates produced sequences. Reduce(into:) mutates one accumulator through scoped access. Lazy adapters defer work to consumption without memoizing repeated traversal.

**Common pitfall:** CompactMap over a parser silently loses rejected rows. Return a per-row outcome or partition successes and failures. Repeated lazy iteration can also rerun expensive or side-effectful transforms; materialize when stable reuse is required.

## Interview questions

### How do this literal and builder differ in accepted whitespace, digits, and trailing input?

Compare the written grammars rather than the notations. A literal using [0-9] and wholeMatch can reject whitespace, non-ASCII digits, and trailing input, while a builder using broader character classes or prefix matching may accept them. Equivalent grammars can be expressed either way; test the boundaries explicitly.

### Which failures become nil, which throw, and what would an importer need to account for every input row?

A shape mismatch commonly returns nil; numeric conversion can also return nil. Runtime pattern construction and some matching APIs can throw, while String matching conveniences may suppress matching errors. An importer needs an explicit rejection record per input, distinguishing shape, conversion, and domain failure when those distinctions guide recovery.

## Focused exercise

**Task:** Specify and parse an ASCII user-ID grammar. Check whitespace, trailing text, invalid digits, and integer overflow.

**Acceptance checks:**

- Valid ASCII digits parse; leading/trailing whitespace and text reject.
- Unicode digits reject even when they look numeric.
- A syntactically valid oversized number produces overflow failure.

**Solution:**

Standalone Regex-based parser using runtime construction, so no slash-literal flag is required. Grammar: exactly user: followed by one or more ASCII digits; no whitespace or sign. Leading zeroes are accepted.

```swift
enum ParseFailure: Error { case grammar; case overflow }
func userID(_ raw: String) throws -> Int {
    let grammar = try Regex("user:[0-9]+")
    guard try grammar.wholeMatch(in: raw) != nil else {
        throw ParseFailure.grammar
    }
    guard let value = Int(raw.dropFirst(5)) else {
        throw ParseFailure.overflow
    }
    return value
}
func probe() throws {
    let valid = try userID("user:42")
    precondition(valid == 42)
    for raw in [" user:42", "user:42x", "user:٤٢", "user:", "user:999999999999999999999999999999"] {
        do { _ = try userID(raw); preconditionFailure("unexpected acceptance") }
        catch is ParseFailure { }
    }
}
try probe()
```

**Reasoning:** Whole matching establishes exact shape before dropFirst uses the fixed prefix length. Int conversion establishes representability separately. Construct and reuse a validated grammar outside a production hot loop; this fixture prioritizes standalone dependencies and observable rejection policy.

## In practice

**Choose it when:** Use regex for a concise stable grammar and explicit parsing for richer recovery or structure.

**Investigate by:** Test every character-class and consumption boundary, then account for every batch input.

**Strong evidence:** Preserve rejection reasons and overflow policy rather than reporting only the successfully parsed subset.

### Complete importer and aggregate overflow — optional

#### Problem and policy

Parse one event per input String, retain classified rejected rows, and total duration by user. Input has exactly this grammar:

> YYYY-MM-DD user=ASCII_DIGITS action=LOWERCASE_ASCII_LETTERS duration=ASCII_DIGITSms

Each separator is one ASCII space. There is no leading/trailing whitespace or terminal newline; upstream framing removes line terminators. Input is limited to 512 UTF-8 bytes. Dates use the proleptic Gregorian calendar with years 1...9999 and valid month/day combinations. User IDs must be positive Ints; duration is a nonnegative Int. Actions are login, logout, or purchase. Leading zeros in numeric fields are allowed. This is a restricted log grammar, not a general date/time or localization API.

Do not force unwrap matched digits or infer calendar validity from punctuation. Do not discard rejected rows with compactMap when the import requires rejection accounting. Do not use unchecked += for arbitrary totals.

#### Parsing and domain validation

```swift
struct EventDate: Equatable, Sendable {
    let year: Int
    let month: Int
    let day: Int

    init?(year: Int, month: Int, day: Int) {
        guard (1...9999).contains(year), (1...12).contains(month) else { return nil }
        let leap = year.isMultiple(of: 400)
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
    let regex = #/([0-9]{4})-([0-9]{2})-([0-9]{2}) user=([0-9]+) action=([a-z]+) duration=([0-9]+)ms/#
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
        case let .failure(reason):
            rejections.append(EventRejection(lineIndex: index, reason: reason))
        case let .success(event):
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

```swift
func eventReportProbe() throws {
    let report = try makeEventReport([
        "2026-09-24 user=42 action=login duration=120ms",
        "bad line",
        "2026-09-24 user=7 action=purchase duration=450ms",
        "2026-09-24 user=42 action=logout duration=80ms"
    ])
    print(report.events.map { "\($0.userID) \($0.action.rawValue) \($0.durationMS)" })
    print(report.rejections.map(\.lineIndex))
    print(report.durationByUser.keys.sorted().map {
        "user \($0): \(report.durationByUser[$0, default: 0])ms"
    })
}
```

Output:

> ["42 login 120", "7 purchase 450", "42 logout 80"]
> [1]
> ["user 7: 450ms", "user 42: 200ms"]

Test valid leap days, century exceptions, missing/wrong/reordered labels, terminal newlines, Unicode lookalikes, zero IDs, unknown actions, oversized integers, the byte limit, empty/all-rejected batches, and per-user sum overflow. Verify rejected indices and successful order, not just the number of outputs. A literal and builder can both be good solutions if they implement the same declared grammar and failure policy.

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
- [Sequence.compactMap](https://developer.apple.com/documentation/swift/sequence/compactmap(_:)
- [LazySequenceProtocol](https://developer.apple.com/documentation/swift/lazysequenceprotocol)
