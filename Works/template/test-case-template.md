# TEST CASE STRUCTURE

When the user asks to write test cases for a feature or for the changes of a ticket (e.g., "write test cases for ticket HASHST_DEV-4556", "draft test cases for the login feature"), produce **one Markdown file** with the following parts, in this exact order:

0. **File Header**: title, function under test, module, environment, known business rules, proposed requirement IDs.
1. **Test Case List**: a summary table where each test case occupies one row.
2. **Test Case Details**: each test case in a separate block containing a detailed step-by-step table.

**File name:** `TC_<TICKET>_<slug>.md` (e.g., `TC_HASHST_DEV-4556_input-character-check.md`).
**Screenshots** (only when tests are executed) are saved in a `screenshot/` folder next to the file.
**Language:** descriptions in English. Keep UI labels and messages in their original language (e.g., Japanese), exactly as shown on screen, wrapped in `「」`.

---

## WORKFLOW

### Inputs to read before writing

1. This template.
2. The ticket content (path provided by the user).
3. The implemented changes: `git diff <base-branch>...HEAD`, `git log <base-branch>..HEAD`, and related ADR / `CONTEXT.md` if they exist.
4. The code of the affected screens and functions. Read the real code. Do not guess screen names, UI labels, routes, or messages.

### Phases

| Phase | What Claude does | Gate |
|---|---|---|
| 1. Outline | Output the file header, the proposed requirement IDs, and a planned list (Test Case ID, Req ID, Title, Priority). List open questions. | **Stop and wait for the user's approval.** Do not write details yet. |
| 2. Write | Write the full file (sections 0, 1, 2). Leave Actual Result, Status, Bug ID blank. | - |
| 3. Execute (optional) | Run the test cases on the local/dev environment and fill the results (see section 6). | Only when the user asks and browser/automation tools are available. |

---

## 0. FILE HEADER

### 0.1. Header Format

```markdown
# Test Cases: <short title> (<TICKET>)

- **Function:** <function or behaviour under test>
- **Module:** <module name> (<repo / screens>)
- **Environment:** <environment>, <repo> `<branch>` @ `<short SHA>`, `<run command>`
- **Known business rules (from <TICKET> / <ADR>):**
  - Rule 1: <rule>
  - Rule 2: <rule>

**Proposed requirement IDs**

| Req ID | Requirement |
|---|---|
| REQ_<MODULE>_01 (proposed) | <requirement> |
| REQ_<MODULE>_02 (proposed) | <requirement> |
```

### 0.2. Header Rules

| Item | Rules |
|---|---|
| **Function** | One sentence describing what is under test. Include concrete examples when useful (e.g., sample characters, sample values). |
| **Module** | Readable module name, plus repo and screens in parentheses. |
| **Environment** | Environment name, repo, branch and short commit SHA, and how the app is run. If unknown, mark "(assumed)". |
| **Known business rules** | Numbered `Rule 1..n`. State each rule in testable terms. Cite the source (ticket, ADR, spec). Only rules found in the sources. Do not invent rules. |
| **Proposed requirement IDs** | One row per requirement verified by the test cases. IDs follow `REQ_<MODULE>_<NN>`. If the user did not provide IDs, propose them and mark `(proposed)`. Every Req ID used in the list must appear here. |

`<MODULE>` in IDs is an **uppercase abbreviation** of the module name (e.g., module `InputCheck` → `INPUTCHK`) and stays the same in Test Case IDs and Req IDs.

---

## 1. TEST CASE LIST

### 1.1. Field Descriptions for Summary Row

| No. | Field Name | Description | Fulfillment Rules / Naming Conventions |
|-----|--------|-------|--------------|
| 1 | **Test Case ID** | Unique identifier for the test case | `TC_<MODULE>_<NNN>`, where `NNN` is a 3-digit sequential number (e.g., `TC_INPUTCHK_001`). |
| 2 | **Module** | Module or functional group containing the test case | State the module name clearly (e.g., `InputCheck`, `Authentication`). |
| 3 | **Req ID** | Requirement verified by the test case | `REQ_<MODULE>_<NN> (proposed)` or the ID provided by the user. Must exist in the header table. |
| 4 | **Test Title** | Concise title describing what is verified | Start with an action verb; state the condition, the field, the screen code, and the expected outcome (e.g., "Accept compatibility kanji 﨑 in 姓 on l2212 without an error"). |
| 5 | **Priority** | Importance / severity | Strictly `High` / `Medium` / `Low`. See the guideline in section 4.2. |
| 6 | **Preconditions** | Required system/data state before execution | Numbered list. Include account type and login state, data setup, device/emulation, the screen the tester starts on, and knowledge the tester needs (e.g., a PIN). |
| 7 | **Test Steps** | Actions the tester performs | Numbered `1, 2, 3...`; **one action per step**; concrete input values in backticks (e.g., ``Enter `山﨑` in the 姓 field``). |
| 8 | **Expected Result** | Expected outcome | Specific and verifiable: messages, target screen, field values, data changes. One line per checkpoint, separated by `<br>`. Avoid vague terms like "works properly". |
| 9 | **Actual Result** | Execution outcome | **Blank** when drafting. Filled only after execution, **in the list only** (see section 6). |
| 10 | **Status** | Result of the run | `PASS` / `FAIL`. **Blank** (or `Untested`) unless the user asks to fill it. |
| 11 | **Bug ID** | Related bug if the test fails | **Blank**. Populate only when provided by the user. |
| 12 | **Notes** | Additional information | See section 1.3. Put `-` if none. |

### 1.2. Summary List Output Format

A `12-column Markdown table`, strictly following the order in section 1.1. Multiple items within one cell are separated by `<br>`.

```markdown
## 1. Test Case List

| Test Case ID | Module | Req ID | Test Title | Priority | Preconditions | Test Steps | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TC_<MODULE>_<NNN> | <module name> | REQ_<MODULE>_<NN> (proposed) | <title> | High | 1. <condition 1><br>2. <condition 2> | 1. <step 1><br>2. <step 2><br>3. <step 3> | <expected result 1><br><expected result 2> | | | | <notes or -> |
```

Each row must stay on **one physical line** (no line break inside a row).

### 1.3. What Goes Into Notes

Use Notes to record, briefly and factually:

- **Assumptions and unverified items.** Mark with "unverified" (e.g., "Error text is from the code review, unverified on screen").
- **Behaviour before the fix** when useful (e.g., "Before the fix the message was 「…」").
- **Existing behaviour / regression** (e.g., "Existing behaviour, not changed by this ticket").
- **Input traps:** invisible characters, text that editors normalise, paste-only input, how to generate the value (e.g., DevTools Console command).
- **Test data prerequisites** that need preparation (e.g., "Requires preparing local test data").
- **Side effects and cautions:** "Do not submit (goes to eKYC)", destructive actions, data that must be restored.
- **Dependencies** between test cases (e.g., "Saving is TC_INPUTCHK_014").
- **Scope exclusions** (e.g., "Out of scope for this ticket").
- **Why this screen was chosen** when the same behaviour exists on several screens.

---

## 2. TEST CASE DETAILS

### 2.1. Field Descriptions

| No. | Field Name | Fulfillment Rules |
|-----|--------|--------------|
| 1 | **Test Case ID** | Identical to the corresponding row in the list. |
| 2 | **Module** | Identical to the list. |
| 3 | **Req ID** | Identical to the list. |
| 4 | **Test Title** | Identical to the list. |
| 5 | **Priority** | Identical to the list. |
| 6 | **Preconditions** | Identical to the list, as a numbered list. |
| 7 | **Test Environment Info** | OS, browser/device or emulation, environment (DEV/STG/UAT/local), URL, repo `branch` @ `short SHA`. Mark "(assumed)" for anything unknown. |
| 8 | **Test Data** | Shared accounts and input values used by the test case. For special characters, give the code point (e.g., `山﨑` (﨑 U+FA11)). Say "made-up data" for fabricated personal data. Step-specific data goes into the "Input Data" column. |
| 9 | **Execution Details Table** | One row per step, see section 2.2. |

### 2.2. Detailed Table Columns

| Column Name | Fulfillment Rules |
|-----|--------------|
| **Step #** | Sequential number starting from 1. |
| **Step Action** | One action per step, starting with an action verb (`Tap` on mobile, `Click` on desktop, `Enter`, `Select`, `Navigate`). Use the exact UI labels in `「」`. |
| **Input Data** | Specific value entered or selected at this step, in backticks. `-` if none. |
| **Expected Result** | Specific, verifiable outcome for **this step** (e.g., "Address fields are displayed"). The final step carries the final outcome. |
| **Actual Result** | **Leave blank.** Actual results are recorded in the list only (section 6). |
| **Status** | **Leave blank.** |
| **Bug ID** | **Leave blank.** |
| **Notes** | Step-specific information or assumptions. `-` if none. |

### 2.3. Detailed Output Format

```markdown
## 2. Test Case Details

### TC_<MODULE>_<NNN>

- **Test Case ID:** TC_<MODULE>_<NNN>
- **Module:** <module name>
- **Req ID:** REQ_<MODULE>_<NN> (proposed)
- **Test Title:** <title>
- **Priority:** High | Medium | Low
- **Preconditions:**
  1. <condition 1>
  2. <condition 2>
- **Test Environment Info:** <OS, browser, environment, URL, repo `branch` @ `short SHA`>
- **Test Data:** <shared account, general input values>

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | <action 1> | <data 1 or -> | <expected result 1> | | | | - |
| 2 | <action 2> | <data 2 or -> | <expected result 2> | | | | - |
```

---

## 3. MANDATORY RULES

1. **Always keep all 12 fields in the list and all 9 items in the details** (including the step table), even when unpopulated. Do not omit, rename, or reorder fields.
2. **One test case tests one objective.** If there are multiple objectives, split them into separate test cases.
3. **Test Case IDs are unique** and strictly incrementing within the same module.
4. **Test steps are reproducible** by another tester without asking for clarification: state input values, screen codes/routes, and where to interact.
5. **Expected results correlate with individual steps** as well as the final outcome, allowing clear PASS/FAIL verification.
6. **Do not pre-fill** Actual Result, Status, or Bug ID when drafting. After execution, only Actual Result (in the list) is filled; Status and Bug ID stay blank unless the user asks or provides them.
7. **Do not fabricate missing information.** Use real UI labels, routes, and messages found in the code or ticket. If business logic or a message is unclear, document the assumption in Notes ("unverified"), or ask the user.
8. **List and Details must match**: identical Test Case ID, Title, Preconditions, and steps. The only allowed difference is Actual Result, which is filled in the list only after execution.
9. **Use exact UI labels** consistently across all test cases (e.g., 「変更」, 「情報の変更」, "Email", "Login").
10. **Output order:** header, then the list, then detail blocks sorted by Test Case ID.
11. **Every Req ID** in the list exists in the header table, and every proposed requirement is covered by at least one test case.
12. **State risky side effects** (submitting to external services, irreversible changes) in Notes, and add "Do not submit" to steps that must stop before them.

---

## 4. SCOPE OF SCENARIOS TO CONSIDER

### 4.1. Test Groups

When the user does not specify the exact quantity, cover the following where applicable:

- **Positive flow (happy path):** valid data, standard correct operations.
- **Negative flow:** invalid, missing, or malformed data.
- **Boundary value analysis:** min/max lengths and edge values. For character or numeric ranges, test the first value inside the range, the last value inside, and the value just outside on each side.
- **Regression:** existing behaviour that must not change (rejections that must still be rejected, redirects, existing messages).
- **Persistence:** when the change touches saving data, verify the value is saved and displayed again after reopening the screen, and verify the API response (e.g., status 200).
- **Authorization and security:** unauthorized access, injection (SQL, HTML/script markup), account locking.
- **UI/UX and display:** error messages and their position, button enabled/disabled state, layout issues noticed during the run (record in Notes or Actual Result).
- **Multiple screens:** when the same behaviour is used on many screens, test the main screens fully and the others once each. For boundaries, pick the screen that isolates the behaviour under test and say why in Notes.

### 4.2. Priority Guideline (adjust per ticket)

| Priority | Typical cases |
|---|---|
| High | Main acceptance behaviour of the ticket, data saving, rejection of dangerous input, behaviour on the most-used screens. |
| Medium | Same behaviour on secondary screens or fields, boundary values. |
| Low | Rare edge cases, unchanged existing behaviour kept as a regression check. |

---

## 5. SAMPLE TEMPLATE: LOGIN FEATURE

**User Prompt:** "Draft test cases for the login feature (ticket HASHST_DEV-0000)."

### 5.1. File Header

```markdown
# Test Cases: Login with email and password (HASHST_DEV-0000)

- **Function:** Login with a registered email and password
- **Module:** Authentication (web app, /login)
- **Environment:** STG (assumed), web app `release/1.0` @ `abc1234` (assumed)
- **Known business rules (from HASHST_DEV-0000):**
  - Rule 1: A user with an active account logs in with the registered email and password and is redirected to /dashboard.
  - Rule 2: An incorrect password shows an error and keeps the user on /login.
  - Rule 3: Email is required.

**Proposed requirement IDs**

| Req ID | Requirement |
|---|---|
| REQ_AUTH_01 (proposed) | A user can log in with valid credentials |
| REQ_AUTH_02 (proposed) | Login is rejected with an error for invalid or missing credentials |
```

### 5.2. Test Case List (draft state)

| Test Case ID | Module | Req ID | Test Title | Priority | Preconditions | Test Steps | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TC_AUTH_001 | Authentication | REQ_AUTH_01 (proposed) | Login successfully with valid credentials | High | 1. Account `testuser_01@example.com` exists and is active<br>2. User is not logged in | 1. Navigate to the login page (/login)<br>2. Enter `testuser_01@example.com` in the Email field<br>3. Enter `Password@2026` in the Password field<br>4. Click the "Login" button | Redirected to /dashboard<br>Welcome message is displayed | | | | - |
| TC_AUTH_002 | Authentication | REQ_AUTH_02 (proposed) | Login fails with incorrect password | High | 1. Account `testuser_01@example.com` exists and is active<br>2. User is not logged in | 1. Navigate to the login page (/login)<br>2. Enter `testuser_01@example.com` in the Email field<br>3. Enter `WrongPassword@1` in the Password field<br>4. Click the "Login" button | An error message is displayed<br>User remains on /login | | | | Error message text is an assumption, needs verification against the spec |
| TC_AUTH_003 | Authentication | REQ_AUTH_02 (proposed) | Login fails when the Email field is left blank | Medium | 1. User is not logged in | 1. Navigate to the login page (/login)<br>2. Leave the Email field empty<br>3. Enter `Password@2026` in the Password field<br>4. Click the "Login" button | Login request is not submitted<br>A validation message requiring Email is displayed | | | | - |

### 5.3. The Same Row After Execution (only Actual Result and Notes change)

| Test Case ID | Module | Req ID | Test Title | Priority | Preconditions | Test Steps | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TC_AUTH_002 | Authentication | REQ_AUTH_02 (proposed) | Login fails with incorrect password | High | 1. Account `testuser_01@example.com` exists and is active<br>2. User is not logged in | 1. Navigate to the login page (/login)<br>2. Enter `testuser_01@example.com` in the Email field<br>3. Enter `WrongPassword@1` in the Password field<br>4. Click the "Login" button | An error message is displayed<br>User remains on /login | The message "Invalid email or password" was displayed and the page stayed on /login.<br>Screenshot: `screenshot/TC_AUTH_002.png` | | | Error message text is an assumption. Observed text is "Invalid email or password" |

### 5.4. Test Case Details

### TC_AUTH_001

- **Test Case ID:** TC_AUTH_001
- **Module:** Authentication
- **Req ID:** REQ_AUTH_01 (proposed)
- **Test Title:** Login successfully with valid credentials
- **Priority:** High
- **Preconditions:**
  1. Account `testuser_01@example.com` exists and is active
  2. User is not logged in
- **Test Environment Info:** Windows 11, Chrome latest, STG (assumed), web app `release/1.0` @ `abc1234` (assumed)
- **Test Data:** Email `testuser_01@example.com`, Password `Password@2026`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the login page | `/login` | The login page displays the Email and Password fields | | | | - |
| 2 | Enter the value in the Email field | `testuser_01@example.com` | The Email field displays the entered text | | | | - |
| 3 | Enter the value in the Password field | `Password@2026` | The password is displayed as masked characters (•) | | | | - |
| 4 | Click the "Login" button | - | Redirected to /dashboard, welcome message is displayed | | | | - |

### TC_AUTH_002

- **Test Case ID:** TC_AUTH_002
- **Module:** Authentication
- **Req ID:** REQ_AUTH_02 (proposed)
- **Test Title:** Login fails with incorrect password
- **Priority:** High
- **Preconditions:**
  1. Account `testuser_01@example.com` exists and is active
  2. User is not logged in
- **Test Environment Info:** Windows 11, Chrome latest, STG (assumed), web app `release/1.0` @ `abc1234` (assumed)
- **Test Data:** Email `testuser_01@example.com`, incorrect Password `WrongPassword@1`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the login page | `/login` | The login page displays the Email and Password fields | | | | - |
| 2 | Enter the value in the Email field | `testuser_01@example.com` | The Email field displays the entered text | | | | - |
| 3 | Enter the value in the Password field | `WrongPassword@1` | The password is displayed as masked characters (•) | | | | - |
| 4 | Click the "Login" button | - | An error message is displayed and the user remains on /login | | | | Error message text is an assumption |

### TC_AUTH_003

- **Test Case ID:** TC_AUTH_003
- **Module:** Authentication
- **Req ID:** REQ_AUTH_02 (proposed)
- **Test Title:** Login fails when the Email field is left blank
- **Priority:** Medium
- **Preconditions:**
  1. User is not logged in
- **Test Environment Info:** Windows 11, Chrome latest, STG (assumed), web app `release/1.0` @ `abc1234` (assumed)
- **Test Data:** Empty Email, Password `Password@2026`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | Navigate to the login page | `/login` | The login page displays the Email and Password fields | | | | - |
| 2 | Leave the Email field empty | - | The Email field contains no text | | | | - |
| 3 | Enter the value in the Password field | `Password@2026` | The password is displayed as masked characters (•) | | | | - |
| 4 | Click the "Login" button | - | The login request is not submitted, a validation message requiring Email is displayed | | | | - |

---

## 6. RECORDING EXECUTION RESULTS (PHASE 3)

Applies only when the user asks to run the test cases.

### 6.1. What to fill

| Field | Draft | After execution |
|---|---|---|
| Actual Result (list) | blank | **Filled** |
| Actual Result (details) | blank | blank |
| Status | blank | blank, unless the user asks to fill it |
| Bug ID | blank | blank, unless provided by the user |
| Notes | assumptions | add findings (see 6.3) |

### 6.2. Actual Result format

1. If test data was prepared or a harness was needed, start with that in parentheses (e.g., "(Run after setting the test account's OCCUPATION_TYPE to 2 in the local DB; reverted afterwards.)").
2. If a step could not be done as written (e.g., a radio button is not shown for this account), say so in parentheses and describe what was done instead.
3. Describe what was observed, using the same wording as the Expected Result. Quote messages exactly.
4. Add layout or UX observations noticed during the run (e.g., a message overlapping a label).
5. End with the screenshot line:
   - one image: ``Screenshot: `screenshot/TC_<MODULE>_<NNN>.png` ``
   - several images: ``Screenshots: `screenshot/TC_<MODULE>_<NNN>_before_submit.png`, `screenshot/TC_<MODULE>_<NNN>_after_submit.png` ``

### 6.3. When the observed result differs from Expected

- Do **not** change Expected Result to match the observation, and do not change code.
- Start Notes with `**FAILED locally (YYYY-MM-DD):**` followed by the evidence (status code, error text, screen shown) and, if known, the cause.
- If the difference is an accepted change (e.g., documented in an ADR), say so in Notes instead of marking it as failed.
- Leave Status and Bug ID blank unless the user asks.

### 6.4. Clean-up

Restore any temporary data changes (database values, headers, settings) and list what was changed and restored in the final message to the user.