---
title: Hadoop 源码阅读之DFS（二）DataNode
updated: 2017-07-11 20:06
---

上一篇把一些零碎的小类集在一起，凑成一篇。这篇打算对比较长的一个类`DataNode`读读。
每个`DataNode`本质上是一定数量的`Block`的集合，能够和`NameNode`，Client以及其他`DataNode`进行通信，以对该Block集合进行操作，无非增删改查几个元操作。
具体实现来说，`DataNode`内部是一个无限循环，不断询问`NameNode`，报告状态（心跳），执行命令（RPC）。

1. 状态信息。[`DataNodeInfo`]({{ site.posturl }}/hadoop-source-DFS#datanode-info)：总大小，剩余大小，上次更新时间。
2. 执行命令。
	1. 客户端读写Blocks
	2. 让其他DataNode复制Blocks
	3. 删除某些Blocks
	





