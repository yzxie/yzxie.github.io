---
layout:     post
title:      "Redis主从同步"
subtitle:   "Redis republication"
date:       2018-11-11 16:49:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - Redis
---

## 概述

在高并发服务当中，如果使用单个Redis实例，由于Redis采用单进程单线程处理所有请求的方式，即每次只有一个请求在处理，后面的请求排队，如果前面请求执行时间长了，则会影响后面所有请求。所以可以拓展到多个Redis实例，采用主从机制，一个master和多个slave，master和多个slave包含相同的数据，master负责处理写请求，slave负责读请求。Redis主从同步即是实现这种机制的，master处理完写请求后，同步给多个slave，从而保证数据的最终一致性。

## 同步架构

Redis的主从同步支持两种机制，一种为master同步给所有slave，另外一种slave级联同步，即master同步给一个slave，这个slave再充当伪“master”同步给下一个slave，这种机制的好处是减少master的数据传输量，节省master的带宽，同时在新slave加入时或者多个slave重连时，避免全部需要master来接收，造成master资源紧张，如fork子进程执行BGSAVE。具体架构如图：

![](/img/articles/20181111/p1.jpg)

配置某个redis使用另外一个redis作为master也很简单，redis.conf的配置如图：同时slave默认也是只读的，但是可以接收并执行master同步过来的写命令进行数据修改，保持与master一致。

![](/img/articles/20181111/p2.jpg)

## 全量同步

### SYNC

全量同步：slave启动或者slave断开重连master的时候，slave会发生SYNC命令给master，master接收到该命令后，则会通过bgsave启动一个子进程将当前时间点的master全部数据的快照写到一个文件中，然后发送给slave。slave接收到之后则清空内存，载入这些数据。具体过程如下：

1 slave发生SYNC命令给master，请求执行全量同步，然后slave在等待期间，根据配置（如下）可以使用旧数据响应客户端请求或者直接报错；

![](/img/articles/20181111/p3.jpg)

2 master接收到SYNC请求后，通过BGSAVE启动一个子进程将当前数据快照写到一个快照文件；同时主进程采用一个缓冲区保存这段时间内的所有写请求；注意如果同时有多个slave发送SYNC给master，master只会执行一次BGSAVE，然后将快照文件发送给这些slaves；

3 子进程完成快照文件的写入，向slave发送这个快照文件（正常逻辑来说是子进程发送待查看源代码确认）；主进程继续采用缓冲区缓存写命令；slave接收到快照文件后，删除旧数据，载入新的快照数据，此期间会阻塞客户端的请求；

4 子进程发送快照文件完毕后，主进程将缓冲区的数据以Redis命令协议形式，如DEL、SET，发送给slave。slave载入完快照数据后，则可以开始处理客户端的请求了。同时slave接收master发送过来的缓冲区的写命令并执行；

5 此后进入增量同步模式，不断接收master的写请求，实现最终一致性。

## 增量同步

增量同步即为master每接收并执行一个写命令都同步给所有的slave，slave接收到该写命令后执行对自身数据的修改从而保持与master的数据一致。

### PSYNC

在Redis2.8+版本，Redis的slave在与master断开连接重连的时候，默认是使用新的PSYNC同步方法，而不是原来的SYNC，因为断线重连时，slave是包含有数据的，只是可能落后于master，所以没必要又进行一次全量同步。PSYNC的实现具体为：
命令格式：
```
PSYNC <runid> <offset>
runid:主服务器ID
offset:从服务器最后接收命令的偏移量
```

1. run_id：master的id，表面master身份，slave使用该slave来记住自己目前是同步哪个master；如果slave没有保存，则发送PSYNC ? -1，执行全量同步；数值为40位的十六进制字符串，Redis重启后会改变，从而保证重启后，slave使用全量复制，保证数据安全性。

2. replication offset：即当前执行主从同步到哪里了，master维护自身当前同步了多少给slaves，即发送了N个字节给slave，则会执行offset+N；slave维护自身当前从master同步了多少，通常slave的小于或等于master的；

3. 复制积压缓冲区：即master已同步写命令缓存列表，master以FIFO的方式保存最近同步给slave的数据，列表大小一定，所以如果slave重连时，slave的replication offset月master的replication offset的差值大于该列表大小，则说明slave丢失的部分数据（写命令）不能从该列表获取了，此时需要执行全量同步，否则master将这个差值对应列表中的数据发送给slave，slave只需执行这些写命令即可。
    
Redis4.0推出了PSYNC2.0，具体特性再分析。

## CAP理论

CAP理论为：在分布式系统中，多个节点之间只能满足CP或AP，即强一致性和高可用是不能同时满足的。Redis的主从同步是AP的，具体对高可用的强度要求，可用通过在redis.conf配置，即有至少有多少个slaves存在和至多多少秒内没响应，则才执行写请求，否则报错，配置与说明如图：默认为关闭这个特性，即master始终接收客户端写请求。
![](/img/articles/20181111/p4.jpg)