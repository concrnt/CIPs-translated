# CIP-3: Subkey

## 0. Abstract

本ドキュメントでは、CIP-1 で定義された Concrnt Signed Document を、主体の秘密鍵を直接用いずに信頼された別鍵（サブキー）で署名する方法を定義します。サブキーの作成、広告、失効、検証手順を示し、安全な鍵委任を可能にします。

## 1. Status of This Memo

本ドキュメントは Concrnt Subkey のインターネット・ドラフトです。Concrnt プロジェクトが公開するバージョン付き仕様であり、実装者およびプロトコル設計者を対象とします。ドラフト期間中に後方互換性のない変更が行われる可能性があるため、実装者は CIP 番号とバージョンを確認し、更新に追従しなければなりません（MUST）。

## 2. 用語 (Terminology)

大文字のキーワードは BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。CKID はサブキーの識別子であり、Bech32 の HRP は `"cck"` とします。その他の用語は CIP-0 および CIP-1 に従います。

## 3. 概要

サブキーはエンティティが所有する鍵とは別のキーペアであり、限定的な署名権限を委任するために用います。サブキーは Document として広告され、署名の検証時に有効性が確認されます。

## 4. サブキーの作成と広告

サブキーは CIP-0 と同様の方法で secp256k1 キーを生成し、Bech32 HRP `"cck"` でエンコードして CKID を得ます。所有者は次のような Document を作成し、署名付きで公開します。

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

この Document はサブキーが有効化されたことを示し、`author` と CKID の対応を明確にします。

## 5. サブキーの失効

侵害や運用終了に伴い、サブキーを失効させる場合は次の Document を発行します。

```json
{
  "author": "con1<bech32-encoded-address>",
  "schema": "https://schema.concrnt.net/subkey-revoke.json",
  "key": "cck<bech32-encoded-subkey-address>",
  "value": {
    "ckid": "cck<bech32-encoded-subkey-address>",
    "enact": "<enact document JSON string>"
  },
  "createdAt": "2025-11-23T12:34:56Z"
}
```

`enact` には有効化 Document を含め、失効の前後関係を検証できるようにします。

## 6. サブキーによる署名

サブキーで署名する場合、CIP-1 の Signed Document の `proof.type` を `"concrnt-ecrecover-subkey"` とし、次の構造を用います。

```json
{
  "document": "<Document JSON string>",
  "proof": {
    "type": "concrnt-ecrecover-subkey",
    "subkey": "cc://<CCID>/cck<bech32-encoded-subkey-address>",
    "signature": "<hex-encoded-signature>",
    "signedAt": "2025-11-23T12:34:56Z"
  }
}
```

署名対象は `document` の JSON 文字列であり、ハッシュと署名のアルゴリズムは CIP-1 の直接署名と同一です。

## 7. 検証手順

検証時は、`subkey` で示される CCURI を解決してサブキーの enact/revoke Document を取得します。`schema` が `subkey-enact` であること、`author` が対象 Document の `author` と一致することを確認しなければなりません（MUST）。`signedAt` が enact の `createdAt` 以降であり、revoke が存在する場合は revoke の `createdAt` より前であることを確認します（MUST）。条件を満たさない場合、署名は無効とみなします。

## 8. セキュリティに関する考慮事項

サブキーは本鍵とは別に保護し、侵害時には速やかに revoke Document を配布する必要があります。古い revoke を無視した再生攻撃を防ぐため、クライアントは最新の `createdAt` を持つ Document を優先するべきです（SHOULD）。サブキーの権限範囲は上位アプリケーションが定義するため、権限の過大委任に注意してください。

## 9. 参考文献 (References)

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, March 1997.  
[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, May 2017.
