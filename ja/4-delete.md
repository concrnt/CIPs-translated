# CIP-4 Delete

## 0. Abstract

本ドキュメントでは、Concrnt Document (CIP-1) を削除するための削除Documentの形式と、その処理を定義する。

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

## 3. 削除Document

Concrnt Documentの削除は、`kind` フィールドに `"delete"` を指定したDocumentを
Commitエンドポイント (CIP-3) に送信することで行う。
削除Documentの `value` フィールドには、削除対象をCCURIで指定する。

```json
{
  "kind": "delete",
  "schema": "https://...",
  "value": "ccfs://<owner>/concrnt/<cdid>",
  "author": "con1...",
  "createdAt": "2025-11-23T12:34:56Z"
}
```

削除対象として指定できるのは以下である。

* `cckv://` URI: キーに紐づくrecordの削除
* `ccfs://<owner>/concrnt/<cdid>` URI: CDID指定によるrecordまたはassociationの削除
* 末尾に `*` を付与した `cckv://` URI: 範囲削除 (§4)

サーバーは削除の実行前に、リクエスタが対象Documentを削除する権限を持つことを
ポリシー評価 (CIP-12) によって確認しなければならない (MUST)。

## 4. 範囲削除 (Range Delete)

削除対象の `cckv://` URIの末尾に `*` を付与することで、キー階層の配下を一括で削除できる。

* `cckv://<owner>/<key>*` — `<key>` 自身と、その配下のすべての子孫キーを削除する。
* `cckv://<owner>/<key>/*` — `<key>` 自身は削除せず、配下のすべての子孫キーのみを削除する。

```json
{
  "kind": "delete",
  "value": "cckv://con1alice/posts/2025*",
  "author": "con1alice",
  "createdAt": "2025-11-23T12:34:56Z"
}
```

### 4.1 対象の選択規則

* 範囲のマッチングは**キーのパス階層**によって行う。文字列の前方一致ではない。
  すなわち `cckv://con1alice/item*` の対象は `item` 自身と `item/...` 配下のみであり、
  兄弟キー `item2` は決して対象にならない (MUST)。
* `*` は削除対象の末尾の範囲指定子としてのみ使用できる。それ以外の位置に `*` を含む
  削除対象は不正であり、拒否しなければならない (MUST)。
* 範囲削除の対象は `cckv` スキームでなければならず (MUST)、key部が空であってはならない (MUST)。
  エンティティ配下全体 (`cckv://<owner>*` 等) を範囲削除の対象にすることはできない。
* 範囲に一致するDocumentが1件も存在しない場合、削除は失敗する (対象が見つからない旨のエラー)。

### 4.2 処理セマンティクス

範囲削除は **all-or-nothing** で処理されなければならない (MUST)。

1. サーバーは、範囲に一致するすべての現存Documentを列挙する。
2. 列挙した**各Documentに対して個別に**削除のポリシー評価 (CIP-12) を行う。
   1件でも権限のないDocumentが含まれる場合、範囲削除全体を失敗させ、
   いかなるDocumentも削除してはならない (MUST NOT)。
3. すべての対象について権限が確認できた場合にのみ、列挙した集合を削除する。
   列挙後に新たに作成されたDocumentは、この削除の対象に含めてはならない (MUST NOT)。

削除された各Documentについて、単一削除と同様に、tombstoneの記録 (§5)、
削除イベントの発行、配布先への伝搬 (§6)、Chunkline撤回リストへの掲載 (§7) を行う。

レスポンスとして返す対象は、`<key>*` 形式 (自身を含む) の場合は `<key>` 自身のDocumentである。

## 5. tombstoneとリプレイ防御

削除の永続性は、CIP-3 §3.4 で定義されるリプレイガード (backdate windowと
削除文書のコンテンツID tombstone) によって担保される。
削除が受理された場合、サーバーは削除された各Documentのccfs URIをtombstoneとして記録する。

tombstoneの記録は、削除対象を管理するサーバー (authoritative server) の責務である。
自身が管理するDocumentへの削除を受理した場合はtombstoneを記録するが、
リモートのEntityに帰属する削除を中継・転送する場合には、中継サーバーはtombstoneを記録しない。
リプレイ防御は最終的にauthoritative server側のtombstoneによって担保される。

## 6. 削除の伝搬

削除が受理された場合、サーバーは削除イベント (`deleted` / `unassociated`, CIP-11) を発行する。
削除対象のDocumentが `distributes` を持つ場合は、削除を各配布先へ伝搬する (CIP-7)。

リモートのリソースへ削除を伝搬する場合、配送されるSigned Documentの `references` には、
削除された対象のSigned Documentを同梱しなければならない (MUST)。
これにより、受信側は追加の問い合わせなしに削除対象を特定できる。
範囲削除の場合は、削除された対象ごとに1つの配送を行い、それぞれに該当対象を同梱する。


## 7. Security Considerations

* 範囲削除は1コミットで多数のDocumentを削除できる。対象ごとのポリシー評価 (§4.2) を
  省略してはならない (MUST NOT)。
* 範囲マッチングを文字列前方一致で実装すると、兄弟キー (`item*` に対する `item2`) を
  誤って削除する。実装は必ずパス階層でマッチングすること (§4.1)。

## 8. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-3 – Commit (リプレイガード、tombstone)
