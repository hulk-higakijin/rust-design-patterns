# `Option`に対する反復処理

## 説明

`Option`は、0個または1個の要素を含むコンテナとして見ることができます。
特に、`IntoIterator`トレイトを実装しているため、そのような型を必要とする
ジェネリックなコードで使用できます。

## 例

`Option`は`IntoIterator`を実装しているため、
[`.extend()`](https://doc.rust-lang.org/std/iter/trait.Extend.html#tymethod.extend)の引数として使用できます：

```rust
let turing = Some("Turing");
let mut logicians = vec!["Curry", "Kleene", "Markov"];

logicians.extend(turing);

// equivalent to
if let Some(turing_inner) = turing {
    logicians.push(turing_inner);
}
```

既存のイテレータの末尾に`Option`を追加する必要がある場合は、
[`.chain()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.chain)に渡すことができます：

```rust
let turing = Some("Turing");
let logicians = vec!["Curry", "Kleene", "Markov"];

for logician in logicians.iter().chain(turing.iter()) {
    println!("{logician} is a logician");
}
```

`Option`が常に`Some`である場合は、要素に対して
[`std::iter::once`](https://doc.rust-lang.org/std/iter/fn.once.html)を使用する方が
より慣用的であることに注意してください。

また、`Option`は`IntoIterator`を実装しているため、`for`ループを使用して
反復処理することが可能です。これは`if let Some(..)`でマッチングするのと同等ですが、
ほとんどの場合、後者の方が望ましいでしょう。

## 参考

- [`std::iter::once`](https://doc.rust-lang.org/std/iter/fn.once.html)は、
  正確に1つの要素を生成するイテレータです。`Some(foo).into_iter()`よりも
  読みやすい代替手段です。

- [`Iterator::filter_map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map)は、
  [`Iterator::map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map)の
  バージョンで、`Option`を返すマッピング関数に特化しています。

- [`ref_slice`](https://crates.io/crates/ref_slice)クレートは、
  `Option`を0個または1個の要素を持つスライスに変換する関数を提供します。

- [Documentation for `Option<T>`](https://doc.rust-lang.org/std/option/enum.Option.html)
