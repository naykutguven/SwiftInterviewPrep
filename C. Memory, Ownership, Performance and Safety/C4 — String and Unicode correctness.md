---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - performance
  - safety
  - string
  - unicode
---

# C4 — String and Unicode Correctness

**Learning goals:**

- Choose Character, scalar, UTF-8, UTF-16, or layout units from the actual boundary contract.
- Preserve canonical equality, encoded representation, snapshot identity, and selection boundaries as distinct policies.
- Implement and verify complete-Character truncation, overflow-safe range validation, and explicit UTF-8 validation.

**Prerequisites:** [[A4 — Optionals and Nil Modeling]] for optional failure, [[A16 — Regex and standard-library idioms]] for text processing, and [[C5 — Collection Protocols, Complexity, and Index Invalidation]] for non-integer collection indices. The Foundation example uses NSRange.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

The visible text 🇺🇸é has two Swift Characters, but more scalars and encoded units. A **Character** is an extended grapheme cluster: a user-perceived text unit that may contain multiple Unicode scalars. A **scalar** is one Unicode value, and an encoding turns those values into UTF-8 bytes or UTF-16 code units.

Choose the unit before counting, selecting, or truncating. Character count is not byte length or rendered width. String equality also permits canonical equivalence: precomposed é and e plus a combining accent can compare equal while their bytes differ. [Strings and Characters](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/)

## Essential mechanics

### Example setup

Use the [[D1 — Structured Concurrency Fundamentals#Example setup and how to run it|D1 compiler command]] with this chapter's baseline flags. Each example is a separate Swift script; prepend only the named shared setup. Top-level statements are main-actor isolated. Function-only blocks marked **compile-only** use `-c -o /tmp/Example.o` instead of linking. Object compilation is important: type checking alone can miss transfer and noncopyable-ownership diagnostics. Rejected examples are checked separately from their corrections; examples with invalid memory access or data races are never run. Specified traps, where demonstrated, are isolated in separate processes as stated.

### Count the same text through four views

Standalone probe matching the rubric. Unicode escapes make the scalar sequence explicit.

```swift
func unicodeCounts(_ text: String) -> (Int, Int, Int, Int) {
    (text.count, text.unicodeScalars.count, text.utf16.count, text.utf8.count)
}
func unicodeProbe() {
    let text = "\u{1F1FA}\u{1F1F8}e\u{301}"
    let (characters, scalars, utf16, utf8) = unicodeCounts(text)
    precondition(characters == 2 && scalars == 4 && utf16 == 6 && utf8 == 11)
    print(characters, scalars, utf16, utf8)
}
unicodeProbe()
```

**Walkthrough:** It prints 2 4 6 11. The flag uses two regional-indicator scalars, each with a surrogate pair in UTF-16; the accent is a separate scalar in the second Character. A String.Index belongs to a text view/snapshot, not a universal integer character offset.

### Translate a known Character range into a Foundation range

Standalone Foundation example. NSRange expresses UTF-16 units for this API.

```swift
import Foundation

let text = "🇺🇸e\u{301}"
let first = text.startIndex..<text.index(after: text.startIndex)
let foundation = NSRange(first, in: text)
precondition(foundation.location == 0 && foundation.length == 4)
let restored = Range(foundation, in: text)
precondition(restored == first)
precondition(String(text[first]) == "🇺🇸")
```

**Walkthrough:** One Character maps to four UTF-16 units here. The conversion uses the same unchanged text in both directions. An arbitrary external range needs additional bounds, snapshot, and boundary checks before it can be a valid selection.

### Shared whole-grapheme range setup

graphemeRange implements a deliberately strict selection policy: endpoints must be Character boundaries, including insertion at endIndex. The initial guards validate components with subtraction before any unchecked location + length. The helper scans boundaries; it makes no constant-time claim. Some editing APIs intentionally permit interior positions and need a different policy.

```swift
import Foundation

func graphemeRange(
    _ nsRange: NSRange, in text: String
) -> Range<String.Index>? {
    let units = text.utf16.count
    guard nsRange.location != NSNotFound,
        nsRange.location >= 0, nsRange.length >= 0,
        nsRange.location <= units,
        nsRange.length <= units - nsRange.location,
        let range = Range(nsRange, in: text)
    else { return nil }

    func isBoundary(_ index: String.Index) -> Bool {
        index == text.endIndex || text.indices.contains(index)
    }
    guard isBoundary(range.lowerBound), isBoundary(range.upperBound) else {
        return nil
    }
    return range
}

func replacingGraphemes(
    in text: String, range: NSRange, with replacement: String
) -> String? {
    guard let range = graphemeRange(range, in: text) else { return nil }
    return text.replacingCharacters(in: range, with: replacement)
}
```

## Rules and pitfalls

### Successful NSRange conversion is not a whole-grapheme policy

**Common pitfall:** Treating any successful conversion as permission for a UI selection inside a combining sequence.

**Mistake:** Standalone runnable example.

```swift
import Foundation

let text = "e\u{301}"
let range = Range(NSRange(location: 1, length: 0), in: text)
let acceptedAsSelection = range != nil
precondition(acceptedAsSelection)
precondition(!text.indices.contains(range!.lowerBound))
```

**Consequence:** The selected Foundation accepts this position, but it lies inside the single Character. The example only inspects the converted index; it does not perform an invalid grapheme edit.

**Why:** Conversion and selection validation ask different questions. Positions from encoding views can be inside a Character, so conversion success alone is insufficient.

**Correction:** Apply the explicit boundary policy. Prepend graphemeRange/replacingGraphemes setup.

```swift
let text = "e\u{301}"
precondition(graphemeRange(NSRange(location: 1, length: 0), in: text) == nil)
precondition(replacingGraphemes(in: text, range: NSRange(location: 0, length: 2), with: "X") == "X")
precondition(graphemeRange(NSRange(location: 2, length: 0), in: text) != nil)
```

**What changes:** Interior endpoints reject, while a whole-grapheme replacement and end insertion are allowed. This is a chosen selection contract, not a universal restriction on every text system.

### Replacement decoding can conceal invalid byte truncation

**Common pitfall:** Cutting an encoded prefix at an arbitrary byte and repairing the malformed UTF-8 into a String.

**Mistake:** Standalone runnable example.

```swift
let text = "🇺🇸"
let bytes = Array(text.utf8.prefix(1))
let repaired = String(decoding: bytes, as: UTF8.self)
precondition(repaired == "\u{FFFD}")
precondition(repaired != text)
```

**Consequence:** The result contains a replacement character instead of a valid selected prefix of source Characters.

**Why:** One byte is not a complete encoding of the first scalar, and repairing decoding cannot restore the discarded bytes or the intended selection.

**Correction:** For this policy, keep the longest prefix of complete source Characters that fits the byte budget. This shared prefixFittingUTF8 implementation appends no indicator.

```swift
func prefixFittingUTF8(_ text: String, maxBytes: Int) -> String {
    precondition(maxBytes >= 0)
    var remaining = maxBytes
    var result = ""
    for character in text {
        let piece = String(character)
        let size = piece.utf8.count
        guard size <= remaining else { break }
        result += piece
        remaining -= size
    }
    return result
}
precondition(prefixFittingUTF8("🇺🇸", maxBytes: 1) == "")
precondition(prefixFittingUTF8("🇺🇸", maxBytes: 8) == "🇺🇸")
```

**What changes:** The undersized budget returns empty rather than repaired text. Other boundaries may require rejection or only scalar validity; do not silently substitute this complete-Character policy for a specified wire format.

## Worked example

### Equal text can still carry different bytes

**Setup:** Standalone canonical-equivalence example. Explicit scalars avoid dependence on an editor’s normalization.

**Predict:** Will the two strings make one textual dictionary key, and are their UTF-8 bytes identical?

```swift
let composed = "\u{E9}"
let decomposed = "e\u{301}"
precondition(composed == decomposed)
precondition(!composed.utf8.elementsEqual(decomposed.utf8))
precondition(composed.unicodeScalars.count == 1 && decomposed.unicodeScalars.count == 2)
var byText = [composed: 1]
byText[decomposed] = 2
precondition(byText.count == 1 && byText[composed] == 2)
```

**Outcome:** They are equal textual keys with different encoded bytes.

**Why:** String’s equality/hash contract follows canonical equivalence, while its encoding views preserve representation differences. A byte-identity protocol needs its own specified encoding/canonicalization contract.

## Interview questions

### Why can equal String keys have different encoded bytes?

**Answer:** String equality uses canonical equivalence. Hashable String keys follow that equality, while UTF-8 exposes the encoded representation. Equal text therefore does not establish byte identity.

**Example:** Standalone runnable example.

```swift
let first = "\u{E9}"
let second = "e\u{301}"
precondition(first == second)
precondition(Set([first, second]).count == 1)
precondition(Array(first.utf8) == [0xC3, 0xA9])
precondition(Array(second.utf8) == [0x65, 0xCC, 0x81])
```

**Walkthrough:** The Set has one textual member, while the byte arrays differ. Neither result is a hash collision argument: these Strings are equal under the actual equality contract.

**Boundary:** Text equality does not define the normalization policy for a file format, signature, identifier, or network protocol. Use that boundary’s required representation and do not infer exact hash integers.

### What must be checked after UTF-16 range conversion, and why avoid location + length before validation?

**Answer:** Check the current snapshot, nonnegative/bounded components, and the boundary policy the operation requires. Validate with remaining-length subtraction before addition, because an untrusted sum can overflow before a comparison runs.

**Example:** Prepend graphemeRange/replacingGraphemes setup.

```swift
import Foundation

struct Snapshot {
    let revision: Int
    let text: String
}
struct Selection {
    let revision: Int
    let range: NSRange
}
func replacing(in snapshot: Snapshot, selection: Selection) -> String? {
    guard selection.revision == snapshot.revision else { return nil }
    return replacingGraphemes(in: snapshot.text, range: selection.range, with: "X")
}
let current = Snapshot(revision: 2, text: "e\u{301}!")
for range in [
    NSRange(location: NSNotFound, length: 0),
    NSRange(location: -1, length: 1),
    NSRange(location: 0, length: -1),
    NSRange(location: Int.max, length: Int.max),
    NSRange(location: 1, length: 0),
] {
    precondition(graphemeRange(range, in: current.text) == nil)
}
let valid = NSRange(location: 0, length: 2)
precondition(replacing(in: current, selection: Selection(revision: 2, range: valid)) == "X!")
precondition(replacing(in: current, selection: Selection(revision: 1, range: valid)) == nil)
```

**Walkthrough:** Malformed/overflowing components and interior Character positions reject. The same numerically valid range also rejects when it belongs to an earlier revision. Bounds checking alone could not detect that semantic mismatch.

**Boundary:** The revision check requires the application to associate the range with its real source snapshot. Reusing a boundary map is valid only while that snapshot remains unchanged; the demonstrated helper scans Character boundaries.

### Extension: How can an ellipsis fit inside a Character budget without splitting source clusters?

**Answer:** Use one declared policy: return the original if it fits, otherwise keep complete source Characters and reserve one Character for fixed U+2026. Zero returns empty; negative input is a caller precondition failure.

**Example:** Standalone runnable example.

```swift
func truncatedForDisplay(_ text: String, maxCharacters: Int) -> String {
    precondition(maxCharacters >= 0)
    guard maxCharacters > 0 else { return "" }
    guard
        let limit = text.index(text.startIndex, offsetBy: maxCharacters, limitedBy: text.endIndex),
        limit != text.endIndex
    else { return text }
    return String(text[..<text.index(before: limit)]) + "…"
}
let text = "🇺🇸e\u{301}Hello"
let expected = ["…", "🇺🇸…", "🇺🇸e\u{301}…", "🇺🇸e\u{301}H…"]
for limit in 1...4 {
    let result = truncatedForDisplay(text, maxCharacters: limit)
    precondition(result.unicodeScalars.elementsEqual(expected[limit - 1].unicodeScalars))
    precondition(result.count == limit)
}
precondition(truncatedForDisplay("", maxCharacters: 1) == "")
precondition(truncatedForDisplay(text, maxCharacters: 0) == "")
precondition(truncatedForDisplay("ab", maxCharacters: 2) == "ab")
precondition(truncatedForDisplay("ab", maxCharacters: 20) == "ab")
```

**Walkthrough:** The one-Character result is only the ellipsis. Larger limits retain the listed source clusters without normalizing their scalar sequence. The limited index walk avoids an initial full count just to decide whether the text fits.

**Boundary:** This is neither visual width nor UTF-8 size. Layout requires the actual text framework. A configurable indicator needs a specified clipping/join policy; it must not be subtracted blindly from a smaller budget.

Prepend truncatedForDisplay. These additional runnable cases cover cluster shapes that simple ASCII tests miss:

```swift
for cluster in [
    "e\u{301}", "🇺🇸", "👩🏽‍💻", "\r\n", "\u{301}", "a" + String(repeating: "\u{301}", count: 20),
] {
    let source = cluster + "xyz"
    let result = truncatedForDisplay(source, maxCharacters: 2)
    let expected = cluster + "…"
    precondition(result.unicodeScalars.elementsEqual(expected.unicodeScalars))
}
```

### Extension: How is a complete-Character UTF-8 budget different from a Character quota or UTF8Span validation?

**Answer:** A byte budget constrains encoded length; a Character quota constrains grapheme count. UTF8Span validates well-formed UTF-8 and borrows those bytes, but does not impose either truncation policy or rendered width.

**Example:** Prepend prefixFittingUTF8 setup.

```swift
let text = "🇺🇸e\u{301}B"
let budgets = [7, 8, 10, 11, 12]
let expected = ["", "🇺🇸", "🇺🇸", "🇺🇸e\u{301}", text]
for (budget, wanted) in zip(budgets, expected) {
    let result = prefixFittingUTF8(text, maxBytes: budget)
    precondition(result.unicodeScalars.elementsEqual(wanted.unicodeScalars))
    precondition(result.utf8.count <= budget)
}
```

**Walkthrough:** The first flag consumes eight bytes and the accented e consumes three. A ten-byte budget stops before the accented e; it does not skip that Character and append B.

**Boundary:** Negative budgets are precondition failures and are not run. For arbitrary encoded input, validate rather than silently repair when malformed bytes must be rejected. The next independent example shows that operation.

Independent Apple OS 26+ UTF8Span example, run on the chapter’s macOS 27 baseline. The valid borrowed view remains inside its byte owner’s lifetime:

```swift
let bytes: [UInt8] = [0x65, 0xCC, 0x81]
let view = try UTF8Span(validating: bytes.span)
precondition(String(copying: view) == "e\u{301}")
let invalid: [UInt8] = [0xC3]
do {
    _ = try UTF8Span(validating: invalid.span)
    preconditionFailure("Expected invalid UTF-8")
} catch {}
```

Validation accepts the complete decomposed accent and rejects the truncated encoding. The view does not own storage or become an indefinitely retainable String. [UTF8Span](https://developer.apple.com/documentation/swift/utf8span)

## Focused exercise

**Task:** Count and truncate text by complete Characters under an explicit nonnegative-result policy.

**Acceptance checks:**

- Combining accents, flags, and a ZWJ emoji remain whole at a one-Character boundary.
- Empty, zero/negative, exact, and oversized limits have defined outcomes.
- Counts name their units; representation checks do not rely only on canonical String equality.

**Solution:** Standalone implementation. Negative limits are defined as zero for this simpler preview API:

```swift
func preview(_ text: String, limit: Int) -> String {
    String(text.prefix(max(0, limit)))
}
for cluster in ["e\u{301}", "🇩🇪", "👩🏽‍💻"] {
    let result = preview(cluster + "x", limit: 1)
    precondition(result.unicodeScalars.elementsEqual(cluster.unicodeScalars))
    precondition(result.count == 1)
}
precondition(preview("", limit: 1) == "")
precondition(preview("ab", limit: 0) == "")
precondition(preview("ab", limit: -1) == "")
precondition(preview("ab", limit: 2) == "ab")
precondition(preview("ab", limit: 20) == "ab")
let combining = "e\u{301}x"
precondition(combining.count == 2 && combining.utf8.count == 4)
```

**Reasoning:** prefix traverses Character boundaries and String produces an owned result rather than storing a long-lived Substring. This API deliberately differs from the stricter negative-limit precondition in the ellipsis/byte-budget examples. Neither contract measures width.

## In practice

**Choose it when:** Choose text units and normalization/selection policies from the consuming API rather than treating count as a universal size.

**Investigate by:** Test scalar and encoding views on the exact snapshot, including combining sequences, flags, ZWJ emoji, malformed ranges, and invalid bytes.

**Strong evidence:** Preserve representation and revision identity across boundaries, validate before arithmetic or slicing, and use text layout for actual width constraints.

## References

### Related guides

- [[A16 — Regex and standard-library idioms]]
- [[B8 — Equatable, Hashable, and Collection Correctness]]
- [[C2 — Copy-on-write behavior of standard-library types]]
- [[C5 — Collection Protocols, Complexity, and Index Invalidation]]
- [[C12 — `Span`, `InlineArray`, and modern low-level abstractions]]
- [[E2 — Foundation and Objective-C Bridging Behavior]]
- [[F3 — Performance investigation habits]]

### Official sources

- [String](https://developer.apple.com/documentation/swift/string)
- [Range conversion API](<https://developer.apple.com/documentation/swift/range/init(_:in:)-5qfor>)
- [The Swift Programming Language: Strings and Characters](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/)
- [implementation and decoding contracts](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/String.swift)
- [Character collection operations](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/StringCharacterView.swift)
- [UTF-16 view and NSString correspondence](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/StringUTF16View.swift)
- [Unicode UAX #29: Text Segmentation](https://www.unicode.org/reports/tr29/)
- [UTF8Span](https://developer.apple.com/documentation/swift/utf8span)
