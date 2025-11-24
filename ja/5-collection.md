# CIP-5 Collection

## 0. Abstract
この仕様では、Concrnt上で複数のDocumentを論理的にまとめて扱うための **Collection** フォーマットを定義する。
また、その中でも特に時系列データを効率的に扱うための **Timeline** フォーマットについても定義する。

## 1. Status of This Memo

Concrnt プロジェクトにより公開されるバージョン付き仕様であり、
実装者およびプロトコル設計者を対象とする。

本仕様はドラフトであり、後方互換性のない変更が行われる可能性がある。
実装者は CIP-番号とバージョンを確認の上、適宜追従すること。

## 2. 表記規則

このドキュメントにおける以下の語は、必ず大文字で記述される場合、
BCP 14 [RFC2119] [RFC8174] にしたがって解釈される。

> MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
> RECOMMENDED, NOT RECOMMENDED, MAY, OPTIONAL

## 3. Collection

### 3.1 Collectionの作成

CIP-1で定義されたConcrnt Documentを用いて、Collectionを作成することができる。
Content-Typeに`application/concrnt.collection+json`を指定する。

### 3.2 Collectionに対する操作

レスポンス例:
```
{
    "metadata": { // document中のvalueフィールドがそのまま入る
        "title": "My Collection",
        "description": "A collection of my favorite posts"
    },
    "apis": {
        "items": {
            "url": "/api/v1/collection/<id>/list",
        }
    }
}
```

Collectionにアクセスした場合、Concrntサーバーはdocument内のvalueを全てmetadataフィールドにコピーし、Collection Documentを生成して返却します。

apis.items.urlにアクセスすることで、Collection内のアイテム一覧を取得できます。

## 4. Timeline

CIP-1で定義されたConcrnt Documentを用いて、Timelineを作成することができる。
Content-Typeに `application/chunked-timeline+json` を指定する。

該当リソースにアクセスした場合、Concrntサーバーはdocument内のvalueを全てmetadataフィールドにコピーし、Chunked Timeline Documentを生成して返却します。

