# クロージャに変数を渡す

## 説明

デフォルトでは、クロージャは環境を借用してキャプチャします。または、`move`クロージャを使用して環境全体をムーブすることもできます。しかし、多くの場合、一部の変数だけをクロージャにムーブしたり、データのコピーを渡したり、参照で渡したり、その他の変換を行いたいことがあります。

そのためには、別のスコープで変数の再束縛を使用します。

## 例

以下のように使用します。

```rust
use std::rc::Rc;

let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);
let closure = {
    // `num1` is moved
    let num2 = num2.clone();  // `num2` is cloned
    let num3 = num3.as_ref();  // `num3` is borrowed
    move || {
        *num1 + *num2 + *num3;
    }
};
```

以下の代わりに

```rust
use std::rc::Rc;

let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);

let num2_cloned = num2.clone();
let num3_borrowed = num3.as_ref();
let closure = move || {
    *num1 + *num2_cloned + *num3_borrowed;
};
```

## 利点

コピーされたデータはクロージャの定義とともにグループ化されるため、その目的がより明確になります。また、クロージャで消費されない場合でも、すぐにドロップされます。

クロージャは、データがコピーされたかムーブされたかにかかわらず、周囲のコードと同じ変数名を使用します。

## 欠点

クロージャ本体の追加のインデント。
