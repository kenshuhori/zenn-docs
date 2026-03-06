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

`thiserror` は、エラー型を定義する際にとても便利なクレートでした。

一方で、アプリケーションコードでは「エラー型を厳密に定義するもあるけど、柔軟に扱いたい」という場面も多くあります。

そのような場合に便利なのが `anyhow` クレートです。

## anyhow クレートとは

`anyhow` は、アプリケーションコードでのエラーハンドリングを簡潔に書くためのクレートで、`Result<T, E>` の `E` を具体的な型ではなく `anyhow::Error` にまとめて扱うことができます。

## anyhow クレートをインストール

まずはインストールですね。

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

上記のようなメッセージが表示されながら追加されます。
`anyhow` には `std` と `backtrace` というフィーチャーがあり、デフォルトでは `std` が有効になるようです。

`Cargo.toml` に以下の依存関係が追加されました。

```toml
[dependencies]
anyhow = "1.0.102"
```

## anyhow クレートを使ってみる

ごく普通に使ってみます。

```rust
fn main() {
    match read_file() {
        Ok(_) => println!("成功"),
        Err(e) => println!("エラー: {e}"),
    }
}

fn read_file() -> anyhow::Result<String> {
    let file = std::fs::read_to_string("not_found.txt")?;
    Ok(file)
}
```

## もう一段だけ深ぼってみる

すっかり使い慣れた `?`演算子（question mark operator）ですが、これってそもそも何なのでしょうか？

TRPLの `9.2. Resultで回復可能なエラー` でも紹介されています。

https://doc.rust-jp.rs/book-ja/ch09-02-recoverable-errors-with-result.html#%E3%82%A8%E3%83%A9%E3%83%BC%E5%A7%94%E8%AD%B2%E3%81%AE%E3%82%B7%E3%83%A7%E3%83%BC%E3%83%88%E3%82%AB%E3%83%83%E3%83%88-%E6%BC%94%E7%AE%97%E5%AD%90

先ほどのコード例を、もう少し分解してみます。

```rust
fn main() {
    match read_file_ext() {
        Ok(_) => println!("成功"),
        Err(e) => println!("エラー: {e}"),
    }
}

// ? 演算子は std::result::Result<T,E> を T に変換しているっぽい
fn read_file_ext() -> anyhow::Result<String> {
    let file = read_file()?;
    Ok(file)
}

fn read_file() -> std::result::Result<String, std::io::Error> {
    std::fs::read_to_string("not_found.txt")
}
```

つまり `?` 演算子は `std::result::Result<T,E>` を `T` に変換してくれるように見えます。

それであれば `std::result::Result` あたりのソースコードを覗いてみようと思います。

https://doc.rust-lang.org/src/core/result.rs.html#226

`core/result.rs` のソースコードを覗いてみたところ、以下のような記述を見つけました。

```rust
//! [`?`]: crate::ops::Try
```

`?`演算子は `crate::ops::Try` 由来なのかなと思います。次はそっちを覗いてみます。

https://doc.rust-lang.org/src/core/ops/try_trait.rs.html#133

```rust
#[doc(alias = "?")]
#[lang = "Try"]
#[rustc_const_unstable(feature = "const_try", issue = "74935")]
pub const trait Try: [const] FromResidual {

}
```

## 振り返り

今回は `anyhow` クレートを改めて足を止めて見てみました。

これで明日から、もっと堂々と `anyhow` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_9_anyhow_crate
