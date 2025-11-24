# CIP-7: Realtime API

## 0. Abstract

本仕様は、Concrnt サーバがリソースの変更イベントをリアルタイムに通知するための Realtime エンドポイントを定義します。WebSocket を用いてクライアントが購読し、作成や削除イベントを受け取ります。

## 1. Status of This Memo

本ドキュメントは Concrnt Realtime API のインターネット・ドラフトです。Concrnt プロジェクトが公開するバージョン付き仕様であり、実装者およびプロトコル設計者を対象とします。ドラフト期間中に後方互換性のない変更が行われる可能性があるため、実装者は CIP 番号とバージョンを確認し、更新に追従しなければなりません（MUST）。

## 2. 表記規則

大文字のキーワードは BCP 14 [RFC2119] [RFC8174] にしたがって解釈されます。

## 3. エンドポイント広告

サーバは `.well-known/concrnt` に `net.concrnt.core.realtime` を含め、WebSocket ハンドシェイクを受け付けるパスを示します。

```json
{
  "version": "2.0",
  "csid": "ccs1<bech32-encoded-address>",
  "endpoints": {
    "net.concrnt.core.entity": "/entity/${ccid}",
    "net.concrnt.core.resource": "/resource/${uri}",
    "net.concrnt.core.realtime": "/realtime"
  }
}
```

## 4. 接続と購読

クライアントは広告されたパスに対して WebSocket 接続を確立します。接続後、購読を開始するために次のような JSON メッセージを送信します。

```json
{
  "action": "subscribe",
  "resources": [
    "cc://<CCID>/<key>",
    "cc://<CCID>/<collection>"
  ]
}
```

複数回送信された場合、サーバは最新の購読リストに置き換えます。購読対象が別サーバにある場合、代理で購読するかどうかは実装に委ねられます。

## 5. イベント配信

購読したリソースに変更が発生したとき、サーバは次の形式で通知します。

```json
{
  "event": "created",
  "source": "cc://<CCID>/<key>",
  "resource": "cc://<CCID>/<changed>",
  "signed_document": { ... }
}
```

`event` には `"created"` または `"deleted"` を用います。`signed_document` は CIP-1 の Signed Document であり、必要に応じて省略されることがあります。順序や再送を管理するためのフィールドを追加しても構いません（MAY）。

## 6. ハートビートと切断

長時間の接続維持のため、サーバとクライアントは定期的に ping/pong などのハートビートを交換することが望まれます（SHOULD）。切断後はクライアントが再接続し、必要に応じて購読を再送します。

## 7. セキュリティに関する考慮事項

Realtime 接続は TLS 上で確立し、認証が必要な場合はトークンやクッキーで認証します。過剰な購読やイベント送出はレート制限を設け、悪用を防ぎます。代理購読を行う場合は中継先の信頼性を確認し、漏洩や改ざんに注意してください。

## 8. 参考文献 (References)

[RFC2119] Bradner, S., “Key words for use in RFCs to Indicate Requirement Levels”, March 1997.  
[RFC8174] Leiba, B., “Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words”, May 2017.  
[RFC6455] Fette, I. and A. Melnikov, “The WebSocket Protocol”, December 2011.  
[RFC7231] Fielding, R., et al., “Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content”, June 2014.
