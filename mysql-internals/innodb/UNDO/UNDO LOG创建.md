### UNDO LOG创建

在本文章中我们主要研究MYSQL在initialize如何创建与UNDO LOG相关的文件和对象，同时在启动时如何从文件中加载这些对象。‌

我们知道，UNDO表空间主要管辖回滚段（Rollback Segment），回滚段内再组织成为UNDO LOG、UNDO PAGE等。因而，UNDO表空间的创建主要便是构建好回滚段。‌

#### install时创建回滚段

**创建第一个回滚段**

```c++
dberr_t srv_start(bool create_new_db, ...) {
  ...
  if (create_new_db) {
    ...
    err = srv_undo_tablespaces_init(true);
    trx_sys_create_sys_pages();
    ...
  }
}

// 创建系统表空间页面
void trx_sys_create_sys_pages()
{
  trx_sysf_create(&mtr);
}

static void trx_sysf_create(mtr_t *mtr)
{
  block = fseg_create(TRX_SYS_SPACE, 0, TRX_SYS + TRX_SYS_FSEG_HEADER, mtr);

  page = buf_block_get_frame(block);

  // 获取trx sys page，即space 0的第5个page
  // 并在其中写入初始TRX_ID, 1
  sys_header = trx_sysf_get(mtr);

  mach_write_to_8(sys_header + TRX_SYS_TRX_ID_STORE, 1);

  // 开始初始化系统表空间回滚段对应的slot,每个slot占据8个字节
  ptr = TRX_SYS_RSEGS + sys_header;
  len = ut_max(TRX_SYS_OLD_N_RSEGS, TRX_SYS_N_RSEGS) * TRX_SYS_RSEG_SLOT_SIZE;
  memset(ptr, 0xff, len);
  ptr += len;

  memset(ptr, 0, UNIV_PAGE_SIZE - FIL_PAGE_DATA_END + page - ptr);

  // 为系统表空间（space id为0）创建第一个回滚段
  // 不知道系统表空间到底存储什么玩意儿?
  slot_no = trx_sysf_rseg_find_free(mtr);
  page_no = trx_rseg_header_create(TRX_SYS_SPACE, univ_page_size, PAGE_NO_MAX,
                                   slot_no, mtr);
}
```

‌**为独立UNDO表空间创建回滚段**

根据代码描述，为了兼容性考虑，第一个回滚段在创建trx sys page时一并创建并存放于系统表空间。接下来在完成double write初始化后会继续创建剩余的回滚段，分别为临时表空间和undo表空间内创建。

需要说明的是，创建新的回滚段不仅仅在install db时存在，也有可能发生在数据库重启时，因为我们有可能会调整回滚段数量这个参数，从而导致在节点启动时创建额外的回滚段。

INNODB中包含三种类型的表空间：

- 系统表空间
- 临时表空间
- UNDO独立表空间(如果开启独立表空间的话)

```c++
bool trx_rseg_adjust_rollback_segments(...)
{
  // 为临时表空间创建回滚段,回滚段数量受参数target_rollback_segments控制,默认为128个
  // 创建的临时表空间回滚段保存在trx_sys->tmp_rsegs
  trx_rseg_add_rollback_segments(srv_tmp_space.space_id(),
                                 target_undo_tablespaces,
                                 target_rollback_segments,
                                 &(trx_sys->tmp_rsegs));

  // 如果开启undo独立表空间
  // 为每一个undo表空间创建回滚段,数量由target_rollback_segments控制,默认为128
  // 创建的回滚段保存在undo space的回滚段成员中
  if (target_undo_tablespaces > 0) {
    for (auto undo_space : undo::spaces->m_spaces) {
      trx_rseg_add_rollback_segments(undo_space->id(),
                                     target_undo_tablespaces,
                                     target_rollback_segments,
                                     undo_space->rsegs());
    }
  }

  // 如果使用共享表空间,那么为系统表空间(space id = 0)内创建回滚段
  // 数量默认也为128,创建的的结果也保存在trx_sys->rsegs中
  if (target_undo_tablespaces == 0) {
    trx_rseg_add_rollback_segments(TRX_SYS_SPACE, target_undo_tablespaces,
                                   target_rollback_segments,
                                   &(trx_sys->rsegs));
  }
}

// 为表空间space_id创建数量为target_rsegs的回滚段,并保存在数组rsegs中返回
bool trx_rseg_add_rollback_segments(space_id_t space_id, ulong target_spaces,
                                    ulong target_rsegs, Rsegs *rsegs)
{
  for (ulint num = 0; num < FSP_MAX_ROLLBACK_SEGMENTS; num++) {
    ulint rseg_id = num;
    // 如果该回滚段已经存在,有可能发生在数据库重新并调整了回滚段数量参数
    // 导致需要在已有回滚段基础上创建新的回滚段
    rseg = rsegs->find(rseg_id);
    if (rseg != nullptr) {
      continue;
    }

    if (type == UNDO) {
      page_no = trx_rseg_get_page_no(space_id, rseg_id);
    } else if (type == SYSTEM) {
      page_no = trx_sysf_rseg_find_page_no(rseg_id);
    } else {
      page_no = FIL_NULL;
    }

    // 如果回滚段的header page不存在,那么为其创建并初始化header page
    // 一旦header page创建好,也就意味着该回滚段创建完成
    if (page_no == FIL_NULL) {
        page_no = trx_rseg_create(space_id, rseg_id);
      } else {
        break;
      }
    }

    // 创建回滚段内存结构
    rseg = trx_rseg_mem_create(rseg_id, space_id, page_no, univ_page_size,
                               purge_sys->purge_queue, &mtr);

    if (rseg != nullptr) {
      rsegs->push_back(rseg);
    }
  }
}

// 为表空间space_id创建回滚段,回滚段id为rseg_id
page_no_t trx_rseg_create(space_id_t space_id, ulint rseg_id) {
  // 为回滚段分配header page
  page_no_t page_no = trx_rseg_header_create(space_id, univ_page_size,
                                             PAGE_NO_MAX, rseg_id, &mtr);
  return (page_no);
}

// 为表空间space_id创建并初始化回滚段(rseg_slot)header page
page_no_t trx_rseg_header_create(space_id_t space_id,
                                 const page_size_t &page_size,
                                 page_no_t max_size, ulint rseg_slot,
                                 mtr_t *mtr)
{
  // 分配回滚段空间
  block = fseg_create(space_id, 0, TRX_RSEG + TRX_RSEG_FSEG_HEADER, mtr);

  page_no = block->page.id.page_no();

  // 获得该回滚段header page内page data指针
  rsegf = trx_rsegf_get_new(space_id, page_no, page_size, mtr);

  // 初始化header page的TRX_RSEG_MAX_SIZE字段
  mlog_write_ulint(rsegf + TRX_RSEG_MAX_SIZE, max_size, MLOG_4BYTES, mtr);

  // 初始化header page的TRX_RSEG_HISTORY_SIZE和TRX_RSEG_HISTORY字段
  // 该字段用来存储回滚段上已经提交事务产生的UNDO LOG
  mlog_write_ulint(rsegf + TRX_RSEG_HISTORY_SIZE, 0, MLOG_4BYTES, mtr);
  flst_init(rsegf + TRX_RSEG_HISTORY, mtr);

  // 初始化header page内的undo log slot,全部设置为FIL_NULL
  // 这些slot内存储的是page no
  for (i = 0; i < TRX_RSEG_N_SLOTS; i++) {
    trx_rsegf_set_nth_undo(rsegf, i, FIL_NULL, mtr);
  }

  // 接下来将创建的回滚段header page信息记录在表空间的特定page的特定字段
  if (space_id == TRX_SYS_SPACE) {
    sys_header = trx_sysf_get(mtr);
    trx_sysf_rseg_set_space(sys_header, rseg_slot, space_id, mtr);
    trx_sysf_rseg_set_page_no(sys_header, rseg_slot, page_no, mtr);
  } else if (fsp_is_system_temporary(space_id)) {
  } else {
    // 对于独立表空间,将回滚段的header page信息记录在表空间的第FSP_RSEG_ARRAY_PAGE_NO(3)
    // 个page内,具体是记录在该page的偏移RSEG_ARRAY_PAGES_OFFSET处
    // 每个回滚段只需要记录header page no即可,占据4B, 因此然后根据回滚段id计算其在page内的offset
    rsegs_header = trx_rsegsf_get(space_id, mtr);
    trx_rsegsf_set_page_no(rsegs_header, rseg_slot, page_no, mtr);
  }
  return (page_no);
}
```

‌创建回滚段的主要动作是创建并初始化回滚段的header page。而主要是为header page初始化undo log slot数组为0，表示该slot尚未分配。以后从回滚段分配undo log时，便是寻找header page中第一个为0的slot。

‌如果使用独立的undo table space，可很容易计算最大支持的并发的事务数可以为8 * 128 * 1024 = 1048576（假设undo table space数量为8，128为回滚段数量，1024为每个回滚段内undo log slot数量）。以后我们在说undo log 分配时会涉及到该计算。

#### mysql启动时加载回滚段加载

```c++
void trx_rsegs_init(purge_pq_t *purge_queue)
{
  for (slot = 0; slot < TRX_SYS_N_RSEGS; slot++) {
    // 从系统表空间中获取trx sys page,也即系统表空间TRX_SYS_SPACE(0)的
    // 第TRX_SYS_PAGE_NO(5)个page
    trx_sysf_t *sys_header = trx_sysf_get(&mtr);

    // 从上面的trx sys page的header中找到第slot个回滚段所在的page no
    page_no = trx_sysf_rseg_get_page_no(sys_header, slot, &mtr);
  
    if (page_no != FIL_NULL) {
      // 从上面的trx sys page的header中找到第slot个回滚段所位于的space id
      space_id = trx_sysf_rseg_get_space(sys_header, slot, &mtr);
      // 根据上面的信息创建内存中的回滚段对象并将其放入全局trx_sys的rsegs
      rseg = trx_rseg_mem_create(slot, space_id, page_no, univ_page_size,
                                 purge_queue, &mtr);
      // 系统表空间的回滚段
      trx_sys->rsegs.push_back(rseg);
    }
  }

  // 如果使用独立UNDO表空间,那需要加载每个表空间的每个回滚段信息
  // 并构建内存结构
  for (auto undo_space : undo::spaces->m_spaces) {
    for (slot = 0; slot < FSP_MAX_ROLLBACK_SEGMENTS; slot++) {
      // 首先找到表空间的第FSP_RSEG_ARRAY_PAGE_NO个page,其中记录每个回滚段的header page no
      // 进而再找到每个回滚段其他信息
      page_no = trx_rseg_get_page_no(undo_space->id(), slot);
      if (page_no == FIL_NULL) {
        break;
      }
      // 独立表空间的回滚段
      rseg = trx_rseg_mem_create(slot, undo_space->id(), page_no,
                                 univ_page_size, purge_queue, &mtr);
      undo_space->rsegs()->push_back(rseg);
    }
  }
}
```

根据上面初始化流程可以发现：

1. 系统表空间(SPACE ID为0)的第5个page存储的是trx sys页面，其中持久化了事务相关信息字段，后面我们会详细描述该page的具体结构
2. 系统中共存在128个回滚段，每个回滚段在trx sys page中对应一个slot，记载该回滚段所在位置信息（space和page no），每个slot占据8个字节，因此，128个回滚段占据了1024字节；