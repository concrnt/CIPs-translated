# CIP-2: Commit

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

## 3. Commit エンドポイント
Concrnt サーバーは、HTTP POST リクエストを受け付けるエンドポイントを提供する。
これは、CIP-0で定義されるサービスディスカバリにおいて、"net.concrnt.commit" エンドポイント名で広告されなければなりません (MUST)。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.resource": "/resource/{uri}",
    "net.concrnt.commit": "/commit"
  }
}
```

### 3.1 リクエスト形式

クライアントは、Commitエンドポイントに対してCIP-1で定義されたConcrnt Signed DocumentをHTTP POSTリクエストのボディとして送信します。


クライアントは、ドキュメントに含まれる`owner`フィールドで指定されるEntityを管理するConcrntサーバーに対してリクエストを送信しなければなりません (MUST)。
送信されたdocument中のownerが管理外のEntityであった場合、CIP-2はこの挙動を定義しません。

### 3.2 レスポンス形式
サーバーは、リクエストが成功した場合、HTTP 201 Created ステータスコードを返し、レスポンスボディに以下のJSONオブジェクトを含めます。

```json
{
  "cdid": "<生成されたCDID>",
  "uri": "<CCURI>",
  "url": "<リソースのURL>"
}
```

## 4 要素の削除
Concrnt Documentの削除は、`"https://schema.concrnt.net/delete.json"` スキーマを使用して行うことができる。
削除リクエストは、Commitエンドポイントに対してCIP-1で定義されたConcrnt Signed DocumentをHTTP POSTリクエストのボディとして送信します。
削除Documentの`value`フィールドは、削除対象のDocumentをCCURIで指定します。

## 5. セキュリティと認証
サーバーは、Commitエンドポイントへのリクエストが適切に署名されていることを検証し、署名者がリソースの変更を行う権限を持っていることを確認しなければなりません (MUST)。
不正な署名や権限のないリクエストに対しては、HTTP 400 Bad Request または HTTP 403 Forbidden ステータスコードを返します。

