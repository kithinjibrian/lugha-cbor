# MEMORY.md — lugha-cbor

Records resolved architectural decisions and current project state.
Read this at the start of every session before writing any code.

---

## ARCHITECTURAL DECISIONS

### 1. A general-purpose, pure-Lugha package

**Decision:** CBOR for Lugha lives in its own repository and package (`cbor`), with no C and no dependencies.

**Why:** The owner's direction to grow Lugha's ecosystem (`../rapidkoi/docs/source/meetings/2026-10-17-ui-contract.md`). Pure Lugha means nothing unchecked between untrusted bytes and the program.

**Rules out:** Rapidkoi-specific code; C shims; dependencies.

---

### 2. Safe on hostile input

**Decision:** Any input to `decode` gives a value or a typed `Err`: lengths checked before allocation, limited depth, items and bytes, strict UTF-8.

**Why:** The first user decodes data from untrusted programs; length-claim and deep-nesting attacks are classic against decoders.

**Rules out:** Panics, unbounded recursion, trusting a claimed length.

---

### 3. GPL-3.0-only

**Decision:** Licensed GPL-3.0-only, like Lugha, lugha-gtk, kivuli and lugha-wasmtime.

**Why:** Consistency with the owner's ecosystem (as in lugha-wasmtime DECISION-003).

**Rules out:** Nothing for now; the owner can relicense while sole author.

---

### 4. Duplicate map keys found by sorting

**Decision:** Duplicate keys are found by merge-sorting each map's keys by their preferred encoding: O(n log n), never quadratic (DECISION-003).

**Why:** Lugha's `Map` hash is unkeyed and the package has no source of a secret seed, so a hash table can be attacked.

**Rules out:** Hash-based duplicate detection.

---

### 5. Float widths converted on bits

**Decision:** Half, single and double convert to and from `f64` by integer bit manipulation, with `f64.to_bits`/`f64.from_bits` (lughac m108) at the edges (DECISION-002).

**Why:** NaN payloads must survive; a hardware `f32`↔`f64` conversion quiets signalling NaNs.

**Rules out:** `as f32`/`as f64` in float encoding or decoding; arithmetic float decoding.

---

## CURRENT PROJECT STATE

### Fully Working
- `lughac init --lib` skeleton; context system (session 1).
- PRP-001 (session 3): `Value`, `decode` with `Limits`, `encode`, the reading helpers; 86 tests.
- Released v0.1.0 (session 5): github.com/kithinjibrian/lugha-cbor, tag `v0.1.0`. Releases follow lugha-gtk: a `lugha-cbor X.Y.Z: …` commit bumping `lugha.toml` and the README, plus a lightweight tag.


---

## NEXT SESSION START POINT

See CONTEXT.md.
