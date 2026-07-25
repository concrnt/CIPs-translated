# CIP-11: Realtime API

## 0. Abstract

本仕様は、Concrnt サーバーのエンドポイントを拡張し、サーバーで発生したリソースの変更イベントを
リアルタイムにクライアントへ通知する手段を提供する。

## 1. Status of This Memo

このドキュメントは Realtime API の仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0, CIP-1 を前提とする。イベントの redaction には
CIP-12 を必要とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Realtime エンドポイント

本 CIP を実装するサーバーは、WebSocket [RFC6455] へアップグレード可能な HTTP GET エンドポイントを
提供し、CIP-0 のサービスディスカバリにおいて `net.concrnt.core.realtime` エンドポイント名で
広告しなければならない (MUST)。

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

### 3.1 購読の開始

クライアントは、Realtime エンドポイントに対して WebSocket ハンドシェイクを行い、
接続確立後に以下の JSON メッセージを送信して、リソースの変更イベントの購読を開始する。

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
  サーバーは `"listen"` を `"subscribe"` の別名として受理してもよい (MAY。旧仕様との互換のため)。
  このほか、`"h"` はハートビートを表すメッセージ種別であり、サーバーは何も行わず受理する。

* `prefixes`
  購読するリソースの CCURI の配列。

購読のマッチングは、イベントの `source` (§3.2) の CCURI 文字列に対する**単純な前方一致**とする (MUST)。
指定した CCURI そのものに加え、それを接頭辞とするリソースで発生したイベントも通知される
(CIP-5 §3.1 の `prefix` と同じ規則であり、`item` の購読には兄弟キー `item2` もマッチする点に注意)。
prefix 中の `*` `?` `[` 等の文字をワイルドカードとして解釈してはならない (MUST NOT)。

各 prefix は owner 部を持つ有効な CCURI でなければならない (MUST)。owner 部を持たない prefix
(空文字列や `cckv://` 等) は、サーバー全体のイベントの購読を許すため、無視しなければならない (MUST)。
サーバーは、1 接続あたりの prefix 数に上限を設けるべきである (SHOULD)。

複数回 subscribe メッセージが送信された場合、サーバーは最新の購読リストのみを保持する
(購読リストの全置換)。明示的な unsubscribe メッセージは存在せず、購読の解除は縮小した購読リストに
よる subscribe の再送によって行う。

サーバーは、未知の `type` を持つメッセージを受信した場合、接続を切断せず無視するべきである (SHOULD)。
サーバーは接続の生存確認に WebSocket の Ping フレーム (RFC6455) を用いるべきであり (SHOULD)、
アプリケーションレベルの `"h"` メッセージの受信を接続維持の条件としてはならない (MUST NOT)。

#### 3.1.1 外部リソースの購読 (中継)

`prefixes` に指定するリソースは、そのサーバーが管理しているものでなくてもよい。
外部リソースが要求された場合、サーバーはそのリソースの権威サーバーの Realtime エンドポイントへ
代理で購読を行い、イベントを中継してもよい (MAY)。
中継を行わないサーバーは、管理外の prefix に対するイベントを配信しない
(購読メッセージ自体はエラーとしない)。クライアントは、確実に受信するためには
対象リソースの権威サーバーへ直接接続することができる。

中継を行う場合、以下に従う。

* 上流接続の宛先が、プライベート・ループバック・リンクローカルアドレスへ解決される場合、
  接続してはならない (MUST NOT)。
* 1 接続あたり、およびサーバー全体で確立する上流接続数に上限を設けなければならない (MUST)。
* 中継されたイベントを、それを受信した上流接続へ再送してはならない (MUST NOT)。
  中継のホップ数に上限を設けるべきである (SHOULD)。

サーバー間中継の詳細プロトコル (上流の多重化、購読の張り替え、再接続方式等) は本仕様では
規定せず、将来の CIP に委ねる。

### 3.2 イベントの受信

サーバーは、購読されたリソースに変更が発生した場合、以下の JSON メッセージをクライアントに送信する。

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
  変更イベントの種類を示す文字列。とりうる値は以下の通り。
  * `"created"`: 新しい Document が作成されたことを示す。既存キーの上書き (CIP-3 §3.4) の場合も
    `"created"` を発行する (Document の作成を示すのであり、キーの新規性を示すものではない)。
  * `"deleted"`: Document が削除されたことを示す。
  * `"associated"`: Document に Association (CIP-9) が作成されたことを示す。
  * `"unassociated"`: Document から Association が削除されたことを示す。

  クライアントは、未知の `type` を持つメッセージを、接続を切断せず無視しなければならない (MUST)。

* `source` (required)
  イベントが発行されたチャンネル (変更が発生したリソース側) の CCURI を設定しなければならない (MUST)。
  購読プレフィックスの値そのものを設定してはならない (MUST NOT)。
  クライアントは、自身の購読プレフィックスと `source` の前方一致により、由来する購読を判別する。

* `uri` (required)
  変更対象となったリソースの CCURI。
  `"created"` / `"deleted"` では作成・削除された Document の URI、
  `"associated"` / `"unassociated"` では関連付けの対象となった Document の URI が入る。

* `association` (OPTIONAL)
  `"associated"` / `"unassociated"` イベントにおいて、作成・削除された Association Document の
  ccfs URI を設定するべきである (SHOULD)。

* `documents` (OPTIONAL)
  イベントに関連する Concrnt Signed Document (CIP-1) を、そのリソースの CCURI をキーとした
  マップで格納する。
  * `"created"`: 作成された Document (キーは `uri` と同じ) を含む。
  * `"associated"`: 作成された Association Document (キーは `association` と同じ) を含む。
  * `"deleted"` / `"unassociated"`: 通常このフィールドは省略される。

  **保護された Document の編集 (redaction)**: サーバーは、イベントの送出時に、対象 Document の
  読み取りアクション (`record:read` / `association:read`, CIP-12) を**ゲスト (匿名) リクエスタ**で
  評価し、匿名での読み取りが許可されない Document については `documents` フィールドから
  その内容を省略しなければならない (MUST)。この場合もイベント自体
  (`type` / `source` / `uri` / `timestamp` 等) は配送される。
  同梱される Signed Document がネストされた `references` (CIP-1) を持つ場合
  (例: 配布 Reference Document に内包された元 Document)、その各エントリにも同じ
  匿名 read 評価を適用し、許可されないエントリは省略しなければならない (MUST)。
  さらに深い階層のネストは一律に省略してよい (MAY)。参考実装は省略する。
  クライアントは、内容が省略されたイベントを受信した場合、必要に応じて自身の認証情報を
  用いた通常の Resolve / Query (CIP-0, CIP-5) で Document を取得する。

* `timestamp` (required)
  イベントの発生時刻。`"created"` イベントでは Document の `createdAt` の値が入る。
  それ以外のイベントでは、サーバーがイベントを発行した時刻が入る。

## 4. Security Considerations

* Realtime API の配信内容は**匿名で閲覧可能な情報**に限定される。購読自体には認証を要求しない
  代わりに、匿名リクエスタで読み取りが許可されない Document の内容は §3.2 の redaction 規則により
  イベントから省略されなければならない (MUST)。これにより、購読者ごとの認可評価や
  サーバー間中継時のアイデンティティの伝搬を必要とせず、保護された Document の内容が
  未認可の購読者 (および中継先) へ漏えいすることを防ぐ。
* イベントのメタデータ (`uri` 等のキー情報・イベントの発生事実) は保護の対象外である。
  購読は前方一致 (§3.1) であるため、キー名そのものが機微である場合は保護されない。
  リソースの存在自体を秘匿する必要がある用途では、Realtime API によるイベント配信を
  行うべきではない (SHOULD NOT)。
* イベントは配信元サーバーが構成する**助言的な通知**であり、それ自体は暗号学的な証明を伴わない。
  クライアントは、イベントのみを根拠にリソースの状態を確定せず、必要に応じて対象を resolve で
  確認すること。特に `deleted` / `unassociated` イベントは、伝搬削除の受信処理 (CIP-4 §6.1) の
  性質上、実際には削除されていない対象について発行されうる。
* 中継 (§3.1.1) は匿名クライアントの指定に基づく外向き接続であり、SSRF・増幅・ループの
  温床となる。中継を行うサーバーは §3.1.1 の防御を省略してはならない。

## 5. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 6455 – The WebSocket Protocol
* CIP-12 – Policy (record:read / association:read)
