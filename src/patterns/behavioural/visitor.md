# Visitor

## 説明

Visitorは、異種のオブジェクトコレクションに対して動作するアルゴリズムをカプセル化します。これにより、データ(またはその主要な動作)を変更することなく、同じデータに対して複数の異なるアルゴリズムを記述できます。

さらに、Visitorパターンは、オブジェクトコレクションの走査と各オブジェクトに対して実行される操作を分離することを可能にします。

## 例

```rust,ignore
// The data we will visit
mod ast {
    pub enum Stmt {
        Expr(Expr),
        Let(Name, Expr),
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

// The abstract visitor
mod visit {
    use ast::*;

    pub trait Visitor<T> {
        fn visit_name(&mut self, n: &Name) -> T;
        fn visit_stmt(&mut self, s: &Stmt) -> T;
        fn visit_expr(&mut self, e: &Expr) -> T;
    }
}

use ast::*;
use visit::*;

// An example concrete implementation - walks the AST interpreting it as code.
struct Interpreter;
impl Visitor<i64> for Interpreter {
    fn visit_name(&mut self, n: &Name) -> i64 {
        panic!()
    }
    fn visit_stmt(&mut self, s: &Stmt) -> i64 {
        match *s {
            Stmt::Expr(ref e) => self.visit_expr(e),
            Stmt::Let(..) => unimplemented!(),
        }
    }

    fn visit_expr(&mut self, e: &Expr) -> i64 {
        match *e {
            Expr::IntLit(n) => n,
            Expr::Add(ref lhs, ref rhs) => self.visit_expr(lhs) + self.visit_expr(rhs),
            Expr::Sub(ref lhs, ref rhs) => self.visit_expr(lhs) - self.visit_expr(rhs),
        }
    }
}
```

ASTデータを変更することなく、型チェッカーなどのさらなるVisitorを実装できます。

## 動機

Visitorパターンは、異種のデータにアルゴリズムを適用したい場合に便利です。データが同種である場合は、イテレータのようなパターンを使用できます。Visitorオブジェクトを使用すること(関数型アプローチではなく)により、Visitorがステートフルになり、ノード間で情報を伝達できるようになります。

## 議論

`visit_*`メソッドがvoidを返すこと(例のように戻り値を持たない)は一般的です。その場合、走査コードを分離し、アルゴリズム間で共有することが可能です(また、デフォルトのno-opメソッドを提供することもできます)。Rustでは、これを行う一般的な方法は、各データに対して`walk_*`関数を提供することです。例えば、

```rust,ignore
pub fn walk_expr(visitor: &mut Visitor, e: &Expr) {
    match *e {
        Expr::IntLit(_) => {}
        Expr::Add(ref lhs, ref rhs) => {
            visitor.visit_expr(lhs);
            visitor.visit_expr(rhs);
        }
        Expr::Sub(ref lhs, ref rhs) => {
            visitor.visit_expr(lhs);
            visitor.visit_expr(rhs);
        }
    }
}
```

他の言語(例えばJava)では、データに同じ役割を果たす`accept`メソッドを持たせることが一般的です。

## 参照

Visitorパターンは、ほとんどのオブジェクト指向言語で一般的なパターンです。

[Wikipedia article](https://en.wikipedia.org/wiki/Visitor_pattern)

[fold](../creational/fold.md)パターンはVisitorと似ていますが、訪問されたデータ構造の新しいバージョンを生成します。
