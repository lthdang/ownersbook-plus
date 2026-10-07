# HASHST_DEV-4562 Interview Record

- Date: 2026-10-06
- Ticket: app-vuejs: remove 「当初」 from the label shown when an offering ends (`common.js` `getInStockStr`)
- Branch: `backlog/HASHST_DEV-4562` (app-vuejs, already checked out)

## 1. Summary of understanding (confirmed)

1. On the investor app (app-vuejs), a fund in its primary offering (`PRIMARY` / `PRIENTRY`) that has sold out (remaining ratio 0 and remaining units 0) currently shows 「当初募集終了」.
2. The wording changes to 「募集終了」. The label is defined in only one place, `getInStockStr` in `app-vuejs/src/utils/common.js:998`.
3. The work is done test-first: change the 3 expected values in the spec and confirm they fail (Red), fix `common.js` (Green), then check f1200 and f1202_pre in the integration environment.
4. The ticket already settled two points: 仮募集 (provisional offering) is unused, so it needs no separate branch; and the label means "sold out", not "offering period over", which is accepted.

## 2. Affected directories (verified by grep)

| Directory | Affected? | Evidence |
|---|---|---|
| app-vuejs | Yes | `src/utils/common.js:998`, and spec `test/unit/specs/utils/common.getInStockStr.spec.js` lines 11, 12, 24 (3 cases expecting 「当初募集終了」) |
| st-admin | No | Only the admin field name 「当初募集終了日時」 (end of the offering period), in `App/Configs/Lang.inc.php:199,203` and comments in `App/Models/OperationModel.php` / `FundModel.php`. A different concept, not investor-facing. |
| basis-admin, common-front-api, company-admin, mobile-registration, passport, st-bc-explorer | No | No matches |
| st-li | n/a | This directory doesn't exist in the workspace |

Callers of `getInStockStr`:
- `src/views/Resiger/f1200.vue:158`: 募集状況·期間, inside the `fundInfo.PERIOD == 'PRIMARY'` block
- `src/views/Resiger/f1200.vue:202`: 在庫状況, inside the `v-else` block
- `src/views/Resiger/f1202_pre.vue:110`: 募集状況・期間

## 3. Findings from the code

- **f1200:202 edge case.** The block switch tests `fundInfo.PERIOD == 'PRIMARY'` (f1200.vue:123), so a `PRIENTRY` fund falls into the `v-else` block. That block calls `getInStockStr(PRIENTRY, '0', SELF_FUND_QTY_SETTLEMENT)`. If the quantity is 0, the 在庫状況 row would show the label. 仮募集 is unused, so there is no impact in practice and nothing to do.
- **HASHST_DEV-4558 conflict.** It has no branch locally or on the remote, and nothing is merged into `js-ph2`. There is no conflict right now. Re-check before merging.
- The spec comment says 「現状の挙動を記録する characterization test」. With the expected values changed, that still describes the file correctly, so it needs no edit.

## 4. Q&A

### Q1: Is 「募集終了」 a separate term from 「当初募集終了日時」?

- **Proposed answer:** Yes. Record them as two terms. 募集終了 is the investor-facing status for a sold-out fund in its primary offering, with no link to the period. 当初募集終了日時 is the configured date and time the primary offering period ends. No ADR.
- **User's answer:** Agreed.
- **Result:** Added 当初募集, 募集終了 and 当初募集終了日時 to `CONTEXT.md`, plus one line under Relationships.

### Points settled in the ticket itself

| Point | Decision |
|---|---|
| Does 仮募集 (f1202_pre) need a separate label? | No. 仮募集 is unused, so no branch is needed |
| Is it fine to show 「募集終了」 when the fund sells out during the offering period? | Yes, that meaning is accepted |

## 5. Decisions

- Change the label from 「当初募集終了」 to 「募集終了」, shared by `PRIMARY` and `PRIENTRY`, with no new condition.
- Change the expected value in the 3 spec cases to 「募集終了」 (Red → Green).
- st-admin's 「当初募集終了日時」 is out of scope.
- No ADR: the change is easy to reverse and involves no real trade-off.

## 6. Remaining items (for the implementation and verification steps)

- Integration check: prepare a sold-out fund (`PRIMARY`, remaining ratio 0, remaining units 0) and confirm f1200 and f1202_pre show 「募集終了」. Decide how to get that test data when doing the check.
- HASHST_DEV-4558 has not started (confirmed by the user). This ticket does not wait for it: this one goes first, and HASHST_DEV-4558 rebases onto it. When HASHST_DEV-4558 is done, any case where null/undefined is treated as sold out must expect 「募集終了」, not 「当初募集終了」.
