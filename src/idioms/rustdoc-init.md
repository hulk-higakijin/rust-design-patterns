# 簡単なドキュメント初期化

## 説明

ドキュメントを書く際に構造体の初期化に大きな労力がかかる場合、構造体を引数として受け取るヘルパー関数で例をラップすると、より迅速に記述できます。

## 動機

複数または複雑なパラメータといくつかのメソッドを持つ構造体が存在することがあります。これらの各メソッドには例が必要です。

例えば：

````rust,ignore
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// Sends a request over the connection.
    ///
    /// # Example
    /// ```no_run
    /// # // Boilerplate are required to get an example working.
    /// # let stream = TcpStream::connect("127.0.0.1:34254");
    /// # let connection = Connection { name: "foo".to_owned(), stream };
    /// # let request = Request::new("RequestId", RequestType::Get, "payload");
    /// let response = connection.send_request(request);
    /// assert!(response.is_ok());
    /// ```
    fn send_request(&self, request: Request) -> Result<Status, SendErr> {
        // ...
    }

    /// Oh no, all that boilerplate needs to be repeated here!
    fn check_status(&self) -> Status {
        // ...
    }
}
````

## 例

`Connection` と `Request` を作成するためのこのような定型文を全て記述する代わりに、それらを引数として受け取るラッピングヘルパー関数を作成する方が簡単です：

````rust,ignore
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// Sends a request over the connection.
    ///
    /// # Example
    /// ```
    /// # fn call_send(connection: Connection, request: Request) {
    /// let response = connection.send_request(request);
    /// assert!(response.is_ok());
    /// # }
    /// ```
    fn send_request(&self, request: Request) {
        // ...
    }
}
````

**注意** 上記の例では、`assert!(response.is_ok());` という行は、呼び出されることのない関数の内部にあるため、テスト中に実際には実行されません。

## 利点

これははるかに簡潔で、例の中の繰り返しコードを避けることができます。

## 欠点

例が関数内にあるため、コードはテストされません。ただし、`cargo test` の実行時にコンパイルできることは確認されます。そのため、このパターンは `no_run` が必要な場合に最も有用です。これを使用すれば、`no_run` を追加する必要はありません。

## 議論

アサーションが必要ない場合、このパターンはうまく機能します。

アサーションが必要な場合、代替案として `#[doc(hidden)]` で注釈されたヘルパーインスタンスを作成する公開メソッドを作成することができます（ユーザーには表示されません）。このメソッドはクレートの公開 API の一部であるため、rustdoc 内で呼び出すことができます。
