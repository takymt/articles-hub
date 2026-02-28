---
title: 'node:sqlite って何で今更作られてんの？'
emoji: '🕊️'
type: 'tech'
topics: ['sqlite', 'nodejs', 'database', 'javascript', 'typescript']
published: true
---

## TL;DR

- [`node:sqlite`](https://nodejs.org/docs/latest/api/sqlite.html) が [Node.js v25.7.0 で Stability 1.2 Release candidate になった](https://nodejs.org/en/blog/release/v25.7.0) ので、何で今？と思い気になって調べた
- 背景には 「Web API 互換への需要」と、「既存ネイティブアドオンへの課題感」があった
- まだ発展中ライブラリだが、依存もビルド手間も減らせてアツい。要チェックや！

## なんで今さら標準化？

https://nodejs.org/docs/latest/api/sqlite.html

2026 年 2 月 24 日、Node.js v25.7.0 において、 Node.js の組み込みライブラリとして `node:sqlite` が RC になりました。

なんで今やねんと思ったので、その理由と背景を探って見ようと思います。

## 背景1: Web LocalStorage API 互換

ここ数年 Node.js では、Web API と互換性のある LocalStorage API の実装が進められています。

[LocalStorage の永続化層の実装として SQLite を使用する流れ](https://github.com/nodejs/node/pull/50169#issuecomment-1764543512) があり、 SQLite を Node.js の組み込み機能としてメンテする必要性が生じたことが、`node:sqlite` の元々の発端になっているようです。

### じゃあなんで LocalStorage API に SQLite が必要なの？

LocalStorage API を標準機能として実装することは、 Node 標準の永続化層を実装することを意味します。

永続化層が実装された場合、複数のワーカが同じ永続ストアを読むようになるのは自然な拡張です。それをファイルとグローバルロックを用いて自力で解決するよりも、SQLite 使って回避しようという流れのようです ([Issue#52435](https://github.com/nodejs/node/pull/52435#issuecomment-2051909386))。

### じゃあなんで　LocalStorage API 互換が必要なの？

LocalStorage API 互換の意義の一つとして、[Hydration Mismatch](https://nextjs.org/docs/messages/react-hydration-error) 対応コストの削減が挙げられます。

今日の Web開発において、Isomorphic + Hydration という構成で SSR する手法は非常に一般的です。

- Isomorphic: ブラウザと実質的に同じコードを、サーバサイドでも動作させること
- Sever-Side Rendering (SSR): サーバサイドで HTML を生成すること
- Hydration: SSR された HTML に、ブラウザ側のフレームワークがイベントや状態といった動的要素を紐づけること

一方で、JS における Isomorphic + Hydration な開発には、Hydration Mismatch が起こらないための制御が伴います。

- `window` や `localStorage` といった Node.js 非互換の Web API をフォールバックする
- `Date` や `randomUUID` といった実行状況によって異なる API を使用しない
- 副作用は `useEffect` を用いてエスケープする
- ...etc

Node に LocalStorage が存在しないことで、 SSR や hydration を伴うコードでは `if (typeof window === undefined)` みたいなボイラープレートを書かねばならず、地味にユーザ負担となっています。

別の観点として、Node よりも比較的新しい JS ランタイムである Deno には [LocalStorage API](https://docs.deno.com/api/web/~/localStorage) が標準機能として搭載されており、相対的に Node の UX の悪さが指摘されています([Issue#50169](https://github.com/nodejs/node/pull/50169))。

このあたりが互換 API 実装のモチベーションになっているようです。

## 背景2: SQLite ネイティブアドオン

npm エコシステムには、既に非常に安定しているサードパーティの sqlite ライブラリがあります。

- [node-sqlite3](https://www.npmjs.com/package/sqlite3)
- [better-sqlite3](https://github.com/WiseLibs/better-sqlite3)

しかしながらこれらは [SQLite](https://sqlite.org/) の C ライブラリを叩くバインディングなので、 環境によっては prebuilt binary を利用するか、ネイティブビルド(= C/C++ のコンパイル) が必要になります。この挙動は実行環境 (OS, CPU, Node バージョン等) に左右されます。

これが `node:sqlite` として標準化されることで、ネイティブビルド責任は Node 側に移ります。ユーザやライブラリ開発者は、自分自身でビルド周りを意識する必要がなくなりハッピーというわけです。

このあたりのユーザ利便性および API 設計 (同期/非同期/トランザクション等) についての議論は [Issue#53264](https://github.com/nodejs/node/issues/53264) で議論されているので、興味のある方は読んでみてください。

## API はまだ同期が中心

https://nodejs.org/docs/latest/api/sqlite.html#class-sqltagstore

先述の [Issue#53264](https://github.com/nodejs/node/issues/53264) での議論の通り、 現時点(Node.js v25.7.0) では同期 API を中心に提供されています

基本的な Class は [`DatabaseSync`](https://nodejs.org/docs/latest/api/sqlite.html#class-databasesync) と [`StatementSync`](https://nodejs.org/docs/latest/api/sqlite.html#class-statementsync) の2つで、基本的な API は既に揃っているようです。

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

みたいな挙動になるようです。
ボイラープレート減るのは嬉しいですが、仕様知らないと寧ろ SQLi できそうに見えるのは自分だけだろうか...。

なお、Async API や Transaction API は 現在も活発に議論が進んでいるようです([#54307](https://github.com/nodejs/node/issues/54307), [#57431](https://github.com/nodejs/node/issues/57431))。こちらも興味ある方はチェケラ。

## ベンチマーク

https://github.com/takymt/node-builtin-sqlite-bench

自分の利用範囲だと同期APIな時点でほぼ誤差ですが、一応取ってみました。

200,000行に対しての 10 回クエリの平均値 (ms) を比較しています

```
Query            node:sqlite  node:sqlite(cached)  better-sqlite3
---------------  -----------  -------------------  --------------
WHERE id = ?     0.0072       0.0053               0.0021
LIMIT 10         0.0083       0.0074               0.0047
LIMIT 1000       0.5508       0.4210               0.3188
(all 200k rows)  51.92        47.54                32.66
```

環境依存値なのであくまで参考程度ですが、`better-sqlite3` のほうが最適化を謳っている分速いです (知ってた)
とはいえ組み込みライブラリの責務は性能最適化よりも依存としての安定だと思うので全然OK。

キャッシュ当たればちゃんと速そうですが、先述の「仕様知らないと寧ろ SQLi できそうに見える」問題があるので、性能に困らない限り自分はキャッシュ使わない気がします。

## まとめ

今回は Node.js v25.7.0 で RC になった `node:sqlite` について、その経緯と背景を調べてみました。

性能は `better-sqlite3` に劣りますが、ネイティブアドオンの複雑性から解放されて依存を作らなくてすむのが非常に魅力的なので、自分個人だと `node:sqlite` がファーストチョイスになりそうです！

それではみなさま、よき SQLite ライフを！

（余談: この手の PR を Deep Reading するの、めっちゃ勉強になる）
