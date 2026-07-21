# CIP-7 Distribution

## 0. Abstract
本ドキュメントでは、CIP-1で定義されたConcrnt Signed Documentにおいて、該当ドキュメントを指すReferenceの作成をサーバー側に依頼するための仕様を定義する。


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

## 3. Distributionの指定

CIP-1で定義されたConcrnt Documentを拡張し、該当ドキュメントを指すReferenceを作成したい他ドキュメントを表現するために、`distributes`フィールドを追加する。


```json
{
  "key": undefined,                   // CIP-1
  "contentType": "application/json",  // CIP-1
  "schema": "https://...",            // CIP-1
  "value": { ... },                   // CIP-1

  "author": "con1...",                // CIP-1
  "owner": "con1...",                 // CIP-1

  "distributes": [                    // CIP-5
    "cckv://<owner>/<key1>",
    "cckv://<owner>/<key2>",
    ...
  ],

  "createdAt": "2025-11-23T12:34:56Z" // CIP-1
}
```

## 5. Distributionの解決

### 5.1 Reference Documentの自動生成

サーバーは、`distributes` を持つDocumentのコミットに成功した場合、配布先URIごとに
次のReference Document (CIP-6) を自動生成しなければならない (MUST)。

```json
{
  "kind": "record",
  "key": "<配布先CCURI>/<元DocumentのCDID>",
  "schema": "https://schema.concrnt.net/reference.json",
  "value": {
    "href": "<元DocumentのCCURI>"
  },
  "author": "<元Documentのauthor>",
  "createdAt": "<サーバーによる生成時刻>"
}
```

* `key` は配布先CCURIの子キーとして、元Documentの CDID (CIP-1) をパスセグメントに用いる。
* `href` には元Documentを指すCCURIを指定する。`kind: "record"` の場合はそのcckvキー、
  `kind: "association"` 等の場合はそのccfs URIとなる。

このReference Documentは、後述の `document-reference` proofを付与したSigned Documentとして構成される。
このとき `references` フィールド (CIP-1) には、受信側が追加の問い合わせなしに検証を完了できるよう、
元DocumentのSigned Documentおよびauthorのentity documentを同梱するべきである (SHOULD)。

### 5.2 配布の実行

生成したReference Documentは、配布先の所属サーバーへ通常のコミットとして投入する。

* 配布先のownerが自サーバーの管理下にある場合、自サーバーのコミット処理へ再投入する。
* 配布先のownerが他サーバーの管理下にある場合、CIP-0の名前解決によって所属サーバーを特定し、
  そのサーバーの `net.concrnt.core.commit` エンドポイント (CIP-3) へ代理で送信しなければならない (MUST)。


### 5.3 削除の伝搬

`distributes` を持つDocumentが削除された場合 (CIP-4)、サーバーは削除を各配布先へ
同様に伝搬しなければならない (MUST)。

