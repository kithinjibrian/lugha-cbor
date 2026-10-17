# DECISIONS.md — lugha-cbor

Tracks architectural and design questions that are open, deferred, or resolved.

Rules:
- Every open decision blocks implementation of the code it affects.
- The AI must not implement anything that depends on an open decision.
- When a decision is resolved, move it to RESOLVED and copy the outcome to MEMORY.md.

---

## OPEN — Requires human input before implementation

(none)

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

(none yet; the license is MEMORY.md decision 3)
