# CIP-18: Service Key Certificate

## 0. Abstract

本ドキュメントでは、Concrnt のエンティティ (CIP-0) が、外部のサービスが生成した鍵ペア
(サービス鍵) に対して「この鍵は、そのサービスの中で私の名義として振る舞うことを認められている」
と証明する **Service Key Certificate** の形式と検証方法を定義する。

Service Key Certificate はエンティティ本体の秘密鍵で署名された JWT であり、サービス鍵の公開鍵を
RFC 7800 の `cnf` クレームで保持する。証明書とサービス鍵はサービス側が保管し、
エンティティの Concrnt リポジトリには何も記録されない。第三者は、証明書とサービス鍵による署名を
受け取るだけで、Concrnt サーバーに問い合わせることなく「ある成果物がこの CCID の名義で作られた」
ことを検証できる。

Service Key Certificate は **CIP-13 のサブキーではない**。サービス鍵は Concrnt Document の署名や
Concrnt サーバーの API 認証に使用できず、Concrnt サーバーはこれを受理しない。

## 1. Status of This Memo

このドキュメントは Service Key Certificate の仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0、CIP-1 (§7.2 の署名方式)、CIP-2 (§4 のトークン形式) および
CIP-17 (発行の配送路) を前提とする。CIP-13 (Subkey) には依存しない。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

### サービス (Service)

Service Key Certificate の発行を受ける外部のサービス。CIP-17 の RP と同一の概念であり、
`client_id` (CIP-17 §5.1) によって識別される。

### サービス鍵 (Service Key)

サービスが生成し、秘密鍵をサービスが保管する secp256k1 の鍵ペア。本文中では `K` と記す。

### 発行者 (Issuer)

サービス鍵を承認するエンティティ。Service Key Certificate の署名者であり、
証明書の `iss` および `sub` はこのエンティティの CCID である。

### Service Key Certificate

発行者がサービス鍵に対して署名した JWT (§4)。本文中では単に「証明書」とも記す。

## 3. Introduction

サービスがエンティティの認証 (CIP-17) を受けた後、そのサービスの中でエンティティの名義で
何かを生成する場面がある。例えば、画像投稿サービスが「この画像は CCID `con1...` のユーザーによって
作成された」ことを、サービスの外にいる第三者に対して検証可能にしたい場合である。

ログインのたびにエンティティ本体の秘密鍵で個々の成果物に署名させることは、Authenticator が
別端末にある cross-device の状況では現実的でない。そこで、発行者が一度だけサービス鍵を承認し、
以後はサービスがサービス鍵で署名する。第三者は証明書とサービス鍵の署名を連鎖として検証する。

本 CIP が定めるのは **サービス鍵とエンティティの紐付け (証明書) のみ**である。
サービス鍵で何に、どのような形式で署名するかはサービスが定義する (§6)。

### 3.1 設計上の決定

* **状態はサービス側に置く。** CIP-13 のサブキーは Enact Document をエンティティのリポジトリに
  コミットするが、本 CIP は外部サービスの認証情報を Concrnt リポジトリに持ち込まない。
  証明書とサービス鍵の保管はサービスの責任である。
* **委任は一段のみ。** 証明書はエンティティ本体の秘密鍵で署名されなければならず、サービス鍵が
  さらに別の鍵に証明書を発行することはできない。
* **サービス鍵はエンティティごとに別。** 1 つのサービス鍵は 1 つの CCID にのみ紐付く (§5.2)。
* **有効期限と失効を持たない。** 証明書の効力はサービス (`aud`) の内部に閉じており、他のサービスや
  Concrnt 本体に対する名乗りには使えない。したがって本 CIP は失効機構を定義しない (§7.1)。
* **Concrnt の API アクセス権ではない。** サービス鍵は CIP-1 の proof にも CIP-2 の `kid` にも
  使用できない (§5.3)。

## 4. Service Key Certificate

### 4.1 ヘッダー

```json
{
  "alg": "CONCRNT",
  "typ": "concrnt-service-key+jwt"
}
```

* `alg` — 常に `"CONCRNT"` (MUST)。署名方式は CIP-2 §4.3 と同一である。
* `typ` — 常に `"concrnt-service-key+jwt"` (MUST)。他の JWT (CIP-2 の auth token、CIP-17 の ID Token)
  との取り違えを防ぐための明示的な型である。検証者は `typ` を確認しなければならない (MUST)。
* `kid` — 含めてはならない (MUST NOT)。証明書は常にエンティティ本体の秘密鍵で署名する。

### 4.2 クレーム

```json
{
  "iss": "con1t0tey8uxhkqkd4wcp4hd4jedt7f0vfhk29xdd2",
  "sub": "con1t0tey8uxhkqkd4wcp4hd4jedt7f0vfhk29xdd2",
  "aud": "https://example.com/auth/concrnt/callback",
  "iat": 1763901296,
  "jti": "0f8c1c2a-...",
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "secp256k1",
      "x": "...",
      "y": "..."
    }
  }
}
```

* `iss` (REQUIRED) — 発行者の CCID。
* `sub` (REQUIRED) — 発行者の CCID。`iss` と同一でなければならない (MUST)。
  サービス鍵による署名は、この CCID の名義で行われたものとして解釈される。
* `aud` (REQUIRED) — サービスの `client_id` (CIP-17 §5.1)。証明書はこの値で識別されるサービスの
  内部でのみ意味を持つ。
* `iat` (REQUIRED) — 発行時刻 (NumericDate)。
* `jti` (REQUIRED) — 証明書の一意識別子。
* `cnf` (REQUIRED) — RFC 7800 の confirmation クレーム。`jwk` メンバにサービス鍵の公開鍵を
  JWK [RFC7517] 形式で含めなければならない (MUST)。`kty` は `"EC"`、`crv` は `"secp256k1"` [RFC8812]
  でなければならない (MUST)。`cnf` に `jwk` 以外のメンバを含めてはならない (MUST NOT)。
* `exp` — 含めてはならない (MUST NOT)。証明書は無期限である (§3.1)。
* `nbf` — 含めてはならない (MUST NOT)。

検証者が認識しないクレームは無視しなければならない (MUST)。

### 4.3 署名

署名対象は `base64url(header) + "." + base64url(payload)` の ASCII 文字列であり、
CIP-2 §4.3 と同一の方式 (Keccak256 ハッシュに対する secp256k1 ECDSA、(r, s, v) 65 バイトを base64url)
でエンティティ本体の秘密鍵により署名する。

## 5. 発行

### 5.1 CIP-17 による要求と応答

証明書の発行は CIP-17 の Authorization Request の一部として要求する。
本 CIP は CIP-17 の Request Object に次を追加する。

* `scope` に `concrnt_service_key` を含める。
* `service_key` (REQUIRED when `concrnt_service_key` in scope) — サービス鍵の公開鍵を含むオブジェクト。

```json
{
  "client_id": "https://example.com/auth/concrnt/callback",
  "response_type": "id_token",
  "response_mode": "direct_post",
  "scope": "openid profile concrnt_service_key",
  "nonce": "n-0S6_WzA2Mj",
  "exp": 1763901596,
  "service_key": {
    "jwk": {
      "kty": "EC",
      "crv": "secp256k1",
      "x": "...",
      "y": "..."
    }
  }
}
```

Authenticator は、`scope` に `concrnt_service_key` が含まれるが `service_key.jwk` が存在しないか
不正である場合、要求全体を拒否しなければならない (MUST)。

発行に同意した場合、Authenticator は証明書を CIP-17 §8 の応答パラメータ `service_key_certificate`
として ID Token と共に返す。

```text
id_token=eyJ...&service_key_certificate=eyJ...&state=af0ifjsldkj
```

ユーザーが ID Token の発行には同意したが証明書の発行には同意しなかった場合、
Authenticator は `service_key_certificate` を省略して応答してよい (MAY)。
サービスはその場合を扱えなければならない (MUST)。

証明書の `aud` は Request Object の `client_id`、`cnf.jwk` は `service_key.jwk` と一致しなければ
ならない (MUST)。Authenticator は、受け取った JWK が secp256k1 曲線上の有効な点であることを
確認してから証明書に含めるべきである (SHOULD)。

### 5.2 サービス鍵の単位

サービスは、エンティティごとに別のサービス鍵を生成しなければならない (MUST)。
1 つのサービス鍵に対して複数の CCID から証明書を取得してはならない (MUST NOT)。
これにより、1 つのサービス鍵の漏洩の影響が 1 エンティティに限定され、
また証明書の `cnf.jwk` と `sub` の対応が一意になる。

同じエンティティに対してサービスが複数回証明書を要求すること (鍵のローテーション等) は妨げない。
各証明書は独立しており、新しい証明書の発行は古い証明書を無効化しない (§7.1)。

### 5.3 同意画面の要件

Authenticator は、CIP-17 §6 の要件に加えて、証明書の発行を ID Token の発行とは**別の項目として**
明示し、次を提示しなければならない (MUST)。

* このサービスが、ユーザーの名義でそのサービス内の成果物に署名できる鍵を受け取ること。
* 証明書に有効期限がなく、Concrnt 側から取り消す手段がないこと。
* 鍵の効力がそのサービス (`aud` の host) の内部に限られ、Concrnt サーバーや他のサービスに対する
  名乗りには使えないこと。
* サービス鍵の識別子。表示用には `cnf.jwk` の公開鍵から CIP-13 §3 と同一の手順で導出した
  CKID (HRP `cck`) を用いることを推奨する (RECOMMENDED)。

### 5.4 Concrnt サーバーでの扱い

Service Key Certificate は Concrnt Document ではなく、Concrnt サーバーにコミットされない。
Concrnt サーバーは次を保証する。

* 証明書を CIP-13 の Enact Document として解釈してはならない (MUST NOT)。形式が異なるため、
  CIP-13 §6 の検証は自然に失敗する。
* サービス鍵で署名された Concrnt Signed Document (CIP-1) および auth token (CIP-2) は、
  その鍵に対応する Enact Document がリポジトリに存在しない限り検証に失敗する。
  本 CIP はそのような Enact Document の作成を規定せず、サービスがエンティティに
  サービス鍵の Enact Document のコミットを求めることを **想定しない**。Authenticator は、
  CIP-17 の要求を契機として CIP-13 の Enact Document をコミットしてはならない (MUST NOT)。

## 6. サービス鍵の利用

本 CIP は、サービス鍵で何に署名するか、署名をどのような形式で表現するか、証明書をどのように
同梱・参照するかを定義しない。これらはサービスが定義し、サービスの検証者に周知する。

ただし、第三者が検証可能であるためには、サービス鍵による署名の検証者が次を入手できなければならない。

* Service Key Certificate そのもの。
* サービス鍵で署名された対象と、その署名。

参考として、サービス鍵による署名を JWS として表現する場合、`cnf.jwk` の公開鍵に対して標準の
JWS アルゴリズム (例: `ES256K` [RFC8812]) を用いることができる。サービス鍵の署名方式は
`CONCRNT` である必要はない。証明書の検証 (§7) と、サービス鍵による署名の検証は独立した手順である。

## 7. 検証

Service Key Certificate の検証者 (サービス自身、またはサービス鍵の署名を受け取った第三者) は、
次の手順で証明書を検証しなければならない (MUST)。

1. JWT として構文解析する。ヘッダーの `alg` が `"CONCRNT"`、`typ` が `"concrnt-service-key+jwt"` であり、
   `kid` が存在しないことを確認する。
2. `sub` が `iss` と等しく、CIP-0 §5.2 の CCID (小文字の Bech32、HRP `con`) であることを確認する。
3. `aud` が、検証者の文脈において当該サービスを指すことを確認する。
   サービス自身が検証する場合は自身の `client_id` との一致。第三者が検証する場合は、
   その署名が「`aud` のサービスの中での名乗り」であることを理解した上で受理する。
   第三者は、`aud` が自身の期待するサービスでない証明書を受理してはならない (MUST NOT)。
4. `cnf.jwk` が §4.2 の要件を満たし、secp256k1 曲線上の有効な点であることを確認する。
5. `exp` または `nbf` が存在する場合、検証は失敗する (MUST)。
6. `base64url(header) + "." + base64url(payload)` を Keccak256 でハッシュ化し、署名から ECRECOVER で
   公開鍵を復元、CIP-0 §5.2 の手順でアドレスを導出し、`sub` の CCID と一致することを確認する。

検証に成功した場合、検証者は「`cnf.jwk` の鍵による署名は、サービス `aud` の中で `sub` の
エンティティの名義で行われたもの」として扱う。

証明書の検証は Concrnt サーバーへの問い合わせを必要とせず、完全にオフラインで行える。

### 7.1 有効期限と失効を持たないことの帰結

証明書は発行時刻 `iat` 以降、無期限に有効である。サービス鍵の秘密鍵が漏洩した場合、
その鍵による署名を本仕様の範囲で無効化する手段はない。本仕様はこれを、
効力が `aud` のサービス内部に閉じていることを理由に許容する。

サービス鍵自身による自己無効化の宣言 (サービス鍵で署名した「この鍵は時刻 T 以降無効」という文書)
の形式は、将来の CIP で定義される可能性がある。本仕様の検証者はそのような宣言を解釈する義務を
負わない。

## 8. Security Considerations

* **サービス鍵の保護はサービスの責任である。** 秘密鍵の漏洩は、そのサービス内でのエンティティへの
  永続的ななりすましを許す。サービスは秘密鍵を HSM / KMS 等で保護するべきであり (SHOULD)、
  エンティティごとの鍵分離 (§5.2) を省略してはならない (MUST NOT)。
* **`aud` の束縛を省略してはならない。** 検証者が `aud` を確認しない場合、あるサービスに発行された
  証明書を別のサービスでの名乗りに流用できてしまう (§7 手順 3)。
* **証明書は Concrnt への権限を一切含まない。** サービス鍵は Concrnt Document の署名にも
  Concrnt サーバーの認証にも使えない (§5.4)。悪意あるサービスが得られるのは、
  自身の `aud` の中でエンティティの名義を使う能力のみである。
* **`typ` の確認を省略してはならない。** 同じ `alg` を用いる CIP-2 auth token や CIP-17 ID Token を
  証明書として誤って受理すると、`cnf` を持たないため実害は限定的だが、検証ロジックの混同を招く。
* **同意画面の表現が安全性を決める。** 証明書は無期限かつ取り消し不能であるため、
  ユーザーが CIP-17 のログイン同意と区別して理解できることが前提である (§5.3)。
* **cross-device の同意フィッシング (CIP-17 §10.1) は本 CIP にも適用される。** 攻撃者は自身の
  サービス鍵に対する証明書を被害者に発行させうる。得られる効力は `aud` のサービス内に限られるが、
  そのサービスが被害者の名義で成果物を生成できるようになる。CIP-17 の緩和策はすべて本 CIP の
  発行時にも適用しなければならない (MUST)。

## 9. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 7517 – JSON Web Key (JWK)
* RFC 7519 – JSON Web Token (JWT)
* RFC 7800 – Proof-of-Possession Key Semantics for JSON Web Tokens (`cnf` クレーム)
* RFC 8812 – CBOR Object Signing and Encryption (COSE) and JSON Object Signing and Encryption (JOSE)
  Registrations for Web Authentication (WebAuthn) Algorithms (`secp256k1` 曲線、`ES256K`)
* CIP-0 – Concrnt Core (CCID)
* CIP-1 – Concrnt Document System (§7.2 署名方式)
* CIP-2 – Auth (`CONCRNT` アルゴリズム)
* CIP-13 – Subkey (本 CIP との対比)
* CIP-17 – External Service Authentication (発行の配送路)
