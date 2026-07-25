# CIP-2: Auth

## 0. Abstract

本ドキュメントでは、Concrnt サーバーの API に対するリクエストにおいて、リクエスタが特定の
Entity (CIP-0) 本人であることを証明するための認証トークン (auth token) の形式・作成方法・検証方法を定義する。

## 1. Status of This Memo

このドキュメントは Concrnt の認証トークンの仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0 および CIP-1 を前提とする。
サブキーによる署名の検証には CIP-13 を必要とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Introduction

Concrnt Signed Document (CIP-1) の署名は、Document そのものの真正性を証明する。
一方、リソースの読み取り (CIP-5) やアクセス制御 (CIP-12)、通報 (CIP-15) など、
Document の提出を伴わない API 呼び出しにおいては、「誰がリクエストしているか」を
HTTP リクエストのレベルで示す必要がある。auth token はこのための手段である。

認証は任意である。auth token を持たないリクエストは、ゲスト (未認証のリクエスタ) として扱われる。

## 4. トークンの形式

auth token は JWT [RFC7519] の構造 (`base64url(header).base64url(payload).base64url(signature)`) を持つが、
署名アルゴリズムとして Concrnt 独自の `CONCRNT` を使用する。
header・payload・signature の base64url は、いずれもパディングなしでなければならない (MUST)。

### 4.1 ヘッダー

```json
{
  "alg": "CONCRNT",
  "typ": "JWT",
  "kid": "cckv://<CCID>/subkeys/<any-name>"
}
```

* `alg` — 常に `"CONCRNT"` (MUST)。§4.3 の署名方式を示す。
* `typ` — 常に `"JWT"` (MUST)。
* `kid` (OPTIONAL) — 署名に使用する鍵を指す cckv URI。
  省略された場合、`iss` のエンティティ本体の鍵 (master key) で署名されたものとして扱われる。
  詳細は §5.2。

### 4.2 クレーム

```json
{
  "iss": "con1<bech32-encoded-address>",
  "sub": "concrnt",
  "aud": "example.com",
  "iat": "1763901296",
  "exp": "1763904896",
  "jti": "<unique id>"
}
```

* `iss` (required) — リクエスタの Entity の CCID。
* `sub` (required) — 常に `"concrnt"`。
* `aud` (required) — トークンの宛先サーバー。サーバーの FQDN または CSID (CIP-0)。
* `iat` (OPTIONAL) — 発行時刻。UNIX タイムスタンプ (秒) の 10 進文字列。
* `exp` (OPTIONAL) — 失効時刻。UNIX タイムスタンプ (秒) の 10 進文字列。設定するべきである (SHOULD)。
* `jti` (OPTIONAL) — トークンの一意識別子。

`iat` / `exp` は JWT の慣例 (数値) と異なり**文字列**である点に注意。

### 4.3 署名

署名対象は `base64url(header) + "." + base64url(payload)` の ASCII 文字列である。
これを Keccak256 でハッシュ化し、secp256k1 の ECDSA 署名を行う。
署名は (r, s, v) の順で連結した 65 バイト (CIP-0 §8.2 と同一形式) を base64url エンコードしたものである。

署名に用いる鍵は、エンティティ本体の秘密鍵、または有効化済みのサブキー (CIP-13) の秘密鍵である。

## 5. リクエストへの付与と検証

### 5.1 リクエストへの付与

クライアントは、auth token を `Authorization` ヘッダーで送信する。

```text
Authorization: Bearer <token>
```

### 5.2 サーバーによる検証

サーバーは、auth token を次の手順で検証しなければならない (MUST)。

1. `typ` が `"JWT"`、`alg` が `"CONCRNT"` であることを確認する。
2. **サービスアカウントの判別**: `iss` が自身の CSID と一致する場合、このトークンは
   サービスアカウントトークン (§6) であり、以降の手順ではなく §6 の手順で検証する。
   サービスアカウントトークンの `sub` は `"concrnt"` ではないため、
   この判別は `sub` の検証より**先に**行わなければならない (MUST)。
3. `aud` が自身の FQDN または CSID と一致することを確認する。
4. `sub` が `"concrnt"` であることを確認する。
5. `iss` の Entity を解決できることを確認する。
6. 鍵の特定: `kid` が省略されている場合、`iss` (裸の CCID) を `cckv://<iss>` (key 部が空の cckv URI、
   すなわち master key) として扱う。
   `kid` (または上記のように解釈した `iss`) を cckv URI としてパースし、その owner 部が `iss` の CCID と
   一致することを確認する (MUST)。このバインディングがない場合、ある鍵の正当な署名で任意の `iss` を
   名乗ることができてしまう。
   * **key 部が空の場合 (master key)**: 署名から ECRECOVER で公開鍵を復元し、
     導出したアドレスが `iss` の CCID と一致することを確認する。
   * **key 部がある場合 (subkey)**: その URI の指す Enact Document を CIP-13 §6 の手順で解決・検証し、
     サブキーが現在有効であること、および `value.ckid` に対して署名が有効であることを確認する。
     `kid` に付与されたリゾルバヒントを解決先の決定に用いてはならない (MUST NOT。CIP-13 §6 手順1)。
     CIP-13 を実装しないサーバーは、`kid` に key 部を持つトークンの検証を失敗として扱う。
7. `exp` が設定されている場合、現在時刻が `exp` を過ぎていないことを確認する。

検証に成功した場合、サーバーはこのリクエストのリクエスタを `iss` の Entity として扱う。

検証に失敗した場合 (トークンの形式不正・署名不一致・`exp` 超過等を含む)、サーバーはリクエストを
直ちに拒否するのではなく、未認証 (ゲスト) のリクエストとして処理を続行するべきである (SHOULD)。
すなわち、認証の結果は「特定の Entity として識別された」か「ゲスト」かの 2 値であり、
不正な資格情報の提示それ自体はエラーを構成しない。
認証済みのリクエスタを必要とするエンドポイント (CIP-15 等) は、ゲストに対して
HTTP 403 Forbidden を返さなければならない (MUST)。

ただし、検証に成功したトークンの `iss` の Entity またはその所属サーバーをブロックしている場合は、
ゲストへの降格ではなく HTTP 403 Forbidden で拒否するべきである (SHOULD)。

## 6. サーバー発行トークン (サービスアカウント)

サーバー運用者は、自身のサーバーに対する管理操作のために、サーバー自身の鍵で署名したトークンを使用できる。

* `iss` および `aud` にはサーバー自身の CSID を指定する。
* `sub` にはサービスアカウントの種別を表す文字列 (例: `"system"`) を指定する。

サーバーは、`iss` が自身の CSID と一致するトークンを受理した場合、自身の公開鍵 (CSID) で署名を検証し、
`sub` の値をサービスアカウント種別としてリクエストに関連付ける。
このとき、`aud` が自身の CSID と一致すること、および `exp` が設定されている場合は失効していないことを
確認しなければならない (MUST)。

サービスアカウントに与えられる権限は実装定義であるが、コミット時の検証バイパス等の
極めて強い権限を伴いうるため、このトークンは運用者の管理経路以外に発行してはならない (MUST NOT)。

## 7. Security Considerations

* auth token の漏洩は、`exp` までの間その Entity へのなりすましを許す。トークンは秘密情報として扱い、
  TLS で保護された経路でのみ送信しなければならない (MUST)。
* `exp` を持たないトークンは無期限に有効となる。クライアントは短い有効期限を設定し、
  必要に応じて再発行するべきである (SHOULD)。
* `aud` のバインディングにより、あるサーバー向けに発行されたトークンを他のサーバーへ
  転用することはできない。サーバーは `aud` の検証を省略してはならない (MUST NOT)。
* subkey で署名されたトークンの有効性は、Enact Document が現在有効であること (Revoked Subkey Document
  で上書きされていないこと) に依存する (CIP-13)。解決結果をキャッシュする場合、失効の反映が
  遅延するリスクを考慮し、キャッシュ期間を適切に制限するべきである (SHOULD)。

## 8. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 7519 – JSON Web Token (JWT) (構造の参考。署名アルゴリズムは独自)
* CIP-0 – Concrnt Core (CCID, CSID)
* CIP-13 – Subkey (Enact Document の解決と検証)
