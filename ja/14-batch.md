# CIP-14: Batch

## 0. Abstract

本ドキュメントでは、複数の HTTP リクエストを 1 回の HTTP ラウンドトリップにまとめて送信するための
Batch エンドポイントを定義する。

## 1. Status of This Memo

このドキュメントは Batch エンドポイントの仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様であり、実装者およびプロトコル設計者を対象とする。

本 CIP はオプショナルな拡張であり、CIP-0 を前提とする。認証の扱いは CIP-2 に依存する。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Introduction

Concrnt のクライアントおよびサーバーは、多数の小さなリクエスト (たとえば CIP-8 の Chunkline
iterator の解決) を発行する場面がある。これらを個別の HTTP リクエストとして送信すると
ラウンドトリップ回数がボトルネックになるため、複数のリクエストを 1 つの HTTP リクエストに
集約する手段として Batch エンドポイントを定義する。

## 4. Batch エンドポイント

サーバーは、Batch エンドポイントを提供することができる (MAY)。
提供する場合、CIP-0 のサービスディスカバリにおいて `net.concrnt.core.batch` エンドポイント名で
広告しなければならない (MUST)。

クライアントは、このエンドポイント名が広告されていない場合、個別のリクエストに
フォールバックするべきである (SHOULD)。

## 5. リクエスト形式

クライアントは、Batch エンドポイントに対して HTTP POST リクエストを送信する。

リクエストボディは `multipart/mixed` 形式 [RFC2046] であり、`Content-Type` ヘッダに boundary
パラメータを含めなければならない (MUST)。boundary が欠落している、または `Content-Type` が
`multipart/mixed` でない場合、サーバーは HTTP 400 Bad Request を返さなければならない (MUST)。

各パートは以下のヘッダを持つ。

* `Content-Type: application/http` [RFC9112] — パート本文が HTTP リクエストのシリアライズ表現で
  あることを示す。この値以外の `Content-Type` を持つパートは無視しなければならない (MUST)
  (対応するレスポンスパートも生成しない)。
* `Content-ID` — レスポンスとの対応付けに使用される識別子。`application/http` の各パートに
  必須であり (MUST)、リクエスト内で一意でなければならない (MUST)。
  `Content-ID` が欠落または重複している場合、サーバーはいずれのパートも実行する前に、
  バッチ全体を HTTP 400 Bad Request で拒否しなければならない (MUST)。
  `Content-ID` の値は不透明なオクテット列として扱い、山括弧の付加・除去その他の正規化を
  行ってはならない (MUST NOT)。

パート本文には、実行したい HTTP リクエストを HTTP/1.1 のワイヤ形式 (リクエストライン、ヘッダ、
空行、ボディ) でシリアライズしたものを格納する。パート内の HTTP メッセージは次に従う。

* 行終端には CRLF を使用しなければならない (MUST)。サーバーは LF のみの行終端を
  受理してもよい (MAY)。
* ボディを持つリクエストは `Content-Length` ヘッダを含めなければならない (MUST)。
  サーバーは、`Content-Length` を持たないパートのボディを MIME 境界までの残余として
  解釈してはならない (MUST NOT)。
* `Host` ヘッダは任意であり、サーバーはその値を無視する。

リクエストラインのターゲットには、対象サーバーにおける実際のエンドポイントパス
(クエリ文字列を含む) を origin-form で指定する。
absolute-form (絶対 URI) のターゲットが指定された場合、サーバーはこれを origin-form へ変換して
処理しなければならない (MUST)。このとき、絶対 URI のホスト部が自サーバーの構成 (FQDN) と
一致する (大文字小文字は区別しない) ことを確認し、異なるホストを指すパートは
拒否しなければならない (MUST)。拒否は該当パートのエラーレスポンス
(参考実装は 421 Misdirected Request) として行い、他のパートの処理は継続する。
Batch エンドポイントを他ホストへのリクエストのプロキシとして使用することはできない (SSRF 防止)。

リクエスト例:

```
POST /api/v2/batch HTTP/1.1
Content-Type: multipart/mixed; boundary=batch_boundary

--batch_boundary
Content-Type: application/http
Content-ID: req-1

GET /api/v2/chunkline/itr/2939839?uri=cckv%3A%2F%2Fcon1alice%2Ftimeline HTTP/1.1

--batch_boundary
Content-Type: application/http
Content-ID: req-2

GET /api/v2/chunkline/itr/2939839?uri=cckv%3A%2F%2Fcon1bob%2Ftimeline HTTP/1.1

--batch_boundary--
```

## 6. レスポンス形式

サーバーは、バッチ全体に対して HTTP 200 を返し、レスポンスボディを `multipart/mixed` 形式で
構成する。レスポンスの boundary は、リクエストと同一のものを使用してよい (MAY)。

`application/http` パートの各リクエストに対して、対応するレスポンスパートを
1 つ返さなければならない (MUST)。各レスポンスパートは以下のヘッダを持つ。

* `Content-Type: application/http`
* `Content-ID` — 対応するリクエストパートのヘッダ値を、バイト単位でそのまま
  複写しなければならない (MUST)。

パート本文には、対応する HTTP レスポンスを HTTP/1.1 のワイヤ形式 (ステータスライン、ヘッダ、
空行、ボディ) でシリアライズしたものを格納する。ボディを持つレスポンスには `Content-Length` を
付与しなければならない (MUST)。

レスポンス例:

```
HTTP/1.1 200 OK
Content-Type: multipart/mixed; boundary=batch_boundary

--batch_boundary
Content-Type: application/http
Content-ID: req-1

HTTP/1.1 200 OK
Content-Type: text/plain; charset=UTF-8
Content-Length: 7

2939835
--batch_boundary
Content-Type: application/http
Content-ID: req-2

HTTP/1.1 404 Not Found
Content-Length: 0

--batch_boundary--
```

クライアントは `Content-ID` を用いてリクエストとレスポンスを対応付けなければならない (MUST)。
パートの返却順序に依存してはならない (MUST NOT)。

## 7. 処理セマンティクス

サーバーは、各パートのリクエストを、そのパスに対応する通常のエンドポイントへ個別にリクエスト
した場合と同等に処理しなければならない (MUST)。すなわち、Batch エンドポイントは汎用のリクエスト
集約機構であり、特定のエンドポイント専用ではない。GET 系の取得だけでなく、`POST /commit` (CIP-3)
のような副作用を伴うリクエストも実行できる。

パートは、multipart 中の出現順に**逐次**処理しなければならない (MUST)。
あるパートの処理は、それより前のパートの処理が完了した後の状態を観測する。

サーバーは、Batch エンドポイント自身を対象とするパートを拒否しなければならない (MUST)。
バッチのネストは認められない。

サーバーは、特定のリクエストパターン (例: CIP-8 の Chunkline iterator の解決) について、
複数パートをまとめて処理する最適化を行ってよい (MAY)。
その場合でも、各パートに対するレスポンスは個別処理した場合と同等でなければならない (MUST)。

あるパートの処理が失敗した場合でも、サーバーは他のパートの処理を継続し、
失敗したパートには対応する HTTP エラーレスポンス (パートとしては正常に構成されたもの) を
格納するべきである (SHOULD)。個別パートの失敗をバッチ全体の HTTP ステータスに
反映させてはならない (MUST NOT)。

## 8. Security Considerations

* パート内のリクエストは、Batch リクエスト自体 (外側の HTTP リクエスト) の認証コンテキストの
  もとで処理される。認証 (CIP-2) は外側のリクエストに対して一度だけ評価しなければならず (MUST)、
  パートごとに異なる認証を持つことはできない。サーバーは、パート内の資格情報ヘッダ
  (`Authorization` 等) を無視 (除去) しなければならない (MUST)。
  これにより、あるパートが後続パートの認証状態を変化させることはできない。
* サーバーは、1 リクエストあたりのパート数に上限を設けるべきである (SHOULD)。
* 副作用を伴うパート (コミット等) を含むバッチにおいても、各パートの独立性は保たれる。
  あるパートの失敗が他のパートの実行を妨げることはなく (§7)、バッチ全体のアトミック性は
  提供されない。アトミックな一括操作が必要な場合は、範囲削除 (CIP-4 §4) のような専用の手段を
  用いること。

## 9. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 2046 – MIME Part Two: Media Types (multipart/mixed)
* RFC 9112 – HTTP/1.1 (application/http)
