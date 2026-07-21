# CIP-9 Association

## 0. Abstract
この仕様では、CIP-1で定義されるConcrnt Documentを拡張し、あるConcrnt Documentから他のConcrnt Documentへの関連付け（アソシエーション）を表現する方法を定義する。

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

## 3. Association

CIP-1で定義されたConcrnt Documentのうち、`kind` が `"association"` であるものを
Association Documentと呼ぶ。Association Documentは `associate` フィールドおよび
任意の `associationVariant` フィールドを持つ。

```json
{
  "kind": "association",              // CIP-1
  "schema": "https://...",            // CIP-1
  "value": { ... },                   // CIP-1

  "author": "con1...",                // CIP-1

  "associate": "cckv://<target-owner>/<document-key>", // CIP-8
  "associationVariant": "example-variant",             // CIP-8 (optional)

  "createdAt": "2025-11-23T12:34:56Z" // CIP-1
}
```

* `associate` フィールドには、関連付け先のDocumentを一意に識別するCCURIを指定する。
  schemeは `cckv` でなければならない (MUST)。
* Association Documentは、`key` に値が入っていてはならない (MUST NOT)。
  つまり、Association DocumentはCDID (ccfs URI) でのみ参照可能なオブジェクトである。
* `associationVariant` フィールドは、Association Documentのバリエーションを表現するための任意の文字列である。
  `associationVariant` は512バイト以内のバイト列でなければならない (MUST)。

クライアントは、Association Documentを、`associate` のCCURIのownerを管理する
Concrntサーバーに対して、CIP-3で定義されるcommitエンドポイントへ送信しなければならない (MUST)。
コミットに成功したAssociation Documentのccfs URIは
`ccfs://<associateのowner>/concrnt/<CDID>` となる。

### 3.1 一意性

Associationは、次の5要素の組み合わせで一意に識別される。

* `associate`
* `author`
* `schema`
* `associationVariant` (存在する場合)
* `value` (JSONシリアライズしたボディ)

サーバーは、この組み合わせが一致するAssociationを重複して受理してはならない (MUST NOT)。

### 3.2 Associationの配布とイベント

Association Documentのコミットに成功した場合、サーバーは関連付け先Document
(`associate`) およびその `distributes` (CIP-7) で指定される各配布先に対して、
`associated` イベント (CIP-11) を配信する。
また、Association Document自身も `distributes` フィールドを持つことができ、
その場合CIP-7の規定に従いReference Documentが配布される。

### 3.3 Associationの削除

Associationの削除は、CIP-4で定義される削除Document (`kind: "delete"`) の `value` に、
削除対象Associationのccfs URI (`ccfs://<owner>/concrnt/<CDID>`) を指定して行う。
削除に成功した場合、サーバーは関連付け先リソースに対して `unassociated` イベント (CIP-11) を配信する。

## 4. Association の取得

CIP-0で定義されるサービスディスカバリにおいて、以下の複数種類のエンドポイントを追加して広告する。
以下はテンプレートを使い、エンドポイントを広告している例である。

{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.resolve": "/resource/{uri}",
    "net.concrnt.core.associations": "/associations{?uri,schema,variant,author}",
    "net.concrnt.core.association-counts": "/association-counts"{?uri,schema},
  }
}

### 4.1 associations

`net.concrnt.core.associations` エンドポイントにアクセスすることで、
対象Documentに関連付けられたAssociation Documentの一覧を、Signed Documentの配列として取得できる。

queryパラメータとして以下をサポートする。

* `uri`: 関連付け先のDocumentを指定するCCURI。必須である (MUST)。
* `schema`: 取得するAssociation Documentのschemaを指定する。省略時は全てのschemaを対象とする。
* `variant`: 取得するAssociation Documentのvariantを指定する。省略時は全てのvariantを対象とする。
* `author`: 取得するAssociation Documentのauthorを指定する。省略時は全てのauthorを対象とする。

### 4.2 association-counts

`net.concrnt.core.association-counts` エンドポイントにアクセスすることで、
対象Documentに関連付けられたAssociation Documentの件数を取得できる。

queryパラメータとして以下をサポートする。

* `uri`: 関連付け先のDocumentを指定するCCURI。必須である (MUST)。
* `schema`: 集計モードを指定する。
  * 省略時: schemaごとの件数のマップを返す。
  * 指定時: 当該schemaのAssociationについて、variantごとの件数のマップを返す。
    このマップは、各variantの最初のAssociationが作成された順序で並ぶ。
