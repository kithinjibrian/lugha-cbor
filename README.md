# lugha-cbor

[CBOR](https://www.rfc-editor.org/rfc/rfc8949) for [Lugha](https://github.com/kithinjibrian/lugha), in pure Lugha: a decoder that is safe on untrusted input, and an encoder in preferred serialisation.

```
import cbor { Value, Entry, Limits, decode, encode };

/** The text under "props" → "text", if there is one. */
fun props_text(node: Value): Option<string> = node.get("props")?.get("text")?.as_text();

fun main() {
    let node = Value.Map([
        Entry { key: Value.Text("kind"), value: Value.Text("text") },
        Entry { key: Value.Text("props"), value: Value.Map([
            Entry { key: Value.Text("text"), value: Value.Text("Count: 3") },
        ]) },
    ]);
    let bytes = encode(node);
    match decode(bytes, Limits.default()) {
        Ok(v) => println(props_text(v)), // Some("Count: 3")
        Err(e) => println("bad data: {e}"),
    }
}
```

## Requirements

- `lughac` with milestones 99 and 100 (a package's `lib.la`, glob re-exports) and 108 (a float's bits, `x.to_bits()` and `f64.from_bits(b)`).
- Nothing else: no C, no `[link]`, no dependencies, no file, process or network access.

## Using it

List it in your program's `lugha.toml`:

```toml
[packages]
cbor = { git = "https://github.com/kithinjibrian/lugha-cbor", version = "0.1" }
```

Inside this repository:

```sh
lughac run examples/round_trip.la
lughac test test/
```

## Untrusted input

`decode(bytes, limits)` gives a `Value` or a typed `DecodeError` for any input, never a panic, a hang or a huge allocation:

- Every claimed length and count is checked against the bytes left before anything is copied or looped over, so a 9-byte head claiming 2^64 items fails at once.
- Nothing recurses: nesting is an explicit stack, checked against `max_depth` before every push.
- `Limits` caps depth, total items, total string bytes and one string's length. The defaults are 128, 1,000,000, 64 MiB and 16 MiB. Change what you need: `Limits { max_depth: 64, ..Limits.default() }`.
- Text must be strict UTF-8, chunk by chunk. Duplicate map keys are an error, found by merge-sorting the keys' encodings, O(n log n) with no hash to attack.
- Every example in RFC 8949 Appendix A and every not-well-formed input in Appendix F is a test.

## What's covered (0.1.0)

| Item | What it is |
| --- | --- |
| `Value` | the whole data model: `Unsigned(u64)`, `Negative(u64)` (the value is -1 - n, down to -2^64), `Bytes`, `Text`, `Array`, `Map` (entries kept in order), `Tag(u64, Value)` (no tag semantics), `Bool`, `Null`, `Undefined`, `Simple(SimpleValue)`, `Float(f64)` (half, single and double, NaN payloads kept) |
| `Entry` | a map's `key` and `value` |
| `SimpleValue` | simple values 0–19 and 32–255: `SimpleValue.new(n)`, `s.number()` |
| `Value` helpers | `Value.int(i64)`; `v.get(key)` for a text key in a map; `as_text`, `as_bytes`, `as_bool`, `as_i64`, `as_u64`, `as_f64`, `as_array`, `as_map`, each an `Option` |
| `decode(bytes, limits)` | exactly one data item, definite or indefinite lengths, into a `Result<Value, DecodeError>` |
| `encode(value)` | preferred serialisation (RFC 8949 §4.1): shortest heads, shortest exact floats, definite lengths, map entries in the order given |
| `Limits` | `max_depth`, `max_items`, `max_bytes`, `max_string`; `Limits.default()` |
| `DecodeError` | `Truncated`, `Malformed(Malformation)`, `TooDeep`, `TooManyItems`, `TooLarge`, `InvalidUtf8`, `DuplicateKey`, `TrailingBytes` |
| `Malformation` | `ReservedInfo`, `IndefiniteNotAllowed`, `ReservedSimple`, `BadChunk`, `UnexpectedBreak` |

Not covered yet: a streaming API, encoding indefinite lengths, tag semantics (dates, bignums), deterministic key sorting (§4.2).

## License

GPL-3.0-only, like Lugha.
