---
layout:     post
title:		缓存架构之 redis+ehcache+nginx三级缓存
subtitle:	缓存架构之 redis+ehcache+nginx三级缓存
date:       2018-07-01
author:     Fogeeksw
header-img: img/post-bg-re-vs-ng2.jpg
catalog:    true
tags:
    - Redis
    - Ehcache
    - Nginx
---

# 缓存架构之 redis+ehcache+nginx三级缓存

缓存架构中要更好地支撑起大数据高可用高并发的需求，除了要对实时性高的数据做实时的缓存+数据库双写一致之外，还要对实时性不高的数据采取三级缓存架构来处理。
而且一般这些实时性不高的数据都是比较大的，如果将大型缓存直接存入redis会产生缓存更新问题，会严重拉低redis性能，我们可以把他们拆分成不同维度分别存取，也就是所谓的** 缓存维度化 **，这样可以有效减轻redis存取压力。

#### ehcache

三级缓存中的ehcache作用是什么，就是redis缓存出现了大面积瘫痪，这时候需要有本地堆缓存来作为最后一道防线，防止**mysql裸奔**
作为一个java缓存框架， ehcache可以为系统提供坚强的后盾，提高可用性

当有一个读请求过来时，先去redis找，可能由于**lru最近最少使用算法**淘汰了找不着，就去ehcache找，找到了就相应，找不着就去mysql查，查到了就依次刷新到ehcache和redis
当有更新请求过来时候，直接删除redis和ehcache缓存，更新之后在依次刷新就可以了

#### nginx

可以把请求都先打入nginx让他先做个初步的缓存，而且还要设置好**nginx分发层和nginx应用层**，把缓存压力分发给各个nginx节点，同时还要设置缓存过期时间，因为nginx缓存最好不要放太多东西
这里可以用**openResty**来部署nginx，同时配合**lua脚本**可以完成这些缓存操作和请求分发操作

 
 

 













