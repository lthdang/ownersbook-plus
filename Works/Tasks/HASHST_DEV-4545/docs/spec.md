## Problem Statement

The **Masked email** shown on screen (16 call sites in app-vuejs) is the only thing standing between a customer's address and plain display. Its behavior is not covered by any automated test, so a change to the masking could silently expose personal information or corrupt the display. Jira ticket: HASHST_DEV-4545 (ticket E of Phase 1, A–E).

## Solution

Add a characterization unit test suite for `regEmail` that records its current behavior. No production code changes. Suspicious behavior is recorded with an explanatory comment and raised separately, not fixed here.

## User Stories

1. As a developer, I want `regEmail` covered by unit tests, so that a refactor of the masking cannot silently change what customers see.
2. As a developer, I want the 3-vs-4 character **Local part** boundary pinned, so that an off-by-one change is caught immediately.
3. As a developer, I want a **Local part** of 4 or more characters to show the first 2 and last 1 characters with `*` in between, so that the documented display format `xx*****X@domain` is guaranteed.
4. As a reviewer, I want **Unmasked short address** cases (3, 2 and 1 characters) asserted as unchanged, with a comment citing the confirmed-intended decision (ADR 0001), so that nobody mistakes them for a leak.
5. As a developer, I want the domain part asserted never to be masked, so that the masking scope is explicit.
6. As a developer, I want addresses containing `.` or `+` in the **Local part** covered, so that realistic addresses are represented.
7. As a developer, I want strings without `@`, with a leading `@`, or with an empty domain asserted as returned unchanged, so that malformed input behavior is documented.
8. As a developer, I want the multiple-`@` behavior pinned with a "possibly unintended, not fixed" comment, so that it can be triaged later without being lost.
9. As a developer, I want `null`, `undefined` and `''` asserted to return `''`, so that the empty-value contract used by the views is protected.
10. As a developer, I want the throw on non-string input (e.g. a number) recorded as a live test with a comment, so that a later hardening change is noticed and the test consciously updated.
11. As a security-minded maintainer, I want no real email addresses in test data (only `example.com`-style RFC 2606 domains), so that no personal data enters the repo.
12. As a maintainer, I want the spec to run under the existing `npm run unit` (Jest 22, no `test.each`) and pass the `unit_test` CI job, so that it fits the current pipeline.
13. As a product owner, I want the confirmed intent for short **Local parts** recorded in the ticket, so that the decision is traceable.

## Implementation Decisions

- No production code changes; only the test directory is touched.
- Single test seam: the public `regEmail` function reached through the default `common` utility object. Assertions cover returned strings, or whether the call throws, only.
- Follow the sibling specs' style: Japanese `describe` titles, table-driven cases with plain `forEach`, a header comment stating this is a characterization test.
- Case groups: Local part of 4+ characters (including the 4-character boundary and longer ones); Local part of 3 or fewer (3, 2, 1 characters); no `@` / leading `@` / empty domain; multiple `@`; `null` / `undefined` / `''`; non-string input throws; realistic `.`/`+` local parts; domain never masked.
- Current observed outputs to pin: `abcdef@example.com` → `ab***f@example.com`; `abcd@example.com` → `ab*d@example.com`; `abc@example.com`, `abc`, `@example.com`, `abcd@` → unchanged; `abcde@x@example.com` → `ab**x@example.com`; `ab@x@example.com` → unchanged; `first.last@example.com` → `fi*******t@example.com`; `123` → throws.
- Vocabulary follows CONTEXT.md (Masked email, Local part, Unmasked short address); the short-local-part rule follows ADR 0001.

## Testing Decisions

- A good test asserts external behavior (input → output or throw) and does not depend on the regex or loop inside the function.
- Only `regEmail` is tested; the 16 Vue call sites are not, as they add no logic.
- Prior art: the other specs in the same utils spec directory (e.g. the `handleInt` spec).
- Done when `npm run unit` exits 0, the `unit_test` job passes in the MR pipeline, and the short-local-part confirmation is commented on the Backlog ticket.

## Out of Scope

- Fixing any suspicious behavior (short local parts unmasked, multiple-`@` output, throw on non-string); each needs its own ticket and, if the rule changes, a new ADR.
- The identical `regEmail` in mobile-registration (no spec there; to be mentioned in the ticket comment, follow-up ticket at the team's discretion).
- Masking of phone numbers or other fields.

## Further Notes

- Decision record: ADR 0001 (short local parts intentionally unmasked, confirmed by 簡 大喬).
- Interview record: Works/Tasks/HASHST_DEV-4545/docs/interview.md.
