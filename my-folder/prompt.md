## Task Name
Fix PDF loading error for the `electronic.vue` screen.

## Goal
Fix issues related to the new personal account registration flow at `http://local-register-hsec.crudist.tech:8081/`

## Context:
- Currently, when proceeding to step 2 of the registration process on the `/mobile-registration/src/views/PersonalResiger/Confirm/electronic.vue` screen, an API error occurs while loading the PDF file, triggering a pop-up indicating the PDF loading failure:
```
読み込みに失敗しました
```
...so it is not possible to click "Confirm" to proceed to the next step. This error needs to be checked and resolved to complete the new account registration process.
- The files have reportedly been uploaded to the appropriate directory at `http://localhost:8900/browser/hhd-hsec-general/public%2Fdocument%2Fpdf%2F`, with the following path details:
```
 - public/document/pdf/df47f747181d148513a6e2945ac09b2d/金銭・有価証券の預託、記帳及び振替に関する契約のご説明（LSEC）20260401.pdf
 - public/document/pdf/4181de55d5fb9542c2a1b4b121128471/約款・規程集（LSEC）20260401.pdf 
 - public/document/pdf/02bd229a50fac7b767dfd959aaaeb40a/A003プライバシーポリシー（個人情報保護方針）（LSEC）20260401.pdf 
 - public/document/pdf/d5b5e62fd465cef60be66f18a9dc6aa3/電子交付等に関する説明書（LSEC）20260401.pdf 
 - public/document/pdf/70e0e4d7a846bcafb38b2be7780cd563/反社会的勢力でないことの確約に関する同意書（LSEC）20260401.pdf 
```

## Tasks:
1. Investigate the root cause of the error.
2. Identify the error and determine the optimal fix.
3. Implement the fix.
4. Verify the fix to ensure the error has been resolved.

## Rules
Plan before executing.
