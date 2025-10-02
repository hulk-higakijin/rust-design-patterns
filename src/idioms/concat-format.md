# `format!` を使った文字列の連結

## 説明

`push` や `push_str` メソッドを可変な `String` に使用したり、`+` 演算子を使用することで文字列を構築することができます。しかし、特にリテラル文字列と非リテラル文字列が混在する場合は、`format!` を使用する方が便利なことが多いです。

## 例

```rust
fn say_hello(name: &str) -> String {
    // We could construct the result string manually.
    // let mut result = "Hello ".to_owned();
    // result.push_str(name);
    // result.push('!');
    // result

    // But using format! is better.
    format!("Hello {name}!")
}
```

## 利点

`format!` を使用することは、通常、文字列を結合する最も簡潔で読みやすい方法です。

## 欠点

これは通常、文字列を結合する最も効率的な方法ではありません - 可変文字列に対する一連の `push` 操作が通常最も効率的です（特に文字列が予想されるサイズに事前に割り当てられている場合）。
