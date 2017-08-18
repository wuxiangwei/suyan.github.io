---
layout: post
title: "Ceph BlueStore 解析：Object IO到磁盘的映射"
category: [存储]
tags: [BlueStore]
keywords: BlueStore
---

简单回顾下Ceph OSD后端存储引擎的历史。

为解决事务原子性问题，最早的FileStore存储引擎引入了Journal机制，一个IO先同步写日志，再异步写文件，这在解决原子性问题的同时也引入了写放大的问题，一次IO两次落盘。NewStore通过Onode索引解耦Object和文件，让IO直接写文件，写成功后更新Onode元数据，从而避免两次落盘问题（特殊情况下还是需要先写日志）。NewStore虽然解决了写放大问题，但不够彻底，无法消除本地文件系统带来的消耗。于是，BlockStore登场，在NewStore基础上抛开本地文件系统直接管理裸块设备。直接管理磁盘首先要做两件事情：

(1) 磁盘空间分配；
(2) 将Object数据映射到磁盘；

本文重点分析BlueStore对这两个问题的解决。


### 概念


![](http://ohn764ue3.bkt.clouddn.com/ceph/bluestore/allocator/onode2disk.png-name)

**Onode**    代表一个Object，类似于VFS中的Inode索引节点；
**lextent**  Object的逻辑范围；
**blob**     为Onode分配的一组磁盘空间，可以被多个lextent共享；
**pextent**  一段连续的物理磁盘空间，默认最小单位为4K字节。一个blob的磁盘空间由多段不连续的物理磁盘空间组成。


### 磁盘空间分配


![](http://ohn764ue3.bkt.clouddn.com/ceph/bluestore/allocator/bitallocator.png-name)

概念。
Block 大小为4K字节的一段磁盘空间；
BmapEntry 为一整型，整型的每个bit位代表一个Block；
Zone 一组空间上连续的Block的集合，Block数目默认为1024；
Span 一组空间上连续的Zone的集合，Zone数目默认为1024；


磁盘分配器Allocator将裸块设备划分成固定大小的Block，分配、释放磁盘空间时以Block作为最小单元。目前BlueStore提供了两种磁盘分配策略，一种称为Stupid，另一种称为Bitmap，Bitmap为默认配置，本节重点分析Bitmap磁盘管理。Bitmap（位图）管理磁盘空间的基本思路是，使用一个bit位的两种状态0或者1来表示一个Block空闲或已分配两种状态。Bitmap是本地FS管理磁盘的一种常用方式，对于一个已经格式化过的块设备，文件系统会在Superblock中持久化元数据，记录哪些Block已被使用哪些Block空闲。BlueStore同样需要记录这些信息，不过将它记录在KV数据库，OSD进程启动时从KV中加载Block块状态，构建出如上图所示的树形结构。

树形结构的每个节点会统计以此为根的子树中包含的空闲磁盘空间和已分配磁盘空间的字节数，这在分配连续大块的磁盘空间时可以跳过空间不足的子树，快速定位到剩余空间能够满足要求的子树，从而提高分配效率。不过，树形结构也会带来一些问题。两棵相邻子树会将原本连续的磁盘空间分隔开，可能存在这样的情况，左边子树中最右边的Block是空闲的，右边子树的最左边Block是空闲的，那么将两棵子树合并在一起也能够分配出一段连续空间。对于这个问题，BlueStore只考虑Zone内不同BmapEntry间的情况，对粒度更大的子树就不做考虑了。


### Object数据对齐

向bluestore_min_alloc_size对齐。
bluestore_min_alloc_size是Onode申请磁盘空间的最小单元，固态硬盘默认为64K字节，HDD默认4K字节。Onode每次申请磁盘空间的大小都是bluestore_min_alloc_size的整数倍，申请到的磁盘空间由blob结构来管理。

如何确定是否要为一个写IO分配磁盘空间，分配多少磁盘空间？

![](http://ohn764ue3.bkt.clouddn.com/ceph/bluestore/allocator/write_io.png-name)

Onode对磁盘空间的分配是“精简配置”模式，对一个只在1024K到1088K位置有数据的Object，只要为其分配64K字节而不是1088K字节的磁盘空间。上图给出了写IO修改范围和Object旧数据间的重叠关系，假设IO修改范围和旧范围都足够大，那么每个旧范围可能都有自己独立的磁盘空间。通过把IO修改范围对齐到bluestore_min_alloc_size，将IO修改范围分割成3部分：头部、中间部分和尾部。对中间部分，直接丢弃重叠的旧范围所分配的磁盘空间，为中间部分重新分配物理上连续的磁盘空间；对头部和尾部，考虑将IO修改范围应用到已分配的磁盘空间。

为什么单独为中间部分重新分配磁盘空间，而不为头、尾部分也重新分配磁盘空间？
跟中间部分重叠的若干旧范围，这些旧范围分配的磁盘空间在物理上可能并不连续，在不连续的物理磁盘中读取逻辑上连续的数据, 性能必然会低。第二个原因是这些旧范围中的旧数据，即将被新数据覆盖，已经成为无效数据，弃之并不可惜。头部和尾部可能需要新分配磁盘空间，也可能直接使用已分配的磁盘空间。在使用已有磁盘空间的情况下，已有磁盘空间中包含旧数据，这些旧数据可能与新数据有重叠部分(如上图的尾部)，可能和新数据没有重叠部分(如上图的头部)。已有磁盘空间中没有和新数据重叠的数据仍然是有效数据，如果重新分配磁盘空间则需要将这部分有效数据迁移到新磁盘空间，得不偿失。


向bdev_block_size对齐。
bdev_block_size是磁盘能分配的最小单元，默认4K字节。传统机械硬盘的最小读写单元是扇区，扇区大小为512字节，如果写入的数据小于512字节，则需要先读取扇区内容合并后再写入。SSD最小读写单元是页，页大小为4K字节。读取，合并，再写入，将严重降低写性能，不可不考虑。解决方法很直接，对没对齐的修改数据，**补零**，将其对齐到bdev_block_size大小。

![](http://ohn764ue3.bkt.clouddn.com/ceph/bluestore/allocator/pad_zeros.png-name)
中间部分已经按照bdev_block_size对齐，原因是，中间部分按照bluestore_min_alloc_size对齐，而bluestore_min_alloc_size已经按照bdev_block_size对齐，故而中间部分默认已经bdev_block_size对齐。对首、尾部分又可以再分为两种情况：一种情况是对齐后，补零部分和Object的旧数据没有重叠，如上图(b)和(c)所示；另一种情况是对齐后，补零部分和Object的旧数据重叠，如上图(a)所示，这种情况不能直接补零，否则将覆盖旧数据。针对覆盖旧数据的问题，先从磁盘读取数据本应该补零的数据，用这部分数据填充新数据以达到对齐的目的。
