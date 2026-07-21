# CIP-14 Batch

## 0. Abstract

本ドキュメントでは、複数のHTTPリクエストを1回のHTTPラウンドトリップにまとめて送信するための Batch エンドポイントを定義する。

## 1. Status of This Memo

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、
実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 用語 (Terminology)

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Introduction

Concrntのクライアントおよびサーバーは、多数の小さなリクエスト (たとえばCIP-8のChunkline iterator の解決) を発行する場面がある。
これらを個別のHTTPリクエストとして送信するとラウンドトリップ回数がボトルネックになるため、
複数のリクエストを1つのHTTPリクエストに集約する手段として Batch エンドポイントを定義する。

## 4. Batch エンドポイント

サーバーは、Batch エンドポイントを提供することができる (MAY)。
提供する場合、CIP-0で定義されるサービスディスカバリにおいて、`net.concrnt.core.batch` エンドポイント名で広告しなければならない (MUST)。

クライアントは、このエンドポイント名が広告されていない場合、個別のリクエストにフォールバックするべきである (SHOULD)。

## 5. リクエスト形式

クライアントは、Batch エンドポイントに対して HTTP POST リクエストを送信する。

リクエストボディは `multipart/mixed` 形式 [RFC2046] であり、`Content-Type` ヘッダに boundary パラメータを含めなければならない (MUST)。
boundary が欠落している、または `Content-Type` が `multipart/mixed` でない場合、サーバーは 400 Bad Request を返す。

各パートは以下のヘッダを持つ。

* `Content-Type: application/http` [RFC9112] — パート本文がHTTPリクエストのシリアライズ表現であることを示す。
  この値以外の `Content-Type` を持つパートは無視される。
* `Content-ID` — レスポンスとの対応付けに使用される識別子。リクエスト内で一意でなければならない (MUST)。

パート本文には、実行したいHTTPリクエストを HTTP/1.1 のワイヤ形式 (リクエストライン、ヘッダ、空行、ボディ) でシリアライズしたものを格納する。
リクエストラインのパスには、対象サーバーにおける実際のエンドポイントパス (クエリ文字列を含む) を指定する。

リクエスト例:

```
POST /api/v2/batch HTTP/1.1
Content-Type: multipart/mixed; boundary=batch_boundary

--batch_boundary
Content-Type: application/http
Content-ID: req-1

GET /api/v2/chunkline/itr/5785524?uri=cckv%3A%2F%2Fcon1alice%2Ftimeline HTTP/1.1

--batch_boundary
Content-Type: application/http
Content-ID: req-2

GET /api/v2/chunkline/itr/5785524?uri=cckv%3A%2F%2Fcon1bob%2Ftimeline HTTP/1.1

--batch_boundary--
```

## 6. レスポンス形式

サーバーは、バッチ全体に対して HTTP 200 を返し、レスポンスボディを `multipart/mixed` 形式で構成する。
レスポンスの boundary は、リクエストと同一のものを使用してよい (MAY)。

`application/http` パートの各リクエストに対して、対応するレスポンスパートを1つ返さなければならない (MUST)。
各レスポンスパートは以下のヘッダを持つ。

* `Content-Type: application/http`
* `Content-ID` — 対応するリクエストパートと同一の値。

パート本文には、対応するHTTPレスポンスを HTTP/1.1 のワイヤ形式 (ステータスライン、ヘッダ、空行、ボディ) でシリアライズしたものを格納する。

レスポンス例:

```
HTTP/1.1 200 OK
Content-Type: multipart/mixed; boundary=batch_boundary

--batch_boundary
Content-Type: application/http
Content-ID: req-1

HTTP/1.1 200 OK
Content-Type: text/plain; charset=UTF-8

5785524
--batch_boundary
Content-Type: application/http
Content-ID: req-2

HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=UTF-8

no iterator for the given uri and chunk
--batch_boundary--
```

クライアントは `Content-ID` を用いてリクエストとレスポンスを対応付けなければならない (MUST)。
パートの返却順序に依存してはならない (MUST NOT)。

## 7. 処理セマンティクス

サーバーは、各パートのリクエストを、そのパスに対応する通常のエンドポイントへ個別にリクエストした場合と同等に処理しなければならない (MUST)。
すなわち、Batch エンドポイントは汎用のリクエスト集約機構であり、特定のエンドポイント専用ではない。
パートに含められるリクエストに制限はなく、GET系の取得だけでなく、`POST /commit` (CIP-3) のような
副作用を伴うリクエストも実行できる。

サーバーは、特定のリクエストパターン (例: CIP-8 の chunkline iterator の解決) について、
複数パートをまとめて処理する最適化を行ってよい (MAY)。
その場合でも、各パートに対するレスポンスは個別処理した場合と同等でなければならない (MUST)。

あるパートの処理が失敗した場合でも、サーバーは他のパートの処理を継続し、
失敗したパートには対応するHTTPエラーレスポンス (パートとしては正常に構成されたもの) を格納するべきである (SHOULD)。
個別パートの失敗はバッチ全体のHTTPステータスに影響させない。

## 8. Security Considerations

* パート内のリクエストは、Batch リクエスト自体の認証コンテキストのもとで処理される。パートごとに異なる認証を持つことはできない。
* サーバーは、1リクエストあたりのパート数に上限を設けるべきである (SHOULD)。
* 副作用を伴うパート (コミット等) を含むバッチにおいても、各パートの独立性は保たれる。
  あるパートの失敗が他のパートの実行を妨げることはなく (§7)、バッチ全体のアトミック性は提供されない。
  アトミックな一括操作が必要な場合は、範囲削除 (CIP-4 §4) のような専用の手段を用いること。

## 9. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 2046 – MIME Part Two: Media Types (multipart/mixed)
* RFC 9112 – HTTP/1.1 (application/http)
