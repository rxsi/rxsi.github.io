---
layout: post
title: Mysql InnoDB特性
date: 2021-12-23 15:12:21 +0800
categories: MySQL
tags: mysql innodb 
author: Rxsi
---

* content
{:toc}

## 插入缓冲（insert buffer，提升性能）
当索引是聚集索引时（主键索引），通常如果我们使用自增值作为主键，在插入时按照主键递增的顺序进行插入，那么是不需要**磁盘的随机读取的**，效率高。
```
CREATE TABLE t {
    a INT AUTO_INCREMENT,
    b VARCHAR(30),
    PRIMARY KEY(a),
    KEY(b)
};
```
因此我们应该优先使用自增值作为主键，如果使用非自增值依然会在插入时造成随机读取

对于非聚集索引，大部分索引值有随机性，因此会造成大量的随机读（页分裂和B+树节点自旋等）
因此 Innod b设计了`insert buffer`。
对于满足条件的非聚集索引的插入或者更新，不是每一次都直接插入到索引页中，而是先判断是否在缓冲池中，若在，则直接插入；若不在，则先放入到一个 insert buffer 对象中，再以一定的频率和情况进行 insert buffer 和辅助索引叶子节点的合并。
<!--more-->
### 非聚集索引可使用insert buffer 的条件

1. **索引是辅助索引**
2. **索引不是唯一的**（即定义该字段时不能是 unique，因为如果是唯一的，那么引擎还需要在插入时进行扫描判断是否与已有索引相同，也就没有使用 insert buffer 的意义了）

**现在升级为了change buffer，可以对insert、update、delete同时缓存，分为insert buffer、purge buffer、delete buffer**

### 内部实现
内部结构是一棵 **B+树**，且全局唯一，放在共享表空间，默认是 ibdata1 中。

#### 非叶子节点结构
对于应用于 insert buffer 的 B+ 树的非叶子节点结构实现如下：
![no_leaf.png](/images/mysql_innodb/no_leaf.png)

- space：对应着表空间id（每个表有唯一的space id），4字节
- marker：用以区分新老版本，1字节
- offset：表示当前索引页在原有表中的偏移量，4字节

一个索引是根据(sapce, offset)去确定的

#### 叶子节点结构
相较于非叶子节点，增加了 metadata 结构和插入数据
![leaf.png](/images/mysql_innodb/leaf.png)

metadata 字段的前两个字节存储的是插入该(space, offset)索引页的顺序，所以一条记录是根据(space, offset, counter)去确定的
因此每个数据都会额外增加 4 + 1 + 4 + 4 = **13字节**的额外数据

### ibuf bitmap
ibuf bitmap 存在于每一个 ibd 文件中（ibd文件存储的是表数据和索引数据），每隔**16384个页**（innodb中一个页是16KB，因此16384*16/1024= **256MB**），有一个 ibuf bitmap，且每个 page 占 4bits，用以标识当前辅助索引页是否被加载到了 insert buffer 中
![ibuf.png](/images/mysql_innodb/ibuf.png)

4bits的数据代表的意义如下：

![4bits.png](/images/mysql_innodb/4bits.png)

### merge insert buffer
将 insert buffer 的数据合并到磁盘的时机为：

- 辅助索引页被读取时
即当执行了 select 操作时，会根据`ibuf bitmap`判断该页是否在 insert buffer 有缓存，当有缓存时，会一次性 merge 回写到原有的索引页中
- ibuf bitmap 追踪到该辅助索引页已无可用空间时
当检测到剩余的页空间小于 1/32 页时（由 4bits 数据中的前两位决定），会强制触发一次 select 操作，即利用上面的规则引发一次 merge
- master thread
定时的 merge insert buffer

## 两次写（double write，提升可靠性）
MySQL采用 **WAL（Write Ahead Log）机制（先顺序写磁盘中的日志文件，再随机写磁盘中的数据页）** 实现了在脏页写入磁盘前如果断电的情况下，能够进行重执行或回滚（Redo Log 和 Undo Log）。
**doublewrite** 解决的则是当在写入磁盘的过程中，发生宕机情况依然能够恢复，不能只依赖于 WAL 的原因是此时的宕机会造成原有的磁盘空间破坏。（Redo log 记录的某页某偏移量的当前状态，因此如果是某页已被修改了部分磁盘结构，此时的 Redo log 并不能做到恢复）

### 内部实现
![double_write.png](/images/mysql_innodb/double_write.png)

由两部分组成：

- 内存中的 doublewrite buffer，大小为2MB
- 共享表空间中连续的128个页，大小同样为2MB

### 执行过程
当脏页需要写回磁盘时，会先通过 memcpy 函数复制到内存中的 doublewrite buffer，然后通过 doublewrite buffer 分两次，每次 1MB **顺序**写入共享表空间中的 doublewrite 磁盘空间，这个过程写的是连续的磁盘空间，即顺序写，因此开销不大。
当成功写入到共享表空间的 doublewrite 磁盘空间后，再把 doublewrite buffer 中的数据写入到各个表空间中（.ibd文件）。
如果此时发生宕机，那么会从共享表空间中的 doublewrite 恢复。

### 小结
我们一般说持久化是依赖于`Redo Log`，这是因为在将数据页加载到内存，并完成修改之后，第一步的操作是把修改后数据页对应偏移量的当前的状态记录下来，写入的日志文件就是`Redo Log`。这个过程是顺序写，因此性能较高，而如果在这个过程中，系统宕机了，则表现出来的就是事务没有执行成功，因此不会造成脏数据。`Redo Log`属于物理日志，即记录的内容是对数据页的物理操作：对 xxx 表空间中的 xxx 数据页的 xxx 偏移量的地方的值由 xxx 修改为了 yyy，也就是说`Redo Log`的重执行需要依赖于原始的数据页！！！
当在写入数据页时，如果在写之前宕机了，此时数据页对应的磁盘空间还未被修改，因此可以根据`Redo Log`进行重执行。而如果是在执行的过程中发生了宕机，此时数据页已经遭到破坏，因此`Redo Log`是无法达到数据恢复的作用的。
所以 InnoDB 需要有`double-write`机制。
## 自适应哈希索引
通常一个 B+ 树会设计为3、4层的结构，每一层代表了一次IO，因此至少需要3、4次 IO 才能读取到值。
 innodb 会根据查询频率，自动对热点数据建立 hash 索引，条件如下：

- 该模式查询了100次
- 页通过该模式访问了N次，N = 页中记录 * 1/16

以(a, b)联合索引为例:
where a = xxx;
where a = xxx and b = xxx;
属于两种模式，因此如果是交替的使用，不会建立hash索引
另外，**hash索引只适用于等值查询，对于范围查询不会建立hash索引**

## 异步IO（Asynchronous IO，提升性能）
AIO的优势：

- 用户无需等待前一个 IO 回复之后再请求下一个 IO
- AIO 可以合并多个关联 IO 为一个 IO 请求，提升效率

## 刷新邻接页（Flush Neighot Page，提升性能）
当某个脏页刷回磁盘时，会同时检查**该页所在区的所有页**，如果亦是脏页，则一同刷回磁盘，这种做法的好处是利用了 AIO 的合并 IO 的特性，提升了性能