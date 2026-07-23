# CIP-3: Commit

## 0. Abstract
本ドキュメントでは、Concrntをホストするサーバーが提供するエンドポイントを拡張し、サーバーが管理するリソースを変更するための手段を提供する。

## 1. Status of This Memo

このドキュメントは Concrnt Document フォーマットの仕様を定義する。

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、
実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Commit エンドポイント
Concrnt サーバーは、HTTP POST リクエストを受け付けるエンドポイントを提供する。
これは、CIP-0で定義されるサービスディスカバリにおいて、"net.concrnt.core.commit" エンドポイント名で広告されなければなりません (MUST)。

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

クライアントは、Commitエンドポイントに対してCIP-1で定義されたConcrnt Signed DocumentをHTTP POSTリクエストのボディとして送信します。

クライアントは、コミット対象を管理するConcrntサーバーに対してリクエストを送信しなければならない (MUST)。
コミット対象を管理するサーバーは、Documentの内容から次のように導出される。

* `key` フィールドが存在する場合: `key` のCCURIの owner 部が示すEntityの所属サーバ
* `associate` フィールドが存在する場合 (CIP-9, CIP-10): `associate` のCCURIの owner 部が示すEntityの所属サーバ
* `kind` が `delete` の場合 (CIP-4): `value` の削除対象CCURIの owner 部が示すEntityの所属サーバ
* `kind` が `entity` の場合 (CIP-0): `value.domain` が示すサーバ (所属先サーバ)

owner 部から所属サーバへの解決は、CIP-0で定義される名前解決 (Entity documentの `domain`) によって行う。

サーバーがコミットに対して実行する処理は、Documentの種別と自身の管理範囲によって決まる。
1つのコミットが複数のサーバーにそれぞれ異なる処理を要求しうる点に注意すること。

* コミット対象のownerを管理していれば、対象本体の保存・削除を行う。
* `distributes` の配布先を管理していれば、Reference Documentの生成 (CIP-7 §4.1) や
  削除の伝搬の適用 (CIP-4 §6.1) を行う。
* Ack/unackは `author` と `associate` のownerのいずれか一方でも管理していれば受理する (CIP-10 §3)。
* 該当する処理を実行した場合、対応するイベント (CIP-11) を自身の購読者へ発行する。

自身の管理範囲に該当する処理が1つもないコミットについて、サーバーは
HTTP 421 Misdirected Request で拒否してもよく (MAY)、何も適用せず
no-opとして成功を返してもよい (MAY)。参考実装は後者 (スルー) を採用する。
何も適用しなかった場合、そのコミットをコミットログに記録してはならない (MUST NOT、§3.4)。

また、権威・認可の対象となるフィールド (`key`・`associate`・削除対象・`distributes` の各エントリ) の
owner部に、エイリアス形式 (`@<FQDN>`、CIP-0 §7.2) を使用してはならない (MUST NOT)。
サーバーはこれを含むコミットを拒否しなければならない (MUST)。
署名済みのルーティング識別子が可変のDNSに依存することを防ぐため、クライアントはエイリアスを
事前にCCIDへ解決 (CIP-0 §8.3) してからDocumentを作成すること。
Resolve経路 (CIP-0 §9.3) でのエイリアス利用はこの制限の対象外である。

### 3.2 Document の種別 (kind)

Commitエンドポイントは、CIP-1で定義される `kind` フィールドによってDocumentの処理を分岐する。
受け付ける `kind` は以下の6種であり、それ以外の値を持つDocumentは拒否しなければならない (MUST)。

| kind | 意味 | 定義 |
|---|---|---|
| `entity` | Entity document の登録・更新 | CIP-0 |
| `record` | 一般のDocumentの作成・更新 | CIP-1 |
| `association` | Documentへの関連付け | CIP-9 |
| `ack` | Entityの承認 | CIP-10 |
| `unack` | Entity承認の取消 | CIP-10 |
| `delete` | 要素の削除 | CIP-4 |

### 3.3 レスポンス形式

サーバーは、リクエストが成功した場合、HTTP 200 OK ステータスコードを返し、レスポンスボディに受理されたConcrnt Signed Documentを返却する。

このとき、サーバーは受理したリソースの所在を示す以下のフィールドをSigned Documentに付与して返却する。

```json
{
  "document": "<JSON string>",
  "proof": { ... },
  "cckv": "cckv://<owner>/<key>",
  "ccfs": "ccfs://<owner>/concrnt/<cdid>"
}
```

* `cckv`
  Documentが `key` を持つ場合、そのキーを示すCCURI。
* `ccfs`
  Documentの内容から生成されたCDID (CIP-1) を示すCCURI。

`cckv` / `ccfs` は署名対象外の情報フィールド (CIP-1 §7) であり、サーバーが署名済みバイト列から
**自ら導出した値**を設定する。リクエストにクライアントが付与した外側の `cckv` / `ccfs` は
無視され、サーバー導出値で上書きされる。クライアントは、外側のメタデータを暗号学的な根拠として
扱ってはならない (MUST NOT)。

導出されるccfs URIのowner部は、次のように決定される (MUST)。

* `key` を持つDocument (record): `key` のowner部 (CCIDに限らず、FQDN/CSID形式の場合もそのまま)。
* association: `associate` のowner部。
* ack / unack: `associate` が指す対象Entityの解決済みCCID。

すなわちccfs identityのownerは**名前空間の所有者**であり、Documentの `author` (署名者) ではない。
authorと名前空間所有者が異なるDocument (他者の名前空間への書き込み等) では両者が乖離する点に注意。

kindごとの返却内容は以下の通りである。

* `record`: 送信されたSigned Documentに `cckv` と `ccfs` を付与したもの。
* `association` / `ack` / `unack`: 送信されたSigned Documentに `ccfs` を付与したもの。
* `entity`: 送信されたSigned Documentそのもの。
* `delete`: 削除された対象のSigned Document。

### 3.4 冪等性とリプレイ防止

Commitエンドポイントは署名済みDocumentをそのまま受理するため、攻撃者が捕捉したDocumentの再送 (リプレイ) に対する防御を持たなければならない。サーバーは以下を実装しなければならない (MUST)。

* **未来時刻の拒否**: `createdAt` がサーバー時刻より一定以上未来であるDocumentを拒否する。参考実装では許容スキューは12時間である。遠い未来の `createdAt` を許すと、後述のaccept-if-newer比較においてそのDocumentが以後のすべての更新に優先してしまうためである。
* **過去時刻の拒否 (backdate window)**: `createdAt` が一定以上過去であるDocumentを拒否する。参考実装ではこの窓は7日間である。
* **コミットログによる重複排除**: サーバーは、適用したDocumentのdocument ID (CDID) を**コミットログ**として記録する。Commit処理の冒頭で、受信したDocumentのdocument IDがすでにコミットログに存在する場合には、Documentを再適用せず、no-opとして成功を返す。document IDはDocumentの内容全体から導出されるため、これにより同一Documentの再送 (クライアントのリトライ、連合経路の再配送、ダンプの再インポート) は安全な冪等操作となる。同時に、これがリプレイガードでもある: 削除済み・上書き済みのDocumentのdocument IDもコミットログに残り続けるため、捕捉されたDocumentを再送しても適用されない。削除を行った `kind: "delete"` のDocument自身も同様にコミットログに残るため、捕捉された削除コマンドをリプレイし、同じキー (または範囲) に後から作成されたDocumentを削除させることもできない。
* **コミットログの保持期間**: コミットログのエントリをGC等で破棄する場合、少なくとも「コミット処理時刻」と「対象Documentの `createdAt`」の**遅い方**にbackdate windowを加えた時刻までは保持しなければならない。これにより、捕捉されたDocumentは「まだコミットログに存在してno-opになる」か「すでにbackdate windowより古く拒否される」かのいずれかとなり、削除がリプレイに対して恒久的になる。(処理時刻のみを起点にすると、未来スキュー上限付近の `createdAt` を持つDocumentが、エントリ破棄の時点でまだbackdate windowの範囲内に残り、スキュー分の期間だけ再コミット可能になってしまう。)
* **適用されなかったコミットの非記録**: 後述のaccept-if-newer比較で古いと判定された場合など、実際には何も適用しなかったコミットについては、コミットログを記録してはならない (MUST NOT)。

なお、コミットログの重複チェックは署名検証に先立って行ってよい (MAY)。document IDはDocumentの
全バイト列から導出されるため、チェックの結果がリクエスタのすでに保持する情報以上を漏らすことはなく、
高コストな検証 (リモートのサブキー解決等) を既知のDocumentに対して省略できる。

また、`kind: "entity"` のコミット、および `key` を持つDocumentによる既存キーの上書きは、
**accept-if-newer** 方式で処理されなければならない (MUST)。
サーバーは既存のDocumentと新規Documentのdocument ID (CDID; `createdAt` が先頭に符号化されるため
時刻順の比較が可能であり、同時刻は内容ハッシュによる決定的なタイブレークとなる) を比較し、
新しい場合のみ上書きする。古いまたは同一のDocumentの再送はエラーとせず、保存もコミットログの
記録も行わないno-opとして成功を返す (MUST)。これにより、捕捉された旧バージョンの再送による
保存済みDocumentのロールバックは成立しない (上書きされた旧バージョンはコミットログにも
残っているため、通常は冒頭の重複排除の時点でno-opとなる)。

なお、サーバー運用者が自身の管理経路 (マイグレーション・インポート等) において
§3.4の時刻境界を緩和することは妨げない。その経路・条件は実装定義であり、本仕様のスコープ外である
(CIP-1 §5.6、CIP-2 §6 参照)。

## 4. セキュリティと認証

サーバーは、Commitエンドポイントへのリクエストが適切に署名されていることを検証し、署名者がリソースの変更を行う権限を持っていることを確認しなければならない (MUST)。

* 署名検証に失敗したリクエスト、または形式不正なリクエスト (Documentのサイズ上限 CIP-1 §4.1 の超過、`key` の1024バイト上限 CIP-0 §7.2 の超過、`associationVariant` の512バイト上限 CIP-9 §3 の超過を含む) に対しては、HTTP 400 Bad Request を返す。
* 署名は正当だが権限のないリクエストに対しては、HTTP 403 Forbidden を返す。

また、サーバーはコミット対象のownerがリクエスタ (Documentのauthor) をブロックしている場合、そのコミットを拒否してもよい (MAY)。参考実装は、対象ownerのブロックリストにauthorが含まれる場合にコミットを拒否する。

