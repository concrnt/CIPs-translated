# CIP-2: Commit

## 0. Abstract

本ドキュメントでは、Concrnt サーバが提供するリソースを書き換えるための Commit エンドポイントを定義します。CIP-1 の Concrnt Signed Document を POST し、サーバが管理するリソースの作成・更新・削除を要求します。

## 1. Status of This Memo

本ドキュメントは Concrnt Commit プロトコルのインターネット・ドラフトです。Concrnt プロジェクトが公開するバージョン付き仕様であり、実装者およびプロトコル設計者を対象とします。ドラフト期間中は後方互換性のない変更が行われる可能性があるため、実装者は CIP 番号とバージョンを確認し、更新に追従しなければなりません（MUST）。

## 2. 用語 (Terminology)

このドキュメントにおける大文字のキーワードは BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。その他の用語は CIP-0 および CIP-1 の定義を継承します。

## 3. 概要

Commit は、`.well-known/concrnt` において `net.concrnt.core.commit` として広告される HTTP POST エンドポイントです。クライアントは `owner` で示されるエンティティを管理するサーバに対し、署名済み Document を提出します。サーバは署名と権限を検証し、受理時に CDID やリソース URL を応答します。

`owner` がサーバ管理外である場合の挙動は上位仕様に委ねられますが、CIP-5/6 などからの代理送信を受け付ける場合は、そのポリシーを明示する必要があります。

## 4. エンドポイント広告

サーバは `.well-known/concrnt` に次のような項目を含めます。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.entity": "/entity/${ccid}",
    "net.concrnt.core.resource": "/resource/${uri}",
    "net.concrnt.core.commit": "/commit"
  }
}
```

`version` と `csid` は CIP-0 の定義に従います。テンプレートに未知のプレースホルダを含めてはなりません（SHOULD NOT）。

## 5. リクエスト

クライアントは `Content-Type: application/concrnt.signed-document+json` を指定し、CIP-1 で定義された Concrnt Signed Document を POST します。`owner` フィールドに記載されたエンティティを管理するサーバに送信しなければなりません（MUST）。署名者が所有者と異なる場合の権限付与は本仕様の範囲外です。

`Accept` ヘッダを指定しない場合、サーバは JSON で応答することを想定します。リクエストボディが不正な JSON である、署名が検証できない、または `owner` が欠落している場合、サーバは 400 Bad Request を返します。

## 6. レスポンス

受理に成功した場合、サーバは HTTP 201 Created を返し、以下の JSON を含めます。

```json
{
  "cdid": "<generated CDID>",
  "uri": "cc://<owner>/<key or cdid>",
  "url": "https://example.com/resource/..."
}
```

署名が無効である場合や `owner` が管理外である場合は 403 Forbidden、入力が不正な場合は 400 Bad Request を返します。サーバ内部の障害は 5xx で報告します。重複する再送に対しては同一 CDID を返すなど、冪等性を保つことが望まれます（SHOULD）。

`uri` には `key` がある場合は `cc://<owner>/<key>` を、ない場合は `cc://<owner>/<cdid>` を返します。`url` は `net.concrnt.core.resource` で得られる実際の取得 URL を示します。

## 7. 削除要求

削除は `"https://schema.concrnt.net/delete.json"` を `schema` に持つ Document を用いて行います。`value` に削除対象の CCURI を記載し、通常の Commit と同様に送信します。サーバは署名と権限を確認し、許可されない場合は 403 を返します。

## 8. セキュリティに関する考慮事項

署名検証は必須であり、無署名または無効な署名のリクエストを処理してはなりません（MUST NOT）。リプレイ防止のため、同一 Document の再送を検出し、冪等に処理することが推奨されます。過剰な書き込みを防ぐためのレート制限や監査ログの実装が望まれます。クロスサーバ転送を行う場合は、転送先を信頼できる範囲に限定し、TLS により保護してください。

## 9. 参考文献 (References)

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, March 1997.  
[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, May 2017.  
[RFC7231] Fielding, R., et al., “Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content”, June 2014.
