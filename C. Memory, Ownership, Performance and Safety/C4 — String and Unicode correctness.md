---
tags:
  - swift
  - ios
  - interview-prep
  - memory
  - ownership
  - performance
  - safety
  - string
  - unicode
---
## 0. Rubric snapshot

**Rubric expectation**

Swift `String` is a collection of extended grapheme clusters, not bytes, UTF-16 code units, or Unicode scalars. The rubric explicitly tests whether you understand why apparent character count and storage representation differ, why integer indexing is not the right model, and how to truncate user-visible text safely.

**Caveats**

- Indexing is not integer-based.
- User-visible characters can be made from multiple Unicode scalars.
- UTF-8, UTF-16, Unicode scalar, and `Character` views answer different questions.
- Slicing by byte or UTF-16 offset can split a grapheme cluster and corrupt user-visible text.

**You should be able to answer**

- Why is `String` indexing not O(1) by integer offset?
- What bugs happen when engineers assume one user-visible character equals one Unicode scalar or one UTF-16 code unit?

**You should be able to do**

- Design a truncation function for user-visible text that does not split grapheme clusters.
- Predict the output of the Unicode count probe.

---

## 1. Core mental model

Swift’s default `String` abstraction is user-facing text. Its element type is `Character`, and a `Character` represents one extended grapheme cluster. A grapheme cluster is one or more Unicode scalars that should usually be treated as a single human-readable character. Swift’s documentation uses examples like `é`, Hangul syllables, and flags to show that one visible character can be composed from multiple scalar values. ([Swift.org, "Strings and Characters"](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/))

Unicode is layered. At the bottom, storage is bytes. Above that are encoding code units, such as UTF-8 or UTF-16 code units. Above that are Unicode scalars, which are code points excluding surrogate code points. Above that are grapheme clusters, which are the units Swift exposes as `Character`.

The important staff-level point: there is no single universal meaning of “character.” A database limit, an API payload limit, an SMS limit, a UI truncation limit, and a text cursor movement rule may need different units. Swift’s `String.count` answers “how many Swift `Character` values / extended grapheme clusters?” It does not answer “how many bytes?”, “how many UTF-16 units?”, or “how much rendered width?”

The key idea:

```text
String ≠ [Byte]
String ≠ [UTF16.CodeUnit]
String ≠ [Unicode.Scalar]
String == Collection<Character>
Character ≈ extended grapheme cluster
```

Swift guarantees that normal `String` iteration respects `Character` boundaries. It does not guarantee that all text operations are O(1), that a visual glyph equals one `Character`, or that external API offsets are compatible with `String.Index` without conversion.

---

## 2. Essential mechanics

### 2.1 `Character` is an extended grapheme cluster

A Swift `Character` can contain one or more Unicode scalars.

```swift
let composed = "é"          // U+00E9
let decomposed = "e\u{301}" // U+0065 + U+0301

print(composed.count)
print(decomposed.count)
print(composed == decomposed)
```

Output:

```text
1
1
true
```

Swift treats both strings as one `Character` and compares them as canonically equivalent text. This is why user-facing code should usually work with `Character` boundaries rather than raw scalar boundaries.

### 2.2 `String` has multiple views

The same text can be viewed as characters, Unicode scalars, UTF-8 code units, or UTF-16 code units.

```swift
let s = "e\u{301}"

print(s.count)
print(Array(s.unicodeScalars).count)
print(s.utf8.count)
print(s.utf16.count)
```

Typical output:

```text
1
2
3
2
```

Why?

```text
Character view:
  "é"         -> 1 Character

Unicode scalar view:
  "e" + "◌́"  -> 2 scalars

UTF-8 view:
  65 CC 81    -> 3 bytes

UTF-16 view:
  0065 0301   -> 2 code units
```

Swift exposes these views because different systems care about different units. For example, Foundation and many Apple APIs historically use UTF-16-based `NSRange`, while Swift-native APIs use `String.Index`.

### 2.3 `String.Index` is not an integer

Swift strings are variable-width and grapheme-aware. Moving to the “nth character” means walking character boundaries, not doing pointer arithmetic.

```swift
let text = "A🇺🇸e\u{301}B"

let index = text.index(text.startIndex, offsetBy: 2)
print(text[index])
```

Output:

```text
é
```

`index(_:offsetBy:)` for `String` is O(n), where `n` is the absolute offset distance. Apple documents this complexity directly for `String.index(_:offsetBy:)`. ([Apple Developer, "index(_:offsetBy:)"](https://developer.apple.com/documentation/swift/string/index%28_%3aoffsetby%3a%29))

### 2.4 UTF-16 offsets must be converted, not reused

When bridging from APIs that return `NSRange`, convert the range into a Swift `Range<String.Index>` using the original string.

```swift
import Foundation

let text = "Hi 🇺🇸 e\u{301}"
let nsRange = (text as NSString).range(of: "🇺🇸")

if let range = Range(nsRange, in: text) {
    print(text[range])
}
```

Output:

```text
🇺🇸
```

Do not treat `NSRange.location` and `NSRange.length` as Swift character offsets. They are UTF-16 offsets.

---

## 3. Common traps and misconceptions

### Trap 1: Assuming “one visible character” means one Unicode scalar

Bad:

```swift
func firstScalar(_ text: String) -> String {
    String(text.unicodeScalars.prefix(1))
}

print(firstScalar("e\u{301}"))
```

Output:

```text
e
```

This loses the combining acute accent.

Better:

```swift
func firstCharacter(_ text: String) -> String {
    String(text.prefix(1))
}

print(firstCharacter("e\u{301}"))
```

Output:

```text
é
```

### Trap 2: Treating UTF-16 offsets as Swift string offsets

Bad:

```swift
func substringUsingUTF16Offsets(_ text: String, location: Int, length: Int) -> Substring {
    let start = text.index(text.startIndex, offsetBy: location)
    let end = text.index(start, offsetBy: length)
    return text[start..<end]
}
```

This is wrong when `location` and `length` came from `NSRange`, because `NSRange` counts UTF-16 code units, not Swift `Character` values.

Better:

```swift
import Foundation

func substring(_ text: String, nsRange: NSRange) -> Substring? {
    guard let range = Range(nsRange, in: text) else {
        return nil
    }

    return text[range]
}
```

### Trap 3: Believing `String.count` is a storage-size check

Bad:

```swift
func fitsDatabaseLimit(_ text: String) -> Bool {
    text.count <= 255
}
```

This only checks grapheme cluster count. It does not check UTF-8 byte count, which is often what storage or network limits care about.

Better:

```swift
func fitsDatabaseUTF8ByteLimit(_ text: String, maxBytes: Int) -> Bool {
    text.utf8.count <= maxBytes
}
```

Use `text.count` for user-facing character limits. Use `utf8.count` for byte limits. Use `utf16.count` when interoperating with UTF-16 APIs.

---

## 4. Direct answers to rubric questions

### Q1. Why is `String` indexing not O(1) by integer offset?

Because Swift strings are collections of extended grapheme clusters, and grapheme clusters are variable-length. To find the nth `Character`, Swift must identify grapheme boundaries, which can require walking through the string rather than jumping directly by a fixed number of bytes or code units.

Interview version:

> Swift `String` doesn’t expose integer indexing because its element is `Character`, not byte or UTF-16 code unit. A `Character` can be made from multiple Unicode scalars, and those scalars can occupy variable numbers of code units. So “character 10” is not just base pointer plus 10. Swift uses `String.Index` to preserve valid character boundaries, and offsetting an index is O(n) for `String`.

### Q2. What bugs happen when engineers assume one user-visible character equals one Unicode scalar or one UTF-16 code unit?

They split characters, drop accents, corrupt emoji, miscalculate text limits, break cursor positions, mis-handle ranges from Foundation APIs, and produce invalid or surprising substrings.

Examples:

```text
"e\u{301}"      -> 1 Character, 2 Unicode scalars, 2 UTF-16 code units
"🇺🇸"           -> 1 Character, 2 Unicode scalars, 4 UTF-16 code units
"👨‍👩‍👧‍👦"       -> 1 Character, multiple scalars joined by ZWJ
```

Unicode Standard Annex #29 defines text segmentation boundaries for grapheme clusters and explains why user-perceived characters may consist of multiple code points. It also notes that grapheme clusters matter for UI interactions such as selection, cursor movement, deletion, and character counting. ([Unicode Consortium, "UAX #29: Unicode Text Segmentation"](https://www.unicode.org/reports/tr29/))

Interview version:

> The bug is choosing the wrong unit. If I slice by scalar, UTF-16, or byte offset, I can cut inside a grapheme cluster: for example separating `e` from its combining accent, splitting a flag emoji, or breaking a ZWJ emoji sequence. That creates broken UI text, wrong limits, invalid ranges, and mismatches with APIs that use different offset models. The fix is to use Swift `String.Index` and `Character` boundaries for user-visible text, and convert explicitly when interoperating with UTF-16 or byte-oriented APIs.

---

## 5. Code probe

Given:

```swift
let s = "\u{1F1FA}\u{1F1F8}e\u{301}"
print(s.count)
print(s.utf16.count)
```

### What happens?

```text
2
6
```

### Why?

The string contains:

```text
\u{1F1FA} \u{1F1F8} e \u{301}
```

Breakdown:

```text
Character view:

1. "\u{1F1FA}\u{1F1F8}" -> 🇺🇸
   - U+1F1FA REGIONAL INDICATOR SYMBOL LETTER U
   - U+1F1F8 REGIONAL INDICATOR SYMBOL LETTER S
   - Together they form one flag grapheme cluster.

2. "e\u{301}" -> é
   - U+0065 LATIN SMALL LETTER E
   - U+0301 COMBINING ACUTE ACCENT
   - Together they form one grapheme cluster.

s.count == 2
```

UTF-16 breakdown:

```text
U+1F1FA -> surrogate pair -> 2 UTF-16 code units
U+1F1F8 -> surrogate pair -> 2 UTF-16 code units
U+0065  -> 1 UTF-16 code unit
U+0301  -> 1 UTF-16 code unit

s.utf16.count == 2 + 2 + 1 + 1 == 6
```

So:

```text
Swift Character count: 2
UTF-16 code unit count: 6
```

### Fix or redesign

There is nothing to fix in the probe. It is demonstrating that different views count different units.

For user-visible truncation, use `Character` boundaries:

```swift
func truncatedForDisplay(
    _ text: String,
    maxCharacters: Int,
    truncationIndicator: String = "…"
) -> String {
    precondition(maxCharacters >= 0)

    guard text.count > maxCharacters else {
        return text
    }

    guard maxCharacters > 0 else {
        return ""
    }

    if maxCharacters == 1 {
        return truncationIndicator
    }

    let visibleCount = maxCharacters - truncationIndicator.count
    let prefix = text.prefix(visibleCount)
    return prefix + truncationIndicator
}
```

Example:

```swift
let text = "🇺🇸e\u{301}Hello"

print(truncatedForDisplay(text, maxCharacters: 2))
print(truncatedForDisplay(text, maxCharacters: 3))
print(truncatedForDisplay(text, maxCharacters: 4))
```

Output:

```text
…
🇺🇸…
🇺🇸é…
```

### Why this fix is correct

`prefix(_:)` on `String` works in terms of `Character` elements. It does not split the flag or the decomposed `é`. This matches the rubric’s requirement: truncating user-visible text without splitting grapheme clusters.

### Alternative fixes and tradeoffs

|Option|When it is appropriate|Tradeoff|
|---|---|---|
|`text.prefix(n)`|User-facing character limits|O(n); not a byte-size limit|
|`text.utf8.prefix(n)`|Protocols, storage, network payloads with byte limits|Can split characters unless you validate/reconstruct carefully|
|`Range(nsRange, in: text)`|Foundation/Objective-C interop using `NSRange`|Requires original string; conversion can fail if range is not on valid boundaries|
|TextKit / UIKit / SwiftUI layout truncation|Visual truncation based on rendered width|Depends on font, layout, locale, line breaking, and UI framework behavior|

---

## 6. Exercise

### Problem

Design a truncation function for user-visible text that does not split grapheme clusters.

### Bad / naive version

```swift
func naiveTruncateUTF16(_ text: String, maxUTF16Units: Int) -> String {
    let units = text.utf16.prefix(maxUTF16Units)
    return String(decoding: units, as: UTF16.self)
}

let text = "🇺🇸e\u{301}"
print(naiveTruncateUTF16(text, maxUTF16Units: 3))
```

### What is wrong?

```text
The function truncates by UTF-16 code unit count, not by Character count.

It can split:
- surrogate pairs
- flag emoji made from regional indicators
- combining character sequences
- zero-width-joiner emoji sequences

Even if String(decoding:as:) produces a valid String, the result may contain replacement characters or semantically broken text.
```

### Improved version

```swift
func truncateForDisplay(
    _ text: String,
    maxCharacters: Int,
    truncationIndicator: String = "…"
) -> String {
    precondition(maxCharacters >= 0)

    guard text.count > maxCharacters else {
        return text
    }

    guard maxCharacters > 0 else {
        return ""
    }

    let indicatorCount = truncationIndicator.count

    guard indicatorCount < maxCharacters else {
        return String(truncationIndicator.prefix(maxCharacters))
    }

    let contentCount = maxCharacters - indicatorCount
    return text.prefix(contentCount) + truncationIndicator
}
```

Usage:

```swift
let samples = [
    "Cafe\u{301}",
    "🇺🇸Welcome",
    "👨‍👩‍👧‍👦 family"
]

for sample in samples {
    print(truncateForDisplay(sample, maxCharacters: 3))
}
```

Possible output:

```text
Ca…
🇺🇸W…
👨‍👩‍👧‍👦 …
```

### Why this is better

It defines the limit in terms of user-visible `Character` values. It preserves grapheme cluster boundaries and keeps truncation semantics aligned with what Swift `String` normally means.

This is the right function for labels, profile names, preview titles, and UI-facing text limits.

It is not the right function for byte-constrained protocols or database columns. For those, write a separate API with explicit byte-limit semantics.

---

## 7. Production guidance

Use this in production when:

```text
- Displaying user-generated names, titles, comments, bios, or search snippets.
- Implementing user-visible character limits.
- Converting Foundation NSRange values into Swift ranges.
- Handling emoji, accents, multilingual text, or pasted text from arbitrary sources.
- Writing text-processing code where correctness matters more than ASCII-only convenience.
```

Be careful when:

```text
- Bridging to NSString, NSAttributedString, NSTextCheckingResult, regex APIs, or old Objective-C APIs.
- Receiving offsets from servers, databases, analytics pipelines, or search indexes.
- Enforcing limits that may mean bytes, UTF-16 units, Unicode scalars, grapheme clusters, or rendered width.
- Measuring performance of repeated String indexing inside loops.
```

Avoid when:

```text
- You are tempted to write string[i].
- You are using NSRange.location as a Swift character offset.
- You are truncating raw UTF-8 or UTF-16 and then assuming the result is valid user-visible text.
- You are using String.count to enforce byte-size constraints.
```

Debugging checklist:

```text
What unit does this API use: Character, Unicode scalar, UTF-8 byte, or UTF-16 code unit?
Did this range come from Swift-native code or Foundation/Objective-C?
Could the text contain emoji, combining marks, flags, Indic scripts, or ZWJ sequences?
Are we truncating for display, storage, transport, or indexing?
Are we accidentally doing O(n²) work by repeatedly offsetting from startIndex?
Can Range(nsRange, in: string) fail for this input?
```

---

## 8. Senior vs staff-level signal

### Basic answer

> Swift strings are Unicode, so you should not index them with integers.

### Senior answer

> Swift `String` is a collection of `Character`, where `Character` means an extended grapheme cluster. One visible character can contain multiple Unicode scalars and multiple UTF-16 code units. That is why `String.Index` exists and why user-visible slicing should use character boundaries.

### Staff-level answer

> I first clarify the unit the product or API actually needs. For UI truncation, I use grapheme cluster boundaries. For storage or network limits, I use bytes. For Foundation interop, I convert UTF-16 `NSRange` into `Range<String.Index>` using the original string. I also watch for performance traps: repeated integer-style offsetting over `String` can become O(n²). The design goal is to make the unit explicit in the API name so future engineers do not accidentally mix display semantics with storage or interop semantics.

Staff-level questions to ask:

```text
Is this a display limit, byte limit, UTF-16 interop limit, or rendered-width limit?
Where did the offsets come from?
Can the input contain arbitrary Unicode, or is it constrained?
Does the API name reveal the unit being counted?
Could repeated index offsetting make this algorithm O(n²)?
Do we need locale-specific or layout-specific behavior instead of plain grapheme clusters?
```

---

## 9. Interview-ready summary

Swift `String` is a collection of `Character`, and `Character` represents an extended grapheme cluster, not a byte, UTF-16 code unit, or Unicode scalar. That means `String.count`, `utf8.count`, `utf16.count`, and `unicodeScalars.count` can all differ. Integer indexing is not appropriate because grapheme clusters are variable-length and boundary detection requires walking the string. For user-visible operations like truncation, slice with `String.Index`, `prefix`, and `Character` boundaries. For Foundation interop, convert `NSRange` using `Range(nsRange, in:)`. For storage or protocol limits, be explicit about byte or UTF-16 semantics.

---

## 10. Flashcards

Q: What is the element type of Swift `String`?  
A: `Character`.

Q: What does Swift `Character` represent?  
A: An extended grapheme cluster: one or more Unicode scalars treated as one user-facing character.

Q: Why does `"e\u{301}".count` return `1`?  
A: Because `e` plus the combining acute accent forms one extended grapheme cluster.

Q: Why does `"🇺🇸".utf16.count` return `4`?  
A: The flag is two regional indicator scalars, and each scalar is outside the BMP, so each needs a UTF-16 surrogate pair.

Q: Why is `String.Index` not an `Int`?  
A: Because characters are variable-length and valid positions must respect grapheme cluster boundaries.

Q: What does `NSRange` usually count for Swift string interop?  
A: UTF-16 code units.

Q: How should you convert an `NSRange` into a Swift string range?  
A: Use `Range(nsRange, in: string)`.

Q: When should you use `text.utf8.count` instead of `text.count`?  
A: When enforcing byte-size limits for storage, networking, or protocols.

Q: What is the main correctness bug in truncating by byte or UTF-16 offset?  
A: It can split a grapheme cluster and corrupt user-visible text.

Q: What is the performance trap with repeated string offsetting?  
A: Repeatedly calling `index(_:offsetBy:)` from `startIndex` can turn a loop into O(n²).

---

## 11. Related sections

- [[A16 — Regex and standard-library idioms]]
- [[C5 — Collection protocols, complexity, and index invalidation]]
- [[E2 — Foundation and Objective-C bridging behavior]]
- [[F3 — Performance investigation habits]]

---

## 12. Sources

- [Project Notes, "Swift Senior & Staff Rubric and Prioritized Study Checklist"](<../Swift Senior & Staff Rubric and Prioritized Study Checklist.md>) — C4 String and Unicode correctness.
- Swift.org. "Strings and Characters." The Swift Programming Language. https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/
- Apple Developer. "String." Apple Developer Documentation. https://developer.apple.com/documentation/swift/string
- Apple Developer. "index(_:offsetBy:)." Apple Developer Documentation. https://developer.apple.com/documentation/swift/string/index%28_%3aoffsetby%3a%29
- Unicode Consortium. "UAX #29: Unicode Text Segmentation." https://www.unicode.org/reports/tr29/
