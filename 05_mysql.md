# 当日実際にやること

## [ ] 本ファイルの文字置換

置換元 => 置換先
----------------
ubuntu => 当日DBサーバへログインできるユーザ名
isubata => 当日DB名
52.194.230.212 => 当日DBサーバIPアドレス


## [ ] スキーマ&サイズ&インデックス確認

1. スキーマ確認

```mysql
> show variables like 'version';
> show status;
> show databases;
> use isubata;
> show tables;

# スキーマ確認
> desc テーブル名;
.
.
```

2. テーブルレコード数を確認

```mysql
> select table_name, table_rows from information_schema.TABLES where table_schema = 'isubata';
```

3. インデックスを確認

```mysql
> use isubata;
> show index from テーブル名;
.
.
```

## [ ] MySQL設定ファイルのバックアップ

ローカルPCにて

```bash
scp ubuntu@52.194.230.212:/etc/mysql/my.cnf ~
scp ubuntu@52.194.230.212:/etc/security/limits.conf ~
scp ubuntu@52.194.230.212:/etc/pam.d/common-session ~
```

## [ ] ダンプ => ローカル転送(バックアップ) => (ローカルにDBリストア)

1. ダンプする

```bash
mysqldump -u isucon -p isubata | gzip > db.dump.gz
```

2. ローカルPC上でローカルにダンプしたものを転送する

```bash
scp ubuntu@52.194.230.212:~/db.dump.gz ~
```

3. (ローカルにMySQLのDBを立てる)

## [ ] (Centosでは必要か当日確認する)systemctlが使えるように設定する)

MySQLを自動起動を有効化する

```bash
sudo systemctl enable mysqld
```

## [ ] my.cnfの配置

`/etc/my.cnf` または `/etc/mysql/my.cnf` に以下の設定を持つようにする

```
[mysqld]
# For slow query
slow_query_log=ON
long_query_time = 0
slow_query_log_file = /tmp/mysql-slow.sql

# InnoDB setting : ref https://gist.github.com/south37/d4a5a8158f49e067237c17d13ecab12a#innodb-buffer
innodb_buffer_pool_size = 1GB
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT

# kamipo TRADITIONAL
sql_mode = TRADITIONAL,NO_AUTO_VALUE_ON_ZERO,ONLY_FULL_GROUP_BY

# Max Connection
max_connections=10000
```

MySQL再起動

```bash
sudo systemctl restart mysqld
```

## [ ] スローログ設定の確認

```mysql
# MySQLへログイン
> show variables like 'slow%';
```

## [ ] Max Connection 設定 (OS側の設定も変更するためリスクがあり。なので他と分けて最後にした。)

> Too many connections error が出る時は、max connections を大きくする。 ただし、5_os にしたがって「OS 側での connection, open file limit の設定の更新」をする必要がある（OS側で制限されてると、mysql 側ではその制限の範囲内でしか設定を変えれない）

OS側の設定

```bash
# For max connection for MySQL
echo '*    soft nofile 65535' >> /etc/security/limits.conf
echo '*    hard nofile 65535' >> /etc/security/limits.conf
echo 'root soft nofile 65535' >> /etc/security/limits.conf
echo 'root hard nofile 65535' >> /etc/security/limits.conf
echo 'session    required     pam_limits.so' >> /etc/pam.d/common-session
echo 'session    required     pam_limits.so' >> /etc/pam.d/common-nonsession
```

systemctlとMySQLをリロードする

```bash
sudo systemctl daemon-reload
sudo systemctl restart mysqld
```

# パフォーマンスをあげるためにやること

## [ ] 必要そうなインデックスを貼る

ポイント: 主キーになかったら貼ろう。クエリのキーに使っていたらやろう。

ISUCON7の場合

messageテーブルのchannel_idカラム
userテーブルのnameカラム
imageテーブルのnameカラム

```mysql
> ALTER TABLE message ADD INDEX channel_id(channel_id);
```

# [ ] SQL解析

スローログに入っているデータを集計。mysqldumpslowというMySQL標準ツールを使う

```bash
mysqldumpslow -s t /tmp/mysql-slow.sql > ikedamp.log
```


# Common

MySQLへのログイン方法 Login

```bash
mysql -u isucon -p
Enter password: isucon
```

Restart MySQL

```bash
sudo systemctl restart mysql
```

設定ファイル優先度確認

```bash
mysql --help | grep my.cnf
```

Ho to do SCP from VPS to local @local machine

```bash
scp ubuntu@52.194.230.212:~/for-dump/mysqld.cnf ~
```

Status Confirmation

```mysql
> SHOW variables LIKE "%max_connections%";
> SHOW variables LIKE "%open_files_limit%";
```

インデックスの貼り方

```mysql
> ALTER TABLE テーブル名 ADD INDEX インデックス名(カラム名);
```

起動ログの確認

```bash
sudo journalctl -u mysql
```