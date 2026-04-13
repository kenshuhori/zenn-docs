---
title: "足を止めて見る #12 〜 Rustのtracingクレート 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
published_at: 2026-04-14 12:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの12回目です。

[前回](https://zenn.dev/doctormate/articles/dive_11_anyhow_crate)の記事では `anyhow` クレートが提供しているメソッドやマクロを確認しました。

今回は `anyhow` クレートを離れて、`tracing` クレートを見ていこうと思います。

Rust でアプリケーションを書いていると、関数の宣言に `#[tracing::instrument]` が付いているコードを見かけることはないでしょうか。

これがあると、関数名がログに出力される、という挙動は目にしたことがあると思います。

では、その `#[tracing::instrument]` の裏側で何が起きているか、理解できているでしょうか？

自分は勿論できていません。なので、今回も足を止めて見ていこうと思います。

## tracing クレートとは

とにかくまずは基本のキ。[docs.rs](https://docs.rs/) で `tracing` クレートを見に行きます。

https://docs.rs/tracing/latest/tracing/

かなりざっくりまとめると、次のようなことが書かれています。

コア概念が3つ

- Event
    - 瞬間（点）を表すもの。単発の出来事を表す
    - 例: 「user_id=42 の login 処理が成功した」
- Span
    - 継続（文脈）を表すもの。意味のある処理のまとまりを表す
    - 例: 関数Aの処理、DBアクセスの処理
- Subscriber
    - Event や Span を受け取り、それらをどう扱うかを決めるもの
    - 例: ログとして出力する、外部に送信する

つまり `tracing` は、**文脈（Span）の中で発生した出来事（Event）を、Subscriber に渡す仕組み**と捉えることができそうです。

ただ、これだけではまだピンとこないので、簡単なコードを書いて動かしてみます。

## tracing クレートをインストール

まずはインストールです。

```console
$ cargo add tracing tracing-subscriber
```

`Cargo.toml` に以下の依存関係が追加されました。

```toml
[dependencies]
tracing = "0.1.44"
tracing-subscriber = "0.3.23"
```

`tracing` 自体には Subscriber の trait しか用意されておらず、実装は提供されていません。  
そのため、自分で実装するか、既存の実装を利用する必要があります。

今回は、代表的な実装である `tracing-subscriber` クレートを利用します。

※ `tracing` と `tracing-subscriber` はどちらも Tokio プロジェクトによって開発・メンテナンスされています。

## tracing クレートを使ってみる

まずは最低限のコードを書いて、Event と Span が標準出力にどう出力されるかを見てみます。

```rust
use tracing::{Level, event, span};
use tracing_subscriber;

fn main() {
    let subscriber = tracing_subscriber::fmt::Subscriber::new();
    tracing::subscriber::set_global_default(subscriber)
        .expect("failed to set global default subscriber");

    event!(Level::INFO, "hello, tracing!");
    let span = span!(Level::INFO, "main");
    let _enter = span.enter();
    event!(Level::INFO, "inside span");
    sub(42);
    event!(Level::INFO, "back in main span");
}

fn sub(user_id: u64) {
    let span = span!(Level::INFO, "sub", user_id = user_id);
    let _enter = span.enter();
    event!(Level::WARN, "inside sub function");
    event!(Level::ERROR, "inside sub span");
}
```

実行するとこうなりました。

```.sh
2026-04-13T12:45:15.499469Z  INFO dive_12_tracing_crate: hello, tracing!
2026-04-13T12:45:15.499500Z  INFO main: dive_12_tracing_crate: inside span
2026-04-13T12:45:15.499517Z  WARN main:sub{user_id=42}: dive_12_tracing_crate: inside sub function
2026-04-13T12:45:15.499529Z ERROR main:sub{user_id=42}: dive_12_tracing_crate: inside sub span
2026-04-13T12:45:15.499538Z  INFO main: dive_12_tracing_crate: back in main span
```

なるほど、つまり以下のような構造になるみたいです。

```
{タイムスタンプ} {Level} {span名{field...}}:{span名{field...}}: {クレート名}: {イベント}
```

span を enter すると、そのスコープでの event は、span に関連付けされるようですね。

またさらに span を作って enter すると、span の中に span がネストされて作られ、内側の span に入るほど出力の右側に追加されていくようです。

span には名前だけでなく field も持たせることができ、今回の例では `sub{user_id=42}` のように出力されていました。

なるほど、こうやって span をたとえば関数ごとに区切るなどして文脈を与えつつ、eventを記録していくという使い方ができるわけですね。

## もう一段だけ深掘ってみる

eventマクロやspanマクロを利用して見てきましたが、実際によく使われているのは `#[instrument]` かと思います。

試しに `#[instrument]` を利用して、どのような挙動になるか比較してみます。

```rust
use tracing::{Level, event, instrument, span};
use tracing_subscriber;

fn main() {
    let subscriber = tracing_subscriber::fmt::Subscriber::new();
    tracing::subscriber::set_global_default(subscriber)
        .expect("failed to set global default subscriber");

    event!(Level::INFO, "hello, tracing!");
    let span = span!(Level::INFO, "main");
    let _enter = span.enter();
    event!(Level::INFO, "inside span");
    sub(42);
    event!(Level::INFO, "back in main span");
}

#[instrument]
fn sub(user_id: u64) {
    // let span = span!(Level::INFO, "sub", user_id = user_id);
    // let _enter = span.enter();
    event!(Level::WARN, "inside sub function");
    event!(Level::ERROR, "inside sub span");
}
```

`sub()` 関数のところに `#[instrument]` をつけて、spanの作成とenter()をコメントアウトしました。

実行するとこうなります。

```.sh
2026-04-13T12:46:09.636203Z  INFO dive_12_tracing_crate: hello, tracing!
2026-04-13T12:46:09.636233Z  INFO main: dive_12_tracing_crate: inside span
2026-04-13T12:46:09.636251Z  WARN main:sub{user_id=42}: dive_12_tracing_crate: inside sub function
2026-04-13T12:46:09.636262Z ERROR main:sub{user_id=42}: dive_12_tracing_crate: inside sub span
2026-04-13T12:46:09.636274Z  INFO main: dive_12_tracing_crate: back in main span
```

見比べてみてください、タイムスタンプを除いて完全一致です。

つまり手動で書いていた

- span の生成
- enter()

といった処理を、`#[instrument]` が自動で行ってくれていることがわかります。

さらに、呼び出し元の span（今回でいう main）を親として、その下にネストされる形で span が作られていることも確認できるし、

引数の `user_id` も勝手に span の field に組み込まれているのも確認できます。

## 振り返り

`tracing` を足を止めて、基礎から確認していきました。

`tracing` って、なんとなく `ログ` を出力するためのライブラリと思っていましたが、

`ログ` を出力する役割は `subscriber` であり、`tracing` は構造化するための仕組みを提供しているのだと理解できました。

これで明日から、もっと堂々と `tracing` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_12_tracing_crate
