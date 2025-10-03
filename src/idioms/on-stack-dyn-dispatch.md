# スタック上での動的ディスパッチ

## 説明

複数の値に対して動的にディスパッチできますが、そのためには異なる型のオブジェクトをバインドする複数の変数を宣言する必要があります。必要に応じてライフタイムを延長するために、以下に示すような遅延条件付き初期化を使用できます：

## 例

```rust
use std::io;
use std::fs;

# fn main() -> Result<(), Box<dyn std::error::Error>> {
# let arg = "-";

// We need to describe the type to get dynamic dispatch.
let readable: &mut dyn io::Read = if arg == "-" {
    &mut io::stdin()
} else {
    &mut fs::File::open(arg)?
};

// Read from `readable` here.

# Ok(())
# }
```

## 動機

Rustはデフォルトでコードを単相化します。これは、使用される各型に対してコードのコピーが生成され、独立して最適化されることを意味します。これによりホットパス上で非常に高速なコードが実現できますが、パフォーマンスが重要でない場所ではコードが肥大化し、コンパイル時間とキャッシュ使用量のコストがかかります。

幸いなことに、Rustは動的ディスパッチの使用を許可していますが、明示的に要求する必要があります。

## 利点

ヒープ上に何も割り当てる必要はありません。また、後で使用しないものを初期化する必要もなく、`File`または`Stdin`の両方で動作するように後続のコード全体を単相化する必要もありません。

## 欠点

Rust 1.79.0以前では、コードには遅延初期化を伴う2つの`let`バインディングが必要であり、`Box`ベースのバージョンよりも可動部分が多くなりました：

```rust,ignore
// We still need to ascribe the type for dynamic dispatch.
let readable: Box<dyn io::Read> = if arg == "-" {
    Box::new(io::stdin())
} else {
    Box::new(fs::File::open(arg)?)
};
// Read from `readable` here.
```

幸いなことに、この欠点は現在解消されています。やった！

## 議論

Rust 1.79.0以降、コンパイラは関数のスコープ内で可能な限り`&`または`&mut`内の一時的な値のライフタイムを自動的に延長します。

これは、遅延初期化に必要だった`let`バインディングに内容を配置することを心配せずに、ここで単純に`&mut`値を使用できることを意味します（これはその変更前に使用されていた解決策でした）。

各値に対する場所（その場所が一時的であっても）はまだあり、コンパイラは各値のサイズを知っており、各借用された値はそこから借用されたすべての参照よりも長生きします。

## 参照

- [デストラクタでのファイナライゼーション](dtor-finally.md)と[RAIIガード](../patterns/behavioural/RAII.md)は、ライフタイムの厳密な制御から恩恵を受けることができます。
- 条件付きで埋められた（可変）参照の`Option<&T>`については、`Option<T>`を直接初期化し、その[`.as_ref()`]メソッドを使用してオプショナル参照を取得できます。

[`.as_ref()`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.as_ref
