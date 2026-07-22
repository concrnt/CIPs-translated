# CIP-15 Abuse Report

## 0. Abstract

本ドキュメントでは、Concrnt上のリソースに対する通報 (abuse report) をサーバー運用者へ届けるためのエンドポイントを定義する。

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

## 3. Abuse エンドポイント

サーバーは、リソースに対する通報を受け付けるエンドポイントを提供してもよい (MAY)。
提供する場合、CIP-0で定義されるサービスディスカバリにおいて、`net.concrnt.core.abuse` エンドポイント名で広告しなければならない (MUST)。

```json
{
  "endpoints": {
    "net.concrnt.core.abuse": "/abuse"
  }
}
```

このエンドポイントは HTTP POST リクエストを受け付ける。

### 3.1 リクエスト形式

リクエストボディは以下のJSONオブジェクトである。

```json
{
  "target": "<通報対象リソースのURI>",
  "body": "<通報内容の説明>"
}
```

* `target` (string, required)
  通報対象のリソースを示すURI。通常はCCURI (CIP-0) を指定する。

* `body` (string, required)
  通報内容の説明。自由記述のテキスト。

### 3.2 認証とレスポンス

このエンドポイントは認証 (CIP-2) を必要とする (MUST)。未認証のリクエストに対しては HTTP 403 Forbidden を返す。

サーバーは、通報を受理した場合、通報者のEntity (CCID) とともに通報を記録し、HTTP 200 を返す。

### 3.3 通報への対応

通報への対応 (コンテンツの削除・モデレーション等) はサーバー運用者の裁量であり、本仕様では定義しない。
通報の転送 (対象リソースを管理する他サーバーへの中継) も本仕様では定義しない。
通報の保持期間・プライバシー・重複排除・本文サイズ上限・管理者向けの取得手段といった
運用上の要件も、意図的に本仕様のスコープ外とする (運用者・実装の裁量)。

## 4. Security Considerations

* 通報内容には通報者の主観的な主張が含まれる。サーバー運用者は通報を、対象リソースに対する検証済みの事実としてではなく、調査の契機として扱うべきである (SHOULD)。
* 通報の悪用 (スパム通報・嫌がらせ通報) を緩和するため、サーバーはレート制限を行ってもよい (MAY)。

## 5. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
