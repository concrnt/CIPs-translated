# CIP-0: Core

## 0. Abstract

Concrnt は、分散型 Social Networking Service (SNS) を構築するためのプロトコル群である。
本 CIP-0 (Core) は、その最小コアとして、暗号学的なエンティティ識別子 (CCID)、エンティティとサーバーの
所属関係 (Affiliation)、および `ccfs://` / `cckv://` スキームによる Concrnt リソース URI (CCURI) の
名前解決を定義する。

## 1. Status of This Memo

このドキュメントは Concrnt Core プロトコルの仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 著作権表示 (Copyright Notice)

Copyright (c) 2025 Concrnt Project and the contributors.

本仕様書は [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)
ライセンスのもとでパブリックドメインに提供される。
誰でも、複製・改変・再配布・商用利用を含め自由に利用できる。一切の保証はない。

## 3. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 4. イントロダクション (Introduction)

ソーシャルメディアのアカウントは、個人・組織のデジタルアイデンティティの中核になりつつある。
しかし、中央集権型のプラットフォームには次のような問題がある。

- アカウント停止や BAN が不可逆的である
- 第三者による一方的な検閲
- 構築したつながりやコンテンツを失うリスク
- データの所在や制御に対するユーザーの主権不足

ActivityPub などの分散プロトコルは一定の改善を提供するものの、依然として次のような課題が存在する。

- アカウント移行がサーバーの協力に依存している
- BAN された時点で移行の道が断たれる
- サーバーローカルなタイムラインがコミュニティを分断する

Concrnt は、暗号学的なアイデンティティと、サーバーに依存しないタイムライン／コミュニティ構造を基盤として、
これらの課題を解決することを目的とする。本 CIP-0 は、そのうちアイデンティティと名前解決の部分のみを定義する。

### 4.1 本仕様群の構成と適合性

Concrnt のプロトコルは CIP (Concrnt Improvement Proposal) 群として定義される。
適合性の要件は次の通りである。

* **必須 (REQUIRED)**: 本 CIP-0 の全体、および CIP-1 のコアサブセット
  (Document 構造・CDID・`concrnt-ecrecover-direct` / `none` proof。CIP-1 §3 参照)。
  Concrnt サーバーを名乗る実装は、これらを実装しなければならない (MUST)。
* **拡張 (OPTIONAL)**: CIP-2 以降のすべての CIP はオプショナルな拡張である。
  各 CIP は冒頭で自身が依存する CIP を宣言する。

拡張の実装有無は、サービスディスカバリ (§9.1) の `endpoints` マップにおけるエンドポイント名の広告に
よって判別する。これが機能検出の正規の手段である。クライアントは、広告されていないエンドポイントを
呼び出すべきではなく (SHOULD NOT)、相手が当該拡張を実装していないことを前提に動作しなければならない (MUST)。

以下は本仕様群の意図的なスコープ外であり、実装・運用の自由とする。

* エラーレスポンスのボディ形式。エラーは HTTP ステータスコードによって表現され、
  ボディの形式は実装定義である。
* エンティティの新規登録、およびリモートエンティティの所属受け入れ (admission) の可否判断・ワークフロー。
* 通報 (CIP-15) への対応・保持等の運用。
* blob のアップロード (取り込み) 手段。CIP 群が定義するのは blob の命名と解決 (§7.1, §9.3.3.4) のみである。

## 5. Entities と CCID

Entity (エンティティ) は、Concrnt エコシステムにおける個人・組織・アプリケーションなどの主体を表す。
エンティティは `secp256k1` 曲線に基づく暗号学的な鍵ペアによって識別される。
異なる鍵ペアのエンティティは、異なる主体を表す。

### 5.1 鍵ペアの生成

Concrnt は鍵生成アルゴリズムそのものを規定しない。鍵導出パスは相互運用要件ではないが、
HD Wallet (BIP32/BIP44) に準拠する場合、クライアント間のシード互換のため次のパスを使用するべきである (SHOULD)。

```text
m/44'/118'/0'/0/0
```

### 5.2 CCID

CCID はエンティティを一意に識別するための識別子であり、公開鍵から次の手順で導出される。

1. 公開鍵の圧縮表現 (33バイト) に対して SHA256 を適用する。
2. その結果に RIPEMD160 を適用し、20バイトのアカウントアドレスを得る。
3. このアドレスを、"con" を HRP (Human Readable Part) とする Bech32 でエンコードする。

これは Cosmos SDK のアカウントアドレス導出と同一の手順である。結果として CCID は 42 文字の文字列になる。

CCID・CSID は小文字表現のみを正とする (MUST)。大文字を含む識別子を持つ Document・トークンは
拒否しなければならない (MUST)。識別子の比較は常にバイト同一で行う。

```text
con1t0tey8uxhkqkd4wcp4hd4jedt7f0vfhk29xdd2
```

## 6. CSID

CSID は Concrnt サーバーを一意に識別するための識別子である。サーバーの公開鍵から §5.2 と同一の手順で
導出したアカウントアドレスを、"ccs" を HRP とする Bech32 でエンコードして表現する。

```text
ccs16djx38r2qx8j49fx53ewugl90t3y6ndgye8ykt
```

## 7. CCURI

CCURI (Concrnt Resource Identifier) は、Concrnt エコシステム内のリソースを指し示すための URI スキームである。
CCURI は 2 つの形式で表現される。

### 7.1 ccfs:// スキーム

ccfs:// スキームは、コンテンツを一意のハッシュで指し示すために使用される。
パスの第 1 セグメントでコンテンツの種別 (type) を、第 2 セグメントで識別子を示す。

```text
ccfs://<owner>/concrnt/<cdid>
ccfs://<owner>/blob/<sha256 hash>
```

* `concrnt` 種別
  CIP-1 で定義される Concrnt Document を、その CDID で指し示す。

* `blob` 種別
  バイナリ形式のファイルを、その内容の SHA256 ハッシュで指し示す。
  ハッシュ値は 64 文字の小文字16進数文字列でなければならない (MUST)。

`<owner>` には通常、リソースの所有者の CCID を指定する。cckv の FQDN / CSID 名前空間 (§7.2) に属する
Document の ccfs URI では、owner 部にその FQDN / CSID をそのまま用いる (CIP-3 §3.3 参照)。
リゾルバヒント (§7.2) を付与した `ccfs://<owner>@<FQDN>/...` の形式も使用できる。

### 7.2 cckv:// スキーム

cckv:// スキームは、コンテンツをキー・バリュー形式で指し示すために使用される。

```text
cckv://<CCID>
cckv://<CCID>/<key>
cckv://<FQDN>
cckv://<FQDN>/<key>
cckv://@<alias FQDN>
cckv://@<alias FQDN>/<key>

cckv://<CCID>@<resolver FQDN>
cckv://<CCID>@<resolver FQDN>/<key>
```

CCURI は owner 部と key 部から構成される。owner 部はリソースの所有者を示し、
key 部はその所有者の名前空間内でのリソースの位置を示す。

owner 部は次のいずれかである。

* `<CCID>` — エンティティを所有者とする。
* `<FQDN>` — サーバーそのものを指す。この形式の解決結果は §9.3.3.2 を参照。
* `@<alias FQDN>` — エイリアスによるエンティティ指定。DNS を介して CCID に解決される (§8.3 参照)。

`<CCID>@<resolver FQDN>` の形式では、`@` の後ろに **リゾルバヒント** として FQDN を付与できる。
クライアントはこのヒントを、owner の所属サーバーを解決する際の初期候補として使用してもよい (MAY) が、
ヒントを信頼の根拠としてはならない (MUST NOT)。リソースの検証は常に署名によって行われる。

key 部の規則は次の通りである。

* key 部が省略された場合、リソースではなくエンティティ (またはサーバー) そのものを指し示す。
* key 部は 1 文字以上・1024 バイト以下の UTF-8 文字列でなければならない (MUST)。
  長さはスキーム・owner 部を含まない key 部の UTF-8 バイト数で計測する。
* key 部に `*` を含めてはならない (MUST NOT)。`*` は範囲削除 (CIP-4) の範囲指定子として予約される。
* key のパス階層の区切りはバイト `0x2F` (`/`) のみとする。階層の判定・マッチングは
  生のバイト列に対して行い、事前にパーセントデコード等の変換を行ってはならない (MUST NOT)。

## 8. Entity Document

### 8.1 Entity Document の構造

エンティティは、自身が特定のサーバーに所属していることを示す Entity Document を発行できる。

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
  所属先サーバーの FQDN。

* `value.alias` (OPTIONAL)
  エンティティに人間可読な別名 (FQDN 形式) を与える。§8.3 参照。

* `value.alias_proof_type` (OPTIONAL)
  alias の所有権を証明する方式の識別子。本仕様は `"dns"` (§8.3) のみを定義する。

* `createdAt`
  Entity Document が署名された時刻 (UTC, RFC3339 形式。CIP-1 §5.6 の正規形に従う)。

### 8.2 Entity Document の署名

Entity Document は JSON 文字列としてシリアライズされ、エンティティの秘密鍵で署名される。
署名の外側の構造は次の通りである (CIP-1 の Signed Document 形式)。

```json
{
  "document": "<JSON string above>",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

署名対象文字列を Keccak256 でハッシュ化し、対象エンティティの秘密鍵で secp256k1 ECDSA 署名を行う。
`signature` フィールドには、署名の **(r, s, v)** をこの順で連結した 65 バイトを16進エンコードして格納する。
`v` は末尾 1 バイトの recovery id であり、値は 0 または 1 でなければならない (MUST)。
検証者は 27 / 28 を受理する場合、27 を減じて正規化してもよい (MAY)。

検証には ECRECOVER アルゴリズムを使用し、署名から公開鍵を復元して §5.2 の手順でアドレスを導出し、
CCID と照合する。

Entity Document の proof は `concrnt-ecrecover-direct` でなければならない (MUST)。
サブキー (CIP-13) や document-reference (CIP-6) による proof を持つ Entity Document を
受理してはならない (MUST NOT)。所属の表明はアカウントレベルの操作であり、また Entity Document は
backdate window の適用外である (CIP-3 §3.4) ため、サブキー署名を許すと、漏洩後に失効された
サブキーによる過去日時への所属偽造 (CIP-13 §8) を時間で制限できなくなる。

### 8.3 エイリアスと DNS による所有権検証

エンティティは `value.alias` に FQDN 形式の別名を設定できる。
alias が設定された Entity Document を受理するサーバーは、次の手順で所有権を検証しなければならない (MUST)。

1. `_concrnt.<alias>` の DNS TXT レコードを取得する。
2. TXT レコードのうち、CCURI としてパース可能で、その owner が Entity Document の `author` と一致する
   ものが存在することを確認する。

クライアントは `cckv://@<alias FQDN>` 形式の CCURI を、同様に `_concrnt.<alias>` の TXT レコードに
含まれる CCURI (リゾルバヒント付きの CCID 形式) を介して解決できる。
複数の TXT レコードが互いに異なる owner を指す場合、解決は失敗として扱うべきである (SHOULD)。

```text
_concrnt.alice.example.net. IN TXT "cckv://con1t0te...@example.com"
```

### 8.4 Entity Document の公開

サーバーは、所属しているエンティティの Entity Document を保持し、
`net.concrnt.core.resolve` エンドポイントを通じて提供しなければならない (MUST)。

サーバーは自身のローカルユーザーだけでなく、他の手段 (フェデレーション、キャッシュなど) で取得した
Entity Document を保存し、提供してもよい (MAY)。

同一エンティティ (同一 CCID) に対して複数の Entity Document を受け取った場合の新旧判定・保存規則は、
CIP-3 §3.4 の accept-if-newer に従う (より新しいもののみを保存し、古い・同一の再送は no-op 成功とする)。

### 8.5 未知のエンティティの解決

Concrnt にはグローバルなディレクトリが存在しないため、エンティティの所属は
署名済み Entity Document の `value.domain` からのみ得られる。
未知の CCID を解決しようとするサーバー・クライアントは、次の順序で候補を試みるべきである (SHOULD)。

1. ローカルに保存・キャッシュ済みの Entity Document。
2. CCURI に付与されたリゾルバヒント (`@<FQDN>`、§7.2)、またはリクエストに同梱された
   Entity Document (CIP-1 の `references` 等) から得たドメインを初期候補として、
   その候補サーバーの resolve エンドポイントから Entity Document を取得する。

いずれの経路で取得した場合も、Entity Document は proof の検証を通過し、`author` が要求した CCID と
一致しなければならない (MUST)。ヒント自体を信頼の根拠としてはならない (MUST NOT、§7.2)。
検証を通過した Entity Document はキャッシュしてよく (MAY)、以後はヒントなしの裸の CCID も解決できる。

上記の候補がすべて存在しない場合、**裸の未知の CCID はプロトコル上解決不能**であり、
解決の失敗として扱う。ヒントの伝搬 (リゾルバヒント付き CCURI の使用、Entity Document の同梱) は、
この初回接触問題を緩和するためにクライアント・サーバー双方が行うべき (SHOULD) 慣行である。

## 9. サーバー

Concrnt サーバーは、Concrnt エコシステム内でリソースをホストし、配信する役割を担う。

### 9.1 サービスディスカバリ: .well-known/concrnt

サーバーは次の URL で Concrnt をサポートしていることを広告する。

```text
GET https://<domain>/.well-known/concrnt
```

レスポンスは JSON で、少なくとも以下の要素を含まなければならない (MUST)。

```json
{
  "version": "2.0",
  "domain": "example.com",
  "csid": "ccs1<bech32-encoded-address>",
  "layer": "mainnet",
  "endpoints": {
    "net.concrnt.core.resolve": "/resource/{uri}"
  }
}
```

* `version`
  Concrnt プロトコルのメジャーバージョン。現段階では `"2.0"` を使用する。

* `domain`
  サーバーの FQDN。

* `csid`
  サーバーの CSID。

* `layer`
  そのサーバーが所属するネットワーク名。慣用的に `"mainnet"`、`"testnet"` などが使用されるが、
  これに限られない。サーバーは、この識別子が一致する場合のみ相手サーバーとリソースのやり取りを
  行わなければならない (MUST)。

* `endpoints`
  サーバーが提供するエンドポイント名と、その URL テンプレートのマッピング (§9.3.1)。

サーバーは、少なくとも `net.concrnt.core.resolve` エンドポイントを実装しなければならない (MUST)。
他のエンドポイントは、各拡張 CIP によって定義される (§4.1)。

### 9.2 エンドポイント名の命名規約

`endpoints` マップのキー (エンドポイント名) は、名前の衝突を避けるため、
命名者が保有するドメイン名の逆ドメイン記法 (Reverse Domain Name Notation) を
接頭辞として使用しなければならない (MUST)。

例: `example.com` の保有者は `com.example.` で始まるエンドポイント名を定義できる。

### 9.3 net.concrnt.core.resolve エンドポイント

`net.concrnt.core.resolve` エンドポイントは、CCURI で指し示されるリソース
(エンティティ情報、サーバー情報、Concrnt Document、blob) を取得するために使用される。

#### 9.3.1 テンプレート構文

`endpoints` の値は URI テンプレートであり、その構文は RFC 6570 に従わなければならない (MUST)。
ただし RFC 6570 は構文の参照のみであり、展開は次の文字どおりの置換で行わなければならない (MUST)。

* `{uri}` — 完全な CCURI を、RFC 3986 の unreserved 文字以外すべてパーセントエンコードした文字列に置換する。
* `{owner}` — CCURI の owner 部 (`con1...` など) をそのまま置換する。
* `{key}` — CCURI の key 部 (先頭の `/` を除いたパス) を、各セグメントをパーセントエンコードし、
  セグメント区切りの `/` は保持した文字列に置換する。

テンプレートは、相対パス (`"/api/v2/resolve?uri={uri}"`) または
絶対 URL (`"https://cdn.example.com/{owner}/{key}"`) のいずれの形式でもよい (MAY)。

サーバーは、上記以外のプレースホルダを使用すべきではない (SHOULD NOT)。
クライアントは、仕様で定義されていないプレースホルダを含むエンドポイントを利用しなくてもよい (MAY)。

#### 9.3.2 テンプレートとリクエストの例

**例1: 動的 API サーバー** — `"net.concrnt.core.resolve": "/api/v2/resolve?uri={uri}"`

`cckv://con1alice/keys/profile` を解決する場合:

```text
GET https://example.com/api/v2/resolve?uri=cckv%3A%2F%2Fcon1alice%2Fkeys%2Fprofile
```

**例2: 静的ホスティングレイアウト** — `"net.concrnt.core.resolve": "https://static.example.com/cckv/{owner}/{key}"`

`cckv://con1alice/posts/2025-11-23/hello` を解決する場合:

```text
GET https://static.example.com/cckv/con1alice/posts/2025-11-23/hello
```

#### 9.3.3 レスポンス

要求されたリソースが存在しない場合、サーバーは HTTP 404 Not Found を返さなければならない (MUST)。
サーバーは、リソースの所在に応じて HTTP 302 等のリダイレクトで応答してもよく (MAY)、
クライアントはリダイレクトに追従できなければならない (MUST)。

Accept ヘッダによって、リソースの別表現を要求できる場合がある (例: CIP-8 の `application/chunkline+json`)。
対応する表現の詳細は各 CIP で定義される。

##### 9.3.3.1 エンティティ情報の返却

key 部を持たない CCID 形式の CCURI (`cckv://<CCID>`) が要求された場合、サーバーは
そのエンティティの最新の Entity Document を、CIP-1 で定義される Signed Document 形式で
返却しなければならない (MUST)。

```json
{
  "document": "{...Entity Document...}",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

サーバーは追加のフィールド (`cckv`, `ccfs`, `references` など。CIP-1 参照) を含めてもよい (MAY)。

##### 9.3.3.2 サーバー情報の返却

owner 部が FQDN または既知の CSID である CCURI が要求された場合、サーバーは次のように
キャッシュしているサーバー記述子を返却するべきである (SHOULD)。

```json
{
  "tag": "",
  "wellKnown": {
    "version": "2.0",
    "domain": "example.com",
    "csid": "ccs1<bech32-encoded-address>",
    "layer": "mainnet",
    "endpoints": { }
  }
}
```

* `wellKnown`
  対象サーバーの well-known 情報 (§9.1 と同形式)。

* `tag` (OPTIONAL)
  リクエスト先サーバーが対象サーバーに付与している管理タグ。

##### 9.3.3.3 リソースの返却

その他の CCURI (key 部を持つ cckv、`concrnt` 種別の ccfs) が要求された場合、サーバーは
Signed Document 形式で署名されたリソースを返却する。

```json
{
  "document": "{...}",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

##### 9.3.3.4 blob の返却

`blob` 種別の ccfs URI が要求された場合、サーバーは blob の実体を返却するか、
実体を保持するストレージエンドポイントまたは owner の所属サーバーへの HTTP リダイレクトで
応答してもよい (MAY)。取得側は、取得したコンテンツが URI 中の SHA256 ハッシュと一致することを
検証するべきである (SHOULD)。blob のアップロード手段は本仕様のスコープ外である (§4.1)。

## 10. Security Considerations

* 秘密鍵はクライアント側で生成・保持するべきであり (SHOULD)、サーバーに送信してはならない (MUST NOT)。
* CCID は公開鍵から導出されるため、秘密鍵の漏洩はエンティティの乗っ取りに直結する。
  実装者は鍵生成・保管・バックアップについて十分に注意する必要がある。
* `.well-known/concrnt` 文書自体は署名されていない。ドメインとの結びつきは TLS (Web PKI) にのみ
  依存する。エンティティの帰属の信頼の根拠は、あくまで署名済み Entity Document の検証
  (§8.2, §8.5) であり、well-known 文書の記載を単独で信頼の根拠としてはならない。
  リソースをやり取りする前の `layer` 一致の検査 (§9.1) を省略してはならない。

## 11. Abuse Potential

Concrnt Core は名前解決の枠組みを提供するのみであり、スパム・嫌がらせ・違法コンテンツ等の問題は、
主に上位プロトコルや運用ポリシーにおける課題となる。

## 12. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 3986 – Uniform Resource Identifier (URI): Generic Syntax
* RFC 6570 – URI Template
* BIP32, BIP44 – Hierarchical Deterministic Wallets
* CIP-1 – Concrnt Document System
