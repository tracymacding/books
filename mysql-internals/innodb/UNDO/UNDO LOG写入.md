### UNDO LOG写入

我们在之前的章节中提到，UNDO LOG分为INSERT和UPDATE两种类型。接下来我们分别描述这两种记录如何写入UNDO LOG RECORD。

行记录在插入或者更新时，除写入索引记录外，还会同时写入UNDO LOG。对于新插入的行记录，会将新插入的行记录写入UNDO LOG，而对于被更新的行记录，则是将更新前的记录旧值写入UNDO LOG，并在新记录的隐藏字段rollptr中记录该UNDO LOG位置信息。

接下来我们分别描述INSERT和UPDATE时的UNDO LOG写入实现。

#### INSERT记录

新插入的记录会写到undo log中，在函数*btr_cur_ins_lock_and_undo*中实现：

```c++
db_err_t btr_cur_ins_lock_and_undo(...)
{
  rec = btr_cur_get_rec(cursor);
  index = cursor->index;
  ...
  err = trx_undo_report_row_operation(flags, TRX_UNDO_INSERT_OP, thr, index,
                                      entry, NULL, 0, NULL, NULL, &roll_ptr);

  row_upd_index_entry_sys_field(entry, index, DATA_ROLL_PTR, roll_ptr);
  return (DB_SUCCESS);
}

// 最终走到这里
ulint trx_undo_page_report_insert(...)
{
  // 找到该undo page下一个空闲位置
  first_free = mach_read_from_2(undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_FREE);
  ptr = undo_page + first_free;

  // 预留的2字节用于记录下一个undo log record位置
  ptr += 2;

  *ptr++ = TRX_UNDO_INSERT_REC;
  ptr += mach_u64_write_much_compressed(ptr, trx->undo_no);
  ptr += mach_u64_write_much_compressed(ptr, index->table->id);

  // index->n_uniq代表主键中包含的column数量
  // 如果是联合主键,则>1
  for (i = 0; i < dict_index_get_n_unique(index); i++) {
    const dfield_t *field = dtuple_get_nth_field(clust_entry, i);
    ulint flen = dfield_get_len(field);
    ptr += mach_write_compressed(ptr, flen);
    ut_memcpy(ptr, dfield_get_data(field), flen);
  }
  return (trx_undo_page_set_next_prev_and_add(undo_page, ptr, mtr));
}
```

‌对于insert 操作，会将新记录写入至undo log。写入的只是聚簇索引包含的column。例如，创建的表结构如下：

```sql
create table t(id1 int, id2 int, value int, primary key (id1, id2));
insert into t values(1, 2, 3);
```

‌此时写入内容其实只包含列id1和id2的内容。完成后，会生成一个rollptr，且记录在dtuple_t的rollptr列中。

#### UPDATE记录

更新一个已存在记录时，会将该记录的老版本（即修改前版本）记录在UNDO LOG。与INSERT操作类似：只记录其聚簇索引中包含的column的值以及更新涉及的column值。记完UNDO LOG后，再将数据页中的记录更新为新值，同时将UNDO LOG位置记录在更新后行记录的rollptr字段，形成一个历史更新链。

```c++
dberr_t
btr_cur_upd_lock_and_undo(...)
{
    // rec指向了待更新记录的内容(更新前value)
    rec = btr_cur_get_rec(cursor);
    index = cursor->index;

    // 更新非聚簇索引不记录undo log
    if (!dict_index_is_clust(index)) {
        return(lock_sec_rec_modify_check_and_lock(
                   flags, btr_cur_get_block(cursor), rec,
                   index, thr, mtr));
    }
    // 记录undo log
    return(trx_undo_report_row_operation(
               mtr, flags, TRX_UNDO_MODIFY_OP, thr,
               index, NULL, update,
               cmpl_info, rec, offsets, roll_ptr));
}

dberr_t trx_undo_report_row_operation(...)
{
  do {
    undo_page = buf_block_get_frame(undo_block);
    switch (op_type) {
    default:
      offset = trx_undo_page_report_modify(
                undo_page, trx, index, rec, offsets, update,
                cmpl_info, &mtr);
    }
  }
}

ulint
trx_undo_page_report_modify(...)
{
    table = index->table;
    first_free = mach_read_from_2(undo_page + TRX_UNDO_PAGE_HDR + TRX_UNDO_PAGE_FREE);
    ptr = undo_page + first_free;
    ptr += 2;

    // 需要注意这玩意儿：如果update为null时,undo log record中的type会被设置为
    // TRX_UNDO_DEL_MARK_REC
    // 那什么时候update会为null呢?
    // 跟踪了一下发现可能有以下两种场景:
    // 1. btr_cur_ins_lock_and_undo:即插入一条新记录
    // 2. btr_cur_del_mark_set_clust_rec: 从聚簇索引中删除一条老记录
    // 而2可能会发生在两种场景下:
    // 1. 更新一个已有主键值的主键:此时会先插入新的主键，再将老的主键值标记删除
    // 2. 删除一个已有主键值
    if (!update) {
        // 对已有记录标记删除
        type_cmpl = TRX_UNDO_DEL_MARK_REC;
    } else if (rec_get_deleted_flag(rec, dict_table_is_comp(table))) {
        // 更新删除记录,什么时候会这样呢?
        type_cmpl = TRX_UNDO_UPD_DEL_REC;
    } else {
        // 更新一个已存在的记录
        type_cmpl = TRX_UNDO_UPD_EXIST_REC;
    }

    type_cmpl |= cmpl_info * TRX_UNDO_CMPL_INFO_MULT;
    type_cmpl_ptr = ptr;

    *ptr++ = (byte) type_cmpl;
    ptr += mach_ull_write_much_compressed(ptr, trx->undo_no);

    ptr += mach_ull_write_much_compressed(ptr, table->id);
    // 不确定info_bits内到底存储什么玩意
    *ptr++ = (byte) rec_get_info_bits(rec, dict_table_is_comp(table));

    // 获得老记录的trx id和roll ptr值
    // 并将其记录在该UNDO LOG的rollptr和trx_id字段
    // 这样才可以将所有的更新串成一个更新链
    // ----------        ----------        ----------        -----------
    // | UNDO-1 |  <--   | UNDO-2 |  <--   | UNDO-3 |  <--   | row rec |
    // ----------        ----------        ----------        -----------
    // 假如现在来了一次更新记录在UNDO-4中，那形成的历史链应该如下：
    // ----------        ----------        ----------        ----------       ----------- 
    // | UNDO-1 |  <--   | UNDO-2 |  <--   | UNDO-3 |  <--   | UNDO-4 |  <--  | row rec |
    // ----------        ----------        ----------        ----------       -----------
    // 更新前的row rec中记录的rollptr指向UNDO-3
    // 因此UNDO-4中记录的内容是row rec，且其rollptr指向UNDO-3
    field = rec_get_nth_field(rec, offsets, dict_index_get_sys_col_pos(index, DATA_TRX_ID), &flen);

    trx_id = trx_read_trx_id(field);
    ptr += mach_ull_write_compressed(ptr, trx_id);
    field = rec_get_nth_field(rec, offsets,
                  dict_index_get_sys_col_pos(
                      index, DATA_ROLL_PTR), &flen);
    ptr += mach_ull_write_compressed(ptr, trx_read_roll_ptr(field));

    // 记录聚簇索引包含的column的旧值,如果是联合索引,那么可能会包含多个column
    for (i = 0; i < dict_index_get_n_unique(index); i++) {
        field = rec_get_nth_field(rec, offsets, i, &flen);
        ptr += mach_write_compressed(ptr, flen);
        if (flen != UNIV_SQL_NULL) {
            ut_memcpy(ptr, field, flen);
            ptr += flen;
        }
    }

    // 接下来记录被更新column的旧值
    // 注意：只需要记录旧值即可,更新后的值无需记录,因为无论的回滚还是MVCC,都只需要旧值即可
    // 每个column记录三个字段: 
    // 1. column的field no
    // 2. column field的长度
    // 3. column field的值
    if (update) {
        ptr += mach_write_compressed(ptr, upd_get_n_fields(update));
        for (i = 0; i < upd_get_n_fields(update); i++) {
            // pos保存field no
            ulint pos = upd_get_nth_field(update, i)->field_no;
            ptr += mach_write_compressed(ptr, pos);
            // field保存被更新column的旧值, flen记录其长度
            field = rec_get_nth_field(rec, offsets, pos, &flen);
            ptr += mach_write_compressed(ptr, flen);
            if (flen != UNIV_SQL_NULL) {
                ut_memcpy(ptr, field, flen);
                ptr += flen;
            }
        }
    }
}
```

#### 删除记录

在innodb中，删除一个行记录实现上是标记删除，即不立即从物理页面中删除该记录(因为该记录很有可能还被其他事务所访问)，只是将该行记录标记为删除，并记录UNDO LOG，以后在回收UNDO LOG时判断该行记录不再被访问时再清理该行记录。

```c++
dberr_t btr_cur_del_mark_set_clust_rec(...)
{
  ...
  // 为标记删除记录undo log
  // 注意倒数第五个参数为nullptr,代表的是删除
  err =
      trx_undo_report_row_operation(flags, TRX_UNDO_MODIFY_OP, thr, index,
                                    entry, nullptr, 0, rec, offsets, &roll_ptr);

  // 更新记录的trx_id和rollptr列
  row_upd_rec_sys_fields(rec, page_zip, index, offsets, trx, roll_ptr);

  return (err);
}

// 在记录删除时传入的update为nullptr
ulint
trx_undo_page_report_modify(ulint flags,                
    ulint op_type,              
    que_thr_t *thr,
    dict_index_t *index,
    const dtuple_t *clust_entry,
    const upd_t *update,
    const rec_t *rec,
    const ulint *offsets,
    roll_ptr_t *roll_ptr)
{
    // 需要注意这玩意儿：如果update为null时,undo log record中的type会被设置为
    // TRX_UNDO_DEL_MARK_REC
    // 那什么时候update会为null呢?
    // 跟踪了一下发现可能有以下两种场景:
    // 1. btr_cur_ins_lock_and_undo:即插入一条新记录
    // 2. btr_cur_del_mark_set_clust_rec: 从聚簇索引中删除一条老记录
    // 不过对于1是insert场景,最终会走trx_undo_page_report_insert()
  	// 而2可能会发生在两种场景下:
    // 1. 更新一个已有主键值的主键:此时会先插入新的主键，再将老的主键值标记删除
    // 2. 删除一个已有主键值
    if (!update) {
        // 对已有记录标记删除
        type_cmpl = TRX_UNDO_DEL_MARK_REC;
    } else if (rec_get_deleted_flag(rec, dict_table_is_comp(table))) {
        // 更新删除记录,什么时候会这样呢?
        type_cmpl = TRX_UNDO_UPD_DEL_REC;
    } else {
        // 更新一个已存在的记录
        type_cmpl = TRX_UNDO_UPD_EXIST_REC;
    }
  	...
    // 获得老记录的trx id和roll ptr值
    // 并将其记录在该UNDO LOG的rollptr和trx_id字段
  	// 这个与上面的update记录流程一致,不再重复介绍

    // 记录聚簇索引包含的column的旧值,如果是联合索引,那么可能会包含多个column
   	// 这个也与上面的update记录流程一致,不再重复介绍
  	...
    // 如果是删除,那要将所有的column都记录在undo log中
    if (!update || !(cmpl_info & UPD_NODE_NO_ORD_CHANGE)) {
        ...
        trx->update_undo->del_marks = TRUE;
        ptr += 2;
        for (col_no = 0; col_no < dict_table_get_n_cols(table); col_no++)
        {
          ...
        }
        mach_write_to_2(old_ptr, ptr - old_ptr);
    }
}
```

删除记录的内部实现其实也比较简单：

1. 将行记录设置标记删除
2. 记录UNDO LOG RECORD，需要搞清楚里面到底记录了哪些内容
3. 最后，更新原纪录的系统列：trx_id和rollptr

