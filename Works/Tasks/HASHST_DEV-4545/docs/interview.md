# HASHST_DEV-4545 — Grill session record

Ticket: add characterization unit tests for `regEmail` (app-vuejs `src/utils/common.js`).

## Understanding
- Add `test/unit/specs/utils/common.regEmail.spec.js`; only `test/` changes, no `src/` edits.
- Local part >= 4 chars: keep first 2 + last 1, middle → `*`. <= 3 chars: returned unchanged (intended per 簡 大喬). Falsy → `''`.
- Characterization test: record suspicious behavior with comments, do not fix.

## Affected directories (from code)
- app-vuejs: target (`common.js:907`, 16 call sites, Jest 22 specs exist).
- mobile-registration: identical copy (`common-js.js:931`), 1 caller, no spec → out of scope.
- basis-admin, common-front-api, company-admin, passport, st-admin, st-bc-explorer: no `regEmail`. st-li: not in workspace.

## Code facts (run in scratchpad)
- `abcdef@example.com` → `ab***f@example.com`; `abcd@example.com` → `ab*d@example.com`
- `abc@example.com`, `abc`, `@example.com`, `abcd@` → unchanged
- `abcde@x@example.com` → `ab**x@example.com`; `ab@x@example.com` → unchanged
- `123` → throws `email.replace is not a function`

## Decisions
- Q1 Scope: app-vuejs only; mention the mobile-registration duplicate in the ticket comment.
- Q2 Multiple `@`: pin actual outputs with a "possibly unintended, not fixed" comment; flag as low priority in the ticket.
- Q3/Q4 Non-string input: include a live test `expect(() => common.regEmail(123)).toThrow()` with an explanatory comment.
- Q5 Layout: follow `common.handleInt.spec.js` (JP describe titles, `forEach` tables). Groups: >=4 chars incl. boundary, <=3 chars incl. boundary, no `@`/leading `@`/`abcd@`, multiple `@`, null/undefined/'', non-string throw. Add `first.last@example.com` → `fi*******t@example.com` and one assertion that the domain is never masked. Only `example.com` (RFC 2606) domains.
- Q6 Docs: created `CONTEXT.md` (Masked email, Local part, Unmasked short address) and `docs/adr/0001-short-email-local-part-is-not-masked.md`.

## Follow-ups for the ticket comment
- Confirmation that <= 3 chars unmasked is intended (already in ticket).
- mobile-registration has an identical `regEmail` with no spec.
- Multiple-`@` behavior is odd; optional separate ticket.
