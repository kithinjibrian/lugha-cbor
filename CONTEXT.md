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

## NEXT SESSION START POINT

Open session 3 first, on `prp-001-decode-encode`. Check whether the installed lughac documents float bit casts (`lughac spec | grep -i bits`). If yes: resolve DECISION-002, update PRP-001's "Lugha spec sections relied on" with their real names, get the owner's approval of PRP-001 if not yet given, then build it test-first. If no: PRP-001 stays blocked; ask the owner what to do next.
