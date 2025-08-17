---
title: "足を止めて見る #2 〜 RustのFromトレイト 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの2つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_1_display_trait)は Display トレイトについてでした。

## std::convert::From トレイトを実装しよう

ある型に対して、別の型から変換してその型を作る方法を定義できるのが `std::convert::From` トレイトですよね。

今回は足を止めて `std::convert::From` トレイトを実装してみます。


## std::convert::From トレイトを実装する構造体を用意してみる

下準備として、Person という構造体を用意してみます。
構造体には、nickname と age フィールドを持っています。
ageフィールドは、プリミティブ型ではなくAge型を取るようにしてみます。


```rust
#[derive(Debug)]
struct Person {
    nickname: String,
    age: Age,
}

#[derive(Debug)]
struct Age(u8);

fn main() {
    let age_value = 35_u8;
    let yoshida = Person {
        nickname: String::from("yosshi-"),
        age: Age::from(age_value),
    };
    println!("{:?}", yoshida);
}
```

Ageはいわゆる[ユニット様構造体](https://doc.rust-jp.rs/book-ja/ch05-01-defining-structs.html#%E3%83%95%E3%82%A3%E3%83%BC%E3%83%AB%E3%83%89%E3%81%AE%E3%81%AA%E3%81%84%E3%83%A6%E3%83%8B%E3%83%83%E3%83%88%E6%A7%98%E3%82%88%E3%81%86%E6%A7%8B%E9%80%A0%E4%BD%93)という型です。

無理やり実行してみると `mismatched types` エラーになってしまいます（当然ですね）。

age_value が `expected Age, found u8` とエラーメッセージが出ているので、 `Age型` は `Age型` からの変換しか知らないよ、ということでしょう。

では `std::convert::From` トレイトを利用して、u8から変換できるようにしてみます。

## std::convert::From トレイトを実装してみる

`Age` 構造体に `std::convert::From` トレイトを実装してみます。

```rust
#[derive(Debug)]
struct Person {
    nickname: String,
    age: Age,
}

#[derive(Debug)]
struct Age(u8);

impl From<u8> for Age {
    fn from(value: u8) -> Self {
        Age(value)
    }
}

fn main() {
    let age_value = 35_u8;
    let yoshida = Person {
        nickname: String::from("yosshi-"),
        age: Age::from(age_value),
    };
    println!("{:?}", yoshida);
}
```

`impl From<u8> for Age` を書いてみました。u8であればそのまま、ユニット様構造体のAge型に当てはめられるのですんなりです。

実行してみると、期待通りエラーにならずデバッグ出力されます。
`Person { nickname: "yosshi-", age: Age(35) }`

## もう一段だけ深ぼってみる #1

なんで `impl std::convert::From<u8> for Age` と書かなくて済んでいるんでしょうか？

これは自分知っていました。答えは、Rustのprelude（プレリュード）というシステムのおかげですよね。
全てのRustプログラムで自動的にインポートされるモジュールがいくつか存在していて `std::convert::From` もその一つということです。

以下のページでプレリュードに含まれるモジュールを確認できます。`std::convert::From` は最初期からプレリュードに含まれているようですね。

https://doc.rust-lang.org/std/prelude/index.html

Rustの成長とともに、プレリュードに含まれるモジュールも増えていきそうですね。

## もう一段だけ深ぼってみる #2

`std::convert::From` トレイトを実装すると `into` メソッドが利用できると聞いたことがあります。ちょっと試してみましょう。

```rust
#[derive(Debug)]
struct Person {
    nickname: String,
    age: Age,
}

#[derive(Debug)]
struct Age(u8);

impl From<u8> for Age {
    fn from(value: u8) -> Self {
        Age(value)
    }
}

fn main() {
    let age_value = 35_u8;
    let yoshida = Person {
        nickname: String::from("yosshi-"),
        age: age_value.into(),
    };
    println!("{:?}", yoshida);
}
```

1箇所だけ変更し `age: age_value.into()` に書き換えてみましたが、問題なく実行できました。噂は本当のようです。

逆に `std::convert::Into` トレイトを実装してみたらどうなるのでしょうか？

```rust
#[derive(Debug)]
struct Person {
    nickname: String,
    age: Age,
}

#[derive(Debug)]
struct Age(u8);

impl Into<Age> for u8 {
    fn into(self) -> Age {
        Age(self)
    }
}

fn main() {
    let age_value = 35_u8;
    let yoshida = Person {
        nickname: String::from("yosshi-"),
        age: age_value.into(),
    };
    println!("{:?}", yoshida);
}
```

書いてみると `Into` と `From` は逆方向の関係性であることがわかりますね。
そしてこれも問題なく動作しました。

ちょっと公式のソースコードも覗いてみます。

https://doc.rust-lang.org/src/core/convert/mod.rs.html#757

```rust
// From implies Into
#[stable(feature = "rust1", since = "1.0.0")]
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    /// Calls `U::from(self)`.
    ///
    /// That is, this conversion is whatever the implementation of
    /// <code>[From]&lt;T&gt; for U</code> chooses to do.
    #[inline]
    #[track_caller]
    fn into(self) -> U {
        U::from(self)
    }
}
```

今回のケースに置き換えると、Tはu8、UはAge型になりそうです。
つまり `From<u8>` を実装しているAge型には、自動的に `Into<Age> for u8` が実装されることが見てとれます。

そのため `into` メソッドが自動的に使えるようになっているようですね。

今回も、足を止めて見た甲斐がありました。


## 振り返り

今回も、Rustの `convert/mod.rs` まで踏み込んで見てみることができました。やっぱり読めるもんですね、自信になります。

これで明日から、もっと堂々と `from` や `into` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_from_trait

