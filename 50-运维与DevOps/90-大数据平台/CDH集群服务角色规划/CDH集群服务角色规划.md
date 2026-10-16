---
title: CDH集群服务角色规划
date: 2026-10-16
tags:
  - 大数据
  - cdh
  - 架构
---

# CDH集群服务角色规划

> Cloudera 官方对不同规模集群的主机与角色分布建议，覆盖 3-10 到 500-1000 个工作节点。

官网地址：

[Recommended Cluster Hosts and Role Distribution](https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_host_allocations.html)

集群中的主机大致分为以下几类：

+ Master 节点运行 Hadoop Master 进程，例如 HDFS NameNode 和 YARN ResourceManager；
+ Utility 节点运行其他非 Master 进程的集群进程，例如 Cloudera Manager 和 Hive Metastore；
+ Gateway 节点是集群的客户端访问点，用于启动作业。所需的 Gateway 主机数量取决于工作负载的类型和大小。
+ Worker 节点主要运行 DataNodes 和其他分布式进程，例如 Impalad。

## 3-10个工作节点非高可用集群
![[assets/cdh-role-plan-01.png]]

## 3-20个工作节点高可用集群
![[assets/cdh-role-plan-02.png]]

## 20-80个工作节点高可用集群
![[assets/cdh-role-plan-03.png]]

## 80-200个工作节点高可用集群
![[assets/cdh-role-plan-04.png]]

## 200-500个工作节点高可用集群
![[assets/cdh-role-plan-05.png]]

## 500-1000个工作节点高可用集群
![[assets/cdh-role-plan-06.png]]

## 相关

- [[50-运维与DevOps/90-大数据平台/00-索引|90-大数据平台]]
