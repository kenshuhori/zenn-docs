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

`thiserror` はライブラリ側でエラー型を定義する際にとても便利なクレートでした。

一方で、アプリケーションコードでは「エラー型を厳密に定義する」よりも「柔軟に扱いたい」という場面も多くあります。

そのような場合に便利なのが `anyhow` クレートです。

今回は `anyhow` クレートを改めて足を止めて見てみます。

## anyhow クレートとは

`anyhow` は、アプリケーションコードでのエラーハンドリングを簡潔に書くためのクレートです。

Rustでは通常、関数の戻り値として次のような型を返します。

```rust
Result<T, E>
```

ここで E はエラー型です。

ライブラリを書く場合は、thiserror を使って独自のエラー型を定義することがよくあります。しかしアプリケーションコードでは、エラー型を細かく定義するよりも「とりあえずエラーとして扱いたい」ということも多くあります。

anyhow を使うと、この E を anyhow::Error にまとめて扱うことができます。

## anyhow クレートをインストール

まずは anyhow をインストールします。

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

すると Cargo.toml に次の依存関係が追加されます。

```toml
[dependencies]
anyhow = "1.0.102"
```

anyhow には std と backtrace という feature があり、デフォルトでは std が有効になります。

## anyhow クレートを使ってみる

簡単な例を書いてみます。

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

ここで read_file の戻り値に注目してみます。

```rust
anyhow::Result<String>
```

これは実際には次の型のエイリアスです。

```rust
std::result::Result<String, anyhow::Error>
```

つまり `anyhow::Result<T>` は `Result<T, anyhow::Error>` を簡潔に書くための型エイリアスになっています。

## もう一段だけ深ぼってみる (1)

さて、ここで改めて ? 演算子について考えてみます。

すっかり使い慣れた `?`演算子（question mark operator）ですが、これってそもそも何なのでしょうか？

先ほどのコードを少し分解してみます。

```rust
fn main() {
    match read_file_ext() {
        Ok(_) => println!("成功"),
        Err(e) => println!("エラー: {e}"),
    }
}

fn read_file_ext() -> anyhow::Result<String> {
    let file = read_file()?;
    Ok(file)
}

fn read_file() -> std::result::Result<String, std::io::Error> {
    std::fs::read_to_string("not_found.txt")
}
```

ここでは `read_file()?` という形で ? 演算子を使っています。

これは概念的には次のような処理になります。

```rust
match read_file() {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),
}
```

つまり ? 演算子は

- Ok の場合 → 中身を取り出す
- Err の場合 → return Err(...) する

という処理を簡潔に書くための構文です。

この挙動は、TRPL（The Rust Programming Language）の
「9.2 Resultで回復可能なエラー」でも説明されています。

https://doc.rust-jp.rs/book-ja/ch09-02-recoverable-errors-with-result.html

もう一段だけ深ぼってみる (2)

では、この ? 演算子の実装はどこに書かれているのでしょうか。

調べていくと、Rustの RFC1859 に辿り着きます。

https://rust-lang.github.io/rfcs/1859-try-trait.html

この RFC では、? 演算子が Try トレイトを利用して実装されていることが説明されています。

つまり ? 演算子は

Try トレイトのシンタックスシュガー

として設計されています。

Rustコンパイラのコードを見てみると、? 演算子がデシュガリング（構文変換）されることを示す箇所があります。

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

ここでは TryDesugar という名前で、? 演算子が構文変換されることが示されています。

## もう一段だけ深ぼってみる (3)

ちなみに、`?`演算子が導入される以前は `try!`マクロが使われていました。

その定義は次のようになっています。

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

このコードを見ると

- Ok(val) の場合 → val を返す
- Err(err) の場合 → return Err(From::from(err))

という処理をしていることが分かります。

つまり現在の ? 演算子とほぼ同じ挙動をしています。

## 振り返り

今回は `anyhow` クレートを改めて足を止めて見てみました。

また、その途中で ? 演算子の仕組みについても少し深掘りしてみました。

普段は何気なく使っている機能ですが、こうしてソースコードや RFC を辿ってみると、Rustの設計がよく考えられていることが分かります。

これで明日から、もっと堂々と `anyhow` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_9_anyhow_crate
