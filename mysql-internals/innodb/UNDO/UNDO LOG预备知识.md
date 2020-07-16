### UNDO LOG预备知识‌

#### 举例说明‌

测试方法：

```sql
--create table and index
create table test (id int primary key, comment char(50)) engine=innodb;
create index test_idx on test(comment);

--Insert
insert into test values(1, ‘aaa’);
insert into test values(2, ‘bbb’);

--更新主键值
update test set id = 9 where id = 1;

--更新二级索引值
update test set comment = ‘ccc’ where id = 9;

--更新二级索引值,但值不变
update test set comment = ‘bbb’ where id = 2 and comment = ‘bbb’;

--read 隔离级别
repeatable read（RR）
```

‌

#### 主键更新



这种一般是低频操作，毕竟谁会没事去改主键的值玩呢?‌

代码调用流程：

```
ha_innobase::update_row ->
    row_update_for_mysql ->
        row_upd_step ->
            row_upd ->
                row_upd_clust_step -> 
                    row_upd_clust_rec_by_insert ->
                        btr_cur_del_mark_set_clust_rec ->
                            row_ins_index_entry
```

‌简单来说，就是将聚簇索引中的的旧记录标记为删除然后插入一条新聚簇索引纪录。该语句执行完之后，数据结构如下：

![img](https://gblobscdn.gitbook.com/assets%2F-LeuGf4juyuq9zjuATOD%2F-MCELbAQVfKiJGpi5DXJ%2F-MCEM4KRiebF-lZG9fjD%2Fimage.png?alt=media&token=a0e73cdf-c87b-493b-a921-99f2a97514ac)



‌老版本依然存储在聚簇索引之中。其DATA_TRX_ID被设置为1811，Deleted bit设置为1，且生成了一条新update undo log record，其中记录前镜像的事务id = 1809。新版本记录的DATA_TRX_ID也为1811，且也生成了一条新的insert undo log record（见上图空白）。‌

通过此图，还可以发现，虽然新老版本是一条记录，但是在聚簇索引中是通过两条记录来标识的。同时， 由于更新了主键，二级索引也需要做相应的更新(二级索引中包含主键项)。

‌

具体函数调用实现：

```c++
dberr_t row_upd_clust_rec_by_insert(...)
{
    switch (node->state) {
    default:
      ...
    case UPD_NODE_UPDATE_CLUSTERED:
      rec = btr_cur_get_rec(btr_cur);
      offsets = rec_get_offsets(rec, index, nullptr, ULINT_UNDEFINED, &heap);
      // 将老记录标记为删除
      err =
          btr_cur_del_mark_set_clust_rec(flags, btr_cur_get_block(btr_cur), rec,
                                         index, offsets, thr, node->row, mtr);
    }
    // 插入新记录
    err = row_ins_clust_index_entry(index, entry, thr, false);
}

dberr_t btr_cur_del_mark_set_clust_rec(...)
{
  // 将老记录写入UNDO LOG
  err =
      trx_undo_report_row_operation(flags, TRX_UNDO_MODIFY_OP, thr, index,
                                    entry, nullptr, 0, rec, offsets, &roll_ptr);

  // 设置记录的delete bit
  btr_rec_set_deleted_flag(rec, page_zip, TRUE);

  // 将undo log记录在老记录的rollptr字段
  row_upd_rec_sys_fields(rec, page_zip, index, offsets, trx, roll_ptr);
}
```

‌*row_ins_clust_index_entry*就不再赘述了。

#### 更新非主键值

上例中执行第二个更新sql：更新 comment 字段（从aaa更新为ccc），代码调用流程与上面有部分不同，可以自

行跟踪，此处省略。更新操作执行完之后，索引结构变更如下：

![img](https://gblobscdn.gitbook.com/assets%2F-LeuGf4juyuq9zjuATOD%2F-MCELbAQVfKiJGpi5DXJ%2F-MCEMGf8RuAR7EdRsjTo%2Fimage.png?alt=media&token=460fa4e3-d287-4623-93e2-1983b934f369)



‌

由于只更新二级索引的键值，因而聚簇索引本身并不会产生新的记录项，而只是将旧版本信息记录在 undo 之中。

与此同时，二级索引将会产生新的索引项，但其 PK 值保持不变，指向聚簇索引的同一条记录（9）。

细心的读者可能会发现，二级索引页面中有一个 MAX_TRX_ID，此值记录的是更新二级索引页面的最大事务 ID。‌

通过 MAX_TRX_ID 的过滤，INNODB 能够实现大部分的二级索引覆盖性扫描(仅仅扫描辅助索引，不需要回聚簇索‌

引)。具体过滤方法，将在后面的内容中给出。‌

#### 更新非主键值（键值不变）

最后一个测试用例，是更新 comment 项为同样的值。在我的测试中，更新之后的索引结构如下：

![img](https://gblobscdn.gitbook.com/assets%2F-LeuGf4juyuq9zjuATOD%2F-MCELbAQVfKiJGpi5DXJ%2F-MCEMUR1OnIVnvcDMowU%2Fimage.png?alt=media&token=55fa23de-87e9-428a-b52a-426dec8993ad)



‌聚簇索引仍旧会更新，但是二级索引保持不变。

‌

#### 总结

1. 无论是聚簇索引，还是二级索引，只要其键值更新，就会产生新版本。将老版本数据deleted bit 设置为 1；同时插入新版本。

2. 对于聚簇索引，如果更新操作没有更新 primary key，那么更新不会产生新版本，而是在 原有版本上进行更新，老版本进入 undo 表空间，通过记录上的 undo 指针进行回滚。

3. 对于二级索引，如果更新操作没有更新其键值，那么二级索引记录保持不变。

4. 对于二级索引，更新操作无论更新 primary key，或者是二级索引键值，都会导致二级索 引产生新版本数据。

5. 聚簇索引设置记录 deleted bit 时，会同时更新 DATA_TRX_ID 列。老版本 DATA_TRX_ID 进入 undo 表空间；二级索引设置 deleted bit 时，不写入 undo。

   ‌

#### UNDO LOG与可见性判断

**主键查找**

```c++
-- case 1
select * from test where id = 1;
```

- 针对测试 1，如果 1811(DATA_TRX_ID) < read_view.up_limit_id，证明被标记为删除的记 录 1 可见。因而无记录返回。
- 针对测试 1，如果 1811(DATA_TRX_ID) >= read_view.low_limit_id，证明被标记为删除的记 录 1 不可见，通过 DATA_ROLL_PTR 回滚记录，得到 DATA_TRX_ID = 1809。如果 1809 可 见，则返回记录(1，aaa)；否则无记录返回。
- 针对测试 1，如果 up_limit_id，low_limit_id 都无法判断可见性，那么遍历 read_view 中 的 trx_ids，依次对比事务 id，如果在 DATA_TRX_ID 在 trx_ids 数组中，则不可见(更新未 提交)。

```
-- case 2
select * from test where id = 9;
```

- 针对测试 2，如果 1816 可见，返回(9，ccc)
- 针对测试 2，如果 1816 不可见，通过 DATA_ROLL_PTR 回滚到 1811，如果 1811 可见， 返回(9, aaa)。
- 针对测试 2，如果 1811 不可见，无结果返回。

```
--case 3
select * from test where id > 0;
```

- 针对测试 1，索引中，满足条件的同一记录，有两个版本(pk=1，delete bit =1和pk = 9)。那么 是否会一条记录返回两次呢？必定不会，这是因为 pk = 1 的可见性与 pk = 9 的可见性是 一致的（因为DATA_TRX_ID都为1811），同时 pk = 1 是标记了 deleted bit 的版本。如果事务 ID = 1811 可见。那么 pk = 1delete 可见，无记录返回，pk = 9 返回记录；如果 1811 不可见，回滚到 1809 可见，那 么 pk = 1 返回记录，pk = 9 回滚后无记录。因而，只会有1条记录被返回

‌

**总结**

1. 通过主键查找记录，需要配合 read_view，记录 DATA_TRX_ID，记录 DATA_ROLL_PTR 指 针共同判断。
2. read_view 用于判断当前记录是否可见(判断 DATA_TRX_ID)。DATA_ROLL_PTR 用于将当前 记录回滚到前一版本。

‌

**非主键查找**

```
select comment from test where comment > '';
```

- 针对测试 2，二级索引，当前页面的最大更新事务 MAX_TRX_ID = 1816。如果 MAX_TRX_ID < read_view.up_limit_id，当前页面所有数据均可见，本页面可以进行索引覆盖性扫描。 丢弃所有 deleted bit = 1 的记录，返回 deleted bit = 0 的记录；此时返回 (ccc)。 (row_select_for_mysql -> lock_sec_rec_cons_read_sees)

- 针对测试 2，二级索引，如果当前页面不能满足 MAX_TRX_ID < read_view.up_limit_id， 说明当前页面无法进行索引覆盖性扫描。此时需要针对每一项，到聚簇索引中判断可见 性。回到测试 2，二级索引中有两项 pk = 9 (一项 deleted bit = 1，另一个为 0)，对应的 聚簇索引中只有一项 pk= 9。如何保证通过二级索引过来的同一记录的多个版本，在聚 簇索引中最多只能被返回一次（什么意思？）？如果当前事务 id 1811 可见。二级索引 pk = 9 的记录(两 项)，通过聚簇索引的 undo，都定位到了同一记录项。此时，innodb 通过以下的一个表 达式，来保证来自二级索引，指向同一聚簇索引记录的多个版本项，有且最多仅有一个 版本将会返回数据：

  ```c++
  if (clust_rec &&
      (old_vers || rec_get_deleted_flag(rec,dict_table_is_comp(sec_index->table))) &&
      !row_sel_sec_rec_is_for_clust_rec(rec, sec_index, clust_rec, clust_index))
  ```

  满足 if 判断的所有聚簇索引记录，都直接丢弃，以上判断的逻辑如下： i. 需要回聚簇索引扫描，并且获得记录 ii. 聚簇索引记录为回滚版本，或者二级索引中的记录为删除版本 iii. 聚簇索引项，与二级索引项，其键值并不相等

  为什么满足 if 判断，就可以直接丢弃数据？用白话来说，就是我们通过二级索引记录，定位聚簇索引记录，定位之后，还需要再次检查聚簇索引记录是否仍旧是我在二级索引中看到的记录。如果不是，则直接丢弃；如果是，则返回。

  根据此条件，结合查询与测试 2 中的索引结构。可见版本为事务 1811。二级索引中的两项 pk = 9 都能通过聚簇索引回滚到 1811 版本。但是，二级索引记录(ccc,9)与聚簇索引回滚后的版本(aaa,9)不一致，直接丢弃。只有二级索引记录(aaa,9)保持一致，直接返回。

  总结：

  1. 二级索引的多版本可见性判断，需要通过聚簇索引协助完成。
  2. 二级索引页面中保存了 MAX_TRX_ID，可以快速判断当前页面中，是否所有项均可见， 可以实现二级索引页面级别的索引覆盖扫描。一般而言，此判断是满足条件的，保证了 索引覆盖扫描 (index only scan)的高效性。
  3. 二级索引中的项，需要与聚簇索引中的可见性进行比较，保证聚簇索引中的可见项，与 二级索引中的项数据一致。