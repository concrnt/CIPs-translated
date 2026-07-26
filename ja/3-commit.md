# CIP-3: Commit

## 0. Abstract

本ドキュメントでは、Concrnt サーバーのエンドポイントを拡張し、サーバーが管理するリソースを
変更するための手段 (Commit) を定義する。

## 1. Status of This Memo

このドキュメントは Commit エンドポイントの仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0 および CIP-1 を前提とする。
`kind` ごとの処理は、対応する CIP (CIP-4, CIP-9, CIP-10 等) の実装を前提とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Commit エンドポイント

本 CIP を実装するサーバーは、HTTP POST リクエストを受け付ける Commit エンドポイントを提供し、
CIP-0 のサービスディスカバリにおいて `net.concrnt.core.commit` エンドポイント名で
広告しなければならない (MUST)。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.resolve": "/resource/{uri}",
    "net.concrnt.core.commit": "/commit"
  }
}
```

### 3.1 リクエスト形式

クライアントは、Commit エンドポイントに対して、CIP-1 で定義された Concrnt Signed Document を
HTTP POST リクエストのボディとして送信する。

クライアントは、コミット対象を管理する Concrnt サーバーに対してリクエストを送信しなければならない (MUST)。
コミット対象を管理するサーバーは、Document の内容から次のように導出される。

* `key` フィールドが存在する場合: `key` の owner 部が示す名前空間の権威サーバー
  (owner が CCID の場合はその Entity の所属サーバー、FQDN / CSID の場合はそのサーバー自身)。
* `associate` フィールドが存在する場合 (CIP-9): `associate` の owner 部が示す Entity の所属サーバー。
  ただし `kind` が `ack` / `unack` の場合は CIP-10 §3 の例外 (author 側でも受理) がある。
* `kind` が `delete` の場合 (CIP-4): `value` の削除対象 CCURI の owner 部が示す名前空間の権威サーバー。
* `kind` が `entity` の場合 (CIP-0): `value.domain` が示すサーバー (所属先サーバー)。

owner 部から所属サーバーへの解決は、CIP-0 で定義される名前解決 (Entity Document の `domain`) によって行う。

`kind` が `record` の Document は `key` を持たなければならない (MUST)。
key を持たない record はコミット対象の管理サーバーを導出できないため、拒否される。

サーバーがコミットに対して実行する処理は、Document の種別と自身の管理範囲によって決まる。
1 つのコミットが複数のサーバーにそれぞれ異なる処理を要求しうる点に注意すること。

* コミット対象の owner を管理していれば、対象本体の保存・削除を行う。
* `distributes` の配布先を管理していれば、Reference Document の生成 (CIP-7 §4.1) や
  削除の伝搬の適用 (CIP-4 §6.1) を行う。
* ack / unack は `author` と `associate` の owner のいずれか一方でも管理していれば受理する (CIP-10 §3)。
* 該当する処理を実行した場合、対応するイベント (CIP-11) を自身の購読者へ発行する。

自身の管理範囲に該当する処理が 1 つもないコミットについて、サーバーは
HTTP 421 Misdirected Request で拒否してもよく (MAY)、何も適用せず no-op として成功を
返してもよい (MAY)。参考実装は後者 (スルー) を採用する。
何も適用しなかった場合、そのコミットをコミットログに記録してはならない (MUST NOT、§3.4)。

また、権威・認可の対象となるフィールド (`key`・`associate`・削除対象・`distributes` の各エントリ) の
owner 部には、エイリアス形式 (`@<FQDN>`) およびリゾルバヒント付き形式 (`<CCID>@<FQDN>`)
(いずれも CIP-0 §7.2) を使用してはならない (MUST NOT)。サーバーはこれらを含むコミットを
拒否しなければならない (MUST)。署名済みの識別子が可変の DNS や提出者の指定するホストに依存すること、
および同一対象が異なる文字列表現を持つことを防ぐためである。クライアントはエイリアスを事前に
CCID へ解決 (CIP-0 §8.3) してから Document を作成すること。
Resolve 経路 (CIP-0 §9.3) でのエイリアス・ヒントの利用はこの制限の対象外である。

### 3.2 Document の種別 (kind)

Commit エンドポイントは、CIP-1 §5.1 で定義される `kind` レジストリに従って Document の処理を分岐する。
受け付ける `kind` は、当該サーバーが実装する CIP が定義するものに限られる。
未知の `kind`、および実装していない拡張の `kind` を持つ Document は
拒否しなければならない (MUST。CIP-1 §5.1)。

### 3.3 レスポンス形式

サーバーは、リクエストが成功した場合、HTTP 200 OK ステータスコードを返し、
レスポンスボディに受理された Concrnt Signed Document を返却する。

このとき、サーバーは受理したリソースの所在を示す以下のフィールドを Signed Document に付与して返却する。

```json
{
  "document": "<JSON string>",
  "proof": { ... },
  "cckv": "cckv://<owner>/<key>",
  "ccfs": "ccfs://<owner>/concrnt/<cdid>"
}
```

* `cckv`
  Document が `key` を持つ場合、そのキーを示す CCURI。
* `ccfs`
  Document の内容から生成された CDID (CIP-1) を示す CCURI。

`cckv` / `ccfs` は署名対象外の情報フィールド (CIP-1 §7) であり、サーバーが署名済みバイト列から
**自ら導出した値**を設定する。リクエストにクライアントが付与した外側の `cckv` / `ccfs` は
無視され、サーバー導出値で上書きされる。クライアントは、外側のメタデータを暗号学的な根拠として
扱ってはならない (MUST NOT)。

サーバーは、導出される ccfs URI の owner 部を次のように決定しなければならない (MUST)。

* `key` を持つ Document (record): `key` の owner 部 (CCID に限らず、FQDN / CSID 形式の場合もそのまま)。
* association: `associate` の owner 部。
* ack / unack: `associate` が指す対象 Entity の解決済み CCID。

すなわち ccfs identity の owner は**名前空間の所有者**であり、Document の `author` (署名者) ではない。
author と名前空間所有者が異なる Document (他者の名前空間への書き込み等) では両者が乖離する点に注意。

kind ごとの返却内容は以下の通りである。

* `record`: 送信された Signed Document に `cckv` と `ccfs` を付与したもの。
* `association` / `ack` / `unack`: 送信された Signed Document に `ccfs` を付与したもの。
* `entity`: 送信された Signed Document そのもの。
* `delete`: 削除された対象の Signed Document。

no-op 成功 (§3.4 の重複排除ヒット・accept-if-newer による古い Document の受理・管理外スルー等) の
場合も HTTP 200 を返し、ボディには提出された Signed Document を返却する。
このとき導出フィールド (`cckv` / `ccfs`) の付与は OPTIONAL である。

### 3.4 冪等性とリプレイ防止

Commit エンドポイントは署名済み Document をそのまま受理するため、攻撃者が捕捉した Document の
再送 (リプレイ) に対する防御を持たなければならない。サーバーは以下を実装しなければならない (MUST)。

* **未来時刻の拒否**: `createdAt` がサーバー時刻より一定以上未来である Document を拒否する。
  参考実装の許容スキューは 12 時間である。遠い未来の `createdAt` を許すと、accept-if-newer 比較に
  おいてその Document が以後のすべての更新に優先してしまう。
* **過去時刻の拒否 (backdate window)**: `createdAt` が一定以上過去である Document を拒否する。
  参考実装の窓は 7 日間である。
  例外として、`kind: "entity"` の Document には backdate window を適用してはならない (MUST NOT)。
  Entity Document は長寿命の署名であり、フェデレーションでの解決・再コミットを通じて発行から
  何年も経った同一 Document が恒常的に提示される。後述の accept-if-newer とコミットログの
  重複排除により、古い Entity Document のリプレイは no-op となるため巻き戻しは成立しない。
  また Entity Document の proof は `concrnt-ecrecover-direct` に限定される (CIP-0 §8.2) ため、
  この免除が漏洩サブキーによるバックデート署名 (CIP-13 §8) の受理範囲を広げることはない。
* **コミットログによる重複排除**: サーバーは、適用した Document の CDID を**コミットログ**として
  記録する。Commit 処理の冒頭で、受信した Document の CDID がすでにコミットログに存在する場合には、
  Document を再適用せず、no-op として成功を返す。
* **コミットログの保持期間**: コミットログのエントリを破棄する場合、少なくとも
  「コミット処理時刻」と「対象 Document の `createdAt`」の**遅い方**に backdate window を加えた
  時刻までは保持しなければならない。
* **適用されなかったコミットの非記録**: accept-if-newer 比較で古いと判定された場合など、
  実際には何も適用しなかったコミットについては、コミットログを記録してはならない (MUST NOT)。

CDID は Document の内容全体から導出されるため、重複排除により同一 Document の再送
(クライアントのリトライ、連合経路の再配送、ダンプの再インポート) は安全な冪等操作となる。
同時に、これがリプレイガードでもある。削除済み・上書き済みの Document の CDID も、
削除を行った `kind: "delete"` の Document 自身の CDID もコミットログに残り続けるため、
捕捉された Document を再送しても適用されず、同じキー (または範囲) に後から作成された Document を
過去の削除コマンドで削除させることもできない。保持期間を上記の下限より短くすると、
未来スキュー上限付近の `createdAt` を持つ Document が、エントリ破棄の時点でまだ backdate window の
範囲内に残り、スキュー分の期間だけ再コミット可能になる。

コミットログの重複チェックは署名検証に先立って行ってよい (MAY)。CDID は Document の全バイト列から
導出されるため、チェックの結果がリクエスタのすでに保持する情報以上を漏らすことはなく、
高コストな検証 (リモートのサブキー解決等) を既知の Document に対して省略できる。
ただし、time-based CDID の内容バインディングは 80 ビットである (CIP-1 §8) ため、
重複ヒット時に保存済み Document が参照可能であればバイト列の一致を確認し、
不一致の場合は HTTP 400 で拒否するべきである (SHOULD)。

また、`kind: "entity"` のコミット、および `key` を持つ Document による既存キーの上書きは、
**accept-if-newer** 方式で処理されなければならない (MUST)。
サーバーは既存の Document と新規 Document の CDID (`createdAt` が先頭に符号化されるため
時刻順の比較が可能であり、同時刻は内容ハッシュによる決定的なタイブレークとなる) を比較し、
新しい場合のみ上書きする。古いまたは同一の Document の再送はエラーとせず、保存もコミットログの
記録も行わない no-op として成功を返す (MUST)。これにより、捕捉された旧バージョンの再送による
保存済み Document のロールバックは成立しない。

サーバー運用者が自身の管理経路 (マイグレーション・インポート等) において
本節の時刻境界を緩和することは妨げない。その経路・条件は実装定義であり、本仕様のスコープ外である
(CIP-1 §5.6、CIP-2 §6 参照)。

## 4. Security Considerations

サーバーは、Commit エンドポイントへのリクエストが適切に署名されていることを検証し、
署名者がリソースの変更を行う権限を持っていることを確認しなければならない (MUST)。

* 署名検証に失敗したリクエスト、または形式不正なリクエストに対しては、HTTP 400 Bad Request を返す。
  形式不正には、Document のサイズ上限 (CIP-1 §4.1) の超過、重複 JSON メンバ (CIP-1 §4)、
  `key` の制約 (CIP-0 §7.2) の違反、`associationVariant` の 512 バイト上限 (CIP-9 §3) の超過を含む。
* 署名は正当だが権限のないリクエストに対しては、HTTP 403 Forbidden を返す。

また、サーバーはコミット対象の owner がリクエスタ (Document の author) をブロックしている場合、
そのコミットを拒否してもよい (MAY)。参考実装は、対象 owner のブロックリストに author が含まれる場合に
コミットを拒否する。

## 5. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-0 – Concrnt Core (CCURI, 名前解決)
* CIP-1 – Concrnt Document System (Document, CDID, Signed Document)
