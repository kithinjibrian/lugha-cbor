# CONTEXT.md — lugha-cbor

Session handoff file. Updated at the end of every session.
Read at the start of the next session alongside CLAUDE.md, MEMORY.md, and DECISIONS.md.

---

## SESSION 1 — 2026-10-17 — Set up the repository — closed

Branch: main

### WHAT WAS DONE

Created the package with `lughac init --lib --name cbor` and set up the context system from `docs/setup.md`, from a Rapidkoi session on the owner's go (rapidkoi session 4).

### FILES CREATED OR MODIFIED

```
lugha.toml, src/lib.la, examples/hello.la, .gitignore — lughac init (+ /examples/hello ignored)
CLAUDE.md MEMORY.md CONTEXT.md DECISIONS.md CHANGELOG.md .llmignore .claude/settings.json README.md LICENSE
PRPs/TEMPLATE.md PRPs/DISCOVERY.md docs/CODE_STYLE.md docs/setup.md
test/, docs/source/…, docs/specs …, reports/ — empty folders
```

### TESTS WRITTEN

- None.

### DECISIONS MADE

- Package name `cbor`; pure Lugha; GPL-3.0-only (MEMORY.md 1–3).

### PENDING DECISIONS OPENED

- DECISION-001 streaming / indefinite-length encoding (deferred).

### STILL OPEN AT CLOSE

- PRP-001 not written. No remote.

---

## SESSION 2 — 2026-10-17 — PRP-001: decode and encode — closed

Branch: prp-001-decode-encode (not merged: nothing to build yet)

### WHAT WAS DONE

Wrote PRP-001 from the owner's brief (Value, decode with Limits, preferred encode, reading helpers, Appendix A/F tests). Checking the spec turned up two gaps, each asked one at a time:

1. lughac has no float bit casts (no `to_bits`/`from_bits`; `as` converts by value). The owner chose to **block PRP-001** until lughac has them, rather than decode floats by arithmetic and lose NaN payloads. Recorded as DECISION-002 (open).
2. Duplicate-key detection: Lugha's `Map` hash is unkeyed FNV-1a and the package can't get a secret seed, so the owner chose a sort-based check, O(n log n) and never quadratic. Recorded as DECISION-003 (resolved) and MEMORY.md decision 4.

No code was written.

### FILES CREATED OR MODIFIED

```
PRPs/prp-001-decode_encode.md   — new: PRP-001, waiting for approval, blocked by DECISION-002
DECISIONS.md                    — DECISION-002 (open), DECISION-003 (resolved)
MEMORY.md                       — decision 4; PRP-001 state
CONTEXT.md                      — this entry
```

### TESTS WRITTEN

- None (no code before approval and before DECISION-002 is resolved).

### DECISIONS MADE

- DECISION-003: duplicate keys found by merge-sorting preferred encodings.
- In the PRP, for approval: iterative decode and encode (no recursion at all); `SimpleValue` with a private field so `encode` can't fail; default limits depth 128, 1,000,000 items, 64 MiB of string bytes, 16 MiB per string.

### PENDING DECISIONS OPENED

- DECISION-002: float bit casts in lughac, needed in `../lugha` (fixed there, never from here).

### STILL OPEN AT CLOSE

- PRP-001 needs the owner's approval, and DECISION-002 resolved, before any code.
- No remote.

---

## SESSION 3 — 2026-10-17 — PRP-001: resolve DECISION-002 and build — closed

Branch: prp-001-decode-encode (merged into main, fast-forward)

### WHAT WAS DONE

lughac m108 (Lugha PR #65) added float bit casts. A probe confirmed a signalling NaN's bits survive. Resolved DECISION-002, updated PRP-001, got the owner's approval, and built it test-first: the tests were committed red (6ff8e00), then the code.

Deviations from the PRP, each recorded in it:
- `head.la` + `strings.la` became one `reader.la`: an `impl` must sit in its type's file and only externs take `mut` parameters.
- `decode.la`/`encode.la` are named `decoder.la`/`encoder.la`: a glob re-export would make `decode` both a module and a function in `lib` (E0314).
- Map claims are checked as count ≤ bytes left (not left / 2), so `a1 ff` reads its break and is `Malformed(UnexpectedBreak)`, as RFC 8949 Appendix F files it.
- The FNV-collision key test was dropped: this package doesn't hash, so it would only test lughac's `Map`. A reversed-order test took its place.

### FILES CREATED OR MODIFIED

```
src/lib.la (entry, re-exports only), value.la, limits.la, error.la, reader.la, decoder.la, keys.la, float.la, encoder.la
test/support.la + decode_appendix_a, decode_appendix_f, decode_limits, decode_hostile, keys, float, value, encode, round_trip
examples/round_trip.la (replaces hello.la); .gitignore, .llmignore
PRPs/prp-001-decode_encode.md, DECISIONS.md, MEMORY.md (decision 5, state), CLAUDE.md (architecture, layout), CHANGELOG.md
```

### TESTS WRITTEN

- 86 tests, all passing (`lughac test test/`, exit 0): every Appendix A example decoded and, where preferred, encoded; every Appendix F input with its error; each limit at its edge; huge claims, invalid UTF-8 (including a code point split across chunks), duplicate keys in other encodings, trailing bytes; 10,000-key maps (the slowest takes 361 ms under `--timeout 2`); 100,000 nested arrays decoded and re-encoded with no stack (114 ms); every half round-trips; NaN payloads kept.
- `lughac fmt --check src examples test` clean; `examples/round_trip.la` builds and runs. Largest file 237 lines.

### DECISIONS MADE

- DECISION-002 resolved: float widths convert on integer bits (MEMORY.md 5).

### PENDING DECISIONS OPENED

- None. DECISION-001 (streaming, indefinite-length encoding) stays deferred.

### STILL OPEN AT CLOSE

- No remote: nothing pushed or tagged (ask the owner first).
- Rapidkoi can now depend on `cbor` (in `../rapidkoi`, under its own rules).

---

## NEXT SESSION START POINT

Open session 4 first. PRP-001 is merged and complete. Possible next steps, for the owner to choose: create the GitHub repository and push (ask first), tag v0.1.0, or a new PRP (e.g. DECISION-001's streaming/indefinite encoding, if a user needs it). Rapidkoi's step 1 can use `cbor` now.
