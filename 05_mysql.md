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

Ho to do SCP from VPS to local

```
scp ubuntu@ec2-54-95-78-211.ap-northeast-1.compute.amazonaws.com:~/for-dump/mysqld.cnf ~
```

MySQLの状況・状態 確認 Stat Confirmantion

```
> show variables like 'version';
> show status;
> show databases;
> use isubata;
> show tables;
```

起動ログの確認

```
sudo journalctl -u mysql
```

# 以降 実際にやること

# systemctlが使えるように設定する

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

# ダンプ => ローカル転送(バックアップ) => (ローカルにDBリストア)

[注意] 下記を置き換える
isubata <= DB名
isucon <= User名

ダンプする

```
$ mysqldump -u isucon isubata | gzip > isubata.dump.gz
```

ローカルPC上でローカルにダンプしたものを転送する

```
$ scp ubuntu@ec2-54-95-78-211.ap-northeast-1.compute.amazonaws.com:~/isubata.dump.gz ~
```

(ローカルにMySQLのDBを立てる)

```
$ mysql -uroot
mysql> create database isubata;
```

(ローカル上でリストアする)

```
gzcat isubata.dump.gz | mysql -u root isubata
```

# Max Connection

Too many connections error が出る時は、max connections を大きくする。 ただし、5_os にしたがって「OS 側での connection, open file limit の設定の更新」をする必要がある（OS側で制限されてると、mysql 側ではその制限の範囲内でしか設定を変えれない）

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
echo 'max_connections=10000' >> /etc/mysql/my.cnf
```

設定の確認

```
> SHOW variables LIKE "%max_connections%";
> SHOW variables LIKE "%open_files_limit%";
```

# スローログ

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

## innodb buffer

参考
https://gist.github.com/south37/d4a5a8158f49e067237c17d13ecab12a#innodb-buffer

設定する

```bash
echo 'innodb_buffer_pool_size = 1GB' >> /etc/mysql/my.cnf
echo 'innodb_flush_log_at_trx_commit = 2' >> /etc/mysql/my.cnf
echo 'innodb_flush_method = O_DIRECT' >> /etc/mysql/my.cnf
```

## kamipo TRADITIONAL

```bash
echo 'sql_mode = TRADITIONAL,NO_AUTO_VALUE_ON_ZERO,ONLY_FULL_GROUP_BY' >> /etc/mysql/my.cnf
```