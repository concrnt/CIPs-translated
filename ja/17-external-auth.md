# CIP-17: External Service Authentication (Sign-In with Concrnt)

## 0. Abstract

本ドキュメントでは、Concrnt のエンティティ (CIP-0) が、Concrnt ではない外部のサービス
(以下 RP: Relying Party) に対して「自分がある CCID の所有者である」ことを証明し、ログインするための
プロトコルを定義する。

プロトコルは OpenID Connect の Self-Issued OpenID Provider v2 (SIOPv2) の cross-device /
same-device フローに準拠した構造を持ち、所有証明は CIP-2 と同じ `CONCRNT` アルゴリズムで署名された
JWT (ID Token) として表現される。RP は Concrnt サーバーに一切問い合わせることなく、
ID Token の署名からオフラインで CCID を検証できる。

本 CIP は **Concrnt サーバーの API に対するアクセス権を RP に与えるものではない**。
RP が得るのは「このリクエストの相手はこの CCID の所有者である」という事実のみである。

## 1. Status of This Memo

このドキュメントは外部サービス向け認証の仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0、CIP-1 (§7.2 の署名方式) および CIP-2 (§4 のトークン形式)
を前提とする。CIP-13 (Subkey) には依存しない。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

### RP (Relying Party)

エンティティの認証を要求する外部のサービス。Concrnt サーバーである必要はなく、
Concrnt のアカウントを持つ必要もない。RP は HTTPS でアクセス可能なオリジンによって識別される (§5.1)。

### Authenticator

エンティティ本体の秘密鍵 (master key) を保持し、ユーザーの同意のもとで ID Token に署名する
Concrnt クライアントアプリケーション。SIOPv2 における Self-Issued OP に相当する。

### Authorization Request

RP が Authenticator に対して発行する認証の要求 (§5)。

### ID Token

Authenticator がエンティティの秘密鍵で署名し、RP に返す JWT (§7)。

## 3. Introduction

Concrnt の auth token (CIP-2) は Concrnt サーバーに対する認証手段であり、`aud` は Concrnt サーバーの
FQDN または CSID に限られる。一方、Concrnt の外部にあるサービスが Concrnt のアイデンティティを
ログインに用いるには、次が必要である。

* RP を宛先とする所有証明の形式。
* RP が事前登録なしに Authenticator へ要求を届け、応答を受け取る配送路。
  特に、RP が PC のブラウザに表示され Authenticator がスマートフォン上にある cross-device の場合。
* Authenticator が RP の身元をユーザーに提示するための規則。

本 CIP はこれらを、既存の OpenID Connect の語彙と構造を可能な限り流用して定義する。
これにより、RP 側の実装は一般的な JWT ライブラリに `CONCRNT` アルゴリズム (CIP-2 §4.3) の検証を
追加するだけで済み、将来 Concrnt サーバーが OpenID Provider として振る舞う拡張を妨げない。

### 3.1 本 CIP が扱わないもの

* RP による Concrnt サーバー API へのアクセス。ID Token は CIP-2 の auth token ではなく、
  Concrnt サーバーはこれを受理しない (§7.4)。
* RP への鍵の委任。これは CIP-18 (Service Key Certificate) が本 CIP の配送路の上で定義する。
* OpenID Provider としての Concrnt サーバーの振る舞い。

## 4. プロトコルの概要

```text
  RP (ブラウザ + バックエンド)             Authenticator (スマートフォン等)
  ------------------------------           ---------------------------------
  1. Authorization Request を生成
  2. Authorization Request URI を
     QR コード / リンクとして提示
                                    --->   3. URI を読み取り request_uri から
                                              Request Object を取得
                                           4. RP のオリジンを検証し同意画面を表示
                                           5. ユーザーが承認 (master key で署名)
                                    <---   6. ID Token を redirect_uri へ送信
  7. ID Token を検証しセッションを確立
```

同一端末フロー (Authenticator と RP のブラウザが同じ端末にある場合) では、手順 2 の URI を
リンクとして開き、手順 6 の応答はブラウザへのリダイレクトで返す (§8.2)。
どちらのフローを用いるかは URI の `<response_mode>` (§5.2) で決まり、RP のブラウザ側が選ぶ。
RP のバックエンドは応答方式を区別しない: 同じ Request Object を返し (§5.3)、受け口の役割の違い
(§8.1 は預かる、§8.2 は提示者をログインさせる) だけで処理が決まる。

## 5. Authorization Request

### 5.1 RP の識別子 (オリジン)

RP は HTTPS のオリジンによって識別される。事前登録は存在せず、`client_id` のような別個の識別子も持たない。

* `request_uri` (§5.2) のオリジン (scheme, host, port) を **RP のオリジン**と呼ぶ。
  Authenticator がユーザーに提示する RP の身元は、このオリジンの host である (§6)。
* 応答の送り先 `redirect_uri` (§5.3) は RP のオリジンと同一オリジンでなければならない (MUST)。
* ID Token の `aud` には RP のオリジンを `scheme://host[:port]` の文字列 (末尾スラッシュ無し) で用いる (§7.2)。

### 5.2 Authorization Request URI

RP は Authorization Request を次の URI として表現し、QR コードまたはリンクとして提示する。

```text
concrnt-auth://<response_mode>/<host>/<path>[?query]
```

これは `request_uri = https://<host>/<path>[?query]` と同値である。

* `<response_mode>` (REQUIRED) — 応答の配送方法 (§8)。`direct_post` または `fragment`。
  Authenticator はこれ以外の値を持つ URI を拒否しなければならない (MUST)。将来の CIP は
  新しい値 (例: `signal`) をここに追加できる。応答方式が現れるのはこの URI だけであり、
  Request Object (§5.3) には含まれない。同じ `request_uri` から `direct_post` 用と `fragment` 用の
  URI を両方作ってよい (RP のブラウザ側が QR コードには `direct_post`、リンクには `fragment` を使う)。
* `<host>/<path>[?query]` (REQUIRED) — Request Object (§5.3) を取得する `request_uri` から `https://` を
  除いたもの。`<host>` は RP のオリジンの host (§5.1) であり、userinfo を含んではならない (MUST NOT)。
  `<path>` は空であってはならない (MUST NOT)。
  `request_uri` は推測困難な一意の値を含み、一回限りであるべきである (SHOULD)。

```text
concrnt-auth://direct_post/example.com/auth/concrnt/request/atuQzACZAhdDWdWSKoLkInh7injQzhX80bKtkkJjcQMfsefydw
concrnt-auth://fragment/example.com/auth/concrnt/request/atuQzAGZAhdDWdWSKoLkInh7injQ6udlSnGGls5kc82mnJ2jaA
```

Authorization Request URI には、`request_uri` の一部であるクエリを除き、クエリやフラグメントで
他のパラメータを足してはならない (MUST NOT)。要求の内容はすべて Request Object に置く。
これは QR コードを小さく保ち、要求内容を RP のオリジンから取得したものに限定するためである。

応答方式を URI 自体に載せるのは、Authenticator が URI の開き方 (アプリ内のスキャナーで読んだか、
OS から deep link として渡されたか) から応答方式を推定できないためである (§10.1)。

Authenticator は、`concrnt-auth` スキームに加えて、自身が claimed HTTPS URL
(Universal Links / App Links) を登録している場合、同じ `<response_mode>/<host>/<path>` を
パスに持つその URL を同一の Authorization Request URI として受理してもよい (MAY)。

### 5.3 Request Object

`request_uri` に対する HTTP GET は、`application/json` の Request Object を返す。

```json
{
  "redirect_uri": "https://example.com/auth/concrnt/callback/atuQzACZAhdDWdWSKoLkInh7injQzhX80bKtkkJjcQMfsefydw",
  "response_type": "id_token",
  "scope": "openid profile",
  "nonce": "n-0S6_WzA2Mj",
  "exp": 1763901596,
  "display_code": "🐧🌽🍊🧩☀️🪁",
  "client_metadata": {
    "client_name": "Example Service",
    "logo_uri": "https://example.com/logo.png",
    "policy_uri": "https://example.com/privacy",
    "tos_uri": "https://example.com/tos"
  }
}
```

* `redirect_uri` (REQUIRED) — 応答の送り先。`direct_post` では POST 先、`fragment` ではブラウザで開く URL (§8)。
  `https` スキームの絶対 URL でなければならず (MUST)、フラグメントを含んではならない (MUST NOT)。
  オリジンは `request_uri` のオリジン (= RP のオリジン、§5.1) と一致しなければならない (MUST)。
  要求ごとに異なる URL でよく (MAY)、RP は応答が届いた URL によってその応答をどの要求に対するものか
  対応付ける。応答パラメータには要求を識別する値を含めない (§8)。
* `response_type` (REQUIRED) — 常に `"id_token"`。
* `scope` (REQUIRED) — スペース区切り。`openid` を含まなければならない (MUST)。
  本 CIP は `openid` と `profile` を定義する (§7.3)。他の CIP が追加の scope を定義できる
  (CIP-18 の `concrnt_service_key` 等)。Authenticator は未知の scope を無視する (MUST)。
* `nonce` (REQUIRED) — RP が生成する一回限りのランダム値。推測困難でなければならない (MUST)。
  ID Token の `nonce` クレームとして署名の内側に戻ってくるので、ID Token をこの要求に束縛する役割を持つ (§9、§10.3)。
* `exp` (REQUIRED) — Request Object の失効時刻。UNIX タイムスタンプ (秒) の数値。
  Authenticator は失効した要求を拒否しなければならない (MUST)。
  RP は短い有効期間 (数分程度) を設定するべきである (SHOULD。§10.1)。
* `display_code` (REQUIRED) — RP が QR コードの傍らに表示する短い照合コード。Authenticator は
  `<response_mode>` が `direct_post` のとき同意画面に同じ値を表示する (§6)。`fragment` では応答が
  同じ端末のブラウザに戻り見比べる画面が無いので表示しないが、Request Object は応答方式に
  依存しないため常に含める。
* `client_metadata` (OPTIONAL) — RP の表示用メタデータ。メンバ名は OpenID Connect Dynamic Client
  Registration の Client Metadata に従う。本 CIP が参照するのは `client_name`, `logo_uri`,
  `policy_uri`, `tos_uri` である。

### 5.4 Authenticator による Request Object の取得と検証

Authenticator は Request Object を次の手順で取得・検証しなければならない (MUST)。

1. Authorization Request URI が §5.2 の構文に従い、`<response_mode>` が既知の値であることを確認する。
   `request_uri` を復元し、そのオリジンを RP のオリジンとする。
2. `request_uri` に HTTPS で GET を行う。リダイレクトに追従してはならない (MUST NOT)。
   TLS 検証を省略してはならない (MUST NOT)。
3. `redirect_uri` が `https` の絶対 URL であり、そのオリジンが RP のオリジンと一致することを確認する。
4. `response_type`, `scope`, `nonce`, `exp`, `display_code` の必須性と値を確認し、`exp` を過ぎていないことを
   確認する。`scope` に `openid` が含まれることを確認する。

いずれかに失敗した場合、Authenticator は要求を拒否し、ユーザーにその旨を表示する。
失敗した要求に対して ID Token を発行してはならない (MUST NOT)。

## 6. 同意画面の要件

Authenticator は、ID Token に署名する前に、ユーザーに次を提示しなければならない (MUST)。

* RP のオリジンの host (§5.1)。`client_metadata.client_name` を併せて表示してもよいが (MAY)、
  host の表示を `client_name` で置き換えてはならない (MUST NOT)。`client_name` は RP の自己申告であり、
  信頼の根拠にならない。
* 署名に用いるエンティティの CCID (複数アカウントを扱う Authenticator の場合は選択肢)。
* `scope` に応じて RP へ渡される情報の種類 (§7.3)。

URI の `<response_mode>` が `direct_post` の場合、Authenticator はさらに次を、URI の開き方 (QR コードの
スキャン、deep link のいずれか) に関わらず提示しなければならない (MUST)。

* `display_code` の値と、RP の画面に同じ値が表示されていることを確認するよう促す文言。
* これが**別の端末の画面へのログイン**であり、ユーザー自身がその QR コードやリンクを開いたのでなければ
  拒否するよう促す警告。

これらは §10.1 の攻撃に対する緩和策である。`fragment` の場合は応答が同じ端末のブラウザに戻り、
その結果は提示したブラウザ自身のログインにしかならない (§8.2) ため、これらの提示は要求しない
(`display_code` は Request Object に含まれるが表示しない)。

Authenticator は、そのエンティティが当該 RP のオリジンに対して初めてログインする場合、
その旨を明示的に警告するべきである (SHOULD)。これも §10.1 の攻撃に対する緩和策である。

ID Token への署名はエンティティ本体の秘密鍵で行う (§7)。Authenticator が master key の使用に
ユーザー認証 (生体認証・PIN 等) を要求する実装である場合、その認証をもって本節の承認操作と
してよい (MAY)。

## 7. ID Token

### 7.1 ヘッダー

```json
{
  "alg": "CONCRNT",
  "typ": "JWT"
}
```

* `alg` — 常に `"CONCRNT"` (MUST)。署名方式は CIP-2 §4.3 と同一である。
* `typ` — 常に `"JWT"` (MUST)。
* `kid` — 含めてはならない (MUST NOT)。ID Token は常にエンティティ本体の秘密鍵 (master key) で署名する。
  サブキー (CIP-13) で署名した ID Token を発行してはならず (MUST NOT)、RP は `kid` を持つ ID Token を
  拒否しなければならない (MUST)。

### 7.2 クレーム

```json
{
  "iss": "con1t0tey8uxhkqkd4wcp4hd4jedt7f0vfhk29xdd2",
  "sub": "con1t0tey8uxhkqkd4wcp4hd4jedt7f0vfhk29xdd2",
  "aud": "https://example.com",
  "nonce": "n-0S6_WzA2Mj",
  "iat": 1763901296,
  "exp": 1763901596,
  "jti": "7b5d9f2e-...",
  "concrnt_domain": "concrnt.example.net",
  "name": "Alice",
  "picture": "https://concrnt.example.net/..."
}
```

* `iss` (REQUIRED) — エンティティの CCID。
* `sub` (REQUIRED) — エンティティの CCID。`iss` と同一でなければならない (MUST)。
  CIP-2 の auth token は `sub` が `"concrnt"` 固定であるため、Concrnt サーバーは本 ID Token を
  auth token として受理しない (§7.4)。
* `aud` (REQUIRED) — RP のオリジン (§5.1)。`scheme://host[:port]` の形式で、末尾にスラッシュを付けない
  (WHATWG URL の origin のシリアライズと同じ)。
* `nonce` (REQUIRED) — Request Object の `nonce`。
* `iat` (REQUIRED) — 発行時刻。
* `exp` (REQUIRED) — 失効時刻。ID Token はログイン操作の証明であり、短命でなければならない。
  `iat` から 10 分以内とするべきである (SHOULD)。
* `jti` (REQUIRED) — トークンの一意識別子。
* `concrnt_domain` (REQUIRED) — エンティティの所属サーバーの FQDN (CIP-0 §8 の Entity Document の
  `value.domain`)。Authenticator の自己申告であり、RP が Concrnt のリソースを解決する際の初期候補
  (CIP-0 §8.5 のリゾルバヒント) として用いてよいが (MAY)、信頼の根拠としてはならない (MUST NOT)。

`iat` / `exp` は JWT [RFC7519] の NumericDate (数値) である。**CIP-2 の auth token が文字列を
用いるのと異なる**ことに注意。本 CIP は OpenID Connect との互換性を優先する。

RP が認識しないクレームは無視しなければならない (MUST)。

### 7.3 scope とプロフィールクレーム

`scope` に `profile` が含まれる場合、Authenticator は OpenID Connect Core §5.1 の標準クレームのうち、
次をエンティティのプロフィールから充填して含めるべきである (SHOULD)。対応する情報がない場合は省略する。

| クレーム | 内容 |
|---|---|
| `name` | 表示名 |
| `picture` | アバター画像の URL |
| `profile` | プロフィールページの URL |
| `preferred_username` | Entity Document の `value.alias` (CIP-0 §8.3)。設定されている場合のみ |
| `updated_at` | プロフィールの最終更新時刻 (NumericDate) |

これらはすべて Authenticator の自己申告であり、RP は表示用途にのみ用いるべきである (SHOULD)。
どのプロフィール (複数プロフィールを持つエンティティの場合) を用いるかは Authenticator の選択であり、
同意画面で選ばせてもよい (MAY)。

`scope` が `openid` のみの場合、プロフィールクレームを含めてはならない (MUST NOT)。

本 CIP は上記以外のクレーム追加の枠組みを定義しない。

### 7.4 CIP-2 auth token との区別

ID Token と CIP-2 の auth token は同じ `alg` と署名方式を共有するが、次の点で相互に流用できない。

* ID Token の `aud` は RP の HTTPS オリジン (`https://…`) であり、Concrnt サーバーの FQDN / CSID ではない。
* ID Token の `sub` は CCID であり、`"concrnt"` ではない。
* ID Token の `iat` / `exp` は数値であり、文字列ではない。

Concrnt サーバーは CIP-2 §5.2 の手順 3・4 により ID Token を拒否する。RP は `sub` が `iss` と等しい
CCID であることの確認 (§9 手順 4) により auth token を拒否する。

## 8. 応答の配送

応答方式は Authorization Request URI の `<response_mode>` (§5.2) で決まる。両方式の応答は同じ
パラメータを運ぶが、RP における意味が異なる: `direct_post` で届いた応答は**要求を開始したブラウザのために
預かる**もの、`fragment` で届いた応答は**それを提示したブラウザ自身をログインさせる**ものである。
RP はこの 2 つを別の受け口で扱い、互いに混ぜてはならない (MUST NOT)。この区別が §10.1 の攻撃に対する
構造的な防御であり、RP が応答方式を推定したり開始ブラウザとの一致を照合したりする必要を無くす。

### 8.1 direct_post (cross-device)

`<response_mode>` が `direct_post` の場合、Authenticator は Request Object の `redirect_uri` に対して
`application/x-www-form-urlencoded` で HTTP POST を行う。

```text
POST /auth/concrnt/callback/atuQzACZAhdDWdWSKoLkInh7injQzhX80bKtkkJjcQMfsefydw HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded

id_token=eyJhbGciOiJDT05DUk5UIi...
```

* `id_token` (REQUIRED) — §7。
* 他の CIP が応答パラメータを追加できる (CIP-18 の `service_key_certificate` 等)。RP は未知のパラメータを
  無視する (MUST)。

どの要求に対する応答かは、応答が届いた `redirect_uri` で決まる (§5.3)。応答パラメータに要求の識別子は無い。

Authenticator はこの POST でリダイレクトに追従してはならない (MUST NOT)。
RP は受理した場合 HTTP 200 または 204 を返す。レスポンスボディは本仕様では定義しない。

RP は、この POST を受け取った時点で ID Token を検証 (§9) し、**要求を生成したブラウザセッション**のために
結果を預かる。RP のブラウザ側は、要求生成時に発行されたセッション識別子
(QR コードに含まれない秘密) を用いてポーリングまたはプッシュで結果を受け取る。
`request_uri` や `nonce` など Authorization Request URI から導出できる値のみによって
ログイン結果を取得できる設計にしてはならない (MUST NOT)。これらは QR コードを見た第三者にも既知である。
RP はこの POST の送信者に対してセッションを確立してはならない (MUST NOT)。送信者は Authenticator であり、
RP のブラウザではない。

### 8.2 fragment (same-device)

`<response_mode>` が `fragment` の場合、Authenticator は応答パラメータを URL フラグメントに付与した
Request Object の `redirect_uri` をブラウザで開く。

```text
https://example.com/auth/concrnt/callback/atuQzACZAhdDWdWSKoLkInh7injQzhX80bKtkkJjcQMfsefydw#id_token=eyJhbGciOiJDT05DUk5UIi...
```

RP のフロントエンドは、開かれた URL (= `redirect_uri`。どの要求かを示す) とフラグメントの内容をバックエンドへ送り、
バックエンドは ID Token を検証 (§9) して**それを提示したブラウザ**に対してセッションを確立する。フラグメントは HTTP リクエストに含まれないため、
サーバーログや Referer に ID Token が漏れない。

この経路で届いた結果を、要求を開始したブラウザセッション (§8.1 の預かり先) に結び付けてはならない
(MUST NOT)。開始したブラウザと提示したブラウザが別であるのは、他人に開かせたリンクの結果であり
(§10.1)、その場合に起きるのは提示した本人が自分のアカウントでログインすることだけである。
提示したブラウザが要求を開始したブラウザと同じである必要はなく、RP はそれを照合しなくてよい。
スマートフォンのブラウザから RP を利用する場合、RP は cross-device ではなく本モードを
用いるべきである (SHOULD)。

### 8.3 エラー応答

ユーザーが要求を拒否した場合、Authenticator は `id_token` の代わりに次を同じ配送路 (同じ `redirect_uri`) で
返してもよい (MAY)。

* `error` — `"access_denied"`。

Request Object の検証失敗 (§5.4) の場合はエラー応答を返してはならない (MUST NOT)。
検証されていない `redirect_uri` への送信になるためである。

## 9. RP による ID Token の検証

RP は ID Token を次の手順で検証しなければならない (MUST)。

1. JWT として構文解析する。ヘッダーの `alg` が `"CONCRNT"`、`typ` が `"JWT"` であり、
   `kid` が存在しないことを確認する。
2. `aud` が自身のオリジン (§5.1) と一致することを確認する。
3. `nonce` が自身の発行した未使用の値と一致することを確認し、使用済みとして記録する (MUST)。
   記録は少なくとも対応する Request Object の `exp` まで保持する。同じ要求から `direct_post` と
   `fragment` の両方の URI を作った場合 (§5.2) も `nonce` は 1 つであり、どちらの経路で受理しても
   以後はもう一方の経路でも受理しない。
4. `sub` が `iss` と等しく、CIP-0 §5.2 の CCID (小文字の Bech32、HRP `con`) であることを確認する。
5. 現在時刻が `exp` を過ぎていないこと、`iat` が妥当な範囲にあることを確認する。
6. `base64url(header) + "." + base64url(payload)` を Keccak256 でハッシュ化し、署名から ECRECOVER で
   公開鍵を復元、CIP-0 §5.2 の手順でアドレスを導出し、`sub` の CCID と一致することを確認する。

検証に成功した場合、RP は `sub` のエンティティとしてセッションを確立する。セッションを与える相手は
応答の経路で決まる: `direct_post` では要求を開始したブラウザ (§8.1)、`fragment` では応答を提示した
ブラウザ (§8.2) である。
セッションは `sub` の CCID に結び付けなければならず (MUST)、プロフィールクレームのような
変化しうる値に結び付けてはならない (MUST NOT)。

手順 6 は Concrnt サーバーへの問い合わせを必要としない。RP は `concrnt_domain` を用いて
エンティティの Entity Document やプロフィールを取得してもよいが (MAY)、それは認証の成否に影響しない。

## 10. Security Considerations

### 10.1 cross-device における同意フィッシング

QR コードを介した cross-device フローは、攻撃者が正規の RP で生成した Authorization Request を
被害者に提示し、被害者の承認によって攻撃者のブラウザセッションがログインされる攻撃に
構造的に弱い。これは IETF の OAuth cross-device security BCP および SIOPv2 が指摘する既知の問題であり、
完全な防止策は存在しない。本仕様は次の緩和策を要求・推奨する。

* 同意画面での RP オリジンの表示 (§6、MUST)。
* `direct_post` での `display_code` の表示と「別の端末の画面へのログインである」旨の警告 (§6、MUST)。
* Request Object の短い有効期間 (§5.3、SHOULD)。
* 初めての RP へのログイン時の警告 (§6、SHOULD)。
* `direct_post` のログイン結果の取得をブラウザセッションの秘密に結び付けること (§8.1、MUST)。
* `fragment` の応答は提示したブラウザ自身のログインにしか使わず、開始したセッションに届けないこと (§8.2、MUST)。
  これにより、他人に `fragment` のリンクを開かせても、攻撃者には何も届かない。
* 同一端末の場合は `fragment` モードを用いること (§8.2、SHOULD)。

Authenticator が URI の開き方で応答方式を制限すること (例: deep link で届いた `direct_post` の要求を
拒否する) は防御にならない。OS のカメラアプリで読み取った QR コードは deep link として Authenticator に
届き、攻撃者に踏まされたリンクと区別できないためである。応答方式は URI の `<response_mode>` (§5.2) で
明示し、`direct_post` であれば開き方に関わらず §6 の提示を行う。同様に、RP が応答の送信者や
開始ブラウザとの一致から応答方式を推定することも防御にならない。RP は受け口ごとに意味を固定する (§8)。

高い保証が必要な RP は、cross-device フローを用いず同一端末フローに限定するか、
追加の近接確認などを独自に実装するべきである。

### 10.2 `request_uri` の取得

Authenticator は RP から指定された URL へリクエストを行う。SSRF を防ぐため、`https` スキーム以外、
およびリダイレクトを拒否する (§5.4)。Authenticator はプライベートアドレス空間への接続を
拒否してもよい (MAY)。

### 10.3 ID Token の漏洩とリプレイ

ID Token は `aud`, `nonce`, `exp` によって、特定の RP の特定の要求に対してのみ有効である。
漏洩した ID Token を別の RP や別の要求に転用することはできない。一方、`fragment` の経路では
ID Token を提示したブラウザがログインする (§8.2) ため、有効期間内の ID Token はその要求に対する
bearer トークンである。RP は `nonce` の一回限りの検証 (§9 手順 3) を省略してはならず (MUST NOT)、
Authenticator は ID Token の有効期間を短く保つべきである (SHOULD。§7.2)。

### 10.4 プライバシー

CCID はグローバルに一意な識別子であり、複数の RP 間でユーザーを相関付けることができる。
本仕様は RP ごとの仮名識別子 (pairwise identifier) を提供しない。Authenticator は同意画面で
CCID が RP に渡されることを明示する (§6)。

### 10.5 `client_metadata` の信頼

`client_name`, `logo_uri` 等は RP の自己申告である。Authenticator はこれらを RP のオリジン host と
混同されない形で表示しなければならない (MUST)。`logo_uri` の画像は RP のオリジンから
のみ取得するべきである (SHOULD)。

## 11. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 7519 – JSON Web Token (JWT)
* OpenID Connect Core 1.0 (§5.1 Standard Claims)
* OpenID Connect Dynamic Client Registration 1.0 (Client Metadata)
* Self-Issued OpenID Provider v2 (cross-device フロー、`direct_post`、ID Token 検証手順の参考)
* draft-ietf-oauth-cross-device-security – Cross-Device Flows: Security Best Current Practice
* CIP-0 – Concrnt Core (CCID, Entity Document)
* CIP-1 – Concrnt Document System (§7.2 署名方式)
* CIP-2 – Auth (`CONCRNT` アルゴリズム)
* CIP-18 – Service Key Certificate (本 CIP の配送路を用いる鍵の委任)
