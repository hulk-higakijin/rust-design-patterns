# `#![deny(warnings)]`

## 説明

善意のあるクレート作成者は、自分のコードが警告なしでビルドされることを保証したいと考えています。そのため、クレートのルートに以下のアノテーションを付けます:

## 例

```rust
#![deny(warnings)]

// All is well.
```

## 利点

短く、何か問題があればビルドを停止します。

## 欠点

コンパイラが警告付きでビルドすることを禁止することで、クレート作成者はRustの有名な安定性からオプトアウトすることになります。時には新機能や古い誤機能が、物事の行い方の変更を必要とすることがあり、そのため一定の猶予期間中に`warn`を出すlintが書かれ、その後`deny`に変更されます。

例えば、ある型が同じメソッドを持つ2つの`impl`を持つことができることが発見されました。これは悪い考えだと判断されましたが、移行をスムーズにするために、将来のリリースでハードエラーになる前に、この事実に遭遇した人に警告を与えるために`overlapping-inherent-impls` lintが導入されました。

また、時にはAPIが非推奨になることがあり、以前は警告がなかった場所で警告が発生するようになります。

これらすべてが相まって、何かが変更されるたびにビルドが壊れる可能性があります。

さらに、追加のlintを提供するクレート（例：[rust-clippy]）は、アノテーションを削除しない限り使用できなくなります。これは[--cap-lints]によって緩和されます。`--cap-lints=warn`コマンドライン引数は、すべての`deny` lintエラーを警告に変えます。

## 代替案

この問題に取り組むには2つの方法があります。第一に、ビルド設定をコードから切り離すことができ、第二に、明示的に拒否したいlintを指定することができます。

以下のコマンドラインは、すべての警告を`deny`に設定してビルドします:

`RUSTFLAGS="-D warnings" cargo build`

これは、コードの変更を必要とせずに、個々の開発者が実行できます（またはTravisのようなCIツールで設定できますが、何かが変更されたときにビルドが壊れる可能性があることを覚えておいてください）。

あるいは、コード内で`deny`したいlintを指定することもできます。以下は、（おそらく）安全に拒否できる警告lintのリストです（rustc 1.48.0時点）:

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

さらに、以下の`allow`されたlintを`deny`するのも良いかもしれません:

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

`deprecated` lintを明示的に追加しなかったことに注意してください。将来さらに非推奨のAPIが追加されることはほぼ確実だからです。

## 関連項目

- [すべてのclippy lintのコレクション](https://rust-lang.github.io/rust-clippy/master)
- [deprecate attribute] のドキュメント
- システム上のlintのリストを見るには`rustc -W help`と入力してください。また、一般的なオプションのリストを見るには`rustc --help`と入力してください
- [rust-clippy]は、より良いRustコードのためのlintのコレクションです

[rust-clippy]: https://github.com/rust-lang/rust-clippy
[deprecate attribute]: https://doc.rust-lang.org/reference/attributes.html#deprecation
[--cap-lints]: https://doc.rust-lang.org/rustc/lints/levels.html#capping-lints
