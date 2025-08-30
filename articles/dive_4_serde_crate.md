---
title: "足を止めて見る #4 〜 RustのSerdeクレート 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2025-09-11 12:00
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

`-F derive` としたことで、serde の derive feature が追加されました。こちらについては後ほど深ぼってみます。

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

## もう一段だけ深ぼってみる

## 振り返り

これで明日から、もっと堂々と `serde` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_4_try_from_trait

