---
bg: "tree.jpg"
layout: post
title:  "named"
crawlertitle: "named"
summary: "named"
date:   2019-03-25
author: snailqh
---
### 下载相关软件
```shell
wget https://repo.mysql.com//mysql57-community-release-el7-11.noarch.rpm
rpm -ivh mysql57-community-release-el7-11.noarch.rpm
yum install mysql mysql-devel mysql-server gcc cmake  -y


wget https://jaist.dl.sourceforge.net/project/mysql-bind/mysql-bind/mysql-bind-0.2%20src/mysql-bind.tar.gz
wget https://www.isc.org/downloads/file/bind-9-11-2/ -O  bind-9.11.2.tar.gz
tar zxvf bind-9.11.2.tar.gz
tar zxvf mysql-bind.tar.gz
```

### 安装
```shell
# 拷贝两个文件
cp mysql-bind/mysqldb.c bind-9.11.2/bin/named/
cp mysql-bind/mysqldb.h bind-9.11.2/bin/named/include/

# 获取本机信息
mysql_config --cflags
-I/usr/include/mysql -m64
 mysql_config --libs
-L/usr/lib64/mysql -lmysqlclient -lpthread -lm -lrt -ldl

vim bind-9.11.2/bin/named/Makefile.in #添加
DBDRIVER_OBJS = mysqldb.@O@
DBDRIVER_SRCS = mysqldb.c
DBDRIVER_INCLUDES = -I/usr/include/mysql -m64
DBDRIVER_LIBS = -L/usr/lib64/mysql -lmysqlclient -lpthread -lm -lrt -ldl


# vim bind-9.11.2/bin/named/main.c
/* #include "xxdb.h" */ 下添加
#include "mysqldb.h"
/* xxdb_init(); */下添加
mysqldb_init();
/* xxdb_clear(); */下添加
mysqldb_clear();

vim bind-9.11.2/bin/named/mysqldb.c
#include <named/mysqldb.h> #修改
#include <include/mysqldb.h>

#编译安装
./configure --prefix=/usr/local/named --enable-threads --without-openssl
make&make install
```

### 配置
```shell
#配置环境变量
echo "export PATH=${PATH}:/usr/local/named/sbin/:/usr/local/named/bin/" >> /etc/profile
source /etc/profile
#生成rndc-key 导入到named.conf
/usr/local/named/sbin/rndc-confgen  > /usr/local/named/etc/rndc.conf
tail -10 /usr/local/named/etc/rndc.conf | head -9 | sed s/#\//g > /usr/local/named/etc/named.conf
```
#### named.conf 修改 view方式
```shell
options {
        listen-on port 53    { 10.60.6.10;127.0.0.1; };
        directory "/usr/local/named/var/";
        recursion yes;
        allow-query { any; };
        blackhole { none; };
};
# 任何区域必须放在view中
acl c1 {
  10.120.0.0/16;
};
##匹配acl中的地址
view c1 {
        match-clients           { c1; };
        allow-query-cache       { any; };
        allow-recursion         { any; };
        allow-transfer          { none; };

    zone "test.info" {
            type master;
            notify yes;
            database "mysqldb named named 10.60.6.10 named named";##连接数据库信息
 };
};
## any 匹配任何解析
view any {
    match-clients { any; };
    recursion     yes;

    zone "." {
            type hint;
            file "named.ca";
    };
};
```
#### named.conf 不通过view
```shell
options {
        listen-on port 53    { 10.60.6.10;127.0.0.1; };
        directory "/usr/local/named/var/";
        recursion yes;
        forwarders { 223.5.5.5;223.6.6.6; };
        allow-query { any; };
        blackhole { none; };
};
zone "test.info" {
        type master;
        notify yes;
        database "mysqldb named test_info 10.60.6.10 named named";
};

zone "." IN {
        type hint;
        file "named.ca";
};
include "/usr/local/named/etc/named.rfc1912.zones";
```
### 其他
```shell
#下载root
wget -O /usr/local/named/etc/named.ca http://www.internic.net/domain/named.root
#重新加载conf
/usr/local/named/sbin/rndc reload
#查看log
tail -100 /var/log/messages
```

### MYSQL
```shell
systemctl start mysqld
grep pass /var/log/mysqld.log
mysql -uroot -p
mysql> set password = password ('passwd');
# 或者
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';

# 修改密码复杂度要求
mysql> show variables like "%password%";
+---------------------------------------+--------+
| Variable_name                         | Value  |
+---------------------------------------+--------+
| default_password_lifetime             | 0      |
| disconnect_on_expired_password        | ON     |
| log_builtin_as_identified_by_password | OFF    |
| mysql_native_password_proxy_users     | OFF    |
| old_passwords                         | 0      |
| report_password                       |        |
| sha256_password_proxy_users           | OFF    |
| validate_password_check_user_name     | OFF    |
| validate_password_dictionary_file     |        |
| validate_password_length              | 4      |
| validate_password_mixed_case_count    | 1      |
| validate_password_number_count        | 1      |
| validate_password_policy              | MEDIUM |
| validate_password_special_char_count  | 1      |
+---------------------------------------+--------+
set global validate_password_policy = LOW;
# 或者/etc/my.cnf配置文件中增加
[mysqld]
validate_password=off
# 查看
show plugins;
```
```shell
#创建相关数据库和表：
create database named;
use named;
use named;
CREATE TABLE test (
  name varchar(128) NOT NULL default '@',
  ttl int(11) NULL default '600',
  rdtype varchar(128) default 'A',
  rdata varchar(128) NOT NULL,
  PRIMARY KEY (`name`,`rdata`)
) ENGINE=MyISAM  DEFAULT CHARSET=utf8 AUTO_INCREMENT=1 ;

# 插入NS数据
INSERT INTO test VALUES ('test.com', 600, 'SOA', 'ns1.test.com. ns2.test.com. 2017122900 60 30 1814400 1080');
INSERT INTO test VALUES ('test.com', 600, 'NS', 'ns1.test.con');
INSERT INTO test VALUES ('test.com', 600, 'NS', 'ns2.test.com');
INSERT INTO test VALUES ('ns1.test.com', 600, 'A', '10.136.27.102');
INSERT INTO test VALUES ('ns2.test.com', 600, 'A', '10.136.27.103');
# 创建named用户并赋权
grant all privileges on named.* to named@'%' identified by "named";
ALTER USER 'named'@'%' PASSWORD EXPIRE NEVER;
flush privileges;
```
### 排错
```shell
#mysql登录报错“Access denied for user 'root'@'localhost' (using password: YES”
mysqld --user=mysql --skip-grant-tables --skip-networking
#设置root密码
update mysql.user set authentication_string=password('123qwe') where user='root'
flush privileges;
```
