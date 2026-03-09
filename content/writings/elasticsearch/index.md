---
title: "Elasticsearch 架构探索"
date: 2024-12-03T15:49:51+08:00
description: "深入探索 Elasticsearch 集群架构，包括三节点高可用设计、分片治理、健康检查与索引生命周期管理。"
categories:
- 架构设计
tags:
- elasticsearch
- 架构设计
- 运维部署
---

ElasticSearch是一个高可用的搜索引擎，最近在项目中由于客户基础设施的成熟度不够，需要Ops来进行ElasticSearch的高可用部署，了解了其底层集群相关的知识，本文将对其进行总结，介绍ElasticSearch中集群的架构及数据的存储。

<!--more-->

# 测试环境的搭建

在实际部署 Elasticsearch 生产环境之前，我们需要先搭建一个可靠的测试环境来验证集群的行为。以下是使用 Docker Compose 搭建 Elasticsearch 集群的完整步骤。

## Docker Compose 配置

首先创建一个 `docker-compose.yml` 文件来定义三个节点的集群：

```yaml
version: '3.8'

services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es01_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - es_network

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es02_data:/usr/share/elasticsearch/data
    networks:
      - es_network

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es03_data:/usr/share/elasticsearch/data
    networks:
      - es_network

volumes:
  es01_data:
  es02_data:
  es03_data:

networks:
  es_network:
    driver: bridge
```

## 启动集群

```bash
# 启动集群
docker-compose up -d

# 查看集群状态
docker-compose logs -f es01

# 等待集群启动完成（约 30-60 秒）
# 检查健康状态
curl -XGET 'http://localhost:9200/_cluster/health?pretty'
```

## 验证集群

集群启动后，可以验证以下关键指标：

```bash
# 查看节点信息
curl -XGET 'http://localhost:9200/_cat/nodes?v'

# 查看分片信息
curl -XGET 'http://localhost:9200/_cat/shards?v'

# 查看索引信息
curl -XGET 'http://localhost:9200/_cat/indices?v'
```

---

## 为什么是三个?

在 Elasticsearch 集群部署中，**三个节点**是最小推荐配置，这背后有三个核心原因：

### 1. 脑裂问题的防止

"脑裂"（Split-Brain）是分布式系统中常见的问题，指的是集群分裂成多个独立的部分，每个部分都认为自己是主节点。如果集群只有两个节点，当网络分区发生时，两个节点会各自认为自己是主节点，导致数据不一致。

```
2 节点集群的脑裂场景：
┌─────────┐     网络分区     ┌─────────┐
│  es01   │ ─────────────── │  es02   │
│ (主节点) │                 │ (主节点) │
└─────────┘                 └─────────┘
两个节点都认为自己是主节点！
```

三个节点可以通过**多数派投票**避免脑裂：只有获得至少 2 票（超过半数）的节点才能成为主节点。

### 2. 主节点选举的稳定性

Elasticsearch 使用 Raft 协议进行主节点选举：

| 节点数 | 可用节点数 | 能否选主 | 脑裂风险 |
|--------|-----------|---------|---------|
| 1 | 1 | ✅ | N/A |
| 2 | 1 | ❌ 需2票 | 高 |
| **3** | **2** | ✅ 需2票 | **低** |
| 4 | 2 | ❌ 需3票 | 中 |
| **5** | **3** | ✅ 需3票 | **最低** |

三个节点是**成本与安全性的最佳平衡点**。

### 3. 分片副本的要求

为了保证数据高可用，索引通常会配置：
- **1 个主分片**（Primary Shard）
- **1 个副本分片**（Replica Shard）

只有当副本分片分布在**不同的节点**上时，副本才有意义。如果只有两个节点，副本只能分布在 2 个节点上，无法真正实现高可用。

---

## 数据的分片治理

Elasticsearch 的数据分片策略直接影响集群的性能和可靠性。以下是分片治理的核心概念。

### 分片类型

```
┌─────────────────────────────────────────────┐
│               Index (索引)                   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Shard 0 │  │ Shard 1 │  │ Shard 2 │      │
│  │ (主分片) │  │ (主分片) │  │ (主分片) │      │
│  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │            │            │            │
│       ▼            ▼            ▼            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Replica │  │ Replica │  │ Replica │      │
│  │   0     │  │   1     │  │   2     │      │
│  │(副本)    │  │(副本)    │  │(副本)    │      │
│  └─────────┘  └─────────┘  └─────────┘      │
└─────────────────────────────────────────────┘
```

### 分片数量策略

**分片数量 = 节点数 × (1 ~ 3)**

建议的分片数量：

```bash
# 创建索引时指定分片数
PUT /my_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

**注意事项：**

1. **分片不宜过多**
   - 每个分片都是一个 Lucene 索引，有内存开销
   - 建议每个节点的分片数不超过 20-30 个

2. **副本数量根据可用性要求设置**
   - 开发环境：0 副本
   - 生产环境：至少 1 副本
   - 高可用要求：2 副本

### 分片的健康检查

Elasticsearch 提供了完善的健康检查机制，通过 `_cluster/health` API 可以监控集群和分片的状态：

```bash
# 查看集群整体健康状态
curl -XGET 'http://localhost:9200/_cluster/health?pretty'

# 响应示例：
{
  "cluster_name" : "es-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 10,
  "active_shards" : 20,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0
}
```

**健康状态说明：**

| 状态 | 含义 |
|------|------|
| `green` | 所有分片都正常 |
| `yellow` | 主分片正常，但有副本分片未分配 |
| `red` | 有主分片未分配，服务不可用 |

```bash
# 查看特定索引的健康状态
curl -XGET 'http://localhost:9200/_cluster/health/my_index?pretty'

# 查看分片详情
curl -XGET 'http://localhost:9200/_cat/shards/my_index?v'
```

### 分片分配策略

```bash
# 查看分片分配过滤器
curl -XGET 'http://localhost:9200/_cluster/settings?pretty'

# 设置分片分配过滤器（按节点属性）
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.awareness.attributes": "zone"
  }
}
```

### 索引生命周期管理

生产环境中，应该使用 Index Lifecycle Management (ILM) 来自动管理分片：

```bash
# 创建 ILM 策略
PUT /_ilm/policy/my_lifecycle_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "30gb",
            "max_primary_shard_size": "50gb"
          }
        }
      },
      "warm": {
        "min_age": "30d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## 总结

本文介绍了 Elasticsearch 集群架构的核心概念，包括：

1. **测试环境搭建**：使用 Docker Compose 快速搭建三节点集群
2. **为什么是三个**：从脑裂防护、主节点选举、高可用角度分析
3. **数据分片治理**：分片类型、数量策略、健康检查、生命周期管理

理解这些概念对于构建高可用的搜索服务至关重要。在实际生产环境中，还需要考虑：
- 节点角色分离（Master、Data、Ingest 节点）
- 跨机房/跨区域部署
- 监控和告警配置
- 备份和恢复策略

## 参考资料

1. [Elasticsearch Architecture Best Practices](https://www.elastic.co/pdf/architecture-best-practices.pdf)
2. [System Design Series: Elasticsearch - Architecting for Search](https://betterprogramming.pub/system-design-series-elasticsearch-architecting-for-search-5d5e61360463)
3. [Elasticsearch 官方文档 - Cluster Health](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html)
4. [Elasticsearch 官方文档 - ILM](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
