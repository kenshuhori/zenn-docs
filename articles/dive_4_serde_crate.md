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

具体的に言うと、Rustの構造体をたとえばJSON形式に変換したり、たとえばCSV形式のデータをRustの構造体に変換したりする際に利用できます。

## serde クレートを使ってみる

## もう一段だけ深ぼってみる

## 振り返り

これで明日から、もっと堂々と `serde` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_4_try_from_trait

