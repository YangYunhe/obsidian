---
title: CDH 集群上线操作手册
date: 2026-10-16
tags:
  - 运维
  - 大数据
  - cdh
  - 部署
---

# CDH 集群上线操作手册

> 面向 **5 节点、离线环境** 的 CDH 6.3 上线手册：先给角色规划，再按「基础环境 → 离线源 → CM → 集群安装 → HDFS（NameNode HA）→ YARN → Hive → Hue」顺序部署，末尾附验证清单。

## 〇、适用范围与前提

| 项    | 值                                                           |
| ---- | ----------------------------------------------------------- |
| 集群   | `sy-fqz01` ~ `sy-fqz05`（5 节点）                               |
| IP   | **待定**，本文暂按 `10.0.0.11` ~ `10.0.0.15` 书写（上线前以实际为准，全文替换即可）   |
| 系统   | CentOS 7.x（x86_64）                                          |
| 版本   | Cloudera Manager **6.3.1** + CDH **6.3.2**（Parcel）          |
| JDK  | Oracle JDK **1.8（8u181）**，随 CM 源中的 `oracle-j2sdk1.8` RPM 安装 |
| 网络   | **离线环境**：无外网，所有软件包须提前备齐并内网分发                                |
| 时间同步 | **NTP 已统一配置**，本文只做验证（见 §2.10），不重复配置                         |
| 目标组件 | HDFS（**NameNode HA**）、YARN（**ResourceManager HA**）、Hive、Hue |
| 依赖组件 | ZooKeeper、JournalNode（两处 HA 必需）、Cloudera Manager、MySQL 5.7  |

> [!warning] 前提假设（上线前逐条确认）
> - HDFS 与 YARN **均启用 HA**：2 个 NameNode、2 个 ResourceManager，都依赖同一套 3 节点 ZooKeeper 仲裁。
> - 节点角色见 §1.2，可按实际硬件调整。
> - 磁盘：如有独立数据盘，请先挂载（本文不展开），并在 CM 中把 DataNode/NodeManager 数据目录指到数据盘。

## 一、角色规划

### 1.1 节点信息

| 主机名      | IP（暂定）    | CPU  | 内存   | 系统盘 | 数据盘                    | 操作系统       |
| -------- | --------- | ---- | ---- | --- | ---------------------- | ---------- |
| sy-fqz01 | 10.0.0.11 | 32核  | 512G |     | 960G \* 4 + 1.92T \* 4 | CentOS-7.9 |
| sy-fqz02 | 10.0.0.12 | 32 核 | 512G |     | 视情况                    | CentOS-7.9 |
| sy-fqz03 | 10.0.0.13 | 32 核 | 512G |     | 视情况                    | CentOS-7.9 |
| sy-fqz04 | 10.0.0.14 | 32 核 | 512G |     | 视情况                    | CentOS-7.9 |
| sy-fqz05 | 10.0.0.15 | 32核  | 512G |     | 视情况                    | CentOS-7.9 |

### 1.2 角色分布

| 服务 / 角色 | sy-fqz01 | sy-fqz02 | sy-fqz03 | sy-fqz04 | sy-fqz05 |
| --- | :-: | :-: | :-: | :-: | :-: |
| Cloudera Manager Server | ● | | | | |
| Cloudera Manager Agent | ● | ● | ● | ● | ● |
| MySQL（CM / Hive / Hue 元库） | ● | | | | |
| Httpd（离线 yum 源） | ● | | | | |
| ZooKeeper Server | | ● | ● | ● | |
| JournalNode | | ● | ● | ● | |
| NameNode | | ● Active | ● Standby | | |
| ZKFC（Failover Controller） | | ● | ● | | |
| DataNode | | ● | ● | ● | ● |
| ResourceManager | | ● Active | ● Standby | | |
| JobHistory Server | | ● | | | |
| NodeManager | | ● | ● | ● | ● |
| Hive Metastore Server | ● | | | | |
| HiveServer2 | ● | | | | |
| Hue Server | ● | | | | |
| Gateway（客户端） | ● | ● | ● | ● | ● |

### 1.3 规划说明

- **NameNode HA 三要素**：2 个 NameNode（sy-fqz02/03）+ 3 个 JournalNode（sy-fqz02/03/04）+ 3 节点 ZooKeeper 仲裁（sy-fqz02/03/04）。ZKFC 与 NameNode 同机。
- **ResourceManager HA 三要素**：2 个 ResourceManager（sy-fqz02/03）+ 共用上述 ZooKeeper 仲裁；RM 状态存于 ZooKeeper，Active/Standby 自动切换。（JobHistory Server 为单实例，不支持 HA。）
- sy-fqz01 为 **Utility/管理节点**：CM、MySQL、离线源、Hive/Hue 前端。
- sy-fqz02/03 为 **Master**，同时兼跑 DataNode/NodeManager（小集群常见）。
- sy-fqz04/05 为 **Worker**。
- JournalNode 与 ZooKeeper 数量必须为**奇数且 ≥3**，否则 HA 无仲裁能力。

## 二、基础环境准备（**在所有节点执行**）

### 2.1 主机名

```bash
# 各节点分别执行，主机名与 IP 对应
hostnamectl set-hostname sy-fqz01   # sy-fqz02 … sy-fqz05 同理
```

### 2.2 hosts 解析（全节点一致）

```plain
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.0.0.11   sy-fqz01
10.0.0.12   sy-fqz02
10.0.0.13   sy-fqz03
10.0.0.14   sy-fqz04
10.0.0.15   sy-fqz05
```

### 2.3 root 免密登录（CM 需要 root SSH）

```bash
# 在 sy-fqz01 执行，生成密钥并把公钥分发到全部节点（含自身）
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa
for h in sy-fqz01 sy-fqz02 sy-fqz03 sy-fqz04 sy-fqz05; do
  ssh-copy-id -i ~/.ssh/id_rsa.pub root@$h
done
# 验证
ssh root@sy-fqz05 'hostname'
```

### 2.4 禁用 SELinux（全节点）

```bash
setenforce 0                 # 临时
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config   # 永久
getenforce                   # 期望输出 Disabled
```

### 2.5 禁用防火墙（全节点）

```bash
systemctl stop firewalld && systemctl disable firewalld
```

### 2.6 关闭交换分区（全节点）

```bash
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab            # 永久关闭（谨慎，确认无需 swap）
echo 'vm.swappiness = 1' >> /etc/sysctl.conf
sysctl -p
```

### 2.7 禁用透明大页面压缩（全节点）

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# 写入开机脚本
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.local
echo 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'  >> /etc/rc.local
chmod +x /etc/rc.local
```

### 2.8 禁用 IPv6（全节点，可选）

```bash
cat >> /etc/sysctl.conf <<'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
sysctl -p
```

### 2.9 确认 Python 2.7（Hue 依赖）

```bash
python -V      # 期望 Python 2.7.5（CentOS 7 自带）
```

### 2.10 时间同步验证（**NTP 已配置，仅验证**）

```bash
# 各节点时区应为 Asia/Shanghai
timedatectl | grep -E 'Time zone|synchronized'

# 各节点查看同步源，remote 前带 * 表示当前正在同步（客户端启动后可能需等待几分钟）
ntpq -p
```

期望：所有节点时区一致，且 `ntpq -p` 中存在带 `*` 的正常同步源。

## 三、离线软件准备与内网 yum 源

> 无外网，需提前将下列软件包拷入 sy-fqz01。

### 3.1 软件清单

| 分类 | 文件 |
| --- | --- |
| CM | `cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm`、`cloudera-manager-daemons-...`、`cloudera-manager-agent-...`、`cloudera-manager-server-db-2-...`、`enterprise-debuginfo-...`、`oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm`、`allkeys.asc` |
| CDH Parcel | `CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel`、`.parcel.sha1`、`.parcel.sha256`、`manifest.json` |
| 扩展包 | `GPLEXTRAS-6.0.1-1.gplextras6.0.1.p0.590678-el7.parcel`（+ `.sha`） |
| JDBC | `mysql-connector-java-5.1.49.jar` |
| MySQL | `mysql-community-*` 5.7 的离线 RPM 集合 |
| 系统依赖 | `createrepo`、`httpd`、`cyrus-sasl-gssapi`、`fuse-libs`、`MySQL-python`、`openssl-devel`、`libxslt` 等（建议用内部 CentOS 7 镜像或 ISO 直接 yum） |

### 3.2 配置内网 yum 源（sy-fqz01）

```bash
# 1) 关闭联网 repo，指向内部镜像/ISO
mkdir -p /etc/yum.repos.d/bak && mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak/ 2>/dev/null
cat > /etc/yum.repos.d/local-base.repo <<'EOF'
[local-base]
name=Local CentOS 7 Base
baseurl=http://<内网镜像或本机挂载 ISO>/centos7
enabled=1
gpgcheck=0
EOF
```

### 3.3 制作 CM yum 源（sy-fqz01）

```bash
yum -y install httpd createrepo
systemctl enable httpd --now

mkdir -p /var/www/html/cm6.3.1
cp /path/to/cm-rpms/*.rpm /path/to/cm-rpms/allkeys.asc /var/www/html/cm6.3.1/
cd /var/www/html/cm6.3.1
createrepo .
```

各节点配置 CM 源（`/etc/yum.repos.d/cm.repo`）：

```plain
[cmrepo]
name = cmrepo
baseurl = http://sy-fqz01/cm6.3.1
enabled = 1
gpgcheck = 0
```

验证：

```bash
yum repolist        # 能看到 cmrepo 即成功
```

### 3.4 放置 CDH Parcel（sy-fqz01）

```bash
mkdir -p /opt/cloudera/parcel-repo
cp CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel* manifest.json /opt/cloudera/parcel-repo/
# 用 sha1 重命名为 .sha（CM 校验需要）
cd /opt/cloudera/parcel-repo
mv CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha1 CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha
chown cloudera-scm:cloudera-scm ./*
```

> 离线环境无需配置远端 parcel 地址；启动 CM 时若出现 `Unable to retrieve remote parcel repository manifest`（访问 archive.cloudera.com 超时），属**可忽略**告警。

## 四、安装 Cloudera Manager Server

> 以下在 **sy-fqz01** 执行。

### 4.1 安装 JDK

```bash
yum -y install oracle-j2sdk1.8.x86_64
ls /usr/java/          # 期望 jdk1.8.0_181-cloudera
```

### 4.2 安装并初始化 MySQL 5.7（离线）

```bash
# 卸载自带 mariadb（如有）
rpm -qa | grep -E 'mariadb|mysql' | xargs -r rpm -e --nodeps
rm -rf /etc/my.cnf /var/lib/mysql

# 离线安装 mysql-community-server 及其依赖 RPM
yum -y localinstall /path/to/mysql57/*.rpm
systemctl enable mysqld --now

# 取临时密码并登录
grep 'temporary password' /var/log/mysqld.log
mysql -uroot -p'<临时密码>'
```

降低密码策略并设置 root 密码（MySQL 内执行）：

```sql
SET GLOBAL validate_password_policy=0;
SET GLOBAL validate_password_mixed_case_count=0;
SET GLOBAL validate_password_number_count=3;
SET GLOBAL validate_password_special_char_count=0;
SET GLOBAL validate_password_length=3;
SET PASSWORD = '<你的密码>';
USE mysql;
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '<你的密码>' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

字符集（`/etc/my.cnf`）：

```plain
[client]
default-character-set=utf8mb4

[mysqld]
character_set_server=utf8mb4
character-set-client-handshake=FALSE
collation-server=utf8mb4_unicode_ci
```

```bash
systemctl restart mysqld
mysql -e "SHOW VARIABLES LIKE 'character%'"
```

### 4.3 创建 CM 服务端数据库

```sql
CREATE DATABASE cmf DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON cmf.* TO 'cmf'@'%' IDENTIFIED BY 'cmf';
FLUSH PRIVILEGES;
```

### 4.4 安装 CM Server

```bash
# 依赖（离线源需包含）
yum -y install cyrus-sasl-gssapi fuse-libs MySQL-python openssl-devel fuse libxslt
yum -y install cloudera-manager-server cloudera-manager-daemons cloudera-manager-agent
```

### 4.5 放置 JDBC 驱动

```bash
cp /path/to/mysql-connector-java-5.1.49.jar /usr/share/java/mysql-connector-java.jar
chown cloudera-scm:cloudera-scm /usr/share/java/mysql-connector-java.jar
```

### 4.6 初始化 CM 数据库

```bash
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql cmf cmf cmf
# 期望结尾：All done, your SCM database is configured correctly!
```

### 4.7 启动 CM Server

```bash
systemctl enable cloudera-scm-server --now
tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log
```

成功标志（日志出现）：

```plain
INFO WebServerImpl:org.eclipse.jetty.server.Server: Started @xxxxx ms
INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.
```

访问 CM Web UI：`http://sy-fqz01:7180`（默认 `admin` / `admin`）。

## 五、通过 CM 安装 CDH 集群

浏览器打开 `http://sy-fqz01:7180`，按向导操作：

1. **欢迎页 → 选择版本**：选「Cloudera Express（免费）」或试用版均可（试用到期自动回退免费版）。
2. **集群基本信息**：给集群命名，如 `sy-fqz-cluster`。
3. **主机**：填写 `sy-fqz01, sy-fqz02, sy-fqz03, sy-fqz04, sy-fqz05`（或按 IP 段扫描）。
4. **存储库**：使用**本地 Parcel 仓库**（默认即可，CM Server 自身分发 parcel）。
5. **JDK 安装选项**：选「使用已有 JDK / 由 CM 安装」均可（源里已有 `oracle-j2sdk1.8`）。
6. **SSH 凭据**：填 root 的**登录方式**（端口 22，用户名 root，密码或私钥）。
7. **安装 Agent**：等待各节点 `cloudera-scm-agent` 安装并启动。

```bash
# 各节点验证 Agent
systemctl status cloudera-scm-agent
```

8. **安装 Parcel**：分发并激活 `CDH 6.3.2`。
9. **检查主机正确性**：CM 会提示网络/主机解析问题，按提示修复（常见为 `/etc/hosts` 与集群 DNS 冲突）。

## 六、部署 HDFS（含 NameNode HA）

### 6.1 先装 ZooKeeper（HA 的仲裁基础）

在 CM 中添加服务 **ZooKeeper** → 角色分配到 `sy-fqz02 / sy-fqz03 / sy-fqz04`（奇数 3 台）→ 启动。

```bash
# 验证仲裁
echo ruok | nc sy-fqz02 2181; echo ruok | nc sy-fqz03 2181; echo ruok | nc sy-fqz04 2181
# 期望均输出 imok
```

### 6.2 添加 HDFS 并启用 HA

CM → 添加服务 **HDFS**：

- **启用 High Availability**：勾选。
- **Nameservice 名称**：如 `nameservice1`（或自定义 `hdfs-ns`）。
- **NameNode 主机**：`sy-fqz02`、`sy-fqz03`（一主一备，CM 自动在其上部署 ZKFC）。
- **JournalNode 主机**：`sy-fqz02`、`sy-fqz03`、`sy-fqz04`（3 台，存放共享 edits）。
- **DataNode 主机**：`sy-fqz02`、`sy-fqz03`、`sy-fqz04`、`sy-fqz05`。
- **容错（Fencing）**：保持 CM 默认（基于 ZKFC 的自动故障转移）。

关键配置项（HDFS → 配置）：

```plain
dfs.nameservices            = nameservice1
dfs.ha.namenodes.nameservice1 = nn1,nn2
dfs.namenode.rpc-address.nameservice1.nn1 = sy-fqz02:8020
dfs.namenode.rpc-address.nameservice1.nn2 = sy-fqz03:8020
dfs.namenode.shared.edits.dir = qjournal://sy-fqz02:8485;sy-fqz03:8485;sy-fqz04:8485/nameservice1
fs.defaultFS                = hdfs://nameservice1
```

> **要点**：客户端与上层组件（Hive/Hue）一律使用 **nameservice**（`hdfs://nameservice1`），**不要**写具体某个 NameNode，否则故障切换后连接会失效。

### 6.3 HDFS 验证

```bash
# 查看主备状态（在任一 NameNode 节点执行）
hdfs haadmin -getServiceState nn1     # 期望 active
hdfs haadmin -getServiceState nn2     # 期望 standby

# 查看数据节点
hdfs dfsadmin -report | head -20      # 期望 Live datanodes = 4

# 手工故障切换演练（生产慎用，需 --forcefence --forceactive）
hdfs haadmin -failover --forcefence --forceactive nn1 nn2

# 简单读写
hdfs dfs -mkdir -p /tmp/verify && echo ok | hdfs dfs -put - /tmp/verify/a.txt
hdfs dfs -cat /tmp/verify/a.txt
```

## 七、部署 YARN（ResourceManager HA）

> YARN HA 复用 §6.1 的 3 节点 ZooKeeper：RM 状态存于 ZK，实现 Active/Standby 自动切换，无需额外组件。

CM → 添加服务 **YARN (MR2 Included)**：

- **启用 ResourceManager HA**：勾选。
- **ResourceManager 主机**：`sy-fqz02`、`sy-fqz03`（一主一备）。
- **JobHistory Server**：`sy-fqz02`（**单实例，不支持 HA**）。
- **NodeManager**：`sy-fqz02`、`sy-fqz03`、`sy-fqz04`、`sy-fqz05`。

关键配置（YARN → 配置）：

```plain
yarn.resourcemanager.ha.enabled        = true
yarn.resourcemanager.ha.rm-ids         = rm1,rm2
yarn.resourcemanager.hostname.rm1      = sy-fqz02
yarn.resourcemanager.hostname.rm2      = sy-fqz03
yarn.resourcemanager.cluster-id        = sy-fqz-yarn

yarn.resourcemanager.recovery.enabled  = true
yarn.resourcemanager.store.class       = org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore
yarn.resourcemanager.zk-address        = sy-fqz02:2181,sy-fqz03:2181,sy-fqz04:2181

yarn.nodemanager.resource.memory-mb    = 24576     # 按各节点物理内存的 ~75% 调整
yarn.scheduler.maximum-allocation-mb   = 8192
yarn.nodemanager.resource.cpu-vcores   = 6
```

验证：

```bash
# 在任一 RM 节点查看主备
yarn rmadmin -getServiceState rm1      # 期望 active
yarn rmadmin -getServiceState rm2      # 期望 standby

# 节点与应用
yarn node -list                        # 期望 4 个 RUNNING 节点
yarn application -list                 # 应为空、无报错

# RM 故障切换演练（生产慎用）
yarn rmadmin -failover rm1 rm2
```

> 客户端 / Hive / Hue 通过 `yarn.resourcemanager.ha.id` 与 ZooKeeper 自动定位当前 Active RM，**不要**写死某一台 RM 地址。

## 八、准备业务元数据库（MySQL）

在 sy-fqz01 的 MySQL 中创建 Hive / Hue / CM 管理服务所需库：

```sql
-- Hive 元数据
CREATE DATABASE hive DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';
GRANT ALL ON hive.* TO 'hive'@'%';

-- Hue
CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'hue'@'%' IDENTIFIED BY 'hue';
GRANT ALL ON hue.* TO 'hue'@'%';

-- Cloudera Management Service（Activity Monitor 等，向导亦可自动创建）
CREATE DATABASE activity_monitor DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON activity_monitor.* TO 'activity_monitor'@'%' IDENTIFIED BY 'activity_monitor';

FLUSH PRIVILEGES;
```

> 所有节点确保有 MySQL JDBC 驱动：`/usr/share/java/mysql-connector-java.jar`（见 §4.5）。

## 九、部署 Hive 与 Hue

### 9.1 Hive

CM → 添加服务 **Hive**：

- **Hive Metastore Server**：`sy-fqz01`
- **HiveServer2**：`sy-fqz01`
- **Gateway**：全部节点

Metastore 数据库配置（Hive → 配置）：

```plain
Hive Metastore 数据库类型 = MySQL
数据库主机名 = sy-fqz01
数据库名称   = hive
用户名 / 密码 = hive / hive
```

`hive.metastore.warehouse.dir` 使用 **nameservice**：

```plain
hive.metastore.warehouse.dir = /user/hive/warehouse
# 依赖 fs.defaultFS = hdfs://nameservice1，CM 会自动落到 HA 地址
```

验证：

```bash
beeline -u 'jdbc:hive2://sy-fqz01:10000/'
# 进入后：
SHOW DATABASES;      # 期望能看到 default
```

### 9.2 Hue

CM → 添加服务 **Hue**：

- **Hue Server**：`sy-fqz01`
- Hue 数据库：类型 MySQL，主机 `sy-fqz01`，库/用户/密码 = `hue`

Hue 会自动集成 HDFS / YARN / Hive（若页面提示未连接 Hive，检查 Hue 配置中的 HiveServer2 地址 `sy-fqz01:10000`）。

访问 `http://sy-fqz01:8888`，首次登录设置管理员账号。

## 十、上线验证清单

| 项 | 命令 / 位置 | 期望 |
| --- | --- | --- |
| CM 各服务健康 | CM 主页 | 全部绿色，无红色告警 |
| ZooKeeper | `echo ruok \| nc sy-fqz0{2,3,4} 2181` | `imok` |
| NameNode HA | `hdfs haadmin -getServiceState nn1/nn2` | 一 `active` 一 `standby` |
| 故障切换 | `hdfs haadmin -failover` 演练 | 主备互换成功、HDFS 读写不中断 |
| DataNode | `hdfs dfsadmin -report` | Live datanodes = 4 |
| HDFS 读写 | `hdfs dfs -put / -cat` | 正常 |
| YARN RM HA | `yarn rmadmin -getServiceState rm1/rm2` | 一 `active` 一 `standby` |
| YARN 节点 | `yarn node -list` | 4 个 NodeManager RUNNING |
| YARN 切换 | `yarn rmadmin -failover` 演练 | 主备互换成功、提交作业不中断 |
| Hive | `beeline` → `SHOW DATABASES` | 正常返回 |
| Hue | `http://sy-fqz01:8888` | 可登录，HDFS/YARN/Hive 图标正常 |
| 时间同步 | `ntpq -p`（各节点） | 存在带 `*` 的同步源 |
| Agent | `systemctl status cloudera-scm-agent`（各节点） | active (running) |

## 附录

### A. 常用端口

| 服务 | 端口 |
| --- | --- |
| Cloudera Manager | 7180 |
| HDFS NameNode RPC / HTTP | 8020 / 9870 |
| JournalNode | 8485 |
| ZooKeeper | 2181 |
| YARN ResourceManager / JobHistory | 8088 / 19888 |
| HiveServer2 / Hive Metastore | 10000 / 9083 |
| Hue | 8888 |

### B. 离线环境注意

- 全部 RPM、Parcel、JDBC 驱动必须提前备齐；上线当天不得依赖外网。
- `Unable to retrieve remote parcel repository manifest` 属可忽略告警。
- 若 CM 提示某依赖包缺失，从内部 yum 源补齐后重试。

### C. 回滚/下线

- 优先在 CM 中「停止」再「删除」服务，避免残留进程与数据。
- 涉及 NameNode HA 的下线：先在 CM 中解除 HA（Disable High Availability），再删除 HDFS 服务。

## 相关

- [[50-运维与DevOps/90-大数据平台/00-索引|90-大数据平台]]
- [[CDH 6.3 集群部署记录]]
- [[使用阿里云服务器安装部署Cloudera6.3服务]]
- [[CDH集群服务角色规划]]
- [[CDH各版本对应的组件版本]]
- [[CDH集群JDK版本兼容性]]
- Cloudera 官方安装文档：<https://docs.cloudera.com/documentation/enterprise/6/latest/topics/installation.html>
