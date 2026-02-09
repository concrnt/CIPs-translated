# CIP-4 Reference Document

## 0. Abstract
本ドキュメントでは、CIP-1で定義されたConcrnt Signed Documentにおいて、別のリソースを参照するための Reference フォーマットを定義する。


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

## 3. Referenceの作成

Referenceを表す特殊なConcrnt Documentとして、schema `https://schema.concrnt.net/reference.json` を定義する。

reference.jsonスキーマはつぎのように定義される。

```json
{
  "type": "object",
  "properties": {
    "href": {
      "type": "string",
      "format": "uri",
      "description": "参照先のリソースを示すURI"
    },
    "contentType": {
      "type": "string",
      "description": "参照先リソースのContent-Type"
    }
  },
  "required": ["href", "contentType"],
  "additionalProperties": false
}
```

## 4. Referenceの解決

CIP-0で定義される"net.concrnt.core.resolve"エンドポイントに対して、Reference DocumentのURIを指定してアクセスした場合、Concrntサーバーはreferenceフィールド内のhrefで指定されたURLを304リダイレクトレスポンスでクライアントに返却しなければならない(MUST)。
但し、リクエストのAcceptヘッダーが"application/concrnt.document+json"及び"application/concrnt.signed-document+json"を含む場合、Concrntサーバーはリダイレクトの代わりにreferenceドキュメントそのものを返却する。


