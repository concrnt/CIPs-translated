# CIP-4: Chunked Timeline

## 0. Abstract

本仕様では、大規模なフィードを時間軸に沿ってチャンクに分割し、効率的に配信・取得するための Chunked Timeline フォーマットを定義します。タイムラインを複数のノードで表現し、部分取得や CDN 配信を容易にします。

## 1. Status of This Memo

本ドキュメントは Concrnt Chunked Timeline のインターネット・ドラフトです。Concrnt プロジェクトが公開するバージョン付き仕様であり、実装者を対象とします。ドラフト期間中に後方互換性のない変更が行われる可能性があるため、実装者は CIP 番号とバージョンを確認し、更新に追従しなければなりません（MUST）。

## 2. 表記規則

大文字のキーワードは BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。

## 3. イントロダクション

従来のフィード形式では全体を単一ドキュメントで扱うため、部分取得や差分取得が非効率でした。Chunked Timeline はフィードを固定長のチャンクに分割し、最新部分へのポインタを Iterator Node が示します。クライアントは必要なチャンクだけを取得することでトラフィックを削減できます。

## 4. Chunked Timeline Document

Chunked Timeline Document はタイムラインのメタデータと、チャンクを取得するためのパスを含む JSON オブジェクトです。`version`、`chunkSize`、`firstChunk` は必須で、`ascending` または `descending` に各チャンクの Iterator と Body のパスを含めます。`metadata` にはタイトルや説明など任意の情報を置くことができます。

## 5. Iterator Node

Iterator Node は最新の Body Node を指すチャンク番号を返します。クライアントは `firstChunk` から `lastChunk`（存在する場合）までの範囲で Iterator にアクセスできます。レスポンスは本文を持たないプレーンテキストまたは JSON とし、チャンク番号を一意に返します。

## 6. Body Node

Body Node は指定されたチャンクに含まれるエントリを配列で返します。エントリはタイムスタンプ順に並べられ、`ascending` では昇順、`descending` では降順です。エントリは本文データを直接含めても、外部リソースへの参照を含めても構いません。空のチャンクは空配列で返します。

## 7. チャンク ID の計算

チャンク ID は投稿の UNIX タイムスタンプを `chunkSize` で割った商の整数部分です。時刻は UTC で扱い、丸めは切り捨てとします。例として、`chunkSize` が 300 秒で `"2025-11-23T12:34:56Z"` の投稿は UNIX 時刻 1761280496 から 300 で割った 5870934 がチャンク ID になります。

## 8. エラー処理とキャッシュ

範囲外のチャンクを要求された場合、サーバは 404 Not Found を返します。チャンク内容が更新されない場合は適切なキャッシュヘッダを付与し、CDN 配信を可能にします。最新チャンクが頻繁に更新される場合でも、Iterator を通じて新しい Body の位置を提供します。

## 9. セキュリティに関する考慮事項

Chunked Timeline は公開フィードを前提としますが、改ざんや差し替えを防ぐため HTTPS を利用し、キャッシュの検証を行うことが望まれます。エントリが外部参照を含む場合、その可用性と整合性を確認する責任はクライアントにあります。

## 10. 参考文献 (References)

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, March 1997.  
[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, May 2017.  
[RFC7231] Fielding, R., et al., “Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content”, June 2014.
