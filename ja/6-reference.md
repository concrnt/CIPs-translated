# CIP-6 Reference Document

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
    "schema": {
      "type": "string",
      "description": "参照先リソースのschema (任意)"
    },
    "createdAt": {
      "type": "string",
      "format": "date-time",
      "description": "参照先リソースのcreatedAt (任意)"
    }
  },
  "required": ["href"]
}
```

* `href`
  参照先のリソースを示すURI。通常はCCURIを指定する。

* `schema` (任意)
  参照先Documentの `schema` フィールドの値。

* `createdAt` (任意)
  参照先Documentの `createdAt` フィールドの値。

### 3.2 Referenceの連鎖の禁止

Reference Documentの `href` は、別のReference Documentを指してはならない (MUST NOT)。
すなわち、Referenceの参照先は常に実体のDocumentでなければならず、Referenceを多段に連鎖させることはできない。

### 3.1 参照先メタデータの解決

サーバーは、Reference Documentを受理する際、参照が指す先のDocumentの `schema` および `createdAt` を、Reference自身の保存レコードに関連付けて保持するべきである (SHOULD)。

これにより、CIP-5のQueryにおける `schema` フィルタや時刻範囲フィルタが、参照先の属性に基づいて機能する。

## 4. Referenceの解決

CIP-0で定義される "net.concrnt.core.resolve" エンドポイントに対して、Reference DocumentのURIを指定してアクセスした場合、
Concrntサーバーは HTTP 302 Found レスポンスを返却しなければならない (MUST)。

* `Location` ヘッダには、参照先リソースへ到達可能なURLを設定する。
  サーバーは、`href` に指定されたCCURIをそのまま返すのではなく、CIP-0の名前解決手順によって解決した具体的なHTTP URLを設定してよい (MAY)。
* レスポンスボディには、Reference Documentそのもの (CIP-1のSigned Document形式) を含めるべきである (SHOULD)。
  これにより、クライアントはリダイレクトへの追従と、Reference自体の検証・表示のいずれも選択できる。

```
HTTP/1.1 302 Found
Location: https://example.com/api/v2/resolve?uri=cckv%3A%2F%2Fcon1alice%2Fposts%2Fhello
Content-Type: application/json

{
  "document": "{...reference document...}",
  "proof": { "type": "concrnt-ecrecover-direct", "signature": "..." }
}
```

## 5. document-reference proof

Reference Documentをauthorの秘密鍵で直接署名できない場合 (サーバーによる代行生成、CIP-7参照) のために、
CIP-1のSigned Documentにproofタイプ `document-reference` を定義する。

```json
{
  "document": "<JSON string of the Reference Document>",
  "proof": {
    "type": "document-reference",
    "href": "<参照先DocumentのCCURI>"
  }
}
```

* `href` は必須であり (MUST)、Reference Documentの `value.href` と一致しなければならない (MUST)。
* `signature` フィールドは持たない。

### 5.1 検証方法

検証者は、タイプ `document-reference` のproofを次の手順で検証しなければならない (MUST)。

1. Documentの `schema` が `https://schema.concrnt.net/reference.json` であることを確認する。
2. Documentの `value.href` が `proof.href` と一致することを確認する。
3. `href` で示される参照先のSigned Documentを取得する。
   `references` フィールドに同梱されている場合はそれを利用してよい (MAY)。
   同梱されていない場合はリゾルバ (CIP-0) を通じて取得する。
4. 取得したSigned Documentを検証する。
   参照先Documentがさらに `document-reference` proofを持つこと (Referenceの連鎖) を、
   作成者は行ってはならない (MUST NOT、§3.2)。
   検証者はロバストネス原則にもとづき誤った連鎖を許容してよいが (MAY)、その場合も
   検証深度に上限を設けなければならない (MUST)。リファレンス実装の許容上限は4段である。
5. 参照先Documentの `author` が、このReference Documentの `author` と一致することを確認する。

参照先Document自体が正しく署名されており、かつauthorが一致する場合、
このReference Documentは「author自身が自分のDocumentを配布した」ものとみなせるため、
インラインの `references` を信頼しても偽造は成立しない。
これにより、参照先のオリジンサーバーが到達不能な場合 (移行中のインポート等) でもコミットの検証が可能となる。

## 5. References

- RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
- RFC 8174 – Clarifications to RFC 2119
- RFC 9110 – HTTP Semantics (302 Found)
