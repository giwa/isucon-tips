# Common

MySQLへのログイン方法 Login

```
mysql -u isucon -p
Enter password: isucon
```

Restart MySQL

```
sudo systemctl restart mysql
```

設定ファイル優先度確認

```
mysql --help | grep my.cnf
```

Ho to do SCP from VPS to local @local machine

```
scp ubuntu@52.194.230.212:~/for-dump/mysqld.cnf ~
```

Status Confirmation

```
> SHOW variables LIKE "%max_connections%";
> SHOW variables LIKE "%open_files_limit%";
```

インデックスの貼り方

```
> ALTER TABLE テーブル名 ADD INDEX インデックス名(カラム名);
```

起動ログの確認

```
sudo journalctl -u mysql
```

# 以降 当日実際にやること

## [ ] 本ファイルの文字置換

置換元 => 置換先
----------------
ubuntu => 当日DBサーバへログインできるユーザ名
isubata => 当日DB名
テーブル名 => 当日テーブル名ら
52.194.230.212 => 当日DBサーバドメイン


## [ ] スキーマ&サイズ&インデックス確認

1. スキーマ確認

```
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

```
> select table_name, table_rows from information_schema.TABLES where table_schema = 'isubata';
```

3. インデックスを確認

```
> use isubata;
> show index from テーブル名;
.
.
```

## [ ] MySQL設定ファイルのバックアップ

ローカルPCにて

```
scp ubuntu@52.194.230.212:/etc/mysql/my.cnf ~
scp ubuntu@52.194.230.212:/etc/security/limits.conf ~
scp ubuntu@52.194.230.212:/etc/pam.d/common-session ~
```

## [ ] ダンプ => ローカル転送(バックアップ) => (ローカルにDBリストア)

1. ダンプする

```
$ mysqldump -u isucon -p isubata | gzip > isubata.dump.gz
```

2. ローカルPC上でローカルにダンプしたものを転送する

```
$ scp ubuntu@52.194.230.212:~/isubata.dump.gz ~
```

3. (ローカルにMySQLのDBを立てる)


## [ ] (Centosでは必要か当日確認する)systemctlが使えるように設定する)

(Ubuntuの場合)

```
$ cp /lib/systemd/system/mysql.service /etc/systemd/system/
$ echo 'LimitNOFILE=infinity' >> /etc/systemd/system/mysql.service
$ echo 'LimitMEMLOCK=infinity' >> /etc/systemd/system/mysql.service
```

systemctlをリロードする

```
$ sudo systemctl daemon-reload
```

MySQLを再起動する

```
$ sudo systemctl restart mysql
```

自動起動の有効化

```
$ sudo systemctl enable mysql
```

## [ ] Max Connection 設定

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

MySQL側の設定

```bash
echo '[mysqld]' >> /etc/mysql/my.cnf
echo 'max_connections=10000' >> /etc/mysql/my.cnf
```

## [ ] スローログ設定

設定する

```bash
echo 'slow_query_log=ON' >> /etc/mysql/my.cnf
echo 'long_query_time = 0' >> /etc/mysql/my.cnf
echo 'slow_query_log_file = /tmp/mysql-slow.sql' >> /etc/mysql/my.cnf
```

設定の確認

```
# MySQLへログイン
> show variables like 'slow%';
```

## [ ] innodb buffer 設定

参考
https://gist.github.com/south37/d4a5a8158f49e067237c17d13ecab12a#innodb-buffer

設定する

```bash
echo 'innodb_buffer_pool_size = 1GB' >> /etc/mysql/my.cnf
echo 'innodb_flush_log_at_trx_commit = 2' >> /etc/mysql/my.cnf
echo 'innodb_flush_method = O_DIRECT' >> /etc/mysql/my.cnf
```

## kamipo TRADITIONAL 設定

```bash
echo 'sql_mode = TRADITIONAL,NO_AUTO_VALUE_ON_ZERO,ONLY_FULL_GROUP_BY' >> /etc/mysql/my.cnf
```

# パフォーマンスをあげるためにやること

## [ ] 必要そうなインデックスを貼る

ポイント: 主キーになかったら貼ろう。クエリのキーに使っていたらやろう。

ISUCON7の場合

messageテーブルのchannel_idカラム
userテーブルのnameカラム
imageテーブルのnameカラム

```
> ALTER TABLE message ADD INDEX channel_id(channel_id);
```

# [ ] SQL解析

スローログに入っているデータを集計。mysqldumpslowというMySQL標準ツールを使う

```
mysqldumpslow -s t /tmp/mysql-slow.sql > ikedamp.log
```