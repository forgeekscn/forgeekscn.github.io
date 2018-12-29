---
layout:  post
title:		Concurrent包中的四种Lock锁的适用场景深入研究
subtitle:		Concurrent包中的四种Lock锁的适用场景深入研究
date:     2018-07-19
author:   Fogeeks
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - 锁
    - 并发安全
---
 
#	Concurrent包中的四种Lock锁的适用场景深入研究
 计数器 等待n个线程全部执行完成才解除阻塞
 栏栅 设置一个切面等待特定数量的线程到达才解除阻塞
 信号量 把有限的资源提供给多线程竞争
 互斥 只有一个资源的特殊的信号量模式
 
 reentranLock 可重入锁 可以被线程重复获取的锁
 readWriteLock 读写锁 读锁可以被多个线程持有但是写锁只能被一个线程持有 如数据库隔离级别read commited用的就是读锁共享写锁互斥
 
 
 
 
 
  
 
 
 