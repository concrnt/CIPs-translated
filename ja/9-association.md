# CIP-9: Association

## 0. Abstract

この仕様では、CIP-1 で定義される Concrnt Document を拡張し、ある Concrnt Document から
他の Concrnt Document への関連付け (アソシエーション) を表現する方法を定義する。

## 1. Status of This Memo

このドキュメントは Association Document の仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0, CIP-1, CIP-3 を前提とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Association

CIP-1 で定義された Concrnt Document のうち、`kind` が `"association"` であるものを
Association Document と呼ぶ。Association Document は `associate` フィールドおよび
任意の `associationVariant` フィールドを持つ。

```json
{
  "kind": "association",              // CIP-1
  "schema": "https://...",            // CIP-1
  "value": { ... },                   // CIP-1

  "author": "con1...",                // CIP-1

  "associate": "cckv://<target-owner>/<document-key>", // CIP-9 (本仕様)
  "associationVariant": "example-variant",             // CIP-9 (本仕様, optional)

  "createdAt": "2025-11-23T12:34:56Z" // CIP-1
}
```

* `associate` フィールドには、関連付け先の Document を一意に識別する CCURI を指定する。
  scheme は `cckv` でなければならない (MUST)。owner 部にリゾルバヒント・エイリアスを含めては
  ならない (MUST NOT。CIP-3 §3.1)。一意性判定 (§3.1)・状態の識別・ccfs URI の導出には、
  正規化された裸の owner を用いる (MUST)。
* Association Document は `key` に値が入っていてはならない (MUST NOT)。
  つまり、Association Document は CDID (ccfs URI) でのみ参照可能なオブジェクトである。
* `associationVariant` は、Association のバリエーションを表現するための任意の文字列である。
  UTF-8 表現で 512 バイト以内でなければならない (MUST)。
  省略された場合、空文字列 `""` と等価とみなす (MUST)。
* Association Document 自身の `policy` フィールド (CIP-12) は、ポリシースタックには積まれない
  (評価コンテキストの `self` としてのみ参照される。CIP-12 §5.3)。

クライアントは、Association Document を、`associate` の owner を管理する権威サーバーの
Commit エンドポイント (CIP-3) へ送信しなければならない (MUST)。
コミットに成功した Association Document の ccfs URI は
`ccfs://<associateのowner>/concrnt/<CDID>` となる。

### 3.1 一意性

Association は、次の 5 要素の組み合わせで一意に識別される。

* `associate`
* `author`
* `schema`
* `associationVariant` (省略時は `""`)
* `value` (JSON シリアライズしたボディ)

サーバーは、この組み合わせが一致する Association を重複して受理してはならない (MUST NOT)。

一意性の判定は、`associate` の owner を管理する権威サーバーが行う。
`value` の比較のためのシリアライズ方法は実装定義であるが、同一サーバー内で安定でなければならない
(一意性は権威サーバー内で完結するため、サーバー間でシリアライズが一致する必要はない)。

重複する Association のコミットを受信した場合、サーバーはエラーとせず、
**何も保存しない no-op として成功応答を返す (MUST)**。
これは、配布 (CIP-7) の再試行等による同一 Association の再配送を冪等にするためである。
重複と判定されたコミットに対しては、Reference 配布やイベント配信 (§3.2) を行ってはならない (MUST NOT)。

### 3.2 Association の配布とイベント

Association Document のコミットに成功した場合、サーバーは関連付け先 Document (`associate`) および
その `distributes` (CIP-7) で指定される各配布先に対して、`associated` イベント (CIP-11) を配信する。
また、Association Document 自身も `distributes` フィールドを持つことができ、
その場合 CIP-7 の規定に従い Reference Document が配布される。

### 3.3 Association の削除

Association の削除は、CIP-4 で定義される削除 Document (`kind: "delete"`) の `value` に、
削除対象 Association の ccfs URI (`ccfs://<owner>/concrnt/<CDID>`) を指定して行う。
削除に成功した場合、サーバーは関連付け先リソースに対して `unassociated` イベント (CIP-11) を配信する。

## 4. Association の取得

本節のエンドポイントを提供するサーバーは、CIP-0 のサービスディスカバリにおいて、
それぞれのエンドポイント名で広告しなければならない (MUST)。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.resolve": "/resource/{uri}",
    "net.concrnt.core.associations": "/associations{?uri,schema,variant,author,since,until,limit,order}",
    "net.concrnt.core.association-counts": "/association-counts{?uri,schema}"
  }
}
```

サーバーは、リクエスタ (CIP-2。未認証の場合はゲスト) が読み取り (`association:read`, CIP-12) を
許可されない Association を、`associations` の結果および `association-counts` の集計から
除外しなければならない (MUST)。

### 4.1 associations

`net.concrnt.core.associations` エンドポイントにアクセスすることで、
対象 Document に関連付けられた Association Document の一覧を取得できる。

query パラメータとして以下をサポートする。

* `uri`: 関連付け先の Document を指定する CCURI。必須である (MUST)。
* `schema`: 取得する Association Document の schema を指定する。省略時は全ての schema を対象とする。
* `variant`: 取得する Association Document の variant を指定する。空値 (`variant=`) は
  variant を持たない (空文字列の) Association への絞り込みを意味する。省略時は全ての variant を
  対象とする。
* `author`: 取得する Association Document の author を指定する。省略時は全ての author を対象とする。

サーバーは、これらに加えて CIP-5 §3.1 と同様の `limit` / `since` / `until` / `order`
パラメータを受け付けなければならない (MUST)。

レスポンス形式およびページングの意味論は CIP-5 §3.2 / §3.3 に従う
(`{"items": [...], "prev": ..., "next": ...}` 形式。ソートキーは Association の `createdAt`)。

### 4.2 association-counts

`net.concrnt.core.association-counts` エンドポイントにアクセスすることで、
対象 Document に関連付けられた Association Document の件数を取得できる。

query パラメータとして以下をサポートする。

* `uri`: 関連付け先の Document を指定する CCURI。必須である (MUST)。
* `schema`: 集計モードを指定する。
  * 省略時: schema ごとの件数のマップを返す。
  * 指定時: 当該 schema の Association について、variant ごとの件数のマップを返す。

JSON オブジェクトのメンバ順序に規範的な意味はない。参考実装は、各 variant の最初の Association が
作成された順序でメンバを出力するが、クライアントはこの順序に依存すべきではない (SHOULD NOT)。

## 5. Security Considerations

* Association はリモートエンティティの名前空間 (associate の権威サーバー) への書き込みである。
  重複判定 (§3.1) とポリシー評価 (`association:create`, CIP-12) が受理の関門となる。
* 読み取り除外 (§4) を省略すると、ポリシーで保護された Association が一覧・件数として漏えいする。

## 6. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-4 – Delete
* CIP-7 – Distribution
* CIP-12 – Policy (association:read / association:create)
