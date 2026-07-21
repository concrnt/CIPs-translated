# CIP-0: Core

## 0. Abstract

Concrnt は、分散型 Social Networking Service (SNS) を構築するためのプロトコル群です。
Concrnt Core では、最小プロトコルとして暗号学的なエンティティ識別子（CCID）、エンティティとサーバの所属関係（Affiliation）、および ccfs://あるいはcckv:// スキームによる Concrnt リソース URI（CCURI）の名前解決を定義します。

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

CCID は Entity を一意に識別するための識別子です。

CCID は、Entity の公開鍵から次の手順で導出されます。

1. 公開鍵の圧縮表現 (33バイト) に対して SHA256 を適用する。
2. その結果に RIPEMD160 を適用し、20バイトのアカウントアドレスを得る。
3. このアドレスを、"con" を HRP (Human Readable Part) とする Bech32 でエンコードする。

これは Cosmos SDK のアカウントアドレス導出と同一の手順です。
結果として CCID は 42 文字の文字列になります。

次のテキストは、CCIDの一例です。

```text
con1t0tey8uxhkqkd4wcp4hd4jedt7f0vfhk29xdd2
```
## 6. CSID

CSIDはConcrntサーバを一意に識別するための識別子です。サーバの公開鍵から 5.2 と同一の手順で導出したアカウントアドレスを、"ccs" を HRP とする Bech32 でエンコードして表現します。
次のテキストは、CSIDの一例です。

```text
ccs16djx38r2qx8j49fx53ewugl90t3y6ndgye8ykt
```

## 7 CCURI

CCURI (Concrnt Resource Identifier) は、Concrnt エコシステム内のリソースを指し示すための URI スキームです。
CCURIは2つの形式で表現されます。

### 7.1 ccfs:// スキーム
ccfs:// スキームは、コンテンツを一意のハッシュで指し示すために使用されます。
パスの第1セグメントでコンテンツの種別 (type) を示し、第2セグメントでハッシュ値を示します。

```text
ccfs://<owner>/concrnt/<cdid>
ccfs://<owner>/blob/<sha256 hash>
```

* `concrnt` 種別
  CIP-1 で定義される Concrnt Document を、その CDID で指し示します。

* `blob` 種別
  バイナリ形式のファイルを、その内容の SHA256 ハッシュで指し示します。
  ハッシュ値は 64 文字の16進数文字列でなければなりません (MUST)。

`<owner>` にはリソースの所有者の CCID を指定します。
また、リゾルバヒント (7.2 参照) を付与した `ccfs://<owner>@<FQDN>/...` の形式も使用できます。


### 7.2 cckv:// スキーム

cckv:// スキームは、コンテンツをキー・バリュー形式で指し示すために使用されます。

```text
cckv://<CCID>
cckv://<FQDN>
cckv://<CCID>/<key>
cckv://<FQDN>/<key>

cckv://<CCID>@<resolver FQDN>
cckv://<CCID>@<resolver FQDN>/<key>
```

CCURIは、owner部とkey部から構成されます。owner部はリソースの所有者を示し、key部はその所有者の名前空間内でのリソースの位置を示します。

owner部は次のいずれかです。

* `<CCID>` — エンティティを所有者とする。
* `<FQDN>` — サーバそのものを指す。この形式の解決結果は 9.3.3.2 を参照。
* `@<alias FQDN>` — エイリアスによるエンティティ指定。DNS を介して CCID に解決される (8.3 参照)。

`<CCID>@<resolver FQDN>` の形式では、`@` の後ろに **リゾルバヒント** として FQDN を付与できます。
クライアントはこのヒントを、owner の所属サーバを解決する際の初期候補として使用してもよい (MAY) ですが、
ヒントを信頼の根拠としてはなりません (MUST NOT)。リソースの検証は常に署名によって行われます。

key部が省略された場合、リソースではなくエンティティ（またはサーバ）そのものを指し示します。
keyが指定される場合、keyは1文字以上・最大1024バイトのUTF-8文字列であるべきです (SHOULD)。

## 8. Entity Document

### 8.1 Entity Document の構造
エンティティは、自身が特定のサーバに所属していることを示す Entity Document を発行できます。

Entity Document の JSON 形式:

```json
{
  "kind": "entity",
  "author": "con1<bech32-encoded-address>",
  "schema": "https://schema.concrnt.net/entity.json",
  "value": {
    "domain": "example.com",
    "alias": "alice.example.net",
    "alias_proof_type": "dns"
  },
  "createdAt": "2025-11-23T12:34:56Z"
}
```

* `kind`
  常に `"entity"` を指定する (CIP-1 参照)。

* `author`
  Entity Document を発行したエンティティの CCID。

* `schema`
  常に `"https://schema.concrnt.net/entity.json"` を指定する。

* `value.domain`
  所属先サーバの FQDN。

* `value.alias` (optional)
  エンティティに人間可読な別名 (FQDN 形式) を与える。8.3 参照。

* `value.alias_proof_type` (optional)
  alias の所有権を証明する方式の識別子。

* `createdAt`
  Entity Document が署名された時刻（UTC, RFC3339 形式）。

### 8.2 Entity Document の署名

Entity Document は JSON 文字列としてシリアライズされ、エンティティの秘密鍵で署名されます。

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
signatureフィールドには、署名の **(r, s, v)** をこの順で連結した65バイトを16進エンコードして格納します
(v は末尾1バイトの recovery id です)。

検証には、ECRECOVER アルゴリズムを使用し、署名から公開鍵を復元して 5.2 の手順でアドレスを導出し、CCID と照合します。

### 8.3 エイリアスと DNS による所有権検証

エンティティは `value.alias` に FQDN 形式の別名を設定できます。
alias が設定された Entity Document を受理するサーバは、次の手順で所有権を検証しなければなりません (MUST)。

1. `_concrnt.<alias>` の DNS TXT レコードを取得する。
2. TXT レコードのうち、CCURI としてパース可能で、その owner が Entity Document の `author` と一致するものが存在することを確認する。

クライアントは `cckv://@<alias FQDN>` 形式の CCURI を、同様に `_concrnt.<alias>` の TXT レコードに含まれる
CCURI（リゾルバヒント付きの CCID 形式）を介して解決できます。

TXT レコードの例:

```text
_concrnt.alice.example.net. IN TXT "cckv://con1t0te...@example.com"
```

### 8.4 Entity Document の公開

サーバは、所属しているエンティティの Entity Document を保持し、
`net.concrnt.core.resolve` エンドポイントを通じて提供しなければなりません (MUST)。

サーバは自身のローカルユーザーだけでなく、他の手段（フェデレーション、キャッシュなど）で取得した Entity Document を保存し、提供してもよい (MAY)。

サーバは同一エンティティ (同一 CCID) に対して複数の Entity Document を受け取った場合、より新しいもの
（`createdAt` を先頭に持つ CDID の辞書順比較で判定できる）を優先して保存しなければなりません (MUST)。
古い、あるいは同一の Entity Document の再送は、エラーとせず no-op として成功応答してよい (MAY)。
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
  "domain": "example.com"
  "csid": "ccs1<bech32-encoded-address>",
  "layer": "mainnet"
  "endpoints": {
    "net.concrnt.core.resolve": "/resource/{uri}"
  }
}
```

* `version`
  Concrnt プロトコルのメジャーバージョン。現段階では `"2.0"` を使用する。

* `domain`
  サーバの FQDN。

* `csid`
  サーバの CSID。

* `layer`
  そのサーバーが所属するネットワーク名。慣用的に `"mainnet"`、`"testnet"` などが使用されるが、これに限られない。
  あるサーバーはこの識別子を見て、同一である場合のみそのサーバーとのリソースのやり取りを行わなければならない (MUST)。

* `endpoints`
  サーバが提供するエンドポイント名と、その URL テンプレートのマッピング。URIテンプレートの形式は、RFC6570に従います。

サーバーは、少なくとも`net.concrnt.core.resolve`エンドポイントを実装しなければなりません (MUST)。

他のエンドポイントは、別の CIP によって定義されます。

### 9.2 エンドポイント名の命名規約

`endpoints` マップのキー（エンドポイント名）は、名前の衝突を避けるため、
命名者が保有するドメイン名の逆ドメイン記法 (Reverse Domain Name Notation) を
接頭辞として使用しなければなりません (MUST)。

例: `example.com` の保有者は `com.example.` で始まるエンドポイント名を定義できます。

### 9.3 net.concrnt.core.resolve エンドポイント

`net.concrnt.core.resolve` エンドポイントは、CCURI で指し示されるリソース（エンティティ情報、サーバ情報、Concrnt Document、blob）を取得するために使用されます。

#### 9.3.1 テンプレート構文

`endpoints` の値は URI テンプレートであり、その構文は RFC 6570 に従わなければなりません (MUST)。

テンプレートは、次のいずれかの形式を取ることができます (MAY)。

* 相対パス: `"/api/v2/resolve?uri={uri}"`
* 絶対 URL: `"https://cdn.example.com/{owner}/{key}"`

テンプレート内では、以下のプレースホルダを使用できます。

* `{uri}`
  完全な CCURI を URL エンコードした文字列。

* `{owner}`
  CCURI の owner 部分 (`con1...` など)。通常は URL エンコード不要。

* `{key}`
  CCURI の `<key>` 部分（先頭の `/` を除いたパス）。
  必要に応じてクライアントが URL エンコードして埋め込みます。

サーバは、未知のプレースホルダを使用すべきではありません (SHOULD NOT)。
クライアントは、仕様で定義されていないプレースホルダを見つけた場合、そのエンドポイントを利用しないことができます。

#### 9.3.2 テンプレートとリクエストの例

**例1: 動的 API サーバ**

```json
{
  "endpoints": {
    "net.concrnt.core.resolve": "/api/v2/resolve?uri={uri}"
  }
}
```

この場合、クライアントは次のようにリクエストを構築します。

`cckv://con1alice/keys/profile` を解決する場合:

1. クライアントは `cckv://con1alice/keys/profile` を URL エンコードする。
`cckv%3A%2F%2Fcon1alice%2Fkeys%2Fprofile`

2. テンプレートに埋め込む。

```text
GET https://example.com/api/v2/resolve?uri=cckv%3A%2F%2Fcon1alice%2Fkeys%2Fprofile
```

**例2: 静的ホスティングレイアウト**

```json
{
  "endpoints": {
    "net.concrnt.core.resolve": "https://static.example.com/cckv/{owner}/{key}"
  }
}
```

`cckv://con1alice/posts/2025-11-23/hello` を解決する場合:

```text
GET https://static.example.com/cckv/con1alice/posts/2025-11-23/hello
```

#### 9.3.3 レスポンス

##### 9.3.3.1 エンティティ情報の返却

key部を持たない CCID 形式の CCURI (`cckv://<CCID>`) が要求された場合、サーバーは
そのエンティティの最新の Entity Document を、CIP-1 で定義される Signed Document 形式で返却しなければなりません (MUST)。

```json
{
  "document": "{...Entity Document...}",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

サーバは追加のフィールド（`cckv`, `ccfs`, `references` など。CIP-1 参照）を含めてもよい (MAY)。

##### 9.3.3.2 サーバー情報の返却

owner部が FQDN または既知の CSID である CCURI が要求された場合、サーバーは次のようにキャッシュしているサーバー記述子を返却するべきです (SHOULD)。

```json
{
  "tag": "",
  "wellKnown": {
    "version": "2.0",
    "domain": "example.com",
    "csid": "ccs1<bech32-encoded-address>",
    "layer": "concrnt-mainnet",
    "endpoints": { }
  }
}
```

* `wellKnown`
  対象サーバの well-known 情報 (9.1 と同形式)。

* `tag` (optional)
  リクエスト先サーバが対象サーバに付与している管理タグ。

##### 9.3.3.3 リソースの返却

その他の CCURI（key部を持つ cckv、`concrnt` 種別の ccfs）が要求された場合、サーバーは
Signed Document 形式で署名されたリソースを返却します。

```json
{
  "document": "{...}",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

Accept ヘッダによって、リソースの別表現を要求できる場合があります（例: CIP-8 の `application/chunkline+json`）。
対応する表現の詳細は各 CIP で定義されます。

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
* RFC 6570 – URI Template
* BIP32, BIP44 – Hierarchical Deterministic Wallets

