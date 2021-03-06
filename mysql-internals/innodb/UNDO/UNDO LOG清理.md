### UNDO LOG清理

#### 带着问题去思考

- 清理的粒度是什么？（undo log record）
- 所谓清理到底是在清理什么东西？（undo log以及对应的原始数据记录）
- 哪些undo log及其对应的原始记录可以被清理？（事务已经提交且其修改的行记录不可能再被当前已经未来事务所访问的）
- 是不是要按照事务提交顺序来进行清理？（是），如何实现？
- 如何回收被清理完的UNDO LOG所占用的磁盘空间？

而下面的文章内容主要就是描述这些问题在INNODB中的具体实现。

#### 数据结构

**trx_purge_t**

```c++
struct trx_purge_t {
  // 清理开始时的最老的ReadView,任何在该ReadView之前提交的事务UNDO方可被purge
  ReadView view;               

  purge_iter_t iter;  
  purge_iter_t limit; 
  ibool next_stored; 
  // 下一个要purge的回滚段
  trx_rseg_t *rseg;
  // 下一个要purge的undo log record所在page
  // 每次扫描完后
  page_no_t page_no; 
  ulint offset;  
  page_no_t hdr_page_no;
  ulint hdr_offset; 
  // 用来选择下一个要purge的undo log的辅助对象
  TrxUndoRsegsIterator *rseg_iter;

  // 小根堆,用于保存所有需要purge的回滚段
  // 按照trx_no有序插入
  purge_pq_t *purge_queue;
};
```

一些关键成员的说明：

> * purge_queue：维护了一个最小堆，每次pop最顶元素，可以得到trx_no最小的回滚段(*TrxUndoRsegsIterator::set_next*)。每个回滚段第1次事务提交时向最小堆插入数据 (见函数*trx_serialisation_number_get*)。后面每次处理完1个rseg后，会把下一个undo记录的trx_no压入到这个最小堆（*trx_purge_rseg_get_next_history_log*），作为rseg的cursor。
> * hdr_page_no：
> * hdr_offset：
> * space_id：
> * page_no：记录上一次搜寻结束时的undo log record所在的page no
> * offset：记录上一次搜寻结束时的undo log record所在的page内的offset
> * rseg：记录下一次搜寻的回滚段
> * next_stored：为false代表当前的回滚段已经无效（即没有undo log record可以purge），在*trx_purge_fetch_next_rec*中，会判断该值，如果为false，会调用*trx_purge_choose_next_log*来选择下一个回滚段，调用函数*purge_sys->rseg_iter->set_next()*。该值在函数*trx_purge_rseg_get_next_history_log*一进来的时候就被设置为FALSE，又在函数*trx_purge_read_undo_rec*中被设置为TRUE

‌**TrxUndoRsegsIterator**

```c++
/** Choose the rollback segment with the smallest trx_no. */
struct TrxUndoRsegsIterator {
  const page_size_t set_next();
 private:
  trx_purge_t *m_purge_sys;

  TrxUndoRsegs m_trx_undo_rsegs;
  Rseg_Iterator m_iter;
};
```

‌该对象是一个辅助结构，用来选择具有最小*trx_no*的回滚段的UNDO LOG。

**trx_no和trx_id**‌

trx_id为事务生成时的id

trx_no为事务提交时的id，通过trx_no维护了事务的提交序。事务commit时按照trx_no顺序，把事务当前的undo log挂到回滚段的undo history list的表头，指向事务最近的undo log。‌

#### 事务提交时对UNDO LOG处理

**回收INSERT UNDO**‌

insert undo log直到事务释放记录锁、从读写事务链表清除、以及关闭read view后才开始清理，调用函数*trx_undo_insert_cleanup*：

```c++
// 事务结束时会尝试设置trx_undo_t的状态:
// TRX_UNDO_CACHED、TRX_UNDO_TO_FREE或者TRX_UNDO_TO_PURGE
// 如果是INSERT UNDO：
//  如果只占用一个page且page利用率小于3/4设置为TRX_UNDO_CACHED
//  否则,设置为TRX_UNDO_TO_FREE
// 如果是UPDATE UNDO，设置为TRX_UNDO_TO_PURGE
page_t *trx_undo_set_state_at_finish(...)
{
  ...
  if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) <
                         TRX_UNDO_PAGE_REUSE_LIMIT) {
    state = TRX_UNDO_CACHED;
  } else if (undo->type == TRX_UNDO_INSERT) {
    state = TRX_UNDO_TO_FREE;
  } else {
    state = TRX_UNDO_TO_PURGE;
  }
  ...
}

// 事务提交时清理insert undo log
void trx_commit_in_memory(...) 
{
  if (trx->rsegs.m_redo.insert_undo != nullptr) {
    trx_undo_insert_cleanup(&trx->rsegs.m_redo, false);
  }
}

void trx_undo_insert_cleanup(...) {
  // 从回滚段对象的insert undo 链表中摘除
  UT_LIST_REMOVE(rseg->insert_undo_list, undo);

  // 如果状态是TRX_UNDO_CACHED，加入到回滚段的insert undo cache链表,以备复用
  // undo->state在上面的trx_undo_set_state_at_finish()中被设置
  if (undo->state == TRX_UNDO_CACHED) {
    UT_LIST_ADD_FIRST(rseg->insert_undo_cached, undo);
  } else {
    // 如果不能复用,则直接释放insert undo log占用的空间
    trx_undo_seg_free(undo, noredo);
    rseg->curr_size -= undo->size;
    // 释放undo log的内存对象
    trx_undo_mem_free(undo);
  }
}
```

1. 因为INSERT UNDO保存的是事务新插入的行记录内容，事务提交后这些记录已经被写入聚簇索引，UNDO LOG不再有用，因此，可以立即释放
2. 如果Undo状态为TRX_UNDO_CACHED，则加入到回滚段的insert_undo_cached链表上，后面直接复用
3. 否则，将该undo所占的segment及其所占用的回滚段的slot全部释放掉（*trx_undo_seg_free*），修改当前回滚段的大小(*rseg->curr_size*)，并释放undo对象分配的内存（*trx_undo_mem_free*），与update_undo不同，insert_undo不会放到History list上
4. 事务完成提交后，需要将其使用的回滚段引用计数rseg->trx_ref_count减1。

**回收UPDATE UNDO**

如果当前事务包含update undo，还需要将当前undo所在的回滚段（及当前最大的事务号）加入Purge队列（purge_sys->purge_queue）中。

```c++
void trx_undo_update_cleanup(...)
{
  // 将update undo log加入到回滚段的history list中
  trx_purge_add_update_undo_to_history(
      trx, undo_ptr, undo_page, update_rseg_history_len, n_added_logs, mtr);

  // 将update undo从回滚段的update_undo_list中摘除
  UT_LIST_REMOVE(rseg->update_undo_list, undo);

  // 在trx_undo_set_state_at_finish中设置了state
  // update undo log也可以被cache以便复用
  if (undo->state == TRX_UNDO_CACHED) {
    UT_LIST_ADD_FIRST(rseg->update_undo_cached, undo);
  } else {
    trx_undo_mem_free(undo);
  }
}

// 将update undo log加入到回滚段的TRX_RSEG_HISTORY_SIZE链表头部
// 保证该链表是按照事务提交顺序逆向排列
void trx_purge_add_update_undo_to_history(...)
{
  rseg_header = trx_rsegf_get(undo->rseg->space_id, undo->rseg->page_no,
                              undo->rseg->page_size, mtr);
  undo_header = undo_page + undo->hdr_offset;

  if (undo->state != TRX_UNDO_CACHED) {
    ulint hist_size;
    // 将回滚段header page该undo log对应的slot清空,可以用于下次分配
    trx_rsegf_set_nth_undo(rseg_header, undo->id, FIL_NULL, mtr);
    hist_size =
        mtr_read_ulint(rseg_header + TRX_RSEG_HISTORY_SIZE, MLOG_4BYTES, mtr);
    mlog_write_ulint(rseg_header + TRX_RSEG_HISTORY_SIZE,
                     hist_size + undo->size, MLOG_4BYTES, mtr);
  }

  // 将该update undo log加入至回滚段的history undo list链表上
  flst_add_first(rseg_header + TRX_RSEG_HISTORY,
                 undo_header + TRX_UNDO_HISTORY_NODE, mtr);

  mlog_write_ull(undo_header + TRX_UNDO_TRX_NO, trx->no, mtr);

  if (!undo->del_marks) {
    mlog_write_ulint(undo_header + TRX_UNDO_DEL_MARKS, FALSE, MLOG_2BYTES, mtr);
  }
  // 设置回滚段的last_page_no等字段,以便后台purge线程知道该从该回滚段
  // 的哪个位置处开始扫描undo log record以便进行清理
  if (rseg->last_page_no == FIL_NULL) {
    rseg->last_page_no = undo->hdr_page_no;
    rseg->last_offset = undo->hdr_offset;
    rseg->last_trx_no = trx->no;
    rseg->last_del_marks = undo->del_marks;
  }
}
```

‌update undo不能像insert undo一样在事务结束后直接释放，因为其中包含的老版本数据，在本事务结束后可能仍然被其他事务所访问，只能等到不可能再被访问时才可以释放。对于这类undo log的清理：

1. 将undo log加入到history list上，调用*trx_purge_add_update_undo_to_history*：
   - 如果该undo log不满足可缓存的条件（状态为TRX_UNDO_CACHED，如上述），则将其占用的slot设置为FIL_NULL，意为slot空闲，同时更新回滚段头的TRX_RSEG_HISTORY_SIZE值，将当前undo占用的page数累加上去
   - 将当前undo加入到回滚段的TRX_RSEG_HISTORY链表上，作为链表头节点，节点指针为UNDO头的TRX_UNDO_HISTORY_NODE
   - 更新*trx_sys->rseg_history_len*，如果只有普通的update_undo，则加1，如果还有临时表的update_undo，则加2，然后唤醒purge线程
   - 将当前事务的trx_no（事务提交number）写入undo头的TRX_UNDO_TRX_NO段；
   - 如果不是delete-mark操作，将undo头的TRX_UNDO_DEL_MARKS更新为false;
   - 如果undo所在回滚段的`rseg->last_page_no`为FIL_NULL，表示该回滚段的旧的清理已经完成，进行如下赋值，记录这个回滚段上下一个需要purge的undo记录信息：
2. 如果undo需要cache，将undo对象放到回滚段的update_undo_cached链表上；否则释放undo对象（trx_undo_mem_free）。

由于UPDATE UNDO LOG在事务结束时不能被直接释放，我们需要额外的机制来处理这些UNDO LOG。为此，INNODB引入了后台purge线程。我们重点关注以下几个问题：

- 如何收集需要被purge的undo log？
- 如何清理UNDO LOG以及其相关的索引记录？
- 如何释放UNDO LOG占用的物理文件空间？

#### 收集需要被清理的undo log record

每个UNDO LOG中可能会包含多个RECORD，每个RECORD包含对行记录的一次更新。purge也是按照UNDO LOG RECORD为单位来进行。

```c++
ulint trx_purge(...)
{
  // 首先, 保存一份当前最老的ReadView
  // 接下来判断UNDO LOG是否可以被purge还得靠它
  trx_sys->mvcc->clone_oldest_view(&purge_sys->view);
  // 在函数trx_purge_attach_undo_recs中扫描可以被purge的undo log record
  // batch_size限定了每轮purge可以处理的最大undo page数(每个page内可以存储一个或者多个log record)
  n_pages_handled =
      trx_purge_attach_undo_recs(n_purge_threads, purge_sys, batch_size);
}

ulint trx_purge_attach_undo_recs(...) 
{
  for (ulint i = 0; n_pages_handled < batch_size; ++i) {
    purge_node_t::rec_t rec;
    // 获取下一个要被purge的undo log record
    rec.undo_rec = trx_purge_fetch_next_rec(&rec.modifier_trx_id, &rec.roll_ptr,
                                            &n_pages_handled, heap);
    // 将所有的undo log record按照table id进行group以减少后续锁冲突概率
    // 对于同一个batch内属于同一个table的undo record放在一起,交由同一个worker线程处理
    // 我们忽略这段又臭又长的代码
    ...
  }
  return (n_pages_handled);
}
```

‌函数*trx_purge_attach_undo_recs*啰嗦地写了一堆，目的只有一个：扫描所有undo log record，将可以被清理的undo log record收集起来，保存在一个专门对象内。

‌这里很多的代码是为了一个优化：将本批次内所有属于同一个table的undo log record保存在一起，然后交由同一个purge线程处理，这样可以减少多线程并发加锁导致的锁争抢。

‌这段代码的关键在于如何扫描可以被清理的的undo log record。这里需要解决以下几个问题：

1. 从哪个回滚段开始扫描？是每次无脑从第一个回滚段还是从上次扫描结束位置继续扫描？
2. 如何判定一个undo log record可以被purge？‌

带着以上几个问题我们来翻阅代码：

```c++
trx_undo_rec_t *trx_purge_fetch_next_rec(...)
{
  // 如果超过了purge_sys->view的最小trx_id
  // 那说明该undo log中的记录可能还被当前活跃事务引用
  // 还不能做purge,调用者在也会判断返回值,如果为null,那么就跳出本轮扫描
  // 因为每次选择的undo log一定是当前拥有最小trx_no的
  // 所以只要它超过了purge_sys->view,那么其他的已提交事务的undo也一定超过
  // 因而没有必要再扫描其他的了
  if (purge_sys->iter.trx_no >= purge_sys->view.low_limit_no()) {
    return (nullptr);
  }
  ...
  return (trx_purge_get_next_rec(n_pages_handled, heap));
}

trx_undo_rec_t *trx_purge_get_next_rec(...)
{
  space = purge_sys->rseg->space_id;
  page_no = purge_sys->page_no;
  offset = purge_sys->offset;

  // 首先从上次搜寻结束处的page继续寻找下一个可回收的undo log record
  undo_page =
      trx_undo_page_get_s_latched(page_id_t(space, page_no), page_size, &mtr);
  rec = undo_page + offset;
  rec2 = rec;

  // 首先从上次搜寻结束处的page继续寻找下一个可回收的undo log record
  for (;;) {
    next_rec = trx_undo_page_get_next_rec(rec2, purge_sys->hdr_page_no,
                                          purge_sys->hdr_offset);
    // 如果该page内没有undo log record了,就开始从该undo log的下一个page继续寻找
    // undo log包含的所有undo page构成了一个双向链表
    // 表头被记录在undo segment header中
    if (next_rec == NULL) {
      rec2 = trx_undo_get_next_rec(rec2, purge_sys->hdr_page_no,
                                   purge_sys->hdr_offset, &mtr);
      break;
    }
    rec2 = next_rec;
    type = trx_undo_rec_get_type(rec2);
    if (type == TRX_UNDO_DEL_MARK_REC) {
      break;
    }
  }

  // 如果一个undo log内的所有undo page都遍历完了还无法找到可purge的undo log record
  // 这时候需要搜寻下一个undo log
  // 因为回滚段内已经释放的undo log都被串联成为一个history list
  // 而每个undo log中都记录了自己的前后项信息,所以这种查询是很快的
  if (rec2 == NULL) {
    // 注意: 当前回滚段(purge_sys->rseg)内的undo log history list可能为空
    // trx_purge_rseg_get_next_history_log内获取下一个undo log, 更新该回滚段的
    // trx_no, 再将其插入至purge_sys->purge_queue中
    // 这是因为purge需要按照事务的trx_no来有序进行
    // 因此,下一次可能会选择其它拥有更小trx_no的回滚段的undo log进行处理
    trx_purge_rseg_get_next_history_log(purge_sys->rseg, n_pages_handled);
    // 选择下一个undo log处理,该undo log一定拥有最小的trx_no
    trx_purge_choose_next_log();
    undo_page =
        trx_undo_page_get_s_latched(page_id_t(space, page_no), page_size, &mtr);
    rec = undo_page + offset;
  } else {
    // 如果找到了可purge的undo log record
    // 这时候更新下次搜寻的起始位置等信息
    page = page_align(rec2);
    purge_sys->offset = rec2 - page;
    purge_sys->page_no = page_get_page_no(page);
    purge_sys->iter.undo_no = trx_undo_rec_get_undo_no(rec2);
    purge_sys->iter.undo_rseg_space = space;

    if (undo_page != page) {
      (*n_pages_handled)++;
    }
  }
  rec_copy = trx_undo_rec_copy(rec, heap);
  return (rec_copy);
}
```

*trx_purge_get_next_rec*的算法也比较简单：

1. 首先从上次搜寻结束位置处开始向后寻找下一个可清理的undo log record，该位置被记录在purge_sys->page_no和purge_sys->offset中；
2. 如果1中的undo page中找不到undo log record，那么会继续查找该undo log中的下一个undo page，因为一个undo log内的所有undo page通过链表串联，因此这种查找很容易，见*trx_undo_get_next_rec*
3. 如果2还是找不到，说明当前undo log所有record都扫描完毕，接下来要选择下一个扫描的undo log。原则是：选择剩余undo log中trx_no最小的。为此，*purge_sys->purge_queue*被构建为一个小根堆：每个回滚段第1次事务提交时向最小堆插入数据 (见函数*trx_serialisation_number_get*)。后面每次处理完回滚段的undo log后，会把其下一个undo log的trx_no压入到这个最小堆（*trx_purge_rseg_get_next_history_log*）。然后，清理系统在选择下一个undo log就很容易了：只要选择堆顶的那个回滚段的undo log即可。

看看*trx_purge_rseg_get_next_history_log*的实现：

```c++
// 从回滚段rseg中找到下一个需要被purge的undo log
// 当前的回滚段的某个undo log上的所有undo log record被扫描完成后
// 接下来找到该回滚段上的下一个undo log(按照事务的trx_no排序)
// 将其加入至purge_sys->purge_queue, 接下来在trx_purge_choose_next_log
// 函数中调用purge_sys->purge_queue->pop()会找到下一个需要扫描的回滚段
// 之所以这么做是为了保证undo log的扫描要按照trx_no的顺序来进行
void trx_purge_rseg_get_next_history_log(
    trx_rseg_t *rseg,       
    ulint *n_pages_handled) 
{
  purge_sys->iter.trx_no = rseg->last_trx_no + 1;
  purge_sys->iter.undo_no = 0;
  purge_sys->iter.undo_rseg_space = SPACE_UNKNOWN;
  purge_sys->next_stored = FALSE;

  undo_page = trx_undo_page_get_s_latched(
      page_id_t(rseg->space_id, rseg->last_page_no), rseg->page_size, &mtr);

  log_hdr = undo_page + rseg->last_offset;

  // 找到当前undo log的在回滚段history链表上的前一个undo log
  // 因为回滚段的history链表上的事务是按照事务提交逆向排序
  // 因而找undo log的前向记录即可
  prev_log_addr = trx_purge_get_log_from_hist(
      flst_get_prev_addr(log_hdr + TRX_UNDO_HISTORY_NODE, &mtr));

  // 如果没有,直接返回
  if (prev_log_addr.page == FIL_NULL) {
    ...
    return;
  }

  log_hdr = trx_undo_page_get_s_latched(page_id_t(rseg->space_id, prev_log_addr.page),
                                  rseg->page_size, &mtr) + prev_log_addr.boffset;
  trx_no = mach_read_from_8(log_hdr + TRX_UNDO_TRX_NO);
  del_marks = mach_read_from_2(log_hdr + TRX_UNDO_DEL_MARKS);

  // 记录当前回滚段下一次要扫描的undo log的信息
  rseg->last_page_no = prev_log_addr.page;
  rseg->last_offset = prev_log_addr.boffset;
  rseg->last_trx_no = trx_no;
  rseg->last_del_marks = del_marks;

  // 加入至purge_sys->purge_queue中
  TrxUndoRsegs elem(rseg->last_trx_no);
  elem.push_back(rseg);
  purge_sys->purge_queue->push(elem);
}

void trx_purge_choose_next_log(void) {
  // 从purge_sys->purge_queue中pop出拥有最小trx_no的回滚段
  // 并保存在purge_sys->rseg中
  // 在set_next()中同时会更新purge_sys->iter.trx_no
  // 表示下一次要扫描的undo log的trx_no
  // 下一次调用trx_purge_get_next_rec()内会判断该trx_no和view的关系
  // 以决定该undo log内的undo log record是否可以被purge
  const page_size_t &page_size = purge_sys->rseg_iter->set_next();
  if (purge_sys->rseg != NULL) {
    trx_purge_read_undo_rec(purge_sys, page_size);
  } else {
    os_thread_yield();
  }
}
```

‌至此，我们就搞清楚了purge系统如何收集待清理的undo log record。接下来研究如何清理undo log record。

**UNDO LOG RECORD清理**

```c++
que_thr_t *row_purge_step(que_thr_t *thr) {
  node = static_cast<purge_node_t *>(thr->run_node);
  if (node->recs != nullptr && !node->recs->empty()) {
    rec = node->recs->front();
    node->recs->pop_front();

    node->roll_ptr = rec.roll_ptr;
    node->modifier_trx_id = rec.modifier_trx_id;
    row_purge(node, rec.undo_rec, thr);
  }
}

void row_purge(...)
{
  while (row_purge_parse_undo_rec(node, undo_rec, &updated_extern, thd, thr)) {
    bool purged = row_purge_record(node, undo_rec, thr, updated_extern, thd);

    if (purged || srv_shutdown_state.load() != SRV_SHUTDOWN_NONE) {
      return;
    }
    // 等待一段时间后重新尝试purge
    os_thread_sleep(1000000);
  }
}

bool row_purge_record_func(...)
{
  dict_index_t *clust_index;
  clust_index = node->table->first_index();
  // node->index:记录二级索引
  node->index = clust_index->next();

  switch (node->rec_type) {
    case TRX_UNDO_DEL_MARK_REC:
      purged = row_purge_del_mark(node);
      ...
      break;
    default:
      ...
    case TRX_UNDO_UPD_EXIST_REC:
      row_purge_upd_exist_or_extern(thr, node, undo_rec);
      break;
  }
  ...
}
```

‌清理UNDO LOG RECORD时需要判断：

- 如果该log record有TRX_UNDO_DEL_MARK_REC标记，那说明这是一个删除操作产生的undo log record。这时候清理需要做的是删除这个聚簇索引记录。而从以前的知识我们知道，这一般会在删除聚簇索引或者更新聚簇索引值时候产生该记录，这时候同样会在二级索引中产生一个删除记录。因此，我们除了删除聚簇索引中的记录外，还要删除二级索引中的标记删除的记录(*row_purge_remove_sec_if_poss*)。
- 否则，对于聚簇索引来说，这只是一个原地更新，但有可能是二级索引上产生了标记删除操作（例如针对二级索引值进行更新），这时候我们就需要根据回滚段记录去检查二级索引记录序是否发生变化，并执行清理操作（*row_purge_upd_exist_or_extern*）

‌

#### UNDO LOG文件回收

*trx_purge*在处理完一个batch（通常是300）之后，调用*trx_purge_truncate_history*。purge_sys对每一个rseg尝试释放undo log（*trx_purge_truncate_rseg_history*）。

```c++
ulint srv_do_purge(ulint *n_total_purged)
{
  ...
	ulint undo_trunc_freq = purge_sys->undo_trunc.get_rseg_truncate_frequency();
  ulint rseg_truncate_frequency = ut_min(
        static_cast<ulint>(srv_purge_rseg_truncate_frequency), undo_trunc_freq);
	// 是否执行UNDO SPACE文件的截断受参数srv_purge_rseg_truncate_frequency的控制
  // 默认每128次purge执行一次undo space文件的截断
  n_pages_purged = trx_purge(n_use_threads, srv_purge_batch_size,
                               (++count % rseg_truncate_frequency) == 0);
  ...
}

ulint trx_purge(
			ulint n_purge_threads, 
      ulint batch_size,
      bool truncate)
{
  ...
  // 完成purge后再由参数truncate决定是否需要进行undo space的截断
  if (truncate || srv_upgrade_old_undo_found) {
    trx_purge_truncate();
  }
}

void trx_purge_truncate() {
  if (purge_sys->limit.trx_no == 0) {
    trx_purge_truncate_history(&purge_sys->iter, &purge_sys->view);
  } else {
    trx_purge_truncate_history(&purge_sys->limit, &purge_sys->view);
  }
}

void trx_purge_truncate_history(...)
{
  // 检查独立表空间内所有的回滚段
  // 对每个回滚段,检查其HISTORY LOG链表上的每个UNDO LOG空间是否可以被释放
  for (auto undo_space : undo::spaces->m_spaces) {
    undo::Truncate &ut = purge_sys->undo_trunc;
    // 清理每个回滚段上的undo logs
    for (auto rseg : *undo_space->rsegs()) {
      trx_purge_truncate_rseg_history(rseg, limit);
    }
  }
    
  // 处理系统表空间和临时表空间
	...

  for (i = 0; i < space_count; i++) {
    // 寻找可以被truncate的undo space
    if (!trx_purge_mark_undo_for_truncate()) {
      break;
    }
		...
		// truncate undo space
    if (!trx_purge_truncate_marked_undo()) {
      break;
    }
  }
}

// 检查并释放回滚段内可以被清理的undo log空间
// 其原理很简单: 扫描TRX_RSEG_HISTORY链表上的所有undo log
// 如果其可以被清理,那么释放其占用的undo segment
// 关键在于如何判断是否可以被清理：其实也就是看该undo log的所有record是否在前面的purge线程中被处理了
void trx_purge_truncate_rseg_history(...)
{
  rseg_hdr = trx_rsegf_get(rseg->space_id, rseg->page_no, rseg->page_size, &mtr);
  hdr_addr = trx_purge_get_log_from_hist(
      flst_get_last(rseg_hdr + TRX_RSEG_HISTORY, &mtr));

loop:
  // 如果history链表为空,已经遍历完了所有已经提交事务的undo log
  if (hdr_addr.page == FIL_NULL) {
    return;
  }
  undo_page = trx_undo_page_get(page_id_t(rseg->space_id, hdr_addr.page),
                                rseg->page_size, &mtr);
  log_hdr = undo_page + hdr_addr.boffset;
  undo_trx_no = mach_read_from_8(log_hdr + TRX_UNDO_TRX_NO);

  // limit->trx_no代表什么？
  // limit->trx_no来自purge_sys->limit
  // 而purge_sys->limit来自purge_sys->iter,见trx_purge_attach_undo_recs
  // 而purge_sys->iter则在trx_purge_rseg_get_next_history_log中
  // 被设置为rseg->last_trx_no + 1
  // 这代表了前面我们已经扫描和处理过的最大的事务提交no
  // 如果当前undo log上记载的trx_no超过了该值,说明该undo log还没有执行前面的清理操作
  // 因此该undo log空间不能被释放
  // 另外请记住：回滚段trx history链表上的undo log都是按照trx_no排序的
  // 一旦这个undo log的trx_no超过了limit->trx_no,那在链表上位于其前面的undo log必然也超过了
  // 因而无需再遍历了
  if (undo_trx_no >= limit->trx_no) {
    if (undo_trx_no == limit->trx_no &&
        rseg->space_id == limit->undo_rseg_space) {
      trx_undo_truncate_start(rseg, hdr_addr.page, hdr_addr.boffset,
                              limit->undo_no);
    }
    return;
  }

  prev_hdr_addr = trx_purge_get_log_from_hist(
      flst_get_prev_addr(log_hdr + TRX_UNDO_HISTORY_NODE, &mtr));

  seg_hdr = undo_page + TRX_UNDO_SEG_HDR;

  // 可以释放整个undo segemnt的条件：
  // 1. undo state为TRX_UNDO_TO_PURGE，这在事务提交时被设置，见trx_undo_set_state_at_finish
  // 2. 该undo log page上没有别的undo log
  // 在这里面也会将该undo log从回滚段的TRX_RSEG_HISTORY链表中摘除
  if ((mach_read_from_2(seg_hdr + TRX_UNDO_STATE) == TRX_UNDO_TO_PURGE) &&
      (mach_read_from_2(log_hdr + TRX_UNDO_NEXT_LOG) == 0)) {
    trx_purge_free_segment(rseg, hdr_addr, is_temp);
  } else {
    // 将undo log从回滚段的TRX_RSEG_HISTORY链表中摘除
    trx_purge_remove_log_hdr(rseg_hdr, log_hdr, &mtr);
  }
  // 继续处理下一个undo log
  hdr_addr = prev_hdr_addr;
  goto loop;
}
```

‌大致过程是：把每个purge过的undo log从回滚段的history list移除，如果undo segment中所有的undo log都被释放，可以尝试释放undo segment，这里隐式释放file segment到达释放存储空间的目的。

接下来就是处理表空间truncate。这在函数*trx_purge_mark_undo_for_truncate*中进行：

```c++
bool trx_purge_mark_undo_for_truncate(size_t truncate_count) {
  // 如果当前已经存在undo space被标记截断,那么不用继续扫描,
  auto undo_trunc = &purge_sys->undo_trunc;
  if (undo_trunc->is_marked()) {
    return (true);
  }

  /* In order to implicitly select an undo space to truncate, we need
  at least 2 active UNDO tablespaces.  As long as there is one undo
  tablespace active the server will continue to operate. */
  ulint num_active = 0;

  // 如果某个undo space被标记为inactive,那么直接设置该space为下一个要被
  // truncate的对象即可,然后返回
  for (auto undo_ts : undo::spaces->m_spaces) {
    if (undo_ts->is_inactive_explicit()) {
      undo_trunc->mark(undo_ts);
      undo::spaces->s_unlock();
      return (true);
    }
    num_active += (undo_ts->is_active() ? 1 : 0);
  }

  // 接下来遍历所有的undo space,检查是否有undo space可以被truncate
  undo::spaces->s_lock();

  space_id_t space_num = undo_trunc->get_scan_space_num();
  space_id_t first_space_num_scanned = space_num;

  do {
    auto undo_space = undo::spaces->find(space_num);
    // 判断undo space是否需要截断
    // undo表空间的文件大小，如果超过了innodb_max_undo_log_size
    // 就会被truncate到初始大小，但有一个前提，就是表空间中的undo不再被使用
    // 好像没有哪里判断表空间的undo不再被使用啊？？？
    if (undo_space->needs_truncation()) {
      undo_trunc->increment_scan();
      undo_trunc->mark(undo_space);
      break;
    }
    space_num = undo_trunc->increment_scan();
  } while (space_num != first_space_num_scanned);

  if (!undo_trunc->is_marked()) {
    return (false);
  }
  return (true);
}
```

该函数的目标是检查是否有可以进行truncate的undo 表空间，关键的数据结构是Truncate：

```c++
class Truncate {
 public:
  Truncate()
      : m_space_id_marked(SPACE_UNKNOWN),
        m_purge_rseg_truncate_frequency(
            static_cast<ulint>(srv_purge_rseg_truncate_frequency)),
        m_timer() {
  }

 private:
  // 被标记为需要阶段的undo space id
  space_id_t m_space_id_marked;
};
```

接下来我们看看如何截断一个undo space：

```c++
bool trx_purge_truncate_marked_undo() {
  // 获取mdl锁避免undo space被drop等
  MDL_ticket *mdl_ticket;
  dd_tablespace_get_mdl(space_name.c_str(), &mdl_ticket, false);

  if (!trx_purge_truncate_marked_undo_low(space_num, space_name)) {
    ...
  }
  // 释放MDL锁
  dd_release_mdl(mdl_ticket);
  return (true);
}

bool trx_purge_truncate_marked_undo_low(space_id_t space_num,
                                        std::string space_name)
{
  undo::Truncate *undo_trunc = &purge_sys->undo_trunc;
  undo::Tablespace *marked_space = undo::spaces->find(space_num);

  // 记一个log
  dberr_t err = undo::start_logging(marked_space);

  bool success = trx_undo_truncate_tablespace(marked_space);

  space_id_t new_space_id = marked_space->id();

  // 为该space在DD中设置新的space id
  if (DD_FAILURE == dd_tablespace_set_id_and_state(space_name.c_str(),
                                                   new_space_id, next_state)) {
    return (false);
  }

  return (true);
}

// 真正地truncate undo log space
// 这里复用了原undo space的内存对象,但会为其进行重新初始化
// 同时删除了原undo space的物理文件,创建新文件,分配新的space id
bool trx_undo_truncate_tablespace(undo::Tablespace *marked_space)
{
  bool success = true;
  auto old_space_id = marked_space->id();
  auto space_num = undo::id2num(old_space_id);
  auto marked_rsegs = marked_space->rsegs();

  // 分配一个新的space id
  auto new_space_id = undo::use_next_space_id(space_num);
  const auto n_pages = SRV_UNDO_TABLESPACE_SIZE_IN_PAGES;

  fil_space_t *space = fil_space_get(old_space_id);

  // 删除老的undo space文件,并使用新的space id创建新的undo space文件
  success = fil_replace_tablespace(old_space_id, new_space_id, n_pages);

  fsp_header_init(new_space_id, n_pages, &mtr, false);

  // 设置回滚段array成员
  trx_rseg_array_create(new_space_id, &mtr);

  // 初始化原undo space的每个内存回滚段
  // 分配回滚段header page并初始化等
  for (auto rseg : *marked_rsegs) {
    rseg->space_id = new_space_id;
    rseg->page_no = trx_rseg_header_create(new_space_id, univ_page_size,
                                           PAGE_NO_MAX, rseg->id, &mtr);
 		...
  }
  marked_space->set_space_id(new_space_id);
  return (success);
}
```



#### 问题总结

在文章开始的时候我们提了几个问题，在阅读完代码后我们尝试来回答和总结：‌

**清理的粒度是什么**

UNDO LOG RECORD。在扫描阶段会选择从上次扫描结束位置处开始扫描下一个UNDO LOG RECORD。如果某个UNDO LOG内所有的RECORD都扫描完毕，那会选择下一个拥有最小trx_no的UNDO LOG来继续扫描。

**UNDO LOG的清理到底是在清理什么**‌

因为INNODB中对于主键和二级索引的更新是对老记录标记删除同时插入新记录。所谓的UNDO LOG清理其实就是清理这些被标记删除的老记录。

**哪些undo log及其对应的原始记录可以被清理**‌

事务已经提交且其修改的行记录不可能再被当前已经未来事务所访问。因此，在清理开始前需要拿到当前系统中的最老的ReadView，对于那些在该ReadView之前已提交的事务产生的UNDO LOG可以被清理。

**是不是要按照事务提交顺序来进行清理**‌

是。在purge_sys中维护了一个小根堆(purge_queue)，每次pop便会得到拥有最小trx_no的回滚段的UNDO LOG，选择它purge即可。同时每个回滚段在清理完当前的UNDO LOG后，会从trx_history_list中获得下一个要purge的UNDO LOG，然后将其插入到purge_queue，它在将来会被pop出来执行清理。‌

**如何回收被清理完的UNDO LOG所占用的磁盘空间**