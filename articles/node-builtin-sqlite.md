---
title: 'node:sqlite って何で今更作られてんの？'
emoji: '🕊️'
type: 'tech'
topics: ['sqlite', 'nodejs', 'database']
published: true
---

## TL;DR

- `node:sqlite` が 2026 年 2 月 22 日に Release candidate になったので、何で今なのか気になって調べた
- 背景には Web API 互換への需要と、既存ネイティブビルドライブラリへの課題感があった
- まだまだ experimental なライブラリだが、依存減らせてアツいので要チェックや！

## なんで今さら標準化？

https://nodejs.org/docs/latest/api/sqlite.html

[2026年2月22日、Node.js の組み込みライブラリとして `node:sqlite` が Release Candidate になりました。](https://github.com/nodejs/node/pull/61262#event-22969865069)

なんで今やねんと思ったので、その理由と背景を探って見ようと思います。

## 背景1: Web LocalStorage API 互換

ここ数年 Node.js では、Web API と互換性のある LocalStorage API の実装が進められています。

[LocalStorage の永続化層の実装として SQLite を使用する流れ](https://github.com/nodejs/node/pull/50169#issuecomment-1764543512) があり、 SQLite を Node.js の組み込み機能としてメンテする必要性が生じたことが、`node:sqlite` の元々の発端になっていると思われます。

### なぜ LocalStorage API に SQLite が必要か

LocalStorage API を標準機能として実装することは、 Node 標準の永続化層を実装することを意味します。

永続化層が実装された場合、複数のワーカが同じ永続ストアを読むようになるのは自然な拡張です。SQLite を使っておけばこの問題を回避可能であるため、ファイルとグローバルロックを用いて自力解決するよりもいい選択だと指摘されています ([Issue#52435](https://github.com/nodejs/node/pull/52435#issuecomment-2051909386))。

### なぜ　LocalStorage API 互換が必要か

LocalStorage API 互換のモチベーションの一つとして、[Hydration Mismatch](https://nextjs.org/docs/messages/react-hydration-error) への懸念が挙げられます。

今日の Web開発において、Isomorphic + Hydration という構成で SSR する手法は非常に一般的です。

- Isomorphic: ブラウザと実質的に同じコードを、サーバサイドでも動作させること
- Sever-Side Rendering (SSR): サーバサイドで HTML を生成すること
- Hydration: SSR された HTML に、ブラウザ側のフレームワークがイベントや状態といった動的要素を紐づけること

一方で、JS における Isomorphic + Hydration な開発には、Hydration Mismatch が起こらないための制御が伴います。

- `window` や `localStorage` といった Node.js 非互換の Web API をフォールバックする
- `Date` や `randomUUID` といった実行状況によって異なる API を使用しない
- 副作用は `useEffect` を用いてエスケープする
- ...etc

Node に LocalStorage API が存在しないことで、 hydration mismatch 対策としての `if (window === undefind)` のようなボイラープレート処理をユーザがもれなく記載する必要があり、長らくユーザ負担となっていました。

また、Node よりも比較的あたらしい JS ランタイムである Deno には [LocalStorage API](https://bun.com/reference/node/async_hooks/AsyncLocalStorage) が標準機能として搭載されており、相対的に Node の UX の悪さが指摘されています([Issue#50169](https://github.com/nodejs/node/pull/50169))。

このあたりが互換 API 実装のモチベーションになっているようです。

## 背景2: SQLite ネイティブビルド

npm エコシステムには、既に非常に安定しているサードパーティの sqlite ライブラリがあります。

- [node-sqlite3](https://www.npmjs.com/package/sqlite3)
- [better-sqlite3](https://github.com/WiseLibs/better-sqlite3)

しかしながらこれらは [SQLite](https://sqlite.org/) の C 言語ライブラリを叩くバインディングなので、 ユーザが自身でネイティブビルド(= C や C++ のコンパイル)を実行する必要がありますが、この結果は実行環境 (OS, CPU, コンパイラ等) に左右され得ります。

これが `node:sqlite` として標準化されることで、ネイティブビルドの責任は Node.js のバイナリ配布側に移ります。ユーザやサードパーティライブラリの開発者は、より安定したバイナリ依存先を手に入れられることになります。

ユーザ利便性および API 設計 (同期/非同期/トランザクション等) についての議論は [Issue#53264](https://github.com/nodejs/node/issues/53264) で議論されているので、興味のある方は読んでみてください。

## API はまだ同期が中心

https://nodejs.org/docs/latest/api/sqlite.html#class-sqltagstore

先述の [Issue#53264](https://github.com/nodejs/node/issues/53264) での議論の通り、 現時点(Node.js v25.7.0) では同期 API を中心に提供されています

基本的な Class は [`DatabaseSync`](https://nodejs.org/docs/latest/api/sqlite.html#class-databasesync) と [`StatementSync`](https://nodejs.org/docs/latest/api/sqlite.html#class-statementsync) の2つで、一通り API は揃っています。

```js
const { DatabaseSync } = require('node:sqlite');
const database = new DatabaseSync(':memory:');

// 文字列からSQLを実行
database.exec(`
  CREATE TABLE data(
    key INTEGER PRIMARY KEY,
    value TEXT
  ) STRICT
`);

// prepared statement 作成して実行
const insert = database.prepare('INSERT INTO data (key, value) VALUES (?, ?)');
insert.run(1, 'hello');
insert.run(2, 'world');
```

また、少し独特？なAPIとして [`SQLTagStore`](https://nodejs.org/docs/latest/api/sqlite.html#class-sqltagstore) があります。

これは プレースホルダを安全に活用しつつ、 Prepared Statement を LRU キャッシュで再利用してくれる仕組みになっていて、

```js
// prepared statement キャッシュの作成
const sql = db.createTagStore();

// 同じ statement になる場合はキャッシュヒット & prepare をスキップ
sql.get`SELECT * FROM t1 WHERE id = ${id} AND active = 1`;
sql.get`SELECT * FROM t1 WHERE id = ${12345} AND active = 1`;

// これは別物扱いなので prepare -> キャッシュ -> 実行
sql.get`SELECT * FROM t1 WHERE id = ${id} AND active = 1`;
sql.get`SELECT * FROM t1 WHERE id = 12345 AND active = 1`;

// case-sensitive なので別物扱い
sql.get`SELECT ...`;
sql.get`select ...`;
```

仕様知らないと寧ろ SQLi できそうに見えるのは自分だけだろうか...。

なお、Async API や Transaction API は 現在も活発に議論が進んでいるようです([#54307](https://github.com/nodejs/node/issues/54307), [#57431](https://github.com/nodejs/node/issues/57431))

## ベンチマーク

https://github.com/takymt/node-builtin-sqlite-bench

同期な時点で自分の利用範囲だと誤差ですが、一応取ってみました。

200,000行 の sqlite データベース に対して 10 回クエリの平均値(ms)を計算しています

```
Query            node:sqlite  node:sqlite(cached)  better-sqlite3
--------------   -----------  -------------------  --------------
WHERE id = ?     0.0072       0.0053               0.0021
LIMIT 10         0.0083       0.0074               0.0047
LIMIT 1000       0.5508       0.4210               0.3188
(all 200k rows)  51.92        47.54                32.66
```

環境に左右される値なのであくまで参考値ですが、`better-sqlite3` のほうが最適化を謳っている分速いです (知ってた)

キャッシュはちゃんと速そうですが、先述の「仕様知らないと寧ろ SQLi できそうに見える」問題があるので、性能に困らない限り自分は使わない気がします。

## まとめ

今回は 2026年2月22日に Rerease candidate になった `node:sqlite` について、その経緯と背景を調べてみました。

性能は `better-sqlite3` に劣りますが、ネイティブビルドの複雑性から解放されて依存を作らないのが非常に魅力的なので、自分個人レベルだとファーストチョイスになりそうです！

それではみなさま、よき SQLite ライフを！

（余談: この手の PR を Deep Reading するの、めっちゃ勉強になる）
