app-vuejs\_verifyInputText で「﨑」「𠮷」など CJK 統合漢字の範囲外の漢字が弾かれる（common.js）

# 詳細

## 現状と背景

[HASHST\_DEV-4542](file:///view/HASHST_DEV-4542)（単体テスト追加 B: 入力チェックの関数）で `src/utils/common.js` の挙動を記録するテストを書いた際に見つかった挙動。4542 では修正せず、テストに現状の挙動を記録している。仕様か不具合かの判断が必要なため起票する。

`verifyInputText`（入力文字種のチェック）は、漢字を「CJK 統合漢字」と呼ばれる Unicode の範囲 `U+4E00〜U+9FFF` と `々`（U+3005）だけで許可している。この範囲に入らない漢字は、氏名によく使われるものでも弾かれる。

| 呼び出し | 実際の戻り値 | 理由 |
| --- | --- | --- |
| `verifyInputText('髙橋', 1)` | `true` | 「髙」U+9AD9 は範囲内 |
| `verifyInputText('山﨑', 1)` | `false` | 「﨑」（たつさき）U+FA11 は CJK 互換漢字の範囲 |
| `verifyInputText('𠮷田', 1)` | `false` | 「𠮷」（つちよし）U+20BB7 は範囲外（JavaScript の文字列では2文字分のサロゲートペアになる） |

漢字を許可する type（1・3・4・5・7・9・10・11・12）はすべて同じ範囲指定のため、氏名だけでなく住所・法人名・内部者氏名などでも同様。

### 実際に起きうるか

-   「﨑」「𠮷」は氏名で一定数使われる字のため、該当するお客様が氏名（type1、`l2212.vue` お客様情報変更：名前・住所 など）を戸籍どおりに入力するとエラーになり、先に進めない可能性がある
-   実際には「崎」「吉」で入力してもらう運用になっている可能性もある。また、サーバー側・連携先（証券会社のシステム等）がこれらの字を扱えるかによっては、フロントで弾いているのが意図どおりの可能性もある

## 対応内容(期待結果)

※ 1 の確認結果と、それを踏まえた対応方針は下記「対応方針」に記載（「許可する方針」に決定）

1.  氏名・住所などで範囲外の漢字を許可しない方針か（「崎」「吉」などで代替入力してもらう運用か）を確認する
2.  許可しない方針であれば、本チケットは「対応不要」としてクローズしてよい。4542 で追加したテストのコメント「【現状の挙動】」を仕様として書き換え、あわせて入力エラー時の案内文言で代替入力を促せているかを確認する
3.  許可する方針であれば、サーバー側・連携先がこれらの字を保存・送信できるかを先に確認したうえで、許可する範囲（CJK 互換漢字 U+F900〜U+FAFF、CJK 統合漢字拡張 B 以降など）を決める。サロゲートペアを扱うには正規表現に `u` フラグが必要（babel 6 の変換対象になるかも要確認）
4.  修正する場合は、4542 で追加したテストの該当ケース（下記）の期待値を `true` に書き換えてから（Red）、実装を直す（Green）
    -   `test/unit/specs/utils/common.verifyInputText.spec.js` の「verifyInputText（漢字の範囲）」

## 対応方針

2026-09-30 の簡さんのコメント（[https://hashdash.backlog.com/view/HASHST\_DEV-4556#comment-822040432](https://hashdash.backlog.com/view/HASHST_DEV-4556#comment-822040432) ）を受けて追記。

### 方針：フロントでは範囲外の漢字も許可する

氏名などの文字は「フロントでは緩くチェックして入り口を広く受け、最終的には審査で保振制度内字（証券保管振替機構＝保振のシステムで使える文字）に変更する」運用。そのため「﨑」「𠮷」がフロントで弾かれているのは**改善対象**とする。

文字のチェックは次の3段階になっている。

| 段階 | 場所 | チェック内容 |
| --- | --- | --- |
| ① | app-vuejs `src/utils/common.js` の `verifyInputText` | 入力文字種（**本チケットの対象**） |
| ② | common-front-api の `checkJpText`（フロントからは `/common/check_jp_text`） | SJIS-win（Windows 版の Shift\_JIS）で表せるか |
| ③ | company-admin `fuel/app/config/jasdeccharacter.php` | 保振制度内字への変換 |

### 各文字が3段階をどう通るか（調査結果）

| 文字 | ① verifyInputText | ② SJIS-win | ③ 保振制度内字 |
| --- | --- | --- | --- |
| 﨑 U+FA11 | 不可（本チケット） | 可（SJIS-win の IBM 拡張文字に含まれる） | 変換表に「﨑 → 崎」あり |
| 𠮷 U+20BB7 | 不可（本チケット） | **不可**（SJIS-win に存在しない） | 変換表になし（審査での手動変更になる） |
| 㐂 U+3402 | 不可（本チケット） | **不可**（SJIS-win に存在しない） | 変換表になし |

-   ② は common-front-api の `findUnsupportCharacter`（`app/helpers.php`）で判定している。1 文字ずつ PHP の `mb_convert_encoding` で UTF-8 → SJIS-win → UTF-8 と変換して戻し、元の文字と一致しなければ不可とする（変換できない字は `?` に置き換わるため不可になる）。PHP 8.3 で同じ関数を実行し、「﨑」は可、「𠮷」は不可を確認した。拡張 A の「㐂」U+3402 も不可
-   不可のときのエラー文言は `E103-1045`「{文字}の入力はできません。ひらがなで入力してください。」。app-vuejs の `checkJpText`（`src/utils/common-api.js`）がこの文言をそのまま画面に出す。`checkJpText` は ① を通った場合だけ呼ばれる
-   つまり「﨑」はフロントを直せば通るようになる。「𠮷」はフロントを直しても、`checkJpText` を呼ぶ画面（`l2212` / `l2221` / `l2223` / `l2228` / `l2229` / `l2232` / `components/ruleAddress`）では API から返るエラー文言で弾かれる見込み

### 対応内容

1.  **許可する範囲を広げる**（漢字を許可する type 1・3・4・5・7・9・10・11・12 のすべて）
    -   CJK 統合漢字拡張 A：U+3400〜U+4DBF
    -   CJK 互換漢字：U+F900〜U+FAFF（「﨑」など）
    -   CJK 統合漢字拡張 B 以降・CJK 互換漢字補助：U+20000〜U+3FFFF（「𠮷」など。サロゲートペアになる字）
2.  **実装上の注意**
    -   サロゲートペアを1文字として扱うため、正規表現に `u` フラグを付ける。babel 6 の `babel-preset-env` に含まれる `transform-es2015-unicode-regex` が、`u` フラグ非対応のブラウザ向けに変換する（プラグインがインストール済みであることは確認済み。ビルド後の出力は実装時に確認する）
    -   `u` フラグを付けると、文字クラス内の不要なエスケープ（type10・11 の `\&` `\'` `\,` `\·` `\＆` `\’` など）が構文エラーになる（node で確認済み）。エスケープを外す必要があるため、4542 で追加した type10・11 の記号のテストで挙動が変わらないことを確認する
    -   差分を追いやすくするため、今回は9つの正規表現それぞれへの範囲の追記にとどめ、範囲指定の共通化は行わない
3.  **「𠮷」が②で弾かれる件の確認**
    -   testing 環境で、`l2212`（お客様情報変更：名前・住所）に「𠮷」を入力し、`checkJpText` で弾かれて上記のエラー文言が画面に表示されることを確認する
    -   文言は「𠮷の入力はできません。ひらがなで入力してください。」で、代替の字（「吉」）の入力は案内していない。案内を変える、または SJIS-win チェックを緩める必要があると判断したら、common-front-api 側で別チケットを起票する（本チケットでは app-vuejs のみ修正する）
4.  **TDD の手順**
    -   Red：`test/unit/specs/utils/common.verifyInputText.spec.js` の「verifyInputText（漢字の範囲）」で、`山﨑`・`𠮷田` の期待値を `true` にし、「【現状の挙動】」のコメントを削除する。拡張 A の字（例：「㐂」U+3402）と、type1 以外（type3 住所、type9 法人名など）のケースも追加する
    -   Green：正規表現を修正する
    -   Refactor：既存のテスト（type10・11 の記号など）がすべて通ることを確認する
5.  **今回の対象外**
    -   異体字セレクタ（IVS。字形の違いを表すために漢字の後ろに付く見えない文字、U+E0100〜）付きの字は今回も弾かれたまま。要否は別途判断する

## 対応根拠となるドキュメント

-   [HASHST\_DEV-4542](file:///view/HASHST_DEV-4542)
-   ブランチ: `backlog/HASHST_DEV-4542`（MR は作成後に追記）
-   対応方針の根拠：簡さんのコメント [https://hashdash.backlog.com/view/HASHST\_DEV-4556#comment-822040432](https://hashdash.backlog.com/view/HASHST_DEV-4556#comment-822040432)

## 改修差分

【対応後記入】

## 結合環境確認履歴

【対応後記入】

[@簡 大喬](file:///user/*NPpLHVZql0) さんに確認中  
[https://teams.microsoft.com/l/message/19:85678eab1e76462ebf17731930cda951@thread.skype/1790732376190?tenantId=a963622d-e1d7-4199-a479-b0779a8f2c2d&groupId=5f185cb2-4f5d-48da-9482-68e2274d438f&parentMessageId=1790660224711&teamName=%E3%83%86%E3%82%AF%E3%83%8E%E3%83%AD%E3%82%B8%E3%83%BC%E6%8E%A8%E9%80%B2%E9%83%A8&channelName=OwnersBook%EF%BC%8B%E9%96%8B%E7%99%BA&createdTime=1790732376190](https://teams.microsoft.com/l/message/19:85678eab1e76462ebf17731930cda951@thread.skype/1790732376190?tenantId=a963622d-e1d7-4199-a479-b0779a8f2c2d&groupId=5f185cb2-4f5d-48da-9482-68e2274d438f&parentMessageId=1790660224711&teamName=%E3%83%86%E3%82%AF%E3%83%8E%E3%83%AD%E3%82%B8%E3%83%BC%E6%8E%A8%E9%80%B2%E9%83%A8&channelName=OwnersBook%EF%BC%8B%E9%96%8B%E7%99%BA&createdTime=1790732376190)

氏名等で利用できる文字コードに関しては、基本的にはフロント側で緩いチェックをし、審査で利用できる文字に変更していくイメージです

vuejs: src/utils/common.js → 入力が全角かどうか  
common-front-api : [checkJpText](https://gitlab.hashdash-group.com/sto_fund/common-front-api/-/blob/master/app/Http/Controllers/CommonController.php#L30)　→ SJIS-win　かどうか  
company-admin: [保振制度内字に変換](https://gitlab.hashdash-group.com/sto_fund/company-admin/-/blob/master/fuel/app/config/jasdeccharacter.php#L5)　→ 制度内文字に変更

最終的には保振制度内字にするのですが、入り口は広く受けられるように最低限のチェックをする形かと思います

現状　𠮷、﨑などの字が入力として受付れないのであれば、改善する箇所になると思います

[@lthdang](file:///user/*UZ6vXjhID4) please check. thank you.

app-vuejs\_verifyInputText で「﨑」「𠮷」など CJK 統合漢字の範囲外の漢字が弾かれる（common.js）
