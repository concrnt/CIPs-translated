# CIP-1 Concrnt Document System

## 0. Abstract

本ドキュメントでは、Concrnt エコシステム内で用いられる Concrnt Document の構造と意味について定義する。

Concrnt Document Systemは署名により操作が証明された、jsonドキュメントのための階層型データベースである。

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

### CDID (Concrnt Document ID)
CDIDは、Concrnt Document を一意に識別するためのIDである。
time-based CDID(Documentの作成日時と内容のハッシュから生成される)と、
hash-based CDID(内容のハッシュのみから生成される)の2種類が存在する(6章参照)。

## 3. 位置づけとスコープ

CIP-1 は以下のみを扱う。

* Concrnt Document の JSON 構造
* 各フィールドの意味と制約
* CDID の生成方法
* Concrnt Signed Document の構造と proof の検証方法

## 4. Concrnt Document

Concrnt Document は、JSON オブジェクトとして表現される不変のレコードであり、概念的には次のような型を持つ。
MIMEタイプは `application/concrnt.document+json` である。

```json
{
  "kind": "record",                   // required
  "key": "cckv://con1.../profile",    // optional
  "schema": "https://...",            // optional
  "value": { ... },                   // required

  "author": "con1...",                // required

  "createdAt": "2025-11-23T12:34:56Z", // required
  "onUpdate": "retain"                // optional
}
```

* `value` の中身や構造は `schema` や上位の CIP によって定義される。
* 上位のCIPによって、追加のフィールドが定義されることがある。

### 4.1 Document のサイズ上限

Document の JSON シリアライズ (署名対象となる文字列そのもの) は、UTF-8 表現で
**32768 バイト (32 KiB)** を超えてはならない (MUST NOT)。
サーバは、これを超える Document のコミット (CIP-3) を拒否しなければならない (MUST)。

Concrnt Document は投稿・プロフィール・ポリシーといった構造化メタデータを保持するための
ものであり、大きなデータの格納には適さない。画像・動画・長文コンテンツなどの大きなデータは
blob としてストレージに格納し、Document からは `ccfs://` URI (CIP-0) で参照すること。

## 5. フィールド定義

### 5.1 `kind` (string, required)

Document の操作種別を表すディスクリミネータ。サーバーは `kind` の値に基づいて Document の処理を分岐する。
未知の `kind` を持つ Document は拒否されなければならない (MUST)。

現在定義されている値は以下の通り。

| kind | 意味 | 定義 |
|---|---|---|
| `entity` | エンティティ文書 (サーバ所属の宣言) | CIP-0 |
| `record` | 一般のレコード | CIP-1 (本章) |
| `association` | 他 Document への関連付け | CIP-9 |
| `delete` | Document の削除 | CIP-4 |
| `ack` | エンティティ間の承認 | CIP-10 |
| `unack` | 承認の取消 | CIP-10 |

### 5.2 `key` (string, optional)

Document に付与される cckv URI (CIP-0 参照)。

* `key` は `cckv://<owner>/<path>` 形式の CCURI でなければならない (MUST)。
  サーバーは `key` の owner 部から、その Document を管理するエンティティを導出する。
* `key` が省略された場合、その Document は CDID (ccfs URI) でのみ参照可能である。
* 同一の`key`をもつ文書が複数存在する場合、最新の`createdAt`を持つ文書が優先される(MUST)。


### 5.3 `schema` (string, optional)

`value` の構造を定義するスキーマの識別子。

* URL（`https://schema.concrnt.net/...` など）として表現される。
* URLは解決可能なエンドポイントであり、JSON Schema形式で型が定義されているべきである (SHOULD)。
* アプリケーションは `schema` に基づいて `value` の検証・パースを行ってもよい (MAY)。
* サーバーは `schema` の解決可能性や JSON Schema としての妥当性を検証しない。

### 5.4 `value` (any JSON, required)

Document が表現する実データ。

* `value` は任意の JSON 値であり、オブジェクト・配列・文字列・数値・真偽値・null のいずれでもよい (MAY)。
* 具体的な構造と意味は `schema` によって定義される。

### 5.5 `author` (string, required)

この Document を作成したエンティティの CCID。

* `author` は CIP-0 で定義された CCID (`con1...`) でなければならない (MUST)。


### 5.6 `createdAt` (string, required)

Document が作成された時刻。

* MUST: RFC3339 形式の UTC 時刻文字列（例: `"2025-11-23T12:34:56Z"`）。
* サーバは、受理可能な `createdAt` の範囲を制限する。リファレンス実装の既定値は以下の通り
  (`MaxBackdate` / `MaxFutureSkew`, `internal/domain/commit.go`)。
  * 現在時刻より **7日** より古い Document は拒否される。
  * 現在時刻より **12時間** より未来の Document は拒否される (クロックスキュー許容)。
* この時刻境界は削除の tombstone (CIP-3 参照) と組み合わさることで、
  削除済み Document のリプレイに対する恒久的な防御を構成する。
* マイグレーション経路 (システムサービスアカウントによる import、および認証済みユーザーによる
  自身のリポジトリの再インポート) では、過去方向の制限が免除される (CIP-3 参照)。


### 5.7 `onUpdate` (string, optional)

同一 `key` の Document が新しい Document で上書きされたときの、旧 Document の扱いを指定する。

* `"retain"` (デフォルト): 旧 Document は保持される。
* `"forget"`: 旧 Document は削除され、そのコミットログは GC 候補としてマークされる
  (`postgres/record.go:242-251`)。

頻繁に上書きされるが履歴を残す必要のない Document (プレゼンス情報など) に `"forget"` を指定することで、
ストレージの肥大化を防ぐことができる。

## 6. CDID の生成

CDID には time-based と hash-based の2種類が存在する。

### 6.1 Base32 エンコーディング

いずれの CDID も、以下のテーブルを使った Base32 エンコード (パディングなし) で文字列表現される。

```text
"0123456789abcdefghjkmnpqrstuvwyz"
```

(`i`, `l`, `o`, `x` を除外した32文字)

このテーブルは Crockford Base32 (`i`, `l`, `o`, `u` を除外) とは異なり、`u` の代わりに **`x` を除外**する。
`x` は CDID のタイプを表す接頭辞文字として予約されているためである (§6.3 参照)。
エンコード出力に `x` が現れないことにより、先頭が `x` である文字列は曖昧さなく
hash-based CDID として識別できる。

### 6.2 time-based CDID

16バイトの値であり、次の構造を持つ。

```text
 <- 6bytes -> <-    10 bytes    ->
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| timestamp  |        hash        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* `hash` は Document の JSON シリアライズを Keccak256 でハッシュ化したものの先頭10バイト。
* `timestamp` は Document の `createdAt` フィールドの UNIX タイムスタンプ（ミリ秒単位）を
  ビッグエンディアンの6バイトで表現したもの。

これを Base32 エンコードした 26 文字の文字列が time-based CDID となる。

```text
9t4r7by29zwbr43c06dadzwz84
```

time-based CDID は時刻が先頭に来るため、文字列比較がそのまま `createdAt` 順
(同時刻の場合は内容ハッシュによる決定的なタイブレーク) となる。
サーバーはこの性質を Document の順序付けおよび新旧判定に利用する。

### 6.3 hash-based CDID

時刻成分を持たない Document (内容のみで同一性が決まるリソース) には hash-based CDID を使用する。

* 対象バイト列を Keccak256 でハッシュ化したものの先頭15バイトを Base32 エンコードし、
  先頭に `x` を付与した 25 文字の文字列となる。
* 先頭が `x` であることをもって hash-based CDID と識別する。
  `x` は Base32 テーブル (§6.1) から除外された予約文字であり、エンコード出力には決して現れないため、
  この識別に曖昧さは生じない。

## 7. Concrnt Signed Document

## 7. Concrnt Signed Document

Documentの発行を証明する必要がある場合、これに署名を付与することができる。
署名付き文書は次の形式を持つ。
MIMEタイプは `application/concrnt.signed-document+json` である。

```json
{
  "cckv": "cckv://<owner>/<key>",        // optional
  "ccfs": "ccfs://<owner>/concrnt/<cdid>", // optional
  "document": "<JSON string of the Document>",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  },
  "references": {                        // optional
    "<uri>": { <SignedDocument> }, ...
  }
}
```

* `document`
  署名対象の Document の JSON 文字列。
* `proof`
  署名情報を含むオブジェクト (7.1 参照)。
* `cckv` / `ccfs`
  この Document を指す CCURI。サーバーがレスポンスに付与する情報フィールドであり、署名対象ではない。
* `references`
  proof の検証に必要な関連 Document を URI をキーとしてインライン同梱するためのマップ。
  検証時の利用可否は proof type ごとに異なる (7.2〜7.5 参照)。

署名や CDID 生成に用いる JSON は、署名者が生成した文字列をそのまま使い、追加の正規化処理は行わない。

### 7.1 Proof オブジェクト

```json
{
  "type": "<proof type>",
  "signature": "<hex>",   // optional (type依存)
  "href": "<uri>",        // optional (type依存)
  "key": "<uri>"          // optional (type依存)
}
```

proof の `type` として以下の4種が定義される。

| type | 概要 | 定義 |
|---|---|---|
| `concrnt-ecrecover-direct` | author の秘密鍵による直接署名 | 7.2 (本章) |
| `concrnt-ecrecover-subkey` | サブキーによる委譲署名 | 7.3 (詳細は CIP-13) |
| `document-reference` | 参照元 Document の存在による証明 | 7.4 (詳細は CIP-7) |
| `none` | 無署名 | 7.5 |

### 7.2 concrnt-ecrecover-direct

CCID 所有者の秘密鍵で直接署名する方式。`signature` フィールドが必須である (MUST)。

署名方法:
* ハッシュ関数: Keccak256
* 署名形式: secp256k1 ECDSA。(r, s, v) の順で連結した65バイトを16進エンコードする。

検証方法:
* ECRECOVER による公開鍵復元 → author の CCID と一致するか確認。

### 7.3 concrnt-ecrecover-subkey

エンティティが有効化したサブキーの秘密鍵で署名する方式。詳細は CIP-13 で定義される。

* `signature` および `key` フィールドが必須である (MUST)。
  `key` にはサブキーの enact 文書を指す cckv URI を指定する。
* 検証時、enact 文書は**必ず authoritative なサーバーから取得しなければならない (MUST)**。
  `references` にインライン同梱された enact 文書を信用してはならない (MUST NOT)。
  これは、失効済み (削除済み) の enact 文書を同梱してリプレイする攻撃を防ぐためである。
* enact 文書の `author` と、署名対象 Document の `author` が一致することを確認しなければならない (MUST)。

### 7.4 document-reference

参照元 Document の存在をもって、自動生成された Reference Document (CIP-6/CIP-7) の
正当性を証明する方式。

* `href` フィールドが必須である (MUST)。
* この proof type は `schema` が reference.json である Document に対してのみ有効である (MUST)。
* Document の `value.href` と proof の `href` が一致することを確認しなければならない (MUST)。
* 参照先 Document は `references` のインライン同梱を優先して解決してよい (MAY)。
  同梱された参照先はそれ自体の proof 検証を再帰的に通過し、かつ `author` が本 Document の
  `author` と一致しなければならない (MUST)。これにより、正しく署名されたインライン参照は
  author 自身による自己証明となり、参照先のオリジンサーバーが到達不能な場合
  (マイグレーション中の import 等) でもコミットの検証可能性が保たれる。

### 7.5 none

署名を持たない proof。検証は常に失敗しなければならない (MUST)。

サーバー自身の鍵で認証されたシステムサービスアカウント (マイグレーション・import 用途) のみが、
署名検証そのものをスキップする形で none proof の Document をコミットできる (CIP-3 参照)。
通常のコミット経路で none proof を受理してはならない (MUST NOT)。
