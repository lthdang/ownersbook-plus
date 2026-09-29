# Email local parts of 3 characters or fewer are not masked

The on-screen **Masked email** keeps the first 2 and last 1 characters of the local part. A local part of 3 or fewer characters would have nothing left to hide, so it is shown in full (**Unmasked short address**).

This was the original specification, confirmed as intended by 簡 大喬 in HASHST_DEV-4545: the display format is `xx*****X@domain`. We accept that very short local parts are visible in exchange for matching that display rule.

## Consequences

- Unit tests for `regEmail` (HASHST_DEV-4545) record this as current behavior, not a defect.
- Changing it later (e.g. masking short local parts) is a product decision and needs a new ticket and a new ADR.
