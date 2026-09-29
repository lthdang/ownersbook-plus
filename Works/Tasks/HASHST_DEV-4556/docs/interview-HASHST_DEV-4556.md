# HASHST_DEV-4556 Interview notes

Date: 2026-10-01
Ticket: `verifyInputText` in app-vuejs rejects **Out-of-range kanji** such as 﨑 and 𠮷 (`src/utils/common.js`)
Related: [CONTEXT.md](../../../../CONTEXT.md), [ADR 0002](../../../../docs/adr/0002-front-end-input-character-check-is-loose.md), new ticket draft [mobile-registration_verifyInputText-kanji-range.md](../new-issue/mobile-registration_verifyInputText-kanji-range.md)

## 1. Understanding (confirmed)

1. `verifyInputText` (`src/utils/common.js:684`) allows kanji only in U+4E00–U+9FFF and `々`, so 﨑 (U+FA11), 𠮷 (U+20BB7) and 㐂 (U+3402) are rejected for all kanji types 1, 3, 4, 5, 7, 9, 10, 11, 12.
2. Policy (簡さん, 2026-09-30): the **Input character check** is loose. Characters are narrowed later by the **SJIS-win check** (`checkJpText`, common-front-api) and by conversion to **保振制度内字** at 審査 (company-admin).
3. Fix: add Ext-A U+3400–4DBF, Compatibility U+F900–FAFF and U+20000–3FFFF to each of the 9 regexes, using the `u` flag. Remove escapes that become invalid under `u` (types 10 and 11). The shared range is not refactored.
4. TDD on the 4542 spec "verifyInputText（漢字の範囲）". Then check on testing that 𠮷 on `l2212` is rejected by `checkJpText` with E103-1045. IVS characters stay out of scope.

## 2. Affected directories (checked in the code)

| Dir | Result | Evidence |
|---|---|---|
| app-vuejs | **Changed** | Only `src/utils/common.js` has kanji-range regexes (pdfjs in `static/` is not relevant) |
| mobile-registration | **Same problem, separate ticket** | `src/utils/common-js.js:202-236` has an identical `verifyInputText`. Used in account registration screens |
| common-front-api | No change | `CommonController::checkJpText` uses `findUnsupportCharacter` (SJIS-win round trip, `app/helpers.php:387`). Request rules have no kanji regex. The default DB connection is `utf8mb4` |
| company-admin | No change | `fuel/app/config/jasdeccharacter.php` is the conversion table to 保振制度内字 |
| basis-admin, passport, st-admin, st-bc-explorer | Not affected | No kanji-range checks found |
| st-li | n/a | This directory does not exist. `st-cli` has no matches either |

## 3. Facts found during the session

- **Screens that skip the SJIS-win check.** These app-vuejs screens call a kanji type of `verifyInputText` but never call `checkJpText`, so after the fix 𠮷 reaches the server unchecked:
  `l2213` (nationality, type4), `l2216` (occupation, type5), `l2217` (insider, type9/12), `l2251` (fund insider, type12), `l2500` (取引暗証番号再設定), `x1201` (アカウントロック解除), `y1200` (ログインID忘れ), `x2301`, `y2100`, `l2504` / `x1204` / `x2304` / `y1204` / `y2104` (type1/11).
  Screens that do call `checkJpText`: `l2212`, `l2221`, `l2223`, `l2228`, `l2229`, `l2232`, `components/ruleAddress`.
- **History tables use 3-byte utf8.** `migration/app/Console/Migration/MyTriggerTrait.php:108` creates history tables with `DEFAULT CHARSET=utf8`, while MySQL defaults to `utf8mb4` (`local-docker/etc/*/mysql/my.cnf`). A 4-byte character written by a history trigger may fail.
- **Identity-verification screens** (`l2500` etc.) send `NAME` to the server (e.g. `/user/secret/send_reset_verify_code`), which compares it with the registered name.
- **Babel.** `.babelrc` uses `babel-preset-env` (babel 6). `babel-plugin-transform-es2015-unicode-regex` is installed in app-vuejs and mobile-registration.
- **Length limits.** `maxlength` is 30 for names on `l2212`, 50 for addresses, 60 for insider names on `l2217`, and 100 on `l2216`. A surrogate pair such as 𠮷 counts as 2.
- **Branch.** `backlog/HASHST_DEV-4556` was fast-forwarded by the user and is now equal to `origin/js-ph2` (`f68dad7d`), including the 4542 merge `4e0bb8f2`. `config/dev.env.js` has local changes and must not be committed.

## 4. Q&A

### Round 1

**Q1 – Is mobile-registration in scope?**
It has the same `verifyInputText` and is where a name is first registered. If only app-vuejs is fixed, a customer can't register as 山﨑 but can later change their name to 山﨑.
Proposed: fix app-vuejs only and raise a separate ticket for mobile-registration.
**Decision: separate ticket.** Draft saved to `new-issue/mobile-registration_verifyInputText-kanji-range.md`.

**Q2 – What about the 14 screens that skip `checkJpText`?**
Options: (a) accept, (b) add `checkJpText` to those screens, (c) leave out the 4-byte range.
Proposed: (a), plus a testing check that saving 𠮷 on a screen without `checkJpText` (e.g. `l2216`) works without a DB error. If it fails, raise a separate ticket and choose between (b) and (c).
**Decision: OK (a), with the testing check.**

**Q3 – Name mismatch on identity-verification screens**
Typing 﨑 when 崎 is registered used to give a format error. After the fix it gives a "details don't match" error.
Proposed: accept and note it in the ticket.
**Decision: OK.**

**Q4 – Branch base**
Proposed: fast-forward `backlog/HASHST_DEV-4556` to `origin/js-ph2`, and target the MR at `js-ph2`.
**Decision: OK.** The user updated the branch, and it was checked to be equal to `origin/js-ph2`.

### Round 2

**Q5 – How much test coverage?**
Proposed: a table of 3 characters (﨑 / 㐂 / 𠮷) × all 9 kanji types, built with `forEach` because Jest 22 has no `test.each`. Add boundary values (U+3400 / U+4DBF / U+F900 / U+FAFF / U+20000) on type1 only. Add tests showing that 😀 U+1F600 and a character followed by an IVS selector are still rejected.
**Decision: OK.**

**Q6 – Length limits with surrogate pairs**
Proposed: out of scope, recorded as a known limitation.
**Decision: OK.**

## 5. Final scope (app-vuejs only)

1. **Red:** in `test/unit/specs/utils/common.verifyInputText.spec.js`, "verifyInputText（漢字の範囲）":
   - change `山﨑` and `𠮷田` to `true` and remove the `【現状の挙動】` comment
   - add the 3 characters × 9 types table (types 1, 3, 4, 5, 7, 9, 10, 11, 12)
   - add range-boundary cases on type1
   - add cases showing 😀 and IVS characters are still rejected
2. **Green:** in `src/utils/common.js`, add `㐀-䶿`, `豈-﫿` and `\u{20000}-\u{3FFFF}` to the 9 regexes, add the `u` flag, and remove the escapes that become invalid under `u` in types 10 and 11.
3. **Refactor:** check that all existing tests still pass, especially the type10 and type11 symbol tests. Check that the build output transforms the `u` regexes.
4. **Testing environment:**
   - On `l2212`, 𠮷 is rejected with 「𠮷の入力はできません。ひらがなで入力してください。」 and 山﨑 is accepted.
   - On a screen that skips `checkJpText` (e.g. `l2216`), 𠮷 saves and displays without a DB error. If it fails, raise a new ticket.
5. **Commit:** branch `backlog/HASHST_DEV-4556`. Message `HASHST_DEV-4556 <summary>` followed by a Japanese description, without a Co-Authored-By line. Leave out `config/dev.env.js`, `AGENTS.md` and `CLAUDE.md`. MR to `js-ph2`.

## 6. Out of scope / known limitations

- mobile-registration (separate ticket)
- Characters with IVS selectors (U+E0100–)
- Changes to `checkJpText` or the E103-1045 wording (common-front-api). Raise a separate ticket if needed
- Adding `checkJpText` to the screens that skip it
- `maxlength` / `.length` count 𠮷 as 2 characters
- On identity-verification screens, 﨑 now gives a mismatch error instead of a format error (accepted)

## 7. Suggested additions to the Backlog ticket

- 対応方針 3: add the testing check on a screen without `checkJpText` (e.g. `l2216`), saving 𠮷 without a DB error.
- 対応方針 4: add the 3 characters × 9 types table, the boundary cases, and the 😀 / IVS rejection cases.
- 対応方針 5: add the identity-verification behavior change, the `maxlength` limitation, and a link to the mobile-registration ticket.
