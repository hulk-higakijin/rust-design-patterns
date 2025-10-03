# 独立した借用のための構造体の分解

## 説明

大きな構造体は、借用チェッカーで問題を引き起こすことがあります。フィールドは独立して借用できますが、構造体全体が一度に使用されてしまい、他の使用を妨げることがあります。解決策として、構造体をいくつかの小さな構造体に分解することが考えられます。その後、これらを元の構造体に合成します。こうすることで、各構造体を個別に借用でき、より柔軟な動作が可能になります。

このパターンは、他の面でもより良い設計につながることが多いです。この設計パターンを適用すると、より小さな機能単位が明らかになることがよくあります。

## 例

借用チェッカーが構造体の使用計画を阻む不自然な例を示します:

```rust,ignore
struct Database {
    connection_string: String,
    timeout: u32,
    pool_size: u32,
}

fn print_database(database: &Database) {
    println!("Connection string: {}", database.connection_string);
    println!("Timeout: {}", database.timeout);
    println!("Pool size: {}", database.pool_size);
}

fn main() {
    let mut db = Database {
        connection_string: "initial string".to_string(),
        timeout: 30,
        pool_size: 100,
    };

    let connection_string = &mut db.connection_string;
    print_database(&db);
    *connection_string = "new string".to_string();
}
```

コンパイラは次のエラーを出力します:

```ignore
let connection_string = &mut db.connection_string;
                        ------------------------- mutable borrow occurs here
print_database(&db);
               ^^^ immutable borrow occurs here
*connection_string = "new string".to_string();
------------------ mutable borrow later used here
```

この設計パターンを適用して、`Database` を3つの小さな構造体にリファクタリングすることで、借用チェックの問題を解決できます:

```rust
// Database は現在、ConnectionString、Timeout、PoolSize の3つの構造体で構成されています。
// より小さな構造体に分解しましょう
#[derive(Debug, Clone)]
struct ConnectionString(String);

#[derive(Debug, Clone, Copy)]
struct Timeout(u32);

#[derive(Debug, Clone, Copy)]
struct PoolSize(u32);

// 次に、これらの小さな構造体を `Database` に合成します
struct Database {
    connection_string: ConnectionString,
    timeout: Timeout,
    pool_size: PoolSize,
}

// print_database は、ConnectionString、Timeout、Poolsize 構造体を受け取るようになります
fn print_database(connection_str: ConnectionString, timeout: Timeout, pool_size: PoolSize) {
    println!("Connection string: {connection_str:?}");
    println!("Timeout: {timeout:?}");
    println!("Pool size: {pool_size:?}");
}

fn main() {
    // 3つの構造体で Database を初期化します
    let mut db = Database {
        connection_string: ConnectionString("localhost".to_string()),
        timeout: Timeout(30),
        pool_size: PoolSize(100),
    };

    let connection_string = &mut db.connection_string;
    print_database(connection_string.clone(), db.timeout, db.pool_size);
    *connection_string = ConnectionString("new string".to_string());
}
```

## 動機

このパターンは、独立して借用したい多くのフィールドを持つ構造体がある場合に最も有用です。結果として、より柔軟な動作が得られます。

## 利点

構造体の分解により、借用チェッカーの制限を回避できます。また、多くの場合、より良い設計が生まれます。

## 欠点

コードが冗長になる可能性があります。また、小さな構造体が良い抽象化でない場合もあり、結果として設計が悪化することがあります。これはおそらく「コードの臭い」であり、プログラムを何らかの方法でリファクタリングすべきことを示しています。

## 議論

このパターンは、借用チェッカーを持たない言語では必要ないため、その意味で Rust に固有のものです。しかし、より小さな機能単位を作ることは、多くの場合、よりクリーンなコードにつながります。これは、言語に依存しない、広く認められたソフトウェアエンジニアリングの原則です。

このパターンは、Rust の借用チェッカーがフィールドを互いに独立して借用できることに依存しています。この例では、借用チェッカーは `a.b` と `a.c` が別々のものであり、独立して借用できることを認識しています。`a` 全体を借用しようとはしないため、このパターンは有用です。
