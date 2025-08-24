---
title: "足を止めて見る #3 〜 RustのTryFromトレイト 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの3つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_2_from_trait)は From トレイトについてでした。

## std::convert::TryFrom トレイトを実装しよう

ある型に対して、別の型から変換してその型を作る方法を定義できるのが `std::convert::From` トレイトでしたね（復習）

変換に失敗するケースが考えられる場合に利用するのが `std::convert::TryFrom` トレイトです。

今回は足を止めて `std::convert::TryFrom` トレイトを実装してみます。

## std::convert::TryFrom トレイトを実装する構造体を用意してみる

下準備として、Person という構造体を用意してみます。
構造体には、nickname と age フィールドを持っています。
ageフィールドは、プリミティブ型ではなくAge型を取るようにしてみます。


```rust
#[derive(Debug)]
pub struct Person {
    nickname: String,
    age: Age,
}

#[derive(Debug)]
struct Age(u8);

fn main() {
    let age_value = 35_u8;
    let nickname_value = "yosshi-";
    let yoshida = Person {
        nickname: String::from(nickname_value),
        age: Age::try_from(age_value), // try_from を呼び出してみます
    };
    assert_eq!(yoshida.nickname, nickname_value);
    assert_eq!(yoshida.age.0, age_value);

    println!("実行は正常に終了しました");
}
```

`try_from` メソッドを使ってみたいので `Age::try_from(age_value)` と書いてみました。

無理やり実行してみると、いくつかのエラーになってしまいます（当然ですね）。

```shell
the trait bound 'Age: TryFrom<u8>' is not satisfied
required for 'u8' to implement 'Into<Age>'
required for 'Age' to implement 'TryFrom<u8>'

expected struct 'Age'
  found enum 'Result<Age, Infallibley'
```

1つ目は `'u8' to implement 'Into<Age>'` や `'Age' to implement 'TryFrom<u8>'` から、実装が足りていないですと言われていますね。まだ実装していないので当然ですね。

2つ目は、age には `Age型` が来て欲しいのだけど、`Result<Age, Infallibley>` が受け取れてしまいました、と言われていますね。
それもそのはずで、`try_from` は `Result` を返すことで、型変換が失敗する可能性を表しているからですね。

## std::convert::TryFrom トレイトを実装してみる

[rust-analyzer](https://github.com/rust-lang/rust-analyzer)の力を借りながら、`std::convert::TryFrom` トレイトを実装し、`try_from` メソッドを書いていきます。

```rust
#[derive(Debug)]
pub struct Person {
    nickname: String,
    age: Age,
}

#[derive(Debug)]
struct Age(u8);

// POINT: あり得ない（0〜130に収まらない）年齢だった場合のエラーを定義
#[derive(Debug)]
enum AgeError {
    Impossible,
}

// POINT: TryFromの実装を追加する
impl TryFrom<u8> for Age {
    type Error = AgeError;

    // POINT: u8なので0以上は保証されている
    // POINT: 世界最高齢は122歳なので、130超はあり得ないことにしている
    fn try_from(value: u8) -> Result<Self, Self::Error> {
        if value > 130 {
            return Err(AgeError::Impossible);
        }
        Ok(Age(value))
    }
}

fn main() {
    let age_value = 135_u8;
    let nickname_value = "yosshi-";
    let yoshida = Person {
        nickname: String::from(nickname_value),
        // POINT: Result<Age, Infallibley> から無理やり Age を取り出してみる
        age: Age::try_from(age_value).expect("年齢の変換は成功するはず"),
    };
    assert_eq!(yoshida.nickname, nickname_value);
    assert_eq!(yoshida.age.0, age_value);

    println!("実行は正常に終了しました");
}
```

なんとか書いてみました。年齢ということで `0歳未満` または `130歳超` はあり得ないということにしました。

`impl TryFrom<u8> for Age {` の次の行に `type Error = (省略)` という見慣れないものが登場しています。

これは[関連型](https://doc.rust-jp.rs/rust-by-example-ja/generics/assoc_items/types.html)というやつですが、今回は無視してしまいます。

`Age::try_from(age_value)` は `Result` を返すため、無理やり中身の `Age型` を取り出すために `.expect("年齢の変換は成功するはず")` としてみました。

こうして実行してみると、なんとか成功しました。

## もう一段だけ深ぼってみる #1

(TODO)

## 振り返り

(TODO)

これで明日から、もっと堂々と `try_from` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_3_try_from_trait

