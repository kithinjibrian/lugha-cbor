# PRP-001 — Decode and encode CBOR

**Status:** built and merged, session 3 (approved by the owner the same session). DECISION-002 resolved by lughac m108.
**Branch:** `prp-001-decode-encode`
**Source:** the owner's brief, session 2 (2026-10-17); answers to the two gaps are DECISION-002 and DECISION-003.

## FEATURE: A `Value` type for the whole CBOR data model, a decoder that is safe on hostile input, and a preferred-serialisation encoder.

## OBJECTIVE

Any Lugha program can turn CBOR bytes (RFC 8949) into a `Value`, or get a typed error, whatever the bytes are, and turn a `Value` back into bytes. The decoder is built for bytes from untrusted programs: every claimed length is checked before it is used, depth, items and bytes are limited, and nothing recurses.

```
import cbor;

fun main() {
    let bytes = cbor.encode(cbor.Value.Map([
        cbor.Entry { key: cbor.Value.Text("kind"), value: cbor.Value.Text("text") },
        cbor.Entry { key: cbor.Value.Text("size"), value: cbor.Value.int(-3) },
    ]));
    match cbor.decode(bytes, cbor.Limits.default()) {
        Ok(v) => println(kind_of(v)), // Some("text")
        Err(e) => println("bad data: {e}"),
    }
}

fun kind_of(v: cbor.Value): Option<string> = v.get("kind")?.as_text();
```

## CONTEXT

- **Starting state:** `src/lib.la` and `examples/hello.la` are the `lughac init` placeholders; `test/` is empty.
- **Ending state:**

```
src/lib.la        entry: re-exports the public API ({ * } from each module below); greet() removed
src/value.la      Value, Entry, SimpleValue, the reading helpers, Value.int
src/limits.la     Limits, Limits.default()
src/error.la      DecodeError, Malformation
src/reader.la     Reader: one head (initial byte + argument), definite and indefinite
                  byte/text strings (strict UTF-8), and the item/byte budget, all
                  bounds-checked (*)
src/decoder.la    decode(): the iterative decoder (an explicit frame stack) (**)
src/keys.la       duplicate-key detection (DECISION-003)
src/float.la      half/single/double <-> f64, on the integer bits (DECISION-002)
src/encoder.la    encode(): preferred serialisation, iterative (**)
test/…            one file per area, mirroring src/ (see TESTS TO WRITE)
examples/round_trip.la   replaces examples/hello.la; .gitignore updated
```

(*) Session 3, before any code: the PRP first split this into `head.la` and `strings.la`. An `impl` must be in its type's file (spec §4) and only externs take `mut` parameters (§3), so everything that advances the shared cursor is a `mut self` method of one `Reader`. Same behaviour, one file.

(**) Session 3: named `decoder`/`encoder`, not `decode`/`encode`. A glob re-export of module `decode` would make `decode` both the module and its function in `lib` (E0314).

- **RFC 8949 sections covered:** §3 (the encoding: major types 0–7, heads, additional information 0–27 and 31, reserved 28–30), §3.1 (major types; strict UTF-8 for text), §3.2 (indefinite lengths and the break code, decode only), §3.3 (floats and simple values), §3.4 (tags as number + content, no semantics), §4.1 (preferred serialisation, for the encoder), §5.3.1 (basic validity: UTF-8, duplicate keys), §5.4 (resource limits), §5.6 (duplicate keys), Appendix A (examples), Appendix F (not-well-formed inputs).
- **Lugha spec sections relied on** (each checked in `lughac spec`, v17):
  - §4 Enums with `Self` payloads and arrays of themselves (`Array(Self[])`, `Tag(u64, Self)`); recursive enums need no `Box`.
  - §4 Structs with a `private` field, built only through an associated function (Visibility, §6), for `SimpleValue`.
  - §4 Struct update syntax `Limits { max_depth: 64, ..Limits.default() }` (v17, milestone 93).
  - §4 `Result`, `Option`, `?`, `match`, `if let`, `let … else`.
  - §4 Arrays: `push`, `pop`, `.len`, copy-on-write; `sort_by` (a stable bottom-up merge sort in the prelude, O(n log n); confirmed in `../lugha/src/prelude/core.la`).
  - §5 Integer methods: `checked_add`, shifts, `&`, `|`; `u64`/`i64` casts; `u64.MAX`, `i64.MAX`.
  - §5 `string_from_utf8` (strict: rejects overlong forms, surrogates, > U+10FFFF, truncated sequences).
  - §6 Package entry `src/lib.la` (`import cbor;`) and glob re-exports `public import value { * };`.
  - §9 `lughac test` with `@test`, `assert`, `assert_eq`, `--timeout`; programs in `test/` and `examples/` see the package's items.
  - §4 "A float's bits" (v17, milestone 108): `x.to_bits()` on `f64` gives a `u64`, and `f64.from_bits(b)` reads it back, NaN payloads and -0.0 kept. Only these two are used. Half and single are widened and narrowed on the integer bits, never with `as`, which would quiet a signalling NaN.
- **Open decisions that must be resolved first:** none. DECISION-002 is resolved (lughac m108) and DECISION-003 is resolved; both are followed. DECISION-001 (deferred) stays out of scope.

## IMPLEMENTATION REQUIREMENTS

### Must Do

**The `Value` type** (`src/value.la`), one representation per data item, so structural `==` is data-item equality except where IEEE floats differ (NaN ≠ NaN):

```
public enum Value {
    Unsigned(u64),           // major 0: 0 … 2^64-1
    Negative(u64),           // major 1: the value is -1 - n, so -1 … -2^64
    Bytes(u8[]),             // major 2
    Text(string),            // major 3, always valid UTF-8
    Array(Self[]),           // major 4
    Map(Entry[]),            // major 5, entries in the order received or given
    Tag(u64, Self),          // major 6: number and content, no semantics
    Bool(bool),              // simple 20, 21
    Null,                    // simple 22
    Undefined,               // simple 23
    Simple(SimpleValue),     // every other simple value: 0–19, 32–255
    Float(f64),              // half, single and double all decode to f64
}
public struct Entry { key: Value, value: Value }
public struct SimpleValue { private number: u8 }
```

- `SimpleValue.new(n: u8): Option<SimpleValue>` gives `None` for 20–31 (20–23 have their own variants; 24–31 are reserved, §3.3), so every `Value` can be encoded and `encode` never fails. `s.number(): u8` reads it.
- `Value.int(n: i64): Value` builds `Unsigned` or `Negative`, so callers needn't do the `-1 - n` arithmetic.

**Helpers** on `Value` (methods, `src/value.la`). None panics. Each returns `Option`:

| Method | Gives |
|---|---|
| `v.get(key: string): Option<Value>` | the value of the entry whose key is `Text(key)`, when `v` is a `Map`; linear in the entries (keys are unique after `decode`) |
| `v.as_text(): Option<string>` | a `Text` |
| `v.as_bytes(): Option<u8[]>` | a `Bytes` |
| `v.as_bool(): Option<bool>` | a `Bool` |
| `v.as_i64(): Option<i64>` | an `Unsigned` or `Negative` that fits `i64` |
| `v.as_u64(): Option<u64>` | an `Unsigned` |
| `v.as_f64(): Option<f64>` | a `Float` |
| `v.as_array(): Option<Value[]>` | an `Array` |
| `v.as_map(): Option<Entry[]>` | a `Map` |

They chain with `?` in a function returning `Option`: `v.get("props")?.get("text")?.as_text()`.

**Limits** (`src/limits.la`):

```
public struct Limits { max_depth: i64, max_items: i64, max_bytes: i64, max_string: i64 }
```

| Field | Default | Meaning |
|---|---|---|
| `max_depth` | 128 | most arrays, maps and tags open inside each other; a top-level scalar is depth 0, `[1]` is 1, `[[1]]` 2, `6(1)` 1 |
| `max_items` | 1,000,000 | most data items in total: every scalar, string, array, map and tag, and every key, value, element and tag content, the top item included |
| `max_bytes` | 67,108,864 (64 MiB) | most content bytes of all byte and text strings together (chunks summed) |
| `max_string` | 16,777,216 (16 MiB) | most content bytes of one string (its chunks concatenated) |

`Limits.default()` gives these. A negative field acts as 0. A `Limits` is a plain struct with public fields, so `Limits { max_depth: 64, ..Limits.default() }` works.

**`decode(bytes: u8[], limits: Limits): Result<Value, DecodeError>`** (`src/decode.la`):

- Exactly one data item that fills `bytes`. Anything after it is `Err(TrailingBytes)`. Empty input is `Err(Truncated)`.
- **Iterative, never recursive:** an explicit stack of frames (array, map, tag), one per open container, so its length is the depth. Opening a container when the stack already holds `max_depth` frames is `Err(TooDeep)`, checked before the frame is pushed. Indefinite strings don't nest (chunks must be definite), so they need no frame.
- Every head is read with bounds checks (`src/reader.la`): a missing argument byte is `Err(Truncated)`. Additional information 28–30 is `Malformed(ReservedInfo)`. 31 on major 0, 1 or 6 is `Malformed(IndefiniteNotAllowed)`. A two-byte simple value below 32 (`f8 00`–`f8 1f`) is `Malformed(ReservedSimple)`.
- **Claimed lengths are checked against the bytes left before anything is allocated or looped over** (see SECURITY). A string's claimed length or an array's claimed count beyond the bytes left is `Err(Truncated)`. Then the limits apply.
- Indefinite byte and text strings are decoded by concatenating their chunks. A chunk that isn't a definite string of the same major type is `Malformed(BadChunk)`. A missing break is `Err(Truncated)`.
- Indefinite arrays and maps decode. A break where an item is expected in a definite container, a tag's content, a map's value position, or at the top level is `Malformed(UnexpectedBreak)`.
- Text is strict UTF-8, checked with `string_from_utf8` per chunk (RFC 8949 §3.2.3: a code point can't span chunks). Invalid text is `Err(InvalidUtf8)`, never replaced.
- Duplicate keys in one map are `Err(DuplicateKey)`, found per DECISION-003 (`src/keys.la`). When a map closes, each key is encoded in preferred serialisation (`encode`, so `0x01` and `0x18 0x01` are the same key, and so are a definite and an indefinite `"a"`). The keys are merge-sorted by those bytes with `sort_by`, and neighbours compared.
- Floats: half (`f9`), single (`fa`) and double (`fb`) decode to an `f64` holding the same value, NaN payloads included, signalling NaNs too. Every width is converted on the integer bits (DECISION-002).
- Tags decode as `Tag(number, content)` for every number, with no checks on the content. Bignums (tags 2 and 3) stay `Tag(2, Bytes(…))`.

**`encode(value: Value): u8[]`** (`src/encode.la`), preferred serialisation (§4.1):

- Every head in its shortest form. Strings, arrays and maps always definite (DECISION-001), map entries in the order given (no sorting).
- Floats in the shortest of half, single and double that keeps the value exactly, -0.0 and infinities included. A NaN keeps its payload in the shortest width that holds it, so the canonical quiet NaN is `f9 7e 00`.
- `Simple(s)` is `e0 | n` below 24, else `f8 n`.
- Iterative: a work stack of `Value`s, children pushed in reverse. So encoding never recurses, whatever the depth.
- Never fails: every `Value` is encodable (see `SimpleValue`).

**The entry** `src/lib.la`: module doc, `public import` with `{ * }` of `value`, `limits`, `error`, `decoder` and `encoder`. Nothing else is public. `reader`, `keys` and `float` are package-level.

**The example** `examples/round_trip.la` replaces `examples/hello.la`. It builds a small nested value (a map holding text, a negative int, a float, an array and a nested map), encodes it, prints the bytes in hex, decodes them, reads fields back with the helpers, and shows that a decode error prints. `.gitignore`'s `/examples/hello` and `/hello` become `/examples/round_trip` and `/round_trip`.

### Must NOT Do

- **No streaming API** and **no encoding of indefinite lengths** (DECISION-001, deferred).
- **No tag semantics:** no dates, bignum arithmetic, decimal fractions or tag validity checks (§3.4 leaves them to the application).
- **No deterministic encoding (§4.2) and no key sorting:** the encoder keeps the order given.
- **No derive or serialisation of Lugha structs.**
- **No C, no `extern`, no `[link]`, no package dependencies, no `std` modules that touch the OS** (CLAUDE.md SECURITY RULE).
- **No recursion anywhere in the package**: decode, encode and duplicate detection all use explicit stacks.
- **No hash-based duplicate detection** (`Map`, `Set` or an unkeyed hash): it can be attacked (DECISION-003).
- **No `as f32` / `as f64` in float conversion, and no arithmetic float decoding** (DECISION-002): bits only.
- **No Rapidkoi-specific code or limits.** Rapidkoi sets its own `Limits`.

## ERROR HANDLING REQUIREMENTS

```
public enum DecodeError {
    Truncated,                 // the input ends inside an item, or claims more than is left
    Malformed(Malformation),   // not well-formed, and what is wrong
    TooDeep,                   // more nested containers than max_depth
    TooManyItems,              // more data items than max_items
    TooLarge,                  // one string over max_string, or all strings over max_bytes
    InvalidUtf8,               // a text string (or chunk) that isn't strict UTF-8
    DuplicateKey,              // two equal keys in one map
    TrailingBytes,             // bytes left after the one item
}
public enum Malformation {
    ReservedInfo,              // additional information 28, 29 or 30
    IndefiniteNotAllowed,      // additional information 31 on major type 0, 1 or 6
    ReservedSimple,            // a two-byte simple value below 32
    BadChunk,                  // an indefinite string's chunk that isn't a definite string of its type
    UnexpectedBreak,           // a break anywhere but closing an indefinite item (or in a map's value position)
}
```

- Order of checks on each item: the head is well-formed (Truncated, Malformed); then the claim fits the bytes left (Truncated); then the limits (TooDeep, TooManyItems, TooLarge); then the content (InvalidUtf8, DuplicateKey). TrailingBytes comes last, after the item is complete. So an input that is both not well-formed and over a limit reports the well-formedness problem first.
- Early claims: an array or map whose claimed count would take the item total past `max_items` is `TooManyItems` at its head, before any element is read. A string whose claimed length is over `max_string`, or would take the total past `max_bytes`, is `TooLarge` at its head.
- **Nothing in the package panics on any input.** `panic` only for a broken internal invariant, with a message starting `cbor: internal error:` (for example, a frame stack that is empty when an item completes).

## SECURITY CONSIDERATIONS

- **Claimed lengths:** checked in `src/reader.la` before any loop or allocation. A string needs `len ≤ left`. A definite array or map needs `count ≤ left` (each element, and each map key, takes at least one byte). Not `left / 2` for a map: `a1 ff` must read its break and be `Malformed(UnexpectedBreak)`, as Appendix F files it (session 3). All comparisons are in `u64` against `left as u64` (`left ≥ 0`), so nothing is added to a claim and nothing overflows. Nothing is preallocated from a claim: elements are pushed as they are decoded.
- **Depth:** the frame stack's length, checked before each push. No recursion in decode, encode or key encoding, so a large `max_depth` can't overflow the stack. (lughac's own generated `==` and printing of `Value` do recurse, but a decoded value's depth is at most `max_depth`.)
- **Items and bytes:** counters checked with `checked_add` against the limits, at each head and each early claim.
- **Overflow in offsets:** the position only advances by amounts already checked to fit in the bytes left, so `pos + n ≤ bytes.len` always holds and never overflows `i64`.
- **Time:** every byte of input is read once. String bytes are copied once (plus once per chunk joined). Duplicate detection costs O(K log n) per map, where K is the size of its keys' encodings (DECISION-003). Keys nested inside keys are encoded once per enclosing map, so the worst case is bounded by `max_depth` × input size: no input reaches quadratic time.
- **Memory:** at most `max_items` values and `max_bytes` string bytes, plus the input.
- **UTF-8:** strict, via `string_from_utf8`. Never replaced.

## TESTS TO WRITE

Test first, each file under 400 lines, in `test/` mirroring `src/`:

- [x] `test/decode_appendix_a.la`: every RFC 8949 Appendix A example decodes to its value. Covers the integers 0 … 2^64-1 and -1 … -2^64; the bignum tags 2 and 3 as `Tag`; every float example (half, single, double, ±0, subnormal 5.960464477539063e-8, 65504.0, 100000.0, 3.4028234663852886e+38, 1.0e+300, -4.1, ±Infinity and NaN in all three widths); false, true, null, undefined, simple(16), simple(255); the tags 0, 1, 23, 24 and 32; the byte and text strings ("ü", "水", "𐅑"); arrays, maps and nested ones; and every indefinite-length example.
- [x] `test/encode.la`: every Appendix A example whose encoding is preferred encodes to exactly those bytes. That's all of them except the indefinite-length ones and the non-shortest Infinity/NaN encodings (`fa…`, `fb…`), which instead encode to their preferred forms (`f9 7c 00`, `f9 7e 00`, `f9 fc 00`). Also NaN payloads preserved through decode → encode.
- [x] `test/decode_appendix_f.la`: every Appendix F not-well-formed example, with its expected error:
  - end of input in a head: `18`, `19`, `1a`, `1b`, `19 01`, `1a 01 02`, `1b 01 02 03 04 05 06 07`, `38`, `58`, `78`, `98`, `9a 01 ff 00`, `b8`, `d8`, `f8`, `f9 00`, `fa 00 00`, `fb 00 00 00` → Truncated
  - definite strings with short data: `41`, `61`, `5a ff ff ff ff 00`, `5b ff ff ff ff ff ff ff ff 01 02 03`, `7a ff ff ff ff 00`, `7b 7f ff ff ff ff ff ff ff 01 02 03` → Truncated
  - definite arrays and maps without enough items: `81`, `81 81 81 81 81 81 81 81 81`, `82 00`, `a1`, `a2 01 02`, `a1 00`, `a2 00 00 00` → Truncated
  - a tag without content: `c0` → Truncated
  - indefinite strings without a break: `5f 41 00`, `7f 61 00` → Truncated
  - indefinite arrays and maps without a break: `9f`, `9f 01 02`, `bf`, `bf 01 02 01 02`, `81 9f`, `9f 80 00`, `9f 9f 9f 9f 9f ff ff ff ff`, `9f 81 9f 81 9f 9f ff ff ff` → Truncated
  - reserved additional information: `1c`–`1e`, `3c`–`3e`, `5c`–`5e`, `7c`–`7e`, `9c`–`9e`, `bc`–`be`, `dc`–`de`, `fc`–`fe` → Malformed(ReservedInfo)
  - reserved two-byte simple values: `f8 00`, `f8 01`, `f8 18`, `f8 1f` → Malformed(ReservedSimple)
  - chunks of the wrong type: `5f 00 ff`, `5f 21 ff`, `5f 61 00 ff`, `5f 80 ff`, `5f a0 ff`, `5f c0 00 ff`, `5f e0 ff`, `7f 41 00 ff` → Malformed(BadChunk)
  - indefinite chunks: `5f 5f 41 00 ff ff`, `7f 7f 61 00 ff ff` → Malformed(BadChunk)
  - a lone break: `ff` → Malformed(UnexpectedBreak)
  - a break in a definite array, map or tag: `81 ff`, `82 00 ff`, `a1 ff`, `a1 ff 00`, `a1 00 ff`, `a2 00 00 ff`, `9f 81 ff`, `9f 82 9f 81 9f 9f ff ff ff ff` → Malformed(UnexpectedBreak)
  - a break in a map's value position: `bf 00 ff`, `bf 00 00 00 ff` → Malformed(UnexpectedBreak)
  - indefinite length on major 0, 1, 6: `1f`, `3f`, `df` → Malformed(IndefiniteNotAllowed)
- [x] `test/decode_limits.la`, one test per limit at its edge (exactly at the limit passes, one over fails):
  - depth: arrays, maps, tags and a mix → TooDeep; `max_depth` 0 accepts a scalar and rejects `80`
  - items: many small items → TooManyItems; an early claim (`9a 00 0f 42 41` …) → TooManyItems before reading elements
  - one string over `max_string`, definite and as indefinite chunks → TooLarge
  - many strings over `max_bytes` → TooLarge
  - a deep input (100,000 nested `81`) with a huge `max_depth` decodes without a stack overflow (proves no recursion)
- [x] `test/decode_hostile.la`:
  - huge claimed lengths in tiny inputs: `5b ff…ff`, `7b ff…ff`, `9b ff…ff`, `bb ff…ff`, and the same with 32-bit heads → Truncated, at once
  - a map claiming 2^63 entries → Truncated
  - invalid UTF-8 → InvalidUtf8: overlong (`62 c0 80`), surrogate (`63 ed a0 80`), above U+10FFFF (`64 f4 90 80 80`), truncated sequence (`61 c3`), and a code point split across two chunks (`7f 61 c3 61 bc ff`)
  - duplicate keys → DuplicateKey: plain (`a2 01 00 01 00`), non-preferred (`a2 01 00 18 01 00`), definite vs indefinite text, equal nested keys, the duplicate as the last of 10,000 keys
  - trailing bytes → TrailingBytes (`00 00`, `80 00`)
  - empty input → Truncated
- [x] `test/keys.la`: a 10,000-key map decodes fast (the suite runs it with `--timeout 2`). So do the same keys in reverse order and 10,000 keys sharing a 200-byte prefix (worst case for comparisons). A duplicate as the last key is found. (Session 3: the FNV-collision case was dropped. This package doesn't hash, so it would only test lughac's `Map`. Hashing is ruled out by DECISION-003 and Must NOT Do instead.)
- [x] `test/float.la`: half, single and double ↔ f64 at the edges: subnormals, the largest finite, ±0, ±inf, quiet and signalling NaN payloads. Shortest-width choice: 1.5 → half, 100000.0 → single, 1.1 → double, 65504.0 → half, 65505.0 → single.
- [x] `test/value.la`: every helper, happy path and `None` path (`as_i64` on `Unsigned(u64.MAX)` and on `Negative(i64.MAX as u64 + 1)` → None, `get` on a non-map → None, a missing key → None). `Value.int` at `i64.MIN`, -1, 0 and `i64.MAX`. `SimpleValue.new` for 0, 19, 20, 31, 32 and 255.
- [x] `test/round_trip.la`: `decode(encode(v)) == Ok(v)` for a set of values covering every variant, nested; and `encode(decode(b))` gives `b` back for every preferred Appendix A encoding.
- [x] Error paths: every `DecodeError` and `Malformation` variant appears in at least one test above.

## ROLLBACK PLAN
- Branch to return to: `main` at `67c0128` (session 1).
- State the codebase should be in: the `lughac init` placeholders and the context system, unchanged. The PRP branch is deleted unmerged.

## ACCEPTANCE CRITERIA
- [x] Every Appendix A example decodes to its value, and every preferred one encodes to its exact bytes.
- [x] Every Appendix F example gives its expected `Err`.
- [x] Each limit is enforced, exactly at its edge.
- [x] No input panics, hangs or allocates past the limits; huge claims in tiny inputs fail at once.
- [x] A 10,000-key map (plain, reversed, common-prefix) decodes within the 2-second test timeout.
- [x] 100,000 nested arrays decode with a raised `max_depth` without a stack overflow.
- [x] Round trips hold for every variant.
- [x] `examples/round_trip.la` builds and runs; `examples/hello.la` is gone.
- [x] `lughac test test/` passes; every example builds
- [x] `lughac fmt --check src examples test` is clean
- [x] Every `public` item has a `/** … */` doc comment
- [x] No file over 400 lines
- [x] CHANGELOG.md updated

## VALIDATION
- `lughac fmt --check src examples test`
- `lughac test test/`
- `lughac test test/keys.la --timeout 2`
- `for f in examples/*.la; do lughac build "$f" || exit 1; done`
- Check each command's real exit status (no `| tail`).
