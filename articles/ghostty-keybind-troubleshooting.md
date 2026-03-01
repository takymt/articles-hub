---
title: 'Ghostty のキーバインドが動かないときは Terminal Inspecter を見よう'
emoji: '👻'
type: 'tech'
topics: ['ghostty', 'cli', 'dotfiles', 'keybinding']
published: true
---

Ghostty のキーバインドを設定してた時にちょっと詰まったので小ネタをメモ

## キーバインドが動かない

Ghostty のタブ切り替えを macOS でよくある `cmd` + `shift` + `[` or `]` にしたかったのですが、

```conf
keybind = super+shift+left_bracket=previous_tab
keybind = super+shift+right_bracket=next_tab
```

あれ？

`previous_tab` は動くのに、`next_tab` は動かない...。

## 原因

Terminal Inspector (Ghostty の ブラウザ Dev Tools ライクな機能) で `]` を叩いてみると、

![ghostty-terminal-inspector](/images/ghostty-keybind-troubleshooting/ghostty-terminal-inspector.png)
_Terminal Inspector 機能で `]` キーを叩いてみる_

どうやら `backslash` として認識されているようです。

自分のキーボード配列は JIS 配列なので、おそらく US と JIS の差分が原因だと思われます。
[iTerm2](https://iterm2.com/) 使ってたときは問題なく動いてたので気づかなかった...。

というわけで正しい設定はこんな感じでした。

```conf:$XDG_CONFIG_HOME/.config/ghostty/config
# For JIS layout
keybind = super+shift+left_bracket=previous_tab
keybind = super+shift+backslash=next_tab
```

（他の ghostty 設定については [takymt/dotfiles](https://github.com/takymt/dotfiles/blob/main/dot_config/ghostty/config) にて公開しています）

## まとめ

というわけで、Ghostty (のキーバインド) が上手く動かないときは Terminal Inspector を見てみましょう〜！

にしても、こういうことがあるとUSキー使ってみたくなりますね。
