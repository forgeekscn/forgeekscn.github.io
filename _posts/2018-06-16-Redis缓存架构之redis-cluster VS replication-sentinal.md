---
layout:     post
title:      Redis缓存架构
subtitle:   Redis缓存架构之 redis-cluster VS replication-sentinal
date:       2018-06-16
author:     Fogeeks
header-img: img/post-bg-re-vs-ng2.jpg
catalog:    true
tags:
    - Redis
---

#  redis cluster vs. replication + sentinal

大数据高并发场景下，传统的关系数据库已经无法支撑如此频繁大量的读写请求，毕竟数据库读写io十分耗内存，除了考虑数据库性能优化sql优化之外，这时候还要考虑更加高效合理的方法应对高并发，
此时就要用redis缓存来减轻数据库压力，缩短请求响应时间，提高并发量。

### 单机redis性能分析
下面测试下单机redis的性能，虚拟机centos系统master的内存2G单核，

```text

====== GET ======

34094.78 requests per second

```


表头|表头|表头
---|:--:|---:
内容|内容|内容
内容|内容|内容

```java

package com.qyf404.learn.maven;
import org.junit.After;
import org.junit.Assert;

import org.junit.Before;

import org.junit.Test;

public class AppTest {

 

```   

### redis持久化+备份方案+容灾方案+replication（主从+读写分离）+sentinal（哨兵集群，3个节点，高可用性）
如果你的数据量不大，单master就可以容纳，一般来说你的缓存的总量在10G以内就可以，那么建议按照以下架构去部署redis

### redis cluster
如果你的数据量很大，比如我们课程的topic，大型电商网站的商品详情页的架构（对标那些国内排名前三的大电商网站，*宝，*东，*宁易购），数据量是很大的













