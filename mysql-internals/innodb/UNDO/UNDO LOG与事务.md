### UNDO LOG与事务

#### 事务prepare

入口函数：*trx_prepare_low*

当事务完成需要提交时，为了和BINLOG做XA，MYSQL中的事务执行被划分成了PREPARE和COMMIT两个阶段：prepare阶段和commit阶段，本小节主要讨论下prepare阶段undo相关的逻辑。

为了在崩溃重启时知道事务状态，需要将事务设置为Prepare，MySQL 5.7对临时表undo和普通表undo分别做了处理，前者在写undo日志时总是不需要记录redo，后者则需要记录。

分别设置insert undo 和 update undo的状态为prepare，调用函数*trx_undo_set_state_at_prepare*，过程也比较简单，找到undo page的UNDO_SEG_HDR，将其中的**TRX_UNDO_STATE**设置为**TRX_UNDO_PREPARED**，同时修改其他对应字段（主要是XID）。

```c++
lsn_t trx_prepare_low(...) 
{
    ...
    if (undo_ptr->insert_undo != NULL) {
      trx_undo_set_state_at_prepare(trx, undo_ptr->insert_undo, false, &mtr);
    }

    if (undo_ptr->update_undo != NULL) {
      trx_undo_set_state_at_prepare(trx, undo_ptr->update_undo, false, &mtr);
    }
    ...
}

page_t *trx_undo_set_state_at_prepare(...)
{
  undo_page = trx_undo_page_get(page_id_t(undo->space, undo->hdr_page_no),
                                undo->page_size, mtr);

  seg_hdr = undo_page + TRX_UNDO_SEG_HDR;
  // 设置undo内存结构状态为TRX_UNDO_PREPARED
  undo->state = TRX_UNDO_PREPARED;
  undo->xid = *trx->xid;

  // 将undo log状态写入undo page的UNDO_SEG_HDR中的TRX_UNDO_STATE字段
  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, undo->state, MLOG_2BYTES, mtr);

  // 接下来将xid写入undo log header中的XID部分
  offset = mach_read_from_2(seg_hdr + TRX_UNDO_LAST_LOG);
  undo_header = undo_page + offset;

  mlog_write_ulint(undo_header + TRX_UNDO_XID_EXISTS, TRUE, MLOG_1BYTE, mtr);

  trx_undo_write_xid(undo_header, &undo->xid, mtr);

  return (undo_page);
}
```



说明：

InnoDB层的XID是如何获取的呢？ 当Innodb的参数innodb_support_xa打开时，在执行事务的第一条SQL时，就会去注册XA，根据第一条SQL的query id拼凑XID数据，然后存储在事务对象中。参考函数*trans_register_ha*。

#### 事务Commit

当事务commit时，需要将事务状态设置为COMMIT状态，这里同样是更新undo log的状态。

入口函数：*trx_commit_low --> trx_write_serialisation_history*

```c++
void trx_commit_low(...)
{
  ...
  if (mtr != NULL) {
    mtr->set_sync();

    serialised = trx_write_serialisation_history(trx, mtr);
    ...
  }
}

bool trx_write_serialisation_history(...)
{
  ...
  if (trx->rsegs.m_redo.insert_undo != NULL) {
    trx_undo_set_state_at_finish(trx->rsegs.m_redo.insert_undo, mtr);
  }

  if (trx->rsegs.m_noredo.insert_undo != NULL) {
    trx_undo_set_state_at_finish(trx->rsegs.m_noredo.insert_undo, &temp_mtr);
  }
  ...
}

page_t *trx_undo_set_state_at_finish(trx_undo_t *undo, mtr_t *mtr)   
{

  undo_page = trx_undo_page_get(page_id_t(undo->space, undo->hdr_page_no),
                                undo->page_size, mtr);

  seg_hdr = undo_page + TRX_UNDO_SEG_HDR;
  page_hdr = undo_page + TRX_UNDO_PAGE_HDR;
  
  if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) <
                             TRX_UNDO_PAGE_REUSE_LIMIT) {
    // 此时将该UNDO PAGE加入CACHED链表准备复用
    state = TRX_UNDO_CACHED;
  } else if (undo->type == TRX_UNDO_INSERT) {
    // INSERT UNDO可以直接FREE
    state = TRX_UNDO_TO_FREE;
  } else {
    // UPDATE UNDO需要交由专门的purge线程来回收
    state = TRX_UNDO_TO_PURGE;
  }

  undo->state = state;
 	// 更新undo log的事务状态字段
  mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, state, MLOG_2BYTES, mtr);
  return (undo_page);
}
```

‌在该函数中，需要将该事务包含的Undo都设置为完成状态，先设置insert undo，再设置update undo（调用函数*trx_undo_set_state_at_finish*），完成状态包含三种：

> - 如果当前的undo log只占一个page，且占用的header page大小使用不足其3/4时(TRX_UNDO_PAGE_REUSE_LIMIT)，则状态设置为**TRX_UNDO_CACHED**，该undo对象会随后加入到undo cache list上；
> - 如果是insert undo（undo类型为TRX_UNDO_INSERT），则状态设置为**TRX_UNDO_TO_FREE**；
> - 如果不满足a和b，则表明该undo可能需要Purge线程去执行清理操作，状态设置为**TRX_UNDO_TO_PURGE**。

‌在确认状态信息后，写入undo header page的TRX_UNDO_STATE中。‌

#### 事务回滚

如果事务因为异常或者被显式的回滚了，那么所有数据变更都要恢复修改前值。这里就要借助UNDO LOG中的数据来进行恢复了。

入口函数为：`row_undo_step --> row_undo`‌

操作也比较简单，获取老版本记录，做逆向操作即可：对于标记删除的记录清理标记删除标记；对于in-place更新，将数据回滚到最老版本；对于插入操作，直接删除聚集索引和二级索引记录（*row_undo_ins*）。

具体的操作中，先回滚二级索引记录（*row_undo_mod_del_mark_sec*、*row_undo_mod_upd_exist_sec*、*row_undo_mod_upd_del_sec*），再回滚聚集索引记录（*row_undo_mod_clust*）。这里不展开描述，可以参阅对应的函数。