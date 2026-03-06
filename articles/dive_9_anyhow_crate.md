---
title: "足を止めて見る #9 〜 Rustのanyhowクレート(1) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2026-02-16 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの9つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_8_thiserror_crate)の記事では `thiserror` の `helper attributes` をそれぞれ確認し、カスタムエラー型を定義する方法を見てきました。

`thiserror` は、エラー型を定義する際にとても便利なクレートです。

一方で、アプリケーションコードでは「エラー型を厳密に定義するよりも、柔軟に扱いたい」という場面も多くあります。

そのような場合に便利なのが `anyhow` クレートです。

## anyhow クレートとは

`anyhow` は、アプリケーションコードでのエラーハンドリングを簡潔に書くためのクレートで、`Result<T, E>` の `E` を具体的な型ではなく `anyhow::Error` にまとめて扱うことができます。

## anyhow クレートをインストール

まずはインストールですね。

```console
$ cargo add anyhow
```

以下のようなメッセージが表示されながら追加されます。
`anyhow` には `std` と `backtrace` というフィーチャーがあり、デフォルトでは `std` が有効になるようですね。

```console
$ cargo add anyhow
    Updating crates.io index
      Adding anyhow v1.0.102 to dependencies
             Features:
             + std
             - backtrace
    Updating crates.io index
     Locking 1 package to latest Rust 1.89.0 compatible version
      Adding anyhow v1.0.102
```

`Cargo.toml` に以下の依存関係が追加されました。

```toml
[dependencies]
anyhow = "1.0.102"
```

準備はたったこれだけですね

## anyhow クレートを使ってみる

## もう一段だけ深ぼってみる

## 振り返り

今回は `anyhow` クレートを改めて足を止めて見てみました。

これで明日から、もっと堂々と `anyhow` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_9_anyhow_crate
