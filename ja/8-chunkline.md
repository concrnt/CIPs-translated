# CIP-8 Chunkline

## 0. Abstract
この仕様では、フィードをチャンクに分割し、効率的に配信・取得するための Chunkline フォーマットを定義する。
Chunklineは "Chunked Timeline" の略であり、大規模なタイムラインデータの配信に適した構造を提供する。

## 1. Status of This Memo

このドキュメントは、Concrnt プロジェクトにより公開されるバージョン付き仕様ではあるが、
CIP-8はその適用範囲をConcrntプロジェクトのみに限定せず、広く一般に利用可能な仕様として提供されることを目的としている。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL


## 3. Introduction
RSSやAtomのような従来のフィードフォーマットは、全体を一つのドキュメントとして扱うため、大規模なタイムラインの配信や部分的な取得に非効率的である。
Chunkline フォーマットは、タイムラインを複数のチャンクに分割し、かつ間接参照を用いることで、「ある時期からの最新の投稿のみを取得する」などの効率的なアクセスを可能にする。

また、これらは動的に生成される他、静的にホスティングされることも可能であり、CDNを活用した配信も容易になる。

## 4. Chunkline Document

この仕様では、Chunkline Document とそれが指し示す Iterator Node、そして Body Node について説明する。

Chunkline Document は、Chunkline の表現であり、フィードに関するメタデータとそれに関連付けられている Iterator Node 及び Body Node への解決方法を提供する。

Chunkline Document は、以下のような JSON 構造で表現される。また、MIME タイプは `application/chunkline+json` である。

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

* `version`
  Chunkline フォーマットのバージョン。現行は `"1.0"`。

* `chunk_size`
  チャンクの時間幅（秒）。Chunk ID の算出に用いる（5章参照）。

* `first_chunk`
  このフィードに存在する最古のデータが属する Chunk ID。
  フィードにデータが1件も存在しない場合は `null` とする (MUST)。
  クライアントは `first_chunk` が `null` のマニフェストを「存在するが空のフィード」として
  扱わなければならない (MUST)。この場合、Iterator Node へのアクセスは 404 を返す。

* `last_chunk`
  このフィードに存在する最新のデータが属する Chunk ID。動的に更新されるフィードでは省略（null）してよい (MAY)。

* `ascending` / `descending`
  それぞれ昇順・降順方向の Iterator Node / Body Node への URI テンプレート。少なくともどちらか一方を提供しなければならない (MUST)。
  テンプレート内のプレースホルダ `{chunk}` は、クライアントが Chunk ID に置換して解決する。
  テンプレートは、Chunkline Document を取得したサーバーからの相対パス、または絶対 URL で表現できる。
  テンプレートはクエリ文字列を含んでもよい (MAY)。

  例（Concrnt サーバーが生成するテンプレート）:

  ```text
  /api/v2/chunkline/itr/{chunk}?uri=cckv%3A%2F%2Fcon1alice%2Fhome
  /api/v2/chunkline/body/{chunk}?uri=cckv%3A%2F%2Fcon1alice%2Fhome
  ```

* `removed`
  このフィードから最近撤回されたエントリの識別子リストを返す Removed Node への URI テンプレート（7章参照）。
  プレースホルダを持たない。撤回を追跡しないサーバーは省略してよい (MAY)。

* `metadata`
  フィードに関する任意のメタデータ。

### 4.1. Iterator Node

Iterator Node は、指定された Chunk ID **以前**のデータのうち、最新のデータが属するチャンクの Chunk ID を返す間接参照である。

指定されたチャンクにデータが存在する場合はその Chunk ID 自身を、存在しない場合はそれより過去に遡って、最初にデータが見つかるチャンクの Chunk ID を返す。
これにより、クライアントは空のチャンクを一つずつ辿ることなく、データの存在するチャンクへ直接到達できる。

レスポンスは Chunk ID をプレーンテキストで返却する。

`/desc/itr/5785524` へのリクエストに対するレスポンス例:

```txt
5785520
```

（この例では、チャンク 5785524 から 5785521 までは空であり、最新のデータはチャンク 5785520 に属している。）

Iterator Node は、Chunkline Document の `first_chunk` から `last_chunk`（省略時は現在時刻に対応するチャンク）までの範囲ですべてアクセス可能でなければならない (MUST)。

指定された Chunk ID 以前にデータが一切存在しない場合、サーバーは 404 Not Found を返すべきである (SHOULD)。

### 4.2. Body Node

Body Node は、該当チャンクに含まれる実際の投稿データおよび投稿への参照を提供する。

レスポンスは、以下の構造を持つエントリの JSON 配列である。

`/desc/body/5785520` へのリクエストに対するレスポンス例:

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

* `content` (optional)
  エントリの本文をインラインで表現する文字列。

* `href` (optional)
  エントリの実体への参照 URI。

* `content_type` (optional)
  `content` あるいは `href` の指す実体のメディアタイプ。

各エントリは `content` または `href` の少なくとも一方を含まなければならない (MUST)。

Body Node のレスポンスは、チャンクに含まれる投稿を全て含む配列でなければならない (MUST)。また、チャンクに含まれない投稿を含んでもよい (MAY)。
該当チャンクに投稿が存在しない場合は、404 ではなく空の JSON 配列を返す。
Body Node のレスポンスは、ascending ノードであれば昇順、descending ノードであれば降順でエントリが並んだ配列を返却しなければならない (MUST)。

## 5. Chunk ID の計算方法

Chunk ID は、投稿のタイムスタンプに基づいて計算される。
具体的には、投稿の UNIX タイムスタンプ（秒）を `chunk_size` で割り、その商の整数部分を Chunk ID とする。

例えば、`chunk_size` が 600 秒（10分）の場合、タイムスタンプが `2025-11-23T12:34:56Z` の投稿は、UNIX タイムスタンプに変換すると 1763901296 となり、これを 600 で割ると 2939835.49... となる。
整数部分の 2939835 が Chunk ID となる。

逆に、Chunk ID からそのチャンクの開始時刻は `chunk_id * chunk_size` で得られる。

## 6. アイテムの同一性と重複排除

複数の Chunkline を横断して読み取る場合や、チャンク境界をまたいで走査する場合、クライアントは同一エントリの重複を排除する必要がある。

エントリの識別子は次のように導出する。

1. `href` が存在する場合、`href` の値をそのまま識別子とする。
2. `href` が存在しない場合、`content` の xxh3 ハッシュを用いて `urn:xxh3:<hex>` 形式の識別子を生成する。

クライアントは、この識別子が一致するエントリを同一とみなし、重複を排除すべきである (SHOULD)。

## 7. アイテムの撤回 (Removed Items)

チャンクは静的にキャッシュ・配信されることを想定しているため、公開済みチャンクからエントリを直接削除できない場合がある。
そこで Chunkline は、フィードごとに「最近撤回されたエントリの識別子リスト」を配信する **Removed Node** を定義する。

### 7.1. Removed Node

Removed Node は、Chunkline Document の `removed` フィールドに指定された URI テンプレートで解決される。
レスポンスは、このフィードから最近撤回されたエントリの識別子（6章の導出規則によるもの）の JSON 配列である。

レスポンス例:

```json
["cckv://con1alice/posts/hello", "urn:xxh3:1a2b3c4d5e6f7890"]
```

* 識別子は、Body Node が同一エントリに対して返す識別子と一致しなければならない (MUST)。
* サーバーは、撤回されたエントリを、少なくとも読者が Body チャンクをキャッシュしうる期間以上リストに掲載し続けるべきである (SHOULD)。
  参考実装の掲載期間は 48 時間であり、Iterator / Body のキャッシュ TTL（2日）と揃えている。
* 掲載期間を過ぎたエントリはリストから取り除いてよい (MAY)。Removed Node は撤回の恒久的な記録ではなく、キャッシュ更新のためのヒントである。

