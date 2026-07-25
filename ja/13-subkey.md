# CIP-13: Subkey

## 0. Abstract

本ドキュメントでは、CIP-1 で定義された Concrnt Signed Document において、署名者の秘密鍵を
直接使用することなく、信頼された別の鍵 (サブキー) を用いて署名を行う方法を定義する。

## 1. Status of This Memo

このドキュメントはサブキーの仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0, CIP-1, CIP-3 を前提とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

### CKID (Concrnt Key ID)

サブキーの鍵ペアから CIP-0 §5.2 と同一の手順 (公開鍵からアカウントアドレスを導出し Bech32
エンコード) で導出される、サブキーの識別子。

### Enact Document

あるサブキーの利用をエンティティが承認したことを示す Concrnt Document
(schema `https://schema.concrnt.net/subkey.json`)。

### Revoked Subkey Document

あるサブキーの失効を表明する Concrnt Document (schema `https://schema.concrnt.net/revoked-subkey.json`)。
失効対象の Enact Document を `value` にそのまま埋め込む。

## 3. Subkey の作成

サブキーは、CIP-0 で定義されたエンティティの作成と同様の手順で鍵ペアを生成する。
これを Bech32 エンコードしたものを CKID とする。

CKID の HRP は `"cck"` とするべきである (SHOULD)。
これは Entity (`con`) やサーバー (`ccs`) と鍵の種別を接頭辞で区別できるようにするための規約であり、
検証者は HRP を検証の条件としてはならない (MUST NOT)。照合は HRP を除くアドレス部に対して行われ、
`cck` 以外の HRP を持つ CKID であっても検証は成立する (§6 手順 6)。

サブキーを利用するエンティティは、以下の Enact Document を作成し、自身のリポジトリに
広告しなければならない (MUST)。

```json
{
  "kind": "record",
  "author": "con1<bech32-encoded-address>",
  "schema": "https://schema.concrnt.net/subkey.json",
  "key": "cckv://con1<bech32-encoded-address>/subkeys/<any-name>",
  "value": {
    "ckid": "cck1<bech32-encoded-subkey-address>"
  },
  "createdAt": "2025-11-23T12:34:56Z"
}
```

* `kind`
  通常の Document と同様 `"record"` を指定する。
* `schema`
  常に `"https://schema.concrnt.net/subkey.json"` を指定する (MUST)。
* `key`
  Enact Document を配置する cckv キー。owner 部は `author` 自身の CCID でなければならない (MUST)。
  author の名前空間内の任意のキーでよいが、慣用的に `subkeys/` 配下への配置を推奨する (RECOMMENDED)。
* `value.ckid`
  承認するサブキーの CKID。
* `createdAt`
  承認時刻 (UTC, RFC3339 形式)。

Enact Document は、CIP-1 で定義された `concrnt-ecrecover-direct` proof、すなわちエンティティ本体の
秘密鍵によって署名されなければならない (MUST)。

## 4. Subkey の失効

特定のサブキーが侵害された場合など、サブキーの利用を終了する必要がある場合、
エンティティは **Enact Document と同じキーを Revoked Subkey Document で上書き**することで
失効を表明する。

Revoked Subkey Document は、`value` に失効対象の Enact Document の Signed Document
(`{document, proof}`) をそのまま埋め込んだ Document である。

```json
{
  "kind": "record",
  "author": "con1<bech32-encoded-address>",
  "schema": "https://schema.concrnt.net/revoked-subkey.json",
  "key": "cckv://con1<bech32-encoded-address>/subkeys/<any-name>",
  "value": {
    "document": "<Enact DocumentのJSON文字列>",
    "proof": {
      "type": "concrnt-ecrecover-direct",
      "signature": "<hex-encoded-signature>"
    }
  },
  "createdAt": "2025-12-01T00:00:00Z"
}
```

* `schema`
  常に `"https://schema.concrnt.net/revoked-subkey.json"` を指定する (MUST)。
* `key`
  失効対象の Enact Document と**同じキー**を指定する (MUST)。
  CIP-3 §3.4 の同一キー上書き規則により、以後このキーの解決結果は Revoked Subkey Document となる。
* `value`
  失効対象の Enact Document の Signed Document そのもの (MUST)。
* `createdAt`
  失効時刻。

Revoked Subkey Document は、Enact Document と同様にエンティティ本体の秘密鍵
(`concrnt-ecrecover-direct` proof) で署名されなければならない (MUST)。

**失効は一方向の操作である。** サーバーは、解決結果が Revoked Subkey Document であるキーに対する、
schema が `subkey.json` または `revoked-subkey.json` である Document の上書きコミットを
拒否しなければならない (MUST)。これを許すと、より新しい `createdAt` の再上書きによって
有効期間の終端を前進させ、失効を実質的に取り消すことができてしまう。
サブキーを再度利用する場合は、新しいキーと新しい鍵ペアを使用しなければならない (MUST)。

### 4.1 有効期間のセマンティクス

Revoked Subkey Document は、当該サブキーの**有効期間**を表明する。

* 有効期間の開始: `value` に埋め込まれた Enact Document の `createdAt` (境界値を含む)。
* 有効期間の終了: Revoked Subkey Document 自身の `createdAt` (境界値を含む)。

すなわち、キーの解決結果が Enact Document (schema `subkey.json`) であればサブキーは現在有効であり、
Revoked Subkey Document (schema `revoked-subkey.json`) であれば「埋め込まれた Enact の `createdAt`
から Revoked 本体の `createdAt` までの期間に限り有効だった」ことを意味する。

このモデルにより、失効後もサブキーの**失効前の正当な署名は引き続き検証可能**である (§6)。
一方、失効以後にそのサブキーで作成された署名は検証に失敗する。

なお、上書きされた旧 Enact Document の CDID はコミットログ (CIP-3 §3.4) に残り続けるため、
捕捉された旧 Enact Document を再コミットして失効を巻き戻すこと (rollback) はできない。

### 4.2 削除による無効化との関係

Enact Document のキーを削除 (CIP-4) した場合もサブキーは利用不能になるが、
その場合は有効期間の情報が失われるため、**失効前の過去署名も検証不能になる**。
過去署名の検証可能性を保つため、失効は Revoked Subkey Document による上書きで
行うべきである (SHOULD)。削除による無効化は推奨されない (NOT RECOMMENDED)。

## 5. Subkey による Concrnt Document の署名

CIP-1 で定義された Concrnt Signed Document において、proof タイプ `"concrnt-ecrecover-subkey"` を
定義する。

```json
{
  "document": "<JSON string of the document>",
  "proof": {
    "type": "concrnt-ecrecover-subkey",
    "key": "cckv://con1<bech32-encoded-address>/subkeys/<any-name>",
    "signature": "<hex-encoded-signature>"
  }
}
```

* `type`
  常に `"concrnt-ecrecover-subkey"`。
* `key`
  署名に使用したサブキーの Enact Document を指す cckv URI。必須 (MUST)。
* `signature`
  Document の文字列に対する署名。CIP-1 と同様、Keccak256 ハッシュに対する secp256k1 ECDSA 署名
  (r, s, v) の16進エンコード表現。必須 (MUST)。

署名は、document の JSON 文字列を Keccak256 でハッシュ化し、**サブキーの秘密鍵**で ECDSA 署名を
行うことで生成する。

## 6. 検証

subkey proof を持つ Signed Document の検証は、以下の手順で行わなければならない (MUST)。

1. proof の `key` に指定された cckv URI を解決し、Signed Document を取得する。
   解決先のサーバーは、`key` の owner 部の CCID から CIP-0 §8 の名前解決 (Entity Document の
   `domain`) によって決定しなければならない (MUST)。`key` に付与されたリゾルバヒント
   (`@<FQDN>`, CIP-0 §7.2) を取得先の決定に用いてはならない (MUST NOT) —
   提出者が指定したホストから失効前の Enact Document を配り続けることで、失効を無効化できて
   しまうためである。同様に、検証対象の Signed Document に同梱されたインライン参照
   (`references`, CIP-1) を**使用してはならない (MUST NOT)**。取得に失敗した場合、検証は失敗する。
2. 取得した Signed Document を検証する。
   Enact Document / Revoked Subkey Document の proof は `concrnt-ecrecover-direct` で
   なければならない (MUST)。それ以外の proof タイプを持つ場合、検証は失敗する
   (サブキーで署名された Enact / Revoked Document は認められない)。
3. 取得した Document の `schema` に応じて、Enact Document と有効期間を決定する。
   * `"https://schema.concrnt.net/subkey.json"` (Enact Document):
     サブキーは現在有効である。有効期間は Enact Document の `createdAt` 以降。
   * `"https://schema.concrnt.net/revoked-subkey.json"` (Revoked Subkey Document):
     `value` に埋め込まれた Signed Document を Enact Document として取り出し、
     その署名を検証し、`schema` が `subkey.json` であることを確認する (MUST)。
     埋め込まれた Enact Document の proof も同様に `concrnt-ecrecover-direct` で
     なければならない (MUST)。
     有効期間は、埋め込まれた Enact Document の `createdAt` から
     Revoked Subkey Document の `createdAt` まで。
   * それ以外の schema: 検証は失敗する (MUST)。
4. Enact Document (および Revoked Subkey Document) の `author`、および proof の `key` の owner 部が、
   いずれも検証対象 Document の `author` と一致することを確認する (MUST)。
   後者の確認がない場合、第三者の名前空間に置かれた Enact Document が受理され、
   失効の可否がその名前空間の所有者に委ねられてしまう。
5. 検証対象 Document の `createdAt` が、手順 3 で決定した有効期間内であることを確認する (MUST)。
   有効期間の境界は**両端とも境界値を含む (inclusive)**。すなわち、Enact Document の `createdAt` と
   同時刻の署名、および Revoked Subkey Document の `createdAt` と同時刻の署名は有効である。
6. 検証対象 Document の文字列を Keccak256 でハッシュ化し、`signature` から ECRECOVER で公開鍵を
   復元、CIP-0 §5.2 の手順でアドレス (20 バイト) を導出する。`value.ckid` を Bech32 デコードして
   得たデータ部と、このアドレスを**バイト列として**比較し、一致することを確認する (MUST)。
   HRP は比較の対象としない (§3)。`value.ckid` が Bech32 としてデコードできない場合、検証は失敗する。

## 7. Subkey による API 認証

サブキーは、Document への署名だけでなく、auth token (CIP-2) の署名にも使用できる。
auth token のヘッダー `kid` に Enact Document を指す cckv URI を指定することで、
サーバーはサブキーによる署名としてトークンを検証する。
キーの解決結果が Revoked Subkey Document であるサブキーは、トークンの署名に使用できない
(認証は「現在有効」なサブキーのみが行える)。
トークンの形式・検証手順の詳細は CIP-2 §5.2 を参照。

## 8. Security Considerations

* サブキーの秘密鍵の漏洩は、失効 (Revoked Subkey Document による上書き) を行うまでの間、
  エンティティ名義での Document 発行を許すことになる。サブキーの用途はエンドデバイスごとの発行など
  限定的に留め、漏洩を検知した場合は速やかに失効すべきである (SHOULD)。
* エンティティ本体の秘密鍵はサブキーの失効に必要となるため、サブキーとは独立して安全に
  保管するべきである (SHOULD)。
* 漏洩したサブキーによる失効前の日時へのバックデート署名は、CIP-3 §3.4 の backdate window
  によって受理範囲が制限される。失効が疑われる場合、有効期間の終了は window 分だけ早めに
  見積もるのが安全である。
* 検証者は Enact Document / Revoked Subkey Document を必ず権威サーバーから取得すること (§6 手順 1)。
  キャッシュを行う場合、失効の反映が遅延するリスクを考慮し、キャッシュ期間を適切に
  制限すべきである (SHOULD)。

## 9. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-0 – Concrnt Core (CCID, Bech32 アドレス導出, 名前解決)
* CIP-2 – Auth (サブキーによる API 認証)
* CIP-3 – Commit (同一キー上書き, backdate window)
