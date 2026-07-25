# CIP-10: Ack

## 0. Abstract

この仕様では、あるエンティティが他のエンティティを承認したこと (フォロー等) を表現する
**Ack** Document を定義する。

## 1. Status of This Memo

このドキュメントは Ack Document の仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0, CIP-1, CIP-3 を前提とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Ack Document

Ack は、あるエンティティが他のエンティティを承認したことを示す Document である。
CIP-1 で定義された Concrnt Document のうち、`kind` が `"ack"` または `"unack"` であるものを
Ack Document と呼ぶ。

Association (CIP-9) と同様に `associate` フィールドで対象を指定するが、
Association とは状態モデル・データモデルの完全に異なる独立した Document タイプである
(Association は不変のレコードの集合、Ack は 3 つ組ごとの単一の状態)。

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

* `associate` フィールドには、承認先エンティティそのものを指す、key 部を持たない cckv URI
  (`cckv://<CCID>`) を指定する (MUST)。owner 部にリゾルバヒント・エイリアスを含めてはならない
  (MUST NOT。CIP-3 §3.1)。
* `schema` は必須であり (REQUIRED)、その値を変化させることで様々な種類の承認 (フォロー等) を表現できる。
* `key` に値が入っていてはならない (MUST NOT)。

クライアントは、Ack Document を、自身 (`author`) の所属サーバー、または `associate` の owner の
所属サーバーの Commit エンドポイント (CIP-3) へ送信する。
サーバーは、`author` と `associate` の owner の**いずれか一方でも**自身の管理下にある場合、
その Ack Document を受理し状態を保存しなければならない (MUST)。
これは CIP-3 §3.1 のコミット対象導出 (associate owner 側のみ) に対する例外である。

`author` と `associate` の owner の**いずれも**管理していないサーバーの挙動は
CIP-3 §3.1 の一般規則に従う: 421 Misdirected Request で拒否してもよく (MAY)、
no-op として成功を返してもよい (MAY)。no-op とする場合、あわせて `associate` の owner の
所属サーバーへ Document を中継してもよい (MAY)。参考実装は、署名が有効であれば no-op 成功としつつ
中継を行う。

## 4. 状態モデル

Ack は、(送信元 `author`, 送信先 `associate` の owner, `schema`) の 3 つ組ごとに
1 つの状態 (有効 / 無効) を持つ。

* `kind: "ack"` の Document をコミットすると、該当する 3 つ組の Ack 状態が**有効**になる。
* `kind: "unack"` の Document をコミットすると、該当する 3 つ組の Ack 状態が**無効**になる。
  unack Document の形式は ack と同一で、`kind` のみが異なる。

同一の 3 つ組に対する再コミットは、状態と対応する Document を上書きする (upsert)。
これにより ack → unack → ack のような状態遷移を表現でき、操作は冪等である。

ただし、上書きは無条件であってはならない。サーバーは、保存済みの Ack / unack Document と
新規 Document の CDID (`createdAt` 順の比較が可能) を比較し、**新しい遷移である場合のみ**
状態を変更しなければならない (MUST)。古いまたは同一の遷移は、状態の変更・代理送信 (§5)・配布の
いずれも行わない副作用なしの no-op として成功応答する (MUST)。
これにより、捕捉された古い ack / unack Document の再送による状態の巻き戻しは成立しない。

## 5. Ack の配布

Ack は送信元エンティティと送信先エンティティの両方のサーバーで保持されなければならない (MUST)。
送信先エンティティが他サーバーの管理下にある場合、送信元のサーバーは Ack / unack Document を
送信先エンティティを管理するサーバーの Commit エンドポイントへ代理で送信しなければならない (MUST)。
送信先の所属サーバーは、代理送信された Ack / unack Document を、署名が有効である限り
通常のコミットとして受理しなければならない (MUST)。

代理送信の再試行は、CIP-7 §4.2.1 の規則に従う: 再試行では最初に受理したものと同一の
Signed Document を再送しなければならず (MUST)、指数バックオフによる再試行を行うべきである (SHOULD)。
再送は §4 の新旧比較により冪等である。

### 5.1 認可

Ack / unack のコミットは、CIP-12 のポリシースタック評価の**対象外**である。
サーバーが行う検査は、Signed Document の署名検証、§3 の経路検査 (author / associate owner の管理)、
および相互ブロック関係の検査 (CIP-3 §4) のみである。
すなわち、有効な署名を持つ author は、経路とブロックの制約の範囲内で自身の Ack 状態を自由に制御できる。
CIP-12 のアクション語彙には Ack / unack に対応するアクションが存在しない (将来の拡張とする)。

また、Ack Document は `distributes` フィールド (CIP-7) を持つことができ、
その場合 CIP-7 の規定に従い Reference Document が配布される。

## 6. Ack の取得

本節のエンドポイントを提供するサーバーは、CIP-0 のサービスディスカバリにおいて、
それぞれのエンドポイント名で広告しなければならない (MUST)。

```json
{
  "endpoints": {
    "net.concrnt.core.acknowledges": "/acknowledges{?from,to,schema,since,until,limit,order}",
    "net.concrnt.core.acknowledge-counts": "/acknowledge-counts{?from,to,schema}"
  }
}
```

**acknowledges**: 有効な Ack の Document 一覧を返す。

query パラメータとして以下をサポートする。

* `from`: 送信元エンティティの CCID で絞り込む。
* `to`: 送信先エンティティの CCID で絞り込む。
* `schema`: schema で絞り込む。
* `from` と `to` の少なくとも一方は指定しなければならない (MUST)。

サーバーは、これらに加えて CIP-5 §3.1 と同様の `limit` / `since` / `until` / `order`
パラメータを受け付けなければならない (MUST)。デフォルトの順序は `desc` である。

レスポンス形式およびページングの意味論は CIP-5 §3.2 / §3.3 に従う
(`{"items": [...], "prev": ..., "next": ...}` 形式。ソートキーは Ack Document の `createdAt`)。

**acknowledge-counts**: 同じパラメータを受け付け、有効な Ack の schema ごとの件数マップを返す。

なお、`net.concrnt.core.acknowledges` はポリシー評価における ConcrntCall (CIP-12) の
呼び出し先としても利用される (例: Ack されている場合のみ閲覧許可)。

## 7. Security Considerations

* Ack / unack はポリシー評価の対象外である (§5.1) ため、`schema` を変えるだけで
  (author, associate owner) の組に対して任意個の状態レコードを作成できる。
  サーバーは、組あたりの schema 数および単位時間あたりの Ack コミット数に上限を
  課すべきである (SHOULD)。上限超過は HTTP 429 で拒否してよい (MAY)。
  §5 の受理義務 (MUST) は、これらの資源保護のための拒否を妨げない。
* 代理送信の恒久的な失敗により、送信元と送信先の Ack 状態は不整合になりうる。
  ConcrntCall による認可 (CIP-12) は問い合わせ先サーバーの状態に依存するため、
  ポリシー作成者は resolver の選択に注意すること。

## 8. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-7 – Distribution (配送の再試行)
* CIP-9 – Association
* CIP-12 – Policy (ConcrntCall)
