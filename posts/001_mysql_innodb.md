# 当我们讨论innodb时，我们在讨论什么

我们讨论这样一个结构:

![](../pics/innodb-architecture.png)

## 内存结构

* 内存结构
  * Buffer Pool 一个磁盘页缓存
    * **用来干嘛的** innodb扫描磁盘上的表文件或索引文件时，会把磁盘数据加到到内存，加快后续的读取过程
    * **结构如何** Page List
      * 一个指向Page的列表；每个page固定大小，缓存磁盘数据
      * 列表分为前后两个部分。前2/3分是**New Pages**，代表最近访问的一些page；后1/3是**Old Pages**,最近没有访问的page
  
  ![innodb buffer pool list](../pics/innodb-buffer-pool-list.png)
    * **算法策略** 
      * 变种的LRU
        * NewPages代表Recently Used Pages，是相对hot的缓存
        * OldPages代表Not Recently Used Pages，已经不那么hot，即将被淘汰
        * 如果一个query扫描了不在Buffer Pool里的page，innodb会把这些page加载到Buffer Pool 2/3的位置，并驱逐一些Old Pages
        * 如果访问了Old Pages，innodb会把这些Page移到New Pages
        * 这种变种的LRU，保持了相对好的弹性。如果新访问的Page，直接从磁盘移到New Pages最前面的位置，会造成一些空间浪费。特别是这些Page有可能只被访问一次就再也不访问了
      * 多个pool实例
        * 为什么允许多个pool实例呢？避免线程竞争。万一几个read线程，同时触发了磁盘页加载到Buffer Pool的过程，就会有竞争和等待。多个实例，允许innodb通过hash的方式去选择pool实例，提高了并发性能
      * 全表扫描、read ahead时，page插入哪个位置？
    * **如何观察** `show engine innodb status;` 查看`BUFFER POOL AND MEMORY`

  * Change Buffer 另一个磁盘页缓存
    * **用来干嘛** innodb修改某些数据时，并不会直接写磁盘，而是直接写内存缓存。一定时机下，才会写回磁盘页。当然，为了保证Persistance，innodb会先写redo log和undo log，避免宕机造成的内存数据丢失
    * **适用对象** 只适用于secondary索引 
    * **为什么要设计这个Change Buffer** 一般用到secondary索引时，我们更新的数据记录肯定是分布在B Tree某些节点位置上，这些位置很可能跨越了多个磁盘分页。如果我们直接写磁盘，就导致机械磁盘的非线性写入，性能下降。把写操作暂存到Change Buffer，合并一些随机写操作，变成批量写入，可以适当减少磁盘寻址的开销。
    * **什么时候发生merge** Change Buffer可以理解为脏页，
    * **遗留问题** 
      * Change Buffer在磁盘上是什么形式
      * 为什么可以survive a crash？
   
  ![innodb change buffer](../pics/innodb-change-buffer.png)
  * Adaptive Hash Index
  * Log Buffer

## 磁盘结构

* 磁盘结构
  * Tables
  * Indexes
  * Tablespaces
  * Doublewrite Buffer
  * Redo Log
  * Undo Logs

## 锁和事务

## 备份与恢复

## 复制