# 拡張性のための `#[non_exhaustive]` とプライベートフィールド

## 説明

ライブラリの作者が、後方互換性を壊すことなく、公開構造体に公開フィールドを追加したり、列挙型に新しいバリアントを追加したりしたい場合があります。

Rustはこの問題に対して2つの解決策を提供しています：

- `struct`、`enum`、および `enum` のバリアントに `#[non_exhaustive]` を使用する。`#[non_exhaustive]` が使用できるすべての場所に関する詳細なドキュメントについては、[the docs](https://doc.rust-lang.org/reference/attributes/type_system.html#the-non_exhaustive-attribute)を参照してください。

- 構造体にプライベートフィールドを追加して、直接インスタンス化されたりマッチされたりするのを防ぐことができます（代替案を参照）

## 例

```rust
mod a {
    // Public struct.
    #[non_exhaustive]
    pub struct S {
        pub foo: i32,
    }

    #[non_exhaustive]
    pub enum AdmitMoreVariants {
        VariantA,
        VariantB,
        #[non_exhaustive]
        VariantC {
            a: String,
        },
    }
}

fn print_matched_variants(s: a::S) {
    // Because S is `#[non_exhaustive]`, it cannot be named here and
    // we must use `..` in the pattern.
    let a::S { foo: _, .. } = s;

    let some_enum = a::AdmitMoreVariants::VariantA;
    match some_enum {
        a::AdmitMoreVariants::VariantA => println!("it's an A"),
        a::AdmitMoreVariants::VariantB => println!("it's a b"),

        // .. required because this variant is non-exhaustive as well
        a::AdmitMoreVariants::VariantC { a, .. } => println!("it's a c"),

        // The wildcard match is required because more variants may be
        // added in the future
        _ => println!("it's a new variant"),
    }
}
```

## 代替案：構造体の `プライベートフィールド`

`#[non_exhaustive]` はクレートの境界を越えてのみ機能します。クレート内では、プライベートフィールドの方法を使用できます。

構造体にフィールドを追加することは、ほぼ後方互換性のある変更です。しかし、クライアントがパターンを使用して構造体インスタンスを分解する場合、構造体のすべてのフィールドに名前を付ける可能性があり、新しいフィールドを追加するとそのパターンが壊れます。クライアントがいくつかのフィールドに名前を付け、パターンで `..` を使用する場合、別のフィールドを追加することは後方互換性があります。構造体のフィールドの少なくとも1つをプライベートにすることで、クライアントは後者の形式のパターンを使用せざるを得なくなり、構造体が将来にわたって安全であることが保証されます。

このアプローチの欠点は、他の方法では不要なフィールドを構造体に追加する必要がある場合があることです。`()` 型を使用すると実行時のオーバーヘッドがなく、フィールド名の前に `_` を付けることで未使用フィールドの警告を回避できます。

```rust
pub struct S {
    pub a: i32,
    // Because `b` is private, you cannot match on `S` without using `..` and `S`
    //  cannot be directly instantiated or matched against
    _b: (),
}
```

## 議論

`struct` において、`#[non_exhaustive]` は後方互換性のある方法で追加のフィールドを追加することを可能にします。また、すべてのフィールドが公開されている場合でも、クライアントが構造体コンストラクタを使用することを防ぎます。これは役立つかもしれませんが、追加のフィールドをコンパイラエラーとしてクライアントに見つけてもらうか、それとも静かに見過ごされる可能性があるものとして扱うか、検討する価値があります。

`#[non_exhaustive]` は列挙型のバリアントにも適用できます。`#[non_exhaustive]` バリアントは `#[non_exhaustive]` 構造体と同じように動作します。

これを意図的かつ慎重に使用してください：フィールドやバリアントを追加する際にメジャーバージョンをインクリメントすることが、多くの場合より良い選択肢です。`#[non_exhaustive]` は、ライブラリと同期せずに変更される可能性のある外部リソースをモデル化するシナリオでは適切かもしれませんが、汎用的なツールではありません。

### デメリット

`#[non_exhaustive]` は、特に未知の列挙型バリアントを処理することを強制される場合、コードの人間工学を大幅に低下させる可能性があります。これは、メジャーバージョンをインクリメント**せずに**このような進化が必要な場合にのみ使用すべきです。

`#[non_exhaustive]` が `enum` に適用されると、クライアントはワイルドカードバリアントを処理することを強制されます。この場合に適切なアクションがない場合、これは不自然なコードと、極めて稀な状況でのみ実行されるコードパスにつながる可能性があります。クライアントがこのシナリオで `panic!()` することを決定した場合、このエラーをコンパイル時に公開した方が良かったかもしれません。実際、`#[non_exhaustive]` はクライアントに「その他」のケースを処理することを強制します；このシナリオでは適切なアクションはほとんどありません。

## 参照

- [RFC introducing #[non_exhaustive] attribute for enums and structs](https://github.com/rust-lang/rfcs/blob/master/text/2008-non-exhaustive.md)
