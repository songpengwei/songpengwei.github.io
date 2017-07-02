---
title: Hadoop 源码阅读之DFS（一）
updated: 2017-07-02 23:15
---

计划花一个月左右的时间，通读一遍Hadoop 0.0.1的源码，尽量少写一些废话，多记录一些思考。

Random一下，就从分布式文件系统（DFS）开始吧。
DFS即分布式文件系统，集合多台机器存储在预定义位置上的一组文件作为存储构件，在此基础上实现一些分布式操作，从而对外抽象出一套基本文件读写API。

#Block#
### blkid和len ###
**Block**是HDFS的文件存储的基本单位，有两个关键属性`blkid` 和`len`，前者用来标识一个操作系统上的文件，并且通过`"blk_" + String.valueOf(blkid)`拼接出文件名；后者是该文件以字节为单位的长度。
它抽象出了**存储**的两个基本维度，起始和大小。变量，数组，文件等等莫不如此。

###注册工厂方法###
另一个有意思的地方是所有实现Writable接口的类，都注册了一个工厂方法，具体有什么用，以后来补。
```Java
static {                                      // register a ctor
 WritableFactories.setFactory
   (Block.class,
    new WritableFactory() {
      public Writable newInstance() { return new Block(); }
    });
}
```

###序列化###
实现`Writable`利用Java的序列化接口（`DataOutput`），实现Block基本字段的序列化和反序列化。
每个待序列化类单独实现自己一对序列化和反序列化函数，是一个常用的基本设计，我在实习写桌面程序的时候，想将一些控件信息存储为xml，用的想法和这个是相同的，但是做的不好的事没有定义出这个Writable接口作为对这个行为的抽象。


