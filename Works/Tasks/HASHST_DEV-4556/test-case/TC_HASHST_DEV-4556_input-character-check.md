# Test Cases: Input character check accepts out-of-range kanji (HASHST_DEV-4556)

- **Function:** Input character check (`verifyInputText`) accepts out-of-range kanji such as 﨑, 㐂 and 𠮷 in name, address and company-name fields
- **Module:** InputCheck (app-vuejs, お客様情報変更 / 本人確認系 screens)
- **Environment:** Local DEV (local-docker), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`, `npm run dev`
- **Known business rules (from HASHST_DEV-4556 / ADR 0002):**
  - Rule 1: Kanji fields (types 1, 3, 4, 5, 7, 9, 10, 11, 12) accept CJK Unified U+4E00–U+9FFF, Ext-A U+3400–U+4DBF, Compatibility U+F900–U+FAFF and U+20000–U+3FFFF.
  - Rule 2: Characters with IVS selectors, emoji and other symbols stay rejected.
  - Rule 3: On screens that call the SJIS-win check, characters not in SJIS-win (㐂, 𠮷) are rejected on blur with E103-1045 「{文字}の入力はできません。ひらがなで入力してください。」.
  - Rule 4: Name changes on l2212 go to 審査 and are not applied immediately.

**Proposed requirement IDs**

| Req ID | Requirement |
|---|---|
| REQ_INPUTCHK_01 (proposed) | The Input character check accepts 3-byte out-of-range kanji (Ext-A U+3400–U+4DBF, Compatibility U+F900–U+FAFF) in every kanji field |
| REQ_INPUTCHK_02 (proposed) | The Input character check accepts 4-byte kanji (U+20000–U+3FFFF) in every kanji field |
| REQ_INPUTCHK_03 (proposed) | The Input character check still rejects characters outside the allowed ranges (emoji, IVS, symbols, markup) |
| REQ_INPUTCHK_04 (proposed) | The SJIS-win check still rejects characters not in SJIS-win after the Input character check passes |
| REQ_INPUTCHK_05 (proposed) | Values accepted by the Input character check are saved when the form is submitted |
| REQ_INPUTCHK_06 (proposed) | Identity-verification screens accept out-of-range kanji and match them against the registered name |
| REQ_INPUTCHK_07 (proposed) | Customer-information change screens can only be opened through 設定 → お客様情報の確認・変更 (existing behaviour, regression) |

## 1. Test Case List

| Test Case ID | Module | Req ID | Test Title | Priority | Preconditions | Test Steps | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TC_INPUTCHK_001 | InputCheck | REQ_INPUTCHK_01 (proposed) | Accept compatibility kanji 﨑 in 姓 on l2212 without an error | High | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. The お名前 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200 (マイページ → 設定 → お客様情報の確認・変更 → 取引暗証番号 entered) | 1. Tap 「変更」 on the お名前 card<br>2. Select 「情報の変更」<br>3. Enter `山﨑` in the 姓 field of お名前（全角）<br>4. Tap the 名 field | /l2212 is displayed<br>No error message is shown under 姓 | /l2212 displayed. 姓 kept `山﨑`; after tapping 名, no error message was shown under 姓.<br>Screenshot: `screenshot/TC_INPUTCHK_001.png` | | | SJIS-win check runs on blur; 﨑 is in SJIS-win (IBM extension) |
| TC_INPUTCHK_002 | InputCheck | REQ_INPUTCHK_01 (proposed) | Accept compatibility kanji 﨑 in 市区町村・町域名 on l2212 without an error | Medium | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. The お名前 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200 | 1. Tap 「変更」 on the お名前 card<br>2. Select 「情報の変更」<br>3. Enter `山﨑町` in 市区町村・町域名<br>4. Tap the 番地（丁目、番、号はハイフンで入力） field | No error message is shown under 市区町村・町域名 | 市区町村・町域名 kept `山﨑町`; after tapping 番地, no error message was shown.<br>Screenshot: `screenshot/TC_INPUTCHK_002.png` | | | Whether the address fields are editable depends on the account's address state (unverified) |
| TC_INPUTCHK_003 | InputCheck | REQ_INPUTCHK_01 (proposed) | Accept Ext-A kanji 㐂 in 国名 on l2213 without an error | Medium | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. The 国籍 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200 | 1. Tap 「変更」 on the 国籍 card<br>2. Select 「情報の変更」<br>3. Select 「日本以外」 in 国籍<br>4. Enter `㐂` in 国名<br>5. Tap outside the 国名 field | The 国名 field is displayed after step 3<br>No error message is shown under 国名 | The 国籍 option is labelled 「日本以外」 (not 「その他」). After selecting it, 国名 was displayed; `㐂` was kept and no error message was shown after leaving the field.<br>Screenshot: `screenshot/TC_INPUTCHK_003.png` | | | l2213 has no SJIS-win check. Do not submit: submit goes to eKYC |
| TC_INPUTCHK_004 | InputCheck | REQ_INPUTCHK_01 (proposed) | Accept compatibility kanji 﨑 in 内部者氏名 on l2251 without an error | Medium | 1. Individual test account is logged in<br>2. Tester is on /l2200 | 1. Tap the 当社取扱ファンド内部者登録 card<br>2. Tap 「内部者登録を新しく追加」<br>3. Tap 内部者登録区分 of the added 内部者登録, select 「役員の配偶者及び同居者」 and tap 「OK」<br>4. Enter `山﨑` in 内部者氏名<br>5. Tap outside the 内部者氏名 field | /l2251 内部者登録 is displayed<br>内部者氏名 becomes editable after step 3<br>No error message is shown under 内部者氏名 | After 「内部者登録を新しく追加」, 内部者氏名 stayed disabled until 内部者登録区分 「役員の配偶者及び同居者」 was selected (the only option that enables it). Then `山﨑` was kept and no error message was shown.<br>Screenshot: `screenshot/TC_INPUTCHK_004.png` | | | 内部者氏名 is enabled only for 内部者登録区分 「役員の配偶者及び同居者」. No SJIS-win check on l2251. Do not submit |
| TC_INPUTCHK_005 | InputCheck | REQ_INPUTCHK_01 (proposed) | Accept compatibility kanji 﨑 in 法人名（全角） on l2221 without an error | High | 1. Corporate test account is logged in<br>2. The 法人名 card on /l2900 is not 「変更中」<br>3. Tester is on /l2900 (マイページ → 設定 → お客様情報の確認・変更 → 取引暗証番号 entered) | 1. Tap 「変更」 on the 法人名 card<br>2. Select 「情報の変更」 (only if the radio is shown)<br>3. Enter `株式会社山﨑` in 法人名（全角）<br>4. Tap the 法人名（全角カタカナ） field | /l2221 is displayed<br>No error message is shown under 法人名（全角） | (Corporate account lthdang+2. /l2221 shows no 「情報の変更 / 確認書類のみ再提出」 radio for this account; the fields are editable directly, so step 2 was skipped.) /l2221 displayed. 法人名（全角） kept `株式会社山﨑`; after tapping 法人名（全角カタカナ）, no error message was shown.<br>Screenshot: `screenshot/TC_INPUTCHK_005.png` | | | SJIS-win check runs on blur. Some corporate accounts show no 「情報の変更 / 確認書類のみ再提出」 radio; the fields are then editable directly |
| TC_INPUTCHK_006 | InputCheck | REQ_INPUTCHK_05 (proposed) | Save 勤務先名 containing 﨑 on l2216 | High | 1. Individual test account is logged in<br>2. The account has an occupation registered (/l2216 is blank when OCCUPATION_TYPE is null)<br>3. The 職業 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200<br>5. Tester knows the account's 取引暗証番号 | 1. Tap 「変更」 on the 職業 card<br>2. Confirm 職業 is an occupation that shows 勤務先名<br>3. Enter `山﨑商事` in 勤務先名<br>4. Enter `営業部` in 所属部署<br>5. Enter `課長` in 役職名<br>6. Enter the 取引暗証番号 in the 取引暗証番号を入力してください field<br>7. Tap 「変更する」 | No error message is shown under 勤務先名<br>/l2203 「変更が完了しました」 is displayed<br>Reopening /l2216 shows 勤務先名 `山﨑商事` | (Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.) (Harness added the X-CT header: the local dev build never sends it, so without it every /user/job call returns an empty 200 and the app shows the /z1200 error page.) No error under 勤務先名. 業種 is not on the screen (commented out) and the 取引暗証番号 is entered on the page, not in a popup. After 「変更する」, /user/job returned 200 and /l2203 「変更が完了しました」 was shown; reopening /l2216 showed 勤務先名 `山﨑商事`.<br>Screenshots: `screenshot/TC_INPUTCHK_006_before_submit.png`, `screenshot/TC_INPUTCHK_006_after_submit.png`, `screenshot/TC_INPUTCHK_006_reopened.png` | | | The local dev build never sends the X-CT header, so /user/job returns an empty 200 there; test on testing or add X-CT. Restore the original value afterwards |
| TC_INPUTCHK_007 | InputCheck | REQ_INPUTCHK_02 (proposed) | Accept Ext-B kanji 𠮷 in 勤務先名 on l2216 without an error | Medium | 1. Individual test account is logged in<br>2. The 職業 card on /l2200 is not 「変更中」<br>3. Tester is on /l2200 | 1. Tap 「変更」 on the 職業 card<br>2. Select an occupation that shows 勤務先名 in 職業<br>3. Enter `𠮷田商事` in 勤務先名<br>4. Tap the 所属部署 field | No error message (「特殊文字は入力できません。」) is shown under 勤務先名 | (Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.) `𠮷田商事` (U+20BB7 …) was kept and no error message was shown under 勤務先名.<br>Screenshot: `screenshot/TC_INPUTCHK_007.png` | | | Front-end check only; saving is TC_INPUTCHK_014 |
| TC_INPUTCHK_008 | InputCheck | REQ_INPUTCHK_06 (proposed) | Pass the identity check on l2500 with a registered name containing 﨑 | Medium | 1. Individual test account is logged in<br>2. The account's registered name is `山﨑 太郎` (local test data)<br>3. The account's registered 生年月日 and 電話番号 are known<br>4. Tester is on マイページ → 設定 | 1. Tap 「取引暗証番号をお忘れの方はこちら」<br>2. Enter `山﨑` in 姓<br>3. Enter `太郎` in 名<br>4. Enter the registered 生年月日 in 生年月日<br>5. Enter the registered 電話番号 in 電話番号<br>6. Tap 「認証メール送信」 | /l2500 取引暗証番号の再設定 is displayed<br>No error message is shown under お名前（全角）<br>/l2501 (2段階認証) is displayed | (Run after setting the registered NAME to 「山﨑　太郎」 in the local DB; reverted afterwards.) No error under お名前（全角）. 「認証メール送信」 → /user/secret/send_reset_verify_code returned OK and /l2501 was displayed.<br>Screenshot: `screenshot/TC_INPUTCHK_008.png` | | | Requires preparing local test data with 﨑 in NAME. 𠮷 cannot be prepared (DB is utf8mb3) |
| TC_INPUTCHK_009 | InputCheck | REQ_INPUTCHK_04 (proposed) | Show the SJIS-win error for Ext-A kanji 㐂 in 姓 on l2212 | High | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. The お名前 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200 | 1. Tap 「変更」 on the お名前 card<br>2. Select 「情報の変更」<br>3. Enter `㐂` in the 姓 field of お名前（全角）<br>4. Tap the 名 field | 「㐂の入力はできません。ひらがなで入力してください。」 is shown under 姓 | 「㐂の入力はできません。ひらがなで入力してください。」 was shown under 姓. The message overlaps the 「お名前（全角カタカナ）」 label below (layout).<br>Screenshot: `screenshot/TC_INPUTCHK_009.png` | | | Before the fix the message was 「姓を正しく入力してください。」 |
| TC_INPUTCHK_010 | InputCheck | REQ_INPUTCHK_04 (proposed) | Show the SJIS-win error for Ext-B kanji 𠮷 in 姓 on l2212 | High | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. The お名前 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200 | 1. Tap 「変更」 on the お名前 card<br>2. Select 「情報の変更」<br>3. Enter `𠮷田` in the 姓 field of お名前（全角）<br>4. Tap the 名 field | 「𠮷の入力はできません。ひらがなで入力してください。」 is shown under 姓 | 「𠮷の入力はできません。ひらがなで入力してください。」 was shown under 姓. The message overlaps the 「お名前（全角カタカナ）」 label below (layout).<br>Screenshot: `screenshot/TC_INPUTCHK_010.png` | | | The message does not suggest 吉 as an alternative (out of scope, common-front-api) |
| TC_INPUTCHK_011 | InputCheck | REQ_INPUTCHK_04 (proposed) | Show the SJIS-win error for Ext-B kanji 𠮷 in 法人名（全角） on l2221 | Medium | 1. Corporate test account is logged in<br>2. The 法人名 card on /l2900 is not 「変更中」<br>3. Tester is on /l2900 | 1. Tap 「変更」 on the 法人名 card<br>2. Select 「情報の変更」 (only if the radio is shown)<br>3. Enter `株式会社𠮷田` in 法人名（全角）<br>4. Tap the 法人名（全角カタカナ） field | 「𠮷の入力はできません。ひらがなで入力してください。」 is shown under 法人名（全角） | (Corporate account lthdang+2. /l2221 shows no 「情報の変更 / 確認書類のみ再提出」 radio for this account; the fields are editable directly, so step 2 was skipped.) /l2221 displayed. After tapping 法人名（全角カタカナ）, 「𠮷の入力はできません。ひらがなで入力してください。」 was shown under 法人名（全角）.<br>Screenshot: `screenshot/TC_INPUTCHK_011.png` | | | Before the fix the message was 「法人名を全角で入力してください。」. Some corporate accounts show no 「情報の変更 / 確認書類のみ再提出」 radio; the fields are then editable directly |
| TC_INPUTCHK_012 | InputCheck | REQ_INPUTCHK_03 (proposed) | Remove an emoji from 姓 on l2212 on blur without showing an error | High | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. The お名前 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200 | 1. Tap 「変更」 on the お名前 card<br>2. Select 「情報の変更」<br>3. Enter `山田😀` in the 姓 field of お名前（全角）<br>4. Tap the 名 field | The emoji is removed and 姓 shows `山田`<br>No error message is shown under 姓 | The emoji was removed from 姓 on blur (value became 「山田」) and no error message was shown; 「姓を正しく入力してください。」 did not appear. Value entered by paste; typing the emoji is blocked by the keypress handler.<br>Screenshot: `screenshot/TC_INPUTCHK_012.png` | | | Existing behaviour: the 姓/名 blur handler strips emoji (removeEmojiNSpace). Typing an emoji is blocked by the keypress handler, so paste it |
| TC_INPUTCHK_013 | InputCheck | REQ_INPUTCHK_03 (proposed) | Reject a kanji with an IVS selector in 勤務先名 on l2216 | Low | 1. Individual test account is logged in<br>2. The 職業 card on /l2200 is not 「変更中」<br>3. Tester is on /l2200 | 1. Tap 「変更」 on the 職業 card<br>2. Select an occupation that shows 勤務先名 in 職業<br>3. Paste `葛󠄀城商事` (葛 + U+E0100) into 勤務先名<br>4. Tap the 所属部署 field | 「特殊文字は入力できません。」 is shown under 勤務先名 | (Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.) `葛󠄀城商事` (U+845B U+E0100 …) → 「特殊文字は入力できません。」 was shown under 勤務先名.<br>Screenshot: `screenshot/TC_INPUTCHK_013.png` | | | IVS is out of scope for this ticket and stays rejected. The selector is invisible; copy the value from this document |
| TC_INPUTCHK_014 | InputCheck | REQ_INPUTCHK_05 (proposed) | Save 勤務先名 containing Ext-B kanji 𠮷 on l2216 | High | 1. Individual test account is logged in<br>2. The account has an occupation registered (/l2216 is blank when OCCUPATION_TYPE is null)<br>3. The 職業 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200<br>5. Tester knows the account's 取引暗証番号<br>6. DevTools Network tab is open | 1. Tap 「変更」 on the 職業 card<br>2. Confirm 職業 is an occupation that shows 勤務先名<br>3. Enter `𠮷田商事` in 勤務先名<br>4. Enter `営業部` in 所属部署<br>5. Enter `課長` in 役職名<br>6. Enter the 取引暗証番号 in the 取引暗証番号を入力してください field<br>7. Tap 「変更する」 | /l2203 「変更が完了しました」 is displayed<br>The `/user/job` response status is 200<br>Reopening /l2216 shows 勤務先名 `𠮷田商事` | (Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.) (Harness added the X-CT header: the local dev build never sends it, so without it every /user/job call returns an empty 200 and the app shows the /z1200 error page.) No error under 勤務先名 before submit. After 「変更する」, /user/job returned **500** `SQLSTATE[HY000]: General error: 3988 Conversion from collation utf8mb4_unicode_ci into utf8mb3_general_ci impossible for parameter` (insert into `hhd`.`ECO_USER_CHANGE`) and the app showed the /z1200 system error page. Nothing was saved (勤務先名 stayed `山﨑商事`).<br>Screenshots: `screenshot/TC_INPUTCHK_014_before_submit.png`, `screenshot/TC_INPUTCHK_014_after_submit.png` | | | **FAILED locally (2026-10-05):** /user/job returned 500 `General error: 3988 Conversion from collation utf8mb4_unicode_ci into utf8mb3_general_ci impossible` and /z1200 was shown. Columns are utf8mb3. Decide per ADR 0002. The local dev build also needs the X-CT header |
| TC_INPUTCHK_015 | InputCheck | REQ_INPUTCHK_06 (proposed) | Reject the identity check on l2500 when 﨑 is entered but 崎 is registered | Medium | 1. Individual test account is logged in<br>2. The account's registered name is `山崎 太郎`<br>3. The account's registered 生年月日 and 電話番号 are known<br>4. Tester is on マイページ → 設定 | 1. Tap 「取引暗証番号をお忘れの方はこちら」<br>2. Enter `山﨑` in 姓<br>3. Enter `太郎` in 名<br>4. Enter the registered 生年月日 in 生年月日<br>5. Enter the registered 電話番号 in 電話番号<br>6. Tap 「認証メール送信」 | No error message is shown under お名前（全角）<br>/l2501 is not displayed; an error saying the entered information does not match is shown | (Run after setting the registered NAME to 「山崎　太郎」 in the local DB; reverted afterwards.) No error under お名前（全角） for 姓 `山﨑`. 「認証メール送信」 → /user/secret/send_reset_verify_code returned E103-0201 and the dialog 「登録情報を正確に入力してください。」 was shown; /l2501 was not displayed.<br>Screenshot: `screenshot/TC_INPUTCHK_015.png` | | | Accepted behaviour change (ADR 0002): format error before, mismatch error after. The exact mismatch message comes from the server and is unverified |
| TC_INPUTCHK_016 | InputCheck | REQ_INPUTCHK_01 (proposed) | Accept the first Ext-A code point U+3400 in 勤務先名 on l2216 | Medium | 1. Individual test account is logged in<br>2. The 職業 card on /l2200 is not 「変更中」<br>3. Tester is on /l2200 | 1. Tap 「変更」 on the 職業 card<br>2. Select an occupation that shows 勤務先名 in 職業<br>3. Paste `㐀` (U+3400) into 勤務先名<br>4. Tap the 所属部署 field | No error message is shown under 勤務先名 | (Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.) U+3400 `㐀` was kept and no error message was shown under 勤務先名.<br>Screenshot: `screenshot/TC_INPUTCHK_016.png` | | | l2216 is used for boundaries because it has no SJIS-win check, so only the Input character check is observed |
| TC_INPUTCHK_017 | InputCheck | REQ_INPUTCHK_01 (proposed) | Accept the first compatibility code point U+F900 in 勤務先名 on l2216 | Medium | 1. Individual test account is logged in<br>2. The 職業 card on /l2200 is not 「変更中」<br>3. Tester is on /l2200 | 1. Tap 「変更」 on the 職業 card<br>2. Select an occupation that shows 勤務先名 in 職業<br>3. Run `copy(String.fromCodePoint(0xF900))` in the DevTools Console<br>4. Paste the clipboard into 勤務先名<br>5. Tap the 所属部署 field | No error message is shown under 勤務先名 | (Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.) U+F900 (entered as `String.fromCodePoint(0xF900)`, value confirmed as U+F900) was kept and no error message was shown under 勤務先名.<br>Screenshot: `screenshot/TC_INPUTCHK_017.png` | | | Do not copy U+F900 from a document: editors normalise it (NFC) to U+8C48, which is inside the old range and does not test the boundary. Check with `document.activeElement.value.codePointAt(0).toString(16)` = `f900` if needed |
| TC_INPUTCHK_018 | InputCheck | REQ_INPUTCHK_03 (proposed) | Reject U+4DC0, just after Ext-A, in 勤務先名 on l2216 | Medium | 1. Individual test account is logged in<br>2. The 職業 card on /l2200 is not 「変更中」<br>3. Tester is on /l2200 | 1. Tap 「変更」 on the 職業 card<br>2. Select an occupation that shows 勤務先名 in 職業<br>3. Paste `䷀` (U+4DC0) into 勤務先名<br>4. Tap the 所属部署 field | 「特殊文字は入力できません。」 is shown under 勤務先名 | (Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.) U+4DC0 `䷀` → 「特殊文字は入力できません。」 was shown under 勤務先名.<br>Screenshot: `screenshot/TC_INPUTCHK_018.png` | | | U+4DC0 is a hexagram symbol, not a kanji |
| TC_INPUTCHK_019 | InputCheck | REQ_INPUTCHK_02 (proposed) | Accept the first Ext-B code point U+20000 in 勤務先名 on l2216 | Low | 1. Individual test account is logged in<br>2. The 職業 card on /l2200 is not 「変更中」<br>3. Tester is on /l2200 | 1. Tap 「変更」 on the 職業 card<br>2. Select an occupation that shows 勤務先名 in 職業<br>3. Paste `𠀀` (U+20000) into 勤務先名<br>4. Tap the 所属部署 field | No error message is shown under 勤務先名 | (Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.) U+20000 `𠀀` was kept and no error message was shown under 勤務先名.<br>Screenshot: `screenshot/TC_INPUTCHK_019.png` | | | Do not submit (see TC_INPUTCHK_014) |
| TC_INPUTCHK_020 | InputCheck | REQ_INPUTCHK_03 (proposed) | Reject HTML markup in 姓 on l2212 with the format error | High | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. The お名前 card on /l2200 is not 「変更中」<br>4. Tester is on /l2200 | 1. Tap 「変更」 on the お名前 card<br>2. Select 「情報の変更」<br>3. Enter `<script>alert(1)</script>` in the 姓 field of お名前（全角）<br>4. Tap the 名 field | 「姓を正しく入力してください。」 is shown under 姓<br>No alert dialog appears | The value was converted to full-width 「<ｓｃｒｉｐｔ>ａｌｅｒｔ(１)</ｓｃｒｉｐｔ>」 and 「姓を正しく入力してください。」 was shown under 姓. No alert dialog appeared.<br>Screenshot: `screenshot/TC_INPUTCHK_020.png` | | | Regression check that widening the ranges did not allow symbols |
| TC_INPUTCHK_021 | InputCheck | REQ_INPUTCHK_07 (proposed) | Redirect to /l2100 when /l2212 is opened directly by URL | Low | 1. Individual test account is logged in<br>2. 取引暗証番号 has not been entered in this session | 1. Enter `http://localhost:8080/#/l2212` in the address bar<br>2. Press Enter | /l2100 設定 is displayed instead of /l2212 | Opening /l2212 directly after login redirected to /l2100 (設定).<br>Screenshot: `screenshot/TC_INPUTCHK_021.png` | | | Existing behaviour, not changed by this ticket. URL form (hash vs history mode) is assumed |
| TC_INPUTCHK_022 | InputCheck | REQ_INPUTCHK_03 (proposed) | Keep the send button disabled while 法人名 contains an emoji on y1204 | Medium | 1. User is not logged in<br>2. Tester is on the login screen | 1. Tap 「ログインIDを忘れた方はこちら」<br>2. Tap 「法人のお客様」<br>3. Enter `株式会社😀` in 法人名<br>4. Tap the 姓 field of 取引責任者のお名前 | An error message is shown under 法人名<br>The 「認証メール送信」 button is disabled | /y1204 displayed. 「法人名を正しく入力してください。」 was shown under 法人名. The send button is labelled 「認証コード送信」 and was in the disabled state (other required fields were also empty).<br>Screenshot: `screenshot/TC_INPUTCHK_022.png` | | | y1204 contains both 「認証メール送信」 and 「認証コード送信」; which one is shown on the 法人のお客様 tab is unverified. Error text 「法人名を正しく入力してください。」 is from the code review, unverified on screen |
| TC_INPUTCHK_023 | InputCheck | REQ_INPUTCHK_03 (proposed) | Hide the お名前 error on l2500 when the 姓 field gets focus again | Low | 1. Individual test account is logged in<br>2. Tester is on マイページ → 設定 | 1. Tap 「取引暗証番号をお忘れの方はこちら」<br>2. Enter `山田😀` in 姓<br>3. Tap the 「お名前（全角）」 label (outside the input fields)<br>4. Tap the 姓 field | After step 3, 「お名前を正しく入力してください。」 is shown under お名前（全角）<br>After step 4, the error message is hidden | Tapping 名 at step 3 hides the error immediately, because focusing either name field clears it. With step 3 done by tapping outside the fields (on 「お名前（全角）」), 「お名前を正しく入力してください。」 was shown; after tapping 姓 (step 4) it was hidden.<br>Screenshots: `screenshot/TC_INPUTCHK_023_step3.png`, `screenshot/TC_INPUTCHK_023_step4.png` | | | Focusing either 姓 or 名 clears the お名前 error, so step 3 must tap outside the fields |

## 2. Test Case Details

### TC_INPUTCHK_001

- **Test Case ID:** TC_INPUTCHK_001
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_01 (proposed)
- **Test Title:** Accept compatibility kanji 﨑 in 姓 on l2212 without an error
- **Priority:** High
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. The お名前 card on /l2200 is not 「変更中」
  4. Tester is on /l2200 (マイページ → 設定 → お客様情報の確認・変更 → 取引暗証番号 entered)
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account (made-up data); 姓 `山﨑` (﨑 U+FA11)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the お名前 card | - | /l2212 is displayed | | | | PC UA is redirected to /l2325 |
| 2 | Select 「情報の変更」 | - | お名前（全角） 姓 and 名 fields are displayed | | | | - |
| 3 | Enter the value in the 姓 field of お名前（全角） | `山﨑` | 姓 shows `山﨑` | | | | - |
| 4 | Tap the 名 field | - | No error message is shown under 姓 | | | | SJIS-win check runs on blur; 﨑 is in SJIS-win |

### TC_INPUTCHK_002

- **Test Case ID:** TC_INPUTCHK_002
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_01 (proposed)
- **Test Title:** Accept compatibility kanji 﨑 in 市区町村・町域名 on l2212 without an error
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. The お名前 card on /l2200 is not 「変更中」
  4. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 市区町村・町域名 `山﨑町`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the お名前 card | - | /l2212 is displayed | | | | - |
| 2 | Select 「情報の変更」 | - | Address fields are displayed | | | | Editability depends on the account's address state (unverified) |
| 3 | Enter the value in 市区町村・町域名 | `山﨑町` | 市区町村・町域名 shows `山﨑町` | | | | - |
| 4 | Tap the 番地（丁目、番、号はハイフンで入力） field | - | No error message is shown under 市区町村・町域名 | | | | - |

### TC_INPUTCHK_003

- **Test Case ID:** TC_INPUTCHK_003
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_01 (proposed)
- **Test Title:** Accept Ext-A kanji 㐂 in 国名 on l2213 without an error
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. The 国籍 card on /l2200 is not 「変更中」
  4. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 国名 `㐂` (U+3402)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 国籍 card | - | /l2213 is displayed | | | | - |
| 2 | Select 「情報の変更」 | - | 国籍 options are displayed | | | | - |
| 3 | Select 「日本以外」 in 国籍 | 「日本以外」 | The 国名 field is displayed | | | | - |
| 4 | Enter the value in 国名 | `㐂` | 国名 shows `㐂` | | | | Copy 㐂 from this document |
| 5 | Tap outside the 国名 field | - | No error message is shown under 国名 | | | | No SJIS-win check on l2213. Do not submit (goes to eKYC) |

### TC_INPUTCHK_004

- **Test Case ID:** TC_INPUTCHK_004
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_01 (proposed)
- **Test Title:** Accept compatibility kanji 﨑 in 内部者氏名 on l2251 without an error
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 内部者氏名 `山﨑`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap the 当社取扱ファンド内部者登録 card | - | /l2251 内部者登録 is displayed | | | | - |
| 2 | Tap 「内部者登録を新しく追加」 | - | A new 内部者登録 block is displayed; 内部者氏名 is disabled | | | | - |
| 3 | Tap 内部者登録区分 of the added block, select the option and tap 「OK」 | 「役員の配偶者及び同居者」 | The option is selected and 内部者氏名 becomes editable | | | | 内部者氏名 is enabled only for this option |
| 4 | Enter the value in 内部者氏名 | `山﨑` | 内部者氏名 shows `山﨑` | | | | - |
| 5 | Tap outside the 内部者氏名 field | - | No error message is shown under 内部者氏名 | | | | No SJIS-win check on l2251 |

### TC_INPUTCHK_005

- **Test Case ID:** TC_INPUTCHK_005
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_01 (proposed)
- **Test Title:** Accept compatibility kanji 﨑 in 法人名（全角） on l2221 without an error
- **Priority:** High
- **Preconditions:**
  1. Corporate test account is logged in
  2. The 法人名 card on /l2900 is not 「変更中」
  3. Tester is on /l2900 (マイページ → 設定 → お客様情報の確認・変更 → 取引暗証番号 entered)
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Corporate test account; 法人名（全角） `株式会社山﨑`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 法人名 card | - | /l2221 is displayed | | | | - |
| 2 | Select 「情報の変更」 (only if the radio is shown) | - | 法人名（全角） field is editable | | | | Some corporate accounts show no radio; skip this step then |
| 3 | Enter the value in 法人名（全角） | `株式会社山﨑` | 法人名（全角） shows `株式会社山﨑` | | | | - |
| 4 | Tap the 法人名（全角カタカナ） field | - | No error message is shown under 法人名（全角） | | | | SJIS-win check runs on blur |

### TC_INPUTCHK_006

- **Test Case ID:** TC_INPUTCHK_006
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_05 (proposed)
- **Test Title:** Save 勤務先名 containing 﨑 on l2216
- **Priority:** High
- **Preconditions:**
  1. Individual test account is logged in
  2. The account has an occupation registered (/l2216 is blank when OCCUPATION_TYPE is null)
  3. The 職業 card on /l2200 is not 「変更中」
  4. Tester is on /l2200
  5. Tester knows the account's 取引暗証番号
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account and its 取引暗証番号; 勤務先名 `山﨑商事`, 所属部署 `営業部`, 役職名 `課長`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 職業 card | - | /l2216 is displayed | | | | - |
| 2 | Confirm 職業 is an occupation that shows 勤務先名 | e.g. 非上場企業の役員・社員… | 勤務先名, 所属部署 and 役職名 are displayed | | | | Occupations 7–10 hide 勤務先名; option labels unverified |
| 3 | Enter the value in 勤務先名 | `山﨑商事` | No error message is shown under 勤務先名 after moving focus | | | | - |
| 4 | Enter the value in 所属部署 | `営業部` | 所属部署 shows `営業部` | | | | - |
| 5 | Enter the value in 役職名 | `課長` | 役職名 shows `課長` | | | | - |
| 6 | Enter the 取引暗証番号 in the 取引暗証番号を入力してください field | Account's 取引暗証番号 | The 「変更する」 button becomes enabled | | | | The PIN is entered on the page; there is no popup or 業種 field |
| 7 | Tap 「変更する」 | - | /l2203 「変更が完了しました」 is displayed; reopening /l2216 shows 勤務先名 `山﨑商事` | | | | Local dev build: needs the X-CT header (see Notes). Restore the original value afterwards |

### TC_INPUTCHK_007

- **Test Case ID:** TC_INPUTCHK_007
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_02 (proposed)
- **Test Title:** Accept Ext-B kanji 𠮷 in 勤務先名 on l2216 without an error
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. The 職業 card on /l2200 is not 「変更中」
  3. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 勤務先名 `𠮷田商事` (𠮷 U+20BB7)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 職業 card | - | /l2216 is displayed | | | | - |
| 2 | Select an occupation that shows 勤務先名 in 職業 | e.g. company employee | 勤務先名 is displayed | | | | Option labels unverified |
| 3 | Enter the value in 勤務先名 | `𠮷田商事` | 勤務先名 shows `𠮷田商事` | | | | Copy 𠮷 from this document |
| 4 | Tap the 所属部署 field | - | No error message (「特殊文字は入力できません。」) is shown under 勤務先名 | | | | Do not submit; saving is TC_INPUTCHK_014 |

### TC_INPUTCHK_008

- **Test Case ID:** TC_INPUTCHK_008
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_06 (proposed)
- **Test Title:** Pass the identity check on l2500 with a registered name containing 﨑
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. The account's registered name is `山﨑 太郎` (local test data)
  3. The account's registered 生年月日 and 電話番号 are known
  4. Tester is on マイページ → 設定
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account registered as `山﨑 太郎`; its 生年月日 and 電話番号

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「取引暗証番号をお忘れの方はこちら」 | - | /l2500 取引暗証番号の再設定 is displayed | | | | - |
| 2 | Enter the value in 姓 | `山﨑` | No error message is shown under お名前（全角） | | | | - |
| 3 | Enter the value in 名 | `太郎` | No error message is shown under お名前（全角） | | | | - |
| 4 | Enter the registered value in 生年月日 | Registered 生年月日 (e.g. `19880101` format) | 生年月日 shows the value | | | | - |
| 5 | Enter the registered value in 電話番号 | Registered 電話番号 | 電話番号 shows the value | | | | - |
| 6 | Tap 「認証メール送信」 | - | /l2501 (2段階認証) is displayed | | | | Requires local test data with 﨑 in NAME |

### TC_INPUTCHK_009

- **Test Case ID:** TC_INPUTCHK_009
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_04 (proposed)
- **Test Title:** Show the SJIS-win error for Ext-A kanji 㐂 in 姓 on l2212
- **Priority:** High
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. The お名前 card on /l2200 is not 「変更中」
  4. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 姓 `㐂` (U+3402)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the お名前 card | - | /l2212 is displayed | | | | - |
| 2 | Select 「情報の変更」 | - | お名前（全角） 姓 and 名 fields are displayed | | | | - |
| 3 | Enter the value in the 姓 field of お名前（全角） | `㐂` | 姓 shows `㐂` | | | | - |
| 4 | Tap the 名 field | - | 「㐂の入力はできません。ひらがなで入力してください。」 is shown under 姓 | | | | Before the fix: 「姓を正しく入力してください。」 |

### TC_INPUTCHK_010

- **Test Case ID:** TC_INPUTCHK_010
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_04 (proposed)
- **Test Title:** Show the SJIS-win error for Ext-B kanji 𠮷 in 姓 on l2212
- **Priority:** High
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. The お名前 card on /l2200 is not 「変更中」
  4. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 姓 `𠮷田` (𠮷 U+20BB7)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the お名前 card | - | /l2212 is displayed | | | | - |
| 2 | Select 「情報の変更」 | - | お名前（全角） 姓 and 名 fields are displayed | | | | - |
| 3 | Enter the value in the 姓 field of お名前（全角） | `𠮷田` | 姓 shows `𠮷田` | | | | - |
| 4 | Tap the 名 field | - | 「𠮷の入力はできません。ひらがなで入力してください。」 is shown under 姓 | | | | The message does not suggest 吉 (out of scope) |

### TC_INPUTCHK_011

- **Test Case ID:** TC_INPUTCHK_011
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_04 (proposed)
- **Test Title:** Show the SJIS-win error for Ext-B kanji 𠮷 in 法人名（全角） on l2221
- **Priority:** Medium
- **Preconditions:**
  1. Corporate test account is logged in
  2. The 法人名 card on /l2900 is not 「変更中」
  3. Tester is on /l2900
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Corporate test account; 法人名（全角） `株式会社𠮷田`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 法人名 card | - | /l2221 is displayed | | | | - |
| 2 | Select 「情報の変更」 (only if the radio is shown) | - | 法人名（全角） field is editable | | | | Some corporate accounts show no radio; skip this step then |
| 3 | Enter the value in 法人名（全角） | `株式会社𠮷田` | 法人名（全角） shows `株式会社𠮷田` | | | | - |
| 4 | Tap the 法人名（全角カタカナ） field | - | 「𠮷の入力はできません。ひらがなで入力してください。」 is shown under 法人名（全角） | | | | Before the fix: 「法人名を全角で入力してください。」 |

### TC_INPUTCHK_012

- **Test Case ID:** TC_INPUTCHK_012
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_03 (proposed)
- **Test Title:** Remove an emoji from 姓 on l2212 on blur without showing an error
- **Priority:** High
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. The お名前 card on /l2200 is not 「変更中」
  4. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 姓 `山田😀` (😀 U+1F600)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the お名前 card | - | /l2212 is displayed | | | | - |
| 2 | Select 「情報の変更」 | - | お名前（全角） 姓 and 名 fields are displayed | | | | - |
| 3 | Paste the value into the 姓 field of お名前（全角） | `山田😀` | 姓 shows `山田😀` | | | | Typing the emoji is blocked by the keypress handler |
| 4 | Tap the 名 field | - | The emoji is removed (姓 shows `山田`) and no error message is shown under 姓 | | | | Existing behaviour (removeEmojiNSpace) |

### TC_INPUTCHK_013

- **Test Case ID:** TC_INPUTCHK_013
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_03 (proposed)
- **Test Title:** Reject a kanji with an IVS selector in 勤務先名 on l2216
- **Priority:** Low
- **Preconditions:**
  1. Individual test account is logged in
  2. The 職業 card on /l2200 is not 「変更中」
  3. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 勤務先名 `葛󠄀城商事` (葛 followed by IVS selector U+E0100)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 職業 card | - | /l2216 is displayed | | | | - |
| 2 | Select an occupation that shows 勤務先名 in 職業 | e.g. company employee | 勤務先名 is displayed | | | | Option labels unverified |
| 3 | Paste the value into 勤務先名 | `葛󠄀城商事` | 勤務先名 shows `葛󠄀城商事` | | | | The selector is invisible; copy the value from this document |
| 4 | Tap the 所属部署 field | - | 「特殊文字は入力できません。」 is shown under 勤務先名 | | | | IVS is out of scope and stays rejected |

### TC_INPUTCHK_014

- **Test Case ID:** TC_INPUTCHK_014
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_05 (proposed)
- **Test Title:** Save 勤務先名 containing Ext-B kanji 𠮷 on l2216
- **Priority:** High
- **Preconditions:**
  1. Individual test account is logged in
  2. The account has an occupation registered (/l2216 is blank when OCCUPATION_TYPE is null)
  3. The 職業 card on /l2200 is not 「変更中」
  4. Tester is on /l2200
  5. Tester knows the account's 取引暗証番号
  6. DevTools Network tab is open
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account and its 取引暗証番号; 勤務先名 `𠮷田商事`, 所属部署 `営業部`, 役職名 `課長`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 職業 card | - | /l2216 is displayed | | | | - |
| 2 | Confirm 職業 is an occupation that shows 勤務先名 | e.g. 非上場企業の役員・社員… | 勤務先名, 所属部署 and 役職名 are displayed | | | | Option labels unverified |
| 3 | Enter the value in 勤務先名 | `𠮷田商事` | No error message is shown under 勤務先名 after moving focus | | | | - |
| 4 | Enter the value in 所属部署 | `営業部` | 所属部署 shows `営業部` | | | | - |
| 5 | Enter the value in 役職名 | `課長` | 役職名 shows `課長` | | | | - |
| 6 | Enter the 取引暗証番号 in the 取引暗証番号を入力してください field | Account's 取引暗証番号 | The 「変更する」 button becomes enabled | | | | The PIN is entered on the page; there is no popup or 業種 field |
| 7 | Tap 「変更する」 | - | `/user/job` returns 200; /l2203 「変更が完了しました」 is displayed; reopening /l2216 shows 勤務先名 `𠮷田商事` | | | | Local result: 500 with `General error: 3988` (utf8mb3 columns) and /z1200. Record the status code and response, then decide per ADR 0002 |

### TC_INPUTCHK_015

- **Test Case ID:** TC_INPUTCHK_015
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_06 (proposed)
- **Test Title:** Reject the identity check on l2500 when 﨑 is entered but 崎 is registered
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. The account's registered name is `山崎 太郎`
  3. The account's registered 生年月日 and 電話番号 are known
  4. Tester is on マイページ → 設定
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account registered as `山崎 太郎`; 姓 `山﨑`, 名 `太郎`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「取引暗証番号をお忘れの方はこちら」 | - | /l2500 取引暗証番号の再設定 is displayed | | | | - |
| 2 | Enter the value in 姓 | `山﨑` | No error message is shown under お名前（全角） | | | | Before the fix: 「お名前を正しく入力してください。」 |
| 3 | Enter the value in 名 | `太郎` | No error message is shown under お名前（全角） | | | | - |
| 4 | Enter the registered value in 生年月日 | Registered 生年月日 | 生年月日 shows the value | | | | - |
| 5 | Enter the registered value in 電話番号 | Registered 電話番号 | 電話番号 shows the value | | | | - |
| 6 | Tap 「認証メール送信」 | - | /l2501 is not displayed; an error saying the entered information does not match is shown | | | | Accepted behaviour change (ADR 0002). Exact server message unverified; record it |

### TC_INPUTCHK_016

- **Test Case ID:** TC_INPUTCHK_016
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_01 (proposed)
- **Test Title:** Accept the first Ext-A code point U+3400 in 勤務先名 on l2216
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. The 職業 card on /l2200 is not 「変更中」
  3. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 勤務先名 `㐀` (U+3400)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 職業 card | - | /l2216 is displayed | | | | - |
| 2 | Select an occupation that shows 勤務先名 in 職業 | e.g. company employee | 勤務先名 is displayed | | | | Option labels unverified |
| 3 | Paste the value into 勤務先名 | `㐀` | 勤務先名 shows `㐀` | | | | Copy from this document |
| 4 | Tap the 所属部署 field | - | No error message is shown under 勤務先名 | | | | l2216 has no SJIS-win check |

### TC_INPUTCHK_017

- **Test Case ID:** TC_INPUTCHK_017
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_01 (proposed)
- **Test Title:** Accept the first compatibility code point U+F900 in 勤務先名 on l2216
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. The 職業 card on /l2200 is not 「変更中」
  3. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 勤務先名 = U+F900 (CJK COMPATIBILITY IDEOGRAPH-F900), created in the DevTools Console

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 職業 card | - | /l2216 is displayed | | | | - |
| 2 | Select an occupation that shows 勤務先名 in 職業 | e.g. company employee | 勤務先名 is displayed | | | | Option labels unverified |
| 3 | Run the command in the DevTools Console | `copy(String.fromCodePoint(0xF900))` | The character is copied to the clipboard | | | | Do not copy U+F900 from a document (NFC turns it into U+8C48) |
| 4 | Paste the clipboard into 勤務先名 | Clipboard (U+F900) | 勤務先名 shows one kanji | | | | Optional: `document.activeElement.value.codePointAt(0).toString(16)` returns `f900` |
| 5 | Tap the 所属部署 field | - | No error message is shown under 勤務先名 | | | | - |

### TC_INPUTCHK_018

- **Test Case ID:** TC_INPUTCHK_018
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_03 (proposed)
- **Test Title:** Reject U+4DC0, just after Ext-A, in 勤務先名 on l2216
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. The 職業 card on /l2200 is not 「変更中」
  3. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 勤務先名 `䷀` (U+4DC0)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 職業 card | - | /l2216 is displayed | | | | - |
| 2 | Select an occupation that shows 勤務先名 in 職業 | e.g. company employee | 勤務先名 is displayed | | | | Option labels unverified |
| 3 | Paste the value into 勤務先名 | `䷀` | 勤務先名 shows `䷀` | | | | Copy from this document |
| 4 | Tap the 所属部署 field | - | 「特殊文字は入力できません。」 is shown under 勤務先名 | | | | Hexagram symbol, not a kanji |

### TC_INPUTCHK_019

- **Test Case ID:** TC_INPUTCHK_019
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_02 (proposed)
- **Test Title:** Accept the first Ext-B code point U+20000 in 勤務先名 on l2216
- **Priority:** Low
- **Preconditions:**
  1. Individual test account is logged in
  2. The 職業 card on /l2200 is not 「変更中」
  3. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 勤務先名 `𠀀` (U+20000)

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the 職業 card | - | /l2216 is displayed | | | | - |
| 2 | Select an occupation that shows 勤務先名 in 職業 | e.g. company employee | 勤務先名 is displayed | | | | Option labels unverified |
| 3 | Paste the value into 勤務先名 | `𠀀` | 勤務先名 shows `𠀀` | | | | Copy from this document |
| 4 | Tap the 所属部署 field | - | No error message is shown under 勤務先名 | | | | Do not submit (see TC_INPUTCHK_014) |

### TC_INPUTCHK_020

- **Test Case ID:** TC_INPUTCHK_020
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_03 (proposed)
- **Test Title:** Reject HTML markup in 姓 on l2212 with the format error
- **Priority:** High
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. The お名前 card on /l2200 is not 「変更中」
  4. Tester is on /l2200
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 姓 `<script>alert(1)</script>`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「変更」 on the お名前 card | - | /l2212 is displayed | | | | - |
| 2 | Select 「情報の変更」 | - | お名前（全角） 姓 and 名 fields are displayed | | | | - |
| 3 | Enter the value in the 姓 field of お名前（全角） | `<script>alert(1)</script>` | 姓 shows the text as typed | | | | Half-width input may be converted to full-width by the field (unverified) |
| 4 | Tap the 名 field | - | 「姓を正しく入力してください。」 is shown under 姓; no alert dialog appears | | | | Regression: widening the ranges must not allow symbols |

### TC_INPUTCHK_021

- **Test Case ID:** TC_INPUTCHK_021
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_07 (proposed)
- **Test Title:** Redirect to /l2100 when /l2212 is opened directly by URL
- **Priority:** Low
- **Preconditions:**
  1. Individual test account is logged in
  2. 取引暗証番号 has not been entered in this session
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Enter the URL in the address bar | `http://localhost:8080/#/l2212` | The URL is entered | | | | URL form (hash vs history mode) is assumed |
| 2 | Press Enter | - | /l2100 設定 is displayed instead of /l2212 | | | | Existing behaviour, not changed by this ticket |

### TC_INPUTCHK_022

- **Test Case ID:** TC_INPUTCHK_022
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_03 (proposed)
- **Test Title:** Keep the send button disabled while 法人名 contains an emoji on y1204
- **Priority:** Medium
- **Preconditions:**
  1. User is not logged in
  2. Tester is on the login screen
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** 法人名 `株式会社😀`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「ログインIDを忘れた方はこちら」 | - | ログインID（メールアドレス）の再設定 is displayed | | | | - |
| 2 | Tap 「法人のお客様」 | - | 法人名 and 取引責任者のお名前 fields are displayed (/y1204) | | | | - |
| 3 | Enter the value in 法人名 | `株式会社😀` | 法人名 shows `株式会社😀` | | | | - |
| 4 | Tap the 姓 field of 取引責任者のお名前 | - | An error message is shown under 法人名; the 「認証メール送信」 button is disabled | | | | Whether the button reads 「認証メール送信」 or 「認証コード送信」 on this tab is unverified. Expected text 「法人名を正しく入力してください。」 is from the code, unverified on screen |

### TC_INPUTCHK_023

- **Test Case ID:** TC_INPUTCHK_023
- **Module:** InputCheck
- **Req ID:** REQ_INPUTCHK_03 (proposed)
- **Test Title:** Hide the お名前 error on l2500 when the 姓 field gets focus again
- **Priority:** Low
- **Preconditions:**
  1. Individual test account is logged in
  2. Tester is on マイページ → 設定
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4556` @ `4df5005a`
- **Test Data:** Individual test account; 姓 `山田😀`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Tap 「取引暗証番号をお忘れの方はこちら」 | - | /l2500 取引暗証番号の再設定 is displayed | | | | - |
| 2 | Enter the value in 姓 | `山田😀` | 姓 shows `山田😀` | | | | - |
| 3 | Tap the 「お名前（全角）」 label (outside the input fields) | - | 「お名前を正しく入力してください。」 is shown under お名前（全角） | | | | Tapping 名 would clear the error immediately |
| 4 | Tap the 姓 field | - | The error message is hidden | | | | - |
