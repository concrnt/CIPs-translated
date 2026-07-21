# CIP-2 Auth

## 0. Abstract

本ドキュメントでは、ConcrntサーバーのAPIに対するリクエストにおいて、リクエスタが特定のEntity (CIP-0) 本人であることを証明するための認証トークン (auth token) の形式・作成方法・検証方法を定義する。

## 1. Status of This Memo

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、
実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Introduction

Concrnt Signed Document (CIP-1) の署名は、Document そのものの真正性を証明する。
一方、リソースの読み取り (CIP-5) やアクセス制御 (CIP-12)、通報 (CIP-15) など、
Document の提出を伴わないAPI呼び出しにおいては、「誰がリクエストしているか」を
HTTPリクエストのレベルで示す必要がある。auth token はこのための手段である。

認証は任意である。auth token を持たないリクエストは、ゲスト (未認証のリクエスタ) として扱われる。

## 4. トークンの形式

auth token は JWT [RFC7519] の構造 (`base64url(header).base64url(payload).base64url(signature)`) を持つが、
署名アルゴリズムとして Concrnt 独自の `CONCRNT` を使用する。

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
* `kid` (optional) — 署名に使用する鍵を指す cckv URI。
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

* `iss` (required) — リクエスタのEntityのCCID。
* `sub` (required) — 常に `"concrnt"`。
* `aud` (required) — トークンの宛先サーバー。サーバーのFQDNまたはCSID (CIP-0)。
* `iat` (optional) — 発行時刻。UNIXタイムスタンプ (秒) の10進文字列。
* `exp` (optional) — 失効時刻。UNIXタイムスタンプ (秒) の10進文字列。設定するべきである (SHOULD)。
* `jti` (optional) — トークンの一意識別子。

`iat` / `exp` は JWT の慣例 (数値) と異なり**文字列**である点に注意。

### 4.3 署名

署名対象は `base64url(header) + "." + base64url(payload)` のASCII文字列である。
これを Keccak256 でハッシュ化し、secp256k1 のECDSA署名を行う。
署名は (r, s, v) の順で連結した65バイト (CIP-0 §8.2 と同一形式) を base64url エンコードしたものである。

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
2. `aud` が自身のFQDNまたはCSIDと一致することを確認する。
3. `sub` が `"concrnt"` であることを確認する。
4. `iss` のEntityを解決できることを確認する。
5. 鍵の特定: `kid` が省略されている場合は `iss` を鍵URIとして扱う。
   `kid` (または `iss`) を cckv URI としてパースし、その owner 部が `iss` のCCIDと一致することを確認する (MUST)。
   このバインディングがない場合、ある鍵の正当な署名で任意の `iss` を名乗ることができてしまう。
   * **key部が空の場合 (master key)**: 署名からECRECOVERで公開鍵を復元し、
     導出したアドレスが `iss` のCCIDと一致することを確認する。
   * **key部がある場合 (subkey)**: そのURIから Subkey Enact Document (CIP-13) を権威サーバーから
     署名検証付きで取得し、`schema` が `https://schema.concrnt.net/subkey.json` であること、
     `author` が `iss` のCCIDと一致することを確認したうえで、`value.ckid` に対して署名を検証する。
     解決結果が Revoked Subkey Document (CIP-13 §4) である場合、そのサブキーは失効済みであり、
     トークンの検証は失敗する (認証に使用できるのは現在有効なサブキーのみ)。
6. `exp` が設定されている場合、現在時刻が `exp` を過ぎていないことを確認する。

検証に成功した場合、サーバーはこのリクエストのリクエスタを `iss` のEntityとして扱う。

検証に失敗した場合、サーバーはリクエストを拒否する代わりに、未認証 (ゲスト) のリクエストとして
処理を続行してもよい (MAY)。ただし、`iss` のEntityまたはその所属サーバーをブロックしている場合は、
HTTP 403 Forbidden で拒否するべきである (SHOULD)。

## 6. サーバー発行トークン (サービスアカウント)

サーバー運用者は、自身のサーバーに対する管理操作 (リポジトリのインポート等、CIP-3 §3.6 参照) のために、
サーバー自身の鍵で署名したトークンを使用できる。

* `iss` および `aud` にはサーバー自身のCSIDを指定する。
* `sub` にはサービスアカウントの種別を表す文字列 (例: `"system"`) を指定する。

サーバーは、`iss` が自身のCSIDと一致するトークンを受理した場合、自身の公開鍵 (CSID) で署名を検証し、
`sub` の値をサービスアカウント種別としてリクエストに関連付ける。

サービスアカウントの権限は極めて強い (CIP-3 の検証バイパス等) ため、
このトークンは運用者の管理経路以外に発行してはならない (MUST NOT)。

## 7. Security Considerations

* auth token の漏洩は、`exp` までの間そのEntityへのなりすましを許す。トークンは秘密情報として扱い、
  TLS で保護された経路でのみ送信しなければならない (MUST)。
* `exp` を持たないトークンは無期限に有効となる。クライアントは短い有効期限を設定し、
  必要に応じて再発行するべきである (SHOULD)。
* `aud` のバインディングにより、あるサーバー向けに発行されたトークンを他のサーバーへ
  転用することはできない。サーバーは `aud` の検証を省略してはならない (MUST NOT)。
* subkey で署名されたトークンの有効性は、Enact Document が現在有効であること (Revoked Subkey Document
  で上書きされていないこと) に依存する (CIP-13)。
  解決結果をキャッシュする場合、失効の反映が遅延するリスクを考慮し、
  キャッシュ期間を適切に制限するべきである (SHOULD)。

## 8. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 7519 – JSON Web Token (JWT) (構造の参考。署名アルゴリズムは独自)
