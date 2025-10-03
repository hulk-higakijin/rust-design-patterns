# コンストラクタ

## 説明

Rustには言語構造としてのコンストラクタはありません。代わりに、オブジェクトを作成するために[関連関数][associated function]である`new`を使用することが慣習となっています：

````rust
/// 秒単位の時間。
///
/// # Example
///
/// ```
/// let s = Second::new(42);
/// assert_eq!(42, s.value());
/// ```
pub struct Second {
    value: u64,
}

impl Second {
    // Constructs a new instance of [`Second`].
    // Note this is an associated function - no self.
    pub fn new(value: u64) -> Self {
        Self { value }
    }

    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}
````

## デフォルトコンストラクタ

Rustは[`Default`][std-default]トレイトを使用したデフォルトコンストラクタをサポートしています：

````rust
/// 秒単位の時間。
///
/// # Example
///
/// ```
/// let s = Second::default();
/// assert_eq!(0, s.value());
/// ```
pub struct Second {
    value: u64,
}

impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}

impl Default for Second {
    fn default() -> Self {
        Self { value: 0 }
    }
}
````

`Default`は、`Second`のように全てのフィールドの型が`Default`を実装している場合、派生させることもできます：

````rust
/// 秒単位の時間。
///
/// # Example
///
/// ```
/// let s = Second::default();
/// assert_eq!(0, s.value());
/// ```
#[derive(Default)]
pub struct Second {
    value: u64,
}

impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}
````

**注意:** 型が`Default`と空の`new`コンストラクタの両方を実装することは一般的であり、期待されています。`new`はRustにおけるコンストラクタの慣習であり、ユーザーはそれが存在することを期待しているため、基本的なコンストラクタが引数を取らないことが妥当であれば、たとえdefaultと機能的に同一であっても、実装すべきです。

**ヒント:** `Default`を実装または派生させる利点は、`Default`実装が必要な場面で型を使用できるようになることです。最も顕著なのは、[標準ライブラリの`*or_default`関数][std-or-default]です。

## 参照

- `Default`トレイトの詳細な説明については[defaultイディオム](default.md)を参照してください。

- 複数の設定があるオブジェクトを構築する場合は[ビルダーパターン](../patterns/creational/builder.md)を参照してください。

- `Default`と`new`の両方を実装することについては[API Guidelines/C-COMMON-TRAITS][API Guidelines/C-COMMON-TRAITS]を参照してください。

[associated function]: https://doc.rust-lang.org/stable/book/ch05-03-method-syntax.html#associated-functions
[std-default]: https://doc.rust-lang.org/stable/std/default/trait.Default.html
[std-or-default]: https://doc.rust-lang.org/stable/std/?search=or_default
[API Guidelines/C-COMMON-TRAITS]: https://rust-lang.github.io/api-guidelines/interoperability.html#types-eagerly-implement-common-traits-c-common-traits
