# MySQL 8 GA Installation Operation document

## [ ] Install MySQL 8 into Centos7 

```
sudo rpm -ivh https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
sudo yum install mysql-community-server
```

## [ ] Set prepaired my.cnf onto /etc/my.cnf and Restart

```bash
# On local machine
scp ./mysql8/my.cnf ubuntu@52.194.230.212:~
```

```bash
# On server
sudo su
cp ./my.cnf /etc/my.cnf
```

## [ ] Start MySQL and Enable auto starting

```
sudo systemctl start mysqld
sudo systemctl enable mysqld
```

## [ ] Check root user pass

```
grep -e 'A temporary password is generated for root@localhost' /var/log/mysqld.log
```

## [ ] Create isucon user with pass, "IsuCon8_user"

```
mysql -u root -p
# Enter init pass
# Create DB
> CREATE DATABASE IF NOT EXISTS isubata;
# Create isucon user
> CREATE USER 'isucon'@'%' IDENTIFIED BY 'IsuCon8_user';
> GRANT ALL ON isubata.* TO 'isucon'@'%';
```

## [ ] Restore db data from dump file

```
# Run on local machine
scp ./db.dump.gz ubuntu@52.194.230.212:~
```

```
# Run on server
zcat db.dump.gz | mysql -u isucon -pIsuCon8_user -D isubata
```

## [ ] Connect setting from outside

firewalldを起動

```
sudo systemctl start firewalld
sudo systemctl status firewalld
```

MySQLをオープン

```
# Check current setting
sudo firewall-cmd --list-services --zone=public  --permanent
# Open service for MySQL
sudo firewall-cmd --add-service=mysql --zone=public --permanent
sudo firewall-cmd --reload
# Check mysql is open
sudo firewall-cmd --list-services --zone=public  --permanent
```

外部からのMySQLへの接続確認 

```
# From local machine
mysql -h52.194.230.212 -uisucon -pIsuCon8_user
```

# [ ] Max Connection Setting (OS側の設定変更のためリスクあり。なので他と分けて最後にした。)

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

```bash
sudo sudo
mkdir -p /usr/lib/systemd/system/mysql.service.d/
vi /usr/lib/systemd/system/mysql.service.d/limit_nofile.conf
# Add followings
[Service]
LimitNOFILE=4096
```

```bash
systemctl daemon-reload
systemctl restart mysqld
```

# Common

## login to mysql

```
mysql -u isucon -pIsuCon8_user
```

## Restart MySQL

```
sudo systemctl restart mysqld
```