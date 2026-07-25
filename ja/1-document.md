# CIP-1: Concrnt Document System

## 0. Abstract

本ドキュメントでは、Concrnt エコシステム内で用いられる Concrnt Document の構造と意味について定義する。
Concrnt Document System は、署名により操作が証明された、JSON ドキュメントのための階層型データベースである。

## 1. Status of This Memo

このドキュメントは Concrnt Document フォーマットの仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

### CDID (Concrnt Document ID)

Concrnt Document を一意に識別するための ID。time-based CDID (Document の作成日時と内容のハッシュから
生成される) と、hash-based CDID (内容のハッシュのみから生成される) の 2 種類が存在する (§6)。

### 権威サーバー (authoritative server)

あるリソースについて、その owner (エンティティの場合は所属サーバー、FQDN / CSID の場合はそのサーバー
自身) としてリソースを管理するサーバー。

## 3. 位置づけとスコープ

CIP-1 は以下のみを扱う。

* Concrnt Document の JSON 構造、各フィールドの意味と制約
* CDID の生成方法
* Concrnt Signed Document の構造と proof の検証方法

本 CIP のうち、§4〜§6 (Document 構造・CDID)、§7.2 (`concrnt-ecrecover-direct`)、§7.5 (`none`) は
Concrnt の必須コアである (CIP-0 §4.1)。§7.3 / §7.4 の proof type は拡張 CIP (CIP-13 / CIP-6) が
定義するものであり、それらを実装しない検証者は該当 proof の検証を失敗として扱う (§7.1)。

## 4. Concrnt Document

Concrnt Document は、JSON オブジェクトとして表現される不変のレコードであり、概念的には次のような型を持つ。
MIME タイプは `application/concrnt.document+json` である。

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
* 上位の CIP によって、追加のフィールドが定義されることがある。

Document および Signed Document (§7) の各 JSON オブジェクトは、妥当な UTF-8 の JSON でなければならず、
重複するメンバ名を含んではならない (MUST NOT)。サーバーはパース時に重複メンバを検出した場合、
その Document を拒否しなければならない (MUST)。重複メンバの解釈はパーサ依存であり、
同一の署名済みバイト列がサーバー間で異なる意味に解釈されることを防ぐためである。

### 4.1 Document のサイズ上限

Document の JSON シリアライズ (署名対象となる文字列そのもの) は、UTF-8 表現で
**32768 バイト (32 KiB)** を超えてはならない (MUST NOT)。
サーバーは、これを超える Document を、あらゆる取り込み経路において拒否しなければならない (MUST)。

Concrnt Document は投稿・プロフィール・ポリシーといった構造化メタデータを保持するためのものであり、
大きなデータの格納には適さない。画像・動画・長文コンテンツなどの大きなデータは blob として
ストレージに格納し、Document からは `ccfs://` URI (CIP-0) で参照すること。

## 5. フィールド定義

### 5.1 `kind` (string, required)

Document の操作種別を表すディスクリミネータ。サーバーは `kind` の値に基づいて Document の処理を分岐する。

`kind` の語彙はレジストリとして拡張可能であり、現在は以下が定義されている。

| kind | 意味 | 定義 |
|---|---|---|
| `entity` | エンティティ文書 (サーバー所属の宣言) | CIP-0 |
| `record` | 一般のレコード | CIP-1 (本章) |
| `association` | 他 Document への関連付け | CIP-9 |
| `delete` | Document の削除 | CIP-4 |
| `ack` | エンティティ間の承認 | CIP-10 |
| `unack` | 承認の取消 | CIP-10 |

未知の `kind`、および受信サーバーが実装していない拡張 CIP の `kind` を持つ Document は、
拒否されなければならない (MUST)。

### 5.2 `key` (string, optional)

Document に付与される cckv URI (CIP-0 §7.2。key 部の文字数・`*` 禁止等の制約もそこで規定される)。

* `key` は `cckv://<owner>/<path>` 形式の CCURI でなければならない (MUST)。
  サーバーは `key` の owner 部から、その Document を管理する名前空間を導出する。
* `key` が省略された場合、その Document は CDID (ccfs URI) でのみ参照可能である。
* 同一の `key` を持つ Document が複数コミットされた場合の新旧判定・保存規則は、
  CIP-3 §3.4 の accept-if-newer に従う。

### 5.3 `schema` (string, optional)

`value` の構造を定義するスキーマの識別子。

* URL (`https://schema.concrnt.net/...` など) として表現される。
* URL は解決可能なエンドポイントであり、JSON Schema 形式で型が定義されているべきである (SHOULD)。
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

Document が作成された時刻。RFC3339 形式の UTC 時刻文字列でなければならない (MUST)。
CDID の導出 (§6.2) がこの値に依存するため、次の正規形に従う。

* 日付と時刻の区切りは大文字 `T`、UTC 指定は大文字 `Z` を用いる。
  数値オフセット表記 (`+00:00` 等) を用いてはならない (MUST NOT)。
* 小数秒は 3 桁 (ミリ秒) まで含めてもよい (MAY)。
* サーバーは、正規形に従わない `createdAt` を持つ Document を拒否しなければならない (MUST)。

サーバーが受理可能な `createdAt` の時刻範囲 (未来スキュー・backdate window) は CIP-3 §3.4 で規定される。

### 5.7 `onUpdate` (string, optional)

この Document 自身が、同一 `key` のより新しい Document で上書きされたときの扱いを宣言する。

* `"retain"` (デフォルト): 旧 Document は保持され、引き続き CDID (ccfs URI) で参照可能である。
* `"forget"`: 旧 Document は破棄され、以後参照できない。

頻繁に上書きされるが履歴を残す必要のない Document (プレゼンス情報など) に `"forget"` を指定することで、
ストレージの肥大化を防ぐことができる。`"forget"` による破棄を行う場合も、コミットログの保持期間は
CIP-3 §3.4 の下限に従わなければならない (MUST)。

## 6. CDID の生成

CDID には time-based と hash-based の 2 種類が存在する。

### 6.1 Base32 エンコーディング

いずれの CDID も、以下のテーブルを使った Base32 エンコードで文字列表現される。
エンコードは RFC 4648 §6 の手順のアルファベットを本テーブルに置き換えて適用し、
パディング文字を付さない (MUST)。

```text
"0123456789abcdefghjkmnpqrstuvwyz"
```

(`i`, `l`, `o`, `x` を除外した 32 文字)

このテーブルは Crockford Base32 (`i`, `l`, `o`, `u` を除外) とは異なり、`u` の代わりに **`x` を除外**する。
`x` は CDID のタイプを表す接頭辞文字として予約されているためである (§6.3)。
エンコード出力に `x` が現れないことにより、先頭が `x` である文字列は曖昧さなく
hash-based CDID として識別できる。

### 6.2 time-based CDID

16 バイトの値であり、次の構造を持つ。

```text
 <- 6bytes -> <-    10 bytes    ->
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| timestamp  |        hash        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* `hash` は Document の JSON シリアライズを Keccak256 でハッシュ化したものの先頭 10 バイト。
* `timestamp` は Document の `createdAt` の UNIX タイムスタンプ (ミリ秒単位。ミリ秒未満は切り捨てる)
  をビッグエンディアンの 6 バイトで表現したもの。

これを Base32 エンコードした 26 文字の文字列が time-based CDID となる。

```text
9t4r7by29zwbr43c06dadzwz84
```

time-based CDID は時刻が先頭に来るため、文字列比較がそのまま `createdAt` 順
(同時刻の場合は内容ハッシュによる決定的なタイブレーク) となる。
サーバーはこの性質を Document の順序付けおよび新旧判定に利用する。

### 6.3 hash-based CDID

時刻成分を持たない Document (内容のみで同一性が決まるリソース) には hash-based CDID を使用する。

* 対象バイト列を Keccak256 でハッシュ化したものの先頭 15 バイトを Base32 エンコードし、
  先頭に `x` を付与した 25 文字の文字列となる。
* 先頭が `x` であることをもって hash-based CDID と識別する。
  `x` は Base32 テーブル (§6.1) から除外された予約文字であり、エンコード出力には決して現れないため、
  この識別に曖昧さは生じない。

## 7. Concrnt Signed Document

Document の発行を証明する必要がある場合、これに署名を付与することができる。
署名付き文書は次の形式を持つ。MIME タイプは `application/concrnt.signed-document+json` である。

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
  署名情報を含むオブジェクト (§7.1)。
* `cckv` / `ccfs`
  この Document を指す CCURI。サーバーがレスポンスに付与する情報フィールドであり、署名対象ではない。
* `references`
  proof の検証や配送先での処理に必要な関連 Document を、URI をキーとしてインライン同梱するためのマップ。
  検証時の利用可否は proof type ごとに異なる (§7.2〜§7.5)。

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

proof の `type` はレジストリとして拡張可能であり、現在は以下の 4 種が定義されている。

| type | 概要 | 定義 |
|---|---|---|
| `concrnt-ecrecover-direct` | author の秘密鍵による直接署名 | §7.2 (本章) |
| `concrnt-ecrecover-subkey` | サブキーによる委譲署名 | CIP-13 |
| `document-reference` | 参照元 Document の存在による証明 | CIP-6 |
| `none` | 無署名 | §7.5 |

未知の proof type、および検証者が実装していない拡張 CIP の proof type を持つ Signed Document の検証は、
失敗として扱わなければならない (MUST)。

### 7.2 concrnt-ecrecover-direct

CCID 所有者の秘密鍵で直接署名する方式。`signature` フィールドが必須である (MUST)。

* 署名方法: `document` の文字列を Keccak256 でハッシュ化し、secp256k1 ECDSA 署名を行う。
  (r, s, v) の順で連結した 65 バイトを16進エンコードする (v の値域は CIP-0 §8.2)。
* 検証方法: ECRECOVER による公開鍵復元 → 導出したアドレスが author の CCID と一致するか確認する。

### 7.3 concrnt-ecrecover-subkey

エンティティが有効化したサブキーの秘密鍵で署名する方式。`signature` および `key` フィールドが
必須である (MUST)。`key` にはサブキーの Enact Document を指す cckv URI を指定する。

検証手順は CIP-13 §6 に従わなければならない (MUST)。特に、Enact Document を `references` の
インライン同梱から取得してはならず (MUST NOT)、失効済み (Revoked Subkey Document で上書きされた)
サブキーについては CIP-13 §4.1 の有効期間の検証を行わなければならない (MUST)。

### 7.4 document-reference

参照元 Document の存在をもって、自動生成された Reference Document (CIP-6 / CIP-7) の正当性を
証明する方式。`href` フィールドが必須である (MUST)。

この proof type は `schema` が `https://schema.concrnt.net/reference.json` と完全一致する Document に
対してのみ有効であり (MUST)、検証手順 (href との同一性バインディングを含む) は CIP-6 §5.1 に
従わなければならない (MUST)。

### 7.5 none

署名を持たない proof。検証は常に失敗しなければならない (MUST)。
通常のコミット経路で none proof を受理してはならない (MUST NOT)。
サーバー運用者が自身の管理経路 (CIP-2 §6) において署名検証をスキップして Document を投入するか
どうかは実装定義であり、本仕様のスコープ外である。

## 8. Security Considerations

* time-based CDID の内容バインディングは Keccak256 の先頭 10 バイト (80 ビット) であり、
  同一 `createdAt` を自己申告できる作成者自身による誕生日衝突 (約 2^40 計算) は現実的な脅威である。
  サーバーはコミットログの重複判定時にバイト列一致の確認を行うことで緩和できる (CIP-3 §3.4)。
* 重複 JSON メンバを含む Document の受理 (§4) は、署名の有効性を保ったままサーバー間で
  異なる解釈を生む攻撃 (JSON smuggling) を許すため、拒否は省略してはならない。
* proof の検証をスキップした Document を保存・配信してはならない。検証不能な proof type は
  「検証失敗」であり「検証不要」ではない (§7.1)。

## 9. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 3339 – Date and Time on the Internet: Timestamps
* RFC 4648 – The Base16, Base32, and Base64 Data Encodings
* RFC 8259 – The JavaScript Object Notation (JSON) Data Interchange Format
* CIP-0 – Concrnt Core (CCID, CCURI)
* CIP-3 – Commit (accept-if-newer, コミットログ)
