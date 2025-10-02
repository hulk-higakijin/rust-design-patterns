# 借用チェッカーを満たすためのクローン

## 説明

借用チェッカーは、可変参照が1つだけ存在するか、または多数の不変参照が存在するかのいずれかを保証することで、Rustユーザーが安全でないコードを開発することを防ぎます。記述されたコードがこれらの条件を満たさない場合、開発者が変数をクローンすることでコンパイラエラーを解決しようとするときに、このアンチパターンが発生します。

## 例

```rust
// define any variable
let mut x = 5;

// Borrow `x` -- but clone it first
let y = &mut (x.clone());

// without the x.clone() two lines prior, this line would fail on compile as
// x has been borrowed
// thanks to x.clone(), x was never borrowed, and this line will run.
println!("{x}");

// perform some action on the borrow to prevent rust from optimizing this
//out of existence
*y += 1;
```

## 動機

初心者にとっては特に、借用チェッカーの分かりにくい問題を解決するために、このパターンを使用したくなることがあります。しかし、深刻な結果があります。`.clone()`を使用すると、データのコピーが作成されます。2つの間の変更は同期されません――まるで2つの完全に別々の変数が存在するかのようです。

特別なケース――`Rc<T>`はクローンをインテリジェントに処理するように設計されています。内部的には、データのコピーを正確に1つだけ管理します。`Rc`で`.clone()`を呼び出すと、参照カウントを増やしながら、ソース`Rc`と同じデータを指す新しい`Rc`インスタンスが生成されます。同じことが、`Rc`のスレッドセーフな対応物である`Arc`にも当てはまります。

一般的に、クローンは意図的であり、結果を完全に理解した上で行うべきです。借用チェッカーのエラーを消すためにクローンが使用されている場合、それはこのアンチパターンが使用されている可能性がある良い兆候です。

`.clone()`がバッドパターンの兆候であるとしても、次のような場合には**非効率的なコードを書くことが問題ない**こともあります：

- 開発者が所有権についてまだ初心者である
- コードに大きな速度やメモリの制約がない（ハッカソンプロジェクトやプロトタイプなど）
- 借用チェッカーを満たすことが本当に複雑で、パフォーマンスよりも可読性を優先したい

不要なクローンが疑われる場合、クローンが必要かどうかを評価する前に、
[Rust Bookの所有権に関する章](https://doc.rust-lang.org/book/ownership.html)を完全に理解する必要があります。

また、プロジェクトで常に`cargo clippy`を実行することを忘れないでください。これにより、`.clone()`が不要な場合がいくつか検出されます。

## 参照

- [`mem::{take(_), replace(_)}` to keep owned values in changed enums](../idioms/mem-replace.md)
- [`Rc<T>` documentation, which handles .clone() intelligently](http://doc.rust-lang.org/std/rc/)
- [`Arc<T>` documentation, a thread-safe reference-counting pointer](https://doc.rust-lang.org/std/sync/struct.Arc.html)
- [Tricks with ownership in Rust](https://web.archive.org/web/20210120233744/https://xion.io/post/code/rust-borrowchk-tricks.html)
