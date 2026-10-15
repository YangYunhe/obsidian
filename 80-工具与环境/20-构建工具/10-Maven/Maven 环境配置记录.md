---
title: Maven 环境配置记录
date: 2026-10-15
tags:
  - 环境配置
  - maven
  - java
  - linux
环境: Linux (bash) / 用户 yangyunhe
---

# Maven 环境配置记录

> 记录时间：2026-10-15 22:19 (CST)
> 操作目标：把 Maven 加入用户环境变量（PATH），配置阿里云镜像加速，并对 Maven 运行环境做优化，最后留存本记录。

## 一、环境信息

| 项目 | 值 |
| --- | --- |
| 操作系统 | Linux 7.0.0-38-generic (x86_64) |
| 登录用户 | `yangyunhe` |
| 默认 Shell | `/bin/bash` |
| JDK | Oracle JDK `25.0.4.1`，路径 `/usr/java/jdk-25.0.4.1` |
| Maven | Apache Maven `3.9.16`，路径 `/home/yangyunhe/应用/apache-maven-3.9.16` |
| 系统区域 | `zh_CN.UTF-8` |

配置前状态：`mvn` 命令不可用（未加入 PATH），`~/.m2/settings.xml` 不存在。

## 二、变更清单

| 文件                       | 类型  | 说明                           |
| ------------------------ | --- | ---------------------------- |
| `~/.config/maven/env.sh` | 新建  | Maven / Java 环境变量统一维护文件      |
| `~/.bashrc`              | 追加  | 交互式 shell 加载上述 env 文件        |
| `~/.profile`             | 追加  | 登录 shell / 图形桌面应用加载上述 env 文件 |
| `~/.m2/settings.xml`     | 新建  | 用户级 Maven 配置，含阿里云镜像          |

> 采用「独立 env 文件 + 被两个启动脚本 source」的方式，避免在 `.bashrc` 与 `.profile` 中重复维护同一份配置（DRY）。

## 三、操作步骤

### 1. 创建 Maven 环境变量文件

新建 `~/.config/maven/env.sh`，内容如下：

```bash
#!/usr/bin/env bash
# Maven / Java 用户环境变量（由 ~/.bashrc 与 ~/.profile 共同 source）
# 幂等追加式写法，重复 source 不会污染 PATH

export JAVA_HOME="/usr/java/jdk-25.0.4.1"

export MAVEN_HOME="/home/yangyunhe/应用/apache-maven-3.9.16"
export M2_HOME="$MAVEN_HOME"   # 兼容读取 M2_HOME 的老脚本

case ":$PATH:" in
    *":$MAVEN_HOME/bin:"*) ;;
    *) export PATH="$MAVEN_HOME/bin:$PATH" ;;
esac

export MAVEN_OPTS="-Xms512m -Xmx2048m \
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
-XX:+UseG1GC \
-Dfile.encoding=UTF-8 -Duser.timezone=Asia/Shanghai \
-Dmaven.artifact.threads=8 \
--add-opens java.base/java.lang=ALL-UNNAMED \
--add-opens java.base/java.util=ALL-UNNAMED"

alias mvnci='mvn clean install'
alias mvncp='mvn clean package'
alias mvnct='mvn clean test'
alias mvndep='mvn dependency:tree'
alias mvnh='mvn help:effective-settings'
```

**关键点说明：**

- 用 `case ":$PATH:"` 判断后再追加，而不是直接 `export PATH=...:$PATH`，避免每次 source 重复累积路径。
- `MAVEN_OPTS` 各项含义：

  | 参数 | 作用 |
  | --- | --- |
  | `-Xms512m -Xmx2048m` | 堆初始/上限，构建大项目时减少 GC 抖动 |
  | `-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m` | 元空间（注解处理/反射多时更稳） |
  | `-XX:+UseG1GC` | 使用 G1 垃圾回收器 |
  | `-Dfile.encoding=UTF-8` | 统一编码，避免中文乱码 |
  | `-Duser.timezone=Asia/Shanghai` | 固定时区，构建产物时间戳一致 |
  | `-Dmaven.artifact.threads=8` | 并发下载依赖线程数，加速依赖解析 |
  | `--add-opens java.base/...=ALL-UNNAMED` | 适配 JDK 17+ 模块系统，兼容部分老插件 |

### 2. 让启动脚本加载环境文件

`.bashrc` 末尾追加（交互式终端）：

```bash
# Maven / Java 环境（统一维护于 ~/.config/maven/env.sh）
[ -f "$HOME/.config/maven/env.sh" ] && . "$HOME/.config/maven/env.sh"
```

`.profile` 末尾追加（登录 shell / 图形桌面启动的 IDE 等）：

```bash
# Maven / Java 环境（供登录 shell / 图形桌面启动的应用使用）
[ -f "$HOME/.config/maven/env.sh" ] && . "$HOME/.config/maven/env.sh"
```

> 之所以两处都加：`.bashrc` 开头对**非交互式** shell 会提前 `return`，而图形桌面/IDE 启动的登录 shell 往往是非交互式的，只在 `.profile` 中生效才能保证 IntelliJ IDEA 等工具也能拿到 `mvn`。

### 3. 配置阿里云镜像

新建 `~/.m2/settings.xml`（用户级，优先级高于 `${MAVEN_HOME}/conf/settings.xml`，且升级 Maven 不会覆盖）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">

  <localRepository>${user.home}/.m2/repository</localRepository>

  <mirrors>
    <mirror>
      <id>aliyun-public</id>
      <name>阿里云公共仓库 (central + jcenter)</name>
      <url>https://maven.aliyun.com/repository/public</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>

  <profiles>
    <profile>
      <id>aliyun-repos</id>
      <repositories>
        <repository>
          <id>aliyun-spring</id>
          <url>https://maven.aliyun.com/repository/spring</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
        <repository>
          <id>aliyun-google</id>
          <url>https://maven.aliyun.com/repository/google</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>aliyun-plugin</id>
          <url>https://maven.aliyun.com/repository/public</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>aliyun-repos</activeProfile>
  </activeProfiles>

</settings>
```

**关于 `mirrorOf` 的选择：**

- `*`：拦截**所有**仓库统一走阿里云，个人开发机最省心（当前采用）。
- `central,!aliyun-*`：仅代理 `central`，保留项目自定义仓库（如公司私服）。若你有企业私服，请改用这一行。

## 四、验证结果

### 1. `mvn` 可用（全新登录 shell）

```console
$ which mvn
/home/yangyunhe/应用/apache-maven-3.9.16/bin/mvn

$ mvn -v
Apache Maven 3.9.16 (2bdd9fddda4b155ebf8000e807eb73fd829a51d5)
Maven home: /home/yangyunhe/应用/apache-maven-3.9.16
Java version: 25.0.4.1, vendor: Oracle Corporation, runtime: /usr/java/jdk-25.0.4.1
Default locale: zh_CN, platform encoding: UTF-8
OS name: "linux", version: "7.0.0-38-generic", arch: "amd64", family: "unix"
```

### 2. 镜像生效（实际下载走阿里云）

下载一个测试依赖后，本地仓库的 `_remote.repositories` 显示来源为 `aliyun-public`：

```console
$ mvn -B dependency:get -Dartifact=org.apache.commons:commons-lang3:3.14.0
$ cat ~/.m2/repository/org/apache/commons/commons-lang3/3.14.0/_remote.repositories
commons-lang3-3.14.0.jar>aliyun-public=
commons-lang3-3.14.0.pom>aliyun-public=
```

`mvn help:effective-settings` 中也可看到生效的镜像：

```xml
<mirror>
  <id>aliyun-public</id>
  <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

## 五、注意事项

1. **安装路径含中文**：Maven 装在 `~/应用/...`，该路径含非 ASCII 字符。JVM 依赖 `LANG`/locale 才能正确解码该路径——**在 `LANG` 未设置的极简环境**（如 cron、systemd 服务、`env -i`）中启动 mvn 会报
   `Could not find or load main class org.codehaus.plexus.classworlds.launcher.Launcher`。
   日常终端（`LANG=zh_CN.UTF-8`）不受影响。如遇此错，导出 `LANG=zh_CN.UTF-8` 即可，或把 Maven 移到纯 ASCII 路径。
2. **代理不影响 Maven**：`.bashrc` 中的 `HTTP_PROXY` 等变量 Java/Maven **不读取**（Java 只认 `-Dhttp.proxyHost` 系统属性）。阿里云为国内源、可直连，故未在 `settings.xml` 中配置代理；若日后需要走代理，应在 `settings.xml` 的 `<proxies>` 段或 `MAVEN_OPTS` 中配置。
3. **生效方式**：新开终端即自动生效；当前已打开的终端执行 `source ~/.config/maven/env.sh` 或 `source ~/.bashrc`。
4. **镜像作用域**：当前 `mirrorOf` 为 `*`，会拦截所有仓库。若引入需要私有仓库的项目，请按上文改为 `central,!aliyun-*`。

## 六、回滚方法

```bash
# 1. 删除环境文件与 Maven 配置
rm -f ~/.config/maven/env.sh ~/.m2/settings.xml

# 2. 从 ~/.bashrc 与 ~/.profile 中移除下面这行（各一处）
#    [ -f "$HOME/.config/maven/env.sh" ] && . "$HOME/.config/maven/env.sh"

# 3. 使改动生效
source ~/.bashrc
```

## 相关

- Maven 官方配置文档：<https://maven.apache.org/settings.html>
- 阿里云 Maven 仓库：<https://maven.aliyun.com/>
