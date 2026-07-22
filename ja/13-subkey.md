# CIP-13 Subkey

## 0. Abstract
本ドキュメントでは、CIP-1で定義されたConcrnt Signed Documentにおいて、署名者の秘密鍵を直接使用することなく、信頼された別の鍵（サブキー）を用いて署名を行う方法を定義する。


## 1. Status of This Memo

このドキュメントは Concrnt Document フォーマットの仕様を定義する。

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、
実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

### CKID (Concrnt Key ID)

サブキーの鍵ペアから CIP-0 §5.2 と同一の手順（公開鍵からアカウントアドレスを導出し Bech32 エンコード）で導出される、サブキーの識別子。

### Enact Document

あるサブキーの利用をエンティティが承認したことを示す Concrnt Document (schema `https://schema.concrnt.net/subkey.json`)。

### Revoked Subkey Document

あるサブキーの失効を表明する Concrnt Document (schema `https://schema.concrnt.net/revoked-subkey.json`)。
失効対象の Enact Document を `value` にそのまま埋め込む。

## 3. Subkeyの作成

subkeyはCIP-0で定義されたエンティティの作成と同様の手順でキーペアを生成する。
これをbech32エンコードしたものをCKIDとする。

CKIDのHRPは `"cck"` とするべきである (SHOULD)。
これはEntity (`con`) やサーバ (`ccs`) と鍵の種別を接頭辞で区別できるようにするための規約であり、
検証者はHRPを検証の条件としてはならない (照合はHRPを除くアドレス部に対して行われ、
`cck` 以外のHRPを持つCKIDであっても検証は成立する)。

subkeyを利用するエンティティは、以下のEnact Documentを作成し、自身のリポジトリに広告しなければならない(MUST)。

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
  通常のDocumentと同様 `"record"` を指定する。
* `schema`
  常に `"https://schema.concrnt.net/subkey.json"` を指定する (MUST)。
* `key`
  Enact Documentを配置するcckvキー。author自身の名前空間内の任意のキーでよい。
  慣用的に `subkeys/` 配下への配置を推奨する (RECOMMENDED)。
* `value.ckid`
  承認するサブキーのCKID。
* `createdAt`
  承認時刻（UTC, RFC3339形式）。

Enact Documentは、CIP-1で定義された `concrnt-ecrecover-direct` proof、すなわちエンティティ本体の秘密鍵によって署名されなければならない (MUST)。

## 4. Subkeyの失効

特定のsubkeyが侵害された場合など、subkeyの利用を終了する必要がある場合、
エンティティは**Enact Documentと同じキーをRevoked Subkey Documentで上書き**することで失効を表明する。

Revoked Subkey Documentは、`value` に失効対象のEnact DocumentのSigned Document
(`{document, proof}`) をそのまま埋め込んだDocumentである。

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
  失効対象のEnact Documentと**同じキー**を指定する (MUST)。
  CIP-1の同一キー上書き規則により、以後このキーの解決結果はRevoked Subkey Documentとなる。
* `value`
  失効対象のEnact DocumentのSigned Documentそのもの (MUST)。
* `createdAt`
  失効時刻。

Revoked Subkey Documentは、Enact Documentと同様にエンティティ本体の秘密鍵
(`concrnt-ecrecover-direct` proof) で署名されなければならない (MUST)。

### 4.1 有効期間のセマンティクス

Revoked Subkey Documentは、当該サブキーの**有効期間**を表明する。

* 有効期間の開始: `value` に埋め込まれたEnact Documentの `createdAt` (境界値を含む)。
* 有効期間の終了: Revoked Subkey Document自身の `createdAt` (境界値を含む)。

すなわち、キーの解決結果がEnact Document (schema `subkey.json`) であればサブキーは現在有効であり、
Revoked Subkey Document (schema `revoked-subkey.json`) であれば「埋め込まれたEnactの `createdAt` から
Revoked本体の `createdAt` までの期間に限り有効だった」ことを意味する。

このモデルにより、失効後もサブキーの**失効前の正当な署名は引き続き検証可能**である (§6)。
一方、失効以後にそのサブキーで作成された署名は検証に失敗する。

なお、上書きされた旧Enact Documentのdocument IDはコミットログ (CIP-3 §3.4) に
残り続けるため、捕捉された旧Enact Documentを再コミットして
失効を巻き戻すこと (rollback) はできない。

### 4.2 削除による無効化との関係

Enact Documentのキーを削除 (CIP-4) した場合もサブキーは利用不能になるが、
その場合は有効期間の情報が失われるため、**失効前の過去署名も検証不能になる**。
過去署名の検証可能性を保つため、失効はRevoked Subkey Documentによる上書きで
行うべきである (SHOULD)。削除による無効化は推奨されない (NOT RECOMMENDED)。

## 5. SubkeyによるConcrnt Documentの署名

CIP-1で定義されたConcrnt Signed Documentにおいて、proofタイプ `"concrnt-ecrecover-subkey"` を定義する。

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
  署名に使用したサブキーのEnact Documentを指すcckv URI。必須 (MUST)。
* `signature`
  Documentの文字列に対する署名。CIP-1と同様、Keccak256ハッシュに対するsecp256k1 ECDSA署名 (r, s, v) の16進エンコード表現。必須 (MUST)。

署名は、documentのJSON文字列をKeccak256でハッシュ化し、**サブキーの秘密鍵**でECDSA署名を行うことで生成する。

## 6. 検証

subkey proofを持つSigned Documentの検証は、以下の手順で行わなければならない (MUST)。

1. proof の `key` に指定されたcckv URIを解決し、Signed Documentを取得する。
   このとき、検証対象のSigned Documentに同梱されたインライン参照（`references` フィールド, CIP-1）を**使用してはならない (MUST NOT)**。
   必ずURIの権威サーバー（owner所属サーバー）から取得すること。
   インライン参照を信頼すると、失効済みのEnact Documentを提出者が同梱することで、失効後も署名を永久に再生できてしまうためである。
   取得に失敗した場合、検証は失敗する。
2. 取得したSigned Documentを検証する。
   Enact Document / Revoked Subkey Documentのproofは `concrnt-ecrecover-direct` で
   なければならない (MUST)。それ以外のproofタイプを持つ場合、検証は失敗する
   (サブキーで署名されたEnact/Revoked Documentは認められない)。
3. 取得したDocumentの `schema` に応じて、Enact Documentと有効期間を決定する。
   * `"https://schema.concrnt.net/subkey.json"` (Enact Document):
     サブキーは現在有効である。有効期間は Enact Documentの `createdAt` 以降。
   * `"https://schema.concrnt.net/revoked-subkey.json"` (Revoked Subkey Document):
     `value` に埋め込まれたSigned DocumentをEnact Documentとして取り出し、
     その署名を検証し、`schema` が `subkey.json` であることを確認する (MUST)。
     埋め込まれたEnact Documentのproofも同様に `concrnt-ecrecover-direct` で
     なければならない (MUST)。
     有効期間は、埋め込まれたEnact Documentの `createdAt` から
     Revoked Subkey Documentの `createdAt` まで。
   * それ以外のschema: 検証は失敗する (MUST)。
4. Enact Document (および Revoked Subkey Document) の `author` フィールドが、
   検証対象Documentの `author` フィールドと一致することを確認する (MUST)。
5. 検証対象Documentの `createdAt` が、手順3で決定した有効期間内であることを確認する (MUST)。
   有効期間の境界は**両端とも境界値を含む (inclusive)**。すなわち、Enact Documentの `createdAt` と
   同時刻の署名、およびRevoked Subkey Documentの `createdAt` と同時刻の署名は有効である。
6. 検証対象Documentの文字列をKeccak256でハッシュ化し、`signature` からECRECOVERで公開鍵を復元、
   導出したアドレスがEnact Documentの `value.ckid` と一致することを確認する (MUST)。

## 7. SubkeyによるAPI認証

サブキーは、Documentへの署名だけでなく、auth token (CIP-2) の署名にも使用できる。
auth token のヘッダー `kid` に Enact Document を指す cckv URI を指定することで、
サーバーはサブキーによる署名としてトークンを検証する。
キーの解決結果がRevoked Subkey Documentであるサブキーは、トークンの署名に使用できない
(認証は「現在有効」なサブキーのみが行える)。
トークンの形式・検証手順の詳細は CIP-2 §5.2 を参照。

## 8. Security Considerations

* サブキーの秘密鍵の漏洩は、失効（Revoked Subkey Documentによる上書き）を行うまでの間、エンティティ名義でのDocument発行を許すことになる。サブキーの用途はエンドデバイスごとの発行など限定的に留め、漏洩を検知した場合は速やかに失効すべきである (SHOULD)。
* エンティティ本体の秘密鍵はサブキーの失効に必要となるため、サブキーとは独立して安全に保管しなければならない。
* 漏洩したサブキーによる失効前の日時へのバックデート署名は、CIP-3 §3.4 のbackdate window (7日) によって受理範囲が制限される。失効が疑われる場合、有効期間の終了はwindow分だけ早めに見積もるのが安全である。
* 検証者はEnact Document / Revoked Subkey Documentを必ず権威サーバーから取得すること（§6手順1）。キャッシュを行う場合、失効の反映が遅延するリスクを考慮し、キャッシュ期間を適切に制限すべきである (SHOULD)。
