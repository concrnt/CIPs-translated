# CIP-7 Policy

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
Concrntでは、リソースの読み書きをポリシーと呼ばれる仕組みを用いてプログラマブルに設定することができる。 これにより、バックエンドに手を加えなくても鍵垢・サークルツイート・簡易的なDMなどを実装することを目的としている。

## 4. Policy Document

TODO: この構造を仕様として定義する
```
type Policy struct {
	Statements map[string]Statement `json:"statements"`
	Defaults   map[string]bool      `json:"defaults"`
}

type Statement struct {
	Dominant       bool `json:"dominant"`
	DefaultOnTrue  bool `json:"defaultOnTrue"`
	DefaultOnFalse bool `json:"defaultOnFalse"`
	Condition      Expr `json:"condition"`
}
```

## 5. Actions

record.read
record.update // 同じキーで作成を許可する
record.delete
record.associate
record.disassociate
collection.write // CIP-5

## 5. 演算子

And
input: bool[] output: bool

inputの全ての要素がtrueと評価されたときにtrueを返します

Or
input: bool[] output: bool

inputのうち1つでもfalseと評価された要素があるときfalseを返します

Not
input: bool output: bool

inputのnotなるboolを返します

Eq
input: [T, T] output: bool

2つの引数の値が等しいときtrueを返します

Const
const: T output: T

constパラメータに与えられた値をそのまま返します

Contains
input: [T[], T] output: bool

inputのT[]の中にTとなる要素があればtrueを返します

LoadParam
const: string output: T

このポリシーとペアで設定されているパラメーターから値を読み出します

LoadDocument
const: string output: T

このポリシーが呼び出されたときに評価中のDocumentから値を読み出します

LoadSelf
const: string output: T

このポリシーがアタッチされているリソースのDocumentから値を読み出します

LoadResource
const: string output: T

このポリシーが呼び出されたときに評価中のリソースのDocumentから値を読み出します

DomainFQDN
output: string

このポリシーを評価中のサーバーのドメイン名を返します

DomainCSID
output: string

このポリシーを評価中のサーバーのCSIDを返します

IsCCID
input: string output: bool

inputで与えられた文字列がCCIDとしてパースできるかを返します

IsCSID
input: string output: bool

inputで与えられた文字列がCSIDとしてパースできるかを返します

IsCKID
input: string output: bool

inputで与えられた文字列がCKIDとしてパースできるかを返します

IsRequesterLocalUser
output: bool

このポリシーを呼び出しているユーザーが、当サーバーが現住所であるユーザーであるかどうかを返します

IsRequesterRemoteUser
output: bool

このポリシーを呼び出しているユーザーが、当サーバーが現住所ではないユーザーであるかどうかを返します

IsRequesterGuestUser
output: bool

このポリシーの呼び出しが無認証アクセスであるかを返します

RequesterHasTag
const: string output: bool

このポリシーの呼び出しユーザーが、constで指定したタグを持っているかどうかを返します

RequesterID
output: string

このポリシーの呼び出しユーザーのCCIDを返します

RequesterDomainHasTag
const: string output: bool

このポリシーの呼び出しユーザーの所属ドメインが、constで指定したタグを持っているかどうかを返します

## 6. Policyの評価



