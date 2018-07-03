---
layout:     post
title:      Redis缓存架构之 redis-cluster VS replication-sentinal
subtitle:   Redis缓存架构之 redis-cluster VS replication-sentinal
date:       2018-06-16
author:     Fogeeks
header-img: img/post-bg-re-vs-ng2.jpg
catalog:    true
tags:
    - Redis
---

# redis cluster vs. replication + sentinal

大数据高并发场景下，传统的关系数据库已经无法支撑如此频繁大量的读写请求，毕竟数据库读写io十分耗内存，除了考虑数据库性能优化sql优化之外，这时候还要考虑更加高效合理的方法应对高并发，
此时就要用redis缓存来减轻数据库压力，缩短请求响应时间，提高并发量。

#### 单机redis性能分析
下面测试下单机redis的性能，虚拟机centos系统master的内存2G单核，下面测试下`` [root@192 redis]# redis-benchmark -d 1024  -h master -p 6379 ``
requests per second结果如下
get|set|lpush
---|:--:|---:
34094.78|34364.26|31259.77
可以看到简单场景下单机的读写qps都可以上万，且99%的请求都能再2ms内相应，那么根据28法则，将2成热数据放入redis，性能的提升对比成本是非常可观的，

但是引入了redis又会出现新的问题，例如单机部署的redis突然宕机了，所有读写请求都打入mysql，情况严重的话MySQL会裸奔，即使mysql可以承受，如果没有**redis灾备恢复**，宕机期间打入redis中的写请求的数据也会丢失，

这时要同时开启aof和rdb，默认是rdb，rdb是默认的隔指定时间检查增量 `` save 900 1 save 300 10 save 60 10000 ``, 他会检查近60s的增量数据是否达到10000，redis一崩近一分钟的数据全都没有了，
所以如果对于数据要求高的要配置更加频繁rdb的备份，同时还要开启aof备份，开启`appendonly yes` 并且每秒写入一次`` appendfsync everysec ``，这时就能保证做多只有1s的数据丢失。

同时还要做好小时级别和天级别的备份rdb和aof文件，写两个脚本crontab定时执行，最好每天上传到云服务器，小时级别备份前删除前一天的历史数据，天级别的备份前删除前一个月的数据，这样就不会占用太多硬盘空间，一旦redis崩了可以直接拉取最新的备份文件放入指定文件夹，重启redis让其开始恢复。

#### replication
即使这样还是会在备份时导致系统不可用，这时候就要考虑redis集群，采用**redis主从架构读写分离**，运用redis自带的replication模式，master写slave读，支持**横向扩容**，更新时master会将数据异步同步到slave，新的写请求也可以继续打入master，那边复制完了这边把新的数据再次发过去同步，不会阻塞master写和slave的读请求，只是slave接收到同步请求后会删掉就数据覆盖新的数据，这时候会短暂暂停服务一会但是这个过程是很快的，这样不仅可以大大提高读并发量，还可以避免宕机带来的风险，如果某个slave宕机只会降低读并发。

但是如果master宕机这里还是会有丢失数据的风险，因此这种方法不算完美。

#### sentinal
replication主从架构虽然提高读并发降低宕机带来的风险，但是仍然不能保证系统的**高可用**，这里就要引入**哨兵sentinel**，他可以用来**监控集群**，一个sentinel监控一个redis实例，一个sentinel集群监控一个主从架构，经典的哨兵集群最少3个哨兵节点，哨兵集群可以监控master主从架构，即使master宕机，其下slave会通过** 连接时间，offset，优先级，runid**等来投票选举出新的master，这就是所谓的**故障转移**，而且需要达到**quorum**数量的节点主观觉得master崩了**sdown**了那就代表它真的**odown**了，这时需要一个大于quorum的数量的**majority**的节点给你投票了你才能选举上，这里就说最少三台，因为两台的话他的是quorum=1，mojority=2，表示起码要1个人说master老大凉了要2个人投你当老大才行，宕机一个就剩一个，最多一票，这样全真的凉了。三台的话quorum=2，majority=2，master老大凉了，两个小弟都发现了，所以达到quorum数量，再公正客观选举出适合的那个两个人都给投票，就达到了majority的数量投票了，正好，这里的majority是固定的，节点<5都是majority=2，quorum是可以设置。

 新问题又来了，多个节点的话，如果master宕机，虽然会重新选举出新的master，但如果正好这时正在向slave异步复制数据你给宕机了，如果没有配置好，**主从复制**没复制完的数据全丢失。
 
 还有一种丢失数据的情况，就是master没有宕机，但是由于网络故障master和slave同步数据非常慢超过了设置的timeout，或者slave长时间根本就没有接收到master的**ack同步命令**，这时候slave也会主观认为您凉了，他们就开始选举新的master，选出来之后就有两个master存在了，这就是所谓的**脑裂**，关键是此时客户端的写请求仍然会打入旧的master，他觉得能写入没超时就ok，过段时间检测到脑裂问题再一重启redis，他会以slave身份加入集群，新的master存的是旧的数据，给他发一个同步命令，旧的master存入的数据全没了

这时要去配置文件配置两个参数 ``min-slaves-to-write 1`` ,  ``min-slaves-max-lag 10``  表示master最少一个slave，master主从复制延迟超过10秒他就不会接受写请求了，这样的话脑裂问题和主从复制问题都可以解决

这样看起来就完美了，哨兵实现了高可用，提高了读并发，但是如果数据量很大情况会出现什么，master主从架构瓶颈在哪里，在数据量，master能写入多少slave就复制多少，也就是说要提高写吞吐他就不灵了该怎么办，下面就有了rediscluster

查看哨兵集群状态
>[root@192 redis]# redis-cli -h master -p 5000

```shell
master:5000> SENTINEL sentinels mymaster
1)  1) "name"
    2) "4225b366cd995fdae172c568314722d973d363c1"
    3) "ip"
    4) "192.168.111.131"
    5) "port"
    6) "5000"
    7) "runid"
    8) "4225b366cd995fdae172c568314722d973d363c1"
    9) "flags"
   10) "sentinel"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "900"
   19) "last-ping-reply"
   20) "900"
   21) "down-after-milliseconds"
   22) "30000"
   23) "last-hello-message"
   24) "1121"
   25) "voted-leader"
   26) "?"
   27) "voted-leader-epoch"
   28) "0"
2)  1) "name"
    2) "eec4acf75da8a296414fee8b1865989f6591b191"
    3) "ip"
    4) "192.168.111.132"
    5) "port"
    6) "5000"
    7) "runid"
    8) "eec4acf75da8a296414fee8b1865989f6591b191"
    9) "flags"
   10) "sentinel"
   11) "link-pending-commands"
   12) "0"
   13) "link-refcount"
   14) "1"
   15) "last-ping-sent"
   16) "0"
   17) "last-ok-ping-reply"
   18) "900"
   19) "last-ping-reply"
   20) "900"
   21) "down-after-milliseconds"
   22) "30000"
   23) "last-hello-message"
   24) "295"
   25) "voted-leader"
   26) "?"
   27) "voted-leader-epoch"
   28) "0"
```

#### redis-cluster
cluster集群站在一个更高的高度，他支持多个master+slave，那么数据量大了我就横向扩容master节点就行了，同时自动将数据分片，将所有数据分成16384个**hashslot**，再根据平均分配到各个master，任何一个节点发生改变如宕机他都会检测到，有master宕机他就把他下面的slave挂到其他master，然后将它上面的hashslot分到其他master，节点之间用了一种二进制的协议gossip通信，效率更高

那么hashslot具体是怎么分配的数据该怎么查，这里要介绍三个算法，最老土的算法是**hash算法**，计算key的hash值对master节点数取模，数据就存入对应master，显然有问题，如果某个master宕机，master节点数就变了，在有读请求过来对key取hash再对新的节点数取模，他很可能就到错误的master查，肯定是查不到的，查不到就去mysql，导致大部分数据查不到，那么就会产生大量的缓存重建，甚至是mysql宕机

这时候就要优化，就有了**一致性hash算法**，master节点都分配在一个虚拟的环上面，他们之间的区间代表key对应的hash区间，读请求过来，也会先计算hash再对节点取模，然后会计算落到那个区间，这时他就会顺时针去最近的master节点查数据，正常都能查到，如果有master宕机，虚拟环的结构还在，他就会跳过这个master读取下一个master，肯定也是读不到的，但是也只是这一个master 的数据会缓存重建，有局限性

接下来就是redis-cluster采用的算法**hashslot算法**，任何一个master宕机都对其他没影响，因为数据分成16384个hashslot，key取crc16再对16384取模就得到对应hashslot，你宕机了你的hashslot会被分配到其他的master，不用担心数据丢失问题

rediscluster支持**横向扩容**，可以手动加入或者删除master和slave，也可以对数据**reshard重新分片**

查看cluster集群状态
>[root@slave1 hello]# redis-trib.rb check master:7001

```shell
M: 703d497c41334dd946bd5456bb941102230d9aea master:7001
   slots:1333-5460 (4128 slots) master
   1 additional replica(s)
S: be1b6a8e84de078b10748a0f93b192f02efb2056 192.168.111.132:7005
   slots: (0 slots) slave
   replicates 28ca5c85a9cd570cfa32bf7d1d1197e1420948b8
S: 11c52136fcdcee5410966e238f19fd86674ae047 192.168.111.132:7008
   slots: (0 slots) slave
   replicates d5e816750cc857971a9fce4ded05043e855d9b59
M: 2323b351e52ea209cf71f883d59e1deed595c0d9 192.168.111.130:7002
   slots:6795-10922 (4128 slots) master
   1 additional replica(s)
M: d5e816750cc857971a9fce4ded05043e855d9b59 192.168.111.132:7007
   slots:0-1332,5461-6794,10923-12255 (4000 slots) master
   1 additional replica(s)
M: 28ca5c85a9cd570cfa32bf7d1d1197e1420948b8 192.168.111.132:7006
   slots:12256-16383 (4128 slots) master
   1 additional replica(s)
S: 995a76165b569bf49395deb74591146c66154a49 192.168.111.131:7004
   slots: (0 slots) slave
   replicates 703d497c41334dd946bd5456bb941102230d9aea
S: 4960fe405854250d277a6bb060bfc5b8fe02d757 192.168.111.131:7003
   slots: (0 slots) slave
   replicates 2323b351e52ea209cf71f883d59e1deed595c0d9
[OK] All nodes agree about slots configuration.
[OK] All 16384 slots covered.
```

#### 总结
如果数据量不大，单master就可以容纳，一般来说缓存的总量在10G以内就可以，应该可以按照以下架构去部署redis
redis持久化+备份方案+容灾方案+replication（主从+读写分离）+sentinal（哨兵集群，3个节点，高可用性）
如果你的数据量很大，  就用 redis cluster 做高可用高并发架构。













