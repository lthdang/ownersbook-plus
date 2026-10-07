The ticket says f1200:202 never shows this label. That is mostly true, with one exception:
The f1200 block switch tests fundInfo.PERIOD == 'PRIMARY' (f1200.vue:123). So a PRIENTRY fund falls into the v-else block.
That block calls getInStockStr(PRIENTRY, '0', SELF_FUND_QTY_SETTLEMENT). If the quantity is 0, the 在庫状況 (stock status) row would show the label.
Since 仮募集 is unused, this doesn't matter in practice. I'm noting it here and not opening it as a question.


Thông tin ticket:
---
# 詳細

## 現状と背景

ファンドの募集終了時に「当初募集終了」と表示される。「当初」を削除し、「募集終了」と表示する。

文言は `src/utils/common.js:998` の `getInStockStr`（在庫・募集状況の表示文言を作る共通関数）の 1 箇所だけで定義されている。

```js
if(period=='PRIMARY' || period=='PRIENTRY'){
  if(ratio==0 && inStock == 0){
    str='当初募集終了'
```

- 表示される条件：募集区分（`PERIOD`）が `PRIMARY`（当初募集）または `PRIENTRY`（優先エントリー・仮募集）で、残り比率（`IN_STOCK_POSITION_EMPTY`）と残口数（`IN_STOCK_NUM`）がどちらも 0 のとき
- 文言は HASHST_DEV-1420（2022-05、`75c2be07`）で追加されてから変わっていない

### 表示される画面

| 画面 | 箇所 | 表示欄 |
|---|---|---|
| f1200 ST詳細：トップ（`PERIOD == 'PRIMARY'` のとき） | `f1200.vue:158` | 募集状況·期間 |
| f1202_pre 仮募集 ST詳細：トップ（プライマリ） | `f1202_pre.vue:110` | 募集状況・期間 |

- `f1200.vue:202`（在庫状況）も同じ関数を呼んでいるが、`PRIMARY` 以外のときのブロックのため、この文言は表示されない

### 影響範囲の確認結果

- app-vuejs の中で「当初募集終了」「募集終了」を含むのは上記の 1 箇所だけ（定数ファイル・`index.html` にはない）
- st-admin にも「当初募集終了」があるが、いずれも管理画面の項目名「当初募集終了日時」のエラーメッセージ・コメントで、投資家向けの表示ではないため対象外

## 対応内容(期待結果)

1. HASHST_DEV-4541 で追加したテストの期待値を `'当初募集終了'` から `'募集終了'` に書き換え、失敗することを確認する（Red）
   - `test/unit/specs/utils/common.getInStockStr.spec.js` の「当初募集（PRIMARY / PRIENTRY）」の 3 ケース
2. `src/utils/common.js` の文言を `'募集終了'` に修正し、テストが通ることを確認する（Green）
3. 結合環境で、f1200 / f1202_pre の完売状態のファンドで「募集終了」と表示されることを確認する

### 確認事項

- 文言は `PRIMARY` と `PRIENTRY` で共通のため、仮募集（f1202_pre）の表示も「募集終了」になる。仮募集だけ別の文言にする場合は条件の分岐が必要
- この文言は「募集期間が終わった」ではなく「残口数が 0 になった（完売）」ときに表示されるため、募集期間中でも完売すれば「募集終了」と表示される。この意味で問題ないかを確認する
- HASHST_DEV-4558（同じ `getInStockStr` の null・undefined の扱い）と同じ関数・同じテストファイルを触るため、並行して進める場合は競合に注意する

## 対応根拠となるドキュメント

- HASHST_DEV-4541（`getInStockStr` の単体テスト追加）
- HASHST_DEV-4558

## 改修差分

【対応後記入】

## 結合環境確認履歴

【対応後記入】
---
Link MR: https://gitlab.hashdash-group.com/sto_fund/app-vuejs/-/merge_requests/1057

