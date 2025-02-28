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

で、終了にしてよいぐらいの出来なんですが、あまりにもアッサリしすぎていて、どうやって使ったらええねん。という感じなので、記事にしておきますが、どちらかというと、各 OAuth プロバイダー側の話のが長くなる予感がします。

## お題
あるプロバイダーのユーザープールに登録されているユーザーだけに、この Nuxt3 アプリの使用を許可したい。
途中で折れなければ、以下の認証を Nuxt3 に組み込む方法を説明したいと思います。
- Google（Google Workspace 導入企業ならこれで）
- Microsoft（Microsoft365 導入企業ならこれで）
- Auth0（全社員じゃなくて、とか、いろいろやりたい場合）
- Cognito（うちはAWSなので）

## 環境構築

https://nuxt.com/modules/auth-utils#quick-setup

こちらに書いてあるとおりですが、やると、

```
$ pnpx nuxi module add auth-utils
ℹ Resolved nuxt-auth-utils, adding module...
ℹ Installing nuxt-auth-utils@0.5.16 as a dependency
 WARN  2 deprecated subdependencies found: glob@7.2.3, inflight@1.0.6
Packages: +21
+++++++++++++++++++++
Progress: resolved 954, reused 833, downloaded 21, added 21, done

dependencies:
+ nuxt-auth-utils 0.5.16

Done in 6.3s using pnpm v10.4.1
ℹ Adding nuxt-auth-utils to the modules
✔ Types generated in .nuxt
```

このまま `pnpm dev` で初回実行すると、以下の状態の .env が作られます。自動生成される値はランダムですが、自分で指定もできます。

```.env
NUXT_SESSION_PASSWORD=e54e5108e1ef46549b3c3deb9d533b33
```

package.json の更新は当然ですが、nuxt.config.ts の更新もしてくれます。

```diff ts:nuxt.config.ts
@@ -14,6 +14,7 @@
   "dependencies": {
     "@mdi/font": "^7.4.47",
     "nuxt": "^3.15.4",
+    "nuxt-auth-utils": "0.5.16",
     "vue": "latest",
     "vue-router": "latest"
   },
```

## Google Cloud

https://console.cloud.google.com/

`APIとサービス`の`OAuth同意画面`を開きます。

![](/images/a7b7f3fba60baf/oauth.png)

スクショ展示会は嫌いなのですが、文章力が足りず…

![](/images/a7b7f3fba60baf/oauth2.png)
![](/images/a7b7f3fba60baf/oauth3.png)
![](/images/a7b7f3fba60baf/oauth4.png)
![](/images/a7b7f3fba60baf/oauth5.png)
![](/images/a7b7f3fba60baf/oauth6.png)

な感じで作ったら、以下のものを確保しておきます。
- クライアントID
- クライアントシークレット

## nuxt-auth-utils 再び

上で、しらっと設定したリダイレクトURI `http://localhost:3000/auth/google` には、以下をコピーします。

https://github.com/atinux/nuxt-auth-utils/blob/main/playground/server/routes/auth/google.get.ts

あとは、認証が必要な画面から、上記の URL へ飛ばします。そういうのは、VueRouter の Router Guard 的なやつでやってきたと思いますが、Nuxt では、以下でやります。

https://nuxt.com/docs/guide/directory-structure/middleware

```ts: middleware/auth.ts
export default defineNuxtRouteMiddleware(
  async (_to, _from) => {
    const { loggedIn } = useUserSession()
    if (!loggedIn.value) {
      return navigateTo('/auth/google', { external: true })
    }
  },
)
```

で、この middleware を画面に差し込みます。

```diff html:pages/customer/new.vue
@@ -1,6 +1,10 @@
 <script setup lang="ts">
 import type { Customer } from '~/shared/types/customer'
 
+definePageMeta({
+  middleware: 'auth',
+})
+
 const customer = ref<Partial<Customer>>({})
 const formRef = useTemplateRef('form')
```

動かす前に、前段で取得しておいた `クライアントID` と `クライアントシークレット` を `.env` に仕込みます。

```.env
NUXT_SESSION_PASSWORD=e54e5108e1ef46549b3c3deb9d533b33
# Google OAuth
NUXT_OAUTH_GOOGLE_CLIENT_ID=■■■■■■■■■■■■■■■■■■■■.apps.googleusercontent.com
NUXT_OAUTH_GOOGLE_CLIENT_SECRET=GO■■-■■■■■■■■■■■■■■■■■■■■-■■■
```

これで、`http://localhost:3000/customer/new` へ行くと、認証が動きます。

![](/images/a7b7f3fba60baf/login.png)

おしまい。と言いたいところですが、認証を突破すると `http://localhost:3000/` に飛ばされます。これは、上でパクッてきた `google.get.ts` の最後が `return sendRedirect(event, '/')` なってるからなので、以下のように修正して、割り込まれた際の行先にしておきます。

```diff ts:middleware/auth.ts
@@ -1,7 +1,9 @@
 export default defineNuxtRouteMiddleware(
-  async (_to, _from) => {
+  async (to, _from) => {
     const { loggedIn } = useUserSession()
     if (!loggedIn.value) {
+      const cookie = useCookie<string | null>('REDIRECT_COOKIE_NAME')
+      cookie.value = to.fullPath
       return navigateTo('/auth/google', { external: true })
     }
   },
```

```diff ts:server/routes/auth/google.get.ts
@@ -12,6 +12,8 @@ export default defineOAuthGoogleEventHandler({
       loggedInAt: Date.now(),
     })
 
-    return sendRedirect(event, '/')
+    const to = getCookie(event, 'REDIRECT_COOKIE_NAME') || '/'
+    deleteCookie(event, 'REDIRECT_COOKIE_NAME')
+    return sendRedirect(event, to)
   },
 })
```

# おわりに
いかがでしたでしょうか。わかっていれば作るのは簡単ですよね。
次回は、他のプロバイダーも紹介するかもしれません。
