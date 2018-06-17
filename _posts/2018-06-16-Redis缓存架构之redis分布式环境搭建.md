---
layout:     post
title:      Redis缓存架构
subtitle:   Redis缓存架构之Redis分布式环境搭建
date:       2018-06-16
author:     Fogeeks
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Redis
---


# 前言

### LRU缓存清除算法
 

### rdb 
save 5 1

### aof 
appendonly yes
appendsync always/no/eveysec
rewrite


### 灾备演练
时间层级备份 ，rdb冷备 
停掉redis ，复制rdb ，重启redis ， 手动开启aof













