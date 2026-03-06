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

## もう一段だけ深ぼってみる(1)

すっかり使い慣れた `?`演算子（question mark operator）ですが、これってそもそも何なのでしょうか？

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

つまり `?`演算子は `std::result::Result<T,E>` を `T` に変換してくれるように見えます。

`?`演算子については、TRPLの `9.2. Resultで回復可能なエラー` でも紹介されています。

https://doc.rust-jp.rs/book-ja/ch09-02-recoverable-errors-with-result.html#%E3%82%A8%E3%83%A9%E3%83%BC%E5%A7%94%E8%AD%B2%E3%81%AE%E3%82%B7%E3%83%A7%E3%83%BC%E3%83%88%E3%82%AB%E3%83%83%E3%83%88-%E6%BC%94%E7%AE%97%E5%AD%90

> Resultの値がOkなら、Okの中身がこの式から返ってきて、プログラムは継続します。
> 値がErrなら、 returnキーワードを使ったかのように関数全体からErrが返ってくる

つまり `?`演算子は `std::result::Result<T,E>` がOkなら `T` を返し、Errなら `return Err` する、がもう一歩正しい理解のようです。

次に `?`演算子の実装はどこに書いてあるのか見つけにいきたいと思います。

ここで、探せども中々わからず、色々と調べた結果、Rust の RFC1859-try-trait に辿り着きました。

https://rust-lang.github.io/rfcs/1859-try-trait.html?utm_source=chatgpt.com

つまり、`?`演算子とは `Try`トレイトのシンタックスシュガーとして生まれたようです。

ここで、本家Rustコンパイラのソースコードを覗いてみてみようと思います。

色々探した結果、`?`演算子とは `Try`トレイトのシンタックスシュガーと記述されている箇所はこの辺りのようです。

https://doc.rust-lang.org/beta/nightly-rustc/src/rustc_hir/hir.rs.html#3018

```rust
/// Hints at the original code for a `match _ { .. }`.
pub enum MatchSource {
    /// A `match _ { .. }`.
    Normal,
    /// A `expr.match { .. }`.
    Postfix,
    /// A desugared `for _ in _ { .. }` loop.
    ForLoopDesugar,
    /// A desugared `?` operator.
    TryDesugar(HirId),
    /// A desugared `<expr>.await`.
    AwaitDesugar,
    /// A desugared `format_args!()`.
    FormatArgs,
}

impl MatchSource {
    #[inline]
    pub const fn name(self) -> &'static str {
        use MatchSource::*;
        match self {
            Normal => "match",
            Postfix => ".match",
            ForLoopDesugar => "for",
            TryDesugar(_) => "?",
            AwaitDesugar => ".await",
            FormatArgs => "format_args!()",
        }
    }
}
```

いやー難しいですね。

## もう一段だけ負荷ぼってみる(2)

ちなみに、`?`演算子が生まれる以前は、`try!` マクロがその立場にあったようです。

`try!` マクロの宣言はここにありました。

https://doc.rust-lang.org/src/core/macros/mod.rs.html#506

```rust
#[macro_export]
#[stable(feature = "rust1", since = "1.0.0")]
#[deprecated(since = "1.39.0", note = "use the `?` operator instead")]
#[doc(alias = "?")]
macro_rules! r#try {
    ($expr:expr $(,)?) => {
        match $expr {
            $crate::result::Result::Ok(val) => val,
            $crate::result::Result::Err(err) => {
                return $crate::result::Result::Err($crate::convert::From::from(err));
            }
        }
    };
}
```

たしかに、`std::result::Result::Ok(val)` なら `val` を返しているし、

`std::result::Result::Err(err)` なら `return err(From::from(err))` してるのが分かりますね。

## 振り返り

今回は `anyhow` クレートを改めて足を止めて見てみました。

これで明日から、もっと堂々と `anyhow` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_9_anyhow_crate
