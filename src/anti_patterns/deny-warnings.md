# `#![deny(warnings)]`

## 説明

善意のあるクレート作成者は、自分のコードが警告なしでビルドされることを保証したいと考えています。そのため、クレートのルートに以下のようなアノテーションを追加します：

## 例

```rust
#![deny(warnings)]

// All is well.
```

## 利点

短くて簡潔であり、何か問題があればビルドを停止します。

## 欠点

警告付きでのコンパイラのビルドを許可しないことで、クレート作成者はRustの有名な安定性から離脱することになります。時には新機能や古い誤機能の変更が必要になり、そのためlintが作成されます。これらのlintは一定の猶予期間中は`warn`を発し、その後`deny`に変更されます。

例えば、ある型が同じメソッドを持つ2つの`impl`を持つことができることが発見されました。これは良くないアイデアとされましたが、移行をスムーズにするために、`overlapping-inherent-impls` lintが導入され、将来のリリースでハードエラーになる前に、この事実に遭遇した人に警告を出すようになりました。

また、時にはAPIが非推奨になることがあり、以前は警告がなかった場所で警告が発せられるようになります。

これらすべてが相まって、何かが変更されるたびにビルドが壊れる可能性があります。

さらに、追加のlintを提供するクレート（例：[rust-clippy]）は、アノテーションを削除しない限り使用できなくなります。これは[--cap-lints]によって緩和されます。`--cap-lints=warn` コマンドライン引数は、すべての`deny` lintエラーを警告に変換します。

## 代替案

この問題に対処する方法は2つあります。第一に、ビルド設定をコードから分離すること、第二に、明示的にdenyしたいlintを指定することです。

以下のコマンドラインは、すべての警告を`deny`に設定してビルドします：

`RUSTFLAGS="-D warnings" cargo build`

これは個々の開発者が実行できます（またはTravisのようなCIツールで設定できますが、何かが変更されたときにビルドが壊れる可能性があることを覚えておいてください）。コードの変更は必要ありません。

あるいは、コード内で`deny`したいlintを指定することもできます。以下は、denyしても（おそらく）安全な警告lintのリストです（rustc 1.48.0時点）：

```rust,ignore
#![deny(
    bad_style,
    const_err,
    dead_code,
    improper_ctypes,
    non_shorthand_field_patterns,
    no_mangle_generic_items,
    overflowing_literals,
    path_statements,
    patterns_in_fns_without_body,
    private_in_public,
    unconditional_recursion,
    unused,
    unused_allocation,
    unused_comparisons,
    unused_parens,
    while_true
)]
```

さらに、以下の`allow`されたlintも`deny`するのが良いかもしれません：

```rust,ignore
#![deny(
    missing_debug_implementations,
    missing_docs,
    trivial_casts,
    trivial_numeric_casts,
    unused_extern_crates,
    unused_import_braces,
    unused_qualifications,
    unused_results
)]
```

`missing-copy-implementations`をリストに追加したい人もいるかもしれません。

明示的に`deprecated` lintを追加しなかったことに注意してください。将来さらに非推奨のAPIが増えることはほぼ確実だからです。

## 参照

- [A collection of all clippy lints](https://rust-lang.github.io/rust-clippy/master)
- [deprecate attribute] documentation
- Type `rustc -W help` for a list of lints on your system. Also type
  `rustc --help` for a general list of options
- [rust-clippy] is a collection of lints for better Rust code

[rust-clippy]: https://github.com/rust-lang/rust-clippy
[deprecate attribute]: https://doc.rust-lang.org/reference/attributes.html#deprecation
[--cap-lints]: https://doc.rust-lang.org/rustc/lints/levels.html#capping-lints
