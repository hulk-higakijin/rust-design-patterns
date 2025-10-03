# Builder

## 説明

ビルダーヘルパーへの呼び出しを使用してオブジェクトを構築します。

## 例

```rust
#[derive(Debug, PartialEq)]
pub struct Foo {
    // Lots of complicated fields.
    bar: String,
}

impl Foo {
    // This method will help users to discover the builder
    pub fn builder() -> FooBuilder {
        FooBuilder::default()
    }
}

#[derive(Default)]
pub struct FooBuilder {
    // Probably lots of optional fields.
    bar: String,
}

impl FooBuilder {
    pub fn new(/* ... */) -> FooBuilder {
        // Set the minimally required fields of Foo.
        FooBuilder {
            bar: String::from("X"),
        }
    }

    pub fn name(mut self, bar: String) -> FooBuilder {
        // Set the name on the builder itself, and return the builder by value.
        self.bar = bar;
        self
    }

    // If we can get away with not consuming the Builder here, that is an
    // advantage. It means we can use the FooBuilder as a template for constructing
    // many Foos.
    pub fn build(self) -> Foo {
        // Create a Foo from the FooBuilder, applying all settings in FooBuilder
        // to Foo.
        Foo { bar: self.bar }
    }
}

#[test]
fn builder_test() {
    let foo = Foo {
        bar: String::from("Y"),
    };
    let foo_from_builder: Foo = FooBuilder::new().name(String::from("Y")).build();
    assert_eq!(foo, foo_from_builder);
}
```

## 動機

多数のコンストラクタが必要になる場合や、構築に副作用がある場合に有用です。

## 利点

構築用のメソッドを他のメソッドから分離します。

コンストラクタの増殖を防ぎます。

ワンライナーでの初期化にも、より複雑な構築にも使用できます。

## 欠点

構造体オブジェクトを直接作成したり、シンプルなコンストラクタ関数を使用するよりも複雑です。

## 議論

このパターンは、Rustにオーバーロードがないため、他の多くの言語よりもRust（そしてよりシンプルなオブジェクト）でより頻繁に見られます。特定の名前を持つメソッドは1つしか持てないため、複数のコンストラクタを持つことは、C++やJavaなどと比較してRustではあまり適していません。

このパターンは、ビルダーオブジェクトが単なるビルダーではなく、それ自体が有用である場合によく使用されます。例えば、
[`std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html)
は
[`Child`](https://doc.rust-lang.org/std/process/struct.Child.html)（プロセス）
のビルダーです。このような場合、`T`と`TBuilder`の命名パターンは使用されません。

この例では、ビルダーを値で受け取り、値で返します。ビルダーを可変参照として受け取り、返す方が、より人間工学的（そしてより効率的）であることがよくあります。借用チェッカーはこれを自然に機能させます。このアプローチには、次のようなコードを書けるという利点があります

```rust,ignore
let mut fb = FooBuilder::new();
fb.a();
fb.b();
let f = fb.build();
```

`FooBuilder::new().a().b().build()` スタイルと同様に。

## 参考

- [Description in the style guide](https://web.archive.org/web/20210104103100/https://doc.rust-lang.org/1.12.0/style/ownership/builders.html)
- [derive_builder](https://crates.io/crates/derive_builder), a crate for
  automatically implementing this pattern while avoiding the boilerplate.
- [Constructor pattern](../../idioms/ctor.md) for when construction is simpler.
- [Builder pattern (wikipedia)](https://en.wikipedia.org/wiki/Builder_pattern)
- [Construction of complex values](https://web.archive.org/web/20210104103000/https://rust-lang.github.io/api-guidelines/type-safety.html#c-builder)
