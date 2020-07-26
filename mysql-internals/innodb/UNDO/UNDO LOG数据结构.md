### UNDO LOG数据结构

**Rsegs**

```c++
using Rsegs_Vector = std::vector<trx_rseg_t *, ut_allocator<trx_rseg_t *>>;

class Rsegs {
  ...
  Rsegs_Vector m_rsegs;
};
```

‌该对象核心是一个由回滚段对象*trx_rseg_t*组成的vector。

‌**trx_rseg_t**‌

这是回滚段内存对象。每个读写事务都会分配一个事务id和回滚段。

```c++
struct trx_rseg_t {
  // 回滚段id
  ulint id;

  // 该回滚段所属space id
  space_id_t space_id;

  // 回滚段header page no
  page_no_t page_no;

  // 回滚段undo page大小
  page_size_t page_size;

  // 回滚段允许分配的最大容量
  ulint max_size;

  // 回滚段当前已使用的容量
  ulint curr_size;

  // 回滚段由undo log构成
  // 这是已分配的update undo log组成的链表
  UT_LIST_BASE_NODE_T(trx_undo_t) update_undo_list;
  // update undo log 对象缓存
  UT_LIST_BASE_NODE_T(trx_undo_t) update_undo_cached;

  // 已分配的insert undo log组成的链表
  UT_LIST_BASE_NODE_T(trx_undo_t) insert_undo_list;
  // insert undo log 对象缓存
  UT_LIST_BASE_NODE_T(trx_undo_t) insert_undo_cached;
  ...
};

struct trx_undo_ptr_t {
  trx_rseg_t *rseg;
  trx_undo_t *insert_undo;
  trx_undo_t *update_undo;
};

struct trx_rsegs_t {
  trx_undo_ptr_t m_redo;
  trx_undo_ptr_t m_noredo;
};

struct trx_t {
  ...
  // 每个事务在启动时会为其分配两个undo log对象
  // insert_undo和update_undo
  // 在事务运行过程中,会一直使用,不会分配新的undo log对象
  trx_rsegs_t rsegs;
}
```

‌*trx_reg_t*代表回滚段内存对象，在回滚段的header page中有一个slot数组，数组大小为TRX_RSEG_N_SLOTS（1024），每个slot对应到一个*trx_undo_t*对象。

‌回滚段对象中维护了多个链表，包括insert undo和update undo list，存储当前已分配正使用的undo log对象，同时存在对应的cache链表以加速undo log分配。‌

有几个问题：

- 为什么要将insert和update undo分开保存，有什么区别？是因为insert和update undo log的回收机制有所不同，后续文章会详细描述。
- 每个事务trx_t和trx_undo_t之间是一一对应关系，但每个事务内可能会更改多个行记录，每次更改完后，trx_t的trx_undo_t对象不会被释放，下一次插入或者更新还是继续使用该trx_undo_t对象，而且会继续复用该trx_undo_t上的page，如果该page内空间不足，还会分配新的空间来继续保存新内容。因此trx_t和trx_undo_t确实是一一对应关系，可参考函数*trx_undo_report_row_operation*。

**trx_undo_t**

记录一个事务的所有undo log信息。

```c++
struct trx_undo_t {
  ulint id;             // 该undo log位于segment内的slot number
  ulint type;           // TRX_UNDO_INSERT 还是 TRX_UNDO_UPDATE
  ulint state;          // 代表事务状态,取值为
  ibool del_marks;      // 不确定
  trx_id_t trx_id;      // 记录事务的id
  XID xid;              // 对于XA事务记录XID
  ibool dict_operation; // 表示该事务是否是DDL
  trx_rseg_t *rseg;     // 所属回滚段 
  space_id_t space; 
  page_size_t page_size;
  page_no_t hdr_page_no;    // undo log的segment header page no
  ulint hdr_offset;       
  page_no_t last_page_no;   // 该undo log可能包含多个undo page,这个成员记录最后一个page no 
  ulint size;             
  ulint empty;              /*!< TRUE if the stack of undo log
                            records is currently empty */
  page_no_t top_page_no;    // undo log中最后一个log record所在page no
  ulint top_offset;         // undo log中最后一个log record所在page内的offset
  undo_no_t top_undo_no;    // undo log中最后一个log record的undo no
  ...
  UT_LIST_NODE_T(trx_undo_t) undo_list;
};
```

‌*trx_undo_t*会通过*undo_list*将自身添加至回滚段的*insert_undo_list*或者*update_undo_list*上。

‌**Tablespace && Tablespaces**

```c++
struct Tablespace {
  ...
 private:
  space_id_t m_id;
  space_id_t m_num;
  char *m_space_name;
  char *m_file_name;
  char *m_log_file_name;
  Rsegs *m_rsegs;
};

class Tablespaces {
  using Tablespaces_Vector =
      std::vector<Tablespace *, ut_allocator<Tablespace *>>;
  ...
  Tablespaces_Vector m_spaces;
};

undo::Tablespaces *undo::spaces;
```

‌UNDO表空间内存对象，其中记录表该表空间id以及对应的文件信息。

‌Tablespaces维护所有UNDO表空间信息，且存在全局的undo spaces对象。如果开启独立表空间，那么每个表空间会对应一个物理文件。

**trx_purge_t**

当undo log不再有用时，会通过后台线程清理该undo log占用的资源，而*trx_purge_t*是控制全局undo purge的核心结构。

```c++
struct trx_purge_t {
  ...
  // 清理开始时的最老的ReadView,任何在该ReadView之前提交的事务UNDO方可被purge
  ReadView view; 
  purge_iter_t iter;     // ???
  purge_iter_t limit;    // 控制purge的进度

  ibool next_stored;     // ???
  trx_rseg_t *rseg;      // 接下来要purge的undo log所属的回滚段信息
  page_no_t page_no;     // 接下来要purge的undo log所属的undo page
  ulint offset;          // 接下来要purge的undo log在undo page内的偏移
  page_no_t hdr_page_no; // ???
  ulint hdr_offset;      // ???

   // 用来选择下一个要purge的undo log的辅助对象
  TrxUndoRsegsIterator *rseg_iter; // ???

  // 小根堆,用于保存所有需要purge的回滚段, 按照trx_no有序插入
  purge_pq_t *purge_queue;

  undo::Truncate undo_trunc;
};
```

‌关于undo log purge的原理和详细实现会在接下来专门的文章中作介绍。