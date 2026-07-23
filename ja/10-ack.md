# CIP-10 Ack

## 0. Abstract

この仕様では、あるエンティティが他のエンティティを承認したこと (フォロー等) を表現する **Ack** Documentを定義する。

## 1. Status of This Memo

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、
実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Ack Document

Ackは、あるentityが他のentityを承認したことを示すDocumentである。
CIP-1で定義されたConcrnt Documentのうち、`kind` が `"ack"` または `"unack"` であるものをAck Documentと呼ぶ。

Association (CIP-9) と同様に `associate` フィールドで対象を指定するが、
Associationとは状態モデル・データモデルの完全に異なる独立したDocumentタイプである
(Associationは不変のレコードの集合、Ackは3つ組ごとの単一の状態)。

```json
{
  "kind": "ack",
  "schema": "https://...",
  "value": { ... },
  "author": "con1alice",
  "associate": "cckv://con1bob",
  "createdAt": "2025-11-23T12:34:56Z"
}
```

* `associate` フィールドには、承認先entityそのものを指す、key部を持たないcckv URI (`cckv://<CCID>`) を指定する (MUST)。
* `schema` の値を変化させることで、様々な種類の承認 (フォロー等) を表現できる。
* `key` に値が入っていてはならない (MUST NOT)。

クライアントは、Ack Documentを、自身 (`author`) の所属サーバー、または `associate` のownerの
所属サーバーの、CIP-3で定義されるcommitエンドポイントへ送信する。
サーバーは、`author` と `associate` のownerの**いずれか一方でも**自身の管理下にある場合、
そのAck Documentを受理し状態を保存しなければならない (MUST)。
これはCIP-3 §3.1のコミット対象導出 (associate owner側のみ) に対する例外である。
`author` と `associate` のownerの**いずれも**管理していないサーバーの挙動は
CIP-3 §3.1の一般規則に従う: 421 Misdirected Request で拒否してもよく (MAY)、
no-opとして成功を返してもよい (MAY)。参考実装は、署名が有効であれば
`associate` のownerの所属サーバーへDocumentを中継する。

## 4. 状態モデル

Ackは、(送信元 `author`, 送信先 `associate` のowner, `schema`) の3つ組ごとに
1つの状態 (有効/無効) を持つ。

* `kind: "ack"` のDocumentをコミットすると、該当する3つ組のAck状態が **有効** になる。
* `kind: "unack"` のDocumentをコミットすると、該当する3つ組のAck状態が **無効** になる。
  unack Documentの形式はackと同一で、`kind` のみが異なる。

同一の3つ組に対する再コミットは、状態と対応するDocumentを上書きする (upsert)。
これにより ack → unack → ack のような状態遷移を表現でき、操作は冪等である。

ただし、上書きは無条件であってはならない。サーバーは、保存済みのAck/unack Documentと
新規Documentのdocument ID (CDID; `createdAt` 順の比較が可能) を比較し、
**新しい遷移である場合のみ**状態を変更しなければならない (MUST)。
古いまたは同一の遷移は、状態の変更・代理送信 (§5)・配布のいずれも行わない
副作用なしのno-opとして成功応答する (MUST)。
これにより、捕捉された古いack/unack Documentの再送による状態の巻き戻しは成立しない。

## 5. Ackの配布

Ackは送信元entityと送信先entityの両方のサーバーで保持される (MUST)。
送信先entityが他サーバーの管理下にある場合、送信元のサーバーはAck/unack Documentを
送信先entityを管理するサーバーのcommitエンドポイントへ代理で送信しなければならない (MUST)。
送信先の所属サーバーは、代理送信されたAck/unack Documentを、署名が有効である限り
通常のコミットとして受理しなければならない (MUST)。

### 5.1 認可

Ack/unackのコミットは、CIP-12のポリシースタック評価の**対象外**である。
サーバーが行う検査は、Signed Documentの署名検証、§3の経路検査 (author/associate ownerの管理)、
および相互ブロック関係の検査 (CIP-3 §4) のみである。
すなわち、有効な署名を持つauthorは、経路とブロックの制約の範囲内で自身のAck状態を自由に制御できる。
CIP-12のアクション語彙にはAck/unackに対応するアクションが存在しない (将来の拡張とする)。

また、Ack Documentは `distributes` フィールド (CIP-7) を持つことができ、
その場合CIP-7の規定に従いReference Documentが配布される。

## 6. Ackの取得

CIP-0で定義されるサービスディスカバリにおいて、以下のエンドポイントを追加して広告する。

```json
{
  "endpoints": {
    "net.concrnt.core.acknowledges": "/api/v2/acknowledges{?from,to,schema}",
    "net.concrnt.core.acknowledge-counts": "/api/v2/acknowledge-counts{?from,to,schema}"
  }
}
```

**acknowledges**: 有効なAckのDocument一覧を、`createdAt` 昇順のSigned Document配列として返す。

queryパラメータとして以下をサポートする。

* `from`: 送信元entityのCCIDで絞り込む。
* `to`: 送信先entityのCCIDで絞り込む。
* `schema`: schemaで絞り込む。
* `from` と `to` の少なくとも一方は指定しなければならない (MUST)。

**acknowledge-counts**: 同じパラメータを受け付け、有効なAckのschemaごとの件数マップを返す。

なお、`net.concrnt.core.acknowledges` はポリシー評価における ConcrntCall (CIP-12) の
呼び出し先としても利用される (例: Ackされている場合のみ閲覧許可)。

## 7. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-9 – Association
