# デザイン原則

## 一般的なデザイン原則の概要

---

## [SOLID](https://en.wikipedia.org/wiki/SOLID)

- [単一責任の原則 (SRP)](https://en.wikipedia.org/wiki/Single-responsibility_principle):
  クラスは単一の責任のみを持つべきです。つまり、ソフトウェア仕様の一部への変更のみが、そのクラスの仕様に影響を与えるべきです。
- [オープン・クローズドの原則 (OCP)](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle):
  「ソフトウェアエンティティは拡張に対して開いており、修正に対して閉じているべきです。」
- [リスコフの置換原則 (LSP)](https://en.wikipedia.org/wiki/Liskov_substitution_principle):
  「プログラム内のオブジェクトは、そのサブタイプのインスタンスで置き換え可能であり、プログラムの正確性を変更しないべきです。」
- [インターフェース分離の原則 (ISP)](https://en.wikipedia.org/wiki/Interface_segregation_principle):
  「多くのクライアント固有のインターフェースは、1つの汎用インターフェースよりも優れています。」
- [依存性逆転の原則 (DIP)](https://en.wikipedia.org/wiki/Dependency_inversion_principle):
  「具象ではなく、抽象に依存すべきです。」

## [CRP（複合再利用の原則）または継承よりコンポジション](https://en.wikipedia.org/wiki/Composition_over_inheritance)

「クラスは、基底クラスや親クラスからの継承よりも、コンポジション（望ましい機能を実装する他のクラスのインスタンスを含めること）によって、ポリモーフィックな動作とコードの再利用を優先すべきであるという原則」 -
Knoernschild, Kirk (2002). Java Design - Objects, UML, and Process

## [DRY (Don't Repeat Yourself)](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)

「すべての知識は、システム内で単一、明確、権威ある表現を持たなければならない」

## [KISS原則](https://en.wikipedia.org/wiki/KISS_principle)

ほとんどのシステムは、複雑にするよりもシンプルに保つことで最もうまく機能します。したがって、シンプルさは設計における主要な目標であるべきであり、不要な複雑さは避けるべきです。

## [デメテルの法則 (LoD)](https://en.wikipedia.org/wiki/Law_of_Demeter)

特定のオブジェクトは、「情報隠蔽」の原則に従って、他のもの（そのサブコンポーネントを含む）の構造やプロパティについて、可能な限り少ない仮定をすべきです。

## [契約による設計 (DbC)](https://en.wikipedia.org/wiki/Design_by_contract)

ソフトウェア設計者は、ソフトウェアコンポーネントの正式で正確かつ検証可能なインターフェース仕様を定義すべきです。これは、抽象データ型の通常の定義を、事前条件、事後条件、不変条件で拡張したものです。

## [カプセル化](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming))

データと、そのデータを操作するメソッドとのバンドリング、またはオブジェクトのコンポーネントの一部への直接アクセスの制限。カプセル化は、構造化されたデータオブジェクトの値や状態をクラス内に隠し、権限のない者による直接アクセスを防ぐために使用されます。

## [コマンド・クエリ分離 (CQS)](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)

「関数は抽象的な副作用を生成すべきではない...コマンド（手続き）のみが副作用を生成することが許される。」 - Bertrand Meyer: Object-Oriented
Software Construction

## [最小驚きの原則 (POLA)](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)

システムのコンポーネントは、ほとんどのユーザーが期待する方法で動作すべきです。その動作はユーザーを驚かせたり、当惑させたりすべきではありません。

## 言語的モジュール単位

「モジュールは、使用される言語の構文単位に対応しなければならない。」 - Bertrand
Meyer: Object-Oriented Software Construction

## 自己文書化

「モジュールの設計者は、モジュールに関するすべての情報をモジュール自体の一部にするよう努めるべきです。」 - Bertrand Meyer: Object-Oriented Software
Construction

## 統一アクセス

「モジュールが提供するすべてのサービスは、それらがストレージを通じて実装されているか、計算を通じて実装されているかを明かさない統一された表記法を通じて利用可能であるべきです。」 - Bertrand Meyer: Object-Oriented Software Construction

## 単一選択

「ソフトウェアシステムが一連の選択肢をサポートする必要がある場合、システム内の1つのモジュールのみが、それらの完全なリストを知っているべきです。」 - Bertrand Meyer:
Object-Oriented Software Construction

## 永続性クロージャ

「ストレージメカニズムがオブジェクトを格納するときは必ず、そのオブジェクトの依存物も一緒に格納しなければなりません。検索メカニズムが以前に格納されたオブジェクトを取り出すときは必ず、まだ取り出されていないそのオブジェクトの依存物も取り出さなければなりません。」 - Bertrand Meyer: Object-Oriented Software Construction
