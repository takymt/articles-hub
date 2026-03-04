---
title: 'Hono の核となる 5 ファイルを読んで、Hono を理解した気になろう！'
emoji: '🔥'
type: 'tech'
topics: ['hono', 'typescript', 'nodejs', 'javascript']
published: false
---

先日 [Hono](https://hono.dev/) のドキュメンタリーが Youtube にアップされていました。

https://www.youtube.com/watch?v=lgy0FKho2yI

この動画の中で、 Hono 作者 の Yusuke Wada 氏が

> 本当に核になるコードって、hono-base.ts っていうのと、compose.ts っていうのと、context.ts と、request.ts か。で、この 4 つのファイルだけなんだよねコア。

と発言されており、せっかくなので読んでみることにしました。

ちなみに私は hono すげーーーと思ってる一般通過初心者ですので、同じく初心者の皆さんは是非一緒に読んでみましょう。

:::message
本記事中に登場する hono のソースコードは執筆時のスナップショットです。
また、説明のために一部を組み替えていますのでご了承下さい。
:::

## compose.ts

https://github.com/honojs/hono/blob/main/src/compose.ts

まずは一番短い `compose.ts` から読んでいきます。

### Koa-compose ベースのミドルウェアエンジン

compose.ts のメイン関数である `compose()` は [koa-compose](https://github.com/koajs/compose) の流れを組むオニオン系のミドルウェアエンジンです。

Koa / Hono の特徴を分けて考えるために、Koa の特徴を先に整理しておきましょう。

- 複数のミドルウェアを畳み込んで合成ミドルウェアを作る `fn = compose([a, b, c, ...])` 方式
- 各ミドルウェアは、リクエストコンテキスト `ctx` と、次のミドルウェアを実行する `next` 関数を引数にとる `mw = async (ctx, next)` 方式
- 各ミドルウェア内で、 `await next()` の前後に前処理と後処理が定義されるオニオン方式

```ts
const mwA = async (ctx, next) => {
  /* 前処理 -> */ await next(); /* 後処理 */
};
const mwB = async (ctx, next) => {
  /* 前処理 -> */ await next(); /* 後処理 */
};
const handler = async (ctx) => {
  /* 終端処理 */
};

// mwA前処理 -> mwB前処理 -> handler -> mwB後処理 -> mwA後処理 の順に呼ばれる
const composed = compose([mwA, mwB, handler]);
await composed(ctx);
```

Express のような callback スタイルとは異なり、async/await を活用した自然な後処理が書けます。

このスタイルは Hono 以外に、 [Egg.js](https://www.eggjs.org/intro/egg-and-koa) や [Middy.js](https://middy.js.org/docs/intro/how-it-works) にも取り入れられています。

### Hono ミドルウェアエンジンの特徴

コードから読み取れる Koa との差分をフックにして、Hono の雰囲気を掴みたいと思います。

`compose` の型シグネチャを見てみましょう。

```ts:src/compose.ts
export const compose = <E extends Env = Env>(
  middleware: [[Function, unknown], unknown][] | [[Function]][],
  onError?: ErrorHandler<E>,
  onNotFound?: NotFoundHandler<E>,
): ((context: Context, next?: Next) => Promise<Context>) => {
  /* ... */
};
```

いきなり特徴的です。

- `middleware` が単なるミドルウェアの配列ではない
- `onError` / `onNotFound` をミドルウェアエンジンの引数として処理する
- 返り値の関数がミドルウェアの実行結果ではなく `Context` をそのものを返すようになっている

一つずつ見ていきましょう。

### `middleware` が単なるミドルウェア配列ではない

`compose()` の実際の使われ方を見てみると、ルータでマッチした `matchResult` が `middlewares` として渡されています。

```ts:src/hono-base.ts
const matchResult = this.router.match(method, path)
// 中略
compose(matchResult[0], this.errorHandler, this.#notFoundHandler);
```

ルータのマッチ結果である `Result` 型 には丁寧な jsdoc がついているので読んでみましょう。

````ts:src/types.ts
/**
 * Type representing the result of a route match.
 *
 * The result can be in one of two formats:
 * 1. An array of handlers with their corresponding parameter index maps, followed by a parameter stash.
 * 2. An array of handlers with their corresponding parameter maps.
 *
 * Example:
 *
 * [[handler, paramIndexMap][], paramArray]
 * ```typescript
 * [
 *   [
 *     [middlewareA, {}],                     // '*'
 *     [funcA,       {'id': 0}],              // '/user/:id/*'
 *     [funcB,       {'id': 0, 'action': 1}], // '/user/:id/:action'
 *   ],
 *   ['123', 'abc']
 * ]
 * ```
 *
 * [[handler, params][]]
 * ```typescript
 * [
 *   [
 *     [middlewareA, {}],                             // '*'
 *     [funcA,       {'id': '123'}],                  // '/user/:id/*'
 *     [funcB,       {'id': '123', 'action': 'abc'}], // '/user/:id/:action'
 *   ]
 * ]
 * ```
 */
export type Result<T> = [[T, ParamIndexMap][], ParamStash] | [[T, Params][]]
````

どうやら特定のパス (e.g. `/user/:id/:action`) に対するパスパラメータの持ち方として、

- `{'id': 0, 'action': 1}` と `['123', 'abc']` から `{'id': '123', 'action': 'abc'}` を復元するパターン
- `{'id': '123', 'action': 'abc'}` を直接持つパターン

があるようですね。`Result` が Union 型になっているのはそのためのようです。

また先程の `matchResult` は `Result<[H, RouterRoute]>` 型なので、あわせて `RouterRoute` 型も見ておきましょう。

```ts:src/types.ts
export interface RouterRoute {
  basePath: string
  path: string
  method: string
  handler: H
}
```

リクエストに対してどのルートがマッチしたかのメタデータが格納されているようです。

ここまでの `middlewares` の型の理解を整理しましょう。

```ts
// 実際の実装
middleware: [[Function, unknown], unknown][] | [[Function]][],

// 意味的な理解
middleware: [[H, RouterRoute], ParamIndexMap | Params][] | [[Function]][]
```

みたいな感じでしょうか。

Koa-compose に見られるような、基本的な `fn = compose([a, b, c, ...])` 方式との差分責務として、

- マッチしたルートのメタデータをあわせて持っている
- ハンドラごとのパスパラメータをあわせて持っている
- パスパラメータの持ち方が2種類ある
- 実際の `middleware` は、 `Function` や `unknown` (あるいは `[[Function]][]` ) といった緩い型で定義される

が見えてきました。ちょっと解像度が上がってきましたね。

#### なぜマッチしたルートのメタデータをあわせて持っている？

ここら辺の細かい所は、コードを読むよりも `git blame` して issue を見るほうが早そうです。

この変更が取り入れられた [Issue#1736](https://github.com/honojs/hono/issues/1736) によると、

- ユーザは実際の `/users/123` ではなく、ノイズの少ない `/users/:userId` をロギングしたかった
- これを実装するために、「各ミドルウェアがどのルートにマッチして実行されているか」を特定する必要があった
- マッチしたルート一覧を配列で持ち、ミドルウェア実行時に `context.req.routeIndex` を更新する形で参照可能にした
- 「現在実行中のミドルウェア」と「最後に実行したミドルウェア」に対応するルートのパスが取れるようになった

という流れのようです。
結果的に、`RouterRoute` とハンドラ/ミドルウェアを紐づける形でシグネチャが更新されました。

#### なぜハンドラごとのパスパラメータをあわせて持っている？

関連する [Issue#1542](https://github.com/honojs/hono/issues/1542) によると、

- ユーザは `/:type/:url` と `/foo/:type/:url` のように、同じ位置で異なるパスパラメータ名を処理したかった
- これらのパスパラメータの衝突は `express` や `itty-router` では許容されるが、当時の Hono ではエラーになっていた
- この対応のために、ハンドラごとにパスパラメータを持つようにした
- `RegExpRouter` の性能を考慮し、オブジェクトではなく配列のまま併せ持つ事になった `[Function, ParamIndexMap | Params][]`

という流れのようです。
可読性よりも性能を優先させたのは、[ルータ速度は Hono のコアである](https://hono.dev/docs/concepts/routers) ためだと思われます。

#### なぜパスパラメータの持ち方が2種類ある？

関連する [Issue#1566](https://github.com/honojs/hono/pull/1566) によると、

- `ParamIndexMap` と `ParamStash` は `RegExpRouter` 用の最適化
- 他のルータはより可読性の高い `Params` オブジェクトを使用

という流れのようです。
[`RegExpRouter is the fastest router in the JavaScript world`](https://hono.dev/docs/concepts/routers#regexprouter) と謳われている `RegExpRouter` の歴史において、重要な PR に見えますね。

ルータ周りの話はそれだけで 1 本記事が書けそうなので、今はここまでの理解にとどめておこうと思います。

#### なぜ実際の `middleware` は、緩めの型で定義されている？

関連する [Issue#3663](https://github.com/honojs/hono/pull/3663) を見ると、

- ミドルウェア合成用の高レベルユーティリティである `every()` の実装にも `compose()` が使用されていた
- `every()` の実装では実質的にハンドラ部分のみが必要だったが、型都合で残りの部分も形式的に補う必要があった
- Koa-compose 由来の本来の役割である「複数のミドルウェア関数を畳み込む」に小さく対応できるように型を緩くした

という流れに見えます（ここは自分の予想も込みです）。

TypeScript は Structural Typing なので、`compose()` の意味論的にもインターフェースを広めに取ってるみたいな感じなんですかね。

### `onError` / `onNotFound` をミドルウェアエンジンの引数として処理する

Koa-compose では `compose()` の役目はミドルウェアの純粋な合成でしたが、Hono ではその責務がエラーハンドリングと終端制御に拡大しています。

[`onError` が取り入れられたコミット](https://github.com/honojs/hono/pull/111/changes/4b81bdb2a17ff72cf0326b3483b691fde43c016c) では、

```diff
- throw errors[0] // XXX
+ if (onError && context instanceof Context) {
+   context.res = onError(errors[0], context)
+ } else {
+   throw errors[0]
+ }
```

エラーを `throw` するのではなく、`Context` に埋め込む方式に変わっています。

### 返り値の関数がミドルウェアの実行結果ではなく `Context` をそのものを返すようになっている
