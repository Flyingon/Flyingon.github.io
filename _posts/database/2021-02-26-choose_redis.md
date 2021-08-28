---
layout: post
title: redis选型-高可用
category: 数据库
tags: [redis]
keywords: redis
---


## 参数对比

| 产品名称   | 腾讯云Redis标准版                      | 腾讯云Redis集群版            | Tendis存储版                  | Tendis ssd                   |
| ---------- | -------------------------------------- | ---------------------------- | ----------------------------- | ---------------------------- |
| 实现原理   | 原生的 Redis                           | Redis官方集群方案            | 自研，实现redis协议，磁盘存储 | 自研，实现redis协议，ssd存储 |
| 可扩展性   | **0.25GB  - 64GB，****不支持水平扩容** | 4GB - 20TB，支持水平扩容     | 240GB - 32TB，支持水平扩容    | 240GB - 32TB，支持水平扩容   |
| 命令兼容性 | 最全                                   | 极少数集群命令不支持         | **较多命令不支持**            | 少数命令不支持               |
| 性能       | 8万  -  10万(2核8G),延时低             | 24-30万 (单节点2核8G),延时低 | **延时高**                    | 24万(24分片)，热数据延时低   |
| 成本       | 较高                                   | **极高**                     | 极低                          | 极低                         |
| 轻松运维   | 腾讯云，工具链完善                     | 腾讯云，工具链完善           | SCR平台，基本满足             | SCR平台，基本满足            |


### 参考：

腾讯云redis: [https://cloud.tencent.com/document/product/239/3205](https://cloud.tencent.com/document/product/239/3205)
腾讯云tendis: [https://cloud.tencent.com/document/product/1363/50795](https://cloud.tencent.com/document/product/1363/50795)
tedis官网: [http://tendis.cn/#/](http://tendis.cn/#/)
Redis vs Tendis：冷热混合存储版架构揭秘: [https://aijishu.com/a/1060000000199100](https://aijishu.com/a/1060000000199100)
tedis视频: [https://cloud.tencent.com/developer/salon/live-1326](https://cloud.tencent.com/developer/salon/live-1326)

## Redis主备切换
参考(阿里云): [https://help.aliyun.com/document_detail/100734.html](https://help.aliyun.com/document_detail/100734.html)

参考(腾讯云): [https://cloud.tencent.com/document/product/239/4106](https://cloud.tencent.com/document/product/239/4106)
### 介绍
#### 容灾
云数据库Redis版提供了全球范围的异地多活服务，即Redis全球多活，适用于需要在多地域同步部署的业务场景，与传统的灾备方案最大的区别在于多活。异地多活架构使业务能够在多个地域同时进行，各地域中的全球多活子实例实时双向同步。

Redis全球多活实例由多活子实例、同步通道以及通道管理器构成。
- 多活子实例是基本服务单元，所有子实例均可读写。
- 同步通道支持子实例间的实时双向同步，以及容忍度达到天级别的断点续传。
- 通道管理器管理同步通道的生命周期，同时处理子实例在故障时的主从切换以及备份重搭，保证多活实例的高可用。

读写分离架构实例由代理服务器、主从架构的读写节点以及若干只读副本（Read-Only Replica）构成。
- 高可用HA模块实时监测各节点的状态，在读写节点的主节点发生故障时发起主从切换，同时将只读节点连接到新的主节点上来。
- 当只读节点发生故障时， HA模块将重建只读节点，并更新相应的路由及权重信息。
- Proxy实时监控只读节点的服务状态，当发现某个只读节点状态异常时会降低该节点的权重。如果只读节点多次连接失败，Proxy将停止该节点的服务，直至其恢复正常。

集群架构（双副本）实例由配置服务器、代理服务器和分片服务器组成：
- 配置服务器（Config Server）是用于提供全局路由信息和配置信息的集群管理工具，采用遵循Raft协议的三副本集群架构。
- 代理服务器（Proxy Server）为单节点架构，集群版结构中会有多个Proxy，系统自动对所有Proxy进行负载均衡及故障转移。
- 分片服务器（Shard Server）同样采用双副本高可用架构，与标准版-双副本实例相同，主节点故障之后，HA模块会自动进行主从切换保证服务高可用，并更新Proxy Server和Config Server的信息。

全球多活同步架构:
![aliyun_redis_architect.png](/assets/img/architect/aliyun_redis_architect.png)

#### 工具
redis容量预估: [http://www.redis.cn/redis_memory/](http://www.redis.cn/redis_memory/)

### Raft协议
__参考__: [https://zhuanlan.zhihu.com/p/27207160](https://zhuanlan.zhihu.com/p/27207160)