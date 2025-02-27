---
title: "いまさらNuxt3（その３）"
emoji: "⛰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nuxt3", "vue", "vuetify", "auth", "googlecloud"]
published: false
publication_name: "robon"
---

# はじめに
その１で「環境構築」して、その２で「普通のアプリケーションの作り方」を説明したので、その２の最後で予告した「認証」をやろうと思います。

# やってみる
## nuxt-auth-utils
こいつです。

https://nuxt.com/modules/auth-utils

で終了にしてよいぐらいな出来なんですが、どうやって使ったらええねん。という感じなので、記事にしておきますが、どちらかというと、各 OAuth プロバイダー側の話のが長くなる予感がします。

## お題
あるプロバイダーのユーザープールに登録されているユーザーだけに Nuxt3 アプリの使用を許可したい。途中で折れなければ、以下の認証を Nuxt3 に組み込む方法を説明したいと思います。
- Google（Google Workspace 導入企業ならこれで）
- Microsoft（Microsoft365 導入企業ならこれで）
- Auth0（全社員じゃなくて、いろいろやりたい場合）
- Cognito（うちはAWSなので）

## Google Cloud


# おわりに