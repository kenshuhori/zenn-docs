---
title: "足を止めて見る #8 〜 Rustのthiserrorクレート(2) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
published_at: 2026-02-16 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの8つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_7_thiserror_crate)の記事では thiserror の基本的な使い方を見ました。

enum に `#[derive(thiserror::Error)]` を付けただけで `std::fmt::Display` や `std::fmt::Debug` が自動実装されることを確認しました。

しかし実際のコードでは、ただ単にエラー型を定義するだけではなく「ちゃんと情報を渡したい」「原因を保持したい」 といった要件が出てきます。

その要求を満たすために `thiserror` にはいくつかの `helper attributes` が用意されています。

この記事では `thiserror` が提供する `helper attributes` を、1つずつ実際のコードで確認しながら、

- どの属性が何をするのか
- いつ使うべきか
- 互いの違い

を整理していきます。

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
use std::error::Error as _;

#[derive(Debug, thiserror::Error)]
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
        Ok(_) => println!("パースに成功しました"),
        Err(e) => {
            match e.source() {
                Some(source) => println!("パースに失敗しました。1つ下のエラー: {}", source),
                None => println!("パースに失敗しました。1つ下のエラー情報はありません"),
            }
        }
    }
    // パースに失敗しました。1つ下のエラー: invalid digit found in string
}
```

上記の場合、 `e.source()` によって `std::num::ParseIntError` が導き出され、 `std::num::ParseIntError` のエラー文言である `invalid digit found in string` が出力できるようになります。

---

#[error(transparent)]

この属性は、`source` と同様に、内部に持つ下位エラーに対する属性です。
下位エラーの Display 実装をそのまま使いたいときに便利で、内部エラーを隠さずそのまま表示したい、というケースに使います。


```rust
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error(transparent)]
    InvalidFormatForTransparent(#[from] std::num::ParseIntError),
}

fn parse_number_for_transparent(s: &str) -> Result<i32, MyError> {
    s.parse::<i32>()
        .map_err(MyError::InvalidFormatForTransparent)
}

fn main() {
    match parse_number_for_transparent("abc") {
        Ok(_) => println!("パースに成功しました"),
        Err(e) => {
            println!("パースに失敗しました。原因: {}", e);
        }
    }
    // パースに失敗しました。原因: invalid digit found in string
}
```

`source` と同様の `invalid digit found in string` というエラーメッセージが表示されました。

`transparent` を利用すれば、わざわざ.`source()`メソッドを利用して下位のエラーを辿ることなく `Display` できることが見て取れます。

---

#[backtrace]

backtrace は nightly (Rust 1.73+) 以降で利用できる attributes です。
下位のエラーが持つ追加情報を `provide` メソッドを通して利用できるようになりますが、
直接利用することはないと思うので、今回は触れずに終わろうと思います。


## もう一段だけ深ぼってみる

thiserror が提供する `helper attributes` を、「何をしてくれるのか」という観点で整理します。

| 属性 | 主な役割 | `?` 変換可能か | Display への影響 |
|------|----------|------------------|-----------------|
| `#[error("...")]` | エラー表示メッセージの定義 | ❌ | ✔（メッセージを定義） |
| `#[from]` | `From<T>` 実装を生成 | ✔ | ❌ |
| `#[source]` | 原因エラーの登録 | ❌ | ❌ |
| `#[error(transparent)]` | 原因エラーの Display を透過利用 | ❌（※） | ✔（下位エラーのメッセージをそのまま表示） |
| `#[backtrace]` | Backtrace を共有（nightly） | ❌ | ❌ |

※ `transparent` は `#[from]` を併用すると `?` が使える

## 振り返り

いかがでしょうか。

自分自身は、thiserrorについてかなり理解が深まりましたし、それぞれの `helper_attributes` に対する整理もできてきたと思います。

`#[error("...")]` は必須で利用するとして、原因となるエラーが存在する場合は `#[from]` と `#[error(transparent)]` を同時に利用する形が一般的なのかなと思っています。

この辺りはいずれ、ロギングをサポートするRustのフレームワーク [tracing](https://docs.rs/tracing/latest/tracing/) クレートにも話を繋げていけたらと思っています。

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_8_thiserror_crate

