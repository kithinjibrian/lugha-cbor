## FEATURE: [one sentence]

## OBJECTIVE
[2–3 sentences: what a Lugha program can do when this is done, with a short `.la` example]

## CONTEXT

- Starting state / ending state: [files]
- RFC 8949 sections covered: [list]
- Lugha spec sections relied on: [confirm each feature exists]
- Open decisions that must be resolved first: [DECISIONS.md entries]

## IMPLEMENTATION REQUIREMENTS

### Must Do
- [specific requirement]

### Must NOT Do
- [explicit exclusion, and why]

## ERROR HANDLING REQUIREMENTS

- [Error enum variants, and which inputs produce each]
- [Conditions that panic (broken invariants only), and the message]

## SECURITY CONSIDERATIONS

- [Every claimed length checked before allocation: where]
- [Depth, item and byte limits: defaults and how they're enforced]
- [Overflow in length arithmetic: how it's prevented]

## TESTS TO WRITE

- [ ] RFC 8949 Appendix A examples: [which]
- [ ] Appendix F malformed inputs: [which]
- [ ] Limits: [too deep, too many items, too many bytes, huge claimed lengths]
- [ ] Error path: [each variant]

## ROLLBACK PLAN
- Branch to return to: [branch]
- State the codebase should be in: [describe]

## ACCEPTANCE CRITERIA
- [ ] [testable criterion]
- [ ] `lughac test test/` passes; every example builds
- [ ] `lughac fmt --check src examples test` is clean
- [ ] Every `public` item has a `/** … */` doc comment
- [ ] No file over 400 lines
- [ ] CHANGELOG.md updated

## VALIDATION
- `lughac fmt --check src examples test`
- `lughac test test/`
- `for f in examples/*.la; do lughac build "$f" || exit 1; done`
