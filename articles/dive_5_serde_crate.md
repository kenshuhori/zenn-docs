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

構造体を Json 形式にデシリアライズして、その Json 形式をしリアライズして、元の構造体に戻る様子を見ました。

今回は `serde` クレートの `Attributes` について足を止めて見ようと思います。

https://serde.rs/attributes.html

## serde の attributes とは何なのか

[前回](https://zenn.dev/doctormate/articles/dive_4_serde_crate)は、dervieマクロによって自動的に `serde::ser::Serialize` や `serde::de::Deserialize` が実装されることを確認しました。

attributes とは、そのderiveマクロによる振る舞いを調整するための機能のようで、3種類あります。

- [Container attributes](https://serde.rs/container-attrs.html)
- [Variant attributes](https://serde.rs/variant-attrs.html)
- [Field attributes](https://serde.rs/field-attrs.html)



## もう一段だけ深ぼってみる

## 振り返り

今回も `serde` クレートを改めて足を止めて見てみました。

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_5_serde_crate

