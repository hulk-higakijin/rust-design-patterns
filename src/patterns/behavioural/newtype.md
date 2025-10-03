# Newtype

ある型を別の型と似たように振る舞わせたい、または型エイリアスだけでは不十分な場合にコンパイル時に特定の振る舞いを強制したい場合、どうすればよいでしょうか？

たとえば、セキュリティ上の理由（パスワードなど）で `String` にカスタムの `Display` 実装を作成したい場合などです。

このような場合、`Newtype` パターンを使用して **型安全性** と **カプセル化** を提供できます。

## 説明

単一のフィールドを持つタプル構造体を使用して、型の不透明なラッパーを作成します。
これにより、型のエイリアス（`type` 項目）ではなく、新しい型が作成されます。

## 例

```rust
use std::fmt::Display;

// Create Newtype Password to override the Display trait for String
struct Password(String);

impl Display for Password {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "****************")
    }
}

fn main() {
    let unsecured_password: String = "ThisIsMyPassword".to_string();
    let secured_password: Password = Password(unsecured_password.clone());
    println!("unsecured_password: {unsecured_password}");
    println!("secured_password: {secured_password}");
}
```

```shell
unsecured_password: ThisIsMyPassword
secured_password: ****************
```

## 動機

newtype の主な動機は抽象化です。これにより、型間で実装の詳細を共有しながら、インターフェースを正確に制御できます。
API の一部として実装型を公開するのではなく、newtype を使用することで、後方互換性を保ちながら実装を変更できます。

Newtype は単位を区別するために使用できます。たとえば、`f64` をラップして区別可能な `Miles` と `Kilometres` を提供します。

## 利点

ラップされた型とラッパー型は型互換性がありません（`type` を使用する場合とは異なります）。そのため、newtype のユーザーがラップされた型とラッパー型を「混同」することはありません。

Newtype はゼロコスト抽象化です - ランタイムオーバーヘッドはありません。

プライバシーシステムにより、ユーザーはラップされた型にアクセスできません（フィールドがプライベートの場合、デフォルトではプライベートです）。

## 欠点

newtype の欠点（特に型エイリアスと比較して）は、特別な言語サポートがないことです。つまり、*大量の* ボイラープレートが発生する可能性があります。
ラップされた型で公開したいすべてのメソッドに対して「パススルー」メソッドが必要であり、ラッパー型にも実装したいすべてのトレイトに対して impl が必要です。

## 議論

Newtype は Rust コードで非常に一般的です。抽象化や単位の表現が最も一般的な用途ですが、他の理由でも使用できます：

- 機能の制限（公開される関数や実装されるトレイトを減らす）
- コピーセマンティクスを持つ型をムーブセマンティクスにする
- より具体的な型を提供することによる抽象化、つまり内部型を隠す、例：

```rust,ignore
pub struct Foo(Bar<T1, T2>);
```

ここで、`Bar` は何らかの公開されたジェネリック型であり、`T1` と `T2` は内部型です。モジュールのユーザーは、`Foo` を `Bar` を使用して実装していることを知る必要はありませんが、ここで実際に隠しているのは型 `T1` と `T2`、およびそれらが `Bar` でどのように使用されているかです。

## 参照

- [Advanced Types in the book](https://doc.rust-lang.org/book/ch19-04-advanced-types.html?highlight=newtype#using-the-newtype-pattern-for-type-safety-and-abstraction)
- [Newtypes in Haskell](https://wiki.haskell.org/Newtype)
- [Type aliases](https://doc.rust-lang.org/stable/book/ch19-04-advanced-types.html#creating-type-synonyms-with-type-aliases)
- [derive_more](https://crates.io/crates/derive_more), a crate for deriving many
  builtin traits on newtypes.
- [The Newtype Pattern In Rust](https://web.archive.org/web/20230519162111/https://www.worthe-it.co.za/blog/2020-10-31-newtype-pattern-in-rust.html)
