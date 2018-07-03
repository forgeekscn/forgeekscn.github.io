---
layout:  post
title:		缓存架构之Kafka+Zookeeper实现缓存数据生产服务
subtitle:		缓存架构之Kafka+Zookeeper实现缓存数据生产服务 
date:     2018-07-02
author:   Fogeeks
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Redis
    - Kafka
    - Zookeeper
---
 
#	缓存架构之Kafka+Zookeeper实现缓存数据生产服务

对于实时性不高的数据我们采用缓存生产服务来操作，可以抽象成一个是**生产者消费者**模式，可以监听一个kafka消息队列，一旦有该类数据的变更请求，那么该消息就会推送到kafka队列，缓存数据生产服务可以去消费这个消息，去数据源拉取数据，然后刷新到ehcacheh和redis中，这里的zookeeper集群主要是用来管理kafka集群，java程序里面先配置好，写好数据生产变更服务，监听zookeeper的端口来实现监听kafka消息队列，下面可以去linux模拟下这个过程

``生产者创建一条数据变更请求消息``

```shell
[root@192 kafka]# bin/kafka-console-producer.sh --broker-list  192.168.111.130:9092,192.168.111.131:9092,192.168.111.132:9092  --topic cache-message
{ "serviceId":"productInfoService","id": 200,  "name":  "iphone7",  "price": 5599,  "pictureList": "a.jpg,b.jpg ",  "specification":  "iphone7  ",  "service":  "iphone7  ",  "color":  "red ",  "size":  "5.5 ",  "shopId": 300}
```

`` 生产服务检测并消费该消息 ``

```text
 #######################
 kafka传过来的信息：{ "serviceId":"productInfoService","id": 200,  "name":  "iphone7",  "price": 5599,  "pictureList": "a.jpg,b.jpg ",  "specification":  "iphone7  ",  "service":  "iphone7  ",  "color":  "red ",  "size":  "5.5 ",  "shopId": 300}
 ###########################
json转换之后的objProductInfo [id=200, name=iphone7, price=5599.0, pictureList=a.jpg,b.jpg , specification=iphone7  , service=iphone7  , color=red , size=5.5 , shopId=300, modifiedTime=null]
 ################################
 将商品信息保存到本地的ehcache缓存中：ProductInfo [id=200, name=iphone7, price=5599.0, pictureList=a.jpg,b.jpg , specification=iphone7  , service=iphone7  , color=red , size=5.5 , shopId=300, modifiedTime=null]
 ##################################
 获取刚保存到本地缓存的商品信息：ProductInfo [id=200, name=iphone7, price=5599.0, pictureList=a.jpg,b.jpg , specification=iphone7  , service=iphone7  , color=red , size=5.5 , shopId=300, modifiedTime=null]
 ################################
将商品信息保存到redis中：ProductInfo [id=200, name=iphone7, price=5599.0, pictureList=a.jpg,b.jpg , specification=iphone7  , service=iphone7  , color=red , size=5.5 , shopId=300, modifiedTime=null]
```

 



 