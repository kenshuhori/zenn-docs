---
title: "足を止めて見る #5 〜 RustのSerdeクレート(2) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
published_at: 2025-09-18 12:00
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

いくつかの attributes をピックアップして動作を見て見ようと思います。

## serde クレートの attributes を使ってみる

今回は Person という構造体を用意して、いくつか attributes を使ってみようと思います。3種類の attributes を試すために、Person 構造体のフィールドには、構造体 `Age` をとる age フィールド、enum `Gender` をとる gender フィールドなどを用意してみました。

```rust
#[derive(serde::Serialize)]
struct Person {
    first_name: String,
    last_name: String,
    age: Age,
    gender: Gender,
}

#[derive(serde::Serialize)]
struct Age {
    value: u8,
}

#[derive(serde::Serialize)]
enum Gender {
    Male,
    Female,
    Other,
}

fn main() {
    let person = Person {
        first_name: String::from("太郎"),
        last_name: String::from("田中"),
        age: Age { value: 30 },
        gender: Gender::Male,
    };

    let serialized = serde_json::to_string(&person).unwrap();

    assert_eq!(
        serialized,
        "{\"first_ame\":\"太郎\",\"last_ame\":\"田中\",\"age\":{\"value\":30},\"gender\":\"Male\"}"
    );
}

```

この実行は成功して、以下のJsonになります。

```json
{
    "first_name": "太郎",
    "last_name": "田中",
    "age": { "value": 30 },
    "gender": "Male"
}
```

### rename_all

rename_all を使うと、スネークケースで記述しているフィールド名をキャメルケースでシリアライズできました

```rust
#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")] // 追加
struct Person {
    first_name: String,
    last_name: String,
    age: Age,
    gender: Gender,
}
```

シリアライズされた Json の形

```json
{
    "firstName": "太郎", // first_name -> firstName
    "lastName": "田中",  // last_name -> lastName
    "age": { "value": 30 },
    "gender": "Male"
}
```

### rename

rename を使うと、enum の `Variant` や struct の `フィールド` の名前を変更できました

```rust
#[derive(serde::Serialize)]
enum Gender {
    #[serde(rename = "男")] // 追加
    Male,
    #[serde(rename = "女")] // 追加
    Female,
    #[serde(rename = "その他")] // 追加
    Other,
}
```

シリアライズされた Json の形

```json
{
    "firstName": "太郎",
    "lastName": "田中",
    "age": { "value": 30 },
    "gender": "男" // Male -> 男
}
```

### transparent

transparent を使うと、フィールドを1つしか持たない構造体がネストされている時に、まるでプリミティブ型のフィールドがそこにあるかのような振る舞いができました。これは New Type パターンを想定した構造体に対して利用する狙いがあるようです。

文字に起こしてもちょっと伝わりづらいので書いてみます。


```rust
#[derive(serde::Serialize)]
#[serde(transparent)] // 追加
struct Age {
    value: u8,
}
```

シリアライズされた Json の形

```json
{
    "firstName": "太郎",
    "lastName": "田中",
    "age": 30, // { "value": 30 } -> 30
    "gender": "男"
}
```

わかりやすくなりました。後で試してみたのですが、もし Age がu8をとるタプル構造体だった場合もこの形になりました。

```rust
// 以下2つのシリアライズ結果は同じになりました

#[derive(serde::Serialize)]
#[serde(transparent)]
struct Age {
    value: u8,
}

#[derive(serde::Serialize)]
struct Age (u8)
```

## もう一段だけ深ぼってみる(#1)

そもそも `#[serde(xxx = "yyy")]` といった記述は何者なんでしょうか。

これは自分知ってました、カスタムのdervieマクロってやつでしょう（違ったらごめん）

https://doc.rust-jp.rs/book-ja/ch19-06-macros.html#%E5%B1%9E%E6%80%A7%E9%A2%A8%E3%83%9E%E3%82%AF%E3%83%AD

おそらく `serde` のどこかに `proc_macro_derive` といった記述があり、そこにロジックが書かれているはずです。

## もう一段だけ深ぼってみる(#2)

では `#[serde(xxx = "yyy")]` はどのようなメカニズムなのでしょうか？

これを紐解くために、まずは `serde` クレートの `derive` という `crate feature` から辿ってみます。

[serdeのCargo.toml](https://github.com/serde-rs/serde/blob/b4677fde9d743cb5e5951a53a79e3072b9c190d3/serde/Cargo.toml)を覗いてみます。

```toml
{{ 省略 }}

[dependencies]
serde_derive = { version = "1", optional = true, path = "../serde_derive" }

{{ 省略 }}

### FEATURES #################################################################

[features]
default = ["std"]

# Provide derive(Serialize, Deserialize) macros.
derive = ["serde_derive"]

{{ 省略 }}
```

`derive` feature を指定するということは、この `Carto.toml` の1階層上にある `serde_derive` というディレクトリに依存するようです。

この `serde_derive` ですが、[crates.io](https://crates.io/) でも検索できるように、1つの独立したクレートのようです。

https://crates.io/crates/serde_derive

どうやらこのクレートにロジックがあるようです。宣言部分のソースコード [serde_derive/lib.rs](https://docs.rs/serde_derive/latest/src/serde_derive/lib.rs.html#91) をのぞいてみます。

```rust
#[proc_macro_derive(Serialize, attributes(serde))]
pub fn derive_serialize(input: TokenStream) -> TokenStream {
    let mut input = parse_macro_input!(input as DeriveInput);
    ser::expand_derive_serialize(&mut input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```

やりました！ `proc_macro_dervie` という記述に辿り着きました！

`attributes(serde)` とあるので `#[serde]` という名前のカスタムderiveマクロになっているのでしょうね（TRPLではここまでは教えてくれなかったぞ）。

細かくロジックを追いかけるのは、また今度にします。

## 振り返り

今回も `serde` クレートを改めて足を止めて見てみました。

attributes の使い方がざっくりですが分かりました。

また `crate feature` を Cargo.toml から辿っていき、カスタムのdervieマクロの宣言まで追いかけることができたのは、Rustの習熟を深めるためにも良かったのではないでしょうか。

これで明日から、もっと堂々と `serde` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_5_serde_crate

