---
title: "足を止めて見る #10 〜 ?演算子の正体 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
published_at: 2026-03-18 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの10つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_9_anyhow_crate)の記事では `anyhow` の基本的な使用方法を見てきました。

今回は `anyhow` クレートを利用する際に必ず利用するであろう ?演算子（question mark operator）を、足を止めて深掘りしようと思います。

## ?演算子の正体は？

まず結論から、 ?演算子 とは以下です。

```
「失敗したら早期 return する」というパターンを抽象化した構文
```

これがどういう意味なのか、順を追って見ていきます。

## ?演算子の振る舞いを再確認してみる

改めて、[前回](https://zenn.dev/doctormate/articles/dive_9_anyhow_crate)扱った以下のコードを確認してみます。

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

ここで `read_file()?` という形で ? 演算子を使っており、概念的には次のような処理になります。

```rust
fn read_file_ext() -> anyhow::Result<String> {
    let file = match read_file() {
        Ok(v) => v,
        Err(e) => return Err(From::from(e)),
    }
    Ok(file)
}
```

つまり ? 演算子は

- Ok の場合 → 中身を取り出す
- Err の場合 → return Err(...) する

という処理を簡潔に書くための構文であることを前回確認しました。

また `anyhow::error::Error` では `std::error::Error` トレイトを実装した型 `E` に対する `std::convert::From` が実装されています。

https://docs.rs/anyhow/latest/anyhow/struct.Error.html#impl-From%3CE%3E-for-Error

`std::io::Error` はそもそも `std::error::Error` を実装したものです。

そのため `std::result::Result<String, std::io::Error>` が `anyhow::Result<String>` へ ? 演算子を使うだけで変換されるというカラクリであることが分かりました。


## もう一段だけ深ぼってみる (1)

では、この ?演算子の実装はどこに書かれているのでしょうか。

調べていくと、Rustの RFC0243 に辿り着きます。

https://rust-lang.github.io/rfcs/0243-trait-based-exception-handling.html

RFC243では、?演算子は `try!` マクロの糖衣構文として導入されました。

Rustコンパイラのコードを見てみると、?演算子がデシュガリング（構文変換）されることを示す箇所があります。

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

ここでは `TryDesugar` という名前で、?演算子が構文変換されることが示されています。

## もう一段だけ深ぼってみる (2)

ちなみに、?演算子が導入される以前は `try!`マクロが使われていたようです。

その定義は次のようになっています。

https://doc.rust-lang.org/src/core/macros/mod.rs.html#506

```rust
#[macro_export]
#[stable(feature = "rust1", since = "1.0.0")]
#[deprecated(since = "1.39.0", note = "use the ? operator instead")]
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

つまり現在の ?演算子と同じ挙動をしています。

ためしに、?演算子の代わりに `try!` マクロを使ってみます。

```rust
pub(crate) fn main() {
    match read_file_ext() {
        Ok(_) => println!("成功"),
        Err(e) => println!("エラー: {e}"),
    }
}

fn read_file_ext() -> anyhow::Result<String> {
    #[warn(deprecated)]
    let text = r#try!(read_file());
    Ok(text)
}

fn read_file() -> std::result::Result<String, std::io::Error> {
    std::fs::read_to_string("not_found.txt")
}
```

「try!マクロはdeprecatedですよ」というwarningは出ますが、?演算子を置き換えて機能しました。

## もう一段だけ深ぼってみる (3)

似た話で `try_opt!` マクロという存在があります。

https://docs.rs/try_opt/latest/try_opt/macro.try_opt.html

こちらも現在は deprecated になっており、?演算子を利用せよとなっています。

じゃあ `try!` マクロと `try_opt!` マクロの違いは何なのでしょうか？

`try!`マクロは `Result` 型 に対するもので `std` に含まれています。

対して `try_opt!` マクロは `Option` 型に対するもので `std` には含まれておらず、あくまでコミュニティによる `try_opt` クレートによる提供です。

`try_opt!` マクロのソースコードを見てみましょう。

https://docs.rs/try_opt/latest/src/try_opt/lib.rs.html#31-38

```rust
#[deprecated(since = "0.2.0", note = "Use the question mark (?) operator in newer versions of Rust.")]
macro_rules! try_opt {
    ($e:expr) =>(
        match $e {
            Some(v) => v,
            None => return None,
        }
    )
}
```

このコードを見ると

- Some(val) の場合 → val を返す
- None の場合 → return None

という処理をしていることが分かり、現在の ?演算子と同じ挙動をしています。

つまり `try_opt!` マクロは `try!` マクロを参考に、

「`None`なら早期にreturnしたい」という願いを叶えるために考案されたものなんだなあ、ということが分かってきます。

## もう一段だけ深ぼってみる (4)

そしてさらに Rustの RFC1859 にて、

「?演算子をResult専用から“任意の型に拡張する仕組み”を導入しませんか？」

と展開され、?演算子をOptionでも使えるようになっていったわけです。

https://rust-lang.github.io/rfcs/1859-try-trait.html

そのため、現在 `?` 演算子は Option型でも Result型でも、同じように利用できているわけですねー。

## 振り返り

今回は ?演算子に足を止めて見てみました。

普段は何気なく使っている?演算子ですが、こうしてソースコードや RFC を辿ってみると、Rustの設計や歴史的背景を感じられます。

冒頭の結論に戻りますが、?演算子とは

```
「失敗したら早期 return する」というパターンを抽象化した構文
```

です。今ならスッと意味が分かりますね、これで明日から、もっと堂々と `anyhow` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_10_anyhow_crate
