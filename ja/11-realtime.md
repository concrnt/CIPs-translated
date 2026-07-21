# CIP-11 Realtime API

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
    "net.concrnt.core.resolve": "/resource/{uri}",
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

* `type`
  メッセージ種別。購読の開始は `"subscribe"` を指定する。
  後方互換のため、サーバーは `"listen"` を `"subscribe"` の別名として受理しなければなりません (MUST)。
  新規のクライアント実装は `"subscribe"` を使用するべきです (SHOULD)。

* `prefixes`
  購読するリソースのCCURIの配列。
  購読はプレフィックスマッチで行われ、指定したCCURIそのものに加え、そのCCURIを接頭辞とするリソースで発生したイベントも通知される。

`prefixes` にフィード (キー階層) のCCURIを指定することで、そのキー配下に新たに発生したDocumentの追加イベントを購読できます。

複数回subscribeメッセージが送信された場合、サーバーは最新の購読リストを保持し、以前の購読リストは上書きされます。

CCURIに指定するリソースは、そのサーバーが管理しているものでなくてもよい。
外部リソースが要求された場合、サーバーはそのリソースを所有するサーバーのrealtimeエンドポイントに対して代理で購読リクエストを送信し、イベントを中継しなければなりません (MUST)。中継の詳細は4章で述べる。

サーバーは、未知の `type` を持つメッセージを受信した場合、接続を切断せず無視するべきです (SHOULD)。

### 3.2 イベントの受信
サーバーは、購読されたリソースに変更が発生した場合、以下のJSONメッセージをクライアントに送信します。

```json
{
    "type": "created",
    "source": "<CCURI>",
    "uri": "<CCURI>",
    "association": "<CCURI>",
    "documents": {
        "<CCURI>": { ... }
    },
    "timestamp": "2025-11-23T12:34:56Z"
}
```

* `type` (required)
  変更イベントの種類を示す文字列。とりうる値は以下の通りです。
  * `"created"`: 新しいDocumentが作成されたことを示す。
  * `"deleted"`: Documentが削除されたことを示す。
  * `"associated"`: DocumentにAssociation(CIP-9)が作成されたことを示す。
  * `"unassociated"`: DocumentからAssociationが削除されたことを示す。

* `source` (required)
  イベントの配信元となった購読チャンネルのCCURI。クライアントは、このフィールドを見ることで、どの購読プレフィックスに由来するイベントかを判別できる。

* `uri` (required)
  変更対象となったリソースのCCURI。
  `"created"` / `"deleted"` では作成・削除されたDocumentのURI、`"associated"` / `"unassociated"` では関連付けの対象となったDocumentのURIが入る。

* `association` (optional)
  `"associated"` イベントの場合に、作成されたAssociation DocumentのccfsURIが入る。

* `documents` (optional)
  イベントに関連するConcrnt Signed Document(CIP-1)を、そのリソースのCCURIをキーとしたマップで格納する。
  * `"created"`: 作成されたDocument(キーは `uri` と同じ)を含む。
  * `"associated"`: 作成されたAssociation Document(キーは `association` と同じ)を含む。
  * `"deleted"` / `"unassociated"`: 通常このフィールドは省略される。
  documentが保護されているなどの場合において、サーバーはこのフィールドを省略してもよい (MAY)。

* `timestamp` (required)
  イベントの発生時刻。
  `"created"` イベントでは、Documentの `createdAt` の値が入る。
  それ以外のイベントでは、サーバーがイベントを発行した時刻が入る。
