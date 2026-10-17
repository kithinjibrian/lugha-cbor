# DECISIONS.md — lugha-cbor

Tracks architectural and design questions that are open, deferred, or resolved.

Rules:
- Every open decision blocks implementation of the code it affects.
- The AI must not implement anything that depends on an open decision.
- When a decision is resolved, move it to RESOLVED and copy the outcome to MEMORY.md.

---

## OPEN — Requires human input before implementation

### DECISION-002 — Float bit casts (lughac)

**Status:** open, blocks PRP-001
**Raised:** 2026-10-17 — Session 2
**Blocks:** all of PRP-001 (the owner's choice, session 2)

**Question:** How does the package turn CBOR's half, single and double float bits into an `f64`, and back?

**Finding:** lughac 0.1.0 (spec v17) has no way to reinterpret bits. `as` converts numbers by value (spec §4, Casts), and neither the spec nor `../lugha/src` has `to_bits`/`from_bits` or anything like them. Exact arithmetic could decode and encode every float value, including infinities and -0.0, but not NaN payloads: every NaN would come out as the same NaN and encode as `f97e00`.

**Owner's decision (session 2):** don't work around it. PRP-001 waits until lughac has float bit casts, so NaN payloads are kept.

**Needed in `../lugha`** (fixed there under that repo's rules, never from here): reinterpreting casts both ways, for example `f64.from_bits(u64): f64`, `x.to_bits(): u64` on `f64`, and the same for `f32` with `u32`. Half precision stays in this package (it has no Lugha type).

**Resolves when:** a lughac release documents float bit casts in `lughac spec`. PRP-001 is then updated to name them and built.

---

## DEFERRED — Acknowledged, not yet needed

### DECISION-001 — Streaming and indefinite-length encoding

**Status:** deferred
**Raised:** 2026-10-17 — Session 1
**Revisit when:** A user needs to decode a stream, or emit items whose length isn't known up front.

**Question:** Offer a streaming decoder, and an encoder for indefinite-length items?

**Notes:** Decoding indefinite-length items is part of RFC 8949 and in scope for PRP-001; this is only about a streaming API and encoding them.

---

## RESOLVED

### DECISION-003 — Duplicate map keys detected by sorting, not hashing

**Status:** resolved
**Raised and resolved:** 2026-10-17 — Session 2 (owner)

**Question:** The brief asks for duplicate-key detection that an attacker can't push into quadratic time. Hash, or sort?

**Outcome:** Sort. Each map's keys are encoded in preferred serialisation, merge-sorted (`sort_by`, a bottom-up merge sort in the prelude), and neighbours compared: O(n log n) comparisons, guaranteed.

**Why:** Lugha's `Map` hashes with FNV-1a and no seed, and the spec says it "isn't hardened against keys chosen to collide". This package can't draw a secret seed either, because OS access is banned. An unkeyed hash table can be pushed into quadratic time; a merge sort can't.

**Rules out:** `Map`/`Set` or any unkeyed hash for duplicate detection.

---

(the license is MEMORY.md decision 3)
