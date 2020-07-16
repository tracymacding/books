#### UNDO LOG与奔溃恢复

当数据库因种种原因奔溃后，在重启时需要对系统奔溃时那些未竟事务进行处理：要么回滚，要么提交。保证在重

‌启完成后系统处于一个干净的状态。而未竟事务信息的构造依赖UNDO LOG，因为纵观MYSQL，事务信息的持久

‌化是通过UNDO LOG的写入和更新来进行的。

因而，在数据库系统启动时，需要：‌

- 收集UNDO LOG并构建事务信息
- 对未竟事务进行处理：回滚或者提交

#### 扫描回滚段并加载undo log

```c++
void trx_rsegs_init(purge_pq_t *purge_queue)
{
  // 处理系统表空间的回滚段
  for (slot = 0; slot < TRX_SYS_N_RSEGS; slot++) {
   ...
  }
  // 处理独立undo表空间的回滚段
  for (auto undo_space : undo::spaces->m_spaces) {
    for (slot = 0; slot < FSP_MAX_ROLLBACK_SEGMENTS; slot++) {
      page_no = trx_rseg_get_page_no(undo_space->id(), slot);
      if (page_no == FIL_NULL) {
        break;
      }
      rseg = trx_rseg_mem_create(slot, undo_space->id(), page_no,
                                 univ_page_size, purge_queue, &mtr);
      undo_space->rsegs()->push_back(rseg);
    }
  }
}

trx_rseg_t *trx_rseg_mem_create(...)
{
  rseg = static_cast<trx_rseg_t *>(ut_zalloc_nokey(sizeof(trx_rseg_t)));
  ...
  UT_LIST_INIT(rseg->update_undo_list, &trx_undo_t::undo_list);
  UT_LIST_INIT(rseg->update_undo_cached, &trx_undo_t::undo_list);
  UT_LIST_INIT(rseg->insert_undo_list, &trx_undo_t::undo_list);
  UT_LIST_INIT(rseg->insert_undo_cached, &trx_undo_t::undo_list);

  // 读取回滚段header page
  rseg_header = trx_rsegf_get_new(space_id, page_no, page_size, mtr);

  // 关键: 读取回滚段内的所有undo log并加入至insert_undo_list/update_undo_list
  sum_of_undo_sizes = trx_undo_lists_init(rseg);
  ...
  return (rseg);
}

ulint trx_undo_lists_init(trx_rseg_t *rseg)
{
  rseg_header = trx_rsegf_get_new(rseg->space_id, rseg->page_no, rseg->page_size, &mtr);
  // 遍历回滚段header page内的每个undo log slot
  // 取出其中的undo log page no,进而恢复出undo log信息
  for (i = 0; i < TRX_RSEG_N_SLOTS; i++) {
    page_no = trx_rsegf_get_nth_undo(rseg_header, i, &mtr);
    if (page_no != FIL_NULL &&
        srv_force_recovery < SRV_FORCE_NO_UNDO_LOG_SCAN)
    {
      trx_undo_t undo = trx_undo_mem_create_at_db_start(rseg, i, page_no, &mtr);
      ...
    }
  }
}

trx_undo_t *trx_undo_mem_create_at_db_start(...)
{
  // 读取undo log的segment header
  // 根据header便可以恢复出undo log的很多元信息
  // 不会读取undo log里面的每一条记录,因为没有必要
  undo_page = trx_undo_page_get(page_id_t(rseg->space_id, page_no),
                                rseg->page_size, mtr);

  page_header = undo_page + TRX_UNDO_PAGE_HDR;

  type = mtr_read_ulint(page_header + TRX_UNDO_PAGE_TYPE, MLOG_2BYTES, mtr);
  seg_header = undo_page + TRX_UNDO_SEG_HDR;

  // undo log代表的事务的状态
  state = mach_read_from_2(seg_header + TRX_UNDO_STATE);

  // undo log代表的事务的id
  trx_id = mach_read_from_8(undo_header + TRX_UNDO_TRX_ID);

  // 读取xid信息
  ...
  undo = trx_undo_mem_create(rseg, id, type, trx_id, &xid, page_no, offset);
  ...
  // 将构建的trx_undo_t对象插入至回滚段的insert_undo_list/update_undo_list
add_to_list:
  ...
}
```

‌至此，便在启动时收集好了所有的undo log，接下来便就是要根据undo log构建活跃事务。‌

#### 收集活跃事务

```c++
void trx_lists_init_at_db_start(void) 
{
  // 扫描系统表空间内的回滚段
  for (auto rseg : trx_sys->rsegs) {
    trx_resurrect(rseg);
  }

  // 扫描独立表空间的回滚段
  for (auto undo_space : undo::spaces->m_spaces) {
    for (auto rseg : *undo_space->rsegs()) {
      trx_resurrect(rseg);
    }
  }
}

// 根据回滚段内的UNDO LOG构造活跃事务加入至全局事务hash表
void trx_resurrect(trx_rseg_t *rseg)
{
  for (undo = UT_LIST_GET_FIRST(rseg->insert_undo_list); undo != NULL;
       undo = UT_LIST_GET_NEXT(undo_list, undo)) {
    ...
    // 根据undo log中的记录构造trx_t对象
    trx = trx_resurrect_insert(undo, rseg);
    trx_sys->rw_trx_hash.insert(trx);
    trx_sys->rw_trx_hash.put_pins(trx);

    trx_resurrect_table_ids(trx, &trx->rsegs.m_redo, undo);
  }

  for (undo = UT_LIST_GET_FIRST(rseg->update_undo_list); undo != NULL;
       undo = UT_LIST_GET_NEXT(undo_list, undo))
  {
    // 有可能在前面的insert undo log的遍历中已经插入了
    // 所以这里先查找下
    trx = trx_sys->rw_trx_hash.find(0, undo->trx_id, false);
    // 根据undo log中的记录构造trx_t对象并插入至trx hash中
    trx_resurrect_update(trx, undo, rseg);
    trx_sys->rw_trx_hash.insert(trx);
    trx_sys->rw_trx_hash.put_pins(trx);
  }
}
```

‌#### 活跃事务处理

```c++
// 收集活跃事务和已提交事务,保存在链表中返回
// 忽略PREPARED状态事务
bool trx_rollback_recovered_callback(...) {
  trx_t* trx = element->trx;
  if (trx) {
    if (trx->is_recovered &&
        (trx_state_eq(trx, TRX_STATE_ACTIVE) ||
         trx_state_eq(trx, TRX_STATE_COMMITTED_IN_MEMORY)))
    {
      UT_LIST_ADD_FIRST(*trx_list, trx);
    }
  }
  return(false);
}

void trx_rollback_or_clean_recovered(ibool all) 
{
  // 遍历rw_trx_hash,收集非PREPARED状态事务
  trx_sys->rw_trx_hash.iterate_no_dups(reinterpret_cast<hash_walk_action>
      (trx_rollback_recovered_callback),
      &trx_list);

  // 根据事务状态作回滚或者提交处理
  while (trx_t *trx= UT_LIST_GET_FIRST(trx_list)) {
    UT_LIST_REMOVE(trx_list, trx);
    trx_rollback_resurrected(trx, all);
  }
}

ibool trx_rollback_resurrected(trx_t *trx, ibool all)
{
  switch (state) {
    // 已提交事务清理其残余
    case TRX_STATE_COMMITTED_IN_MEMORY:
      trx_cleanup_at_db_startup(trx);
      trx_free_resurrected(trx);
      return (TRUE);
    // 活跃的未提交事务做回滚处理
    // 如何回滚事务就不在这里详细描述
    case TRX_STATE_ACTIVE:
      if (all || trx->ddl_operation) {
        trx_rollback_active(trx);
        return (TRUE);
      }
      return (FALSE);
    ...
  }
}
```

‌关于活跃事务的回滚等实现我们会在以后的事务相关章节中作详细描述，关于如何处理PREPARED状态的事务我们不在这里描述，而是留到binlog实现章节。