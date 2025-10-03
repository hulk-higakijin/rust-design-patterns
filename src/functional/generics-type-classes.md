# ジェネリクスを型クラスとして使用する

## 説明

Rustの型システムは、命令型言語(JavaやC++など)よりも関数型言語(Haskellなど)のように設計されています。その結果、Rustは多くの種類のプログラミング問題を「静的型付け」の問題に変換できます。これは関数型言語を選択する最大のメリットの1つであり、Rustのコンパイル時保証の多くにとって重要です。

この考え方の重要な部分は、ジェネリック型の動作方法です。例えばC++やJavaでは、ジェネリック型はコンパイラのためのメタプログラミング構造です。C++の`vector<int>`と`vector<char>`は、`vector`型(テンプレートとして知られる)の同じボイラープレートコードを2つの異なる型で埋めた、単なる2つの異なるコピーです。

Rustでは、ジェネリック型パラメータは関数型言語で「型クラス制約」として知られるものを作成し、エンドユーザーによって埋められる各異なるパラメータは*実際に型を変更します*。言い換えれば、`Vec<isize>`と`Vec<char>`は*2つの異なる型*であり、型システムのすべての部分によって区別されます。

これは**単相化(monomorphization)**と呼ばれ、**多相的(polymorphic)**なコードから異なる型が作成されます。この特別な動作では、`impl`ブロックでジェネリックパラメータを指定する必要があります。ジェネリック型の異なる値は異なる型を引き起こし、異なる型は異なる`impl`ブロックを持つことができます。

オブジェクト指向言語では、クラスは親から動作を継承できます。しかし、これにより型クラスの特定のメンバーに追加の動作だけでなく、追加の振る舞いも付加できます。

最も近い同等物は、JavaScriptやPythonの実行時多相性です。これらの言語では、任意のコンストラクタによってオブジェクトに新しいメンバーを自由に追加できます。しかし、これらの言語とは異なり、Rustのすべての追加メソッドは使用時に型チェックできます。なぜなら、ジェネリクスが静的に定義されているからです。これにより、安全性を保ちながらより使いやすくなります。

## 例

一連の研究室マシン用のストレージサーバーを設計しているとします。関連するソフトウェアのため、サポートする必要がある2つの異なるプロトコルがあります: BOOTP(PXEネットワークブート用)とNFS(リモートマウントストレージ用)です。

目標は、Rustで書かれた1つのプログラムで両方を処理できるようにすることです。プロトコルハンドラを持ち、両方の種類のリクエストをリッスンします。メインアプリケーションロジックは、研究室管理者が実際のファイルのストレージとセキュリティコントロールを設定できるようにします。

研究室のマシンからのファイルリクエストには、どのプロトコルから来たかに関係なく、同じ基本情報が含まれています: 認証方法と取得するファイル名です。単純な実装は次のようになります:

```rust,ignore
enum AuthInfo {
    Nfs(crate::nfs::AuthInfo),
    Bootp(crate::bootp::AuthInfo),
}

struct FileDownloadRequest {
    file_name: PathBuf,
    authentication: AuthInfo,
}
```

この設計は十分にうまく機能するかもしれません。しかし今、*プロトコル固有の*メタデータを追加する必要があるとします。例えば、NFSでは、追加のセキュリティルールを適用するためにマウントポイントを特定したいとします。

現在の構造体が設計されている方法では、プロトコルの決定は実行時まで残されます。つまり、一方のプロトコルには適用され他方には適用されないメソッドでは、プログラマが実行時チェックを行う必要があります。

NFSマウントポイントを取得する方法は次のようになります:

```rust,ignore
struct FileDownloadRequest {
    file_name: PathBuf,
    authentication: AuthInfo,
    mount_point: Option<PathBuf>,
}

impl FileDownloadRequest {
    // ... other methods ...

    /// Gets an NFS mount point if this is an NFS request. Otherwise,
    /// return None.
    pub fn mount_point(&self) -> Option<&Path> {
        self.mount_point.as_ref()
    }
}
```

`mount_point()`のすべての呼び出し元は`None`をチェックし、それを処理するコードを書かなければなりません。これは、特定のコードパスでNFSリクエストのみが使用されることを知っている場合でも当てはまります!

異なるリクエストタイプが混同された場合にコンパイル時エラーを引き起こす方が、はるかに最適です。結局のところ、ライブラリから使用する関数を含むユーザーコードの全体のパスは、リクエストがNFSリクエストかBOOTPリクエストかを知っているでしょう。

Rustでは、これは実際に可能です! 解決策は、APIを分割するために*ジェネリック型を追加する*ことです。

これは次のようになります:

```rust
use std::path::{Path, PathBuf};

mod nfs {
    #[derive(Clone)]
    pub(crate) struct AuthInfo(String); // NFS session management omitted
}

mod bootp {
    pub(crate) struct AuthInfo(); // no authentication in bootp
}

// private module, lest outside users invent their own protocol kinds!
mod proto_trait {
    use super::{bootp, nfs};
    use std::path::{Path, PathBuf};

    pub(crate) trait ProtoKind {
        type AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo;
    }

    pub struct Nfs {
        auth: nfs::AuthInfo,
        mount_point: PathBuf,
    }

    impl Nfs {
        pub(crate) fn mount_point(&self) -> &Path {
            &self.mount_point
        }
    }

    impl ProtoKind for Nfs {
        type AuthInfo = nfs::AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo {
            self.auth.clone()
        }
    }

    pub struct Bootp(); // no additional metadata

    impl ProtoKind for Bootp {
        type AuthInfo = bootp::AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo {
            bootp::AuthInfo()
        }
    }
}

use proto_trait::ProtoKind; // keep internal to prevent impls
pub use proto_trait::{Bootp, Nfs}; // re-export so callers can see them

struct FileDownloadRequest<P: ProtoKind> {
    file_name: PathBuf,
    protocol: P,
}

// all common API parts go into a generic impl block
impl<P: ProtoKind> FileDownloadRequest<P> {
    fn file_path(&self) -> &Path {
        &self.file_name
    }

    fn auth_info(&self) -> P::AuthInfo {
        self.protocol.auth_info()
    }
}

// all protocol-specific impls go into their own block
impl FileDownloadRequest<Nfs> {
    fn mount_point(&self) -> &Path {
        self.protocol.mount_point()
    }
}

fn main() {
    // your code here
}
```

このアプローチでは、ユーザーが誤って間違った型を使用した場合:

```rust,ignore
fn main() {
    let mut socket = crate::bootp::listen()?;
    while let Some(request) = socket.next_request()? {
        match request.mount_point().as_ref() {
            "/secure" => socket.send("Access denied"),
            _ => {} // continue on...
        }
        // Rest of the code here
    }
}
```

構文エラーが発生します。型`FileDownloadRequest<Bootp>`は`mount_point()`を実装していません。`FileDownloadRequest<Nfs>`型のみが実装しています。そしてそれはもちろん、BOOTPモジュールではなくNFSモジュールによって作成されます!

## 利点

第一に、複数の状態に共通するフィールドを重複排除できます。共有されないフィールドをジェネリックにすることで、一度実装されます。

第二に、`impl`ブロックが状態ごとに分解されるため、読みやすくなります。すべての状態に共通するメソッドは1つのブロックに一度だけ記述され、1つの状態に固有のメソッドは別のブロックにあります。

これらの両方により、コード行数が減り、より良く整理されます。

## 欠点

現在、これはコンパイラでの単相化の実装方法により、バイナリのサイズを増加させます。将来、実装が改善できることを期待しています。

## 代替案

- 構築または部分的な初期化のために型が「分割API」を必要とするように見える場合は、代わりに[ビルダーパターン](../patterns/creational/builder.md)を検討してください。

- 型間でAPIが変わらず、動作のみが変わる場合は、代わりに[ストラテジーパターン](../patterns/behavioural/strategy.md)を使用する方が良いです。

## 関連項目

このパターンは標準ライブラリ全体で使用されています:

- `Vec<u8>`はStringからキャストできますが、他のすべての型の`Vec<T>`はできません。[^1]
- イテレータはバイナリヒープにキャストできますが、`Ord`トレイトを実装する型を含む場合のみです。[^2]
- `to_string`メソッドは、型`str`の`Cow`にのみ特殊化されました。[^3]

また、API柔軟性を可能にするために、いくつかの人気のあるクレートでも使用されています:

- 組み込みデバイス用に使用される`embedded-hal`エコシステムは、このパターンを広範に使用しています。例えば、組み込みピンを制御するために使用されるデバイスレジスタの設定を静的に検証できます。ピンがモードに設定されると、`Pin<MODE>`構造体を返し、そのジェネリックがそのモードで使用可能な関数を決定します。これらの関数は`Pin`自体にはありません。[^4]

- `hyper` HTTPクライアントライブラリは、異なるプラガブルリクエストのためにリッチなAPIを公開するためにこれを使用しています。異なるコネクタを持つクライアントは、それらに異なるメソッドと異なるトレイト実装を持ちますが、コアメソッドのセットは任意のコネクタに適用されます。[^5]

- 「型状態」パターン -- オブジェクトが内部状態や不変条件に基づいてAPIを獲得・喪失する -- は、同じ基本概念と若干異なる手法を使用してRustで実装されています。[^6]

[^1]: 参照:
[impl From\<CString\> for Vec\<u8\>](https://doc.rust-lang.org/1.59.0/src/std/ffi/c_str.rs.html#803-811)

[^2]: 参照:
[impl\<T: Ord\> FromIterator\<T\> for BinaryHeap\<T\>](https://web.archive.org/web/20201030132806/https://doc.rust-lang.org/stable/src/alloc/collections/binary_heap.rs.html#1330-1335)

[^3]: 参照:
[impl\<'\_\> ToString for Cow\<'\_, str>](https://doc.rust-lang.org/stable/src/alloc/string.rs.html#2235-2240)

[^4]: 例:
[https://docs.rs/stm32f30x-hal/0.1.0/stm32f30x_hal/gpio/gpioa/struct.PA0.html](https://docs.rs/stm32f30x-hal/0.1.0/stm32f30x_hal/gpio/gpioa/struct.PA0.html)

[^5]: 参照:
[https://docs.rs/hyper/0.14.5/hyper/client/struct.Client.html](https://docs.rs/hyper/0.14.5/hyper/client/struct.Client.html)

[^6]: 参照:
[The Case for the Type State Pattern](https://web.archive.org/web/20210325065112/https://www.novatec-gmbh.de/en/blog/the-case-for-the-typestate-pattern-the-typestate-pattern-itself/)
および
[Rusty Typestate Series (an extensive thesis)](https://web.archive.org/web/20210328164854/https://rustype.github.io/notes/notes/rust-typestate-series/rust-typestate-index)
