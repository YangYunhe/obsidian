---
title: CDH集群JDK版本兼容性
date: 2026-10-16
tags:
  - 大数据
  - cdh
  - java
  - 参考
---

# CDH集群JDK版本兼容性

> CDH 各版本支持的 JDK 版本对照，以及压缩普通对象指针（Compressed OOPs）与堆内存设置建议。

官网地址：[Java Requirements](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_java_requirements.html)

JDK 使用了一种名为压缩普通对象指针（Compressed OOPs，Compressed Ordinary Object Pointers）的优化技术，其特点如下：

+ 64位 JVM 可以使用 4 字节指针来寻址最多 32GB 的堆内存
+ 大于 32GB 的堆内存，需要 8 字节指针
+ 稍小于 32GB 的堆内存可以比稍大于 32GB 的堆内存容纳更多的对象

所以：

+ 如果不需要超过 32GB 的堆内存，则设置堆内存 <= 31GB
+ 如果需要超过 32GB 的堆内存，则设置堆内存 >= 48GB
+ 通常，如果需要超过 32GB 的堆内存，则设置为所需堆内存的 1.5 倍

| **CDH 版本** | **支持的 Oracle JDK 版本** | **支持的 Open JDK 版本** |
| --- | --- | --- |
| 5.3-5.15 | 1.7、1.8 | none |
| >= 5.16 | 1.7、1.8 | 1.8 |
| 6.0 | 1.8 | none |
| 6.1 | 1.8 | 1.8 |
| 6.2 | 1.8 | 1.8 |
| 6.3 | 1.8 | 1.8、>= 11.0.3 |

说明：

+ 所有支持的 JDK 版本都必须是 64位
+ CDH 集群中的所有节点都必须使用相同版本的 JDK
+ 不能使用的 JDK-8 有：8u271、8u281、8u291、8u40、8u45、8u60
+ 经过 CDH 官方测试并建议使用的 Oracle JDK-8 版本：8u181、8u162、8u141、8u131、8u121、8u111、8u102、8u91、8u74、8u31
+ 经过 CDH 官方测试并建议使用的 Open JDK-8 版本：8u212、8u181

## 相关

- [[50-运维与DevOps/90-大数据平台/00-索引|90-大数据平台]]
