---
title: "足を止めて見る #11 〜 Rustのanyhowクレート(2) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2026-03-18 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの11つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_10_anyhow_crate)の記事では `?演算子` とは？を歴史的な経緯と主に見てきました。

今回は `anyhow` クレートが提供しているメソッドを確認して、引き出しを増やす回にしようと思います。

## anyhow クレートが提供しているメソッドたち

### context

エラーに追加の文脈（コンテキスト）を付与するメソッドです。

`context()` は [anyhow::Context トレイト](https://docs.rs/anyhow/latest/anyhow/trait.Context.html)によって提供されているため、
使用するには `use anyhow::Context;` が必要になります。

```rust
use anyhow::Context;

fn main () {
    match read_file() {
        Ok(content) => println!("ファイルの内容: {}", content),
        Err(e) => eprintln!("{}", e),
    }
}

fn read_file() -> anyhow::Result<String> {
    std::fs::read_to_string("nonexist.txt")
        .context("ファイルの読み込みに失敗しました")
}
```

エラーが発生したときに `「ファイルの読み込みに失敗しました」` という情報が追加され、原因の特定がしやすくなります。

```sh
$ cargo run

// 出力
ファイルの読み込みに失敗しました
```

### with_context

こちらも、エラーに追加の文脈（コンテキスト）を付与するメソッドです。違いは、クロージャを渡すかどうかです。

`with_context()` も [anyhow::Context トレイト](https://docs.rs/anyhow/latest/anyhow/trait.Context.html)によって提供されているため、
使用するには `use anyhow::Context;` が必要になります。

```rust
use anyhow::Context;

fn read_file() -> anyhow::Result<String> {
    fs::read_to_string("nonexist.txt")
        .with_context(|| "ファイルの読み込みに失敗しました")
}
```

クロージャに関しては [TRPL13章](https://doc.rust-jp.rs/book-ja/ch13-01-closures.html) が参考になります。エラー時にのみ評価されるため、 `format!` や関数呼び出しを伴う場合は `with_context` を使う方が良いでしょう。


```sh
$ cargo run

// 出力
ファイルの読み込みに失敗しました
```

### anyhow! マクロ

型を気にせず、とりあえずエラーを作れる便利マクロです。

```rust
fn read_file() -> anyhow::Result<String> {
    let content = std::fs::read_to_string("nonexist.txt");
    match content {
        Ok(content) => Ok(content),
        Err(e) => Err(anyhow::anyhow!("ファイルの読み込みに失敗しました: {}", e)),
    }
}
```

`"ファイルの読み込みに失敗しました"` という文字列リテラルを `anyhow::Error` に包んでくれています。

```sh
$ cargo run

// 出力
ファイルの読み込みに失敗しました: No such file or directory (os error 2)
```

### bail! マクロ

エラーを生成して即座に return する便利マクロです。

```rust
fn read_file() -> anyhow::Result<(String)> {
    let content = std::fs::read_to_string("nonexist.txt");
    match content {
        Ok(content) => Ok(content),
        Err(e) => anyhow::bail!("ファイルの読み込みに失敗しました: {}", e),
    }
}
```

`return Err(anyhow!(...))` のショートハンドです。

内部的に `anyhow!` でエラーを生成し、そのエラーを即座に返して関数を抜けるところまでやってくれるわけですね。

```sh
$ cargo run

// 出力
ファイルの読み込みに失敗しました: No such file or directory (os error 2)
```

### ensure! マクロ

条件を満たさない場合に、エラーを生成して即座に return する便利マクロです。

```rust
fn read_file() -> anyhow::Result<(String)> {
    let content = std::fs::read_to_string("exist.txt")?;
    anyhow::ensure!(
        content.len() > 0,
        "ファイルの中身が空です"
    );
    Ok(content)
}
```

振る舞いは bail! マクロとは似ており、実質は以下を表していると言えます。

```rust
if !(content.len() > 0) {
    anyhow::bail!("error");
}
```

```sh
$ cargo run

// 出力
ファイルの中身が空です
```

### downcast_ref / downcast

anyhow::Error の中身を具体的な型に戻すためのメソッドです。

```rust
if let Some(e) = err.downcast_ref::<std::io::Error>() {
    println!("io error: {}", e);
}
```

基本はあまり使いませんが、「どうしても型を見たいとき」に使います。

バリデーション処理でかなり便利です。

### chain

エラーの原因チェーンを辿ることができます。

```rust
for cause in err.chain() {
    println!("{}", cause);
}
```

ログ出力やデバッグで役立ちます。

## もう一段だけ深ぼってみる

## 振り返り

これで明日から、もっと堂々と `anyhow` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_10_anyhow_crate
