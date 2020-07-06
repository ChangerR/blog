---
title: MySQL Innodb 内存池实现简介
tags:
  - mysql
categories:
  - mysql
  - innodb
date: 2020-06-27 16:59:19
---
#### MySQL的内存池实例
下图是MySQL innodb中内存的大致组织形式：
<style type="text/css">
.post-svg-container{
  display: block;
  overflow-x: auto;
  overflow-y: hidden;
}
.post-svg-container > object{
      justify-content: center;
      height:100%;
}      
</style>
<div class="post-svg-container">
    <object type="image/svg+xml" data="/images/MySQL_Buffer_Pool.svg"></object>
</div>

##### buf_pool_t(内存池):
每个buf_pool_t结构体下面都会有自己的锁，物理内存块（上图所示的Chunks)以及各种逻辑链表，各个buf_pool_t之间没有竞争关系，函数buf_pool_get通过传入的page_id取模分配对应的buffer pool instance。每个buf_pool_t结构都会维护一个LRU 的page hash链表，通过space_id和page_no的hash可以快速定位到内存页，不需要遍历整个LRU List。

##### buf_page_t(页)：
InnoDB中，数据管理的最小单位为页，默认是16KB，页中除了存储用户数据，还可以存储控制信息的数据。InnoDB IO子系统的读写最小单位也是页。在InnoDB中还支持压缩页，当前先不做介绍。

##### Free List(空闲页链表):
在当前节点中存储的是未被使用的空闲页，当InnoDB想要从磁盘载入新的数据页，就会从当前的链表中取出空闲页用于载入数据。如果Free List上没有足够的脏页提供，则会从 Flush List 或者是 LRU list 淘汰一些节点用于补充Free List。在buf_pool_t初始化时会把所有的空白页都加入到Free List中。

##### Flush List(脏页链表)
在此链表上的所有节点都是脏页，当LRU List中的页面发生改动时会把对应的数据页加入到当前的链表中。一个数据页可能会在不同的时刻被修改多次，在数据页上记录了最老(也就是第一次)的一次修改的lsn，即oldest_modification。不同数据页有不同的oldest_modification，FLU List中的节点按照oldest_modification排序，链表尾是最小的，也就是最早被修改的数据页，当需要从FLU List中淘汰页面时候，从链表尾部开始淘汰。加入FLU List，需要使用flush_list_mutex保护，所以能保证Flush List中节点的顺序。

##### LRU List
这个是InnoDB中最重要的链表。所有新读取进来的数据页都被放在上面。链表按照最近最少使用算法排序，最近最少使用的节点被放在链表末尾，如果Free List里面没有节点了，就会从中淘汰末尾的节点。LRU List被分为两部分，默认前63%为young list，存储经常被使用的热点page，后37%为old list。新读入的page默认被加在old list头，只有满足一定条件后（在Old链表中存在了一定的时间，可以通过配置innodb_old_blocks_time配置），才被移到young list上，主要是为了预读的数据页和全表扫描污染buffer pool。

- 引用 http://mysql.taobao.org/monthly/2017/05/01/