# CIP-5 Chunkline

## 0. Abstract
この仕様では、フィードをチャンクに分割し、効率的に配信・取得するための Chunkline フォーマットを定義する。
Chunklineは "Chunked Timeline" の略であり、大規模なタイムラインデータの配信に適した構造を提供する。

## 1. Status of This Memo

このドキュメントは、Concrnt プロジェクトにより公開されるバージョン付き仕様ではあるが、
CIP-4はその適用範囲をConcrntプロジェクトのみに限定せず、広く一般に利用可能な仕様として提供されることを目的としている。

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

この仕様では、Chunkline Documentとそれが指し示すIterator Node、そしてBody Nodeについて説明しています。

Chunkline Documentは、Chunklineの表現であり、フィードに関するメタデータとそれに関連付けられているIterator Node及びBody Nodeへの解決方法を提供します。

Chunkline Documentは、以下のようなJSON構造で表現されます。また、MIMEタイプは `application/chunkline+json`です。

```json
{
    "version": "1.0",                        // required

    "chunkSize": 300,                        // required
    "firstChunk": 5785524,                   // required
    "lastChunk": 5890644,                    // optional

    "ascending": {                           // optional
        "iterator": "/asc/itr/{chunk}",      // required
        "body": "/asc/body/{chunk}",         // required
    },

    "descending": {                          // optional
        "iterator": "/desc/itr/{chunk}",     // required
        "body": "/desc/body/{chunk}",        // required
    },

    "metadata": {                            // optional
        "title": "User's Timeline",          // optional
        "description": "A chunked timeline", // optional
    }
}
```

### 4.1. Iterator Node

iteratorノードは、該当チャンクのうち、最新のデータを含むbodyノードへの参照を提供します。
通常、iteratorノードはチャンクIDをのみを返却します。

`/asc/itr/5785524` へのリクエストに対するレスポンス例:
```txt
5785524
```

iteratorノードは、Chunkline documentのfirstChunkから、lastChunkまでの範囲ですべてアクセス可能でなければなりません (MUST)。

### 4.2. Body Node
bodyノードは、該当チャンクに含まれる実際の投稿データおよび投稿への参照を提供します。

`/asc/body/5785524` へのリクエストに対するレスポンス例:
```json
[
    {
        "timestamp": "2025-11-23T12:34:56Z",
        "content": "Hello, world!"
    },
    {
        "timestamp": "2025-11-23T12:30:00Z",
        "href": "https://example.com/posts/12345"
    }
]
```

bodyノードのレスポンスは、チャンクに含まれる投稿を全て含む配列でなければなりません (MUST)。また、チャンクに含まれない投稿を含んでもよい (MAY)。
bodyノードのレスポンスは、ascendingノードであれば昇順、descendingノードであれば降順でエントリが並んだ配列を返却しなければなりません (MUST)。

## 5. Chunk IDの計算方法
Chunk IDは、投稿のタイムスタンプに基づいて計算されます。
具体的には、投稿のUNIXタイムスタンプをチャンクサイズで割り、その商を整数として扱います。
例えば、チャンクサイズが300秒（5分）の場合、タイムスタンプが "2025-11-23T12:34:56Z" の投稿は、UNIXタイムスタンプに変換すると 1761280496 となり、これを300で割ると 5870934.986666... となります。
整数部分の 5870934 がChunk IDとなります。


