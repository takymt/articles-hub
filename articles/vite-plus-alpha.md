---
title: 'vite+を試してみる！'
emoji: '⚡️'
type: 'tech'
topics: ['vite', 'typescript', 'rolldown', 'oxlint', 'oxfmt']
published: false
---

2026年3月13日、ついに[Vite Plus](https://viteplus.dev/)が Alpha リリースされましたね。

しかも事前のプレスとは異なり、なんとMITオープンソースでの発表でした。

https://www.youtube.com/watch?v=AJxH3Gvb-PU

構想が話題になってから約半年くらいでしょうか？
さっそく触ってみたいと思います。

## Vite Plus おさらい

https://voidzero.dev/posts/announcing-vite-plus-alpha#what-is-vite

- Vite, Vitest, Oxlint, Oxfmt, Rolldown, tsdownを統合したツールチェーン
- 新たなタスクランナー[Vite Task](https://github.com/voidzero-dev/vite-task)も同梱
- Rust製で高速
  - Vite + Rolldown: RollupベースのVite 7よりも最大1.6-7.7倍高速な商用ビルド
  - Oxlint: ESLintよりも最大50-100倍高速なESLint互換リンター
  - Oxfmt: Prettierよりも最大30倍高速なPrettier互換フォーマッタ

## さわってみよう

### インストール

[インストール](https://viteplus.dev/guide/#install-vp) すると、NodeJS のバージョン管理を Vite Plus に任せるかを聞かれます。

```console
❯ curl -fsSL https://vite.plus | bash

Setting up VITE+...

Would you want Vite+ to manage Node.js versions?
Press Enter to accept (Y/n):
```

Node のバージョン管理をやってくれるみたいですね。せっかくなので `Y` でいきましょう。

- `node` / `npm` / `npx` の shim を `.node-version` または `package.json` のバージョンに解決する
- `vp env pin|unpin` で `.node-version` に node バージョンを固定

という挙動のようです（[参考](https://viteplus.dev/guide/env)）。

また `vp env current` で使用中 shim の各バージョンが確認できます。

```console
❯ vp env current
VITE+ - The Unified Toolchain for the Web

Environment:
  Version  24.14.0
  Source   lts

Tool Paths:
  node  /Users/takuto-yamamoto/.vite-plus/js_runtime/node/24.14.0/bin/node
  npm   /Users/takuto-yamamoto/.vite-plus/js_runtime/node/24.14.0/bin/npm
  npx   /Users/takuto-yamamoto/.vite-plus/js_runtime/node/24.14.0/bin/npx
```

このあたりの設定は `$HOME/.vite-plus/bin` に保存されるようです。

### プロジェクト作成

`vp create` でプロジェクトを init します。

```console
❯ vp create
VITE+ - The Unified Toolchain for the Web

  › Vite+ Monorepo: Create a new Vite+ monorepo project
    Vite+ Application: Create vite applications
    Vite+ Library: Create vite libraries
```

今回は `Monorepo` を作ってみましょう

```console
❯ vp create
VITE+ - The Unified Toolchain for the Web

  Vite+ Monorepo

◇ Package name:
  vite-plus-playground

◇ Which package manager would you like to use?
  pnpm

◇ Which agents are you using?
  ChatGPT (Codex), Claude Code

◇ Which editor are you using?
  None

◇ Set up pre-commit hooks to run formatting, linting, and type checking with auto-fixes?
  Yes

◑  Creating monorepo (4s)
```

これにより以下のようなディレクトリ構成が scaffold されます。いい感じに攻めてますね。

```console
.
├── .gitignore
├── .vite-hooks
│   ├── _/
│   └── pre-commit
├── AGENTS.md
├── CLAUDE.md
├── README.md
├── apps
│   └── website
│       ├── index.html
│       ├── node_modules/
│       ├── package.json
│       ├── public
│       │   ├── favicon.svg
│       │   └── icons.svg
│       ├── src
│       │   ├── assets
│       │   │   ├── hero.png
│       │   │   ├── typescript.svg
│       │   │   └── vite.svg
│       │   ├── counter.ts
│       │   ├── main.ts
│       │   └── style.css
│       └── tsconfig.json
├── node_modules/
├── package.json
├── packages
│   └── utils
│       ├── node_modules/
│       ├── package.json
│       ├── src
│       │   └── index.ts
│       ├── tests
│       │   └── index.test.ts
│       ├── tsconfig.json
│       ├── tsdown.config.ts
│       └── vite.config.ts
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
├── tsconfig.json
└── vite.config.ts
```

### プロジェクト依存関係のインストール

pnpm ライクに依存関係をインストールできます。

```sh
vp install
vp install --frozen-lockfile
vp install --filter web
vp install -w
```

### 開発サーバの起動

`vp dev` で開発サーバを起動できます。
今回はモノレポなので、タスクランナー経由の `vp run dev` で起動します。

![vp-dev-server](/images/vite-plus-alpha/vp-dev-server.png)

### リンタ/フォーマッタ

`vp check [--fix]` で各種チェックとその fix が可能です

- `oxfmt` によるフォーマット
- `oxlint` によるリント
- `tsgolint` による type-aware なリント (ついでに型チェック)

```console
❯ vp check
VITE+ - The Unified Toolchain for the Web

pass: All 19 files are correctly formatted (205ms, 8 threads)
pass: Found no warnings, lint errors, or type errors in 7 files (213ms, 8 threads)
```

#### 現時点での互換性

便利ツールが出てきた時に気になるのはやはり互換性です。

2026年3月15日時点だと、

- `oxfmt` vs `prettier`
  - 純粋な JS / TS のフォーマットは対応済み ([参考](https://oxc.rs/docs/guide/usage/formatter.html#prettier-compatible))
  - プラグインについては未対応だが、`sortImports` / `sortTailwindcss` / `sortPackageJson` がビルトインで存在 ([参考](https://oxc.rs/docs/guide/usage/formatter/unsupported-features.html#prettier-plugins))
- `oxlint` + `tsgolint` vs `eslint`
  - `typescript-eslint` のルールは [59 / 61](https://github.com/oxc-project/tsgolint/tree/main?tab=readme-ov-file#implemented-rules) が `tsgolint` によってカバー済み
  - ただし `tsgolint` 自体が TypeScript 7.0+ を前提にしている
  - [JS Plugin の API 対応はまだ Alpha](https://oxc.rs/docs/guide/usage/linter/js-plugins.html) であり、特に Vue とか Svelte のようなフレームワークのプラグインは [絶賛対応中](https://github.com/oxc-project/oxc/issues/19918)

正確な情報はその都度確認してほしいですが、自分は以下の感想を持ちました。

- `oxfmt`: 現時点で既にアリ
  - 知らないうちに互換がかなり進んでいる！
  - `prettier` プラグイン依存または一部のエッジケースを踏まない限りは安定して採用できそう
- `oxlint`: type-aware な場合はまだ厳しい
  - ビルトインプラグインで対応できる構成であれば移行可能性もあるが、エッジケースも多く `oxfmt` ほどの安定性はない。
  - [`eslint` との併用パターン](https://oxc.rs/docs/guide/usage/linter/migrate-from-eslint.html#running-oxlint-and-eslint-together) もあるが、eslint 専用ルールがボトルネックだと改善幅は小さい
  - type-aware なルールについては TypeScript 7.0+ が前提とされているので、TS 6 以下 (つまり現行のほとんど) のプロジェクトにとっては移行コストがまだ大きい
    - 一応 `baseUrl` などの一部のレガシー設定を使用しなければ動きはするっぽい
- `vp-check` 自体

### テスト

`vp test` で vitest によるテスト実行が可能です。

vitest 自体は既にまあまあ枯れていると思うのであまり言及することがないですね。

### タスク定義

タスクランナーである `vp run` も同梱されています。

```json
{
  "scripts": {
    "build": "node compile-legacy-app.js",
    "test": "jest"
  }
}
```

`package.json.scripts` を `vp run build` のように呼び出すことができます。

#### タスク実行キャッシュ

`vp run` はキャッシュ付きで、引数/環境変数/入力ファイルが同じなら再実行せず即座に再生可能です。

`vite.config.ts` がキャッシュONなのに対して、`package.json.scripts` だとキャッシュがOFFになっています。

`vite.config.ts` だと `cache` 有無やトラックする環境変数なども制御できるようですので、こっちがファーストチョイスになるかもしれません。

#### タスク依存関係

`vite.config.ts` 経由でタスク間の依存関係を定義し、スクリプトをシンプルに保てます

```ts
import { defineConfig } from 'vite-plus';

export default defineConfig({
  run: {
    tasks: {
      build: {
        command: 'vp build',
        dependsOn: ['lint'],
        env: ['NODE_ENV'],
      },
      deploy: {
        command: 'deploy-script --prod',
        cache: false,
        dependsOn: ['build', 'test'],
      },
    },
  },
});
```

これはいいですね。DAG は `npm scripts` だと弱いところだったと思います。

### ビルド

vite 8 が dev/prod ビルドともに rolldown になり、`vp` もその恩恵を受けることになりました

```sh
# 商用ビルド
vp build
# プレビュー
vp preview
```

詳しくは [Vite 公式情報](https://vite.dev/config/) を参照して下さい

### CI

GitHub Actions に [voidzero-dev/setup-vp](https://viteplus.dev/guide/ci) が提供されています

パッケージマネジャーや Node 自体の管理が不要になったことで、 `setup-vp` アクションで解決するようになっています。

```yml
- uses: voidzero-dev/setup-vp@v1
  with:
    node-version: '24'
    cache: true

- run: vp install && vp run dev:setup
- run: vp check
- run: vp test
```

### pre-commit

`vp config` により、`.vite-hooks` を hook ディレクトリにし、pre-commit を登録することができます。

また、`vp staged` により lint-staged 相当のことが可能です

```ts
import { defineConfig } from 'vite-plus';

export default defineConfig({
  staged: {
    '*.{js,ts,tsx,vue,svelte}': 'vp check --fix',
  },
});
```

## 個人的まとめ

Alpha 時点の Vite Plus を触ってみたまとめです。

- Vite, Vitest, Oxlint, Oxfmt, Rolldown, tsdownを統合した次世代 JS ツールチェーン Vite Plus がついに Alpha リリース
- 既存互換ツールと比較するとどれも非常に高速で開発体験がいい
- staged やタスクランナーなど、かゆいところに手が届く
- まだ Alpha なので当然だが、商用で使えるほど安定ではない (特に oxlint や tsgolint)
- [この手のツールは過去にもあった](https://github.com/rome/tools) が挫折した。二の舞いにならないよう、各位できる範囲でのコントリビュートで支援していきましょう！