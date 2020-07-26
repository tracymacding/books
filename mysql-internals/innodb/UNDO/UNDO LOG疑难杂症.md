### UNDO LOG疑难杂症

#### Innodb history list无法下降到0

熟悉InnoDB的朋友都知道，innodb的history list长度代表了有多少undo日志还没有被清理掉，可以通过`show engine innodb status` 命令来获得。如果发现history list的长度越大，要么就是实例的复杂非常高，要么就是可能有大查询，或者事务没提交，导致Undo log无法分析。

但如果仔细观察，大家是否发现，history list居然无法降到0，即使做一次slow shutdown也不行。因为理论上来说，如果undo日志都已经purge干净了，理论上应该能下降为0。

为了更好的理解，我们先普及几个概念。首先innodb支持多个回滚段，每个回滚段包含约1024个slot。

当事务开启时，会给它指定使用哪个回滚段，然后在真正执行操作时，分配具体的slot，通常会有两种slot：

- update_undo: 只用于事务内的update语句
- insert_undo：只用于事务内的insert语句

通常如果事务内只包含一种操作类型，则只使用一个slot。但也有例外，例如insert操作，如果insert的记录在page上已经存在了，但是是无效的，那么久可以直接通过更新这条无效记录的方式来实现插入，这时候使用的是update_undo.

为什么要分成两种undo slot，而不是只用一个slot处理所有呢？这是因为在提交阶段的undo处理不同：

对于Insert undo, 有两种处理方式

- Free: 直接清理掉，因为我们知道新插入的记录产生的Undo不会被任何查询语句所引用，因此可以直接释放undo，这里的undo log不会累加到history list上
- reuse: 当undo 只占用一个page，且page使用低于一定比例时（事实上，第二个条件对于insert undo可以移除掉），放到cachd list上，以备重用。 在重用时，会将该page reset掉 对于update_undo: 也有两种处理方式：
- Purge: 这里会加入到其对应rollback segment的history list数据页列表上，history list长度加1
- Reuse: 同样会将undo加到history list上，history list长度加1。by the way, update undo和insert的重用方式不同，它会在undo page上新建一个undo log header， 而不是重置page。这意味着一个undo页上可能有多个undo log分属不同的事务，但只有一个可能是活跃的。

那么回到最初的问题，既然undo log都加到history list了，为啥在undo purge完成后，未重置为0呢？

我们来看看如下函数

```javascript
trx_purge_truncate --->
  trx_purge_truncate_history --->
    trx_purge_truncate_rseg_history
```

在函数*trx_purge_truncate_rseg_history*中，有如下代码段：

```c++
void trx_purge_truncate_rseg_history()
{
  ...
	if ((mach_read_from_2(seg_hdr + TRX_UNDO_STATE) == TRX_UNDO_TO_PURGE)
       && (mach_read_from_2(log_hdr + TRX_UNDO_NEXT_LOG) == 0))
  {
     /* We can free the whole log segment */
     mutex_exit(&(rseg->mutex));
     mtr_commit(&mtr);
     trx_purge_free_segment(rseg, hdr_addr, n_removed_logs);
     n_removed_logs = 0;
  } else {
     mutex_exit(&(rseg->mutex));
     mtr_commit(&mtr);
  }
  ...
}
```

这里做了特殊判断，只有状态为PURGE的undo log才做了free segment清理。对于cached状态的undo留在原地。个人猜测是因为这些undo log可以留作重用， 在重用之后，再做一次性清理。

为了验证猜测，修改函数*trx_undo_set_state_at_finish*，使undo log状态，要么为TRX_UNDO_TO_FREE， 要么为TRX_UNDO_TO_PURGE。

在给实例加了一定的负载，再做一次slow shutdown重启后，history list length的长度果然变成了0。验证了其无法重置为0是由于cached undo导致。