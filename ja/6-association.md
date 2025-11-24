# CIP-6: Association

## 0. Abstract

本仕様では、ある Concrnt Document から別の Concrnt Document への関連付け（アソシエーション）を表現する方法を定義します。関連付けの作成方法、取得 API、検証上の注意点を示します。

## 1. Status of This Memo

本ドキュメントは Concrnt Association のインターネット・ドラフトです。Concrnt プロジェクトが公開するバージョン付き仕様であり、実装者およびプロトコル設計者を対象とします。ドラフト期間中は後方互換性のない変更が行われる可能性があるため、実装者は CIP 番号とバージョンを確認し、更新に追従しなければなりません（MUST）。

## 2. 表記規則

大文字のキーワードは BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。

## 3. Association の表現

Association は CIP-1 の Document を拡張し、`associate` フィールドに関連先の CCURI を記載します。`associate` の `owner` は関連先 Document の `owner` と同一でなければなりません（MUST）。複数の関連付けを行う場合は、別々の Association Document を作成します。クライアントは関連先の `owner` を管理するサーバに対して、CIP-2 の Commit エンドポイントへ送信します。

`associate` で示される Document のスキーマやバリアントを用いて関連種別（返信、いいね、フォローなど）を表現できます。必要に応じて `schema` 内で追加メタデータを定義してください。

## 4. 関連の取得

サーバはリソースの応答に関連一覧 API のパスを含めてもかまいません（MAY）。例として、`apis.associations` は関連付けられた Association Document の一覧を返し、`apis.associationCounts` は `schema` ごとの件数を返します。クエリパラメータで `schema` や `author` を指定して絞り込みを行うことができます。ページングやソート方法は実装が定義します。

遠隔サーバに存在する関連を取得する場合、サーバは対象ドメインの API を参照し、その結果を中継する実装を採用してもかまいません（MAY）。中継時には応答の署名や Affiliation を検証し、改ざんを防ぎます。

## 5. セキュリティと整合性

関連付けを受け付けるサーバは、`associate` の所有関係を検証し、無権限の関連付けを拒否しなければなりません（MUST）。リスト API に対しては認可とレート制限を適用し、スパム的な関連付けを抑制してください。遠隔サーバからの関連付け取得では、応答の署名や Affiliation を検証することが推奨されます。

## 6. 参考文献 (References)

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, March 1997.  
[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, May 2017.***
