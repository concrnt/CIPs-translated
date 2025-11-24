# CIP-0: Core

## 0. Abstract

Concrnt Core は、分散型 SNS のための最小プロトコルです。暗号学的なエンティティ識別子（CCID）、エンティティとサーバの所属関係（Affiliation）、および `cc://` スキームによる Concrnt リソース URI（CCURI）の名前解決だけに責務を限定します。投稿やプロフィールなどリソースの内容・形式・署名方法は上位のプロトコル（CIP-1 以降）で定義され、本仕様は薄い基盤層としての相互運用性を提供します。

## 1. Status of This Memo

本ドキュメントは Concrnt Core プロトコルのインターネット・ドラフトです。Concrnt プロジェクトが公開するバージョン付き仕様であり、実装者および貢献者を対象とします。ドラフト期間中は後方互換性のない変更が行われる可能性があるため、実装者は本仕様の版数と CIP 番号を確認し、更新に追従しなければなりません（MUST）。

## 2. 著作権表示 (Copyright Notice)

Copyright (c) 2025 Concrnt Project and the contributors.

本仕様書は [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) のもとでパブリックドメインに提供される。誰でも複製・改変・再配布・商用利用を含め自由に利用できるが、いかなる保証も与えられない。

## 3. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

本仕様で使用する主要な用語を以下に示します。

Entity: 公開鍵／秘密鍵ペアおよび Concrnt Canonical ID (CCID) によって表される暗号学的主体です。  
CCID: エンティティを一意に識別する Bech32 文字列で、公開鍵から導出され、ライフタイムを通じて不変です。  
CSID: Concrnt Server ID。サーバの公開鍵から導出された Bech32 文字列であり、サーバ識別や広告に用います。  
Resource: Concrnt Core の名前解決により到達可能な HTTP(S) リソースであり、内容やフォーマットは Core の対象外です。  
CCURI: Concrnt Canonical URI。`cc://<CCID>/<key>` 形式でリソースを指し示す URI です。

## 4. イントロダクション (Introduction)

ソーシャルメディアのアイデンティティは中心的な資産となっていますが、中央集権型プラットフォームでは一方的な停止や移行阻害が生じます。既存の分散プロトコルは改善をもたらしたものの、移行手続きやコミュニティ分断などの課題が残ります。Concrnt は暗号学的アイデンティティとサーバ非依存の名前空間により、ユーザー主権を回復することを目指します。本仕様はその基盤として、識別・所属・名前解決のみを定義します。

## 5. アーキテクチャ概要 (Architecture Overview)

Concrnt Core は三つの機能で構成されます。第一に、CCID による暗号学的アイデンティティ管理です。第二に、エンティティが自身の所属先サーバを示す Affiliation Document とその検証です。第三に、`.well-known/concrnt` で広告されたテンプレートを用いた CCURI から HTTP(S) URL への解決です。これら以外の行動モデルやデータ構造は上位の CIP に委ねられます。

## 6. Entities と CCID

### 6.1 CCID の形式

CCID は secp256k1 公開鍵から導出された Bech32 文字列であり、HRP は `"con"` とします。形式は `con1<bech32-encoded-address>` で表現され、同一エンティティのライフタイムにわたり変更してはならない（MUST NOT）とします。鍵生成手順は本仕様で固定しませんが、HD Wallet の BIP32/BIP44 を用いる場合は `m/44'/118'/0'/0/0` のパスを推奨します（RECOMMENDED）。

## 7. Affiliation（所属）

### 7.1 Affiliation Document

エンティティは自らの所属先ドメインを示す JSON オブジェクトを発行し、これを Affiliation Document と呼びます。

```json
{
  "author": "con1<bech32-encoded-address>",
  "schema": "https://schema.concrnt.net/affiliation.json",
  "value": {
    "domain": "example.com"
  },
  "createdAt": "2025-11-23T12:34:56Z"
}
```

`author` は CCID、`schema` は上記の固定 URI、`value.domain` は所属先サーバの FQDN、`createdAt` は RFC3339 UTC 時刻です。

### 7.2 署名形式

Affiliation Document は JSON 文字列を Keccak256 でハッシュ化し、secp256k1 ECDSA (v, r, s) で署名します。署名付きオブジェクトは次の構造を持ちます。

```json
{
  "document": "<JSON string above>",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

検証時は ECRECOVER により復元した公開鍵が `author` の CCID に一致することを確認しなければなりません（MUST）。

### 7.3 サーバでの保持と更新

サーバは所属エンティティの Affiliation Document を保持し、`net.concrnt.core.entity` で提供しなければなりません（MUST）。同一 CCID に対して複数の Affiliation が存在する場合、`createdAt` が最新のものを優先して保存します（MUST）。Affiliation の失効や多重所属は本仕様の範囲外であり、将来の CIP で定義されます。

## 8. Servers と CSID

CSID はサーバの公開鍵から導出された Bech32 文字列であり、HRP は `"ccs"` とします。形式は `ccs1<bech32-encoded-address>` で、`.well-known/concrnt` の広告や相互識別に用います。本仕様は CSID を識別子以上には解釈しませんが、上位プロトコルは認証や署名に利用してもよい（MAY）とします。

## 9. サービスディスカバリと名前解決

### 9.1 `.well-known/concrnt` の取得

クライアントは HTTPS で `https://<domain>/.well-known/concrnt` を取得し、以下の構造を含む JSON を受け取ります。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.entity": "/entity/${ccid}",
    "net.concrnt.core.resource": "/resource/${uri}"
  }
}
```

`version` は Core のメジャーバージョンを示し、`endpoints` にはテンプレートが含まれます。テンプレートでは `${ccid}` が CCID、`${uri}` が URL エンコード済み CCURI、`${owner}` が CCID 部分、`${key}` がパス部分を表します。未知のプレースホルダは使用してはならない（SHOULD NOT）とします。

### 9.2 Affiliation の検証

1. クライアントは `net.concrnt.core.entity` テンプレートに CCID を埋め込み、最新の Affiliation を取得します。  
2. 応答に含まれる Affiliation Document を JSON として復元し、署名を検証します。  
3. `value.domain` が現在問い合わせているサーバのドメインと一致しない場合、クライアントはそのサーバを所属先として扱ってはなりません（MUST NOT）。  
4. `createdAt` が過去の値に巻き戻っていないかを確認し、旧い値であれば無視します。

### 9.3 CCURI の解決手順

1. クライアントは CCURI `cc://<CCID>/<key>` を解析し、`<CCID>` に対して Affiliation を検証します。  
2. `.well-known/concrnt` の `net.concrnt.core.resource` テンプレートに `${uri}` または `${owner}`/`${key}` を埋め込み、HTTP(S) URL を得ます。  
3. 生成した URL に対して HTTP GET を実行し、リソースを取得します。`Accept` ヘッダにより表現を指定してもよい（MAY）とします。  
4. キャッシュや CDN を利用する場合、`version` が一致するか、サーバが返すキャッシュディレクティブに従います。

## 10. net.concrnt.core.entity エンドポイント

`net.concrnt.core.entity` は CCID に対応する Affiliation 情報を返します。典型的なリクエストは次の通りです。

```
GET /entity/con1abc123... HTTP/1.1
Host: example.com
Accept: application/json
```

成功時の応答例を示します。

```json
{
  "ccid": "con1abc123...",
  "affiliationDocument": "{...}",
  "affiliationSignature": "...",
  "metadata": {
    "hints": []
  }
}
```

`metadata` は追加情報を含めてもよい（MAY）とします。Affiliation を持たない場合は HTTP 404 を返す（SHOULD）とします。

## 11. net.concrnt.core.resource エンドポイント

`net.concrnt.core.resource` は CCURI に対応するリソースを返します。動的 API での例を示します。

```
GET /api/v1/resource/cc%3A%2F%2Fcon1alice%2Fkeys%2Fprofile HTTP/1.1
Host: example.com
Accept: application/json
```

静的ホスティングの場合は `/con1alice/posts/2025-11-23/hello` のように `${owner}` と `${key}` を埋め込んだパスを用います。`Accept: application/json` が指定されたとき、サーバは JSON で応答しなければなりません（MUST）。

```json
{
  "contentType": "application/json",
  "schema": "https://schema.concrnt.net/resource.json",
  "value": {},
  "apis": {}
}
```

その他の表現が要求された場合、サーバは対応する MIME タイプでリソース本体を返してもよい（MAY）とします。

## 12. エラー処理

`resource` パラメータが欠落または無効な場合、サーバは 400 Bad Request を返します。Affiliation が見つからない場合や指定されたリソースが存在しない場合は 404 Not Found を返します。サーバ内部の問題やバックエンドの失敗は 5xx で報告します。リダイレクトを行う場合は HTTPS のみを使用します（MUST）。

## 13. Security Considerations

輸送の保護として HTTPS を必須とし、証明書が無効な場合に HTTP へフォールバックしてはなりません（MUST NOT）。Affiliation の署名や `createdAt` を検証することで、巻き戻しや改ざんを検出します。`.well-known/concrnt` やエンドポイントテンプレートのキャッシュは、信頼できるキャッシュバリデータに基づいて行い、古い情報による解決を防ぎます。秘密鍵はクライアント側で保持し、サーバに送信してはなりません（MUST NOT）。ホスト委譲やリダイレクトを利用する場合、依存先のドメインを信頼できる範囲に限定し、CORS 設定を慎重に行います。

## 14. References

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, March 1997.  
[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, May 2017.  
[RFC3986] Berners-Lee, T., et al., “Uniform Resource Identifier (URI): Generic Syntax”, January 2005.  
[RFC5785] Nottingham, M., and J. Levine, “Well-Known URIs”, April 2010.  
[RFC7231] Fielding, R., et al., “Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content”, June 2014.  
BIP32, BIP44 – Hierarchical Deterministic Wallets.***
