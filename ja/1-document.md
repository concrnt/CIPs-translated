# CIP-1 Concrnt Document

## 0. Abstract

本ドキュメントでは、Concrnt エコシステム内で用いられる **Concrnt Document** の構造と意味について定義する。

Concrnt Document は、Concrnt Core (CIP-0) で定義された CCID / CCURI の上に載る「署名付きデータコンテナ」であり、以下のような用途に用いられる。

* 投稿、プロフィール、リアクションなどのアプリケーションデータの表現
* タイムラインやコレクションへの所属の宣言
* 参照関係（reply / like / follow 等）の記録

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

その他の用語は、CIP-0 (Core) で定義されたものを継承する。

* **Entity**: CCID を持つ暗号学的主体
* **CCID**: `con1...` 形式のエンティティ識別子
* **CCURI**: `cc://<CCID>/<key>` 形式の Concrnt URI

### CDID (Concrnt Document ID)
CDIDは、Concrnt Document を一意に識別するためのIDであり、Documentの作成日時と内容のハッシュから生成される。

## 3. 位置づけとスコープ

CIP-1 は以下のみを扱う。

* Concrnt Document の JSON 構造
* 各フィールドの意味と制約
* CDID の生成方法

## 4. Concrnt Document

Concrnt Document は、JSON オブジェクトとして表現される不変のレコードであり、概念的には次のような型を持つ。
MIMEタイプは `application/concrnt.document+json` である。

```json
{
  "key": "profile",                   // optional
  "contentType": "application/json",  // optional
  "schema": "https://...",            // required
  "value": { ... },                   // required

  "author": "con1...",                // required
  "owner": "con1...",                 // optional

  "createdAt": "2025-11-23T12:34:56Z" // required
}
```

* `value` の中身や構造は `schema` や上位の CIP によって定義される。
* 上位のCIPによって、追加のフィールドが定義されることがある。

## 5. フィールド定義

### 5.1 `key` (string, optional)

Document に付与されるオプションの「論理キー名」。

* `key` は、同じ `owner` の Document 群の中で **mutable な “ヘッド”** を指す名前として利用されることがある。
* `key`は1024バイト以内のUTF-8文字列でなければならない(MUST)。
* `key`を指定した場合、ccURIでは`cc://<owner>/<key>`として参照される。
* 同一の`owner`と`key`の組み合わせの文書が複数存在する場合、最新の`createdAt`を持つ文書が優先される(MUST)。

### 5.2 `contentType` (string, optional)
Document のメディアタイプを表す文字列。
* `contentType` が省略された場合、デフォルトで `"application/json"` と見なしてよい (SHOULD)。

### 5.3 `schema` (string, required)

`value` の構造を定義するスキーマの識別子。

* URL（`https://schema.concrnt.net/...` など）として表現される。
* URLは解決可能なエンドポイントであり、JSON Schema形式で型が定義されていなければならない (MUST)。
* `schema` は省略してはならない (MUST)。
* アプリケーションは `schema` に基づいて `value` の検証・パースを行ってもよい (MAY)。

### 5.4 `value` (any JSON, required)

Document が表現する実データ。

* `value` は任意の JSON 値であり、オブジェクト・配列・文字列・数値・真偽値・null のいずれでもよい (MAY)。
* 具体的な構造と意味は `schema` によって定義される。

### 5.5 `author` (string, required)

この Document を作成したエンティティの CCID。

* `author` は CIP-0 で定義された CCID (`con1...`) でなければならない (MUST)。

### 5.6 `owner` (string, optional)

Document の「論理的な所有者」を表す CCID。

* `owner` が省略された場合、その Document の所有者は `author` と見なされる (MUST)。

### 5.7 `createdAt` (string, required)

Document が作成された時刻。

* MUST: RFC3339 形式の UTC 時刻文字列（例: `"2025-11-23T12:34:56Z"`）。
* サーバは、あまりにも過去または未来の `createdAt` の Document を拒否してもよい (MAY)。

## 6. CDID の生成

CDIDは16バイトの値であり、次の構造を持つ。

 <-    10 bytes    -> <- 6bytes -> 
-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
|        hash        | timestamp  |
-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-

hashはDocumentのJSONシリアライズをKeccak256でハッシュ化したものの先頭10バイト。
timestampはDocumentのcreatedAtフィールドのUNIXタイムスタンプ（ミリ秒単位）を6バイトで表現したもの。

これを、以下のテーブルを使ってBase32エンコードしたものがCDIDとなる。
"0123456789abcdefghjkmnpqrstvwxyz"


最終的に次のような26文字の文字列になる。
9t4r7by29zwbr43c06dadzwz84


## 7. Documentの参照

DocumentはCCURI形式で参照できることが期待される。
* CCURI形式: `cc://<owner>/<key or CDID>`

## 8. Concrnt Signed Document

Documentの発行を証明する必要がある場合、これに署名を付与することができる。
署名は次の形式を持つ。
MIMEタイプは `application/concrnt.signed-document+json` である。

```json
{
  "document": "<JSON string above>",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

* `document`
  署名対象の Document の JSON 文字列。
* `proof`
    署名情報を含むオブジェクト。
    * `type`
        署名アルゴリズムの識別子。
    * `signature`
        Document の JSON 文字列に対する署名の16進エンコード表現。

### 8.1 concrnt-ecrecover-direct

CCID 所有者の秘密鍵で直接署名する方式。

署名方法:
* ハッシュ関数: Keccak256
* 署名形式: secp256k1 ECDSA, (v, r, s) 形式

検証方法:
* ECRECOVER による公開鍵復元 → CCID の公開鍵と一致するか確認

署名や CDID 生成に用いる JSON は、署名者が生成した文字列をそのまま使い、追加の正規化処理は行わない。
