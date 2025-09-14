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

`proc_macro_derive(Serialize` とあることから、構造体に `#[derive(Serialize)]` という記述を添えることでこの `derive_serialize` が実行される形になります。

また derive マクロは追加の属性を受け取ることができ、それは `attributes(serde)` の記載で見て取れます。これにより[前回](https://zenn.dev/doctormate/articles/dive_5_serde_crate)の記事で触れた serde の `Attributes` という機能が `#[serde]` という属性を付与することで実現できているわけですね。

[TRPL19章](https://doc.rust-jp.rs/book-ja/ch19-06-macros.html)にあるとおり、deriveマクロを含む手続き的マクロは、引数にTokenStream型のinputを受け取ります。

通常はこのinputを `syn::parse(input).unwrap();` とするようですが、serdeでは `parse_macro_input!(input as DeriveInput);` としています。

この [parse_macro_input!](https://docs.rs/syn/2.0.104/src/syn/parse_macro_input.rs.html#108-128) の実装を見てみましたが、内部的に `syn::parse` が実行されていました。`syn::parse` とするとResultが返ってくるのですが、 `parse_macro_input!` マクロではErrの場合はコンパイルエラーにしてしまうようですね。

では本題の `expand_derive_serialize` 関連関数に入っていきます。[serde_derive/ser.rs#11](https://docs.rs/serde_derive/latest/src/serde_derive/ser.rs.html#11-61)ですね。

```rust
pub fn expand_derive_serialize(input: &mut syn::DeriveInput) -> syn::Result<TokenStream> {
    replace_receiver(input);

    let ctxt = Ctxt::new();
    let cont = match Container::from_ast(&ctxt, input, Derive::Serialize) {
        Some(cont) => cont,
        None => return Err(ctxt.check().unwrap_err()),
    };
    precondition(&ctxt, &cont);
    ctxt.check()?;

    let ident = &cont.ident;
    let params = Parameters::new(&cont);
    let (impl_generics, ty_generics, where_clause) = params.generics.split_for_impl();
    let body = Stmts(serialize_body(&cont, &params));
    let serde = cont.attrs.serde_path();

    let impl_block = if let Some(remote) = cont.attrs.remote() {
        let vis = &input.vis;
        let used = pretend::pretend_used(&cont, params.is_packed);
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
    } else {
        quote! {
            #[automatically_derived]
            impl #impl_generics #serde::Serialize for #ident #ty_generics #where_clause {
                fn serialize<__S>(&self, __serializer: __S) -> #serde::__private::Result<__S::Ok, __S::Error>
                where
                    __S: #serde::Serializer,
                {
                    #body
                }
            }
        }
    };

    Ok(dummy::wrap_in_const(
        cont.attrs.custom_serde_path(),
        impl_block,
    ))
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

