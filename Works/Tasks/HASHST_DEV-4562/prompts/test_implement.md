Run the test cases in `/home/haidang/Works/ownersbook+/Works/Tasks/HASHST_DEV-4562/test-case/TC_HASHST_DEV-4562_sold-out-label.md` on the local environment.
For each test case:
- Follow each step correctly; if temporary data needs to be prepared (DB, header), record it and revert after running.
- Fill in the Actual Result in the list table (do not fill in the detail table), describe the observed behavior and screenshot path: `screenshot/TC_xxx.png` (if multiple images, distinguish with suffix _step3...).
- Take screenshots and save them in the screenshot/ directory next to the TC file.
- If the result differs from Expected: do not modify the code or Expected. Record in Notes "FAILED locally ([date])" along with the observed reason (status code, message, stack trace if any).
- If a step cannot be performed as described (e.g., a radio button does not display for this account), note the discrepancy in Actual Result in parentheses at the beginning.
- Leave Status and Bug ID blank.
- Finally, list the temporary data that was changed and confirm it has been restored.