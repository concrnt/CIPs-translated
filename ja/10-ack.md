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

クライアントは、Ack Document を、自身 (`author`) の所属サーバーの Commit エンドポイント (CIP-3) へ
送信する。Ack / unack Document のコミット対象は `author` の所属サーバーであり (CIP-3 §3.1)、
`associate` の owner 側への伝達は §5 の acked / unacked Document によって行われる。
`author` を管理していないサーバーの挙動は CIP-3 §3.1 の一般規則に従う:
421 Misdirected Request で拒否してもよく (MAY)、no-op として成功を返してもよい (MAY)。
raw の Ack / unack Document を他サーバーへ中継してはならない (MUST NOT)。

## 4. 状態モデル

Ack は、(送信元 `author`, 送信先 `associate` の owner, `schema`) の 3 つ組ごとに
1 つの状態 (有効 / 無効) を持つ。

* `kind: "ack"` の Document をコミットすると、該当する 3 つ組の Ack 状態が**有効**になる。
* `kind: "unack"` の Document をコミットすると、該当する 3 つ組の Ack 状態が**無効**になる。
  unack Document の形式は ack と同一で、`kind` のみが異なる。

同一の 3 つ組に対する再コミットは、状態と対応する Document を上書きする (upsert)。
これにより ack → unack → ack のような状態遷移を表現でき、操作は冪等である。

ただし、上書きは無条件であってはならない。サーバーは、保存済みの Document と
新規 Document の `createdAt` を比較し、新規 Document の `createdAt` が**厳密に新しい場合のみ**
状態を変更しなければならない (MUST)。`createdAt` が古いまたは同一の遷移は、状態の変更・
acked / unacked の送信 (§5)・配布のいずれも行わない副作用なしの no-op として成功応答する (MUST)。
これにより、捕捉された古い ack / unack Document の再送による状態の巻き戻しは成立しない。
比較キーは `createdAt` のみであり、CDID の内容ハッシュ部を順序付けに用いてはならない (MUST NOT)。
`createdAt` は acked / unacked Document (§5.2) にそのまま継承されるため、送信元・送信先の両サーバーは
同一のキーで同一の判定を行う。

## 5. Ack の配布

Ack の**状態** (§4) は、送信元エンティティと送信先エンティティの両方のサーバーで
保持されなければならない (MUST)。ただし、両側が保持する **Document は異なる**:

* **送信元 (author) の所属サーバー**は、Ack / unack Document をコミットログとして記録する。
  このコミットの所有者は author のみである。
* **送信先 (associate owner) の所属サーバー**は、Ack / unack Document から導出された
  acked / unacked Document (§5.2) をコミットログとして記録する。このコミットの所有者は
  associate owner のみである。raw の Ack / unack Document 自体を記録**してはならない** (MUST NOT)。

送信元の所属サーバーは、Ack / unack Document を受理したとき、そこから acked / unacked Document を
導出し、`associate` の owner の所属サーバーの Commit エンドポイントへ送信しなければならない (MUST)。
これは CIP-7 の配布が元 Document ではなく Reference Document を配布先へ送るのと同じ構造であり、
raw の Ack / unack Document は author の所属サーバーを離れない (リポジトリダンプ→リプレイ経由の
移動を除く)。**送信先が自サーバー自身であっても同じ経路で処理する** (author と associate owner が
同一サーバーに所属する場合、Ack コミット (author 所有) と acked コミット (associate owner 所有) の
両方がそのサーバーに記録される)。これにより送信先が自サーバーか他サーバーかで処理が分岐しない。

送信先の所属サーバーは、受信した acked / unacked Document を、§5.2 の検証に合格する限り
受理し状態へ適用しなければならない (MUST)。

リポジトリダンプの再生 (サーバー運用者の管理経路によるコミット) では acked / unacked Document を
送信しない。CIP-7 の Reference 配布と同様、再生は自サーバーの保持を復元するだけであり、
送信先側の保持はその側のダンプに含まれている。

これにより、あらゆるコミットの所有者は正確に 1 エンティティに定まり、退会時の GC が
特殊ケースなしに機能する。重複排除・順序制御は §4 の `createdAt` 比較が担う (古いまたは同一の遷移は
コミットログを残さない no-op — CIP-3 の「何も適用しなかったコミットは記録してはならない」規則と
整合する)。

acked / unacked Document の送信の再試行は、CIP-7 §4.2.1 の規則に従う: 再試行では最初に生成したものと
同一の Signed Document を再送しなければならず (MUST。導出は決定的なので同一 CDID になる)、
指数バックオフによる再試行を行うべきである (SHOULD)。再送は §4 の新旧比較により冪等である。

### 5.1 認可

Ack / unack のコミットは、CIP-12 のポリシースタック評価の**対象外**である。
サーバーが行う検査は、Signed Document の署名検証、§3 の経路検査 (author の管理)、
および相互ブロック関係の検査 (CIP-3 §4) のみである。
すなわち、有効な署名を持つ author は、経路とブロックの制約の範囲内で自身の Ack 状態を自由に制御できる。
CIP-12 のアクション語彙には Ack / unack に対応するアクションが存在しない (将来の拡張とする)。

また、Ack Document は `distributes` フィールド (CIP-7) を持つことができ、
その場合 CIP-7 の規定に従い Reference Document が配布される。

### 5.2 Acked / Unacked Document

acked / unacked Document は、受理した Ack / unack から author の管理サーバーが自動生成する、
被承認側の保持を表す Document である。§5 のとおり associate owner の管理サーバーへ送信され、
そのサーバー自身のコミットとして記録される。

**導出規則 (MUST)**: acked / unacked Document は、元の Ack / unack Document をパースし、
`kind` のみを対応する値 (`ack` → `acked`, `unack` → `unacked`) に置換して、
CIP-1 の正準フィールド集合で再シリアライズしたものである。他のフィールド
(`schema`, `value`, `author`, `associate`, `createdAt` 等) は元 Document と同一でなければ
ならない (MUST)。導出は決定的であり、同一の Ack からは常に同一の Document (同一の CDID) が
得られる — 再送・遡及生成は既存コミットの重複として no-op になる。

**proof (MUST)**: proof は `document-direct` type (CIP-1 §7.4.1) であり、
`document` / `proof` フィールドに元の Ack / unack の Signed Document を丸ごと埋め込む。
これにより acked / unacked Document は外部解決なしに自己完結で検証できる (リポジトリリプレイを含む)。

**検証規則 (MUST)**: 検証者は (1) 埋め込まれた Signed Document を再帰的に検証し
(埋め込み Document の proof は `concrnt-ecrecover-direct` または `concrnt-ecrecover-subkey` で
なければならない)、(2) acked / unacked Document が埋め込み Document からの正準導出とバイト列一致する
ことを確認する。この 1 つの比較が kind の対応 (acked→ack / unacked→unack) と全フィールドの
一致を同時に束縛する。第三者が有効な Ack から正規の acked / unacked Document を構築・送信しても、
結果は正規生成と同一の冪等な適用にしかならず、無害である。`kind` が `acked` / `unacked` の
Document は `document-direct` 以外の proof で受理してはならない (MUST NOT。author が直接署名した
`acked` は無効である)。

**適用規則 (MUST)**: acked / unacked のコミットは、§4 の規則に従い自身の `createdAt`
(元 Ack / unack から継承した値) を保存済み状態の `createdAt` と比較して状態遷移を適用する。
associate owner を管理しないサーバーの挙動は CIP-3 §3.1 の一般規則に従う (何も適用せず no-op として
成功を返してよく (MAY)、421 で拒否してもよい (MAY))。いずれにせよ状態を保存してはならない (MUST NOT)。

**検査の免除**: acked / unacked の `createdAt` は元 Ack から継承されるため、コミット時の
backdate 検査 (CIP-3 §3.4) の適用外である (時間的正当性は元 Ack の受理時に検査済み。
配送の再試行遅延やリポジトリリプレイで backdate window を超えうる)。
また §5.1 と同様にポリシー評価の対象外であり、相互ブロック検査も再適用しない
(元 Ack の受理時に検査済みであり、事後のブロックが被承認側自身の保持のリプレイを
妨げてはならない)。author の Entity が解決できない場合 (author のサーバー消滅後のリプレイ等) も
受理する (埋め込み Document の署名が author の身元を保証する)。

**同時刻の遷移について**: 同一の 3 つ組に対して同一の `createdAt` を持つ ack と unack が存在する場合、
§4 の比較により後から届いた方は no-op になる。これは author 自身が意図的に作り出せる状況に限られ、
仕様上の順序は定義しない (両側のサーバーで結果が異なりうることを許容する)。

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
* `from` と `to` は**いずれか一方のみ**を指定しなければならない (MUST)。両方を指定した、
  またはどちらも指定しないリクエストは 400 Bad Request で拒否する (MUST)。

サーバーは、これらに加えて CIP-5 §3.1 と同様の `limit` / `since` / `until` / `order`
パラメータを受け付けなければならない (MUST)。デフォルトの順序は `desc` である。

レスポンス形式およびページングの意味論は CIP-5 §3.2 / §3.3 に従う
(`{"items": [...], "prev": ..., "next": ...}` 形式。ソートキーは Ack Document の `createdAt`)。

各アイテムとして返す Signed Document は、サーバーが保持している側の Document である:
`from` で絞り込んだ場合は author 側の保持である Ack Document (`kind: "ack"`) を、
`to` で絞り込んだ場合は associate owner 側の保持である acked Document (`kind: "acked"`、§5.2) を返す。
acked は `kind` 以外の全フィールドが元 Ack と同一であるため、クライアントから見える情報は等価である。

**acknowledge-counts**: 同じパラメータを受け付け、有効な Ack の schema ごとの件数マップを返す。

なお、`net.concrnt.core.acknowledges` はポリシー評価における ConcrntCall (CIP-12) の
呼び出し先としても利用される (例: Ack されている場合のみ閲覧許可)。

## 7. Security Considerations

* Ack / unack はポリシー評価の対象外である (§5.1) ため、`schema` を変えるだけで
  (author, associate owner) の組に対して任意個の状態レコードを作成できる。
  サーバーは、組あたりの schema 数および単位時間あたりの Ack コミット数に上限を
  課すべきである (SHOULD)。上限超過は HTTP 429 で拒否してよい (MAY)。
  §5 の受理義務 (MUST) は、これらの資源保護のための拒否を妨げない。
* acked / unacked の送信の恒久的な失敗により、送信元と送信先の Ack 状態は不整合になりうる。
  ConcrntCall による認可 (CIP-12) は問い合わせ先サーバーの状態に依存するため、
  ポリシー作成者は resolver の選択に注意すること。

## 8. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-7 – Distribution (配送の構造・再試行)
* CIP-9 – Association
* CIP-12 – Policy (ConcrntCall)
