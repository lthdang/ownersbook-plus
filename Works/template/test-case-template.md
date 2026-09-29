# TEST CASE STRUCTURE

When users request to write test cases for a specific feature (e.g., "draft test cases for the login feature"), you need to return two parts in the exact following order:

1. **Test Case List**: A summary table where each test case occupies one row.
2. **Test Case Details**: Each test case is presented in a separate block containing a detailed step-by-step table..

## 1. TEST CASE LIST

### 1.1. Field Descriptions for Summary Row

| No. | Field Name | Description | Fulfillment Rules / Naming Conventions |
|-----|--------|-------|--------------|
| 1 | **Test Case ID** | Unique identifier for the test case | `TC_<MODULE>_<NNN>`, where `NNN` is a 3-digit sequential number incrementing sequentially (e.g., `TC_CUSTOMER_001`) |
| 2 | **Module** | Module or functional group containing the test case | State the module name clearly(e.g., `Customer`, `Authentication`) |
| 3 | **Req ID** | Requirement ID verified by the test case | `REQ_<MODULE>_<NN>`(e.g.,`REQ_CUSTOMER_01`).If not provided by the user, propose based on convention and mark "(proposed)" |
| 4 | **Test Title** | Concise title describing what needs to be verified | Start with an action verb, clearly state conditions and expected outcomes (e.g., "Login successfully with valid credentials"). |
| 5 | **Priority** | Importance / Severity level | Strictly use: `High` / `Medium` / `Low` |
| 6 | **Preconditions** | Required system/data state prior to execution | List each condition, numbered. Specify necessary data setup (account, role, environment). |
| 7 | **Test Steps** | Sequence of actions the tester must perform | Numbered 1, 2, 3...; `one action per step`; specify `concrete input data` |
| 8 | **Expected Result** | Observed result during test execution | Specific and verifiable (messages, target screen redirection, data state changes). Avoid vague terms like "works properly". |
| 9 | **Actual Result** | Execution outcome | `Leave blank` when drafting new test cases; populated by tester after execution. |
| 10 | **Status** | Kết quả của lần chạy test | `PASS` / `FAIL`. `Leave blank` or mark as `Untested` when drafting new |  
| 11 | **Bug ID** | Mã bug liên quan nếu test FAIL | Leave blank when drafting new; populate only when provided by the user. |
| 12 | **Notes** | Thông tin bổ sung | State risks, special data, dependencies between test cases, or assumptions. Put - if none. |

### 1.2. Summary List Output Format

Presented as a `12-column Markdown table`, strictly following the order defined in section 1.1. Multiple items within a single cell must be separated by `<br>`.

| Test Case ID | Module | Req ID | Test Title | Priority | Preconditions | Test Steps | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TC_\<MODULE\>_\<NNN\> | \<module name\> | REQ_\<MODULE\>_\<NN\> | \<title\> | High | 1. \<condition 1\><br>2. \<condition 2\> | 1. \<step 1\><br>2. \<step 2\><br>3. \<step 3\> | \<expected result 1\><br>\<expected result 2\> | | | | \<notes or -\> |

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
| 6 | **Preconditions** | Identical to the list, with numbered conditions. |
| 7 | **Test Environment Info** | OS, browser/device, environment (DEV/STG/UAT), system URL, build version. State assumptions and mark "(assumed)" if unknown. |
| 8 | **Test Data** | List shared accounts and input values used for the test case. Step-specific data goes into the "Input Data" column. |
| 9 | **Execution Details Table** | Each row represents a single step, see section 2.2 |

### 2.2. Detailed Table Columns

| Column Name | Fulfillment Rules |
|-----|--------------|
| **Step #** | Sequential step number, starting from 1.    |
| **Step Action** | Single action per step, starting with an action verb. Use exact UI field/button labels |
| **Input Data** | Specific value entered/selected at that step. Put - if none. |
| **Expected Result** | Specific, verifiable expected outcome for that specific step |
| **Actual Result** | **Leave blank** when drafting new test cases.|
| **Status** | `PASS` / `FAIL`. **Leave blank** when drafting new test cases|
| **Bug ID** | **Leave blank** when drafting new test cases |
| **Notes** | Additional information or assumptions. Put - if none |

### 2.3. Detailed Output Format

```markdown
### TC_<MODULE>_<NNN>

- **Test Case ID:** TC_<MODULE>_<NNN>
- **Module:** <module name>
- **Req ID:** REQ_<MODULE>_<NN>
- **Test Title:** <title>
- **Priority:** High | Medium | Low
- **Preconditions:**
  1. <condition 1>
  2. <condition 2>
- **Test Environment Info:** <OS, browser, environment, URL, build version>
- **Test Data:** <shared account, general input values>

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|--------|-------------|------------|-----------------|---------------|--------|--------|-------|
| 1 | <action 1> | <data 1> | <expected result 1> | | | | - |
| 2 | <action 2> | <data 2> | <expected result 2> | | | | - |
```

## 3. MANDATORY RULES

1. **Always maintain all 12 fields in the summary list and all 9 items in the details section** (including the step-by-step table), even for unpopulated fields. Do not omit fields, rename fields, or change their order.
2. **One test case tests only one objective.** If multiple objectives exist, split them into separate test cases.
3. **Test Case IDs must be unique** and strictly incrementing within the same module.
4. **Test steps must be reproducible** by another tester without needing to ask for clarification: clearly state input values and UI interaction locations.
5. **Expected results must correlate** with individual steps as well as the final outcome, allowing clear PASS/FAIL verification.
6. *Do not pre-fill** Actual Result, Status, or Bug ID prior to actual execution. Templates and newly created test cases must leave these columns blank.
7. **Do not fabricate missing information.** If business logic is unclear (e.g., password length rules, maximum allowed failed login attempts), document assumptions under **Notes** or request clarification from the user.
8. **Summary List and Details must match perfectly**: identical Test Case ID, Title, Preconditions, and the steps listed in the summary table must align with the rows in the detailed step table.
9. **Use exact UI labels** (e.g., "Email", "Password", "Login") consistently across all test cases.
10. **Output Order:** The summary table must appear first, followed by detail blocks sorted by Test Case ID.


## 4. SCOPE OF SCENARIOS TO CONSIDER

When the user does not specify the exact quantity, cover the following test groups where applicable:   
- **Positive Flow (Happy Path)**: Valid data, standard correct operations.   
- **Negative Flow**: Invalid, missing, or malformed data.   
- **Boundary Value Analysis**: Minimum/maximum input lengths, edge values.   
- **Authorization & Security**: Unauthorized access, SQL injection, account locking, etc.   
- **UI/UX & Display**: Error messages, button enabled/disabled states, visual responsiveness.   


## 5. SAMPLE TEMPLATE: LOGIN FEATURE

**User Prompt:** "Draft test cases for the login feature."

### 5.1. Test Case List

| Test Case ID | Module | Req ID | Test Title | Priority | Preconditions | Test Steps | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TC_AUTH_001 | Authentication | REQ_AUTH_01 (proposed) |Login successfully with valid credentials | High | 1. Account testuser_01@example.com exists and is active<br>2. User is not logged in | 1. Navigate to login page (/login)<br>2. Enter valid email<br>3. Enter valid password<br>4. Click "Login" button | Redirected to /dashboard, welcome message displayed | | | | - |
| TC_AUTH_002 | Authentication | REQ_AUTH_01 (proposed) | Login fails with incorrect password | High | 1. Account testuser_01@example.com exists and is active<br>2. User is not logged in | 1. Navigate to login page (/login)<br>2. Enter valid email<br>3. Enter incorrect password<br>4. Click "Login" button
| Error message displayed, user remains on login page | | | | Error message text is an assumption, needs verification against SRS |
| TC_AUTH_003 | Authentication | REQ_AUTH_01 (proposed) | Login fails when Email field is left blank | Medium | 1. User is not logged in | 1. Navigate to login page (/login)<br>2. Leave Email field empty<br>3. Enter valid password<br>4. Click "Login" button | Login request is not submitted, validation message requiring Email is displayed | | | | - |

### 5.2. Test Case Details

### TC_AUTH_001

- **Test Case ID:** TC_AUTH_001
- **Module:** Authentication
- **Req ID:** REQ_AUTH_01 (proposed)
- **Test Title:** Login successfully with valid credentials
- **Priority:** High
- **Preconditions:**
  1. Account `testuser_01@example.com` exists and is active.
  2. User is not logged in.
- **Test Environment Info:** Windows 11, Latest Chrome Browser, STG Environment (assumed)
- **Test Data:** Email `testuser_01@example.com`, Password `Password@2026`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|
|1| Navigate to login page URL | /login | Login page displays Email and Password fields properly | | | | - |
|2| Enter valid email into Email field | `testuser_01@example.com` | Email field displays entered text correctly | | | | - |
|3| Enter valid password into Password field | Password@2026 | Password displays as masked characters (•) | | | | - |
|4| Click "Login" button | Mouse click | Successfully logged in, redirected to /dashboard, welcome message displayed | | | | - |

### TC_AUTH_002

- **Test Case ID:** TC_AUTH_002
- **Module:** Authentication
- **Req ID:** REQ_AUTH_01 (proposed)
- **Test Title:** Login fails with incorrect password
- **Priority:** Cao
- **Preconditions:**
  1. Account `testuser_01@example.com` exists and is active.   
  2. User is not logged in.
- **Test Environment Info:** Windows 11, Latest Chrome Browser, STG Environment (assumed)
- **Test Data:** Email `testuser_01@example.com`, Incorrect Password `WrongPassword@1`

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Navigate to login page URL | /login | Login page displays Email and Password fields properly | | | | - |
| 2 | Enter valid email into Email field | testuser_01@example.com | Email field displays entered text correctly | | | | - |
| 3 | Enter valid password into Password field | WrongPassword@1 | Password displays as masked characters (•) | | | | - |
| 4 | Click "Login" button | Mouse click     | Login fails, error message "Invalid email or password" displayed, remains on login page | | | | Error message content is an assumption |

### TC_AUTH_003

- **Test Case ID:** TC_AUTH_003
- **Module:** Authentication
- **Req ID:** REQ_AUTH_01 (proposed)
- **Test Title:** Login fails when Email field is left blank
- **Priority:** Medium
- **Preconditions:**
  1. User is not logged in
- **Test Environment Info** Windows 11, Latest Chrome Browser, STG Environment (assumed)
- **Test Data:** Empty Email, Password Password@2026

| Step # | Step Action | Input Data | Expected Result | Actual Result | Status | Bug ID | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Navigate to login page URL | /login | Login page displays Email and Password fields properly | | | | - |
| 2 | Leave Email field blank | (blank) | Email field contains no text | | | | - |
| 3 | Enter valid password into Password field | Password@2026 | Password displays as masked characters (•) | | | | - |
| 4 | Click "Login" button | Mouse click | Login request is not submitted, validation error requiring Email input is displayed | | | | - |
