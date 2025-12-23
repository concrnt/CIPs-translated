# CIP-9 Policy

## 0. Abstract
本仕様は、Concrntのリソースに対するアクセス制御ポリシーを定義する手段を提供する。

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

## 3. Introduction

## 4. Policy Document

{
  "name": "string",
  "versions": {
    "2025-12-23": {
      "statements": {
        "<action-name>": [
          (see 4.1 Statement)
        ]
      }
    }
  }
}

name: ポリシー名
versions: ポリシードキュメントのバージョン
  現在の最新バージョンは "2025-12-23"であり、このドキュメントではこのバージョンを定義する
statements: アクションごとのステートメントリスト

### 4.1. Statement

{
    "effect": "ok" | "allow" | "deny",
    "rule": {
        (see 4.2 Rule)
    },
    "message": "string (optional)"
}

### 4.2. Rule

以下の演算子から構成される
- And
- Or
- Not
- Eq
- Const
- Contains
- LoadParam
- LoadDocument
- LoadSelf
- LoadResoruce
- DomainFQDN
- DomainCSID
- IsCCID
- IsCSID
- IsCKID
- IsRequesterLocalUser
- IsRequesterRemoteUser
- IsRequesterGuestUser
- RequesterHasTag
- RequesterID

## 4.3. Actions


## 5. 評価方法

## 6. 例


