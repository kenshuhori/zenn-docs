---
title: "足を止めて見る #10 〜 Rustのanyhowクレート(2) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2026-02-16 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの10つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_9_anyhow_crate)の記事では `anyhow` の基本的な使用方法を見てきました。

今回は `anyhow` クレートを足を止めて見てみます。


## もう一段だけ深ぼってみる (1)

さて、ここで改めて ? 演算子について考えてみます。

すっかり使い慣れた `?`演算子（question mark operator）ですが、これってそもそも何なのでしょうか？

以下のコードを少し分解してみます。

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

つまり ? 演算子は `Try トレイトのシンタックスシュガー` として設計されています。

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
