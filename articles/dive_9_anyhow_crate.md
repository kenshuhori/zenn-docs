---
title: "足を止めて見る #9 〜 Rustのanyhowクレート(1) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2026-02-16 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの9つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_8_thiserror_crate)の記事では `thiserror` の `helper attributes` をそれぞれ確認し、カスタムエラー型を定義する方法を見てきました。

`thiserror` は、エラー型を丁寧に定義する際にとても便利なクレートでした。

`anyhow` は一方で、エラーへの変換を容易にしたりと、簡潔で柔軟な管理ができます。

今回はそんな `anyhow` クレートに改めて足を止めて見てみます。


## anyhow クレートをインストール

まずは `anyhow` をインストールします。

```console
$ cargo add anyhow
    Updating crates.io index
      Adding anyhow v1.0.102 to dependencies
             Features:
             + std
             - backtrace
    Updating crates.io index
     Locking 1 package to latest Rust 1.89.0 compatible version
      Adding anyhow v1.0.102
```

すると `Cargo.toml` に次の依存関係が追加されます。

```toml
[dependencies]
anyhow = "1.0.102"
```

`anyhow` には `std` と `backtrace` という `feature` があり、デフォルトでは `std` が有効になるようですね。


## anyhow クレートを使ってみる(1)

簡単な例を書いてみます。

```rust
fn main() {
    match read_file_ext() {
        Ok(_) => println!("成功"),
        Err(e) => println!("エラー: {e}"),
    }
}

fn read_file_ext() -> anyhow::Result<String> {
    let text = match read_file() {
        Ok(text) => text,
        Err(e) => return Err(anyhow::anyhow!(e)),
    };
    Ok(text)
}

fn read_file() -> std::result::Result<String, std::io::Error> {
    std::fs::read_to_string("not_found.txt")
}
```

`read_file_ext` では、内部で `read_file` を呼び出し、`anyhow::Result` 型を返しています。

```rust
anyhow::Result<String>
```

これは実際には次の型のエイリアスです。

```rust
std::result::Result<String, anyhow::Error>
```

つまり `anyhow::Result<T>` は `Result<T, anyhow::Error>` を簡潔に書くための型エイリアスになっています。

これは [anyhow の Docs.rs](https://docs.rs/anyhow/latest/anyhow/type.Result.html) の冒頭にも記載があります。

```rust
pub type Result<T, E = Error> = Result<T, E>;
```

左辺の `Result` は `std::result::Result` で、`E=Error` は `anyhow::Error` で、指定がなければ `anyhow::Error` になるというわけですね。

また `anyhow!` マクロは、`anyhow::Error` 型を生成します。

これにより、`read_file_ext` は、成功時はテキスト（`String`）、失敗時はanyhowエラー（`anyhow::Error`）を返す関数になっており、成立していることがわかります。

## anyhow クレートを使ってみる(2)

`anyhow` のメリットをもう少し享受します。先ほどのコードはさらに以下のように書くことができます。

```rust
pub(crate) fn main() {
    match read_file_ext() {
        Ok(_) => println!("成功"),
        Err(e) => println!("エラー: {e}"),
    }
}

fn read_file_ext() -> anyhow::Result<String> {
    let text = read_file()?;
    Ok(text)
}

fn read_file() -> std::result::Result<String, std::io::Error> {
    std::fs::read_to_string("not_found.txt")
}
```

`read_file_ext` の中でパターンマッチの代わりに、?演算子（`question mark operator`）が使われるだけの簡潔な形が成立しました。

この ?演算子の挙動は、TRPL（The Rust Programming Language）の
「9.2 Resultで回復可能なエラー」でも説明されています。

https://doc.rust-jp.rs/book-ja/ch09-02-recoverable-errors-with-result.html

つまり ?演算子は

- Ok の場合 → 中身を取り出す
- Err の場合 → return Err(...) する

という処理を簡潔に書くための構文です。

ではなぜ ?演算子 が利用できたのでしょうか？

?演算子はこの場合、エラー時は `std::io::Error` を `anyhow::Error` に変換してreturnできる必要があります。

`anyhow::Error` は `std::error::Error` トレイトを実装した型をトレイト境界にて指定しています。

また `std::io::Error` は `std::error::Error` トレイトを実装したものです。

なので、?演算子が使えるという流れでした。


## 振り返り

今回は `anyhow` クレートを改めて足を止めて見てみました。

`anyhow::Error` は `std::error::Error` トレイトを実装するものであれば何でも扱えるという点によって、

?演算子による変換が簡単に行えるんだな、という理解ができました。

`std::error::Error` との関係性は掴めたので、次は `thiserror` との組み合わせなども見ていきたいと思います。

これで明日から、もっと堂々と `anyhow` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_9_anyhow_crate
