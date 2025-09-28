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

今回もPerson構造体を用意しました。こちらを自分で実装したSerializerでJSON形式へシリアライズされることをゴールとしてやってみます。

```.rust
struct Person {
    nickname: String,
    age: u8,
}

impl serde::ser::Serialize for Person {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        todo!()
    }
}

fn main() {
    let person = Person {
        nickname: String::from("タロー"),
        age: 30,
    };

    let serialized = serde_json::to_string(&person).unwrap();
    assert_eq!(serialized, "{\"nickname\":\"タロー\",\"age\":30}");
}
```

言うまでもなくPersonは構造体（Struct）です。なので [Serialize A Struct](https://serde.rs/impl-serialize.html#serializing-a-struct) を参照します。

参照のコード例を確認しつつ書いてみると、自ずと以下のようになりました。

```.rust
impl serde::ser::Serialize for Person {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        let mut state = serializer.serialize_struct("Person", 2)?;
        state.serialize_field("nickname", &self.nickname)?;
        state.serialize_field("age", &self.age)?;
        state.end()
    }
}
```

シリアライズ定義のプロセスとして、まず構造体自身の定義、続いてフィールドの定義、最後に終了処理の3ステップがあるようです。それぞれ詳しく追いかけてみます。

### serialize_struct の定義を確認

今回は JSON 形式へシリアライズしていくため、serde_json が実装している Serializer の `serialize_struct` が呼ばれる形になります（個人的にココが地味に迷いポイントでした）

```.rust
fn serialize_struct(self, name: &'static str, len: usize) -> Result<Self::SerializeStruct> {
    match name {
        #[cfg(feature = "arbitrary_precision")]
        crate::number::TOKEN => Ok(Compound::Number { ser: self }),
        #[cfg(feature = "raw_value")]
        crate::raw::TOKEN => Ok(Compound::RawValue { ser: self }),
        _ => self.serialize_map(Some(len)),
    }
}
```

`name` には構造体名が渡され、`len` にはフィールド数が渡されます。

`match name {}` していますが、今回は最後の `_` にマッチする形になります。`_` はワイルドカードパターンと呼ばれるもので、matchの中の手前のパターンにマッチしなかったもの全てマッチする動作になります。

つまり `serialize_map` メソッドが内部的に呼ばれています。次はそちらを見てみます。

```.rust
fn serialize_map(self, len: Option<usize>) -> Result<Self::SerializeMap> {
    tri!(self
        .formatter
        .begin_object(&mut self.writer)
        .map_err(Error::io));
    if len == Some(0) {
        tri!(self
            .formatter
            .end_object(&mut self.writer)
            .map_err(Error::io));
        Ok(Compound::Map {
            ser: self,
            state: State::Empty,
        })
    } else {
        Ok(Compound::Map {
            ser: self,
            state: State::First,
        })
    }
}
```

最初に `begin_object` を呼びだし、もしフィールド数が0なら `end_object` を呼び出し、戻り値として `Compound::Map` 構造体を返すようです。一旦 `begin_object` と `end_object` を確認してみます。

```.rust
/// Called before every object.  Writes a `{` to the specified
/// writer.
#[inline]
fn begin_object<W>(&mut self, writer: &mut W) -> io::Result<()>
where
    W: ?Sized + io::Write,
{
    writer.write_all(b"{")
}

/// Called after every object.  Writes a `}` to the specified
/// writer.
#[inline]
fn end_object<W>(&mut self, writer: &mut W) -> io::Result<()>
where
    W: ?Sized + io::Write,
{
    writer.write_all(b"}")
}
```



### serialize_field の定義を確認



### end の定義を確認



## もう一段だけ深ぼってみる

TODO

## 振り返り

今回も `serde` クレートを改めて足を止めて見てみました。

これで明日から、もっと堂々と `serde` を使っていけるぞー 🙌

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_6_serde_crate

