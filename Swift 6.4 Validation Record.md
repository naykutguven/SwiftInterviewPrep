# Swift 6.4 Validation Record

Last updated: 2026-09-27. All 70 chapter reviews and the whole-collection audit are complete. Individual sections preserve the dates and scope of earlier checks; they do not claim that every command was rerun on this date.

## Environment and limits

- Apple Swift 6.4: `swiftlang-6.4.0.34.1`, clang `2100.3.34.1`.
- Xcode 27.0, build `27A266a`, selected at `/Applications/Xcode_27.app/Contents/Developer`.
- Host: macOS 27.0, build `26A428`; arm64.
- SDKs: Xcode’s `MacOSX27.0.sdk` (via `MacOSX.sdk`) and iOS 27 (`iPhoneOS.sdk`) for Apple deployment-target compile-only checks.
- Baseline: `-swift-version 6 -default-isolation nonisolated`, no experimental/upcoming feature flags, default debug optimization.
- Temporary work directory: `/private/tmp/swift64-interview-validation`. Source, binaries, caches, and raw JSON/logs are temporary; the repository remains Markdown-only.
- No older compiler or older OS runtime was executed. An older deployment-target compile check is not a runtime test on that OS. A12 includes iOS 18-target compilation with the iOS 27 SDK. No iOS or non-Apple runtime was executed. Untested platforms and archive/distribution scenarios are coverage limits, not implied passing checks.

## Reproduction

Copy one standalone Swift fence into a temporary `Probe.swift`. Examples with shared declarations say which earlier declaration they use; add that context. Use a separate file for each intentionally invalid example.

```sh
probe_dir=$(mktemp -d)
probe_swiftc=/Applications/Xcode_27.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swiftc
probe_sdk=/Applications/Xcode_27.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk
# Save the chosen standalone example as "$probe_dir/Probe.swift" first.
"$probe_swiftc" -swift-version 6 -default-isolation nonisolated \
  -sdk "$probe_sdk" -module-cache-path "$probe_dir/ModuleCache" \
  -parse-as-library "$probe_dir/Probe.swift" -o "$probe_dir/Probe"
"$probe_dir/Probe"
```

Use `-parse-as-library` for examples with `@main`; omit it for top-level script examples. For compile-only probes replace the output step with `-c -o "$probe_dir/Probe.o"` and do not execute. Ownership errors can require SIL/ownership checking reached by `-c`, so `-typecheck` alone does not establish their absence. Run intentional traps in a separate process with a timeout and core dumps disabled; never combine them with a passing suite.

## Checked source/status boundaries

| Question | Evidence | Curriculum decision |
| --- | --- | --- |
| Is `-swift-version 6.4` valid? | Rejected; compiler lists `4`, `4.2`, `5`, `6`. | Separate compiler release from language mode. |
| Can async defer target an older runtime? | A minimal async-defer function compiled targeting macOS 13. | Syntax/lifetime change is separate from newer library availability; old-runtime execution not tested. |
| Can shields target macOS 13 without a guard? | Rejected as requiring macOS 27.0. Both SDK overloads carry `anyAppleOS 27.0`. | State availability in cleanup guidance. |
| Is noncopyable Continuation available? | SDK declaration plus success/error, negative compilation, and runtime-trap probes. | Teach with Apple OS 27 availability. |
| Are `Optional.ref` and public `RigidArray` available? | Minimal probes fail; installed Swift interface has no Optional `ref`/`mutableRef`/`put`, and uses internal `_RigidArray` under public `UniqueArray`. | Do not teach these proposed spellings as shipped baseline APIs. |
| Does ~Sendable suppress inference? | Generic Sendable-constrained call is rejected with an explicit-suppression note; a sending transfer compiles. | Separate type conformance from a particular safe handoff. |
| Is private-initializer compatibility retained? | A call to the old memberwise signature inside its type compiles. | Preserve concise migration/overload guidance. |

The Optional release announcement conflicts with the proposal/status report. Local compiler/SDK evidence above establishes absence in this tested distribution; it does not assert that all distributions or later patches are identical. The roadmap retains this distinction. Relevant sources: [SE-0532](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0532-optional-noncopyable-improvements.md), [SE-0527](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0527-rigidarray-uniquearray.md), [implementation report](https://forums.swift.org/t/implementation-status-update-for-recent-language-and-library-proposals/89295).

## Initial semantic probes

These 21 focused checks establish behavior used in the first editing batch. A pass for an intentional error means it failed for the expected reason. They do not certify all existing code fences.

| Probe | Outcome | Evidence |
| --- | --- | --- |
| `async_defer` | Compiled and ran | A6 ordering assertions passed; `body,async,outer,caught`. |
| `shield` | Compiled and ran | D2 cancellation observations passed across a yield. |
| `memberwise` | Compiled and ran | A3 `Profile(name:)` with private initialized fields printed `Ada`. |
| `memberwise_required_private` | Expected rejection | `Secret` initializer inaccessible. |
| `continuation` | Compiled and ran | D9 success/typed-error probe printed `42 missing`. |
| `async_result` | Compiled and ran | A13 captured `.missing`; printed `typed failure`. |
| `optional_ref` | Expected rejection | `Token?` has no member `ref`. |
| `rigid_array` | Expected rejection | `RigidArray` not found in scope. |
| `unsendable` | Expected rejection | Explicitly suppressed type does not satisfy Sendable. |
| `sync_defer_await` | Expected rejection | Async defer requires an async context. |
| `defer_return` | Expected rejection | Return cannot transfer control out of defer. |
| `continuation_double_resume` | Expected rejection during object compilation | Handle consumed more than once. |
| `continuation_dropped` | Expected isolated runtime trap | Exit signal 5; message: `Continuation was deinitialized without being resumed.` |
| `continuation_callback_gate` | Compiled and ran | Synchronous duplicate callback accepted first result, printed `1`. |
| `discarded_throwing_task` | Compiled with expected warning | `NoUseUnstructuredThrowingTask`. |
| `handled_throwing_task` | Compiled without diagnostics | Explicit `try await handle.value`. |
| `unsendable_transfer` | Compiled without diagnostics | D6 isolated boundary accepts `sending Local`. |
| `language_mode_6_4` | Expected rejection | Invalid language-mode value. |
| `defer_backdeploy` | Compiled targeting macOS 13 | Compilation only. |
| `shield_backdeploy` | Expected availability rejection targeting macOS 13 | Requires macOS 27.0. |
| `memberwise_old_signature` | Compiled without diagnostics | Compatibility signature remains callable within its type. |

The principal passing and negative sources are embedded in A3, A6, A13, D2, D6, D9, and D11. Additional small boundary probes used for the table are below.

### Missing APIs (separate compile-only probes)

```swift
struct Token: ~Copyable { var value: Int }
func inspect(_ x: borrowing Token?) {
    if let ref = x.ref { print(ref.value.value) }
}
```

```swift
func example() {
    let x = RigidArray<Int>(capacity: 2)
    print(x.count)
}
```

### Dropped-handle trap (isolated process only)

```swift
@main struct Main {
    static func main() async {
        let _: Int = await withContinuation { _ in }
    }
}
```

### Availability and compatibility probes

For `defer_backdeploy`, compile the A6 minimal `cleanup` / `performWork` example with `-target arm64-apple-macosx13.0`. For `shield_backdeploy`, compile this unguarded function with the same target:

```swift
func work() {
    withTaskCancellationShield { print("cleanup") }
}
```

For `memberwise_old_signature`, compile:

```swift
struct Profile {
    var name: String
    private var cache = 0
    static func seeded() -> Profile { Profile(name: "Ada", cache: 5) }
}
let value = Profile.seeded()
```

For `language_mode_6_4`, compile `let value = 1` with `-swift-version 6.4` instead of `6`.

## First-batch published-source checks

After the initial semantic probes, 15 additional checks extracted the new Swift fences directly from A3, A6, A13, D2, D6, D9, and D11. They reproduced the intended successful compilation, five runtime results, expected ownership/isolation errors, and discarded-task warning. Context was added only where a fence explicitly refers to an earlier helper. This checks that the teaching examples match the earlier probes; it does not certify all old fences in those chapters.

## Settings and migration review: D13, D14, F5

The focused settings suite has 13 checks. A further 47 checks used published chapter sources or their manifest fragments: 38 Swift compile/run configurations and nine manifest API compilations. All final outcomes match the documented expectations. Four of the published checks ran D14’s settings matrix; most other checks are compile-only, including intentionally unsafe designs that happen to compile.

| Area | Configuration / result |
| --- | --- |
| D14 execution matrix | Swift 6 mode; both default-isolation values; feature off/on. Plain nonisolated async prints `nil` without the feature and `Optional(Swift.MainActor)` with it. Explicit caller-bound and concurrent functions retain their respective results. |
| Default class inference | Calling an ordinary class method synchronously from a nonisolated function passes with nonisolated defaults and fails with MainActor defaults. |
| D13 migration diagnostic | Published ImageCache capture warns in Swift 5 complete checking; errors in Swift 6, including with `-strict-concurrency=minimal`. |
| Shared globals | Nonisolated library global is rejected. Explicit MainActor global passes when compiled as library code. Top-level executable variables have different rules. |
| Checked synchronization | D13’s immutable Mutex-backed wrapper compiles with checked Sendable, including a macOS 15 target. No unchecked conformance is necessary for that example. |
| F5 source warning control | Published deprecation adapter remains a warning with `-warnings-as-errors`; independent variants verify `ignored` and `error`. Annotating the scope of a native Swift 6 capture error does not suppress it. |
| Networking sketches | F5 class and actor variants compile under their stated isolation policies. No live network behavior, cancellation policy, or old-OS execution was tested. |
| Memory-safety settings | A separate UnsafePointer read warns under both `-strict-memory-safety` and `-strict-memory-safety:migrate` in this build. This confirms diagnostic scope/severity for the small probe, not memory correctness. |
| Manifest syntax/API | Two D14 manifests, F5’s complete manifest, and six F5 target fragments compile against the installed PackageDescription 6.4 API. This validates the settings spellings; it is not a package build. |

SwiftPM `dump-package` initially failed with `sandbox-exec: sandbox_apply: Operation not permitted`. Instead of treating that environment failure as a source error, the nine manifest files were compiled directly with the installed manifest API:

```sh
probe_manifest_api=/Applications/Xcode_27.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/pm/ManifestAPI
# Use probe_swiftc, probe_sdk, and probe_dir from Reproduction above.
# Save a complete manifest as "$probe_dir/Package.swift".
"$probe_swiftc" -swift-version 6 -package-description-version 6.4 \
  -sdk "$probe_sdk" -module-cache-path "$probe_dir/ModuleCache" \
  -I "$probe_manifest_api" "$probe_dir/Package.swift" \
  -c -o "$probe_dir/Manifest.o"
```

The extra memory-setting probe was `func read(_ ptr: UnsafePointer<Int>) -> Int { ptr.pointee }`, compiled separately with each flag and no runtime execution.

For target fragments, wrap them in `import PackageDescription` and a `Package(name: "Check", targets: [/* fragment */])` declaration with a 6.4 tools-version header. Application Image/Cart/Profile sketches, omitted legacy SDKs, and API-shape fragments were reviewed as sketches, not claimed as executable tests. D13’s duplicate capture example is the same source as its checked probe. Old full diagnostic transcripts were replaced with the fresh semantic outcomes above.

### Observed compiler limitation

The initial D14 probe below reached an LLVM verifier failure (`PHINode should have one entry for each predecessor`) during object compilation in this exact Apple Swift build:

```swift
nonisolated func plain() async -> Bool { #isolation != nil }
nonisolated(nonsending) func callerBound() async -> Bool { #isolation != nil }
@concurrent func offActor() async -> Bool { #isolation != nil }

@main struct Main {
    @MainActor static func main() async {
        print("plain:", await plain())
        print("callerBound:", await callerBound())
        print("offActor:", await offActor())
    }
}
```

It is not an intended language error. D14 uses `let current: (any Actor)? = #isolation` and prints that typed value; all four configurations then compile and run. The successful substitute establishes the recorded actor observations without relying on the crashing form. No upstream bug report was submitted.

## Availability review: A12

39 published-source configurations passed their expected checks: macOS 13 and iOS 18 deployment targets, normal and deliberate-invalid guards, the UTF-8 byte-count example, compiler/language conditionals, and module-present/module-absent factory examples. Three configurations ran on macOS 27; iOS checks were compilation only.

| Probe | Observed behavior |
| --- | --- |
| anyAppleOS guard around utf8Span | Compiles for both older deployment targets. On macOS 27, `"é🙂"` has six UTF-8 bytes through the modern branch. |
| `anyAppleOS 26.0, macOS 27.0` override | A shared 26.0 guard is insufficient on macOS 13, accepted targeting iOS 18. Adding the macOS 27.0 guard compiles. |
| anyAppleOS 25.0 declaration | Invalid-version warning in this build; fails with warnings as errors. This is not a supported pre-26 shorthand. |
| Compiler/language conditionals | In mode 6, compiler(>=6.4), swift(>=6), swift(>=6.4), and os(anyAppleOS) are all true on macOS. In mode 5, only the compiler and OS tests are true. |
| Existing iOS guard | The guarded iOS 26 function compiles targeting iOS 18; the unguarded call fails for the expected availability reason. |
| Optional framework factories | Section 5 and exercise branches compile with and without a temporary module named ModernTextFramework, on both Apple targets. The stub contains only `public enum Presence {}`; this proves source selection, not a real framework integration. |

The source conditional uses an effective language version, distinct from accepted command-line mode arguments. This agrees with the compiler’s [version mapping](https://github.com/swiftlang/swift/blob/main/lib/Basic/Version.cpp); it does not make `-swift-version 6.4` valid.

For iOS compilation, substitute `-sdk /Applications/Xcode_27.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk -target arm64-apple-ios18.0` and use `-c`. For macOS deployment checks add `-target arm64-apple-macosx13.0`. To test the module-present branch, compile the stub with `-emit-module -module-name ModernTextFramework -emit-module-path <temporary-directory>/ModernTextFramework.swiftmodule` for the matching target, then add `-I <temporary-directory>` to the example check. Do not reuse the macOS stub for iOS.

Older runtime fallbacks, older compilers, Linux, actual optional frameworks, and full networking behavior remain unexecuted. Deprecation wrappers with omitted domain models and signature-only API design sketches are explicitly labeled as partial examples.

## Ownership review: C7 and C8

22 focused semantic checks and 49 published-source configurations matched their final expectations. The latter cover all 46 Swift fences in C7/C8, plus three older macOS deployment checks; 15 configurations ran. Referenced helpers or minimal domain stubs were supplied only for explicitly partial examples. No real file descriptors or OS-resource adoption were tested.

| Area | Observed behavior / teaching decision |
| --- | --- |
| Borrow/mutate accessors | Stored noncopyable struct projection compiles without feature flags, including targeting macOS 13; runtime reads 7 then 8. Borrowing a local temporary or a class stored property through these accessor forms is rejected. |
| Ref and MutableRef | Noncopyable payload access, copied immutable references, and scoped exclusive mutation run successfully. Mutating the borrowed owner or reading it while a live MutableRef remains in use is rejected. These library APIs require Apple OS 27. |
| Copyable getter result | With `var values = [10]; let ref = Ref(values[0]); values[0] += 1; print(ref.value, values[0])`, the observed result is `10 11`. The reference can borrow a getter temporary; it need not track subsequent array mutation. |
| Consumption | Use after consuming a noncopyable value fails during object compilation. A consumed mutable binding can be reinitialized. A consuming Copyable parameter alone does not force caller invalidation. |
| Borrowed String passed to print | The original two examples failed object compilation because printing consumes an argument. They now say `print(copy value)` explicitly; runtime expectations pass. |
| Unique owners | UniqueBox mutation/consume and UniqueArray borrowing iteration/removal run successfully. UniqueBox.clone needs a Copyable payload; cloning a class reference preserves shared object identity. |
| Protocol support | Manually implemented noncopyable Hashable/Equatable conformance compiles, including targeting macOS 13. Set still rejects that noncopyable element. CustomStringConvertible conformance is rejected because this SDK still requires Copyable. |
| Associated types | A generic Holder accepting a noncopyable Item needs suppression on both Holder and its associated type at the use site. The consuming take example compiles. |
| Synchronous lock scope | Private noncopyable, explicitly non-Sendable token and checked-Sendable Gate compile. Throwing inside withLock followed by a second acquisition runs and prints 42. This checks the example's release path, not contention, fairness, or an escaping thread-affine token. |

SE-0499 describes broader string/stream protocol generalization than the installed interface provides. C8 teaches the supported subset and distinguishes conformance from container requirements. See [SE-0499](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0499-support-non-copyable-simple-protocols.md). This distribution-specific finding does not establish what every other 6.4 build ships.

An exploratory stored-property witness for a `borrow` protocol requirement returning a concrete noncopyable Token was rejected (`cannot witness borrow requirement`). The teaching example uses a checked consuming method instead. An unannotated function returning Ref from a local value was also rejected because its nonescapable result lacked an allowed lifetime source; this is not recorded as proof of a more specific escape diagnostic. Neither rejected experiment is required for the published examples.

## Span and inline-storage review: C12

15 focused configurations and 69 published-source configurations matched their documented outcomes. Every one of C12's 28 Swift fences was compiled for both macOS 13 and iOS 18 deployment targets, including the intentional failures. Thirteen additional configurations ran on macOS 27. Design-only SearchResult fragments used their stated shared model; the Header extension used the preceding Header declaration.

| Area | Observed behavior / scope |
| --- | --- |
| Core span availability | Indexed Span algorithms and withTemporaryAllocation compile for macOS 13 and iOS 18. InlineArray and Array.span require OS 26; the checked Iterable and explicit ByteOrder load operations require OS 27. |
| Borrowing iteration | InlineArray with noncopyable elements and a named Span view iterate without consuming their owner; output is 1, 2, 1, 2, 2. The unguarded operation is rejected targeting macOS 26. |
| Protocol boundaries | InlineArray does not satisfy Sequence. An ordinary Parser storing Span is rejected for its Escapable requirement. |
| Temporary storage | Capacity three with two appends reports count two and sums to 30. A thrown error destroys the one initialized noncopyable token before catch. Returning the span is rejected by the result's Escapable constraint. |
| Raw loads | An unaligned big-endian UInt32 field loads as 258. Loading Bool through the safe overload fails its ConvertibleFromBytes requirement. The native-order UInt32 overload compiles targeting macOS 13. |
| Parser bounds | Bytewise and RawSpan forms agree for nine offsets including Int.min, -1, the last valid field, past-end, and Int.max. Truncated and empty buffers return nil; no overflowing offset-plus-width calculation is used. |
| Owner mutation | Mutating UniqueArray while an earlier span is still used is rejected for overlapping access. |
| Existing examples | PNG signature checks, checksums, indexed bytes, fixed header, and HeaderParser outputs match; the missing InlineArray.append and ungated old-target Packet are deliberate compile failures. SHA-256 fragments are storage/traversal sketches, not a tested hash implementation. |

Two exploratory lifetime forms are kept outside the baseline examples. `for token in tokens.span` was rejected with a lifetime-dependent generator escape diagnostic in this build; introducing `let view = tokens.span` resolves it. A custom `Parser: ~Escapable` storing Span failed synthesized accessor/initializer checks, and an attempted `@lifetime` annotation produced an unsupported-use warning. The record does not present adding ~Escapable alone or enabling development features as a complete fix. The normal Span APIs above work without experimental flags.

For published checks, use the reproduction compiler command and save each Swift fence separately; add the stated shared declarations for fragments. Use the macOS/iOS target substitutions recorded under A12. The positive parser runtime harness compared offsets `[Int.min, -1, 0, 1, 2, 3, 6, 7, Int.max]` in `[0xFF, 0, 0, 1, 2, 3]`, plus truncated `[0, 1, 2]` and empty buffers. This is semantic/bounds validation, not a performance benchmark or an older-OS runtime test.

## Collection review: C5

10 focused checks and 75 published-source configurations matched their final expectations. The 27 Swift fences were each compiled for macOS 13 and iOS 18 deployment targets, followed by 21 macOS runtime configurations. Usage snippets include the preceding binarySearch definition; the UUID example now imports Foundation explicitly.

| Area | Observed behavior / scope |
| --- | --- |
| Borrowing visitor | UniqueArray<Token> and a named Span<Token> run without consuming elements; an array's named span also works. The generic signature retains throws(I.Failure), while these concrete inputs use Never. No custom failing iterator was executed. |
| Rejected visitor variants | Omitting for-try, the associated Element Copyable suppression, or the input Escapable suppression produces the expected error. Consuming a borrowed noncopyable loop element is rejected. |
| Array conformance | Array<Int> is rejected as a direct Iterable argument in this SDK. This does not affect its existing Sequence conformance or ordinary for-loops. |
| Hashability | Two dictionaries with the same keys and different values produce one entry in a Set of keys views. Duplicate CollectionOfOne and EmptyCollection values also produce one entry. Ungated use targeting macOS 26 is rejected. Hash integers and dictionary traversal order are not asserted. |
| Binary search | All five published variants pass searches in arrays of 0 through 20 elements, including absent and boundary targets; duplicates return a matching element. Published array/slice usages print index 3 and value 7. This validates results, not an empirical complexity benchmark. |
| Index and traversal behavior | Generic integer subscripting fails as expected. ArraySlice starts at 1. The stale array selection prints A after insertion; recomputing the index prints B. Both everyOther algorithms return the expected Character values for String and handle empty input. |
| Remaining algorithms | First/last, first-element, upper-middle, and generic search-result examples match expected outputs, including empty cases. The middle-element helper is named for its actual even-count behavior, not an arithmetic median. |

For the binary-search reproduction, create sorted input `(0..<count).map { $0 * 2 }` for each count in `0...20`; search targets `-1...41` and compare the result with firstIndex(of:) for this unique-element input. Separately check that searching `[1, 2, 2, 2, 3]` for 2 returns an index containing 2, without requiring the first duplicate. Both everyOther forms were checked against `"abcdef"` and empty arrays. The source and command conventions are the same as the chapter checks above.

## Unsafe-memory review: C6

74 published-source configurations matched their expected outcomes. All 27 Swift fences were compiled for both macOS 13 and iOS 18 targets. Eighteen configurations ran valid examples on macOS; two additional compile-only settings checks tested strict-memory-safety diagnostics. The deliberately invalid pointer escape, unproven alignment, layout reinterpretation, and retained-Data-pointer examples were never executed.

| Area | Observed behavior / scope |
| --- | --- |
| Raw-span initialization/load | Temporary six-byte capacity with a UInt8 prefix and big-endian UInt32 field reports five initialized bytes and loads 258 at offset one. It also compiles/runs with `-strict-memory-safety -Werror StrictMemorySafety`, without unsafe acknowledgements. |
| Pointer-escape probe | Compiles in the baseline configuration. Strict-memory-safety warns, and escalating its diagnostic group produces the expected compile failure. These diagnostics do not establish pointer correctness or specifically detect the escape. |
| Empty-buffer paths | Packet callback and both manual owners expose count zero safely. The optional pointer/count callback and Data first-byte operation return nil for empty input. No empty path force-unwraps or dereferences a base address. |
| Owned storage | ByteBuffer and OwnedRawBytes copy expected bytes, preserve the owner during callbacks, and propagate a callback error. These are byte-only wrappers; no claim of a general nontrivial-element allocator, sanitizer audit, or leak-profile run is made. |
| Binary parsing | The explicit little-endian Header parser returns length 16/kind 1 and throws for truncated input. The guarded unaligned UInt32 example prints 1. |
| Existing safe uses | Scoped byte/pointee reads, allocation/initialization, copied scalar results, and the Data checksum match their outputs. Raw UInt32 byte inspection printed 4, 3, 2, 1 on this little-endian arm64 host; that is not a portable serialized format. |
| Context-only sketches | Darwin socket-rebinding shape compiles but makes no socket call. The deliberately escaping Data sketch uses a temporary Data declaration and a no-op useLater signature for compilation only. |

Run the unsafe acknowledgement check by adding `-strict-memory-safety`, then separately `-Werror StrictMemorySafety`, to the reproduction command with object output. Do not execute the pointer-escape source. The valid raw-span example is the counterpart that also runs with those settings. For wrappers, check empty input and `[4, 5]`, then throw from the scoped body; note that an unsafe pointer can still be returned/stored by a misbehaving caller despite the callback contract.

## Exclusivity review: C3

80 published-source configurations matched their expectations: every one of 31 Swift fences compiled for macOS 13 and iOS 18, followed by 16 successful runtime checks and two deliberate dynamic traps. The trap processes ran separately with core dumps disabled and a timeout; both -Onone and -O reported Fatal access conflict detected.

| Area | Observed behavior / correction |
| --- | --- |
| Evaluated value argument | `modify(&numbers[0], using: numbers)` compiles and prints `[4, 2, 3]` in both checked optimization configurations. The previous claim of an inevitable exclusivity failure was incorrect: the value argument is evaluated before the inout access. |
| Static overlap | Aliased inout variables, two inout array elements, the clamp call using x three times, and a direct owner read during a live MutableRef borrow are rejected for their intended access conflicts. |
| Dynamic overlap | The global stepSize example needs an explicit MainActor function in Swift 6 to reach its intended runtime check. Without it, the earlier example stopped at an isolation error. Annotated source traps in both -Onone and -O. |
| Borrow/mutate projection | An inout mutation through a noncopyable Slot.token accessor prints 6 and compiles targeting both older Apple targets without feature flags. |
| Scoped reference | MutableRef mutation followed by a read after its scope prints 2; its guarded OS 27 API compiles for older targets. |
| Existing repairs | Distinct-variable swap, collection swapAt, aggregate Bounds normalization, immutable-bound clamping, and the remaining successful examples match their outputs. Signature-only sketches were compile-only. |

The abbreviated `add(&x, 1)`, `swapValues(&x, &x)`, and extension-Int examples used their preceding helper declarations. Reproduce the dynamic check from the standalone annotated stepSize fence, compile separately with -Onone and -O, and run each in a process with `ulimit -c 0`; the expected outcome is a nonzero exit with the exclusivity diagnostic, not successful execution. No disabled-exclusivity setting was used.

## Strict memory-safety review: C11

72 published-source configurations matched their expected outcomes, including nine runtime checks. The complete manifest compiled against PackageDescription 6.4. Each of the remaining 20 Swift fences compiled for macOS 13, compiled with strict-memory-safety warnings for iOS 18, and received a host check with StrictMemorySafety escalated to errors. The unacknowledged sum, escaping Packet view, pointer-storing AudioBuffer, and naive decoder are expected failures under the error policy; the acknowledged wrappers pass.

| Area | Observed behavior / correction |
| --- | --- |
| Diagnostic policy | Both -strict-memory-safety and -strict-memory-safety:migrate warn for the unacknowledged pointer sum. @safe/@unsafe declarations still need the appropriate implementation acknowledgements. Unnecessary markers were removed where this compiler diagnosed them; this is not evidence of correct pointer use. |
| Empty buffers | Checked first/last wrappers return nil for empty input and 8/9 for `[8, 9]`. They check count as well as pointer optionality. Invalid empty-input versions were compile-only. |
| Scoped callback | Packet returns copied bytes for empty and populated Data under the error policy. The callback can still escape an unsafe pointer if a caller violates its documented contract. |
| C checksum | A temporary C implementation sums bytes modulo UInt32 and defines empty input as zero. Both Swift wrapper forms tested return 0 and 6 for empty and `[1, 2, 3]`. |
| Opaque decoder | Actual imported forward-declared Decoder uses OpaquePointer, not UnsafeMutablePointer of Swift's Decoder protocol. The improved @safe, explicitly non-Sendable wrapper passes strict checking. |
| Decoder lifecycle | Temporary C fixture exercises empty/successful decode, error code 7 with copied `bad prefix` text, recovery, creation failure, and destruction. The fixture's live-handle count is zero after normal/error paths. This is not validation of a real decoder, concurrency behavior, thread affinity, or an external library's contracts. |
| Other coverage | Public AudioBuffer safe access and loadByte signatures compile. Raw-address read, unsafe storage designs, and parser API-shape sketches are compile-only; parser helpers were minimal stubs and no complete packet parser is claimed. |

The C fixture is temporary at `/private/tmp/swift64-interview-validation/safety-c-fixture`. Its header uses C11's shown signatures plus two fixture controls: `void fixture_fail_next_creation(void); int fixture_live_count(void);`. The implementation owns each Decoder with malloc/free, returns a null handle on an injected creation failure, accepts null bytes only for count zero, does not retain input, and returns a stable nullable C string. An input starting with 255 sets `bad prefix` and returns 7; all other input clears the error and returns zero. Those explicit contracts justify this teaching wrapper, not an unspecified decoder API.

To reproduce the import, create a temporary module map `module SafetyFixture { header "SafetyFixture.h" export * }`, compile the fixture implementation with Xcode's clang and the macOS SDK, and add `-I <fixture-directory>` plus its object file when linking the wrapper harness. Compilation-only iOS checks use the matching iOS SDK and do not link the macOS object. The final wrapper checks use `-strict-memory-safety -Werror StrictMemorySafety`. The harness decodes `[]`, `[1, 2, 3]`, `[255]`, and `[1]` in that order, then releases the wrapper and checks the live count; a separate creation-failure case checks the thrown error and zero live count. No framework package or permanent executable source was added.

## Mutability and property-wrapper review: A2 / B13

142 published-source configurations matched their expectations: all 55 concrete Swift fences received compilation checks for macOS 13 and iOS 18, including deliberate rejections, followed by 32 successful host runs. Expected failures remain failures: immutable-owner mutation, non-Sendable storage, actor-isolated access from a nonisolated wrapper, and unsafe global mutable storage. B13's generated expansions and omitted domain/framework designs are labeled pseudocode rather than claimed executable sources.

| Area | Observed behavior / correction |
| --- | --- |
| Borrow/mutate access | A noncopyable stored-value projection reads 1 then mutates to 2. Mutation through a let owner is rejected. A simple copyable generic property wrapper prints 1 without feature flags. |
| Reference-backed mutation | Let-bound wrappers can change their referenced storage, including through a copied outer value; the shared-state probes produce their documented outputs. This does not establish Sendable or meaningful synthesized equality. |
| Initializer access | Private initialized Uppercased cache is omitted and Profile(name:) prints SEED. A required private wrapped value makes the initializer inaccessible; a cache argument is rejected even inside a static factory in the initialized-wrapper case. This omission predates Swift 6.4. |
| Validation policy | Validated permits an invalid initial empty string, accepts Ada, and ignores a later empty assignment. The clamping wrappers consistently process construction and assignment for the tested Int values. Comparable alone does not define a NaN policy. |
| Property observers | Direct Profile(age: 200) initialization skips didSet; later assignment clamps/logs. A separate initializer that calls a mutating helper after initialization prints observed 2, then 2, confirming that observers are not universally disabled throughout an initializer's execution. |
| Isolation/synchronization | An explicitly MainActor owner of MainActorBacked prints Ready. Ten task-group children perform 100 synchronized increments each, producing 1000 through a Mutex-backed Sendable Counter. The whole update is inside one lock scope. This bounded check is not a general concurrency proof. |
| Other published examples | Value snapshots, reset methods, wrapper logging/projections, and three actor storage harnesses match their expected values. Signature-only protocol/conformance sketches are compile-only. |

Seven exploratory checks covered the new wrapper, private initializer variants, shared storage, and observer behavior. Six produced the expected semantic outcomes. The seventh, a noncopyable generic property-wrapper variant, crashed swift-frontend with signal 11 during compilation on the recorded build. This is a compiler failure, not a successful negative language test, and the variant is excluded from supported examples. Its source uses Borrowed<Value: ~Copyable>: ~Copyable, consuming wrappedValue initialization, borrow/mutate accessors over its stored Value, and a noncopyable Settings containing @Borrowed var count = Token(id: 0); mutating settings.count.id triggers the crash. No bug report was filed, and no claim is made about other compiler builds or all noncopyable wrapper designs.

Reproduce published checks using the baseline commands above. Usage-only fences use the immediately referenced Counter, Preferences, MainActorBacked, or Clamped declaration. The three actor harnesses use an async main and -parse-as-library. The Mutex harness runs ten child tasks with 100 increment calls each and checks read() after the group completes. Keep compiler-crash exploration separate from the successful/rejected published-source matrix.

## Initialization review: A3

82 published-source configurations matched their expectations: 31 Swift fences checked for macOS 13 and iOS 18, plus 20 successful host runs. Expected rejections cover subclass storage ordering, premature whole-instance use, missing memberwise synthesis, private required fields, and convenience-initializer assignment. The combined bad hierarchy exercise first diagnoses direct let initialization in the convenience initializer; its designated-initializer problem is independently exercised in the simpler hierarchy probes.

| Area | Observed behavior / correction |
| --- | --- |
| Stored versus computed access | Pair initializes first, then reads that field to initialize second, printing 2 3. A separate computed-property variant fails because self is not fully initialized. The corresponding direct stored-field pattern also succeeds before super.init in the focused class probe. |
| Memberwise changes | Private initialized and implicitly nil-initialized fields can be omitted; the required private field remains inaccessible. The published ordinary-property compatibility example prints 0 7. A bare initializer reference selects the shorter form in the checked case, while an extension declaring that signature is rejected as a redeclaration. |
| All-private storage | An explicit private-field argument is not exposed to an external caller. A separate no-argument default initializer must not be confused with access to the memberwise signature. |
| Observers | Direct initial assignment skips didSet. A mutating helper invoked after initialization prints observed 2, then 2. A subclass assignment after super.init prints base observed 2, then 2 in a separate focused probe. |
| Construction paths | Corrected subclasses, convenience delegation, normalization helpers, defaults, extension initialization, and a deterministic timestamp example produce their expected fields. Failable and throwing validation cases cover valid input, empty/invalid input, and the documented error cases. PaymentRequest validates only the shown constraints, not full payment/currency semantics. |

Nine focused checks add six successful runs and three intended rejections for stored/computed access, observer behavior, initializer overloads, and private argument access. Foundation imports were supplied in the published Foundation-dependent examples. The three abbreviated Resource subclasses use the preceding Resource declaration during checks. Older full compiler transcripts were replaced with the fresh semantic result; no old compiler, external-client module, UIKit/AppKit subclass lifecycle, or complete domain-validation policy was tested.

## Cleanup review: A6

61 published-source configurations matched their expectations: 21 concrete Swift fences checked for macOS 13 and iOS 18, plus 19 successful host runs. Return from a defer and async defer inside a synchronous function remain intentional compile failures. Seven application/resource fragments with omitted dependencies are labeled sketches. The shield example carries its OS 27 guard; the scoped Mutex example carries macOS 15/iOS 18 and corresponding Apple-platform availability.

| Area | Observed behavior / correction |
| --- | --- |
| Synchronous control flow | LIFO and nested ordering, return/throw paths, loop iteration scope, an unreached defer, and Int return evaluation match the recorded traces. |
| Async scope exit | The existing success/throw probe prints body,async,outer,caught. The new nested early-return function prints body, second, first, outer before its caller prints returned. Cleanup is awaited sequentially in reverse registration order. |
| Temporary files | Both one-byte implementations read empty input as nil and a byte 42 as Optional(42). Missing input produces openFailed with ENOENT; reading a directory produces readFailed with EISDIR on this host. The fixtures live only in the temporary validation directory. |
| Error/cleanup policy | The improved source captures errno into the thrown value before deferred close. The naive source happens to preserve errno in this run; that does not establish a contract for reading errno after another system call. Interrupted reads, close failures, and resource-leak instrumentation were not exercised. |
| UI/locking examples | The MainActor loading-state sketch restores false; it contains no real service call. The NSLock and Mutex caches return nil for missing keys. The checks do not validate overlapping loads, lock contention, or external I/O. |

The two abbreviated async-defer examples use the preceding cleanup declaration. Async entry harnesses use -parse-as-library; the standalone @main ordering probe already includes its entry point. The temporary-file harness passes an empty file, a file containing byte 42, a nonexistent path, and a directory to firstByte. Existing cancellation probes above establish cancellation observation separately; these new runs do not expand that cancellation matrix. Fatal traps and process termination are outside the ordinary scope-exit guarantee.

## Error-handling review: A13

58 published-source configurations matched their expectations: 20 Swift fences checked for macOS 13 and iOS 18, plus 18 successful host runs. The two wrong-error-type examples are deliberate rejections. Thirteen signature-only or omitted application designs are labeled sketches. Earlier output/diagnostic claims were rerun; they were not relabeled without validation.

| Area | Observed behavior / correction |
| --- | --- |
| Typed failure and Never | The minimal example prints LoginError missingEmail, nil, and 42. Throwing NetworkError from throws(ParseError) is rejected. The wrapper correction preserves its underlying NetworkError. |
| Async Result | The typed failure, cancellation, and noncopyable success examples run as documented and compile for both older deployment targets. Result's async initializer appears in the installed _Concurrency interface with explicit nonisolated(nonsending) behavior and @_alwaysEmitIntoClient. |
| Closure inference | Removing the explicit async throws(Failure) annotation from the focused Result example fails: the inferred any Error cannot convert to Failure. No experimental inference feature was enabled. |
| Cancellation | The published probe captures CancellationError while Task.isCancelled remains true. A focused variant omitting the cancellation check returns success 42 from the same cancelled task; the initializer does not automatically reject cancelled work. |
| Caller isolation | A focused MainActor state example accesses its actor-owned field inside the async Result operation across a yield and prints 1. This does not constrain the executors of other functions called by that operation. |
| Catch exhaustiveness | Separate enum-case catch clauses are rejected as non-exhaustive in a nonthrowing function. A catch is Failure pattern is accepted with an always-true-test warning. The unconditional catch plus switch example prints missing email. |
| Rethrows and transforms | Independent failure in a rethrows function is rejected. Plain throws, rethrows, and generic typed-throws transforms produce the expected doubled array; typed and erased failures preserve the documented handling behavior. The helper allowing zero is now named requireNonnegative. |
| Validation example | Missing/invalid email and short password reach the three declared errors; the sample accepted input succeeds. These checks do not establish a complete email/password policy or exercise a network/UI flow. |

Eight focused checks complement the published matrix: seven initial compile checks (three rejection outcomes, one accepted type-test warning, and three successful runs), followed by the cancellation-ignoring variant's successful run. Usage-only fences receive the preceding validate or transform declaration. All standalone async probes use -parse-as-library. The safe try! call is checked only on valid input; no intentional try! trap or external service was executed. Foundation is explicitly imported for the UUID example.

## Task lifetime, cancellation, and reentrancy review: D1 / D2 / D7

83 published-source configurations matched their documented expectations, including 19 host runs: D1 has 38 configurations, D2 has 11, and D7 has 34. Every concrete Swift fence has a compilation disposition. Portable examples target macOS 13 and iOS 18; the three UIKit cell/pipeline examples target iOS 18 only. Seven additional compilation configurations use default MainActor isolation with NonisolatedNonsendingByDefault for the adapter, observable model/view, task-local/shield probes, cleanup actor, and shared-work cache. This is a selected-settings check, not the complete four-way matrix for every example.

| Area | Observed behavior / correction |
| --- | --- |
| Basic async calls | Sequential calls print first, second, 3. Both synchronous-await counterexamples are rejected. Foundation's Thread.sleep is rejected from an async context; cancellation-aware Task.sleep throws for the cancelled harness. The explicit boundary task returns a handle and handles errors. |
| Structured scope exit | The unawaited async-let probe prints parent body, child observed cancellation, parent returned. Child work cooperates; this is not a guarantee that arbitrary children terminate promptly. |
| Callback adapter | A Sendable legacy protocol with a Sendable callback and Sendable DTO compiles; controlled exactly-once callbacks produce Ada and the expected missing error. No callback cancellation API is invented. |
| Observable state | The MainActor @Observable model and SwiftUI owner compile with explicit availability. Controlled callbacks verify that an older success, an older failure, and cancellation followed by an ordinary failure cannot overwrite newer state, even for repeated identical IDs. The view itself was not rendered or lifecycle-driven in a simulator. |
| Task-local values | The structured child and ordinary Task see request-42; detached work sees nil; leaving the binding restores nil. The context namespace is explicitly nonisolated to keep it usable under both tested default-isolation settings. |
| UIKit lifecycle | Naive and repaired cells compile against the iOS SDK. The repaired source uses per-configuration identity, cancellation checks for success/error, guarded handle cleanup, and deinit cancellation. The pipeline rechecks its cache after suspension. No UIKit runtime, image decoding, HTTP response, cell layout, or real network was exercised. |
| Actor counter | The original sleeping driver accepts either 1 or 2; the result is scheduling-dependent. Revalidation and pre-suspension reservation each produce 1. The sleep's try? discards cancellation only for this demonstration. |
| Async cleanup ownership | The one-use gate forces a newer operation to start during old teardown. Published guarded cleanup prints finishing 1 then Optional(2); the temporary variant without the identity check prints nil. This distinguishes scope-exit waiting from uninterrupted actor execution. |
| Cache overwrite/revalidation | Injected downloads complete request 2 before request 1. The naive cache returns [1], [2], then caches [1]; revalidation returns/caches [2] for both. Both make two fetches. No network timing is involved. |
| Shared producer | Two callers receive [7] from one fetch, including a cancelled consumer, and a cached read adds no fetch. A failing producer clears its slot; the next call successfully retries with [9]. This implements waiting for shared completion, not prompt consumer cancellation. |

Four focused runtime configurations supplement that matrix: two external cancellation-handler controls, one shield/child observation suite, and one regular throwing-group suite. The latter catches a first child error inside the group body and observes both group.isCancelled and the remaining child's cancellation state as false. Letting the observed error escape the body instead cancels and waits for the sibling before the outer catch runs. This is not a test of every throwing/discarding-group operation.

### Cancellation shield nesting: observed limit

With an outer cancellation handler installed, the test enters a shield, installs an inner handler, and suspends at a controlled continuation gate. A separate task waits for that gate, cancels the worker handle, then releases the gate. In the unshielded control, both handlers fire and the operation sees cancellation. With the shield, neither handler fires; the operation sees false, the task handle reports true, and the worker observes true after leaving the shield. The self-cancelling variant also records no handler invocation. An initial assertion expecting the outer handler to run failed; the notes now report the observed result instead of presenting that assumption as a guarantee.

This is recorded behavior of macOS 27.0 build 26A428 with this toolchain, not a claim about every runtime or nesting arrangement and not a diagnosed runtime bug. SE-0504 explicitly protects handlers registered under shielding; code requiring an outside handler to react during a shield needs its own runtime validation. No external bug report was filed.

The same shield suite confirms that Task.checkCancellation does not throw inside the shield but throws afterward, an enclosed awaited async-let child is not automatically cancelled by the parent, explicit group.cancelAll still cancels its child, and shielding only group.addTask leaves a child of an already-cancelled group cancelled. An unawaited async-let child inside a shield is still cancelled and awaited at scope exit. Core dumps were disabled for the final focused checks.

### Reproduction and environment notes

D1's short usage fences use the preceding fetch/User helpers; the adapter/model/view form one explicitly referenced chain. The naive D2 cells use the pipeline declaration from the complete exercise. D7's driver uses its preceding Counter and -parse-as-library; its network call-site example is compile-only.

The D1 callback fixture stores Sendable callbacks in Mutex-protected storage, signals registration through an actor, completes newer calls first, and then delivers older success/error or a cancellation-followed-by-error. The cache fixtures use actor-owned continuations to control fetch start/completion. A host-only helper uses Task.immediate (OS 26+) from the cache actor to enter the second consumer before releasing the shared producer; this test-helper requirement is not added to the published cache's deployment requirements. Each fake callback/continuation completes exactly once, and no fixture calls a network service.

Xcode's bundled Observation and TaskLocal macro processes initially failed with sandbox_apply: Operation not permitted inside the outer sandbox. The affected temporary compiler checks were rerun with approved outer-sandbox execution so those SDK processes could run their own sandbox; all passed. This was an environment failure, not a source diagnostic. The repository still contains no executable validation harness or build configuration.

## Task-group policy and bounded concurrency review: D3

All 39 published-source configurations passed their documented expectations: 12 concrete Swift fences compiled for both macOS 13 and iOS 18 (24 checks), four selected helpers compiled under default MainActor isolation plus NonisolatedNonsendingByDefault, and 11 host executions checked behavior. Twelve domain/API fragments are explicitly labeled sketches. These counts include the two intended diagnostic examples; a rejection is a pass only when the expected semantic diagnostic appears.

| Example | Observed result |
| --- | --- |
| Invalid async-let dependency | Capturing one async-let binding from another initializer is rejected. Await into an ordinary value first. |
| Synchronous async-let context | The declaration/read are rejected because the enclosing function does not support concurrency. The corrected async function prints 1. |
| Sequential loop / dynamic group | The immediate-await loop prints start/end for 1, then 2, then 3. The group collects and sorts [10, 20, 30]. The loop now yields instead of relying on a delay. |
| Home-data aggregation | Sequential and async-let fixtures produce the documented HomeData value. No network operation or latency measurement is involved. |
| Async-let failure cleanup | The profile error is observed first; the cooperative sibling prints recommendations cancelled before the outer catch prints profileFailed. There is no timer assumption. |
| Bounded ordered map | Controlled completions arrive in order 1, 2, 3, 0; output is [0, 10, 20, 30]. Four transforms run with a measured peak of two pending operations. A limit-one run preserves optional nil elements; valid empty input returns empty, and limits zero/-1 throw invalidLimit. |
| Map cancellation and error | Cancelling after the first two operations start prevents replacements; both initial continuations are released and CancellationError propagates. Already-cancelled entry starts no transform. An observed child error cancels/waits for its cooperative sibling. |
| Partial-data fallback | Ordinary optional-request errors produce empty/zero fallback while the profile remains required. Explicit CancellationError, a service error after self-cancellation, a successful value after self-cancellation, and cancellation before entry remain cancellation failures. A profile error still fails the aggregate. |

Six further focused configurations passed, including one host run. The behavioral suite also compiled for macOS 13, iOS 18, and default MainActor plus NonisolatedNonsendingByDefault. Two negative availability configurations confirm unguarded discarding-group calls require macOS 14 / iOS 17. Guarded calls compile for older targets; execution occurred only on macOS 27.

The focused runtime observations were:

- Returning 7 from a regular throwing-group body without collecting a failed child still returns 7; a sibling observes false cancellation.
- An unobserved throwing async-let child does not automatically make its enclosing nonthrowing async function throw.
- waitForAll catches the child failure only after the group is empty; group cancellation and the sibling's observed cancellation are false.
- nextResult collects one failure and one successful false cancellation value; the group remains uncancelled.
- After cancelAll, addTaskUnlessCancelled returns false, while addTask still creates a child that observes true cancellation.
- A throwing discarding group's failure cancels its cooperative sibling and waits for that sibling before the outer catch prints the error.

All final positive configurations were free of compiler warnings. The settings checks cover selected examples rather than every configuration combination. Existing D2 shield tests supply the linked scope-exit/explicit-cancellation evidence; no new network, UI, non-Apple runtime, or older-OS execution claim is made.

### Reproduction notes

Use the toolchain, SDK, module cache, Swift 6 language mode, and explicit default isolation recorded above. Compile standalone @main probes with -parse-as-library; use -c for deployment-only and expected-diagnostic checks. The short good() fence uses work() from the preceding counterexample, the sequential HomeData fence uses the improved example's data/fetch helpers, and the partial-data fence uses only that example's Sendable HomeData declaration.

The temporary map fixture owns checked continuations in an actor. It waits until two transforms are registered, resumes indices 1 then 2 while waiting for replacements, and finally resumes 3 then 0. It records starts/peak and releases every continuation exactly once. The cancellation variant cancels after two registrations, releases both original continuations, and verifies no additional registration. Separate temporary tasks keep self-cancellation cases isolated from the runner. Published helper bodies are used unchanged.

## Actor ownership and MainActor review: D4 / D5

103 published-source configurations matched their expectations, including 29 host runs: D4 has 62 configurations / 20 runs; D5 has 41 configurations / 9 runs. All 32 concrete Swift fences have a compilation disposition for macOS 13 and iOS 18 (64 checks), with 10 additional MainActor-default plus NonisolatedNonsendingByDefault configurations. Expected diagnostics are checked separately from positive sources. Twenty incomplete application/API fragments are explicitly labeled sketches.

| Area | Observed behavior / correction |
| --- | --- |
| Basic actor boundaries | Counter/cache/account examples produce 1, [1, 2, 3], and 100. The synchronous isolated-property access and retained non-Sendable profile crossing are rejected. Bank withdrawal rejects zero/negative amounts and insufficient balance. |
| Potential reentrancy | The yielding download probe may print one or two starts. A temporary barrier variant forces both checks to precede mutation and prints two. The notes no longer treat yielding or sleeping as proof of a particular schedule. |
| Awaited cleanup | Swift 6.4 async defer releases the slot before its child completes, including throwing exit. A slot can be reused afterward. Separate starts may still succeed sequentially; the gate limits simultaneous ownership rather than lifetime start count. |
| Shared download operation | Two callers await one in-flight producer, then a later caller starts a new producer. The code no longer unnecessarily awaits an already-isolated clearTask call. Exposing the Task handle also exposes shared cancellation authority. |
| Naive token refresh | Two controlled producers complete new then old; the naive actor caches old, confirming a logic race without a data race. |
| Shared token refresh | Both coalescing examples use one producer. The simpler example lets a cancelled waiter receive the shared result. The version with explicit waiter policy throws CancellationError after shared completion while leaving the successful shared token cached. |
| Invalidation | The original enum-only implementation returns/caches old after a newer token already completed. The repaired implementation returns superseded to the old waiter and keeps new cached. An old producer failure cannot remove the replacement slot; a third caller still joins the replacement. A current producer failure permits retry. |
| MainActor compiler probes | Synchronous nonisolated calls and detached UI mutation are rejected; the async caller and MainActor caller fixes execute successfully. Capturing the nonisolated mutable search model in detached work is rejected during compilation. |
| Global actor | Two calls through MetadataActor produce revision 2. The example distinguishes one globally shared domain from per-instance actor domains without making thread-count claims. |
| Observable declaration | The available MainActor @Observable model compiles and its query/result properties read back correctly. This does not validate Observation delivery timing or a rendered view. |
| Search state/lifetime | Controlled identical-query calls preserve the newer ranked result when old success or ordinary failure arrives. Old cleanup leaves a newer handle intact while loading. Explicit cancellation followed by a service error leaves idle; a current error publishes failed. Dropping the model clears a weak reference and cancels its task while the injected operation is still pending. |
| CPU work | The @concurrent sorter returns [3, 2, 1]; a cancelled invocation throws. Checks surround the synchronous sort, whose midpoint is not interruptible. No throughput or physical-thread measurement was made. |

One additional focused negative configuration removes the explicit nonisolated annotation from the Sendable Token type under MainActor defaults. Accessing its computed isValid property from the instance actor is rejected. This exposed a real settings dependency; the published Token/AuthClient declarations now state their intended nonisolated contract and pass the selected settings checks.

Final positive configurations are free of compiler warnings. The SDK Observation macro checks used approved outer-sandbox compiler execution, as described for the prior macro batch. All execution used the recorded macOS 27 host. iOS and older deployment targets were compiled only; no UI rendering, real HTTP/authentication/Keychain behavior, other-platform runtime, prompt waiter cancellation, token revocation, server expiry validation, or refresh-token rotation is claimed.

### Reproduction notes

Use the common Swift 6/nonisolated baseline commands above, compiling @main drivers with -parse-as-library and negative cases with -c. Cache usage uses its preceding ImageCache declaration; token examples use the explicit Foundation-backed Token model, and the naive client protocol supplies the injected refresh fixture's conformance. D5's short render fixes use the preceding ViewModel declaration.

Temporary actors retain checked continuations to register refresh/search calls and complete them in a chosen order. Every pending continuation is completed exactly once. A host-only Task.immediate helper enters the token store before returning each waiter handle so a joining caller is registered before the producer is released. That OS 26+ helper is not a requirement of the published token API. The old enum-only token implementation is extracted from the original Git version solely to demonstrate the invalidation failure with the same fixture. The temporary gate/barrier variants are labeled as such; they are not reported as unchanged published source.

## Additional 6.3 catch-up found during lifetime validation: weak let

The compiler suggested weak let for the temporary model-lifetime observer. SE-0481 records this feature as implemented in 6.3. The roadmap now assigns the remaining ARC, capture-mutability/ImmutableWeakCaptures, and sendability coverage to C1/A8/D6; A2's binding explanation and rubric were updated immediately.

Five focused configurations passed: positive local/stored weak-let bindings and a checked Sendable holder compile for macOS 13 and iOS 18; the host run observes both references present while the owner is explicitly kept alive and nil after its last strong reference is cleared. Negative reassignment probes fail for both deployment targets. No extra feature flag is required for these bindings on the checked 6.4 compiler. This does not certify the separate weak-capture setting or execution on a 6.3 toolchain.

## Isolation controls and synchronization review: D8 / D12

171 final configurations matched their stated expectations, including 45 host executions: D8 has 93 configurations / 25 executions, and D12 has 78 / 20. These totals include focused availability and behavioral variants, intentional diagnostic cases, and one isolated-process runtime trap. An expected failure passes only when its intended semantic diagnostic or trap appears. All final positive sources are free of compiler warnings.

All 54 concrete Swift fences have a compilation disposition: D8's 30 fences target macOS 13 and iOS 18; D12's 24 target macOS 15 and iOS 18 because its Mutex examples use that library baseline. Each set includes its deliberate counterexamples. Six D8 and eight D12 examples additionally compile with default MainActor plus NonisolatedNonsendingByDefault. Three incomplete sketches in each chapter are explicitly labeled as such. Swift 6 language mode, explicit nonisolated defaults, the common SDK/module-cache setup, and default Onone optimization apply unless a configuration says otherwise.

| Area | Observed behavior / correction |
| --- | --- |
| Sync isolation | Immutable Sendable names, pure normalization, actor counts, and isolated Ledger batches produce their documented values. Reading mutable storage from nonisolated members is rejected. A function declaring two isolated parameters is rejected independently of what arguments a caller might provide. |
| Async isolation | The explicit nonisolated(nonsending) function returns MainActor identity, while the @concurrent function returns nil isolation. Both assertions are true in all four combinations of default isolation and NonisolatedNonsendingByDefault. This is an isolation check, not a thread-count or performance measurement. |
| Protocols | The actor's isolated synchronous displayName cannot satisfy the nonisolated synchronous requirement. An async Sendable protocol requirement and immutable nonisolated metadata each work. The actual diagnostic uses ConformanceIsolation; the harness matches the semantic note instead of older wording. |
| Executor assumptions | The synchronous legacy callback succeeds when called through its actor, printing 1; calling it directly from outside the actor terminates an isolated subprocess with Incorrect actor executor assumption. The Parser variant also retains its event. Core dumps are disabled for the final harness. |
| Task-executor hashability | Two owning TaskExecutor objects supply three unowned handles, with the first repeated: the set count is 2. Removing the conformance-use availability annotation fails for macOS 26 and iOS 26; the checked SDK's Hashable conformance starts at OS 27. Earlier equality/hash methods do not imply earlier Hashable conformance. The fixture keeps owners alive and does not enqueue jobs or test scheduling. |
| Cache identities | The AvatarCache exercise preserves two case-distinct opaque user IDs, storing separate [1] and [2] payloads with count 2. Label normalization is no longer applied to cache identity. |
| Mutex basics | Counters, generic cache insertion/replacement/removal, session snapshots, and image-cache reads produce expected values. Both async critical-section counterexamples fail because withLock requires a synchronous closure. Unguarded Mutex use fails for macOS 14 and iOS 17. |
| Try-lock and throwing | An uncontended closure returning nil produces an outer success. With a different Dispatch worker deliberately holding the lock, try-lock returns outer nil without executing the closure. After release it returns 7. Both withLock and withLockIfAvailable release after a thrown error, permitting the next try-lock to succeed. No recursive lock acquisition or busy spinning is tested. |
| Invariants | Two synchronized readers are forced to observe stock 1 before either removes it; the split API ends at -1. Twenty competing calls to the one-operation reserveOne API record exactly one success. |
| Work outside the lock | While an old decode is paused, raw data is replaced and a new decode commits [12]. The released old decode returns false; [12] remains stored. This tests freshness, not decoding performance or physical-thread placement. |
| Shared image producer | Two consumers share one fetch and both receive [7], including a cancelled consumer. A cached read adds no fetch; synchronous memory lookup agrees. A failed producer permits retry, which returns [9]. The example intentionally has no invalidation API or prompt consumer cancellation. |
| UI state | Old success and old failure leave a newer request loading; its completion publishes loaded2. Finishing new before old also preserves loaded2. Current cancellation followed by an ordinary service error leaves idle; a current uncancelled error publishes failed. Repeated-URL consumers coalesce into one fetch, and cancelling the old caller cannot reset loaded3. Already-cancelled entry stays idle and performs zero fetches. No SwiftUI view was rendered. |

### Retained-reference boundary observation

The ReferenceCounter example safely keeps a non-Sendable mutable class inside Mutex and returns only Int snapshots. A separate temporary variant adds a method returning the stored class directly with mutex.withLock { $0 }. Contrary to the initial expected-diagnostic assumption, that method compiles on this toolchain; two sequential calls compare identical by reference. The final record treats acceptance as an observation, not an expected rejection or proof of safe cross-thread use. No concurrent access to the leaked object is executed. The note and rubric therefore require an encapsulation audit instead of promising the compiler rejects every retained-reference escape.

### Compiler crash and replacement isolation probe

An earlier isolation probe returned Bool directly from #isolation != nil inside each of nonisolated(nonsending) and @concurrent async functions. Compiling the @main driver failed with signal 6 and LLVM verification messages including PHINode should have one entry for each predecessor of its parent basic block and Broken module found, compilation aborted. This is a toolchain failure, not a source-level negative test; it is excluded from the 171 successful final configurations. No external bug report was filed and the cause was not diagnosed.

The published replacement returns (any Actor)? from each helper and compares the values in the MainActor caller. It compiles for both chapter deployment targets and executes successfully in the four settings configurations. A separate attempted switch spelling had an ordinary parser error and was discarded; it provides no isolation evidence.

### Reproduction notes and limits

Use the common direct swiftc commands above. Compile @main drivers with -parse-as-library and deployment/diagnostic cases with -c. D8's usage fences explicitly reuse Ledger, Store, Displayable, ImageCache, or AvatarCache declarations above them. D12's good async function reuses only the cache/fetch helpers, omitting the bad function; the session and image exercise use their preceding models and repository declarations.

Mutex behavior fixtures run synchronous main functions. Dispatch semaphores coordinate a worker holding the lock or pausing its injected decode; these blocking gates are test infrastructure, not a production concurrency pattern. The image fixture owns checked continuations in an actor and releases each exactly once in chosen order. A host-only Task.immediate helper enters repository/model work before releasing the producer; its OS 26+ requirement is not a deployment requirement of the published examples.

Execution is limited to the recorded macOS 27 host. iOS and older targets are compilation checks only. No Linux/Windows runtime, network, real image decoder, UI rendering, contention benchmark, fairness guarantee, or leaked-reference data race was exercised. SE-0512 supplies the documented non-spurious guarantee (backdated to Swift 6.0); the controlled run confirms the two tested lock states, rather than experimentally proving every possible schedule.

## Detached-task and stream ownership review: D10 / D11

101 direct-compiler configurations matched their expectations: D11 has 54 configurations with 12 host executions; D10 has 46 with 13 executions, including one intentionally unfinished subprocess stopped by a two-second timeout, plus one explicit Swift 6 compilation of the package-dependent channel fence. Two further temporary package builds and two executions validate the channel example and its focused lifecycle variants. Twelve compiler configurations intentionally emit NoUseUnstructuredThrowingTask; other final positive sources are warning-free. Diagnostic failures, warnings, normal runs, and the bounded timeout are distinct expected outcomes.

D11's 13 concrete Swift fences compile for macOS 13 and iOS 18; eight selected configurations additionally use default MainActor plus NonisolatedNonsendingByDefault. D10 has 14 directly compiled fences plus one package-dependent channel fence. Its core sources target macOS 13 / iOS 18, and the notification/subscription owners use macOS 15 / iOS 18 for Mutex. Five selected D10 sources additionally compile with both settings enabled. Domain/API fragments are labeled as sketches rather than standalone Swift programs.

| Area | Observed behavior / correction |
| --- | --- |
| Task actor inheritance | The directly nested Task sees MainActor, while Task.detached and a Task created by an explicitly nonisolated synchronous helper return nil actor isolation. The three assertions pass in all four default-isolation/NonisolatedNonsendingByDefault combinations. Direct detached mutation of a MainActor property is rejected; the inherited Task and an explicit MainActor.run bridge each publish Done. |
| Task-local context | The ordinary Task sees abc-123, detached work sees nil, and leaving the binding restores nil. A detached closure explicitly annotated MainActor runs there but still sees nil task-local state. This distinguishes chosen actor isolation from inherited task metadata. |
| Cancellation ownership | A cancelled creator creates ordinary and detached tasks that both report false cancellation. Cancelling a task that awaits a separately owned handle does not cancel that operation. No exact priority or physical-thread assertions are made. |
| Throwing-task warning | Unused Task, Task.detached, Task.immediate, and Task.immediateDetached variants emit NoUseUnstructuredThrowingTask. Immediate variants are annotated for OS 26; they also compile for older deployment targets within that guard. Explicit discard removes the warning but does not observe failure. Awaiting the handle propagates the work error. |
| Feed model | Controlled old success and old ordinary failure leave the newer request loading with its handle intact and no stale error. New completion publishes new and clears its handle; finishing old after new leaves that result unchanged. Explicit cancellation followed by ordinary failure leaves no error; an uncancelled current failure reports one. Dropping the model clears its weak observer and cancels the pending task, which observes cancellation when the fake service is released. |
| Finite and unfinished streams | Finite examples print their values and terminate. The unchanged missing-finish consumer, with only stdout flushing added, prints 0/1/2 and remains unfinished until the harness kills it at two seconds. A separate controlled consumer is cancelled after receiving those values and exits with its cancellation flag true. Neither test leaves a running process. |
| Yield and buffering | bufferingNewest(1) drops 1 and retains 2; bufferingOldest(1) drops newly offered 2 and retains 1. A zero-capacity offer without a waiting consumer drops 9. Yield after finish reports terminated. Early break with both stream and continuation retained leaves termination count zero; explicit finish changes it to one, and repeated finish leaves it at one. |
| Notification ownership | Two posted Int payloads coalesce to the latest 2. Explicit repeated close removes the stored observer and ends iteration. Dropping a never-consumed owner clears its weak observer and ends its retained stream. Raw Notification objects and observer tokens are not sent to the consumer. |
| Subscription policy | Normal completion drains [1, 2]; an overflow retains [1] then throws overflow; an injected transport failure drains [1] then throws failed. Each stops the producer once and clears its stored handle. Explicit repeated close, consumer cancellation, and dropping the owner while receive is pending all trigger one stop and release the fake pending receive. |
| Completion during installation | A temporary Task.immediate variant of the producer completes synchronously with nil before handle installation. The owner reports closed with no stored task and one stop. This tests the installation branch using a labeled host-only source variant, not a scheduling guarantee for ordinary Task. |
| Channel lifecycle | The published structured producer sends 0/1/2 and exits. An early-break variant prints only 0 and exits without leaving a blocked sender. A sender is unfinished before handoff, then returns before application processing is complete. Cancelling the sender or finishing the channel also releases send; those cases produce no consumed value. |

### Temporary package evidence

The channel probe uses tools version 6.4, Swift 6 mode, an executable target with explicit nonisolated defaults, the macOS 27 SDK, and a package minimum of macOS 13. SwiftPM resolved Swift Async Algorithms 1.1.5 at 3da39bbc4e687d4192af7c9cf4eab805745a0b9c and Swift Collections 1.6.0 at a0cb0954ecb21e4e31b0070e6ed5674e8556685a. The package source, lockfile, build logs, and executable remain under /private/tmp/swift64-interview-validation/channel-package. There is no permanent package or dependency in the curriculum.

Both builds used the installed swift build with --package-path pointing to that temporary package and --sdk pointing to the recorded macOS SDK. Build output was piped through xcsift -f toon -w; both reported zero errors/warnings. The first build ran the unchanged published transfer function. The second added a source-labeled early-break variant and controlled send/consume/cancel/finish fixtures. A final direct swiftc -c check compiled the unchanged channel function with explicit -swift-version 6, -default-isolation nonisolated, macOS 13 target, and an -I path to the package's built modules. Swift Build's recorded command selects MacOSX27.0.sdk and arm64-apple-macos13.0; the executable also records minimum OS 13.0. Mutex-based test signals are availability-guarded and require macOS 15; this does not raise the channel example's package minimum. Runtime execution occurred only on macOS 27.

### Reproduction and limits

Direct checks use the common swiftc baseline, -c for compile-only checks, and -parse-as-library for @main drivers. D10's short consumers reuse the preceding makeStream declarations; the websocket usage compiles with the generic StreamSubscription and adapter. No websocket was connected. The missing-finish timeout variant adds import Darwin and fflush(nil) after each print so captured output is visible before termination.

Temporary actors and mutex-protected continuations coordinate service completion and producer shutdown. Each continuation is resumed once, outside the mutex where applicable. The feed fixture records cancellation when an injected service finishes, rather than claiming the service stops promptly. Stream fixtures explicitly release pending receive calls when their synchronous stop callback runs. The installation fixture uses host-only Task.immediate (OS 26+) and the notification tests use a private NotificationCenter.

SDK TaskLocal macro checks and SwiftPM operations used approved outer-sandbox execution. The first package tag lookup failed inside the sandbox because GitHub DNS was unavailable; the approved read resolved the official tags, after which the package was pinned. This was an environment constraint, not a source failure. No UI rendering, real network protocol, authentication/reconnect behavior, fairness benchmark, older-OS execution, non-Apple runtime, or asynchronous resource-teardown guarantee is claimed. D10's Observation comparison is source-backed; the later F6 review below completes the detailed API/availability audit.

## Sendability and transfer review: D6

152 final compiler configurations matched their expectations, including 24 host executions. All 39 Swift fences have a compilation disposition for macOS 13 / iOS 18, except the two Mutex examples, whose macOS minimum is 15. Twenty-six selected sources additionally compile with default MainActor plus NonisolatedNonsendingByDefault. The two weak-capture sources also compile with ImmutableWeakCaptures, with reassignment intentionally rejected. One API-audit configuration intentionally warns; other final positive sources are warning-free.

The chapter replaces empty task-launching helpers with actual returned Task handles, puts negative capture probes in explicit nonisolated functions, and replaces unnecessary unchecked Mutex-owner conformances with checked Sendable. These changes make both behavior and task ownership observable. SDK/compiler settings, explicit SDK paths, default Onone optimization, and the common temporary module cache are as recorded above.

| Area | Observed behavior / correction |
| --- | --- |
| Checked representation | A non-Sendable stored reference and mutable stored property each fail checked conformance. Profile/configuration examples compile and run. LockedCache performs replacement and missing-key lookup; TokenStore commits two fields under one lock. Twenty concurrent updates produce a snapshot count of 20. |
| Captures | Mutable class captures, mutable local capture, and unconstrained generic capture fail for their intended reason. A String snapshot prints Done! even after the source binding changes to ?. The actor repair returns 1 after awaiting its task. Adding sending to a generic parameter does not make an explicit @Sendable capture satisfy conformance checking. |
| Task transfer | The unconstrained sending generic API successfully transfers a mutable non-Sendable printable object into its Task and prints 7. This differs from the explicit @Sendable API. The returned handle is awaited. |
| Parameters/results | The disconnected buffer examples return 2 and 1. Post-transfer mutation through an alias fails, as does returning the retained MainActor buffer as sending. Alias mutation before handoff succeeds; a later discard expression (_ = alias) also compiles because it does not access mutable state. A Sendable Int is reused after sending and produces 14. |
| Installed state | The actor retains a transferred DraftTokenStore and returns abc. Mutating the caller's alias afterwards fails. The factory's sending result is disconnected; no detached task is created. |
| Suppression and inference | ~Sendable rejects the generic Sendable call but permits the isolated sending handoff. Conditional Container conformance succeeds for Int and fails for a mutable class. Unconditional conformance plus suppression fails. A public non-frozen value lacks inference; @frozen infers it unless explicitly suppressed. |
| Suppression scope | Extension, protocol, generic-parameter, and actor suppression forms are rejected. An audited subclass can add unchecked conformance above a suppressed base. A direct MainActor class with suppression compiles and fails a generic Sendable requirement; explicit simultaneous conformance and inherited-conformance suppression fail. |
| Weak references | The read-only @Sendable capture and weak-let holder compile with or without ImmutableWeakCaptures. Before release both see the owner; after release both see nil. Replacing weak let with weak var fails checked class conformance; a non-Sendable referent also fails. Explicit weak-capture reassignment compiles under the baseline and fails with ImmutableWeakCaptures. |
| Public API audit | -Wwarning ExplicitSendable warns on an unannotated public struct, while the adjacent explicit Sendable and ~Sendable declarations satisfy the audit. No experimental TildeSendable flag is supplied. |
| Image data cache | Two injected fetches are held independently. The second fetch completes with [2] and publishes it; releasing the first with [1] still returns [2]. A cached read stays [2], with exactly two fetch calls. This model revalidates but does not coalesce or invalidate. |
| Token service | Two callers share one producer and both receive shared, including a cancelled waiter. A cached read keeps fetch count and committed refresh count at 1. A failed shared producer reaches both callers with zero successful commits; retry returns retry and produces one successful commit from two attempts. |
| Value client | The injected immutable TokenClient returns u-d and the supplied timestamp 100.0. Its Sendable conformance is not treated as an execution-isolation guarantee. |

### Global-actor and alias distinctions

The initial focused expectation that every global-actor class rejected ~Sendable was too broad. The tested direct declaration is accepted and its generic Sendable use is rejected. Inheriting from a MainActor base and attempting suppression instead produces the conflicting-conformance error. A separate direct-class async transfer probe also compiles; it is recorded as compiler acceptance, not a concurrent race test. The note now distinguishes direct annotation from inherited conformance rather than attributing one rule to all global-actor types.

Similarly, replacing a later alias mutation with a discard expression compiles. The lesson concerns source-side accesses that can race, not a blanket syntactic ban on every mention of an alias. The final negative tests retain actual mutable access.

### Reproduction and limits

Use the common swiftc baseline and -c for compilation/diagnostic checks. The short capture examples reuse run; weak-capture reassignment reuses Owner; buffer counterexamples reuse ScratchBuffer and the MainActor transfer functions; exercise usage reuses TokenService or the installer declarations. Runtime drivers use -parse-as-library. Temporary artifacts are under /private/tmp/swift64-interview-validation, including check_d6.py, d6_runtime_cases.py, and d6-published-results.json.

Controlled actor gates own and resume checked continuations once. A host-only Task.immediate helper enters TokenService on its actor before returning the waiter's handle, so the cancelled and failing second waiters are established before producer release. That fixture requires OS 26+, without raising the published actor's deployment requirement. No network, real authentication/token expiry, UIKit rendering, lock-performance benchmark, unsafe concurrent mutation, older-OS runtime, or non-Apple execution was tested.

## Continuation ownership and callback lifecycle review: D9

79 final compiler configurations matched their expectations, including 20 host executions: 17 normal runs, two isolated expected traps, and one intentionally unfinished process stopped after two seconds. All final positive compilations are warning-free. All 18 concrete Swift fences have a compilation disposition for macOS 13 / iOS 18; the checked Mutex gate uses macOS 15 / iOS 18. New Continuation declarations are guarded for OS 27. Fourteen selected sources additionally compile with default MainActor plus NonisolatedNonsendingByDefault.

| Area | Observed behavior / correction |
| --- | --- |
| Basic bridge and ordering | The functional Data callback returns [1, 2]. The synchronous MainActor probe records before, after resume, then after await, establishing that resume does not jump past the bridging body's remaining statements. |
| Typed errors | An explicitly typed CheckedContinuation<Int, Problem> selects the checked throwing factory's generic Failure overload, returning 42 or throwing missing. Its Result form behaves the same. The new factory also returns 42/throws missing. Typed failures are not exclusive to the new handle in this SDK. |
| Noncopyable ownership | Packet returns sequence 7 through withContinuation. A checked-factory variant rejects the noncopyable payload's missing Copyable conformance. Consuming a handle twice and consuming a captured handle from an arbitrary escaping callback each fail for the expected ownership reason. |
| Transfer and conversion | A Continuation whose result is a non-Sendable mutable class satisfies a Sendable handle requirement and returns a disconnected value with number 7. Converting the new handle to CheckedContinuation consumes it and permits the true one-shot callback to return 1. |
| Misuse | The checked double-callback probe traps with a duplicate-resume diagnostic; dropping the new handle traps with Continuation was deinitialized without being resumed. The forgotten checked handle emits a leaked-continuation diagnostic and remains unfinished until the bounded subprocess timeout. Both traps exit with signal 5 on this host; no exact source line is part of the teaching contract. |
| Duplicate gating and sequences | The checked gate returns 1 for synchronous duplicate callbacks. Two concurrent submissions yield exactly one accepted value in {1, 2}. The finite stream emits 1 then 2 and finishes; no real asynchronous event producer is claimed. |
| Start outcomes | Synchronous success returns 7; synchronous failure throws failed; nil registration throws couldNotStart; success followed by nil registration preserves the earlier value. Each registers once and makes zero cancellation calls. |
| Cancelled entry | Cancelling the current task before entering awaitLegacy produces the typed cancelled error, with zero registration calls and zero request cancellations. The operation still installs/resumes its continuation without starting the request. |
| Cancellation during registration | The fake cancels its current task synchronously inside start, before returning its request. The adapter reports cancelled and forwards cancellation once when the request is installed. The reentrant failure callback cannot resume again. |
| Installed request cancellation | Cancelling after immediate entry has completed registration produces cancelled, forwards one request cancellation, and ignores a later success. Callback success that claims state first instead preserves value 8 and makes zero request-cancellation calls. |
| Concurrent callback/cancel submission | Dispatch submissions race task cancellation with success. The result is either accepted success with zero forwarded cancellations or the typed cancelled error with one forwarded cancellation. The test checks that relationship, not a particular scheduling winner. |
| Request destruction | A returned request whose captured resource calls completion from deinit is released after synchronous success without deadlock. A stored request does the same after asynchronous completion. Values remain 7 and 8, and release counts advance to 1 and 2. |
| Owner lifetime | A separate legacy owner explicitly reports ownerEnded from deinit; releasing it completes the waiting adapter with that error. This does not claim a silently discarded callback can be detected. |
| Optional result/error pairs | Value-only succeeds with ID 1; error-only throws failed; neither and both throw invalidCallbackState. Normalization and the shared gate serve separate roles. |
| Availability/fallback | Removing guards from the new factories, Packet factory, or gate fails targeting macOS 26 / iOS 26. A temporary checked-handle implementation of the same gate compiles for macOS 15 / iOS 18 and passes the same nine cancellation/start/concurrent-callback scenarios on the host. |

### SDK evidence and compatibility distinction

The installed _Concurrency.swiftinterface declares the caller-isolated generic checked throwing overload with a CheckedContinuation<T, E> body and throws(E), alongside its any Error overload. Its availability starts at the older concurrency baseline, and the chapter's typed examples compile for the stated older deployment targets. The selected compiler/SDK evidence establishes this project's behavior; no older compiler was run and no claim is made about the exact introduction release of that checked overload. The current [standard-library implementation](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/CheckedContinuation.swift) corroborates the API shape, but main-branch source alone is not treated as release evidence.

Continuation itself is noncopyable and Sendable, accepts noncopyable Success, consumes its handle/result on resume, and carries OS 27 availability in this SDK. The checked conversion also requires OS 27. The code generation checks distinguish those constraints from type checking alone.

### Reproduction and limits

Use the common direct swiftc baseline and -c for deployment, ownership, and capture diagnostics. The typed Result/new-factory snippets reuse Problem; conversion reuses only legacyOnce, excluding its negative function. The checked gate and stream reuse legacy, excluding the trapping adapter. The user-normalization adapter reuses the complete ResumeGate implementation. Runtime @main drivers use -parse-as-library.

Temporary evidence lives in check_d9.py, d9_runtime_cases.py, and d9-published-results.json under /private/tmp/swift64-interview-validation. Each expected trap runs separately with core dumps disabled. The leak probe has a two-second timeout and is killed; no intentionally unfinished process remains.

Task.immediate is host-only test infrastructure for entering the caller-isolated adapter through the end of synchronous registration before returning its handle. Fake callbacks and request counts use Mutex; simultaneous submissions use DispatchQueue.concurrentPerform without timing sleeps or an asserted winner. Request resource deinit deliberately reenters the gate to test release outside synchronization.

The checked fallback variant replaces the stored handle/factory and passes a copy of the checked factory's borrowed parameter into install, removing the explicit consume at that call site. A mechanical type-name replacement alone initially produced the appropriate borrowed-value consumption error; the corrected fallback is the one counted above. No real legacy SDK, network, owner teardown protocol, performance comparison, fairness proof, unsafe-continuation misuse execution, older-OS runtime, or non-Apple runtime was tested. The adapter requests cancellation; it does not promise external teardown has finished when the async result arrives.

## Optional syntax, absence, and state modeling review: A4

84 final configurations matched their expectations, including 19 host executions: 17 normal runs and two isolated expected nil-unwrap traps. All 22 Swift fences compile for macOS 13 / iOS 18 with their stated positive or negative disposition; thirteen selected sources additionally compile with default MainActor plus NonisolatedNonsendingByDefault. The UIKit outlet example is conditionally compiled: its actual declarations are checked on the iOS target, while the macOS build excludes that branch. Final positive compilations are warning-free.

| Area | Observed behavior / correction |
| --- | --- |
| Binding and chaining | Guard/if/optional-pattern bindings produce the expected present/absent branches. The profile/address chain returns nil for missing data and Berlin for the populated value. Assignment and Void-call probes return (0, false) for an absent receiver and (1, true) for a present receiver, demonstrating skipped RHS evaluation and Void? completion status. |
| map/flatMap | Nil, unparseable text, and 42 produce the distinct mapped states missing/null/value 42, while flatMap merges the first two into nil. A nil input invokes the map transform zero times. Uppercasing preserves absence. |
| try? | Missing lookup and thrown error both become nil. An existing Int?? result retains exactly that depth: a successful .some(nil) stays null, while a thrown error becomes outer missing. The correction reflects Swift 5+ behavior, not a new 6.4 semantic change. |
| Defaults and dictionaries | A present value does not evaluate the fallback; nil evaluates it once. The dictionary distinguishes absent key, stored nil, and stored 7. Subscript assignment of nil removes the entry; .some(nil) retains a present nil value. |
| IUOs and forcing | An inferred copy of a nil String! prints Optional<String> and nil; an explicit String context succeeds after assignment to Ada. Explicit and implicit nil forcing each trap in an isolated host process. Required dependency injection increments the session count once. |
| Optional protocol syntax | New single-protocol, nested, IUO, and grouped-composition forms compile without experimental flags. Parenthesized equivalents and a some-composition variant compile for both deployment targets. Present/absent opaque and existential calls behave as declared. Untyped nil alone cannot infer the opaque wrapped type, and a bare optional composition is rejected with a parentheses diagnostic. |
| Noncopyable constraints | Token? storage and take return ID 7 and leave the original slot nil. map from Int? produces a noncopyable Token?; map on a Token? input is rejected for its Copyable requirement. Separate ref/mutableRef/put probes each fail because the selected Optional API has no such member. Ordinary Int cannot receive nil. |
| Refresh state | Initial loading becomes loaded A. Refresh failure retains A in refreshFailed; retry retains it while pending and then publishes new A. Refreshing B from A instead shows loading B with no A data, and its failure stays associated with B. |
| Cancellation | A cancelled refresh followed by an ordinary service error restores the previous A. Cancelled initial load followed by success returns to idle. A pre-cancelled B request does not start another fetch or replace an already pending A request. A service CancellationError also restores idle under the stated policy. |
| Freshness and clearing | Old error leaves a newer B request loading and cannot prevent B's later success. Completing the second same-user request first preserves its new value when the old success arrives. Clearing invalidates publication, leaving idle after late completion. |

### SDK and syntax boundaries

SE-0521 records the optional opaque/existential spelling improvement in Swift 6.4. The tests use explicit nonisolated protocol/data declarations to keep target default isolation from changing the example's purpose. The new and parenthesized spellings are tested as variants of the same declarations, not as a claim that opaque types from different declarations share identity.

The installed Swift.swiftinterface declares Optional storage over noncopyable/nonescapable payloads and take on the appropriate generalized extension. Its public map overload is on the ordinary Copyable-Wrapped extension, while the result U permits ~Copyable. Underscored borrowing/consuming helpers are not promoted into recommended public APIs. The absent reference-accessor probes agree with the earlier release-status gate; these proposal operations remain excluded from the shipped baseline.

### Reproduction and limits

Use the common swiftc baseline with -c for compile/diagnostic checks and -parse-as-library for @main drivers. Optional protocol counterexamples reuse the Named/Identified declarations, ownership snippets reuse Token, and parsingStates reuses parse/describe. The conceptual Optional declaration and naive UI flags are labeled text sketches rather than fabricated standalone programs.

Temporary sources, commands, and results are stored under /private/tmp/swift64-interview-validation in check_a4.py, a4_runtime_cases.py, and a4-published-results.json. Nil traps run separately with core dumps disabled. The state fixture uses an actor holding checked continuations and releases them once in a chosen order; host-only Task.immediate starts each request without relying on a timing delay. Every pending fake request is completed.

The model assumes its service returns a profile for the requested identifier. It validates whether retained display data belongs to the new request, while a service lacking that identity guarantee needs boundary validation. The caller owns the async task; clear invalidates publication without stopping the service, and cancellation restoration waits for a noncooperative service to return. No UIKit outlet was connected or view rendered, no network/schema decoder was exercised, and no older-OS or non-Apple runtime was executed.

## B2 — opaque and existential API contracts

Reviewed on 2026-09-22 with the common Swift 6.4 compiler/SDK baseline. All 25 Swift fences have a compile disposition for macOS 13 and iOS 18. Ten selected examples also compile with default MainActor plus NonisolatedNonsendingByDefault. Two additional configurations check the equivalent parenthesized optional spelling. Ten published-source drivers and one focused class-conversion driver run on the macOS 27 host. Including a focused hidden-result rejection, 74 compiler configurations and 11 executions match their expectations; positive sources are warning-free.

| Area | Observed result |
| --- | --- |
| Opaque identity | Repeated firstNumber calls compare successfully. Comparing firstNumber and secondNumber fails despite both returning Int. hide accepts Remote and Local generic inputs; assigning its opaque result back to Remote is rejected. |
| Generic relationships | Two independent some parameters accept Remote/Local. One shared S rejects those unrelated structs. A focused pair of statically Base arguments containing Base and Child succeeds and reads base/child, demonstrating that the generic relation does not prove equal dynamic subclass types. |
| Existential opening | A plain any ImageSource passes to the generic name and identity operations; identity's result is erased again. Holder<S> escaping, two erased arguments requiring one S, and returning an existential directly as some ImageSource are rejected for the intended conformance/type-relationship reasons. |
| Optional syntax | some ImageSource?, any ImageSource?, and grouped composition compile without a feature flag. The corresponding parenthesized source compiles for both deployment targets. Runtime presence/absence yields remote/nil, local/nil, id 7, and the expected none fallback. |
| Optional inference/opening | Bare nil at a some-optional parameter and as the only opaque result cannot infer the wrapped type. An ungrouped optional composition is rejected. An optional existential does not open through the some-optional parameter; unwrapping and calling the plain generic name succeeds for remote/local/nil inputs. |
| SwiftUI | The Text/Image ViewBuilder function and AnyView-backed body's opaque result compile. A stored any View used directly as the body's opaque result is rejected. These are compile checks, without view rendering or identity/update measurements. |
| Canonical probe and fixes | The mixed Local/Remote opaque factory fails with nonmatching underlying types. Existential selection, enum delegation, and caller-selected construction produce the expected local/remote names. The rubric contains the same canonical negative source. |
| Sequence exercise | The erased Array input, opaque Array input, and opaque ArraySlice input all produce [1, 3]. Runtime-selected Array/reversed collection produces [1, 2, 3] / [3, 2, 1]; an explicitly Int-typed reduction returns 6. No allocation or speed comparison was performed. |

### Reproduction and limits

Extract Swift fences from B2. Reuse the named ImageSource/Remote/Local foundation for its mechanics; the number-identity negative additionally uses its two factories. Generic-call negatives reuse the relevant helpers. Optional negatives reuse optionalName or Identified as indicated; unwrapping reuses name. The canonical negative factory uses its own empty protocol, while the enum fix uses the named foundation. The sequence exercise combines User and the two functions. SwiftUI and ConstructibleSource blocks are standalone.

Use the common swiftc command with explicit SDK, -swift-version 6, -default-isolation nonisolated, and -c for compile checks; @main execution drivers add -parse-as-library. The alternate policy checks use -default-isolation MainActor -enable-upcoming-feature NonisolatedNonsendingByDefault. No optimization flag or experimental existential flag was used. The parenthesized-source check runs on 6.4, not on an installed 6.2/6.3 compiler.

Temporary check_b2.py and b2-published-results.json under /private/tmp/swift64-interview-validation retain exact source assembly, commands, diagnostics, and expected output. Focused variants add `func cannotNameHiddenResult() -> Remote { hide(Remote()) }` (rejected) or a Base/Child hierarchy whose statically Base arguments enter twoNames (accepted). These supplement the published fences and are labeled separately in the results.

Runtime checks use synchronous bounded programs. There was no benchmark, binary-library evolution test, SwiftUI rendering, older-OS runtime, iOS runtime, or non-Apple execution. Compilation for an older deployment target is not execution on that OS. Introductory type-system rules remain historical fundamentals; SE-0521 is the 6.4 addition.

## B3 — generic defaults and type relationships

Reviewed on 2026-09-22 with the common Swift 6.4 baseline. All 26 Swift fences have compile dispositions for macOS 13 and iOS 18; ten selected examples also compile with default MainActor plus NonisolatedNonsendingByDefault. Six deliberately unguarded variants verify Array.span, InlineArray, and Iterable availability. Including three focused constraint cases and host drivers, 88 final compiler configurations and 19 executions match expectations. Positive sources are warning-free. Exploratory diagnostics are recorded separately below.

| Area | Observed result |
| --- | --- |
| Ordinary defaults | ordinaryIdentity accepts Int/String but rejects Token for Copyable and Span<Int> for Escapable. withBorrowed accepts Token and Int, leaves the token usable, and obtains a local span count of 3. |
| Collection relationships | Search returns the actual ArraySlice index 2 and the String index locating the fox character; missing search is nil. Append yields [1, 2, 3, 4], a copied slice yields [2, 3, 4], and an empty append preserves it. Merge preserves duplicate 2 and handles empty inputs. |
| Conditional conformance | Box<Int> comparisons yield true/false, and nested boxes compare. Label without declared Equatable conformance prevents its box comparison. |
| Associated-type defaults | TokenSource uses default PayloadToken and produce returns owned payload id 7. Ordinary generic use and extension reject that primary payload for Copyable. The generalized extension consumes a created payload through its callback and returns 7. |
| Focused suppression cases | A noncopyable UniqueSource with Int payload can be borrowed by produce twice, returning 9 each time. A separate non-primary Storage associated type defaults to PayloadToken and remains usable through an ordinary generic parameter; its MemoryLayout size agrees with the concrete payload type. |
| Integer generics | A generic function returns InlineArray counts 3 and 4. Assigning the 3-element type to the 4-element type fails. Removing its OS 26 annotation is rejected for both older deployment targets. |
| Borrowed iteration | UniqueArray<Token> and its Span both visit ids 1/2; combined output [1, 2, 1, 2] leaves owner count 2. Removing the visitor's OS 27 annotation fails for the older targets. This reuses C5's contract; no new throwing iterator implementation is claimed. |
| Pairing and inference | Matching Array/ArraySlice pairs produce [(1, 1), (2, 4), (3, 9)]; shorter input truncates and empty input produces []. Explicit Int/String inputs plus a typed result fail. Uncontextualized mixed array literals instead infer Array<(Any, Any)> and successfully produce [(1, one), (2, two)] with string values. |
| Domain filtering | Requested IDs [2, 1, 1] preserve item order Ada/Bea/Ada again. Empty or absent requests return []; a slice/set combination returns Ada again. Matching optional-valued records retain both nil and Bea. Specializing the function to UserID items and OrderID requests is rejected for the intended same-type requirement. |
| Other algorithms | Heterogeneous erasure retains 1/A values. firstCommonElement returns 3 according to left traversal, with nil for missing/empty cases. No speed, allocation-count, or hashing-complexity benchmark was run. |

### Compiler diagnostic limitation and corrected old example

Initial explicitly typed mismatched direct calls to pairElements, valuesMatchingIDs, and appendAll were rejected with the compiler message “failed to produce diagnostic for expression.” This was not a successful explanatory diagnostic and was not treated as a passing semantic check. No process crash or accepted invalid program was observed for those calls. No external report was filed.

The published pairing negative adds a typed [(Int, Int)] result; the domain-ID negative uses an explicitly typed function specialization. Both produce the expected same-type diagnostic on macOS/iOS targets. The focused append mismatch likewise specializes its function type. The original diagnostics remain in b3-initial-results.json and the follow-up alternatives in b3-diagnostic-cases.json under the temporary validation directory.

A separate check of the original production implementation rejects its recursive Items.Element == Identified<Items.Element.Value, IDs.Element> constraint. The corrected published implementation introduces independent Value and ID generic parameters. The original mixed-literal pairing claim was also corrected: those literals can widen to Any, as confirmed by the new positive runtime example.

### Reproduction and limits

Use -c for each source with the common SDK/language/default-isolation flags; add -parse-as-library for @main drivers. The Token negative reuses ordinaryIdentity; withBorrowed supplies the helper for countBorrowedSpan. The boxed negative reuses Box. Associated-type negatives and the extension reuse PayloadSource/TokenSource. Pairing and domain-ID negatives reuse their respective functions and models. The literal-inference example reuses pairElements. Each standalone block is identified in the chapter.

Temporary check_b3.py and b3-published-results.json retain source assembly, exact commands, diagnostics, and output. The focused UniqueSource declares noncopyable conformance with makePayload returning Int; the auxiliary protocol uses a non-primary `associatedtype Storage: ~Copyable = PayloadToken`. Availability-negative variants remove only the corresponding annotation. No experimental flags were used, and historical proposal development flags were not copied into the baseline.

Execution was on macOS 27. Older Apple deployment targets were compiled, not run; iOS/non-Apple execution, UI rendering, binary compatibility, and optimized performance were not tested. The helper's R parameter retains ordinary Copyable/Escapable defaults, so its success does not claim arbitrary borrowed-result escape support.

## B1 — protocol relationships and typed cache boundaries

Reviewed on 2026-09-22 against the common Swift 6.4 baseline. All 25 Swift fences have compile dispositions for macOS 13 and iOS 18. Sixteen selected sources additionally compile with default MainActor plus NonisolatedNonsendingByDefault. With host drivers and focused associated-type checks, 85 final compiler configurations and 18 executions match expectations; positive sources are warning-free.

| Area | Observed result |
| --- | --- |
| Runtime capability | Foundation Data fixtures return byte counts 3 and 0 through any ImageLoading. The fixture performs no actual loading, network access, or decoding. |
| Associated output | Generic Int/Word parsers retain Int?/String? and handle invalid/empty input. An unconstrained existential produces Any?; a constrained Parser<Int> retains Int?. Returning the unconstrained result as Int? is rejected. |
| Self relationships | Generic combination yields Amount 3. Two erased inputs cannot call combined. duplicate erases its result successfully; a generic doubled helper accepts one opened value and returns erased Amount 6. |
| Composition/refinement | Named/Identified values produce the expected description and optional none fallback. A declared Record provides revision 2. Matching component conformances and members without Record conformance are rejected. |
| Noncopyable protocol support | Borrowed Ticket IDs and equality work; its explicit hash method executes without asserting unstable hash integers. Borrowed noncopyable existential access returns 9 and leaves the separate original ticket usable. Set<Ticket> and noncopyable CustomStringConvertible are rejected for Copyable constraints. |
| Associated-type defaults | DefaultFactory produces a Packet id 7. A focused IntFactory overrides the default by inference and yields 9. The ordinary generic factory use fails because its primary Output defaults to Copyable. |
| Semantic marker example | The required redactedDescription path emits only the placeholder. This tests the implementation, not a compiler guarantee that all other access paths redact. |
| Lookup probe | The unknown-key existential fails at get. Generic and constrained String/Int readers both return 42 or nil for present/missing keys. The rubric embeds the same negative source. |
| Typed cache consumers | Generic, existential, and conforming-erased repository forms each pass missing/insert/replace/independent-key/remove/missing-remove scenarios, under both baseline and MainActor-default plus NonisolatedNonsendingByDefault configurations. |
| Optional cache values | AnyCache around InMemoryCache<String, Int?> observes missing, cached nil, cached 7, and missing after removal. The implementation explicitly uses updateValue to store the optional payload. |
| Shared and separate storage | Constructing an eraser from a constrained existential succeeds through opening. Separately constructed and directly copied wrappers around one actor observe shared updates/removal. A new actor's storage remains independent. |
| Concurrent distinct keys | Twenty task-group children insert different keys through the eraser. All twenty values are present, sum 380. The fixture asserts no order among writes to the same key. |
| Rejected cache contracts | A String/Int cache cannot initialize the generic User repository or a String/User constrained existential. A synchronous untyped counterexample accepts the wrong payload and its String cast returns nil. |

### Reproduction and limits

Use the common direct swiftc baseline with explicit SDK, Swift 6 mode, and nonisolated defaults; -c checks compilation/diagnostics and @main drivers add -parse-as-library. Context-dependent fences state their prerequisites. Lookup readers reuse only the protocol/struct from the negative probe, excluding its invalid function. Cache repository checks combine Cache/InMemoryCache, User, and the relevant repository/AnyCache declarations.

Temporary check_b1.py, b1_runtime_cases.py, and b1-published-results.json under /private/tmp/swift64-interview-validation retain source assembly, commands, and observed output. Async fixture callbacks are explicitly @Sendable because they capture Sendable repositories and cross into an async helper under the baseline policy. The first fixture version omitted that callback annotation and was correctly rejected; the corrected fixture passes in both settings without changing the published cache design.

The cache fixtures use fresh actor instances, sequential awaited operations for replacement/removal, and a task group for independent-key writes. They require no timer, network service, UI, or scheduling-order assumption. Execution was on macOS 27; older Apple targets were compiled, not run. No iOS/non-Apple runtime, cache capacity/expiry policy, persistence, fetch coalescing, same-key ordering guarantee, binary compatibility, or performance measurement is claimed.

## B4 — dispatch, witnesses, and runtime integration

Reviewed on 2026-09-22 with the common Swift 6.4 baseline. All 19 Swift fences compile or reject as documented for macOS 13 and iOS 18; the Mutex-based KVO fixture uses macOS 15 for its positive older-target check. Thirteen selected sources also compile under default MainActor plus NonisolatedNonsendingByDefault. There are 80 final source/driver configurations, including 26 executions, plus four successful SIL emission/inspection checks. Eight executions use -O to check unchanged semantics; the KVO fixture additionally executes under the alternate isolation configuration. Positive sources are warning-free.

| Area | Observed result |
| --- | --- |
| Class and final methods | A HomeScreen passed through Screen calls HomeScreen.render and retains the final identifier. Overriding the final member is rejected. |
| Requirement defaults | HomeRenderer supplies home render; DefaultRenderer uses default render through generic and existential calls. |
| Inherited default witness | The child direct call returns child member while generic/existential calls return protocol default. An override without a base class member fails; redeclaring the inherited conformance is redundant. |
| Base-method redesign | Giving the base a real overridable witness member makes direct, superclass, existential, and generic child calls all return child override. |
| Constrained extension lookup | Concrete/strongly constrained calls choose equatable; weak generic/existential calls choose general. Optimized output matches. |
| Conditional witness selection | Box<Int> direct lookup chooses equatable while its unconditional conformance supplies general. EquatableBox's conditional conformance yields equatable on all three paths. Optimized output matches both cases. |
| Canonical probe/exercise | The f/g existential and generic calls print S.f / extension g; direct S calls print S.f / S.g. Adding g as a requirement changes the existential result. The title exercise changes Product/default/default to Product/Product/Product; a derived helper still uses its property witness while a direct same-named member differs. |
| Default isolation | Nonisolated protocol declarations did not keep unannotated extension defaults nonisolated under the MainActor-default configuration. Explicit nonisolated default/helper methods make the final examples pass. A focused removal from Renderer still rejects its DefaultRenderer conformance for crossing into MainActor-isolated code. |
| Objective-C selector | An exposed title method invoked through perform returns selector without needing dynamic on the declaration. No real UIKit/AppKit control was driven. |
| KVO | The final Player's @objc dynamic status produces idle/playing. Invalidating the observation prevents stopped from appearing. The callback records through a checked Sendable class with Mutex storage. The fixture passes under both policies; removing its Mutex availability guard fails for macOS 13. |
| Native dynamic | A struct's dynamic value method compiles without Objective-C exposure and returns 7. No replacement library, swizzling, or underscored replacement API is used. |

### SIL evidence

Four separate -emit-silgen compilations inspect named function bodies rather than merely searching an entire module for an instruction:

| Function | Observed lowering in this compiler |
| --- | --- |
| genericCalls | witness_method for P.f and function_ref to the extension P.g |
| renderScreen | class_method for Screen.render |
| Focused callNative helper | dynamic_function_ref for NativeValue.value |
| observedStatuses | objc_method for Player.status's foreign setter |

These inspect the unoptimized lowering and do not assert identical optimized instructions, timing, allocation counts, or stable mangled names. The optimized semantic runs are separate evidence, not a performance measurement or proof of cross-module ABI behavior.

### Reproduction and limits

Use the common swiftc command with -c for compiler checks and -parse-as-library for @main drivers. Negative overrides reuse their base class declarations; EquatableBox reuses Categorized and its defaults; genericCalls reuses the canonical probe. The alternative redesigns are standalone sources. Optimized variants add -O without whole-module or library-evolution flags. The SIL script combines the relevant declarations, adds only callNative for the native-method call, and uses -emit-silgen.

Temporary check_b4.py, b4-published-results.json, b4_sil.py, and b4-sil-results.json under /private/tmp/swift64-interview-validation retain commands, source assembly, outputs, and extracted SIL function bodies. Initial exploration that captured a mutable local array in the KVO callback warned under the SDK's concurrency annotation; the published Mutex recorder removes that issue without claiming Player itself supports concurrent mutation. Initial default-isolation failures are preserved in b4-initial-results.json.

Execution was on macOS 27, including the Objective-C and KVO runtime cases. Older Apple targets and iOS were compiled, not run. No non-Apple runtime, UI integration, dynamically loaded replacement, binary framework evolution, or performance benchmark was tested. Native dynamic support and the witness rules are historical semantics, not newly introduced Swift 6.4 features.

## B7–B8 — synthesis, equality, hashing, and collection policy

Reviewed on 2026-09-22 against the common Swift 6.4 baseline. B7 has 17 Swift fences and 66 final source/driver configurations, including 11 executions; B8 has 10 fences and 46 configurations, including 13 executions. Every fence has a compile disposition for macOS 13, iOS 18, and default MainActor plus NonisolatedNonsendingByDefault. The new collection conformances are availability-guarded for macOS/iOS 27. Positive compilations are warning-free. One additional SIL emission/inspection verifies partial Hashable synthesis.

### Synthesis and model contracts

| Area | Observed result |
| --- | --- |
| Structural values | Point/LoadState comparisons return true/false/false. Box conditionally derives equality/hashing; Box<Int> deduplicates to two values, while Box<closure> comparison fails. |
| Partial synthesis | BrokenUser supplies ID-only equality but compiles with synthesized hashing. Its emitted BrokenUser.hash(into:) body extracts id and name and invokes Hasher.combine for both. It is never inserted into a hash collection in validation. No exact or unequal hash outputs are asserted. |
| Matching witnesses | IdentityUser's manual ID-only equality/hash produces one valid Set member from two names with one ID. Its name is explicitly nonvalue metadata, not an equality-based content signal. |
| Coding keys | Settings decodes persisted sortMode and re-encodes that key. An unknown raw enum value is rejected. A synthesized mutable pageSize with an initializer still throws keyNotFound when its included key is missing. |
| Case enumeration | FeedFilter yields declaration order for allCases and a different explicit display order. An enum with an Int associated value fails automatic CaseIterable conformance. |
| Sendable | Immutable SearchQuery passes; Payload storing MutableBox fails checked conformance for its non-Sendable field, including under the alternate isolation settings. |
| Other synthesis limits | A closure-containing ScreenModel fails Equatable synthesis; a separate ScreenPresentation compares and its independent action executes once. A plain class fails automatic Hashable synthesis. |
| Identity and snapshots | Same-ID snapshots with different names return true for ID equality and false for content equality. A Set of snapshots holds two values while a Set of their IDs holds one. |
| Profile exercise | Same ID / changed content / preserved selection / replaced stored snapshot yields true/false/true/true. JSON preserves name, omits displayName, represents UserID as an object with rawValue, and round-trips the snapshot. |

### Noncopyable boundary

The installed compiler synthesizes Equatable/Hashable for the noncopyable Ticket struct; equality borrows two instances, hashing can inspect a borrowed receiver, and both IDs remain readable. B8's manual version also passes. Set<Ticket> and noncopyable Codable each fail Copyable requirements.

The noncopyable TicketState enum with empty and issued(Int) cases fails compilation with a diagnostic about a tuple containing a noncopyable element. The manual nested-switch equality and case-tag/payload hash compile under all three configurations. Runtime checks cover equal payloads, unequal payloads, different cases in both orders, empty/empty, hashing both cases, and reuse after borrowing. This is a tested compiler-synthesis limitation for that exact shape; no compiler crash or general prohibition on enum conformance is inferred. It has not been filed as an upstream issue. Initial focused probes are retained in b7-preflight-results.json.

### Collection behavior

| Area | Observed result |
| --- | --- |
| Independent value mutation | Changing a local Tag after insertion leaves the Set containing the original value, not the updated local value. |
| Stable reference key | A UserRecord name changes to B while its immutable UserID key still retrieves it. This is synchronous code with no concurrent-safety claim. |
| Insert versus replacement | With equal ID 100, insert retains the old object and reports false. update(with:) returns old and stores target. Changing target.title preserves membership, exposes Published, and removes exactly target. All seven exercise predicates are true. |
| New member paths | update(with:) on an empty Set returns nil and inserts; insert with a distinct ID reports true and returns the inserted instance. |
| Snapshot replacement | Dictionary update returns the old snapshot, stores the new snapshot under its ID, preserves one key, and structural equality detects changed content. |
| Structured persistent identity | ProductID round-trips region and SKU containing colons. Two distinct pairs collide under delimiter-only concatenation but remain distinct ProductID values and Set members. No canonical JSON encoding or cryptographic hash is claimed. |
| New Hashable conformances | Dictionary.Keys with the same keys and different values deduplicates to one; CollectionOfOne(1/1/2) produces two; two EmptyCollection<NotHashable> produce one. Dictionaries with non-Hashable values also have usable Hashable keys views. |
| Availability and constraints | Removing the collection function's availability annotation fails below OS 27 for macOS 13 and iOS 18. CollectionOfOne<NotHashable> fails its Hashable element constraint. |

The collection-count, explicit object-replacement, and snapshot-replacement drivers additionally run with default MainActor plus NonisolatedNonsendingByDefault. All other runtime checks use the baseline nonisolated-default setting. The invalid mutable-key canonical probe is compiled only; the review removes the old purported Swift 6.2.1 exact-output guarantee. No invalid-key Set lookup, removal, enumeration, or data race was executed as a correctness test.

### Reproduction and limits

Temporary check_b7.py, b7-published-results.json, check_b8.py, and b8-published-results.json retain exact commands, fence indices, appended drivers, expected errors, and outputs under /private/tmp/swift64-interview-validation. B8's Set<Ticket> negative source includes its preceding Ticket declaration. Other fences are standalone. Most host drivers use -parse-as-library with a small @main entry point; compilation negatives use -c. b7_sil.py and b7-sil-results.json retain the extra -emit-silgen command and the inspected hash function body. SIL inspection establishes this compiler's derivation, not stable instruction spelling or performance.

Execution was on macOS 27 only. Older Apple targets and iOS were compiled, not run. No older Swift toolchain, non-Apple runtime, real UI update, backend schema migration, concurrency stress, or hashing performance benchmark was tested. The checked new collection conformance availability is separate from source/compiler support; legacy identity, snapshot, Codable, and replacement corrections are not presented as new Swift 6.4 semantics. The repository remains Markdown-only.

## B5 — key paths, capabilities, transfer, and sorting

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 19 Swift fences compile or reject as documented for macOS 13, iOS 18, and default MainActor plus NonisolatedNonsendingByDefault. There are 84 final source/driver configurations, including 19 executions. Positive compilations are warning-free. Initial exploration is separate from the final passing checks and includes the compiler abort described below.

| Area | Observed result |
| --- | --- |
| Projection and function conversion | Literal map/filter and explicit application of a stored path produce the expected names. Passing the stored path variable directly to map fails. |
| Writable access | A mutable Counter root increments to 2. The let-root variant fails. A Screen struct with let Settings reference supports ReferenceWritableKeyPath and changes volume to 8. A focused nonmutating-setter struct also supports reference-writable assignment. |
| Static capability erasure | A let Settings class root fails assignment through a statically WritableKeyPath input, despite the underlying property permitting reference-style mutation. |
| Optional and subscript projection | Account optional chaining returns the initial name and nil after clearing. An index captured as 0 stays 0 after the local variable changes to 1. Optional chaining fails a requested WritableKeyPath type. A non-Hashable subscript argument is rejected. |
| Composition | Explicit plain WritableKeyPath operands compose to a writable path and rename A to B. Inferred Sendable operands produce a usable plain read-only path; returning it as KeyPath & Sendable or WritableKeyPath & Sendable fails. A directly expressed full literal satisfies WritableKeyPath & Sendable and updates to B. The installed appending signatures were inspected alongside the compiler results. |
| Partial/full erasure | PartialKeyPath<User> reads an Int as Any. AnyKeyPath matching User returns its name; applying the same erased path to Int returns nil. |
| Transferable descriptor | An inferred name path fits a KeyPath & Sendable property; an already plain KeyPath input cannot recover that guarantee. An immutable User and its Sendable path cross a real actor boundary and return A. |
| Captured reference | A locally used path capturing a non-Sendable IndexToken returns its value. Requiring the same path to be Sendable is rejected. Identity-based Hashable does not imply Sendable. |
| Global/instance actor isolation | MainActor path formation/application increments the model. Nonisolated formation and a Sendable result are rejected. An actor's path to its own isolated property is rejected even inside an actor method. |
| Getter effects | Reusing the same Sendable path yields values 1 and 2, with access count 2. Its mutable Reading root is not transferred and is not declared Sendable. |
| Typed sorting | Ascending/descending ages preserve Mina/Noa input order on ties, empty input stays empty, invalid email field parses to nil, and validated name sorting produces A/B. A missing property path fails compilation. |
| Other access boundaries | A method-call path is rejected. The typed Field pairs Int projection and formatting to produce Age: 34. No binding or observer is installed by forming a path. |

The name descriptor, MainActor operation, and sorting drivers also execute with default MainActor plus NonisolatedNonsendingByDefault. The remaining host drivers use the baseline nonisolated-default policy. Examples reuse earlier declarations where noted: User, Settings, Profile/Account, and CounterModel; the harness records the exact source assembly rather than assuming every excerpt is standalone.

### Noncopyable-root compiler limitation

The explicit signature KeyPath<Token, Int>, for Token declared ~Copyable, fails with a normal Copyable-constraint diagnostic under all three checked configurations. During earlier exploration, an inferred global path declared as `let path = \Token.id` instead caused swiftc to abort with signal 6 during SIL generation, reporting an invalid conformance in SILGenLazyConformance.cpp. The source, diagnostic, and return code are retained in b5-preflight-results.json. The chapter does not present that inferred declaration as a working feature, and the crash was not repeatedly executed as a validation target or filed upstream. A focused explicit-type follow-up is retained in b5-extra-results.json.

### Reproduction and limits

Use temporary check_b5.py and b5-published-results.json under /private/tmp/swift64-interview-validation for the 84 final commands, sources/drivers, diagnostics, and outputs. The final harness uses -c for negative checks and disables core dumps; runtime drivers have bounded execution. b5-preflight-results.json and b5-extra-results.json retain earlier exploration, including the composition/ownership boundary that determined the final explanation.

Runtime evidence is macOS 27 only; older Apple targets and iOS were compiled, not run. No invalid subscript or force unwrap, concurrent mutable-root access, SwiftUI binding/KVO/Observation integration, key-path performance benchmark, non-Apple runtime, or older compiler was executed in this review. Stable sorting, key-path literal conversion, and Sendable inference are existing features revalidated for the 6.4 baseline. The recorded appending behavior and noncopyable-root abort are limited to this toolchain/SDK.

## B6 — erasure, metatypes, registry lifecycle, and demangling

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 23 Swift fences have compile dispositions for macOS 13, iOS 18, and default MainActor plus NonisolatedNonsendingByDefault. The two Runtime fences use canImport guards and are excluded by the iOS build; this is not evidence of working iOS demangling. There are 116 final source/driver configurations, including 32 executions: 31 normal completions and one intended failed-cast trap. Published positive examples compile without warnings. Separate deliberately unbounded metatype variants exercise the diagnostic policy and are expected to warn.

### Erasure, construction, and identity

| Area | Observed result |
| --- | --- |
| Any/casts | Mixed values dispatch to Int/String/Other. An Int? nil erased to Any can successfully cast to Int? while retaining its nil payload; casting to Int returns nil. A local valid forced cast returns hello; an Int-to-String forced cast terminates with the expected cast-failure diagnostic. |
| Ownership/lifetime | Consuming Token into Any fails Copyable. A focused Span-to-Any attempt fails Escapable. UniqueToken.self cannot become Any.Type, but a generalized any (~Copyable & ~Escapable).Type works with ObjectIdentifier, type(of:), and a subsequent read of the unconsumed token. |
| Apple object boundary | Direct String-to-AnyObject assignment is rejected. Explicit bridging round-trips String; NSNumber(42) conditionally casts to Int; Int does not cast to String. A copied class reference preserves === identity. |
| Dynamic construction | Passing Dog.self through Animal.Type yields Animal from T() and Dog from type.init(). An unconstrained T.Type cannot call init. SearchPlugin.makeDefault returns SearchPlugin through Self. |
| Protocol metatypes | any Named.Type holds Person.self, supplies its static name, and constructs Person. It differs from (any Named).Type. Assigning the existential container metatype to the conformer-metatype variable fails. |
| Generic inspection | Direct existential type(of:) and the opening generic call report Value. A specialized function value fixing T to any Marker reports the existential container. All three predicate results are true. |
| Runtime identity | A typealias has the same ObjectIdentifier as its underlying type; an unrelated type differs. No identifier value is persisted or interpreted as a durable address. |
| Reflection and lookup | CustomReflectable exposes display rather than the stored actual field. Swift::Int selects the imported module type while Swift.Int names a nested local declaration; the two identities differ. |
| Dynamic payload | A nonempty screen name produces the typed event. Missing, wrong-type, and empty name inputs each throw invalidName. The loose registry accepts a mismatched Payments value under analytics and subsequently resolves nil for Analytics. |

### Metatype transfer and warning policy

The checked TypeLabel & SendableMetatype reader creates a warning-free @Sendable closure; executing it in Task.detached returns box. Requiring the mutable instance to satisfy Sendable fails. An explicitly MainActor-isolated TypeLabel conformance cannot satisfy the transferable metatype contract, even when the caller is on MainActor.

Removing SendableMetatype from this protocol-constrained capture emits SendableClosureCaptures and SendableMetatypes warnings for macOS 13, iOS 18, and the alternate default-isolation policy. Compilation succeeds. In two focused checks, -warnings-as-errors and @diagnose(SendableMetatypes, as: error) also leave the diagnostics as warnings and return success. The final expectations record that behavior; they do not claim the warning was promoted. b6-warning-policy-initial.json retains the initial expectation mismatch. This finding is linked from D6/F5 without generalizing it to other warning groups or native Swift 6 errors.

A separate unconstrained generic T.Type capture compiles without these warnings; the relevant difference is the protocol-conformance contract. Earlier exploratory uses of an upcoming StrictSendableMetatypes flag are not required or recommended by the chapter; the final examples and diagnostic checks use the normal baseline, with only the explicit warning-policy variants adding their stated control.

### Actor registry behavior

| Area | Observed result |
| --- | --- |
| Canonical usage | Missing, first registration, duplicate rejection, tracking result, removal, and absence after removal all match the six true predicates. |
| Lifecycle | Registering InstancePlugin twice yields true/false. Two resolves produce distinct live object instances. Removing it twice yields true/false and leaves an independently registered PaymentsPlugin available. The checked final class uses immutable storage and checked Sendable conformance. |
| Dynamic subtype regression | A ChildPlugin metatype stored as RootPlugin.Type registers under the child's identity. Resolving through both base-typed selection and ChildPlugin.self constructs ChildPlugin; RootPlugin.self remains unregistered. Both configurations pass with explicit nonisolated required initializers. |
| Duplicate concurrency | Twenty child tasks register the same concrete type; exactly one registration returns true and later resolution succeeds. No order among those tasks is asserted. |
| Type contract | An unrelated type cannot be registered because it does not conform to Plugin. Factories invoke the supplied type.init(); erasure and the result cast stay behind the private map. |

Canonical usage, lifecycle, and subtype drivers run under both default-isolation policies. The initial subclass fixture relied on an inherited initializer and failed under default MainActor; the fixture now explicitly declares the nonisolated required initializer and delegates to the base. Published registry declarations did not need an unchecked conformance. Constructors are synchronous; no async factory, singleton cache, protocol-interface resolution, real plugin loader, or multi-call transaction was tested.

### Runtime module and API availability

The macOS SDK provides Runtime.swiftmodule interfaces whose demangle overloads carry anyAppleOS 27 availability. The iPhoneOS and iPhoneSimulator SDK inventories contain no Runtime interfaces. Unguarded iOS imports fail with no such module for both iOS 18 and iOS 27 deployment targets. Raising only the deployment target therefore does not fix module presence.

The published sources guard import/declarations with canImport(Runtime). The portable String wrapper additionally checks OS availability and returns the original symbol on the unsupported/invalid path. iOS source compilation covers the module-absent fallback; only macOS runtime behavior was executed. Removing OS availability annotations from the macOS String and bounded implementations fails for macOS 13.

On macOS 27, the String API converts $sSi to Swift.Int and throws invalidSymbol for invalid text. The bounded implementation reports requiredBytes 9 at capacities 0 and 3, and returns Swift.Int at capacities 9 and 30. Invalid text reports invalidSymbol. An exploratory span probe also observed count 0 at capacity 0 and prefix Swi at capacity 3; the published helper deliberately does not publish truncated prefixes and validates full UTF-8 before making a String. The exploratory String(decoding: span, as:) attempt failed because Span is not Collection; UTF8Span validation plus String(copying:) is the checked implementation.

The portable wrapper returns Swift.Int for the valid host symbol and preserves not a symbol for invalid input. The inherited older-OS availability branch is compiled, not exercised on an older OS. The fallback does not symbolize memory addresses or guarantee demangled spelling across compiler versions. Module selectors are 6.3 syntax, independently checked here; Runtime demangling is the 6.4 API addition.

### Reproduction and limits

Temporary check_b6.py, b6_registry_cases.py, and b6-published-results.json under /private/tmp/swift64-interview-validation retain exact source assembly, commands, expected errors/warnings, and runtime output. The two excerpt dependencies are Named for the metatype-assignment rejection and Plugin declarations for registryDemo. Negative/compiler checks use -c; runtime @main drivers use -parse-as-library. Core dumps are disabled and processes have bounded execution. b6-preflight-results.json, b6-extra-results.json, b6-metatype-results.json, and b6-initial-results.json preserve exploratory failures and the decisions they informed.

All execution was on macOS 27. Older Apple targets and iOS were compiled only, with Runtime-excluded source where noted. No Linux/Windows/Embedded/WebAssembly runtime, older compiler, live UI, persistent schema deployment, performance benchmark, arbitrary Foundation bridge matrix, signal-handler execution, or real dynamic library loading was tested. The repository remains Markdown-only.

## B9 — Codable contracts and versioned schema migration

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 10 Swift fences have compile dispositions for macOS 13, iOS 18, and default MainActor plus NonisolatedNonsendingByDefault. There are 41 final source/driver configurations, including 11 successful executions. Positive examples are warning-free. The AliasOnly fence intentionally fails Encodable synthesis in all three configurations because oldName does not match a stored property.

| Area | Observed result |
| --- | --- |
| Stable key | User.displayName decodes from name and re-encodes only name. The Swift rename need not change the wire key. |
| Presence | A required String rejects absence with keyNotFound, null with valueNotFound, and a number with typeMismatch; an optional String accepts absence/null as nil and still rejects the number. contains distinguishes absence from present null, and decodeNil distinguishes present null from a string. |
| Missing-only Boolean default | Hints defaults to true for an absent key, preserves false, and rejects null/string values. The installed Foundation build reports typeMismatch for the Boolean null case. The published table is explicitly String-specific. |
| Stored default | Page.size = 20 does not supply a synthesized decoding fallback: empty JSON fails keyNotFound; an explicit size 12 decodes. |
| Enum shape | RawMode.byName encodes as a string and rejects manual. AssociatedMode.unknown(manual) produces a nested unknown/_0 object and rejects a bare manual string. JSON objects are compared structurally, without key-order assumptions. |
| Canonical probe | sortOrder/name reports missing: sortMode; sortMode/name reports invalid value: sortMode; sortMode/byName decodes. |
| Accepted migration fixtures | Twelve fixtures cover unversioned/explicit-v1/current-v2 data, missing legacy values, known and unknown strings under either key, current-key precedence over a conflicting/null legacy key, false hints, and an unknown JSON member. Output has exactly schemaVersion, sortMode, and showHints, with version 2; decoding that output preserves the modeled value. |
| Rejections | Fourteen fixtures cover null/wrong-type preferred and legacy values, missing current key, invalid/null/unsupported versions, null/wrong-type hints, an incorrect outer shape, and malformed JSON. Migration throws and the original in-memory Data is unchanged. This does not test a file-writing transaction. |
| Enum normalization | All six legacy/current recognized spellings normalize correctly and round-trip semantically. manual remains unknown and re-encodes as manual while its separate display fallback is byName. A manually constructed unknown(name) decodes back as byName; the chapter explicitly documents this constructible-state limitation. |
| Compatibility direction | The v1 reader decodes sortOrder/name but rejects the v2 writer's output for a missing sortOrder. It also rejects byPriority under the old key. New-reader success is not old-reader compatibility. An extra JSON member is intentionally absent after current-model re-encoding. |
| Lossy list | A mixed list retains IDs 1/2 and records indices 1/2. Empty, all-invalid, and unknown-member inputs follow the stated policy. Wrong outer shape/null rejects; a custom non-DecodingError propagates. Child-decoder acquisition failures are not swallowed. |
| Key strategies | The matching snake-case decoder/encoder pair round-trips and writes display_name. A raw display_name CodingKey works with the default strategy but fails when incoming keys are additionally converted to camel case. |

The migration, lossy-list, and strategy drivers execute under both default-isolation configurations. The other five drivers execute under the standard nonisolated baseline. All runtime checks use macOS 27; iOS and older deployment targets were compiled only. No older compiler, older Foundation runtime, cross-platform runtime, actual disk write/recovery, extension rollout, or sync conflict was exercised.

The initial suite incorrectly expected valueNotFound for the Boolean null case. b9-initial-results.json preserves those three assertion failures (the presence driver and both migration drivers); b9-diagnostic-results.json records focused output identifying typeMismatch. The three affected drivers were corrected and rerun successfully; unrelated passing checks were not repeated. No production decoding logic changed to accommodate the harness expectation.

Temporary check_b9.py and b9-published-results.json retain source assembly, exact commands, semantic expectations, and output. Fence 7 includes the SortMode declarations from fence 6; other fences compile independently with Foundation imported as needed by drivers. Tests and executables remain under /private/tmp/swift64-interview-validation, with bounded execution and core dumps disabled. The repository remains Markdown-only.

## B10 — API contracts, ownership, and public client usability

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 11 Swift fences have compile dispositions for macOS 13, iOS 18, and default MainActor plus NonisolatedNonsendingByDefault. There are 47 final compiler configurations, including two library-module builds and seven successful executions. Positive examples are warning-free. Negative examples reject for their intended type, ownership, mutation, or access-control boundary.

| Area | Observed result |
| --- | --- |
| Ordered chaining | Mapping before filtering returns [4, 6]; reversing those operations returns an empty array. Order-dependence is documented behavior, not an inherent fluent-API flaw. |
| Sendability suppression | Copying LocalOptions and mutating the copy leaves the original independent, giving 4. A generic Sendable requirement rejects the explicitly suppressed type in all three configurations. ~Sendable does not imply ~Copyable. |
| Ownership | Borrowing preview followed by consuming finalization returns Document(title: Notes). Reusing the noncopyable draft rejects during object compilation in all three configurations. A focused variant removing ~Copyable compiles and returns Notes after the same finalization/reuse sequence. |
| Mutation naming | mutationProbe returns three true predicates; sorting the returned copy leaves the original unchanged until sort is called. A let binding permits sorted and rejects sort. |
| Public construction | Separate consumer modules construct all image and payment models using their explicit public initializers. A public MemberwiseOnly fixture with only a public stored property still has an inaccessible synthesized initializer. Both isolation configurations verify that rejection. |
| Domain constraints | The client rejects a CardID where CustomerID is required. An empty CustomerID string is accepted as documented. PaymentAmount rejects zero/negative minor units and accepts Int64.max; external service limits are explicitly outside that local invariant. PixelSize rejects a zero width or negative height. |
| Defaults and hashing | PaymentOptions defaults to false/empty. ImageRequest defaults to aspectFit; equal requests collapse in a Set while changed mode and changed pixel size remain distinct (three values). No cache freshness or durable hash claim is made. |
| Actor service | A temporary actor conforms to PaymentServing. Through an existential service, customer lookup, amount/options, both deletion modes, and returned receipt preserve the requested values. The helper propagates an injected missing error and an injected CancellationError unchanged. Checked Sendable generic uses accept the request, customer, amount, and options. |

Both complete library/client pairs compile and execute under the standard and alternate isolation policies. The actor is an in-memory test double; it performs no networking or payment operation. The injected CancellationError verifies error propagation only, not cancellation of a live task or rollback of a remote side effect. No image is rendered, no physical allocation limit is established, and no public-library binary-evolution claim follows from the source consumer test.

The three initial sendability-negative assertions matched an incorrect diagnostic substring (omitting the word the); compilation had already rejected the source as intended. b10-initial-results.json preserves that harness mismatch. The matcher was corrected and only those three configurations were rerun. All 47 final expectations pass.

Temporary check_b10.py and b10-published-results.json retain exact source assembly, commands, expected failures, and output. Dependencies are LocalOptions for its rejection, Draft/Document for use-after-consume, shared payment values for both service protocols, and those values plus PaymentServing for loadCustomer. The public library combines the image request, payment values, and service protocol, plus the isolated MemberwiseOnly access-control fixture. Consumers import the emitted module and link its object file for execution. Core dumps are disabled and compiler/runtime processes are bounded.

All runtime execution was on macOS 27; older deployment targets and iOS were compiled only. Detailed Ref/MutableRef behavior and OS 27 availability remain the previously checked C7/C12 material; B10 adds API-selection guidance without claiming another runtime reference test. No older compiler, other-platform runtime, actual UI context, service retry/idempotency implementation, or ABI-compatibility test was run. The repository remains Markdown-only.

## B11 — resilience, definition visibility, and old-client replacement

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 13 Swift fences have compile dispositions for macOS 13, iOS 18, and default MainActor plus NonisolatedNonsendingByDefault. The client fences compile against a real separately emitted DeliveryAPI module. Across the final, focused, and boundary records there are 102 compiler configurations, including one optimized SIL emission/inspection, and seven successful executions. Eight additional interface/symbol inspections check the already-built artifacts. Expected warning variants are explicit; ordinary positive sources are warning-free.

### Enum and package boundaries

The focused matrix builds five Shipping modules with -package-name DeliveryKit: plain non-resilient, plain resilient, explicit nonexhaustive, nonexhaustive(warn), and frozen/resilient. A separate client handles all known cases, supplies an unknown fallback, uses the same package identity, or intentionally omits a known case.

| Configuration | Observed result |
| --- | --- |
| Plain non-resilient / frozen resilient | External all-known-case switches without fallback compile. |
| Plain resilient / explicit nonexhaustive | An external switch lacking fallback is rejected for possible unknown cases. |
| nonexhaustive(warn) | The same external missing-fallback source compiles with a warning. A focused missing-known-case source without any fallback still errors. |
| Same package | Exhaustive all-known-case switches compile in every tested module variant. The module's own exhaustive helper also compiles. |
| Unknown fallback | All-known-case switches with @unknown default compile. Omitting a known case with that fallback warns. A plain default version is warning-free. |
| New published case | Recompiling the chapter's exact statusLabel against DeliveryAPI v2 warns specifically about delayed under both isolation configurations. |
| Invalid attributes | Combining frozen and nonexhaustive is rejected. |

A focused -warnings-as-errors check on the missing-known-case warning still emits a warning and exits successfully. The initial expectation of an error is retained in b11-initial-results.json; the final expectation was corrected and only that check rerun. This observation is linked from F5. It does not imply that the missing-future-fallback native error is suppressible or that every diagnostic ignores promotion.

### Published types and visibility controls

| Area | Observed result |
| --- | --- |
| Token versions | Both the private String representation and replacement private reference-storage/generation representation compile under all three target/settings configurations. |
| Frozen storage | A private HiddenStorage field in a public frozen struct fails ABI visibility requirements. Point's public stored x/y layout appears in the generated interface. Token's rawValue/storage/generation do not. |
| Inlinable helper | The usableFromInline normalized helper supports public score, which returns 0 for -2 and 3 for 3. Direct client source cannot name normalized. A private unannotated helper reference from an inlinable body is rejected. |
| Export helper limit | Both internal and private helpers annotated only with export(interface) are rejected when referenced from an inlinable function. This differs from the broader helper eligibility described in SE-0497. The chapter keeps the working usableFromInline pattern. |
| Export syntax | Interface and implementation modes each compile. Combining both arguments or combining implementation export with inlinable is rejected. |
| Public mandatory inlining | A public inline(always) body calling an ordinary internal helper fails inlinable restrictions; the old inline(__always) spelling accepts that source. Recursive mandatory inlining fails with an inlining-cycle diagnostic. |
| Explicit interface plus inline(always) | The checked function can call a private helper. Its emitted interface contains neither that helper nor the function body; explicit visibility changes the otherwise implied body exposure. |
| Raw status code | pending and delivered have known labels; returned retains its raw string and maps to unknown. No Codable or whole-payload preservation is implied. |

### Binary replacement and module interface

Two complete v1/v2 library pairs are built with -O -enable-library-evolution, one per isolation configuration. Each v1 client is compiled once, then run with the v1 and v2 dynamic libraries. The SHA-256 digest before/after each execution and across both loaded library versions is unchanged.

| Loaded version | Both unchanged optimized clients print |
| --- | --- |
| v1 | abc pending 1 1 1 1 2 |
| v2 | abc unknown 2 1 1 1 2 |

The new Token storage preserves the accessor result. The expanded resilient enum reaches the old client's unknown branch. libraryRevision uses the updated interface-backed implementation; clientRevision, visibleRevision, and forcedRevision retain the bodies compiled into these clients. The visibleRevision observation is an optimizer result for this fixture, not a guarantee about every inlinable call.

The textual interfaces expose clientRevision/visibleRevision/forcedRevision bodies, hide libraryRevision/countEqual bodies, and omit Token's private representation. nm -gU finds exported entry points for libraryRevision, visibleRevision, forcedRevision, and only the generic countEqual entry point; clientRevision has no exported library symbol. Exact Token and Point interface sections are inspected separately to avoid confusing a different type's public rawValue with Token storage.

An additional client imports from a directory containing only the v1 swiftinterface, using a fresh module cache, then runs against the v2 library with the same expected output. This is a same-compiler textual-interface import test. No older compiler is inferred to support the new attribute syntax. Two further consumer executions verify Point, score, both Axis cases, raw status codes, and countEqual on String arrays, Int ArraySlice, and empty Int arrays.

The dynamic builds use -emit-library -emit-module -emit-module-interface-path and a stable @rpath/libDeliveryAPI.dylib install name. Consumers link with -I/-L/-lDeliveryAPI and an rpath to a temporary runtime directory; only that directory's dylib changes between executions. These commands, sources, executable hashes, and outputs are retained in the temporary result files.

### Specialization evidence

The exact published countEqual source compiles with specialized(where S == [Int]). The misspelled specialize attribute is rejected, as is a partial constraint leaving U unspecified in a two-parameter function. The optimized SIL inspection is scoped to countEqual's function body: it checks S and Array<Int> metatypes, an exact-type comparison/branch, the generic witness-based fallback, and array element loads with concrete integer equality in the optimized path. That path is folded into the generic function in the checked output; no separate exported specialized ABI entry point is claimed. Runtime fixtures preserve the generic results. No benchmark or timing improvement was measured.

### Reproduction and limits

Temporary check_b11.py and b11-published-results.json hold 66 result rows (58 compiler configurations, including SIL; four standalone binary-swap executions; four interface/symbol inspections). check_b11_boundaries.py classifies 41 existing focused compiler results without rerunning passing probes and adds seven boundary rows (three compiler configurations and four precise interface/symbol inspections). b11-focused-results.json and b11-boundary-results.json retain those passing expectations. The combined compiler count is 58 + 41 + 3 = 102; three compiler rows also execute clients, giving seven total executions with the four swaps.

Exploratory b11_preflight.py/b11-preflight-results.json, b11-export-extra-results.json, and b11_dynamic_preflight.py/b11-dynamic-preflight-results.json preserve the initial decisions and a preliminary binary swap. Final validation uses the current chapter fences, real module boundaries, and bounded processes with core dumps disabled. No temporary code, packages, scripts, or binaries were added to the repository.

All runtime execution is on macOS 27 with the same Swift 6.4 toolchain. Older deployment targets and iOS compile only. No older runtime/compiler, actual iOS framework distribution, non-Apple runtime, Embedded build, production ABI baseline, security fix rollout, or workload performance claim was tested. The binary replacement demonstrates these specific changes; it is not a general proof for arbitrary future library changes.

## B12 — parameter packs, evaluation, and deployment

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 14 Swift fences have compile dispositions for macOS 13, iOS 16, and default MainActor plus NonisolatedNonsendingByDefault. There are 72 final compiler configurations: 42 published-fence checks, 24 focused rejection checks, four nominal-type availability checks, and two complete semantic drivers. Both drivers execute successfully on macOS 27. Positive checks are warning-free; deliberate failures match their intended boundary.

| Area | Observed result |
| --- | --- |
| Arity and forwarding | Empty tuple/unpack/forwarding results are Void, one result is the scalar, and multiple results retain their heterogeneous types. A concrete tuple or array passed to packCount counts as one argument. Empty, one-, two-, and seven-argument counts pass. Diagnostic type names for Int/String/Bool match. |
| Shared shapes | makePairs supports empty, scalar, and heterogeneous paired results. Different corresponding types are allowed, while mismatched pack lengths fail. A following unlabeled scalar parameter is rejected. |
| Iteration evaluation | firstTwo handles zero and one values and stops after two. The instrumented three-argument call logs all three argument constructions before entering the function, then only the first two visited pattern evaluations. |
| Nominal availability | Route with its availability annotations compiles at older deployment targets. Removing those annotations rejects macOS 13/iOS 16 and accepts macOS 14/iOS 17. The Renderer typealias and all other positive function examples compile for macOS 13/iOS 16. Host drivers return the stated route/renderer strings. |
| Erasure and overloads | Uncontextualized mixed collect arguments fail; the explicit Any result context preserves the two values through erasure. ValueBox<Int> cannot be assigned to ValueBox<Any>, while explicit reconstruction works. The published overloads select zero/scalar/pack; the focused pair of differently labeled pack overloads has an ambiguous empty call. |
| Current restrictions | Pack enums, nominal types with two packs, and inheritance from a variadic generic class reject. Passing a noncopyable Token to packCount fails its implicit Copyable requirement. |
| Equality | Empty/scalar/heterogeneous/seven-element cases return the expected results. Instrumented Equatable values record only the first comparison for an initial mismatch, the first two for a later mismatch, and all three for equal values. Incompatible element types and lengths reject. |
| Parser behavior | The chapter examples parse 42 and -12 into the stated tuples, propagate invalidInteger, and return (3, true) for parserEdges. A seven-component parser preserves every typed result. No parse occurs during construction. Successful components run a/b/c; a throw from b leaves the a/b log and skips c. |
| Parser inference | Context supplies Input for zero components; one component produces a scalar Output. A context-free empty call fails inference, and mixing String/Int input parsers fails their shared Input requirement. |

The two complete drivers use the exact positive chapter declarations under both isolation policies. They combine assertions for output values with instrumented argument, iteration, equality, and parsing effects. All effects are synchronous local test state; no actor transfer, asynchronous scheduling, networking, URL escaping, allocation elimination, or timing improvement is claimed.

The initial run caught an invalid nonisolated modifier on the Renderer typealias; the declaration was corrected in the chapter. It also caught two temporary-driver mistakes (a nonisolated @main method and a function-reference spelling used instead of the empty makePairs call). Two negative diagnostic matchers were adjusted to the actual shared-Input note and required-label wording. b12-initial-results.json retains all nine failed expectations; only those affected configurations were rerun. The final 72 expectations pass.

Temporary check_b12.py and b12-published-results.json retain exact source assembly, commands, expected failures, and runtime output. Fence 1 includes fence 0's tuple function; fences 12/13 include fence 11's Parser/zipParsers declarations. Other published fences compile independently. Preliminary b12_preflight.py/b12-preflight-results.json evidence is retained separately and is not added to the final count. Compiler/runtime processes are bounded and core dumps disabled.

All execution uses macOS 27 with the same Swift 6.4 compiler. Older deployment targets and iOS compile only; no older Apple runtime, older compiler, Linux runtime, ABI migration, builder implementation, or workload benchmark was tested. No temporary code, packages, scripts, or binaries were added to the Markdown repository.

## B14 — result-builder transforms, composition, and 6.4 parsing

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 16 Swift fences have compile dispositions for macOS 13, iOS 16, and default MainActor plus NonisolatedNonsendingByDefault. There are 90 final compiler configurations and eight successful executions. The directly attributed return example intentionally compiles with a builder-disabled warning in each configuration; the missing-loop example intentionally fails. Other ordinary positive chapter examples are warning-free.

| Area | Observed result |
| --- | --- |
| List composition | Both listExample branches produce their stated arrays. Empty lists, computed/tuple locals, optional branches, loops, prebuilt lists, and every branch of a switch preserve the expected values/order. Local declarations themselves do not contribute entries. |
| Explicit return | explicitReturnExample compiles warning-free and returns A/B. The directly attributed function compiles with a warning that return disables the builder. A wrong String return in an inline generic builder call produces a compiler diagnostic failure; a separately typed ordinary closure gives the expected conversion error. |
| Hook boundaries | The published minimal builder fails for a for-in loop without buildArray. Removing optional/either hooks fails the corresponding construct. Void-valued print is rejected by the String builder. while, break, continue, and guard are rejected inside transformed bodies. |
| Partial blocks | Empty, one-, and three-component totals are total=0, total=42, and total=6. The nonempty sentinel buildBlock overload is not selected. Removing both buildBlock overloads makes the empty body fail even though the partial-block pair remains. |
| Partial evaluation order | A focused actor-isolated trace records expression:1/2/3 before first:1, next:2, next:3, and final:6. The explicit tracing fixture runs under both policies. It is distinct from the uninstrumented published SumBuilder. |
| Packs | Empty, scalar, three-, and seven-position builder results preserve their types and values. A missing optional hook in an inline generic call produces a diagnostic failure; the directly attributed counterpart diagnoses buildOptional. |
| New array/dictionary spelling | The user Array initializer makes direct and .init spellings both yield A/B. Empty and conditional/loop array bodies work. Without that initializer, direct syntax fails type checking. The ordinary dictionary closure yields attempts=2 using the supplied last-value-wins policy; an empty dictionary works. |
| Parser edge | A bare closure after an array-literal declaration is treated as a call on the array and rejected. Naming the action works; the focused semicolon repair also produces one closure returning 3 under both policies. A separate user callAsFunction on Array makes [1,2,3] with a trailing transform produce [2,4,6]. |
| Tree model | Logged-in/out pages match their documented descriptions. Builder and plainPage results are equal for empty, ordinary, repeated, and empty-string names. Empty elements, prebuilt child arrays, and either branches preserve the expected tree. No escaping/HTML serialization is inferred. |
| Route model | Both sampleRoutes branches match the printed order. Empty groups/lists, nested prefixes, prebuilt route arrays, and all four parent/child authentication combinations pass. Removing the list-expression conversion reproduces the group result rejection. The original chapter's route example is also retained as a rejected preflight source. |
| Effects and errors | Component calls run only in selected branches/iterations. A thrown failure leaves the first/failure log and skips the later component. A stored closure contributes no action effect until invoked. The underlying makeList implementation calls its body once; no rollback is provided. |
| Availability / SwiftUI | A guarded modernText branch compiles at macOS 13/iOS 16 and runs the modern branch on macOS 27. The explicit MainActor UserList compiles under all three configurations. Replacing its ForEach view with a source for-in loop fails with unsupported ViewBuilder control flow on both Apple deployment targets. |

The complete semantic drivers run under both isolation policies. Additional executions cover partial-block tracing twice, the statement-separator repair twice, the guarded availability branch once, and a callable-array fixture once, giving eight final executions. No SwiftUI app was launched or rendered; view layout, identity updates, and older runtime behavior remain outside this validation.

The first final-suite attempt contained 86 rows. Two complete drivers passed their value assertions but emitted warnings because a switch used a literal condition; those drivers were changed to call a parameterized helper and now test all branches warning-free. Ten negative expectations matched overly specific diagnostic wording. Their already-produced output was reclassified without rerunning unchanged sources: break/continue/ViewBuilder report unsupported control flow, and the two malformed generic expressions report failed to produce diagnostic. Four focused ordinary-return/directly-attributed-tuple checks were then added to establish the intended semantic errors. b14-initial-results.json preserves the original expectations; all 90 final expectations pass. The diagnostic failures return an error status, not a compiler process crash.

Temporary check_b14.py/b14-published-results.json retain the exact sources, commands, warning/error expectations, and outputs. Dependency assembly adds ListBuilder/makeList to its dependent examples, Node declarations to plainPage, and ListBuilder plus Route helpers to sampleRoutes. SwiftUI remains an independent imported fence. b14_preflight.py/b14-preflight-results.json preserve seven exploratory compiler probes (four executions); these are not included in the final count. Compiler/runtime processes are bounded and core dumps disabled.

All execution was on macOS 27 with Swift 6.4. Older deployment targets and iOS are compile checks only. No older compiler was run to demonstrate .init spelling; the historical syntax and new parser feature are identified from SE-0508, while both forms compile with this toolchain. No server, networking, authentication middleware, URL normalization, HTML serializer, cross-platform runtime, source-compatibility corpus, or performance benchmark was exercised. The repository remains Markdown-only.

## A7 — function contracts, defaults, and external request clients

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 19 Swift fences have compile dispositions for macOS 13, iOS 16, and default MainActor plus NonisolatedNonsendingByDefault. The DefaultsAPI consumer fence imports an actual separately emitted module for each configuration. The final records contain 96 compiler configurations and ten successful executions. Positive examples are warning-free; deliberate failures match their intended declaration, ownership, access, or isolation boundary.

| Area | Observed result |
| --- | --- |
| Labels / function values | Full move(from:to:) and move(from:by:) references produce 5 and 9. Bare move is ambiguous even with their shared function type. Labeled function-type syntax rejects. greet's direct/wrapped defaults yield Hello Guest while its function value takes Ada explicitly; invoking that value without an argument rejects. |
| Variadics | Mixed Any formatting, custom separator, zero items, and a single array-valued Any argument match the documented strings. sum returns 0/6 for zero/three arguments and rejects one [Int] argument. |
| Trailing closures | Explicit and multiple-trailing forms both return value=42. An instrumented wrapper invokes process then completion once. The single trailing closure on choose supplies first; explicitly supplying second gives the opposite pair. |
| Array / optional syntax | The custom ordinary generator returns [0,1,2] through both direct and .init calls; the terminating nil requires a fourth invocation. An immediately nil generator returns an empty array. Optional existential/opaque functions return 42/none and accept other conforming values as documented. |
| Ownership / mutation | Borrowing then consuming Draft returns two Notes strings; reuse after consume rejects. Integer clamping covers above/below/in-range values. A computed property records get then set:10 and writes back storage. add(&value, amount: value) returns 6. Two inout aliases and an escaping inout capture reject. |
| Async inout / actors | localMutation returns 2 under both settings. Passing Counter's actor-isolated stored value to the async helper rejects under both settings. No claim is made that Task.yield always suspends or changes threads. |
| Isolated defaults | The awaited outside caller returns Context 7; a MainActor caller with an explicit UIContext returns Context 8. A nonisolated declaration's isolated default and an unawaited outside call reject. An attempted await inside a default expression is rejected. |
| Retained function contracts | Converting a MainActor function to an ordinary synchronous function loses a global actor and rejects; converting clamp's inout function to a by-value function type rejects. |
| Default ordering / dispatch | The exact event trace is explicit/default/body, again/default/body, supplied/second/body. The provided third argument pair skips default evaluation. Derived/base/function-value calls return derived:2, derived:1, derived:3. |

### Real library replacement

For each isolation configuration, the DefaultsAPI v1 and v2 libraries are built with -O -enable-library-evolution -emit-library -emit-module -emit-module-interface-path and the stable install name @rpath/libDefaultsAPI.dylib. V2 changes the three policy literals from 20 to 100 without changing declarations. The v1 client is compiled once, and its SHA-256 digest is identical across its v1/v2 runs; only the temporary runtime directory's dylib changes. A separate new client is then compiled against v2.

| Consumer / library | Both configurations print |
| --- | --- |
| Old client, v1 library | 20 20 20 7 |
| Same old client, v2 library | 20 100 100 7 |
| Newly compiled client, v2 library | 100 100 100 7 |

The literal default remains old in the unchanged client; default expressions calling the library getter/helper observe their new values. The explicit argument remains 7 throughout. The generated v1 interfaces were inspected and contain the default expressions/declarations without publishing the getter/helper function bodies. This evidence is scoped to the checked ordinary public definitions and resilient build; client-visible body or optimization changes need separate analysis.

### Public request construction and forwarding

Two separately emitted RequestAPI modules contain the exact request model, constructors, protocol, and helper. External consumers import Foundation/RequestAPI, include the chapter's upload helper, and supply an in-memory actor transport. They construct every model using public initializers, check Sendable use, and verify the default GET description with timeout 30 seconds, one total attempt, protocol cache policy, and no authentication.

Explicit options retain timeout 5 seconds, three total attempts, reload cache policy, and a fixture bearer-token value. Options reject zero/negative/nonfinite timeout values and nonpositive attempt limits. All four modeled methods preserve their method/body values; direct and helper-constructed descriptions compare equal. The transport receives the complete POST/default descriptions, returns Data([4,2]) unchanged, and propagates an injected domain error and CancellationError unchanged.

These are forwarding/error-propagation checks. They do not test a network request, live task cancellation, timeout enforcement, retry/backoff scheduling, cache behavior, authentication, URL validation, or HTTP completeness. The method/body restrictions are the example application's model, not a claim about every legal HTTP request.

### Reproduction and limits

Temporary check_a7.py/a7-published-results.json retain 88 compiler configurations and four executions (two complete semantic drivers and two separate request clients). check_a7_defaults.py/a7-defaults-results.json add eight compiler configurations and six library/client executions, for 96 and ten respectively. Module dependencies, source assembly, exact commands, outputs, and old-client hashes are retained. The preliminary a7_preflight.py/a7-preflight-results.json contains six compiler probes and five executions, outside the final totals.

The initial published suite matched an overly specific missing-await diagnostic in two configurations. Compilation already rejected both calls for calling a MainActor function from outside its actor. The matcher was corrected against retained output without repeating unchanged compiles; a7-initial-results.json preserves the first expectations. No chapter code changed because of that harness mismatch.

All execution uses macOS 27 and the same Swift 6.4 compiler. macOS 13/iOS 16 deployment and alternate isolation configurations are compile checks where no host execution is stated. No older compiler/runtime, other-platform Foundation runtime, production framework distribution, cross-version optimization matrix, or benchmark was tested. Temporary sources, modules, libraries, scripts, and binaries remain outside the Markdown repository.

## A8 — captures, weak references, callback ownership, and task lifetime

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 16 Swift fences have compile dispositions for macOS 13, iOS 16, and default MainActor plus NonisolatedNonsendingByDefault. The final record contains 75 compiler configurations and seven executions: six successful semantic runs and one expected isolated unowned-access trap. The nested strong-outer/weak-inner example intentionally emits ImplicitStrongCapture; ordinary positive sources are warning-free.

| Area | Observed result |
| --- | --- |
| Callback lifetime | The nonescaping and immediate-escaping helpers invoke during their calls. The explicitly caller-isolated async helper returns 3. CallbackSlot stores both direct escaping and optional function values, fires once per registration, clears explicitly, and preserves a reentrantly installed replacement. Storing a nonescaping parameter rejects. |
| Variable/value/reference captures | A counter and its alias share 1/2/3 while a separate counter starts at 1. captureExample returns [10,1,20], distinguishing shared variable storage, a value snapshot, and a captured class reference. No allocation claim follows. |
| Strong / weak / bound / unowned | A strong reader retains CaptureOwner after the original reference is removed; releasing that reader lets the weak reader observe nil. A bound read method similarly retains its instance until the method value is released. An unowned reader works while the owner is explicitly kept alive; a separate post-release invocation terminates with the expected unowned-reference diagnostic. |
| Explicit self | The published explicit-strong and unwrapped-weak forms compile. Removing the explicit capture or reading an optional weak self implicitly before unwrapping rejects. |
| Immutable weak controls | WeakLink and read-only Sendable weakReader compile with both ImmutableWeakCaptures settings under both isolation policies. The baseline resettable reader returns 9 once, then nil. Enabling the feature rejects reassignment to that explicit weak capture. Assigning to weak let also rejects. |
| Sendability / isolation | A weak capture of non-Sendable CountBox still fails a Sendable closure requirement. Mutable local capture fails; a value snapshot succeeds. The explicitly MainActor Sendable reader mutates ActorCounter serially to 1/2. An outside synchronous callback-delivery call rejects actor isolation. |
| Autoclosure | Disabled/one/twice invocation evaluates the tracked expression zero/one/two times. The escaped autoclosure is not evaluated on construction and evaluates on each subsequent call. A mutable captured string is read after its later assignment. Returning a nonescaping autoclosure rejects. |
| Nested / stored cycles | The strong outer closure retains NestedOwner even before its weak inner callback exists; the diagnostic is observed and clearing outer breaks the cycle. Weak captures at both boundaries allow release and later harmless callback invocation. Screen's explicit closure and bound method both retain; its weak callback permits release. Every intentional cycle is cleared in the driver. |
| Callback presenter | Delivery updates the presenter's name on MainActor. Explicit unbind stops this registration's delivery; copied weak callbacks and a source surviving the presenter tolerate owner release. This tests handler removal, not producer cancellation. |

### Async task and lifecycle policy

The exact ProfileModel declarations run under both isolation policies against a temporary ManualService actor. Each request suspends in a checked continuation and announces its ID through another continuation. Tests explicitly resolve every request and await every returned task; no sleeps or race-prone timing guesses establish the outcomes.

- Success publishes loading then loaded(Ada); an injected domain error publishes failed(unavailable).
- Cancelling the task followed by either success or a domain error yields idle. An explicit CancellationError also yields idle.
- stop resets immediately, and a later successful service result is ignored.
- Replacing a request allows the new result to win; late old success and late old failure cannot overwrite it.
- Stopping before the task starts skips the service call entirely.
- A synchronous loading observer can call stop; no later handle assignment restores the cancelled work. A loaded observer can start a replacement, which remains stoppable after the earlier task completes.
- Releasing ProfileModel while its service is suspended permits the weak observer to become nil. Deinit requests cancellation, and the test then resolves the service so the remaining task can finish.
- A contrasting temporary EarlyPromotion helper unwraps self before awaiting and uses self afterward. Releasing its external owner leaves it alive during suspension; resolving and awaiting completion then permits release.
- All pending request and listener counts are zero at the end of each driver.

These checks demonstrate the chosen state, lifetime, and cooperative-cancellation policy. They do not force a noncooperative service to terminate. The service is a deterministic in-memory fixture, not a network implementation. The UIKit controller compiles for iOS 16 under both isolation policies; canImport excludes UIKit on macOS. No simulator or real screen navigation was run, and disappear/reappear behavior is not runtime-certified.

### Settings-sensitive async closure boundary

The first complete driver exposed a real default-setting distinction: a plain nonisolated async receiver attempted to transfer a MainActor-inherited non-Sendable closure under the baseline execution policy. It failed with SendingRisksDataRace. The same source works with NonisolatedNonsendingByDefault. The chapter's performDuring now explicitly uses nonisolated(nonsending), so its intended caller-isolated contract works under both policies.

A focused old-boundary source remains in the final record: baseline rejection and alternate-policy execution returning 3. The three changed published-fence configurations and both complete drivers were rerun after correcting the declaration. Temporary initialized weak observers were changed to weak let to remove WeakMutability warnings; observers assigned later remain weak var. The two optional-escaping diagnostic matchers were corrected from retained output without repeating unchanged sources. a8-initial-results.json preserves the initial 70-row record.

### Reproduction and limits

Temporary check_a8.py/a8-published-results.json retain exact chapter assembly, sources, commands, expected errors/warnings, runtime assertions, and the isolated trap output. Dependencies are SharedOwner for resettableReader, Profile/ProfileLoading for ProfileModel, and those declarations plus ProfileModel for the conditional UIKit controller. All 75 final expectations pass; one run deliberately terminates for invalid checked unowned access. Compiler/runtime processes are bounded, and core dumps are disabled. a8_preflight.py/a8-preflight-results.json retains five preliminary compiler probes and three executions outside the final totals.

All execution uses macOS 27 with Swift 6.4. Older deployment targets and iOS are compile checks. No older runtime/compiler, Linux runtime, allocation/SIL measurement, optimization matrix, autorelease behavior, real networking, UIKit lifecycle execution, or multithreaded CallbackSlot safety was tested. The repository remains Markdown-only.

## A14 — operators, evaluation, literals, and validation policy

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 13 Swift fences have compile dispositions for macOS 13, iOS 16, and default MainActor plus NonisolatedNonsendingByDefault. The final record contains 70 compiler configurations and ten executions: seven successful semantic/control runs and three expected isolated traps. Positive compilation is warning-free. Literal protocol requirements were also inspected in the installed arm64e macOS Swift.swiftinterface.

| Area | Observed result |
| --- | --- |
| Grid offsets | Addition produces (6,2), leaves its operands unchanged, and the supplied compound operator updates its left operand. A separate type supplying only + rejects +=; an unrelated FloatingPoint candidate appears in that diagnostic, not an automatically synthesized implementation. |
| Overflow | UInt8 wrapping/reporting returns (0,0,true). addingExactly succeeds for positive/negative ordinary inputs and throws on Int.max+1 and Int.min-1. The same runtime-input checked-add source terminates with SIGTRAP (return code -5, no textual diagnostic) at both -Onone and -O; both binaries return 21 for control input 20. No -Ounchecked execution was attempted. |
| Grouping | groupingExample produces the two published expression strings. Explicitly left-parenthesized composition produces ((x\|y)\|z), different from the right-associative form. An unspecified group's repeated operator rejects nonassociative chaining; unrelated custom groups reject mixing without parentheses. |
| Predicate effects | Both predicate factories execute before ready. False matching skips the second predicate; true matching invokes it. Every eligibility input combination agrees between the named/symbolic forms, and instrumented closures match the intended adult/verified/override short-circuit trace. The alternate parenthesization rejects a nonadult override and demonstrates a different policy. |
| Ordinary versus autoclosure operands | The exact rubric probe returns eager then needed. Four Bool combinations confirm that the ordinary right operand executes once regardless of lhs, while the autoclosure right operand executes zero or one times according to lhs. |
| Contextual literals | The published return type checks String/Double/TaggedText/ByteToken inference. Empty/Unicode strings and byte values 0/255 succeed. 256 rejects the UInt8 literal representation. Assigning a stored String to TaggedText or StaticString and a stored UInt8 to ByteToken rejects. Explicit TaggedText construction, including a String-interpolated argument, succeeds; direct contextual interpolation to TaggedText rejects. |
| Overloads | Two unrelated string-literal wrapper overloads make a bare literal ambiguous. Explicit construction selects the intended overloads and prints 1 2 under both isolation policies. |
| Literal failure | The failable initializer fails the non-failable requirement. The throwing example explicitly fixes its literal associated types so the diagnostic identifies that the candidate throws but the protocol does not allow it. A temporary FixturePath compiles with an invalid literal and traps at its runtime prefix precondition; the same binary succeeds with /ok. |
| Validated Percentage | 0,20,100 succeed. -1,101,Int.min,Int.max throw the expected outOfRange payload; no invalid instance is returned. |
| SE-0508 | The published ordinary user initializers return [1,2,3] and attempts:2. Explicit .init calls, empty containers, exactly-once builder closure invocation, and duplicate-key last-value-wins behavior pass. Removing those initializers rejects trailing closures; an array value without callAsFunction rejects calling, and a scalar String literal does not acquire trailing-closure support from a callAsFunction extension. |

### Reproduction and review adjustments

Temporary check_a14.py/a14-published-results.json retain the exact sources, commands, flags, expected diagnostics, assertions, bounded process results, and positive controls for each trap binary. Use -c with the common flags for compiler checks and -parse-as-library for the @main drivers. The invalid String conversion reuses TaggedText; predicateTrace and the eligibility exercise reuse Predicate and its two operators. The combined driver includes the shared &&& declaration once and keeps both its Bool and Predicate overloads. The rubric probe is independently compilable.

The initial 69-row attempt passed all ordinary runtime assertions but had eleven mismatched expectations. Six diagnostic matchers were corrected from retained output for the missing compound operator and missing Array/Dictionary initializers. Two overflow executions had already trapped correctly but produced no crash text, so their expectations now match SIGTRAP. The throwing-literal example initially produced associated-type inference diagnostics; its three aliases were added, the three published configurations rerun, and their matchers aligned with the resulting candidate-throws diagnostic. a14-initial-results.json preserves the initial evidence. A focused literal-precondition configuration and three successful trap-binary controls brought the final totals to 70 configurations and ten executions. All final expectations pass.

Execution used macOS 27 with Swift 6.4. Older Apple deployment targets and iOS were compiled, not run. No older compiler, Linux runtime, UI, production authorization system, public-library compatibility corpus, or performance benchmark was exercised. B14 retains the separate callable-array and standalone-closure migration-edge checks. This review does not claim that an operator symbol proves purity, Sendable safety, algebraic laws, or cost. Temporary sources, executables, and validation scripts remain outside the Markdown repository.

## A15 — type choice, unique cleanup, shared caches, and UI request ordering

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 11 Swift fences have compile dispositions for macOS 13, iOS 16, and default MainActor plus NonisolatedNonsendingByDefault. The final record contains 59 compiler configurations and eight successful executions. Every positive compilation is warning-free; the intentional failures diagnose the intended ownership, sendability, or isolation constraint. No compiler or runtime expectation failed during this chapter's validation.

| Area | Observed result |
| --- | --- |
| Value/domain/reference identity | Changing copied FeedItem.title preserves the original and the common ID but changes synthesized equality. The exact rubric probe returns A,B,new,new,true, showing a copied Profile and shared Session. Applying === to FeedItem rejects. |
| Class contracts | CancellationHandle aliases share one MainActor-isolated flag and a separately constructed handle remains false. Nonisolated access to its flag rejects. ImageDescriptor's immutable Sendable representation is read from a detached task. A mutable nonisolated Sendable class rejects; adding Sendable to the snapshot with Session also rejects. |
| Unique cleanup | leaseExample returns [7] after consuming finish. Reusing the lease fails during compilation with used-after-consume. A copyable struct deinitializer rejects. The example only records a synthetic release; it does not open an OS resource. |
| State representation | Idle, empty/previous loading, loaded, failure with previous data, and cancellation with previous data return the expected visibleValue. The conditional Equatable conformance supports exact state assertions. A Sendable but noncopyable Token fails LoadState's inherited Copyable payload requirement. |
| Actor access | The published external mutation and its await-assignment variant both reject. An outside synchronous increment rejects. The repaired async caller plus a synchronous same-actor twice method produce 3. |
| Cache and repository | Empty/absent data are distinct; insert replaces, remove is harmless when absent, and removeAll clears. Repository copies share the same actor identity. A first miss fetches once, a hit skips fetching, and reload fetches and replaces. Stub failure propagates without inserting. Generic and existential service calls both support the checked boundary. |
| UI state | Initial load, cache hit, reload, failure, and reset match their stated states. A real MainActor class is used; observation integration and task ownership are not inferred from the annotation. |

### Deterministic completion and cancellation

The temporary GatedFetcher actor records request IDs, suspends each response in a checked continuation, and exposes arrival continuations to the driver. No sleep establishes ordering. The exact published repository/model declarations run under both isolation policies.

- With prior Data([0]) visible, each reload publishes loading(previous: Data([0])).
- If the old request completes first, it cannot publish while the newer request is pending; the newer success ultimately displays and caches Data([2]).
- If the new request completes first and the old success arrives last, displayed state stays Data([2]) while the cache ends at Data([1]). A later repository cache hit reads that stored value. This deliberately demonstrates separate UI generation and cache insertion-order contracts.
- Late old failure and late success from a cancelled old task do not replace the newer display; the cache remains Data([2]).
- Reset during a pending request leaves the model idle after completion. Successful underlying work can still populate the cache, as documented.
- Active cancellation followed by either a value or domain failure produces cancelled(previous: Data([0])). An explicitly reported CancellationError has the same displayed result. Prior cache data remains unchanged in these gated cases because cancellation is observed before insertion.
- An uncancelled status(503) failure keeps prior data in failed(message: status(503), previous: Data([0])).
- Cancelling before the task begins invokes no fetcher and preserves previous data in the cancelled state.
- Two concurrent cache misses both reach the fetcher. Both returned values are checked; deliberately ordered insertions leave the second result in the cache. No request coalescing is claimed.
- Every started request is resolved, every task awaited, and pending/listener counts are zero after the gated scenarios.

These cases do not establish that cancellation rolls back every possible side effect. In particular, an insert completed before a later cancellation check remains possible. A per-model generation check also does not establish globally newest-request-wins caching.

### URLSession adapter with an in-memory transport fixture

The exact URLSessionImageFetcher is tested using a temporary URLProtocol subclass and an ephemeral session whose protocolClasses intercept the fixture URLs. The fixture has no shared mutable response configuration; it selects its response from each request URL. No external server or network is used. Both isolation policies run the same cases:

- HTTP 200, 201, and 299 return Data([42]); HTTP 300, 404, and 503 throw status with the matching code.
- An empty successful response returns empty Data.
- A non-HTTP URLResponse throws nonHTTPResponse.
- A fixture transport failure preserves URLError.cannotConnectToHost.

The fixture session is invalidated after use. This is byte/response validation, not image decoding, real network cancellation, redirects, TLS, HTTP caching, or server behavior. The conditional FoundationNetworking import is source portability context, not evidence of a Linux build.

### Reproduction and limits

Temporary check_a15.py/a15-published-results.json retain source assembly, all commands/flags, compiler diagnostics, and runtime assertions. Use -c with common flags for ownership checks; @main drivers additionally use -parse-as-library. The repository fence depends on ImageFetching and ImageCache; FeedImageModel additionally depends on FeedItem, LoadState, and ImageRepository. Other published fences are independent. The negative state-payload probe combines LoadState with a ~Copyable Sendable Token.

All runtime checks were on macOS 27 with Swift 6.4. Older deployment targets and iOS were compilation checks only. No older compiler, non-Apple runtime, UI rendering/Observation integration, allocation benchmark, real OS resource release, cache eviction/freshness policy, or load test was performed. All temporary code and executables remain outside the Markdown curriculum.

## A11 — source access, construction, packages, testable clients, and exports

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All 13 Swift fences have compiler dispositions for macOS 13, iOS 16, and default MainActor plus NonisolatedNonsendingByDefault. The final record contains 117 compiler configurations: 62 successful compilations and 55 intended rejections, including separately built module/library/client configurations. Sixteen executable runs pass. Positive compilation/linking is warning-free.

### Real boundaries, not comments within one file

AccessAPI, ShoppingCore, and ShoppingTestSupport are compiled as separate modules with -package-name SamplePackage. For macOS configurations the compiler emits both their Swift modules and dynamic libraries; iOS 16 is an emit-module compile check. Each client explicitly imports the relevant module and uses the recorded include/library paths. Ordinary external clients use OtherPackage. A second AccessAPI build uses -enable-testing in a distinct output directory. Standalone package-context rejection omits -package-name entirely.

| Area | Observed result |
| --- | --- |
| Private/fileprivate | LocalCounter's same-file extension increments and reads private value. An unrelated same-file helper reads fileprivate revision but cannot read value. Separate-file helpers/extensions cannot read revision/value. The successful runtime reports 1 1. |
| Public/open | An ordinary external subclass of PublicBase rejects; the same-module subclass overrides value and returns 2. Plugin subclasses OpenBase and overrides its open customize hook, so inherited render returns [plugin]. An external render override rejects. Final subclassing and open struct both reject. |
| Package construction | Same-package NameFormatter construction produces Dr. Ada. OtherPackage cannot name it. A separately compiled MissingInit module removes only the explicit package initializer; a same-package client then fails construction because synthesis remains internal. Missing compiler package context also rejects. |
| Signature/witness limits | The public function using package CacheKey fails. A public alias of an internal type and a public generic constraint using an internal protocol fail. A public type's insufficiently visible public-protocol witness fails. The package StubCartLoader witness is accepted. |
| ProgressState | Negative total is rejected by the initializer. Zero total permits a zero advance, refuses one, and stays zero. A total of 10 advances 3 then 7; negative/past-total requests fail without changing state. Int.max advances once exactly and rejects a further increment. External getters/public construction succeed. Direct setter access fails, and OtherPackage cannot call the package advance method. |
| SE-0502 | The rubric probe returns [0,7] and compiles without an experimental flag. Another source file in the same module constructs Draft(title:) and reads zero. An unrelated function's explicit revision argument is rejected; the same-file type extension uses the compatibility initializer successfully. An uninitialized private field and all-private explicit memberwise arguments remain rejected. The separate default initializer for all-defaulted private fields succeeds and prints 0:true. |
| Public synthesis | Another module can name PublicLabel, but its memberwise initializer remains inaccessible. Public type/property access does not imply public construction. |
| Export versus source names | Ordinary clients call publicAnswer and inlinableAnswer, receiving 42/7. They cannot name private implementationAnswer despite export(interface), or internal stableHelper despite usableFromInline. These are source-access checks plus successful linking, not a new symbol/interface or binary-swap audit; B11 contains that evidence. |
| Testable access | With a test-enabled module, an OtherPackage testable client reads internalAnswer/internalValue and overrides PublicBase.value to return 9. Private/fileprivate member reads still reject, as does subclassing FinalBase. Importing the ordinary module as testable rejects because it was not compiled for testing. |

### Shopping package exercise

The exact published ShoppingCore and ShoppingTestSupport compile as separate dependencies. The same-package client prints Keyboard, Mouse using package CartRow/makeRowStore. Package tests verify empty/nonempty row mapping, stub success including an empty result, and propagation of StubFailure.unavailable. An ordinary external client implements public CartLoading, constructs CartItem/CartStore through explicit public initializers, and receives the expected External item. OtherPackage cannot name CartRow, makeRowStore, or StubCartLoader. The app client imports only Core; test support is a separate test dependency.

This verifies the complete in-memory adapter and boundary design. It does not claim database persistence, networking, a shipped SwiftPM product graph, or ABI stability. The compiler's package identity flag tests the language access boundary; no temporary SwiftPM manifest is required for these probes.

### Adjustments revealed by validation

The first 115-row record exposed fourteen mismatched expectations. Under default MainActor, the initial Draft extension members were inferred actor-isolated despite the intended synchronous helper use; the published Draft and LocalCounter extension members now explicitly use nonisolated. That fixes an isolation issue separately from private access. The failing library emission also caused three dependent runtime links to fail; their clients were rebuilt after the corrected module.

The temporary linker initially repeated an identical rpath for clients linking multiple libraries. The driver now deduplicates it, and those successful-but-warning-bearing executions were rerun cleanly. Diagnostic matching was also corrected: inaccessible compatibility/private memberwise overloads can leave only an accessible shorter/default initializer in overload resolution, producing extra-argument diagnostics rather than the word inaccessible. The original all-private failure was reclassified from its retained diagnostics; the affected Draft examples were rerun after their source correction.

Forty-four affected or dependent configurations were rerun after these changes. Two focused default-initializer executions then brought the final record to 117 configurations and sixteen executions. a11-initial-results.json preserves the initial results; all final expectations pass. No unrelated chapter compiler matrix was rerun.

### Reproduction and limits

Temporary check_a11.py/a11-published-results.json retain complete sources, extra-file inputs, compiler/linker commands, module paths, package identities, testing flags, diagnostics, and output assertions. A11's Client/Plugin fences import AccessAPI; ShoppingTestSupport and PackageClient import ShoppingCore. Other fences are independent or assembled into their explicitly named library. Use -emit-module for compile-only clients, -emit-library with -emit-module-path for runtime dependencies, and -parse-as-library for @main clients. The records include install names and rpaths for the temporary dylibs.

All execution uses macOS 27 with Swift 6.4; macOS 13/iOS 16 are deployment compile targets, not older runtime evidence. No older compiler, Linux runtime, Xcode project, SwiftPM dependency resolver, security boundary, source-compatibility corpus, or performance claim was tested. The repository remains Markdown-only.

## A10 — extensions, retroactive diagnostics, module visibility, and owned DTOs

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All ten Swift fences have compile dispositions for macOS 13, iOS 16, and default MainActor plus NonisolatedNonsendingByDefault. The final record contains 104 compiler configurations: 30 intended rejections, nine expected-warning configurations, and 65 warning-free successful compilations. Sixteen executable runs pass.

### Owned behavior, file placement, and diagnostics

| Area | Observed result |
| --- | --- |
| Owned User | Computed displayName and static anonymousName behave as documented. Equal IDs with different names are unequal values; Set retains the two distinct values. Codable round-trips the model. An extension's stored instance property rejects. |
| Constrained/conditional behavior | allElementsEqual returns true for empty/single/repeated Int/String collections and a repeated ArraySlice, false for a differing element. Envelope's conditional equality compares values; a non-Equatable payload rejects its use. |
| Dispatch | dispatchProbe returns screen,screen,screen helper,default helper. Requirement and extension-only helper calls differ as documented. |
| File-sensitive conformances | Separate source files containing empty Codable, Equatable, Hashable, or checked Sendable extensions reject. A separately implemented manual Codable extension successfully uses the owned User's accessible initializer and prints 7 A after round trip. |
| Other extension restrictions | Private access on a conformance extension rejects. A designated class initializer in an extension rejects. The ordinary Swift subclass override attempted in an extension also rejects. |
| Date diagnostics | The unannotated Date/Identifiable block warns on all three configurations. The independently compiled retroactive and fully qualified variants are warning-free. warnings-as-errors promotes the unannotated conformance warning in the two focused macOS configurations. |
| Owned event identity | A single annotated Date conformance returns epoch ID zero in its isolated fixture. DatedEvent instances at the same timestamp retain their caller-supplied distinct IDs and unequal whole values. Encoding/decoding with matching secondsSince1970 strategies round-trips the event; global ID uniqueness is not enforced. |
| Attribute scope | Applying retroactive to a conformance of a type declared locally rejects because the attribute does not apply. It is not another identity or visibility mode. |

### Cross-module ownership and source/runtime visibility

MapVendor owns Coordinate under VendorPackage; NamingContracts owns Labelled under ContractsPackage; Adapter declares their one retroactive conformance under AdapterPackage. The modules are separately compiled, with dynamic libraries for the macOS runtime fixtures. The checked results under both isolation policies are:

- The unannotated external-type/external-protocol extension warns. Explicit retroactive or fully qualified type/protocol names acknowledge that warning.
- Using VendorPackage or ContractsPackage as the conformance module's package identity suppresses the warning through the corresponding same-package exception. The module owning its own Labelled protocol also conforms Coordinate without warning.
- A local typealias to Coordinate still produces the imported-type conformance warning; it has not created a new type.
- A static generic client importing only MapVendor and NamingContracts fails the Labelled constraint. Adding the Adapter import compiles and prints map:1.0.
- A runtime-cast client imports only MapVendor and NamingContracts in source. With only those libraries linked it prints none; linking the one Adapter library makes it print map:1.0. This demonstrates loaded conformance discovery without claiming that arbitrary static calls work without imports.
- MemberOne and MemberTwo each add a same-spelled trackingLabel member. A client importing both cannot choose an unqualified member access. Both libraries can nevertheless link and run together through their distinct labelFromOne/labelFromTwo functions, printing one two. This is a source ambiguity, not a duplicate conformance.

No competing Labelled conformances are linked into one runtime test. Package-exception and acknowledgment variants are independent compiler probes.

### Codable adaptation and upstream changes

The exact published empty imported Coordinate/Codable extension fails synthesis, despite retroactive acknowledgment. A separate temporary manual RetroCodable module implements accessible initialization and lat/lng encoding. It compiles and round-trips a Coordinate with those exact keys under both isolation policies. This establishes that a manually written bridge is possible, not that its ownership is preferable.

A separate upstream MapVendor fixture adds Codable at the type declaration. Recompiling the manual bridge against that fixture succeeds with duplicate-conformance warnings for Decodable/Encodable and notes that the adapter witnesses will not satisfy those conformances. retroactive does not suppress these warnings. The duplicate fixture is inspected at compile time only; no conflicting-conformance executable is run, and no runtime winner is asserted.

### Owned CoordinateDTO schema and external client

TrackingSDK publishes the exact DTO from the chapter as a separate module depending on MapVendor. An ordinary ClientPackage executable validates its public construction, coding, and mapping under both isolation policies:

- Current latitude/longitude and legacy lat/lng fixtures decode to equal values, including the representative 52.52/13.405 pair.
- Mapping to the vendor type and back preserves both coordinates; invalid vendor coordinates reject through the DTO initializer.
- Encoding writes only latitude/longitude and round-trips. Unknown extra JSON keys are ignored and not re-encoded.
- (-90,-180), (90,180), and (0,0) are accepted. Values beyond each latitude/longitude boundary, NaN, and positive/negative infinity in programmatic construction reject with outOfRange.
- Empty objects, incomplete current/legacy pairs, null, wrong types, mixed formats even when their values agree, and out-of-range JSON values all fail as DecodingError.

The evidence establishes the stated new-reader policy and current writer schema, not old-reader compatibility in every direction. No external map SDK, geospatial normalization, persisted database migration, or schema version negotiation is implemented.

### Reproduction and limits

Temporary check_a10.py/a10-published-results.json retain complete sources, separate-file companions, module/package identities, compiler flags, libraries/rpaths, warnings/errors, and runtime assertions. The stored-property negative reuses User; the imported synthesis and DTO fences import the separately built MapVendor. All other published fences are independent. Expected Date variants and conformance variants must remain separate to avoid introducing duplicate conformances accidentally.

The first 104-row record passed every runtime assertion. Its only two mismatched expectations classified the local retroactive misuse as a warning rather than an error. They were corrected from the retained compiler diagnostics without rerunning unchanged sources; a10-initial-results.json preserves that evidence. All final expectations pass.

Execution was on macOS 27 with Swift 6.4. Older Apple targets and iOS were compile checks only. No older compiler, Linux runtime, SwiftUI List, real vendor SDK, allocation/performance measurement, or duplicate-conformance runtime behavior was tested. The original historical Linux 6.2.1 diagnostic was replaced with current scoped evidence. The repository remains Markdown-only.

## F6 — Observation dependencies, advanced tracking, and observable request state

Reviewed on 2026-09-23 using the common Swift 6.4 compiler/SDK baseline. All ten Swift fences have dispositions for macOS 14, iOS 17, and the default-MainActor plus NonisolatedNonsendingByDefault configuration. The final record has 72 compiler configurations, including 25 intended rejections, two macro-expansion inspections, and sixteen successful executions. All final expectations pass. The repo remains Markdown-only.

### SDK surface and availability

The inspected interface is MacOSX.sdk/usr/lib/swift/Observation.swiftmodule/arm64e-apple-macos.swiftinterface under the selected Xcode 27 developer directory. It exports withContinuousObservation(options:apply:), not withContinuousObservationTracking. The proposal spelling fails lookup under both isolation policies. Options uses SetAlgebra/Sendable; requesting rawValue fails. Event is noncopyable; passing it to an ordinary Copyable generic fails. Kind is a struct; a switch listing the four known static values without a default fails exhaustiveness. Token is noncopyable and its cancel method consumes it; calling cancel twice fails ownership checking.

Basic Observable/tracking declarations compile for macOS 14/iOS 17 and reject unguarded use at macOS 13/iOS 16. Guarded advanced declarations compile at the older targets; unguarded advanced use rejects both there and at Apple OS 26. Observations rejects unguarded use at 14/17 and compiles unguarded at 26. All advanced runtime executions used macOS 27. No older Apple runtime or iOS runtime execution is implied by these compiler checks.

The macro expansion contains registrar storage, access instrumentation, withMutation, and shouldNotifyObservers. This inspection is specific to the selected compiler. An equal Int assignment produces no callback in the focused runtime fixture; the notes do not generalize that to all property types, access paths, or compiler releases.

### Base dependencies and corrected bugs

Under both isolation configurations, legacyProbe prints change then 2. The untracked property and ignored bookkeeping do not trigger its registration; the first count mutation triggers it once. A detached mutation of CounterModel.count rejects. A synchronous options callback cannot directly read that main-actor property or form its actor-isolated key path from the nonisolated callback.

Library checks cover filter-off short-circuiting, filter-on exclusion of an unread book title, included nested title mutation, favorite selection, array replacement, empty-array reads, and repopulation. The results follow actual executed reads rather than a declared fixed set of computed-property dependencies.

FeedModel checks stable rank/ID ordering, first calculation, cache-hit tracking, invalidation after array mutation, and empty input. SettingsModel checks explicit access/withMutation instrumentation for ordinary and applyExternalValue writes. Independently compiled variants reproduce both old defects: moving the source read below the cache-hit return yields no invalidation, and removing manual instrumentation from the forwarding adapter yields no callback despite changed backing state.

The three SwiftUI views compile on the stated Apple deployment targets under the checked policies. They are not mounted, and no rendered-frame count, state installation/recreation, UI task cancellation, or actual preference persistence was runtime-tested.

### Synchronous events and proposal discrepancies

The exact eventProbe returns will:true:false and did:true:false for the first value mutation; its next mutation produces no new event. Cancelling during willSet suppresses didSet. A separate main-actor-confined mutation fixture uses assumeIsolated only to verify the known synchronous execution context and reads old value zero at willSet, new value one at didSet. No event or actor-isolated key path is sent through that closure, and the chapter does not recommend assumeIsolated as a general observer isolation remedy.

A deinit-only registration fires once, and its event does not match Gauge.value. However, a separate combined willSet/didSet/deinit registration emits only will,did across two writes followed by destruction. SE-0506 illustrates a later deinit notification after that mutation pair; this runtime result differs. Both isolation configurations produce the same discrepancy. The result is explicitly recorded as a limitation of the tested distribution; cleanup must not depend on receiving all three callbacks from that registration.

Options set insertion/removal/contains/empty behavior passes. A separate options-overload fixture returns a noncopyable Permit whose consuming finish returns seven, and propagates a typed ProbeFailure.refused on its failing path. Event matching and phase timing do not establish total ordering among multiple independent observers.

### Continuous observation and async snapshots

Explicit AsyncStream callback handshakes await initial delivery and each relevant update; fixed sleeps or a fixed number of Task.yield calls are not used as synchronization evidence. The checked CounterBinding:

- Does not render synchronously in its main-actor initializer; its first callback arrives after suspension on MainActor.
- Receives initial zero and then two after sequential writes of one/two without an intervening suspension, demonstrating current-state coalescing in this fixture.
- Stops rendering after stop drops its token; an independent observation callback provides a scheduling checkpoint, and the retained log contains only the earlier values.
- Does not retain its weakly captured model after the last external strong reference is removed.

Additional direct-token fixtures check consuming cancellation, continuous willSet subscription reading the later coalesced value, a deinit-only callback that observes a nil weak target and cancels, and re-evaluation from a left branch to a right branch after selection changes. The explicitly retained tokens are cancelled and streams finished. CounterBinding's arbitrary render closure can still form a retain cycle if the caller captures its owner strongly; the weak model capture is not a universal cycle solution.

counterValues receives the initial value and a coalesced later value through Observations. This checks its snapshot behavior, not lossless logging, broadcast, backpressure, or every sequence cancellation edge. D10 contains the separate general AsyncSequence lifetime/cancellation guidance.

### Search exercise

GatedSearch stores each request's own continuation and reports request-start handshakes. The final fixtures resolve every request, await every task, and assert that pending requests/arrival queues/listeners are drained. Under both isolation policies they verify:

- Trimmed input, empty input with no service call, and successful empty results.
- Two submissions of the same query where the older success arrives before the newer result, after it, or as a stale failure; an older cancelled request may also return success without replacing the newer state.
- Reset and an empty submission invalidating pending publication.
- Active cancellation followed by success or a domain error, an explicit CancellationError, and an uncancelled domain error, each producing the documented state.
- Cancellation before the task starts avoids service invocation.
- Editing draft text without submitting retains the documented latest-submitted policy.
- A public state mutation is observed, while bookkeeping is ignored in the isolated model probe.

This is a controlled service/model validation. It does not implement networking, debouncing, shared cache ordering, persistent storage, or rendered UI behavior. Awaiting an instance method may retain the model until completion; caller ownership of the task does not promise immediate object destruction when the view disappears.

### Reproduction, adjustments, and limits

Temporary check_f6.py/f6-published-results.json contain complete sources, compiler flags, diagnostics, runtime output, and assertions. Fences 4/6/7 reuse CounterModel from fence 3, and fence 9 reuses the SearchModel/service declarations from fence 8; other fences are independent. The ordinary compiler shape is swiftc -swift-version 6 -default-isolation nonisolated -sdk SDK -module-cache-path CACHE SOURCE -c -o OBJECT. Runtime clients use -parse-as-library and @main instead of -c. Alternate runs add -default-isolation MainActor -enable-upcoming-feature NonisolatedNonsendingByDefault; availability checks set explicit -target values. Macro inspections add -Xfrontend -dump-macro-expansions.

Apple's Observation macro plugin could not launch in the nested filesystem sandbox, so the local temporary compiler matrix ran with approved sandbox escalation. This was an environment failure, not an Observation source diagnostic. Compiler/runtime subprocesses have 45/15-second bounds and core dumps are disabled.

The first 62-row matrix passed every runtime assertion but reported two never-mutated weak-var warnings in the test harness. Changing those probes to Swift 6.3+ weak let removed the warnings. Subsequent focused checks exposed the combined-event discrepancy described above; the two retained runtime records were reclassified against their actual will,did output without repeating unchanged execution. A typed-result fixture also needed the required main-actor entry point and consumption outside precondition's autoclosure. Only affected fixtures and newly added boundary cases were compiled again. f6-initial-results.json and f6-lifecycle-initial-results.json preserve the earlier evidence.

The resulting 72 configurations and sixteen executions all meet their final expectations. No older Swift compiler, Linux/Android runtime, iOS app, public binary evolution scenario, or performance benchmark was tested. Documentation source prose is distinguished from compiler/interface/runtime evidence, particularly where the proposal and SDK disagree.

## F1 — Testing migration, runner behavior, and deterministic concurrent tests

Reviewed on 2026-09-23 using Swift 6.4/Xcode 27 and Testing library version 2084. All twelve Swift fences have compile dispositions at macOS 15, iOS 18, and default MainActor plus NonisolatedNonsendingByDefault. The direct matrix has 47 compiler configurations, including six expected rejections. Sixteen published/diagnostic package-run configurations plus eight focused interop configurations make 24 retained test-runner checks. All final expectations pass; several runs intentionally report test failure, including one strict-mode process trap. No permanent package or executable was added.

### Compiler and deployment checks

Direct swiftc validation needs both the platform Developer/Library/Frameworks search path and Developer/usr/lib for XCTest's Swift overlay, plus the toolchain's usr/lib/swift/host/plugins/testing path for TestingMacros. The initial matrix omitted those plugin/overlay paths, causing harness configuration failures rather than language incompatibilities. Adding the paths fixed the representative checks; only failed configurations were repeated. CancellationLog also needed an explicit discard of Set.insert's result, and the final drain assertions were split into separate awaited actor reads.

All twelve final fences compile without warnings under the three stated configurations. Transferable attachment export rejects unguarded use at macOS 15/iOS 18 and compiles at 15.2/18.2. The inspected _Testing_CoreTransferable interface confirms those boundaries. The controlled-operation helper's Mutex rejects use at macOS 14/iOS 17. TimeLimitTrait accepts minutes and rejects seconds in the focused checks. A macOS AppKit/Testing image-attachment helper compiles; no image encoding or screenshot test was run.

The standard Testing framework and its new test-library APIs are provided by Xcode, not assumed to be Apple OS 27 system APIs. All runtime tests ran on macOS 27. The iOS and older macOS deployment checks are compilation evidence only.

### Interoperability against Xcode XCTest

The eight focused runs use the same temporary package and explicit SWIFT_TESTING_XCTEST_INTEROP_MODE values. The Swift Testing filter includes one failing and one successful XCTAssertEqual test; the reverse filter runs one XCTest method containing a failing #expect. The observations are:

| Mode | XCTest failure inside Swift Testing | Swift Testing failure inside XCTest |
| --- | --- | --- |
| none | Both Swift Testing functions pass; the intended failure is lost; exit 0 | Intended failure is lost; exit 0 |
| limited | The failing assertion produces two warning issues, and the test remains passing; exit 0 | Expectation fails; exit 1 |
| complete | Failure plus migration warning; exit 1 | Expectation fails; exit 1 |
| strict | Misuse terminates the test process with signal code 5; command exits 1 | Expectation fails; exit 1 |

Successful XCTest assertions in the focused Swift Testing fixture do not produce a warning in the recorded non-strict runs. The chapter avoids promising that every successful XCTest API call necessarily warns or traps. The strict failure fixture is isolated with core dumps disabled; no production test process is deliberately crashed.

The exact published legacyUniquenessCheck/bridgeXCTestFailure and BridgeTestingFailure probes additionally run in complete mode and report the intended failures in their respective frameworks. These are excluded from the passing test suite. A thrown XCTSkip inside Swift Testing is reported as a caught error and fails, rather than skipping the test; its custom skip message is not retained in that console diagnostic.

Separate runs remove the mode environment override and change only the temporary manifest's tools-version. A 6.3 manifest defaults to warning-only limited behavior, while 6.4 defaults to complete failure behavior using the same 6.4 compiler. This is SwiftPM/Xcode evidence, not a verification of Corelibs XCTest, every Xcode project default, or arbitrary third-party runners.

### Assertions, warnings, cancellation, reflection, and attachments

The ordinary suites pass under both isolation policies: each run reports thirteen Swift Testing test functions in one suite, plus one XCTest callback test. Parameterized functions include three normalization cases, two optional-fixture cases, and seven search-outcome cases. A deliberate runtime warning remains non-failing. These are runtime issue counts, distinct from compiler warnings.

The standalone rubric probe records two issues, prints after expect, and never prints after require. The confirmation negative leaves its closure with zero confirmations and then calls a saved confirmation too late; it fails with the expected miscount. The joined-task-group confirmation passes. Cancelling a test after recording F1-prior-error preserves that failure. The optional-fixture test cancels the unavailable case without failing the function and runs the available case. No test cancellation is used as a substitute for asserting cancellation of the search operation.

CustomTestReflectable produces the requested query/resultCount/generation child labels through Mirror(reflectingForTest:); ordinary reflection still reports resultIDs. A deliberately failing equality check shows the custom children in the console, while its parent description still includes the original resultIDs arrays. This is concrete evidence that a test mirror alone is not a complete redaction boundary.

The Transferable attachment test exports the exact bytes request=2 state=loaded followed by a newline. The retained file under the explicitly supplied attachments directory is read and compared byte-for-byte. Other runs may report an attachment without retaining it because no attachments path was supplied. No external document, network service, image-generation system, or persistent app store was used.

### Per-case repetition and process status

A controlled parameterized fixture records its own per-argument attempt count. Argument zero always passes; argument one fails its first attempt and passes afterward. Each command starts a fresh process with fresh counters.

| Configuration | Observed attempts: argument 0 / argument 1 | Final command status |
| --- | --- | --- |
| maximum-repetitions 3 | 3 / 3 | exit 1 |
| maximum-repetitions 3, repeat-until pass | 1 / 2 | exit 1 |
| maximum-repetitions 3, repeat-until fail | 3 / 1 | exit 1 |

The until-pass run's Swift Testing summary says passed after the second argument succeeds, but SwiftPM reports that the test target had failures and exits 1. xcsift's formatted summary also says success for that run. The wrapper preserves the actual pipeline exit status with pipefail and inspects the retained raw log. The notes therefore do not equate the last printed summary with CI success or claim that repetition proves correctness.

The command help confirms these repetition flags are Swift Testing-only and that attachments-path expects an existing writable directory. No hypothetical repeating trait was presented as shipped API.

### Debounced search and regression strength

The exact production SearchPipeline and SearchController are tested with ControlledOperation. Its cancellation handler records the unique call ID synchronously through Mutex, without spawning another task to record cancellation. Request-arrival handshakes ensure continuations are installed before completion; each resumed operation is then joined before its publication is asserted.

Published tests verify automatic replacement cancellation, identical normalized query strings with distinct IDs, latest-result retention after an old success, search remaining unstarted before delay release, and an older cancelled delay remaining unable to start a search after release. Both controlled actors are drained. Cleanup is registered before potentially failing prerequisites, tracks every acquired task handle, stops the controller, releases pending continuations, and joins all acquired operations through async defer.

Additional fixtures verify empty input, pre-start cancellation, loading state, empty successful results, active/current versus stale domain failures, explicit CancellationError, caller cancellation after the service starts, stop/empty submission invalidation, and release of the controller while the service is suspended. Weak ownership permits destruction; its deinit requests cancellation while the separately held task/service completion is still joined.

Removing only the generation guards in a separate broken package makes both regression tests fail: an old cancelled completion replaces a loaded latest state with cancelled, and an old failure replaces a newer loading state with cancelled. The normal implementation passes under both isolation configurations. This demonstrates sensitivity to the actual publication bug, rather than merely observing a cancellation flag.

The manual delay fixture proves ordering around release/cancellation, not an exact elapsed 300 ms policy. No virtual clock, network client, persistent cache, UI application, or XCTest performance benchmark was implemented. nextCall still needs a surrounding bounded process because an absent request cannot produce an arrival; cancellation of a test alone is not a forceful hang terminator.

### Reproduction, adjustments, and limits

Temporary check_f1.py/f1-published-results.json retain complete direct-compiler sources, command lines, and diagnostics. Fences 10/11 reuse the production declarations in fence 8 and controlled helper in fence 9. run_f1.py/f1-run-results.json retain complete package manifests, published and supplemental source snapshots, filters, runtime environment, exit codes, raw/formatted output, repetition counts, and attachment bytes. f1-interop-check.py/f1-interop-preflight-results.json retain the independent four-mode/two-direction checks.

Run the temporary packages using swift test --package-path PACKAGE with their recorded --filter/--skip choices. MainActor-default variants use an explicit target setting plus NonisolatedNonsendingByDefault; the language mode remains 6. The default Swift Build engine is used. Every build/test output is piped through xcsift with pipefail; raw logs are retained before formatting. Direct compiler processes are bounded at 45 seconds and each package command at 60 seconds, with core dumps disabled. Apple's compiler macros run with approved sandbox escalation.

The first direct record's missing plugin/overlay failures are preserved in f1-initial-results.json. The first runtime record misexpected the skip message text and until-pass exit status; both were corrected from retained evidence, without rerunning unchanged fixtures. f1-run-initial-results.json preserves that record. Only the two search-test cleanup changes and their affected normal/broken runs were repeated afterward.

All final 47 direct compiler and 24 runner checks meet their scoped expectations. No older compiler, Corelibs XCTest/Linux execution, iOS runtime, Xcode UI runner, test-plan configuration, or performance result was validated. The obsolete uploaded-template reference and scheduler-yield-based proof claims were removed, and questions/exercise/flashcards/rubric were rewritten together.

## F2 — Compiler diagnostics, debugger evaluation, and request evidence

Reviewed 2026-09-23 with the recorded Apple Swift 6.4 compiler, Xcode 27 SDKs, and macOS 27 host. The chapter, questions, exercise, flashcards, and rubric now share complete source instead of the old 6.2.1 diagnostic transcript and undefined production sketches.

### Source and configuration coverage

All 10 published Swift fences are checked under three configurations: macOS 15 / nonisolated default, iOS 18 / nonisolated default, and macOS 15 / MainActor default plus NonisolatedNonsendingByDefault. Language mode remains 6. Explicit nonisolated declarations keep intended boundaries stable across defaults. The debugger executable is compiled with parse-as-library for its main entry point; SwiftUI examples are compile-only.

The final record contains 48 compiler configurations, including 19 intended rejections and 3 intended deprecation warnings. Eight host executions succeed: the repository, named-task, portable-demangling, and snapshot fixtures each run under both macOS isolation policies.

| Check | Observed result |
| --- | --- |
| Generic relationship | Unconstrained Repository.Entity cannot supply User; the same-type constraint accepts. The runtime repository returns the expected two users. |
| Result builder | An array of Text rejects as a ViewBuilder expression; ForEach compiles despite its branch differing from ProgressView. No rendered UI is claimed. |
| Warning policy | The published adapter warns under warnings-as-errors in all three configurations. Ignored/error variants suppress/reject under both host settings. |
| Actor boundary | The retained ImageCache result rejects in all three settings with RegionIsolation. The same native error remains after @diagnose(RegionIsolation, as: ignored). |
| Sending control | Merely adding sending to the retained return rejects with a data-race diagnostic; removing the actor-owned alias and returning a fresh ImageCache accepts under both host policies. |
| Snapshot behavior | Mutating a returned Data value and a dictionary's Data element leaves actor storage unchanged. Later actor mutation leaves the earlier snapshot unchanged. Empty Data and added dictionary keys are checked independently. This proves observable value behavior, not allocation counts. |
| Named operation | Success, empty results, and a domain failure propagate as specified. After a request-start handshake, cancellation followed by successful service completion becomes CancellationError; cancellation followed by a domain failure preserves that failure. Each task is joined and its gate drained. |
| Name accessors | Task.name rejects at macOS 15 / iOS 18 and accepts at 26. A task handle's name rejects at macOS/iOS 26 and accepts at 27. The macOS 27 runs verify both names. Creation itself compiles at the earlier baseline. |
| Demangling | The exact B6 helper compiles on both platforms. Host runs format the Int symbol and preserve invalid/empty input. iOS follows canImport's unsupported-module branch; no iOS runtime is claimed. |

### Live LLDB observation

One successful debugger session used lldb-2103.0.34.103 against a temporary, freshly built macOS executable. Its source exactly matches published fence 5. It was compiled with Swift 6 language mode, explicit nonisolated defaults, g/Onone, a retained emitted module, and parse-as-library. The session disabled user lldbinit loading and stopped at the checkpoint before print.

The selected terminal observations are retained in temporary f2-lldb-results.json:

- Reading the stored reads field and displaying the Counter leaves reads at 0.
- Evaluating counter.computed returns 42 and changes reads to 1.
- po counter prints Counter(42) and changes reads to 2 through CustomStringConvertible.description.
- thread backtrace identifies the checkpoint/main call path.
- expression -- await worker.currentState() fails because the evaluator's function in this synchronous stopped context does not support concurrency.
- swift-healthcheck reports a log location. That transient file was no longer available after LLDB exited, so the record preserves the command's reported outcome rather than claiming retained health-check contents.

The batch deliberately continues after the expected await-expression error and kills only its temporary debuggee on completion. A first sandboxed launch failed before reaching a breakpoint; approved escalation allowed the subsequent session. The original failed launch is preserved in f2-preflight-results.json. No existing application or user debugger session was attached.

Installed help for frame variable, expression, and swift-healthcheck informed the command descriptions. The actual live sequence above is narrower than the chapter's general command reference. Watchpoints, async stepping/UI task views, iOS devices, Thread Sanitizer, Instruments, optimized locals, and release-only defects were not exercised.

### Reproduction and scope

Temporary check_f2.py and f2-published-results.json retain complete source, SDK/target/flag commands, diagnostics, and runtime output. Every published fence is an exact suffix of its recorded compilation source; runtime-only support is prefixed. All checks compile to objects or executables, so region/dataflow diagnostics are included rather than relying on typecheck alone. Compiler/runtime bounds are 45/15 seconds with core dumps disabled. The successful LLDB session's selected observations came from the terminal; it is explicitly a structured observation record, not a full raw transcript.

The first matrix produced an avoidable inferred-Void? warning in the temporary request-arrival helper. Giving that helper an explicit type removed the warning; only its two affected checks were repeated. f2-initial-results.json preserves the initial result. The first availability extension used unreleased OS-number gaps as negative targets; iOS 25 accepted the current-name getter. Final negatives instead use released macOS 15 / iOS 18 baselines, both of which reject. f2-availability-initial-results.json preserves that discovery; no runtime claim is made about an iOS 25 target.

The SDK provides the handle-name accessor at Apple OS 27 even though the fetched SE-0469 page still labels its amendment unimplemented. The chapter reports checked SDK behavior and names that source discrepancy. Current-task name access starts at 26 in these checks; source-compatible named creation is a separate API.

Swift 6.4 explicit-module tracking, debug-module-path, removal of old module packaging, and archived-expression requirements are sourced from Swift.org's module-tracking article. The live fixture does not validate a custom explicit-module build, a relocated archive, Linux/Windows debugging, binary-size reductions, or debugger speed. Demangling remains distinct from address symbolication. F1 owns the full deterministic stale-search regression exercise, avoiding a second incomplete controller here.

## F3 — Performance evidence, owned concurrency, and controlled storage experiments

Reviewed 2026-09-23. F3 now distinguishes sampled CPU, wall time, contention, blocking, allocation activity, and live/peak memory. It updates Instruments 27 and SwiftUI investigation guidance, replaces undefined benchmark fixtures and a detached-task production sketch, and keeps user-visible device results separate from local kernel measurements.

### Published sources and functional checks

All seven Swift fences compile in 21 configurations: macOS 15 and iOS 18 with explicit nonisolated defaults, plus macOS 15 with MainActor defaults and NonisolatedNonsendingByDefault. Language mode is 6. Fences reuse the preceding definitions exactly as specified in the chapter. Observation expansion uses the installed Apple plugin; XCTest uses the platform's developer framework and Swift overlay.

The first pass failed for three Observation macro launches blocked by the nested sandbox and six copies of one borrowed-Array source pattern. Approved escalation resolved the macro launches. Replacing for-in over the explicitly borrowed Array with indexed access resolved its consume diagnostic. f3-initial-results.json preserves the first pass; only affected configurations were repeated. The chapter now explains that source-level borrowing constraint without claiming that every manual loop is faster.

Two standalone functional executables run under nonisolated and MainActor+nonsending defaults, using Onone on macOS 27. They verify:

- Case-sensitive matching, empty-query inclusion, hidden-item filtering, order, absent results, and identical signposted/plain output.
- Four cancellation callbacks for 513 items, a callback for empty input, and propagation of an injected callback error from the signposted transform.
- The actual concurrent helper's normal result and cancellation of its caller task before preparation starts.
- FeedModel's real preparation, loaded-empty state, and reset.
- Controlled repeated identical requests with distinct arrivals: an old success or failure cannot replace a newer loaded result. Replacement does not itself cancel the old caller-owned task.
- Existing rows persist through loading and a current failure. Current cancellation followed by service success or a domain error publishes cancelled; an uncancelled domain error publishes failed. Reset invalidates the completion without cancelling the external task.

Request-start handshakes and continuation completion establish ordering; all acquired tasks are joined and the gates are drained. The fixtures use no timing guesses. The tests exercise the model/transform, not SwiftUI rendering, image processing, diffing, or a real network. Signpost calls execute on success and throwing paths; no Instruments signpost timeline was exported.

### Storage and borrowing measurement design

Four separate library/client builds exercise the exact StorageWork source: Onone, O, Osize, and O with MainActor default plus NonisolatedNonsendingByDefault. Every declaration involved in these kernels is explicitly nonisolated. Both the library and its client use the same recorded optimization/configuration. The library exposes no inlinable body; the client imports its separately emitted module and loads its temporary dylib through a recorded rpath.

Each executable takes count/repetitions/seed at runtime: 10,000 UInt64 elements, 20 kernel invocations per trial, initial seed 9173. It discards one warm-up round, records ten rounds, rotates the five workload orders across rounds, and uses the same per-round seed for comparisons. The 50 recorded samples per executable contain summed elapsed durations, not 50 individual app events. There are 200 recorded samples across configurations, with raw data retained.

Array/UniqueArray mutation runs start from equivalent fresh values and apply the same salts. The ordinary one-owner input has no deliberately retained second owner. The snapshot variant retains and validates the earlier Array value across every mutation; that is an additional semantic requirement. Snapshot summation/validation is outside the measured interval. Unique-buffer construction and capacity reservation are outside its interval. Read-only variants use identical initialized values, with the Span projection included in the measured call path.

Every mutation checksum is checked against the known wrapping arithmetic result. The printed chapter probe, empty input, empty UniqueArray, empty span, and overflow/wrapping behavior also pass. These checks demonstrate outputs and the measured fixture's setup; they do not count allocations or ARC operations.

One optimized client SIL emission confirms that calls to all four library kernels remain in the sample function, including the two Array mutation workloads. The selected function/reference/application evidence is retained in f3-sil-inspection.json. This is static evidence about generated call sites, not a measured dynamic call count or proof about a whole-module build.

### Local observations and limits

The following numbers are microseconds per kernel invocation: median of the ten per-trial average durations. They describe one run per configuration on the macOS 27 arm64 host, which reports 14 processors. There was no controlled thermal state, CPU pinning, power experiment, or population of hardware. They are illustrative local observations, not stable thresholds or statistically established rankings.

| Configuration | Array, one owner | Array, live snapshot | UniqueArray | Borrowed Array read | Span read |
| --- | ---: | ---: | ---: | ---: | ---: |
| Onone | 757.053 | 758.528 | 679.945 | 687.210 | 723.130 |
| O | 1.870 | 6.551 | 1.937 | 1.650 | 1.665 |
| Osize | 5.275 | 7.750 | 5.197 | 5.099 | 4.854 |
| O, MainActor default + nonsending flag | 1.658 | 6.150 | 1.700 | 1.405 | 1.421 |

The optimized UniqueArray timing is not consistently lower than an ordinary one-owner Array. Retaining a snapshot changes the Array workload and its observed cost. Debug/optimized results differ substantially, while small differences among optimized read/container paths should not be treated as reliable product gains. The MainActor-default row is a configuration check, not evidence that the flag improves speed. No p95/p99 or confidence interval is inferred from these ten trial averages.

The harness includes clock and cross-module-call overhead. Setup, extra post-interval validation, cache effects, process order, and host activity can influence results. It compares neither allocation counts nor physical device scrolling. It does not measure noncopyable payload costs, growing-buffer behavior, borrow/mutate accessor dispatch, image pipelines, or public inlining/specialization controls; those require their own experiments. B11 supplies the already checked implementation/ABI guidance, and C8/C12 establish the ownership/API behavior.

### Release-mode XCTest integration

The exact FeedPerformanceTests fence and its shared model/transform were placed in a temporary dependency-free SwiftPM package with tools version 6.4, Swift 6 language mode, explicit nonisolated defaults, and macOS 15 deployment. The command uses swift test --configuration release --filter FeedPerformanceTests and SwiftPM's default Swift Build engine. It passed one XCTest with ten measured iterations and no failures or warnings.

The runner reports monotonic clock time, CPU time/cycles/instructions, physical-memory delta, and physical-memory peak. The clock samples ranged approximately 1.22–1.49 ms. Physical-memory deltas included zeros and reached 114.688 kB; their reported relative standard deviation was approximately 183.9%. The test still passed with an empty baseline field. This is evidence that the example measures/checks its result, not an established performance regression policy. Peak physical-memory measurements are not the allocation count of makeRows.

The measurement includes allocating and replacing the result array; fixture creation, expected IDs, and the final equality check remain outside. All 3,333 expected IDs are compared, preventing an empty or partial result from being accepted merely because it was fast. The XCTest sample is synchronous; no unjoined async work is hidden outside its measured closure.

Build/test output is captured raw before xcsift formatting, with pipefail preserving command failure. f3-xctest-results.json retains the manifest, exact combined source, command, return code, and raw/formatted output. No saved Xcode baseline/test plan, CI threshold, UI test, or third-party Benchmark dependency was introduced.

### Reproduction and final coverage

Temporary check_f3.py/f3-published-results.json retain the 21 fence configurations. run_f3.py/f3-run-results.json retain two functional builds and four library/client pairs: ten further compiler invocations and six successful standalone executions. One additional compiler invocation emits optimized SIL. run_f3_xctest.py/f3-xctest-results.json retain the separate release test-runner integration. f3-functional.swift and f3-benchmark.swift hold the temporary fixture code; all permanent changes are Markdown.

The final coverage is 31 direct object/library/executable compiler invocations, one SIL emission, six successful standalone executions, and one successful release-mode test-runner configuration. Compiler/functional execution bounds are 45/15 seconds; each measurement executable is bounded at 45 seconds and the package command at 60. Core dumps are disabled. Later functional assertions strengthened the retained-row and caller-cancellation checks; only those two affected executables were repeated, leaving the timing observations untouched.

All final checks satisfy their stated expectations. No iOS runtime, physical-device trace, Instruments capture, Linux/Windows execution, energy result, or allocation-count improvement is claimed. The profiling exercise names the missing evidence needed for those conclusions. Questions, answers, exercise, flashcards, and rubric are synchronized with these limits.

## F4 — Macro package, expansion, and client boundaries

Reviewed 2026-09-23. F4 replaces incomplete package/HTTP sketches and an overly broad additive-only claim with a complete temporary macro package. The permanent repository remains Markdown-only. The examples separate custom expression/member/body macros, typed endpoint metadata/decoding, and the additional design obligations of a production HTTP client.

### Toolchain, dependency, and package run

The selected Xcode 27 Swift 6.4 toolchain builds the chapter's tools-version 6.4 manifest in Swift 6 language mode. Each target explicitly uses .defaultIsolation(nil). The manifest resolves exactly swift-syntax 604.0.0, revision 050f1a346fbbac0ca2cfb15a95274f7bd1cf0ccf. SwiftPM obtains the dependency from its cache and creates a temporary checkout; no project dependency or source package was added permanently.

The package has a host macro target, a public declaration library, an executable client, and an XCTest target importing the implementation and SwiftSyntaxMacrosTestSupport. It uses the default Swift Build engine. The package command exits 0 with three passing tests and no warnings: expression expansion, route-member expansion, and the relative-path diagnostic's message/source location. All published file contents are retained in f4-spm-results.json and match the chapter fences.

Build output is captured before xcsift formatting, with pipefail preserving the actual exit status. The first sandboxed curl attempt to read the official manifest failed DNS resolution; an approved network retry succeeded. The package build and direct plugin-launch checks then ran with approved escalation and passed on their first code-validation pass. No experimental macro feature flag was used. No speed improvement is inferred from this one build.

### Direct compiler and runtime matrix

check_f4.py and f4-direct-results.json retain 36 direct compiler invocations, all meeting expectations. Three configurations use language mode 6: macOS 15 deployment with nonisolated defaults, iOS 18 deployment with nonisolated defaults, and macOS 15 deployment with MainActor defaults plus NonisolatedNonsendingByDefault. The installed macOS/iOS SDK paths and every argument are recorded. All share the package-built macOS host plugin through -load-plugin-executable; the plugin is not rebuilt for iOS.

In each configuration, the macro API is emitted as a separate InterviewMacros module, and three real clients compile: the published behavior probe, the endpoint/decoding example with its entry-point call, and an additional escaping/nesting/error fixture. Both macOS configurations execute all three; the SwiftPM-built client executes once more. These seven standalone executions pass on macOS 27. The iOS clients produce object files only; macOS 15 deployment does not mean execution on macOS 15 was tested.

The behavior probe prints 7 next() 1 /users 0 and asserts each result. It proves a single evaluation of the expression in this case and the route metadata value. The body macro replaces a syntactically valid unresolved name with return 0. The extra fixture checks a quoted/backslashed raw route, an escaped leading slash, generic/nested structs, source-value string escaping, and one throwing call with one attempted evaluation and preserved error propagation.

The endpoint fixture compiles a protocol with a Decodable/Sendable associated response and both ordinary and generated static paths. It decodes a known payload to the same concrete UserDTO array, checks equality, and prints 7 /users. It is not a URL encoder, HTTP transport, request scheduler, authentication layer, or cancellation implementation. Those remain explicit requirements of the design exercise; no network behavior is inferred from decoding a local payload.

### Intentional failures and expansion evidence

Twenty-one invocations intentionally fail with the expected diagnostics. Nine cases run under both macOS default-isolation configurations: relative route, variable route argument, interpolated route, class attachment, duplicate path member, incompatible expression result, incompatible generated body result, illegal MainActor call, and a response that fails the protocol constraints. These distinguish the plugin's syntax diagnostics from native type/isolation checks.

Two additional invocations confirm that a separately emitted Domain module's public decorated struct still has an internal generated path. The Domain module itself compiles successfully in both configurations. Public visibility is not inherited automatically from the decorated struct.

One invocation omits the plugin while importing the compiled macro declarations and using them; it fails to find the external implementation. One successful invocation uses -Xfrontend -dump-macro-expansions and retains the generated tuple, nonisolated path property, and return 0 body. The dump option is diagnostic tooling, not a proposed application setting.

An otool -L inspection of the SwiftPM executable lists its Foundation, Objective-C, system, and Swift core dependencies, with no dynamic SwiftSyntax/plugin dependency. This is a narrow dynamic-dependency inspection, not a complete size or static-symbol audit. The architectural host/target distinction is also supported by the separate iOS compiler checks and SwiftPM's macro target model.

### Reproduction and limits

Create the files in F4's section 3–5 layout in a temporary directory, then run swift test --filter MacroTests with the selected toolchain. The captured package command uses --package-path /private/tmp/swift64-interview-validation/f4-package. The products are under its .build/out/Products/Debug directory on this host; that path is a build result, not a portable assumption for every build engine.

For the direct checks, run temporary check_f4.py with the package's built plugin present. Each compiler command is bounded at 45 seconds and each standalone execution at 15 seconds; core dumps are disabled for that harness. The retained JSON contains exact sources, commands, diagnostics, return codes, and expected runtime output. f4-spm-results.json contains the package sources, resolved revision, full raw build/test output, and dependency inspection. Temporary setup/rewrite scripts are work records and should not be rerun over later edits.

The nine published Swift fences match their checked package/direct sources, including the two explicitly rejected clients. The client code probe also matches the rubric. Questions, answers, exercise, flashcards, and migration guidance are synchronized. No Linux/Windows host, iOS runtime, older macOS runtime, archive distribution, performance comparison, or comprehensive URL/HTTP validation is claimed. The package is an instructive supported-input example, not a general-purpose endpoint generation library.

## A1 — Observable state, copyability, identity, and drafts

Reviewed 2026-09-23. A1 now qualifies ordinary-copy statements with Copyable, follows reference and closure aliases through value wrappers, and distinguishes object identity from domain identity and equality. Its original box probe is retained with explicit contexts and an additional array-property extraction case. The worked cart exercise verifies independent nested value items around a shared MainActor controller.

### Compiler matrix and expected failures

All 15 Swift fences are compiled to object code in three configurations: macOS 13 deployment with explicit nonisolated defaults, iOS 16 deployment with explicit nonisolated defaults, and macOS 13 with MainActor default plus NonisolatedNonsendingByDefault. Each uses the installed Xcode 27 SDK and Swift 6 language mode. The target triples, SDK paths, module cache, exact source and compiler command are retained in a1-published-results.json. Snippets that depend on preceding definitions are combined exactly as the chapter states.

These 45 fence configurations include twelve expected failures: three configurations each reject reassignment of a let reference binding, use of a consumed Ticket, a Sendable struct containing a non-Sendable Counter, and mutation of a let ValueBox. Using object compilation ensures that the use-after-consume check reaches ownership analysis.

Eight focused configurations add four expected failures under both macOS default-isolation settings: reference-identity comparison of structs, reference-identity comparison of closures, passing a noncopyable Ticket to an implicitly Copyable generic parameter, and synchronously calling an actor-isolated method from a nonisolated function. The closure case specifically diagnoses that reference equality of functions is unsupported; it is not the generic class-constraint diagnostic of the struct case.

The initial run already rejected every invalid source and passed all positive code/runtime checks. Five result entries were marked failed because the checker expected the wrong capitalization for non-Sendable or the wrong closure diagnostic. a1-initial-results.json retains that run. Only those five affected checks were repeated after correcting the diagnostic expectations; no chapter Swift source change was needed.

### Runtime behavior and exercise checks

Four complete executables run the exact published positive fences: Onone and O, each with nonisolated defaults and with MainActor+nonsending defaults. All execute on the macOS 27 host, not on the older deployment OS. The harness compares the full output to the chapter predictions and checks the additional assertions:

- Independent Profile names versus two references observing one mutated ProfileRef.
- Two let wrappers retaining the same ItemStorage; mutation through the stored reference is visible through both.
- Array copies retaining the same Counter, followed by independent element replacement and append in one array.
- Copied CounterAction values sharing one captured count, while a fresh factory result has independent captured state.
- Separate equal UserRecord instances, later inequality after an edit, stable matching domain IDs, independent UserSnapshot edits, and identity comparison of two nil optional references.
- Successful Ticket transfer/consumption, plus a noncopyable wrapper whose Counter can still be mutated through an existing alias.
- Checked Sendable value and immutable reference representations.
- Actor identity shared through aliases, with an earlier [Int] snapshot staying unchanged after a sequential awaited append.
- The original ValueBox/RefBox result and an extracted array that can be independently appended to.
- Cart draft item edits and append before commit, visibility through the shared controller alias after commit, preserved old snapshots, and draft edits after commit that leave committed data unchanged.
- An initially empty cart, insertion through a draft, clearing that draft after commit without clearing the stored cart, and explicitly committing an empty replacement while a previously retained value remains intact.
- ObjectIdentifier equality for live aliases and inequality for two simultaneously live separate UserRecord instances; alias mutation preserves the separate record's value.

The cart uses ordinary Int quantity data and an unconditional whole-cart replacement policy. It does not validate a commerce schema or resolve conflicting revisions. MainActor access protects the example's UI-model ownership, not stale-draft merge semantics. No UI rendering, persistence, network, or concurrent stress test is inferred from this sequential model fixture.

### Evidence and limits

Final coverage is 57 compiler configurations: 45 fence configurations, four executable builds, and eight focused negative cases. Twenty compilations intentionally fail; all 57 outcomes and all four executions satisfy their stated expectations. There are no unexpected warnings or runtime failures. Temporary check_a1.py/a1-published-results.json retain sources, commands, diagnostics, expected outputs, and actual results. Compiler/runtime process bounds are 45/15 seconds, and core dumps are disabled.

The checks establish the illustrated observable state and ownership rules. They do not count buffers, ARC operations, allocations, or closure boxes; they do not benchmark CoW, prove deep cloning for arbitrary object graphs, or demonstrate concurrent access to a shared variable. No iOS runtime, older macOS runtime, Linux/Windows, or new container API execution is claimed by A1. C7/C8/C12 and F3 carry the separately checked newer ownership/storage and measurement material.

Questions, answers, flashcards, exercise, rubric probe, and chapter links are synchronized. The repository stays Markdown-only, and the chapter's changed prose explicitly distinguishes preexisting Swift semantics from Swift 6.4 migration guidance.

## C2 — CoW invariants, slice retention, and storage choices

Reviewed 2026-09-24. C2 now separates independent collection behavior, object retention, and measured cost. It removes a fixed deinitializer-order prediction and an unconditional performance ranking of functional array updates. The chapter supplies a private custom CoW teaching type, a weak-reference retention fixture, a lightweight result projection, and an explicitly available UniqueArray snapshot example.

### Compiler and semantic checks

All eleven Swift fences compile to object code in three configurations: macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor default with NonisolatedNonsendingByDefault. The compiler is the installed Xcode 27 Swift 6.4 compiler in Swift 6 language mode, with explicit SDK/target and temporary module-cache paths. The two UniqueArray functions carry anyAppleOS 27 availability even when compiled for older deployment targets.

The 33 fence configurations include nine expected failures: isKnownUniquelyReferenced cannot inspect an Array as a class reference; CowInts cannot acquire a checked Sendable conformance while storing its private non-Sendable Storage; and a consumed UniqueArray cannot subsequently be used through its original binding. The Sendable extension is compiled with the actual CowInts implementation. Object compilation reaches the ownership failure.

Four more availability configurations remove the annotation from the successful UniqueArray function. macOS 26 and iOS 26 reject the unguarded API; macOS 27 and iOS 27 compile it. The source is identical apart from that removed annotation, rather than a separate surrogate example.

Four executable builds combine the published positive sources. Each is compiled for macOS 13 and runs on macOS 27, using Onone or O with either nonisolated defaults or MainActor+nonsending defaults. The entry point checks runtime availability before executing the UniqueArray probe. All four satisfy the complete expected output and semantic assertions:

- Array replacement/append preserve the earlier value; nested dictionary-array mutation, Set membership, and String append preserve their originals. No dictionary/set iteration order is asserted.
- Arrays retain shared Box identity while their membership/counts differ after append.
- A slice keeps indices 3..<6 and mutates independently of the parent array.
- Ordinary-return, inout, and consuming helpers produce the requested values while retained input snapshots remain unchanged. The local builder handles empty and nonempty counts.
- CowInts append, setter-first, and compound-modification-first each preserve a separately retained original. Separate first-mutation cases prevent an earlier append from masking a missing setter detach. Later copies, getter-result edits, constructor input edits, and empty initialization/append are also checked.
- The summary model preserves the first twenty IDs in order, title/URL fields, an earlier result snapshot, and independence of a locally edited result array. Replacement with fewer results, reset, empty input, zero limit, and a one-result limit pass. No image loading or byte allocation measurement is performed.
- UniqueArray borrowing iteration constructs an ordinary Int snapshot, then changing the unique container leaves that snapshot unchanged: [1, 2] 9 2.

### Retention evidence

The published fixture creates three pure Swift Item objects in a noninlined initializer, retains weak probes, and stores either the one-element slice or Array(selectedSlice). After initialization returns, the slice fixture reports [true, true, true]; clearing the slice reports all false. The owned selected-array fixture reports [false, true, false]; clearing that array reports all false. Both expose only Item 1 before clearing.

These four liveness observations pass in all four full executable configurations. An earlier isolated preflight also runs the same retention source under Onone/O with nonisolated or MainActor defaults at the host's macOS 27 target. Those four additional compiler/runtime configurations pass. The preflight's first attempts could not write the compiler's default module cache; the same commands with an explicit writable temporary module cache succeeded. No sandbox escalation was needed for this correction.

The weak probes do not retain the Item instances. Their observations establish this fixture's retained object set at the checked boundaries, without relying on the order of element destruction. They do not establish a universal last-use/lexical-scope rule, a byte allocation count, resident-memory reduction, or Foundation/autorelease behavior. An owned selection can still retain selected class instances or selected payload data; the UI example deliberately projects away image bytes because that particular list needs only metadata.

### Reproduction and final coverage

Temporary check_c2.py/c2-published-results.json contain 41 primary compiler configurations, including eleven expected failures and four successful executions. c2-retention-preflight.json records four additional successful retention builds/executions; c2-retention-initial.json preserves the cache-path failures. c2-retention.swift is the exact published retention source, and c2-retention-main.swift supplies its simple entry point.

The final evidence therefore covers 45 successful-as-expected compiler configurations and eight standalone executions, with setup failures recorded separately. The main matrix preserves exact sources, compiler arguments, diagnostics, expected stdout, actual stdout, and return codes. Its compiler/runtime bounds are 45/15 seconds and core dumps are disabled. No performance result is inferred from the fixture: F3 retains the separate, already checked Array/UniqueArray/Span benchmark and its limits.

There is no iOS runtime, older macOS runtime, Linux/Windows, Objective-C bridging benchmark, physical-device memory trace, or unsafe concurrent-mutation test here. CowInts is an educational representation with an outer class and inner Array, not a faster collection claim or an audited unchecked Sendable implementation. Questions, answers, exercise, flashcards, rubric probe, and links are synchronized, with permanent changes confined to Markdown.

## C1 — ARC, weak bindings, and cleanup completion

Reviewed 2026-09-24. C1 replaces an invalid self-before-initialization example, adds weak let and the separate weak-capture setting, and validates strong/weak/unowned behavior without a universal lexical-scope destruction promise. The timer exercise has a private main-run-loop scheduling contract, explicit invalidation, weak callback ownership, and an isolated-deinit backstop. Questions, flashcards, migration notes, and the rubric use these same contracts.

### Compiler coverage

All fourteen published Swift fences have an object-compilation disposition for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. The installed Swift 6.4 compiler uses Swift 6 language mode, explicit SDK/target and a writable temporary module cache. Shared declarations are prepended as stated in each example. The UIKit adapter body is checked on iOS; its conditional body is excluded on macOS.

The 42 fence configurations include six expected rejections: reassigning a weak let and passing self to a child before the parent's let property is initialized. Additional checks cover the separate ImmutableWeakCaptures flag, checked Sendable storage, callback isolation, ordinary versus isolated deinit, and UniqueBox availability. The final 68-configuration matrix has 17 expected compile failures and three expected imported-callback warnings; all other compilations are warning-free.

- The read-only weak capture compiles with ImmutableWeakCaptures enabled. An explicitly weak captured binding can be read and reset without the flag; enabling the flag rejects its assignment.
- A mutable weak property on a checked Sendable class is rejected, as is a weak let holding a non-Sendable mutable referent. Weak storage does not make the referent Sendable.
- Removing MainActor.assumeIsolated from the actual Timer callback emits a warning about mutating an actor-isolated property from a Sendable closure in all three configurations. This imported-API diagnostic is recorded as a warning, not a hard-error claim.
- Removing isolated from the cleanup deinitializer rejects mutation of the MainActor log in all three configurations. The positive isolated form compiles for the older deployment targets in this SDK.
- Removing availability from the actual UniqueBox example is rejected for macOS/iOS 26 and accepted for macOS/iOS 27. Published availability is anyAppleOS 27.

### Runtime evidence

Four combined executables use the exact positive chapter sources, Onone/O optimization, and nonisolated or MainActor+nonsending defaults. They compile for macOS 13 and execute on macOS 27. All assertions and expected output pass:

- Another strong alias keeps Tracked alive through withExtendedLifetime; clearing the final fixture alias produces the expected weak nil observation and one destruction event.
- Weak delegate registration/replacement works while a delegate is retained; callbacks become no-ops after release. Both the weak let property and read-only weak closure return nil after their referent is released.
- The valid unowned relationship returns Ada with its parent explicitly kept alive. Nil optional unowned storage and a live non-nil optional referent are successful controls.
- A returned pair of weak observations sees the intentional strong cycle persist. Calling the stored callback changes its owner's count; explicit stop releases both objects exactly once, without asserting their destruction order. A weak callback works while its owner lives and becomes a no-op when a retained child outlives it. Every intentional test cycle is cleared.
- The actual Timer is registered on RunLoop.main with a long interval and manually fired only from MainActor. Replacement invalidates the old timer, repeated stop is harmless, invalidated timers do not increment ticks, and dropping the owner while retaining a timer handle observes owner release and invalidation. No elapsed-time or physical-screen behavior is inferred.
- The concurrent consuming helper releases the isolated cleanup owner; the harness awaits its explicit AsyncStream notification and stream completion before checking the log and weak observation. This verifies completed cleanup, without claiming to measure the exact executor-hop timing or ordering of weak zeroing versus deinit.
- A UniqueBox consumes its noncopyable Resource once; a separate class payload remains identical to another strong alias. The Foundation autoreleasepool API returns the expected UTF-16 counts for empty and Unicode input, without a memory-footprint claim.

Two additional isolated processes deliberately access stale checked unowned references: the chapter's escaped card and a stale optional unowned property. Both terminate unsuccessfully with the expected unowned-reference diagnostic. Core dumps are disabled; no unsafe unowned access is executed. These are expected negative tests, not passing application runs.

### Reproduction and limits

Temporary check_c1.py/c1-published-results.json preserve all 68 final compiler commands, exact sources, diagnostics, expectations, return codes, and six executions (four passing applications and two expected traps). Bounds are 45 seconds per compilation and 15 seconds per execution. The first matrix had two auxiliary weak-capture controls with an unused-written-variable warning; those captures now read the value before resetting it. Their flag-enabled counterparts were rechecked as well. The initial output is retained separately in c1-published-initial-results.json.

Fifteen earlier compiler preflights and four timer/cleanup executable preflights are recorded in c1-preflight-results.json and c1-runtime-preflight-results.json. Initial inferred-Void? warnings in the runtime harness were corrected with explicit annotations and those four runs repeated. These supplemental checks are not silently added to the 68 primary configurations. The final preflight and published timer/cleanup source match.

Only macOS 27 runtime behavior was exercised. Older Apple deployment targets and iOS are compile-only evidence; there is no iOS UI lifecycle run, periodic-timer accuracy test, non-Apple execution, Objective-C allocation measurement, resource-pressure test, or forced asynchronous cleanup timing guarantee. ARCLog and the cycle probes are intentionally single-execution-context teaching fixtures, not shared mutable Sendable types. Permanent artifacts remain Markdown-only.

## C4 — Unicode units, range boundaries, and truncation

Reviewed 2026-09-24. C4 removes conflicting truncation implementations and incorrect predicted output, distinguishes text equality from byte identity, and checks explicit whole-grapheme selection policy. It also covers strict/repairing decoding and existing UTF8Span APIs without relabeling them as new 6.4 features.

### Compiler and runtime coverage

All thirteen Swift fences have object-compilation dispositions for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. Shared context for rangeProbe is the actual graphemeRange/replacingGraphemes source. Integer String subscripting is the sole intentionally rejected chapter fence, rejected in all three configurations. Positive sources are warning-free.

Four combined executables use all positive chapter sources under Onone/O and both isolation configurations, target macOS 13, and execute on macOS 27. Exact printed counts, text, booleans, and semantic assertions pass. The runtime guards require macOS 15 for String(validating:as:) and macOS 26 for the UTF8Span helpers.

The seventeen-input corpus includes empty/ASCII text, composed/decomposed accents, flags, a family emoji, skin-tone/ZWJ sequences, CRLF, leading and long combining sequences, embedded NUL, a prepend scalar, an odd regional-indicator sequence, and an Indic sequence. The checks cover:

- Exact Character/scalar/UTF-16/UTF-8 counts for the chapter's named fixtures; canonical equality, differing byte sequences, Set key behavior, case sensitivity, and distinct Latin/Cyrillic scalars.
- Successful Character offsets and nil for negative/end/beyond-end/Int.max requests; append-induced grapheme combination; and repaired malformed UTF-16 output from the intentionally naive prefix.
- Character budgets from zero through two beyond each input's Character count, plus Int.max. Expected scalar/byte content and the output quota are checked against independently enumerated source Characters. The named limits 1–4 produce the corrected ellipsis outputs.
- Every UTF-8 budget from zero through one beyond each input's byte count, plus Int.max. The result must equal the longest enumerated whole-Character prefix that fits, preserve encoded content, and satisfy the byte quota. The named 7/8/10/11/12-byte example passes.
- Every in-bounds UTF-16 start/end pair for each input. A separately derived set of Character-boundary UTF-16 offsets determines acceptance. Accepted ranges round-trip to NSRange, select exact UTF-16 content, and produce the expected byte-preserving replacement; rejected ranges cannot edit. Negative locations/lengths, NSNotFound, Int.min/Int.max, overflowing combinations, and out-of-bounds endpoints return nil without computing an overflowing sum.
- Strict versus repairing UTF-8 decoding for all valid corpus entries, including embedded NUL, and six malformed sequences (bad continuation, overlong encoding, surrogate encoding, out-of-range scalar, invalid leading byte). The published malformed example repairs to scalars 65533 and 40.
- Owned first-field behavior for empty, leading delimiter, no delimiter, ordinary, and Unicode input. This is explicitly not a CSV grammar or memory-footprint measurement.
- UTF8Span byte counts and owned copies for every valid corpus string; typed validation failure for the malformed byte set. The six-byte é-plus-emoji result passes. Returned Strings preserve exact UTF-8 bytes.

### Range conversion observation

On the tested Darwin Foundation runtime, Range(NSRange(location: 1, length: 0), in: "e\u{301}") returns a range even though the offset is inside the Character. All in-bounds empty UTF-16 ranges for the decomposed accent, flag, family emoji, and CRLF examples also convert, including surrogate-interior positions in those fixtures. The harness only compares these raw converted indices; it does not slice using a surrogate-interior range. The published helper separately rejects endpoints absent from the String's Character boundaries.

This is a recorded platform observation, not a claim that every historical or non-Apple Foundation implementation behaves identically. Conversion, policy validation, and document-revision validity remain distinct. The helper intentionally accepts whole-grapheme edits; IME/marked-text behavior may require a different framework-specific policy.

### Availability and negative checks

Removing String(validating:as:)'s annotation fails for macOS 14/iOS 17 and succeeds for macOS 15/iOS 18. Removing the UTF8Span helpers' annotations fails for macOS 15/iOS 18 and succeeds for macOS/iOS 26. Both helpers retain their actual published annotations for older-target fence compilation. The SDK's thrown error spelling is UTF8.ValidationError, differing from the earlier proposal text.

Two auxiliary functions declared to return UTF8Span fail with "a function cannot return a ~Escapable result" in the selected baseline. The check does not claim user-authored lifetime-return annotations are supported or that a finer local-storage escape diagnostic was reached. The first expectation looked for a lifetime diagnostic; it was corrected to the actual declaration-level rejection. A borrowed-input variant produced the same rejection. These are fixture expectation refinements, not unresolved compiler failures.

Three separate processes intentionally violate the nonnegative-limit precondition of the display, byte-budget, and naive UTF-16 helpers. Each fails with the expected precondition diagnostic. These do not run in the positive suite; core dumps are disabled and no unchecked optimization mode is used.

### Reproduction and limits

Temporary check_c4.py/c4-published-results.json record 56 final compiler configurations, nine expected compile failures, and seven executions: four passing applications plus three expected traps. Exact sources, commands, diagnostics, stdout, and return codes are retained; compiler/runtime bounds are 45/15 seconds. Only the additional UTF8Span cases, modified combined executables, and revised negative expectation were rerun after the initial 47-configuration matrix passed.

c4-preflight.swift/c4-preflight-results.json record one additional successful exploratory build/run. The first exploratory command lacked an explicit SDK and could not load the standard library; adding SDK/target then exposed the validating initializer's older-target availability requirement. Those two failed setup variants are retained separately. The final guarded preflight passed. c4-before-escape-refinement.json preserves the auxiliary diagnostic mismatch from the later matrix.

No older compiler/runtime, iOS runtime, non-Apple Foundation, actual text layout, editor/IME interaction, normalization service, network protocol, Unicode conformance suite, memory trace, or performance benchmark was executed. The corpus is a targeted regression fixture, not proof that every possible Unicode sequence or product text policy is covered. Questions, exercises, flashcards, migration guidance, and rubric are synchronized; permanent files remain Markdown-only.

## A5 — Pattern coverage and external enum fallbacks

Reviewed 2026-09-24. A5 now uses complete current examples for guarded/optional/tuple/grouped patterns, custom matching, fallthrough, borrowed noncopyable payloads, external nonexhaustive enums, and a pure networking-to-view-state mapping. It removes obsolete template references and 6.2.1 diagnostic/output labels. The networking example supplies the missing Equatable payload conformance and makes cancellation, authentication, permission denial, and retry policy explicit.

### Compiler matrix

All fourteen Swift fences have compile dispositions for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. The installed Swift 6.4 compiler uses Swift 6 language mode except for the explicitly identified mode-5 controls. Shared Response declarations are prepended as indicated. The SDKStates fence emits a real module and object without library evolution; the consumer fence imports that corresponding target's compiled module.

The final matrix has 78 compiler configurations, including 27 expected failures and eight expected warnings. Other compilations are warning-free. All final results match expectations:

- The three intentional negative chapter functions are rejected for missing completed-payload coverage, missing cancellation, and fallthrough into a binding that was not established.
- A grouped-case variant missing one shared binding is rejected. A different control removes the first alternative's where clause; the first alternative then accepts a negative horizontal value and the compiler warns that the later horizontal pattern is already covered.
- Adding a new Response case rejects render, while the separate lossyRender catch-all control still compiles. Adding a new outer NetworkResult case or nested NetworkError case rejects the complete mapper.
- Removing User's Equatable conformance rejects the containing UserListViewState conformance. The checked positive source retains the actual conforming payload, rather than replacing the array with a simpler surrogate.
- The external nonexhaustive consumer without a future-case fallback is rejected in Swift 6 mode and warns in Swift 5 mode. The compiler release is unchanged between these controls.
- A consumer with unknown default but no waiting case warns in Swift 6 mode. A plain-default version has no such warning. These controls operate across a real module import, not two declarations in the same source module.

### Runtime evidence

Four combined executables run the exact positive sources with Onone/O and both isolation configurations, targeting macOS 13 and executing on macOS 27. Each links the separately compiled SDKStates object. Assertions cover:

- Completed status boundaries 199/200/204/299/300/404 and Int extrema, failure/cancellation, optional nil and unmatched cases, guard-return values, and positive optional filtering with order preserved.
- All four reachability/authentication combinations; both axis alternatives at negative, zero, and positive values; origin; custom matcher/range overlap at zero; negative, small, and outside-range integers.
- Exact fallthrough event sequences, intentional coalescing of failure/cancellation by the lossy mapping, repeated borrowing of the same issued noncopyable permit, and the denied permit's nil result.
- Both known external status labels through the complete consumer.
- Empty/nonempty 200/204/299 success, invalid success statuses, explicit and URLError cancellation, offline, timeout, connection loss, certificate rejection, bad URL, 401 versus 403, 500/503/599, and other statuses including 429. Server branches are tested with nil and nonempty response bodies. A decoding failure maps to the chosen stable UI message. View-state equality distinguishes changed payloads.

Four more executions test omitted-known-case consumers under both isolation settings. The unknown-default and plain-default variants both print ready future for ready/waiting inputs; only the former warns. Two grouped-pattern control executions print positive -2 for horizontal(-2) and other -2 for vertical(-2), demonstrating that the trailing predicate does not constrain the first alternative.

These ten executions succeed. They exercise known values and mapping policy; this chapter does not fabricate an unknown enum representation. B11 already records the separate resilient-library/new-case unchanged-client experiment, as well as frozen/warn/same-package checking. That evidence is referenced rather than duplicated or generalized to this source-only module.

### Reproduction and limits

Temporary check_a5.py/a5-published-results.json contain the 78 final configurations and ten executions with exact sources, module paths, compiler arguments, diagnostics, stdout, and return codes. Compiler/runtime bounds are 45/15 seconds and core dumps are disabled. The initial matrix passed; the two grouped-pattern controls were then rerun with a stricter diagnostic expectation matching the actual already-handled-case warning.

An earlier nine-configuration exploratory matrix in a5-preflight.py/a5-preflight-results.json checked a plain public enum compiled with library evolution and four client forms in language modes 5 and 6. Its observations agree with the documented missing-fallback error/warning and missing-known-case warning distinction. These preflight configurations are separate from the 78 primary checks.

Runtime coverage is macOS 27 only. iOS and older macOS deployment targets are compile-only; no HTTP requests, UI rendering, authentication flow, retry scheduler, older compiler, non-Apple runtime, new ABI replacement experiment, or asynchronous publication/cancellation implementation was exercised. Exhaustiveness and deterministic mapping results do not certify those separate behaviors. Chapter questions, exercises, flashcards, rubric, migration guide, and links are synchronized, and all permanent changes remain Markdown-only.

## A9 — Enum representation and request-state transitions

Reviewed 2026-09-24. A9 distinguishes case shape, payload validity, allowed transitions, and execution of effects. It preserves legitimate stale-content refresh, corrects blanket deep-copy/Copyable assumptions, checks recursive representation and arithmetic failure, and replaces incomplete request-state sketches with a complete deterministic state/effect model.

### Compiler and runtime coverage

All thirteen Swift fences have dispositions for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. The recursive inline enum and use after consuming LeaseState are intentionally rejected. Dependent probes use the actual published LoadState, LeaseState, or RequestModel source.

The final matrix has 65 compiler configurations with 27 expected compile failures and no unexpected warnings. Four combined applications use the positive chapter sources under Onone/O and both isolation configurations, targeting macOS 13 and running on macOS 27. One additional executable intentionally violates maximumAttempts > 0 and traps with the expected precondition diagnostic. Core dumps are disabled; this process is separate from the positive suite.

The four passing applications verify:

- All eight coarse flag combinations are representable. Idle, loading with no previous value, failed with no previous value, and loaded empty data remain distinct. Loading/failure can preserve existing data; replacing a snapshot does not replace its original copy.
- Copying an enum carrying MutableProfile shares the same object, producing Lin Lin true after mutation. Page rejects nonpositive values; first and numbered(Page(1)) remain distinct under the chosen model. Raw-value decoding and CaseIterable order are checked separately from associated-value equality.
- The recursive expression evaluates to 14, replacement of its copy leaves the original unchanged, and three overflow fixtures throw the typed error. Tree and Array-backed Menu preserve value behavior; no exact allocation count or unbounded recursion guarantee is asserted.
- Borrowing a noncopyable lease repeatedly does not consume it. Explicit consumption and scope exit release the expected resource exactly once; the empty case contains no resource.
- Negative retry delay is rejected. Idle actions, retry exhaustion, permanent failure, limits of 1 and Int.max, zero-delay scheduling, and a new start's reset budget behave as documented.
- Foreign request tokens, old completions/failures, wrong or duplicate timer events, and old cancellation acknowledgments leave the current state unchanged. Matching events advance the expected state and emit the exact start/cancel/wait commands.
- Repeated cancellation preserves the original reason. Replacement and reset retire active requests or retry tickets. Late events after cancellation, completion, reset, or replacement do not restart or overwrite newer work. The short lifecycleProbe returns [2].

Twenty-one auxiliary compiler negatives (seven under each configuration) check non-Sendable reference payloads, a Copyable enum containing a noncopyable lease, an implicitly Copyable generic constraint, attempted automatic CaseIterable with associated values, LoadState's Sendable constraint, the model's private state setter, and a RequestID construction attempt from a separate source file. The latter checks the actual fileprivate boundary rather than pretending two declarations in one file establish external access control.

### Reproduction and limits

Temporary check_a9.py/a9-published-results.json retain all 65 exact compiler commands, sources, diagnostics, expected outcomes, and five executions (four successes and one expected trap). Every configuration passed its first full matrix. Compiler/runtime bounds are 45/15 seconds. a9-preflight-results.json contains two earlier successful object compilations of the exact lifecycle and ownership sources, recorded separately from the primary matrix.

Only macOS 27 runtime behavior was exercised; iOS and older macOS deployment targets are compile-only. The state model returns effects and accepts completion events; it does not implement a network client, actual Task cancellation/joining, a timer, wall-clock delay, UI rendering, or durable identifiers. Tests prove the listed transition/command behavior, not external effect execution or general network retry safety. Arithmetic fixtures are bounded; no adversarial recursion-depth, allocation, ABI, or non-Apple runtime test was performed. A5/B11 provide the separate external-enum evolution checks. Questions, exercises, flashcards, rubric, migration guide, and progress counts are synchronized; permanent artifacts remain Markdown-only.

## A16 — Regex grammar, failure policy, and collection consumption

Reviewed 2026-09-24. A16 removes unsafe forced numeric transforms, misleading failure-dropping language, an incomplete date parser, an incorrect infinity description, and obsolete template references. It teaches existing regex/collection APIs on the 6.4 baseline and makes Apple deployment availability explicit.

### Compiler coverage

All thirteen Swift fences have dispositions for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. The literal-to-integer-capture assignment is intentionally rejected. The report and sample invocation use the actual preceding Event/parser/report declarations.

The final matrix contains 56 compiler configurations, including fourteen expected failures and four debug/optimized executable configurations. All pass; positive compiles are warning-free. Auxiliary negatives check Regex<Substring>'s missing Sendable conformance, attempted tuple-member access on runtime AnyRegexOutput, and an unhandled throwing call. Availability controls reject the unannotated literal parser on macOS 12/iOS 15 and accept the same source with macOS 13/iOS 16 availability annotations.

### Runtime evidence

The exact positive chapter sources run with Onone/O under both isolation configurations, targeting macOS 13 and executing on macOS 27. All four applications produce the recorded output and pass semantic assertions:

- The literal and TryCapture builder agree on zero, ordinary values, leading zeros, Int.max, and a long zero-prefixed value with four permitted spacing forms. Both reject missing/wrong keys or values, signs, overflow, surrounding text, trailing whitespace/newlines/NUL, non-ASCII digit forms, a combining mark, and non-ASCII whitespace.
- first/prefix/whole matching have distinct tested behavior. Runtime pattern construction rejects malformed syntax; a valid nonmatch returns nil, optional captures preserve absence, and an empty pattern matches empty input. The throwing Capture transform propagates through Regex.wholeMatch(in:) and becomes nil through String.wholeMatch(of:).
- Default canonical/grapheme matching accepts the decomposed accent fixture; scalar-dot whole matching does not. Arabic-Indic digit matching succeeds while Int conversion fails; the tested Unicode whitespace class includes newline.
- Exact map/compactMap/flatMap/reduction/helper outputs pass, with dictionary keys sorted before printing. An empty allSatisfy is true. A throwing compactMap stops after its third visited input and returns no partial array; nil-dropping is separately observed.
- The bounded lazy square search evaluates 101 values; repeated materialization of the second lazy map increments its counter from three to six. These are observed consumption counts for this fixture, not allocation or timing measurements.
- EventDate checks every day 0...32 in all twelve months for six selected leap/common/century years (1900, 2000, 2023, 2024, 2100, 2400), with explicit expected month lengths. Invalid extreme years/months/days and the valid year-domain endpoints are checked separately.
- The event parser accepts declared actions, zero durations, leading zeros, valid leap dates, and Int.max fields. It classifies invalid dates, wrong/missing/reordered labels, separators, case, Unicode lookalikes, signs, fractional units, zero IDs, unknown actions, and numeric overflow as documented. Exact 512-byte ASCII and 512/514-byte Unicode boundary fixtures distinguish accepted, invalid-format, and oversized input.
- Empty/all-rejected reports preserve their contracts. Mixed batches retain accepted order, rejected line indices, and exact per-user totals. Int.max plus zero remains valid, separate users can each reach Int.max, and adding one more for the same user throws durationOverflow with the correct original line index. No partial report is returned.

### Reproduction and limits

Temporary check_a16.py/a16-published-results.json retain all 56 final commands, sources, diagnostics, expectations, and four executions. Compiler/runtime bounds are 45/15 seconds. The initial matrix's six diagnostic matchers expected convert value/has no member rather than the compiler's cannot assign value/has no dynamic member wording. Those controls were rejected for the intended reasons; their matchers were corrected and only those six configurations rerun. Initial results are preserved in a16-initial-results.json.

One earlier exploratory successful compile/run is recorded separately in a16-preflight-results.json. It checks the builder, overflow/combining rejection, Unicode behavior, matching-error distinction, and runtime optional captures. The selected SDK's _StringProcessing interface also confirms the availability and try? implementation of the String convenience.

Only macOS 27 execution was tested. iOS and older macOS deployment targets are compile-only. No older compiler, non-Apple runtime, Foundation regex interchangeability, formal Unicode/calendar conformance suite, actual stream/network ingestion, allocation/timing benchmark, hard regex timeout, or arbitrary user-pattern safety guarantee is claimed. Parser validation applies to its emitted values; Event's memberwise initializer is not an enforced external validation boundary. The batch retains its results and does not implement streaming backpressure. Questions, exercises, flashcards, rubric, migration guide, links, and progress counts are synchronized; permanent files remain Markdown-only.

## C9 — Abstraction cost, equivalent work, and emitted dispatch

Reviewed 2026-09-24. C9 replaces blanket optimization claims and incomplete comparisons with equivalent UInt8 transform kernels, explicit dynamic storage choices, measured value layouts, retained closure captures, and a focused emitted-SIL experiment. It distinguishes API semantics from compiler observations and from unperformed timing/allocation measurements.

### Compiler and semantic coverage

All ten Swift fences have object-compilation dispositions for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. Shared protocol/types/apply/pipeline declarations are the actual chapter sources. The opaque function returning Invert versus Clamp is intentionally rejected under all three configurations.

The primary matrix contains 43 compiler configurations, including twelve expected failures and four positive executable configurations. All pass without unexpected warnings. Auxiliary negatives reject a heterogeneous [Invert] literal, a noncopyable type attempting to conform to the Copyable-default PixelTransform protocol, and a Sendable conformance on the eraser with its non-Sendable stored closure.

Four combined applications use Onone/O and both isolation configurations, target macOS 13, and run on macOS 27. They verify:

- Empty/singleton/edge fixtures and every UInt8 value through concrete-generic, existential, opaque, and named wrapper paths.
- Invert-then-clamp equivalence for existential, opened, enum, closure-erased, and fused pure pipelines across five clamp bounds. Reversed transforms have separately computed expected output; no-transform and double-invert cases preserve input.
- Both runtime factory choices through erased and closed types; small/large/object masks with four masks; a multiword LargeMask fixture exercising its truncating XOR conversion.
- Original Array value independence after mutation, without inferring an exact CoW allocation count.
- A closure eraser retaining an ObjectMask after the construction scope ends, and release after the last wrapper is cleared. Weak observations check that lifecycle, not bytes or general closure allocation elimination.
- Exact pass-based versus fused call traces. An auxiliary shared-counter transform produces [4, 7] versus [3, 8], confirming that a legal nonmutating requirement can have observable state and different final output after reordering.

The layout function consistently returns [8, 1, 32, 40, 16] for Int, SmallMask, LargeMask, any PixelTransform, and any ReferenceTransform on this arm64 Apple runtime. These are MemoryLayout.size results, not heap-footprint or allocation evidence; no pointer-layout assumptions are used to manipulate storage.

### SIL evidence

Three additional compiler invocations emit SILGen/Onone, mandatory-pass SIL/Onone, and optimized SIL/O from the exact protocol, concrete transforms, application functions, and public named wrappers (fences 0, 1, and 8). The commands explicitly set Swift 6 mode, nonisolated default, the selected macOS SDK, arm64-apple-macosx13.0, module C9Probe, and a temporary module cache.

The inspection follows function_ref targets among the chapter's apply/wrapper functions:

| Artifact | genericInvert reachable PixelTransform witness calls | knownExistentialInvert reachable calls | unknownExistentialApply reachable calls |
| --- | --- | --- | --- |
| SILGen/Onone | 1 | 1 | 1 |
| Mandatory SIL/Onone | 1 | 1 | 1 |
| Optimized SIL/O | 0 | 0 | 1 |

Both optimized known-Invert wrappers contain the UInt8 subtraction builtin and no existential initialization or reachable transform witness call. The optimized unknown wrapper opens the existential and references the specialized existentialApply helper; that helper is still generic over the opened type and contains the witness call. Thus the word specialized in a symbol/comment is not sufficient to conclude that its runtime transform has become a concrete direct operation.

The wrappers deliberately use inline(never) as observation boundaries. These artifacts do not establish a public-library/client optimization result, a dispatch benchmark, the exact number of runtime allocations, or a portable machine-code guarantee. Standard-library helper calls remain outside the narrowly reported PixelTransform dispatch question.

### Reproduction and limits

Temporary check_c9.py/c9-published-results.json contain the 43 primary commands, exact sources, diagnostics, expectations, and four executions, all passing on the initial matrix. Compiler/runtime limits are 45/15 seconds. check_c9_sil.py, inspect_c9_sil.py, and c9-sil-results.json preserve the three separate emission commands, artifact paths, selected function bodies, reference paths, and checked dispatch results. The initial SIL text parser was refined to delimit individual functions and follow helper targets; existing artifacts were inspected without recompiling unchanged sources.

Only macOS 27 runtime behavior was exercised; iOS and older macOS deployment targets are compile-only. No profiler session, malloc trace, timing/code-size benchmark, cross-module replacement, non-Apple runtime, image decoder, SIMD/GPU/Accelerate implementation, or UI workload was tested. The chapter explicitly leaves those measurements to the product's actual performance question and links F3's measurement workflow. Questions, exercise, flashcards, rubric, migration guide, and progress are synchronized; permanent artifacts remain Markdown-only.

## C10 — Optimization settings, assertion evaluation, and mandatory inlining

Reviewed 2026-09-24. C10 replaces old 6.2.1 output labels, incomplete benchmark/unsafe-pointer sketches, and blanket build-mode claims with current configuration evidence. It separates optimization, DEBUG source conditions, assertion configuration, owned results, arithmetic policy, and the 6.3 optimization controls relevant to the 6.4 baseline.

### Compiler and execution matrix

All twelve Swift fences have object-compilation dispositions for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. Three chapter negatives are rejected in each setting: public inline(always) referencing an ordinary internal helper, an overridable method marked inline(always), and a direct recursive mandatory-inlining cycle. Replacing the public negative's attribute with the old inline(__always) hint compiles in all three settings, demonstrating that the change is not a cosmetic spelling substitution.

The primary matrix has 60 compiler configurations, nine expected compile failures, and 21 process executions. Every final expectation passes; positive compiles are warning-free. Four optimization levels (Onone, O, Osize, Ounchecked) run the combined positive source under both isolation configurations. Two additional complete applications override assertion configuration, and two define DEBUG explicitly. The observable results are:

| Configuration | Assertion count | Precondition count | Bad preparation result | DEBUG result |
| --- | --- | --- | --- | --- |
| Onone default | 3 | 3 | true | absent |
| O default | 0 | 3 | false | absent |
| Osize default | 0 | 3 | false | absent |
| Ounchecked default | 0 | 0 | false | absent |
| Onone, assert-config Release | 0 | 3 | false | absent |
| O, assert-config Debug | 3 | 3 | true | absent |
| Onone, D DEBUG | 3 | 3 | true | defined |
| O, D DEBUG | 0 | 3 | false | defined |

Both isolation settings agree for the four default optimization levels. The override/source-condition controls use the nonisolated default. The valid Ounchecked runs contain no failed assumed contract or invalid memory operation, and the harness uses ordinary if/fatalError checks rather than preconditions that unchecked mode could remove.

Every combined application also verifies:

- Required cache preparation executes exactly once for both success and typed failure, independent of the assertion setting.
- checkedTotal handles empty, positive, negative, sliced, and boundary inputs, and throws for positive/negative/intermediate overflow through addingReportingOverflow. These explicit reporting semantics remain checked in the valid unchecked controls.
- The direct complement function and its mandatory-inline caller return 255 minus the input for all UInt8 values.
- The explicitly specialized modulo-2^64 checksum preserves the documented result for Array, ArraySlice, and AnySequence, including UInt64.max wrapping fixtures. This is separate from the throwing Int aggregation policy.
- Both first-byte APIs return owned optional bytes for empty/edge/all-byte input; no pointer is returned or dereferenced after a borrow scope.

Nine separate processes call the exact chapter termination functions at Onone/O/Osize. Seven terminate by signal as expected: assertion at Onone, precondition at all three modes, and fatalError at all three modes. Assertion at O/Osize returns normally and prints returned. Debug diagnostics are matched where available; optimized termination is not required to preserve identical message text. No failed unchecked condition is executed. Core dumps are disabled and runtime bounds are 15 seconds.

### Mandatory-inlining evidence

Four separate emissions use the exact complement/complementProbe fence, module C10Probe, Swift 6 mode, explicit nonisolated default, the macOS SDK, arm64-apple-macosx13.0, and a temporary module cache. SILGen/Onone still contains the direct complement function reference. Mandatory-pass SIL/Onone, optimized SIL/O, and size-optimized SIL/Osize contain the UInt8 subtraction builtin inside complementProbe with no function_ref to complement.

This is evidence for the named direct mandatory-inline call across compiler stages. It does not assert a speedup, a code-size reduction, universal indirect-call inlining, or a cross-module ABI result. B11's existing module/interface/symbol/unchanged-client tests supply the referenced export, public implementation visibility, helper-eligibility limitation, and explicit-specialization structure evidence; those experiments are not counted again here.

### Reproduction and limits

Temporary check_c10.py/c10-published-results.json record all 60 primary commands, sources, diagnostics, expectations, and 21 executions (fourteen normal returns and seven expected traps). The initial matrix correctly rejected six final-method/recursion controls but the expected text was too narrow. Matchers were adjusted to to be marked final/circular loop, and only those six configurations were recompiled. The initial output is retained in c10-initial-results.json.

check_c10_sil.py/c10-sil-results.json retain the four emission commands, sources, artifacts, selected wrapper body, call references, and builtins. Its first SILGen inspection expected an unmangled-name substring that was abbreviated by symbol mangling; selection was corrected to the named fixture's noinline wrapper. The completed four-stage check passes. Compiler/runtime limits are 45/15 seconds.

Execution was on macOS 27 only; iOS and older macOS deployment targets are compile-only. No real app crash, sanitizer invocation, pointer undefined behavior, concurrency race, physical-device benchmark, timing/code-size measurement, non-Apple runtime, or new binary-replacement test is claimed. The fixed first-byte function returns a value; the chapter does not claim a generic pointer callback prevents escape. Questions, debugging exercise, flashcards, rubric, migration guide, and progress counts are synchronized. Permanent artifacts remain Markdown-only.

## E1 — Objective-C exports, native calls, and asynchronous facade lifetime

Reviewed 2026-09-24. E1 corrects the blanket claim that dynamic always requires Objective-C, replaces incomplete callback facades with a complete contract, and validates both sides of the language boundary. It distinguishes source access, runtime exposure, dispatch, compatible representation, actor isolation, NSError mapping, and lifecycle policy. These interop fundamentals predate Swift 6.4; current compiler/SDK/settings evidence establishes the baseline.

### Published-source compiler matrix

All thirteen Swift fences have compilation dispositions under macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. The isolation-negative fence is compiled with the full profile implementation; the imported calls use the actual temporary Clang module. Other fences are standalone. UIKit/AppKit conditional branches both compile against the relevant SDK.

The primary matrix contains 53 compiler configurations: 32 successful builds and 21 intended rejections, plus two successful executable runs. Positive builds are warning-free. The fifteen Swift negatives reject @objc on a struct, a generic @objc method, a payload enum without an Objective-C-compatible raw type, a selector for an unexposed method, and a synchronous nonisolated call to MainActor cancel. Six Clang negatives reject assignment to the readonly profile name and unavailable service initialization.

For each configuration the fixture emits E1Bridge-Swift.h from the exact chapter formatter, async export, and complete profile exercise. Header inspection establishes:

- ACMProfile has readonly identifier/name/avatarURL properties; Swift-only ProfileLoading and the Swift Profile value type do not export as Objective-C APIs.
- ACMProfileService exports loadProfileWithUserID:completionHandler: returning a request object. The service's inherited initialization entry is unavailable; the Swift composition root supplies the loader.
- ACMProfileRequest exposes cancel while waitForCompletion remains Swift-only.
- The service declaration does not expose MainActor/Sendable callback guarantees. Objective-C clients must follow the documented context; this observation is specific to these generated headers.
- ACMNameFormatter has one NSString format: overload; @nonobjc excludes the Int overload. ACMAsyncNames exports nameFor:completionHandler:.

Clang compiles the chapter's actual Objective-C implementation/client with ARC, modules, Wall/Wextra/Werror, the explicit SDK/target, and writable module cache. It uses the generated header, not a hand-written approximation. Swift imports the chapter's E1Legacy.h through the supplied module map and successfully calls both the NSError-out method and its one-shot completion counterpart's async overload.

### Runtime results

The two complete mixed-language applications run on macOS 27, once for each isolation configuration. They verify:

| Boundary/case | Observed result |
| --- | --- |
| Private @objc selector and observer removal | receive:, responds true, one delivery despite two posts |
| KVO initial/change/invalidate | [0, 1]; change after invalidation does not append |
| Native Swift dynamic | Struct method returns 7 without Objective-C exposure |
| @objcMembers / @nonobjc / runtime name | String returns ADA, Int returns 42, format: responds, ACMNameFormatter resolves |
| Native NSError-out and async import | Both return Ada for known; both throw com.acme.legacy/code 7 for unknown |
| Swift async export called from Objective-C | Returns Ada or com.acme.names/code 7 through completion |
| Profile success | One callback after return, correct immutable fields and runtime name; nil avatar also represented |
| Invalid identifier | com.acme.profile/code 1 and the documented description |
| Cancellation before Task execution | Code 2, including repeated cancel; loader never starts |
| Cancellation after loader entry | Code 2 even if the controlled loader returns success or an ordinary error |
| Ordinary loader failure | Code 3 with underlying domain/code/detail retained in NSUnderlyingErrorKey |
| Known mapped error / CancellationError | Existing profile error preserved; CancellationError maps to code 2 |
| Late cancellation | Does not cause a second callback after successful completion |
| Dropped token | Weak let becomes nil; controlled work still completes and callback delivers once |
| Real Objective-C profile invocation | Generated selector called from native code, returned token awaited in Swift, expected DTO delivered |

A controlled actor loader and continuations order operations without sleeps. The recorder checks result/error exclusivity, callback-after-return, expected count, and MainActor isolation where the assertion API is available. The Objective-C-to-Swift callback uses assumeIsolated only under this adapter's independently specified and checked contract. No arbitrary foreign callback is declared actor-safe.

### Reproduction and limits

Temporary check_e1.py/e1-published-results.json retain every command, source, diagnostic, header assertion, and runtime result. The chapter includes the Swift declarations, native header/implementation, and module map; supply a controlled loader and entry point for the stated acceptance cases. Emit the module/header before compiling the native implementation; then link its object with the positive Swift fences and driver under module name E1Bridge. A representative header command is:

```sh
"$probe_swiftc" -swift-version 6 -default-isolation nonisolated \
  -sdk "$probe_sdk" -target arm64-apple-macosx13.0 \
  -module-cache-path "$probe_dir/ModuleCache" -parse-as-library \
  -module-name E1Bridge -emit-module \
  -emit-module-path "$probe_dir/E1Bridge.swiftmodule" \
  -emit-objc-header -emit-objc-header-path "$probe_dir/E1Bridge-Swift.h" \
  "$probe_dir/Formatter.swift" "$probe_dir/AsyncExport.swift" "$probe_dir/Profile.swift"
```

Here the three named source files hold the exact respective chapter fences. Put E1Legacy.h/module.modulemap in the include directory; compile E1Legacy.m with the matching clang, -fobjc-arc -fmodules -Wall -Wextra -Werror, -isysroot, -target, include path, and module-cache path. Add that object and -I for the Clang module when linking the Swift driver. Alternate isolation uses -default-isolation MainActor plus -enable-upcoming-feature NonisolatedNonsendingByDefault. iOS compilation uses the iPhoneOS SDK and arm64-apple-ios16.0. Compiler/runtime bounds are 45/15 seconds; core dumps are disabled.

Initial fixture checks found a never-mutated weak-var warning in the driver and an ambiguous overloaded #selector expression in the formatter probe. The driver now uses weak let and the selector has an explicit function type; affected preflight checks passed before publication. The primary matrix rejected all readonly assignments correctly, but its initial diagnostic matcher incorrectly required a character between read and only. The three existing Clang outputs were reclassified against assignment to readonly property without recompilation; e1-published-initial-results.json retains the initial results. All 53 final dispositions pass.

The older deployment targets and iOS paths are compile-only; execution was on macOS 27. No UI interaction, real network loader, forced background violation, Objective-C exception, swizzling, arbitrary-object cast safety, performance claim, Linux runtime, or binary replacement is tested here. Cancellation of a permanently stalled dependency is not solved by this facade. Rubric, questions, exercise, flashcards, migration guide, counts, and canonical links are synchronized; permanent artifacts remain Markdown-only.

## E2 — Foundation representations, validated boundaries, and progress availability

Reviewed 2026-09-24. E2 replaces fragmentary examples and unmeasured optimization claims with complete bridge/schema probes and a prepared-renderer exercise. It distinguishes collection membership from shared element references, observable values from storage identity, exact numeric conversion from application policy, UTF-16 conversion from Character boundaries, and source-level conversion counts from performance measurements. Basic bridges and optional boxing are historical features; Foundation progress and Subprocess 1.0 receive concise current-release context.

### Compiler and runtime coverage

All fifteen published Swift fences have compilation dispositions for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. The cache-isolation negative includes the full exercise. Five negatives are checked in each setting: implicit Array-to-NSArray assignment, consumption of one Subprogress twice, a nonexistent completion method on the read-only reporter, an unguarded OS 27 ProgressManager call, and synchronous nonisolated cache access. The unguarded progress fence additionally compiles for macOS 27 and iOS 27, distinguishing deployment availability from unsupported API syntax.

The primary record has 51 compiler configurations: 36 positive and fifteen intended rejections. Four full applications compile and run with Onone/O under both isolation configurations. All final expectations pass and positive compilation is warning-free. The driver uses ordinary if/fatalError checks so optimized runs still validate the expected outcomes.

| Probe/contract | Observed result |
| --- | --- |
| Explicit String array bridge | [A, B] round-trips; mixed String/number input fails the conditional [String] cast and empty input succeeds |
| Mutable reference elements | Swift membership 1, Foundation membership 2, identical first object, updated name visible |
| NSMutableString / NSMutableData | Existing Swift snapshots retain before and bytes [1,2]; native mutable values become beforeafter and [9] |
| Dictionary and set membership | Swift unread 3/archived 10 versus native unread 9; Swift A/B/D versus native A/B/C, compared without order assumptions |
| Opaque struct and optional array | Struct round-trip succeeds but JSON validity is false; nil string element becomes NSNull and optional-string container is valid JSON |
| Numeric schema | Accepts exact 0/1/999/1000, numeric 1.0 and Float 7; rejects Booleans, fractional/nonfinite values, negative/out-of-range/UInt64.max, String, and NSNull with the expected error category |
| Unicode policy | Two Characters/six UTF-16 units; accepts full flag/composed character/end-empty and empty-text ranges; rejects interior grapheme/surrogate, negative, beyond-end, NSNotFound, and overflow-sized ranges |
| Date/URL sanity controls | Concrete epoch Date and HTTPS URL round-trip through NSDate/NSURL |
| New progress composition | Root fractions [0,0.25,0.5,1] as child and root complete their assigned portions |
| Existing Progress interoperation | Explicit reporter child integration yields [0,0.5,1] |

Progress checks use the installed Foundation interface: ProgressManager and ProgressReporter are Sendable/Observable; Subprogress is noncopyable with consuming start. The interface records macOS/iOS/tvOS/watchOS/visionOS 27 availability. The tests cover accounting, explicit legacy integration, ownership rejection, and API availability, not asynchronous observation delivery, cancellation/deinitialization policies, custom properties, or multi-parent cleanup. Those broader behaviors are not inferred from a successful fraction check.

### Prepared-renderer equivalence and limits

The renderer is an explicitly labeled Swift stand-in that validates an NSArray and records UTF-16 arrays. Its cache and renderer are MainActor-isolated; domain RenderRow values are Sendable. The four applications verify:

- Three passes over two stable rows have identical observable UTF-16 output with six unprepared versus two prepared outbound conversion-site executions.
- Reordering reuses entries and changes output order correctly without new conversions.
- Changing caller input does not change an existing prepared snapshot. Calling prepare after changing precomposed to decomposed spelling detects different UTF-16 despite canonical String equality and adds one conversion.
- A duplicate ID throws before cache mutation; subsequent rendering preserves the prior snapshot, retained-row count, and conversion count.
- Removing entries evicts them; empty preparation clears the cache and produces no rows; reinsertion needs a new conversion.
- Invalid non-String input is rejected at the renderer boundary.

This demonstrates semantic equivalence, invalidation, snapshot lifetime, and instrumentation counts. It does not measure heap allocations, real Objective-C renderer work, frame latency, memory savings, or speedup. The chapter explicitly includes preparation/comparison/retention in a future measurement and explains why merely moving a bridge into an adapter or preliminary map need not reduce work. E1 supplies separate real native-call/header evidence; it is not counted again here.

### Source review, reproduction, and limitations

Swift's Array source supplies the documented class-element versus value-element cost model, but its historical rejection example for optional String arrays conflicts with the current compiler and older SE-0140 behavior. E2 uses its checked optional/NSNull result and does not reproduce that stale diagnostic or rely on a private opaque-box class name. SE-0072, SE-0116, SE-0139, and SE-0140 support the historical bridge context; SF-0023 supports the progress role model, while the local SDK establishes concrete availability.

The official Swift 6.4 release and tagged Subprocess 1.0.0 sources were reviewed. Package.swift declares tools version 6.2, the Subprocess product, swift-system dependency, default SubprocessFoundation trait, macOS 13, and an iOS 99 placeholder. The tagged README still includes a 0.4-era dependency example. Coverage intentionally remains brief package/platform guidance: no Subprocess dependency was added, built, or executed, and no process-launch or Linux/Windows/Android behavior was validated. The later E5 review supplies the broader package/toolchain/platform coverage.

Temporary check_e2.py/e2-published-results.json retain commands, exact source, expected diagnostics, and four execution records. Copy standalone fences using the common reproduction command; add the full exercise to the isolation-negative fence. Use -c for the multiple-consumption control so ownership checking runs. For progress, retain the chapter's availability annotation/guard when targeting older OS versions. To repeat the combined application, collect positive fences and an entry point implementing the stated acceptance cases, then compile with -Onone or -O and the documented isolation configuration. Compiler/runtime limits are 45/15 seconds and core dumps are disabled.

The initial combined driver omitted return before a switch expression in its error-category helper; all chapter fence checks already passed. Only the four affected application builds were rerun after fixing the helper. Their final compile/run results pass, and e2-published-initial-results.json preserves the initial diagnostic. Exploratory value/cache/progress-interoperation preflights also passed before publication and are not added to the primary configuration count.

Execution was on macOS 27 only. iOS and older deployment targets are compile-only. No custom Foundation subclass behavior, invalid slice, unsafe external buffer, unbounded workload, real app benchmark, race, or older/non-Apple runtime is claimed. Questions, answers, exercise, flashcards, rubric, migration guide, roadmap, counts, and links are synchronized; all permanent artifacts remain Markdown-only.

## E3 — C ownership, generated safe overloads, and native consumers

Reviewed 2026-09-24. E3 replaces placeholder native functions and a fabricated handle with a complete C11 fixture, real allocation/release functions, fault injection, scoped context callbacks, a Swift owner, annotated imports, generated exports, and a native C main. Existing C interoperability rules and Swift 6.3 @c/global @implementation are distinguished from the checked Swift 6.4 baseline.

### Compiler, header, and runtime coverage

All sixteen published Swift fences are compiled separately for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. The non-Sendable negative includes the complete wrapper. Audited positive examples use -strict-memory-safety -Werror StrictMemorySafety. The unacknowledged raw-call fence compiles ordinarily and is separately rejected under the strict policy. No SafeInteropWrappers experimental flag is supplied.

| Compiler work | Configurations |
| --- | ---: |
| Sixteen Swift fences in three settings | 48 |
| Strict-policy rejection of the ordinary raw call | 3 |
| Two C fixtures in three settings | 6 |
| Swift module/generated-header builds | 3 |
| Native C client compilation | 3 |
| Native client links, each followed by execution | 2 |
| Combined Swift/native applications, each followed by execution | 2 |
| AddressSanitizer C objects and Swift application/link | 3 |
| Total | 70 |

The primary record contains 43 positive compiler configurations and 27 expected rejections. Eight independent negatives are checked in each setting: unavailable function-like macro, capturing C function pointer, escaped Span, nonrepresentable Swift struct export, changed count type in @implementation, Span replacing its raw signature, nil for a nonnull handle, and a ~Sendable owner satisfying a Sendable constraint. The three strict raw-call failures make the remaining negatives. All final positive compilations are warning-free. C compilation uses C11, -Wall -Wextra -Werror.

Three generated headers are inspected for E3ModeIdentity = 0, E3ModeInvert = 1, and E3Transform. They omit e3_swift_sum because its declaration remains in E3Safe.h. Three nm inspections find exactly one defined _E3Transform and one defined _e3_swift_sum in each object; Swift helper/overload manglings may also exist. Six explicit header/symbol assertions pass, with the three nm commands recorded separately from compiler counts.

Five applications run successfully on macOS 27: a native C main in each isolation configuration, a combined Swift/native driver in each configuration, and one AddressSanitizer-instrumented combined driver. The sanitizer run instruments both C fixture objects and Swift code and reports no diagnostic for these executed paths. It does not establish global leak freedom, thread safety, or correctness of unexecuted native code. iOS and older deployment targets are compile-only.

### Ownership, bounds, and ABI observations

| Contract/case | Checked outcome |
| --- | --- |
| Integer/C string policy | 0 and CInt.max accepted; -1, CInt.max + 1, and Int.max rejected. Empty/ASCII/flag/decomposed text returns UTF-8 byte count; embedded NUL is rejected. |
| Normal/empty/boundary input | Empty output, every UInt8 transformed by XOR 0xff, and 4096-byte input pass; 4097 bytes reject before native creation. |
| Copied ownership | Caller mutation after make does not change native input; mutating one returned Data does not change a later result; owned results survive native close. |
| Close and ARC | Two closes destroy once; aliases observe closed state; dropping an unclosed owner destroys it; live handle/buffer counts return to zero. |
| Creation failures | Simulated allocation failure, a transferred handle with failure, nil successful handle, and unknown status map to their expected errors and cleanup counts. |
| Output failures | Allocated partial failure is freed; nil/positive count rejects; nil/zero and allocated/zero return empty while freeing any allocation. Over-limit and SIZE_MAX counts reject before reading their deliberately short allocations. |
| Callback context | Empty, representative, and all-256-byte inputs round-trip through the actual synchronous serial C callback. |
| Safe overloads | Empty, representative, and all-byte inputs give the expected sums through C e3_sum, nativeSum, and the imported safe overload for the Swift e3_swift_sum implementation. Array.span calls execute under OS 26 availability checks. |
| Native exports | The C client calls both sum functions, including NULL/zero, and both exported enum cases for every UInt8. |

The local Apple importer exposes these size_t parameters/out values as Int and the anonymous enum constants as Int, while C int status is CInt. The wrapper keeps unknown CInt status, checks nonnegative/bounded output count, registers cleanup before checking status, and copies native bytes into Data before freeing them. The fixture uses malloc/free internally; callers use its matching destroy/free functions. Test counters and mode selection are sequential instrumentation; ~Sendable does not make those globals thread-safe.

The escapedBorrow fence is deliberately compile-only: @unsafe declares a contract, and returning a pointer out of the buffer scope is not diagnosed as a lifetime escape. It is never called or dereferenced. The corresponding Span escape is rejected. No invalid native bounds, dangling pointer access, retained asynchronous callback, allocator mismatch, enum bit pattern, or data race is executed.

### Reproduction and source limits

Temporary check_e3.py/e3-published-results.json preserve exact chapter fences, compiler/link commands, diagnostics, generated header text, symbol output, and execution outcomes. E3 publishes E3Native.h/E3Native.c and its module map; E3Safe.h/E3Safe.c and its module map; Swift exports/implementation; and client.c. Put those into temporary directories, add their -I paths, and use the common explicit Swift 6/SDK/target/module-cache settings from this record. Use -parse-as-library for Swift definitions and the combined @main driver. Add the whole owner to the Sendable-negative snippet.

For the native client, combine the export and implementation under module E3Exports; emit E3Exports.o with -c and E3Exports.h with -emit-objc-header -emit-objc-header-path. Compile client.c and E3Safe.c with the same SDK/target and include paths. Link their objects with E3Exports.o using swiftc so the Swift runtime is linked, then execute on the matching platform. For the sanitizer driver, rebuild both C objects with -fsanitize=address and compile/link the Swift examples and acceptance-case entry point with -sanitize=address. Never include the compile-only escaped pointer in a runtime exercise. Compiler/runtime timeouts are 45/15 seconds; core dumps are disabled.

Swift's built-in safe-interop macro service could not launch inside the nested execution sandbox (sandbox_apply: Operation not permitted). The temporary validation was therefore run with approved host execution; that infrastructure failure is preserved separately and is not classified as a Swift language rejection. The official safe-interop page's historical 6.2 experimental-flag recipe is distinguished from the current successful no-flag import. SE-0495 and the official compiler native-client test support the export design; concrete target types, header spelling, availability, and matching diagnostics come from the installed compiler/SDK and checked consumer.

The initial published run found only an unnecessary inner unsafe marker in the @unsafe escapedBorrow function. Removing it made the three affected fence compilations warning-free; already passing runtime/native/header checks were not rerun. e3-published-initial-results.json retains that initial warning. Earlier exploratory status-width/unsafe-marker corrections and macro/signature preflights are retained separately and are not counted in the primary 70 configurations.

Questions, answers, exercise, flashcards, rubric, migration guide, roadmap, counts, and links are synchronized. No source, module map, object, header, binary, or validation script was added to the Markdown-only repository. Linux/Glibc and other runtime targets remain untested; C++ interoperability follows in E4.

## E4 — C++20 spans, owned boundaries, and an experimental return-path limitation

Reviewed 2026-09-24. E4 replaces hypothetical image/search types, fabricated reference access, and an incorrect claim that a callback prevents raw-pointer escape with a complete C++20 byte fixture, copied Swift snapshots, and actual Swift/C++ consumers. It distinguishes stable noescape span inputs from experimentally gated lifetime-dependent return overloads and from native-header representation. The experimental runtime failure below remains a documented toolchain limitation, not a passing validation claim or a baseline exercise.

### Primary passing coverage

The primary record has 77 compiler configurations: 43 positive and 34 intended rejections. All fifteen published Swift fences have dispositions for macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. Stable positive Swift examples use -strict-memory-safety -Werror StrictMemorySafety, -cxx-interoperability-mode=default, and -Xcc -std=c++20. C++ checks use C++20/libc++, matching SDK/target, and -Wall -Wextra -Werror. Final positive builds are warning-free.

| Compiler work | Configurations |
| --- | ---: |
| Fifteen Swift fences in three settings | 45 |
| Conditional return-overload compilation with both documented flags | 3 |
| C++ header syntax checks | 3 |
| Generated Swift export modules/headers | 3 |
| Native C++ clients | 3 |
| Missing native-Swift-Span export rejections | 3 |
| Separate Swift adapter modules | 3 |
| Swift consumers with C++ mode | 3 |
| Swift consumer rejection without C++ mode | 3 |
| C++17 rejection for the C++20 header | 1 |
| Native client links and executions | 2 |
| Combined Swift/native Onone/O application builds and executions | 4 |
| AddressSanitizer application build and execution | 1 |
| Total | 77 |

Nine Swift negative fences are checked in each setting: Span input without noescape, unavailable baseline return overloads, local-view escape under the explicit experimental configuration, mutation during a dependent view's borrow under that configuration, twice-consumed E4Buffer, implicit std::string conversion, implicit std::optional conversion, C++ value satisfying Sendable, and synchronous access to the actor. The actor negative includes the complete copy helper and service. Six native/module-consumer rejections plus the dialect rejection make the remaining negatives.

Three generated-header inspections confirm exportedNativeSpan has a C++ std::span parameter, while exportedSum with Swift Span is explicitly described as unavailable because its parameter is not representable. The C++ negative attempts to call the omitted function and fails for the missing member. Separate Swift consumers of E4Adapter compile with C++ mode and fail without it even though they only use SampleSnapshot/checksum. In this toolchain the no-mode failure is reported through missing CxxStdlib/CxxStdlibShim underlying modules; the checked paired build establishes the configuration dependency without inventing a different diagnostic.

Seven primary applications run successfully on macOS 27: two actual C++ mains linked to generated Swift exports; four complete Swift/native drivers across Onone/O and both isolation settings; and one stable driver built with -sanitize=address. The latter reports no sanitizer diagnostic on the exercised paths. Header-defined native functions are part of the mixed-language build; this does not claim instrumentation of every system libc++ implementation, general leak freedom, or race detection. iOS and older deployment targets are compile-only.

### Semantic outcomes

| Contract | Observed result |
| --- | --- |
| Native value copies/destruction | Mutating the copy yields 9 while original remains 1; copy count is at least one; live values return to zero. A separate borrowed read leaves the observed copy count unchanged. |
| Explicit conversion | std::string becomes Swift + C++; present/missing std::optional<int> become CInt 42/nil. |
| Stable span inputs | Guarded default-empty Span/MutableSpan and empty array input pass; one-byte, representative, all-256-byte, and 4096-byte inputs produce the expected modular sum; mutable spans invert every byte. Populated native storage is cleared by the explicit empty C++ span path. |
| Owned buffer replacement | Input is copied, later caller mutation is independent, and returned snapshots remain independent after later replacement. |
| Native exception boundary | Injected C++ exception is caught by replace, mapped to nativeFailure, and leaves stored bytes unchanged; a later replacement succeeds. |
| Resource limits | Swift and direct native replacement reject 4097 bytes; the previous stored value remains intact. |
| Actor boundary | Initial/updated snapshots contain coherent bytes/checksum; sixteen child tasks read the same stable snapshot; oversized replacement preserves it; a later replacement does not change an earlier snapshot. |
| Native C++ clients | Empty, representative, and all-byte std::span inputs call the generated Swift exportedNativeSpan wrapper and return 0, 256, and 32640. |

The E4Buffer owner has a deleted copy constructor and supported move operations. Its data never crosses an actor boundary. The baseline copy helper borrows the owner synchronously, acknowledges the native view and Span conversion as unsafe, copies while the owner is unchanged, and returns only owned [UInt8]. It does not claim that a raw C++ span's Swift lifetime automatically tracks the original vector owner. The fixture's copy counters are sequential instrumentation, and conversion/copy examples do not exercise general allocation exhaustion. Only native replace defines the tested catch-and-status contract; Swift do/catch is not a C++ exception handler.

### Default-empty span control and baseline fix

An additional valid-input control directly called the generated e4Sum overload with Span<UInt8>(), outside the initial array-backed input matrix. It compiled without diagnostics but trapped with signal 5; e4-empty-span-control-results.json preserves that result. The selected Cxx overlay's Span-to-C++ conversion requires a nonnil buffer pointer. Thus a successful empty-array-backed call did not establish support for every empty Span representation.

The published bridge and invert wrappers now handle empty input without calling that generated conversion. exportedSum also handles empty input explicitly. The replacement helper must clear prior native state, so it invokes the raw overload with an explicitly constructed empty E4ReadOnlyBytes under an unsafe acknowledgment instead of returning early. The native empty-span contract is already exercised by the real C++ client. New driver cases check default-empty Span, default-empty MutableSpan, and clearing a previously populated buffer.

Only affected fence, adapter, generated-header, consumer, and primary runtime configurations were rerun after these changes; their final results replace the earlier records and retain the same 77-configuration primary count. All seven primary applications pass with the guards, including the stable sanitizer run. e4-before-empty-guard-results.json retains the prior passing array-backed matrix; the direct empty-span failure remains separate limitation evidence, not a passing application or intentionally invalid input. The unconditional empty call is not rerun after its cause is bounded by the interface contract and the guarded paths pass.

### Quarantined experimental runtime failure

The installed compiler does not generate the tested lifetimebound return overloads in the ordinary configuration. With -enable-experimental-feature SafeInteropWrappers -enable-experimental-feature LifetimeDependence, tailSum and borrowedChecksum compile in all three primary settings; the local-array escape and overlapping-mutation controls are then rejected for their intended lifetime/exclusivity reason. Exploratory flag controls additionally distinguish SafeInteropWrappers alone, LifetimeDependence alone, and Lifetimes alone; the minimal function returning Span compiled with the combined legacy configuration.

Compilation did not establish runtime correctness. A separate experimental driver terminated with signal 11 in both macOS isolation configurations before producing its success line. An isolated diagnostic application then printed before empty tail and aborted with AddressSanitizer reporting stack-overflow in native e4Sum, called through the generated overload and tailSum. It had not reached owner replacement or borrowedChecksum. Consequently, no successful runtime claim is made for either return example, no result is inferred for the later nonempty cases, and the root cause is not asserted. The known failing path is not rerun as part of the passing baseline matrix.

These two failed experimental application builds/runs are retained in e4-experimental-runtime-limitation-results.json and excluded from the primary passing counts. The single ASan investigation is retained in e4-investigation-results.json; it is additional failing diagnostic evidence, not the stable sanitizer run counted above. e4-published-initial-results.json preserves the first matrix's failures and overly narrow no-mode diagnostic matcher. The latter was reclassified using the actual existing CxxStdlib diagnostic without rerunning those compiler calls. No chapter executable source changed as a result of that matcher correction.

The official release describes both-direction span bridging. Current importer source in SwiftifyDecl.cpp explicitly returns without generating a wrapper for lifetimebound parameters/receivers unless the legacy feature is enabled, and Features.def labels that behavior legacy. This supports the configuration explanation; the installed toolchain results determine the curriculum baseline. The safe-interop page and official older tests retain historical experimental recipes. Their source examples are not treated as unconditional runtime guarantees. C11, the migration guide, roadmap verification gate, rubric, and E4 flashcards now carry this distinction.

### Reproduction and limitations

Temporary check_e4.py/e4-published-results.json retain the exact fifteen Swift fences, complete E4Core.hpp/module map, C++ client, commands, header text, diagnostics, and primary runs. Add the helper fence to the service and isolation negative. Use -c for the repeated-consumption/lifetime controls so ownership checking runs. All compiler/runtime commands are bounded to 45/15 seconds and core dumps are disabled. The built-in import macro service needs the same approved host execution described for E3.

For a native consumer, compile the final Swift export block under module E4Exports with the common explicit Swift/SDK/target/module-cache flags, C++20 interop, -emit-module, -c, and -emit-objc-header -emit-objc-header-path E4Exports.h. Compile client.cpp as C++20 with the fixture, generated header, and toolchain usr/include on the include path. Link its object and the Swift object using swiftc with C++ interop enabled; run on the matching platform. The native-Swift-Span export is intentionally unavailable in the generated header. For the stable driver, collect the value/conversion/input/copy/service fences and an entry point exercising the table above; compile with Onone/O or -sanitize=address as appropriate. Keep the conditional return-overload fence compile-only.

Earlier exploratory custom SWIFT_NONESCAPABLE member-return probes remained unavailable under the tested single-flag configuration and were not published as working examples. Reference-counted C++ mappings, unsupported template families, Windows/Linux ABI behavior, SwiftPM dependency propagation as a full package graph, and actual performance timings remain source-reviewed context or outside this fixture. E5–E8 cover package/distribution contracts separately; they do not establish all of these C++ scenarios. The direct separate-module consumer test is narrower than a package-distribution test. No invalid native span, dangling-pointer access, escaping C++ exception, unbounded allocation, or race is intentionally executed as a teaching exercise.

The chapter, questions, answers, exercise, flashcards, rubric, migration guide, roadmap, counts, and canonical links are synchronized; permanent files remain Markdown-only.

## E5 — SwiftPM boundaries, resources, Swift Build, and SBOM contents

Reviewed 2026-09-24. The chapter replaces placeholder remote SDK dependencies, fragmented 6.2 manifests, and a fixed-response loader with three complete local packages: ProfileFeature, its conditional DeviceMetrics fixture, and an ordinary Consumer. It corrects the claim that omitting a target's own product hides its public declarations. Explanations, questions, exercise, flashcards, rubric, migration guide, and roadmap are synchronized.

### Configuration and concrete evidence

The installed Swift compiler is the common 6.4 release build recorded above; swift package --version literally reports Swift Package Manager - Swift 6.4.0-dev. Manifests use tools 6.4 and Swift language mode 6. ProfileFeature declares macOS 13/iOS 16 minima and explicitly nonisolated target defaults. The alternate test package changes only the settings to defaultIsolation(MainActor.self) plus NonisolatedNonsendingByDefault. The same source/resource bytes pass both configurations. UI intent remains explicitly MainActor, domain helpers are explicitly nonisolated, and storage is an actor.

There are fifteen published Swift fences: eleven complete package/source/test files and four intended failure sources. Four resource files are specified separately. Fifteen SwiftPM commands are classified: eight successful commands and seven intended rejections. Counts describe command-level evidence, not every compiler invocation inside a build. All seven failures were checked for the intended diagnostic, not merely a nonzero status.

| Check | Observed result |
| --- | --- |
| Baseline swift test | Four tests pass across two test-target runners: normalization, invalid inputs, store resources/errors, and production/test resource ownership. |
| Alternate isolation swift test | The same four tests pass with MainActor defaults and NonisolatedNonsendingByDefault. |
| Ordinary external consumer | Imports ProfileCore/ProfileStore/ProfileUI after depending on ProfileKit; prints Hello: Ada and macOS sample after checking real resources. No @testable import. |
| macOS ProfileKit build | Swift Build produces the static library, target modules, and separate resource bundles. |
| iOS ProfileKit build | --triple arm64-apple-ios16.0 with the iPhoneOS SDK compiles the conditional DeviceMetrics edge and Module::Name use. No iOS execution is claimed. |
| Package description | tools 6.4/mode 6, two product roots, transitive ProfileCore membership, and no production membership for ProfileTestSupport. |
| Artifact location | --show-bin-path with matching arguments returns the actual build-mac/out/Products/Debug directory, containing libProfileKit.a and resource bundles. This is an observation, not a path contract. |
| Product-name import | import ProfileKit fails because that product creates no such module. |
| External package/internal helpers | DisplayNamePolicy and CoreMarker are unavailable to Consumer, while the ordinary public Core API remains usable. |
| Sibling internal helper | A source in ProfileStore cannot use ProfileCore's internal CoreMarker. Its package policy use succeeds in the production build. |
| Nonexistent external product | Depending on .product(name: "ProfileCore", package: "ProfileFeature") fails because the product does not exist. |
| Cyclic target graph | A separate minimal One -> Two -> One manifest is rejected for the cycle. |
| Invalid SBOM target selection | --target ProfileCore combined with --sbom-spec is rejected. |

The resource assertions check the processed profiles.json, the copied Raw/metadata.txt directory, production prefix Hello, and the test target's different prefix TEST ONLY. The nil Bundle.main assertion is specific to this command-line host, not a universal statement about main-bundle contents. The external executable verifies resource delivery beyond the package's own test targets. Asset catalogs, localization, storyboards, and Metal resources are source-reviewed context and are not exercised by these text/JSON fixtures.

### SBOM contents and explicit schema limitation

Five emitted JSON files were parsed and inspected:

| Generation route | Documents | Package/product component names |
| --- | --- | --- |
| macOS Swift Build, --product ProfileKit | CycloneDX 1.7 and SPDX 3.0.1 | profilefeature, ProfileKit |
| Standalone package generate-sbom, same product, no explicit specification | CycloneDX 1.7 and SPDX 3.0.1 | profilefeature, ProfileKit, devicemetrics, DeviceMetrics |
| iOS Swift Build, same product | CycloneDX 1.7 | profilefeature, ProfileKit, devicemetrics, DeviceMetrics |

The result demonstrates build-time filtering of the iOS-only dependency. The standalone command warns that its graph lacks build-time conditions. All three generation routes also warn: Bundle 'SwiftPM_SBOMModel' with schemas not found - skipping SBOM validation. Generation succeeds, but schema conformance has not been established. The fixture's local-path packages have unknown versions and cannot establish immutable release identities. No vulnerability, license-compliance, reproducibility, or complete runtime-component claim follows from this inventory.

Swift Build's default, per-target test runners, output-path changes, Apple resource-rule parity, deprecated alternative engines, and known import-check/SDK/plugin issues were checked against the SwiftPM 6.4 release notes and CLI help. The native fallback was documented, not executed in this package validation. Android/Java, Wasm, Embedded Swift, and non-Apple host/SDK guidance remain brief source-reviewed context; no corresponding SDK installation, compilation, or runtime is claimed. A sandboxed exploratory swift sdk list failed while trying to create its default support directory, so that failure was not interpreted as evidence that SDKs are absent.

### Reproduction and scope

Create the chapter's sibling ProfileFeature, DeviceMetrics, and Consumer directories and copy the eleven Swift files plus four resource files to their shown paths. Select the recorded Swift toolchain. Run swift test in ProfileFeature, swift run Consumer in Consumer, and the chapter's product/build/SBOM commands. All validation files live below /private/tmp/swift64-interview-validation/e5; nothing executable is added permanently to the repository.

The recorded commands isolate --cache-path, --config-path, --security-path, and --swift-sdks-path below that temporary root. Builds also use explicit --scratch-path directories except the first test run, whose normal scratch output stays inside the temporary ProfileFeature package. The iOS command adds the SDK path /Applications/Xcode_27.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk and --triple arm64-apple-ios16.0. MainActorFeature is a temporary copy with only the described manifest settings changed. Negative source replacements are restored after each check; the cycle uses its own package.

Swift build/test output is piped through xcsift with raw tee logs and pipefail. The formatter reports the last two-test runner as two passed tests; raw output proves two runners with two tests each. It must not replace exit-status or complete-runner review. Compiler diagnostics in positive package sources are clean; SBOM tool warnings are retained as described above. Runtime evidence is macOS 27 only, and these small trusted bundle reads are not performance measurements or proof of asynchronous disk I/O.

Temporary e5-published-results.json consolidates the fifteen command dispositions, five SBOM inspections, source snapshots, and explicit schema limitation. check_e5_packages.py and check_e5_followup.py record exact argument lists and raw logs; finalize_e5_evidence.py compares the final fifteen Swift fences byte-for-byte with the checked sources and reclassifies retained output without rerunning successful builds. Root structural checks also require the rubric's consumer probe to match. The two initial tool invocations and their raw logs supply the baseline test and macOS SBOM build results.

## E6 — Import access, lookup, transitive loading, and linkage

Reviewed 2026-09-24. The complete chapter replaces uncompiled framework examples, a fixed-output mapping, an incomplete cancellation-unaware network wrapper, and old Swift 6.2.1/Linux diagnostics with a real ThirdParty decoder, a deliberately exposed LeakySDK, and a domain-owned ProfileSDK. Eleven published Swift source blocks are matched byte-for-byte to the checked files. The exercise, answers, flashcards, rubric, roadmap, migration guide, and F5 settings cross-reference are synchronized.

### Compiler and consumer coverage

The final record contains 73 compiler/linker commands (29 intended rejections), two successful consumer executions, and three paired interface inspections: 78 classified rows. Every published source block is checked under macOS 13/nonisolated, iOS 16/nonisolated, and macOS 13/MainActor plus NonisolatedNonsendingByDefault. The vendor/SDK modules use Swift 6 mode, explicit SDK/target/module-cache paths, library evolution, and testing enabled for the separate testing-access control. Focused feature/access/loading variants are macOS checks unless noted below.

| Boundary | Checked result |
| --- | --- |
| Explicit import ceilings | Public/package/internal/fileprivate/private signatures compile at their allowed level. Public exposure through the four lower levels is rejected. open import is rejected. |
| Ordinary implementation use | A public scalar function can use a privately imported vendor extension in its nonserialized body. |
| Per-file ceiling | An additional file with public import does not fix another file's public signature through internal import. |
| Plain-import setting | Baseline public parameter/result compiles; InternalImportsByDefault rejects it. Explicit public and explicit internal implementations retain their intended behavior with the feature enabled. |
| Public import versus re-export | Importing only LeakySDK does not make the VendorRecord top-level name available in the independent client. |
| Member lookup | The transitive String extension compiles under the baseline. MemberImportVisibility rejects it; adding the defining module's explicit import passes. |
| Module selector | ThirdParty::VendorDecoder bypasses a local variable named ThirdParty, but not a missing import or the internal ceiling on a public result. |
| Signature/body/layout leakage | Typed vendor error, vendor result/typealias/enum payload, an internally imported operation in an inlinable body, and a vendor field in a frozen public struct have the intended access failures. The public-import inline variant compiles. |
| Legacy spelling | @_implementationOnly compiles with the expected deprecation warning, recommending internal import. |
| Unsupported import attribute | @usableFromInline applied directly to internal import is rejected by this installed compiler; it is not presented as a working fix. |

The unsupported-attribute probe was initially expected to compile, based on broader import/ABI design context. Its actual attribute rejection is preserved in e6-initial-results.json and then classified as an explicit negative control without repeating that compiler call. The public-import inline alternative is separately compiled. No baseline warning-free claim includes the deliberately deprecated import; other successful compiler controls are warning-free.

### Module availability is not link availability

For each of the two macOS isolation configurations, the ordinary Client source compiles using only a staged ProfileSDK.swiftmodule, and separately using only ProfileSDK.swiftinterface. Neither staging directory contains ThirdParty or passes its module directory as an import search path. The public interfaces in all three platform/settings configurations contain neither a ThirdParty import nor vendor type names; the paired LeakySDK interfaces expose the vendor import, result, and typed error. The interface-only check uses the same installed compiler and does not establish cross-version module stability.

Omitting ThirdParty.o from the client link fails with the expected undefined symbols in both macOS configurations. Linking Client.o, ProfileSDK.o, and ThirdParty.o succeeds, and both executables print ProfileSDK public consumer passed. Runtime assertions cover actual field mapping, a Unicode name, missing/extra separators, empty ID/name, 4097-byte rejection, and a valid exactly-4096-byte payload. This demonstrates a source/compile boundary with a remaining code dependency; it is not a dynamic-framework embedding or binary-swap test.

Additional macOS clients without the vendor module verify the loading distinctions: a non-resilient build of the same SDK source fails; @testable import of the testing-enabled resilient SDK fails; a publicly imported vendor dependency fails; a separate package-import fixture fails for a same-package client but permits an ordinary public scalar call for a different-package client. The diagnostic is missing required module ThirdParty in each intended failure. The package identities are explicitly E6Library and E6Client. These controls prevent a missing search path or an unbuilt SDK from masquerading as an access-control result.

### Reproduction and limits

Temporary check_e6.py/e6-results.json retain the primary commands and source variants; check_e6_additions.py records the explicit-feature/inline alternatives and interface inspections. The final e6-published-results.json contains all 78 classified rows and e6-published-sources.json preserves the eleven final fences. The earlier preflight's nonisolated @main entry was rejected and corrected to an explicit MainActor main before the complete matrix and final publication; it is not counted as a published passing source.

The chapter's host recipe shows the essential isolated module/object directories. For the recorded matrix, use /Applications/Xcode_27.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swiftc with -swift-version 6, the selected -default-isolation, -module-cache-path, explicit macOS/iPhoneOS SDK, and arm64-apple-macos13.0 or arm64-apple-ios16.0. Build each library with -parse-as-library, -enable-library-evolution, -enable-testing, -emit-module, -c, -module-name, -emit-module-path, -emit-module-interface-path, and the shown source/object paths. ProfileSDK/LeakySDK build with ThirdParty on their import path. Consumer staging deliberately omits that path. The non-resilient control removes library evolution; feature controls add the named upcoming flag. Commands are bounded to 45 seconds and all artifacts remain under /private/tmp/swift64-interview-validation/e6.

The parser fixture uses a deliberately simple two-field grammar with SDK-owned nonempty ID/name validation; it is input-dependent code, not a no-op network stub. No transport, cancellation bridge, vendor-package integration, non-Apple runtime, dynamic-library swap, or performance benefit is claimed. Runtime checks execute on macOS 27; iOS is compile-only. For actual networking, matching cancellation/error/timeout behavior remains a separate requirement and D9 supplies the relevant bridge coverage.

## E7 — Binary replacement, protocol evolution, and actual XCFramework consumers

Reviewed 2026-09-24. The chapter now distinguishes source/ABI/module/behavior/deployment contracts with complete paired SDKs and real consumers. It corrects the blanket claim that adding a public protocol requirement always breaks old conformers, removes incomplete no-op examples and a hardcoded project-wide iOS minimum, and replaces hypothetical framework trees with an inspected XCFramework. Eight published Swift fences match the six core source files and final binary-package manifest/consumer.

### Primary evidence

The primary record contains 53 classified rows: 35 direct compiler/linker commands, eight direct dynamic-client executions, three ABI-digester commands, five inspections, one xcodebuild packaging command, and one final SwiftPM build/run command. Two direct compiler controls are intended rejections: a protocol conformance after removing its new default, and an iOS-device framework used for Simulator. The digester's detected ABI break is a diagnostic assertion, not a nonzero-exit assertion.

Core SDK versions, old/new clients, and frozen-layout definitions are compiled for arm64-apple-macos13.0, arm64-apple-ios16.0, and macOS 13 with default MainActor plus NonisolatedNonsendingByDefault. Libraries use Swift 6 mode, explicit nonisolated public contracts, -enable-library-evolution, emitted binary/textual modules, and an explicit SDK/module cache. Ordinary clients use an explicit MainActor entry point. Optimized dynamic clients execute only on macOS 27.

| Loaded library / consumer | Output in both macOS isolation configurations |
| --- | --- |
| v1 / original optimized executable | 42 \| queued \| hello \| 1 |
| v2 / unchanged original executable | 42 \| unrecognized \| hello + default \| 1 |
| v2 / identical original source rebuilt | 42 \| unrecognized \| hello + default \| 5 |
| v2 / source updated to handle paused | 42 \| paused \| hello + default \| 5 |

The preserved executable SHA-256 hashes match before and after replacing the loaded dylib file in each configuration. Each check starts a new process; there is no live image replacement. Reading changes private storage from Int to String plus [Int], while copied public values still return 42. The old enum switch handles the new case through its fallback. The old protocol conformance lacks the new suffix requirement but uses its library-supplied default witness. A separate no-default v2 compiles, while the old source client is rejected for its missing conformance requirement; that incompatible library is not executed with an old client.

Rebuilding the unchanged source emits the expected missing-known-case warning and changes the caller's default argument from 1 to 5. The updated switch is warning-free. Staging only the v2 textual interface without the binary Swift module permits both macOS client configurations to compile. Only Xcode_27.app was found in /Applications, with no toolchains in the two inspected alternative installation locations. This is same-compiler interface evidence, not a Swift 6.2/6.3 import claim or an exhaustive inventory of every possible user toolchain location.

### Frozen ABI detection and tool-status limitation

FrozenV1/FrozenV2 retain the public initializer but add a private stored Int. Both compile in all three core settings, and the new stored field appears in the emitted frozen interface. swift-api-digester -dump-sdk -abi produces the two baselines. The comparison uses -diagnose-sdk -abi -error-on-abi-breakage with both input paths. It reports that FrozenReading.reserved is added to a non-resilient type and also reports the new API's missing availability annotation.

Both diagnostics are warnings and the comparison returns zero in this installed toolchain. The initial expected-nonzero classification is preserved in e7-initial-results.json; the actual warning/status behavior is then classified without repeating the command. The chapter explicitly requires diagnostic inspection rather than treating the status as proof of compatibility. No old binary is run against the deliberately changed frozen layout.

### Three slices and a real binary-package launch

Three dynamic frameworks are built from LibraryV2 with matching SDKs, explicit install name @rpath/E7SDK.framework/E7SDK, textual/binary modules, and framework metadata. xcodebuild -create-xcframework packages them successfully. The inspected AvailableLibraries entries are ios-arm64, ios-arm64-simulator, and macos-arm64, each with architecture arm64 and the correct platform/variant. The consumer compiles against each selected packaged framework. A Simulator compile against only the device slice is rejected. Intel variants, iOS/Simulator execution, signing identity, release archives, App Store packaging, and cross-toolchain imports are not tested.

The first SwiftPM BinaryConsumer manifest has a local binary target but no explicit runpath. Swift Build compiles/links successfully and copies E7SDK.framework beside the executable. Launch then terminates with status 134: dyld cannot resolve @rpath/E7SDK.framework/E7SDK. otool shows no executable-relative runpath for that copied location. This failed launch remains separately recorded in e7-framework-results.json and is excluded from the primary passing rows.

A single native-backend diagnostic control builds and runs the original manifest. An unchanged original Swift Build executable also runs with DYLD_FRAMEWORK_PATH pointing at the selected macOS slice, confirming the runtime-location distinction. An intermediate explicit Frameworks subdirectory/runpath setup also succeeds; it remains a diagnostic alternative, not the final published recipe.

The final manifest adds the macOS root executable's linker runpath @executable_path. A fresh scratch build under the default Swift Build backend copies the framework alongside the executable and swift run prints XCFramework consumer: 42 without manual copying or an environment override. A separate load-command inspection verifies the final relative runpath and the absence of the intermediate Frameworks directory. The final manifest/source are stored in e7-binary-package-sources.json and matched to the chapter. This scoped CLI correction is not presented as a universal app embedding policy or proof that every Swift Build binary target fails.

Six supplementary diagnostic rows in e7-embedding-results.json preserve the native run, environment-path run, intermediate build/bin-path query/run, and load-command inspection. The final recipe is e7-final-binary-consumer-results.json; it is the only package build/run counted in the primary 53 rows. xcodebuild and SwiftPM output pass through xcsift with raw tee logs and pipefail. The initial failed runtime is retained even though compilation/packaging succeed.

### Reproduction and scope

Temporary check_e7.py retains core SDK/client commands, hashes, outputs, and ABI checks. check_e7_frameworks.py records framework construction, packaging, slice metadata, and the initial runtime limitation. fix_e7_framework_embedding.py and final_e7_binary_consumer.py retain the diagnostic/final configurations. e7-published-results.json consolidates the primary evidence, supplemental controls, initial failure, and exact eight published Swift fences. Do not rerun setup/packaging scripts over existing artifacts without inspecting their non-idempotent stages.

Use the common installed swiftc/SDKs with -swift-version 6, explicit target/default isolation/module-cache options, -parse-as-library, -enable-library-evolution, -emit-module, -emit-library, and -emit-module-interface-path. Both dylibs share @rpath/libE7SDK.dylib; the client links with an rpath to its temporary live directory. Interface-only staging omits binary modules. The framework compile additionally uses arm64-apple-ios16.0-simulator with the iPhoneSimulator SDK and architecture/platform-specific module filenames. The final package isolates cache/config/security/SDK/scratch paths below /private/tmp/swift64-interview-validation/e7. Compiler/core commands are bounded to 60 seconds; package commands to 180 seconds with observable tool sessions.

The published host commands show the same essential stages with simpler paths. Source-reviewed release guidance includes Xcode archive/distribution settings, supported-toolchain policy, and Swift 6.4's changed debug-module tracking. No archived LLDB expression/relocation test, release signing, physical-device run, performance claim, or unrelated ABI compatibility guarantee follows from this fixture. F2 remains the debugger teaching home and E8 retains body/export/optimization details.

## E8 — Body visibility, policy boundaries, and utility contracts

Reviewed 2026-09-27. Nine published Swift fences exactly match e8-sources.json. The final record e8-published-results.json has 54 classified passing rows: 39 compiler/linker/SIL-emission commands, eight runtime executions, and seven inspections. Nine compiler controls are expected rejections: the private helper, private usableFromInline annotation, and external access to the ABI-visible internal helper, each under three settings.

The matrix uses Swift 6 mode, arm64 macOS 13 and iOS 16 deployment targets, and an alternate macOS MainActor default with NonisolatedNonsendingByDefault. Library contracts are explicitly nonisolated. Separate dynamic modules use -enable-library-evolution and -O; clients have explicit MainActor entry points. Only macOS 27 executes. No new feature flag is needed for the 6.3 export/inlining/specialization controls.

### Binary and interface evidence

E8Visibility v1 returns 1 through four public functions: export(interface), export(implementation) with inline(never), inlinable, and inline(always). v2 changes only those literals to 2. Under both macOS settings, the original optimized client prints 1 1 1 1 with v1; the same executable prints 2 1 1 1 after replacing only the dylib; the rebuilt client prints 2 2 2 2. The preserved executable SHA-256 hashes agree. Each launch is a new process.

The textual interface hides libraryRevision's body and exposes the other three. nm -gU finds libraryRevision, visibleRevision, and forcedRevision in the library, but no clientRevision entry point. The fixture distinguishes client emission from inlining and does not generalize its optimized outcome to every inlinable call site.

E8Utilities combines the revised utility, checked index function, and specialized generic function. The naive policy-exposing utility compiles separately. Its revision uses internal import Foundation and explicit interface exports for policy. The resilient interface omits Foundation and eventPrefix, preserves the clamp body and usableFromInline helper declaration, and keeps containsZero's algorithm hidden. The helper's callable symbol is present while ordinary client source lookup rejects it. Both macOS configurations also compile an ordinary client from only the textual interface.

### Correctness and generated-code evidence

The external utility client prints utility boundaries passed under both macOS settings. Assertions cover trimming/case and analytics fixture policy, clamp values including Int.min/Int.max, capacity one, capacity Int.max with the largest valid indices, invalid capacity and index domains, empty and nonmatching arrays, a different integer element type, and a ClosedRange. Invalid inputs produce the stated typed error. No invalid pointer, overflow, modulo-zero, or incomplete ring-buffer implementation is executed.

The optimized containsZero SIL compares the generic type metatype with Array<Int>, branches to an array-specific path, and retains a generic Sequence witness path. This establishes generated structure, not latency or allocation improvements. Existing B11/C10 records supply the broader mandatory-inlining and export-combination diagnostic boundaries; they were not rerun without a new change.

### Reproduction and limits

Temporary files are under /private/tmp/swift64-interview-validation/e8. check_e8.py records each exact command, status, output, and expectation in e8-results.json; e8-published-results.json adds the interface/symbol/SIL assertions and all published source text. setup_e8.py created the fixtures; write_e8.py published the chapter. As elsewhere, these are temporary evidence, not repository dependencies.

For the binary experiment, compile Visibility.swift and its literal-2 revision separately with -parse-as-library -O -enable-library-evolution -emit-module -emit-library, module name E8Visibility, and install name @rpath/libE8Visibility.dylib. Compile the same VisibilityClient.swift against each version, link the old object with a runpath to a separate live directory, preserve its hash, and replace only that directory's library between launches. For the utility exercise, build UtilityAfter.swift, Index.swift, and Specialized.swift together as E8Utilities, then compile/link UtilitiesClient.swift as an ordinary consumer. The common SDK, cache, language, isolation, and target flags are stated above and in Reproduction.

The old Swift 6.2.1 diagnostic labels, incomplete storage-dependent ring buffer, and stale heuristic-only optimization discussion are removed. B11's checked limitation for internal export(interface) helpers remains explicit. There is no older-toolchain import test, iOS runtime execution, Embedded/non-Apple validation, signing result, or performance claim.

## Whole-collection audit — 2026-09-27

All 70 chapter reviews are complete. The final editorial pass reconciled README, roadmap, migration guide, rubric weighting, study order, and the minimum interview loop. The migration guide now separates practical 6.4 changes, 6.3 catch-up, earlier foundations, availability, and source/compiler discrepancies. Specialist ownership and binary tooling do not become mandatory senior syntax trivia merely because they are new.

### Inventory and navigation

| Check | Final result |
| --- | --- |
| Topic organization | 70 notes: A 16, B 14, C 12, D 14, E 8, F 6; IDs preserved |
| Markdown inventory | 75 files, 50,577 lines, 1,271 Swift fences |
| Chapter structure | All 70 retain sections 0–12, baseline context, and a review date |
| Roadmap | 70 reviewed dispositions; editorial and release gates closed |
| Local navigation | 234 Markdown file links and 753 wiki links resolve with canonical spelling/case |
| Public sources | 305 distinct linked web destinations return HTTP 200 after redirects |
| Repository shape | Markdown curriculum plus existing LICENSE/.DS_Store; no validation source, packages, binaries, or caches added |
| License | Byte-for-byte equal to HEAD; SHA-256 1f3ac190126d93427d6ebf433e7fd0766ce990eca949157d3f5096ba7f8d0e82 |

The line/fence counts describe the final collection, not independent runnable programs. Fences include intentional failures, module components, compiler-only examples, and rubric repetitions. The starting inventory in the roadmap is labeled historical. D14 retains its old-title alias; current incoming links use the durable title.

The audit corrected malformed C9/C10 roadmap rows, removed obsolete remaining-batch/follow-up statements, added editorial dates where absent, and formatted 110 older bibliography entries as Markdown links. Five broken destinations were repaired: the checked-throwing-continuation API, moved default-isolation/strict-memory-safety/language-mode documentation, and a misspelled Async Algorithms proposal filename. The setting references now use the relevant official evolution proposals. HTTP success establishes reachability, not the correctness of every page claim; chapter reviews and compiler evidence provide the semantic support.

### Retained evidence and temporary-file limits

The aggregate consistency checker rechecked the surviving result records, documented expected failures, published source text where recorded, corresponding rubric probes, chapter structure, and repository shape. It does not rerun compiler commands or convert prior observations into language guarantees. Previously recorded compiler crashes and the experimental C++ runtime failure remain explicit limitations and were not repeated for this audit.

Three result files required by that checker were unavailable at final audit: a4-published-results.json, d6-published-results.json, and d9-published-results.json. The permanent chapter review records above preserve their earlier execution summaries. The checker reports these as historical-record-only instead of manufacturing replacement results or claiming their complete matrices were re-executed. Temporary validation artifacts are intentionally not repository dependencies and may not remain available indefinitely.

A fresh focused regression for those three chapters adds 36 classified rows: 30 compiler/link commands, including eight intended rejections, and six successful macOS executions. Both explicit nonisolated and MainActor plus NonisolatedNonsendingByDefault configurations pass. A4 checks nested dictionary nil states, optional opaque/existential spelling, noncopyable extraction, and compilation of the complete state model. D6 checks suppressed conformance, safe transfer, rejected retained aliases/non-Sendable captures, and the joined actor fix. D9 checks typed checked/noncopyable continuation success and failure, noncopyable payloads, consumed-handle rejection, and compilation of the complete adapter. These checks run on macOS 27; they supplement, rather than replace or restate, the earlier broader matrices.

One regression matcher initially expected the word suppressed while the compiler correctly reported explicitly suppresses. The actual nonconformance was inspected and reclassified without rerunning that command; the original classification is preserved separately. No published Swift source needed a repair from this focused run.

E8 contributes its separately recorded 54 passing expectation rows, including fresh binary replacement, interface/symbol inspection, and arithmetic-domain tests. No benchmark result is inferred from its SIL.

### Reproduction and completion boundaries

Temporary audit tools are audit_release.py, check_review_structure.py, check_release_regression.py, and check_source_links.py with its follow-up. Their outputs are release-audit-results.json, release-regression-results.json, and release-source-link-results.json under the common temporary directory. The Markdown record and chapter sources are the durable evidence and reproduction context. git diff --check passes after the final edits.

Runtime evidence is from the recorded Apple Swift 6.4/Xcode 27 arm64 macOS environment. Selected older deployment targets and iOS/Simulator slices were compiled, not executed on those systems. Older compiler imports, Intel clients, non-Apple runtimes, release signing, physical devices, relocated archive debugging, and SBOM schema conformance remain unverified where stated. Known unsupported or experimental paths are labeled and excluded from working baseline recipes. These explicit coverage limits do not leave a curriculum migration batch pending.

## Editing and review coverage

| Material | Completed work | Coverage boundary |
| --- | --- | --- |
| A1–A16, B1–B14, C1–C12, D1–D14, E1–E8, F1–F6 | Explanations, questions, answers, probes, exercises, production guidance, flashcards, sources, and links reviewed | Chapter-specific compiler/runtime/source-review limits retained |
| README, rubric, roadmap, migration guide | Baseline, navigation, study priorities, minimum interview loop, release-status distinctions, and final progress synchronized | Focus remains senior/staff iOS/macOS, with brief cross-platform context |
| Validation record | Settings, expected outcomes, observed results, reproduction context, historical evidence, and final audit recorded | Temporary artifacts are not permanent checked-in test infrastructure |
