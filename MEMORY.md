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

## CURRENT PROJECT STATE

### Fully Working
- `lughac init --lib` skeleton; context system (session 1).

### Not Started
- PRP-001: the decoder and encoder.

---

## NEXT SESSION START POINT

See CONTEXT.md.
