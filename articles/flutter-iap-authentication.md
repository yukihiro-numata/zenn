---
title: "IAPで保護されているリソースに対して、FlutterアプリからAPIアクセスしてみたらうまく行かなかった話"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Google", "GCP", "Firebase"]
published: false
---

# 概要
タイトルにある通り、Flutter製のアプリからIAP(Identity Aware Proxy)で保護されたリソースに対してAPIアクセスしようと試みました。
前提として、アプリにはFirebaseが紐づいているのですが、IAPで保護されたリソースはFirebaseが紐づいているGCPプロジェクトとは別のプロジェクト上に存在しています。
[IAPのリファレンス](https://cloud.google.com/iap/docs/authentication-howto?hl=ja)を読みながら進めたのですが、アクセスすることができませんでした。

# 試したこと
前述のリファレンスにある「プログラムによる認証＞モバイルアプリからの認証」を読むと、Android/iOS別に対応が必要なことがわかると思います。
こちらにはFlutterについては記載がないのですが、基本的にやることはあまり変わりないと思います。
Android/iOSのどちらにおいてもOAuthクライアントを使ってOIDC(OpenID Connect)トークンを取得、取得したトークンをAuthorizationヘッダーに付与してAPIリクエストするといった手順です。
OIDCトークンの取得ですが、Flutterの場合は[google_sign_in](https://pub.dev/packages/google_sign_in)というパッケージがあるので、こちらを利用しました。

## iOS
先に結論ですが、iOSの場合は疎通に成功することができました。
前述した通り、OAuthクライアントを作成し`google_sign_in`のパッケージを使ってアプリに組み込むだけです。`
組み込み方については`google_sign_in``のリファレンスに手順が載っているので割愛しますが、記載の通りに実装すればうまくアクセスできます。

## Android
iOSと比較して、、Androidは疎通できませんでした。