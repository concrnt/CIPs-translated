# CIP-12 Policy

## 0. Abstract
本仕様は、Concrntのリソースに対するアクセス制御ポリシーを定義する手段を提供する。

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

## 3. Introduction

Concrntにおいて、タイムラインなどのフィードリソースは特定のサーバーに
閉じたものではなく、複数のサーバーのエンティティから書き込み・参照される。
そのため「誰がこのリソースに投稿できるか」「誰がこのリソースを読めるか」といった
アクセス制御は、サーバーローカルな設定ではなく、リソース自身に関連付けられた
宣言的なポリシーとして表現され、リソースをホストするどのサーバーでも同一の結論が
得られる必要がある。

CIP-12は、このためのポリシー言語を定義する。ポリシーは次の3つの要素から構成される。

1. **Policy Document** — ポリシー本体。名前付き・バージョン付きのJSONドキュメントであり、
   HTTPで解決可能なURLでホストされる。
2. **リソースへの関連付け** — CIP-1で定義されるConcrnt Documentの `policy` フィールドに、
   Policy DocumentへのURL参照(および評価パラメータ)を記述することで、
   そのDocumentが表すリソースにポリシーを適用する。
3. **評価** — サーバーは、リソースへの操作(アクション)が要求されたとき、
   対象リソースとその祖先、および配布先(distributes)のポリシーを重ね合わせた
   **ポリシースタック**を構築し、これを評価して操作の可否を決定する。

## 4. Policy Document

Policy Documentは次のJSON構造を持つ。

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
  対応するバージョンが存在しない場合、そのPolicy Documentは解決不能として扱う(§5.3.3)。

バージョンエントリの中身は次の2要素から構成される。

* `statements`
  Statement(§4.1)のフラットな配列。
* `defaults`
  アクション名をキーとし、Conclusion値(§6.1)を値とするマップ (OPTIONAL)。
  そのアクションについてどのStatementも結論を出さなかった場合のフォールバック値となる(§6.2)。

### 4.1. Statement

Statementは、特定のアクションに対する1つの判定規則である。

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
  このStatementが適用されるアクション名(§4.3)。要求されたアクションと完全一致した場合のみ評価される。
* `key`
  このStatementが適用されるリソースキー(CCURI)のパターン (OPTIONAL)。
  `*` を任意の文字列にマッチするワイルドカードとして使用できる。
  パターンは対象キー全体にマッチしなければならない(前方一致ではない)。
  リソースに関連付けられたポリシーが解決される際、`key` は特別な相対表記を持つ(§5.2)。
* `condition`
  条件式(§4.2)。評価結果が真である場合のみ、このStatementは `emit` の値を出力する。
* `emit`
  条件成立時に出力するConclusion値(§6.1)。`ok` / `ng` / `allow` / `deny` のいずれか。
* `reason`
  条件成立時に評価結果へ付加される説明文字列 (OPTIONAL)。
  アクセス拒否時のエラーメッセージ等に利用される。

条件式の評価がエラーとなった場合(未知の演算子、型不一致、ConcrntCallの失敗等)、
そのStatementは**何も出力しない**(エラーを伝播させない)。
結果として、どのStatementも出力しなかったアクションは `defaults` へフォールバックする。
これにより、ポリシー作成者は `defaults` の設定を通じて、評価不能時に
fail-open(許可側に倒す)とするか fail-closed(拒否側に倒す)とするかを選択できる(MUST)。

### 4.2. Rule

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
  定数値 (OPTIONAL)。§4.2.1参照。

未知の演算子を含む式の評価はエラーとなる(そのStatementは何も出力しない)。

#### 4.2.1. Const と糖衣構文

`op` が `"Const"` であるノードは、`const` フィールドの値をそのまま返す。

```json
{ "op": "Const", "const": "admin" }
```

また糖衣構文として、`Const` 以外の演算子ノードに `const` フィールドが指定された場合、
その値は `Const` ノードとして**引数リストの先頭に**挿入される。
次の2つの式は等価である。

```json
{ "op": "Load", "const": "params.role" }

{ "op": "Load", "args": [ { "op": "Const", "const": "params.role" } ] }
```

#### 4.2.2. 論理演算子

* `And`
  可変長引数。すべての引数はboolでなければならない(MUST)。すべて真なら真を返す。
  引数を順に評価し、偽が見つかった時点で偽を返す。
* `Or`
  可変長引数。すべての引数はboolでなければならない(MUST)。いずれかが真なら真を返す。
* `Not`
  引数1個。boolでなければならない(MUST)。論理否定を返す。

boolでない引数が現れた場合はエラーとなる。

#### 4.2.3. 比較演算子

* `Eq`
  引数2個。両者が等しければ真を返す。比較は言語の値等価性による
  (文字列・数値・boolの単純比較を想定。オブジェクトや配列の深い比較は保証されない)。
* `Contains`
  引数2個。第1引数は配列でなければならない(MUST)。
  第2引数が配列の要素として含まれていれば真を返す。
* `IsNotEmpty`
  引数1個。引数が `null` の場合は偽。配列・文字列・オブジェクトの場合、
  空でなければ真を返す。それ以外の型(数値等)はエラーとなる。

#### 4.2.4. データアクセス演算子

* `Load`
  引数1個(string)。評価コンテキスト(§4.2.6)をドット記法で解決し、その値を返す。
  キーが存在しない場合はエラーとなる。

  ```json
  { "op": "Load", "const": "requester.ccid" }
  ```

* `CCUriOwner`
  引数1個(string)。引数をCCURI(CIP-0)としてパースし、そのowner(CCID)を返す。
  パースに失敗した場合はエラーとなる。

#### 4.2.5. ConcrntCall

`ConcrntCall` は、ポリシー評価の過程でConcrntのAPIを呼び出し、
その結果を条件判定に利用するための演算子である。
「特定のエンティティをAckしている者のみ閲覧可」のような、
外部の状態に依存する条件を表現できる。

引数は次の形式の可変長リストであり、すべてstringでなければならない(MUST)。

```text
[ <resolver>, <api>, <key1>, <value1>, <key2>, <value2>, ... ]
```

* `resolver` (args[0])
  APIを解決する起点となるCCID等の識別子。この識別子の所属サーバーに対してAPIが呼び出される。
* `api` (args[1])
  呼び出すエンドポイント名(例: `"net.concrnt.core.acknowledges"`)。
* 以降の引数は、クエリパラメータのキーと値のペア。ペア数は偶数でなければならない(MUST)。

返り値はAPIのレスポンス(通常はドキュメントの配列)であり、
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

* 呼び出し可能なAPIは評価者(サーバー)が定めるallowlistに制限されなければならない(MUST)。
  allowlistにないAPIの呼び出しは、名前解決やネットワークI/Oを行う前に拒否される。
  リファレンス実装のallowlistは現在 `net.concrnt.core.acknowledges` のみである。
* 呼び出しが失敗した場合(ネットワークエラー、非許可API、呼び出し機構が利用できない
  評価コンテキスト等)、式の評価はエラーとなり、そのStatementは何も出力しない。
  §4.1のとおり `defaults` へフォールバックするため、制限系ポリシーは
  `defaults` に拒否側の値(`ng` 等)を設定することでfail-closedにできる。

allowlistの内容は実装定義である。allowlistに依存するポリシーは、評価者によって
結果が異なりうるため、ポリシー作成者は `defaults` によるフォールバック(fail-open/closed)を
必ず宣言するべきである (SHOULD)。

#### 4.2.6. 評価コンテキスト

`Load` が解決対象とする評価コンテキストは、次のルートを持つ。

| ルート | 内容 |
|---|---|
| `requester` | 操作を要求しているエンティティ。`ccid` / `alias` / `domain` / `tag` フィールドを持つ |
| `self` | 操作対象のDocument(CIP-1の構造。`kind` / `key` / `value` / `author` / `schema` / `createdAt` 等) |
| `params` | ポリシー参照時に指定されたパラメータ(§5.1) |
| `globals` | サーバーのグローバルパラメータ。`fqdn`(サーバーのFQDN)を含む |

**タグ文法**: `requester.tag` は、サーバーがエンティティに付与したタグの
カンマ区切り文字列である(例: `"_admin,role:moderator"`)。

* 各要素は `key` または `key:value` 形式である。`key` および `value` には
  カンマ(`,`)およびコロン(`:`)を含めてはならない (MUST NOT)。
* `_` で始まる `key` はサーバー実装のために予約される(例: `_admin`, `_blocked`, `_invite`)。
* `requester.tag` の評価結果は文字列であるため、`Contains`(配列用)では判定できない。
  現状、個別タグの有無を判定する標準演算子は定義されていない(将来の拡張とする)。
  文字列全体の `Eq` 比較は可能である。

#### 4.2.7. 廃止された演算子

旧版仕様に存在した以下の演算子は廃止された。使用してはならない (MUST NOT)。

`LoadParam`, `LoadDocument`, `LoadSelf`, `LoadResource`,
`DomainFQDN`, `DomainCSID`, `IsCCID`, `IsCSID`, `IsCKID`,
`IsRequesterLocalUser`, `IsRequesterRemoteUser`, `IsRequesterGuestUser`,
`RequesterHasTag`, `RequesterID`

これらの多くは `Load` で代替できる
(例: `RequesterID` → `Load("requester.ccid")`、`DomainFQDN` → `Load("globals.fqdn")`、
`LoadParam` → `Load("params.<key>")`)。

### 4.3. Actions

リファレンス実装は次のアクションを評価する。

| アクション名 | 評価タイミング |
|---|---|
| `record:create` | `kind: record` のDocumentが新規キーへcommitされるとき |
| `record:update` | 既存キーを持つDocumentが上書きcommitされるとき |
| `record:read` | Documentが読み取られるとき |
| `record:delete` | `kind: delete` によりDocumentが削除されるとき |
| `association:create` | `kind: association` のDocumentがcommitされるとき |
| `association:read` | Association Documentが読み取られるとき |
| `association:delete` | Association Documentが削除されるとき |

対象が `kind: association` である場合はassociation系のアクション名が、
それ以外はrecord系のアクション名が使用される。

アクション名の名前空間は拡張可能であり、上位プロトコルや拡張が
独自のアクション名を定義してもよい (MAY)。
評価時に、どのStatementの `action` にもマッチしないアクションは
`defaults` のみで判定される。

## 5. リソースへのポリシーの適用

### 5.1. policy フィールド

CIP-1で定義されるConcrnt Documentを拡張し、`policy` フィールドを定義する。

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
  ポリシー参照の配列。複数のポリシーを1つのリソースに適用できる。
* `entries[].url`
  Policy Document(§4)を解決可能なURL。
* `entries[].params`
  評価時に評価コンテキストの `params` として注入される値 (OPTIONAL)。
  同一のPolicy Documentをパラメータ違いで再利用するために使用する。
* `entries[].defaults`
  アクション名→Conclusionのマップ (OPTIONAL)。
  解決されたPolicy Document自身の `defaults` を上書きする。
  参照先ポリシーが解決できなかった場合のフォールバック値としても使用される(§5.3.3)。

### 5.2. key の相対解決

Policy Documentは特定のリソースから独立して記述されるため、Statementの `key` には
適用先リソースからの相対表記を使用できる。サーバーはポリシーを解決する際、
適用元リソースのURI(以下 `<source>`)を用いて `key` を次のように書き換えなければならない (MUST)。

| 記述 | 書き換え後 | 意味 |
|---|---|---|
| `"."` | `<source>` | 適用元リソース自身のみ |
| `""` または `"*"` | `<source>*` | 適用元リソース自身とその配下すべて |
| `"./*"` | `<source>/*` | 適用元リソースの配下のみ(自身を含まない) |

上記以外の値はそのまま使用される。

### 5.3. ポリシースタックの構築

サーバーは、あるリソースへの操作を評価する際、以下の**レイヤー**を順に積んだ
ポリシースタックを構築する。各レイヤーは0個以上の評価単位
(解決済みポリシー + params)の集合である。

1. **サーバーグローバルポリシー** — サーバー運営者が設定するポリシー。常に先頭レイヤーとなる。
2. **祖先のポリシー** — 対象リソースのキー階層(cckv URI のパス構造)の**ルート側から**
   対象リソースへ向かって、各階層のDocumentに関連付けられたポリシーを
   1階層1レイヤーとして順に積む(ルートが先、直近の親が最後)。
3. **仮想親(配布先)のポリシー** — 対象リソースが `distributes`(CIP-7)によって
   タイムライン等へ配布されている場合、その配布先リソースに関連付けられたポリシー。
   配布先を宣言しているリソースのレイヤーの直前に挿入される。
   これにより、タイムラインのポリシーがそこに配布される投稿を統制できる。
4. **対象リソース自身のポリシー** — 対象Documentの `policy` フィールドから解決されたもの。
   常に最後のレイヤーとなる。

すなわち評価順は **グローバル → ルート → … → 直近の親 → (仮想親 →) 対象リソース自身** である。
§6.3の評価規則と組み合わせると、強い判定(`allow`/`deny`)はグローバル側(外側)のレイヤーが優先され、
弱い判定(`ok`/`ng`)は対象リソース自身に近い(深い)レイヤーが優先される。

associationの評価では、association自身ではなく `associate` が指すリソースを起点として
スタックが構築される。

#### 5.3.0. 作成・更新時の自己ポリシーの扱い

提出者が制御するDocumentが自身の操作を承認することを防ぐため、以下を適用する。

* **作成 (`record:create`) の評価では、コミットされようとしているDocument自身の
  `policy` フィールドを評価レイヤーに含めてはならない (MUST NOT)。**
  新規Documentのポリシーは、保存された後の将来の操作に対してのみ効力を持つ。
* **更新 (`record:update`) の評価では、保存済みの既存Documentのポリシーを
  「対象リソース自身のポリシー」レイヤーとして使用しなければならない (MUST)。**
  置き換え候補として提出されたDocumentのポリシーを認可判定に使用してはならない (MUST NOT)。

いずれの場合も、提出されたDocumentは評価コンテキストの `self` (§4.2.6) としては参照される。

#### 5.3.1. 解決

各エントリの `url` からPolicy Documentを取得し、`versions["2025-12-23"]` を評価単位とする。
サーバーは解決結果をキャッシュしてよい (MAY)。

#### 5.3.2. defaults の上書き

エントリに `defaults` が指定されている場合、解決されたポリシーの `defaults` を
アクション単位で上書きする。これにより、ポリシーを参照する側のドキュメントが
フォールバック時の結論を決定できる。

#### 5.3.3. 解決失敗時の扱い

参照先ポリシーが解決できなかった場合(取得失敗、対応バージョン無し等)、
そのエントリは**エラー状態の評価単位**としてスタックに残されなければならない (MUST)。
エラー状態の評価単位は、評価時にエントリの `defaults` に宣言された結論
(宣言が無いアクションについては UNSET)として折りたたまれる。
解決失敗を「ポリシー無し」として黙って無視してはならない (MUST NOT)。

## 6. 評価方法

### 6.1. Conclusion

評価結果は次の5値をとる。

| 値 | 強度 | 意味 |
|---|---|---|
| `unset` | — | 判定なし |
| `ok` | 弱 | 弱い許可 |
| `ng` | 弱 | 弱い拒否 |
| `allow` | 強 | 強い許可 |
| `deny` | 強 | 強い拒否 |

2つのConclusionの合成(Or合成)は次の規則に従う。

1. 一方が `unset` なら他方を返す。
2. `deny` と `allow` が衝突した場合、**`unset`(相殺)** となる。
3. それ以外で `deny` を含めば `deny`、`allow` を含めば `allow`
   (強い判定は弱い判定に優先する)。
4. `ok` と `ng` が衝突した場合、`unset`(相殺)となる。
5. それ以外で `ok` を含めば `ok`、`ng` を含めば `ng`。

`deny` × `allow` は「拒否優先」ではなく相殺である点に注意。
同一レイヤー内で強い許可と強い拒否が対立した場合、そのレイヤーは判定を放棄し、
判断は後続のレイヤー(またはデフォルト)へ委ねられる。

### 6.2. 単一ポリシーの評価

あるアクション `action`・対象キー `key` について、1つのポリシーは次のように評価される。

1. `statements` から、`action` が一致し、かつ書き換え済み `key` パターン(§5.2)が
   対象キー全体にマッチするStatementを抽出する。
2. 各Statementの `condition` を評価する。評価がエラーとなったStatementはスキップする。
3. 結果が真となったStatementの `emit` を、Or合成(§6.1)で順に折りたたむ。
   このとき `reason` が指定されていれば評価理由として蓄積する。
4. 折りたたみ結果が `unset` の場合、`defaults[action]` が定義されていればその値とする。

### 6.3. スタックの評価

ポリシースタックは先頭レイヤーから順に評価される。

1. レイヤー内の各評価単位を §6.2 で評価する
   (エラー状態の評価単位は `defaults` の宣言値、無ければ `unset`)。
   評価単位に `params` が付与されている場合、評価コンテキストの `params` を
   その値で差し替えて評価する。
2. レイヤー内の結果をOr合成し、レイヤーの結論とする。
3. レイヤーの結論に応じて:
   * `deny` → 直ちに `deny` で評価終了(以降のレイヤーは評価しない)。
   * `allow` → 直ちに `allow` で評価終了。
   * `unset` → 次のレイヤーへ(暫定結論は変更しない)。
   * `ok` / `ng` → 暫定結論を更新し、次のレイヤーへ。
4. 全レイヤー評価後、暫定結論(初期値 `unset`)を最終結論とする。

強い判定(`allow`/`deny`)は先に評価されたレイヤーが優先され、
弱い判定(`ok`/`ng`)は**後に評価されたレイヤーが優先される**。

スタックはグローバル→ルート→…→対象リソース自身の順に構築される(§5.3)ため、
強い判定については外側(サーバー・祖先)がリソース自身に優先し、
弱い判定についてはリソース自身の宣言が祖先のデフォルト的な宣言を上書きする。

### 6.4. 最終判定

最終結論が `allow` または `ok` の場合のみ、操作は許可される。
`deny`、`ng`、および `unset`(どのポリシーも判定しなかった場合)は拒否である。
すなわち、評価系全体としてはデフォルト拒否であり、
通常運用ではサーバーグローバルポリシーが基本操作に `ok` を与えることが想定される。

拒否時のエラーには、評価過程で蓄積された `reason`(§4.1)を含めるべきである (SHOULD)。

## 7. 例

### 7.1. 特定ユーザーのみ書き込み可能なタイムライン

Policy Document(`https://policy.example.com/whitelist.json` でホスト):

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

タイムライン側のDocument:

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

### 7.2. Ackしている相手のみ閲覧可能なリソース (ConcrntCall)

リソースのownerが要求者をAck(CIP-10)している場合のみ読み取りを許可する。
呼び出しに失敗した場合は `defaults` により `ng`(拒否)へ倒れる(fail-closed)。

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

* ポリシーはHTTP経由で解決されるため、Policy DocumentのホスティングはTLSで保護
  されるべきである (SHOULD)。またサーバーは解決結果をキャッシュするため、
  ポリシー変更の反映にはキャッシュTTL分の遅延が生じうる。
* Policy DocumentのURL解決および `ConcrntCall` は、提出者が間接的に指定した宛先への
  サーバー側ネットワークI/O (SSRFの温床) となる。評価者は、プライベートアドレス・
  ループバックアドレス・リンクローカルアドレスへ解決される宛先へのリクエストを
  拒否するべきである (SHOULD)。また、解決にはタイムアウトを課し、結果をキャッシュ
  するべきである (SHOULD)。DNS再バインディングを考慮し、接続時の宛先IPに対して
  検査を行うことが望ましい。
* `ConcrntCall` は評価時にネットワークI/Oを発生させる。評価者はallowlistによる
  API制限(MUST)に加え、タイムアウトや呼び出し回数の制限を課すべきである (SHOULD)。
* ポリシー解決失敗時の挙動(§5.3.3)は、参照エントリの `defaults` に依存する。
  機密性が要求されるリソースでは、拒否側のdefaultsを必ず宣言し
  fail-closedとすべきである (SHOULD)。

## 9. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* CIP-0 – Concrnt Core (CCID, CCURI)
* CIP-1 – Concrnt Document System
* CIP-10 – Ack
