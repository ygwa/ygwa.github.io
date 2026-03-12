---
title: "使用 Docker 来配置 Redis 集群"
date: 2024-12-09T16:30:25+08:00
description: "使用 Docker 搭建 Redis 集群环境，介绍三节点集群的配置方法与高可用架构。"
categories:
- 运维部署
tags:
- redis
- docker
- 运维部署
- 工具使用
---

本文用一套可复现流程搭建 Redis Cluster，并验证高可用能力。

<!--more-->

## 准备工作

推荐使用 Docker Compose 一次性启动 6 个节点（3 主 3 从），这样更接近生产环境的高可用拓扑。

前置条件：

- Docker 24+
- Docker Compose v2
- 本机可用端口：`7001-7006`、`17001-17006`

## 编写 Compose 文件

在任意目录创建 `docker-compose.yml`：

```yaml
services:
  redis-7001:
    image: redis:7.2
    container_name: redis-7001
    command: >
      redis-server --port 7001 --cluster-enabled yes
      --cluster-config-file nodes.conf --cluster-node-timeout 5000
      --appendonly yes --protected-mode no
    ports:
      - "7001:7001"
      - "17001:17001"
    volumes:
      - ./data/7001:/data

  redis-7002:
    image: redis:7.2
    container_name: redis-7002
    command: >
      redis-server --port 7002 --cluster-enabled yes
      --cluster-config-file nodes.conf --cluster-node-timeout 5000
      --appendonly yes --protected-mode no
    ports:
      - "7002:7002"
      - "17002:17002"
    volumes:
      - ./data/7002:/data

  redis-7003:
    image: redis:7.2
    container_name: redis-7003
    command: >
      redis-server --port 7003 --cluster-enabled yes
      --cluster-config-file nodes.conf --cluster-node-timeout 5000
      --appendonly yes --protected-mode no
    ports:
      - "7003:7003"
      - "17003:17003"
    volumes:
      - ./data/7003:/data

  redis-7004:
    image: redis:7.2
    container_name: redis-7004
    command: >
      redis-server --port 7004 --cluster-enabled yes
      --cluster-config-file nodes.conf --cluster-node-timeout 5000
      --appendonly yes --protected-mode no
    ports:
      - "7004:7004"
      - "17004:17004"
    volumes:
      - ./data/7004:/data

  redis-7005:
    image: redis:7.2
    container_name: redis-7005
    command: >
      redis-server --port 7005 --cluster-enabled yes
      --cluster-config-file nodes.conf --cluster-node-timeout 5000
      --appendonly yes --protected-mode no
    ports:
      - "7005:7005"
      - "17005:17005"
    volumes:
      - ./data/7005:/data

  redis-7006:
    image: redis:7.2
    container_name: redis-7006
    command: >
      redis-server --port 7006 --cluster-enabled yes
      --cluster-config-file nodes.conf --cluster-node-timeout 5000
      --appendonly yes --protected-mode no
    ports:
      - "7006:7006"
      - "17006:17006"
    volumes:
      - ./data/7006:/data
```

## 启动节点

```bash
docker compose up -d
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

## 创建 Redis 集群

进入任意一个节点执行集群初始化：

```bash
docker exec -it redis-7001 redis-cli --cluster create \
  host.docker.internal:7001 host.docker.internal:7002 host.docker.internal:7003 \
  host.docker.internal:7004 host.docker.internal:7005 host.docker.internal:7006 \
  --cluster-replicas 1
```

如果你在 Linux 环境，可将 `host.docker.internal` 改为宿主机 IP 或容器网络可达 IP。

## 验证集群状态

```bash
# 查看集群节点分布
docker exec -it redis-7001 redis-cli -p 7001 cluster nodes

# 查看集群健康状态
docker exec -it redis-7001 redis-cli -p 7001 cluster info
```

重点观察：

- `cluster_state:ok`
- 有 3 个 `master`，每个主节点有 1 个 `slave`
- 槽位 `0-16383` 被完整覆盖

## 高可用验证（故障切换）

手动停止一个主节点，观察是否自动提升从节点：

```bash
docker stop redis-7001
sleep 10
docker exec -it redis-7002 redis-cli -p 7002 cluster nodes
```

若看到原 `slave` 节点晋升为 `master`，说明故障转移生效。测试完成后可重新启动节点：

```bash
docker start redis-7001
```

## 总结

通过 3 主 3 从的 Redis Cluster 拓扑，我们可以获得：

1. 分片能力：数据自动分布到不同主节点
2. 高可用能力：主节点故障时可自动故障转移
3. 水平扩展能力：后续可继续加节点并迁移槽位

如果你想进一步扩展到生产环境，建议继续补充：

- 持久化策略（AOF/RDB）与恢复演练
- 监控告警（内存、水位、主从延迟、故障转移事件）
- 跨可用区部署与网络策略

## 适用边界

- 本文默认单机或同网段测试环境，跨机房部署需额外考虑网络抖动与延迟。
- Redis Cluster 对多键操作有槽位限制，业务侧需要配合设计。
