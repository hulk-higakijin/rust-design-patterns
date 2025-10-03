# ガードを用いたRAII

## 説明

[RAII][wikipedia]は「Resource Acquisition is Initialisation（リソース取得は初期化である）」の略で、あまり良い名前とは言えません。このパターンの本質は、リソースの初期化がオブジェクトのコンストラクタで行われ、終了処理がデストラクタで行われることです。このパターンはRustでは、RAIIオブジェクトをリソースのガードとして使用し、型システムに依存してアクセスが常にガードオブジェクトによって仲介されることを保証することで拡張されています。

## 例

Mutexガードは、標準ライブラリからこのパターンの典型的な例です（これは実際の実装を簡略化したバージョンです）：

```rust,ignore
use std::ops::Deref;

struct Foo {}

struct Mutex<T> {
    // We keep a reference to our data: T here.
    //..
}

struct MutexGuard<'a, T: 'a> {
    data: &'a T,
    //..
}

// Locking the mutex is explicit.
impl<T> Mutex<T> {
    fn lock(&self) -> MutexGuard<T> {
        // Lock the underlying OS mutex.
        //..

        // MutexGuard keeps a reference to self
        MutexGuard {
            data: self,
            //..
        }
    }
}

// Destructor for unlocking the mutex.
impl<'a, T> Drop for MutexGuard<'a, T> {
    fn drop(&mut self) {
        // Unlock the underlying OS mutex.
        //..
    }
}

// Implementing Deref means we can treat MutexGuard like a pointer to T.
impl<'a, T> Deref for MutexGuard<'a, T> {
    type Target = T;

    fn deref(&self) -> &T {
        self.data
    }
}

fn baz(x: Mutex<Foo>) {
    let xx = x.lock();
    xx.foo(); // foo is a method on Foo.
              // The borrow checker ensures we can't store a reference to the underlying
              // Foo which will outlive the guard xx.

    // x is unlocked when we exit this function and xx's destructor is executed.
}
```

## 動機

使用後にリソースを終了処理しなければならない場合、RAIIを使ってこの終了処理を行うことができます。終了処理後にそのリソースにアクセスすることがエラーである場合、このパターンを使ってそのようなエラーを防ぐことができます。

## 利点

リソースが終了処理されていない場合や、終了処理後にリソースが使用される場合のエラーを防ぎます。

## 議論

RAIIは、リソースが適切に解放または終了処理されることを保証するための有用なパターンです。Rustでは借用チェッカーを利用して、終了処理が行われた後にリソースを使用することから生じるエラーを静的に防ぐことができます。

借用チェッカーの中心的な目的は、データへの参照がそのデータよりも長生きしないことを保証することです。RAIIガードパターンが機能するのは、ガードオブジェクトが基礎となるリソースへの参照を含み、そのような参照のみを公開するためです。Rustは、ガードが基礎となるリソースよりも長生きできないこと、およびガードによって仲介されるリソースへの参照がガードよりも長生きできないことを保証します。これがどのように機能するかを理解するには、ライフタイム省略なしの`deref`のシグネチャを調べると役立ちます：

```rust,ignore
fn deref<'a>(&'a self) -> &'a T {
    //..
}
```

リソースへの返される参照は、`self`と同じライフタイム（`'a`）を持ちます。したがって、借用チェッカーは`T`への参照のライフタイムが`self`のライフタイムよりも短いことを保証します。

`Deref`を実装することは、このパターンの核心部分ではなく、ガードオブジェクトの使用をより人間工学的にするだけであることに注意してください。ガードに`get`メソッドを実装しても同様に機能します。

## 参照

[デストラクタでの終了処理イディオム](../../idioms/dtor-finally.md)

RAIIはC++では一般的なパターンです：
[cppreference.com](http://en.cppreference.com/w/cpp/language/raii)、
[wikipedia][wikipedia]。

[wikipedia]: https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization

[スタイルガイドエントリ](https://doc.rust-lang.org/1.0.0/style/ownership/raii.html)
（現在はプレースホルダーのみです）。
