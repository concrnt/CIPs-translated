# CIP-3: Commit

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
これは、CIP-0で定義されるサービスディスカバリにおいて、"net.concrnt.core.commit" エンドポイント名で広告されなければなりません (MUST)。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.resolve": "/resource/{uri}",
    "net.concrnt.core.commit": "/commit"
  }
}
```

### 3.1 リクエスト形式

クライアントは、Commitエンドポイントに対してCIP-1で定義されたConcrnt Signed DocumentをHTTP POSTリクエストのボディとして送信します。

クライアントは、コミット対象を管理するConcrntサーバーに対してリクエストを送信しなければならない (MUST)。
コミット対象を管理するサーバーは、Documentの内容から次のように導出される。

* `key` フィールドが存在する場合: `key` のCCURIの owner 部が示すEntityの所属サーバ
* `associate` フィールドが存在する場合 (CIP-9, CIP-10): `associate` のCCURIの owner 部が示すEntityの所属サーバ

owner 部から所属サーバへの解決は、CIP-0で定義される名前解決 (Entity documentの `domain`) によって行う。
コミット対象が受信サーバの管理外である場合の挙動は、CIP-3では定義しない。

### 3.2 Document の種別 (kind)

Commitエンドポイントは、CIP-1で定義される `kind` フィールドによってDocumentの処理を分岐する。
受け付ける `kind` は以下の6種であり、それ以外の値を持つDocumentは拒否しなければならない (MUST)。

| kind | 意味 | 定義 |
|---|---|---|
| `entity` | Entity document の登録・更新 | CIP-0 |
| `record` | 一般のDocumentの作成・更新 | CIP-1 |
| `association` | Documentへの関連付け | CIP-9 |
| `ack` | Entityの承認 | CIP-10 |
| `unack` | Entity承認の取消 | CIP-10 |
| `delete` | 要素の削除 | CIP-4 |

### 3.3 レスポンス形式

サーバーは、リクエストが成功した場合、HTTP 200 OK ステータスコードを返し、レスポンスボディに受理されたConcrnt Signed Documentを返却する。

このとき、サーバーは受理したリソースの所在を示す以下のフィールドをSigned Documentに付与して返却する。

```json
{
  "document": "<JSON string>",
  "proof": { ... },
  "cckv": "cckv://<owner>/<key>",
  "ccfs": "ccfs://<owner>/concrnt/<cdid>"
}
```

* `cckv`
  Documentが `key` を持つ場合、そのキーを示すCCURI。
* `ccfs`
  Documentの内容から生成されたCDID (CIP-1) を示すCCURI。

kindごとの返却内容は以下の通りである。

* `record`: 送信されたSigned Documentに `cckv` と `ccfs` を付与したもの。
* `association` / `ack` / `unack`: 送信されたSigned Documentに `ccfs` を付与したもの。
* `entity`: 送信されたSigned Documentそのもの。
* `delete`: 削除された対象のSigned Document。

### 3.4 冪等性とリプレイ防止

Commitエンドポイントは署名済みDocumentをそのまま受理するため、攻撃者が捕捉したDocumentの再送 (リプレイ) に対する防御を持たなければならない。サーバーは以下を実装しなければならない (MUST)。

* **未来時刻の拒否**: `createdAt` がサーバー時刻より一定以上未来であるDocumentを拒否する。参考実装では許容スキューは12時間である。遠い未来の `createdAt` を許すと、後述のaccept-if-newer比較においてそのDocumentが以後のすべての更新に優先してしまうためである。
* **過去時刻の拒否 (backdate window)**: `createdAt` が一定以上過去であるDocumentを拒否する。参考実装ではこの窓は7日間である。
* **削除文書のtombstone**: `kind: "delete"` によって明示的に削除されたDocumentは、そのccfs URI (コンテンツID) を少なくともbackdate windowと同じ期間tombstoneとして記録し、その期間中の同一Documentの再コミットを拒否する。tombstoneはcckvキーではなくコンテンツIDに対して記録されるため、削除されたキー自体は新しいDocumentで再利用できる (新しいDocumentは異なるCDIDを持つため通常どおりコミットされる)。backdate windowとtombstoneを組み合わせることで、捕捉されたDocumentは「まだtombstoneが有効」か「すでに古すぎて受理されない」かのいずれかとなり、削除がリプレイに対して恒久的になる。
* **上書き時のtombstone**: 既存のキーが異なるDocumentで上書きされた場合、旧バージョンのccfs URIを同様にtombstoneとして記録する。これにより、捕捉された旧バージョンの再コミットによるロールバックが防がれ、後続の削除が旧バージョンに対しても恒久的になる。ただし、保存済みと同一のDocumentの再送 (冪等な再配送) に対してはtombstoneを記録してはならない (MUST NOT)。

また、`kind: "entity"` のコミットは accept-if-newer 方式で処理される。
サーバーは既存のEntity documentとdocument ID (CDID; `createdAt` が先頭に符号化されるため時刻順の比較が可能) を比較し、新しい場合のみ上書きする。
古いまたは同一のDocumentの再送はエラーとせず、no-opとして成功を返す (MUST)。これにより同一Documentの再送は安全である。

## 4. セキュリティと認証

サーバーは、Commitエンドポイントへのリクエストが適切に署名されていることを検証し、署名者がリソースの変更を行う権限を持っていることを確認しなければならない (MUST)。

* 署名検証に失敗したリクエスト、または形式不正なリクエスト (Documentのサイズ上限 CIP-1 §4.1 の超過を含む) に対しては、HTTP 400 Bad Request を返す。
* 署名は正当だが権限のないリクエストに対しては、HTTP 403 Forbidden を返す。

また、サーバーはコミット対象のownerがリクエスタ (Documentのauthor) をブロックしている場合、そのコミットを拒否してもよい (MAY)。参考実装は、対象ownerのブロックリストにauthorが含まれる場合にコミットを拒否する。

