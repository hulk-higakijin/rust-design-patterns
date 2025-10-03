# Interpreter

## 説明

問題が頻繁に発生し、解決するために長く反復的な手順が必要な場合、問題のインスタンスを単純な言語で表現し、インタープリタオブジェクトがこの単純な言語で書かれた文を解釈することで問題を解決できます。

基本的に、あらゆる種類の問題に対して以下を定義します：

- [ドメイン固有言語](https://en.wikipedia.org/wiki/Domain-specific_language)
- この言語の文法
- 問題インスタンスを解決するインタープリタ

## 動機

私たちの目標は、単純な数式を後置記法（または[逆ポーランド記法](https://en.wikipedia.org/wiki/Reverse_Polish_notation)）に変換することです。
簡単にするため、式は10個の数字 `0`, ..., `9` と2つの演算子 `+`, `-` で構成されます。例えば、式 `2 + 4` は `2 4 +` に変換されます。

## 問題の文脈自由文法

タスクは中置式を後置式に変換することです。`0`, ..., `9`, `+`, `-` に対する中置式の集合の文脈自由文法を定義しましょう：

- 終端記号: `0`, `...`, `9`, `+`, `-`
- 非終端記号: `exp`, `term`
- 開始記号は `exp`
- そして以下が生成規則です

```ignore
exp -> exp + term
exp -> exp - term
exp -> term
term -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

**注意:** この文法は、何をするかによってさらに変換する必要があります。例えば、左再帰を除去する必要があるかもしれません。詳細については、[Compilers: Principles,Techniques, and Tools](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)（ドラゴンブックとも呼ばれる）を参照してください。

## 解決策

単純に再帰下降パーサーを実装します。簡単にするため、式が構文的に間違っている場合（例えば、文法定義によると `2-34` や `2+5-` は間違っています）、コードはパニックします。

```rust
pub struct Interpreter<'a> {
    it: std::str::Chars<'a>,
}

impl<'a> Interpreter<'a> {
    pub fn new(infix: &'a str) -> Self {
        Self { it: infix.chars() }
    }

    fn next_char(&mut self) -> Option<char> {
        self.it.next()
    }

    pub fn interpret(&mut self, out: &mut String) {
        self.term(out);

        while let Some(op) = self.next_char() {
            if op == '+' || op == '-' {
                self.term(out);
                out.push(op);
            } else {
                panic!("Unexpected symbol '{op}'");
            }
        }
    }

    fn term(&mut self, out: &mut String) {
        match self.next_char() {
            Some(ch) if ch.is_digit(10) => out.push(ch),
            Some(ch) => panic!("Unexpected symbol '{ch}'"),
            None => panic!("Unexpected end of string"),
        }
    }
}

pub fn main() {
    let mut intr = Interpreter::new("2+3");
    let mut postfix = String::new();
    intr.interpret(&mut postfix);
    assert_eq!(postfix, "23+");

    intr = Interpreter::new("1-2+3-4");
    postfix.clear();
    intr.interpret(&mut postfix);
    assert_eq!(postfix, "12-3+4-");
}
```

## 議論

Interpreterデザインパターンが形式言語の文法設計とこれらの文法のパーサーの実装に関するものだという誤った認識があるかもしれません。実際、このパターンは問題インスタンスをより具体的な方法で表現し、これらの問題インスタンスを解決する関数/クラス/構造体を実装することに関するものです。Rust言語には `macro_rules!` があり、特別な構文とこの構文をソースコードに展開する方法のルールを定義できます。

次の例では、`n` 次元ベクトルの[ユークリッド長](https://en.wikipedia.org/wiki/Euclidean_distance)を計算する単純な `macro_rules!` を作成します。`norm!(x,1,2)` と書く方が、`x,1,2` を `Vec` にパックして長さを計算する関数を呼び出すよりも表現しやすく、効率的かもしれません。

```rust
macro_rules! norm {
    ($($element:expr),*) => {
        {
            let mut n = 0.0;
            $(
                n += ($element as f64)*($element as f64);
            )*
            n.sqrt()
        }
    };
}

fn main() {
    let x = -3f64;
    let y = 4f64;

    assert_eq!(3f64, norm!(x));
    assert_eq!(5f64, norm!(x, y));
    assert_eq!(0f64, norm!(0, 0, 0));
    assert_eq!(1f64, norm!(0.5, -0.5, 0.5, -0.5));
}
```

## 参照

- [Interpreter pattern](https://en.wikipedia.org/wiki/Interpreter_pattern)
- [Context free grammar](https://en.wikipedia.org/wiki/Context-free_grammar)
- [macro_rules!](https://doc.rust-lang.org/rust-by-example/macros.html)
