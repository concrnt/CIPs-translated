# CIP-8: Chunkline

## 0. Abstract

この仕様では、フィードをチャンクに分割し、効率的に配信・取得するための Chunkline フォーマットを定義する。
Chunkline は "Chunked Timeline" の略であり、大規模なタイムラインデータの配信に適した構造を提供する。

## 1. Status of This Memo

このドキュメントは Chunkline フォーマットの仕様を定義する。
Concrnt プロジェクトにより公開されるバージョン付き仕様ではあるが、CIP-8 はその適用範囲を
Concrnt プロジェクトのみに限定せず、広く一般に利用可能な仕様として提供されることを目的としている。
Concrnt サーバーへの適用は §8 で定義する。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Introduction

RSS や Atom のような従来のフィードフォーマットは、全体を一つのドキュメントとして扱うため、
大規模なタイムラインの配信や部分的な取得に非効率的である。
Chunkline フォーマットは、タイムラインを複数のチャンクに分割し、かつ間接参照を用いることで、
「ある時期からの最新の投稿のみを取得する」などの効率的なアクセスを可能にする。

また、これらは動的に生成されるほか、静的にホスティングされることも可能であり、
CDN を活用した配信も容易になる。

## 4. Chunkline Document

この仕様では、Chunkline Document とそれが指し示す Iterator Node、そして Body Node について説明する。

Chunkline Document は Chunkline の表現であり、フィードに関するメタデータと、関連付けられている
Iterator Node および Body Node への解決方法を提供する。
MIME タイプは `application/chunkline+json` である。

```json
{
    "version": "1.0",                        // required

    "chunk_size": 600,                       // required
    "first_chunk": 2939835,                  // required (null 許容)
    "last_chunk": null,                      // optional (null 許容)

    "ascending": {                           // optional
        "iterator": "/asc/itr/{chunk}",      // required
        "body": "/asc/body/{chunk}"          // required
    },

    "descending": {                          // optional
        "iterator": "/desc/itr/{chunk}",     // required
        "body": "/desc/body/{chunk}"         // required
    },

    "removed": "/removed",                   // optional

    "metadata": {                            // optional (null 許容)
        "title": "User's Timeline",          // optional
        "description": "A chunked timeline"  // optional
    }
}
```

* `version` (required)
  Chunkline フォーマットのバージョン。現行は `"1.0"`。

* `chunk_size` (required)
  チャンクの時間幅 (秒)。Chunk ID の算出に用いる (§5)。

* `first_chunk` (required, null 許容)
  このフィードに存在する最古のデータが属する Chunk ID。
  フィードにデータが 1 件も存在しない場合は `null` とする (MUST)。
  クライアントは `first_chunk` が `null` のマニフェストを「存在するが空のフィード」として
  扱わなければならない (MUST)。

* `last_chunk` (OPTIONAL, null 許容)
  このフィードに存在する最新のデータが属する Chunk ID。動的に更新されるフィードでは
  省略 (null) してよい (MAY)。

* `ascending` / `descending` (それぞれ OPTIONAL)
  それぞれ昇順・降順方向の Iterator Node / Body Node への URI テンプレート。
  少なくともどちらか一方を提供しなければならない (MUST)。
  テンプレート内のプレースホルダ `{chunk}` は、クライアントが Chunk ID に置換して解決する。
  テンプレートは相対参照または絶対 URL で表現でき、クエリ文字列を含んでもよい (MAY)。
  相対参照は、Chunkline Document の取得 URL を基底として RFC 3986 により
  解決しなければならない (MUST)。

  例 (Concrnt サーバーが生成するテンプレート):

  ```text
  /api/v2/chunkline/itr/{chunk}?uri=cckv%3A%2F%2Fcon1alice%2Fhome
  /api/v2/chunkline/body/{chunk}?uri=cckv%3A%2F%2Fcon1alice%2Fhome
  ```

* `removed` (OPTIONAL)
  このフィードから最近撤回されたエントリの識別子リストを返す Removed Node への URI テンプレート
  (§7)。プレースホルダを持たない。撤回を追跡しないサーバーは省略してよい (MAY)。

* `metadata` (OPTIONAL)
  フィードに関する任意のメタデータ。

### 4.1 Iterator Node

Iterator Node は、データの存在するチャンクへ直接到達するための間接参照であり、
方向ごとに次の意味を持つ (MUST)。

* **descending iterator**: 指定された Chunk ID **以前** (指定値を含む) のデータのうち、
  最新のデータが属するチャンクの Chunk ID を返す。
* **ascending iterator**: 指定された Chunk ID **以降** (指定値を含む) のデータのうち、
  最古のデータが属するチャンクの Chunk ID を返す。

これにより、クライアントは空のチャンクを一つずつ辿ることなく、データの存在するチャンクへ
直接到達できる。レスポンスは Chunk ID をプレーンテキストで返却する。

`/desc/itr/2939839` へのリクエストに対するレスポンス例:

```txt
2939835
```

(この例では、チャンク 2939839 から 2939836 までは空であり、直近のデータはチャンク 2939835 に属している。)

Iterator Node は、Chunkline Document の `first_chunk` から `last_chunk` (省略時は現在時刻に対応する
チャンク) までの範囲ですべてアクセス可能でなければならない (MUST)。

探索方向にデータが一切存在しない場合 (descending で指定 Chunk ID 以前、ascending で指定 Chunk ID
以降にデータがない場合、および `first_chunk` が `null` の場合)、サーバーは 404 Not Found を
返すべきである (SHOULD)。

### 4.2 Body Node

Body Node は、該当チャンクに含まれる実際の投稿データおよび投稿への参照を提供する。
レスポンスは、以下の構造を持つエントリの JSON 配列である。

`/desc/body/2939835` へのリクエストに対するレスポンス例:

```json
[
    {
        "timestamp": "2025-11-23T12:34:56Z",
        "href": "ccfs://con1alice/concrnt/9t4r7by29zwbr43c06dadzwz84",
        "content_type": "application/concrnt.document+json"
    },
    {
        "timestamp": "2025-11-23T12:30:00Z",
        "content": "Hello, world!"
    }
]
```

各エントリのフィールド:

* `timestamp` (required)
  エントリの時刻。RFC3339 形式。Chunk ID の算出基準となる。

* `content` (OPTIONAL)
  エントリの本文をインラインで表現する文字列。

* `href` (OPTIONAL)
  エントリの実体への参照 URI。

* `content_type` (OPTIONAL)
  `content` あるいは `href` の指す実体のメディアタイプ。

各エントリは `content` または `href` の少なくとも一方を含まなければならない (MUST)。

Body Node のレスポンスは、チャンクに含まれる投稿をすべて含む配列でなければならない (MUST)。
また、チャンクに含まれない投稿を含んでもよい (MAY)。
該当チャンクに投稿が存在しない場合は、404 ではなく空の JSON 配列を返す。
エントリは、ascending ノードであれば昇順、descending ノードであれば降順に
並べなければならない (MUST)。

## 5. Chunk ID の計算方法

Chunk ID は、投稿のタイムスタンプに基づいて計算される。
投稿の UNIX タイムスタンプ (秒) を `chunk_size` で割り、その商を負の無限大方向へ丸めた (floor)
整数を Chunk ID とする。

例えば `chunk_size` が 600 秒 (10 分) の場合、タイムスタンプ `2025-11-23T12:34:56Z` は
UNIX タイムスタンプ 1763901296 であり、600 で割ると 2939835.49... となる。
floor した 2939835 が Chunk ID となる。

逆に、Chunk ID からそのチャンクの開始時刻は `chunk_id * chunk_size` で得られる。

## 6. アイテムの同一性と重複排除

複数の Chunkline を横断して読み取る場合や、チャンク境界をまたいで走査する場合、
クライアントは同一エントリの重複を排除する必要がある。

エントリの識別子は次のように導出する。

1. `href` が存在する場合、`href` の値をそのまま識別子とする。
2. `href` が存在しない場合、`content` の UTF-8 バイト列の XXH3-64 ハッシュを用いて
   `urn:xxh3:<hex>` 形式の識別子を生成する。`<hex>` は 64 ビット値の小文字16進表現であり、
   先頭のゼロは含めない。

クライアントは、この識別子が一致するエントリを同一とみなし、重複を排除すべきである (SHOULD)。

## 7. アイテムの撤回 (Removed Items)

チャンクは静的にキャッシュ・配信されることを想定しているため、公開済みチャンクからエントリを
直接削除できない場合がある。そこで Chunkline は、フィードごとに「最近撤回されたエントリの
識別子リスト」を配信する **Removed Node** を定義する。

### 7.1 Removed Node

Removed Node は、Chunkline Document の `removed` フィールドに指定された URI テンプレートで
解決される。レスポンスは、このフィードから最近撤回されたエントリの識別子 (§6 の導出規則によるもの)
の JSON 配列である。

```json
["cckv://con1alice/posts/hello", "urn:xxh3:1a2b3c4d5e6f7890"]
```

* 識別子は、Body Node が同一エントリに対して返す識別子と一致しなければならない (MUST)。
* サーバーは、撤回されたエントリを、少なくとも読者が Body チャンクをキャッシュしうる期間以上
  リストに掲載し続けるべきである (SHOULD)。参考実装の掲載期間は 48 時間であり、
  Iterator / Body のキャッシュ TTL (2 日) と揃えている。
* 掲載期間を過ぎたエントリはリストから取り除いてよい (MAY)。Removed Node は撤回の恒久的な
  記録ではなく、キャッシュ更新のためのヒントである。

## 8. Concrnt における利用

Concrnt サーバーが Chunkline を提供する場合、以下のバインディングに従う。

* クライアントは、フィード (キー階層) を指す cckv URI の resolve (CIP-0 §9.3) に
  `Accept: application/chunkline+json` を指定することで、そのキー配下をフィードとする
  Chunkline Document を要求できる。本バインディングを提供するサーバーは、この要求に対して
  Chunkline Document を返さなければならない (MUST)。
* 自動生成 Reference (CIP-7 §4.1) に対応するエントリの `href` には、Reference 自身の URI ではなく
  参照先の URI (`value.href`) を用いる (MUST)。これにより、同一の Document が複数のフィードへ
  配布されている場合も、識別子 (§6) がフィード横断で一致する。

## 9. Security Considerations

* Body / Removed の識別子導出 (§6) はサーバーとクライアントで一致する必要がある。
  ハッシュのバリアント・エンコーディングを変更してはならない。
* 静的配信されるチャンクは削除の反映が遅延する。削除の秘匿性が重要な用途では、
  キャッシュ TTL と Removed Node の掲載期間を適切に設計すること (§7.1)。

## 10. References

* RFC 2119 – Key words for use in RFCs to Indicate Requirement Levels
* RFC 8174 – Clarifications to RFC 2119
* RFC 3339 – Date and Time on the Internet: Timestamps
* RFC 3986 – Uniform Resource Identifier (URI): Generic Syntax
* XXH3 – xxHash fast digest algorithm (https://xxhash.com)
