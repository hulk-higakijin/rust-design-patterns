# スタック上での動的ディスパッチ

## 説明

複数の値に対して動的ディスパッチを行うことができますが、そのためには、異なる型のオブジェクトをバインドするために複数の変数を宣言する必要があります。必要に応じてライフタイムを延長するために、以下に示すように遅延条件付き初期化を使用できます：

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

Rustはデフォルトでコードを単相化します。これは、使用される型ごとにコードのコピーが生成され、独立して最適化されることを意味します。これによりホットパス上で非常に高速なコードが実現できますが、パフォーマンスが重要でない箇所ではコードが肥大化し、コンパイル時間とキャッシュ使用量のコストがかかります。

幸いにも、Rustでは動的ディスパッチを使用できますが、明示的に要求する必要があります。

## 利点

ヒープ上に何も割り当てる必要がありません。また、後で使用しないものを初期化する必要もなく、`File`または`Stdin`の両方で動作するように後続のコード全体を単相化する必要もありません。

## 欠点

Rust 1.79.0以前では、コードは遅延初期化を伴う2つの`let`バインディングが必要で、`Box`ベースのバージョンよりも多くの可動部分がありました：

```rust,ignore
// We still need to ascribe the type for dynamic dispatch.
let readable: Box<dyn io::Read> = if arg == "-" {
    Box::new(io::stdin())
} else {
    Box::new(fs::File::open(arg)?)
};
// Read from `readable` here.
```

幸いにも、この欠点は今では解消されました。やった！

## 議論

Rust 1.79.0以降、コンパイラは`&`または`&mut`内の一時的な値のライフタイムを、関数のスコープ内で可能な限り自動的に延長します。

これは、遅延初期化のために必要だった`let`バインディングにコンテンツを配置することを心配せずに、単に`&mut`値をここで使用できることを意味します（これはその変更前に使用されていた解決策でした）。

各値には依然として場所があり（たとえその場所が一時的であっても）、コンパイラは各値のサイズを把握しており、各借用値はそれから借用されたすべての参照よりも長生きします。

## 参照

- [デストラクタでのファイナライゼーション](dtor-finally.md)および[RAIIガード](../patterns/behavioural/RAII.md)は、ライフタイムの厳密な制御から恩恵を受けることができます。
- （可変）参照の条件付きで埋められた`Option<&T>`の場合、`Option<T>`を直接初期化し、その[`.as_ref()`]メソッドを使用してオプショナルな参照を取得できます。

[`.as_ref()`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.as_ref
