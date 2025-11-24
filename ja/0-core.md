# CIP-0: Core

## 0. Abstract

Concrnt は、分散型 Social Networking Service (SNS) を構築するためのプロトコル群です。
本ドキュメントでは、その中核をなす **Concrnt Core プロトコル** を定義します。

Concrnt Core は、次のような「一番薄い層」に責務を限定します。

* 暗号学的なエンティティ識別子（CCID）
* エンティティとサーバの所属関係（Affiliation）
* cc:// スキームを用いた Concrnt リソース URI（CCURI）の名前解決

リソース（投稿やプロフィールなど）の中身・形式・署名方法は、Concrnt Core の対象外です。
これらは上位のプロトコル（CIP-1 以降やアプリケーション仕様）で定義されます。

## 1. Status of This Memo

このドキュメントは Concrnt Core プロトコルの仕様を定義します。

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者および貢献者を対象としています。

本仕様はまだ進化の途上にあり、ドラフト版の間は **後方互換性のない変更が行われる可能性があります**。
実装者は、本ドキュメントのバージョン番号および CIP 番号を確認しながら実装するべきです。

## 2. 著作権表示 (Copyright Notice)

Copyright (c) 2025 Concrnt Project and the contributors.

本仕様書は [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)
ライセンスのもとでパブリックドメインに提供されます。
誰でも、複製・改変・再配布・商用利用を含め自由に利用できます。
一切の保証はありません。

## 3. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

### Entity

公開鍵／秘密鍵ペアおよび Concrnt Canonical ID (CCID) によって表される暗号学的な主体。
ユーザーアカウント、ボット、アプリケーションなどを含む。

### CCID

エンティティを一意に識別する文字列。エンティティの公開鍵から導出される。
CCID はグローバルに一意であり、エンティティのライフタイムを通じて変化してはならない。

### CSID

Concrnt Server ID。サーバ自身のアイデンティティおよび鍵を表す識別子。
サーバは CSID を持つ **MAY** があり、Concrnt Core はサーバの CSID を広告・識別の用途以上には解釈しない。

### Resource

Concrnt Core の名前解決により到達可能な HTTP(S) リソース。
本文、JSON、HTML、画像など **内容やフォーマットは Concrnt Core の対象外** である。

### CCURI

Concrnt Canonical URI。`cc://` スキームを持つ URI であり、Concrnt エコシステム内のリソースを識別する。
例: `cc://con1abc.../posts/2025-11-23/hello`

## 4. イントロダクション (Introduction)

### 4.1 Motivation

ソーシャルメディアのアカウントは、個人・組織のデジタルアイデンティティの中核になりつつあります。
しかし、中央集権型のプラットフォームには次のような問題があります。

* アカウント停止やBANが不可逆的である
* 第三者による一方的な検閲
* 構築したつながりやコンテンツを失うリスク
* データの所在や制御に対するユーザーの主権不足

ActivityPub (Mastodon, Misskey) などの分散プロトコルは一定の改善を提供するものの、依然として次のような課題が存在します。

* アカウント移行がサーバの協力に依存している
* BAN された時点で移行の道が断たれる
* サーバローカルなタイムラインがコミュニティを分断する

Concrnt は、暗号学的なアイデンティティと、サーバに依存しないタイムライン／コミュニティ構造を基盤として、これらの課題を解決することを目的としています。

本 CIP-0 (Core) は、そのうち **「アイデンティティと名前解決」の最小部分** のみを定義します。

### 4.2 Goals / Non-Goals

Concrnt Core の目標:

1. **暗号学的アイデンティティ**
   エンティティは CCID と公開鍵によって識別され、サーバの内部IDやデータベースに依存しない。

2. **サーバ非依存の CCURI 名前空間**
   `cc://<CCID>/<key>` 形式の URI で、どのサーバからでも同じエンティティの同じリソースを指し示せる。

3. **所属（Affiliation）の明示**
   ある時点で「どのサーバがこの CCID の namespace をホストしているか」を暗号学的に証明できる。

4. **薄い Core と豊かな上位プロトコル**
   Core はあくまで識別・所属・名前解決にとどめ、
   投稿、タイムライン、イベントソーシングなどの具体的な挙動は上位の CIP で定義する。

Concrnt Core の非目標:

* 単一巨大インデックスサーバやマス向けコンテンツ配信の仕様を定めること
* リソース（投稿等）の内部構造や署名形式を固定すること
* モデレーションやポリシー言語など、アプリケーションレベルの振る舞いを定義すること

## 5. アーキテクチャ概要 (Architecture Overview)

Concrnt Core は概念的に次の3つの要素から構成されます。

1. **Entities and CCIDs**
   すべての主体は CCID を持つエンティティとして扱われ、公開鍵に基づく暗号学的な所有証明を持つ。

2. **Affiliation（所属）**
   エンティティは「自分はこのサーバに所属している」という宣言 (Affiliation Document) を署名付きで発行できる。
   サーバはこれを保持し、クライアントに提供する。

3. **Name Resolution（名前解決）**
   `cc://<CCID>/<key>` 形式の CCURI が、
   `.well-known/concrnt` によって広告されるエンドポイント・テンプレートを通じて HTTP(S) URL に解決される。

Concrnt Core は、**ここまでしか定めません**。
返却されるリソースの形式・意味・署名方法は、別の CIP（例: CIP-1, CIP-2）で定義されるか、アプリケーション固有の仕様となります。

## 6. Entities と CCID

### 6.1 CCID の形式

CCID は Concrnt エコシステムにおけるエンティティの識別子です。
グローバルに一意であり、エンティティの公開鍵から派生します。

* 曲線: `secp256k1`
* アドレス形式: Bech32
* HRP (Human Readable Part): `"con"`

**形式**

```text
con1<bech32-encoded-address>
```

例:

```text
con1abc123def456...
```

CCID はエンティティのライフタイムを通じて変化しないことが期待されます。

### 6.2 鍵生成

Concrnt Core は鍵生成アルゴリズムそのものを規定しませんが、参考として HD Wallet の BIP32/BIP44 に準拠する場合、次のパスを **推奨** します。

```text
m/44'/118'/0'/0/0
```

この推奨は既存実装との互換性のためのものであり、別の鍵管理方式を用いる実装も許容されます。

## 7. Affiliation（所属）

### 7.1 Affiliation Document

エンティティは、自身が特定のサーバに所属していることを示す **Affiliation Document** を発行できます。

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

### 7.2 Affiliation の署名

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

署名アルゴリズム:

* ハッシュ関数: Keccak256
* 署名形式: secp256k1 ECDSA, (v, r, s) 形式
* 検証: ECRECOVER による公開鍵復元 → CCID の公開鍵と一致するか確認

### 7.3 サーバにおける Affiliation の取り扱い

サーバは、所属しているエンティティの Affiliation 情報を保持し、
`net.concrnt.core.entity` エンドポイントを通じて提供 **しなければなりません (MUST)**。

* サーバは自身のローカルユーザーだけでなく、他の手段（フェデレーション、キャッシュなど）で取得した Affiliation Document を保存し、提供しても **よい (MAY)**。

* サーバは同一エンティティ (同一 CCID) に対して複数の Affiliation Document を受け取った場合、
  `signedAt` がより新しいものを優先して保存 **しなければなりません (MUST)**。

Affiliation の無効化や多重所属などの高度な挙動は、CIP-0 の範囲外とし、将来の CIP で定義される可能性があります。

## 8. Servers と CSID

### 8.1 CSID

Concrnt サーバは、Concrnt エコシステム内でリソースをホストし、配信する役割を担います。

各サーバは、CSID (Concrnt Server ID) を持つことができます。

* CCID と同様に secp256k1 公開鍵から導出
* Bech32 HRP は `"ccs"`

**形式**

```text
ccs1<bech32-encoded-address>
```

CSID は主にサーバ同士の識別および Affiliation Document 内での参照に使用されます。
Core は CSID に特別な意味を与えませんが、上位プロトコルや実装は CSID を用いた認証や署名を行っても構いません。

### 8.2 サーバディスカバリ: .well-known/concrnt

サーバは次の URL で Concrnt Core をサポートしていることを広告します。

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
  Concrnt Core プロトコルのメジャーバージョン。現段階では `"2.0"` を使用する。

* `csid`
  サーバの CSID。

* `endpoints`
  サーバが提供するエンドポイント名と、その URL テンプレートのマッピング。

CIP-0 では、次の2つのエンドポイントを **必須 (MUST)** とします。

* `net.concrnt.core.entity`
  エンティティ情報（Affiliation 等）を取得するためのエンドポイント。

* `net.concrnt.core.resource`
  CCURI に対応するリソースを取得するためのエンドポイント。

他のエンドポイント（例: タイムライン、メッセージ、ポリシー等）は、別の CIP によって定義されます。

## 9. エンドポイントテンプレートとプレースホルダ

### 9.1 テンプレート構文

`endpoints` の値は、次のいずれかの形式を取ることができます (MAY)。

* 相対パス: `"/api/v1/resource/${uri}"`
* 絶対パス: `"/cc/${owner}/${key}"`
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

### 9.2 CCURI の構文

Concrnt Core における CCURI は以下の形式を取ります。

```text
cc://<CCID>/<key>
```

* `<CCID>`
  エンティティの CCID (`con1...`)。

* `<key>`
  エンティティの名前空間内の任意のパス。
  複数の `/` 区切りを含んでもよい (MAY) し、空文字列でもよい (MAY)。

例:

* `cc://con1alice/keys/profile`
* `cc://con1alice/posts/2025-11-23/hello`
* `cc://con1alice/` （`<key>` が空の場合）


## 10. net.concrnt.core.entity エンドポイント

`net.concrnt.core.entity` エンドポイントは、エンティティの情報（少なくとも最新の Affiliation）を取得するために使用されます。

### 10.1 テンプレート例

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

### 10.2 レスポンスの例

サーバーは、レスポンスに最低限次の要素が含まれる JSON を返却 **しなければなりません (MUST)**。

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


## 11. net.concrnt.core.resource エンドポイント

`net.concrnt.core.resource` エンドポイントは、CCURI に対応するリソースを取得するために使用されます。
### 11.1 net.concrnt.core.resource の解決例

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

## 11.2 レスポンスの例
サーバーは、Acceptヘッダが`application/json`であった場合、次のようなJSONレスポンスをHTTPステータス200で返却する**MUST**。

```json
{
  "contentType": "application/json",
  "schema": "https://schema.concrnt.net/resource.json",
  "value": { ... },
  "apis": {}
}
```

その他のAcceptヘッダが指定された場合、サーバーは対応するMIMEタイプでリソースの生データを返却することができます。

## 12. Security Considerations

**鍵管理**

* 秘密鍵はクライアント側で生成・保持されるべきであり、サーバに送信してはならない (MUST NOT)。
* CCID は公開鍵から導出されるため、秘密鍵の漏洩はエンティティの乗っ取りに直結します。
  実装者は鍵生成・保管・バックアップについて十分に注意する必要があります。

## 13. Abuse Potential

Concrnt Core 自体は単に名前解決の枠組みを提供するのみであり、
スパム・嫌がらせ・違法コンテンツ等の問題は、主に上位プロトコルや運用ポリシーにおける課題となります。

## 14. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* BIP32, BIP44 – Hierarchical Deterministic Wallets

