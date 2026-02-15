---
title: 'Go Releaser が便利だったので紹介する'
emoji: '🚀'
type: 'tech'
topics: ['golang', 'githubactions', 'cicd']
published: true
---

先日、Claude Code の `✢ Moonwalking…` みたいな待機文言を集計するツール [cc-flavors](https://github.com/takymt/cc-flavors) をGoの処女作として作成したのですが、その際に使ったGo Releaserが非常に便利だったので紹介しようと思います

https://zenn.dev/hiruno_tarte/articles/cc-flavors-introduction

## Go Releaser とは

https://goreleaser.com/

現時点で自分はかなり表面的な使い方しかしていないのですが、

- 複数OS/アーキテクチャ向けのバイナリビルドとtar.gz作成
- checksums.txt / SBOM 生成
- GitHub Release 作成
- Homebrew / Docker / Scoop などのパッケージマネジャーへの公開
- GPGやcosignでの署名

を設定ファイル `.goreleaser.yaml` 一つで実施してくれる便利ツールです。

## 設定してみる

[実際に自分が使用している設定ファイル](https://github.com/takymt/cc-flavors/blob/main/.goreleaser.yaml) を題材に、Go Releaser の設定のラクさを紹介していきます。

![リリースアセット一覧](/images/go-releaser-introduction/release-assets.png)
_Go Releaser で生成されたアセット一覧_

### マルチOS/アーキテクチャビルド

`builds` セクションにビルド対象のOS/アーキを記載します

```yaml
builds:
  - main: .
    binary: cc-flavors
    ldflags:
      - '-s -w -X main.revision={{ .ShortCommit }}'
    goos:
      - linux
      - darwin
    goarch:
      - amd64
      - arm64
```

ちなみに `ldflags` はリンカに対するフラグです

- `-s`/`-w`: シンボルテーブルやDWARFなどのデバッグ情報を削除してバイナリサイズを削減(今回だと約30%程度圧縮)
- `main.revision={{ .ShortCommit }}`: バイナリへのrevision埋め込み

### 配布設定

`archives` セクションで、配布物の構成/命名/フォーマットを決められます

```yaml
# cc-flavors_1.0.0_darwin_arm64.tar.gz みたいな命名になる
archives:
  - formats:
      - tar.gz
    name_template: '{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}'
    files:
      - LICENSE
      - README.md
```

### チェックサム

`checksum` セクションで、各バイナリの改ざん/破損をチェックするハッシュ値ファイル生成の設定ができます

```yaml
checksum:
  name_template: 'checksums.txt'
```

小規模CLIだとサボってても怒られないと思いますが、設定に1-2行追加するだけなので入れ得です。

実際の改ざん/破損チェックは利用者側が行います

```console
$ grep cc-flavors_1.0.0_linux_amd64.tar.gz checksums.txt | sha256sum -c -
cc-flavors_1.0.0_linux_amd64.tar.gz: OK
```

ただしチェックサムごと改ざんされている場合に改ざん検知ができないリスクがあるので、後述の署名と併用する必要があります。

### 署名

`signs` セクションでチェックサムに署名ができます。

`checksums.txt.sig`として署名をしておくことで、チェックサム自体が改ざんされていないことを証明できます。

```yaml
signs:
  - artifacts: checksum
    cmd: cosign
    signature: '${artifact}.sig'
    output: true
    args:
      - sign-blob
      - --yes
      - --output-signature
      - ${signature}
      - ${artifact}
```

今回は定番のGPGではなく、cosignという手法を使用しています。

cosignについてはそのうち別の記事で解説したいですが、ざっくり言うとOIDCベースのKeylessな証明方法であり、
GPGのように鍵を管理する必要がないという大きなメリットがあります。

### SBOM

`sbom`セクションで 生成アーカイブごとにSBOMを作っています。

```yaml
sboms:
  - artifacts: archive
```

バイナリに含む依存関係を一覧化し透明化することで、監査やサプライチェーン攻撃に役立ちます。

こちらも小規模CLIにはオーバースペックなのですが、2行で設定できるので入れ得です。

### リリース

`release` セクションでGitHub Releaseの設定をしています。

`github`以外にも、`gitlab` / `homebrew` / `npm` / `scoop` / `docker` など色々なプラットフォームに対応しています([参考](https://goreleaser.com/customization/release))

```yaml
release:
  github:
    owner: takymt
    name: cc-flavors
```

## おわりに

Go Releaser でリリース設定をシンプルかつ宣言的に記述する例について紹介しました。いかがだったでしょうか？

今後も便利CDツールに感謝しつつ、最大限活用していきたいと思います。

それでは皆様、よきDevOpsライフを！
