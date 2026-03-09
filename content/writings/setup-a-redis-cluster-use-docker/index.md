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

本文介绍如何使用 Docker 搭建 Redis 集群环境。

<!--more-->

## 准备工作

首先创建一个 Docker 容器：

```sh
docker run -it --name=linx_1 ubuntu sleep infinity
```

## 创建 Redis 集群

请参考官方文档：[create-a-redis-cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/#create-a-redis-cluster)
