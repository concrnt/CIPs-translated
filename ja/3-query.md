# CIP-3: Query

## 0. Abstract
本ドキュメントでは、Concrntをホストするサーバーが提供するエンドポイントを拡張し、サーバーが管理するリソースを変更するための手段を提供する。

## 1. Status of This Memo

このドキュメントは Concrnt Document フォーマットの仕様を定義する。

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、
実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Query エンドポイント
Concrnt サーバーは、HTTP GET リクエストを受け付けるエンドポイントを提供する。
これは、CIP-0で定義されるサービスディスカバリにおいて、"net.concrnt.core.query" エンドポイント名で広告されなければなりません (MUST)。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.resolve": "/resource/{uri}",
    "net.concrnt.core.query": "/query"
  }
}
```

### 3.1 リクエスト形式

"net.concrnt.core.query" エンドポイントは、以下のクエリパラメータを受け付ける。
- `prefix` 検索対象のキーの接頭辞を指定する文字列。 (必須)
- `schema` スキーマを指定する文字列。 (任意)
- `since` 指定されたタイムスタンプ以降に作成されたリソースのみを返すための ISO 8601 形式の日時文字列。 (任意)
- `until` 指定されたタイムスタンプ以前に作成されたリソースのみを返すための ISO 8601 形式の日時文字列。 (任意)
- `limit` 返されるリソースの最大数を指定する整数。デフォルトは 10、最大値は 100。 (任意)
- `order` ソート順序を指定する文字列。`asc` (昇順) または `desc` (降順)。デフォルトは `asc`。 (任意)

### 3.2 レスポンス形式

サーバーは、リクエストに一致するリソースの一覧を JSON 配列として返す。
返却するリソースは、`order` パラメータで指定された順序でソートされなければならない (MUST)。


