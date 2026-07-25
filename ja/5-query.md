# CIP-5: Query

## 0. Abstract

本ドキュメントでは、Concrnt サーバーのエンドポイントを拡張し、サーバーが管理するリソースを
条件付きで検索・列挙するための手段を定義する。

## 1. Status of This Memo

このドキュメントは Query エンドポイントの仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0 および CIP-1 を前提とする。
読み取り認可には CIP-2 および CIP-12 を必要とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Query エンドポイント

本 CIP を実装するサーバーは、HTTP GET リクエストを受け付ける Query エンドポイントを提供し、
CIP-0 のサービスディスカバリにおいて `net.concrnt.core.query` エンドポイント名で
広告しなければならない (MUST)。

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

Query エンドポイントは、以下のクエリパラメータを受け付ける。

- `prefix` — 検索対象のキーの接頭辞を指定する文字列。
  マッチングは CCURI 文字列に対する**単純な前方一致**である。CIP-4 の範囲削除 (パス階層マッチ) とは
  異なり、`cckv://con1alice/item` を指定した場合、兄弟キー `item2` もマッチする点に注意。
  パス階層で厳密に絞り込みたい場合は、末尾に `/` を付与する。
- `parent` — 検索対象の親リソースを CCURI で指定する文字列。指定したリソースの直下の要素
  (キー階層の直接の子) が検索対象となる。
- `schema` — スキーマを指定する文字列。(任意)
- `since` — `createdAt >= since` のリソースのみを返すための RFC3339 形式の日時文字列。境界を含む。(任意)
- `until` — `createdAt <= until` のリソースのみを返すための RFC3339 形式の日時文字列。境界を含む。(任意)
- `limit` — 返されるリソースの最大数を指定する整数。上限は実装定義であり、上限を超える値が
  指定された場合、サーバーは上限に切り詰めなければならない (MUST)。1 未満の値が指定された
  場合、サーバーは 1 に切り上げなければならない (MUST)。
  参考実装の既定値は 10、上限は 100 である。(任意)
- `order` — ソート順序を指定する文字列。`asc` (昇順) または `desc` (降順)。デフォルトは `desc`。
  それ以外の値が指定された場合、サーバーは HTTP 400 Bad Request を返さなければならない (MUST)。(任意)

`prefix` と `parent` は、いずれか一方を必ず指定しなければならず (MUST)、
両方を同時に指定してはならない (MUST NOT)。違反したリクエストに対して、サーバーは
HTTP 400 Bad Request を返さなければならない (MUST)。

### 3.2 レスポンス形式

サーバーは、リクエストに一致するリソースの一覧を、以下の JSON オブジェクトとして
返さなければならない (MUST)。

```json
{
  "items": [
    {
      "document": "{...}",
      "proof": { "type": "concrnt-ecrecover-direct", "signature": "..." },
      "cckv": "cckv://<CCID>/<key>",
      "ccfs": "ccfs://<CCID>/concrnt/<cdid>"
    }
  ],
  "prev": "2026-07-26T12:00:00.123456Z",
  "next": "2026-07-26T11:59:58.000001Z"
}
```

- `items` — CIP-1 で定義される Concrnt Signed Document の JSON 配列。一致するリソースが
  ない場合も空配列であり、`null` であってはならない (MUST NOT)。
- `prev` / `next` — RFC3339 形式の日時文字列によるページングカーソル、または `null`。
  両キーは常に存在しなければならない (MUST)。

返却するリソースは、`order` パラメータで指定された順序でソートされなければならない (MUST)。
ソートキーは対象 Document の `createdAt` である。Reference Document (CIP-6) については、
サーバーが参照先の `createdAt` を保持している場合 (CIP-6 §3.2)、その値をソート・フィルタに用いる。
`createdAt` が同一の Document 間の順序は、CDID の辞書順を決定的なタイブレークとして
用いなければならない (MUST)。タイブレークの方向は `order` に追従する
(`desc` の場合は CDID も降順)。

### 3.3 ページングカーソル

カーソルの値は、境界に位置するリソースのソートキー (§3.2。すなわちサーバーが保持する実効
`createdAt`) である。

- `next` — 反復方向 (`order` の方向) の続きを指すカーソル。サーバーは内部的に `limit` + 1 件を
  取得し、`limit` + 1 件目が存在する場合はそのソートキーを `next` とし、存在しない場合は
  `null` としなければならない (MUST)。クライアントは `order=desc` の場合 `until=next`、
  `order=asc` の場合 `since=next` を指定して次のページを取得する。
- `prev` — 反復方向の逆側 (`order=desc` の場合、より新しい側) を指すカーソル。値はページ先頭
  リソースのソートキーである。ページが空の場合は `null` としなければならない (MUST)。
  クライアントは `order=desc` の場合 `since=prev`、`order=asc` の場合 `until=prev` を
  指定して逆方向のリソースを取得する。

`prev` / `next` は、読み取り認可による除外 (後述) を適用する**前**の結果から計算しなければ
ならない (MUST)。除外後の結果から計算すると、境界に位置する除外リソースを越えてページングが
進めなくなるためである。したがって `items` が空配列であっても `next` が非 `null` であることが
あり、クライアントは `next` が `null` になるまでページングを継続することで全件を列挙できる。

`since` / `until` は境界を含むため、カーソル境界と同一のソートキーを持つリソースは次のページの
先頭に重複して出現しうる。全件列挙を行うクライアントは、`ccfs` URI をキーとして結果を重複排除
すべきである (SHOULD)。また、クライアントはカーソル値を丸めや再解釈を行わず、受け取った文字列を
そのままクエリパラメータとしてエコーバックすべきである (SHOULD)。精度の低い日時型 (例:
ミリ秒精度) を経由すると、境界リソースの取りこぼしが発生しうる。

既知の限界として、同一のソートキーを持つリソースが `limit` を超えて存在する場合、日時カーソル
ではその先へ進むことができない。クライアントは `next` が前回のカーソル値から進まなかった場合、
ページングを停止しなければならない (MUST)。参考実装はマイクロ秒精度のタイムスタンプを用いて
おり、実用上この状況はほぼ発生しない。

サーバーは、リクエスタ (CIP-2 の認証によって識別される。未認証の場合はゲスト) が読み取り権限を
持たないリソースを結果から除外しなければならない (MUST)。
除外が発生したことをエラーとして通知する必要はなく、単に結果から省略してよい (MAY)。
この場合、返却件数が `limit` に満たないことがある。

返却する Signed Document がネストされた `references` (CIP-1 §7) を持つ場合、各エントリについて
リクエスタで読み取りアクション (`record:read` / `association:read`, CIP-12) を評価し、
許可されないエントリを省略しなければならない (MUST)。CIP-11 §3.2 の redaction と同旨であり、
保護された元 Document が公開タイムラインの参照行に同梱されたまま漏えいすることを防ぐ。

## 4. Security Considerations

* 読み取り認可 (§3.3) は、結果の行そのものと、行に同梱されたネスト Document の両方に
  適用しなければならない。前者のみでは、配布 Reference (CIP-7 §4.1) に同梱された保護対象の
  元 Document が漏えいする。

## 5. References

- RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
- RFC 8174 – Clarifications to RFC 2119
- RFC 3339 – Date and Time on the Internet: Timestamps
- CIP-12 – Policy (読み取り認可)
