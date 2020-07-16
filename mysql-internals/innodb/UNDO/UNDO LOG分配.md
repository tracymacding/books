### UNDO LOG分配

本文只描述独立UNDO表空间下的undo log的分配算法以及实现。根据我们之前的UNDO LOG物理格式描述，分配undo log得先分配回滚段，然后再从回滚段内分配undo log。

#### 分配undo回滚段

根据我们之前UNDO LOG物理格式章节的描述，每个独立的UNDO表空间存在若干个(默认128)个回滚段，而每个回滚段又默认存在1024个UNDO LOG SLOT，分配undo log其实质便是在所有UNDO表空间中找到一个空闲的UNDO LOG SLOT。

分配回滚段的工作在函数*trx_assign_rseg_durable*进行，分配策略是采用round-robin方式。

```c++
static void trx_start_low(trx_t *trx, bool read_write)
{
  ...
  trx->state = TRX_STATE_ACTIVE;
  if (!trx->read_only && (read_write || trx->ddl_operation)) {
    trx_assign_rseg_durable(trx);
    ...
  }
  ...
}

void trx_assign_rseg_durable(trx_t *trx) {
  trx->rsegs.m_redo.rseg = srv_read_only_mode ? nullptr : get_next_redo_rseg();
}

trx_rseg_t *get_next_redo_rseg() {
  ulong target = srv_undo_tablespaces;

  // 从系统表空间分配
  if (target == 0) {
    return (get_next_redo_rseg_from_trx_sys());
  } else {
    // 从独立UNDO表空间分配
    return (get_next_redo_rseg_from_undo_spaces(target));
  }
}

trx_rseg_t *get_next_redo_rseg_from_undo_spaces(
    ulong target_undo_tablespaces)
{
  undo::Tablespace *undo_space;
  ulong target_rollback_segments = srv_rollback_segments;

  static ulint rseg_counter = 0;
  trx_rseg_t *rseg = nullptr;
  ulint current = rseg_counter;
  // rseg_counter表示下一次要分配的回滚段编号
  // 然后根据该编号来计算space id和segment id
  os_atomic_increment_ulint(&rseg_counter, 1);
  while (rseg == nullptr) {
    /* Traverse the rsegs like this: (space, rseg_id)
    (0,0), (1,0), ... (n,0), (0,1), (1,1), ... (n,1), ... */
    ulint window =
        current % (target_rollback_segments * target_undo_tablespaces);
    ulint spaces_slot = window % target_undo_tablespaces;
    ulint rseg_slot = window / target_undo_tablespaces;

    current++;

    undo_space = undo::spaces->at(spaces_slot);

    undo_space->rsegs()->s_lock();

    rseg = undo_space->rsegs()->at(rseg_slot);
    rseg->trx_ref_count++;
    undo_space->rsegs()->s_unlock();
  }
  return (rseg);
}
```

‌分配成功时，递增rseg->trx_ref_count，保证rseg的表空间不会被truncate。

‌临时表操作不记redo log，最终调用get_next_noredo_rseg函数进行分配；其他情况调用get_next_redo_rseg。

#### 分配undo log

‌一旦分配好回滚段，接下来就是在回滚段内分配undo log了，这在函数*trx_undo_assign_undo*内完成：

```c++
dberr_t trx_undo_assign_undo(
    trx_t *trx,            
    trx_undo_ptr_t *undo_ptr, 
    ulint type)      
{
  trx_rseg_t *rseg;
  trx_undo_t *undo;
  mtr_t mtr;
  dberr_t err = DB_SUCCESS;

  // 分配的回滚段
  rseg = undo_ptr->rseg;

  mtr_start(&mtr);
  if (&trx->rsegs.m_noredo == undo_ptr) {
    mtr.set_log_mode(MTR_LOG_NO_REDO);
  } else {
    ut_ad(&trx->rsegs.m_redo == undo_ptr);
  }

  mutex_enter(&rseg->mutex);

  // 首先尝试从回滚段缓存中分配
  undo = trx_undo_reuse_cached(trx, rseg, type, trx->id, trx->xid, &mtr);
  if (undo == NULL) {
    err = trx_undo_create(trx, rseg, type, trx->id, trx->xid, &undo, &mtr);
    ...
  }

  if (type == TRX_UNDO_INSERT) {
    UT_LIST_ADD_FIRST(rseg->insert_undo_list, undo);
    // 该事务后续所有的insert涉及的undo log都会使用这个
    undo_ptr->insert_undo = undo;
  } else {
    UT_LIST_ADD_FIRST(rseg->update_undo_list, undo);
    // 该事务后续所有的update涉及的undo log都会使用这个
    undo_ptr->update_undo = undo;
  }
  ...
  return (err);
}
```

**从回滚段缓存中分配**

```c++
trx_undo_t *trx_undo_reuse_cached(...)
{
  if (type == TRX_UNDO_INSERT) {
    undo = UT_LIST_GET_FIRST(rseg->insert_undo_cached);
    UT_LIST_REMOVE(rseg->insert_undo_cached, undo);
  } else {
    undo = UT_LIST_GET_FIRST(rseg->update_undo_cached);
    UT_LIST_REMOVE(rseg->update_undo_cached, undo);
  }

  undo_page = trx_undo_page_get(page_id_t(undo->space, undo->hdr_page_no),
                                undo->page_size, mtr);
  if (type == TRX_UNDO_INSERT) {
    offset = trx_undo_insert_header_reuse(undo_page, trx_id, mtr);
    trx_undo_header_add_space_for_xid(undo_page, undo_page + offset, mtr);
  } else {
    offset = trx_undo_header_create(undo_page, trx_id, mtr);
    trx_undo_header_add_space_for_xid(undo_page, undo_page + offset, mtr);
  }

  trx_undo_mem_init_for_reuse(undo, trx_id, xid, offset);
  return (undo);
}
```

‌使用cache是为了提升undo log的分配效率。一个undo log在使用完成变得不再有效后便会被释放，一旦满足某些条件，它会被加入到回滚段的undo cache链表，insert 和update undo log有自己独立的链表。

从cache分配就很简单了，只需要从相应类型的缓存链表中取出第一项，然后初始化这个被复用的undo log即可。这里的逻辑比较简单，就不再赘述了。感兴趣的读者请自行研究。

**创建新的undo log**

如果无法从缓存中分配undo log，那也只能退化成来实际分配了，在函数*trx_undo_create*中执行：

```c++
dberr_t trx_undo_create(...)
{
  rseg_header =
      trx_rsegf_get(rseg->space_id, rseg->page_no, rseg->page_size, mtr);

  // 创建undo segment, undo page为segment的第一个page
  err = trx_undo_seg_create(rseg, rseg_header, type, &id, &undo_page, mtr);

  page_no = page_get_page_no(undo_page);

  // 创建undo log header
  offset = trx_undo_header_create(undo_page, trx_id, mtr);

  trx_undo_header_add_space_for_xid(undo_page, undo_page + offset, mtr);

  *undo = trx_undo_mem_create(rseg, id, type, trx_id, xid, page_no, offset);

  return (err);
}
```

‌我们在前面的章节“UNDO LOG物理格式”中说过，创建undo log的关键是分配undo segment。它是个独立的段，每个undo segment包含1个header page（第1个undo page）和若干个记录undo日志的undo page。

‌第1个undo page中存储的是元信息： 首先存储的是undo page的元信息，位于TRX_UNDO_PAGE_HDR到TRX_UNDO_SEG_HDR之间。

‌因此，如果理解了undo log的物理格式，上面的过程就非常简单了，这里不作过多描述。

‌

#### UNDO LOG空间不足时如何处理

在函数*trx_undo_report_row_operation*中，如果出现了已分配的undo log的空间不足以容纳当前记录的老版本，这时候就需要对UNDO LOG进行扩充。

```c++
dberr_t trx_undo_report_row_operation(...)
{
  // 省略分配undo逻辑
  page_no = undo->last_page_no;

  undo_block = buf_page_get_gen(
      page_id_t(undo->space, page_no), undo->page_size, RW_X_LATCH...);

  do {
    page_t *undo_page;
    ulint offset;
    undo_page = buf_block_get_frame(undo_block);

    // 开始正式在undo log page内写入旧版本记录内容
    switch (op_type) {
      case TRX_UNDO_INSERT_OP:
        offset = trx_undo_page_report_insert(undo_page, trx, index, clust_entry, &mtr);
        break;
      default:
        offset = trx_undo_page_report_modify(undo_page, trx, index, rec, offsets,
                                             update, cmpl_info, clust_entry, &mtr);
    }

    // offset 返回值为0表示失败
    if (UNIV_UNLIKELY(offset == 0)) {
      // 不确定这是在干什么
      if (!trx_undo_erase_page_end(undo_page, &mtr)) { ... }
    } else {
      // 返回值不为0表示成功,此时直接返回即可,在这里不作讨论
      ...
      return (DB_SUCCESS);
    }
    // 走到这里意味着空间不足,我们需要扩充一个新page
    // 然后尝试用这个新page继续写入
    // 调用函数trx_undo_add_page
    mutex_enter(&undo_ptr->rseg->mutex);
    undo_block = trx_undo_add_page(trx, undo, undo_ptr, &mtr);
    mutex_exit(&undo_ptr->rseg->mutex);
    page_no = undo->last_page_no;
  } while (undo_block != NULL);
  ...
}


buf_block_t *trx_undo_add_page(...)
{
  rseg = undo_ptr->rseg;

  // UNDO LOG SEGMENT header page
  header_page = trx_undo_page_get(page_id_t(undo->space, undo->hdr_page_no),
                                  undo->page_size, mtr);

  if (!fsp_reserve_free_extents(&n_reserved, undo->space, 1, FSP_UNDO, mtr)) {
    return (NULL);
  }

  // 分配新的空闲page其no为undo->top_page_no + 1
  new_block = fseg_alloc_free_page_general(
      TRX_UNDO_SEG_HDR + TRX_UNDO_FSEG_HEADER + header_page,
      undo->top_page_no + 1, FSP_UP, TRUE, mtr, mtr);

  fil_space_release_free_extents(undo->space, n_reserved);

  undo->last_page_no = new_block->page.id.page_no();

  new_page = buf_block_get_frame(new_block);

  // 初始化新分配的page
  trx_undo_page_init(new_page, undo->type, mtr);

  // 将新分配的page加入至UNDO SEGMENT PAGE的page list中
  flst_add_last(header_page + TRX_UNDO_SEG_HDR + TRX_UNDO_PAGE_LIST,
                new_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_NODE, mtr);
  undo->size++;
  rseg->curr_size++;

  return (new_block);
}
```