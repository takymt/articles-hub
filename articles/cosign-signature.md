---
title: 'sigstore/cosign でGPG鍵の管理から解放される'
emoji: '🔑'
type: 'tech'
topics: ['cosign', 'signature', 'gpg', 'sigstore']
published: false
---

この記事では [sigstore/cosign](https://github.com/sigstore/cosign) を使用して鍵管理不要な署名を実現する方法と仕組みを解説します。

https://github.com/sigstore/cosign

## なんの話？

ソフトウェアのアーティファクトにおける「署名」の話

- その成果物が本当に開発者によるものか？
- 改ざんされていないか？
- いつ誰が署名したか？

これらを検証可能にするために、従来は [GPG](https://www.gnupg.org/) を用いる手法が主流でした。
近年は [Cosign](https://github.com/sigstore/cosign) という手法が台頭してきているようです。

## GPGの課題

GPG は OpenPGP 互換のオープンソース実装であり、 PGP の鍵モデルに依存します。[\_ref](https://www.gnupg.org/)
ここからは PGP 方式の既知の課題を挙げていきます。なんとなく知ってる方はスキップで OK。

### 長期鍵管理モデルの構造的な問題

PGP の鍵モデルには以下の特徴があり、「長く使うほど信頼が増える」構造になっています。

- 事実上永久のルート鍵を自身のアイデンティティとして保管する
- 分散信頼システムである `Web of Trust (信頼の輪)` に基づいており、他の誰かに署名されることでアイデンティティの信頼を積み上げる

しかしながら、当然クレデンシャルには常に漏洩の可能性があります。そして、長く使えば使うほど、その可能性は高まります。たとえ [Yubikey](https://www.yubico.com/yubikey/?lang=ja) のような PGP 鍵管理ツールを使用していたとしても、絶対に漏洩しない保証はありません。当時 PGP コミュニティに所属していた Fillipo Valsorda ですら、この構造的問題を内部から痛烈に批判しています。 [\_ref](https://words.filippo.io/giving-up-on-long-term-pgp/)

他にも Python エコシステムの例では、[PEP-761](https://peps.python.org/pep-0761/) で Python アーティファクトの署名に PGP 署名を使用しないようにする仕様が規定されており、リリース管理者が7年以上もの間秘密鍵を保有しつづけるのは不適当だとしています。

> Requiring release managers to maintain and protect PGP private keys for seven or more years is an unnecessary burden in the new age of ergonomic and ephemeral signing keys.

### 前方秘匿性の不足

今日の暗号モデルでは、大抵セッション鍵を用いた[前方秘匿性(Forward Secrecy)](https://ja.wikipedia.org/wiki/Forward_secrecy)が前提となっています。
前方秘匿性とは「仮に秘密鍵が侵害されても過去の内容は復元できない」性質のことで、 TLS などで用いられる[楕円 Diffie–Hellman 鍵交換](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie%E2%80%93Hellman)がその代表例です。

しかしながら PGP の仕様にこの性質は規定されていません。つまり一度長期鍵が漏洩してしまえば、過去の暗号化内容が全て復号できてしまうことになります。
もちろん PGP ツールでも、セッション鍵と長期鍵を使い分けることで前方秘匿性を模倣することはできますが、実際にそこまでやるユーザはほとんどいないことが指摘されています [\_rel](https://www.latacora.com/blog/2019/07/16/the-pgp-problem/#no-forward-secrecy)

### Web Of Trust (信頼の輪) が機能しない

PGP 仕様は `Web of Trust (信頼の輪)` に基づいた分散信頼システムですが、実際には

- 鍵を信頼すべきかどうかの正確な判断は専門家でも難しいため、結局多くのユーザが中央集権的な権威を信頼の基準にしている[\_rel](https://www.latacora.com/blog/2019/07/16/the-pgp-problem/#incoherent-identity)
- そもそも署名してくれる人が少ない( Fillipo Valsorda の場合は年2回程度[\_rel](https://words.filippo.io/giving-up-on-long-term-pgp/))

といった課題があり、見せかけの分散システムだと批判されることがあるようです。

### コンテナや CI との相性が悪い

コンテナや CI といった実行主体は短命であるため、PGP モデルの「永続化された人間のためのアイデンティティ」という思想と本質的に相容れません。

いくら GitHub のシークレット管理が堅牢といっても、長期秘密鍵をクラウド上で管理/ローテーションするコストやリスクを負うことに変わりはありません。

## Cosignが解決すること

### Keyless署名

### ログ公開(Rekor)による透明化

### コンテナ/CIフレンドリー

## Cosignの仕組み

### Keylessフロー

### Fulcio（証明書発行）

### Rekor（透明ログ）

### Cosign CLI

## Cosignの課題

### Sigstore依存

### 中央集権な信頼モデル

### 学習コスト

## 終わりに
