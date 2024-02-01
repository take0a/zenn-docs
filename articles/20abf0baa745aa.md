---
title: "Lightsailでサイトを作る（その２）"
emoji: "⛵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lightsail", "wordpress", "mysql", "loadbalancer", "s3"]
published: true
publication_name: "robon"
---

# はじめに
クラウドソーシングを活用してマーケティングサイトを作ることになりました。

https://zenn.dev/robon/articles/cceae938662f75

前回は、ここまで準備したので、いよいよ主役のLightsailです。

# Lightsail
AWSでWordPressを「早く」「安く」「簡単に」運用するには、Lightsailは良い選択肢ではないかと思います。構成を再掲しておきます。
![](/images/cceae938662f75/deployment.png =600x)

## データベース
Lightsailのコンソールはかわいらしい感じで良いですね。

左のパネルで「データベース」を選んで「データベースの作成」を行います。
高可用性にするのは、後からでもできるので、デフォルトで選ばれているMySQLの新しいバージョンのまま、一番安いプランのまま、一番下の「データベースの作成」ボタンを押します。作成中になるので数分間（ちょっと長め）見守ります。ボタンを２回押すだけなので誰でもできます。

利用可能になったら、リソース名（本当に何もしていなければDatabase-1）を選んで、「接続」タブの設定を確認しておきます。以下の項目は後ほど使用しますが、いつでも確認できますので、メモらなくても大丈夫です。
- ユーザー名
- パスワード
- エンドポイント
- ポート

「ネットワーキング」タブの「パブリックモード」が「無効」の場合は、以下の説明のとおり安全と考えて良いでしょう。
> パブリックモードを有効にすると、データベースのユーザー名とパスワードを持つ誰もが、これに接続できるようになります。このモードが無効の時は、データベースと同じリージョンにある Lightsail リソースのみが接続できます。

データベースをインストールするという仕事は不要な時代なのでしょうね。

## インスタンス
次はWordPressです。
左のパネルで「インスタンス」を選んで「インスタンスの作成」を行います。
デフォルトで選ばれているLinuxのWordPressのまま、プランだけ$5から一番安い$3.5に変更します。以下のように警告されますが、無視して、一番下の「インスタンスの作成」ボタンを押します。こちらは数秒で終わります。
> より大きなインスタンスプランをご検討ください
> 選択したプランでは、アプリケーションの速度が遅くなったり、アプリケーションが応答しなくなったりするなど、パフォーマンスの問題が発生する場合があります。$5 USD (1 GB RAM) インスタンスプラン以上を選択することをご検討ください

今回の構成では、SSL証明書による暗号化はロードバランサーに任せ、画像や動画といったメディアのダウンロードはCloudFrontとS3に任せ、MySQLは上で作ったデータベースに任せますし、リクエスト数が多い場合はインスタンスを追加するので一番小さいサイズにします。
スナップショットからのインスタンス作成をする際、小さいスナップショットから大きなインスタンスはできても逆はできないという理由もあります。

作成したらリソース名（本当に上のとおりにやればWordPress-1）のカードの右上のターミナルアイコンをクリックするとブラウザベースのSSHターミナルが開きます。後でやるパスワードの確認程度ならこれでもいいんですが、viやると調子が悪いので、30年来のvi派としては、SSHターミナルを別途用意することにします。

リソース名を選んで、「接続」タブの設定を確認しておきます。
- 接続先
- ユーザー名
- SSHキー

### VSCode Remote Development
VSCodeがないと何もできない体になってしまったので、WordPressの設定もVSCodeから行います。[VSCode Remote DevelopmentのSSH](https://code.visualstudio.com/docs/remote/ssh)の設定ができていれば、接続先を追加するだけです。

以下、設定を行いますが、お好きなSSHターミナルでもLightsailのブラウザのSSHターミナルでも問題ありません。

SSHで接続すると、こんな感じになります。
```bash
bitnami@ip-172-26-9-97:~$ pwd
/home/bitnami
bitnami@ip-172-26-9-97:~$ ls -al
total 44
drwxr-xr-x 4 bitnami bitnami 4096 Jan  7 06:12 .
drwxr-xr-x 3 root    root    4096 Dec 12 11:33 ..
-rw-r--r-- 1 bitnami bitnami  220 Aug  4  2021 .bash_logout
-rw-r--r-- 1 bitnami bitnami 4220 Jan  7 05:55 .bashrc
-rw------- 1 bitnami bitnami   13 Jan  7 05:56 bitnami_application_password
-r-------- 1 bitnami bitnami  430 Jan  7 05:56 bitnami_credentials
lrwxrwxrwx 1 bitnami bitnami   27 Dec 12 11:36 htdocs -> /opt/bitnami/apache2/htdocs
-rw-r--r-- 1 bitnami bitnami 1501 Jan  7 05:55 .profile
drwx------ 2 bitnami bitnami 4096 Jan  7 05:55 .ssh
lrwxrwxrwx 1 bitnami bitnami   12 Dec 12 11:36 stack -> /opt/bitnami
drwxr-xr-x 6 bitnami bitnami 4096 Jan  7 06:13 .vscode-server
-rw-r--r-- 1 bitnami bitnami  183 Jan  7 06:13 .wget-hsts
```

`bitnami_application_password`ファイルの中身がWordPressのuserユーザーのパスワードです。

`wp-config.php`ファイルにS3とMySQLの設定を追加します。
```bash
bitnami@ip-172-26-9-97:~$ ls -l /opt/bitnami/wordpress/wp-config.php 
lrwxrwxrwx 1 root root 32 Jan  7 05:55 /opt/bitnami/wordpress/wp-config.php -> /bitnami/wordpress/wp-config.php
bitnami@ip-172-26-9-97:~$ cd /bitnami/wordpress
bitnami@ip-172-26-9-97:/bitnami/wordpress$ ls -l
total 12
-rw-r----- 1 bitnami daemon 4419 Jan  7 05:55 wp-config.php
drwxrwxr-x 7 bitnami daemon 4096 Jan  7 05:55 wp-content
bitnami@ip-172-26-9-97:/bitnami/wordpress$ sudo cp -p wp-config.php wp-config.php.org
bitnami@ip-172-26-9-97:/bitnami/wordpress$ ls -l
total 20
-rw-r----- 1 bitnami daemon 4419 Jan  7 05:55 wp-config.php
-rw-r----- 1 bitnami daemon 4419 Jan  7 05:55 wp-config.php.org
drwxrwxr-x 7 bitnami daemon 4096 Jan  7 05:55 wp-content
bitnami@ip-172-26-9-97:/bitnami/wordpress$ sudo vi wp-config.php
```

```diff php:wp-config.php
--- wp-config.php       2024-01-07 06:43:35.861903361 +0000
+++ wp-config.php.org   2024-01-07 05:55:37.019823830 +0000
@@ -48,3 +48,3 @@
 
-define( 'DB_USER', 'dbmasteruser' );
+define( 'DB_USER', 'bn_wordpress' );
 
@@ -53,3 +53,3 @@
 
-define( 'DB_PASSWORD', '■■■■■■' );
+define( 'DB_PASSWORD', '▲▲▲▲▲▲' );
 
@@ -58,3 +58,3 @@
 
-define( 'DB_HOST', '■■■■■■.ap-northeast-1.rds.amazonaws.com:3306' );
+define( 'DB_HOST', '127.0.0.1:3306' );
 
@@ -175,7 +175,2 @@
 define( 'WP_AUTO_UPDATE_CORE', 'minor' );
-define( 'AS3CF_SETTINGS', serialize( array(
-       'provider' => 'aws',
-       'access-key-id' => 'AK■■■■■■',
-       'secret-access-key' => '■■■■■■',
-) ) );
 /* That's all, stop editing! Happy publishing. */
```

ローカルのMySQLをLightsailのMySQLへコピーして、WordPressを再起動します。（もちろん、--host は LightsailのMySQLエンドポイントで、`Enter password:`の後ろには、LaightsailのMySQLのパスワードを入力してください）

```bash
bitnami@ip-172-26-9-97:~$ sudo mysqldump -u root --databases bitnami_wordpress --single-transaction --compress --order-by-primary -p$(cat /home/bitnami/bitnami_application_password) | sudo mysql -u dbmasteruser --host ■■■■■■.ap-northeast-1.rds.amazonaws.com --password
mysqldump: Deprecated program name. It will be removed in a future release, use '/opt/bitnami/mariadb/bin/mariadb-dump' instead
mysql: Deprecated program name. It will be removed in a future release, use '/opt/bitnami/mariadb/bin/mariadb' instead
Enter password: 
bitnami@ip-172-26-9-97:~$ sudo /opt/bitnami/ctlscript.sh restart
Restarting services..
bitnami@ip-172-26-9-97:~$ 
```

### WP Offload Media Lite
ブラウザで`WordPress-1`のPublicなIPアドレスへ接続して設定を行います。`http://xxx.xxx.xxx.xxx/wp-login.php`でユーザーは`user`パスワードは、上の`bitnami_application_password`ファイルの中身になります。

`Dashboard`に接続できたら、表示を日本語に変更します。
左のパネルで「Settings」の「General」を選びます。「Site Language」を`日本語`に変更して、一番したの「Save Changes」ボタンを押します。（機能的にはどうってことはないのですが、ホームグラウンドの安心感は大事です）

左のパネルで「プラグイン」の「新規プラグインの追加」を選びます。
右上の検索フィールドに`WP Offload Media Lite`と入力して探します。（検索は日本語にすると精度が落ちるような気もします）以下の長い名前の「今すぐインストール」ボタンを押します。ボタンが「有効化」に変わったら「有効化」ボタンも押します。
> Amazon S3、DigitalOcean Spaces、Google Cloud Storage 用の WP Offload Media Lite

左のパネルで「設定」の「WP Offload Media」を選びます。
「Storage Provider」は`wp-config.php`が反映されているので「Bucket」から設定します。「1. New or Existing Bucket?」は`Use Existing Bucket`のまま、「2. Select Bucket」は`Browse existing buckets`を選ぶと前回作成したBucketが表示されるので「Save Selected Bucket」ボタンを押します。次の「Security」では警告が表示されますが、CloudFrontを使う場合は無視できますので「Keep Bucket Security As Is」ボタンを押します。

ページが変わって「Storage Settings」は、下のURLを見ながら設定ください。（そのままでも良いです）「Delivery Settings」タブへ移動します。一番上の`S3`を`CloudFront`に変更するので「Edit」ボタンを押します。「1. Select Delivery Provide」で`CloudFront`を選択して「Save Delivery Provider」ボタンを押します。前回、代替ドメイン名と証明書まで設定した方は「Use Custom Domain Name (CNAME)」と「Force HTTPS」を指定できます。変更したら「Save Changes」ボタンを押します。

### WP Mail SMTP
前回の記事でSESの設定が終わっている方はメール送信と管理者ユーザーのメールアドレスの変更を行うことができます。

左のパネルで「プラグイン」の「インストール済みプラグイン」を選びます。
「WP Mail SMTP」を「有効化」します。「WP Mail SMTP セットアップウィザード」に入りますので「始めましょう」ボタンを押してください。

「SMTP メーラーを選択する」は「その他のSMTP」を選びます。
「メーラー設定を調整する」は、前回のSESの「SMTP設定」の値を設定してください。その後は「スキップ」してOKです。

左のパネルで「設定」の「一般」を選びます。
「管理者メールアドレス」を変更することができます。変更するとメールが送信されますので、メール本文中のリンクをクリックして完了します。
左のパネルで「ユーザー」の「プロフィール」を選びます。
同様に「メール」を変更することができます。

### MySQLの後始末
LightsailのDatabase-1を用意したので、WordPress-1上のMySQLは不要です。WordPress-1は最小で構成したのでWordPress-1上のMySQLは停止しましょう。また、LightsailのDatabase-1は「パブリックモード」を「無効」にしたので、Database-1に接続するためにもWordPress-1を「安全な」踏み台にする必要があります。

まずは、踏み台から。WordPress-1には、[phpMyAdmin](https://www.phpmyadmin.net/)が入っていますが、publicなIPアドレスから接続すると、以下のメッセージが表示されます。これはセキュリティ的に正しいのでpublicなIPアドレスから接続できるように「努力」してはいけません。
> For security reasons, this URL is only accessible using localhost (127.0.0.1) as the hostname.

ということで、WordPress-1内から`http://localhost/phpmyadmin/`する必要があります。VSCode Remote Developmentをセットアップした人は、80番ポートを転送設定するだけで、VSCodeを実行しているコンピュータから`http://localhost/phpmyadmin/`できます。ユーザーIDとパスワードは、wp-config.php.orgに設定してあったローカル用のもので接続できます。

phpMyAdminでもいいかなと思ったのですが、最終的には、VSCodeの[MySQL](https://marketplace.visualstudio.com/items?itemName=formulahendry.vscode-mysql)拡張にしました。新旧のMySQLの比較も簡単そうだったのとWordPress-1上へのフットプリントが小さいことから「こういうのでいいんだよ」ということにしました。

MySQLの切り替えがうまくいっていることが確認できたら、WordPress-1上のMySQL（途中から気づいていましたが現在はMariaDBですね）を停止します。

```bash
bitnami@ip-172-26-9-97:~$ sudo /opt/bitnami/ctlscript.sh status
apache already running
mariadb already running
php-fpm already running
bitnami@ip-172-26-9-97:~$ sudo /opt/bitnami/ctlscript.sh stop mariadb
Stopped mariadb
bitnami@ip-172-26-9-97:~$ sudo /opt/bitnami/ctlscript.sh status
apache already running
mariadb not running
php-fpm already running
```

この`/opt/bitnami/ctlscript.sh`の中身を見ると、[gonit](https://github.com/bitnami/gonit)という[monit](https://mmonit.com/monit/)クローンで管理しているようです。この設定を書き換えて再起動しないようにします。

```bash
bitnami@ip-172-26-9-97:~$ cd /etc/monit/conf.d/
bitnami@ip-172-26-9-97:/etc/monit/conf.d$ ls -l
total 16
-rw-r--r-- 1 root root 300 Dec 12 11:36 apache.conf
-rw-r--r-- 1 root root 301 Dec 12 11:36 mariadb.conf
-rw-r--r-- 1 root root 294 Dec 12 11:36 php-fpm.conf
-rw-r--r-- 1 root root 311 Dec 12 11:36 varnish.conf.disabled
bitnami@ip-172-26-9-97:/etc/monit/conf.d$ sudo mv mariadb.conf mariadb.conf.disabled
bitnami@ip-172-26-9-97:/etc/monit/conf.d$ ls -l
total 16
-rw-r--r-- 1 root root 300 Dec 12 11:36 apache.conf
-rw-r--r-- 1 root root 301 Dec 12 11:36 mariadb.conf.disabled
-rw-r--r-- 1 root root 294 Dec 12 11:36 php-fpm.conf
-rw-r--r-- 1 root root 311 Dec 12 11:36 varnish.conf.disabled
bitnami@ip-172-26-9-97:/etc/monit/conf.d$ sudo gonit reload
```

これで良いはずだったのですが…。念のため確認すると「マジか…」

```bash
bitnami@ip-172-26-9-97:/etc/monit/conf.d$ ps -ef | grep maria
bitnami    26606   18188  0 01:50 pts/0    00:00:00 grep maria
bitnami@ip-172-26-9-97:/etc/monit/conf.d$ sudo /opt/bitnami/ctlscript.sh restart
Restarting services..
bitnami@ip-172-26-9-97:/etc/monit/conf.d$ ps -ef | grep maria
mysql      26894       1  0 01:51 ?        00:00:00 /opt/bitnami/mariadb/sbin/mysqld --defaults-file=/opt/bitnami/mariadb/conf/my.cnf --basedir=/opt/bitnami/mariadb --datadir=/bitnami/mariadb/data --socket=/opt/bitnami/mariadb/tmp/mysql.sock --pid-file=/opt/bitnami/mariadb/tmp/mysqld.pid
bitnami    27437   18188  0 01:51 pts/0    00:00:00 grep maria
bitnami@ip-172-26-9-97:/etc/monit/conf.d$ sudo /opt/bitnami/ctlscript.sh status
apache already running
php-fpm already running
```

[ここ](https://nedo.im/blog/2023/12/02/how-to-permanently-disable-the-mariadb-service-in-bitnami)の方法でトドメを刺します。（最初の行でcpしたファイルはVSCode上で先のページのように編集しています）

```bash
bitnami@ip-172-26-9-97:~$ sudo cp /root/.provisioner/stackconfig.json .
bitnami@ip-172-26-9-97:~$ sudo su - root
root@ip-172-26-9-97:~# cd .provisioner/
root@ip-172-26-9-97:~/.provisioner# ls -l
total 508
-rw-r--r-- 1 root root    287 Jan  8 02:11 configuration.json
-rw-r--r-- 1 root root     64 Jan  8 02:11 platform.json
-rw-r--r-- 1 root root    110 Jan  8 02:11 reciperunner.json
-rw-r--r-- 1 root root 507087 Dec 12 11:22 stackconfig.json
root@ip-172-26-9-97:~/.provisioner# mv stackconfig.json stackconfig.json.org
root@ip-172-26-9-97:~/.provisioner# cp /home/bitnami/stackconfig.json .
root@ip-172-26-9-97:~/.provisioner# ls -l
total 920
-rw-r--r-- 1 root root    287 Jan  8 02:11 configuration.json
-rw-r--r-- 1 root root     64 Jan  8 02:11 platform.json
-rw-r--r-- 1 root root    110 Jan  8 02:11 reciperunner.json
-rw-r--r-- 1 root root 420233 Jan  8 02:35 stackconfig.json
-rw-r--r-- 1 root root 507087 Dec 12 11:22 stackconfig.json.org
```

念のため、再起動後に検死しておきます。

```bash
bitnami@ip-172-26-9-97:~$ ps -ef | grep maria
bitnami     1378    1338  0 02:40 pts/0    00:00:00 grep maria
bitnami@ip-172-26-9-97:~$ sudo /opt/bitnami/ctlscript.sh status
apache already running
php-fpm already running
bitnami@ip-172-26-9-97:~$ sudo /opt/bitnami/ctlscript.sh restart
Restarting services..
bitnami@ip-172-26-9-97:~$ sudo /opt/bitnami/ctlscript.sh status
apache already running
php-fpm already running
bitnami@ip-172-26-9-97:~$ ps -ef | grep maria
bitnami     2265    1338  0 02:41 pts/0    00:00:00 grep maria
```

これは普通の人にはできないわ…

## ロードバランサー
WordPress-1 が完成したので、スナップショットを作成します。

Lightsailの管理コンソールに戻って「インスタンス」の「WordPress-1」を選んで「スナップショット」タブを開きます。「手動スナップショット」から「スナップショットの作成」を選んで「作成」リンクをクリックします。インスタンスはスナップショットから簡単に生成できるので、こうして作成した複製をロードバランサーに登録することで多重化ができます。

最後に「ネットワーキング」を選んで「ロードバランサーの作成」を行います。特に変更する必要はなく、一番下の「ロードバランサーの作成」ボタンを押すだけです。

そのまま作成した場合は「LoadBalancer-1」を選択します。「ターゲットインスタンス」で「WordPress-1」を選んで「アタッチ」します。アタッチすると「DNS名」のリンクから接続できるようになります。

### Route 53
Route 53は、AWSのDNSサービスです。Route 53で「LoadBalancer-1」の「DNS名」のエイリアスを作ります。`robon.co.jp`のホストゾーンを管理するAWSアカウントの管理コンソールで`cms1.robon.co.jp`を`A`レコードで作ります。エイリアスをONにして「エンドポイントの種類」を「Application Load BalancerとClassic Load Balancerへのエイリアス」にします。「ロードバランサー」には、「LoadBalancer-1のDNS名」をコピペします。

この後、証明書の検証でも`CNAME`レコードの追加が発生しますので、指示されたとおりのレコードを追加します。

### 証明書の作成
Lightsailの管理コンソールの「Loadbalancer-1」の「カスタムドメイン」タブで「証明書を作成」します。「証明書の名前」は「LoadBalancerTlsCertificate-1」のまま「続行する」とドメインを指定できるようになります。上のRoute 53で作成した`cms1.robon.co.jp`を指定して「証明書を作成する」と「証明書が作成されました」となるので「続行する」と「自動検証に失敗しました。手動検証が必要です。」と言われるので「検証の詳細」に書かれている内容を上のRoute 53に登録します。

検証が終わると「ネットワーキング」タブの「プロトコル」の「HTTPS」も「有効」になります。必要に応じて「HTTP から HTTPS へのリダイレクトは非アクティブです」を「HTTP から HTTPS へのリダイレクトはアクティブです」に切り替えます。

# おわりに
こちらの[マーケティングサイト](https://cms1.robon.co.jp/sodan/zeimu)をこれから運用して参ります。
