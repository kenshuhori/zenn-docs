---
title: "足を止めて見る #12 〜 Rustのtracingクレート 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2026-04-13 17:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの12つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_11_anyhow_crate)の記事では `anyhow` クレートが提供しているメソッドやマクロを確認しました。

今回は anyhow クレートを離れて tracing クレートに移っていこうと思います。

もし Rust でアプリケーション書いていると `#[tracing::instrument]` といった記述を関数の宣言部分に付けていたりしませんか？

これがあると、その関数名がログに書き込まれているのは、きっと観測していると思います。

では `#[tracing::instrument]` と記述する以上のこと出来ていますか？

自分は勿論できていません。なので足を止めて見ようと思います。

## tracing クレートとは

とにかくまずは [docs.rs](https://docs.rs/) で `tracing` クレートを見に行きましょう。

https://docs.rs/tracing/latest/tracing/



## tracing クレートをインストール

まずはインストールですね。

```console
$ cargo add tracing tracing-subscriber
```

`Cargo.toml` に以下の依存関係が追加されました。

```toml
[dependencies]
tracing = "0.1.44"
tracing-subscriber = "0.3.23"
```

準備はたったこれだけですね

## tracing クレートを使ってみる


## 振り返り

これで明日から、もっと堂々と `tracing` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_12_tracing_crate
