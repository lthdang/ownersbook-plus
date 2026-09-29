# The front-end Input character check is deliberately loose

The **Input character check** (`verifyInputText`) accepts **Out-of-range kanji** (CJK Ext-A U+3400–4DBF, Compatibility U+F900–FAFF, and U+20000–3FFFF) even though some of them are later rejected by the **SJIS-win check** or replaced at 審査. The intent is to accept a customer's name as written on their 戸籍 and narrow it to **保振制度内字** downstream, rather than blocking the customer at the first screen. Confirmed by 簡 大喬 in HASHST_DEV-4556 (2026-09-30).

## Considered Options

- **Keep the front-end strict (U+4E00–U+9FFF only) and ask customers to type 崎 / 吉 instead.** Rejected: the customer cannot proceed with their real name, and the narrowing already exists at the **SJIS-win check** and at 審査.
- **Allow only 3-byte ranges (leave out U+20000+).** Rejected for now: it would keep 𠮷 blocked with a format error. Revisit if 4-byte characters fail to save on screens that skip the **SJIS-win check**.

## Consequences

- On screens that do not call the **SJIS-win check** (e.g. app-vuejs `l2213`, `l2216`, `l2217`, `l2251`, and the identity-verification screens `l2500` / `x1201` / `y1200`), an Out-of-range kanji reaches the server. Whether 4-byte characters save correctly (history tables are created with `CHARSET=utf8`) is checked on testing. A failure is handled in a new ticket.
- On identity-verification screens, a customer who types 﨑 when 崎 is registered now gets a "details don't match" error instead of a format error. Accepted.
- Do not "tighten" `verifyInputText` back to U+4E00–U+9FFF without a new product decision.
- mobile-registration has the same check. It is aligned in a separate ticket.
