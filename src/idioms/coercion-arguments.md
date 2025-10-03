# 引数には borrowed 型を使う

## 説明

deref 型強制の対象となる型を使用することで、関数の引数にどの型を使うかを決める際にコードの柔軟性を高めることができます。この方法により、関数はより多くの入力型を受け入れることができるようになります。

これはスライス可能な型やファットポインタ型に限定されません。実際、**所有型への借用**よりも**借用型**を常に優先すべきです。たとえば、`&String` ではなく `&str`、`&Vec<T>` ではなく `&[T]`、`&Box<T>` ではなく `&T` を使用します。

borrowed 型を使用することで、所有型が既に間接参照の層を提供している場合に、間接参照の層を避けることができます。たとえば、`String` は間接参照の層を持っているため、`&String` は2層の間接参照を持つことになります。代わりに `&str` を使用することでこれを避けることができ、関数が呼び出されるたびに `&String` が `&str` に型強制されます。

## 例

この例では、関数の引数として `&String` を使用する場合と `&str` を使用する場合のいくつかの違いを説明しますが、この考え方は `&Vec<T>` と `&[T]` の使用、または `&Box<T>` と `&T` の使用にも同様に適用されます。

3つの連続した母音を含む単語かどうかを判定したい例を考えてみましょう。これを判定するために文字列を所有する必要はないので、参照を取ります。

コードは次のようになります:

```rust
fn three_vowels(word: &String) -> bool {
    let mut vowel_count = 0;
    for c in word.chars() {
        match c {
            'a' | 'e' | 'i' | 'o' | 'u' => {
                vowel_count += 1;
                if vowel_count >= 3 {
                    return true;
                }
            }
            _ => vowel_count = 0,
        }
    }
    false
}

fn main() {
    let ferris = "Ferris".to_string();
    let curious = "Curious".to_string();
    println!("{}: {}", ferris, three_vowels(&ferris));
    println!("{}: {}", curious, three_vowels(&curious));

    // This works fine, but the following two lines would fail:
    // println!("Ferris: {}", three_vowels("Ferris"));
    // println!("Curious: {}", three_vowels("Curious"));
}
```

これは `&String` 型をパラメータとして渡しているため正常に動作します。最後の2行のコメントを外すと、例は失敗します。これは `&str` 型が `&String` 型に型強制されないためです。この問題は、引数の型を単純に変更することで修正できます。

たとえば、関数宣言を次のように変更すると:

```rust, ignore
fn three_vowels(word: &str) -> bool {
```

両方のバージョンがコンパイルされ、同じ出力が表示されます。

```bash
Ferris: false
Curious: true
```

しかし、それだけではありません！この話にはさらに続きがあります。おそらく、あなたは「そんなことは問題にならない、私は入力として `&'static str` を使うことはない」と考えるかもしれません（`"Ferris"` を使用したときのように）。この特別な例を無視しても、`&str` を使用することで `&String` を使用するよりも柔軟性が高まることがわかるでしょう。

では、誰かから文章を与えられ、その文章内の単語のいずれかに3つの連続した母音が含まれているかどうかを判定したい例を考えてみましょう。すでに定義した関数を利用して、文章から各単語を入力すればよいでしょう。

この例は次のようになります:

```rust
fn three_vowels(word: &str) -> bool {
    let mut vowel_count = 0;
    for c in word.chars() {
        match c {
            'a' | 'e' | 'i' | 'o' | 'u' => {
                vowel_count += 1;
                if vowel_count >= 3 {
                    return true;
                }
            }
            _ => vowel_count = 0,
        }
    }
    false
}

fn main() {
    let sentence_string =
        "Once upon a time, there was a friendly curious crab named Ferris".to_string();
    for word in sentence_string.split(' ') {
        if three_vowels(word) {
            println!("{word} has three consecutive vowels!");
        }
    }
}
```

引数の型を `&str` で宣言した関数を使ってこの例を実行すると、次のように出力されます:

```bash
curious has three consecutive vowels!
```

しかし、この例は引数の型を `&String` で宣言した関数では実行できません。これは、文字列スライスは `&str` であって `&String` ではないため、`&String` に変換するにはメモリ割り当てが必要になり、これは暗黙的には行われないからです。一方、`String` から `&str` への変換は安価で暗黙的に行われます。

## 参照

- [Rust Language Reference on Type Coercions](https://doc.rust-lang.org/reference/type-coercions.html)
- `String` と `&str` の扱い方についての詳細な議論は、Herman J. Radtke III による
  [このブログシリーズ (2015)](https://web.archive.org/web/20201112023149/https://hermanradtke.com/2015/05/03/string-vs-str-in-rust-functions.html)
  を参照してください
- [Steve Klabnik のブログ投稿 'When should I use String vs &str?'](https://archive.ph/LBpD0)
