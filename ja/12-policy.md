# CIP-12: Policy

## 0. Abstract

本仕様は、Concrnt のリソースに対するアクセス制御ポリシーを定義する手段を提供する。

## 1. Status of This Memo

このドキュメントはポリシー言語と評価方法の仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0, CIP-1 を前提とする。CIP-3〜CIP-11 の各拡張は、
認可判定の手段として本 CIP を参照する。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Introduction

Concrnt において、タイムラインなどのフィードリソースは特定のサーバーに閉じたものではなく、
複数のサーバーのエンティティから書き込み・参照される。そのため「誰がこのリソースに投稿できるか」
「誰がこのリソースを読めるか」といったアクセス制御は、サーバーローカルな設定ではなく、
リソース自身に関連付けられた宣言的なポリシーとして表現され、リソースをホストするどのサーバーでも
同一の結論が得られる必要がある。

CIP-12 は、このためのポリシー言語を定義する。ポリシーは次の 3 つの要素から構成される。

1. **Policy Document** — ポリシー本体。名前付き・バージョン付きの JSON ドキュメントであり、
   HTTP で解決可能な URL でホストされる。
2. **リソースへの関連付け** — CIP-1 で定義される Concrnt Document の `policy` フィールドに、
   Policy Document への URL 参照 (および評価パラメータ) を記述することで、
   その Document が表すリソースにポリシーを適用する。
3. **評価** — サーバーは、リソースへの操作 (アクション) が要求されたとき、
   対象リソースとその祖先、および配布先 (distributes) のポリシーを重ね合わせた
   **ポリシースタック**を構築し、これを評価して操作の可否を決定する。

## 4. Policy Document

Policy Document は次の JSON 構造を持つ。

```json
{
  "name": "string",
  "description": "string (optional)",
  "versions": {
    "2025-12-23": {
      "statements": [
        (see 4.1 Statement)
      ],
      "defaults": {
        "<action-name>": "<conclusion>"
      }
    }
  }
}
```

* `name`
  ポリシー名。
* `description`
  ポリシーの説明 (OPTIONAL)。
* `versions`
  ポリシー言語のバージョンをキーとし、ポリシー本体を値とするマップ。
  現在の最新バージョンは `"2025-12-23"` であり、このドキュメントではこのバージョンを定義する。
  評価者は自身が対応するバージョンのエントリのみを使用する。
  対応するバージョンが存在しない場合、その Policy Document は解決不能として扱う (§5.3.4)。

バージョンエントリの中身は次の 2 要素から構成される。

* `statements`
  Statement (§4.1) のフラットな配列。
* `defaults`
  アクション名をキーとし、Conclusion 値 (§6.1) を値とするマップ (OPTIONAL)。
  そのアクションについてどの Statement も結論を出さなかった場合のフォールバック値となる (§6.2)。

### 4.1 Statement

Statement は、特定のアクションに対する 1 つの判定規則である。

```json
{
  "action": "record:create",
  "key": "*",
  "condition": {
    (see 4.2 Rule)
  },
  "emit": "allow",
  "reason": "string (optional)"
}
```

* `action`
  この Statement が適用されるアクション名 (§4.3)。要求されたアクションと完全一致した場合のみ
  評価される。
* `key`
  この Statement が適用されるリソースキー (CCURI) のパターン (OPTIONAL)。
  `*` を任意の文字列にマッチするワイルドカードとして使用できる。
  パターンは対象キー全体にマッチしなければならない (前方一致ではない)。
  リソースに関連付けられたポリシーが解決される際、`key` は特別な相対表記を持つ (§5.2)。
* `condition`
  条件式 (§4.2)。評価結果が真である場合のみ、この Statement は `emit` の値を出力する。
* `emit`
  条件成立時に出力する Conclusion 値 (§6.1)。`ok` / `ng` / `allow` / `deny` のいずれか。
* `reason`
  条件成立時に評価結果へ付加される説明文字列 (OPTIONAL)。
  アクセス拒否時のエラーメッセージ等に利用される。

条件式の評価がエラーとなった場合 (未知の演算子、型不一致、ConcrntCall の失敗等)、
その Statement は**何も出力してはならない** (MUST NOT。エラーを伝播させない)。
結果として、どの Statement も出力しなかったアクションは `defaults` へフォールバックする。
これにより、ポリシー作成者は `defaults` の設定を通じて、評価不能時に
fail-open (許可側に倒す) とするか fail-closed (拒否側に倒す) とするかを選択できる。

### 4.2 Rule

条件式は、演算子のツリーとして表現される。各ノードは次の構造を持つ。

```json
{
  "op": "<operator-name>",
  "args": [ <Rule>, ... ],
  "const": <any> (optional)
}
```

* `op`
  演算子名。
* `args`
  引数となる子ノードの配列。各子ノードを評価した結果が演算子への引数となる。
* `const`
  定数値 (OPTIONAL)。§4.2.1 参照。

引数の子ノードは、宣言順に**すべて評価**したうえで演算子に渡される (短絡評価を行わない)。
いずれかの子ノードの評価がエラーとなった場合、式全体の評価はエラーとなる (MUST)。
評価者間で「どの引数まで評価されたか」による結論の差を生まないためである。
未知の演算子を含む式の評価はエラーとなる (その Statement は何も出力しない)。

#### 4.2.1 Const と糖衣構文

`op` が `"Const"` であるノードは、`const` フィールドの値をそのまま返す。

```json
{ "op": "Const", "const": "admin" }
```

また糖衣構文として、`Const` 以外の演算子ノードに `const` フィールドが指定された場合、
その値は `Const` ノードとして**引数リストの先頭に**挿入される。
次の 2 つの式は等価である。

```json
{ "op": "Load", "const": "params.role" }

{ "op": "Load", "args": [ { "op": "Const", "const": "params.role" } ] }
```

#### 4.2.2 論理演算子

* `And`
  可変長引数。すべての引数は bool でなければならない (MUST)。すべて真なら真、
  さもなくば偽を返す。引数が 0 個の場合は真を返す (MUST)。
* `Or`
  可変長引数。すべての引数は bool でなければならない (MUST)。いずれかが真なら真、
  さもなくば偽を返す。引数が 0 個の場合は偽を返す (MUST)。
* `Not`
  引数 1 個。bool でなければならない (MUST)。論理否定を返す。

bool でない引数が現れた場合はエラーとなる。

#### 4.2.3 比較演算子

* `Eq`
  引数 2 個。両者が等しければ真を返す。比較は JSON データモデル上で行う (MUST):
  引数は string / number / boolean / null のいずれかであり、型が異なる場合は偽を返す。
  number は数値として (IEEE 754 double の等価性で) 比較する。
  配列またはオブジェクトが引数に現れた場合はエラーとなる (MUST)。
* `Contains`
  引数 2 個。第 1 引数は配列でなければならない (MUST)。
  第 2 引数が配列の要素として含まれていれば真を返す。要素の比較は `Eq` と同一の規則による (MUST)。
* `IsNotEmpty`
  引数 1 個。引数が `null` の場合は偽。配列・文字列・オブジェクトの場合、
  空でなければ真を返す。それ以外の型 (数値等) はエラーとなる。

#### 4.2.4 データアクセス演算子

* `Load`
  引数 1 個 (string)。評価コンテキスト (§4.2.6) をドット記法で解決し、その値を返す。
  キーが存在しない場合はエラーとなる。

  ```json
  { "op": "Load", "const": "requester.ccid" }
  ```

* `CCUriOwner`
  引数 1 個 (string)。引数を CCURI (CIP-0) としてパースし、その owner (CCID) を返す。
  パースに失敗した場合はエラーとなる。

#### 4.2.5 ConcrntCall

`ConcrntCall` は、ポリシー評価の過程で Concrnt の API を呼び出し、その結果を条件判定に利用する
ための演算子である。「特定のエンティティを Ack している者のみ閲覧可」のような、
外部の状態に依存する条件を表現できる。

引数は次の形式の可変長リストであり、すべて string でなければならない (MUST)。

```text
[ <resolver>, <api>, <key1>, <value1>, <key2>, <value2>, ... ]
```

* `resolver` (args[0])
  API を解決する起点となる CCID 等の識別子。この識別子の所属サーバーに対して API が呼び出される。
* `api` (args[1])
  呼び出すエンドポイント名 (例: `"net.concrnt.core.acknowledges"`)。
* 以降の引数は、クエリパラメータのキーと値のペア。ペア数は偶数でなければならない (MUST)。

返り値は API のレスポンスである。レスポンスが CIP-5 §3.2 のページング形式
(`{"items": [...], ...}`) の場合、評価器は `items` 配列を返り値としなければならない (MUST)。
`IsNotEmpty` と組み合わせて真偽値化するのが典型的な用法である。

```json
{
  "op": "IsNotEmpty",
  "args": [
    {
      "op": "ConcrntCall",
      "args": [
        { "op": "Const", "const": "con1alice" },
        { "op": "Const", "const": "net.concrnt.core.acknowledges" },
        { "op": "Const", "const": "from" },
        { "op": "Const", "const": "con1alice" },
        { "op": "Const", "const": "to" },
        { "op": "Load", "const": "requester.ccid" }
      ]
    }
  ]
}
```

評価規則:

* 呼び出し可能な API は評価者 (サーバー) が定める allowlist に制限されなければならない (MUST)。
  allowlist にない API の呼び出しは、名前解決やネットワーク I/O を行う前に拒否される。
  参考実装の allowlist は現在 `net.concrnt.core.acknowledges` のみである。
* HTTP 2xx 応答は、`items` が空配列であっても成功した (空の) 結果として扱う (MUST)。
  2xx 以外の応答は評価エラーとする (MUST)。
* 呼び出しが失敗した場合 (ネットワークエラー、非 2xx 応答、非許可 API、呼び出し機構が利用できない
  評価コンテキスト等)、式の評価はエラーとなり、その Statement は何も出力しない。
  §4.1 のとおり `defaults` へフォールバックするため、制限系ポリシーは
  `defaults` に拒否側の値 (`ng` 等) を設定することで fail-closed にできる。

allowlist の内容は実装定義である。allowlist に依存するポリシーは、評価者によって結果が異なりうる
ため、ポリシー作成者は `defaults` によるフォールバック (fail-open / closed) を
宣言するべきである (SHOULD)。

#### 4.2.6 評価コンテキスト

`Load` が解決対象とする評価コンテキストは、次のルートを持つ。

| ルート | 内容 |
|---|---|
| `requester` | 操作を要求しているエンティティ |
| `self` | 操作対象の Document (CIP-1 の構造。`kind` / `key` / `value` / `author` / `schema` / `createdAt` 等) |
| `params` | ポリシー参照時に指定されたパラメータ (§5.1) |
| `globals` | サーバーのグローバルパラメータ。`fqdn` (サーバーの FQDN) を含む |

`requester` のフィールドは次の通り導出される。

* `ccid` — CIP-2 で認証されたリクエスタの CCID。ゲスト (未認証) の場合は空文字列 `""`。
* `domain` — リクエスタの Entity Document から解決された所属サーバーの FQDN。ゲストの場合は空文字列。
* `alias` — リクエスタの Entity Document の alias。設定されている場合にのみ存在する。
  値は自己申告であり、CIP-0 §8.3 の検証を経ているかは受理サーバーの実装に依存するため、
  認可の条件として使用すべきではない (SHOULD NOT)。
* `tag` — サーバーがリクエスタに付与したタグ文字列 (後述)。設定されている場合にのみ存在する。

存在しないキーへの `Load` はエラーとなり (§4.2.4)、その Statement は `defaults` へ
フォールバックする。ゲストの判定には `requester.ccid` と空文字列の `Eq` 比較を用いる。

`self` は評価対象の操作により定まる。作成・更新時の扱いは §5.3.1 を参照。
削除 (`record:delete` / `association:delete`) の評価において、`self` は**削除対象として保存済みの
Document** でなければならない (MUST)。提出された `kind: "delete"` の Document を `self` として
用いてはならない (MUST NOT) — 削除コマンドの `author` は常にリクエスタ自身であるため、
これを許すと「author 本人のみ削除可」のポリシーが任意の第三者に成立してしまう。

**タグ文法**: `requester.tag` は、サーバーがエンティティに付与したタグの
カンマ区切り文字列である (例: `"_admin,role:moderator"`)。

* 各要素は `key` または `key:value` 形式である。`key` および `value` には
  カンマ (`,`) およびコロン (`:`) を含めてはならない (MUST NOT)。
* `_` で始まる `key` はサーバー実装のために予約される (例: `_admin`, `_blocked`, `_invite`)。
* `requester.tag` の評価結果は文字列であるため、`Contains` (配列用) では判定できない。
  現状、個別タグの有無を判定する標準演算子は定義されていない (将来の拡張とする)。
  文字列全体の `Eq` 比較は可能である。

### 4.3 Actions

本仕様は次のアクションを定義する。サーバーは、該当する操作の評価時にこれらのアクション名を
使用しなければならない (MUST)。

| アクション名 | 評価タイミング |
|---|---|
| `record:create` | `kind: record` の Document が新規キーへ commit されるとき |
| `record:update` | 既存キーを持つ Document が上書き commit されるとき |
| `record:read` | Document が読み取られるとき |
| `record:delete` | `kind: delete` により Document が削除されるとき |
| `association:create` | `kind: association` の Document が commit されるとき |
| `association:read` | Association Document が読み取られるとき |
| `association:delete` | Association Document が削除されるとき |

対象が `kind: association` である場合は association 系のアクション名が、
それ以外は record 系のアクション名が使用される。

アクション名の名前空間は拡張可能であり、上位プロトコルや拡張が独自のアクション名を
定義してもよい (MAY)。評価時に、どの Statement の `action` にもマッチしないアクションは
`defaults` のみで判定される。

## 5. リソースへのポリシーの適用

### 5.1 policy フィールド

CIP-1 で定義される Concrnt Document を拡張し、`policy` フィールドを定義する。

```json
{
  "kind": "record",
  "key": "cckv://con1alice/my-timeline",
  "schema": "https://...",
  "value": { ... },
  "author": "con1alice...",

  "policy": {
    "entries": [
      {
        "url": "https://policy.concrnt.world/p/whitelist-timeline.json",
        "params": { "allowlist": ["con1alice...", "con1bob..."] },
        "defaults": { "record:create": "ng" }
      }
    ]
  },

  "createdAt": "2025-11-23T12:34:56Z"
}
```

* `entries`
  ポリシー参照の配列。複数のポリシーを 1 つのリソースに適用できる。
* `entries[].url`
  Policy Document (§4) を解決可能な URL。
* `entries[].params`
  評価時に評価コンテキストの `params` として注入される値 (OPTIONAL)。
  同一の Policy Document をパラメータ違いで再利用するために使用する。
* `entries[].defaults`
  アクション名 → Conclusion のマップ (OPTIONAL)。
  解決された Policy Document 自身の `defaults` を上書きする。
  参照先ポリシーが解決できなかった場合のフォールバック値としても使用される (§5.3.4)。

### 5.2 key の相対解決

Policy Document は特定のリソースから独立して記述されるため、Statement の `key` には
適用先からの相対表記を使用できる。サーバーはポリシーを解決する際、
基準 URI (以下 `<source>`。通常はポリシーが関連付けられたリソースの URI。
仮想親レイヤーでは §5.3 の規則による) を用いて `key` を次のように
書き換えなければならない (MUST)。

| 記述 | 書き換え後 | 意味 |
|---|---|---|
| `"."` | `<source>` | 基準リソース自身のみ |
| `""` または `"*"` | `<source>*` | 基準リソース自身とその配下すべて |
| `"./*"` | `<source>/*` | 基準リソースの配下のみ (自身を含まない) |

上記以外の値はそのまま使用される。

### 5.3 ポリシースタックの構築

サーバーは、あるリソースへの操作を評価する際、以下の**レイヤー**を順に積んだポリシースタックを
構築する。各レイヤーは 0 個以上の評価単位 (解決済みポリシー + params) の順序付きリストである。

1. **サーバーグローバルポリシー** — サーバー運営者が設定するポリシー。常に先頭レイヤーとなる。
2. **祖先のポリシー** — 対象リソースのキー階層 (cckv URI のパス構造) の**ルート側から**
   対象リソースへ向かって、各階層の Document に関連付けられたポリシーを
   1 階層 1 レイヤーとして順に積む (ルートが先、直近の親が最後)。
3. **仮想親 (配布先) のポリシー** — ある階層の Document が `distributes` (CIP-7) によって
   タイムライン等へ配布されている場合、その配布先リソースに関連付けられたポリシー。
   配布先を宣言しているリソースのレイヤーの直前に、独立した 1 レイヤーとして挿入される。
   これにより、タイムラインのポリシーがそこに配布される投稿を統制できる。
   仮想親レイヤーの `key` 相対解決 (§5.2) では、`<source>` として配布先リソースの URI ではなく
   **`distributes` を宣言しているリソースの URI の親パス**を用いる (MUST)。
   これにより、配布先ポリシーの相対 `key` パターン (`"./*"` 等) が評価対象リソースのキーに
   マッチする。
4. **対象リソース自身のポリシー** — 対象 Document の `policy` フィールドから解決されたもの。
   常に最後のレイヤーとなる。

すなわち評価順は **グローバル → ルート → … → 直近の親 → (仮想親 →) 対象リソース自身** である。
§6.3 の評価規則と組み合わせると、強い判定 (`allow` / `deny`) はグローバル側 (外側) のレイヤーが
優先され、弱い判定 (`ok` / `ng`) は対象リソース自身に近い (深い) レイヤーが優先される。

association の評価では、association 自身ではなく `associate` が指すリソースを起点として
スタックが構築される。Association Document 自身の `policy` はスタックに積まれない
(評価コンテキストの `self` としてのみ参照される)。

#### 5.3.1 作成・更新時の自己ポリシーの扱い

提出者が制御する Document が自身の操作を承認することを防ぐため、以下を適用する。

* **作成 (`record:create`) の評価では、コミットされようとしている Document 自身の
  `policy` フィールドを評価レイヤーに含めてはならない (MUST NOT)。**
  新規 Document のポリシーは、保存された後の将来の操作に対してのみ効力を持つ。
* **更新 (`record:update`) の評価では、保存済みの既存 Document のポリシーを
  「対象リソース自身のポリシー」レイヤーとして使用しなければならない (MUST)。**
  置き換え候補として提出された Document のポリシーを認可判定に使用してはならない (MUST NOT)。

いずれの場合も、提出された Document は評価コンテキストの `self` (§4.2.6) としては参照される。

#### 5.3.2 解決

各エントリの `url` から Policy Document を取得し、評価者が対応するバージョンのエントリ (§4) を
評価単位とする。サーバーは解決結果をキャッシュしてよい (MAY)。

#### 5.3.3 defaults の上書き

エントリに `defaults` が指定されている場合、解決されたポリシーの `defaults` を
アクション単位で上書きする。これにより、ポリシーを参照する側のドキュメントが
フォールバック時の結論を決定できる。

#### 5.3.4 解決失敗時の扱い

参照先ポリシーが解決できなかった場合 (取得失敗、対応バージョン無し等)、
そのエントリは**エラー状態の評価単位**としてスタックに残されなければならない (MUST)。
エラー状態の評価単位は、評価時にエントリの `defaults` に宣言された結論
(宣言が無いアクションについては `unset`) として折りたたまれる。
解決失敗を「ポリシー無し」として黙って無視してはならない (MUST NOT)。

## 6. 評価方法

### 6.1 Conclusion

評価結果は次の 5 値をとる。

| 値 | 強度 | 意味 |
|---|---|---|
| `unset` | — | 判定なし |
| `ok` | 弱 | 弱い許可 |
| `ng` | 弱 | 弱い拒否 |
| `allow` | 強 | 強い許可 |
| `deny` | 強 | 強い拒否 |

2 つの Conclusion の合成は次の規則に従う。

1. 一方が `unset` なら他方を返す。
2. `deny` と `allow` が衝突した場合、**`unset` (相殺)** となる。
3. それ以外で `deny` を含めば `deny`、`allow` を含めば `allow`
   (強い判定は弱い判定に優先する)。
4. `ok` と `ng` が衝突した場合、`unset` (相殺) となる。
5. それ以外で `ok` を含めば `ok`、`ng` を含めば `ng`。

`deny` × `allow` は「拒否優先」ではなく相殺である点に注意。
同一レイヤー内で強い許可と強い拒否が対立した場合、そのレイヤーは判定を放棄し、
判断は後続のレイヤー (またはデフォルト) へ委ねられる。

この合成は可換でも結合的でもないため、複数の値の折りたたみは、
宣言順 (statements の配列順、`policy.entries` の配列順) の**左畳み込み**で
行わなければならない (MUST)。

### 6.2 単一ポリシーの評価

あるアクション `action`・対象キー `key` について、1 つのポリシーは次のように評価される。

1. `statements` から、`action` が一致し、かつ書き換え済み `key` パターン (§5.2) が
   対象キー全体にマッチする Statement を抽出する。
2. 各 Statement の `condition` を評価する。評価がエラーとなった Statement はスキップする。
3. 結果が真となった Statement の `emit` を、配列順の左畳み込み (§6.1) で合成する。
   このとき `reason` が指定されていれば評価理由として蓄積する。
4. 折りたたみ結果が `unset` の場合、`defaults[action]` が定義されていればその値とする。

### 6.3 スタックの評価

ポリシースタックは先頭レイヤーから順に評価される。

1. レイヤー内の各評価単位を §6.2 で評価する
   (エラー状態の評価単位は `defaults` の宣言値、無ければ `unset`)。
   評価単位に `params` が付与されている場合、評価コンテキストの `params` を
   その値で差し替えて評価する。
2. レイヤー内の結果を、評価単位の宣言順の左畳み込み (§6.1) で合成し、レイヤーの結論とする。
3. レイヤーの結論に応じて:
   * `deny` → 直ちに `deny` で評価終了 (以降のレイヤーは評価しない)。
   * `allow` → 直ちに `allow` で評価終了。
   * `unset` → 次のレイヤーへ (暫定結論は変更しない)。
   * `ok` / `ng` → 暫定結論を更新し、次のレイヤーへ。
4. 全レイヤー評価後、暫定結論 (初期値 `unset`) を最終結論とする。

強い判定 (`allow` / `deny`) は先に評価されたレイヤーが優先され、
弱い判定 (`ok` / `ng`) は**後に評価されたレイヤーが優先される**。

スタックはグローバル → ルート → … → 対象リソース自身の順に構築される (§5.3) ため、
強い判定については外側 (サーバー・祖先) がリソース自身に優先し、
弱い判定についてはリソース自身の宣言が祖先のデフォルト的な宣言を上書きする。

### 6.4 最終判定

最終結論が `allow` または `ok` の場合のみ、操作は許可される。
`deny`、`ng`、および `unset` (どのポリシーも判定しなかった場合) は拒否である。
すなわち、評価系全体としてはデフォルト拒否であり、
通常運用ではサーバーグローバルポリシーが基本操作に `ok` を与えることが想定される。

拒否時のエラーには、評価過程で蓄積された `reason` (§4.1) を含めるべきである (SHOULD)。

## 7. 例

### 7.1 特定ユーザーのみ書き込み可能なタイムライン

Policy Document (`https://policy.example.com/whitelist.json` でホスト):

```json
{
  "name": "whitelist-writers",
  "description": "params.allowlist に含まれるCCIDのみ書き込みを許可する",
  "versions": {
    "2025-12-23": {
      "statements": [
        {
          "action": "record:create",
          "key": "./*",
          "emit": "allow",
          "condition": {
            "op": "Contains",
            "args": [
              { "op": "Load", "const": "params.allowlist" },
              { "op": "Load", "const": "requester.ccid" }
            ]
          }
        },
        {
          "action": "record:create",
          "key": "./*",
          "emit": "deny",
          "reason": "this timeline is restricted to allowlisted members",
          "condition": {
            "op": "Not",
            "args": [
              {
                "op": "Contains",
                "args": [
                  { "op": "Load", "const": "params.allowlist" },
                  { "op": "Load", "const": "requester.ccid" }
                ]
              }
            ]
          }
        }
      ]
    }
  }
}
```

タイムライン側の Document:

```json
{
  "kind": "record",
  "key": "cckv://con1alice/members-timeline",
  "schema": "https://schema.concrnt.net/timeline.json",
  "value": { "name": "Members Only" },
  "author": "con1alice...",
  "policy": {
    "entries": [
      {
        "url": "https://policy.example.com/whitelist.json",
        "params": { "allowlist": ["con1alice...", "con1bob..."] }
      }
    ]
  },
  "createdAt": "2025-11-23T12:34:56Z"
}
```

### 7.2 Ack している相手のみ閲覧可能なリソース (ConcrntCall)

リソースの owner が要求者を Ack (CIP-10) している場合のみ読み取りを許可する。
呼び出しに失敗した場合は `defaults` により `ng` (拒否) へ倒れる (fail-closed)。

```json
{
  "name": "followers-only-read",
  "versions": {
    "2025-12-23": {
      "statements": [
        {
          "action": "record:read",
          "key": "*",
          "emit": "ok",
          "condition": {
            "op": "IsNotEmpty",
            "args": [
              {
                "op": "ConcrntCall",
                "args": [
                  { "op": "Const", "const": "con1alice" },
                  { "op": "Const", "const": "net.concrnt.core.acknowledges" },
                  { "op": "Const", "const": "from" },
                  { "op": "Const", "const": "con1alice" },
                  { "op": "Const", "const": "to" },
                  { "op": "Load", "const": "requester.ccid" }
                ]
              }
            ]
          }
        }
      ],
      "defaults": {
        "record:read": "ng"
      }
    }
  }
}
```

## 8. Security Considerations

* ポリシーは HTTP 経由で解決されるため、Policy Document のホスティングは TLS で保護
  されるべきである (SHOULD)。またサーバーは解決結果をキャッシュするため、
  ポリシー変更の反映にはキャッシュ TTL 分の遅延が生じうる。
* Policy Document の URL 解決および `ConcrntCall` は、提出者が間接的に指定した宛先への
  サーバー側ネットワーク I/O (SSRF の温床) となる。評価者は、プライベートアドレス・
  ループバックアドレス・リンクローカルアドレスへ解決される宛先へのリクエストを
  拒否するべきである (SHOULD)。また、解決にはタイムアウトを課し、結果をキャッシュ
  するべきである (SHOULD)。DNS 再バインディングを考慮し、接続時の宛先 IP に対して
  検査を行うことが望ましい。
* `ConcrntCall` は評価時にネットワーク I/O を発生させる。評価者は allowlist による
  API 制限 (MUST) に加え、タイムアウトや呼び出し回数の制限を課すべきである (SHOULD)。
* `ConcrntCall` の結果は、`resolver` の所属サーバーの応答をそのまま信頼する。
  `resolver` にリクエスタ由来の値 (`requester.ccid` 等) を用いると、認可の判断材料を
  リクエスタの支配下にあるサーバーが提供することになるため、用いるべきではない (SHOULD NOT)。
  評価者は、応答に含まれる Signed Document の署名を検証し、無効な要素を除外してもよい (MAY)。
* ポリシー解決失敗時の挙動 (§5.3.4) は、参照エントリの `defaults` に依存する。
  機密性が要求されるリソースでは、拒否側の defaults を必ず宣言し
  fail-closed とすべきである (SHOULD)。

## 9. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-0 – Concrnt Core (CCID, CCURI)
* CIP-1 – Concrnt Document System
* CIP-10 – Ack
