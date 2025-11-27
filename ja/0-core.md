# CIP-0: Core

## 0. Abstract

Concrnt は、分散型 Social Networking Service (SNS) を構築するためのプロトコル群です。
Concrnt Core では、最小プロトコルとして暗号学的なエンティティ識別子（CCID）、エンティティとサーバの所属関係（Affiliation）、および cc:// スキームによる Concrnt リソース URI（CCURI）の名前解決を定義します。

## 1. Status of This Memo

このドキュメントは Concrnt Core プロトコルの仕様を定義し、改善のための議論と提案を要求します。
本仕様はまだ進化の途上にあり、ドラフト版の間は 後方互換性のない変更が行われる可能性があります。

## 2. 著作権表示 (Copyright Notice)

Copyright (c) 2025 Concrnt Project and the contributors.

本仕様書は [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)
ライセンスのもとでパブリックドメインに提供されます。
誰でも、複製・改変・再配布・商用利用を含め自由に利用できます。
一切の保証はありません。

## 3. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 4. イントロダクション (Introduction)

ソーシャルメディアのアカウントは、個人・組織のデジタルアイデンティティの中核になりつつあります。
しかし、中央集権型のプラットフォームには次のような問題があります。

- アカウント停止やBANが不可逆的である
- 第三者による一方的な検閲
- 構築したつながりやコンテンツを失うリスク
- データの所在や制御に対するユーザーの主権不足

ActivityPub などの分散プロトコルは一定の改善を提供するものの、依然として次のような課題が存在します。

- アカウント移行がサーバの協力に依存している
- BAN された時点で移行の道が断たれる
- サーバローカルなタイムラインがコミュニティを分断する

Concrnt は、暗号学的なアイデンティティと、サーバに依存しないタイムライン／コミュニティ構造を基盤として、
これらの課題を解決することを目的としています。

本 CIP-0 (Core) は、そのうちアイデンティティと名前解決の部分のみを定義します。

## 5. Entities と CCID

Entity (エンティティ) は、Concrnt エコシステムにおける個人・組織・アプリケーションなどの主体を表します。
Entityは、暗号学的な鍵ペアによって識別されます。鍵ペアは、`secp256k1`曲線に基づき作成されます。

異なる鍵ペアの Entity は、異なる主体を表します。

### 5.1 鍵ペアの生成

Concrnt は鍵生成アルゴリズムそのものを規定しませんが、参考として HD Wallet の BIP32/BIP44 に準拠する場合、次のパスを使用するべき(SHOULD)です。

```text
m/44'/118'/0'/0/0
```

### 5.2 CCID

CCID は Entity を一意に識別するための識別子です。Entityの公開鍵を、"con"をHRPとする Bech32 エンコードで表現します。

次のテキストは、CCIDの一例です。

```text
con1t0tey8uxhkqkd4wcp4hd4jedt7f0vfhk29xdd2
```

## 6. CSID

CSIDはConcrntサーバを一意に識別するための識別子です。サーバの公開鍵を、"ccs"をHRPとするBech32エンコードで表現します。
次のテキストは、CSIDの一例です。

```text
ccs16djx38r2qx8j49fx53ewugl90t3y6ndgye8ykt
```

## 7 CCURI

CCURI (Concrnt Resource Identifier) は、Concrnt エコシステム内のリソースを指し示すための URI スキームです。
CCURI の形式は次の通りです。

```text
cc://<CCID>
cc://<CCID>/<key>
```

CCURIは、CCID部とkey部から構成されます。CCID部はリソースの所有者を示し、key部はその所有者の名前空間内でのリソースの位置を示します。
key部が省略された場合、リソースではなくエンティティそのものを指し示します。
keyは最大1024バイトのバイト列である必要があります。また、keyは最低1文字以上でなければなりません (MUST)。

## 8. Affiliation
### 8.1 Affiliation Document

エンティティは、自身が特定のサーバに所属していることを示す Affiliation Document を発行できます。

Affiliation Document の JSON 形式:

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

* `author`
  Affiliation を発行したエンティティの CCID。

* `schema`
  常に `"https://schema.concrnt.net/affiliation.json"` を指定する。

* `value.domain`
  所属先サーバの FQDN。

* `createdAt`
  Affiliation が署名された時刻（UTC, RFC3339 形式）。

### 8.2 Affiliation の署名

Affiliation Document は JSON 文字列としてシリアライズされ、エンティティの秘密鍵で署名されます。

署名の外側の構造は次の通りです。

```json
{
  "document": "<JSON string above>",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

署名対象文字列をハッシュ関数Keccak256でハッシュ化し、対象エンティティの秘密鍵でECDSA署名を行います。
signatureフィードには、署名の (v, r, s) を連結したものを16進エンコードして格納します。

検証には、ECRECOVER アルゴリズムを使用し、署名から公開鍵を復元して CCID と照合します。

### 8.3 Affiliation の公開

サーバは、所属しているエンティティの Affiliation 情報を保持し、
`net.concrnt.core.entity` エンドポイントを通じて提供しなければなりません (MUST)。

サーバは自身のローカルユーザーだけでなく、他の手段（フェデレーション、キャッシュなど）で取得した Affiliation Document を保存し、提供してもよい (MAY)。

サーバは同一エンティティ (同一 CCID) に対して複数の Affiliation Document を受け取った場合、`createdAt` がより新しいものを優先して保存しなければなりません (MUST)。


## 9 サーバ

Concrnt サーバは、Concrnt エコシステム内でリソースをホストし、配信する役割を担います。


### 9.1 サービスディスカバリ: .well-known/concrnt

サーバは次の URL で Concrnt をサポートしていることを広告します。

```text
GET https://<domain>/.well-known/concrnt
```

レスポンスは JSON で、少なくとも以下の要素を含まなければなりません (MUST)。

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

* `version`
  Concrnt プロトコルのメジャーバージョン。現段階では `"2.0"` を使用する。

* `csid`
  サーバの CSID。

* `endpoints`
  サーバが提供するエンドポイント名と、その URL テンプレートのマッピング。

サーバーは、少なくとも次の2つのエンドポイントを実装しなければなりません (MUST)。

* `net.concrnt.core.entity`
  エンティティ情報（Affiliation 等）を取得するためのエンドポイント。

* `net.concrnt.core.resource`
  CCURI に対応するリソースを取得するためのエンドポイント。

他のエンドポイントは、別の CIP によって定義されます。

### 9.2 テンプレート構文

`endpoints` の値は、次のいずれかの形式を取ることができます (MAY)。

* URIを利用したパス: `"/api/v1/resource?uri=${uri}"`
* URIの要素を利用したパス: `"/cc/${owner}/${key}"`
* 絶対 URL: `"https://cdn.example.com/${owner}/${key}"`

テンプレート内では、以下のプレースホルダを使用できます。

* `${uri}`
  完全な CCURI (`cc://<CCID>/<key>`) を URL エンコードした文字列。

* `${owner}`
  CCURI の CCID 部分 (`con1...`)。通常は URL エンコード不要。

* `${key}`
  CCURI の `<key>` 部分（先頭の `/` を除いたパス）。
  必要に応じてクライアントが URL エンコードして埋め込みます。

サーバは、未知のプレースホルダを使用すべきではありません (SHOULD NOT)。
クライアントは、仕様で定義されていないプレースホルダを見つけた場合、そのエンドポイントを利用しないことができます。


### 9.3 net.concrnt.core.entity エンドポイント

`net.concrnt.core.entity` エンドポイントは、エンティティの情報（少なくとも最新の Affiliation）を取得するために使用されます。

#### 9.3.1 テンプレート例

`.well-known/concrnt` における定義例:

```json
{
  "endpoints": {
    "net.concrnt.core.entity": "/entity/${ccid}"
  }
}
```

クライアントは、エンティティ CCID (`con1...`) を `${ccid}` にそのまま埋め込みます。

例:

```text
GET https://example.com/entity/con1abc123...
```

#### 9.3.2 レスポンスの例

サーバーは、レスポンスに最低限次の要素が含まれる JSON を返却しなければなりません (MUST)。

```json
{
  "ccid": "con1<bech32-encoded-address>",
  "affiliationDocument": "{...}",
  "affiliationSignature": "..."
}
```

* `ccid`
  対象エンティティの CCID。

* `affiliationDocument`
  Affiliation Document の JSON 文字列。

* `affiliationSignature`
  Affiliation Document に対する署名。

サーバは追加のメタ情報（例: 既知のプロフィール位置や上位プロトコルへのヒント）を含めてもよい (MAY)。


### 9.4. net.concrnt.core.resource エンドポイント

`net.concrnt.core.resource` エンドポイントは、CCURI に対応するリソースを取得するために使用されます。
#### 9.4.1 net.concrnt.core.resource の解決例

**例1: 動的 API サーバ**

```json
{
  "endpoints": {
    "net.concrnt.core.resource": "/api/v1/resource/${uri}"
  }
}
```

`cc://con1alice/keys/profile` を解決する場合:

1. クライアントは `cc://con1alice/keys/profile` を URL エンコードする。
   → `cc%3A%2F%2Fcon1alice%2Fkeys%2Fprofile`

2. テンプレートに埋め込む。

   ```text
   GET https://example.com/api/v1/resource/cc%3A%2F%2Fcon1alice%2Fkeys%2Fprofile
   ```

**例2: 静的ホスティングレイアウト**

```json
{
  "endpoints": {
    "net.concrnt.core.resource": "/${owner}/${key}"
  }
}
```

`cc://con1alice/posts/2025-11-23/hello` を解決する場合:

```text
GET https://static.example.com/con1alice/posts/2025-11-23/hello
```

`<key>` が空文字列の場合、`${key}` は空文字列に置き換えられます。
末尾のスラッシュをどう扱うかはサーバ実装に依存します（例: `/con1alice/` とするか `/con1alice` とするか）。

#### 9.4.2 レスポンスの例
サーバーは、Acceptヘッダが`application/json`であった場合、次のようなJSONレスポンスをHTTPステータス200で返却しなければなりません (MUST)。

```json
{
  "contentType": "application/json",
  "schema": "https://schema.concrnt.net/resource.json",
  "value": { ... },
  "apis": {}
}
```

その他のAcceptヘッダが指定された場合、サーバーは対応するMIMEタイプでリソースの生データを返却することができます。

## 10. Security Considerations

* 秘密鍵はクライアント側で生成・保持されるべきであり、サーバに送信してはならない (MUST NOT)。
* CCID は公開鍵から導出されるため、秘密鍵の漏洩はエンティティの乗っ取りに直結します。
  実装者は鍵生成・保管・バックアップについて十分に注意する必要があります。

## 11. Abuse Potential

Concrnt 自体は単に名前解決の枠組みを提供するのみであり、
スパム・嫌がらせ・違法コンテンツ等の問題は、主に上位プロトコルや運用ポリシーにおける課題となります。

## 12. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* BIP32, BIP44 – Hierarchical Deterministic Wallets
