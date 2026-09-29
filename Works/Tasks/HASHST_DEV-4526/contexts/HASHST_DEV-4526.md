単体テスト実行環境の整備

# 詳細

## 背景

現在 `app-vuejs` では `npm run unit` が1本も通らない状態になっている。vue-cli の雛形が残ったままで、以下2件がいずれも失敗する。

-   `test/unit/specs/HelloWorld.spec.js` が存在しない `@/components/HelloWorld` を参照している
-   E2E 用の `test/e2e/specs/test.js` が Jest に拾われ、「テストが0件」で落ちる

段階的に単体テストを追加していく方針（Phase 0〜4）を立てており、本チケットはその **Phase 0＝テストが走る状態をつくる** 部分にあたる。テストの中身は次チケット以降（Phase 1）で追加する。

## 前提

-   実行環境は **Node 22** とする
-   **ライブラリのバージョンは上げない**（Jest 22.4.4 / Babel 6 / webpack 3 のまま）
-   Node 22.13.1 上で Jest 22 が問題なく動作することは検証済み
-   新規の npm パッケージ追加は不要（検証済み）

## 作業内容

### 1\. 雛形の削除

-   `test/unit/specs/HelloWorld.spec.js` を削除する

### 2\. `test/unit/jest.conf.js` の修正

-   `testPathIgnorePatterns: ['<rootDir>/test/e2e']` を削除し、代わりに対象を明示指定する

```
testMatch: ['<rootDir>/test/unit/specs/**/*.spec.js'],
```

> 補足: `testPathIgnorePatterns` の値は正規表現として解釈されるため、チェックアウト先のパスに正規表現の記号が含まれると除外が効かない（ローカルの検証環境ではパスに `+` が含まれており、実際に E2E 側を拾ってしまった）。対象を明示指定する方が環境に依存しない。

-   Jest 22 で撤去済みの `mapCoverage: true` を削除する（現状は毎回 deprecation 警告が出ている）

### 3\. router / store のスタブ化

`src/utils/common.js` は冒頭で `../router`（1,596行・184画面を import）と `../store` を読み込んでいるため、純粋関数を1つテストするだけでアプリ全体が起動してしまう。これを `moduleNameMapper` で切り離す。

-   `test/unit/stubs/router.js` を新規作成

```
export default {
  push: jest.fn(),
  replace: jest.fn(),
  currentRoute: { path: '/', query: {} }
}
```

-   `test/unit/stubs/store.js` を新規作成

```
export default {
  commit: jest.fn(),
  dispatch: jest.fn(),
  state: { userInfo: {} }
}
```

-   `jest.conf.js` の `moduleNameMapper` に以下を追加する。**`'^@/(.*)$'` より前に置くこと**（後ろだと先にマッチしてしまい効かない）

```
moduleNameMapper: {
  '^@/router$':      '<rootDir>/test/unit/stubs/router.js',
  '^@/store$':       '<rootDir>/test/unit/stubs/store.js',
  '^\\.\\./router$': '<rootDir>/test/unit/stubs/router.js',
  '^\\.\\./store$':  '<rootDir>/test/unit/stubs/store.js',
  '^@/(.*)$':        '<rootDir>/src/$1'
}
```

> 補足: このスタブ化により `vue-router` / `vuex` / `axios` は不要になる（これらを経由しているのが router と store だけのため）。実際に `common.js` に対する試作テスト13ケースを 1.2 秒で完走させて確認済み。

### 4\. スモークテストを1本追加

設定変更だけだと対象の spec が0件になり、`npm run unit` が終了コード 1 で落ちる（`--passWithNoTests` を付ける手もあるが、常に緑になる無意味なパイプラインになるため採用しない）。

-   `test/unit/specs/utils/common.verifyInputText.spec.js` を新規作成し、`verifyInputText` のテストを1ファイルだけ書く
-   **`test.each` は Jest 23 以降の機能で 22 には存在しない**ため、テーブル駆動は素の `forEach` で書く。この書き方を以降の標準とする

```
import common from '@/utils/common'

describe('verifyInputText（氏名 type1）', () => {
  const cases = [
    ['山田太郎',     1, true],
    ['ヤマダタロウ', 1, true],
    ['山田 太郎',    1, false],  // 半角スペースは type1 では不可
    ['山田1',        1, false]
  ]
  cases.forEach(([value, type, expected]) => {
    it(`${JSON.stringify(value)} / type${type} → ${expected}`, () => {
      expect(common.verifyInputText(value, type)).toBe(expected)
    })
  })
})
```

> 参考: Jest 22 では `test.each` / `describe.each` / `test.todo` / `toStrictEqual` / `toMatchInlineSnapshot` がいずれも未定義（実測）。`jest.spyOn` と `expect().resolves` は使用可。

### 5\. Node バージョンの固定

-   `.nvmrc` を新規作成し、`22` と記載する
-   `package.json` の `engines.node` を `>= 6.0.0` から `>=22` に更新する
-   `package-lock.json` を Node 22 の npm で生成し直してコミットする

> 補足: 現状 `package-lock.json` は lockfileVersion 2（npm 7〜8 が生成）で、新しい npm が触ると v3 に書き換わる。実際に未コミットの差分が 200 行発生している。ここで確定させておく。

### 6\. CI に単体テストジョブを追加

-   `.gitlab-ci.yml` の既存 `test` ステージ（宣言済みだがジョブが0件）に `unit_test` ジョブを追加する
-   イメージは `node:22-alpine` を明示指定する（既存の `deploy:testing` は浮動タグの `node:lts-alpine` を使っているが、テストジョブでは固定する）
-   リポジトリ全体で `GIT_STRATEGY: none` が設定されているため、このジョブでは **ソースを取得するよう上書きが必要**
-   実行は `npm ci && npm run unit`
-   本チケットの時点では全ブランチで実行し、失敗でパイプラインを止める

## 完了条件

-   `npm run unit` が Node 22 上で終了コード 0 で完了する
-   上記コマンドで E2E 配下（`test/e2e/`）が実行対象に含まれていない
-   deprecation 警告（`mapCoverage`）が出ない
-   MR のパイプラインで `unit_test` ジョブが実行され、成功している
-   `.nvmrc` / `engines` / `package-lock.json` が Node 22 に揃っている

## 影響範囲

-   **アプリケーションのコードには一切変更を加えない**（`src/` 配下の変更なし）
-   変更対象は `test/` 配下、`package.json`（`engines` のみ）、`package-lock.json`、`.nvmrc`、`.gitlab-ci.yml`
-   ビルド・デプロイの既存ジョブには手を入れない

## 本チケットに含めないこと

-   `src/utils/common.js` の他の関数のテスト追加 → Phase 1（別チケット）
-   共通コンポーネントのテスト → Phase 3（別チケット、着手前に技術検証が必要）
-   Jest / Babel / webpack のバージョン更新 → 対象外。Node 22 で現行構成が動くことを確認済み
-   E2E（Nightwatch）の整備 → 別テーマ

## 備考

-   テストのフィクスチャには実在の投資家情報（氏名・口座番号・本人確認情報）を使用せず、架空の値のみを用いる

マージされたのでclose  
[https://gitlab.hashdash-group.com/sto\_fund/app-vuejs/-/merge\_requests/1048](https://gitlab.hashdash-group.com/sto_fund/app-vuejs/-/merge_requests/1048)

単体テスト実行環境の整備
