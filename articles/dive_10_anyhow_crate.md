---
title: "足を止めて見る #10 〜 ?演算子の正体 〜"
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

今回は `anyhow` クレートを利用する際に必ず利用する `?`演算子（question mark operator）を、足を止めて深掘りしようと思います。

## ?演算子の正体は？

```
?演算子は Result 専用の構文ではなく、
「失敗したら早期 return する」というパターンを抽象化した構文です。
```

## ?演算子の振る舞いを再確認してみる

改めて、前回扱った以下のコードを確認してみます。

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
match read_file() {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),
}
```

つまり ? 演算子は

- Ok の場合 → 中身を取り出す
- Err の場合 → return Err(...) する

という処理を簡潔に書くための構文であることを前回確認しました。

また `anyhow::error::Error` では `std::error::Error` トレイトを実装した型 `E` に対する `std::convert::From` が実装されています。

https://docs.rs/anyhow/latest/anyhow/struct.Error.html#impl-From%3CE%3E-for-Error

`std::io::Error` はそもそも `std::error::Error` を実装したものです。

これにより `?`演算子を使うだけで `std::result::Result<String, std::io::Error>` が `anyhow::Result<String>` に変換される仕組みがわかりました。


## もう一段だけ深ぼってみる (1)

では、この ? 演算子の実装はどこに書かれているのでしょうか。

調べていくと、Rustの RFC1859 に辿り着きます。

https://rust-lang.github.io/rfcs/1859-try-trait.html

この RFC では `?`演算子は `Try トレイトのシンタックスシュガー` として設計されていることが説明されています。

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

ここでは `TryDesugar` という名前で、`?`演算子が構文変換されることが示されています。

## もう一段だけ深ぼってみる (2)

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

つまり現在の ? 演算子と同じ挙動をしています。

## もう一段だけ深ぼってみる (3)

似た話で `try_opt!` マクロという存在があります。

https://docs.rs/try_opt/latest/try_opt/macro.try_opt.html

現在は Deprecated になっており、`?`演算子を利用せよとなっています。

じゃあ `try!` マクロと `try_opt!` マクロの違いは何なのでしょうか？

`try!`マクロは Result型 に対するもので `std` に含まれています。

対して `try_opt!` マクロは `Option` 型に対するもので `std` には含まれておらず、あくまでコミュニティによる `try_opt` クレートによる提供です。

## 振り返り

今回は `anyhow` クレートを改めて足を止めて見てみました。

また、その途中で ? 演算子の仕組みについても少し深掘りしてみました。

普段は何気なく使っている機能ですが、こうしてソースコードや RFC を辿ってみると、Rustの設計がよく考えられていることが分かります。

これで明日から、もっと堂々と `anyhow` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_9_anyhow_crate
