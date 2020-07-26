### Buffer Pool中的几个Page List

主要描述buffer page管理中几个关键的链表。page无非就在这几个链表中挪来挪去，因此，对这几个链表的作用的理解非常关键。

#### free list

free list上维护的是free page，所谓free page是指未被使用的页，当进行数据访问时首先要申请一个free page，然后再从磁盘上读取page数据进行后续操作。

```c++
struct buf_pool_t {
  ...
  // free page list
  UT_LIST_BASE_NODE_T(buf_page_t) free;
  ...
}
```

‌数据库启动时创建并初始化内存池，此时所有的page均是free状态，所有的page都会被添加至free list。

```c++
buf_chunk_t *buf_chunk_init(...)
{
  ...
  for (i = chunk->size; i--;) {
    buf_block_init(buf_pool, block, frame);
    ...
    /* Add the block to the free list */
    UT_LIST_ADD_LAST(buf_pool->free, &block->page);
    ...
  }
}
```

‌分配free page时首先从free list中获取：

```c++
buf_block_t *buf_LRU_get_free_only(buf_pool_t *buf_pool) {
  buf_block_t *block;

  block = reinterpret_cast<buf_block_t *>(UT_LIST_GET_FIRST(buf_pool->free));

  while (block != NULL) {
    return block;
  }
  ...
}
```

‌如果free list为空，那需要从其他地方释放一些被占用的page，首先将这些page从其他链表中摘除（甚至可能还需要将脏数据写入磁盘），然后将该page插入free page list，例如：

```c++
void buf_LRU_block_free_non_file_page(buf_block_t *block)
{
  ...
  if (buf_get_withdraw_depth(buf_pool) &&
      buf_block_will_withdrawn(buf_pool, block)) {
    ...
  } else {
    buf_block_set_state(block, BUF_BLOCK_NOT_USED);
    mutex_enter(&buf_pool->free_list_mutex);
    UT_LIST_ADD_FIRST(buf_pool->free, &block->page);
    mutex_exit(&buf_pool->free_list_mutex);
  }
}
```

#### lru list

lru上维护的是当前所有正在in use的page。设计lru list的目的在于区分页面的冷热，根据数据的局部性原理，最近被访问的数据页再次被访问的可能性更大。淘汰时需要尽可能保留这些热点数据页。

‌LRU是一个很经典的算法，但InnoDB没有采用这种大路货，而是另辟蹊径的搞了个改进版的LRU，有人管他叫做midpoint LRU，是这样的：

![img](https://gblobscdn.gitbook.com/assets%2F-LeuGf4juyuq9zjuATOD%2F-Lj0-f1HiILqYoOO0JWX%2F-Lj00GmuLfNgt1d11c_j%2Fimage.png?alt=media&token=81a52bca-0b19-4987-8dd4-84fadd7b803e)



‌InnoDB的主要改进点在于每次将磁盘上读出的数据不是直接放到链表的头部，而是放在链表的3/8处（该值可配置），只有在下次再次访问该页时，才会将该页移动到链表头部。这么做的原因在于：

> 某些操作可能会访问表的很多页面（如扫描），这些页面可能是一次性访问，如果直接将其加入LRU链表头部，那么势必会将某些真正的热点页面冲刷至LRU链表尾部，增加了其被淘汰的概率，影响系统性能表现。

‌于是，以midpoint为分割点，lru链表被分为了两部分，midpoint前叫做young list，midpoint后叫做old list。young list保存更热的页面数据，而old list中的数据页更冷，也更容易被淘汰。

初始化时，LRU链表为空。当某个页面被从数据文件中读取后，在*buf_LRU_add_block_low*函数中会判断当前LRU链表长度：

- 如果长度小于BUF_LRU_OLD_MIN_LEN，直接将该page插入至LRU链表头部，此时该page为young
- 如果长度等于BUF_LRU_OLD_MIN_LEN，将LRU链表中的所有page都置为old（*buf_LRU_old_init*）并设置buf_pool->LRU_old
- 如果长度大于BUF_LRU_OLD_MIN_LEN，新页要插入LRU时，调度*buf_LRU_add_block*函数，并将old标记为true，将该页插入到buf_pool->LRU_old的next位置，意味着该page第一次插入，作为old插入至LRU链表中



```c++
void buf_LRU_add_block_low(buf_page_t *bpage, ibool old) {
  buf_pool_t *buf_pool = buf_pool_from_bpage(bpage);

  // 如果当前LRU链表长度小于BUF_LRU_OLD_MIN_LEN阈值,直接将其添加至lru链表头部
  if (!old || (UT_LIST_GET_LEN(buf_pool->LRU) < BUF_LRU_OLD_MIN_LEN)) {
    UT_LIST_ADD_FIRST(buf_pool->LRU, bpage);
    bpage->freed_page_clock = buf_pool->freed_page_clock;
  } else {
    // 如果当前LRU链表长度大于BUF_LRU_OLD_MIN_LEN,将该page插入至LRU链表中的midpoint位置
    // buf_pool->LRU_old就是LRU链表的midpoint,young和old的分界点
    UT_LIST_INSERT_AFTER(buf_pool->LRU, buf_pool->LRU_old, bpage);
    buf_pool->LRU_old_len++;
  }
  // 如果当前LRU链表长度大于BUF_LRU_OLD_MIN_LEN,设置刚刚添加的page为old
  // 并且可能要调整buf_pool->LRU_old的位置,因为此时midpoint可能不再是mid了
  if (UT_LIST_GET_LEN(buf_pool->LRU) > BUF_LRU_OLD_MIN_LEN) {
    buf_page_set_old(bpage, old);
    buf_LRU_old_adjust_len(buf_pool);
  // 如果当前LRU链表长度刚好触及BUF_LRU_OLD_MIN_LEN
  // 此时将LRU链表中的所有page设置为old
  } else if (UT_LIST_GET_LEN(buf_pool->LRU) == BUF_LRU_OLD_MIN_LEN) {
    buf_LRU_old_init(buf_pool);
  } else {
    buf_page_set_old(bpage, buf_pool->LRU_old != NULL);
  }
}
```

‌当该页被再次访问时，会将其从LRU链表的old区移至young区，如下：

```c++
void buf_LRU_make_block_young(buf_page_t *bpage) {
  buf_pool_t *buf_pool = buf_pool_from_bpage(bpage);
  // 先从LRU链表的old区移出
  buf_LRU_remove_block(bpage);
  buf_LRU_add_block_low(bpage, FALSE);
}

void buf_LRU_add_block_low(buf_page_t *bpage, ibool old) {
  buf_pool_t *buf_pool = buf_pool_from_bpage(bpage);
  // 因为传入的old参数为false,所以直接进入if分支
  // 这里直接将page加入至LRU链表的头部(即young区的头部)
  if (!old || (UT_LIST_GET_LEN(buf_pool->LRU) < BUF_LRU_OLD_MIN_LEN)) {
    UT_LIST_ADD_FIRST(buf_pool->LRU, bpage);
    bpage->freed_page_clock = buf_pool->freed_page_clock;
  } else {
    ...  
  }
  ...
}
```

‌总结一下，从启动开始的过程如下:

> 1. 系统初始化时，free链表中的所有页都可以被分配
> 2. 有数据请求的时候，将从磁盘读取到的block放入LRU链表中，该操作直接将所有的block置为young并插入链表头部，直到LRU长度达到BUF_LRU_OLD_MIN_LEN
> 3. 当LRU长度达到BUF_LRU_OLD_MIN_LEN时，InnoDB会做如下操作：
>    1. 将所有的LRU块都置为old（*buf_LRU_old_init*）
>    2. 调度*buf_LRU_old_adjust_len*函数，将buf_pool->LRU_old调整到合适的位置。

> 4. 之后，每次有新的页要插入LRU时，调度*buf_LRU_add_block*函数，并将old标记为true，将该页插入到buf_pool->LRU_old的next位置
>
> 5. 若第四步中的页再次被访问，InnoDB调度*buf_LRU_make_block_young*函数将该页放到LRU链表头部。
>
> 6. free链表分配完，此时需要从LRU尾部寻找可以释放的block，该操作由*buf_LRU_search_and_free_block*执行

#### flush_list

free 类型的 page，一定位于 buf pool 的 free 链表中。clean，dirty 两种类型的 page，一定位于 buf pool 的 LRU 链表中；dirty page 还位于 buf pool 的 flush 链表中。

‌flush list 中的 dirty page，按照 page的 oldest_modification 时间排序，oldest_modification 越大，说明 page 修改的时间越晚，就排在flush链表的头部；oldest_modification 越小，说明 page 修改的时间越早，就排在 flush链表的尾部。当 InnoDB 进行 flush list 的 flush 操作时，从 flush list 链表的尾部开始，写出足够数量的 dirty pages，推进 Checkpoint 点，保证系统的恢复时间。

‌add page to flush list：页面访问/ 修改都被封装为一个 mini-transaction ， 当mini-transactin 提交的时候，也就是该 mini-transaction 修改的页面进入 flsuh list 的时候。（mtr_commit ->mtr_memo_note_modification)

```c++
void mtr_t::commit() {
  m_impl.m_state = MTR_STATE_COMMITTING;
  Command cmd(this);
  if (m_impl.m_n_log_recs > 0 ||
     (m_impl.m_modifications && m_impl.m_log_mode == MTR_LOG_NO_REDO))
  {
    store_log_length();
    cmd.execute();
  } else {
    ...
  }
}

void mtr_t::Command::execute() {
  len = prepare_write();
  if (len > 0) {
    ...
    // handle.start_lsn:代表本事务生成的redo log的start lsn
    // handle.start_lsn:代表本事务生成的redo log的end lsn
    add_dirty_blocks_to_flush_list(handle.start_lsn, handle.end_lsn);
    ...
  }
  ...
}

void mtr_t::Command::add_dirty_blocks_to_flush_list(lsn_t start_lsn,
                                                    lsn_t end_lsn)
{
  Add_dirty_blocks_to_flush_list add_to_flush(start_lsn,
                                              end_lsn,
                                              m_impl->m_log_mode,
                                              m_impl->m_flush_observer);
  Iterate<Add_dirty_blocks_to_flush_list> iterator(add_to_flush);
  m_impl->m_memo.for_each_block_in_reverse(iterator);
}

void add_dirty_page_to_flush_list(mtr_memo_slot_t *slot) const {
  buf_block_t *block;
  block = reinterpret_cast<buf_block_t *>(slot->object);
  buf_flush_note_modification(block, m_start_lsn, m_end_lsn,
                              m_flush_observer);
}

UNIV_INLINE
void buf_flush_note_modification(
    buf_block_t *block,      
    lsn_t start_lsn,         
    lsn_t end_lsn,           
    FlushObserver *observer)
{
  if (end_lsn != 0) {
    block->page.newest_modification = end_lsn;
  } else {
  }

  if (block->page.oldest_modification == 0) {
    buf_pool_t *buf_pool = buf_pool_from_block(block);
    buf_flush_insert_into_flush_list(buf_pool, block, start_lsn);
  } else if (start_lsn != 0) {
  }
}

void buf_flush_insert_into_flush_list(
    buf_pool_t *buf_pool, 
    buf_block_t *block,   
    lsn_t lsn)
{
  ...
  UT_LIST_ADD_FIRST(buf_pool->flush_list, &block->page);
  ...
}
```

‌从上面流程中可以知道，脏页面会在mtr提交时被加入到flush list中，同时我们知道该page在访问时同样也会被加入到LRU list中。因此，脏页面既出现在LRU list，又存在于flush list中。

‌有两个时机会将脏页面从flush list中摘除：

1. 从buffer pool中分配page的时候若发现无free的page，那可能需要淘汰一个脏页面用于分配新的page：

   ```c++
   buf_block_t *buf_LRU_get_free_block(buf_pool_t *buf_pool) {
     ...
   loop:
     ...
     // 从第三轮开始,以后每轮寻找间隔10ms,休眠一段时间可期望有脏页被刷以获得可分配的空闲page
     // 避免无意义的cpu空转
     // 淘汰某些脏页面
     if (!buf_flush_single_page_from_LRU(buf_pool)) {
     }
     goto loop;
   }
   
   bool buf_flush_single_page_from_LRU(buf_pool_t *buf_pool) {
     ...
     } else if (buf_flush_ready_for_flush(bpage, BUF_FLUSH_SINGLE_PAGE)) {
       freed = buf_flush_page(buf_pool, bpage, BUF_FLUSH_SINGLE_PAGE, true);
     }
   }
   
   ibool buf_flush_page(buf_pool_t *buf_pool, buf_page_t *bpage,
                        buf_flush_t flush_type, bool sync) {
     ...
     buf_flush_write_block_low(bpage, flush_type, sync);
   }
   
   void buf_flush_write_block_low(buf_page_t *bpage, buf_flush_t flush_type,
                                  bool sync)
   {
     if (sync) {
       fil_flush(bpage->id.space());
       // 写入成功后调用回调函数
       buf_page_io_complete(bpage, true);
     }
   }
   ```

   在回调函数中会将page从flush list中摘除：

   ```c++
   bool buf_page_io_complete(buf_page_t *bpage, bool evict) {
     ...
     switch (io_type) {
       case BUF_IO_WRITE:
         buf_flush_write_complete(bpage);
         ...
     }
     ...
   }
   
   void buf_flush_write_complete(buf_page_t *bpage) {
     const buf_flush_t flush_type = buf_page_get_flush_type(bpage);
     buf_pool_t *buf_pool = buf_pool_from_bpage(bpage);
   
     if (bpage->copied_page) {
       ...
     } else {
       buf_flush_remove(bpage);
     }
     ...
   }
       
   void buf_flush_remove(buf_page_t *bpage) {
     buf_pool_t *buf_pool = buf_pool_from_bpage(bpage);
     buf_flush_list_mutex_enter(buf_pool);
     buf_flush_remove_low(bpage);
     buf_flush_list_mutex_exit(buf_pool);
   }
   ```

2. Innodb后台会有flush线程，定期地将flush list的脏页flush到磁盘上，这样可以减轻check point的开销，和页面替换时，那些被替换页面的flush开销，而使得读取页面时间增长。flush list的页面根据修改的时间从新到老进行排序，也即是最新的修改，在flush list的头部，最老的修改在flush list的尾部。当flush时，从尾部取page flush到磁盘上。这样的逻辑是跟checkpoint保持一致，checkpoint的流程也是从老到新一步步持久化page，所以可以加快checkpoint。

#### hash list

每个Buffer Pool实例有一个page hash链表，通过它，使用space_id和page_no就能快速找到已经被读入内存的数据页，而不用线性遍历LRU List去查找。注意这个hash表不是InnoDB的自适应哈希，自适应哈希是为了减少Btree的扫描，而page hash是为了避免扫描LRU List。

```c++
struct buf_pool_t {
  ...
  hash_table_t *page_hash;
  ...
}
```

‌初始化page时会将该page插入至page hash中，根据space_id + page_no作为key来计算hash值：

```c++
static void buf_page_init(buf_pool_t *buf_pool, const page_id_t &page_id,
                          const page_size_t &page_size, buf_block_t *block) {
  ...
  // inline ulint fold() const {
  //   if (m_fold == ULINT_UNDEFINED) {
  //     m_fold = (m_space << 20) + m_space + m_page_no;
  //   }
  //   return (m_fold);
  // }
  HASH_INSERT(buf_page_t, hash, buf_pool->page_hash, page_id.fold(),
              &block->page);
}
```

每次要从buffer pool中查找特定page的时候，调用如下函数：

```c++
// 通用函数用于从buffer pool中获取特定page
// 首先便会从hash table中查找
buf_block_t *buf_page_get_gen(const page_id_t &page_id,
                              const page_size_t &page_size, ulint rw_latch,
                              buf_block_t *guess, ulint mode, const char *file,
                              ulint line, mtr_t *mtr, bool dirty_with_no_latch,
                              bool user_req)
{
  ...
  if (block == NULL) {
    block = (buf_block_t *)buf_page_hash_get_low(buf_pool, page_id);
  }
  ...
}

buf_page_t *buf_page_hash_get_low(buf_pool_t *buf_pool,
                                  const page_id_t &page_id)
{
  buf_page_t *bpage;
  HASH_SEARCH(hash, buf_pool->page_hash, page_id.fold(), buf_page_t *, bpage,
              ut_ad(bpage->in_page_hash && !bpage->in_zip_hash &&
                    buf_page_in_file(bpage)),
              page_id.equals_to(bpage->id));
  if (bpage) {
    ...
  }
  return (bpage);
}
```

‌#### withdraw‌

这是buffer pool缩容而设计的链表，这里暂时不作过多讨论。

#### unzip_LRU

暂时不清楚 