---
title: 'Hono の核となる 5 ファイルを読んで、Hono を理解した気になろう！'
emoji: '🔥'
type: 'tech'
topics: ['hono', 'typescript', 'nodejs', 'javascript']
published: false
---

先日こんな動画が Youtube にアップされていました。

https://www.youtube.com/watch?v=lgy0FKho2yI

この動画の中で、 Hono 作者 の Yusuke Wada 氏が

> 本当に核になるコードって、hono-base.ts っていうのと、compose.ts っていうのと、 context.ts と、 request.ts か。
> で、この 4 つのファイルだけなんだよねコア。

と発言されており、「そんなん中身読むしかないですやん...」と思い読んでみることにしました。

ちなみに私は hono 初心者ですので、同じく初心者の皆さんは是非一緒に読んでみましょう。

## compose.ts

https://github.com/honojs/hono/blob/main/src/compose.ts

まずは一番短い `compose.ts` から読んでいきます。

jsdoc に記載されている通り、compose 関数は middlewares をチェーン化する役割を担っているようです。

```ts
/**
 * Compose middleware functions into a single function based on `koa-compose` package.
```

`compose` の型シグネチャを見てみましょう。

```ts
export const compose = <E extends Env = Env>(
  middleware: [[Function, unknown], unknown][] | [[Function]][],
  onError?: ErrorHandler<E>,
  onNotFound?: NotFoundHandler<E>,
): ((context: Context, next?: Next) => Promise<Context>) => {
  /* ... */
};
```

ミドルウェア配列、エラーハンドラ、NotFound ハンドラを受け取って、 コンテキストと Next() を返しています。
実際の使われかたとしては、

```ts

```
