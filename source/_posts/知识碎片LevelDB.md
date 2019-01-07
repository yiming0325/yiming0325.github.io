---
title: 知识碎片：LevelDB
date: 2018-12-21 10:54:16
tags: [LevelDB,知识碎片]
categories: 知识碎片
---

# 0x00 LevelDB介绍

LevelDB是Google传奇工程师Jeff Dean和Sanjay Ghemawat开源的持久化KV存储引擎，具有很高的随机写，	顺序读/写性能，但是随机读的性能很一般，也就是说，**LevelDB很适合应用在查询较少，而写很多的场景 。**   LevelDB应用LSM（Log Structured Merge）策略，lsm_tree对索引变更进行延迟及批量处理，并通过一种类似于归并排序的方式高效地将更新迁移到磁盘，降低索引插入开销。

# 0x01 LevelDB设计思路

LevelDB的数据是存储在磁盘上的，采用LSM-Tree的结构实现。LSM-Tree**将磁盘的随机写转化为顺序写**，从而大大提高了写速度。为了做到这一点LSM-Tree的思路是将索引树结构拆成一大一小两棵树，较小的一个常驻**内存**，较大的一个持久化到**磁盘**，他们共同维护一个有序的key空间，写入操作首先操作内存中的树，随着内存中树的不断变大，会出发与磁盘中树的归并操作，而归并操作本身仅有顺序写。如下图所示：  

![LSM-Tree](http://pk81c6tjy.bkt.clouddn.com/LSM-Tree.png)

随着数据的不断写入，磁盘中的树会不断膨胀，为了避免每次参与归并操作的数据量过大，以及优化读操作的考虑，LevelDB将磁盘中的数据又拆分成多层，每一层的数据达到一定容量后会触发向下一层的归并操作，每一层的数据量比起上一层成倍增长。这也就是LevelDB的名称来源。

# 0x02 LevelDB特点

LevelDB是一个持久化的KV系统，和Redis这种内存型的KV系统不同，LevelDB不会像Redis一样大量消耗内存，而是将大部分数据存储到磁盘上。  

LevelDB支持数据快照(snapshot)功能，使得读取操作不受写操作影响，可以在读操作过程中始终看到一致的数据。  

LevelDB支持数据压缩等操作，这对于减小存储空间以及增快IO效率有直接的帮助。  

LevelDB的写操作要大大快于读操作，顺序读写操作要大大快于随机读写操作。  

LevelDB是一种非关系型数据库(NoSQL)，不支持sql语句，也不支持索引。  

LevelDB没有内置的C/S架构，且一次只允许一个进程访问一个特定的数据库。  

# 0x03 LevelDB整体结构

* **Memtable** : 内存数据结构，调表实现，新的数据会首先写入这里

* **Log文件** ： 写Memtable前会先写Log文件，Log文件通过append的方式顺序写入。Log的存在使得机器宕机导致的存存数据丢失得以恢复。

- **Immutable Memtable** : 达到Memtable设置的容量上线后，Memtable会变为Immutable为之后向SSTW文件的归并做准备。顾名思义，Immutable Memtable不再接受用户写入，同时会有新的Memtable生成。
- **SST文件** ： 磁盘数据存储文件。分为Level 0 到Level N 多层，每一层包含多个SST文件；单层SST文件总量随层次增加成倍增长；文件内数据有序；其中Level 0 的SST文件有Immutable直接Dump产生，其他Level的SST文件由其上一层的文件和本层文件归并产生；SST文件在归并过程中顺序写生成，生成后仅可能在之后的归并中被删除，而不会有任何的修改操作。  
- **Manifest文件** ： Manifest文件中记录SST文件在不同Level的分布，单个SST文件的最大最小key，以及其他一些LevelDB需要的元信息。
- **Current文件** ： 从上面的介绍可以看出，LevelDB启动时的首要任务就是找到当前的Manifest，而Manifest可能有多个。Current文件简单记录了当前Manifest的文件名，从而让这个过程变得非常简单。   

![整体结构](http://pk81c6tjy.bkt.clouddn.com/%E6%95%B4%E4%BD%93%E7%BB%93%E6%9E%84.png)

# 参考链接

https://baike.baidu.com/item/LevelDB/6416354?fr=aladdin

https://www.zhihu.com/question/19887265

https://github.com/machicao2013/gitbook/blob/master/leveldb/implementation/advantages.md

https://www.cnblogs.com/chenny7/p/4026447.html

http://catkang.github.io/2017/01/07/leveldb-summary.html

https://chuansongme.com/n/949438