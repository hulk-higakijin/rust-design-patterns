# `mem::{take(_), replace(_)}` で変更された列挙型の所有値を保持する

## 説明

`&mut MyEnum` があり、（少なくとも）2つのバリアント、
`A { name: String, x: u8 }` と `B { name: String }` を持つとします。ここで、
`x` がゼロの場合は `MyEnum::A` を `B` に変更し、`MyEnum::B` はそのまま維持したいとします。

これは `name` をクローンすることなく実現できます。

## 例

```rust
use std::mem;

enum MyEnum {
    A { name: String, x: u8 },
    B { name: String },
}

fn a_to_b(e: &mut MyEnum) {
    if let MyEnum::A { name, x: 0 } = e {
        // This takes out our `name` and puts in an empty String instead
        // (note that empty strings don't allocate).
        // Then, construct the new enum variant (which will
        // be assigned to `*e`).
        *e = MyEnum::B {
            name: mem::take(name),
        }
    }
}
```

これは、より多くのバリアントでも動作します：

```rust
use std::mem;

enum MultiVariateEnum {
    A { name: String },
    B { name: String },
    C,
    D,
}

fn swizzle(e: &mut MultiVariateEnum) {
    use MultiVariateEnum::*;
    *e = match e {
        // Ownership rules do not allow taking `name` by value, but we cannot
        // take the value out of a mutable reference, unless we replace it:
        A { name } => B {
            name: mem::take(name),
        },
        B { name } => A {
            name: mem::take(name),
        },
        C => D,
        D => C,
    }
}
```

## 動機

列挙型を扱う際、列挙型の値を別のバリアントに変更したい場合があります。
これは通常、借用チェッカーを満足させるために2つのフェーズで行われます。最初のフェーズでは、既存の値を観察し、
その部分を見て次に何をすべきかを決定します。2番目のフェーズでは、
条件付きで値を変更する可能性があります（上記の例のように）。

借用チェッカーは、列挙型から `name` を取り出すことを許可しません（なぜなら、
*何か* がそこになければならないからです）。もちろん、name を `.clone()` して、そのクローンを
`MyEnum::B` に入れることもできますが、それは
[借用チェッカーを満足させるためのクローン](../anti_patterns/borrow_clone.md)
アンチパターンの一例になります。いずれにせよ、可変借用だけで `e` を変更することで、
余分なアロケーションを避けることができます。

`mem::take` は、値を交換して、デフォルト値に置き換え、
以前の値を返すことを可能にします。`String` の場合、デフォルト値は空の
`String` であり、アロケーションを必要としません。結果として、元の
`name` を *所有値として* 取得します。これを別の列挙型でラップできます。

**注：** `mem::replace` は非常に似ていますが、値を何に置き換えるかを指定できます。
`mem::take` 行と同等のものは
`mem::replace(name, String::new())` です。

ただし、`Option` を使用していて、その値を
`None` に置き換えたい場合、`Option` の `take()` メソッドは、より短く、よりイディオマティックな
代替手段を提供します。

## 利点

見てください、アロケーションなしです！また、これを行っている間、インディ・ジョーンズのように感じるかもしれません。

## 欠点

これは少し冗長になります。繰り返し間違えると、借用チェッカーを嫌いになるでしょう。
コンパイラは二重ストアを最適化できない場合があり、アンセーフな言語で行うことと比較して
パフォーマンスが低下する可能性があります。

さらに、取得する型は
[`Default` トレイト](./default.md)を実装する必要があります。ただし、作業している型が
これを実装していない場合は、代わりに `mem::replace` を使用できます。

## 議論

このパターンは Rust においてのみ興味深いものです。GC のある言語では、デフォルトで
値への参照を取得し（GC が参照を追跡します）、C のような他の低レベル言語では、単に
ポインタをエイリアスして、後で修正します。

しかし、Rust では、これを行うためにもう少し作業を行う必要があります。所有値は
1つの所有者しか持つことができないため、それを取り出すには、何かを戻す必要があります -
インディ・ジョーンズのように、アーティファクトを砂の袋に置き換えます。

## 関連項目

これは、特定のケースにおいて
[借用チェッカーを満足させるためのクローン](../anti_patterns/borrow_clone.md)
アンチパターンを取り除きます。
