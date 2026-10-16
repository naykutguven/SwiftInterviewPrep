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

**Goal:** Truncate text by complete Characters and test combining and multi-scalar graphemes.

> **Baseline:** Swift 6.4, Swift 6 mode, Approachable Concurrency enabled, default isolation `nonisolated`. The additional opt-ins are `NonisolatedNonsendingByDefault` and `InferIsolatedConformances`. See [[Swift Senior & Staff Rubric and Prioritized Study Checklist#Shared Example Baseline|shared settings]] and [[F5 — Language mode, build settings, and migration flags]].

## Mental model

String's collection elements are Characters, modeled by extended grapheme clusters. Unicode scalars, UTF-8 bytes, UTF-16 code units, and rendered width are different units. Choose the unit required by the boundary rather than using count as a universal size. Canonically equivalent text can compare equal while having different encoded bytes. [String](https://developer.apple.com/documentation/swift/string)

## Rules and pitfalls

### State the unit and snapshot

Use Character traversal for a whole-grapheme prefix; use explicit encoding views for byte or UTF-16 contracts. Derive indices from the correct string snapshot and invalidate assumptions after mutation. Character count still does not measure visual width.

**Common pitfall:** Slicing a byte or UTF-16 offset can divide a scalar or grapheme. Converting NSRange to Range is only a first check; enforce your selection policy and grapheme boundaries on the actual snapshot.

### Validate before arithmetic and repair

Check nonnegative ranges and bounds without overflowing location + length. Valid String.Index values can represent positions inside a grapheme through encoding views; validity alone does not establish a UI selection boundary.

**Common pitfall:** Decoding a truncated byte prefix with replacement repair hides the invalid truncation decision. Select a valid prefix in the required unit, or reject malformed input explicitly. UTF8Span is a borrowed valid-UTF-8 view with its own lifetime and availability constraints.

## Interview questions

### Why can equal String keys have different encoded bytes?

Swift String equality uses canonical equivalence, so a precomposed character and a decomposed combining sequence can be equal while their UTF-8 bytes differ. Hashable String keys follow equality. Protocols requiring byte identity must compare the specified encoding or canonicalization policy rather than assuming textual equality means identical bytes.

### What else must be checked after converting a UTF-16 range, and why avoid location + length before validation?

Confirm that the range belongs to the current snapshot, is in bounds, and meets the required scalar or grapheme boundary policy. Validate the components and use subtraction-based checks before addition, because an untrusted location + length can overflow before a bounds comparison runs.

## Focused exercise

**Task:** Count and truncate text by a declared unit. Check combining marks, a flag, empty input, and an exact boundary.

**Acceptance checks:**

- A combining sequence and flag are not split at a one-Character boundary.
- Empty, zero/negative, exact, and larger limits have stated behavior.
- The test names Character units and makes no width or byte-budget claim.

**Solution:**

Standalone Character-budget implementation. Negative limits are defined as zero:

```swift
func preview(_ text: String, limit: Int) -> String {
    String(text.prefix(max(0, limit)))
}
let combining = "e\u{301}x"
let flag = "🇩🇪x"
precondition(preview(combining, limit: 1) == "e\u{301}")
precondition(preview(flag, limit: 1) == "🇩🇪")
precondition(preview("", limit: 1) == "")
precondition(preview("ab", limit: 2) == "ab")
precondition(preview("ab", limit: -1) == "")
precondition(combining.count == 2 && combining.utf8.count > 2)
```

**Reasoning:** prefix traverses Character boundaries and the returned String owns its result rather than retaining a Substring view indefinitely. A separate encoded-byte budget requires a different algorithm and validation policy; visually identical input can still occupy different bytes.

## In practice

**Choose it when:** Choose text units from the API contract, including Foundation range interoperability.

**Investigate by:** Compare Character/scalar/encoding views and test against the exact source snapshot.

**Strong evidence:** Preserve normalization, byte, and selection policies across boundaries without treating conversion as validation.

### Range conversion and byte-budget policies — optional

#### Convert ranges, then enforce the intended selection policy

The following API deliberately accepts only whole grapheme selections or insertions at grapheme boundaries. Some text-editing APIs intentionally work inside a cluster; this stricter policy should not be imposed on those APIs blindly.

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
          let range = Range(nsRange, in: text) else { return nil }

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

The subtraction-based bound check avoids overflowing location + length for malformed integer input. The explicit boundary test matters: on the tested macOS 27 Foundation, Range(NSRange(location: 1, length: 0), in: "e\u{301}") succeeds, although that position is inside the single Character. The observed conversion also accepts positions inside the tested flag's UTF-16 representation. Do not force-unwrap a conversion or infer grapheme alignment from success. The helper rejects these interior positions before slicing or replacement. [Range conversion API](https://developer.apple.com/documentation/swift/range/init(_:in:)-5qfor), [validation record](<../Swift 6.4 Validation Record.md>)

This check scans Character boundaries and favors an explicit teaching contract over a claim of constant-time validation. For many ranges, reuse a boundary map associated with an unchanged text snapshot if measurement justifies it. A valid range from an earlier document revision can point to the wrong content in a later revision; bounds checks cannot detect that semantic mismatch.

**Problem:** Return the original text when it fits. Otherwise return a prefix of complete source Characters plus a fixed ellipsis U+2026, with that ellipsis included in the limit. Zero returns an empty string. Negative limits are a caller error. This is a Character quota, not visual width or UTF-8 size.

```swift
func truncatedForDisplay(_ text: String, maxCharacters: Int) -> String {
    precondition(maxCharacters >= 0)
    guard maxCharacters > 0 else { return "" }
    guard let limit = text.index(
        text.startIndex, offsetBy: maxCharacters, limitedBy: text.endIndex
    ), limit != text.endIndex else { return text }

    let contentEnd = text.index(before: limit)
    return String(text[..<contentEnd]) + "…"
}
```

For `🇺🇸e\u{301}Hello`, limits 1, 2, 3, and 4 return `…`, `🇺🇸…`, `🇺🇸é…`, and `🇺🇸éH…`. The former chapter predicted the wrong output for limits 2–4. It also contained two competing implementations, one of which could form a negative prefix length when a configurable indicator exceeded the budget. This exercise now has one explicit fixed-indicator contract.

The limited index walk avoids an initial full String.count just to decide whether the text fits. A large limit returns the original when the end is reached; it does not allocate that many Characters. The implementation still traverses variable-sized clusters and allocates a result when truncating; this is not a measured performance claim.

Test empty, zero, one, exact-fit, oversized, decomposed accents, flags, ZWJ emoji, CRLF, leading combining marks, and long combining sequences. Check the result's scalar content as well as String equality: canonical equality alone could conceal a representation change. Use framework text layout when the requirement is fitting a width or line count. If the product needs a configurable indicator, specify its clipping and join behavior before extending this function.

#### Separate byte-budget policy

This alternative keeps the longest complete-Character prefix that fits a UTF-8 byte budget and appends no indicator:

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
```

For `🇺🇸e\u{301}B`, budgets 7, 8, 10, 11, and 12 yield empty, flag, flag, flag-plus-accented-e, and the whole text. It stops at the first Character that does not fit; it does not skip that Character and select later ones. This is one chosen truncation policy. A protocol may instead require rejecting the input or retaining a well-formed scalar prefix, so do not silently substitute it for a specified wire format.

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
- [Range conversion API](https://developer.apple.com/documentation/swift/range/init(_:in:)
- [The Swift Programming Language: Strings and Characters](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/)
- [implementation and decoding contracts](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/String.swift)
- [Character collection operations](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/StringCharacterView.swift)
- [UTF-16 view and NSString correspondence](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/StringUTF16View.swift)
- [Unicode UAX #29: Text Segmentation](https://www.unicode.org/reports/tr29/)
