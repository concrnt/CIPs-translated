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

削除された各Documentについて、単一削除と同様に、
削除イベントの発行、配布先への伝搬 (§6)、Chunkline撤回リストへの掲載 (§7) を行う。

レスポンス (CIP-3 §3.3) として返す対象は、削除された対象をURIの辞書順 (昇順) で並べたときの
先頭のDocumentである。すなわち `<key>*` 形式 (自身を含む) の場合は `<key>` 自身のDocumentであり、
`<key>/*` 形式 (自身を含まない) の場合は最初の子孫キーのDocumentである。

## 5. リプレイ防御

削除の永続性は、CIP-3 §3.4 で定義されるリプレイガード (backdate windowと
コミットログによる重複排除) によって担保される。
削除された各Document、および削除コマンド (`kind: "delete"`) 自身のdocument IDは
コミットログに残り続けるため、捕捉されたそれらのDocumentを再コミットしても
no-opとなり適用されない。

リプレイ防御は、削除対象を管理するサーバー (authoritative server) 自身のコミットログと
backdate windowによって担保される。リモートのEntityに帰属する削除を中継・転送する
サーバーに、追加の防御は要求されない。

## 6. 削除の伝搬

削除が受理された場合、サーバーは削除イベント (`deleted` / `unassociated`, CIP-11) を発行する。
削除対象のDocumentが `distributes` を持つ場合は、削除を各配布先へ伝搬する (CIP-7)。

リモートのリソースへ削除を伝搬する場合、配送されるSigned Documentの `references` には、
削除された対象のSigned Documentを同梱しなければならない (MUST)。
これにより、受信側は追加の問い合わせなしに削除対象を特定できる。
範囲削除の場合は、削除された対象ごとに1つの配送を行い、それぞれに該当対象を同梱する。

### 6.1 受信側の検証

自身が削除対象のauthoritative serverではないサーバー (配布先を管理するサーバー) が
伝搬された削除を受理してよいのは、以下の**すべて**を確認できた場合に限られる (MUST)。

1. `references` に同梱された削除対象のSigned Documentが構造的・暗号学的に有効である
   (proofの検証を通過し、導出したcckv/ccfs identityが `references` のキーのURIと一致する)。
2. 同梱された対象Documentの `author` が、削除Documentの `author` と一致する。
3. 同梱された対象Documentの署名済み `distributes` に、自身が管理する配布先が含まれている。
4. 削除しようとするローカルのDocumentが、その配布先に対して自動生成されたReference Document
   (`<配布先CCURI>/<対象のCDID>`、CIP-7 §4.1) であり、保存されているReferenceの `value.href` が
   同梱された対象を指している。
5. そのReference Documentに対する削除のポリシー評価 (CIP-12) が許可となる。

伝搬された削除によって削除できるのは、上記の自動生成されたReference Documentのみである。
それ以外のローカルDocumentを伝搬削除の対象としてはならない (MUST NOT)。
確認に失敗した伝搬削除は拒否する。


## 7. Security Considerations

* 範囲削除は1コミットで多数のDocumentを削除できる。対象ごとのポリシー評価 (§4.2) を
  省略してはならない (MUST NOT)。
* 範囲マッチングを文字列前方一致で実装すると、兄弟キー (`item*` に対する `item2`) を
  誤って削除する。実装は必ずパス階層でマッチングすること (§4.1)。

## 8. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-3 – Commit (リプレイガード、コミットログ)
