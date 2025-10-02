# 文字列の受け入れ

## 説明

FFIを通じてポインタ経由で文字列を受け入れる場合、以下の2つの原則に従うべきです：

1. 外部の文字列を直接コピーするのではなく、「借用」した状態に保つ。
2. C形式の文字列からRustのネイティブ文字列への変換に関わる複雑さと`unsafe`コードの量を最小限に抑える。

## 動機

Cで使用される文字列は、Rustで使用される文字列とは異なる振る舞いをします。具体的には：

- C文字列はnull終端であるのに対し、Rust文字列は長さを保持する
- C文字列は任意の非ゼロバイトを含むことができるが、Rust文字列はUTF-8でなければならない
- C文字列は`unsafe`なポインタ操作を使用してアクセス・操作されるが、Rust文字列との対話は安全なメソッドを通じて行われる

Rust標準ライブラリには、Rustの`String`と`&str`に相当するC言語用の型として`CString`と`&CStr`が用意されており、これらによってC文字列とRust文字列間の変換に関わる複雑さと`unsafe`コードの多くを回避できます。

`&CStr`型は借用データを扱うことも可能にし、RustとC間の文字列受け渡しがゼロコスト操作になります。

## コード例

```rust,ignore
pub mod unsafe_module {

    // other module content

    /// Log a message at the specified level.
    ///
    /// # Safety
    ///
    /// It is the caller's guarantee to ensure `msg`:
    ///
    /// - is not a null pointer
    /// - points to valid, initialized data
    /// - points to memory ending in a null byte
    /// - won't be mutated for the duration of this function call
    #[no_mangle]
    pub unsafe extern "C" fn mylib_log(msg: *const libc::c_char, level: libc::c_int) {
        let level: crate::LogLevel = match level { /* ... */ };

        // SAFETY: The caller has already guaranteed this is okay (see the
        // `# Safety` section of the doc-comment).
        let msg_str: &str = match std::ffi::CStr::from_ptr(msg).to_str() {
            Ok(s) => s,
            Err(e) => {
                crate::log_error("FFI string conversion failed");
                return;
            }
        };

        crate::log(msg_str, level);
    }
}
```

## 利点

この例は以下を保証するように書かれています：

1. `unsafe`ブロックができるだけ小さい。
2. 「追跡されていない」ライフタイムを持つポインタが「追跡された」共有参照になる

文字列が実際にコピーされる代替案を考えてみましょう：

```rust,ignore
pub mod unsafe_module {

    // other module content

    pub extern "C" fn mylib_log(msg: *const libc::c_char, level: libc::c_int) {
        // DO NOT USE THIS CODE.
        // IT IS UGLY, VERBOSE, AND CONTAINS A SUBTLE BUG.

        let level: crate::LogLevel = match level { /* ... */ };

        let msg_len = unsafe { /* SAFETY: strlen is what it is, I guess? */
            libc::strlen(msg)
        };

        let mut msg_data = Vec::with_capacity(msg_len + 1);

        let msg_cstr: std::ffi::CString = unsafe {
            // SAFETY: copying from a foreign pointer expected to live
            // for the entire stack frame into owned memory
            std::ptr::copy_nonoverlapping(msg, msg_data.as_mut(), msg_len);

            msg_data.set_len(msg_len + 1);

            std::ffi::CString::from_vec_with_nul(msg_data).unwrap()
        }

        let msg_str: String = unsafe {
            match msg_cstr.into_string() {
                Ok(s) => s,
                Err(e) => {
                    crate::log_error("FFI string conversion failed");
                    return;
                }
            }
        };

        crate::log(&msg_str, level);
    }
}
```

このコードは次の2つの点でオリジナルより劣っています：

1. `unsafe`コードがはるかに多く、さらに重要なことに、維持しなければならない不変条件が増える。
2. 広範な算術演算が必要なため、このバージョンにはRustの`未定義動作`を引き起こすバグがある。

ここでのバグは、ポインタ演算における単純なミスです：文字列はコピーされましたが、その`msg_len`バイト全てです。しかし、末尾の`NUL`終端子はコピーされませんでした。

その後、Vectorのサイズは*ゼロパディングされた文字列*の長さに*設定*されました――末尾にゼロを追加できたはずの*リサイズ*ではなく。結果として、Vector内の最後のバイトは初期化されていないメモリになります。ブロックの最後で`CString`が作成されるとき、Vectorの読み取りが`未定義動作`を引き起こします！

このような問題の多くと同様に、これは追跡が困難な問題です。文字列が`UTF-8`でないためにパニックすることもあれば、文字列の末尾に奇妙な文字が入ることもあれば、完全にクラッシュすることもあります。

## 欠点

なし？
