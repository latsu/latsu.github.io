---
layout: post
category: ['MySQL']
title: MySQL Replication 主从同步原理
---

![Alt text](/res/img/in_posts/20639775_1340695881N7L6.jpg)

一共有三个线程：

1. 主库上的 IO 线程
2. 从库上的 IO 线程
3. 从库上的 SQL 线程

大致描述一下过程：

从库的 IO 线程从主库获取二进制日志，并在本地保存为中继日志，然后通过SQL线程来在从上执行中继日志中的内容，从而使从库和主库保持一致。

1. 主库验证连接。
2. 主库为从库开启一个 IO 线程。
3. 从库将主库日志的偏移位告诉主库。
4. 主库检查该值是否小于当前二进制日志偏移位。
5. 如果小于，则通知从库来取数据。
6. 从库持续从主库取数据，直至取完，这时，从库线程进入睡眠，主库线程同时进入睡眠。
7. 当主库有更新时，主库线程被激活，并将二进制日志推送给从库，并通知从库线程进入工作状态。
8. 从库SQL线程执行二进制日志，随后进入睡眠状态。

详细的基本交互过程：

1. slave 端的 IO 线程连接上 master 端，并请求从指定 binlog 日志文件的指定 pos 节点位置（或者从最开始的日志）开始复制之后的日志内容。

2. master 端在接收到来自 slave 端的 IO 线程请求后，通知负责复制进程的 IO 线程，根据 slave 端 IO 线程的请求信息，读取指定 binlog 日志指定 pos 节点位置之后的日志信息，然后返回给 slave 端的 IO 线程。该返回信息中除了 binlog 日志所包含的信息之外，还包括本次返回的信息在 master 端的 binlog 文件名以及在该 binlog 日志中的 pos 节点位置。

3. slave 端的 IO 线程在接收到 master 端 IO 返回的信息后，将接收到的 binlog 日志内容依次写入到 slave 端的 relaylog 文件（mysql-relay-bin.xxxxxx）的最末端，并将读取到的 master 端的 binlog 文件名和 pos 节点位置记录到 master-info（该文件存在 slave 端）文件中，以便在下一次读取的时候能够清楚的告诉 master “我需要从哪个 binlog 文件的哪个 pos 节点位置开始，请把此节点以后的日志内容发给我”。

4. slave 端的 SQL 线程在检测到 relaylog 文件中新增内容后，会马上解析该 log 文件中的内容。然后还原成在 master 端真实执行的那些 SQL 语句，并在自身按顺丰依次执行这些SQL语句。这样，实际上就是在 master 端和 slave 端执行了同样的 SQL 语句，所以 master 端和 slave 端的数据是完全一样的。

以前的 MySQL 里，Slave 端的复制是由单独的一个 IO 线程来完成的（`简而言之就是没有用 SQL Relay 队列`）。这样存在几个问题：

1. 从库上的 IO 线程要串行完成一系列操作：拉取+解析+执行 master binlog，执行时间比较长。

2. 执行延迟越长，主库数据丢失的风险就越大，主库是希望从库尽可能零延迟、无间歇地来拉取数据的。一旦存在较长拉取延迟，那么在此期间 master 挂了，还没来得及被从库拉取的那一部分数据，将永远的丢失，所以我们要想办法尽可能地提高从库拉取数据的频次和效率。

3. 新版将 Slave 端的复制改为两个线程来完成，IO 线程只负责从主库读取 SQL 并推到中继日志队列里就完事，SQL 线程则只负责从中继日志队列里依次弹出并执行SQL。这样缩短了从库的延迟时间，就减少了潜在的丢失数据的风险。

4. 当然，即使是换成了现在这样两个线程来协作处理之后，Slave 仍然存在丟数据的可能，毕竟复制是异步的，只要数据的更改不是在一个事务中，这些问题都是存在的。