---
title: "足を止めて見る #6 〜 RustのSerdeクレート(3) 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2025-09-25 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの6つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_5_serde_crate)は serde の Attributes という機能を確認し、deriveマクロによって実現されている様子を確認しました。

今回は、その derive マクロを使わずに自分で impl してみるとどうなるか、追いかけてみようと思います。

serde の公式ドキュメントにも [Custom serialization](https://serde.rs/custom-serialization.html) という章があり、deriveマクロよりも更にカスタマイズするために自分で実装する手段について提示してくれています。こちらを参考に進めます。

## serde::ser::Serializeを自分でimplする

https://serde.rs/impl-serialize.html

## もう一段だけ深ぼってみる

TODO

## 振り返り

今回も `serde` クレートを改めて足を止めて見てみました。

これで明日から、もっと堂々と `serde` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_6_serde_crate

