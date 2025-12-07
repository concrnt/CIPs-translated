# CIP-7 Realtime API

## 0. Abstract
本仕様は、Concrntをホストするサーバーが提供するエンドポイントを拡張し、サーバーで発生したリソースの変更イベントをリアルタイムにクライアントに通知する手段を提供する。

## 1. Status of This Memo

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、
実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Realtime エンドポイント

Concrnt サーバーは、HTTP GET リクエストを受け付けるエンドポイントを提供する。
これは、CIP-0で定義されるサービスディスカバリにおいて、"net.concrnt.core.realtime" エンドポイント名で広告されなければなりません (MUST)。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.resource": "/resource/{uri}",
    "net.concrnt.core.realtime": "/realtime"
  }
}
```

realtimeエンドポイントは、websocketプロトコルを使用してクライアントとサーバー間の双方向通信を確立します。


## 3.1 購読の開始
クライアントは、realtimeエンドポイントに対してHTTP GETリクエストを送信し、WebSocketハンドシェイクを開始します。

接続が確立された後、クライアントは以下のJSONメッセージをサーバーに送信して、特定のリソースの変更イベントの購読を開始します。

```json
{
    "action": "subscribe",
    "resources": [
        "<CCURI>",
        "<CCURI>",
        ...
    ]
}
```

uriにcollection及びtimelineリソースを指定することで、そのリソース配下に新たに発生したDocumentの追加イベントを購読できます。
複数回Susbcribeリクエストが送信された場合、サーバーは最新の購読リストを保持し、以前の購読リストは上書きされます。

CCURIに指定するリソースは、そのサーバーが管理しているものでなくてもよい。
外部リソースが要求された場合、サーバーはそのリソースを所有するサーバーのrealtimeエンドポイントに対して代理で購読リクエストを送信し、イベントを中継しなければなりません (MUST)。

## 3.2 イベントの受信
サーバーは、購読されたリソースに変更が発生した場合、以下のJSONメッセージをクライアントに送信します。

```json
{
    "event": "created",
    "source": "<CCURI>", // 変更が発生したリソースのCCURI
    "resource": "<CCURI>", // 変更されたリソースのCCURI
    "signed_document": { ... } // CIP-1で定義されたConcrnt Signed Document
}
```

event: 変更イベントの種類を示す文字列。
とりうる値は以下の通りです。
* "created": 新しいDocumentが作成されたことを示す。
* "deleted": Documentが削除されたことを示す。

source: 変更が発生したリソースのCCURI。
resource: 変更されたリソースのCCURI。
signed_document: CIP-1で定義されたConcrnt Signed Document。
但し、documentが保護されているなどの場合において、このフィールドを省略してもよい (MAY)。

