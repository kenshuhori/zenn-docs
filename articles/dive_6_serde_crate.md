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

```rust
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

```rust
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

```rust
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

```rust
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

```rust
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

JSONっぽい表現が現れています。`begin_object` では `{` を書き込み、`end_object` では `}` を書き込んでいます。たしかにフィールド数が0なら `{}` と書き込まれそうです。

一旦最初に戻ります。つまり `serialize_struct("Person", 2)` では `{` が書き込まれるわけだ。

```rust
// { が書き込まれる
let mut state = serializer.serialize_struct("Person", 2)?;
// 未確認
state.serialize_field("nickname", &self.nickname)?;
state.serialize_field("age", &self.age)?;
// 未確認
state.end()
```

### serialize_field の定義を確認

続いて `serialize_field` を確認します。

`serialize_field` は `SerializeStruct` トレイトに用意されているメソッドです。

serde_jsonでは、先ほどの `serialize_struct` メソッドの最後に登場した `Compound` Enum が `SerializeStruct` トレイトを実装しており、そこの `serialize_field` メソッドを確認することになります。

```rust
fn serialize_field<T>(&mut self, key: &'static str, value: &T) -> Result<()>
where
    T: ?Sized + Serialize,
{
    match self {
        Compound::Map { .. } => ser::SerializeMap::serialize_entry(self, key, value),
        #[cfg(feature = "arbitrary_precision")]
        Compound::Number { ser, .. } => {
            if key == crate::number::TOKEN {
                value.serialize(NumberStrEmitter(ser))
            } else {
                Err(invalid_number())
            }
        }
        #[cfg(feature = "raw_value")]
        Compound::RawValue { ser, .. } => {
            if key == crate::raw::TOKEN {
                value.serialize(RawValueStrEmitter(ser))
            } else {
                Err(invalid_raw_value())
            }
        }
    }
}
```

self は `Compound::Map` となるため、`ser::SerializeMap::serialize_entry` が呼ばれることが分かります。

```rust
fn serialize_entry<K, V>(&mut self, key: &K, value: &V) -> Result<(), Self::Error>
where
    K: ?Sized + Serialize,
    V: ?Sized + Serialize,
{
    tri!(self.serialize_key(key));
    self.serialize_value(value)
}
```

`serialize_entry` ではまず `serialize_key` が呼ばれています。

```rust
fn serialize_key<T>(&mut self, key: &T) -> Result<()>
where
    T: ?Sized + Serialize,
{
    match self {
        Compound::Map { ser, state } => {
            tri!(ser
                .formatter
                .begin_object_key(&mut ser.writer, *state == State::First)
                .map_err(Error::io));
            *state = State::Rest;

            tri!(key.serialize(MapKeySerializer { ser: *ser }));

            ser.formatter
                .end_object_key(&mut ser.writer)
                .map_err(Error::io)
        }
        #[cfg(feature = "arbitrary_precision")]
        Compound::Number { .. } => unreachable!(),
        #[cfg(feature = "raw_value")]
        Compound::RawValue { .. } => unreachable!(),
    }
}
```

なんとなく `serialize_map` メソッドと構成が似ているように見えます。一旦 `begin_object_key` と `end_object_key` を確認してみます。

```rust
/// Called before every object key.
#[inline]
fn begin_object_key<W>(&mut self, writer: &mut W, first: bool) -> io::Result<()>
where
    W: ?Sized + io::Write,
{
    if first {
        Ok(())
    } else {
        writer.write_all(b",")
    }
}

/// Called after every object key.  A `:` should be written to the
/// specified writer by either this method or
/// `begin_object_value`.
#[inline]
fn end_object_key<W>(&mut self, _writer: &mut W) -> io::Result<()>
where
    W: ?Sized + io::Write,
{
    Ok(())
}
```

JSONのカンマを書き込むかどうかを判断しているようですね。1つ目のフィールドであればカンマは不要なので何もせず、2つ目以降のフィールドでは `,` がまず書き込まれるようです。 

例: `{ key1: value1, key2: value2 }`

`end_object_key` は何もしないようです。

残るは `key.serialize(MapKeySerializer { ser: *ser })` です。

`key` は `"nickname"`とか `"age"` などの `str` です。

`serde::ser::Serializer` はRustのプリミティブ型のすべてに実装を用意しており `&str` も例外ではないです。

これ以上は諦めますが、最終的には `serialize_str` が呼び出され、その中では `\"nickname\"` や `\"age\"` のように、ダブルクォーテーションをエスケープしつつ書き込んでいる様子が見えました。

```rust
#[inline]
fn serialize_str(self, value: &str) -> Result<()> {
    format_escaped_str(&mut self.writer, &mut self.formatter, value).map_err(Error::io)
}
```


さて、`serialize_entry` の定義に戻ります。

```rust
fn serialize_entry<K, V>(&mut self, key: &K, value: &V) -> Result<(), Self::Error>
where
    K: ?Sized + Serialize,
    V: ?Sized + Serialize,
{
    tri!(self.serialize_key(key));
    self.serialize_value(value)
}
```

次は `serialize_value` です。

```rust
fn serialize_value<T>(&mut self, value: &T) -> Result<()>
where
    T: ?Sized + Serialize,
{
    match self {
        Compound::Map { ser, .. } => {
            tri!(ser
                .formatter
                .begin_object_value(&mut ser.writer)
                .map_err(Error::io));
            tri!(value.serialize(&mut **ser));
            ser.formatter
                .end_object_value(&mut ser.writer)
                .map_err(Error::io)
        }
        #[cfg(feature = "arbitrary_precision")]
        Compound::Number { .. } => unreachable!(),
        #[cfg(feature = "raw_value")]
        Compound::RawValue { .. } => unreachable!(),
    }
}
```

構成は `serialize_key` とほぼ同じように見えます。一旦 `begin_object_value` と `end_object_value` を確認してみます。

```rust
/// Called before every object value.  A `:` should be written to
/// the specified writer by either this method or
/// `end_object_key`.
#[inline]
fn begin_object_value<W>(&mut self, writer: &mut W) -> io::Result<()>
where
    W: ?Sized + io::Write,
{
    writer.write_all(b":")
}

/// Called after every object value.
#[inline]
fn end_object_value<W>(&mut self, _writer: &mut W) -> io::Result<()>
where
    W: ?Sized + io::Write,
{
    Ok(())
}
```

value側を書き込む際はまず `:` を書き込むようですね。`end_object_value` は何もしていません。

例: `{ key1: value1, key2: value2 }`

その後 `value.serialize(&mut **ser)` が呼び出されますが、こちらは key の時と同じで、`value` がどの型なのかによって変わります。`value` が構造体の場合、この記事が `Person` 構造体を例に見てきた流れがまた始まるわけです😅

一旦最初に戻ります。つまり `serialize_field` では `{}` の中身が書き込まれるわけだ。

```rust
// { が書き込まれる
let mut state = serializer.serialize_struct("Person", 2)?;
// \"nickname\": \"タロー\" が書き込まれる
state.serialize_field("nickname", &self.nickname)?;
// ,\"age\": 30 が書き込まれる
state.serialize_field("age", &self.age)?;
// 未確認
state.end()
```

### end の定義を確認

最後に `end` を確認します。

```rust
fn end(self) -> Result<()> {
    match self {
        Compound::Map { ser, state } => match state {
            State::Empty => Ok(()),
            _ => ser.formatter.end_object(&mut ser.writer).map_err(Error::io),
        },
        #[cfg(feature = "arbitrary_precision")]
        Compound::Number { .. } => unreachable!(),
        #[cfg(feature = "raw_value")]
        Compound::RawValue { .. } => unreachable!(),
    }
}
```

self は `Compound::Map` でした。

フィールド数が0のときは `State::Empty` となるため Okが返るだけのようです。

フィールド数が1以上のときは `end_object` が呼び出されるようです。こちらは既に登場していました。`}`を書き込んで終わります。

```rust
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

最初に戻ります。つまり `end` では `}` が書き込まれるわけだ。

```rust
// { が書き込まれる
let mut state = serializer.serialize_struct("Person", 2)?;
// \"nickname\":\"タロー\" が書き込まれる
state.serialize_field("nickname", &self.nickname)?;
// ,\"age\":30 が書き込まれる
state.serialize_field("age", &self.age)?;
// } が書き込まれる
state.end()
```

つまり結果として `{\"nickname\":\"タロー\",\"age\":30}` というJSON形式にシリアライズされました。（長かった...）

## もう一段だけ深ぼってみる

serialize_structメソッドの戻り値型は `SerializeStruct` 型のはずですが、`serialize_struct`の定義上は `SerializeMap` 型を返すと記載があります。

さらには、serialize_structの内部で呼び出されている `serialize_map` メソッドでは `Compound::Map` が返されています。

これは一体どういうことでしょうか？

パッと思いあたるのは `std::convert::From` が実装されているのでは？でしょう。

しかし、実際にfromが定義されている箇所は見当たりません。ではどうやって...?

serde_json では、`serde::ser::Serializer` トレイトを serde_json自身が内部的に定義している `Serializer` 構造体に実装させている記載があります。

```rust
impl<'a, W, F> ser::Serializer for &'a mut Serializer<W, F>
where
    W: io::Write,
    F: Formatter,
{
    type Ok = ();
    type Error = Error;

    type SerializeSeq = Compound<'a, W, F>;
    type SerializeTuple = Compound<'a, W, F>;
    type SerializeTupleStruct = Compound<'a, W, F>;
    type SerializeTupleVariant = Compound<'a, W, F>;
    type SerializeMap = Compound<'a, W, F>;
    type SerializeStruct = Compound<'a, W, F>;
    type SerializeStructVariant = Compound<'a, W, F>;
```

こちらを確認すると、`SerializeMap` も `SerializeStruct` も `Compound<'a, W, F>` で同じものであると定義されていました。

なるほど、これによって `Compound::Map`型の値を返すと、戻り値型としては `SerializeMap` とも `SerializeStruct` とも書けてしまえたわけですね。

## 振り返り

今回も `serde` クレートを改めて足を止めて見てみました。

`serde::ser::Serizalize` の定義を追いかけるだけで大分疲れてしまいました。

次は `serde::de::Deserialize` の方を追いかけてみようと思います。なんかこっちの方が難しそうです。

## その他

今回書いたRustのコードはこのリポジトリで制作しています。

https://github.com/kenshuhori/rust/tree/main/workspace/dive_6_serde_crate

