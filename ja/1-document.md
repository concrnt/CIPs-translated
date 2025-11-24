# CIP-1: Concrnt Document

## 0. Abstract

本ドキュメントでは、Concrnt エコシステムにおける **Concrnt Document** の形式と意味を定義します。Concrnt Document は、CIP-0 で定義された CCID と CCURI の上に載る署名付きデータコンテナであり、投稿やプロフィールなどのアプリケーションデータの表現、タイムラインやコレクションへの所属の宣言、参照関係の記録に利用されます。

## 1. Status of This Memo

本ドキュメントは Concrnt Document フォーマットのインターネット・ドラフトです。Concrnt プロジェクトが公開するバージョン付き仕様であり、実装者およびプロトコル設計者を対象とします。ドラフト期間中に後方互換性のない変更が行われる可能性があるため、実装者は CIP 番号とバージョンを確認し、更新に追従しなければなりません（MUST）。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。その他の用語は CIP-0 の定義を継承します。CDID は Concrnt Document を一意に識別するための識別子であり、本文でその生成方法を示します。

## 3. 位置づけとスコープ

CIP-1 は Concrnt Document の JSON 構造と各フィールドの意味、CDID の生成方法、署名形式を定義します。ネットワーク伝送や保管の方法、上位アプリケーション固有のフィールドについては扱いません。

## 4. Concrnt Document

Concrnt Document は不変の JSON オブジェクトとして表現されます。基本的な表現には `key`, `contentType`, `schema`, `value`, `author`, `owner`, `createdAt` が含まれます。`value` の構造は `schema` により決定され、追加フィールドは上位の CIP が拡張することがあります。`key` を持つ Document は、同一 `owner` における論理的なヘッドを指し示すために用いられます。

`key` が付与された Document は、同じ `owner` に対する複数のバージョンのうち、最新の `createdAt` を持つものが「現在のヘッド」として解釈されます。`key` がない場合、CDID で直接参照する静的なレコードとなります。

## 5. フィールド定義

`key` はオプションの論理名であり、UTF-8 で 1024 バイト以内とします。`key` が存在する場合、その Document は `cc://<owner>/<key>` で参照されます。同一の `owner` と `key` の組を持つ Document が複数存在する場合、`createdAt` が最新のものを優先します（MUST）。`key` は人間可読なラベルとして扱われることが多く、上位仕様が命名規則を与えることがあります。

`contentType` は Document のメディアタイプです。省略された場合は `"application/json"` とみなしてよいとします（SHOULD）。バイナリやマルチパート形式を格納する場合は適切な MIME タイプを指定し、`value` にエンコード済みデータを格納します。

`schema` は `value` の構造を定める参照 URI であり、解決可能なエンドポイントで JSON Schema を提供しなければなりません（MUST）。クライアントは `schema` に基づいて `value` を検証してもかまいません（MAY）。

`value` は任意の JSON 値であり、具体的な構造は `schema` に従います。オブジェクト・配列・文字列・数値・真偽値・null のいずれでも構いません。

`author` は作成者の CCID であり、CIP-0 の形式に従わなければなりません（MUST）。`owner` は論理的な所有者の CCID であり、省略された場合は `author` と同一とみなします（SHOULD）。`owner` を別に指定する場合でも、その検証方法や権限制御は上位仕様（CIP-3 以降）に委ねられます。

`createdAt` は RFC3339 の UTC 時刻文字列です。サーバは極端に過去または未来の値を拒否してもよいとします（MAY）。時計ずれがある環境では、許容範囲を設けた上で検証してください。

## 6. CDID の生成

CDID は 16 バイトの値で、先頭 10 バイトに Document の JSON シリアライズを Keccak256 した値を配置し、後続 6 バイトに `createdAt` の UNIX 時刻（ミリ秒）をビッグエンディアンで格納します。この 16 バイト列を "0123456789abcdefghjkmnpqrstvwxyz" のアルファベットで Base32 エンコードした 26 文字の文字列が CDID となります。

シリアライズは以下の手順で行います。1) JSON を安定したキー順と最小限の空白で生成する。2) 得られたバイト列を Keccak256 にかけ先頭 10 バイトを取り出す。3) `createdAt` を UNIX 時刻（ミリ秒、ビッグエンディアン 6 バイト）に変換し、後ろに連結する。4) 合計 16 バイトを上記アルファベットで Base32 エンコードする。クライアントとサーバは同一の手順を実装しなければなりません（MUST）。

## 7. Concrnt Signed Document

Document の発行を証明するために署名付きオブジェクトを用います。構造は次のとおりです。

```json
{
  "document": "<Document JSON string>",
  "proof": {
    "type": "concrnt-ecrecover-direct",
    "signature": "<hex-encoded-signature>"
  }
}
```

署名対象は `document` に含まれる JSON 文字列そのものであり、Keccak256 によるハッシュを secp256k1 ECDSA (v, r, s) で署名します。検証時は ECRECOVER により復元した公開鍵が `author` の CCID に対応することを確認しなければなりません（MUST）。他の署名方式を定義する場合は `type` を拡張し、対応する検証手順を明示する必要があります。

署名が欠落している、または `author` と一致しない公開鍵で署名されている場合、受信者はその Document を無効として扱います。上位仕様が追加の `proof` フィールド（例: サブキーやマルチシグ）を定義することがありますが、その場合も署名対象は `document` の安定シリアライズ文字列であることを保ちます。

## 8. セキュリティに関する考慮事項

署名の検証は必須であり、署名が欠落または無効な Document を受け入れてはなりません（MUST NOT）。`createdAt` の巻き戻しや再送による再生攻撃を検出するため、より新しい Document を優先する運用を推奨します。時計ずれが大きい環境では `createdAt` の検証に注意を払ってください。秘密鍵はクライアント側で安全に保持し、他者に共有してはなりません（MUST NOT）。

## 9. 参考文献 (References)

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, March 1997.  
[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, May 2017.  
[RFC3339] Klyne, G. and C. Newman, “Date and Time on the Internet: Timestamps”, July 2002.  
[RFC3986] Berners-Lee, T., et al., “Uniform Resource Identifier (URI): Generic Syntax”, January 2005.  
Keccak Team, “Keccak SHA-3 Submission”, 2012.
