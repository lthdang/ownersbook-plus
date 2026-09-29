mobile-registration_verifyInputText で「﨑」「𠮷」など CJK 統合漢字の範囲外の漢字が弾かれる（common-js.js）

# 詳細

## 現状と背景

[HASHST\_DEV-4556](https://hashdash.backlog.com/view/HASHST_DEV-4556)（app-vuejs の `verifyInputText` で範囲外の漢字が弾かれる件）の調査で、mobile-registration にも同じ関数があり、同じ範囲指定になっていることがわかった。4556 は app-vuejs のみを対象としているため、mobile-registration 側を本チケットで起票する。

`src/utils/common-js.js` の `verifyInputText`（入力文字種のチェック）は、app-vuejs と同じく漢字を CJK 統合漢字 `U+4E00〜U+9FFF` と `々`（U+3005）だけで許可している。漢字を許可する type（1・3・4・5・7・9・10・11・12）の9つの正規表現すべてが同じ範囲指定。

| 入力 | 現在の結果 | 理由 |
| --- | --- | --- |
| `髙橋` | 通る | 「髙」U+9AD9 は範囲内 |
| `山﨑` | 弾かれる | 「﨑」U+FA11 は CJK 互換漢字 |
| `𠮷田` | 弾かれる | 「𠮷」U+20BB7 は範囲外（サロゲートペア） |
| `㐂` | 弾かれる | 「㐂」U+3402 は CJK 統合漢字拡張 A |

### 影響

mobile-registration は口座開設の入力画面のため、お客様が氏名・住所を**最初に登録する**場所になる。4556 で app-vuejs だけを直すと、次のような食い違いが起きる。

- 口座開設（mobile-registration）では「山﨑」で登録できない
- 登録後のお客様情報変更（app-vuejs `l2212`）では「山﨑」に変更できる

`verifyInputText` の呼び出し箇所（漢字を許可する type のみ）：

| 画面 | type | `checkJpText` の呼び出し |
| --- | --- | --- |
| `views/PersonalResiger/BasicInfo/nameBirthdaySex.vue` | 1 | あり |
| `views/ResetResiger/Individual/nameBirthdaySex.vue` | 1 | なし |
| `views/PersonalResiger/AddressInfo/address.vue` | 3 | なし |
| `views/ResetResiger/Individual/address.vue` | 3 | なし |
| `components/Container/cAddress.vue` | 3・7 | あり |
| `views/Resiger/components/ruleAddress.vue` | 3・7 | あり |
| `views/PersonalResiger/AddressInfo/nationalty.vue` | 4 | なし |
| `views/ResetResiger/Individual/nationalty.vue` | 4 | なし |
| `views/PersonalResiger/Occupation/employmentName.vue` | 5 | なし |
| `views/PersonalResiger/Invest/desiredInvestmentType.vue` | 5 | なし |
| `views/Resiger/BasicInfo/corporateNameNumber.vue` | 11 | あり |
| `views/Resiger/Represent/representativeInfo.vue` | 1 | あり |
| `views/Resiger/Transaction/rulerInfo.vue` | 1・11 | あり |
| `views/Resiger/Transaction/transactionManager.vue` | 1 | あり |
| `views/PersonalResiger/Insider/insiderFlg.vue` | 12 | なし |
| `views/PersonalResiger/Insider/fundInsiderFlg.vue` | 12 | なし |
| `views/Resiger/Transaction/corporateFundInsiderFlg.vue` | 12 | なし |

## 対応内容(期待結果)

方針は 4556 と同じ（2026-09-30 簡さんのコメント：「フロントでは緩くチェックして入り口を広く受け、最終的には審査で保振制度内字に変更する」）。

1. **許可する範囲を広げる**（type 1・3・4・5・7・9・10・11・12 のすべて）。範囲は 4556 と揃える
    - CJK 統合漢字拡張 A：U+3400〜U+4DBF
    - CJK 互換漢字：U+F900〜U+FAFF
    - CJK 統合漢字拡張 B 以降・CJK 互換漢字補助：U+20000〜U+3FFFF
2. **実装上の注意**（4556 と同じ）
    - サロゲートペアを1文字として扱うため、正規表現に `u` フラグを付ける。`.babelrc` は app-vuejs と同じ構成（babel 6 `babel-preset-env`）で、`babel-plugin-transform-es2015-unicode-regex` はインストール済み。ビルド後の出力は実装時に確認する
    - `u` フラグを付けると、type10・11 の文字クラス内の不要なエスケープ（`\&` `\'` `\·` `\＆` `\’` など）が構文エラーになるため、エスケープを外す
    - 差分を追いやすくするため、9つの正規表現それぞれに範囲を追記するにとどめ、共通化は行わない
    - app-vuejs とは type4 の正規表現が少し異なる（mobile-registration は `-　` も許可している）。範囲の追記だけを行い、この差分は変えない
3. **単体テスト**
    - mobile-registration には `test/unit/specs/HelloWorld.spec.js` しかなく、Jest が動く状態かは未確認。app-vuejs では HASHST\_DEV-4526 で実行環境を整備しているので、同じ整備が必要かを先に確認する
    - app-vuejs の `test/unit/specs/utils/common.verifyInputText.spec.js`（4542・4556）をもとに、「漢字の範囲」と type10・11 の記号のテストを用意し、Red → Green の順で対応する
4. **testing 環境での確認**
    - 口座開設の氏名（`nameBirthdaySex`）で「山﨑」が登録でき、審査（company-admin）で保振制度内字の変換対象になることを確認する
    - 「𠮷」は `checkJpText` を呼ぶ画面ではエラー `E103-1045`「𠮷の入力はできません。ひらがなで入力してください。」で弾かれることを確認する
    - `checkJpText` を呼ばない画面（例：`address.vue`、`insiderFlg.vue`）では「𠮷」がサーバーまで送られる。DB エラーにならず保存・表示できることを確認する（履歴テーブルは `CHARSET=utf8` で作られるため、4バイト文字の保存で失敗しないかを見る）。失敗した場合は別チケットで対応方針を決める
5. **今回の対象外**
    - 異体字セレクタ（IVS、U+E0100〜）付きの字（4556 と同じ）
    - `checkJpText`（SJIS-win チェック）とエラー文言の変更（common-front-api 側）

## 対応根拠となるドキュメント

- [HASHST\_DEV-4556](https://hashdash.backlog.com/view/HASHST_DEV-4556)（app-vuejs 側の同じ対応。調査結果・3段階のチェックの説明はこちら）
- 方針の根拠：簡さんのコメント [https://hashdash.backlog.com/view/HASHST\_DEV-4556#comment-822040432](https://hashdash.backlog.com/view/HASHST_DEV-4556#comment-822040432)

## 改修差分

【対応後記入】

## 結合環境確認履歴

【対応後記入】
