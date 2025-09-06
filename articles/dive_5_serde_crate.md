---
title: "足を止めて見る #5 〜 RustのSerdeクレート(2) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2025-09-18 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの5つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_4_serde_crate)は serde クレートについてでした。

構造体を Json 形式にシリアライズして、更にその Json 形式をデシリアライズして、元の構造体に戻る様子を見ました。

今回は `serde` クレートの `Attributes` について足を止めて見ようと思います。

https://serde.rs/attributes.html

## serde の attributes とは何なのか

[前回](https://zenn.dev/doctormate/articles/dive_4_serde_crate)は、dervieマクロによって自動的に `serde::ser::Serialize` や `serde::de::Deserialize` が実装されることを確認しました。

attributes とは、そのderiveマクロによって実装されるその振る舞いを調整するための機能のようで、3種類あります。

- [Container attributes](https://serde.rs/container-attrs.html)
- [Variant attributes](https://serde.rs/variant-attrs.html)
- [Field attributes](https://serde.rs/field-attrs.html)

それぞれどこに記述する attributes なのかは、公式Docの例がわかりやすいです。

```rust
#[derive(Serialize, Deserialize)]
#[serde(deny_unknown_fields)]  // <-- this is a container attribute
struct S {
    #[serde(default)]  // <-- this is a field attribute
    f: i32,
}

#[derive(Serialize, Deserialize)]
#[serde(rename = "e")]  // <-- this is also a container attribute
enum E {
    #[serde(rename = "a")]  // <-- this is a variant attribute
    A(String),
}
```

attributes 沢山ありますね...いくつかピックアップして動作を見て見ようと思います。

### rename_all (Container attributes)

今回は Person という構造体を用意して、いくつか attributes を使ってみようと思います。3種類の attributes を試すために、Person 構造体のフィールドには、ユニット様構造体 `Age` をとる age フィールド、enum `Gender` をとる gender フィールドなどを用意してみました。

```rust
#[derive(serde::Serialize)]
struct Person {
    first_name: String,
    last_name: String,
    age: Age,
    gender: Gender,
}

#[derive(serde::Serialize, serde::Deserialize, PartialEq, Debug)]
struct Age(u8);

#[derive(serde::Serialize, serde::Deserialize, PartialEq, Debug)]
enum Gender {
    Male,
    Female,
    Other,
}

fn main() {
    let person = Person {
        first_name: String::from("太郎"),
        last_name: String::from("田中"),
        age: Age(30),
        gender: Gender::Male,
    };

    let serialized = serde_json::to_string(&person).unwrap();

    assert_eq!(
        serialized,
        "{\"first_name\":\"太郎\",\"last_name\":\"田中\",\"age\":30,\"gender\":\"Male\"}"
    );
}
```



## もう一段だけ深ぼってみる

## 振り返り

今回も `serde` クレートを改めて足を止めて見てみました。

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_5_serde_crate

