# 笔记三 Nginx + Consul + UpSync 实现动态负载均衡

[TOC]

## 1.动态负载均衡



### 1.1 传统负载均衡 和 动态负载均衡区别

- **传统负载均衡**

  > 传统的负载均衡，如果**Upstream参数**发生变化，每次都需要**重新加载nginx.conf文件**,
  >
  > 因此**扩展性不是很高**

- **动态负载均衡**

  > 实现Upstream可**配置化**、**动态化**，**无需人工重新加载nginx.conf**。



### 1.2 动态负载均衡的实现方案

- Consul+OpenResty 实现无需raload动态负载均衡
- **Consul+upsync+Nginx**  实现无需raload动态负载均衡



## 2.Consul



### 2.1 Consul 是什么？

> - Consul是一款开源的**分布式服务注册**与**发现系统**。
>
> - 通过**HTTP API**可以使得**服务注册**、**发现实现**起来非常简单。



### 2.2 Consul 有什么特性？

- 服务注册

  > 服务**实现者**可以通过HTTP API或DNS方式，**将服务注册到Consul**。

- 服务发现

  > 服务**消费者**可以通过HTTP API或DNS方式，**从Consul获取服务的IP和PORT**。

- 故障检测

  > 支持如TCP、HTTP等方式的健康检查机制，从而**当服务有故障时自动摘除**。

- K/V存储

  > 使用**K/V存储**实现**动态配置中心**，其使用HTTP长轮询实现**变更触发**和**配置更改**。

- 多数据中心

  > 支持多数据中心，可以按照数据中心注册和发现服务，即支持只消费本地机房服务，使用**多数据中心集群**还可以**避免单数据中心的单点故障**。

- Raft算法

  > Consul使用Raft算法实现集群**数据一致性**。

通过Consul可以管理服务注册与发现，接下来需要有一个与Nginx部署在同一台机器的Agent来实现Nginx配置更改和Nginx重启功能。我们有Confd或者Consul-template两个选择，而Consul-template是Consul官方提供的，我们就选择它了。其使用HTTP长轮询实现变更触发和配置更改（使用Consul的watch命令实现）。也就是说，我们使用Consul-template实现配置模板，然后拉取Consul配置渲染模板来生成Nginx实际配置。



## 3.Consul + nginx + upsync 

### 3.1 原理流程图

![nginx+consul+upsync](C:\Users\Calvin\Pictures\Saved Pictures\nginx+consul+upsync.png)



