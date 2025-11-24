# CIP-6 Association

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

CIP-1で定義されたConcrnt Documentを拡張し、他のConcrnt Documentへの関連付けを表現するために、`associate`フィールドを追加する。

```json
{
  "key": "profile",                   // CIP-1
  "contentType": "application/json",  // CIP-1
  "schema": "https://...",            // CIP-1
  "value": { ... },                   // CIP-1

  "author": "con1...",                // CIP-1
  "owner": "con1...",                 // CIP-1

  "associate": "cc://<owner>/<document-key>", // CIP-6

  "createdAt": "2025-11-23T12:34:56Z" // CIP-1
}
```

associateを作成する場合、そのownerは常にassociate先のDocumentのownerと同一でなければならない (MUST)。
クライアントは、associateのownerを管理するConcrntサーバーに対して、CIP-2で定義されるcommitエンドポイントへ送信しなければならない。


## 4. Associationの取得

CIP-0で定義されるリソースのレスポンス形式を拡張し、apis.associationsフィールドを追加する。

```json
{
    ... CIP-0
    "apis": {
        "associations": "/api/v1/document/<id>/associations",
        "associationCounts": "/api/v1/document/<id>/association_counts"
        "associationsByAuthor": "/api/v1/document/<id>/associations_by_author"
    }
}
```

### 4.1 associations
apis.associationsにアクセスすることで、対象Documentに関連付けられたAssociation Documentの一覧を取得できる。

queryパラメータとして以下をサポートする。
- schema: 取得するAssociation Documentのschemaを指定する。省略時は全てのschemaを対象とする。
- variant: 取得するAssociation Documentのvariantを指定する。省略時は全てのvariantを対象とする。
- author: 取得するAssociation Documentのauthorを指定する。省略時は全てのauthorを対象とする。

### 4.2 associationCounts
apis.associationCountsにアクセスすることで、対象Documentに関連付けられたAssociation Documentのschemaごとの件数を取得できる。

queryパラメータとして以下をサポートする。
- schema: 取得するAssociation Documentのschemaを指定する。省略時は全てのschemaを対象とする。


