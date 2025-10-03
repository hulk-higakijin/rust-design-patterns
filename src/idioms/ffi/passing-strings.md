# 文字列の受け渡し

## 説明

FFI関数に文字列を渡す際には、以下の4つの原則に従うべきです:

1. 所有する文字列の寿命をできるだけ長くする。
2. 変換時の`unsafe`コードを最小化する。
3. Cコードが文字列データを変更する可能性がある場合は、`CString`の代わりに`Vec`を使用する。
4. Foreign Function APIが必要としない限り、文字列の所有権を呼び出し先に移譲すべきではない。

## 動機

Rustには、`CString`と`CStr`型によるC形式文字列への組み込みサポートがあります。しかし、Rust関数から外部関数呼び出しに送信される文字列には、さまざまなアプローチが存在します。

ベストプラクティスはシンプルです:`unsafe`コードを最小化する方法で`CString`を使用することです。ただし、副次的な注意点として、*オブジェクトは十分に長く生存しなければならない*ということがあります。つまり、ライフタイムを最大化する必要があります。さらに、ドキュメントでは、変更後に`CString`を「往復」させることは未定義動作であると説明されているため、その場合には追加の作業が必要です。

## コード例

```rust,ignore
pub mod unsafe_module {

    // other module content

    extern "C" {
        fn seterr(message: *const libc::c_char);
        fn geterr(buffer: *mut libc::c_char, size: libc::c_int) -> libc::c_int;
    }

    fn report_error_to_ffi<S: Into<String>>(err: S) -> Result<(), std::ffi::NulError> {
        let c_err = std::ffi::CString::new(err.into())?;

        unsafe {
            // SAFETY: calling an FFI whose documentation says the pointer is
            // const, so no modification should occur
            seterr(c_err.as_ptr());
        }

        Ok(())
        // The lifetime of c_err continues until here
    }

    fn get_error_from_ffi() -> Result<String, std::ffi::IntoStringError> {
        let mut buffer = vec![0u8; 1024];
        unsafe {
            // SAFETY: calling an FFI whose documentation implies
            // that the input need only live as long as the call
            let written: usize = geterr(buffer.as_mut_ptr(), 1023).into();

            buffer.truncate(written + 1);
        }

        std::ffi::CString::new(buffer).unwrap().into_string()
    }
}
```

## 利点

この例は以下を確実にするように書かれています:

1. `unsafe`ブロックを可能な限り小さくする。
2. `CString`が十分に長く生存する。
3. 型キャストのエラーは可能な限り常に伝播される。

よくある間違い(ドキュメントにも記載されているほど一般的)は、最初のブロックで変数を使用しないことです:

```rust,ignore
pub mod unsafe_module {

    // other module content

    fn report_error<S: Into<String>>(err: S) -> Result<(), std::ffi::NulError> {
        unsafe {
            // SAFETY: whoops, this contains a dangling pointer!
            seterr(std::ffi::CString::new(err.into())?.as_ptr());
        }
        Ok(())
    }
}
```

このコードはダングリングポインタを生成します。なぜなら、参照が作成された場合とは異なり、`CString`のライフタイムはポインタの作成によって延長されないためです。

もう一つ頻繁に提起される問題は、1kのゼロベクタの初期化が「遅い」というものです。しかし、Rustの最近のバージョンでは、実際にその特定のマクロが`zmalloc`への呼び出しに最適化されており、これはオペレーティングシステムがゼロ化されたメモリを返す能力と同じ速さ(非常に高速)であることを意味します。

## 欠点

なし?
