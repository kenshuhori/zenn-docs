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

fn read_file() -> anyhow::Result<String> {
    std::fs::read_to_string("config.json")
        .context("設定ファイルの読み込みに失敗しました")
}
```

エラーが発生したときに `「設定ファイルの読み込みに失敗しました」` という情報が追加され、原因の特定がしやすくなります。

### with_context

context の遅延評価版です。

`with_context()` も [anyhow::Context トレイト](https://docs.rs/anyhow/latest/anyhow/trait.Context.html)によって提供されているため、
使用するには `use anyhow::Context;` が必要になります。

```rust
use anyhow::Context;

fn read_file() -> anyhow::Result<String> {
    fs::read_to_string("config.json")
        .with_context(|| "設定ファイルの読み込みに失敗しました")
}
```

違いは、クロージャで渡すことで「エラーが発生したときだけ評価される」点です。

- 軽い処理 → context
- 重い処理 or フォーマット付き → with_context

と覚えておくと良いです。

### anyhow! マクロ

任意のエラーを簡単に生成できます。

```rust
fn validate(x: i32) -> anyhow::Result<()> {
    if x < 0 {
        return Err(anyhow::anyhow!("x must be positive, got {}", x));
    }
    Ok(())
}
```

### bail! マクロ

エラーを生成して即座に return します。

```rust
fn validate(x: i32) -> anyhow::Result<()> {
    if x < 0 {
        anhhow::bail!("x must be positive");
    }
    Ok(())
}
```

`return Err(anyhow!(...))` のショートハンドです。

### ensure! マクロ

条件を満たさない場合にエラーを返します。

```rust
fn validate(x: i32) -> anyhow::Result<()> {
    anyhow::ensure!(x >= 0, "x must be positive");
    Ok(())
}
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
