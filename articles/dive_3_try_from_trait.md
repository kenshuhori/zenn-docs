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

変換に失敗するケースが存在する場合に利用するのが `std::convert::TryFrom` トレイトです。

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
  found enum 'Result<Age, Infallible>'
```

1つ目は `'u8' to implement 'Into<Age>'` や `'Age' to implement 'TryFrom<u8>'` から、実装が足りていないですと言われていますね。まだ実装していないので当然ですね。

2つ目は、age には `Age型` が来て欲しいのだけど、`Result<Age, Infallible>` が受け取れてしまいました、と言われています。それもそのはずで、`try_from` は `Result` を返すことで、型変換が失敗する可能性を表しているからですね。

ただ、まだ `impl TryFrom<u8> for Age {` を書いていないのですが、どうして `try_from` メソッドを呼び出すところまではいけているんでしょう？

ちょっと自信ないですが、これも `prelude` によるものと考えています（間違ってたら教えてください）。
単純に「`use std::convert::*` を自動でやってくれているもの」という認識だと、僕のようなつまずき方をする気がします。

https://doc.rust-lang.org/std/prelude/index.html


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

// POINT1: あり得ない（0〜130に収まらない）年齢だった場合のエラーを定義
#[derive(Debug)]
enum AgeError {
    Impossible,
}

// POINT2: TryFromの実装を追加する
impl TryFrom<u8> for Age {
    type Error = AgeError;

    // POINT3: u8なので0以上は保証されている
    // POINT4: 世界最高齢は122歳なので、130超はあり得ないことにしている
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
        // POINT5: Result<Age, Infallible> から無理やり Age を取り出してみる
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

`Age::try_from(age_value)` は `Result` を返します。無理やり中身の `Age型` を取り出すために `.expect("年齢の変換は成功するはず")` としてみました。

こうして実行してみると、なんとか成功しました。

## もう一段だけ深ぼってみる #1

最初エラーになったメッセージの中で `Infallible` というあまり見かけないエラー型が登場しました。これは何だったのでしょうか？

```rust
expected struct 'Age'
  found enum 'Result<Age, Infallible>'
```

`Infallible` を辞書でひいてみると `誤りのない` とか `絶対に正しい` といった意味のようです。おそらく `エラーになることがない` エラーみたいな立ち位置なのではないでしょうか。

公式をみると、どうやら `Result` が必ず `Ok` を返す時に指定する慣習?があるみたいですね。

https://doc.rust-lang.org/std/convert/enum.Infallible.html

またRustには[never型](https://doc.rust-jp.rs/book-ja/ch19-04-advanced-types.html#never%E5%9E%8B%E3%81%AF%E7%B5%B6%E5%AF%BE%E3%81%AB%E8%BF%94%E3%82%89%E3%81%AA%E3%81%84)と呼ばれる `!` という名前の特別な型がありますが、これと同じ役割と記載があります。

```rust
pub type Infallible = !;
```

`Infallible` もゆくゆくは `!` に統合されてaliasとなっていく予定があるようです。


## 振り返り

今回も結局 [core/convert/mod.rs](https://doc.rust-lang.org/src/core/convert/mod.rs.html) まで読みにいってしまいました。徐々にですが「これはどんな条件で自動実装されるのか?」みたいな視点で、impl節をパラパラと見て回るような動きができてきた?ので、すこーしだけ成長を感じます。

これで明日から、もっと堂々と `try_from` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_3_try_from_trait

