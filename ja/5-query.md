# CIP-5: Query

## 0. Abstract
本ドキュメントでは、Concrntをホストするサーバーが提供するエンドポイントを拡張し、サーバーが管理するリソースを条件付きで検索・列挙するための手段を提供する。

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
    "net.concrnt.core.query": "/query{?prefix,schema,since,until,limit,order,parent}"
  }
}
```

### 3.1 リクエスト形式

"net.concrnt.core.query" エンドポイントは、以下のクエリパラメータを受け付ける。

- `prefix` 検索対象のキーの接頭辞を指定する文字列。
  マッチングはCCURI文字列に対する**単純な前方一致**である。CIP-4の範囲削除 (パス階層マッチ) とは
  異なり、`cckv://con1alice/item` を指定した場合、兄弟キー `item2` もマッチする点に注意。
  パス階層で厳密に絞り込みたい場合は、末尾に `/` を付与する。
- `parent` 検索対象の親リソースをCCURIで指定する文字列。指定したリソースの直下の要素 (キー階層の直接の子) が検索対象となる。
- `schema` スキーマを指定する文字列。 (任意)
- `since` 指定されたタイムスタンプ以降に作成されたリソースのみを返すための RFC3339 形式の日時文字列。 (任意)
- `until` 指定されたタイムスタンプ以前に作成されたリソースのみを返すための RFC3339 形式の日時文字列。 (任意)
- `limit` 返されるリソースの最大数を指定する整数。デフォルトは 10。100を超える値が指定された場合、サーバーは 100 に切り詰める。 (任意)
- `order` ソート順序を指定する文字列。`asc` (昇順) または `desc` (降順)。デフォルトは `desc`。それ以外の値が指定された場合、サーバーは HTTP 400 Bad Request を返さなければならない (MUST)。 (任意)

`prefix` と `parent` は、いずれか一方を必ず指定しなければならない (MUST)。
両方を同時に指定してはならない (MUST NOT)。いずれも指定されない場合、および両方が指定された場合、サーバーはエラーを返す。

### 3.2 レスポンス形式

サーバーは、リクエストに一致するリソースの一覧を、CIP-1で定義される Concrnt Signed Document の JSON 配列として返す。

```json
[
  {
    "document": "{...}",
    "proof": { "type": "concrnt-ecrecover-direct", "signature": "..." },
    "cckv": "cckv://<CCID>/<key>",
    "ccfs": "ccfs://<CCID>/concrnt/<cdid>"
  }
]
```

返却するリソースは、`order` パラメータで指定された順序でソートされなければならない (MUST)。
ソートキーは対象Documentの `createdAt` である。Reference Document (CIP-6) については、
サーバーが参照先の `createdAt` を保持している場合 (CIP-6 §3.2)、その値をソート・フィルタに用いる。
`createdAt` が同一のDocument間の順序は、CDIDの辞書順を決定的なタイブレークとして用いるべきである (SHOULD)。

カーソル (継続トークン) による安定したページネーションは、本仕様では意図的に定義しない
(将来の拡張とする)。クライアントは `limit` と `since` / `until` の組み合わせによって
ページングを行う。

サーバーは、リクエスタ (CIP-2 の認証によって識別される。未認証の場合はゲスト) が読み取り権限を持たないリソースを結果から除外しなければならない (MUST)。
除外が発生したことをエラーとして通知する必要はなく、単に結果から省略してよい (MAY)。
この場合、返却件数が `limit` に満たないことがある。

## 4. References

- RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
- RFC 8174 – Clarifications to RFC 2119
- RFC 3339 – Date and Time on the Internet: Timestamps
