---
title: Hadoop 源码阅读之DFS（一）
updated: 2017-07-03 23:15
---

计划花一个月左右的时间，通读一遍Hadoop 0.0.1的源码，尽量少写一些废话，多记录一些思考。

Random一下，就从分布式文件系统（DFS）开始吧。
DFS即分布式文件系统，集合多台机器存储在预定义位置上的一组文件作为存储构件，在此基础上实现一些分布式操作，从而对外抽象出一套基本文件读写API。


# Block #
***
### blkid和len ###
**Block**是HDFS的文件存储的基本单位，有两个关键属性`blkid` 和`len`，前者用来标识一个操作系统上的文件，并且通过`"blk_" + String.valueOf(blkid)`拼接出文件名；后者是该文件以字节为单位的长度。
它抽象出了**存储**的两个基本维度，起始和大小。变量，数组，文件等等莫不如此。

### 注册工厂方法 ###
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

### 序列化 ###
实现`Writable`利用Java的序列化接口（`DataOutput`），实现Block基本字段的序列化和反序列化。
每个待序列化类单独实现自己一对序列化和反序列化函数，是一个常用的基本设计，我在实习写桌面程序的时候，想将一些控件信息存储为xml，用的想法和这个是相同的，但是做的不好的事没有定义出这个Writable接口作为对这个行为的抽象。

# BlockCommand #
***
一个命令（instruction）参数的封装，该命令作用于某个`DataNode`下的一系列Blocks；有两种操作，移动这组Blocks到另外一个`DataNode`，或者标记改组Blocks为失效状态。

### 实现 ###

```Java
boolean transferBlocks = false;
boolean invalidateBlocks = false;
Block blocks[];
DatanodeInfo targets[][];
```
用两个标志变量来指明是哪种操作；
用两个数组来存储操作对象。

然后通过构造函数重载，给出了三个构造函数，无参，移动命令或者失效命令。并且提供了各个字段的读权限。

**实现了`Writable`接口**。

### 总结 ###
对一个简单的命令基本信息的封装，用构造函数接受参数，确定操作类型和操作对象；用标志变量+数组对象来进行实现。
将一组数据按照某种语义捆绑在一起，在函数间传递时也方便，复用性也更好。

# LocatedBlock #
***
一个数据对，包含一个`Block`和其几个replicate所在的`DataNode`的信息。
```Java
Block b;
DatanodeInfo locs[];
```
相当于维持某个逻辑Block到其存储位置的指针，用于定位Block物理位置。

**实现了`Writable`接口**。


