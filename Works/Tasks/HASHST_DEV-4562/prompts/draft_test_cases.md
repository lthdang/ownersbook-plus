Please write test cases for the changes in ticket [TC_HASHST_DEV-4562].

Read all these files before writing:
1. Template: `/home/haidang/Works/ownersbook+/Works/template/test-case-template.md`
2. Sample output file: `/home/haidang/Works/ownersbook+/Works/Tasks/HASHST_DEV-4562/test_case/TC_HASHST_DEV-4556_input-character-check.md` (mimic the structure, level of detail, and writing style of the Notes; do not copy the content)
3. Ticket: `/home/haidang/Works/ownersbook+/Works/Tasks/HASHST_DEV-4562/contexts/HASHST_DEV-4562.md`
4. Implemented changes: run `git diff backlog/HASHST_DEV-4562...HEAD` and `git log backlog/HASHST_DEV-4562..HEAD`
5. Related CONTEXT.md, ADR (if any), and code of the affected screens/functions

Follow these two steps:
STEP 1 (stop and wait for my approval, do not write details yet):
- Header: Function, Module, Environment (branch @ short SHA, local-docker), Known business rules numbered Rule 1..n, source (ticket/ADR).
- Table of Proposed requirement IDs (REQ_<MODULE>_NN (proposed)).
- List of proposed Test Case ID, Req ID, Title, Priority. Each test case should have a single objective.
- Cover: happy path, negative, boundary (value just before/at/after threshold), regression (old behavior unchanged), security (markup/XSS), and data flow if the change affects it.
- Note any areas of uncertainty for me to answer.
STEP 2 (after my approval): write the full file according to the sample structure:
header → Proposed Req IDs → 1. Test Case List (12 columns) → 2. Test Case Details (all 9 items + step table).

Rules:
- Describe in English; keep Japanese for UI labels and error messages in 「」.
- Record screen/route codes (e.g., l2212, /l2200) and use the exact UI labels.
- Each step is a single action; Input Data should be specific. For special characters, record the value with code point, and note in Notes how to input if it's easy to make mistakes (paste, DevTools Console, invisible characters).
- Do not fabricate messages or behaviors. For unverified areas, write "unverified" in Notes; for unknown areas, write "(assumed)" in Test Environment Info.
- The list and details must match in ID, Title, Preconditions, and steps.
- Save to: `/home/haidang/Works/ownersbook+/Works/Tasks/HASHST_DEV-4562/test-cases/TC_HASHST_DEV-4562_[slug].md`