# ownersbook-plus

Glossary of domain terms used across the ownersbook-plus repositories.

## Language

**Masked email**:
The form of a customer's email address shown on screen, with the middle of the **Local part** replaced by `*` so the full address is not exposed. The domain is never masked.
_Avoid_: hidden email, obfuscated email

**Local part**:
The portion of an email address before the first `@`.
_Avoid_: email prefix, user name

**Unmasked short address**:
An email whose **Local part** is 3 characters or fewer. It is displayed in full, by design.
_Avoid_: masking bug, masking leak

### Customer name and address characters

**Input character check** (入力文字種チェック):
The front-end check that a value contains only the character kinds allowed for its field (kana, kanji, alphabet, digits, some symbols). It is deliberately loose.
_Avoid_: kanji validation, name validation

**SJIS-win check** (日本語テキストチェック):
The server-side check that every character can be represented in SJIS-win (Windows Shift_JIS). A character that fails is reported to the customer as not allowed.
_Avoid_: Japanese check, encoding check

**保振制度内字** (JASDEC in-system character):
A character usable in the 証券保管振替機構 (保振) system. Customer names are converted to these characters during **審査**, so they are the final, stored form.
_Avoid_: allowed character, standard kanji

**Out-of-range kanji** (範囲外の漢字):
A kanji outside the CJK Unified Ideographs block U+4E00–U+9FFF, such as 﨑 (compatibility), 㐂 (Ext-A) or 𠮷 (Ext-B, a surrogate pair). These are common in real names.
_Avoid_: 外字, invalid kanji

### Fund offering status

**当初募集** (primary offering):
The period in which a fund's units are first sold to investors, before any secondary trading.
_Avoid_: initial sale, first offering

**募集終了** (sold out):
The investor-facing status of a fund in **当初募集** whose remaining units have all been taken. It says nothing about whether the offering period has ended.
_Avoid_: 当初募集終了, offering closed

**当初募集終了日時** (primary offering end date):
The configured date and time at which the **当初募集** period ends.
_Avoid_: 募集終了, sold-out date

## Relationships

- A **Masked email** keeps the first 2 and the last 1 character of the **Local part**.
- An **Unmasked short address** is the one case where a **Masked email** equals the original address.
- A customer name passes through three stages in order: **Input character check** → **SJIS-win check** → conversion to **保振制度内字** at 審査. Each stage is stricter than the one before.
- An **Out-of-range kanji** must pass the **Input character check**. Whether it survives the **SJIS-win check** depends on the character (﨑 passes, 𠮷 and 㐂 do not).
- A fund can show **募集終了** while still before its **当初募集終了日時**, because selling out and the period ending are independent events.
