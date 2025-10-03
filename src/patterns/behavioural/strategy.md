# Strategy (別名 Policy)

## 説明

[Strategyデザインパターン](https://en.wikipedia.org/wiki/Strategy_pattern)は、関心の分離を可能にする技術です。また、[依存性逆転の原則](https://en.wikipedia.org/wiki/Dependency_inversion_principle)を通じてソフトウェアモジュールを疎結合にすることもできます。

Strategyパターンの基本的な考え方は、特定の問題を解決するアルゴリズムが与えられたとき、抽象レベルでアルゴリズムの骨格のみを定義し、具体的なアルゴリズムの実装を異なる部分に分離することです。

この方法により、アルゴリズムを使用するクライアントは特定の実装を選択できますが、一般的なアルゴリズムのワークフローは同じままです。言い換えれば、クラスの抽象仕様は派生クラスの具体的な実装に依存しませんが、具体的な実装は抽象仕様に従う必要があります。これが「依存性逆転」と呼ばれる理由です。

## 動機

毎月レポートを生成するプロジェクトに取り組んでいるとします。レポートをさまざまな形式（戦略）で生成する必要があります。例えば、`JSON`や`プレーンテキスト`形式などです。しかし、状況は時間とともに変化し、将来どのような要件が発生するかわかりません。例えば、まったく新しい形式でレポートを生成する必要があるかもしれませんし、既存の形式の1つを単に修正するだけかもしれません。

## 例

この例では、`Formatter`と`Report`が不変条件（または抽象）であり、`Text`と`Json`が戦略構造体です。これらの戦略は`Formatter`トレイトを実装する必要があります。

```rust
use std::collections::HashMap;

type Data = HashMap<String, u32>;

trait Formatter {
    fn format(&self, data: &Data, buf: &mut String);
}

struct Report;

impl Report {
    // Write should be used but we kept it as String to ignore error handling
    fn generate<T: Formatter>(g: T, s: &mut String) {
        // backend operations...
        let mut data = HashMap::new();
        data.insert("one".to_string(), 1);
        data.insert("two".to_string(), 2);
        // generate report
        g.format(&data, s);
    }
}

struct Text;
impl Formatter for Text {
    fn format(&self, data: &Data, buf: &mut String) {
        for (k, v) in data {
            let entry = format!("{k} {v}\n");
            buf.push_str(&entry);
        }
    }
}

struct Json;
impl Formatter for Json {
    fn format(&self, data: &Data, buf: &mut String) {
        buf.push('[');
        for (k, v) in data.into_iter() {
            let entry = format!(r#"{{"{}":"{}"}}"#, k, v);
            buf.push_str(&entry);
            buf.push(',');
        }
        if !data.is_empty() {
            buf.pop(); // remove extra , at the end
        }
        buf.push(']');
    }
}

fn main() {
    let mut s = String::from("");
    Report::generate(Text, &mut s);
    assert!(s.contains("one 1"));
    assert!(s.contains("two 2"));

    s.clear(); // reuse the same buffer
    Report::generate(Json, &mut s);
    assert!(s.contains(r#"{"one":"1"}"#));
    assert!(s.contains(r#"{"two":"2"}"#));
}
```

## 利点

主な利点は関心の分離です。例えば、この場合、`Report`は`Json`と`Text`の具体的な実装について何も知りませんし、出力実装はデータがどのように前処理され、保存され、取得されるかを気にしません。彼らが知る必要があるのは、実装すべき特定のトレイトと、結果を処理する具体的なアルゴリズム実装を定義するそのメソッド、つまり`Formatter`と`format(...)`だけです。

## 欠点

各戦略に対して少なくとも1つのモジュールを実装する必要があるため、戦略の数とともにモジュールの数も増加します。選択できる戦略が多数ある場合、ユーザーは戦略がどのように異なるかを知る必要があります。

## 議論

前の例では、すべての戦略が単一ファイルに実装されています。異なる戦略を提供する方法には次のものがあります：

- すべて1つのファイルに（この例で示したように、モジュールとして分離するのと同様）
- モジュールとして分離、例：`formatter::json`モジュール、`formatter::text`モジュール
- コンパイラの機能フラグを使用、例：`json`機能、`text`機能
- クレートとして分離、例：`json`クレート、`text`クレート

Serdeクレートは、`Strategy`パターンの良い例です。Serdeは、型に対して`Serialize`と`Deserialize`トレイトを手動で実装することで、シリアライゼーション動作の[完全なカスタマイズ](https://serde.rs/custom-serialization.html)を可能にします。例えば、`serde_json`と`serde_cbor`は同様のメソッドを公開しているため、簡単に交換できます。これにより、ヘルパークレート`serde_transcode`がより便利で人間工学的になります。

しかし、Rustでこのパターンを設計するためにトレイトを使用する必要はありません。

以下のおもちゃの例は、Rustの`クロージャ`を使用したStrategyパターンのアイデアを示しています：

```rust
struct Adder;
impl Adder {
    pub fn add<F>(x: u8, y: u8, f: F) -> u8
    where
        F: Fn(u8, u8) -> u8,
    {
        f(x, y)
    }
}

fn main() {
    let arith_adder = |x, y| x + y;
    let bool_adder = |x, y| {
        if x == 1 || y == 1 {
            1
        } else {
            0
        }
    };
    let custom_adder = |x, y| 2 * x + y;

    assert_eq!(9, Adder::add(4, 5, arith_adder));
    assert_eq!(0, Adder::add(0, 0, bool_adder));
    assert_eq!(5, Adder::add(1, 3, custom_adder));
}
```

実際、Rustはすでに`Options`の`map`メソッドでこのアイデアを使用しています：

```rust
fn main() {
    let val = Some("Rust");

    let len_strategy = |s: &str| s.len();
    assert_eq!(4, val.map(len_strategy).unwrap());

    let first_byte_strategy = |s: &str| s.bytes().next().unwrap();
    assert_eq!(82, val.map(first_byte_strategy).unwrap());
}
```

## 参照

- [Strategy Pattern](https://en.wikipedia.org/wiki/Strategy_pattern)
- [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection)
- [Policy Based Design](https://en.wikipedia.org/wiki/Modern_C++_Design#Policy-based_design)
- [Implementing a TCP server for Space Applications in Rust using the Strategy Pattern](https://web.archive.org/web/20231003171500/https://robamu.github.io/posts/rust-strategy-pattern/)
