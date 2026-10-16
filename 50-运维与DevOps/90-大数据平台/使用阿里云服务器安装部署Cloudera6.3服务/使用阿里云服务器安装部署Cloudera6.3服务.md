---
title: 使用阿里云服务器安装部署Cloudera6.3服务
date: 2026-10-16
tags:
  - 运维
  - 大数据
  - cdh
  - 部署
---

# 使用阿里云服务器安装部署Cloudera6.3服务

> 在 3 台阿里云 CentOS 7.9 服务器（cdh1~cdh3）上，先做基础环境准备（主机名 / hosts / 防火墙 / SELinux / NTP / Python / sudo / IPv6 / swappiness / THP），再安装部署 Cloudera Manager 6.3.1 与 CDH 6.3.2 的完整步骤记录。

## 1. 环境说明
### 1.1 已有环境
新创建的阿里云服务器实例

服务器

| 主机名 | 操作系统 | CPU | 内存 | 磁盘 | 内网IP |
| --- | --- | --- | --- | --- | --- |
| cdh1 | CentOS-7.9 | 8vCPU | 32GB | 100GB | 172.17.129.243 |
| cdh2 | CentOS-7.9 | 8vCPU | 32GB | 100GB | 172.17.129.244 |
| cdh3 | CentOS-7.9 | 8vCPU | 32GB | 100GB | 172.17.129.245 |

### 1.2 磁盘规划
### 1.3 服务规划
| **组件** | **版本** | **cdh1角色** | **cdh2角色** | **cdh3角色** |
| --- | --- | --- | --- | --- |
| NTP | | 外网客户端   内网服务端 | 内网客户端 | 内网客户端 |
| Httpd | | httpd 服务 | | |
| Java | | Java | Java | Java |
| MySQL | 5.7最新版 | MySQL Server | | |
| CM | 6.3.1 | cloudera-scm-server   cloudera-scm-agent | cloudera-scm-agent | cloudera-scm-agent |

## 2. 基础环境准备
### 2.1 修改主机名
在全部节点操作

```bash
[root@iZ0jljb2r6aqwi5irn1iktZ ~]# hostnamectl set-hostname cdh1
[root@iZ0jljb2r6aqwi5irn1iksZ ~]# hostnamectl set-hostname cdh2
[root@iZ0jljb2r6aqwi5irn1ikrZ ~]# hostnamectl set-hostname cdh3
```

重新远程连接后即显示正确的修改后的主机名。

### 2.2 修改 hosts 文件
确保全部节点的 `/etc/hosts` 文件中都是以下内容

```plain
::1       localhost	localhost.localdomain	localhost6	localhost6.localdomain6
127.0.0.1 localhost	localhost.localdomain	localhost4	localhost4.localdomain4

172.17.129.243 cdh1
172.17.129.244 cdh2
172.17.129.245 cdh3
```

### 2.3 禁用防火墙
阿里云服务器已经关闭并且禁用了防火墙，如需手动禁用防火墙，在全部节点操作

```bash
[root@cdh2 ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
     
# 关闭并禁用防火墙
systemctl stop firewalld.service
systemctl disable firewalld.service
```

### 2.4 禁用 SELINUX
阿里云服务器已经禁用了SELINUX，如需手动禁用，在全部节点操作

临时禁用SELINUX

```plain
setenforce 0
```

永久禁用SELINUX，编辑`/etc/selinux/config`文件，将 SELINUX 的值设置为 `disabled`，默认是 `enforcing`：

```plain
SELINUX=disabled
```

验证：

```bash
[root@cdh1 ~]# getenforce 
Disabled
```

### 2.5 时钟同步
![[assets/aliyun-deploy-01.png]]

+ cdh1 作为时钟同步的客户端，向外网的阿里云时钟服务器同步时间
+ cdh1 作为内网的时钟服务器，允许其他 2 个节点向其同步时钟
+ cdh2 和 cdh3 作为内网的时钟同步客户端，向 cdh1 同步时钟

确保所有节点的时区相同：

```bash
[root@cdh2 ~]# timedatectl
      Local time: 二 2023-06-20 15:50:03 CST
  Universal time: 二 2023-06-20 07:50:03 UTC
        RTC time: 二 2023-06-20 07:50:02
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```

在所有节点安装 ntp：

```bash
yum install -y ntp
```

修改 cdh1 的 `/etc/ntp.conf`，下面展示的内容为文件全部内容

```plain
driftfile  /var/lib/ntp/drift
pidfile    /var/run/ntpd.pid
logfile    /var/log/ntp.log

# Access Control Support
# 拒绝所有来源的任何访问
restrict    default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery
# 开放本机的任何访问
restrict 127.0.0.1
restrict -6 ::1
# 允许 172.17.129.0 这个网段，子网掩码是 255.255.240.0 的所有服务器从自身同步时间
restrict 172.17.129.0 255.255.240.0 nomodify notrap
#restrict 192.168.0.0 mask 255.255.0.0 nomodify notrap nopeer noquery
#restrict 172.16.0.0 mask 255.240.0.0 nomodify notrap nopeer noquery
#restrict 100.64.0.0 mask 255.192.0.0 nomodify notrap nopeer noquery
#restrict 10.0.0.0 mask 255.0.0.0 nomodify notrap nopeer noquery

# local clock
# 外部时间服务器不可用时，以本地时间作为时间服务
server 127.127.1.0
fudge  127.127.1.0 stratum 10

# 允许上层时间服务器主动修改本机时间
restrict ntp.aliyun.com nomodify notrap nopeer noquery
restrict ntp.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp1.aliyun.com nomodify notrap nopeer noquery
restrict ntp1.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp10.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp11.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp12.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp2.aliyun.com nomodify notrap nopeer noquery
restrict ntp2.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp3.aliyun.com nomodify notrap nopeer noquery
restrict ntp3.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp4.aliyun.com nomodify notrap nopeer noquery
restrict ntp4.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp5.aliyun.com nomodify notrap nopeer noquery
restrict ntp5.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp6.aliyun.com nomodify notrap nopeer noquery
restrict ntp6.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp7.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp8.cloud.aliyuncs.com nomodify notrap nopeer noquery
restrict ntp9.cloud.aliyuncs.com nomodify notrap nopeer noquery

# 指定外部 NTP 服务器地址
server ntp.aliyun.com iburst minpoll 4 maxpoll 10
server ntp.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp1.aliyun.com iburst minpoll 4 maxpoll 10
server ntp1.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp10.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp11.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp12.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp2.aliyun.com iburst minpoll 4 maxpoll 10
server ntp2.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp3.aliyun.com iburst minpoll 4 maxpoll 10
server ntp3.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp4.aliyun.com iburst minpoll 4 maxpoll 10
server ntp4.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp5.aliyun.com iburst minpoll 4 maxpoll 10
server ntp5.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp6.aliyun.com iburst minpoll 4 maxpoll 10
server ntp6.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp7.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp8.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
server ntp9.cloud.aliyuncs.com iburst minpoll 4 maxpoll 10
```

注意关键配置：

```plain
# 允许 172.17.129.0 这个网段，子网掩码是 255.255.240.0 的所有服务器从自身同步时间
restrict 172.17.129.0 255.255.240.0 nomodify notrap
```

这里的网段和子网掩码不是固定的，查询：

```bash
# netmask 255.255.240.0
[root@cdh2 ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.129.244  netmask 255.255.240.0  broadcast 172.17.143.255
        inet6 fe80::216:3eff:fe04:a6a  prefixlen 64  scopeid 0x20<link>
        ether 00:16:3e:04:0a:6a  txqueuelen 1000  (Ethernet)
        RX packets 58181  bytes 79185502 (75.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10202  bytes 1107867 (1.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 3 台服务器的 IP 分别为 172.17.129.243、172.17.129.244、172.17.129.245
# 可见他们同属于一个网段 172.17.129.0
```

修改 cdh2 和 cdh3 的 `/etc/ntp.conf`，以下为全部配置：

```plain
driftfile  /var/lib/ntp/drift
pidfile    /var/run/ntpd.pid
logfile    /var/log/ntp.log

# local clock
# 外部时间服务器不可用时，以本地时间作为时间服务
server 127.127.1.0
fudge  127.127.1.0 stratum 10

# 允许 cdh1 时间服务器主动修改本机时间
restrict cdh1 nomodify notrap nopeer noquery

# 指定 NTP 服务器地址为 cdh1，即从 cdh1 同步时间
server cdh1 iburst minpoll 4 maxpoll 10
```

先手动同步一下时间，这是因为：如果服务器的时间和真实时间相差太多，就无法从 NTP 服务端同步时间。在所有节点操作：

```bash
ntpdate ntp.aliyun.com
```

启动并启用 NTP 服务，先在服务端（cdh1）启动，然后在客户端（cdh2、cdh3）启动：

```plain
systemctl enable ntpd --now
```

验证：

```bash
# 可以看到 * 才代表正常，* 代表当前正在从那个服务器同步时间
# 客户端 NTP 启动之后，可能需要一段时间（几分钟）之后才能看到以下现象
[root@cdh3 ~]# ntpq -p 
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 LOCAL(0)        .LOCL.          10 l   33   64    1    0.000    0.000   0.000
*cdh1            100.100.61.88    2 u   14   64    3    0.142   -5.006   0.083

[root@cdh2 ~]# ntpq -p 
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 LOCAL(0)        .LOCL.          10 l  49m   64    0    0.000    0.000   0.000
*cdh1            100.100.61.88    2 u   79  128  377    0.190   -7.286   6.025
```

### 2.6 确认 Python 的版本是 2.7
Centos-7.x 系统自带的 Python 版本就是 2.7，因为 CDH 中的组件 Hue 需要使用 Python-2.7，所以此处说明一下。

```bash
[root@cdh3 ~]# python -V
Python 2.7.5
```

### 2.7 创建一个具有 sudo 权限的用户
在所有节点操作

注意：用户名不要带中划线`-`

```bash
useradd admin
echo your_admin_password | passwd admin --stdin
```

编辑`/etc/sudoers`文件

```bash
## Allow root to run any commands anywhere 
root    ALL=(ALL)       ALL
# 这一行是新增加的
# 设置 admin 用户可以执行 sudo 命令，并且不需要输入密码
admin   ALL=(ALL)       NOPASSWD:ALL
```

### 2.8 禁用 IPV6
在所有节点操作

在 `/etc/sysctl.conf` 文件中追加：

```shell
net.ipv6.conf.all.disable_ipv6= 1
net.ipv6.conf.default.disable_ipv6= 1
net.ipv6.conf.lo.disable_ipv6= 1
```

在 `/etc/sysconfig/network` 文件中追加：

```shell
NETWORKING_IPV6=no
IPV6INIT=no
```

修改后执行`sysctl -p`命令

### 2.9 禁用交换分区
在所有节点操作

临时禁用交换分区：

```bash
echo 1 > /proc/sys/vm/swappiness
```

修改 `/etc/sysctl.conf` 文件：

```plain
vm.swappiness = 1
```

修改后执行`sysctl -p`命令

### 2.10 禁用透明大页面压缩
在所有节点操作

```bash
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag' >> /etc/rc.local
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.local
chmod +x /etc/rc.local
```

### 2.11 准备软件
准备以下软件：

+ CM 相关
    - cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
    - cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
    - cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
    - cloudera-manager-server-db-2-6.3.1-1466458.el7.x86_64.rpm
    - enterprise-debuginfo-6.3.1-1466458.el7.x86_64.rpm
    - oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
    - allkeys.asc
+ CDH 相关
    - manifest.json
    - CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha256
    - CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha
    - CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel
+ 扩展包
    - GPLEXTRAS-6.0.1-1.gplextras6.0.1.p0.590678-el7.parcel
    - GPLEXTRAS-6.0.1-1.gplextras6.0.1.p0.590678-el7.parcel.sha
+ JDBC 驱动包
    - mysql-connector-java-5.1.49.jar
+ MySQL

说明：我只有 CM6.3.1 和 CDH6.3.2 的包，所以只能使用不同的版本，经过测试，可以使用。

## 3. 安装部署
### 3.1 配置 CM yum 源
在 cdh1 节点操作

```bash
yum -y install httpd
systemctl enable httpd --now
```

进入 CM 安装包目录，生成 RPM 元数据：

```bash
[root@cdh1 ~]# cd cm6.3.1/
[root@cdh1 cm6.3.1]# tree
.
├── allkeys.asc
├── cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
├── cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
├── cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
├── cloudera-manager-server-db-2-6.3.1-1466458.el7.x86_64.rpm
├── enterprise-debuginfo-6.3.1-1466458.el7.x86_64.rpm
└── oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm

[root@cdh1 cm6.3.1]# yum install createrepo -y
[root@cdh1 cm6.3.1]# createrepo .

Spawning worker 0 with 2 pkgs
Spawning worker 1 with 1 pkgs
Spawning worker 2 with 1 pkgs
Spawning worker 3 with 1 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete

[root@cdh1 cm6.3.1]# cd ../
[root@cdh1 ~]# mv cm6.3.1 /var/www/html/
```

开放 cdh1 节点的 80 端口（后续需要访问 WEB 的各种端口都需要在阿里云的安全组中开放）：

![[assets/aliyun-deploy-02.png]]

![](images/prepare/%E9%98%BF%E9%87%8C%E4%BA%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E6%94%BE%E7%AB%AF%E5%8F%A301.png)

![[assets/aliyun-deploy-03.png]]

![](images/prepare/%E9%98%BF%E9%87%8C%E4%BA%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E6%94%BE%E7%AB%AF%E5%8F%A302.png)

![[assets/aliyun-deploy-04.png]]

![](images/prepare/%E9%98%BF%E9%87%8C%E4%BA%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E6%94%BE%E7%AB%AF%E5%8F%A303.png)

![[assets/aliyun-deploy-05.png]]

![](images/prepare/%E9%98%BF%E9%87%8C%E4%BA%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%BC%80%E6%94%BE%E7%AB%AF%E5%8F%A304.png)

WEB 访问：

![[assets/aliyun-deploy-06.png]]

配置 yum 源，创建文件 `/etc/yum.repos.d/cm.repo`，内容如下：

```properties
[cmrepo]
name = cmrepo
baseurl = http://cdh1/cm6.3.1
enable = true
gpgcheck = false
```

验证：

```plain
yum repolist
```

![[assets/aliyun-deploy-07.png]]

### 3.2 安装 JDK
CDH 对 JDK 版本的支持参考：`https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_java_requirements.html`

CDH6 不再支持 JDK7

CDH6 不支持的 JDK8 版本：

+ 8u40
+ 8u45
+ 8u60
+ 8u242

官网测试过的最新可用的 JDK 为 8u181

JDK的支持的安装方式有两种：

+ 使用 Cloudera Manager 安装：这种方式比较简易，在安装 agent 时可以一同安装，本文采用的就是这种方式
+ 手动安装 JDK：提前下载安装包，在所有的主机配置 JAVA_HOME，相对于大规模的集群会比较麻烦，另外，要把 JDK 安装在 `/usr/java` 目录下，比如设置`JAVA_HOME`=`/usr/java/jdk`，因为 CDH 默认会在 `/usr/java` 目录下寻找 JDK。

在这一步，只在 cdh1 节点安装 JDK，因为安装 Cloudera Manager Server 需要依赖 JDK

```bash
yum install -y oracle-j2sdk1.8.x86_64
```

JDK 安装在 `/usr/java/` 目录下

```bash
[root@cdh1 ~]# ls /usr/java/
jdk1.8.0_181-cloudera
```

使用这种方式安装的 JDK，可以不配置 JAVA_HOME，CM 会直接在 `/usr/java/jdk1.8.0_181-cloudera` 这个目录下执行 java 命令。当然如果有其他组件也要使用 JDK，那么还是需要配置 JAVA_HOME。

### 3.3 安装配置 MySQL
这里使用最简单的 yum 方式安装 MySQL，在 cdh1 节点操作

```bash
yum -y localinstall https://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
yum -y install mysql-community-server --nogpgcheck

# 启动并启用 MySQL
systemctl enable mysqld --now

# 获取 root 的密码
grep 'temporary password' /var/log/mysqld.log

2023-06-20T11:49:40.049570Z 1 [Note] A temporary password is generated for root@localhost: Fwi)Lg#aQ6eJ

# 登陆
mysql -uroot -p"Fwi)Lg#aQ6eJ"

# MySQL5.7 有复杂的密码校验，如果想设置简单密码，需要执行以下 SQL 语句
SET global validate_password_policy=0; 
SET global validate_password_mixed_case_count=0; 
SET global validate_password_number_count=3; 
SET global validate_password_special_char_count=0; 
SET global validate_password_length=3;

# 修改 root 用户密码
SET PASSWORD = '123456';

# 设置最高权限，允许所有 IP 远程连接
USE mysql;
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
DROP USER 'root'@'localhost';
FLUSH PRIVILEGES;

# 验证
SELECT user, host FROM user;
+---------------+-----------+
| user          | host      |
+---------------+-----------+
| root          | %         |
| mysql.session | localhost |
| mysql.sys     | localhost |
+---------------+-----------+
```

MySQL 设置默认的字符集编码，修改 `/etc/my.cnf` 配置文件：

```plain
[client]
default-character-set=utf8mb4
user=root
password=123456

[mysql]
default-character-set=utf8mb4

[mysqld]
character_set_server=utf8mb4
character-set-client-handshake=FALSE
collation-server=utf8mb4_unicode_ci
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

注意：生产环境请修改 datadir 的目录，保证 MySQL 使用的磁盘容量充足

重启 MySQL 服务并验证字符集设置：

```bash
systemctl restart mysqld
mysql -e"SHOW VARIABLES LIKE 'character%'"

+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8mb4                    |
| character_set_connection | utf8mb4                    |
| character_set_database   | utf8mb4                    |
| character_set_filesystem | binary                     |
| character_set_results    | utf8mb4                    |
| character_set_server     | utf8mb4                    |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
```

### 3.4 安装 Cloudera Manager Server
在 cdh1 节点操作

```bash
# 安装依赖
yum -y install cyrus-sasl-gssapi fuse-libs MySQL-python openssl-devel fuse lsb portmap mod_ssl python-psycopg2 libxslt

# 安装 Cloudera Manager Server
yum -y install cloudera-manager-server.x86_64
```

### 3.5 CM 数据库配置
在 cdh1 节点操作

编辑 `/etc/cloudera-scm-server/db.properties`文件

```properties
com.cloudera.cmf.db.type=mysql
com.cloudera.cmf.db.host=localhost
com.cloudera.cmf.db.name=cmf
com.cloudera.cmf.db.user=cmf
com.cloudera.cmf.db.password=cmf
com.cloudera.cmf.db.setupType=EXTERNAL
```

创建数据库和用户：

```sql
CREATE DATABASE cmf DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
SET global validate_password_policy=0; 
SET global validate_password_mixed_case_count=0; 
SET global validate_password_number_count=3; 
SET global validate_password_special_char_count=0; 
SET global validate_password_length=3;
GRANT ALL ON cmf.* TO 'cmf'@'%' IDENTIFIED BY 'cmf';
FLUSH PRIVILEGES;
```

### 3.6 安装 JDBC 驱动包
在 cdh1 节点操作

```bash
cp ~/jars/mysql-connector-java-5.1.49.jar /opt/cloudera/cm/lib/mysql-connector-java.jar
chown cloudera-scm:cloudera-scm /opt/cloudera/cm/lib/mysql-connector-java.jar
```

### 3.7 初始化数据库
在 cdh1 节点操作

```bash
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql cmf cmf cmf
```

### 3.7 配置 parcel
在 cdh1 节点操作

```bash
[root@cdh1 ~]# tree ~/cdh6.3.2
├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel
├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha
├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha256
└── manifest.json

mv ~/cdh6.3.2/* /opt/cloudera/parcel-repo/
chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo/*
```

### 3.8 启动 Cloudera Manager Server
在 cdh1 节点操作

```bash
systemctl enable cloudera-scm-server --now
```

观察启动日志：

```bash
tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log
```

运行成功的关键日志：

```plain
INFO main:com.cloudera.server.cmf.components.ScmActive: ScmActive started.
INFO main:com.cloudera.server.cmf.components.ClouderaManagerMetricsForwarder: ClouderaManagerMetricsForwarder started.
INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.
```

可以忽略的报错：

```plain
ERROR ParcelUpdateService:com.cloudera.parcel.components.ParcelDownloaderImpl: Unable to retrieve remote parcel repository manifest
java.util.concurrent.ExecutionException: java.net.ConnectException: connection timed out: archive.cloudera.com/146.75.112.167:443
```

访问 CM Web UI：`http://cdh1:7180`，需要开通 7180 端口

![[assets/aliyun-deploy-08.png]]

CM Server 成功启动后，在 `/opt/cloudera/parcel-repo` 目录下多了一个文件：`CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.torrent`，这代表着，CM 已经识别了 parcel 文件并且校验通过，这个 parcel 现在可以被正常使用。

```plain
[root@cdh1 parcel-repo]# pwd
/opt/cloudera/parcel-repo
[root@cdh1 parcel-repo]# tree
.
├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel
├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha
├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha256
├── CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.torrent
└── manifest.json
```

### 3.9 CM
#### 3.9.1 欢迎页面 1
![[assets/aliyun-deploy-09.png]]

#### 3.9.2 接受用户条款
![[assets/aliyun-deploy-10.png]]

#### 3.9.3 选择版本
![[assets/aliyun-deploy-11.png]]

说明：选择商业版的试用版也可以，在试用时间结束之后，会自动回退到免费版

#### 3.9.4 欢迎页面 2
![[assets/aliyun-deploy-12.png]]

#### 3.9.5 集群基本信息
![[assets/aliyun-deploy-13.png]]

#### 3.9.6 主机配置
![[assets/aliyun-deploy-14.png]]

![[assets/aliyun-deploy-15.png]]

#### 3.9.7 存储库配置
![[assets/aliyun-deploy-16.png]]

#### 3.9.8 JDK 安装选项
![[assets/aliyun-deploy-17.png]]

#### 3.9.9 SSH 配置
![[assets/aliyun-deploy-18.png]]

#### 3.9.10 安装 Agent
![[assets/aliyun-deploy-19.png]]

![[assets/aliyun-deploy-20.png]]

安装完成后，所有节点的 Agent 服务已经启动并且启用

```bash
[root@cdh2 ~]# systemctl status cloudera-scm-agent
● cloudera-scm-agent.service - Cloudera Manager Agent Service
   Loaded: loaded (/usr/lib/systemd/system/cloudera-scm-agent.service; enabled; vendor preset: disabled)
   Active: active (running) since 三 2023-06-21 11:03:14 CST; 30min ago
 Main PID: 2346 (cmagent)
   CGroup: /system.slice/cloudera-scm-agent.service
           └─2346 /usr/bin/python2 /opt/cloudera/cm-agent/bin/cm agent

6月 21 11:17:26 cdh2 cm[2346]: self.run()
6月 21 11:17:26 cdh2 cm[2346]: File "/usr/lib64/python2.7/multiprocessing/process.py", line 114, in run
6月 21 11:17:26 cdh2 cm[2346]: self._target(*self._args, **self._kwargs)
6月 21 11:17:26 cdh2 cm[2346]: File "/opt/cloudera/cm-agent/lib/python2.7/site-packages/cmf/monitor/host/filesystem_map.py", line 27, in disk_usage_wrapper
6月 21 11:17:26 cdh2 cm[2346]: usage = psutil.disk_usage(p)
6月 21 11:17:26 cdh2 cm[2346]: File "/opt/cloudera/cm-agent/lib/python2.7/site-packages/psutil/__init__.py", line 1947, in disk_usage
6月 21 11:17:26 cdh2 cm[2346]: return _psplatform.disk_usage(path)
6月 21 11:17:26 cdh2 cm[2346]: File "/opt/cloudera/cm-agent/lib/python2.7/site-packages/psutil/_psposix.py", line 131, in disk_usage
6月 21 11:17:26 cdh2 cm[2346]: st = os.statvfs(path)
6月 21 11:17:26 cdh2 cm[2346]: OSError: [Errno 2] No such file or directory: '/run/user/1000'
```

#### 3.9.11 安装 Parcels
![[assets/aliyun-deploy-21.png]]

此过程完成后，在 `/opt/cloudera/parcels/CDH` 目录下，就有了各组件的运行包

```bash
[root@cdh3 lib]# pwd
/opt/cloudera/parcels/CDH/lib
[root@cdh3 lib]# ll
总用量 116
drwxr-xr-x  2 root root 4096 11月  9 2019 avro
drwxr-xr-x  2 root root 4096 11月  9 2019 bigtop-utils
drwxr-xr-x  6 root root 4096 11月  9 2019 flume-ng
drwxr-xr-x 10 root root 4096 11月  9 2019 hadoop
drwxr-xr-x  7 root root 4096 11月  9 2019 hadoop-hdfs
drwxr-xr-x  4 root root 4096 11月  9 2019 hadoop-httpfs
drwxr-xr-x  4 root root 4096 11月  9 2019 hadoop-kms
drwxr-xr-x  5 root root 4096 11月  9 2019 hadoop-mapreduce
drwxr-xr-x  8 root root 4096 11月  9 2019 hadoop-yarn
drwxr-xr-x  7 root root 4096 11月  9 2019 hbase
drwxr-xr-x  6 root root 4096 11月  9 2019 hbase-solr
drwxr-xr-x 12 root root 4096 11月  9 2019 hive
drwxr-xr-x  8 root root 4096 11月  9 2019 hive-hcatalog
drwxr-xr-x  8 root root 4096 11月  9 2019 hue
drwxr-xr-x  9 root root 4096 11月  9 2019 impala
drwxr-xr-x  5 root root 4096 11月  9 2019 impala-shell
drwxr-xr-x  5 root root 4096 11月  9 2019 kafka
drwxr-xr-x  5 root root 4096 11月  9 2019 kite
drwxr-xr-x  8 root root 4096 11月  9 2019 kudu
drwxr-xr-x  8 root root 4096 11月  9 2019 oozie
drwxr-xr-x  6 root root 4096 11月  9 2019 parquet
drwxr-xr-x  8 root root 4096 11月  9 2019 pig
drwxr-xr-x  3 root root 4096 11月  9 2019 search
drwxr-xr-x  6 root root 4096 11月  9 2019 sentry
drwxr-xr-x 15 root root 4096 11月  9 2019 solr
drwxr-xr-x 10 root root 4096 11月  9 2019 spark
drwxr-xr-x  5 root root 4096 11月  9 2019 sqoop
drwxr-xr-x  5 root root 4096 11月  9 2019 zookeeper
drwxr-xr-x  2 root root 4096 11月  9 2019 zookeeper-native
```

#### 3.9.12 网络和主机检测
![[assets/aliyun-deploy-22.png]]

![](images/install-agent/cm-init-15.png)

![[assets/aliyun-deploy-23.png]]

![[assets/aliyun-deploy-24.png]]

这个问题是因为，阿里云服务器重启后，生成了新的主机名：

```bash
[root@cdh1 parcel-repo]# cat /etc/hosts
::1	localhost	localhost.localdomain	localhost6	localhost6.localdomain6
127.0.0.1	localhost	localhost.localdomain	localhost4	localhost4.localdomain4

172.17.129.244	cdh2
172.17.129.245	cdh3

172.17.129.243	cdh1
172.17.129.243	iZ0jljb2r6aqwi5irn1iktZ	iZ0jljb2r6aqwi5irn1iktZ
```

在所有节点操作，确保 `/etc/hosts` 文件内容如下：

```bash
[root@cdh1 parcel-repo]# cat /etc/hosts
::1	localhost	localhost.localdomain	localhost6	localhost6.localdomain6
127.0.0.1	localhost	localhost.localdomain	localhost4	localhost4.localdomain4

172.17.129.244	cdh2
172.17.129.245	cdh3
172.17.129.243	cdh1
```

![[assets/aliyun-deploy-25.png]]

![](images/install-agent/cm-init-18.png)

![[assets/aliyun-deploy-26.png]]

下一步，会自动跳转到安装服务的页面，直接关闭页面即可。

![[assets/aliyun-deploy-27.png]]

### 3.10 安装 Cloudera Management Service
创建数据库

```sql
CREATE DATABASE activity_monitor DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
SET global validate_password_policy=0; 
SET global validate_password_mixed_case_count=0; 
SET global validate_password_number_count=3; 
SET global validate_password_special_char_count=0; 
SET global validate_password_length=3;
GRANT ALL ON activity_monitor.* TO 'activity_monitor'@'%' IDENTIFIED BY 'activity_monitor';
FLUSH PRIVILEGES;
```

访问`http://cdh1:7180`

![[assets/aliyun-deploy-28.png]]

![](images/install-cloudera-management-service/01.png)

![[assets/aliyun-deploy-29.png]]

![[assets/aliyun-deploy-30.png]]

![[assets/aliyun-deploy-31.png]]

![](images/install-cloudera-management-service/02.png)

![[assets/aliyun-deploy-32.png]]

![[assets/aliyun-deploy-33.png]]

到这里，CM 服务就安装部署成功了。

## 附录
CDH6 安装的环境要求：`https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_requirements_supported_versions.html`

CDH6 与 JDK 的兼容性：`https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_java_requirements.html`

CDH6 各组件版本信息：`https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_cdh_6_packaging.html`

CDH5 各组件版本信息：`https://docs.cloudera.com/documentation/enterprise/release-notes/topics/cdh_vd_cdh_package_tarball.html`

CDH6 安装手册：`https://docs.cloudera.com/documentation/enterprise/6/latest/topics/installation.html`

CDH6 磁盘规划：`https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_reqs_space.html`

CDH6 主机与角色规划：`https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_host_allocations.html`

## 相关

- [[50-运维与DevOps/90-大数据平台/00-索引|90-大数据平台]]
