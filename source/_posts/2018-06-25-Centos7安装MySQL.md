---
title: Centos7安装MySQL
tags: [Linux, MySQL]
categories: 数据库
abbrlink: df149a46
date: 2018-06-25 10:51:26
---

总所周知，MySQL 被 Oracle 收购后，CentOS 的镜像仓库中提供的默认的数据库也变为了 MariaDB，如果想了解 MariaDB 和 CentOS 的区别，可以参考[官网介绍](https://link.jianshu.com/?t=https://mariadb.com/kb/en/mariadb/mariadb-vs-mysql-compatibility/)，想用 MariaDB 的同学可以参考 [MariaDB 安装指南](https://link.jianshu.com/?t=https://www.linode.com/docs/databases/mariadb/how-to-install-mariadb-on-centos-7)

言归正传，在 CentOS 上安装 MySQL 差不多有四个步骤

## 添加 MySQL YUM 源

根据自己的操作系统选择合适的[安装源](https://link.jianshu.com/?t=http://dev.mysql.com/downloads/repo/yum/)，和其他公司一样，总会让大家注册账号获取更新，注意是 Oracle 的账号，如果不想注册，下方有直接[下载的地址](https://link.jianshu.com/?t=https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm)，下载之后通过 `rpm -Uvh` 安装。

```shell
$wget 'https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm'
$sudo rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
$yum repolist all | grep mysql
mysql-cluster-7.5-community/x86_64 MySQL Cluster 7.5 Community   disabled
mysql-cluster-7.5-community-source MySQL Cluster 7.5 Community - disabled
mysql-cluster-7.6-community/x86_64 MySQL Cluster 7.6 Community   disabled
mysql-cluster-7.6-community-source MySQL Cluster 7.6 Community - disabled
mysql-connectors-community/x86_64  MySQL Connectors Community    enabled:     51
mysql-connectors-community-source  MySQL Connectors Community -  disabled
mysql-tools-community/x86_64       MySQL Tools Community         enabled:     63
mysql-tools-community-source       MySQL Tools Community - Sourc disabled
mysql-tools-preview/x86_64         MySQL Tools Preview           disabled
mysql-tools-preview-source         MySQL Tools Preview - Source  disabled
mysql55-community/x86_64           MySQL 5.5 Community Server    disabled
mysql55-community-source           MySQL 5.5 Community Server -  disabled
mysql56-community/x86_64           MySQL 5.6 Community Server    disabled
mysql56-community-source           MySQL 5.6 Community Server -  disabled
mysql57-community/x86_64           MySQL 5.7 Community Server    enabled:    267
mysql57-community-source           MySQL 5.7 Community Server -  disabled
mysql80-community/x86_64           MySQL 8.0 Community Server    disabled
mysql80-community-source           MySQL 8.0 Community Server -  disabled
```

可以看出该安装源包含了MySQL5.5，5.6，5.7和8.0四个版本，默认安装的是MySQL5.7版本。

<!-- more -->

## 选择安装版本

如果想安装最新版本的，直接使用 yum 命令即可

```shell
$sudo yum install mysql-community-server
```

如果想要安装 5.6 版本的，有2个方法。命令行支持 `yum-config-manager` 命令的话，可以使用如下命令：

```shell
$ sudo dnf config-manager --disable mysql57-community
$ sudo dnf config-manager --enable mysql56-community
$yum repolist all | grep mysql
mysql-cluster-7.5-community/x86_64 MySQL Cluster 7.5 Community   disabled
mysql-cluster-7.5-community-source MySQL Cluster 7.5 Community - disabled
mysql-cluster-7.6-community/x86_64 MySQL Cluster 7.6 Community   disabled
mysql-cluster-7.6-community-source MySQL Cluster 7.6 Community - disabled
mysql-connectors-community/x86_64  MySQL Connectors Community    enabled:     51
mysql-connectors-community-source  MySQL Connectors Community -  disabled
mysql-tools-community/x86_64       MySQL Tools Community         enabled:     63
mysql-tools-community-source       MySQL Tools Community - Sourc disabled
mysql-tools-preview/x86_64         MySQL Tools Preview           disabled
mysql-tools-preview-source         MySQL Tools Preview - Source  disabled
mysql55-community/x86_64           MySQL 5.5 Community Server    disabled
mysql55-community-source           MySQL 5.5 Community Server -  disabled
mysql56-community/x86_64           MySQL 5.6 Community Server    enabled
mysql56-community-source           MySQL 5.6 Community Server -  disabled
mysql57-community/x86_64           MySQL 5.7 Community Server    disabled:    267
mysql57-community-source           MySQL 5.7 Community Server -  disabled
mysql80-community/x86_64           MySQL 8.0 Community Server    disabled
mysql80-community-source           MySQL 8.0 Community Server -  disabled
```

或者直接修改 `/etc/yum.repos.d/mysql-community.repo` 这个文件

```shell
# Enable to use MySQL 5.6
[mysql56-community]
name=MySQL 5.6 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.6-community/el/7/$basearch/
enabled=1 #表示当前版本是安装
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

[mysql57-community]
name=MySQL 5.7 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/7/$basearch/
enabled=0 #默认这个是 1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
```

通过设置 `enabled` 来决定安装哪个版本。

设置好之后使用 `yum` 安装即可。

## 启动 MySQL 服务

启动命令很简单

```shell
$sudo service mysqld start 
$sudo systemctl start mysqld #CentOS 7
$sudo systemctl status mysqld #检查MySQL运行状态
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2018-06-24 21:23:53 EDT; 20min ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 2776 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 2758 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 2779 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─2779 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
```

说明已经正在运行中了。

为了加强安全性，MySQL5.7为root用户随机生成了一个密码，在error log中，关于error log的位置，如果安装的是RPM包，则默认是`/var/log/mysqld.log`，使用sudo grep 'temporary password' / /var/log/mysqld.log查看。

使用该密码登录数据库，然后修改密码：

```
$ mysql -uroot -p  #输入查看到的密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```

若设置的密码太简单，会报`ERROR 1819 (HY000): Your password does not satisfy the current policy requirements`错误。如果一定要设置简单密码，需要修改两个全局参数：

- 修改`validate_password_policy`参数的值

> mysql> set global validate_password_policy=0;

- 修改密码的长度

> mysql> set global validate_password_length=1;

再次修改密码就可以了。

## MySQL 5.6 的安全设置

由于 5.7 版本在安装的时候就设置好了，不需要额外设置，但是 5.6 版本建议从安全角度完善下，运行官方脚本即可

```
$ mysql_secure_installation
```

会提示设置5个关键位置

1. 设置 root 密码
2. 禁止 root 账号远程登录
3. 禁止匿名账号（anonymous）登录
4. 删除测试库
5. 是否确认修改

## 安装第三方组件

查看 yum 源中有哪些默认的组件：

```
$ yum --disablerepo=\* --enablerepo='mysql*-community*' list available
```

需要安装直接通过 `yum` 命令安装即可。

## 修改编码

在 `/etc/my.cnf` 中设置默认的编码

```
[client]
default-character-set = utf8

[mysqld]
default-storage-engine = INNODB
character-set-server = utf8
collation-server = utf8_general_ci #不区分大小写
collation-server =  utf8_bin #区分大小写
collation-server = utf8_unicode_ci #比 utf8_general_ci 更准确
```

## 创建数据库和用户

创建数据库

```
CREATE DATABASE <datebasename> CHARACTER SET utf8;
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
GRANT privileges ON databasename.tablename TO 'username'@'host';
SHOW GRANTS FOR 'username'@'host';
REVOKE privilege ON databasename.tablename FROM 'username'@'host';
DROP USER 'username'@'host';
```

其中

- username：你将创建的用户名
- host：指定该用户在哪个主机上可以登陆，如果是本地用户可用 localhost，如果想让该用户可以从任意远程主机登陆，可以使用通配符 %
- password：该用户的登陆密码，密码可以为空，如果为空则该用户可以不需要密码登陆服务器
- privileges：用户的操作权限，如 SELECT，INSERT，UPDATE 等，如果要授予所的权限则使用ALL
- databasename：数据库名
- tablename：表名，如果要授予该用户对所有数据库和表的相应操作权限则可用 * 表示，如 *.*

## 远程访问

### MySQL设置

- 允许root用户在任何地方进行远程登录，并具有所有库任何操作权限。登录MySQL并输入以下命令

```shell
mysql> mysql>GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'youpassword' WITH GRANT OPTION; #授权操作
mysql> flush privileges; #刷新权限
```

- 允许root用户在一个特定的IP进行远程登录，并具有所有库任何操作权限。登录MySQL并输入如下命令

```shell
mysql> GRANT ALL PRIVILEGES ON *.* TO root@"172.16.16.152" IDENTIFIED BY "youpassword" WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```

- 允许root用户在一个特定的IP进行远程登录，并具有所有库特定操作权限，具体操作如下：

```shell
mysql> GRANT select，insert，update，delete ON *.* TO root@"172.16.16.152" IDENTIFIED BY "youpassword";
mysql> FLUSH PRIVILEGES;
```

- 删除用户授权，需要使用REVOKE命令，具体命令格式为：

> REVOKE privileges ON 数据库[.表名] FROM user-name;

```shell
mysql> GRANT select，insert，update，delete ON TEST-DB TO test-user@"172.16.16.152" IDENTIFIED BY "youpassword"; # 授权操作
mysql> REVOKE all on TEST-DB from test-user; # 删除授权操作
mysql> FLUSH PRIVILEGES;
```

### 防火墙设置

开启3306端口

```shell
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
```

补充：操作防火墙命

> systemctl list-unit-files #查看服务 
>
> systemctl enable firewalld #允许开机启动
>
> systemctl stop firewalld #停止 
>
> systemctl disable firewalld #禁用开机启动
>
> firewall-cmd --state #查看默认防火墙状态（关闭后显示notrunning，开启后显示running）

### 远程连接测试

> mysql -u 用户名 -p密码 -h 服务器IP地址 -P 服务器端MySQL端口号 -D 数据库名

若正常连接，会进入数据库操作命令行界面。