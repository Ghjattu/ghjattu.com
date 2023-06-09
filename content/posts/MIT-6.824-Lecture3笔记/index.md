---
title: "【MIT 6.824】2021-Lecture3 学习笔记"
date: 2023-06-07T16:17:10+08:00
lastmod: '2023-06-09'
author: 'Ghjattu'
slug: 'mit-6824-lecture3-notes'
categories: ['Distributed System']
description: "MIT 6.824(现6.5840)2021 Spring 课程Lecture3的学习笔记，内容主要包含了GFS(Google File System)的简要架构介绍，GFS中读写操作的流程图和分析。"
tags: ['MIT 6.824']
---

GFS(Google File System)是一个可扩展的，更关注大文件管理、顺序读写和高吞吐量的分布式文件系统，它提供了容错机制和并发控制，具有高扩展性、高可靠性和高可用性等特点，并且具有不错的性能表现。

> 分布式存储系统的难点在于：最开始时把文件存储在多个服务器上做到并行读取，提高吞吐量，但是众多服务器总是会出现个别服务器出错的情况，一个解决方法是对同一个文件存储多个副本，同时这又带来了一致性的问题，不得不使用各种协议来保证同一个文件的多个副本总是保持一致的，但这又会降低性能，所以，找到性能和一致性的平衡点是一件具有挑战性的事情。

## 架构

一个 GFS 集群通常由一个 master 和多个 chunkserver 组成，可以被多个客户端访问。

![GFS Architecture](./GFS-Architecture.jpeg)

根据 Figure1 可以把一次**读取操作**简要概括成：由于文件被分成了相同大小的块存储，因此客户端可以轻易将应用程序指定的字节偏移量换算成相应的块索引，然后客户端把文件名和块索引发送给 master，master 收到信息后在文件树内搜索，返回块句柄和副本的位置，客户端以文件名和块索引为 key 缓存这条信息，然后根据返回的副本位置选择一个 chunkserver，**通常是最近的那个**，向它发送一个包含块句柄和块内字节区间的请求，之后对相同块的读取都是和 chunkserver 交互完成，不需要再次和 master 交互，除非客户端的缓存过期。

此外，客户端通常在一次请求中请求多个块，master 一次性返回这些块的信息，这样做减少了客户端和 master 在未来可能发生的通信开销。

### 单个主服务器(Single Master)

master 作为整个集群的协调者(coordinator)，**客户端从不通过 master 读写文件数据**，而是向 master 询问相应的文件块在哪个 chunkserver 上，然后直接与 chunkserver 交互完成后续的操作。 master 虽然尽可能地不参与读写操作，但是仍会把整个集群的各项操作保存在日志中用于崩溃后的恢复。

### 文件存储

文件被分成若干个 64MB 的块(chunk)保存在 Linux 文件系统中，每个块由一个不可变的全局唯一的 64bit 的块句柄(chunk handle)标识，块句柄是块在创建的时候由 master 赋予的。为了实现高可靠性，每个块会在多个 chunkserver 上保存副本，通常将每个块复制两次，得到一共三个副本保存在三个 chunkserver 上。

选择一个较大的块大小带来了以下优势：因为应用程序更可能在某一个大的数据块上执行多次操作，所以这减少了客户端和 master 的交互次数，并且可以和相应的 chunkserver 维持长时间的 TCP 连接来减少网络开销；此外较大的块大小也有助于减少保存元数据所需要的存储空间，使得能够将元数据加载到内存中。

而劣势在于，一个小文件可能只需要一个块来存储，当很多个客户端都请求这个小文件，存储这个小文件的 chunkserver 就会变成热点(hot spot)，增大了服务器的负担。

### 元数据(Metadata)

master 保存了三种类型的元数据：**文件和块命名空间**、**文件到块的映射**、**每个块的副本位置**，元数据都保存在内存中，因此 master 的操作可以很快完成。此外，master 可以在后台周期性地扫描记录状态，从而实现块的垃圾回收，在 chunkserver 故障时及时重新复制(re-replication)副本，以及为了均衡负载和磁盘空间所需要的块迁移操作。如果需要支持更大的文件系统，额外增加内存会是相比之下成本更小的方法。

前两种类型会通过将变更记录存储到 master 的本地磁盘的操作日志中而持久化保存，并且会发送到远端机器，使得 master 在崩溃后能恢复到最近的状态。master 不会持久化保存块副本位置信息，而是在启动时轮询 chunkserver 来获取这些信息，并且通过周期性的心跳(heartbeat)消息来保持最新状态，更具体的，master 中只会保存每个块副本在哪些 chunkserver 上，块在 chunkserver 上的具体位置信息则由 chunkserver 自己维护。

### 操作日志

操作日志是 GFS 的核心，它持久化保存了关键元数据的变更记录，同时它还说明了并发操作顺序的逻辑时间线。出于操作日志的重要性，需要将操作日志在远端机器上复制多次。并且，在操作日志在本地和远端机器都被刷入(flush)之后才能响应客户端的操作，否则会造成客户端操作丢失的情况（比如，先响应客户端操作创建文件，后写入日志，当 master 在创建文件之后故障，由于日志中并没有这次创建文件的记录，那么恢复之后这个新文件就丢失了。），为了提高效率，master 通常会把多个变更记录集中在一次会话中刷入。

为了最小化 master 的恢复时间，master 会定期对日志的状态生成一个检查点(checkpoint)，当故障发生后，master 从磁盘上读取最新的检查点，然后依次执行检查点后面的日志记录。

### 文件变更

变更是更改块内容或元数据的操作，比如写操作和追加操作，变更在每个块的所有副本上都要执行，GFS 使用租约(lease)来保证在所有副本上变更顺序的一致性。master 把一个块租约赋予给其中一个副本，这个副本就成为了 primary，其余副本成为 secondary，primary 为块上的所有变更定义了一个顺序，所有副本都需要遵循这个顺序。

一个租约初始有 60s 的时限，但是当块发生变更时，primary 可以向 master 申请延时，延时请求和应答都依附于 master 和 chunkserver 之间周期性交换的心跳消息。master 也可能在租约过期前主动将租约撤销。

为了能及时检测到过期的副本，master 为每个块维护了一个**块版本号**(chunk version number)来区分最新的和过期的副本。master 在赋予一个块租约时，会先增加版本号并通知所有的副本，master 和副本都**需要持久化保存这个版本号**。

![Write Control and Data Flow](./Write-Control-and-Data-Flow.png)

Figure 2 展示了 GFS 中**写操作**的流程：

1. client 向 master 询问哪个 chunkserver 持有块的租约和副本的位置，如果没有 chunkserver 持有租约，master 会首先增加块版本号，随机选择一个赋予租约成为 primary，通知 primary 和 seconda 块版本号变更。
2. master 返回 primary 和 secondary 的位置信息以及最新的块版本号，client 缓存这份信息，后续只有当 primary 不可达或 primary 声称不再持有租约时，client 才需要和 master 沟通。
3. client 把数据推送到副本，推送的顺序是不定的，不必先从 primary 开始，**副本收到数据后先保存在缓存中**。
4. 一旦所有副本都确认收到了数据，client 就会向 primary 发出一个写请求，这个请求标识了之前推送的数据。primary 收到请求后检查块版本号是否匹配以及自己的块租约是否有效，若存在条件不满足则拒绝写请求。由于 primary 可能收到了多个 client 的请求，因此 primary 为每个请求分配一个序列号，然后按照序列号的顺序更新自己的本地状态。
5. primary 转发写请求和相同的序列号到所有的 secondary，secondary 也按照序列号顺序更新状态。
6. secondary 回复表明自己完成了操作。
7. primary 回复 client，并表明过程中发生的任何错误。当发生错误时，文件变更已在 primary 和 seconda ry 的一个子集上执行完毕，这造成了副本间的不一致状态，这个问题会交给 client 处理，一般是重试几次操作(3)到操作(7) 。



Figure 2 也表明了，控制流是**从 client 到 primary 再到 secondary** ，而数据流是**以流水线的方式在某个 chunkserver 链上线性推送**。

为了充分利用机器的网络带宽，每台机器的全部出带宽(outbound)都用于向下一台机器尽可能快地传输数据，而不是分发给多个接收者。同时，为了避免网络瓶颈，每台机器都会把数据发送给链上离它最近(closest)且未收到数据的机器。
