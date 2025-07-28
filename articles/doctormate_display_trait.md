---
title: "Rust の Display トレイトについて執筆します"
emoji: "😀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
publication_name: doctormate
---

# std::fmt::Display トレイトを実装しよう

構造体に対して `println!` や `to_string()` を利用したくなった時に必要となるのが `std::fmt::Display` トレイトを実装することです。

Rustを学び始めた最初の頃は、Display ってなんぞ？と思ってました😅

今回は基本に立ち返って `std::fmt::Display` トレイトを実装してみます。


## std::fmt::Display トレイトを実装する構造体を用意してみる

下準備として、Person という構造体を用意してみます。
構造体には、nickname と age フィールド、そのゲッターを定義してみます。

```rust
struct Person {
    nickname: String,
    age: u8,
}

impl Person {
    fn nickname(&self) -> &String {
        &self.nickname
    }
    
    fn age(&self) -> u8 {
        self.age
    }
}

fn main() {
    let yoshida = Person { nickname: String::from("yosshi-"), age: 35 };
    yoshida.to_string();      // エラーになるはず
    println!("{}", yoshida);  // エラーになるはず
}
```

実行してみると `std::fmt::Display` が `Person` 構造体に実装されていませんよ、というコンパイルエラーになりました（狙い通り）

```
error[E0599]: `Person` doesn't implement `std::fmt::Display`
```

## std::fmt::Display トレイトを実装してみる

`Persion` 構造体に `std::fmt::Display` トレイトを実装してみます。

VSCodeを利用して順を追ってやってみます。まずはここまで書きます。

```rust
impl std::fmt::Display for Person {
    
}
```

VSCode が `std::fmt::Display` 部分で問題を検知し赤い波下線を引いてくれているはずです。`std::fmt::Display` 部分にカーソルを当てて `cmd + .` を押します（Macの場合）。

`Implement missing members` という選択肢を押してみます。

```rust

impl std::fmt::Display for Doctor {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        todo!()
    }
}
```

`std::fmt::Display` トレイトを実装する場合に必ず宣言しなければいけない `fmt` メソッドのガワを用意してくれます！（ありがたい）

fmt メソッドの中身を書いていきます。

```rust
impl std::fmt::Display for Person {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "I am {} {} years old.", self.nickname, self.age)
    }
}
```

`write!` マクロについては省略します。

## std::fmt::Display トレイトを実装して際実行

実行してみます。

```
I am yosshi- 35 years old.
```

期待通り出力されました、嬉しいです。

## 振り返り

Rust初学者だった頃は、`Display` を実装しようとなったときに `std::fmt` が出てこずに苦労しました。`use std::fmt;` と書かないといけないんだっけ？レベルです。今となってはですが、最初の頃はそんなことで躓いてしまうものですよね。
皆さんはどうやって乗り越えてたんだろう？
