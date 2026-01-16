---
title: "足を止めて見る #7 〜 Rustのthiserrorクレート(1) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2025-10-02 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの7つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_6_serde_crate)は serde クレートを深掘りしていました。

今回は、serde クレートを一旦卒業して、thiserror クレートに足を伸ばしてみます。

thiserror も所謂「デファクトスタンダード」と見做されているクレートなのではないでしょうか。

普段何気なく使っている色々な記述について、足を止めて見てみようと思います。

## thiserror クレートとは

とにかくまずは [docs.rs](https://docs.rs/) で `thiserror` クレートを見に行きましょう。

https://docs.rs/thiserror/latest/thiserror/

「Rust の標準 trait である std::error::Error に対する便利な derive マクロを提供する」との紹介が冒頭にあります。

## thiserror クレートをインストール

まずはインストールですね。

```console
$ cargo add thiserror
```

`Cargo.toml` に以下の依存関係が追加されました。

```toml
[dependencies]
thiserror = "2.0.17"
```

準備はたったこれだけですね

## thiserror クレートを使ってみる

今回は Person という構造体を用意して new 時の引数チェック用のエラーを表現してみます。

```rust
#[derive(thiserror::Error, Debug)]
pub enum Error {
    #[error("Nickname cannot be empty.")]
    EmptyNickname,
    #[error("You provide {0}, but it must be between {min} and {max}.", min = u8::MIN, max = u8::MAX)]
    InvalidAge(i64),
}

pub struct Person {
    pub nickname: String,
    pub age: u8,
}

impl Person {
    pub fn new(nickname: &str, age: i64) -> Result<Self, Error> {
        if nickname.len() == 0 {
            return Err(Error::EmptyNickname);
        }
        let age_u8 = u8::try_from(age).map_err(|_| Error::InvalidAge(age))?;

        Ok(Self {
            nickname: String::from(nickname),
            age: age_u8,
        })
    }
}
```

たとえば nickname が空文字だったら EmptyNickname エラーとなり、

age が u8 の範囲外だったら InvalidAge エラーとなるように書いてみました。

敢えてエラーとなるように書いてみて、実行してみましょう。

```rust
fn main() {
    let person_with_invalid_nickname = Person::new(
        "",
        30,
    );

    match person_with_invalid_nickname {
        Ok(_) => {
            println!("Person created successfully.");
        }
        Err(e) => {
            println!("Error: {}", e);
        }
    }

    let person_with_invalid_age = Person::new(
        "Alice",
        -50,
    );

    match person_with_invalid_age {
        Err(e) => {
            println!("Error: {}", e);
        }
        Ok(_) => {
            println!("Person created successfully.");
        }
    }
}

// 出力 = Error: Nickname cannot be empty.
// 出力 = Error: You provide -50, but it must be between 0 and 255.
```

意図通りに出力されました。



## もう一段だけ深ぼってみる

TODO

## 振り返り

今回は `thiserror` クレートを改めて足を止めて見てみました。

これで明日から、もっと堂々と `thiserror` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_7_thiserror_crate

