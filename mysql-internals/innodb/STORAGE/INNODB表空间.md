### INNODB表空间结构

#### 概述

除redo日志文件外，InnoDB的物理文件基本上具有相当统一的结构：固定大小页面，使用B-tree来管理。默认情况下，每个页面为16KB，也可以自定义配置该数值。对于压缩表，可以在建表时指定页大小，但在内存中表现的解压页依旧为统一的页大小。

从物理文件的分类来看，有日志文件、主系统表空间文件ibdata、undo tablespace文件、临时表空间文件、用户表空间。我们这里主要关心存储用户数据的用户表空间的物理文件结构。

用户表空间，顾名思义，就是用户创建的表空间，如果开启独立表空间参数，那么一个表空间会对应磁盘上的一个物理文件。

#### segment、extent和page

从外部来看，表空间是由连续的固定大小page构成。其实表空间文件内部还是组织为更复杂的逻辑结构，自顶向下可分为segment、extent和page。

表空间下一级称为segment。segment与数据库中的索引相映射。Innodb引擎内，每个索引对应两个segment：管理叶子节点的segment和管理非叶子节点segment（为什么要使用两个独立的segment？）。创建索引中很关键的步骤便是分配segment，Innodb内部使用INODE来描述segment。

segment的下一级是extent，extent代表一组连续的page，默认为64个page，大小1MB。extent的作用是提高page分配效率，批量分配在效率上总是优于离散、单一的page分配，另外在数据连续性方面也更佳，segment扩容时以extent为单位分配。Innodb内部使用XDES来描述extent。

page则是表空间数据存储的基本单位，innodb将表文件（xxx.ibd）按page切分，依类型不同，page内容也有所区别，最为常见的是存储数据库表的行记录。page格式会在后续文章中进行比较详细的剖析，本文中主要介绍一些特殊的存储表空间元信息的page。

#### 表空间关键page

每个独立表空间都对应磁盘上的一个物理文件，命名形式为{table_name}.ibd。物理文件按page切分，这些page主要分为两类：存储表空间元信息的page和存储表空间用户数据的page。存储用户数据的page暂不关心，这里主要讨论表空间元信息page。

如下图，表空间的前三个page为主要的元信息page。

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_1.jpg)

**FSP HEADER PAGE**

FSP header page是表空间的root page，存储表空间关键元数据信息。由page file header、fsp header、xdes entries三大部分构成。完整格式如下图：

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_2.jpg)

root page也存在38字节头部信息。其中几个关键字段：

> **FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID**：存储表空间的space id
> **FIL_PAGE_SRV_VERSION**：本来被用于存储前一个页面id，在root page中被用来存储SERVER版本号
> **FIL_PAGE_SPACE_VERSION**：本来被用于存储后一个页面id，在root page中被用来存储TABLE SPACE版本号

fsp header主要存储表空间元信息，维护关键结构分配链表，主要成员有：

> **FSP_SIZE**：表空间大小，以Page数量计算
> **FSP_FREE_LIMIT**：目前在空闲的Extent上最小的尚未被初始化的Page的Page Number
> **FSP_FREE**：空闲extent链表，链表中的每一项为代表extent的xdes，所谓空闲extent是指该extent内所有page均未被使用
> **FSP_FREE_FRAG**：free frag extent链表，链表中的每一项为代表extent的xdes，所谓free frag extent是指该extent内有部分page未被使用
> **FSP_FULL_FRAG**：full frag extent链表，链表中的每一项为代表extent的xdes，所谓full frag extent是指该extent内所有Page均已被使用
> **FSP_SEG_ID**：下次待分配的segment id，每次分配新segment时均会使用该字段作为segment id，并将该字段值+1写回
> **FSP_SEG_INODES_FULL**：full inode page链表，链表中的每一项为inode page，该链表中的每个inode page内的inode entry都已经被使用
> **FSP_SEG_INODES_FREE**：free inode page链表，链表中的每一项为inode page，该链表中的每个inode page内上有空闲inode entry可分配

上面提到了各种链表，这些链表其实是磁盘上的同类对象连接形成，可加速对象分配和管理。后文会描述其实现细节。需要注意的是，root page中存在两类链表：第一是链接描述extent的xdes，第二是链接inode page。

第三部分是描述extent的xdes（extent descriptor）信息，每个xdes page中均存储256个xdes，描述接下来连续的256个extent（16384个page）。

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_3.jpg)

每个XDES项占据40字节，一个xdes page可跟踪其后的256个extent的分配情况，XDES结构如下：

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_4.jpg)

由于单个xdes page只能描述256个extent，因此，每隔256个extent（16384个page）便需要一个xdes page。

**IBUF BITMAP PAGE**

作用暂时未知，后续遇到再作分析

**INODE PAGE**

表空间文件的第3个page的类型为*FIL_PAGE_INODE*，存储inode（index node），管理表空间的segment。每个inode对应一个segment。每个inode page默认存储FSP_SEG_INODES_PER_PAGE（85）个inode。每个索引使用2个segment，分别用于管理叶子节点和非叶子节点。

inode page由page header和inode entry组成，page header为38字节。inode entry为192字节。

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_5.jpg)

Inode Entry的结构如下表：

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_6.jpg)

inode entry中最关键的也是各种extent链表（什么是extent链表？应该是代表extent的xdes构成的链表吧）。优先从空闲extent链表中分配page，当无可用extent时，会向表空间中申请若干可用extent加入自身的空闲extent链表。另外，每个segment还有若干（32）碎片page，当segment初始扩容时，首先分配这些碎片page，直到分配完毕才会进入extent分配。

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_7.jpg)

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_8.jpg)

### **关键流程**

**创建表空间**

创建表空间最终落实到函数*fil_create_tablespace*，创建ibd文件并初始化fsp header：

```cpp
dberr_t dict_build_tablespace(trx_t *trx, Tablespace *tablespace) {
  ...
  err = fil_ibd_create(space, tablespace->name(), datafile->filepath(),
                       tablespace->flags(), FIL_IBD_FILE_INITIAL_SIZE);
  ...
  // 初始化fsp header的其他字段
  bool ret = fsp_header_init(space, FIL_IBD_FILE_INITIAL_SIZE, &mtr, false);
  ...
}

dberr_t fil_create_tablespace(space_id_t space_id, const char *name,
                              const char *path, ulint flags,
                              page_no_t size, fil_type_t type)
{
    ...
    // 创建ibd文件
    file = os_file_create(...);
    ...
    // 这里会将新创建的ibd文件填0以预分配空间
    // 调试显示size=7，即分配的空间为7*16KB，但os_file_set_size内会将其向上调整至8*16KB
    // buf2 = static_cast<byte *>(ut_malloc_nokey(buf_size + UNIV_PAGE_SIZE));
    success = os_file_set_size(path, file, 0, size * page_size.physical(),
                             srv_read_only_mode, true);
    ...
    // 初始化fsp header pages，其中前3个page作用比较关键：
    // page 0: root page，也是fsp header page
    // page 1: ibuf_bitmap page，作用暂时不明
    // page 2: inode page，管理segment分配，每个inode entry对应一个segment
    buf2 = static_cast<byte *>(ut_malloc_nokey(3 * page_size.logical()));

    page = static_cast<byte *>(ut_align(buf2, page_size.logical()));
    memset(page, '\0', page_size.logical());
    
    flags = fsp_flags_set_page_size(flags, page_size);
    fsp_header_init_fields(page, space_id, flags);
    mach_write_to_4(page + FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID, space_id);

    mach_write_to_4(page + FIL_PAGE_SRV_VERSION, DD_SPACE_CURRENT_SRV_VERSION);
    mach_write_to_4(page + FIL_PAGE_SPACE_VERSION, DD_SPACE_CURRENT_SPACE_VERSION);
    ...
}
```

### **创建segment**

innodb的space中，inode、extent和page之间环环相扣，inode对应的是segment，extent 对应的是页簇，page是页。必需先创建segment，然后再从segment内分配extent和page。创建segment核心是从inode page中分配空闲的inode。

```cpp
buf_block_t *fseg_create_general(
    space_id_t space_id, 
    page_no_t page,  // 调用者多传入0,表示需要分配一个新page作为segment header
    ulint byte_offset,   
    ibool has_done_reservation, 
    mtr_t *mtr)                 
{

  fil_space_t *space = fil_space_get(space_id);

  const page_size_t page_size(space->flags);
  ...
  space_header = fsp_get_space_header(space_id, page_size, mtr);

  // 分配inode entry，代表该segment
  inode = fsp_alloc_seg_inode(space_header, mtr);

  // 从table space的page header中读取下一个要使用的segment id
  // 将其设置在inode entry的FSEG_ID字段中
  // 同时将刚刚已分配的segment id递增1并写回space的page header的FSP_SEG_ID字段
  seg_id = mach_read_from_8(space_header + FSP_SEG_ID);
  mlog_write_ull(space_header + FSP_SEG_ID, seg_id + 1, mtr);
  mlog_write_ull(inode + FSEG_ID, seg_id, mtr);
  mlog_write_ulint(inode + FSEG_NOT_FULL_N_USED, 0, MLOG_4BYTES, mtr);

  // 初始化inode中的各种extent链表
  flst_init(inode + FSEG_FREE, mtr);
  flst_init(inode + FSEG_NOT_FULL, mtr);
  flst_init(inode + FSEG_FULL, mtr);

  // 初始化segment的frag page数组为空(未分配任何page)
  mlog_write_ulint(inode + FSEG_MAGIC_N, FSEG_MAGIC_N_VALUE, MLOG_4BYTES, mtr);
  for (i = 0; i < FSEG_FRAG_ARR_N_SLOTS; i++) {
    fseg_set_nth_frag_page_no(inode, i, FIL_NULL, mtr);
  }

  // 且该page的类型是FIL_PAGE_TYPE_SYS
  // 该page是Change Buffer的header page，主要用于对ibuf btree的Page管理。
  if (page == 0) {
    block = fseg_alloc_free_page_low(space, page_size, inode, 0, FSP_UP,
                                     RW_SX_LATCH, mtr, mtr);

    header = byte_offset + buf_block_get_frame(block);
    mlog_write_ulint(buf_block_get_frame(block) + FIL_PAGE_TYPE,
                     FIL_PAGE_TYPE_SYS, MLOG_2BYTES, mtr);
  }

  // 初始化page header
  mlog_write_ulint(header + FSEG_HDR_OFFSET, page_offset(inode), MLOG_2BYTES,
                   mtr);
  mlog_write_ulint(header + FSEG_HDR_PAGE_NO,
                   page_get_page_no(page_align(inode)), MLOG_4BYTES, mtr);
  mlog_write_ulint(header + FSEG_HDR_SPACE, space_id, MLOG_4BYTES, mtr);
  ...
}

fseg_inode_t *fsp_alloc_seg_inode(fsp_header_t *space_header, mtr_t *mtr)
{
  ...
  // 如果FSP_SEG_INODES_FREE链表为空
  // 该链表用于链接那些尚有inode空间可分配的inode_page
  // 如果该链表为空,那就要分配一个新的inode page了,这个过程我们在下面描述
  // 调用fsp_alloc_seg_inode_page创建一个新的inode page
  if (flst_get_len(space_header + FSP_SEG_INODES_FREE) == 0 &&
      !fsp_alloc_seg_inode_page(space_header, mtr)) {
    return (NULL);
  }
  
  ...
  // 选择TABLE SPACE的拥有空闲inode的page链表的表头
  // 然后从被选择的inode page中分配一个空闲inode
  const page_id_t page_id(
      page_get_space_id(page_align(space_header)),
      flst_get_first(space_header + FSP_SEG_INODES_FREE, mtr).page);

  block = buf_page_get(page_id, page_size, RW_SX_LATCH, mtr);
  page = buf_block_get_frame(block);

  // 遍历该inode page的所有inode entry
  // 如果发现某个Inode Entry的FSEG_ID字段未设置
  // 即认为该inode entry尚未分配
  n = fsp_seg_inode_page_find_free(page, 0, page_size, mtr);
  inode = fsp_seg_inode_page_get_nth_inode(page, n, page_size, mtr);

  // 如果由于本次分配导致该inode page内不再有可用的inode
  // 那需要将该inode page从space的FSP_SEG_INODES_FREE链表摘除并加入FSP_SEG_INODES_FULL链表尾部
  if (FIL_NULL == fsp_seg_inode_page_find_free(page, n + 1, page_size, mtr)) {
    flst_remove(space_header + FSP_SEG_INODES_FREE, page + FSEG_INODE_PAGE_NODE,
                mtr);
    flst_add_last(space_header + FSP_SEG_INODES_FULL,
                  page + FSEG_INODE_PAGE_NODE, mtr);
  }
  return (inode);
}
```

优先从table space的FSP_SEG_INODES_FREE链表中获取inode page并从中分配未使用的inode。

如果FSP_SEG_INODES_FREE链表为空，说明当前已无空闲的inode page，此时需分配新的inode page，参考函数*fsp_alloc_seg_inode_page*：

```cpp
// 从space中分配一个未使用的inode page
ibool fsp_alloc_seg_inode_page(
    fsp_header_t *space_header,
    mtr_t *mtr)
{
  space = page_get_space_id(page_align(space_header));

  // fsp_alloc_free_page这个函数比较有意思,值得仔细研究一番
  // 分配一个空闲page作为inode page
  block = fsp_alloc_free_page(space, page_size, 0, RW_SX_LATCH, mtr, mtr);
  page = buf_block_get_frame(block);
  // 设置新分配的PAGE类型为FIL_PAGE_INODE
  mlog_write_ulint(page + FIL_PAGE_TYPE, FIL_PAGE_INODE, MLOG_2BYTES, mtr);

  // 初始化inode page中的所有inode信息为空闲
  for (page_no_t i = 0; i < FSP_SEG_INODES_PER_PAGE(page_size); i++) {
    inode = fsp_seg_inode_page_get_nth_inode(page, i, page_size, mtr);
    mlog_write_ull(inode + FSEG_ID, 0, mtr);
  }
  // 将新分配的inode page加入至FSP_SEG_INODES_FREE链表的尾部
  flst_add_last(space_header + FSP_SEG_INODES_FREE, page + FSEG_INODE_PAGE_NODE, mtr);
  return (TRUE);
}
```

### **分配extent**

*fsp_alloc_free_extent*被用来分配一个extent，即xdes，当segment内无可用page时会向表空间申请分配空闲extent。

分配extent的核心是分配空闲xdes，因为根据xdes我们便可以快速定位extent。

```cpp
// 参数@hint给出一个线索:可以尝试先使用该page所在的extent,如果其状态为free的话
// hint表明调用者建议从该page开始分配空间以获得更好的连续性
// 该函数会优先分配该hint page所在的extent,如果其尚未被分配的话
xdes_t *fsp_alloc_free_extent(space_id_t space_id,
                              const page_size_t &page_size,
                              page_no_t hint, mtr_t *mtr)
{
  header = fsp_get_space_header(space_id, page_size, mtr);

  // 计算hint page所在的extent的xdes对象位置
  // 这个函数会在下面仔细分析
  descr = xdes_get_descriptor_with_space_hdr(header, space_id, hint, mtr, false,
                                             &desc_block);
  fil_space_t *space = fil_space_get(space_id);

  // 该xdes状态是FREE，表明该extent未分配给其他人使用
  if (descr && (xdes_get_state(descr, mtr) == XDES_FREE)) {
      /* Ok, we can take this extent */
  } else {
    // 需要从table space header上的FSP_FREE链表中获取一个可分配的extent
    first = flst_get_first(header + FSP_FREE, mtr);
    if (fil_addr_is_null(first)) {
      // 如果FSP_FREE链表为空,那么只能分配一个新的xdes了，其磁盘位置记录在first中(page, boff)
      fsp_fill_free_list(false, space, header, mtr);
      first = flst_get_first(header + FSP_FREE, mtr);
    }
    descr = xdes_lst_get_descriptor(space_id, page_size, first, mtr);
  }
  flst_remove(header + FSP_FREE, descr + XDES_FLST_NODE, mtr);
  space->free_len--;
  return (descr);
}
```

该函数的调用者会给该函数一个提示hint，指示优先分配该hint page所在的extent，如果该extent是空闲状态（描述extent的xdes有state状态位），便分配该extent，否则需重新分配：

1. 从table space的FSP_FREE链表上获取第一个可分配的xdes，将其从FSP_FREE链表摘除并直接返回即可。
2. 如果FSP_FREE链表为空，则需分配新的xdes了，调用函数*fsp_fill_free_list*。它用来分配extent并返回其相应的xdes。

```cpp
void fsp_fill_free_list(bool init_space, fil_space_t *space,
                        fsp_header_t *header, mtr_t *mtr)
{
  // size代表table space当前的大小(可以理解为ibd文件的大小)
  size = mach_read_from_4(header + FSP_SIZE);
  // limit: 暂时还不理解
  limit = mach_read_from_4(header + FSP_FREE_LIMIT);
  flags = mach_read_from_4(header + FSP_SPACE_FLAGS);

  const page_size_t page_size(flags);

  // 需要扩张ibd文件,调用fsp_try_extend_data_file
  // 扩张ibd文件逻辑在下面仔细研究
  if (size < limit + FSP_EXTENT_SIZE * FSP_FREE_ADD) {
    if ((!init_space && !fsp_is_system_tablespace(space->id) &&
         !fsp_is_global_temporary(space->id)) ||
        (space->id == TRX_SYS_SPACE &&
         srv_sys_space.can_auto_extend_last_file()) ||
        (fsp_is_global_temporary(space->id) &&
         srv_tmp_space.can_auto_extend_last_file())) {
      fsp_try_extend_data_file(space, header, mtr);
      size = space->size_in_header;
    }
  }

  // 文件成功扩张后,开始分配extent
  // limit到底是什么意思呢？
  i = limit;

  // 结束条件:
  // 1. 分配的extent未超过当前表空间物理文件大小(size)
  // 2. 分配的extent数量超过了FSP_FREE_ADD(4):这是因为程序中限制了每次填充extent的最大个数
  while ((init_space && i < 1) ||
         ((i + FSP_EXTENT_SIZE <= size) && (count < FSP_FREE_ADD))) {
    // 什么时候init_xdes为true呢? 还没思考明白
    // #define ut_2pow_remainder(n, m) ((n) & ((m)-1))
    bool init_xdes = (ut_2pow_remainder(i, page_size.physical()) == 0);

    space->free_limit = i + FSP_EXTENT_SIZE;
    mlog_write_ulint(header + FSP_FREE_LIMIT, i + FSP_EXTENT_SIZE, MLOG_4BYTES, mtr);

    // 需要初始化xdes page
    if (init_xdes) {
      ...
    }

    buf_block_t *desc_block = NULL;
    // 根据上面分配的extent计算其对应的xdes所在的xdes page以及在page内的偏移
    descr = xdes_get_descriptor_with_space_hdr(header, space->id, i, mtr,
                                               init_space, &desc_block);
    xdes_init(descr, mtr);

    if (init_xdes) {
      fsp_init_xdes_free_frag(header, descr, mtr);
    } else {
      // 将刚创建的extent加入至fsp header的FSP_FREE链表尾部
      flst_add_last(header + FSP_FREE, descr + XDES_FLST_NODE, mtr);
      count++;
    }

    i += FSP_EXTENT_SIZE;
  }
  space->free_len += (uint32_t)count;
}
```

这个函数有几个比较有趣的地方：

1. *fsp_try_extend_data_file*：尝试扩张table space文件，分配extent说到底需要底层物理存储空间
2. *init_xdes*的计算
3. *xdes_get_descriptor_with_space_hdr*：根据分配的extent计算其对应的xdes所在的page以及page内偏移

我们接下来一一阐述。

*fsp_try_extend_data_file*用于扩充ibd文件：

```cpp
ulint fsp_try_extend_data_file(fil_space_t *space,
                               fsp_header_t *header,
                               mtr_t *mtr)
{

  size = mach_read_from_4(header + FSP_SIZE);

  const page_size_t page_size(mach_read_from_4(header + FSP_SPACE_FLAGS));

  if (space->id == TRX_SYS_SPACE) {
      ...
  } else if (fsp_is_global_temporary(space->id)) {
      ...
  } else {
    // 首先判断如果当前ibd文件大小小于extent_size(典型是1MB)
    // 首先将其extent至extent_size(典型是1MB)
    page_no_t extent_pages = fsp_get_extent_size_in_pages(page_size);
    if (size < extent_pages) {
      /* Let us first extend the file to extent_size */
      if (!fsp_try_extend_data_file_with_pages(space, extent_pages - 1, header,
                                               mtr)) {
      }
      size = extent_pages;
    }
    // 接下来判断每次extent的幅度,其算法是:
    // 1. 如果当前ibd大小小于32个extent_size,每次extent单位为extent_size
    // 2. 否则,每次extent单位为FSP_FREE_ADD(4) * extent_size;
    size_increase = fsp_get_pages_to_extend_ibd(page_size, size);
  }

  if (!fil_space_extend(space, size + size_increase)) {
    DBUG_RETURN(false);
  }

  // 修改fsp header中的大小
  space->size_in_header =
      ut_calc_align_down(space->size, (1024 * 1024) / page_size.physical());

  fsp_header_size_update(header, space->size_in_header, mtr);
}
```

*xdes_get_descriptor_with_space_hdr*用来根据extent的起始page no计算该extent对应的xdes所在的page以及在page内的位置。

```cpp
// offset为extent的起始page no
xdes_t *xdes_get_descriptor_with_space_hdr(...)
{
  ulint limit;
  ulint size;
  page_no_t descr_page_no;
  ulint flags;
  page_t *descr_page;

  const page_size_t page_size(flags);

  descr_page_no = xdes_calc_descriptor_page(page_size, offset);

  buf_block_t *block;

  if (descr_page_no == 0) {
    /* It is on the space header page */
    descr_page = page_align(sp_header);
    block = NULL;
  } else {
    block = buf_page_get(page_id_t(space, descr_page_no), page_size,
                         RW_SX_LATCH, mtr);
    descr_page = buf_block_get_frame(block);
  }

  if (desc_block != NULL) {
    *desc_block = block;
  }

  return (descr_page + XDES_ARR_OFFSET +
          XDES_SIZE * xdes_calc_descriptor_index(page_size, offset));
}
```

这个函数是根据extent的起始page no来计算其xdes所在的page以及page内偏移。从前文的描述中我们知道，每个extent对应了一个描述它的xdes，且xdes存储在XDES_PAGE之中（当然 root page中也会存储xdes），每个XDES_PAGE内可存储256个xdes结构，用来描述其后连续256个extent(1M)的情况，因此，一旦我们知道了某个extent的起始page no，便可以反推出其对应的xdes所在的page，*xdes_calc_descriptor_page*函数我们会在后面仔细描述。如下：

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_9.jpg)



### **分配page**

```cpp
buf_block_t *fseg_alloc_free_page_low(...)
{
    ...
}
```

在*fseg_alloc_free_page_low*函数中实现了7种情况从segment中分配page：

1. segment的hint位置的页是空闲状态，直接返回对应的page
2. xdes是空闲状态，但segment中的空闲page数量小于总page数量的1/8且碎片页被全部用完，为其分配一个空闲extent，并直接分配hint对应的page
3. 如果xdes不是空闲状态，且segment inode中的空闲page数量 < 1/8，在inode当中获得一个空闲的extent，并且将这个extent descr对应的页返回。
4. descr是XDES_FSEG状态，且这个extent中还有空闲page,从其中获取一个page.
5. 除了以上情况外，如果descr不是空闲的，但是inode还有其他的空闲extent,从其他的extent获得一个空闲
6. 如果其他的extent没有空闲页，但是fragment array还有空闲的碎片page，从空闲的碎片page中获得一个空闲页。
7. 如果连碎页也没有，直接申请分配一个新的extent，并在其中获取一个空闲的page.

### **关键辅助对象和函数**

**磁盘链表**

Innodb的磁盘链表主要是用来连接存储在磁盘上的对象。链表每项并非基于内存指针的，而是基于对象在磁盘中的位置(page no + offset)来描述。

```cpp
typedef struct fil_addr_struct
{
  ulint page;    /*page在space中的编号*/
  ulint boffset; /*page中的字节偏移量，在内存中使用2字节表示*/
} fil_addr_t;

typedef byte flst_node_t;
typedef byte flst_base_node_t;
/* The physical size of a list base node in bytes */
constexpr ulint FLST_BASE_NODE_SIZE = 4 + 2 * FIL_ADDR_SIZE;
/* The physical size of a list node in bytes */
constexpr ulint FLST_NODE_SIZE = 2 * FIL_ADDR_SIZE;
```

*flst_node_t*代表链表上的一个节点，其大小为12字节，其中前6字节（page:4 boffset:2）表示链表中前一个节点的fil_addr_t信息，后6个字节表示链表中下一个节点的fil_addr_t信息。

*flst_base_node_t*则用于描述链表的整体信息，包括：

> 链表节点个数FLST_LEN（4字节）
> 链表首节点的fil_addr_t (6字节)
> 链表尾节点的fil_addr_t（6字节）

磁盘空间链表的结构如下所示：

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_10.jpg)

Innodb 表空间的fsp header page中定义了多种链表，这在前面已经作过描述，不再赘述。这些链表在fsp header_init时被初始化，初始时链表为空：

```cpp
bool fsp_header_init(space_id_t space_id, page_no_t size, mtr_t *mtr,
                     bool is_boot) 
{
  flst_init(header + FSP_FREE, mtr);
  flst_init(header + FSP_FREE_FRAG, mtr);
  flst_init(header + FSP_FULL_FRAG, mtr);
  flst_init(header + FSP_SEG_INODES_FULL, mtr);
  flst_init(header + FSP_SEG_INODES_FREE, mtr);
}

UNIV_INLINE
void flst_init(flst_base_node_t *base, mtr_t *mtr)          
{
  mlog_write_ulint(base + FLST_LEN, 0, MLOG_4BYTES, mtr);
  flst_write_addr(base + FLST_FIRST, fil_addr_null, mtr);
  flst_write_addr(base + FLST_LAST, fil_addr_null, mtr);
}

buf_block_t *fseg_create_general(...)
{
  ...
  flst_init(inode + FSEG_FREE, mtr);
  flst_init(inode + FSEG_NOT_FULL, mtr);
  flst_init(inode + FSEG_FULL, mtr);
  ...
}
```

每个要连接在该链表上的对象中均需要预留12个字节来记录前后节点位置。例如，xdes和inode page中均有该12字节的预留。

![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_11.jpg)



![img](/Users/dingkai/Documents/work/github/tracymacding/books/mysql-internals/innodb/STORAGE/PIC/table_space_12.jpg)

**xdes_calc_descriptor_page**

该函数根据page no计算描述其所在的extent的xdes对象位置（即xdes所在的page no）。

```cpp
// 以m为16384为例，如果n < 16384，计算的结果是0
// 如果16384 =< n < 32768,计算的结果是16384
// 如果32768 =< n < 49152,计算的结果是32768
// 以此类推
#define ut_2pow_round(n, m) ((n) & ~((m)-1))

page_no_t
xdes_calc_descriptor_page(const page_size_t &page_size, page_no_t offset) 
{
	return (ut_2pow_round(offset, page_size.physical()));
}
```

根据我们上面的知识储备，每个xdes page管理随后的连续256个extent的空间（假如page size为16KB的话，那总大小就是256MB）。假如每个page是16KB，256MB包含的页面数是256MB/16KB = 16384，因此，所有page no小于16384的page其xdes都会位于第0个page（root page），而page no介于16384~32768的page其xdes page便是page 16384，与*xdes_calc_descriptor_page*计算是吻合的。

### **参考**

- [https://blog.jcole.us/2013/01/04/page-management-in-innodb-space-files/](https://link.zhihu.com/?target=https%3A//blog.jcole.us/2013/01/04/page-management-in-innodb-space-files/)
- [http://www.leviathan.vip/2019/04/18/InnoDB%E7%9A%84%E6%96%87%E4%BB%B6%E7%BB%84%E7%BB%87%E7%BB%93%E6%9E%84/](https://link.zhihu.com/?target=http%3A//www.leviathan.vip/2019/04/18/InnoDB%25E7%259A%2584%25E6%2596%2587%25E4%25BB%25B6%25E7%25BB%2584%25E7%25BB%2587%25E7%25BB%2593%25E6%259E%2584/)
- [https://colinback.github.io/szyblogs/database/2018/05/10/innodb-kernel-5/](https://link.zhihu.com/?target=https%3A//colinback.github.io/szyblogs/database/2018/05/10/innodb-kernel-5/)
- [http://mysql.taobao.org/monthly/2016/02/01/](https://link.zhihu.com/?target=http%3A//mysql.taobao.org/monthly/2016/02/01/)
- [https://www.jianshu.com/p/55d861c59bc3](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/55d861c59bc3)