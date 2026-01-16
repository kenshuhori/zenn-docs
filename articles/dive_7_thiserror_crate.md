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
#[derive(thiserror::Error)]
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

意図通りに出力されたかと思います。

見て分かる通り、Error enum に thiserror::Error の derive マクロを記述しています。

また thiserror::Error の derive マクロの helper_attributes の1つ `error` によって、std::fmt::Display が実装されます。

```rust
#[derive(thiserror::Error)]
pub enum Error {
    #[error("You provide {0}, but it must be between {min} and {max}.", min = u8::MIN, max = u8::MAX)]
    InvalidAge(i64),
}
```

`{0}` と記述すると、ユニット様構造体である `InvalidAge(i64)` の `self.0` つまり i64 部分が表現されます。
他にも `{min}` や `{max}` のように、任意のフォーマット引数を利用することができます。

他の helper_attributes については、次回に詳しく見てみようと思います。

## もう一段だけ深ぼってみる

ここまで `thiserror::Error` を利用して見てみましたが、仮に `std::error::Error` を使っていたらどうなっていたのか？比較してみたいと思います。

```rust
pub enum Error {
    EmptyNickname,
    InvalidAge(i64),
}

impl std::error::Error for Error {}

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

Error enum に記述していた thiserror::Error の derive マクロを外してみました。

ここでひとつ、改めて気づいたんですが `Result<T,E>` の `E` って、別に `std::error::Error` を実装しているなどのクレート境界等は特に無いんですね。

とはいえ、実際には `std::error::Error` が実装された構造体が指定されていた方が良いと思うので `std::error::Error` を実装してみました。すると...

```sh
`person_with_std_error::Error` doesn't implement `std::fmt::Display`
the trait `std::fmt::Display` is not implemented for `person_with_std_error::Error`

`person_with_std_error::Error` doesn't implement `Debug`
add `#[derive(Debug)]` to `person_with_std_error::Error` or manually `impl Debug for person_with_std_error::Error`
```

`std::fmt::Display` の実装と `Debug` の実装が足りていないと怒られました。[docs.rs](https://doc.rust-lang.org/std/error/trait.Error.html) を見に行きましょう。

https://doc.rust-lang.org/std/error/trait.Error.html

たしかに `Debug + Display` のトレイト境界が明記されていますね。また `source` などいくつかのメソッドが提供されるようです。

```rust
pub trait Error: Debug + Display {
    // Provided methods
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
    fn description(&self) -> &str { ... }
    fn cause(&self) -> Option<&dyn Error> { ... }
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... }
}
```

ちなみに、 `thiserror` の helper_attributes にも `source` というものがあり、きっと関連があるはずです。次回見てみます。

言われた通り `Debug` と `Display` を実装してみました。

```rust
#[derive(Debug)]
pub enum Error {
    EmptyNickname,
    InvalidAge(i64),
}

impl std::fmt::Display for Error {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Error::EmptyNickname => write!(f, "Nickname cannot be empty."),
            Error::InvalidAge(age) => write!(f, "You provide {0}, but it must be between {min} and {max}.", age, min = u8::MIN, max = u8::MAX),
        }
    }
}

impl std::error::Error for Error {}

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

実行してみます。

```rust
fn main() {
    let person_with_invalid_nickname = person_with_std_error::Person::new(
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

    let person_with_invalid_age = person_with_std_error::Person::new(
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

やりました！完全に同じ出力になりました！

ここまでの内容だけだと `std::fmt::Display` を自前で実装するかどうか程度の違いしかないですね。

とはいえ足を止めて見た甲斐がありました。

## 振り返り

今回は `thiserror` クレートを改めて足を止めて見てみました。

これで明日から、もっと堂々と `thiserror` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_7_thiserror_crate

