---
title: "足を止めて見る #12 〜 Rustのtracingクレート 〜"
emoji: "🚶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
# published_at: 2026-04-13 17:00
publication_name: doctormate
---

# 足を止めて見よう

足を止めて見ようシリーズの12つ目です。

[前回](https://zenn.dev/doctormate/articles/dive_11_anyhow_crate)の記事では `anyhow` クレートが提供しているメソッドやマクロを確認しました。

今回は anyhow クレートを離れて tracing クレートに移っていこうと思います。

もし Rust でアプリケーション書いていると `#[tracing::instrument]` といった記述を関数の宣言部分に付けていたりしませんか？

これがあると、その関数名がログに書き込まれているのは、きっと観測していると思います。

では `#[tracing::instrument]` と記述する以上のこと出来ていますか？

自分は勿論できていません。なので足を止めて見ようと思います。

## tracing クレートとは

とにかくまずは基本のキ。[docs.rs](https://docs.rs/) で `tracing` クレートを見に行きます。

https://docs.rs/tracing/latest/tracing/

超ざっくりで以下のようなことが書いてあります。

コア概念が3つ

- Event
    - 瞬間（点）を表すもの、単発の出来事を表す概念
    - ex. 「user_id=42のlogin処理が成功した」
- Span
    - 継続（文脈）を表すもの、意味のある単位のまとまりを表す概念
    - ex. 関数Aというまとまり、DBアクセスというまとまり
- Subscriber
    - Event や Span を受け取り、それらをどう扱うか決めるもの
    - ex. ファイルに書き出す、外部に送る

つまり tracing は `文脈（Span）の中で発生した出来事（Event）を、Subscriber に渡す仕組み` と捉えることができそうです。

ちょっと、読んだだけではイマイチわからないので、簡単にでもハンズオンして見ようと思います。

## tracing クレートをインストール

まずはインストールですね。

```console
$ cargo add tracing tracing-subscriber
```

`Cargo.toml` に以下の依存関係が追加されました。

```toml
[dependencies]
tracing = "0.1.44"
tracing-subscriber = "0.3.23"
```

tracing自体では Subscriber は trait しか存在せず実装は提供していないので、自分で実装もしくは実装されたライブラリの利用が必要そうです。

今回は、Subscriberの代表的な実装ライブラリ `tracing-subscriber` クレートもあわせて利用してみます。

※ tracing も tracing-subscriber も、Tokio プロジェクトによって開発・メンテナンスされています。

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
    sub();
    event!(Level::INFO, "back in main span");
}

fn sub() {
    let span = span!(Level::INFO, "sub");
    let _enter = span.enter();
    event!(Level::WARN, "inside sub function");
    event!(Level::ERROR, "inside sub span");
}

```

実行するとこうなりました。

```.sh
2026-04-13T12:27:46.616040Z  INFO dive_12_tracing_crate: hello, tracing!
2026-04-13T12:27:46.616065Z  INFO main: dive_12_tracing_crate: inside span
2026-04-13T12:27:46.616082Z  WARN main:sub: dive_12_tracing_crate: inside sub function
2026-04-13T12:27:46.616092Z ERROR main:sub: dive_12_tracing_crate: inside sub span
2026-04-13T12:27:46.616103Z  INFO main: dive_12_tracing_crate: back in main span
```

なるほど、つまり以下のような構造になるみたいです。

```
{タイムスタンプ} {Level} {span(1)名}:{span(2)名}:...:{span(n)名}: {クレート名}: {イベント}
```

span を enter すると、そのスコープでの event は、span に関連付けされるようですね。

またさらに span を作って enter すると、span の中に span がネストされて作られ、内側の span に入るほど出力の右側に追加されていくようです。

なるほど、こうやって span をたとえば関数ごとに区切るなどして文脈を与えつつ、eventを記録していくという使い方ができるわけですね。

## もう一段だけ深ぼってみる

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
    sub();
    event!(Level::INFO, "back in main span");
}

#[instrument]
fn sub() {
    // let span = span!(Level::INFO, "sub");
    // let _enter = span.enter();
    event!(Level::WARN, "inside sub function");
    event!(Level::ERROR, "inside sub span");
}
```

`sub()` 関数のところに `#[instrument]` をつけて、spanの作成とenter()をコメントアウトしました。

実行するとこうなります。

```.sh
2026-04-13T12:30:18.547126Z  INFO dive_12_tracing_crate: hello, tracing!
2026-04-13T12:30:18.547153Z  INFO main: dive_12_tracing_crate: inside span
2026-04-13T12:30:18.547170Z  WARN main:sub: dive_12_tracing_crate: inside sub function
2026-04-13T12:30:18.547180Z ERROR main:sub: dive_12_tracing_crate: inside sub span
2026-04-13T12:30:18.547191Z  INFO main: dive_12_tracing_crate: back in main span
```

見比べてみてください、タイムスタンプを除いて完全一致です。

つまり `#[instrument]` とは、その関数名で span を作って enter まで実行してくれるとうことですね。

## 振り返り

これで明日から、もっと堂々と `tracing` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_12_tracing_crate
