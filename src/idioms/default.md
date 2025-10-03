# `Default` トレイト

## 説明

Rustの多くの型には[コンストラクタ]があります。しかし、これは型に*固有*のものです。Rustは「`new()`メソッドを持つすべてのもの」を抽象化することはできません。これを可能にするために[`Default`]トレイトが考案されました。これはコンテナやその他のジェネリック型で使用できます（例：[`Option::unwrap_or_default()`]を参照）。特に、一部のコンテナは該当する場合にすでにこれを実装しています。

`Cow`、`Box`、`Arc`のような単一要素のコンテナが、含まれる型が`Default`を実装している場合に`Default`を実装するだけでなく、すべてのフィールドが`Default`を実装している構造体に対して自動的に`#[derive(Default)]`できます。そのため、より多くの型が`Default`を実装するほど、より便利になります。

一方で、コンストラクタは複数の引数を取ることができますが、`default()`メソッドは引数を取りません。異なる名前を持つ複数のコンストラクタを持つことさえできますが、型ごとに`Default`の実装は1つしか持てません。

## 例

```rust
use std::{path::PathBuf, time::Duration};

// note that we can simply auto-derive Default here.
#[derive(Default, Debug, PartialEq)]
struct MyConfiguration {
    // Option defaults to None
    output: Option<PathBuf>,
    // Vecs default to empty vector
    search_path: Vec<PathBuf>,
    // Duration defaults to zero time
    timeout: Duration,
    // bool defaults to false
    check: bool,
}

impl MyConfiguration {
    // add setters here
}

fn main() {
    // construct a new instance with default values
    let mut conf = MyConfiguration::default();
    // do something with conf here
    conf.check = true;
    println!("conf = {conf:#?}");

    // partial initialization with default values, creates the same instance
    let conf1 = MyConfiguration {
        check: true,
        ..Default::default()
    };
    assert_eq!(conf, conf1);
}
```

## 参照

- [コンストラクタ]イディオムは、「デフォルト」である場合とそうでない場合があるインスタンスを生成する別の方法です
- [`Default`]ドキュメント（実装者のリストについては下にスクロールしてください）
- [`Option::unwrap_or_default()`]
- [`derive(new)`]

[コンストラクタ]: ctor.md
[`Default`]: https://doc.rust-lang.org/stable/std/default/trait.Default.html
[`Option::unwrap_or_default()`]: https://doc.rust-lang.org/stable/std/option/enum.Option.html#method.unwrap_or_default
[`derive(new)`]: https://crates.io/crates/derive-new/
