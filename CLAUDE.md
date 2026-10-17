# CLAUDE.md — lugha-cbor

Behavioral instructions for AI coding assistants. Each rule exists to prevent a specific mistake.

lugha-cbor is a pure-Lugha package (`cbor`, `import cbor;`) that encodes and decodes CBOR (RFC 8949). It is general-purpose: its first user is Rapidkoi (`../rapidkoi`), which decodes data from untrusted WebAssembly apps, but nothing here may know about Rapidkoi. It exists to grow Lugha's ecosystem.

---

## SESSION HANDOFF RULE — NON-NEGOTIABLE

Every session in `CONTEXT.md` has a **name**, a **state** (`open` / `closed`) and a **branch**:

    ## SESSION {n} — {YYYY-MM-DD} — {Name} — {state}
    Branch: {branch-name}

- **Step 1 of every session, no exceptions:** append a new `open` session entry to `CONTEXT.md` (above NEXT SESSION START POINT) and commit it before reading anything else or writing code.
- Close it only after the handoff is written and committed (and pushed, once a remote exists).
- Never leave a session `open` at the end of a turn. Never start one without closing the last.
- Rewrite NEXT SESSION START POINT at the end of every session. Never delete sessions.

---

## PRP RULE — NON-NEGOTIABLE

Never write code for a new feature without an approved PRP in `/PRPs`.

1. No PRP: run `PRPs/DISCOVERY.md`, one question at a time, each with a recommended answer. When the owner hands over a brief, write the PRP from it and ask only about real gaps.
2. Write `/PRPs/prp-{NNN}-{feature_name}.md` from `PRPs/TEMPLATE.md` (next unused number, snake_case name).
3. Build only after explicit approval.

---

## SCOPE RULE — NON-NEGOTIABLE

One PRP at a time. If the scope grows mid-implementation: stop, document it, update the PRP, get approval.

**A missing or broken lughac feature is never worked around silently.** Record it in `DECISIONS.md` and report it. It is fixed in `../lugha` under that repo's rules.

---

## HOSTILE INPUT RULE — NON-NEGOTIABLE

`decode` is fed bytes from untrusted programs. Any input must give a value or a typed `Err`, never a panic, a hang, or a huge allocation.

- **Check every claimed length against the bytes actually left before allocating or looping.** A 9-byte header can claim 2^64 items.
- **Depth is limited without unbounded recursion.** Nesting deeper than the limit is `Err(TooDeep)`, checked before descending.
- **Total items and total bytes are limited** (`Limits`), so many small items can't add up to a huge value.
- Arithmetic on lengths and offsets can't overflow: check before adding.
- Text strings must be valid UTF-8 (RFC 8949 §3.1); invalid is `Err`, never replaced.
- Every malformed-input case in RFC 8949 Appendix F gets a test.

---

## TESTING RULE — NON-NEGOTIABLE

Write the test before the implementation.

- `@test` files live in `test/`, mirroring `src/`.
- Every example in RFC 8949 Appendix A is a test, both ways (decode, and encode where the encoding is canonical).
- Every public function gets a happy-path test and one per error variant.
- Run only the affected test files while working; the full suite before a merge.

---

## SECURITY RULE — NON-NEGOTIABLE

- No `extern`, no `[link]` and no package dependencies: this package is pure Lugha. Adding one needs a PRP that says why.
- No file, process, network or environment access.
- Never hardcode credentials or secrets.

If unsure whether something has a security implication, stop and ask.

---

## OUTWARD-FACING ACTIONS — NON-NEGOTIABLE

Ask before anything that leaves this machine: creating the GitHub repository, adding a remote, `git push`, tagging. Before any push or tag, check the test run's real exit status (`| tail` hides failures).

---

## CODE DOCUMENTATION RULE — NON-NEGOTIABLE

Read `docs/CODE_STYLE.md` before writing any function, type, or module. Every `public` item has a `/** … */` doc comment.

---

## PENDING DECISIONS RULE — NON-NEGOTIABLE

Check `DECISIONS.md` before code that depends on an architectural question. `open` blocks it; `resolved` is followed, not re-litigated; a new question is added as `open` and you stop.

---

## FILE SIZE RULE

No file exceeds 400 lines. At the limit: propose a specific split, wait for approval, then split.

---

## PROTECTED FILES — NON-NEGOTIABLE

Never read, modify, or delete:

- `.env` and `.env.*`
- `lugha.lock`: written only by lughac
- `LICENSE`: changed only by the human
- Built executables (the paths in `.gitignore`)
- `../lugha/`, `../rapidkoi/` and every other sibling repository: read for reference, never write

Keep this list, `.llmignore` and `.claude/settings.json` in sync.

---

## COMMANDS

```bash
lughac check examples/hello.la
lughac build examples/hello.la
lughac test test/                   # every @test
lughac fmt --check src examples test
lughac spec                         # the Lugha reference: check it before assuming a feature exists
```

A task is done only when `fmt --check` is clean, `lughac test test/` passes and every example builds.

> Needs: `lughac` (the user installs it; never do it yourself). Nothing else.

---

## STACK

- **Lugha** via `lughac` 0.1.0. Use only what `lughac spec` documents. Pure Lugha: no C.
- **RFC 8949** (CBOR) is the specification. Cite its sections in comments when applying a rule.
- License: **GPL-3.0-only**.

---

## ARCHITECTURE RULES

To be settled by PRP-001 (the `Value` type, `Limits`, and the decode/encode API).

---

## ERROR HANDLING — NON-NEGOTIABLE

Expected failures return `Result<T, E>` with typed error enums. Every malformed, truncated, oversized or too-deep input is an expected failure. `panic` only for a broken invariant inside this package.

---

## FILE ORGANIZATION

```
.
├── CLAUDE.md MEMORY.md CONTEXT.md DECISIONS.md CHANGELOG.md .llmignore
├── .claude/settings.json — deny rules mirroring .llmignore
├── LICENSE               — GPL-3.0-only
├── lugha.toml            — package `cbor`, source `src`
├── src/lib.la            — the entry (`import cbor;`): the lughac init placeholder today
├── examples/hello.la     — placeholder example
├── test/                 — @test files mirroring src/
├── PRPs/                 — TEMPLATE.md, DISCOVERY.md, prp-{NNN}-….md
├── docs/                 — CODE_STYLE.md, setup.md, source/, specs/ decisions/ incidents/ status/
└── reports/              — EOD reports
```

---

## ANTI-PATTERNS

1. **Never allocate from a length you haven't checked against the bytes left.**
2. **Never recurse without checking depth first.**
3. **Never use a Lugha feature without checking `lughac spec`.**
4. **Never add Rapidkoi-specific code.** It belongs in `../rapidkoi`.

---

## KNOWN ISSUES — DO NOT FIX

- `src/lib.la` and `examples/hello.la` are the `lughac init` placeholders, kept until PRP-001 replaces them.
