# エラー時に消費された引数を返す

## 説明

失敗する可能性のある関数が引数を消費（ムーブ）する場合、エラー内でその引数を返します。

## 例

```rust
pub fn send(value: String) -> Result<(), SendError> {
    println!("using {value} in a meaningful way");
    // Simulate non-deterministic fallible action.
    use std::time::SystemTime;
    let period = SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap();
    if period.subsec_nanos() % 2 == 1 {
        Ok(())
    } else {
        Err(SendError(value))
    }
}

pub struct SendError(String);

fn main() {
    let mut value = "imagine this is very long string".to_string();

    let success = 's: {
        // Try to send value two times.
        for _ in 0..2 {
            value = match send(value) {
                Ok(()) => break 's true,
                Err(SendError(value)) => value,
            }
        }
        false
    };

    println!("success: {success}");
}
```

## 動機

エラーが発生した場合、代替手段を試したり、非決定的な関数の場合は操作を再試行したりすることがあります。しかし、引数が常に消費される場合、呼び出しのたびにクローンを作成する必要があり、あまり効率的ではありません。

標準ライブラリでは、例えば`String::from_utf8`メソッドでこのアプローチを使用しています。有効なUTF-8を含まないベクタが与えられた場合、`FromUtf8Error`が返されます。`FromUtf8Error::into_bytes`メソッドを使用して元のベクタを取り戻すことができます。

## 利点

可能な限り引数をムーブすることによる、より良いパフォーマンス。

## 欠点

エラー型がわずかに複雑になります。
