---
title: Mysql索引介绍
date: 2021-11-04 21:22:22
tags: mysql索引
categories: 数据库
---

## 

## 索引的本质

> 索引是帮助Mysql高效获取数据的排好序的数据结构

## 索引的数据结构

#### 二叉树：单边增长的场景会导致全表扫描。

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwoa8e7thzj30am0dkwek.jpg" style="zoom: 67%;" />

<!-- more -->

#### 红黑树：相对平衡，比二叉树性能好，大数据下，红黑树的高度过高，会造成磁盘IO频繁。

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwoabk973sj30dy08cjrf.jpg"  />

#### Hash

一次hash算法就可以定位到文件的位置，但存在缺点：

* 不支持范围查找
* 不支持排序

数据库如果比较大，只用到精确查找就可以用hash

#### B-tree： 

1. **特点：**

> * 叶子结点具有相同的深度，叶子结点指针为空。
> * 所有索引元素不重复
> * 节点中的数据索引从左到右递增

2. 如果使用了innoDb存储引擎，结点的data元素就可能存储的是除了索引外的其他所有列，会占用比较大的存储空间，对于一个大结点而言，可以存放的结点数量就会比较小

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gwoaf7s92lj30ne07oq3a.jpg"  />

 #### B+tree(B-tree变种)

1. 特点：

   > - 非叶子结点不存储data ,只存储索引（冗余），可以存放更多的索引。
   > - 叶子结点包含所有的索引字段。
   > - 叶子结点用指针连接，提高区间访问的性能。

   ![](https://tva1.sinaimg.cn/large/008i3skNly1gwoarcvziwj30r80bwjs3.jpg)





## 索引是怎么支持千万级表快速查找？

mysql建议一个结点大小为16kb,这样一次iO速度比较快，一个大结点下的一个索引元素大约是14b，所以一个大结点里面约有1170个索引元素，对于一个高度为3的b+树，可以存储`16* 1170* 1170 `= 2000万。

## 存储引擎

存储引擎是对于表而言的，不同的表可以设置不同的存储引擎

### MyISAM索引实现(非聚集)

* MyISAM索引文件和数据文件是分离的

* 表结构文件是xxx.frm, 索引文件是xxx.MYI, 数据文件是xxx.MYD

  ![](https://tva1.sinaimg.cn/large/008i3skNly1gwqkxbz5qvj30j80f8mxz.jpg)

> 执行流程：当有一条查询语句：where Col1 = 49， 先判断有没有走索引，走索引的话，先根据49快速在MYI文件中定位到结点，获取该结点存储的索引所在行的磁盘文件指针0x90，再去MYD中定位数据。

### InnoDb索引实现(聚集)

* 表数据文件本身就是按B+tree组织的一个索引文件

* 叶子结点包含了完整的数据记录，data里存储的是：索引所在行的其他所有数据

* 表结构文件是xxx.frm, 索引文件+ 数据文件 是xxx.ibd

  

![](https://tva1.sinaimg.cn/large/008i3skNly1gwql7jmw80j30si0dmgmn.jpg)

#### 为什么InnoDb表要尽量设定一个主键

* innnoDb表数据文件本身就是按B+tree组织的一个索引文件
* 主键是数据库确保数据行在整张表唯一性的保障.
* 设定了主键之后，在后续的删改查的时候可能更加快速以及确保操作数据范围安全。 
* 如果没有主键，InnoDB会选择一个唯一键来作为聚簇索 引，如果没有唯一键，会生成一个隐式的主键。 

#### 主键使用整型的自增ID而不是UUID？

* 查找元素的时候会涉及到大量的数据比较，整型比字符串快
* UUID占用的存储空间会大于整形
* 叶子结点是按顺序排列的，如果主键索 引是自增ID，那么只需要不断向后排列即可，如果是UUID，由于到来的ID与原来的大小不确定，在维护B+树的过程中，会造成非常多的数据插入，数据移动，然后导致产生很多的内 存碎片，进而造成插入性能的下降。 

### 辅助索引（非主键索引）

* 叶子结点存储的是索引所在数据行的主键。

* 辅助索引访问数据总是需要二次查找，先遍历非主键索引再遍历主键索引

#### 为什么非主键索引结构叶子结点存储的是主键值?

* 一致性，完整的数据只在主键索引上维护一份就可以，不用考虑分布式的问题。
* 节省存储空间.



## 联合索引  



* 5个单值索引，对应5棵B+树，联合索引就只需要一棵树，所以日常推荐使用联合索引而不是单值索引。
* 索引排序的时候会按照字段顺序，逐个去排序

![](https://tva1.sinaimg.cn/large/008i3skNly1gwqm2wi1m8j30og0c8dgy.jpg)
