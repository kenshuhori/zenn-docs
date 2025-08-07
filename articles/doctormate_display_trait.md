---
title: "足を止めて見る #1 〜 RustのDisplayトレイト 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
publication_name: doctormate
---

# 足を止めて見よう

皆さんは日々仕事をしていく中で、分かっているつもりで利用している技術が一体どのような原理で動いているのか？ちゃんと振り返っていますか？

過去を振り返ると、自分のエンジニア力が向上していた時って「分からないことに直面したら分かるまで調べる」を地道にやっていた期間だなと感じます。

僕自身はこれを「足を止めて見る」と呼んでいます。

僕は会社のテックブログのネタを利用して、この「足を止めて見る」をやっていこうと思います！

## std::fmt::Display トレイトを実装しよう

構造体に対して `println!` や `to_string()` を利用したくなった時に必要となるのが `std::fmt::Display` トレイトを実装することですよね。

Rustを学び始めた最初の頃は、Display ってなんぞ？と思ってました😅

今回は足を止めて `std::fmt::Display` トレイトを実装してみます。


## std::fmt::Display トレイトを実装する構造体を用意してみる

下準備として、Person という構造体を用意してみます。
構造体には、nickname と age フィールド、そのゲッターを定義してみます。

```rust
struct Person {
    nickname: String,
    age: u8,
}

impl Person {
    fn nickname(&self) -> &String {
        &self.nickname
    }
    
    fn age(&self) -> u8 {
        self.age
    }
}

fn main() {
    let yoshida = Person { nickname: String::from("yosshi-"), age: 35 };
    yoshida.to_string();      // エラーになるはず
    println!("{}", yoshida);  // エラーになるはず
}
```

実行してみると `std::fmt::Display` が `Person` 構造体に実装されていませんよ、というコンパイルエラーになりました（狙い通り）

```
error[E0599]: `Person` doesn't implement `std::fmt::Display`
```

## std::fmt::Display トレイトを実装してみる

`Persion` 構造体に `std::fmt::Display` トレイトを実装してみます。

[rust-analyzer](https://github.com/rust-lang/rust-analyzer)という拡張機能をインストールしたVSCode環境を利用して、順を追ってやってみます。
まずはここまで書きます。

```rust
impl std::fmt::Display for Person {
    
}
```

VSCode が `std::fmt::Display` 部分で問題を検知し赤い波下線を引いてくれているはずです。`std::fmt::Display` 部分にカーソルを当てて `cmd + .` を押します（Macの場合）。

`Implement missing members` という選択肢を押してみます。

```rust

impl std::fmt::Display for Doctor {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        todo!()
    }
}
```

`std::fmt::Display` トレイトを実装する場合に必ず宣言しなければいけない `fmt` メソッドのガワを用意してくれます！（ありがたい）

fmt メソッドの中身を書いていきます。

```rust
impl std::fmt::Display for Person {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "I am {} {} years old.", self.nickname, self.age)
    }
}
```

`write!` マクロについては省略します。

## std::fmt::Display トレイトを実装して際実行

実行してみます。

```
I am yosshi- 35 years old.
```

期待通り出力されました、嬉しいです。

## もう一段だけ深ぼってみる

ひとつだけ疑問が残りました。

`fmt` メソッドは `std::fmt::Result` を返していますが、`to_string()` の戻り値型は `Result` ではありません。いったいどういうこと？

では `string.rs` を見に行ってみましょう。


https://doc.rust-lang.org/src/alloc/string.rs.html#2805
```rust
impl<T: fmt::Display + ?Sized> ToString for T {
    #[inline]
    fn to_string(&self) -> String {
        <Self as SpecToString>::spec_to_string(self)
    }
}
```

なるほど `to_string` メソッドを利用したい構造体 `T` のトレイト境界に、たしかに `fmt::Display` が記載されていますね。だから `std::fmt::Display` を実装する必要があったんですね。

`to_string` メソッドは中で `spec_to_string` メソッドを呼び出していますね。

https://doc.rust-lang.org/src/alloc/string.rs.html#2824
```rust
default fn spec_to_string(&self) -> String {
        let mut buf = String::new();
        let mut formatter =
            core::fmt::Formatter::new(&mut buf, core::fmt::FormattingOptions::new());
        // Bypass format_args!() to avoid write_str with zero-length strs
        fmt::Display::fmt(self, &mut formatter)
            .expect("a Display implementation returned an error unexpectedly");
        buf
    }
```

なるほど `spec_to_string` メソッドの中で `fmt::Display::fmt` メソッドが呼ばれていますね。そんでもって `expect` が呼ばれています。

だから `to_string` の戻り値型は `Result` ではなく、失敗するとpanicになるんですね。

足を止めて見た甲斐がありました。


## 振り返り

今回は、Rustの `string.rs` まで踏み込んで見てみることができました。案外読めるもんですね自信になります。

これで明日から、もっと堂々と `println!` や `to_string` を使っていけるぞー 🙌

