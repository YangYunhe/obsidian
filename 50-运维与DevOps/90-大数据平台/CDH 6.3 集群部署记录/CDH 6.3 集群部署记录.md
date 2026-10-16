---
title: CDH 6.3 集群部署记录
date: 2026-10-16
tags:
  - 运维
  - 大数据
  - cdh
  - 部署
---

# CDH 6.3 集群部署记录

> 在 5 台 CentOS 7.9 节点上，用 Cloudera Manager 6.3.1 部署 CDH 6.3.2 最小非高可用集群（HDFS / YARN / HBase / Hive / Hue / Kafka / Spark / ZooKeeper）的完整步骤记录。

## 集群规划
服务器基础信息：

| **主机名** | **内网IP** | **内存** | **CPU** | **磁盘** | **操作系统** |
| --- | --- | --- | --- | --- | --- |
| cdh-01 | 10.0.0.11 | 16G | 4核 | 系统盘200G | CentOS-7.9 |
| cdh-02 | 10.0.0.12 | 16G | 4核 | 系统盘200G | CentOS-7.9 |
| cdh-03 | 10.0.0.13 | 16G | 4核 | 系统盘200G | CentOS-7.9 |
| cdh-04 | 10.0.0.14 | 16G | 4核 | 系统盘200G | CentOS-7.9 |
| cdh-05 | 10.0.0.15 | 16G | 4核 | 系统盘200G | CentOS-7.9 |

大数据服务角色规划：

一套极小规模的非高可用集群（5个节点），包含：HDFS、YARN、HBase、Hive、Hue、Kafka、Spark、Zookeeper，根据官方的建议，规划角色分布如下：

| | **Utility 和 Gateway 共用一个节点 cdh-01** | **Master节点1个 cdh-02** | **Worker节点3个 cdh03~cdh05** |
| --- | --- | --- | --- |
| Cloudera Manager | Cloudera Manager Server   Cloudera Manager Agent | Cloudera Manager Agent | Cloudera Manager Agent |
| Cloudera Manager Management Service | Activity Monitor   Alert Publisher   Event Server   Host Monitor   Service Monitor |  |  |
| HDFS | SecondaryNameNode   Balancer   Gateway | NameNode | DataNode |
| YARN | Gateway | ResourceManager   JobHistory Server | NodeManager |
| Hive | Hive Metastore Server   HiveServer2   Gateway | Gateway |  |
| Hue | Hue Server   Hue Load Balancer |  |  |
| HBase | Gateway | Master   HBase Thrift Server | RegionServer |
| Kafka | Gateway |  | Kafka Broker |
| ZooKeeper |  |  | Server |
| Spark | Gateway | History Server |  |
| Sqoop | Gateway |  |  |
| MySQL | Server   Client | | |

## 准备工作
### 主机名映射文件配置
所有节点同步

编辑 /etc/hosts 文件：

```bash
10.0.0.11       cdh-01
10.0.0.12       cdh-02
10.0.0.13       cdh-03
10.0.0.14       cdh-04
10.0.0.15       cdh-05
```

### 免密码登录配置
设置从 cdh-01 远程登录到其他 4 个机器免密钥：

```bash
# 1. 生成密钥文件
[root@cdh-01 ~]# ssh-keygen
Generating public/private rsa key pair.
# 直接Enter
Enter file in which to save the key (/root/.ssh/id_rsa): 
# 直接Enter
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:3EaTPqyz2psK8NIfzY9gGnhJdhdrwp0Esh3nc1dV/LM root@cdh-01
The key's randomart image is:
+---[RSA 2048]----+
|    . o .     .o+|
|     + =   . .  .|
|    . . = = .   .|
|     . + @ o   ..|
|  . o + S *     o|
|   B o * o .   E |
|  o B + =        |
|   o * + *       |
|    . +o*..      |
+----[SHA256]-----+

# 2. 分发密钥文件
# (1) 需要分发给本机才能实现免密码登录本机
[root@cdh-01 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@cdh-01
/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'cdh-01 (10.0.0.11)' can't be established.
ECDSA key fingerprint is SHA256:7xRPpJRipmwLbHsZLHatiloZHv20QUO/OOgrENd2pk8.
ECDSA key fingerprint is MD5:9f:fa:57:77:f2:a1:7e:78:5d:fa:a0:57:0f:11:e7:63.
# 输入yes
Are you sure you want to continue connecting (yes/no)? yes
/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
# 输入root用户的密码
root@cdh-01's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@cdh-01'"
and check to make sure that only the key(s) you wanted were added.

# (2) 分发给其他机器
[root@cdh-01 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@cdh-02
[root@cdh-01 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@cdh-03
[root@cdh-01 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@cdh-04
[root@cdh-01 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@cdh-05
```

### 禁用 IPV6
所有节点同步。

修改 /etc/sysctl.conf 文件，增加如下内容：

```bash
# 禁用IPV6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

修改 /etc/sysconfig/network 文件，增加如下内容：

```bash
NETWORKING_IPV6 = no
IPV6INIT = no
```

执行 sysctl -p 命令刷新配置。

### 禁用 SELINUX
所有节点同步。

永久禁用 SELINUX，编辑 /etc/selinux/config 文件：

```bash
# 原来为 SELINUX=enforcing
SELINUX=disabled
```

临时禁用 SELINUX：

```bash
setenforce 0
# 或者
setenforce Permissive
```

SELINUX 其他命令：

```bash
# 查看 SELINUX 状态
getenforce
# 临时开启 SELINUX
setenforce 1
setenforce Enforcing
# SELINUX 的三种状态
Disabled # 禁用
Permissive # 违反 SELinux 规则的行为只会记录到日志中，但仍可执行
Enforcing # 违反 SELinux 规则的行为将被阻止并记录到日志中
```

### 禁用防火墙
所有节点同步。

```bash
# 关闭防火墙服务
systemctl stop firewalld

# 禁用防火墙服务
systemctl disable firewalld

# 查看防火墙服务状态
systemctl status firewalld

# 查看防火墙服务状态信息
systemctl is-active firewalld # 检查服务是否正常运行
systemctl is-failed firewalld # 检查服务是否停止
systemctl is-enabled firewalld # 检查确认服务是否开机运行
```

### 禁用交换分区
所有节点同步。

```bash
# 首先临时禁用交换分区
echo 1 > /proc/sys/vm/swappiness
# 然后修改 /etc/sysctl.conf 文件
vm.swappiness = 1
# 然后执行 sysctl -p 命令刷新配置
```

### 禁用透明大页面压缩
所有节点同步。

```bash
# 首先执行以下两条命令
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 给此文件赋予执行权限
chmod +x /etc/rc.d/rc.local

# 在 rc.local 文件中增加如下内容
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### 集群时间同步
确保集群所有机器都在上海时区：

```bash
[root@cdh-01 ~]# timedatectl
      Local time: Sat 2024-01-27 14:23:54 CST
  Universal time: Sat 2024-01-27 06:23:54 UTC
        RTC time: Sat 2024-01-27 06:23:54
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: n/a
NTP synchronized: no
 RTC in local TZ: no
      DST active: n/a
```

所有节点安装 NTP 服务：

```bash
yum -y install ntp
```

选择 cdh-01 机器作为 NTP 服务端，修改 /etc/ntp.conf 文件：

```bash
# 注释掉以下 4 行
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst

# 添加下面 7 行（阿里云提供的 7 个 NTP 时间服务器）
server ntp1.aliyun.com iburst
server ntp2.aliyun.com iburst
server ntp3.aliyun.com iburst
server ntp4.aliyun.com iburst
server ntp5.aliyun.com iburst
server ntp6.aliyun.com iburst
server ntp7.aliyun.com iburst

# 添加下面 3 行，作用是：
# (1) 当外部时间不可⽤时，可使⽤本地硬件时间
server 127.127.1.0
fudge  127.127.1.0 stratum 10
# (2) 允许同网段的其他服务器从本机同步时间
restrict 10.0.0.0 mask 255.255.255.0 nomodify notrap nopeer noquery
```

cdh-[01-04] 为 NTP 客户端，修改 /etc/ntp.conf 文件：

```bash
# 注释掉以下 4 行
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst
# 添加下面 3 行
restrict cdh-01 nomodify notrap nopeer noquery
server cdh-01 iburst minpoll 4 maxpoll 10
server 127.127.1.0
fudge  127.127.1.0 stratum 10
```

先手动同步一下时间，所有节点执行：

```bash
ntpdate cn.pool.ntp.org
```

所有节点启动 NTP 服务：

```bash
systemctl enable ntpd --now
```

查看时间同步状态，NTP 服务端输出如下：

```bash
[root@cdh-01 ~]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
+120.25.115.20   10.137.53.7      2 u  127   64    2   44.010  -17.257   1.399
*203.107.6.88    100.107.25.114   2 u  122   64    2   15.028  -16.661  18.398
 LOCAL(0)        .LOCL.          10 l  131   64    4    0.000    0.000   0.000
```

说明：* 号代表此机器正在从其后的 IP 地址同步时间，203.107.6.88 是阿里云的一个 IP

查看时间同步状态，NTP 客户端输出如下：

```bash
[root@cdh-02 ~]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*cdh-01          203.107.6.88     3 u    1   64   77    0.214    9.571  10.833
 LOCAL(0)        .LOCL.          10 l  174   64   14    0.000    0.000   0.000
```

说明，NTP 服务端和客户端启动后，* 号会先出现在 LOCAL(0) 前方，需要等待几分钟才能切换。

### 安装 JDK
所有节点同步。

把JDK安装在 /usr/java 目录下，我的 JAVA_HOME 是 /usr/java/jdk

```bash
[root@cdh-01 ~]# ll /usr/java/
total 0
lrwxrwxrwx 1 root root  12 Jan 21 00:18 jdk -> jdk1.8.0_401
drwxr-xr-x 8 root root 294 Jan 21 00:18 jdk1.8.0_401
```

### 安装 httpd
在 cdh-01 节点安装：

```bash
yum -y install httpd
systemctl enable httpd --now
```

### 安装 MySQL
在 cdh-01 节点安装：

```bash
# 卸载已有的 MySQL
rpm -qa | grep mysql | xargs -i rpm -e {}
rpm -qa | grep mariadb | xargs -i rpm -e {}
rm -rf /etc/my.*
rm -rf /var/lib/mysql
rm -rf /var/log/mysqld.log
rm -rf /var/run/mysqld
userdel -r mysql

# 下载 yum 源文件
yum -y localinstall https://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm

# 导入 GDG 文件
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

# 安装 MySQL
yum -y install mysql-community-server
```

编辑 /etc/my.cnf 文件，在标签 [mysqld] 下追加配置：

```properties
disable_ssl
character_set_server = utf8mb4
character-set-client-handshake = FALSE
collation-server = utf8mb4_unicode_ci
```

新增 [client] 标签，其下配置为：

```properties
default-character-set = utf8mb4
# 这两行可选，配置为修改后的 root 密码
# 登录时使用 mysql 命令即可，省略 -u -p 选项
user=root
password=123456
```

新增 [mysql] 标签，其下配置为：

```properties
default-character-set = utf8mb4
```

启动 MySQL 服务：

```bash
systemctl enable mysqld --now
```

获取临时密码：

```bash
[root@cdh-01 ~]# grep 'temporary password' /var/log/mysqld.log
2024-01-27T08:27:41.772363Z 1 [Note] A temporary password is generated for root@localhost: Onipp0l..Ftd
```

登录 MySQL：

```bash
mysql -uroot -p"Onipp0l..Ftd"
```

MySQL 中的操作：

```sql
-- 关闭较强的密码校验策略
SET GLOBAL validate_password_policy=0;
SET GLOBAL validate_password_mixed_case_count=0;
SET GLOBAL validate_password_number_count=3;
SET GLOBAL validate_password_special_char_count=0;
SET GLOBAL validate_password_length=3;

-- 修改 root 用户密码
SET PASSWORD = '123456';

-- 允许 root 用户远程连接
USE mysql;
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
DROP USER 'root'@'localhost';

-- 创建 CDH 需要使用的用户并授权
CREATE DATABASE am     DEFAULT CHARACTER SET utf8;
CREATE DATABASE cm     DEFAULT CHARACTER SET utf8;
CREATE DATABASE rm     DEFAULT CHARACTER SET utf8;
CREATE DATABASE hue    DEFAULT CHARACTER SET utf8;
CREATE DATABASE hive   DEFAULT CHARACTER SET utf8;
CREATE DATABASE oozie  DEFAULT CHARACTER SET utf8;
CREATE DATABASE nav_as DEFAULT CHARACTER SET utf8;
CREATE DATABASE nav_ms DEFAULT CHARACTER SET utf8;
CREATE DATABASE sentry DEFAULT CHARACTER SET utf8;
CREATE USER 'am'@'%'     IDENTIFIED BY '123456';
CREATE USER 'cm'@'%'     IDENTIFIED BY '123456';
CREATE USER 'rm'@'%'     IDENTIFIED BY '123456';
CREATE USER 'hue'@'%'    IDENTIFIED BY '123456';
CREATE USER 'hive'@'%'   IDENTIFIED BY '123456';
CREATE USER 'oozie'@'%'  IDENTIFIED BY '123456';
CREATE USER 'nav_as'@'%' IDENTIFIED BY '123456';
CREATE USER 'nav_ms'@'%' IDENTIFIED BY '123456';
CREATE USER 'sentry'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON am.*     TO 'am'@'%';
GRANT ALL PRIVILEGES ON cm.*     TO 'cm'@'%';
GRANT ALL PRIVILEGES ON rm.*     TO 'rm'@'%';
GRANT ALL PRIVILEGES ON hue.*    TO 'hue'@'%';
GRANT ALL PRIVILEGES ON hive.*   TO 'hive'@'%';
GRANT ALL PRIVILEGES ON oozie.*  TO 'oozie'@'%';
GRANT ALL PRIVILEGES ON sentry.* TO 'sentry'@'%';
GRANT ALL PRIVILEGES ON nav_as.* TO 'nav_as'@'%';
GRANT ALL PRIVILEGES ON nav_ms.* TO 'nav_ms'@'%';

-- 刷新权限
FLUSH PRIVILEGES;

USE hive;
ALTER TABLE COLUMNS_V2       MODIFY COLUMN COMMENT      VARCHAR(256)  CHARACTER SET utf8;
ALTER TABLE TABLE_PARAMS     MODIFY COLUMN PARAM_VALUE  VARCHAR(4000) CHARACTER SET utf8;
ALTER TABLE PARTITION_PARAMS MODIFY COLUMN PARAM_VALUE  VARCHAR(4000) CHARACTER SET utf8;
ALTER TABLE PARTITION_KEYS   MODIFY COLUMN PKEY_COMMENT VARCHAR(4000) CHARACTER SET utf8;
ALTER TABLE INDEX_PARAMS     MODIFY COLUMN PARAM_VALUE  VARCHAR(4000) CHARACTER SET utf8;
```

### 安装 JDBC 驱动
所有节点同步。

```bash
mkdir -p /usr/share/java/
wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.49/mysql-connector-java-5.1.49.jar
mv mysql-connector-java-5.1.49.jar /usr/share/java/
cd /usr/share/java/
ln -s mysql-connector-java-5.1.49.jar mysql-connector-java.jar
```

### 确认 Python 版本
Hue 需要使用 Python-2.7，CentOS-7 默认安装的 Python 就是 2.7 版本，此处说明一下。

```bash
python -V
Python 2.7.5
```

## 安装 Cloudera Manager
### CM yum 源制作
在 cdh-01 节点操作。

```properties
mkdir -p /var/www/html/cm6.3.1
```

将以下文件上传到 /var/www/html/cm6.3.1 目录下：

+ allkeys.asc
+ cloudera-manager-server-db-2-6.3.1-1466458.el7.x86_64.rpm
+ cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
+ enterprise-debuginfo-6.3.1-1466458.el7.x86_64.rpm
+ cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
+ oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
+ cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm

执行以下操作：

```properties
cd /var/www/html/cm6.3.1
yum install createrepo -y
createrepo .
```

访问 http://cdh-01/cm6.3.1/ ：

![[assets/cdh-deploy-01.png]]

创建 /etc/yum.repos.d/cm.repo 文件，内容如下：

```plain
[cmrepo]
name = cm_repo
baseurl = http://cdh-01/cm6.3.1
enable = true
gpgcheck = false
```

验证：

```bash
yum repolist

# 看到 cmrepo 那一行就代表 cmrepo 生效了
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
cmrepo                                                                       | 2.9 kB  00:00:00     
cmrepo/primary_db                                                            | 8.7 kB  00:00:00     
repo id                           repo name                                       status
base/7/x86_64                     CentOS-7 - Base - mirrors.aliyun.com            10,072
cmrepo                            cm_repo                                              6
epel/x86_64                       Extra Packages for Enterprise Linux 7 - x86_64  13,786
extras/7/x86_64                   CentOS-7 - Extras - mirrors.aliyun.com             518
mysql-connectors-community/x86_64 MySQL Connectors Community                         242
mysql-tools-community/x86_64      MySQL Tools Community                              104
mysql57-community/x86_64          MySQL 5.7 Community Server                         696
updates/7/x86_64                  CentOS-7 - Updates - mirrors.aliyun.com          5,568
repolist: 30,992
```

### 安装 Cloudera Manager Server
在 cdh-01 节点操作。

```bash
yum -y install cloudera-manager-server
```

安装完成后，在 /opt 目录下生成了 cloudera 目录，将以下文件上传到 /opt/cloudera/parcel-repo目录下：

+ manifest.json
+ CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha256
+ CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha1
+ CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel

执行以下操作：

```bash
cd /opt/cloudera/parcel-repo
mv CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha1 CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha
chown cloudera-scm:cloudera-scm ./*
```

### 初始化数据库
在 cdh-01 节点操作。

```bash
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql cm cm 123456

# 输出
JAVA_HOME=/usr/java/jdk
Verifying that we can write to /etc/cloudera-scm-server
Creating SCM configuration file in /etc/cloudera-scm-server
......
INFO  Successfully connected to database.
All done, your SCM database is configured correctly!
```

### 启动 Cloudera Manager Server
在 cdh-01 节点操作。

```bash
systemctl enable cloudera-scm-server --now
systemctl status cloudera-scm-server
```

启动需要花几分钟时间，观察启动日志 /var/log/cloudera-scm-server/cloudera-scm-server.log，日志中出现以下信息才算真的启动成功：

```bash
2024-01-27 17:27:50,396 INFO WebServerImpl:org.eclipse.jetty.server.AbstractConnector: Started ServerConnector@6b455af6{HTTP/1.1,[http/1.1]}{0.0.0.0:7180}
2024-01-27 17:27:50,402 INFO WebServerImpl:org.eclipse.jetty.server.Server: Started @41408ms
2024-01-27 17:27:50,403 INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.
```

访问 CM Web UI：http://cdh-01:7180

![[assets/cdh-deploy-02.png]]

## 集群初始化
![[assets/cdh-deploy-03.png]]

![[assets/cdh-deploy-04.png]]

![[assets/cdh-deploy-05.png]]

![[assets/cdh-deploy-06.png]]

![[assets/cdh-deploy-07.png]]

![[assets/cdh-deploy-08.png]]

![[assets/cdh-deploy-09.png]]

![[assets/cdh-deploy-10.png]]

![[assets/cdh-deploy-11.png]]

![[assets/cdh-deploy-12.png]]

![[assets/cdh-deploy-13.png]]

![[assets/cdh-deploy-14.png]]

![[assets/cdh-deploy-15.png]]

![[assets/cdh-deploy-16.png]]

![[assets/cdh-deploy-17.png]]

![[assets/cdh-deploy-18.png]]

按照规划选择角色分布：

![[assets/cdh-deploy-19.png]]

![[assets/cdh-deploy-20.png]]

【审核更改】这一页很长，在生产环境部署时，要仔细核对这些配置，保证数据盘配置的正确、邮件告警相关的配置正确

![[assets/cdh-deploy-21.webp]]

![[assets/cdh-deploy-22.webp]]

![[assets/cdh-deploy-23.png]]

![[assets/cdh-deploy-24.png]]

## 相关

- [[50-运维与DevOps/00-索引|50-运维与DevOps]]
- [[50-运维与DevOps/90-大数据平台/00-索引|90-大数据平台]]
- Cloudera 官方文档：<https://docs.cloudera.com/>
