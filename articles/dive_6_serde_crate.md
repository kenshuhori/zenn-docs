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

[前回](https://zenn.dev/doctormate/articles/dive_5_serde_crate)は serde クレートについてでした。

serde の Attributes という機能を確認し、deriveマクロによって実現されている様子を確認しました。

今回は、その derive マクロがどのように作られているのか、追いかけながらderiveマクロについて深掘りしていこうと思います。

## serde の derive マクロを追いかける

前回は [serde_derive/lib.rs:#91](https://docs.rs/serde_derive/latest/src/serde_derive/lib.rs.html#91) に、dervieマクロの宣言が記述されているところまで突き止めました。

```rust
#[proc_macro_derive(Serialize, attributes(serde))]
pub fn derive_serialize(input: TokenStream) -> TokenStream {
    let mut input = parse_macro_input!(input as DeriveInput);
    ser::expand_derive_serialize(&mut input)
        .unwrap_or_else(syn::Error::into_compile_error)
        .into()
}
```



## もう一段だけ深ぼってみる

TODO

## 振り返り

今回も `serde` クレートを改めて足を止めて見てみました。

これで明日から、もっと堂々と `serde` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_6_serde_crate

