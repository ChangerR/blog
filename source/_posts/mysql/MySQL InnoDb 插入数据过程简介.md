---
title: MySQL Innodb 插入数据过程简介
tags:
  - mysql
categories:
  - mysql 
  - innodb
date: 2020-07-05 20:59:19
---

数据库存储引擎主要解决的问题就是内存与硬盘之间速度不匹配的问题，下面让我们来看下Innodb是如何与文件系统交互的吧。

如下图：
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
    <object type="image/svg+xml" data="/images/MySQL_Innodb_buffer_and_filesystem.svg"></object>
</div>

##### 上图的几个核心组件介绍：

1. **缓冲池（buffer pool）**
Innodb主要操作对象是数据页，而缓冲池就是用来缓存数据页的地方，关于缓冲池的具体操作，可以看这篇文章[MySQL Innodb 内存池实现简介](http://b.changer.site/2020/06/27/mysql/MySQL%20InnoDb%E5%86%85%E5%AD%98%E6%B1%A0%E5%AE%9E%E7%8E%B0%E7%AE%80%E4%BB%8B)

2. **日志缓冲（log buffer）**
磁盘的随机写速度是远低于磁盘的顺序写的速度的，如果innodb每更新一个页都要将它与磁盘同步，会导致数据库的插入更新会非常的慢，甚至是不可用的状态，所以一般的数据库都会引入redo log的概念。innodb的redo log会记录事务对物理页的所有物理操作，一般来说redo log落盘了就认为当前的事务已经正常提交了。log buffer是redo log在内存中体现，innodb在更改页面后会把所有的操作都写入到log buffer中，后台的日志线程会定时的把log buffer中的数据刷入到磁盘中，当事务最终提交时，可能只需要刷入很少的数据到磁盘中，可以提高事务提交的速度。
innodb中的 innodb_flush_log_at_trx_commit 配置项可以控制InnoDb刷新事务日志的策略，其默认参数为1
刷新的策略值及其含义：
- 0，没一秒钟刷新一次log buffer,再事务提交时不再等待日志刷盘，直接结束，这个选项可能会导致数据丢失。
- 1，每次事务都会把log buffer写入并刷新（调用fsync）到磁盘中，默认操作。
- 2, 每次事务会把log buffer 写入到磁盘中，但是不会调用fsync函数，这样只是mysql服务经常挂了并不会导致数据丢失，但是如果是操作系统断电会导致数据永久丢失。

3. **double write**
当发生数据库宕机时，可能InnoDB正常将某一个页刷入到磁盘中，而InnoDB的页大小正常为16KB，但是磁盘的一个块的大小只有512B,这样就有可能导致物理页只写入了前4KB,数据库就宕机了，造成了数据库的部分写失效。
为了解决部分写失效的问题，innodb引入了doublewrite功能。doublewrite有两个部分组成，一部分时内存中的doublewrite buffer,大小2MB，另外是共享表空间中连续的128个页，大小同样是2MB。在对缓冲池中的脏页进行刷新时，并不是直接将其刷入到磁盘中，而是会把要写入磁盘的脏页都先拷贝到double write缓冲中，之后分两次每次1MB分别刷入到物理磁盘中，最后调用fsync同步到磁盘。在完成double write 页的写入后，再将double write buffer中的页写入到各表文件中。

下图是innodb double write 的结构：
<div class="post-svg-container">
    <object type="image/svg+xml" data="/images/MySQL_InnoDB_doublewrite.svg"></object>
</div>

我们已经大致了解了innodb的整体架构，那么innodb插入一条数据会大致做什么操作那？

进入innodb引擎层，handler对应ha_innobase 插入的表信息保存在handler中：
``` c++
int
ha_innobase::write_row(
        uchar*  record) /*!< in: a row in MySQL format */
{
    error = row_insert_for_mysql((byte*) record, prebuilt);
}
```

``` c++
UNIV_INTERN
dberr_t
row_insert_for_mysql(
        byte*           mysql_rec,      /*!< in: row in the MySQL format */
        row_prebuilt_t* prebuilt)       /*!< in: prebuilt struct in MySQL
                                        handle */
{
    /*记录格式从MySQL转换成InnoDB*/
    row_mysql_convert_row_to_innobase(node->row, prebuilt, mysql_rec);
	
    thr->run_node = node;
    thr->prev_node = node;
		
    /*插入记录*/
    row_ins_step(thr);
}
```

``` c++
UNIV_INTERN
que_thr_t*
row_ins_step(
  que_thr_t*      thr)    /*!< in: query thread */
{
  /*给表加IX锁*/
  err = lock_table(0, node->table, LOCK_IX, thr);

  /*插入记录*/
  err = row_ins(node, thr);
}
```
因为Innodb的存储形式是索引组织表，所以如果mysql没有创建主键或者是唯一键，mysql会隐式创建一个rowid用来组织索引。

``` c++
static __attribute__((nonnull, warn_unused_result))
dberr_t
row_ins(
/*====*/
        ins_node_t*     node,   /*!< in: row insert node */
        que_thr_t*      thr)    /*!< in: query thread */
{
    if (node->state == INS_NODE_ALLOC_ROW_ID) {
      /*若innodb表没有主键和唯一键，用row_id组织索引*/
      row_ins_alloc_row_id_step(node);

      /*获取row_id的索引*/
      node->index = dict_table_get_first_index(node->table);
      node->entry = UT_LIST_GET_FIRST(node->entry_list);
    }

    /*遍历所有索引，向每个索引中插入记录*/
    while (node->index != NULL) {
      /*如果不是倒排索引*/
      if (node->index->type != DICT_FTS) {
        /* 向索引中插入记录 */
        err = row_ins_index_entry_step(node, thr);

        if (err != DB_SUCCESS) {
          return(err);
        }
      }                                                                                                                                                                                         
      /*获取下一个索引*/
      node->index = dict_table_get_next_index(node->index);
      node->entry = UT_LIST_GET_NEXT(tuple_list, node->entry);

    }
}
```
插入单个索引
``` c++
static __attribute__((nonnull, warn_unused_result))
dberr_t
row_ins_index_entry_step(
/*=====================*/
        ins_node_t*     node,   /*!< in: row insert node */
        que_thr_t*      thr)    /*!< in: query thread */
{
    dberr_t err;

    /*给索引项赋值*/
    row_ins_index_entry_set_vals(node->index, node->entry, node->row);

    /*插入索引项*/
    err = row_ins_index_entry(node->index, node->entry, thr);

    return(err);
}

static
dberr_t
row_ins_index_entry(
/*================*/
        dict_index_t*   index,  /*!< in: index */
        dtuple_t*       entry,  /*!< in/out: index entry to insert */
        que_thr_t*      thr)    /*!< in: query thread */
{

    if (dict_index_is_clust(index)) {
      /* 插入聚集索引 */
      return(row_ins_clust_index_entry(index, entry, thr, 0));
    } else {
      /* 插入二级索引 */
      return(row_ins_sec_index_entry(index, entry, thr));
    }
}
```
row_ins_clust_index_entry 和 row_ins_sec_index_entry 函数结构类似，只分析插入聚集索引
``` c++
UNIV_INTERN
dberr_t
row_ins_clust_index_entry(
/*======================*/
        dict_index_t*   index,  /*!< in: clustered index */
        dtuple_t*       entry,  /*!< in/out: index entry to insert */
        que_thr_t*      thr,    /*!< in: query thread */
        ulint           n_ext)  /*!< in: number of externally stored columns */
{
        if (UT_LIST_GET_FIRST(index->table->foreign_list)) {
                err = row_ins_check_foreign_constraints(
                        index->table, index, entry, thr);
                if (err != DB_SUCCESS) {
                        return(err);
                }
        }
        
        /* flush log，make checkpoint（如果需要） */
        log_free_check();

        /* 先尝试乐观插入，修改叶子节点 BTR_MODIFY_LEAF */
        err = row_ins_clust_index_entry_low(
                0, BTR_MODIFY_LEAF, index, n_uniq, entry, n_ext, thr, 
                &page_no, &modify_clock);
                
        if (err != DB_FAIL) {
                DEBUG_SYNC_C("row_ins_clust_index_entry_leaf_after");
                return(err);
        }    
		
    /* flush log，make checkpoint（如果需要） */
        log_free_check();

    /* 乐观插入失败，尝试悲观插入 BTR_MODIFY_TREE */
        return(row_ins_clust_index_entry_low(
                        0, BTR_MODIFY_TREE, index, n_uniq, entry, n_ext, thr,
                        &page_no, &modify_clock));
}
```
row_ins_clust_index_entry_low 和 row_ins_sec_index_entry_low 函数结构类似，只分析插入聚集索引
``` c++
UNIV_INTERN
dberr_t
row_ins_clust_index_entry_low(
/*==========================*/
        ulint           flags,  /*!< in: undo logging and locking flags */
        ulint           mode,   /*!< in: BTR_MODIFY_LEAF or BTR_MODIFY_TREE,
                                depending on whether we wish optimistic or
                                pessimistic descent down the index tree */
        dict_index_t*   index,  /*!< in: clustered index */
        ulint           n_uniq, /*!< in: 0 or index->n_uniq */
        dtuple_t*       entry,  /*!< in/out: index entry to insert */
        ulint           n_ext,  /*!< in: number of externally stored columns */
        que_thr_t*      thr,    /*!< in: query thread */
        ulint*          page_no,/*!< *page_no and *modify_clock are used to decide
                                whether to call btr_cur_optimistic_insert() during
                                pessimistic descent down the index tree.
                                in: If this is optimistic descent, then *page_no
                                must be ULINT_UNDEFINED. If it is pessimistic
                                descent, *page_no must be the page_no to which an
                                optimistic insert was attempted last time
                                row_ins_index_entry_low() was called.
                                out: If this is the optimistic descent, *page_no is set
                                to the page_no to which an optimistic insert was
                                attempted. If it is pessimistic descent, this value is
                                not changed. */
        ullint*         modify_clock) /*!< in/out: *modify_clock == ULLINT_UNDEFINED
                                during optimistic descent, and the modify_clock
                                value for the page that was used for optimistic
                                insert during pessimistic descent */
{
		/* 将cursor移动到索引上待插入的位置 */
		btr_cur_search_to_nth_level(index, 0, entry, PAGE_CUR_LE, mode,                                                                                                                                     
                                    &cursor, 0, __FILE__, __LINE__, &mtr);
                        
                        /*根据不同的flag检查主键冲突*/
                        err = row_ins_duplicate_error_in_clust_online(
                                n_uniq, entry, &cursor,
                                &offsets, &offsets_heap);
                                
                        err = row_ins_duplicate_error_in_clust(
                                flags, &cursor, entry, thr, &mtr);

      /*
        如果要插入的索引项已存在，则把insert操作改为update操作
        索引项已存在，且没有主键冲突，是因为之前的索引项对应的数据被标记为已删除
        本次插入的数据和上次删除的一样，而索引项并未删除，所以变为update操作		
      */
        if (row_ins_must_modify_rec(&cursor)) {
                /* There is already an index entry with a long enough common
                prefix, we must convert the insert into a modify of an
                existing record */
                mem_heap_t*     entry_heap      = mem_heap_create(1024);
				
                /* 更新数据到存在的索引项 */
                err = row_ins_clust_index_entry_by_modify(
                        flags, mode, &cursor, &offsets, &offsets_heap,
                        entry_heap, &big_rec, entry, thr, &mtr);
                
                /*如果索引正在online_ddl，先记录insert*/
                if (err == DB_SUCCESS && dict_index_is_online_ddl(index)) {
                        row_log_table_insert(rec, index, offsets);
                }

                /*提交mini transaction*/
                mtr_commit(&mtr);
                mem_heap_free(entry_heap);
        } else {
                rec_t*  insert_rec;

                if (mode != BTR_MODIFY_TREE) {
                		/*进行一次乐观插入*/
                        err = btr_cur_optimistic_insert(
                                flags, &cursor, &offsets, &offsets_heap,
                                entry, &insert_rec, &big_rec,
                                n_ext, thr, &mtr);
                } else {
                    /*
                      如果buffer pool余量不足25%，插入失败，返回DB_LOCK_TABLE_FULL
                      处理DB_LOCK_TABLE_FULL错误时，会回滚事务
                      防止大事务的锁占满buffer pool(注释里写的)
                    */
                        if (buf_LRU_buf_pool_running_out()) {

                                err = DB_LOCK_TABLE_FULL;
                                goto err_exit;
                        }

                        if (/*太长了，略*/) {
                                /*进行一次乐观插入*/
                                err = btr_cur_optimistic_insert(
                                        flags, &cursor,
                                        &offsets, &offsets_heap,
                                        entry, &insert_rec, &big_rec,
                                        n_ext, thr, &mtr);
                        } else {
                                err = DB_FAIL;
                        }

                        if (err == DB_FAIL) {
                                /*乐观插入失败，进行悲观插入*/
                                err = btr_cur_pessimistic_insert(
                                        flags, &cursor,
                                        &offsets, &offsets_heap,
                                        entry, &insert_rec, &big_rec,
                                        n_ext, thr, &mtr);
                        }
                }

}
```

- 引用 http://mysql.taobao.org/monthly/2017/09/10/