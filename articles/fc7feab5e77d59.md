---
title: "Envoy入門（その１）TLS"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["envoy", "docker", "TLS", "webapi", "security"]
published: false
publication_name: "robon"
---

# はじめに

マイクロサービスやWeb API界隈では、サービス間のネットワークの制御をライブラリではなく、プロキシのコンテナをサイドカーとして使うのだとか。そのデファクトスタンダード的な立ち位置なのが Envoy さん。

ですが、日本語の新しい情報が少ないので、だったら[本家のサイト](https://www.envoyproxy.io/)を見ながら、自力でなんとかするしかないね。という記事です。

Envoy の[ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/)を見てもらうと、お分かりいただけると思いますが、かなり濃ゆい世界なので、入門の入門である [Getting Started/Sandboxes](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/) から、これは必要でしょ！というものをいくつかご紹介したいと思います。

第一回目は、[Transport layer security (TLS)](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/tls) です。

# やってみた
## 環境構築

[Setup the sandbox environment](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/setup) というのがあるのですが、Linux 環境で Docker（Compose 含む）と Git を使っている普通の開発環境なら、特に準備なく行けると思います。

で、元ネタを持ってきます。

```bash
$ git clone https://github.com/envoyproxy/examples.git
```

私は、今回、Amazon Linux 2023 の docker でやるので、`docker compose` は、`docker-compose` と打ってます。

## TLS

[securing Envoy quick start guide](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/securing#start-quick-start-securing) を読みましょうと書いてありますが、[はじめに](#はじめに)で書いたとおり、濃ゆいので、濃ゆいの苦手な方は、まずは、やってみる！をお勧めします。
（どうせ砂場だし。）

### Step 1

ということで Step 1 では、必要なものを全部作って、動かします。書いてあるとおりにやると動きます。

```bash
$ docker-compose pull
Pulling proxy-https-to-http     ... done
Pulling proxy-https-to-https    ... done
Pulling proxy-http-to-https     ... done
Pulling proxy-https-passthrough ... done
Pulling service-http            ... done
Pulling service-https           ... done

$ docker-compose up --build -d
Creating network "tls_default" with the default driver
Creating tls_proxy-http-to-https_1     ... done
Creating tls_proxy-https-to-http_1     ... done
Creating tls_service-http_1            ... done
Creating tls_proxy-https-to-https_1    ... done
Creating tls_service-https_1           ... done
Creating tls_proxy-https-passthrough_1 ... done

$ docker-compose ps
Name                           Command               State                      Ports                    
---------------------------------------------------------------------------------------------------------------------
tls_proxy-http-to-https_1       /docker-entrypoint.sh /usr ...   Up      0.0.0.0:10002->10000/tcp,:::10002->10000/tcp
tls_proxy-https-passthrough_1   /docker-entrypoint.sh /usr ...   Up      0.0.0.0:10003->10000/tcp,:::10003->10000/tcp
tls_proxy-https-to-http_1       /docker-entrypoint.sh /usr ...   Up      0.0.0.0:10000->10000/tcp,:::10000->10000/tcp
tls_proxy-https-to-https_1      /docker-entrypoint.sh /usr ...   Up      0.0.0.0:10001->10000/tcp,:::10001->10000/tcp
tls_service-http_1              docker-entrypoint.sh node  ...   Up                                                  
tls_service-https_1             docker-entrypoint.sh node  ...   Up             
```

簡単に見ておくと、以下の docker-compose.yaml で環境を作っています。

https://github.com/envoyproxy/examples/blob/main/tls/docker-compose.yaml

最初の４つの proxy で始まるコンテナは、同じ Dockerfile でコンテナを作っています。

https://github.com/envoyproxy/examples/blob/main/shared/envoy/Dockerfile

ターゲットなしのマルチステージビルドなので、`envoy-base` のイメージができて、このイメージを共有して、`ENVOY_CONFIG` の `yaml` ファイルだけ変えて、４種類の Proxy にしています。名前のとおり、`外部から`と`内部へ`のプロトコルを変えていると思いますので、それは、Step 2 以降で。

残り２つのコンテナは、Proxy の内側の本物のサービスを模したもので、こちらも上とは異なる同じ Dockerfile でコンテナを作っています。

https://github.com/envoyproxy/examples/blob/main/shared/echo2/Dockerfile

最新のイメージではなく、80 番と 443 番を使っている古いイメージを指定している（と思われます）。それぞれ、使わない方のポートの環境変数を `0` にして塞いでいます。

### Step 2

ということで、実験します。サイトでは `jq` を使って、狙ったところだけを掲載してますが、大した量でもないので、ここでは、全量を表示しておきます。（-s はサイレント、-k は証明書チェックなし）

```bash
$ curl -sk https://localhost:10000
{
  "path": "/",
  "headers": {
    "host": "localhost:10000",
    "user-agent": "curl/8.3.0",
    "accept": "*/*",
    "x-forwarded-proto": "https",
    "x-request-id": "a70daa38-1bd1-4343-8924-72ca3fd440f8",
    "x-envoy-expected-rq-timeout-ms": "15000"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "localhost",
  "ip": "::ffff:172.18.0.2",
  "ips": [],
  "protocol": "https",
  "query": {},
  "subdomains": [],
  "xhr": false,
  "os": {
    "hostname": "service-http"
  },
  "connection": {}
}
```

10000 番は、`tls_proxy-https-to-http_1` のポートで https で受け付けていて、つながった先は http のみの `tls_service-http_1` （ホスト名はディレクトリ名と枝番なしなので`service-http`）でした。

で、この動きをするための設定は、以下のものです。

https://github.com/envoyproxy/examples/blob/main/tls/envoy-https-http.yaml

外部からの受けは、`listeners` に書かれた唯一の `listener` で、その `filter_chains` に `HttpConnectionManager` の設定が書かれていて、ここで内側の行先の `service-http` という名前の `cluster` が指定されています。`cluster` の詳細は、末尾の `clusters` に書かれているとおり `service-http` というホストの 80 番ポートへの送信になります。

この例では、TLS を Envoy が担う必要があるので、`listener` に `transport_socket` として、`DownstreamTlsContext` の設定として、証明書と秘密鍵の値が直書きされていますが、本番ではファイル名でも書けますし、無停止で動的に外部から取り込むこともできるようです。

### Step 3

Step3 は、両方とも https のパターンです。同じように全量の実験結果です。

```bash
$ curl -sk https://localhost:10001
{
  "path": "/",
  "headers": {
    "host": "localhost:10001",
    "user-agent": "curl/8.3.0",
    "accept": "*/*",
    "x-forwarded-proto": "https",
    "x-request-id": "d8f854d2-d027-4af2-82d0-6c03aa0bd040",
    "x-envoy-expected-rq-timeout-ms": "15000"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "localhost",
  "ip": "::ffff:172.18.0.5",
  "ips": [],
  "protocol": "https",
  "query": {},
  "subdomains": [],
  "xhr": false,
  "os": {
    "hostname": "service-https"
  },
  "connection": {
    "servername": false
  }
}
```

設定もよく似たものです。

https://github.com/envoyproxy/examples/blob/main/tls/envoy-https-https.yaml

Step 2 との違いは、`cluster` に `transport_socket` として、`UpstreamTlsContext` が設定されていることです。

### Step 4

Step 4 は、外側が http で、内側が https のパターンなので、省略します。

### Step 5

Step 5 は、パススルーです。まずは、全量の実験結果から

```bash
$ curl -sk https://localhost:10003
{
  "path": "/",
  "headers": {
    "host": "localhost:10003",
    "user-agent": "curl/8.3.0",
    "accept": "*/*"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "localhost",
  "ip": "::ffff:172.18.0.6",
  "ips": [],
  "protocol": "https",
  "query": {},
  "subdomains": [],
  "xhr": false,
  "os": {
    "hostname": "service-https"
  },
  "connection": {
    "servername": "localhost"
  }
}
```

`x-` のヘッダーは、`x-forwarded-proto` 以外のありません。これは、`filter` で、HTTP も TLS も見てなくて、より低レベルの `TcpProxy` が設定されているからです。

https://github.com/envoyproxy/examples/blob/main/tls/envoy-https-passthrough.yaml

ホスト名だけ Proxy で中継してるだけなので、設定はシンプルですね。

# おわりに

以下のように後片付けしておきましょう。

```bash
$ docker-compose down
Stopping tls_proxy-https-passthrough_1 ... done
Stopping tls_service-https_1           ... done
Stopping tls_proxy-https-to-https_1    ... done
Stopping tls_service-http_1            ... done
Stopping tls_proxy-https-to-http_1     ... done
Stopping tls_proxy-http-to-https_1     ... done
Removing tls_proxy-https-passthrough_1 ... done
Removing tls_service-https_1           ... done
Removing tls_proxy-https-to-https_1    ... done
Removing tls_service-http_1            ... done
Removing tls_proxy-https-to-http_1     ... done
Removing tls_proxy-http-to-https_1     ... done
Removing network tls_default
```

まぁ、よく使いそうなのは、Step 2 ですが、この Sandbox の内容を押さえておけば、TLS は大丈夫そうですね。
それでは、第２回で。
