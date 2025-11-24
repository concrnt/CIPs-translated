# CIP-5: Collection

## 0. Abstract

本仕様では、複数の Concrnt Document を論理的にまとめる Collection と、時系列データを効率的に扱う Timeline の概念を定義します。メタデータの公開とアイテム一覧の取得方法を示し、コレクションを介した配信を可能にします。

## 1. Status of This Memo

本ドキュメントは Concrnt Collection のインターネット・ドラフトです。Concrnt プロジェクトが公開するバージョン付き仕様であり、実装者およびプロトコル設計者を対象とします。ドラフト期間中に後方互換性のない変更が行われる可能性があるため、実装者は CIP 番号とバージョンを確認し、更新に追従しなければなりません（MUST）。

## 2. 表記規則

大文字のキーワードは BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。

## 3. Collection の作成

Collection は CIP-1 の Document を用いて表現します。`contentType` に `application/concrnt.collection+json` を指定し、`value` にタイトルや説明などのメタデータを含めます。サーバはリソースへアクセスがあった際、`value` を `metadata` としてコピーした Collection Document を応答として返します。

メタデータに含めるフィールドは上位仕様で定義しますが、タイトル、説明、カバー画像などの情報を含めることが想定されています。Collection 自体にも CCURI が割り当てられ、`cc://<owner>/<collection-key>` で参照されます。

## 4. Collection の操作

サーバは Collection へのアクセスに対し、メタデータと API エンドポイントを記述した JSON を返します。`apis.items.url` にはアイテム一覧を取得するパスを示し、クライアントはこれを用いてアイテムをページング取得できます。順序やソート方法はサーバ実装に依存しますが、最新順で返すことが推奨されます（SHOULD）。

一覧の応答には、各アイテムの CCURI、作成時刻、必要に応じて抜粋やメタ情報を含めます。ページングのために `next` や `cursor` を返してもかまいません（MAY）。

## 5. Timeline

Timeline は時系列データ向けの Collection であり、`contentType` に `application/chunked-timeline+json` を指定します。アクセス時には CIP-4 で定義された Chunked Timeline Document を生成し、`value` を `metadata` として提供します。

## 6. 要素の追加

Document は `memberOf` フィールドを用いて所属する Collection や Timeline の CCURI を宣言できます。サーバは `memberOf` を解釈し、管理下の Collection に追加しなければなりません（MUST）。対象が別サーバにある場合、サーバは CIP-2 の Commit エンドポイントに代理送信してもかまいません（MAY）。重複や衝突が発生する場合の優先規則は実装が定義します。

サーバは `memberOf` の各 CCURI について所有権やポリシーを確認し、無権限の追加を拒否する必要があります。代理送信を行う場合は、送信結果をクライアントに通知するか、一定回数再試行するポリシーを設けます。

## 7. セキュリティと一貫性の考慮

`memberOf` を信頼して自動配布する場合、権限検証を行い、無権限の追加を拒否する必要があります。リスト API にはレート制限や認可を適用し、偏った配信やスパムを抑制してください。Collection の整合性を保つため、追加や削除のイベントを監査ログに記録することが望まれます。

## 8. 参考文献 (References)

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, March 1997.  
[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, May 2017.
