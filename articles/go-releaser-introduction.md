---
title: 'Go Releaser がリリース配布物を宣言的に定義できて便利だったので紹介する'
emoji: '🚀'
type: 'tech'
topics: ['golang', 'githubactions', 'cicd']
published: true
---

先日、Goの処女作として Claude Code の `✢ Moonwalking…` みたいなアレを集計する [cc-flavors](https://github.com/takymt/cc-flavors) というツールを作成したのですが、その際に使ったGo Releaser が非常に便利だったので紹介しようと思います

https://zenn.dev/hiruno_tarte/articles/cc-flavors-introduction

## Go Releaser とは

https://goreleaser.com/

- 複数OS/アーキテクチャ向けのバイナリビルドとtar.gz作成
- checksums.txt / SBOM 生成
- GitHub Release 作成
- Homebrew / Docker / Scoop などのパッケージマネジャーへの公開
- GPGやcosignでの署名

を設定ファイル `.goreleaser.yaml` 一つで実施してくれる便利ツールです。

この記事では、[実際に自分が使用している設定](https://github.com/takymt/cc-flavors/blob/main/.goreleaser.yaml) を題材に、Go Releaser のラクさを紹介します。

## 設定してみる

![リリースアセット一覧](/images/go-releaser-introduction/release-assets.png)
_実際に Go Releaser で生成されたアセット一覧_

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

これにより OS/アーキテクチャ ごとのビルド済みバイナリが生成されます

ちなみに `ldflags` はリンカに対するフラグです

- `-s`/`-w`: シンボルテーブルやDWARFなどのデバッグ情報を削除してバイナリサイズを削減 (今回は約30%程度)
- `main.revision={{ .ShortCommit }}`: バイナリへのrevision埋め込み

### 配布設定

`archives` セクションで、配布物の構成/命名/フォーマットを決められます

```yaml
# 出力例: cc-flavors_1.0.0_darwin_arm64.tar.gz
archives:
  - formats:
      - tar.gz
    name_template: '{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}'
    files:
      - LICENSE
      - README.md
```

### チェックサム

`checksum` セクションで、各バイナリの改ざん/破損をチェックするハッシュ生成ができます

```yaml
checksum:
  name_template: 'checksums.txt'
```

このあたりは小規模CLIだとサボってても怒られないと思いますが、設定に1-2行追加するだけなので入れ得です。

実際の改ざん/破損チェックは利用者側が行います

```console
$ grep cc-flavors_1.0.0_linux_amd64.tar.gz checksums.txt | sha256sum -c -
cc-flavors_1.0.0_linux_amd64.tar.gz: OK
```

ただし、配布物とチェックサムが両方改ざんされている場合に検知できないリスクがあるので、後述の署名と併用します。

### 署名

`signs` セクションで各種署名ができます。

`checksums.txt.sig`としてチェックサムファイルに署名をしておくことで、チェックサム自体が改ざんされていないことを証明できます。

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

今回は定番のGPGではなく、[cosign](https://github.com/sigstore/cosign) という手法を使用しています。
cosignについてはそのうち別の記事で解説したいですが、ざっくり言うとOIDCベースのKeylessな証明方法であり、GPGのように鍵を管理する必要がないのが大きなメリットです。

### SBOM

`sbom`セクションで 生成アーカイブごとにSBOMを作っています。

```yaml
sboms:
  - artifacts: archive
```

依存関係を透明化することで、監査やサプライチェーン攻撃に役立ちます。
こちらも小規模CLIにはオーバースペックなのですが、2行で設定できるので入れています。

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

Go Releaser でリリース設定をシンプルかつ宣言的に記述する例について紹介しましたが、いかがだったでしょうか？

今後も便利CDツールに感謝しつつ、最大限活用していきたいと思います。
それでは皆様、よきDevOpsライフを！🚀
