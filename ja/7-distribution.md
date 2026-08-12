# CIP-7: Distribution

## 0. Abstract

本ドキュメントでは、CIP-1 で定義された Concrnt Document を拡張し、該当ドキュメントを指す
Reference の作成をサーバー側に依頼するための仕様を定義する。

## 1. Status of This Memo

このドキュメントは Distribution (配布) の仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0, CIP-1, CIP-3, CIP-6 を前提とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Distribution の指定

CIP-1 で定義された Concrnt Document を拡張し、該当ドキュメントを指す Reference を作成したい
他ドキュメント (配布先) を表現するために、`distributes` フィールドを追加する。

```json
{
  "kind": "record",                   // CIP-1
  "key": "cckv://con1.../posts/hello", // CIP-1
  "schema": "https://...",            // CIP-1
  "value": { ... },                   // CIP-1

  "author": "con1...",                // CIP-1

  "distributes": [                    // CIP-7 (本仕様)
    "cckv://<owner>/<key1>",
    "cckv://<owner>/<key2>"
  ],

  "createdAt": "2025-11-23T12:34:56Z" // CIP-1
}
```

`distributes` の各エントリは、key 部を持つ cckv URI でなければならない (MUST)。
owner 部に対するエイリアス・リゾルバヒントの禁止は CIP-3 §3.1 による。

## 4. Distribution の解決

### 4.1 Reference Document の自動生成

サーバーは、`distributes` を持つ Document のコミットに成功した場合、配布先 URI ごとに
次の Reference Document (CIP-6) を自動生成しなければならない (MUST)。

```json
{
  "kind": "record",
  "key": "<配布先CCURI>/<hrefのhash-based CDID>",
  "schema": "https://schema.concrnt.net/reference.json",
  "value": {
    "href": "<元DocumentのCCURI>"
  },
  "author": "<元Documentのauthor>",
  "createdAt": "<サーバーによる生成時刻>"
}
```

* `href` には元 Document を指す CCURI を指定する。`kind: "record"` の場合はその cckv キー
  (record は常に key を持つ。CIP-3 §3.1)、`kind: "association"` 等の key を持たない Document の
  場合はその ccfs URI となる。
* `key` は配布先 CCURI の子キーとして、`href` と同一の文字列を hash-based CDID (CIP-1 §6.3)
  でエンコードしたものをパスセグメントに用いる。

この Reference Document は、CIP-6 §5 で定義される `document-reference` proof を付与した
Signed Document として構成される。このとき `references` フィールド (CIP-1) には、受信側が追加の
問い合わせなしに検証を完了できるよう、元 Document の Signed Document および author の
Entity Document を同梱するべきである (SHOULD)。

key セグメントは `href` のみから導出され、元 Document の内容や `createdAt` に依存しない。
したがって同一キーの Document が上書き (CIP-3 §3.4 accept-if-newer) された場合、新版に対して
生成される Reference Document は旧版のものと同一のキーを持ち、accept-if-newer により旧版の
参照行を置換する — 上書きによって参照行が増殖することはない。

### 4.2 配布の実行

生成した Reference Document は、配布先の権威サーバーへ通常のコミットとして投入する。

* 配布先の owner が自サーバーの管理下にある場合、自サーバーのコミット処理へ再投入する。
* 配布先の owner が他サーバーの管理下にある場合、CIP-0 の名前解決によって権威サーバーを特定し、
  そのサーバーの `net.concrnt.core.commit` エンドポイント (CIP-3) へ代理で
  送信しなければならない (MUST)。

#### 4.2.1 配送の再試行と冪等性

リモートへの配送は失敗しうるため、サーバーは配送の再試行を行うべきである (SHOULD)。
このとき以下を満たさなければならない。

* **再試行は、最初に生成したものと同一の Signed Document を再送しなければならない (MUST)。**
  再試行のたびに Reference Document の `createdAt` を再生成してはならない (MUST NOT)。
  CDID は内容と `createdAt` から決定的に導出されるため、同一の Signed Document の再送は
  受信側で冪等に処理される (CIP-3 §3.4)。
* HTTP 4xx 応答 (408 および 429 を除く) は恒久的な失敗とみなし、再試行するべきではない
  (SHOULD NOT)。5xx 応答およびネットワークエラーのみを再試行の対象とする。
* 再試行の間隔は指数バックオフ等により逓増させるべきである (SHOULD)。
  具体的なスケジュール・試行回数の上限・上限到達後の隔離 (dead-letter) は実装定義とする。
  参考実装は初期間隔 10 秒・倍率 3・上限 1 時間のバックオフで最大 8 回試行し、
  以後は dead-letter キューへ隔離する。

### 4.3 削除の伝搬

`distributes` を持つ Document が削除された場合、サーバーは削除を各配布先へ伝搬する。
配送の単位・同梱要件・受信側の処理は CIP-4 §6 / §6.1 で規定される。

## 5. Security Considerations

* 自動生成 Reference の正当性は `document-reference` proof (CIP-6 §5.1) の検証、とりわけ
  href と参照先の同一性バインディングに依存する。受信側はこれを省略してはならない。
* 配布先のポリシー (CIP-12) は、自動生成 Reference のコミットに対する `record:create` 評価として
  適用される。配布はポリシーの回避手段にはならない。

## 6. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-4 – Delete (削除の伝搬)
* CIP-6 – Reference Document (document-reference proof)
