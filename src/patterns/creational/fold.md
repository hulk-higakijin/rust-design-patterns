# Fold

## 説明

データのコレクション内の各アイテムに対してアルゴリズムを実行し、新しいアイテムを作成することで、全く新しいコレクションを作成します。

ここでの語源は私には不明確です。「fold」と「folder」という用語はRustコンパイラで使用されていますが、通常の意味でのfoldというよりもmapに近いように思えます。詳細については以下の議論を参照してください。

## 例

```rust,ignore
// The data we will fold, a simple AST.
mod ast {
    pub enum Stmt {
        Expr(Box<Expr>),
        Let(Box<Name>, Box<Expr>),
    }

    pub struct Name {
        value: String,
    }

    pub enum Expr {
        IntLit(i64),
        Add(Box<Expr>, Box<Expr>),
        Sub(Box<Expr>, Box<Expr>),
    }
}

// The abstract folder
mod fold {
    use ast::*;

    pub trait Folder {
        // A leaf node just returns the node itself. In some cases, we can do this
        // to inner nodes too.
        fn fold_name(&mut self, n: Box<Name>) -> Box<Name> { n }
        // Create a new inner node by folding its children.
        fn fold_stmt(&mut self, s: Box<Stmt>) -> Box<Stmt> {
            match *s {
                Stmt::Expr(e) => Box::new(Stmt::Expr(self.fold_expr(e))),
                Stmt::Let(n, e) => Box::new(Stmt::Let(self.fold_name(n), self.fold_expr(e))),
            }
        }
        fn fold_expr(&mut self, e: Box<Expr>) -> Box<Expr> { ... }
    }
}

use fold::*;
use ast::*;

// An example concrete implementation - renames every name to 'foo'.
struct Renamer;
impl Folder for Renamer {
    fn fold_name(&mut self, n: Box<Name>) -> Box<Name> {
        Box::new(Name { value: "foo".to_owned() })
    }
    // Use the default methods for the other nodes.
}
```

`Renamer`をASTに対して実行した結果は、古いASTと同一ですが、すべての名前が`foo`に変更された新しいASTです。実際のfolderは、構造体自体にノード間で保持される状態を持つ可能性があります。

folderは、あるデータ構造を別の（通常は類似した）データ構造にマッピングするように定義することもできます。例えば、ASTをHIRツリー（HIRは高レベル中間表現の略）に畳み込むことができます。

## 動機

データ構造内の各ノードに対して何らかの操作を実行してデータ構造をマッピングすることは一般的です。単純なデータ構造に対する単純な操作の場合、これは`Iterator::map`を使用して実行できます。より複雑な操作、おそらく以前のノードが後のノードの操作に影響を与える場合や、データ構造の反復が自明でない場合は、foldパターンを使用する方が適切です。

visitorパターンと同様に、foldパターンは、データ構造のトラバーサルを各ノードに対して実行される操作から分離することを可能にします。

## 議論

この方法でデータ構造をマッピングすることは、関数型言語では一般的です。オブジェクト指向言語では、データ構造をその場で変更する方がより一般的です。「関数型」アプローチはRustでは一般的で、主に不変性を好むためです。古いデータ構造を変更するのではなく、新しいデータ構造を使用することで、ほとんどの状況でコードについての推論が容易になります。

効率性と再利用性のトレードオフは、`fold_*`メソッドがノードをどのように受け入れるかを変更することで調整できます。

上記の例では、`Box`ポインタに対して操作を行っています。これらはデータを排他的に所有するため、データ構造の元のコピーを再利用することはできません。一方で、ノードが変更されていない場合、それを再利用することは非常に効率的です。

借用参照に対して操作を行う場合、元のデータ構造を再利用できます。ただし、変更されていない場合でもノードをクローンする必要があり、これは高コストになる可能性があります。

参照カウントポインタを使用すると、両方の長所が得られます - 元のデータ構造を再利用でき、変更されていないノードをクローンする必要がありません。ただし、使用が人間工学的でなく、データ構造が可変にできないことを意味します。

## 参考

イテレータには`fold`メソッドがありますが、これはデータ構造を新しいデータ構造ではなく値に畳み込みます。イテレータの`map`は、このfoldパターンにより近いものです。

他の言語では、foldは通常、このパターンではなく、Rustのイテレータの意味で使用されます。一部の関数型言語には、データ構造上で柔軟なマップを実行するための強力な構造があります。

[visitor](../behavioural/visitor.md)パターンは、foldと密接に関連しています。両者は、データ構造を歩いて各ノードに対して操作を実行するという概念を共有しています。ただし、visitorは新しいデータ構造を作成せず、古いデータ構造を消費しません。
