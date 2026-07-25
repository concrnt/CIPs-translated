# CIP-6: Reference Document

## 0. Abstract

本ドキュメントでは、CIP-1 で定義された Concrnt Document において、別のリソースを参照するための
Reference フォーマットを定義する。

## 1. Status of This Memo

このドキュメントは Reference Document の仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0 および CIP-1 を前提とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Reference の作成

Reference を表す特殊な Concrnt Document として、schema
`https://schema.concrnt.net/reference.json` を定義する。

reference.json スキーマは次のように定義される。

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
  参照先のリソースを示す URI。通常は CCURI を指定する。

* `schema` (任意)
  参照先 Document の `schema` フィールドの値。

* `createdAt` (任意)
  参照先 Document の `createdAt` フィールドの値。

### 3.1 Reference の連鎖の禁止

Reference Document の `href` は、別の Reference Document を指してはならない (MUST NOT)。
すなわち、Reference の参照先は常に実体の Document でなければならず、Reference を多段に
連鎖させることはできない。

### 3.2 参照先メタデータの解決

サーバーは、Reference Document を受理する際、参照先 Document の `schema` および `createdAt` を、
Reference 自身の保存レコードに関連付けて保持するべきである (SHOULD)。
これにより、CIP-5 の Query における `schema` フィルタや時刻範囲フィルタが、参照先の属性に基づいて
機能する。

保持するメタデータは、**検証済みの参照先 Signed Document 本体** (`references` に同梱され proof の
検証を通過したもの、または権威サーバーから取得したもの) の `schema` / `createdAt` から取得
しなければならない (MUST)。Reference の `value` に記載された値を検証なしにソート・フィルタへ
用いてはならない (MUST NOT)。検証済みの参照先を得られない場合は、Reference 自身の `createdAt` を
用いる。

参照先 Document の `createdAt` が、受理時点のクロックスキュー上限 (CIP-3 §3.4) を超えて未来である
Reference のコミットは、拒否しなければならない (MUST)。これを許すと、どこにもコミットされていない
遠未来の Document への参照によって、フィードの先頭を恒久的に占有できてしまう。

## 4. Reference の解決

CIP-0 の `net.concrnt.core.resolve` エンドポイントに対して、Reference Document の URI を指定して
アクセスした場合、サーバーは HTTP 302 Found レスポンスを返却しなければならない (MUST)。

* `Location` ヘッダには、参照先リソースへ到達可能な **HTTP(S) URL** を設定しなければならない (MUST)。
  CCURI をそのまま設定してはならない (MUST NOT)。参照先の権威サーバーを解決できない場合は、
  自サーバーの resolve エンドポイントに `href` を与えた URL を設定する (下記の例)。
* レスポンスボディには、Reference Document そのもの (CIP-1 の Signed Document 形式) を
  含めるべきである (SHOULD)。これにより、クライアントはリダイレクトへの追従と、
  Reference 自体の検証・表示のいずれも選択できる。

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

Reference Document を author の秘密鍵で直接署名できない場合 (サーバーによる代行生成、CIP-7 参照)
のために、CIP-1 の Signed Document に proof タイプ `document-reference` を定義する。

```json
{
  "document": "<JSON string of the Reference Document>",
  "proof": {
    "type": "document-reference",
    "href": "<参照先DocumentのCCURI>"
  }
}
```

* `href` は必須であり (MUST)、Reference Document の `value.href` と一致しなければならない (MUST)。
* `signature` フィールドは持たない。

### 5.1 検証方法

検証者は、タイプ `document-reference` の proof を次の手順で検証しなければならない (MUST)。

1. Document の `schema` が `https://schema.concrnt.net/reference.json` と完全一致することを確認する。
2. Document の `value.href` が `proof.href` と一致することを確認する。
3. `href` で示される参照先の Signed Document を取得する。
   `references` フィールドに同梱されている場合はそれを利用してよい (MAY)。
   同梱されていない場合はリゾルバ (CIP-0) を通じて取得する。
4. 取得した Signed Document を検証する。
   §3.1 により、参照先 Document 自身は `document-reference` proof を持たない。
   検証者はロバストネス原則にもとづき誤って作られた連鎖を許容してよいが (MAY)、その場合も
   検証深度に上限を設けなければならない (MUST)。参考実装の許容上限は 4 段である。
5. 参照先 Document の `author` が、この Reference Document の `author` と一致することを確認する。
6. **参照先 Document の同一性が `href` と一致することを確認する (MUST)。**
   比較は、リゾルバヒント (`@<FQDN>`, CIP-0 §7.2) を除去した正規形の CCURI に対して行う (MUST)。
   * `href` が cckv URI の場合: 参照先 Document の `key` が `href` と一致すること
     (key 部を持たない Entity 参照の場合は、参照先の Entity Document の `author` が owner と
     一致すること)。
   * `href` が ccfs URI (`concrnt` 種別) の場合: 参照先 Document から導出した CDID と owner が
     `href` のものと一致すること。
   * `href` が HTTP(S) URL または blob を指す場合、`document-reference` proof では正当性を
     証明できない。そのような Reference は通常の direct / subkey proof で署名されなければ
     ならない (MUST)。

   この確認は、参照先が `references` にインライン同梱されている場合にも省略してはならない
   (MUST NOT)。省略すると、同一 author による別の正当な Document を差し替えて、異なる `href` に
   対する証明として流用できてしまう。

参照先 Document 自体が正しく署名されており、かつ author が一致する場合、
この Reference Document は「author 自身が自分の Document を配布した」ものとみなせるため、
インラインの `references` を信頼しても偽造は成立しない。
これにより、参照先の権威サーバーが到達不能な場合 (移行中のインポート等) でもコミットの検証が可能となる。

## 6. Security Considerations

* 手順 6 の同一性バインディングを省略した検証者は、同一 author の別 Document を流用した
  `href` の偽装を受理してしまう。インライン同梱時にも省略してはならない。
* 参照先メタデータの未検証利用 (§3.2) は、遠未来の `createdAt` によるフィード占有を許す。

## 7. References

- RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
- RFC 8174 – Clarifications to RFC 2119
- RFC 9110 – HTTP Semantics (302 Found)
- CIP-7 – Distribution (Reference の代行生成)
