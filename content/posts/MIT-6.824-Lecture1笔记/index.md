---
title: "【MIT 6.824】2021-Lecture1 学习笔记"
date: 2023-05-31T16:19:08+08:00
lastmod: '2023-06-01'
author: 'Ghjattu'
slug: 'mit-6824-lecture1-notes'
categories: ['Distributed System']
description: "MIT 6.824(现6.5840)2021 Spring 课程Lecture1的学习笔记，内容主要包含了分布式系统的简单定义和特点，MapReduce的执行流程、数据流向、容错机制以及一些可能的改进系统性能的方法。"
tags: ['MIT 6.824']
---

## 分布式系统

一个分布式系统是若干自治的计算机系统的集合，这些计算机系统在物理上是分离的，但是系统之间可以互相通信，通过共享资源来共同完成某一个目标，在用户看来就好像只有一台计算机在为自己服务。

一个分布式系统具有以下几个特点：

1. 容错能力，通过在多个计算机上保存数据或服务副本，即使某一台计算机故障，整个系统也能正常工作，并且故障的计算机能使用日志和事务技术快速的恢复。
2. 并行计算，多台计算机可以同时执行相同的任务，增加了吞吐量。
3. 可伸缩性，可以适当地增加和减少计算机来适应请求量的变化。
4. 低延迟，对于用户的一个请求，系统会尽量选择距用户物理距离短的计算机来响应，如果是多台计算机同时完成任务的场景，如果某一台计算机因为某些原因长时间不能结束，系统会另外分配一台计算机来完成它的任务。

## MapReduce

MapReduce 是一种编程模型，它用于处理和生成大规模的数据集，粗略地讲，它首先将数据集分成若干个块(chunk)，预处理成键值对的形式，然后启动多个服务器，运行用户的 map 函数并行地生成中间键值对，之后，模型从服务器中读取中间键值对，交给用户指定的 reduce 函数，合并处理具有相同 key 的 value ，最后将合并后的最终数据返回给用户。

### 执行概述

从 Google 的论文 《MapReduce: Simplified Data Processing on Large Clusters》中可以了解到，初始数据都保存在 GFS （Google File System，解决了海量超大文件的分布式存储问题）服务器中，每个服务器还有两到三个备份，MapReduce 库将应用代码发送到多个服务器上，每个Map 和 Reduce 任务都是在这些服务器上完成的。每个 mapper 从 GFS 并行读取文件，然后开始并行处理，生成的中间键值对保存在本地磁盘上，reducer 通过 RPC 从 mapper 的本地磁盘上读取中间键值对（这里是唯一需要网络通信的地方），处理完成后直接保存到 GFS 中。因此，整个过程的数据流向是：GFS $\to$ mapper $\to$ mapper 的本地磁盘 $\to$ 网络 $\to$ reducer $\to$ GFS 。

下面是具体的流程：

1. **Split** ：GFS 使用许多个 chunk 来存储文件，一个 chunk 的大小是 64MB ，输入文件保存在 GFS 中，也即被拆分成了若干个 chunk ，每个 chunk 都由一个 mapper 来负责。用户也可以通过可选参数来划分成更小的块。
2. **Map** ：在 coordinator 的协调下，每个 mapper 从 GFS 中读取自己负责的部分。下面均以单词计数任务为例，mapper 将输入文件预处理成 <key, value> 的形式，key 表示文档名，value 表示文档内容，然后作为参数调用用户定义的 map 函数，用户定义的 map 函数将文档内容转换成许多中间键值对，这些中间键值对形如 <word, 1> ，表示 word 这个单词出现了一次。
3. **Combine** ：Combine 操作是可选的，它是在 mapper 中完成的。它先把中间键值对按照 key 排序，然后把具有相同 key 的中间键值对的 value 合并在一起，例如在经过 Map 操作后产生了 <word, 1>、<word, 1>、<word, 1> 这样三个中间键值对，那么就可以把它们合并成一个中间键值对 <word, 3> 。在数据量很大的情况下，Combine 操作可以有效减少网络传输量。
4. **Partition** ：Partition 操作也是在 mapper 中完成的。假设有 $R$ 个 reducer，那么会在每个 mapper 的本地磁盘上开辟 $R$ 个分区。上述操作生成的中间键值对一开始缓存在内存中，每隔一段时间，通过对中间键值对的 key 执行 hash 函数把这些键值对写到本地磁盘上的 $R$ 个分区中。写回的位置会传回 coordinator，coordinator 负责将这些位置转发给 reducer 。通常 $R$ 的取值远小于 key 的种类，因此同一个分区中也会包含很多不同的 key 。$R$ 的值和 hash 函数都由用户指定。
5. **Reduce** ：当 reducer 收到 coordinator 发来的位置信息，reducer 通过 RPC 读取每个 mapper 的本地磁盘中属于自己的分区（这个阶段叫做 Shuffle ），因为不同 mapper 间的中间键值对是无序的，所以需要对这些中间键值对再进行一次按照 key 的排序，接着调用用户定义的 reduce 函数，reduce 函数迭代所有的中间键值对，把具有相同 key 的 value 相加，生成最终的计数键值对，reducer 再把这些最终的键值对保存到 GFS 中。

>关于 Reduce 任务是否要在全部 Map 任务完成之后在执行的问题，知乎上也有一个相同的问题（参考资料 2），一般认为要等到所有的 mapper 把数据处理保存完之后再开始 Reduce 任务，Lecture 1 的视频中也有讲到 Reduce 在 Map 之后，Lecture 1 的 Introduction 也有这样一句："after all Maps have finished, coordinator hands out Reduce tasks."。即使两个任务的执行时间有重叠，也是 Reduce 先拉取数据（Shuffle），但正式调用用户的 reduce 函数仍要等待所有 mapper 完成。

在成功完成任务后，MapReduce 的结果会被保存在 $n$ 个输出文件中，每个 reducer 都会产生一个文件，文件名由用户指定。一般来说用户不需要手动将它们合并成一个文件，这些文件可能会传入另一个 MapReduce 调用，或在另一个能处理多个文件的分布式应用中被使用。

### 容错机制

#### Worker故障

coordinator 会周期性地 ping 一下每个 worker 。

对一个运行 Map 任务的 worker 来说，它在一段时间只处理一个 map 任务，任务完成后将中间结果保存在本地磁盘上，然后向 coordinator 请求下一个任务。如果 coordinator 在一定时间内无法收到该 worker 的响应，那么 coordinator 就会把该 worker 标记为 failed ，此时所有由该 worker 已完成的和正在进行的任务都会被标记为未完成等待分配给其他的 worker，这是因为中间结果保存在本地磁盘上，我们认为 failed 的 worker 的本地磁盘也是无法被 reducer 访问的。在重新分配执行 map 任务的 worker 之后，所有执行 reduce 任务的 worker 都会接收到这个重新执行的通知，之后都会从新分配的 worker 中读取数据。

对一个运行 Reduce 任务的 worker 来说，它在处理完 reduce 任务后会将输出保存在 GFS 上，如果一个 worker 被标记为 failed ，只需要重新执行它正在执行的任务，已完成的 reduce 任务无需再执行。

#### Coordinator故障

一个解决方法是使用检查点机制，当 coordinator 故障时就恢复到最近的一个检查点然后启动 coordinator 进程，还有一种简单的方法是直接重启整个应用。

#### 出现故障时的语义(Semantics in the presence of failures)

当用户提供的 map 函数和 reduce 函数是确定性函数时，执行多次后的输出总是相同的。当一个 worker 完成 Map 任务后，它会将中间文件的位置发送给 coordinator ，并且 coordinator 会忽略后续的相同文件的其他位置信息；Reduce 任务完成后，worker 会以原子的方式将临时文件重命名为最终输出文件，当多个 worker 完成同一个 Reduce 任务，它们会对同一个最终输出文件进行多次重命名，从而保证在最终的文件系统中只有一次 Reduce 任务的执行结果（个人理解为以重命名的方式覆盖源文件？）。

### 尾部延迟

如果一台机器花费了异常长的时间来完成 Map 或 Reduce 任务，那么这台机器就称为落伍者（straggler），它导致整个 MapReduce 任务的完成时间变长。一个解决方法是 coordinator 会调度几个备用 worker 来执行剩下的正在执行中的任务，无论是主 worker 完成还是备用 worker 完成都会把任务标记为已完成。

### 改进

#### 分区函数(Partitioning Function)

在 Partition 阶段会将中间键值对映射到 $R$ 个分区中，默认使用的 hash 函数是 $hash(key)\ mod\ R$ ，但是在一些特殊情况，如 key 是 URL 且用户希望每个主机的条目都放在同一个输出文件中，那么可以把 hash 函数更改为 $hash(Hostname(key))\ mod\ R$ 。

#### 顺序保证(Ordering Guarantees)

在 Map 阶段和 Reduce 阶段我们都对键值对进行排序，这些做的好处是能减少代码的复杂性，并且当输出文件需要按 key 进行随机访问或用户需要输出文件有序时，排序会很有用。 

## 参考资料

1. [MapReduce: Simplified Data Processing on Large Clusters](http://nil.csail.mit.edu/6.5840/2023/papers/mapreduce.pdf) 
2. [reduce会等待所有mapper执行后才执行吗，还是会和mapper一起混合执行？](https://www.zhihu.com/question/264582834) 
