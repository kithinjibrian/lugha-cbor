# CHANGELOG — lugha-cbor

Follows [Keep a Changelog](https://keepachangelog.com) format.
Updated at the end of every session when something is completed and merged.

---

## [Unreleased]

## [0.1.0] — 2026-10-17

### Added
- Package skeleton (`cbor`).
- PRP-001: `Value` (the whole RFC 8949 data model), `decode(bytes, Limits)` (safe on hostile input: claims checked against the bytes left, limits on depth, items and string bytes, strict UTF-8, duplicate keys rejected in O(n log n), no recursion), `encode(value)` in preferred serialisation (NaN payloads kept), and the reading helpers `get` and `as_*`.
- `examples/round_trip.la`, replacing `examples/hello.la`.
- 86 tests: every RFC 8949 Appendix A example (both ways) and Appendix F input, each limit at its edge, hostile inputs, 10,000-key maps, floats, round trips.
