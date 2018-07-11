---
title: Hadoop源码阅读之MapReduce（一）：基本概念和接口
updated: 2018-02-24 22:33
tag: Hadoop, Map Reduce
---

## 概述
梳理一下MapReduce框架涉及到的一些基本接口和类。

## 文件读写相关
`RecordReader`：从输入文件中读入键值对，这里是指的map的输入，还是reduce的输入？接口有三个函数`next(Writable key, Writable value)`，`getPos()`和`close()`，由此看来，该接口类似于一个抽象的迭代器。`InputFormat`实现了该接口。

`RecordWriter`：将键值对写到输出文件，`OutputFormat`实现了该接口。包含函数：`write(WritableComparable key, Writable value)`和`close(Reporter reporter)`。

`OutputCollector`：作为参数传送给`Mapper`和`Reducer`来输出结果数据。该接口只有一个函数`collect(key, val)`。













     









