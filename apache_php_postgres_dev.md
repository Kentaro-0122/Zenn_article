---
title: "Apache+PHP＋PostgreSQL 環境構築手順 on AWS EC2"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Apache","PHP","PostgreSQL","Linux"]
published: True
---
# Apache+PHP＋PostgreSQL 環境構築手順 on AWS EC2
  
## 概要

環境構築において、ソフトのインストールまでの説明をされている記事は多く見られますが、その後のセットアップまでフォローしている記事は少ないと感じ、自分で作った環境をもとに、構築手順を執筆しました。
ApacheのVirtualHostやPHPのバージョン変更など、一部の方には不要な設定もあるかと思うので、必要な部分だけ利用していただけますと幸いです。
また、間違いなどございましたら可能であればご指摘をいただきたいです。

## 環境

●AWS EC2
インスタンスタイプ:t3.small
AMI:Amazon linux2

●EC2上の環境
Apache:2.4
PHP:8.1
PostgreSQL:12

## 手順

### PostgreSQLのインストールから起動

#### インストールから常時起動設定まで

```bash
# Postgresql12のインストール
$ sudo amazon-linux-extras install postgresql12

# サーバーのインストール
$ sudo yum -y install postgresql-server

# postgresqlを初期状態にする
$ sudo postgresql-setup initdb

# postgresqlの起動
$ sudo systemctl start postgresql

# 常時起動設定
$ sudo systemctl enable postgresql
```

#### 起動確認

```bash

# ステータス確認
$ systemctl status postgresql

● postgresql.service - PostgreSQL database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-02-17 05:39:44 UTC; 2 days ago
  Process: 29703 ExecStartPre=/usr/libexec/postgresql-check-db-dir %N (code=exited, status=0/SUCCESS)
```

active(running)になっていることを確認。

※ec2-userで実行する場合はインストール、起動・停止に「sudo」コマンドが必要。

### PostgreSQLの操作

```bash

# ユーザーをpostgresに変更
$ sudo su - postgres

# postgres(スーパーユーザー)にユーザー変更
-bash4.2$ psql

# データベースの作成
postgres=♯ create database testDB

# ユーザーからの離脱
postgres=♯ \q
```

```bash

# データベースのリスト確認
-bash4.2$ psql -l

List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 testdb　  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
    +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
```

testDBの作成を確認。

### テストテーブルを作成し、データのインサートも行う

```bash

# testdbに移動
postgres=♯ \c testdb

# テーブル作成のsqlを記載
testdb=♯ CREATE TABLE article (id integer, title text, contents text);
CREATE TABLE

# データのインサート
testdb=♯ INSERT INTO article values (1, 'first article', 'first contents');

# データの確認
testdb=♯ SELECT * FROM article;

# 出力結果
 id |     title     |    contents
----+---------------+----------------
  1 | first article | first contents
(1 row)
```

参照:[【PostgreSQL】MacでPostgreSQLデータベースの環境をつくろう！](https://tektektech.com/psql-mac-environment/#postgresSQL)

### PostgreSQLのconfファイル編集

運用には下記2種類のconfファイルを編集する必要がある。
編集の際、ユーザーはpostgresで行う

- postgresql.conf

```bash

# ユーザーpostgresでdata内のpostgresql.confをvimで編集
-bash4.2$ cd data
-bash4.2$ vi postgresql.conf
```

```postgresql.conf

54 #------------------------------------------------------------------------------
55 # CONNECTIONS AND AUTHENTICATION
56 #------------------------------------------------------------------------------
57 
58 # - Connection Settings -
59 
60 #listen_addresses = 'localhost'         # what IP address(es) to listen on;
61 listen_addresses = '*'                                  # comma-separated list of addresses;
62                                         # defaults to 'localhost'; use '*' for all
63                                         # (change requires restart)
64 #port = 5432                            # (change requires restart)
65 max_connections = 100                   # (change requires restart)
66 #superuser_reserved_connections = 3     # (change requires restart)
67 #unix_socket_directories = '/var/run/postgresql, /tmp'  # comma-separated list of directories
68                                         # (change requires restart)
69 #unix_socket_group = ''                 # (change requires restart)
70 #unix_socket_permissions = 0777         # begin with 0 to use octal notation
71                                         # (change requires restart)
72 #bonjour = off                          # advertise server via Bonjour
73                                         # (change requires restart)
74 #bonjour_name = ''                      # defaults to the computer name
75                                         # (change requires restart) 

~~~

89 # - Authentication -
90 
91 #authentication_timeout = 1min          # 1s-600s
92 password_encryption = scram-sha-256             # md5 or scram-sha-256
93 #db_user_namespace = off

```

61行目に 「listen_addresses = '*'」追加
92行目のpassword_encryptionを「md5」から「scram-sha-256」に変更

パスワードの認証方法でscram-sha-256を使用するために記載。
現在最もセキュアな認証方法である。

他端末から接続するための設定として「listen_addresses = '*'」 を該当箇所に記載。
これはサーバー側の”接続を受け付ける”IPアドレスを記載する箇所である。

参考記事

- 認証方式について
[PostgreSQL 10.5文書 20.3. 認証方式](https://www.postgresql.jp/document/10/html/auth-methods.html)

上記完了のタイミングでpostgresを再起動し、パスワードを設定する。
→先にpg_hba.confで設定変更をしてしまうとmd5の時に設定したパスワードでの暗号化が行われてしまうため、正しいパスワードを入力しても認証されない。

```bash
# confファイルの記載を変更したため、一度postgresを再実行する。
$ sudo systemctl stop postgresql
$ sudo systemctl start postgresql

# ---postgresql.confの内容が反映された---

# DBにpostgresで接続
$ sudo su - postgres
-bash4.2$ psql

# postgresのパスワードを変更する(パスワードはシングルクォートで括るのが必須)
postgres=♯ ALTER ROLE postgres WITH PASSWORD ['新しいパスワード'];
ALTER ROLE
```

- pg_hba.conf

```pg_hba.conf
# dataディレクトリに移動し、pg_hba.confを編集
-bash4.2$ vi pg_hba.comf
```

```pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     scram-sha-256
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
host    all             all             0.0.0.0/0               scram-sha-256
#local   replication     all                                     peer
#host    replication     all             127.0.0.1/32            ident
#host    replication     all             ::1/128                 ident

```

上記内容に変更する。

localはUnixドメインソケット接続方式であり、このEC2上(local)から接続する時の認証方式の設定。
hostはTCP/IPによる接続方式であり、別端末からの接続に用いられる。

上記完了後、postgresqlの再起動を行う。

参考記事

- confファイルの編集について
[PostgreSQLへ他の端末から接続するための設定](https://www.projectgroup.info/documents/PostgreSQL/POS_000007.html)

### postgreSQLへのログインテスト

次回ログイン時にpsqlコマンドを実行するとパスワードが求められるため、先ほど設定したパスワードを入力する。
ログインできたらscram-sha-256での認証成功となる。

DBに入り、postgresのパスワード変更とユーザー作成を行う。

### PostgreSQLのスキーマ、ユーザー作成と編集権限付与

```bash
# スキーマの作成
testdb=♯ CREATE SCHEMA [スキーマ名];

# ログイン可能ユーザーを作成
testdb=♯ \c postgres
postgres=♯ CREATE ROLE [ロール名] WITH LOGIN PASSWORD ['パスワード'];

# スキーマログイン権限付与
testdb=♯ GRANT USAGE on SCHEMA [スキーマ名] to [ロール名];

# スキーマ所有者の変更
testdb=♯ ALTER SCHEMA [スキーマ名] OWNER TO [ロール名];

# スキーマの全テーブル操作権限をロールに付与
testdb=♯ GRANT ALL on ALL TABLES in SCHEMA [スキーマ名] to [ロール名];

# publicスキーマの全テーブル操作権限をロールに付与
testdb=♯ GRANT ALL on ALL TABLES in SCHEMA public to [ロール名];

# 権限確認
testdb=♯ \dp

# 出力結果
                                    Access privileges
 Schema |   Name    | Type  |      Access privileges      | Column privileges | Policies
--------+-----------+-------+-----------------------------+-------------------+----------
 public | article   | table | postgres=arwdDxt/postgres  +|                   |
        |           |       | [ロール名]=arwdDxt/postgres+ |                   |
 public | m_product | table |                             |                   |
 public | m_shop    | table |                             |                   |
 public | m_syain   | table | 
```

ロール名にarwdDxtの記載があり、全権限が付与されている。
それぞれの意味は下記の通りである。
    a: INSERT(add)
    r: SELECT(read)
    w: UPDATE(write)
    d: DELETE
    D: TRUNCATE
    x: REFERENCES
    t: TRIGGER

参考記事
[PostgreSQLの権限系操作まとめ](https://katsusand.dev/posts/postgresql-auth/)

### Apacheのインストール

```bash

# httpdのインストール
sudo yum install -y httpd
```

### VirtualHost使用のためhttpd.confを編集

こちらは一つのサーバーで2つ以上のWebサイトを運営する際に必要な設定であり、必須ではない。

```bash

# /etc/httpd/conf/にあるhttpd.confを編集する
$ sudo vi httpd.conf

# 'Main' server configuration
#
# The directives in this section set up the values used by the 'main'
# server, which responds to any requests that aren't handled by a
# <VirtualHost> definition.  These values also provide defaults for
# any <VirtualHost> containers you may define later in the file.
#
# All of these directives may appear inside <VirtualHost> containers,
# in which case these default settings will be overridden for the
# virtual host being defined.
#

## ↓↓↓　追加　↓↓↓
<VirtualHost *:80>
        ServerName      [URL]
      anri.jp
        DocumentRoot    /var/www/html
</VirtualHost>
<VirtualHost *:80>
        ServerName      [URL]
      anri.jp
        DocumentRoot    /var/www/vhost/[システム名]
</VirtualHost>
```

SeverNameが記載のアドレスでアクセスした場合、該当のDocumentRootを参照するという記述。
これで複数システムが同じIPでも分別可能。

参考記事
[【Apache】VirtualHostを使ってみよう](https://qiita.com/erik_t/items/63f079826a60aaa642d3)

### DocumentRootのディレクトリ作成、所有者・グループの変更

```bash

# /var/www/にvhostディレクトリを作成し、その中にcust_reqディレクトリを作成する
$ sudo mkdir vhost
$ cd vhost
$ sudo mkdir cust_req

# vscodeから編集できるようにするために、www以下の所有者・グループを一括変更する ※例としてec2-user
$ sudo chown -R ec2-user:ec2-user /var/www
```

### PHPのインストール

amazon-linuxのリポジトリからはPHP8.0以上しかインストールできない(2023/03/08現在)ため、それ以下のバージョンを使用したい場合は、次章をご参照ください。

```bash
$ sudo amazon-linux-extras install php8.1

# postgresに接続するためのパッケージをインストール
$ sudo yum install php-pgsql
$ sudo yum install php-mbstring

# インストールされているパッケージを確認
$ yum list installed | grep php

php-cli.x86_64                  8.1.14-1.amzn2                @amzn2extra-php8.1
php-common.x86_64               8.1.14-1.amzn2                @amzn2extra-php8.1
php-fpm.x86_64                  8.1.14-1.amzn2                @amzn2extra-php8.1
php-mbstring.x86_64             8.1.14-1.amzn2                @amzn2extra-php8.1
php-mysqlnd.x86_64              8.1.14-1.amzn2                @amzn2extra-php8.1
php-pdo.x86_64                  8.1.14-1.amzn2                @amzn2extra-php8.1
php-pgsql.x86_64                8.1.14-1.amzn2                @amzn2extra-php8.1
```

amazon-linux-extrasのリポジトリからPHPをインストールすると、デフォルトでPHP-fpmを使用する設定になっているため、注意。

### PHPのバージョン変更

amazon-linux-extrasには最新バージョンのPHPしかなかったので、古いバージョンを使用するためremiリポジトリをダウンロードし、PHPのインストールを行った。

```bash
# epelのインストール
$ sudo amazon-linux-extras install -y epel

# remiリポジトリの取得
$ sudo rpm -ivh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
$ sudo rpm --import http://rpms.famillecollet.com/RPM-GPG-KEY-remi
```

パッケージ管理をしているディレクトリに移動し、レポジトリの使用可否設定を変更する。

/etc/yum.repos.dにあるamzn2-core.repoとamzn2-extras.repoの[amzn2extra-php8.1-source]の箇所のenabledを0に変更する。

また、epel.repoとremi-php73.repoのenabledを1に変更する。

```bash
# phpのインストール
$ sudo yum -y install php --enablerepo=remi-php73 php-pgsql php-mbstring

# バージョン確認
$ php -v

PHP 7.3.33 (cli) (built: Feb 14 2023 14:41:46) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.3.33, Copyright (c) 1998-2018 Zend Technologies
```

### php.iniの編集

```bash
# /etcにあるphp.iniを編集
$ sudo vi php.ini

883 ;;;;;;;;;;;;;;;;;;;;;;
884 ; Dynamic Extensions ;
885 ;;;;;;;;;;;;;;;;;;;;;;
886
887 ; If you wish to have an extension loaded automatically, use the following
888 ; syntax:
889 ;
890 ;   extension=modulename
891 ;
892 ; For example:
893 ;
894 ;   extension=mysqli
895 ;
896 ; When the extension library to load is not located in the default extension
897 ; directory, You may specify an absolute path to the library file:
898 ;
899 ;   extension=/path/to/extension/mysqli.so
900 ;
901 ; Note : The syntax used in previous PHP versions ('extension=<ext>.so' and
902 ; 'extension='php_<ext>.dll') is supported for legacy reasons and may be
903 ; deprecated in a future PHP major version. So, when it is possible, please
904 ; move to the new ('extension=<ext>) syntax.
905 
906 extension=pdo_pgsql.so

```

906行目に「extension=pdo_pgsql.so」の記述を追加。
phpでpgsqlを使えるようにする記述である。
pdo_pgsql.soは /usr/lib64/php/modulesに格納されている。

### 動作確認

phpinfo.php,sql.phpの2種類を作成し動作確認を行う。
これらのファイルを /var/www/html に格納する。

- phpinfo.php

```bash
<?php phpinfo (); ?>
```

phpのインフォメーションが表示されていれば成功。

- sql.php

```bash
<?php

# DBの情報の記載　※envファイルに記載で読み込む方が良いかも
$DBHOST = ["サーバーIPアドレス"];
$DBPORT = "5432"; //デフォルトは5432
$DBNAME = "testdb";
$DBUSER = "postgres";
$DBPASS = "postgres";

try{
  # DB接続
  $dbh = new PDO("pgsql:host=$DBHOST;port=$DBPORT;dbname=$DBNAME;user=$DBUSER;password=$DBPASS");
  print("接続成功".'<br>');

  # SQL作成
  $sql = 'select * from public.article';

  # SQL実行
  foreach ($dbh->query($sql) as $row) {
      # 指定Columnを一覧表示
      print($row['id'].'<br>');
  }

}catch(PDOException $e){
  print("接続失敗".'<br>');
  print($e.'<br>');
  die();
}
# データベースへの接続を閉じる
$dbh = null;
?>
```

接続成功の文字とデータが表示されていれば成功。

参考記事
[【PostgreSQL】【PHP】接続方法](https://qiita.com/asami___t/items/ac159c67effd34d44ab9)

## まとめ

以上でフロントからPHPを用いて、PostgreSQLのデータを利用することができます。
不具合やご指摘がございましたらコメントいただけますと幸いです。
