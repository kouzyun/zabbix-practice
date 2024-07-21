# 構築手順書:VagrantでZabbixサーバーを構築する。
zabbix.menta.me(zabbixサーバー)を構築し、dev.menta.me(wordpress)を立ち上げて監視する。

## 新しくVagrantfileを作成し、zabbixサーバーを構築する。
以下のVagrantfileを新しく作成し、`vagrant up`を行う。

```
# -*- mode: ruby -*-
# vi: set ft=ruby
Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.hostname = "zabbix.menta.me"
  config.vm.network "private_network", ip: "192.168.56.16"
end
```

## nginxのインストールと設定
1. `$ vagrant ssh`でログイン後、`$ yum update -y`を行う。

2. EPELリポジトリを`$ yum -y install epel-release`で追加。

3. remiリポジトリを`$ rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm`で追加。

4. `$ vim /etc/yum.repos.d/nginx.repo`をコマンド入力し、以下のように編集。
```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
gpgcheck=1
enabled=1
gpgkey=http://nginx.org/keys/nginx_signing.key
[nginx-source]
name=nginx source
baseurl=http://nginx.org/packages/mainline/centos/7/SRPMS/
gpgcheck=1
enabled=0
gpgkey=http://nginx.org/keys/nginx_signing.key
```

5. `$ yum -y install nginx`をコマンド入力し、nginxをインストール。

6. 自動アップデート設定を行うため、`$ yum install yum-cron`でyum-cronをインストール。

7. `$ vim /etc/yum/yum-cron.conf`でファイルを開き、`apply_updates = yes`と編集し自動アップデートできるように設定。

8. `$ systemctl start nginx`で、nginxを起動。

9. `$ systemctl enable nginx.service`を入力し、サーバ再起動時にnginxが自動起動するように設定するようにする。

## CentOS7にMySQL8.0をインストール
1. MariaDB関連ライブラリの確認と削除。
  `yum remove mariadb-libs`と`rm -rf /var/lib/mysql/`のコマンド入力。

2. `$ rpm -ivh https://dev.mysql.com/get/mysql80-community-release-el7-2.noarch.rpm`で、リポジトリをインストール。

3. `sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022`で、新しいGPGキーをインストール。

4. mysql-community-serverをインストール。`$ yum install mysql-community-server`をコマンド入力。

5. `$ systemctl start mysqld.service`で、mysqld.serviceを起動。

6. `$ systemctl enable mysqld.service`で、mysqld.serviceを永続化。

## zabbixのインストール
1. Zabbixのリポジトリを登録。
  `$ rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm`

2. `$ sudo yum clean all`で、キャッシュをクリーンする。

3. Zabbixサーバーとエージェントをインストールする。
  `$ yum install zabbix-server-mysql zabbix-agent`

4. Zabbixの管理画面をインストールするため、Red Hat Software Collectionsを有効化。
  `$ yum install centos-release-scl`

5. `/etc/yum.repos.d/zabbix.repo`を編集して`zabbix-frontend`のリポジトリを有効化。
   `[zabbix-frontend]`を`enabled=1`に変更。
    ```
    $ sudo vi /etc/yum.repos.d/zabbix.repo
    ....
    [zabbix-frontend]
    name=Zabbix Official Repository frontend - $basearch
    baseurl=http://repo.zabbix.com/zabbix/5.0/rhel/7/$basearch/frontend
    enabled=1
    gpgcheck=1
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591
    ....
    ```
6. `zabbix-frontend`のパッケージをインストールする。
   `$ yum install zabbix-web-mysql-scl zabbix-nginx-conf-scl`

7. Zabbix用のデータベースとユーザーを作成する。
   ```
   $ mysql -uroot -p
   # ユーザを追加
   mysql> create database zabbix character set utf8 collate utf8_bin;
   mysql> create user zabbix@localhost identified by 'password';
   mysql> grant all privileges on zabbix.* to zabbix@localhost;
   ```

8. Zabbixサーバーホストで初期スキーマとデータをインポート。
  `$ zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix`

9. `/etc/zabbix/zabbix_server.conf`を編集し、Zabbixサーバー用にデータベースを設定。 
  ```
  $ sudo vi /etc/zabbix/zabbix_server.conf
  ...
  # データベースのユーザー作成時のパスワードをセット
  DBPassword=your-password
  ```

10. `zabbix.menta.me.conf`を編集し、ポートとホスト名を変更。
  ```
  $ vi /etc/nginx/conf.d/zabbix.menta.me.conf
  server {
         listen          80;
         server_name     zabbix.menta.me;
      ...
  ```

11. `zabbix.conf`を編集し、ユーザをnginxに変更し、`data.timezone`を`Asia/Tokyo`に変更。
  ```
  [zabbix]
  user = nginx
  group = nginx
  listen = /var/opt/rh/rh-php72/run/php-fpm/zabbix.sock
  listen.acl_users = nginx
  listen.allowed_clients = 127.0.0.1
  pm = static
  pm.max_children = 10
  pm.start_servers = 10
  pm.min_spare_servers = 10
  pm.max_spare_servers = 10
  pm.max_requests = 100
  php_value[session.save_handler] = files
  php_value[session.save_path]    = /var/opt/rh/rh-php72/lib/php/session/
  php_value[max_execution_time] = 300
  php_value[memory_limit] = 128M
  php_value[post_max_size] = 16M
  php_value[upload_max_filesize] = 2M
  php_value[max_input_time] = 300
  php_value[max_input_vars] = 10000
  php_value[date.timezone] = Asia/Tokyo                               
  ```

12. PHPの権限を変更
  ```
  # 実行権限を与える
  $ chmod 755 -R /usr/share/zabbix
  # Nginxユーザを設定
  $ chown -R root:nginx /usr/share/zabbix
  # ライブラリもNginxユーザを設定
  $ chown -R root:nginx /var/opt/rh/rh-php72/lib/php/*
  ```

13. Zabbixを起動して自動起動も有効化。
  ```
  $ sudo systemctl restart zabbix-server zabbix-agent nginx rh-php72-php-fpm
  $ sudo systemctl enable zabbix-server zabbix-agent nginx rh-php72-php-fpm
  ```

## Zabbixにアクセスし、設定
1. Zabbixにアクセスし、MySQLの情報を入力。
  ```
  Database type: MySQL
  Database host: localhost
  Database port: 3306
  Database name: zabbix
  User: zabbix
  Password: Password
  ```

2. ホスト名とポートを設定し、セットアップを進める。
  ```
  Host: zabbix.menta.me
  Port: 10051
  name: zabbix
  ```

4. hostファイルを編集し、zabbix.menta.meでアクセスできるように修正。
  ```
  vi sudo vi /private/etc/hosts
  #以下を追記
  192.168.56.16 zabbix.menta.me
  ```

## dev.menta.meにZabbix Agentをインストール
1. dev.menta.meにZabbix エージェントのインストールする。
  `$ yum install zabbix-agent`

2. `zabbix_agentd.conf`を編集し、設定ファイルを修正。
  ```
  $ vi /etc/zabbix/zabbix_agentd.conf
  # 以下内容を変更
  # Server=127.0.0.1
  Server=192.168.56.15
  # ServerActive=
  ServerActive=192.168.56.15
  # Hostname=Zabbix server
  Hostname=dev.menta.me
  ```

3. zabbix-agentサービスの起動とサービス自動起動有効化。
  ```
  $ systemctl start zabbix-agent
  $ systemctl enable zabbix-agent
  ```

## Zabbixへdev.menta.meのホスト追加、監視アイテム・トリガーを設定

1. Zabbixの設定から`dev.menta.me`のホストを追加。
   以下の内容をホストに設定。
   ```
   ホスト名: dev.menta.me
   グループ名: dev.menta.me
   インターフェース: 192.168.56.15
   ポート: 10050
   ```

2. テンプレートで`Template_dev.menta.me`を追加。
   テンプレートに監視アイテムとトリガーを追加する。

3. `Template_dev.menta.me`に以下の監視アイテムを追加。
   以下、アイテムの名前/キーを抜粋して記述。
   ```
   CPUロードアベレージ(1m): system.cpu.load[,avg1]
   CPUロードアベレージ(5m): system.cpu.load[,avg5]
   CPUロードアベレージ(15m): system.cpu.load[,avg15]
   CPU使用率(system): system.cpu.util[,system]
   CPU使用率(user): system.cpu.util[,user]
   ICMP Ping : icmpping
   プロセス監視(mysql): proc.num[mysqld]
   プロセス監視(nginx): proc.num[nginx]
   メモリ使用量: vm.memory.size[used]
   メモリ空き容量: vm.memory.size[free]
   メモリ総容量: vm.memory.size[total]
   メモリ使用率: vm.memory.size[pused]
   ```

4. `Template_dev.menta.me`に以下の監視トリガーを追加。
   以下、トリガーの名前/条件式を抜粋して記述。
   ```
   CPUロードアベレージの閾値超え: {Template_dev.menta.me:system.cpu.load[,avg1].last()}>2
   CPU使用率 (system) High: {Template_dev.menta.me:system.cpu.util[,system].last(#3)}>=90
   CPU使用率 (user) High:  {Template_dev.menta.me:system.cpu.util[,user].last(#3)}>=90
   Ping応答なし: {Template_dev.menta.me:icmpping.count(#5,0,eq)}>=4
   プロセス監視(mysqld) 停止: {Template_dev.menta.me:proc.num[mysqld].last()}=0
   プロセス監視(nginx) 停止: {Template_dev.menta.me:proc.num[nginx].last()}=0
   メモリ容量の閾値超え: {Template_dev.menta.me:vm.memory.size[pused].last()}>95
   ```

5. `Template_dev.menta.me`に以下のディスカバリルールを追加。
    DiskDiscoveryとして、ディスク容量監視のディスカバリルールを設定。
    ```
    名前: DiskDiscovery
    タイプ: Zabbixエージェント
    キー: vfs.fs.discovery
    監視間隔: 5m
    ```

    アイテムのプロトタイプを追加。
    以下、名前とキーを抜粋して記述。
    ```
    ディスクの使用率 {#FSNAME}: vfs.fs.size[{#FSNAME},pused]
    ディスクの使用量 {#FSNAME}: vfs.fs.size[{#FSNAME},used]
    ディスクの未使用率 {#FSNAME}: vfs.fs.size[{#FSNAME},pfree]
    ディスクの空き容量 {#FSNAME}: vfs.fs.size[{#FSNAME},free]
    ディスクの総容量 {#FSNAME}: vfs.fs.size[{#FSNAME},total]
    ```

    トリガーのプロトタイプを追加。
    名前と条件式を以下のように設定。
    `{#FSNAME} のディスク使用率: {Template_dev.menta.me:vfs.fs.size[{#FSNAME},pused].last()}>=95`

6. zabbixダッシュボードで監視出来ていることを確認。
   ![zabbix監視](https://user-images.githubusercontent.com/63705498/183272267-e04d37d5-9776-438c-ae4c-28068f47c0d3.png)


## Slackと連携し、アラートを通知する。
1. Slack APIにアクセスし、`Zabbix_Slack_Notification`アプリを作成。

2. パーミッションを`calls:write`に設定。

3. 個人のSlackチャンネルと連携し、アプリを追加。

4. Zabbix Server側で、メディアタイプからSlackの通知を設定。
   以下のメディアタイプの内容を設定
   ```
   名前: Slack
   bot_token: Slack APIで発行されたトークン
   zabbix_url: http://192.168.56.16/zabbix/
   ```

5. メディアタイプのテスト。
   以下内容を設定し、slack通知のテストを実施。
   ```
   channel: [slackのチャンネル]
   event_id: 1
   ```

6. テスト結果を確認。
   <img width="916" alt="スクリーンショット 2022-08-07 12 52 18" src="https://user-images.githubusercontent.com/63705498/183274478-30bf8308-4f68-476e-bf39-5ef59d7db0f4.png">


7. zabbix側でユーザーを作成。
   ユーザーの内容
   ```
   エイリアス: Slack_Notification
   グループ: Zabbix_administrator
   パスワード: パスワード
   ```

   タイプの内容
   ```
   タイプ: Slack
   送信先: Zabbix_notification
   ```

   権限の内容
   ```
   ユーザーの種類: 特権管理者
   ```

8. アクションの設定
  Zabbix側で名前と実行条件を設定。
  ```
  名前: Action_Slack_Notification
  実行条件: A メンテナンス期間外
            B トリガーの深刻度 以上 警告
  ```

  実行内容から、`Slack_Notification`を`Slack`メディアのみに使用するように設定。

9. dev.menta.meのディスクの容量を増やしてslackにアラート通知。
   `dd if=/dev/zero of=/dummy bs=10000M count=1000`を入力し、ディスク容量を閾値超えまで増加。

10. 個人のSlackでアラートを確認。
    ![スクリーンショット 2022-08-05 0 09 17](https://user-images.githubusercontent.com/63705498/183274460-cf89a250-53d5-4d98-997a-f98f80bea29f.png)


## ZabbixとGrafanaを連携
Zabbix側でGrafanaをインストール作業を実施。

1. `$ yum install fontconfig`で、システムのフォントを管理するfontconfigをインストール。

2. `$ wget https://dl.grafana.com/enterprise/release/grafana-enterprise-9.0.6-1.x86_64.rpm`で、Grafanaをインストール。

3. `$ systemctl enable grafana-server`で、自動起動設定。

4. `$ systemctl start grafana-server`で、grafana-serverを起動。

5. alexanderzobnin-zabbix-appをインストール。
   `$ grafana-cli plugins install alexanderzobnin-zabbix-app`

7. Grafana連携用にZabbix側にユーザー作成。
   Zabbixのユーザーの作成から、以下の内容でユーザー作成。
   ```
   エイリアス: garafana
   名: grafana
   グループ: Guests
   ```

   ユーザーグループの設定からグループ`Guests`の権限を指定。
   ```
   ホストグループ: dev.menta.me
   権限: 表示のみ
   ```

8. http://192.168.56.16:3000で、Grafanaにアクセス。

9. Data SourcesからZabbixを設定。
   以下の内容で設定し、Save & Testを実施。
   ```
   Name: Zabbix
   Url: http://192.168.56.16/api_jsonrpc.php
   Username: grafana
   Password: パスワード
   Trends: Enable
   ```

10. Dashbordのレイアウトを編集し、正常に表示されていることを確認。
    ![スクリーンショット 2022-08-06 19 44 24](https://user-images.githubusercontent.com/63705498/183274508-752907fc-eefb-4369-ad0b-87dca844b821.png)
