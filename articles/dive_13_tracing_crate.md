---
title: "足を止めて見る #13 〜 Rustのtracingクレート(2) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2026-05-14 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの13回目です。

[前回](https://zenn.dev/doctormate/articles/dive_12_tracing_crate)の記事では `tracing` クレートの基礎的な使い方を確認しました。

## TODO: なにか書く

## TODO: もう一段だけ深掘ってみる

少し脱線になりますが `let _enter = span.enter()` の左辺 `_enter` は必要なのでしょうか？

```rust
fn main() {
    let span = span!(Level::INFO, "main");
    let _enter = span.enter();
}
```

利用しない変数ならば `let _ = span.enter()` や、そもそも `span.enter()` と書いてしまう方が自然です。

でも、この `_enter` は `_` と書いても省略してもダメなんです。なぜか。

`span.enter()` は guard を返しているからです。

「む... guard って... ？」

guard とは、スコープを抜ける時に自動でメモリを解放する振る舞いを持った値を返すものを指します。

つまり `Drop` を実装しているはずです。見てみます。

`span.enter()` の返り値型は `span::Entered` です。こちらのドキュメントを見てみると...

https://docs.rs/tracing/latest/tracing/span/struct.Entered.html

`impl Drop for Entered<'_>` の記載がありますね。


## TODO: 振り返り

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_13_tracing_crate
