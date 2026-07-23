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
  これはauthoritative serverでの処理についての規定であり、伝搬された削除を受信した側の
  扱いは §6.1 (no-op) による。

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
backdate windowによって担保される。伝搬された削除を受信したサーバーは、`references` から
削除対象を復元し処理を行った場合にのみコミットログを記録する (同一の削除が配布先ごとに
複数回配送された場合の重複排除になる)。対象を復元できずno-opとした場合は記録しない (CIP-3 §3.4)。
これを超える追加の防御は、受信側サーバーに要求されない。

## 6. 削除の伝搬

削除が受理された場合、サーバーは削除イベント (`deleted` / `unassociated`, CIP-11) を発行する。
削除対象のDocumentが `distributes` を持つ場合は、削除を各配布先へ伝搬する (CIP-7)。

リモートのリソースへ削除を伝搬する場合、配送されるSigned Documentの `references` には、
削除された対象のSigned Documentを同梱しなければならない (MUST)。
これにより、受信側は追加の問い合わせなしに削除対象を特定できる。
範囲削除の場合は、削除された対象ごとに1つの配送を行い、それぞれに該当対象を同梱する。
削除対象がassociationである場合は、関連先 (`associate`) のSigned Documentも
同梱しなければならない (MUST)。受信側は `unassociated` イベントの構成にこれを必要とし、
欠落した配送を拒否してよい。

### 6.1 受信側の処理 (参照行のスイープ)

削除対象のauthoritative serverではないサーバー (配布先を管理するサーバー) は、
伝搬された削除を、自身が保持する自動生成Reference Document (CIP-7 §4.1) の
**スイープ**として処理する。処理はcreate/associateの配布処理と対称であり、
以下の手順による。

1. **対象の復元**: `references` から削除対象のSigned Documentを取り出す。
   単一削除では削除対象URIと同一のキーのエントリ、範囲削除では範囲に一致する
   キーのエントリが対象である。対象が1件も同梱されていない削除は、エラーとせず
   no-opとして成功を返す (MUST)。このときコミットログを記録してはならない
   (MUST NOT) — 後から対象を同梱した正当な配送が、重複排除によって
   無視されてしまうことを防ぐためである。
2. **参照行のアドレス指定**: 各対象について、同梱されたDocumentのバイト列から
   document ID (CDID) を再導出し、対象の署名済み `distributes` の各配布先 `<dest>` に
   ついて、自動生成Reference Documentのキー `<dest>/<CDID>` をローカルストレージから引く。
   行が存在しない配布先 (リモートの配布先・未配送の配布先) は単にスキップする。
   document IDはDocumentの内容全体から導出されるため、このアドレス指定自体が
   「同梱されたバイト列」と「保存済み配布の対象」の同一性の束縛となる:
   偽造・改変されたDocumentは異なるdocument IDを導出するため、
   いかなる保存済み行にも一致しない。
3. **行ごとのポリシー評価**: 存在した各Reference行について、**保存済みの**
   Reference Documentをselfとして削除のポリシー評価 (CIP-12) を行う (MUST)。
   拒否された行は削除せずに残し、残りの行の処理は継続する
   (配布先のポリシーによる拒否は、削除全体を失敗させない)。
   デフォルトポリシーの下では、Reference Documentのauthor (=削除対象のauthor) と
   配布先名前空間の所有者のみが削除を許可される。
4. **削除と広告**: 許可された行を削除し、Chunkline撤回リスト (§7) への掲載と
   削除イベントの発行を自身の購読者に対して行う。受信側サーバーは削除を
   再連合してはならない (削除の再配送はauthoritative serverのみが行う)。

伝搬された削除によって削除できるのは、手順2でアドレスされた自動生成
Reference Documentのみである。それ以外のローカルDocumentを伝搬削除の
対象としてはならない (MUST NOT)。

なお、authoritative server自身も、対象本体の削除と同一トランザクションで、
自身が管理する配布先の参照行に対して同じスイープ (手順2〜4) を行う。
これにより、配布先が同一サーバー上にある場合も参照行が孤児化しない。


## 7. Security Considerations

* 範囲削除は1コミットで多数のDocumentを削除できる。対象ごとのポリシー評価 (§4.2) を
  省略してはならない (MUST NOT)。
* 範囲マッチングを文字列前方一致で実装すると、兄弟キー (`item*` に対する `item2`) を
  誤って削除する。実装は必ずパス階層でマッチングすること (§4.1)。
* 伝搬された削除の受信処理 (§6.1) において、参照行の削除自体はdocument IDによる
  アドレス指定とポリシー評価で保護される。一方、削除イベント (CIP-11) の発行は
  `references` の同梱内容に基づくため、第三者の作為的な配送により実際には何も
  削除されていない対象の `deleted` / `unassociated` イベントが発行されうる。
  イベントは助言的な通知であり、クライアントは対象の実際の状態を
  resolveで確認すること (CIP-11 §4 参照)。

## 8. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-3 – Commit (リプレイガード、コミットログ)
