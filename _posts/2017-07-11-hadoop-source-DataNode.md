---
title: Hadoop 源码阅读之DFS（二）DataNode
updated: 2017-07-11 20:06
---

上一篇把一些零碎的小类集在一起，凑成一篇。这篇打算对比较长的一个类`DataNode`读读。
每个DataNode代表一个数据节点，对应某台机器的一个文件夹，本质上是一定数量的Block的集合，能够和NameNode，client以及其他DataNode进行通信，以对该Block集合进行操作，主要包括client的读和写，其他DataNode block的复制，以及响应NameNode操作，进行删除等操作。
具体实现来说，数据结构上，维持了一个block到byte array的表；执行时，DataNode内部是一个无限循环，不断询问NameNode，报告状态（心跳），执行命令**（RPC）**。

1. 状态信息。[`DataNodeInfo`]({{ site.posturl }}/hadoop-source-DFS#datanode-info)：总大小，剩余大小，上次更新时间。
2. 执行命令。
	1. 客户端读写Blocks
	2. 让其他DataNode复制Blocks
	3. 删除某些Blocks

此外，DataNode还维持着一个Server Socket以处理来自Client或者其他DataNode请求。其对外暴露的*host:port*对提交给NameNode以进一步下发给相关的其他DataNode或者client。
(摘自类注释)

## StartUp
DataNode启动的时候主要干了以下事情：
为每个`dfs.data.dir`实例化一个`DataNode`，DataNode有以下几个重要字段：
	a. `namenode`，`DatanodeProtocol`类型，和NameNode进行RPC通信。
	b. `data`， `FSDataset`类型，对应一个文件夹，负责具体的本地磁盘操作。
	c. `localName`， machine name + port，对外暴露的网络机器名和端口。
	d. `dataXceiveServer`，一个socket Server，监听上述端口，处理读写请求。
	e. 其他一些配置字段，包括`blockReportInterval`， `datanodeStartupPeriod`等。

## Main Loop When Run
`offerService`，该函数根据当前时间与上次动作时间差值，决定是否再一次执行该动作（`DataNode`对`NameNode`的RPC）；这些事件有**向NameNode**：

* 发送心跳信息
* 上传`block`信息
* 获取`NameNode`指令

下面分别就每一项进行详细说明：
#### 1. 发送心跳信息
心跳信息包括以下几项内容：

*	`DataNode`名字
* `DataNode`数据传输端口
* `DataNode`总容量
* `DataNode`剩余字节数

#### 2. 上传当前`Block`信息
报告本`DataNode`的所有`Block`信息，以更新表machine->block list 和表block->machine list。利用TreeMap实现，能得到按BlockId排序的数组，通过逐一比较新旧上报Block数组的每个元素（`oldReport`和`newReport`），利用`removeStoredBlock`和`addStoredBlock`将旧数组更新为新数组。
#### 3. 报告`ReceivedBlock`
当Client写数据，或者其他DataNode复制数据给当前`DataNode`的时候，该`DataNode`通过RPC，执行此函数。然后NameNode

#### 4. 获取 `NameNode`指令

根据`BlockCommand`类的字段：

```Java
boolean transferBlocks = false;
boolean invalidateBlocks = false;
Block blocks[];
DatanodeInfo targets[][];
```
可以看出，指令动作包括交换数据（transfer）和删除数据（delete or invalidate），动作对象包括一系列blocks和Datanode，表示将`blocks[i]`传送到`targets[i][0] `... `targets[i][j]`的Datanode上去。  
具体传送实现，为每一个!invalid的block，启动一个线程，负责具体数据传送，代码为：
```Java
new Daemon(new DataTransfer(xferTargets[i], blocks[i])).start();
```
后面将对`DataTransfer`类进行详细注解。

## `DataTransfer`
该类实现了`Runnable`接口，在每次有数据需要传输时被启动；其动作主要为：

1. 链接第一个Target DataNode的socket，作为输出。
2. 从`FSDataSet`中获取Block元信息以及本机器上该block对应的数据文件，作为输入。
3. 从输入端读取数据，写到输出端。

因此，该类之负责将block信息写到第一个target DataNode，比如说Node1，剩下的将由Node1机器上的线程进行数据传送。

该block在本机实际的文件夹路径和文件名都可以根据blockId进行确定。对于一个64bit的blockId，从高位到地位，每四位作为一个文件夹的名字（0~15），进行路由，因此文件实际位置的深度可能高达64/4=16层；存储数据的文件命名方式为blk_{blockId}.

## `DataXceiveServer`
该类也实现了`Runnable`接口，在`DataNode`初始化的时候被启动，用于监听Client或者其他DataNodes的请求，以进行block数据的传输。
具体实现为，使用`SocketServer`，根据信号`shouldListen`来循环监听所有请求。当请求到来时，使用`DataXceiver`类进行具体连接的处理；

## `DataXceiver`
该类负责具体时间数据传输的逻辑，包括写和读；
首先，打开socket输入流，读取首字节，判断操作类型；
然后，进行写或者读操作。

#### 写操作（OP_WRITE_BLOCK）
1. 读入header，包括以下几个字段

	```Java
	a. shouldReportBlock --> bool
	b. block info(blkid+len) --> Block
	c. numTargets --> int
	d. targets -->DatanodeInfo[]
	e. encodingType --> byte
	f. data length --> long
	```
2. 然后将这些header信息，去掉该DataNode`targets[0]`的信息后，写入下一个DataNode `target[1]`。
3. 从socker中读取具体存储的数据，同时写入本地存储（当前DataNode）以及下一个DataNode的socket。这里有一点设计，就是如果写Socket异常后，可以终止Socket，但仍然继续写本地存储。
4. encodingType的类型不同，读取数据方式不同：对于`RUNLENGTH_ENCODING`类型，其结构是length(say l)+data(of the length l)，因此读一次就结束；而`CHUNKED_ENCODING`类型，结构为l1 + data1 + l2 + data2 + ... + ln + datan + **l(n+1) (=0)**；因此需要循环继续读如长度，然后读入该长度数据，直到len=0结束。
5. 如果和下一个DataNode间的Socket仍然正常，则从该Socket读回一些关于写数据的反馈，包括long型的结束符和`LocatedBlock`-->写成功后的block网络位置，是一个`Block`和`DatanodeInfo[]`对，表示已经写入该block成功的DataNode list。整个写操作和备份的过程类似于一个递归调用的过程，由client写datanode1， 然后datanode1写datanode2，然后datanode2写datanode3；然后datanode3将写成功信息，以及datanode3位置告诉datanode2，然后datanode2将写成功信息以及datanode2，datanode3位置告诉datanode1。
#### 读操作（`OP_READ_BLOCK` || `OP_READSKIP_BLOCK`）
首先读入待读取的Block信息，然后，如果是`OP_READSKIP_BLOCK`类型，则读取需要跳过的字节数（toSkip-->long）；
然后通过data --> FSDataSet 定位block本地存储文件位置，根据类型决定是否跳过特定字节（toSkip），然后逐字节读取该文件。

## Aside info
如果类需要作为Key，比如`TreeMap`，则需要实现`Comparable`接口，只有可以比较才能进行排序，Hash；如果需要进行序列化和反序列化，则需要实现`Writable`接口。
					



	





