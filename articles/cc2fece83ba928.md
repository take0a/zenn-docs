---
title: "Envoy入門（その４）Lua"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["envoy", "docker", "lua", "webapi", "security"]
published: false
publication_name: "robon"
---

# はじめに

マイクロサービスやWeb API界隈では、サービス間のネットワークの制御をライブラリではなく、プロキシのコンテナをサイドカーとして使うのだとか。そのデファクトスタンダード的な立ち位置なのが Envoy さん。

[（その３）](https://zenn.dev/robon/articles/08af35f4a3672b)では、Envoy さんの Sandbox ではなく、[Auth0](https://auth0.com/jp) を使用した JWT 認証にチャレンジしましたが、今回は、[（その１）](https://zenn.dev/robon/articles/fc7feab5e77d59)や[（その２）](https://zenn.dev/robon/articles/2896faa9bbe72d)と同様、Envoy さんの Sandbox へ戻ります。

# やってみた
## Lua って

Lua さんです。

https://www.lua.org/

聞いたことはありましたが、使ったことはありません。[Wikipedia](https://ja.wikipedia.org/wiki/Lua) さんでは、以下のような説明になっています。

> Lua（ルア）はスクリプト言語およびその処理系の実装で、主にリオデジャネイロ・カトリカ大学（英語版）のコンピュータ科学科 (Department of Computer Science) および/または同大学附属研究所のTecgraf/PUC-Rioに所属するロベルト・イエルサリムスキー Roberto Ierusalimschy、Waldemar Celes、Luiz Henrique de Figueiredoらによって設計開発された。

まぁ、やってみましょう。

今回は、[Lua filter](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/lua) です。

## 環境構築

今回も省略します。必要な方は、[（その１）](https://zenn.dev/robon/articles/fc7feab5e77d59) をご覧ください。

## Lua filter

Envoy の Lua フィルターについては、[ここ](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter) に書かれています。今回は、都度見ることにします。

### Step 1

今回も Step 1 では、必要なものを全部作って、動かします。書いてあるとおりに動きます。

```bash
$ docker-compose pull
Pulling proxy       ... done
Pulling web_service ... done

$ docker-compose up --build -d
Creating network "lua_default" with the default driver
Creating lua_proxy_1       ... done
Creating lua_web_service_1 ... done

$ docker-compose ps
Name                     Command               State                          Ports                       
----------------------------------------------------------------------------------------------------------------
lua_proxy_1         /docker-entrypoint.sh /usr ...   Up      10000/tcp, 0.0.0.0:8000->8000/tcp,:::8000->8000/tcp
lua_web_service_1   /bin/echo-server                 Up      0.0.0.0:8080->8080/tcp,:::8080->8080/tcp         
```

いつものように `docker-compose.yaml` で登場人物（と言っても２人ですが）を見ていきます。

https://github.com/envoyproxy/examples/blob/main/lua/docker-compose.yaml

今回は、Envoy が一つで `lua_proxy_1`。内部サービスも一つで `lua_web_service_1`。
これまでとは異なり、Envoy さんは、マルチステージビルドのターゲット `envoy-lua` を指定してます。こちらを見ておくと、

https://github.com/envoyproxy/examples/blob/main/shared/envoy/Dockerfile#L77-L78

`./lib/mylibrary.lua` を取り込んでます。これも中身を見ておくと

https://github.com/envoyproxy/examples/blob/main/lua/lib/mylibrary.lua

ということで、外部の Lua のソースコードも取り込めるということですね。見たとおり `M` オブジェクトに `foobar` メソッドを付けた感じと思います。

内部サービスは、（その１）の `echo2` ではなくて、`echo` で、全く別のイメージですが、ま、名前のとおりなのでしょう。

https://github.com/envoyproxy/examples/blob/main/shared/echo/Dockerfile

### Step 2

ということで、今回も実験します。今回も出力を絞り込まずに全量表示します。

```bash
$ curl -v localhost:8000
*   Trying [::1]:8000...
* Connected to localhost (::1) port 8000
> GET / HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/8.3.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< content-type: text/plain
< date: Wed, 06 Nov 2024 05:05:57 GMT
< content-length: 226
< x-envoy-upstream-service-time: 0
< server: envoy
< 
Request served by 5d818f883517

GET / HTTP/1.1

Host: localhost:8000
Accept: */*
Foo: bar
User-Agent: curl/8.3.0
X-Envoy-Expected-Rq-Timeout-Ms: 15000
X-Forwarded-Proto: http
X-Request-Id: 7bf57a12-62cf-4e95-b1f1-51233ca1c779
* Connection #0 to host localhost left intact
```

`Foo: bar` というリクエストヘッダーを内部サービスに送るようにした。ということですね。この動きのための設定は、以下の部分です。

https://github.com/envoyproxy/examples/blob/main/lua/envoy.yaml#L38-L48

### Step 3

同じく全量表示です。

```bash

$ curl -v localhost:8000/multiple/lua/scripts
*   Trying [::1]:8000...
* Connected to localhost (::1) port 8000
> GET /multiple/lua/scripts HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/8.3.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< content-type: text/plain
< date: Wed, 06 Nov 2024 05:06:37 GMT
< content-length: 246
< x-envoy-upstream-service-time: 0
< header_key_1: header_value_1
< server: envoy
< 
Request served by 5d818f883517

GET /multiple/lua/scripts HTTP/1.1

Host: localhost:8000
Accept: */*
Foo: bar
User-Agent: curl/8.3.0
X-Envoy-Expected-Rq-Timeout-Ms: 15000
X-Forwarded-Proto: http
X-Request-Id: b3046a4b-f73f-4f9b-9fc3-4a569af1da31
* Connection #0 to host localhost left intact
```

URL を `/multiple/lua/scripts` に変えたら、`Foo: bar` というリクエストヘッダーの追加に変わりありませんが、レスポンスヘッダーに `header_key_1: header_value_1` も追加されています。こちらの設定は、この部分です。

https://github.com/envoyproxy/examples/blob/main/lua/envoy.yaml#L21-L33

# おわりに

今回、Lua を取り上げたのは、Auth0 で RBAC（Role-Based Access Control）を設定して、JWT に反映された内容に応じた認可を Envoy で実現したかったためです。

ですから、たぶん、第５回もありますね。きっと。
