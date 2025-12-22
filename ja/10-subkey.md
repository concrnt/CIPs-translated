# CIP-10 Subkey

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

## 3. Subkeyの作成

subkeyはCIP-0で定義されたエンティティの作成と同様の手順でキーペアを生成する。
HRPを"cck"としてbech32エンコードを用いてこれをSubkeyのIDとする。

subkeyは、以下のconcrnt documentを作成し広告しなければならない(MUST)。

```json
{
  "author": "con1<bech32-encoded-address>",
  "schema": "https://schema.concrnt.net/subkey-enact.json",
  "key": "cck<bech32-encoded-subkey-address>",
  "value": {
    "ckid": "cck<bech32-encoded-subkey-address>"
  },
  "createdAt": "2025-11-23T12:34:56Z"
}
```

## 4. Subkeyの利用終了

特定のsubkeyが侵害された場合など、subkeyの利用を終了する必要がある場合、これを終了することができる。

以下のConcrnt documentを作成し広告しなければならない(MUST)。

```json
{
  "author": "con1<bech32-encoded-address>",
  "schema": "https://schema.concrnt.net/subkey-revoke.json",
  "key": "cck<bech32-encoded-subkey-address>",
  "value": {
    "ckid": "cck<bech32-encoded-subkey-address>",
    "enact": "<Concrnt Document used to enact the subkey>"
  },
  "createdAt": "2025-11-23T12:34:56Z"
}
```

## 5. SubkeyによるConcrnt Documentの署名

CIP-1で定義されたConcrnt Signed Documentにおいて、新しい署名タイプ"concrnt-ecrecover-subkey"を定義します。

```json
{
  "document": "<JSON string above>",
  "proof": {
    "type": "concrnt-ecrecover-subkey",
    "subkey": "cc://<CCID>/cck<bech32-encoded-subkey-address>",
    "signature": "<hex-encoded-signature>",
    "createdAt": "2025-11-23T12:34:56Z"
  }
}
```

検証時は、subkeyのConcrnt Documentを署名付きで取得し、正しく有効化されていることを確認しなければなりません(MUST)。
Documentにおけるschemaフィールドが"https://schema.concrnt.net/subkey-enact.json"であることを確認しなければなりません(MUST)。
schemaフィールドが"https://schema.concrnt.net/subkey-revoke.json"である場合、proof内のcreatedAtフィールドがrevoke DocumentのcreatedAtフィールドよりも前であり、enactフィールドに含まれるDocumentのcreatedAtフィールドよりも後であることを確認しなければなりません(MUST)。
また、documentのauthorフィールドがsubkeyのauthorと一致することを確認しなければなりません(MUST)。
