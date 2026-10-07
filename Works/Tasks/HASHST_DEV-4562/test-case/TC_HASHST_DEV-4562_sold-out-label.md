# Test Cases: Sold-out label without 「当初」 (HASHST_DEV-4562)

- **Function:** The 募集状況 label from `getInStockStr` shows 「募集終了」 instead of 「当初募集終了」 when a fund in 当初募集 is sold out (remaining units 0 and remaining ratio 0)
- **Module:** InStockLabel (app-vuejs, f1200 ST詳細：トップ `/f1200?id=<FUND_CD>`, f1202_pre 仮募集 ST詳細：トップ `/f1202_pre?id=<FUND_CD>`; data from st-api `/fund/detail`)
- **Environment:** Local DEV (local-docker), app-vuejs `backlog/HASHST_DEV-4562` @ `cefc6cf2`, `npm run dev`, fund API http://local-st-front-api-hsec.crudist.tech (st-api), DB `hhd_pfund`
- **Executed (2026-10-06):** Playwright headless Chromium with iPhone 13 emulation, on http://local-st-front-hsec.crudist.tech:8080. That is the origin st-api's CORS allows (`ST_FRONT_VUEJS_BASE_URL`); on http://localhost:8080 every st-api call returns an empty 200 and the app shows /z1200. The browser was registered as a trusted device of the test account through /m2200 (email code), because /fund/detail returns `xdu_credible_error` for an untrusted device.
- **Known business rules (from HASHST_DEV-4562 / CONTEXT.md / st-api `FundHelper.php`):**
  - Rule 1: When `PERIOD` is `PRIMARY` or `PRIENTRY` and both the remaining ratio (`IN_STOCK_POSITION_EMPTY`) and the remaining units (`IN_STOCK_NUM`) are 0, the label is 「募集終了」 (ticket).
  - Rule 2: 「募集終了」 means sold out, not "offering period over" (ticket, CONTEXT.md). It can only appear while `PERIOD` is `PRIMARY` / `PRIENTRY`. After 当初募集終了日時 (`FIRST_ORDER_END_DT`), `PERIOD` becomes `PRE_SECONDARY` and f1200 shows 在庫状況 instead (st-api).
  - Rule 3: `PRIENTRY` (仮募集) shares the label, with no separate branch. 仮募集 is an unused feature (ticket).
  - Rule 4: `IN_STOCK_NUM` = `PST_FUND.MAX_FUND_QTY` − SUM of `PST_FUND_APPRICATION.COUNT` where `INFO_TYPE = 1`. The ratio is `IN_STOCK_NUM / MAX_FUND_QTY` rounded **up** to 2 decimals (st-api).
  - Rule 5: The other labels are unchanged (ticket, code):
    - 当初募集 with remaining units > 0 and ratio < 20%: 「売切れ間近」 + line break + 「N口　残R%」
    - 当初募集 otherwise: 「申込受付中」
    - `SECONDARY` / `PRE_SECONDARY`: 「在庫なし　0口」 / 「在庫あり　　N口」

**Proposed requirement IDs**

| Req ID | Requirement |
|---|---|
| REQ_INSTOCK_01 (proposed) | A sold-out `PRIMARY` fund shows 「募集終了」 in 募集状況·期間 on f1200 |
| REQ_INSTOCK_02 (proposed) | A sold-out `PRIENTRY` fund shows 「募集終了」 in 募集状況・期間 on f1202_pre |
| REQ_INSTOCK_03 (proposed) | 当初募集 funds that are not sold out keep their existing labels (regression) |
| REQ_INSTOCK_04 (proposed) | After 当初募集終了日時, f1200 shows 在庫状況 instead of the 募集状況 label (regression) |
| REQ_INSTOCK_05 (proposed) | The label markup passes through `$sanitize`: the allowed `<br>` renders as a line break, not as text |

**Shared test data setup (all test cases)**

- Use one test fund `<FUND_CD>` in `hhd_pfund.PST_FUND` with `OPEN_FLG = '1'`, `FAILED_FLG = '0'` and `MAX_FUND_QTY = 100`. `/fund/detail` returns nothing for funds that don't match `OPEN_FLG` / `FAILED_FLG`.
- Set the remaining units with:
  `INSERT INTO hhd_pfund.PST_FUND_APPRICATION (FUND_CD, INFO_TYPE, COUNT, LAST_MOD_USER) VALUES ('<FUND_CD>', 1, <COUNT>, 'TEST') ON DUPLICATE KEY UPDATE COUNT = <COUNT>;`
  Remaining units = 100 − `<COUNT>`.
- Before changing anything, record the original values: `SELECT * FROM hhd_pfund.PST_FUND_APPRICATION WHERE FUND_CD = '<FUND_CD>';` and the date and `MAX_FUND_QTY` columns of `PST_FUND`. Restore them afterwards.

## 1. Test Case List

| Test Case ID | Module | Req ID | Test Title | Priority | Preconditions | Test Steps | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TC_INSTOCK_001 | InStockLabel | REQ_INSTOCK_01 (proposed) | Show 「募集終了」 (not 「当初募集終了」) in 募集状況·期間 on f1200 for a sold-out PRIMARY fund | High | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`<br>4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)<br>5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 100` (0 units remaining) | 1. Navigate to `/f1200?id=<FUND_CD>`<br>2. Check the tag at the top of the fund<br>3. Scroll to 「募集状況·期間」 | f1200 is displayed for `<FUND_CD>`<br>The tag 「売切御礼」 is shown<br>「募集状況·期間」 shows 「募集終了」<br>「当初募集終了」 is not shown anywhere on the screen | (Fund JP8010124071, MAX_FUND_QTY 100, COUNT already 100. Run after setting FIRST_ORDER_START_DT / FIRST_ORDER_END_DT to 2026-10-01 10:00 / 2026-10-31 16:00 and ORDER_START_DT / ORDER_END_DT to 2026-11-02 18:00 / 2027-11-01 16:00 in the local DB; reverted afterwards. With only the offering dates changed, /fund/detail returned E000-0000 「予期せぬエラーが発生しました。」 because the original ORDER_START_DT was in the past and st-api looked up a secondary price, so the operation dates were moved too.) f1200 was displayed. The tags 「売切御礼」「任意組合」 were shown. 「募集状況·期間」 showed 「募集終了」 with 「2026/10/01～2026/10/31」 below it; 「当初募集終了」 did not appear anywhere on the page. /fund/detail returned PERIOD `PRIMARY`, IN_STOCK_NUM 0, IN_STOCK_POSITION_EMPTY 0.<br>Screenshots: `screenshot/TC_INSTOCK_001_step2.png`, `screenshot/TC_INSTOCK_001_step3.png` | | | Before the fix the label was 「当初募集終了」. The 「売切御礼」 tag confirms the data is sold out (st-api sets it when APPRICATION_QTY ≥ MAX_FUND_QTY). The dt label uses 「·」 (U+00B7) on f1200. Opened by URL; normal entry is from the fund list (unverified) |
| TC_INSTOCK_002 | InStockLabel | REQ_INSTOCK_02 (proposed) | Show 「募集終了」 in 募集状況・期間 on f1202_pre for a sold-out PRIENTRY fund | Medium | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`<br>4. Now is between `PRI_ENTRY_START_DT` and `PRE_ORDER_END_DT` (PERIOD = PRIENTRY)<br>5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 100` (0 units remaining) | 1. Navigate to `/f1202_pre?id=<FUND_CD>`<br>2. Scroll to 「募集状況・期間」 | f1202_pre is displayed for `<FUND_CD>`<br>「募集状況・期間」 shows 「募集終了」 | (Used the existing local fund JP9010124091, which is already PRIENTRY (PRI_ENTRY_START_DT 2026-09-01 – PRE_ORDER_END_DT 2026-10-15) and sold out (MAX_FUND_QTY 1000, applied 1000); no data change.) f1202_pre was displayed. 「募集状況・期間」 showed 「募集終了」. The period below it showed 「2026/10/01～2024/09/24」 (PRE_ORDER_START_DT～FIRST_ORDER_END_DT of this local fund, whose test dates are out of order). /fund/detail returned PERIOD `PRIENTRY`, IN_STOCK_NUM 0, IN_STOCK_POSITION_EMPTY 0. Tags: 「募集中」「匿名組合」.<br>Screenshots: `screenshot/TC_INSTOCK_002_step1.png`, `screenshot/TC_INSTOCK_002_step2.png` | | | 仮募集 is an unused feature; the period dates must be set up locally to reach PRIENTRY. The dt label uses 「・」 (U+30FB) on f1202_pre. The 「売切御礼」 tag is not set in the PRIENTRY window, so it is not checked here |
| TC_INSTOCK_003 | InStockLabel | REQ_INSTOCK_03 (proposed) | Show 「売切れ間近」 and 「1口　残1%」 on f1200 when 1 of 100 units remains (just above sold out) | Medium | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`<br>4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)<br>5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 99` (1 unit remaining) | 1. Navigate to `/f1200?id=<FUND_CD>`<br>2. Scroll to 「募集状況·期間」 | f1200 is displayed for `<FUND_CD>`<br>「募集状況·期間」 shows 「売切れ間近」 on the first line and 「1口　残1%」 on the second line<br>「募集終了」 is not shown | (Fund JP8010124071, MAX_FUND_QTY 100. Run after setting FIRST_ORDER_START_DT / FIRST_ORDER_END_DT to 2026-10-01 10:00 / 2026-10-31 16:00 and ORDER_START_DT / ORDER_END_DT to 2026-11-02 18:00 / 2027-11-01 16:00 in the local DB, and PST_FUND_APPRICATION INFO_TYPE 1 COUNT to 99; reverted afterwards.) f1200 was displayed. 「募集状況·期間」 showed 「売切れ間近」 on the first line and 「1口　残1%」 on the second line; 「募集終了」 was not shown. /fund/detail returned IN_STOCK_NUM 1, IN_STOCK_POSITION_EMPTY 0.01. Tag: 「募集中」.<br>Screenshots: `screenshot/TC_INSTOCK_003_step1.png`, `screenshot/TC_INSTOCK_003_step2.png` | | | The space in 「1口　残1%」 is a full-width space (U+3000). st-api rounds the ratio up, so 「残0%」 with units remaining cannot come from real data |
| TC_INSTOCK_004 | InStockLabel | REQ_INSTOCK_03 (proposed) | Show 「売切れ間近」 and 「19口　残19%」 on f1200 when 19 of 100 units remain (just below 20%) | Low | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`<br>4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)<br>5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 81` (19 units remaining) | 1. Navigate to `/f1200?id=<FUND_CD>`<br>2. Scroll to 「募集状況·期間」 | f1200 is displayed for `<FUND_CD>`<br>「募集状況·期間」 shows 「売切れ間近」 on the first line and 「19口　残19%」 on the second line | (Fund JP8010124071, MAX_FUND_QTY 100. Run after setting FIRST_ORDER_START_DT / FIRST_ORDER_END_DT to 2026-10-01 10:00 / 2026-10-31 16:00 and ORDER_START_DT / ORDER_END_DT to 2026-11-02 18:00 / 2027-11-01 16:00 in the local DB, and PST_FUND_APPRICATION INFO_TYPE 1 COUNT to 81; reverted afterwards.) f1200 was displayed. 「募集状況·期間」 showed 「売切れ間近」 on the first line and 「19口　残19%」 on the second line. /fund/detail returned IN_STOCK_NUM 19, IN_STOCK_POSITION_EMPTY 0.19.<br>Screenshots: `screenshot/TC_INSTOCK_004_step1.png`, `screenshot/TC_INSTOCK_004_step2.png` | | | Existing behaviour, not changed by this ticket |
| TC_INSTOCK_005 | InStockLabel | REQ_INSTOCK_03 (proposed) | Show 「申込受付中」 on f1200 when 20 of 100 units remain (at the 20% threshold) | Low | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`<br>4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)<br>5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 80` (20 units remaining) | 1. Navigate to `/f1200?id=<FUND_CD>`<br>2. Scroll to 「募集状況·期間」 | f1200 is displayed for `<FUND_CD>`<br>「募集状況·期間」 shows 「申込受付中」 | (Fund JP8010124071, MAX_FUND_QTY 100. Run after setting FIRST_ORDER_START_DT / FIRST_ORDER_END_DT to 2026-10-01 10:00 / 2026-10-31 16:00 and ORDER_START_DT / ORDER_END_DT to 2026-11-02 18:00 / 2027-11-01 16:00 in the local DB, and PST_FUND_APPRICATION INFO_TYPE 1 COUNT to 80; reverted afterwards.) f1200 was displayed. 「募集状況·期間」 showed 「申込受付中」. /fund/detail returned IN_STOCK_NUM 20, IN_STOCK_POSITION_EMPTY 0.20.<br>Screenshots: `screenshot/TC_INSTOCK_005_step1.png`, `screenshot/TC_INSTOCK_005_step2.png` | | | Existing behaviour, not changed by this ticket. Because st-api rounds the ratio up, 191 of 1000 (19.1%) is sent as 0.20 and also shows 「申込受付中」 (from the code, unverified on screen) |
| TC_INSTOCK_006 | InStockLabel | REQ_INSTOCK_04 (proposed) | Show 「在庫なし　0口」 in 在庫状況 on f1200, not 「募集終了」, after 当初募集終了日時 has passed | Low | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`<br>4. Now is after `FIRST_ORDER_END_DT` and before `ORDER_START_DT` (PERIOD = PRE_SECONDARY)<br>5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 100` (0 units remaining)<br>6. The test account does not hold `<FUND_CD>` | 1. Navigate to `/f1200?id=<FUND_CD>`<br>2. Scroll to 「在庫状況」 | f1200 is displayed for `<FUND_CD>`<br>「募集状況·期間」 is not shown<br>「在庫状況」 shows 「在庫なし　0口」<br>「募集終了」 is not shown | (Fund JP8010124071. Run after setting FIRST_ORDER_START_DT 2026-09-01 10:00, FIRST_ORDER_END_DT 2026-10-05 16:00 and ORDER_START_DT 2026-10-31 18:00 in the local DB, with COUNT 100; reverted afterwards. The test account does not hold this fund.) f1200 was displayed without errors; 「募集状況·期間」 was not shown. 「在庫状況」 showed 「在庫なし　0口」, and 「募集終了」 did not appear anywhere on the page. /fund/detail returned PERIOD `PRE_SECONDARY`, SELF_FUND_QTY_SETTLEMENT 0. Tags: 「在庫なし」「任意組合」.<br>Screenshots: `screenshot/TC_INSTOCK_006_step1.png`, `screenshot/TC_INSTOCK_006_step2.png` | | | Shows that 「募集終了」 (sold out) and 当初募集終了日時 (period end) are different (CONTEXT.md). st-api sets SELF_FUND_QTY_SETTLEMENT to 0 in PRE_SECONDARY. Other fields of this block need price data; if the screen fails to load, record it (unverified) |
| TC_INSTOCK_007 | InStockLabel | REQ_INSTOCK_05 (proposed) | Render the 「売切れ間近」 line break on f1200 as a `<br>` element, not as the text `<br/>` | Low | 1. Individual test account is logged in<br>2. Browser is in smartphone device emulation<br>3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`<br>4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)<br>5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 99` (1 unit remaining)<br>6. DevTools Elements panel is available | 1. Navigate to `/f1200?id=<FUND_CD>`<br>2. Scroll to 「募集状況·期間」<br>3. Right-click the 「売切れ間近」 text and select Inspect | The text `<br/>` is not visible on the screen<br>The label's `<span>` contains 「売切れ間近」, a `<br>` element, and 「1口　残1%」 | (Fund JP8010124071, MAX_FUND_QTY 100. Run after setting FIRST_ORDER_START_DT / FIRST_ORDER_END_DT to 2026-10-01 10:00 / 2026-10-31 16:00 and ORDER_START_DT / ORDER_END_DT to 2026-11-02 18:00 / 2027-11-01 16:00 in the local DB, and PST_FUND_APPRICATION INFO_TYPE 1 COUNT to 99; reverted afterwards.) 「売切れ間近」 and 「1口　残1%」 were shown on separate lines, and the text `<br/>` was not visible. Inspecting the label gave `<span data-v-24d4bfd9="">売切れ間近<br>1口　残1%</span>`. In the step 3 screenshot, the red outline around the span was added by the test harness.<br>Screenshots: `screenshot/TC_INSTOCK_007_step1.png`, `screenshot/TC_INSTOCK_007_step2.png`, `screenshot/TC_INSTOCK_007_step3.png` | | | The label is set with `v-html` through `$sanitize`; this checks that the sanitizer still lets `<br>` through. The label text is fixed by this ticket, so no user input reaches it. Same data as TC_INSTOCK_003 |

## 2. Test Case Details

### TC_INSTOCK_001

- **Test Case ID:** TC_INSTOCK_001
- **Module:** InStockLabel
- **Req ID:** REQ_INSTOCK_01 (proposed)
- **Test Title:** Show 「募集終了」 (not 「当初募集終了」) in 募集状況·期間 on f1200 for a sold-out PRIMARY fund
- **Priority:** High
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`
  4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)
  5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 100` (0 units remaining)
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4562` @ `cefc6cf2`, st-api local
- **Test Data:** Individual test account (made-up data); fund `<FUND_CD>`, `MAX_FUND_QTY = 100`, application `COUNT = 100`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the f1200 URL | `/f1200?id=<FUND_CD>` | f1200 is displayed for `<FUND_CD>` | | | | Opened by URL; normal entry is from the fund list (unverified) |
| 2 | Check the tag at the top of the fund | - | The tag 「売切御礼」 is shown | | | | Confirms the data is sold out |
| 3 | Scroll to 「募集状況·期間」 | - | 「募集状況·期間」 shows 「募集終了」; 「当初募集終了」 is not shown anywhere on the screen | | | | Before the fix the label was 「当初募集終了」. The dt label uses 「·」 (U+00B7) |

### TC_INSTOCK_002

- **Test Case ID:** TC_INSTOCK_002
- **Module:** InStockLabel
- **Req ID:** REQ_INSTOCK_02 (proposed)
- **Test Title:** Show 「募集終了」 in 募集状況・期間 on f1202_pre for a sold-out PRIENTRY fund
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`
  4. Now is between `PRI_ENTRY_START_DT` and `PRE_ORDER_END_DT` (PERIOD = PRIENTRY)
  5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 100` (0 units remaining)
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4562` @ `cefc6cf2`, st-api local
- **Test Data:** Individual test account (made-up data); fund `<FUND_CD>`, `MAX_FUND_QTY = 100`, application `COUNT = 100`, `PRI_ENTRY_START_DT` in the past and `PRE_ORDER_END_DT` in the future

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the f1202_pre URL | `/f1202_pre?id=<FUND_CD>` | f1202_pre is displayed for `<FUND_CD>` | | | | 仮募集 is unused; the period dates must be set up locally |
| 2 | Scroll to 「募集状況・期間」 | - | 「募集状況・期間」 shows 「募集終了」 | | | | The dt label uses 「・」 (U+30FB). The 「売切御礼」 tag is not set in the PRIENTRY window |

### TC_INSTOCK_003

- **Test Case ID:** TC_INSTOCK_003
- **Module:** InStockLabel
- **Req ID:** REQ_INSTOCK_03 (proposed)
- **Test Title:** Show 「売切れ間近」 and 「1口　残1%」 on f1200 when 1 of 100 units remains (just above sold out)
- **Priority:** Medium
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`
  4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)
  5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 99` (1 unit remaining)
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4562` @ `cefc6cf2`, st-api local
- **Test Data:** Individual test account (made-up data); fund `<FUND_CD>`, `MAX_FUND_QTY = 100`, application `COUNT = 99`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the f1200 URL | `/f1200?id=<FUND_CD>` | f1200 is displayed for `<FUND_CD>` | | | | - |
| 2 | Scroll to 「募集状況·期間」 | - | 「募集状況·期間」 shows 「売切れ間近」 on the first line and 「1口　残1%」 on the second line; 「募集終了」 is not shown | | | | Full-width space (U+3000) in 「1口　残1%」. st-api rounds 1/100 up to 0.01, so the ratio is 1%, not 0% |

### TC_INSTOCK_004

- **Test Case ID:** TC_INSTOCK_004
- **Module:** InStockLabel
- **Req ID:** REQ_INSTOCK_03 (proposed)
- **Test Title:** Show 「売切れ間近」 and 「19口　残19%」 on f1200 when 19 of 100 units remain (just below 20%)
- **Priority:** Low
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`
  4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)
  5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 81` (19 units remaining)
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4562` @ `cefc6cf2`, st-api local
- **Test Data:** Individual test account (made-up data); fund `<FUND_CD>`, `MAX_FUND_QTY = 100`, application `COUNT = 81`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the f1200 URL | `/f1200?id=<FUND_CD>` | f1200 is displayed for `<FUND_CD>` | | | | - |
| 2 | Scroll to 「募集状況·期間」 | - | 「募集状況·期間」 shows 「売切れ間近」 on the first line and 「19口　残19%」 on the second line | | | | Existing behaviour, not changed by this ticket |

### TC_INSTOCK_005

- **Test Case ID:** TC_INSTOCK_005
- **Module:** InStockLabel
- **Req ID:** REQ_INSTOCK_03 (proposed)
- **Test Title:** Show 「申込受付中」 on f1200 when 20 of 100 units remain (at the 20% threshold)
- **Priority:** Low
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`
  4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)
  5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 80` (20 units remaining)
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4562` @ `cefc6cf2`, st-api local
- **Test Data:** Individual test account (made-up data); fund `<FUND_CD>`, `MAX_FUND_QTY = 100`, application `COUNT = 80`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the f1200 URL | `/f1200?id=<FUND_CD>` | f1200 is displayed for `<FUND_CD>` | | | | - |
| 2 | Scroll to 「募集状況·期間」 | - | 「募集状況·期間」 shows 「申込受付中」 | | | | Existing behaviour. 191 of 1000 (19.1%) is rounded up to 0.20 by st-api and also shows 「申込受付中」 (from the code, unverified on screen) |

### TC_INSTOCK_006

- **Test Case ID:** TC_INSTOCK_006
- **Module:** InStockLabel
- **Req ID:** REQ_INSTOCK_04 (proposed)
- **Test Title:** Show 「在庫なし　0口」 in 在庫状況 on f1200, not 「募集終了」, after 当初募集終了日時 has passed
- **Priority:** Low
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`
  4. Now is after `FIRST_ORDER_END_DT` and before `ORDER_START_DT` (PERIOD = PRE_SECONDARY)
  5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 100` (0 units remaining)
  6. The test account does not hold `<FUND_CD>`
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4562` @ `cefc6cf2`, st-api local
- **Test Data:** Individual test account (made-up data); fund `<FUND_CD>`, `MAX_FUND_QTY = 100`, application `COUNT = 100`, `FIRST_ORDER_END_DT` in the past, `ORDER_START_DT` in the future

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the f1200 URL | `/f1200?id=<FUND_CD>` | f1200 is displayed for `<FUND_CD>`; 「募集状況·期間」 is not shown | | | | Other fields of this block need price data; if the screen fails to load, record it (unverified) |
| 2 | Scroll to 「在庫状況」 | - | 「在庫状況」 shows 「在庫なし　0口」; 「募集終了」 is not shown | | | | Full-width space (U+3000) in 「在庫なし　0口」. Shows that sold out and period end are different (CONTEXT.md) |

### TC_INSTOCK_007

- **Test Case ID:** TC_INSTOCK_007
- **Module:** InStockLabel
- **Req ID:** REQ_INSTOCK_05 (proposed)
- **Test Title:** Render the 「売切れ間近」 line break on f1200 as a `<br>` element, not as the text `<br/>`
- **Priority:** Low
- **Preconditions:**
  1. Individual test account is logged in
  2. Browser is in smartphone device emulation
  3. Test fund `<FUND_CD>` has `MAX_FUND_QTY = 100`, `OPEN_FLG = '1'`, `FAILED_FLG = '0'`
  4. Now is between `FIRST_ORDER_START_DT` and `FIRST_ORDER_END_DT`, and outside `PRI_ENTRY_START_DT`–`PRE_ORDER_END_DT` (PERIOD = PRIMARY)
  5. `PST_FUND_APPRICATION` `INFO_TYPE = 1` `COUNT = 99` (1 unit remaining)
  6. DevTools Elements panel is available
- **Test Environment Info:** Windows 11 + WSL2 (assumed), Chrome latest with DevTools smartphone emulation (assumed), Local DEV (local-docker), http://localhost:8080 (assumed), app-vuejs `backlog/HASHST_DEV-4562` @ `cefc6cf2`, st-api local
- **Test Data:** Individual test account (made-up data); fund `<FUND_CD>`, `MAX_FUND_QTY = 100`, application `COUNT = 99`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the f1200 URL | `/f1200?id=<FUND_CD>` | f1200 is displayed for `<FUND_CD>` | | | | Same data as TC_INSTOCK_003 |
| 2 | Scroll to 「募集状況·期間」 | - | 「売切れ間近」 and 「1口　残1%」 are on separate lines; the text `<br/>` is not visible | | | | - |
| 3 | Right-click the 「売切れ間近」 text and select Inspect | - | The label's `<span>` contains 「売切れ間近」, a `<br>` element, and 「1口　残1%」 | | | | The label is set with `v-html` through `$sanitize` |
