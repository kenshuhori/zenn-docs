---
title: "足を止めて見る #8 〜 Rustのthiserrorクレート(2) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2026-02-03 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの8つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_8_thiserror_crate)の記事では thiserror の基本的な使い方を見ました。

enum に #[derive(thiserror::Error)] を付けただけで Display や Debug が自動実装されることを確認しました。

しかし実際のコードでは、ただ単にエラー型を定義するだけではなく「ちゃんと情報を渡したい」「原因を保持したい」 といった要件が出てきます。

その要求を満たすために thiserror にはいくつかの `helper attributes` が用意されています。ここではそれらをひとつずつ見ていきます。

## thiserror クレートの attributes をひとつずつ紐解いてみる

#[error("…")]

この属性は、エラーの人間向けメッセージ（Display 出力）を定義します。
前回の例でもすでに出てきましたが、文字列中に {} を書くことでフィールドを埋め込むことができます。

```rust
#[derive(Debug, thiserror::Error)]
pub enum ExampleError {
    #[error("指定した名前が空です")]
    EmptyName,
}
```

重要なのは、
Display の実装を直接書かなくても、文字列がそのまま Display 実装になるという点です。
std::error::Error では Display が必須なので、ここを補助してくれています。

---

#[from]

この属性は 指定したエラー型への `std::convert::From` を自動生成する属性です。

`std::convert::From` はこの、足を止めるシリーズの[2回目](https://zenn.dev/doctormate/articles/dive_8_thiserror_crate)でも紹介しています。
これにより ? 演算子でエラーが変換され、上位の戻り型に適合します。

```rust
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error("IO の問題が発生しました")]
    Io(#[from] std::io::Error),
}
```

これがあることで、こんなコードを書くだけで済みます

```rust
fn read_file() -> Result<(), MyError> {
    std::fs::read_to_string("foo.txt")?; // 自動で MyError に変換される
    Ok(())
}
```

---

#[source]

この属性は 元となったエラー（原因）を保持するための属性です。
標準の std::error::Error::source() が返す値になります。

```rust
#[derive(Debug, thiserror::Error)]
pub enum ParseError {
    #[error("パースに失敗しました")]
    InvalidFormat {
        #[source]
        source: std::num::ParseIntError,
    },
}
```

---

#[error(transparent)]

transparent は 元のエラーの Display をそのまま使いたいときに便利です。
内部エラーを隠さずそのまま表示したい、というケースに使います

```rust
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

このとき、MyError::Other(err) として出力すると元のエラーの文字列だけが出力されます。

## もう一段だけ深ぼってみる

<!-- TODO: 記載する -->

## 振り返り


## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_8_thiserror_crate

