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
Concrnt のアカウントを持つ必要もない。RP は応答を受け取る HTTPS の URL によって識別される (§5.1)。

### Authenticator

エンティティ本体の秘密鍵 (master key) を保持し、ユーザーの同意のもとで ID Token に署名する
Concrnt クライアントアプリケーション。SIOPv2 における Self-Issued OP に相当する。

### Authorization Request

RP が Authenticator に対して発行する認証の要求 (§5)。本 CIP では要求の内容はすべて
Authorization Request URI に載る (by-value)。

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
     (nonce を発行し、行として保持)
  2. Authorization Request URI を
     QR コード / リンクとして提示
                                    --->   3. URI を解析する (RP への取得は無い)
                                           4. RP の host を提示して同意画面を表示
                                           5. ユーザーが承認 (master key で署名)
                                    <---   6. ID Token を https://<host>/<path> へ送信
  7. nonce で要求を引き、ID Token を検証して
     セッションを確立
```

要求の内容はすべて URI に載る (§5)。Authenticator は同意の前に RP に接触せず、
RP は要求ごとのエンドポイントを持たない。

同一端末フロー (Authenticator と RP のブラウザが同じ端末にある場合) では、手順 2 の URI を
リンクとして開き、手順 6 の応答はブラウザへのリダイレクトで返す (§8.2)。
どちらのフローを用いるかは URI の `<type>` (§5.2) で決まり、RP のブラウザ側が選ぶ。
RP のバックエンドは応答方式を区別しない: 受け口の役割の違い (§8.1 は預かる、§8.2 は提示者を
ログインさせる) だけで処理が決まる。

## 5. Authorization Request

### 5.1 RP の識別子 (送り先 URL)

RP は、応答の送り先となる HTTPS の URL `https://<host>/<path>` によって識別される。
事前登録は存在せず、`client_id` のような別個の識別子も持たない。

* 送り先 URL は Authorization Request URI (§5.2) の `<host>/<path>` に `https://` を付けて復元したものである。
  RP ごとに 1 本であり、要求ごとに変わらない。
* Authenticator がユーザーに提示する RP の身元は、この URL の `<host>` である (§6)。
* ID Token の `aud` には送り先 URL の文字列をそのまま用いる (§7.2)。オリジン (`scheme://host`) ではない。

### 5.2 Authorization Request URI

RP は Authorization Request を次の URI として表現し、QR コードまたはリンクとして提示する。

```text
concrnt-auth://<type>/<host>/<path>?n=<nonce>[&s=<scope>]
```

* `<type>` (REQUIRED) — 応答の配送方法 (§8)。`post` または `fragment`。
  Authenticator はこれ以外の値を持つ URI を拒否しなければならない (MUST)。将来の CIP は
  新しい値をここに追加できる。同じ要求 (同じ `n`) から `post` 用と `fragment` 用の URI を両方作ってよい
  (RP のブラウザ側が QR コードには `post`、リンクには `fragment` を使う)。
* `<host>/<path>` (REQUIRED) — 応答の送り先 `https://<host>/<path>` から `https://` を除いたもの。
  構文上 https 以外は表現できない。`<host>` に userinfo を含んではならない (MUST NOT)。port は含んでよい。
  `<path>` は空であってはならない (MUST NOT)。クエリは本仕様のパラメータが占めるため、送り先 URL は
  クエリを持てない。
  ローカル開発のために、Authenticator は `<host>` が `localhost` または `127.0.0.1` のとき応答を http で
  送ってもよい (MAY)。その場合も `aud` は `https://` で始まる文字列である (§7.2)。
  ネイティブアプリの RP はカスタムスキームを送り先にできない。claimed HTTPS URL (Universal Links / App Links)
  で受け取る (§10.4)。
* `n` (REQUIRED) — nonce。RP が生成する推測困難な一回限りの値。16 バイト以上の乱数を base64url (パディング無し)
  にしたもの。ID Token の `nonce` クレームとして署名の内側に戻ってくるので、ID Token をこの要求に束縛する
  役割を持ち (§9、§10.5)、RP が応答をどの要求に対応付けるかの鍵でもある (§8)。
* `s` (OPTIONAL) — 追加で要求する情報。スペース区切り。本 CIP が定義する値は `profile` (§7.3) のみである。
  省略時は `profile` を指定したものとみなす。`s=` (空) は CCID のみを要求する。
  `openid` という語は使わない (応答は常に ID Token であり、儀式でしかない)。
  他の CIP が追加の値を定義できる (CIP-18 の `concrnt_service_key` 等)。Authenticator は未知の値を無視する (MUST)。

Authenticator は未知のクエリパラメータを無視しなければならない (MUST)。将来の拡張はパラメータの追加で行う。
フラグメントを持つ URI は拒否する。

要求の有効期限 (`exp`) は URI に載せない。要求の有効期間は RP の内部事情 (保持する行の TTL) であり、
Authenticator は関知しない。期限切れの要求への応答は RP が拒否する (§9 手順 2)。

```text
concrnt-auth://post/example.com/api/auth/concrnt/callback?n=atuQzACZAhdDWdWSKoLkIg
concrnt-auth://fragment/example.com/api/auth/concrnt/callback?n=atuQzACZAhdDWdWSKoLkIg
```

応答方式を URI 自体に載せるのは、Authenticator が URI の開き方 (アプリ内のスキャナーで読んだか、
OS から deep link として渡されたか) から応答方式を推定できないためである (§10.1)。

Authenticator は、`concrnt-auth` スキームに加えて、自身が claimed HTTPS URL
(Universal Links / App Links) を登録している場合、同じ `<type>/<host>/<path>` とクエリを持つ
その URL を同一の Authorization Request URI として受理してもよい (MAY)。

### 5.3 Authenticator による検証

Authenticator は Authorization Request URI を次のとおり検証しなければならない (MUST)。

1. §5.2 の構文に従い、`<type>` が既知の値であること。
2. `<host>` が userinfo を含まず、`<path>` が空でないこと。送り先 URL `https://<host>/<path>` を復元する。
3. `n` が存在し、空でないこと。

いずれかに失敗した場合、Authenticator は要求を拒否し、ユーザーにその旨を表示する。
失敗した要求に対して ID Token を発行してはならず (MUST NOT)、エラー応答 (§8.3) も送ってはならない (MUST NOT)。
検証されていない送り先への送信になるためである。

URI は誰でも作れる (RP のオリジンから取得したものではない) が、偽造された URI は RP にとって無害である (§10.2)。
Authenticator が RP に要求の正当性を問い合わせる必要はない。

## 6. 同意画面の要件

Authenticator は、ID Token に署名する前に、ユーザーに次を提示しなければならない (MUST)。

* 送り先 URL の `<host>` (§5.1)。RP の身元としてユーザーに見せるのはこれだけである。
  URI には RP の自己申告 (名前・ロゴ等) が載らず、Authenticator は `<host>` 以外の RP の身元を表示しない。
* 署名に用いるエンティティの CCID (複数アカウントを扱う Authenticator の場合は選択肢)。
* `s` に応じて RP へ渡される情報の種類 (§7.3)。

URI の `<type>` が `post` の場合、Authenticator はさらに次を、URI の開き方 (QR コードの
スキャン、deep link のいずれか) に関わらず満たさなければならない (MUST)。

* `n` から導出した `display_code` (付録 A) による照合。RP の画面には同じ値が表示されている。
  照合の方法は、値を表示して RP の画面と見比べるよう促すものでも、RP の画面に表示されている値を
  候補の中から選ばせ、正しい選択をもって承認とみなすものでもよい。選択式の場合、誤った選択は
  拒否として扱い、同じ要求で選び直させてはならない (MUST NOT)。
* これが**別の端末の画面へのログイン**であることが分かる提示。ユーザー自身がその QR コードやリンクを
  開いたのでなければ拒否できるようにする。

これらは §10.1 の攻撃に対する緩和策である。`fragment` の場合は応答が同じ端末のブラウザに戻り、
その結果は提示したブラウザ自身のログインにしかならない (§8.2) ため、これらの提示は要求しない。

Authenticator は、そのエンティティが当該 RP に対して初めてログインする場合、
その旨を明示的に警告するべきである (SHOULD)。RP の同一性は `aud` の文字列 (§7.2) で判定する。
これも §10.1 の攻撃に対する緩和策である。

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
  "aud": "https://example.com/api/auth/concrnt/callback",
  "nonce": "atuQzACZAhdDWdWSKoLkIg",
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
* `aud` (REQUIRED) — 送り先 URL (§5.1)。`"https://"` に URI の `<host>/<path>` をそのまま連結した文字列で、
  正規化してはならない (MUST NOT)。RP はバイト単位の一致を見る (§9 手順 3)。
  オリジン (`scheme://host`) ではないことに注意。攻撃者が `<path>` に RP オリジン上のオープンリダイレクタを
  指定した URI を作っても、`aud` が RP の本物の送り先 URL と一致しないので RP が弾ける (§10.3)。
  一般的な JWT ライブラリの `aud` 検証で自動的に照合されるため、RP 実装者が確認を忘れられない。
* `nonce` (REQUIRED) — Authorization Request URI の `n`。
* `iat` (REQUIRED) — 発行時刻。
* `exp` (REQUIRED) — 失効時刻。`iat + 300` でなければならない (MUST)。ID Token はログイン操作の証明であり短命である。
  RP は時計ずれの許容 (300 秒程度) を持つ (§9 手順 5)。
* `jti` (REQUIRED) — トークンの一意識別子。
* `concrnt_domain` (REQUIRED) — エンティティの所属サーバーの FQDN (CIP-0 §8 の Entity Document の
  `value.domain`)。Authenticator の自己申告であり、RP が Concrnt のリソースを解決する際の初期候補
  (CIP-0 §8.5 のリゾルバヒント) として用いてよいが (MAY)、信頼の根拠としてはならない (MUST NOT)。

`iat` / `exp` は JWT [RFC7519] の NumericDate (数値) である。**CIP-2 の auth token が文字列を
用いるのと異なる**ことに注意。本 CIP は OpenID Connect との互換性を優先する。

RP が認識しないクレームは無視しなければならない (MUST)。

### 7.3 `s` とプロフィールクレーム

`s` に `profile` が含まれる場合 (`s` を省略した場合を含む)、Authenticator は OpenID Connect Core §5.1 の
標準クレームのうち、次をエンティティのプロフィールから充填して含めるべきである (SHOULD)。
対応する情報がない場合は省略する。

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

`s=` (空) の場合、プロフィールクレームを含めてはならない (MUST NOT)。

本 CIP は上記以外のクレーム追加の枠組みを定義しない。

### 7.4 CIP-2 auth token との区別

ID Token と CIP-2 の auth token は同じ `alg` と署名方式を共有するが、次の点で相互に流用できない。

* ID Token の `aud` は RP の送り先 URL (`https://…`) であり、Concrnt サーバーの FQDN / CSID ではない。
* ID Token の `sub` は CCID であり、`"concrnt"` ではない。
* ID Token の `iat` / `exp` は数値であり、文字列ではない。

Concrnt サーバーは CIP-2 §5.2 の手順 3・4 により ID Token を拒否する。RP は `sub` が `iss` と等しい
CCID であることの確認 (§9 手順 4) により auth token を拒否する。

## 8. 応答の配送

応答方式は Authorization Request URI の `<type>` (§5.2) で決まる。両方式の応答は同じ送り先 URL
(§5.1) に同じパラメータを運ぶが、RP における意味が異なる: `post` で届いた応答は**要求を開始したブラウザのために
預かる**もの、`fragment` で届いた応答は**それを提示したクライアント自身をログインさせる**ものである。
RP はこの 2 つを別の受け口で扱い、互いに混ぜてはならない (MUST NOT)。この区別が §10.1 の攻撃に対する
構造的な防御であり、RP が応答方式を推定したり開始ブラウザとの一致を照合したりする必要を無くす。

送り先 URL は RP ごとに 1 本であり、要求ごとに変えない。RP はどの要求への応答かを `nonce` で引く:
`id_token` の payload の `nonce` クレーム (署名検証の前に読んでよい。検証は要求の行を引いた後に行う §9)、
または拒否応答 (§8.3) の `nonce` パラメータである。

### 8.1 post (cross-device)

`<type>` が `post` の場合、Authenticator は送り先 URL `https://<host>/<path>` に対して
`application/x-www-form-urlencoded` で HTTP POST を行う。

```text
POST /api/auth/concrnt/callback HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded

id_token=eyJhbGciOiJDT05DUk5UIi...
```

* `id_token` (REQUIRED) — §7。
* 他の CIP が応答パラメータを追加できる (CIP-18 の `service_key_certificate` 等)。RP は未知のパラメータを
  無視する (MUST)。

Authenticator はこの POST でリダイレクトに追従してはならない (MUST NOT)。
RP は受理した場合 HTTP 200 または 204 を返す。レスポンスボディは本仕様では定義しない。

RP は、この POST を受け取った時点で ID Token を検証 (§9) し、**要求を生成したブラウザセッション**のために
結果を預かる。RP のブラウザ側は、要求生成時に発行されたセッション識別子
(QR コードに含まれない秘密。以下 pollToken) を用いてポーリングまたはプッシュで結果を受け取る。
結果は pollToken のようなブラウザの秘密でのみ回収できなければならない (MUST)。
`nonce` など Authorization Request URI から導出できる値のみによってログイン結果を取得できる設計にしてはならない
(MUST NOT)。これらは QR コードを見た第三者にも既知である。
RP はこの POST の送信者に対してセッションを確立してはならない (MUST NOT)。送信者は Authenticator であり、
RP のブラウザではない。

### 8.2 fragment (same-device)

`<type>` が `fragment` の場合、Authenticator は応答パラメータを URL フラグメントに付与した送り先 URL を開く。

```text
https://example.com/api/auth/concrnt/callback#id_token=eyJhbGciOiJDT05DUk5UIi...
```

通常はブラウザが開く。RP がその URL を claimed HTTPS URL (Universal Links / App Links) として登録している
ネイティブアプリであれば、OS がその URL を RP のアプリに渡す。

RP (のフロントエンドまたはアプリ) はフラグメントの内容をバックエンドへ送り、バックエンドは `nonce` で要求を引いて
ID Token を検証 (§9) し、**それを提示したクライアント**に対してセッションを確立する。フラグメントは HTTP リクエストに
含まれないため、サーバーログや Referer に ID Token が漏れない。

この経路で届いた結果を、要求を開始したブラウザセッション (§8.1 の預かり先) に結び付けてはならない
(MUST NOT)。開始したブラウザと提示したクライアントが別であるのは、他人に開かせたリンクの結果であり
(§10.1)、その場合に起きるのは提示した本人が自分のアカウントでログインすることだけである。
提示したクライアントが要求を開始したブラウザと同じである必要はなく、RP はそれを照合しなくてよい。
スマートフォンのブラウザから RP を利用する場合、RP は cross-device ではなく本モードを
用いるべきである (SHOULD)。

### 8.3 エラー応答

ユーザーが要求を拒否した場合、Authenticator は `id_token` の代わりに次を同じ配送路 (同じ送り先 URL) で
返してもよい (MAY)。

* `error` — `"access_denied"`。
* `nonce` — 要求の `n`。RP がどの要求への拒否かを引くために必要である。

URI の検証失敗 (§5.3) の場合はエラー応答を返してはならない (MUST NOT)。

## 9. RP による ID Token の検証

RP は ID Token を次の手順で検証しなければならない (MUST)。

1. JWT として構文解析する。ヘッダーの `alg` が `"CONCRNT"`、`typ` が `"JWT"` であり、
   `kid` が存在しないことを確認する。
2. payload の `nonce` で、自身が発行した要求の行を引く。行が無い・期限切れ・使用済みなら拒否する。
   この時点では署名を検証していないので、payload の内容は行を引く目的以外に使わない。
3. `aud` が自身の送り先 URL (= 発行した `https://<host>/<path>`) と完全一致することを確認する。
4. `sub` が `iss` と等しく、CIP-0 §5.2 の CCID (小文字の Bech32、HRP `con`) であることを確認する。
5. `exp` が `iat + 300` であること、現在時刻が `exp` 以内であること (時計ずれの許容込み)、
   `iat` が妥当な範囲にあることを確認する。
6. `base64url(header) + "." + base64url(payload)` を Keccak256 でハッシュ化し、署名から ECRECOVER で
   公開鍵を復元、CIP-0 §5.2 の手順でアドレスを導出し、`sub` の CCID と一致することを確認する。
7. 要求の行を原子的に消費する (MUST)。`nonce` は一回限りであり、同じ要求から `post` と `fragment` の
   両方の URI を作った場合 (§5.2) も行は 1 つで、どちらの経路で受理しても以後はもう一方の経路でも受理しない。
   検証に失敗した応答は消費しない。

検証に成功した場合、RP は `sub` のエンティティとしてセッションを確立する。セッションを与える相手は
応答の経路で決まる: `post` では要求を開始したブラウザ (§8.1)、`fragment` では応答を提示した
クライアント (§8.2) である。
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

* 同意画面での送り先 `<host>` の表示 (§6、MUST)。
* `post` での `display_code` による照合と「別の端末の画面へのログインである」旨の提示 (§6、MUST)。
* 要求の短い有効期間 (RP が保持する行の TTL。数分程度を推奨)。
* 初めての RP へのログイン時の警告 (§6、SHOULD)。
* `post` のログイン結果の取得をブラウザセッションの秘密に結び付けること (§8.1、MUST)。
* `fragment` の応答は提示したクライアント自身のログインにしか使わず、開始したセッションに届けないこと (§8.2、MUST)。
  これにより、他人に `fragment` のリンクを開かせても、攻撃者には何も届かない。
* 同一端末の場合は `fragment` を用いること (§8.2、SHOULD)。

この攻撃はもともと正規の RP が発行した要求を使うものであり、要求を by-value にしたこと (§5) で攻撃面は増えない (§10.2)。

Authenticator が URI の開き方で応答方式を制限すること (例: deep link で届いた `post` の要求を
拒否する) は防御にならない。OS のカメラアプリで読み取った QR コードは deep link として Authenticator に
届き、攻撃者に踏まされたリンクと区別できないためである。応答方式は URI の `<type>` (§5.2) で
明示し、`post` であれば開き方に関わらず §6 の提示を行う。同様に、RP が応答の送信者や
開始ブラウザとの一致から応答方式を推定することも防御にならない。RP は受け口ごとに意味を固定する (§8)。

高い保証が必要な RP は、cross-device フローを用いず同一端末フローに限定するか、
追加の近接確認などを独自に実装するべきである。

### 10.2 偽造された Authorization Request URI

要求は by-value であり、誰でも任意の `<host>/<path>` と `n` を持つ URI を作れる。これは RP にとって無害である。
偽の `n` に対して発行された ID Token は正規の RP の送り先 URL に届くが、対応する要求の行が無いので
§9 手順 2 で捨てられる。その ID Token の `aud` はその RP であるから、他所へ転用もできない。
したがって要求の内容を RP のオリジンから取得する (request_uri) ことに意味は無く、Authenticator が
同意前に RP へ接触しないことで、Authenticator の IP アドレスや User-Agent が同意前に RP に渡ることも無くなる。

### 10.3 オープンリダイレクタ

攻撃者は `<path>` に RP オリジン上のオープンリダイレクタを指定した `fragment` の URI を作り、被害者に開かせることで、
`https://<host>/<redirector>#id_token=…` のフラグメントを自身のサイトへ運ばせうる。しかし ID Token の `aud` は
`https://<host>/<redirector>` であって RP の本物の送り先 URL ではないので、RP はこれを受理しない (§9 手順 3)。
横取りされた ID Token は他の RP でも使えない。

残る影響は、漏れた ID Token の中身 (CCID、`profile` を要求していればプロフィールクレーム) が攻撃者に読まれることである。
すなわち「リンクを開いて承認した人の CCID が攻撃者に分かる」という情報漏れだけが残る。`fragment` の同意画面にも
送り先 `<host>` は表示され (§6)、ユーザーは身に覚えのない RP への要求を拒否できる。

### 10.4 ネイティブアプリの RP

送り先は https に限られる (§5.2) ため、ネイティブアプリの RP はカスタム URL スキームを送り先にできない。
claimed HTTPS URL (iOS の Universal Links / Android の App Links) で受け取ることになり、OS がドメインの
所有を検証するので、RFC 8252 §8.1 が指摘するカスタムスキームの横取りは起きない。
アプリが未インストール等の理由でブラウザに落ちた場合も、届く先は RP の https ページでしかない。

### 10.5 ID Token の漏洩とリプレイ

ID Token は `aud`, `nonce`, `exp` によって、特定の RP の特定の要求に対してのみ有効である。
漏洩した ID Token を別の RP や別の要求に転用することはできない。一方、`fragment` の経路では
ID Token を提示したクライアントがログインする (§8.2) ため、有効期間内の ID Token はその要求に対する
bearer トークンである。RP は要求の行の一回限りの消費 (§9 手順 7) を省略してはならず (MUST NOT)、
ID Token の有効期間は `iat + 300` に固定されている (§7.2)。

### 10.6 プライバシー

CCID はグローバルに一意な識別子であり、複数の RP 間でユーザーを相関付けることができる。
本仕様は RP ごとの仮名識別子 (pairwise identifier) を提供しない。Authenticator は同意画面で
CCID が RP に渡されることを明示する (§6)。

### 10.7 `display_code` の導出可能性

`display_code` は `n` から誰でも導出できる (付録 A)。これは弱点ではない。`display_code` の役割は
「ユーザーが見ている RP の画面」と「Authenticator に届いた要求」が同じものであることを人間に確認させることであり、
攻撃者が被害者に提示した正規の要求に対して、RP の画面と Authenticator が同じ値を出すのは当然である。
旧来の request_uri 方式でも Request Object を GET すれば誰でも読める値であり、強さは同じである。

### 10.8 任意の https URL への POST

`post` では Authenticator が URI で指示された任意の https URL に POST を行う。リダイレクトに追従しないので (§8.1)、
影響は指示された URL への同一オリジンのフォーム POST 1 回に収まり、Authenticator を踏み台にした内部ネットワークの
探索 (SSRF) には使えない。Authenticator はプライベートアドレス空間への接続を拒否してもよい (MAY)。

## 11. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 7519 – JSON Web Token (JWT)
* RFC 8252 – OAuth 2.0 for Native Apps (claimed HTTPS URL とカスタムスキームの横取り)
* OpenID Connect Core 1.0 (§5.1 Standard Claims)
* Self-Issued OpenID Provider v2 (cross-device フロー、ID Token 検証手順の参考)
* draft-ietf-oauth-cross-device-security – Cross-Device Flows: Security Best Current Practice
* CIP-0 – Concrnt Core (CCID, Entity Document)
* CIP-1 – Concrnt Document System (§7.2 署名方式)
* CIP-2 – Auth (`CONCRNT` アルゴリズム)
* CIP-18 – Service Key Certificate (本 CIP の配送路を用いる鍵の委任)

## 付録 A. `display_code` の導出

`display_code` は Authorization Request URI の `n` の文字列 (URI に載った base64url の文字列そのまま。
デコードしたバイト列ではない) から、RP と Authenticator の双方が次の算法で導出する。
結果は 6 個の絵文字を連結した文字列である。

### A.1 算法

すべての演算は 32 ビット符号なし整数で行う (乗算は下位 32 ビットを取る)。

1. `seed = FNV1a32(UTF-8(n))`
   * FNV-1a (32 ビット): `hash = 0x811c9dc5`。各バイト `b` について `hash = (hash XOR b) * 0x01000193`。
2. `i = 0, 1, …, 5` について:
   * `x = seed + (i + 1) * 0x9e3779b9`
   * `x = mix(x)`
   * `emoji[i] = TABLE[x mod 128]`
3. `display_code = emoji[0] || emoji[1] || … || emoji[5]`

`mix` は次の 32 ビットミキサである。

```text
mix(x):
  x = x XOR (x >> 16)
  x = x * 0x7feb352d
  x = x XOR (x >> 15)
  x = x * 0x846ca68b
  x = x XOR (x >> 16)
  return x
```

例: `n = atuQzACZAhdDWdWSKoLkIg` のとき `seed = 743013609`、`display_code = 🚚🦜🐴🐬🦊🛶`。

### A.2 絵文字表

`TABLE` は次の 128 要素である (0 から 127)。一部の要素は複数のコードポイント (絵文字 + U+FE0F) から成る 1 文字である。
実装は表をこのとおりに、順序も含めて持たなければならない (MUST)。

| | | | | | | | |
|---|---|---|---|---|---|---|---|
| 0: 🐘 | 1: 🦒 | 2: 🐬 | 3: 🐢 | 4: 🐧 | 5: 🦉 | 6: 🐼 | 7: 🦄 |
| 8: 🦓 | 9: 🐪 | 10: 🦭 | 11: 🐟 | 12: 🐙 | 13: 🦜 | 14: 🦌 | 15: 🦙 |
| 16: 🍎 | 17: 🍊 | 18: 🍋 | 19: 🍇 | 20: 🍉 | 21: 🍓 | 22: 🍒 | 23: 🍍 |
| 24: 🥕 | 25: 🌽 | 26: 🍅 | 27: 🥦 | 28: 🥐 | 29: 🧀 | 30: 🍪 | 31: 🍜 |
| 32: 🌲 | 33: 🌵 | 34: 🌻 | 35: 🍀 | 36: 🌈 | 37: ☀️ | 38: 🌙 | 39: ⭐ |
| 40: ☁️ | 41: ⚡ | 42: ❄️ | 43: 🌊 | 44: 🔥 | 45: 🌋 | 46: ⛰️ | 47: 🍄 |
| 48: 🏠 | 49: 🏰 | 50: 🗼 | 51: ⛺ | 52: 🚗 | 53: 🚲 | 54: 🚂 | 55: 🚀 |
| 56: ⛵ | 57: 🚁 | 58: 🚜 | 59: 🚌 | 60: 🚚 | 61: 🚇 | 62: 🚕 | 63: 🛶 |
| 64: 🎈 | 65: 🎁 | 66: 🎸 | 67: 🎹 | 68: 🎯 | 69: 🎲 | 70: 🧩 | 71: 📚 |
| 72: ✏️ | 73: 🎨 | 74: 🎧 | 75: 🎬 | 76: 🎺 | 77: 🎻 | 78: 🎮 | 79: 🪁 |
| 80: 🐶 | 81: 🐱 | 82: 🐭 | 83: 🐹 | 84: 🐰 | 85: 🦊 | 86: 🐻 | 87: 🐨 |
| 88: 🐯 | 89: 🦁 | 90: 🐮 | 91: 🐷 | 92: 🐸 | 93: 🐵 | 94: 🐺 | 95: 🐴 |
| 96: 🔑 | 97: 🔔 | 98: 💎 | 99: 🧲 | 100: 🔭 | 101: ⏰ | 102: 💡 | 103: 📷 |
| 104: 🔒 | 105: 📌 | 106: 🪙 | 107: 💻 | 108: 🧰 | 109: 🧭 | 110: 🪄 | 111: 🧱 |
| 112: 🐔 | 113: 🐤 | 114: 🐦 | 115: 🦆 | 116: 🦅 | 117: 🐳 | 118: 🦈 | 119: 🐊 |
| 120: 🦖 | 121: 🦕 | 122: 🐲 | 123: 🦦 | 124: 🦥 | 125: 🦔 | 126: 🐿️ | 127: 🐐 |
