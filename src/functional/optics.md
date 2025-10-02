# 関数型言語のオプティクス

オプティクスは、関数型言語で一般的なAPI設計の一種です。これは純粋関数型の概念であり、Rustではあまり使われていません。

しかし、この概念を探求することは、[ビジター](../patterns/behavioural/visitor.md)など、RustのAPIにおける他のパターンを理解するのに役立つかもしれません。また、ニッチなユースケースもあります。

これはかなり大きなトピックであり、その能力を完全に理解するには言語設計に関する実際の書籍が必要です。しかし、Rustにおける適用可能性はずっとシンプルです。

この概念の関連部分を説明するために、`Serde`-APIを例として使用します。これは、単にAPIドキュメントから理解することが多くの人にとって困難なものだからです。

その過程で、オプティクスと呼ばれる異なる特定のパターンを取り上げます。これらは、*アイソ(Iso)*、*ポリアイソ(Poly Iso)*、*プリズム(Prism)*です。

## APIの例: Serde

APIを読むだけで*Serde*の動作を理解しようとするのは、特に初めての場合は困難です。新しいデータフォーマットを解析するライブラリによって実装される`Deserializer`トレイトを考えてみましょう:

```rust,ignore
pub trait Deserializer<'de>: Sized {
    type Error: Error;

    fn deserialize_any<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    fn deserialize_bool<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    // remainder omitted
}
```

そして、ジェネリックで渡される`Visitor`トレイトの定義は次のとおりです:

```rust,ignore
pub trait Visitor<'de>: Sized {
    type Value;

    fn visit_bool<E>(self, v: bool) -> Result<Self::Value, E>
    where
        E: Error;

    fn visit_u64<E>(self, v: u64) -> Result<Self::Value, E>
    where
        E: Error;

    fn visit_str<E>(self, v: &str) -> Result<Self::Value, E>
    where
        E: Error;

    // remainder omitted
}
```

ここでは多くの型消去が行われており、複数レベルの関連型が行き来しています。

しかし、全体像は何でしょうか? なぜ`Visitor`が呼び出し側が必要とする部分をストリーミングAPIで返すだけにしないのでしょうか? なぜこれらの追加部分が必要なのでしょうか?

これを理解する一つの方法は、*オプティクス*と呼ばれる関数型言語の概念を見ることです。

これは、Rustに共通するパターン(失敗、型変換など)を促進するように設計された、動作とプロパティの合成を行う方法です。[^1]

Rust言語は、これらを直接的にサポートする機能が非常に貧弱です。しかし、それらは言語自体の設計に現れており、その概念はRustのAPIの一部を理解するのに役立ちます。その結果、これはRustが行う方法で概念を説明しようと試みます。

これは、これらのAPIが達成しようとしているもの、つまり合成可能性の特定のプロパティに光を当てるかもしれません。

## 基本的なオプティクス

### アイソ(Iso)

アイソは、2つの型間の値変換器です。これは非常にシンプルですが、概念的に重要な構成要素です。

例として、ドキュメントの索引として使用されるカスタムハッシュテーブル構造があるとします。[^2] キー(単語)には文字列を使い、値(ファイルオフセットなど)にはインデックスのリストを使います。

主要な機能は、このフォーマットをディスクにシリアライズできることです。「手早く簡単な」アプローチは、JSON形式の文字列との相互変換を実装することです。(エラーは今のところ無視されます。後で処理されます。)

関数型言語ユーザーが期待する通常の形式で書くと:

```text
case class ConcordanceSerDe {
  serialize: Concordance -> String
  deserialize: String -> Concordance
}
```

したがって、アイソは異なる型の値を変換する関数のペアです: `serialize`と`deserialize`。

直接的な実装:

```rust
use std::collections::HashMap;

struct Concordance {
    keys: HashMap<String, usize>,
    value_table: Vec<(usize, usize)>,
}

struct ConcordanceSerde {}

impl ConcordanceSerde {
    fn serialize(value: Concordance) -> String {
        todo!()
    }
    // invalid concordances are empty
    fn deserialize(value: String) -> Concordance {
        todo!()
    }
}
```

これはかなり馬鹿げているように見えるかもしれません。Rustでは、この種の動作は通常トレイトで行われます。結局のところ、標準ライブラリには`FromStr`と`ToString`があります。

しかし、それが次の主題に繋がります: ポリアイソです。

### ポリアイソ(Poly Isos)

前の例は、単に2つの固定型の値間の変換でした。次のブロックはジェネリクスでそれを拡張し、より興味深いものです。

ポリアイソは、操作を任意の型に対してジェネリックにしながら、単一の型を返すことを可能にします。

これにより、解析に近づきます。エラーケースを無視した基本的なパーサーが何をするか考えてみましょう。繰り返しますが、これはその通常の形式です:

```text
case class Serde[T] {
    deserialize(String) -> T
    serialize(T) -> String
}
```

ここに最初のジェネリック、変換される型`T`があります。

Rustでは、これは標準ライブラリの2つのトレイトのペアで実装できます: `FromStr`と`ToString`。Rustバージョンはエラーも処理します:

```rust,ignore
pub trait FromStr: Sized {
    type Err;

    fn from_str(s: &str) -> Result<Self, Self::Err>;
}

pub trait ToString {
    fn to_string(&self) -> String;
}
```

アイソと異なり、ポリアイソは複数の型の適用を許可し、それらをジェネリックに返します。これは基本的な文字列パーサーに必要なものです。

一見、これはパーサーを書くための良い選択肢のように見えます。実際に見てみましょう:

```rust,ignore
use anyhow;

use std::str::FromStr;

struct TestStruct {
    a: usize,
    b: String,
}

impl FromStr for TestStruct {
    type Err = anyhow::Error;
    fn from_str(s: &str) -> Result<TestStruct, Self::Err> {
        todo!()
    }
}

impl ToString for TestStruct {
    fn to_string(&self) -> String {
        todo!()
    }
}

fn main() {
    let a = TestStruct {
        a: 5,
        b: "hello".to_string(),
    };
    println!("Our Test Struct as JSON: {}", a.to_string());
}
```

これはかなり論理的に見えます。しかし、これには2つの問題があります。

まず、`to_string`はAPIユーザーに「これはJSONです」と示しません。すべての型がJSON表現に同意する必要があり、Rust標準ライブラリの多くの型は既にそうなっていません。これを使用するのは適していません。これは独自のトレイトで簡単に解決できます。

しかし、2つ目のより微妙な問題があります: スケーラビリティです。

すべての型が手動で`to_string`を書く場合、これは機能します。しかし、型をシリアライズ可能にしたいすべての人が大量のコード、そして場合によっては異なるJSONライブラリを書かなければならない場合、それはすぐに混乱に陥ります!

答えはSerdeの2つの主要な革新の1つです: データシリアライゼーション言語に共通する構造でRustデータを表現する独立したデータモデルです。その結果、Rustのコード生成能力を使用して、`Visitor`と呼ばれる中間変換型を作成できます。

これは、通常の形式で(再び、簡潔さのためにエラー処理をスキップして)以下を意味します:

```text
case class Serde[T] {
    deserialize: Visitor[T] -> T
    serialize: T -> Visitor[T]
}

case class Visitor[T] {
    toJson: Visitor[T] -> String
    fromJson: String -> Visitor[T]
}
```

結果は1つのポリアイソと1つのアイソです(それぞれ)。これらの両方はトレイトで実装できます:

```rust
trait Serde {
    type V;
    fn deserialize(visitor: Self::V) -> Self;
    fn serialize(self) -> Self::V;
}

trait Visitor {
    fn to_json(self) -> String;
    fn from_json(json: String) -> Self;
}
```

Rust構造を独立した形式に変換する統一されたルールセットがあるため、型`T`に関連する`Visitor`を作成するコード生成を行うことさえ可能です:

```rust,ignore
#[derive(Default, Serde)] // the "Serde" derive creates the trait impl block
struct TestStruct {
    a: usize,
    b: String,
}

// user writes this macro to generate an associated visitor type
generate_visitor!(TestStruct);
```

しかし、実際にそのアプローチを試してみましょう。

```rust,ignore
fn main() {
    let a = TestStruct { a: 5, b: "hello".to_string() };
    let a_data = a.serialize().to_json();
    println!("Our Test Struct as JSON: {a_data}");
    let b = TestStruct::deserialize(
        generated_visitor_for!(TestStruct)::from_json(a_data));
}
```

結局のところ、変換は対称的ではありませんでした! 理論上は対称的ですが、自動生成されたコードでは、`String`から完全に変換するために必要な実際の型の名前が隠されています。型名を取得するには、何らかの`generated_visitor_for!`マクロが必要になります。

不格好ですが、動作します...部屋の中の象に到達するまでは。

現在サポートされているフォーマットはJSONのみです。より多くのフォーマットをサポートするにはどうすればよいでしょうか?

現在の設計では、すべてのコード生成を完全に書き直し、新しいSerdeトレイトを作成する必要があります。これは非常にひどく、まったく拡張可能ではありません!

それを解決するには、より強力な何かが必要です。

## プリズム(Prism)

フォーマットを考慮に入れるには、次のような通常の形式の何かが必要です:

```text
case class Serde[T, F] {
    serialize: T, F -> String
    deserialize: String, F -> Result[T, Error]
}
```

この構造はプリズムと呼ばれます。これはポリアイソよりもジェネリクスで「1レベル高い」です(この場合、「交差する」型Fが鍵です)。

残念ながら、`Visitor`はトレイトであるため(各実装には独自のカスタムコードが必要です)、これにはRustがサポートしていない種類のジェネリック型境界が必要になります。

幸いなことに、以前の`Visitor`型がまだあります。`Visitor`は何をしているのでしょうか? それは、各データ構造が自身が解析される方法を定義できるようにしようとしています。

では、ジェネリックフォーマット用にもう1つのインターフェイスを追加できたらどうでしょうか? そうすれば、`Visitor`は単なる実装の詳細であり、2つのAPIを「橋渡し」することになります。

通常の形式で:

```text
case class Serde[T] {
    serialize: F -> String
    deserialize F, String -> Result[T, Error]
}

case class VisitorForT {
    build: F, String -> Result[T, Error]
    decompose: F, T -> String
}

case class SerdeFormat[T, V] {
    toString: T, V -> String
    fromString: V, String -> Result[T, Error]
}
```

そして、どうでしょう、底部にトレイトとして実装できるポリアイソのペアがあります!

したがって、Serde APIがあります:

1. シリアライズされる各型は、`Serde`クラスに相当する`Deserialize`または`Serialize`を実装します
1. これらは、`Visitor`トレイトを実装する型(実際には2つ、各方向に1つ)を取得します。これは通常(常にではありませんが)deriveマクロによって生成されたコードを通じて行われます。これには、データ型とSerdeデータモデルのフォーマット間で構築または分解するロジックが含まれています。
1. `Deserializer`トレイトを実装する型は、`Visitor`によって「駆動」されながら、フォーマットに固有のすべての詳細を処理します。

この分割とRustの型消去は、実際には間接的にプリズムを達成するためのものです。

これは`Deserializer`トレイトで確認できます:

```rust,ignore
pub trait Deserializer<'de>: Sized {
    type Error: Error;

    fn deserialize_any<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    fn deserialize_bool<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    // remainder omitted
}
```

そしてビジター:

```rust,ignore
pub trait Visitor<'de>: Sized {
    type Value;

    fn visit_bool<E>(self, v: bool) -> Result<Self::Value, E>
    where
        E: Error;

    fn visit_u64<E>(self, v: u64) -> Result<Self::Value, E>
    where
        E: Error;

    fn visit_str<E>(self, v: &str) -> Result<Self::Value, E>
    where
        E: Error;

    // remainder omitted
}
```

そして、マクロによって実装される`Deserialize`トレイト:

```rust,ignore
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```

これは抽象的だったので、具体的な例を見てみましょう。

実際のSerdeは、以前の`struct Concordance`にJSONの一部をどのようにデシリアライズしますか?

1. ユーザーはデータをデシリアライズするためにライブラリ関数を呼び出します。これによりJSON形式に基づいた`Deserializer`が作成されます。
1. 構造体のフィールドに基づいて、`Visitor`が作成されます(これについては後で詳しく説明します)。これは、それを表現するために必要なジェネリックデータモデルの各型を作成する方法を知っています: `Vec`(リスト)、`u64`、`String`。
1. デシリアライザーはアイテムを解析しながら`Visitor`への呼び出しを行います。
1. `Visitor`は、見つかったアイテムが期待されているかどうかを示し、そうでない場合はデシリアライゼーションが失敗したことを示すエラーを発生させます。

上記の非常にシンプルな構造では、期待されるパターンは次のようになります:

1. マップ(*Serde*の`HashMap`またはJSONの辞書に相当するもの)の訪問を開始します。
1. "keys"という文字列キーを訪問します。
1. マップ値の訪問を開始します。
1. 各アイテムについて、文字列キーと整数値を訪問します。
1. マップの終わりを訪問します。
6. マップをデータ構造の`keys`フィールドに格納します。
7. "value_table"という文字列キーを訪問します。
8. リスト値の訪問を開始します。
9. 各アイテムについて、整数を訪問します。
10. リストの終わりを訪問します。
11. リストを`value_table`フィールドに格納します。
12. マップの終わりを訪問します。

しかし、どの「観察」パターンが期待されるかを決定するのは何でしょうか?

関数型プログラミング言語は、カリー化を使用して型自体に基づいて各型のリフレクションを作成できます。Rustはそれをサポートしていないため、すべての単一の型は、そのフィールドとそのプロパティに基づいて独自のコードを書く必要があります。

*Serde*は、deriveマクロでこの使いやすさの課題を解決します:

```rust,ignore
use serde::Deserialize;

#[derive(Deserialize)]
struct IdRecord {
    name: String,
    customer_id: String,
}
```

そのマクロは、単に構造体に`Deserialize`と呼ばれるトレイトを実装させるimplブロックを生成します。

これは、構造体自体を作成する方法を決定する関数です。コードは構造体のフィールドに基づいて生成されます。解析ライブラリが呼び出されたとき(この例ではJSON解析ライブラリ)、それは`Deserializer`を作成し、それをパラメータとして`Type::deserialize`を呼び出します。

`deserialize`コードはその後、`Visitor`を作成します。この呼び出しは`Deserializer`によって「屈折」されます。すべてがうまくいけば、最終的にその`Visitor`は解析される型に対応する値を構築して返します。

完全な例については、[*Serde*ドキュメント](https://serde.rs/deserialize-struct.html)を参照してください。

その結果、デシリアライズされる型はAPIの「最上層」のみを実装し、ファイルフォーマットは「最下層」のみを実装する必要があります。ジェネリック型がそれらを橋渡しするため、各部分はエコシステムの残りの部分と「単に機能する」ことができます。

結論として、Rustのジェネリクスに影響を受けた型システムは、このAPI設計で示されているように、これらの概念に近づき、その力を使用することができます。しかし、そのジェネリクスのための橋を作成するために手続きマクロも必要になるかもしれません。

このトピックについてもっと学ぶことに興味がある方は、次のセクションをご確認ください。

## 参照

- [lens-rs crate](https://crates.io/crates/lens-rs) - これらの例よりもクリーンなインターフェイスを持つ、事前構築されたレンズ実装
- [Serde](https://serde.rs)自体 - 詳細を理解する必要なく、エンドユーザー(つまり構造体を定義する人)にとってこれらの概念を直感的にします
- [luminance](https://github.com/phaazon/luminance-rs) - 同様のAPI設計を使用するコンピュータグラフィックスを描画するためのクレート。異なるピクセルタイプのバッファのための完全なプリズムを作成するための手続きマクロを含み、ジェネリックのままです
- [Scalaにおけるレンズに関する記事](https://web.archive.org/web/20221128185849/https://medium.com/zyseme-technology/functional-references-lens-and-other-optics-in-scala-e5f7e2fdafe) - Scalaの専門知識がなくても非常に読みやすい
- [論文: Profunctor Optics: Modular Data Accessors](https://web.archive.org/web/20220701102832/https://arxiv.org/ftp/arxiv/papers/1703/1703.10857.pdf)
- [Musli](https://github.com/udoprog/musli) - 異なるアプローチで同様の構造を使用しようとするライブラリ。例えば、ビジターを廃止する

[^1]: [School of Haskell: A Little Lens Starter Tutorial](https://web.archive.org/web/20221128190041/https://www.schoolofhaskell.com/school/to-infinity-and-beyond/pick-of-the-week/a-little-lens-starter-tutorial)

[^2]: [Concordance on Wikipedia](https://en.wikipedia.org/wiki/Concordance_(publishing))
