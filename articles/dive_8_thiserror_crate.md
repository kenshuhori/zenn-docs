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

[前回](https://zenn.dev/doctormate/articles/dive_7_thiserror_crate)の記事では thiserror の基本的な使い方を見ました。

enum に `#[derive(thiserror::Error)]` を付けただけで `std::fmt::Display` や `std::fmt::Debug` が自動実装されることを確認しました。

しかし実際のコードでは、ただ単にエラー型を定義するだけではなく「ちゃんと情報を渡したい」「原因を保持したい」 といった要件が出てきます。

その要求を満たすために thiserror にはいくつかの `helper attributes` が用意されています。ここではそれらをひとつずつ見ていきます。

## thiserror クレートの attributes をひとつずつ紐解いてみる

#[error("…")]

この属性は、エラーの人間向けメッセージ `std::fmt::Display` を自動で実装する属性です。

[前回](https://zenn.dev/doctormate/articles/dive_7_thiserror_crate)の例でもすでに出てきましたが、文字列中に {0} を書くことでフィールドを埋め込むこともできます。

```rust
#[derive(Debug, thiserror::Error)]
pub enum ExampleError {
    #[error("指定した名前が空です")]
    EmptyName,
    #[error("指定した年齢は{0}ですが{min}から{max}の間でなければなりません。", min = u8::MIN, max = u8::MAX)]
    InvalidAge(i64),
}
```

`std::error::Error` では `std::fmt::Display` の実装が必須なので、ここを補助してくれています。

---

#[from]

この属性は、指定したエラー型への `std::convert::From` を自動で実装する属性です。

`std::convert::From` はこの、足を止めるシリーズの[2回目](https://zenn.dev/doctormate/articles/dive_8_thiserror_crate)でも紹介しています。

（まだ私の記事では触れてないですが） `std::convert::From` が実装されているため、 ? 演算子でエラーを変換することができます。

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("IO の問題が発生しました")]
    Io(#[from] std::io::Error),
}

fn read_file() -> Result<String, Error> {
    let content = std::fs::read_to_string("nonexist.txt")?;
    Ok(content)
}
```

? 演算子により `std::io::Error` を `Error::Io` へ変換することができるため、上記のような書き方が可能になります。

---

#[source]

この属性は、これまでとは異なり、何かを自動で実装する属性ではありません。
元となったエラー（原因）を保持するための属性で、`source` に指定された下位のエラー型の方でエラーが発生する可能性を示唆します。

```rust
use std::error::Error;

pub enum MyError {
    #[error("パースに失敗しました")]
    InvalidFormat {
        #[source]
        source: std::num::ParseIntError,
    },
}

fn parse_number(s: &str) -> Result<i32, MyError> {
    s.parse::<i32>()
        .map_err(|e| MyError::InvalidFormat { source: e })
}

fn main() {
    match parse_number("abc") {
        Ok(_) => {
            println!("パースに成功しました");
        }
        Err(e) => {
            match e.source() {
                Some(source) => println!("パースに失敗しました。根本のエラー: {}", source),
                None => println!("パースに失敗しました。根本のエラー情報はありません"),
            }
        }
    }
    // パースに失敗しました。根本のエラー: invalid digit found in string
}
```

上記の場合、 `e.source()` によって `std::num::ParseIntError` が導き出され、 `std::num::ParseIntError` のエラー文言である `invalid digit found in string` が出力できるようになります。

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

---

#[backtrace]

backtrace は nightly (Rust 1.73+) 以降で利用できるようになった attributes です。

```rust
#[derive(Debug, Error)]
pub enum MyError {
    #[error("failed")]
    Fail {
        #[backtrace]
        bt: std::backtrace::Backtrace
    }
}
```


## もう一段だけ深ぼってみる

<!-- TODO: 記載する -->
- helper_attributes をどこにつけるか迷う話
- nightlyとは？
- std::error::Error::source()とは
- Error::provide()とは

## 振り返り


## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_8_thiserror_crate

