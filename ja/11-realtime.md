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
    "type": "subscribe",
    "prefixes": [
        "<CCURI>",
        "<CCURI>"
    ]
}
```

* `type`
  メッセージ種別。購読の開始は `"subscribe"` を指定する。
  後方互換のため、サーバーは `"listen"` を `"subscribe"` の別名として受理しなければなりません (MUST)。
  新規のクライアント実装は `"subscribe"` を使用するべきです (SHOULD)。
  このほか、`"h"` はハートビートを表すメッセージ種別であり、サーバーは何も行わず受理します。

* `prefixes`
  購読するリソースのCCURIの配列。
  購読はプレフィックスマッチで行われ、指定したCCURIそのものに加え、そのCCURIを接頭辞とするリソースで発生したイベントも通知される。

`prefixes` にフィード (キー階層) のCCURIを指定することで、そのキー配下に新たに発生したDocumentの追加イベントを購読できます。

複数回subscribeメッセージが送信された場合、サーバーは最新の購読リストを保持し、以前の購読リストは上書きされます (購読リストの全置換)。
明示的なunsubscribeメッセージは存在せず、購読の解除は縮小した購読リストによるsubscribeの再送によって行います。

CCURIに指定するリソースは、そのサーバーが管理しているものでなくてもよい。
外部リソースが要求された場合、サーバーはそのリソースを所有するサーバーのrealtimeエンドポイントに対して代理で購読リクエストを送信し、イベントを中継しなければなりません (MUST)。

> **編集ノート:** サーバー間中継の詳細プロトコル (上流接続の多重化、購読の張り替え、再接続方式、
> 中継イベントの `source` の扱い等) は現時点で未規定である。

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

  **保護されたDocumentの編集 (redaction)**: サーバーは、イベントの送出時に、対象Documentの
  読み取りアクション (`record:read` / `association:read`, CIP-12) を**ゲスト(匿名)リクエスタ**で
  評価し、匿名での読み取りが許可されないDocumentについては `documents` フィールドから
  その内容を省略しなければならない (MUST)。この場合もイベント自体
  (`type` / `source` / `uri` / `timestamp` 等) は配送される。
  同梱されるSigned Documentがネストされた `references` (CIP-1) を持つ場合
  (例: 配布Reference Documentに内包された元Document)、その各エントリにも同じ
  匿名read評価を適用し、許可されないエントリは省略しなければならない (MUST)。
  さらに深い階層のネストは一律に省略してよい (MAY)。参考実装は省略する。
  クライアントは、内容が省略されたイベントを受信した場合、必要に応じて自身の認証情報を
  用いた通常のResolve/Query (CIP-0, CIP-5) でDocumentを取得する。

* `timestamp` (required)
  イベントの発生時刻。
  `"created"` イベントでは、Documentの `createdAt` の値が入る。
  それ以外のイベントでは、サーバーがイベントを発行した時刻が入る。

## 4. Security Considerations

* Realtime APIの配信内容は**匿名で閲覧可能な情報**に限定される。購読自体には認証を要求しない
  代わりに、匿名リクエスタで読み取りが許可されないDocumentの内容は §3.2 のredaction規則により
  イベントから省略されなければならない (MUST)。これにより、購読者ごとの認可評価や
  サーバー間中継時のアイデンティティの伝搬を必要とせず、保護されたDocumentの内容が
  未認可の購読者(および中継先)へ漏えいすることを防ぐ。
* イベントのメタデータ (`uri` 等のキー情報・イベントの発生事実) は保護の対象外である。
  リソースの存在自体を秘匿する必要がある用途では、Realtime APIによるイベント配信を
  行うべきではない (SHOULD NOT)。
* イベントは配信元サーバーが構成する**助言的な通知**であり、それ自体は暗号学的な証明を
  伴わない。クライアントは、イベントのみを根拠にリソースの状態を確定せず、必要に応じて
  対象をresolveで確認すること。特に `deleted` / `unassociated` イベントは、伝搬削除の
  受信処理 (CIP-4 §6.1) の性質上、実際には削除されていない対象について発行されうる。

## 5. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 6455 – The WebSocket Protocol
