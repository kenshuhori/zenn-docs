---
title: "足を止めて見る #4 〜 RustのSerdeクレート(1) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
published_at: 2025-09-11 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの4つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_3_try_from_trait)は TryFrom トレイトについてでした。

今回は、Rustの定番クレートの1つと言ってもいい `serde` クレートを足を止めて見てみます。

## serde クレートとは

とにかくまずは [docs.rs](https://docs.rs/) で `serde` クレートを見に行きましょう。

https://docs.rs/serde/latest/serde/

「Rustの構造体をシリアライズしたりデシリアライズするのに便利なフレームワークです」との紹介が冒頭にあります。

具体的に言うと、Rustの構造体をたとえばJSON形式にシリアライズしたり、たとえばCSV形式のデータをRustの構造体にデシリアライズする際に有用です。

## serde クレートをインストール

まずはインストールですね。

```console
$ cargo add serde -F derive
$ cargo add serde_json
```

`Cargo.toml` に以下の依存関係が追加されました。

```toml
[dependencies]
serde = { version = "1.0.219", features = ["derive"] }
serde_json = "1.0.143"
```

`-F derive` としたことで derive という `crate feature` が追加されました。こちらについては後ほど深ぼってみます。

`serde_json` というクレートも追加しています。これは `JSON` 形式を扱う場合に利用します。

## serde クレートを使ってみる

今回は Person という構造体を用意して、それをJSON形式にシリアライズし、さらにそれをデシリアライズして Person 構造体に戻すことを試してみます。

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct Person {
    nickname: String,
    age: u8,
}

fn main() {
    let person = Person {
        nickname: String::from("タロー"),
        age: 30,
    };

    let serialized = serde_json::to_string(&person).unwrap();
    let deserialized: Person = serde_json::from_str(&serialized).unwrap();

    assert_eq!(serialized, "{\"nickname\":\"タロー\",\"age\":30}");
    assert_eq!(person.nickname, deserialized.nickname);
    assert_eq!(person.age, deserialized.age);
}
```

このように書けました。実行するとこちらは成功します。

`serde_json::to_string` にPerson構造体のデータを渡すと、たしかに以下のようなJSONになります。

```json
{
    "nickname": "タロー",
    "age": 30
}
```

またこのJSONを `serde_json::from_str` に渡すと、たしかにPerson構造体になります。

なんとなく雰囲気は掴めました。

## もう一段だけ深ぼってみる #1

`serde` はどのようにしてPerson構造体のデータをJSONに変換できたのでしょうか？

うっすら気付いていると思いますが `#[derive(serde::Serialize)]` のおかげです。

試しに `#[derive(serde::Serialize, serde::Deserialize)]` の行をコメントアウトして保存するとコンパイルエラーになり、次のようなエラーメッセージが表示されます。

```
the trait bound `Person: serde::ser::Serialize` is not satisfied
```

つまり Person構造体に `serde::ser::Serialize` を `impl` しておくれ、と怒られています。

だとすると `#[derive(serde::Serialize)]` を書けば `serde::ser::Serialize` が自動的に実装されると理解できます。

いつものようにソースコードを見てみます。

https://docs.rs/serde_derive/1.0.219/src/serde_derive/lib.rs.html#92

```rust
#[proc_macro_derive(Serialize, attributes(serde))]
pub fn derive_serialize(input: TokenStream) -> TokenStream {
    let mut input = parse_macro_input!(input as DeriveInput);
    ser::expand_derive_serialize(&mut input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```

`derive` は宣言的マクロと呼ばれているもので、[TRPL19章_高度な機能](https://doc.rust-jp.rs/book-ja/ch19-06-macros.html)にも紹介があります。（難しいですね）

詳しいことは分かりませんが、とりあえず `ser` modの `expand_derive_serialize` 関連関数が呼ばれています。

https://docs.rs/serde_derive/1.0.219/src/serde_derive/ser.rs.html#11-61

```rust
pub fn expand_derive_serialize(input: &mut syn::DeriveInput) -> syn::Result<TokenStream> {
    // { 途中省略 }
    quote! {
        #[automatically_derived]
        impl #impl_generics #ident #ty_generics #where_clause {
            #vis fn serialize<__S>(__self: &#remote #ty_generics, __serializer: __S) -> #serde::__private::Result<__S::Ok, __S::Error>
            where
                __S: #serde::Serializer,
            {
                #used
                #body
            }
        }
    }
    // { 途中省略 }
}
```

`expand_derive_serialize` 関連関数をみてみると、何やら `impl` ブロックを生成しているような記述がありました。きっとこのあたりの記述により、自動的に `serde::ser::Serialize` が実装される状態にしてくれているのでしょうね。

## もう一段だけ深ぼってみる #2

`cargo add serde -F derive` とすることで dervie という `crate feature` が追加されました。こちらは一体何なのでしょうか。

もうお分かりかと思いますが、先ほどの `dervie` マクロを利用するためには、このfeatureが必要とのことです。

https://docs.rs/serde/latest/serde/derive.Serialize.html

今回は一旦そうなんだくらいで終わりにしようと思います。

## 振り返り

今回は `serde` クレートを改めて足を止めて見てみました。

これまでのシリーズと比較すると `crate feature` や `deriveマクロ` など、やっぱり難易度が一段と上がってきました。

`serde` に関しては、もう少し足を止めてみた方が良さそうなので、もう何個か調べて見ようと思います。

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_4_serde_crate

