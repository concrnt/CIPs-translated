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

Ack の**状態** (§4) は、送信元エンティティと送信先エンティティの両方のサーバーで
保持されなければならない (MUST)。送信先エンティティが他サーバーの管理下にある場合、
送信元のサーバーは Ack / unack Document を送信先エンティティを管理するサーバーの
Commit エンドポイントへ代理で送信しなければならない (MUST)。
送信先の所属サーバーは、代理送信された Ack / unack Document を、署名が有効である限り
受理し状態へ適用しなければならない (MUST)。

ただし、Document の**保持形態**は両側で異なる:

* **送信元 (author) の所属サーバー**は、Ack / unack Document をコミットログとして記録する。
  このコミットの所有者は author のみである。
* **送信先 (associate owner) の所属サーバー**は、受理した Ack / unack Document 自体を
  コミットログとして記録**してはならない** (MUST NOT)。代わりに、そこから導出した
  acked / unacked ミラー Document (§5.2) を、associate owner を所有者とする自身のコミットとして
  記録する (SHOULD。被承認側のリポジトリ可搬性のため)。
  author と associate owner が同一サーバーに所属する場合は、Ack コミット (author 所有) と
  ミラーコミット (associate owner 所有) の両方がそのサーバーに記録される。

これにより、あらゆるコミットの所有者は正確に 1 エンティティに定まり、退会時の GC が
特殊ケースなしに機能する。送信先サーバーでの重複排除・順序制御は、コミットログではなく
§4 の CDID 比較が担う (古いまたは同一の Ack はコミットログを残さない no-op — CIP-3 の
「何も適用しなかったコミットは記録してはならない」規則と整合する)。

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

### 5.2 Acked / Unacked Mirror Document

acked / unacked ミラーは、受理した Ack / unack から associate owner の管理サーバーが
自動生成する、被承認側の保持を表す Document である。ミラーは連合ワイヤに流通せず、
生成したサーバー自身のコミットとしてのみ記録される (リポジトリダンプ→リプレイ経由の移動を除く)。

**導出規則 (MUST)**: ミラー Document は、元の Ack / unack Document をパースし、
`kind` のみを対応する値 (`ack` → `acked`, `unack` → `unacked`) に置換して、
CIP-1 の正準フィールド集合で再シリアライズしたものである。他のフィールド
(`schema`, `value`, `author`, `associate`, `createdAt` 等) は元 Document と同一でなければ
ならない (MUST)。導出は決定的であり、同一の Ack からは常に同一のミラー (同一の CDID) が
得られる — 再試行・遡及生成は既存コミットの重複として no-op になる。

**proof (MUST)**: ミラーの proof は `ack-reference` type (CIP-1 §7.4.1) であり、
`document` / `proof` フィールドに元の Ack / unack の Signed Document を丸ごと埋め込む。
これによりミラーは外部解決なしに自己完結で検証できる (リポジトリリプレイを含む)。

**検証規則 (MUST)**: 検証者は (1) 埋め込まれた Signed Document を再帰的に検証し
(埋め込み Document の proof は `concrnt-ecrecover-direct` または `concrnt-ecrecover-subkey` で
なければならない)、(2) ミラー Document が埋め込み Document からの正準導出とバイト列一致する
ことを確認する。この 1 つの比較が kind の対応 (acked→ack / unacked→unack) と全フィールドの
一致を同時に束縛する。第三者が有効な Ack から正規のミラーを構築・再送しても、結果は正規生成と
同一の冪等な適用にしかならず、無害である。

**適用規則 (MUST)**: ミラーのコミットは、埋め込まれた元 Ack の CDID を用いて §4 の状態遷移を
適用する。状態の新旧比較の唯一の基準は常に**元の Ack / unack の CDID**であり、ミラー自身の
CDID を比較に用いてはならない (MUST NOT)。associate owner を管理しないサーバーは、
ミラーのコミットを受理してはならない (MUST NOT)。

**検査の免除**: ミラーの `createdAt` は元 Ack から継承されるため、コミット時の
backdate 検査 (CIP-3 §3.4) の適用外である (時間的正当性は元 Ack の受理時に検査済み)。
また §5.1 と同様にポリシー評価の対象外であり、相互ブロック検査も再適用しない
(元 Ack の受理時に検査済みであり、事後のブロックが被承認側自身の保持のリプレイを
妨げてはならない)。

**同時刻の遷移について**: ack と unack が同一の `createdAt` を持つ場合、§4 の比較は
CDID の内容ハッシュ部で決着する。これは author 自身が意図的に作り出せる状況に限られ、
両側のサーバーは同一の勝者に収束する。

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

各アイテムとして返す Signed Document は、サーバーが保持している側の Document である:
author を管理するサーバーは元の Ack Document (`kind: "ack"`) を、associate owner のみを
管理するサーバーはその acked ミラー (`kind: "acked"`、§5.2) を返す。ミラーは `kind` 以外の
全フィールドが元 Ack と同一であるため、クライアントから見える情報は等価である。

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
