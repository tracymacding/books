### INNODB 表空间EXTENT实现详解

从前面的文章中我们知道INNODB内使用了EXTENT来管理物理PAGE。使用EXTENT的好处有：

1. 加速物理页面分配效率，相比于每次需要时分配单个物理页，通过分配EXTENT然后再从其中分配PAGE的效率更高，因为只需要在一个EXTENT内搜索bitmap空闲位即可
2. 有利于数据连续性，提高读写效率。EXTENT内PAGE在磁盘上连续存放。而连续访问相对于随机访问性能更佳。

EXTENT以XDES来描述，XDES即EXTENT DESCRIPTOR，是描述extent的元信息。其具体的结构会在下面描述。同时，表空间对象FSP_HEADER内还需要通过各种链表来管理不同状态的EXTENT，以加速EXTENT的分配。

#### XDES格式

XDES用于描述EXTENT状态，每个XDES占据40个字节，包含如下字段：

> * segment_id: 该extent所属segment id，占据8个字节
>
> * node: 其所在的链表的连接件，包含prev和next。可能位于FSP_FREE/FSP_FREE_FRAG/FSP_FULL_FRAG某个链表上，链表项的内容是XDES所在的page_no以及page内的offset，总共占据12字节
>
> * state：该extent状态，可能为XDES_FREE / XDES_FREE_FRAG /XDES_FULL_FRAG / XDES_FSEG/XDES_FSEG_FRAG，占据4个字节
>
> * bitmap：位图表示xdes描述的page的状况。每个page使用2个bit，位0表示page是否分配，位1表示page是否干净(不过目前没有使用)，因而64个page总共需要16字节来描述

#### EXTENT可能被加入哪些链表

EXTENT可能被加入到以下链表：

> FSP_FREE:
>
> FSP_FREE_FRAG:
>
> FSP_FULL:
>
> FSP_FULL_FRAG:
>
> FSEG_FREE: segment上所有page均空闲的extent构成的链表
>
> FSEG_NOT_FULL: 至少有一个page被分配使用的extent构成的链表，当该extent所有page均已分配，其会被转移至FSEG_FULL链表，当所有page已释放，其会被转移至FSEG_FREE链表
>
> FSEG_FULL: segment上所有page均已使用的extent构成的链表

extent被创建时（其实是其对应的xdes被初始化）都是位于FSP_FREE或者FSP_FREE_FRAG链表上，这取决于它是否包含xdes page。在系统运行时，当segment上没有可用extent时，会从表空间的这两个链表选择其一：优先从FSP_FREE_FRAG链表上获取，如果失败，再从FSP_FREE链表上分配，然后将分配得到的extent从原链表中摘除并插入至segment的FSEG_NOT_FULL或者FSEG_FREE链表。

#### EXTENT可能的状态

EXTENT可能处于以下几个状态：

> **XDES_FREE**：extent内不包含xdes page且其对应的xdes已经初始化，它会被插入FSP_FREE链表
>
> **XDES_FREE_FRAG**：extent内包含xdes page且其对应的xdes已经初始化，它会被插入FSP_FREE_FRAG链表
>
> **XDES_FULL_FRAG**：extent内包含xdes page且其所有page已经被分配使用，它会被插入FSP_FREE_FRAG链表
>
> **XDES_FSEG**: extent内不包含xdes page且已经被分配给某个特定的segment，它会被插入FSEG_FREE链表
>
> **XDES_FSEG_FRAG**：extent内包含xdes page且已经被分配给某个特定的segment，它会被插入FSEG_FREE_NOT_FULL链表

#### XDES PAGE

INNODB中存在一种专门存储XDES的页类型，称为FIL_PAGE_TYPE_XDES。其内存储的是XDES数组，每个页内固定存储256个XDES项，用以描述接下来16384个page的使用情况。因而，在INNODB中，每隔16384个page都会放置一个XDES PAGE。

除此之外，FSP_HEADER_PAGE中紧随FSP_HEADER信息后也会存储XDES，同样由256项，管理起始的16384个page。

#### 创建空表时的EXTENT

创建用户表时，虽然还未插入任何行记录。但系统已经分配了多个PAGE来存放表的元信息。这个在前面我们已经有所描述，本文章中我们只关注与EXTEND相关的PAGE。

**PAGE-0**

page 0用来存储FILE_SPACE_HDR信息，其中包括:

> * FSP_FREE：free extent构成的链表
> * FSP_FREE_FRAG：部分free的extent构成的链表
> * FSP_FULL_FRAG：未知
> * FSP_FREE_LIMIT：当前已用到的位置，大于该位置表示还未初始化
> * FSP_FRAG_N_USED：FSP_FREE_FRAG链表中已使用的page数目

除此以外，在表空间的page 0中紧接着FILE_SPACE_HDR存放的是xdes数组。每个xdes项占据40个字节大小，描述一个extent。每个page的xdes数组为256项，用于描述接下来256个extent的状态。而在默认16KB的页面配置下，每个extent对应了64个page，因而，每个xdes page可描述接下来连续的16384个page的状态。

建表时，会创建表空间相关page：

```c++
bool fsp_header_init(space_id_t space_id, page_no_t size, mtr_t *mtr,
                     bool is_boot)
{
	header = FSP_HEADER_OFFSET + page;
  mlog_write_ulint(header + FSP_FREE_LIMIT, 0, MLOG_4BYTES, mtr);
  mlog_write_ulint(header + FSP_SPACE_FLAGS, space->flags, MLOG_4BYTES, mtr);
  mlog_write_ulint(header + FSP_FRAG_N_USED, 0, MLOG_4BYTES, mtr);

  // 初始化各种extent链表为空
  flst_init(header + FSP_FREE, mtr);
  flst_init(header + FSP_FREE_FRAG, mtr);
  flst_init(header + FSP_FULL_FRAG, mtr);
  
  fsp_fill_free_list(
      !fsp_is_system_tablespace(space_id) && !fsp_is_global_temporary(space_id),
      space, header, mtr);
}

// 这是一个通用函数,只要创建extent时都会调用它
static void fsp_fill_free_list(bool init_space, fil_space_t *space,
                               fsp_header_t *header, mtr_t *mtr)
{
  // FSP_SIZE在初始化时为7(个page)
  size = mach_read_from_4(header + FSP_SIZE);
  // FSP_FREE_LIMIT在初始化时为0
  limit = mach_read_from_4(header + FSP_FREE_LIMIT);
  flags = mach_read_from_4(header + FSP_SPACE_FLAGS);

  const page_size_t page_size(flags);

  if (size < limit + FSP_EXTENT_SIZE * FSP_FREE_ADD) {
    ...
  }

  i = limit;

  while ((init_space && i < 1) ||
         ((i + FSP_EXTENT_SIZE <= size) && (count < FSP_FREE_ADD))) {
    // init_xdes判断是否需要创建XDES_PAGE
    // 因为每个xdes page可管理16KB个page,因而,只要i达到了16KB的整数倍
    // 便需要创建一个xdes page和一个ibuf bitmap page
    bool init_xdes = (ut_2pow_remainder(i, page_size.physical()) == 0);
		// 设置当前已经初始化的位置
    space->free_limit = i + FSP_EXTENT_SIZE;
    mlog_write_ulint(header + FSP_FREE_LIMIT, i + FSP_EXTENT_SIZE, MLOG_4BYTES,
                     mtr);

    if (init_xdes) {
      buf_block_t *block;
      // i = 0说明xdes entry存储在file space header page(即page 0中)
      // 否则,就需要创建独立的XDES PAGE了
      if (i > 0) {
        const page_id_t page_id(space->id, i);
        block = buf_page_create(page_id, page_size, RW_SX_LATCH, mtr);
        fsp_init_file_page(block, mtr);
        mlog_write_ulint(buf_block_get_frame(block) + FIL_PAGE_TYPE,
                         FIL_PAGE_TYPE_XDES, MLOG_2BYTES, mtr);
      }
      // 创建ibuf bitmap page
      if (!fsp_is_system_temporary(space->id)) {
       ...
      }
    }

    buf_block_t *desc_block = nullptr;
    // 根据i(page number)计算管理它的xdes
    descr = xdes_get_descriptor_with_space_hdr(header, space->id, i, mtr,
                                               init_space, &desc_block);
    // 初始化该xdes,主要是bitmap,标识本次64个page为干净未使用
    xdes_init(descr, mtr);
		// 如果是需要创建xdes page,那么该extent的前两个page已经被xdes page和ibuf bitmap占据
    // 因为该extent非所有的page都free,因而只能将其添加至fsp_frag_free链表了
    // 否则,将其添加至fsp_free链表
    if (init_xdes) {
      fsp_init_xdes_free_frag(header, descr, mtr);
    } else {
      flst_add_last(header + FSP_FREE, descr + XDES_FLST_NODE, mtr);
      count++;
    }
		// 每个xdes管理64个page
    i += FSP_EXTENT_SIZE;
  }
  space->free_len += (uint32_t)count;
}
```

*fsp_fill_free_list*有两个调用时机：

1. 创建表空间调用fsp_header_init中调用以将第一个extent（起始64个page对应的xdes）初始化并将其加入至fsp_free_frag链表上
2. 创建extent时（fsp_alloc_free_extent），如果fsp_free链表为空，那此时需要从底层创建一个extent并初始化其对应的xdes。

用自己写的工具分析了下刚刚创建的空表的fsp header page的详细信息，如下：

```
************ FILE_PAGE_HEADER ****************
     0:   page_crc      : 0x5e64fc2c
     4:   page_no       : 0
     8:   page_version  : 80013
    12:   space_version : 1
    16:   page_lsn      : 46915837
    24:   page_type     : FIL_PAGE_TYPE_FSP_HDR
    26:   page_flush_lsn: 0
    34:   space_id      : 2
************ FILE_PAGE_HEADER ****************
************ FSP_HEADER ****************
    38:   space_id              : 2
    42:   reserve               : 0
    46:   page_nr               : 7
    50:   free_limit            : 64
    54:   flags                 : 0x4021
    58:   frag_n_used           : 5
    62:   free_extent_list      : {Len: 0, Head: nil, Tail: nil}
    78:   free_frag_extent_list : {Len: 1, Head: {0-158}, Tail: {0-158}}
    94:   full_frag_extent_list : {Len: 0, Head: nil, Tail: nil}
   110:   next_segment_id       : 5
   118:   inode_full_list       : {Len: 0, Head: nil, Tail: nil}
   134:   inode_free_list       : {Len: 1, Head: {2-38}, Tail: {2-38}}
************ FSP_HEADER ****************
************ FSP_XDES_ENTRIES ****************
   150:   xdes[  0]: seg_id: 0, node: prev: nil, next: nil, state: XDES_FREE_FRAG, bitmap: 0xaafeffffffffffffffffffffffffffff
   190:   xdes[  1]: seg_id: 0, node: prev: {0-0}, next: {0-0}, state: XDES_NOT_INITED, bitmap: 0x0000
   230:   xdes[  2]: seg_id: 0, node: prev: {0-0}, next: {0-0}, state: XDES_NOT_INITED, bitmap: 0x0000
   270:   xdes[  3]: seg_id: 0, node: prev: {0-0}, next: {0-0}, state: XDES_NOT_INITED, bitmap: 0x0000
```

可以发现，初始化时该表空间总共有7个page，其中使用了5个page。第一个extent被加入至fsp_free_frag链表上，且描述该extent的xdes位于page 0的158字节处。由于初始化时只init了一个xdes，因而fsp_free_limit为64，接下来系统运行过程中如果再分配extent，会继续提升该值。但需要说明的是，这里并不代表文件大小已经达到了该值，这两者之间没有必然的联系。

这些都与我们上面分析的*fsp_fill_free_list*逻辑吻合。

#### 为Segment分配空闲EXTENT

```c++
static xdes_t *fseg_alloc_free_extent(fseg_inode_t *inode, space_id_t space,
                                      const page_size_t &page_size,
                                      mtr_t *mtr)
{
  // 如果segment的FSEG_FREE链表不为空
  // 那么直接从该链表分配即可
  if (flst_get_len(inode + FSEG_FREE) > 0) {
    first = flst_get_first(inode + FSEG_FREE, mtr);
    descr = xdes_lst_get_descriptor(space, page_size, first, mtr);
  } else {
    // 如果表空间header的FSEG_FREE链表不为空
    // 首先尝试从FSP的free_frag链表中分配一个frag extent
    descr = fsp_alloc_xdes_free_frag(space, inode, page_size, mtr);
    if (descr != nullptr) {
      return (descr);
    }

    // 如果还是分配不出来,这里就要从FSP中分配了
    // 这里会尝试从fsp_free链表中分配,如果该链表为空
    // 就要调用上面描述的fsp_fill_free_list函数创建新的extent了
    descr = fsp_alloc_free_extent(space, page_size, 0, mtr);

    if (descr == nullptr) {
      return (nullptr);
    }

    seg_id = mach_read_from_8(inode + FSEG_ID);
		// 将新分配的extent插入至segment的FSEG_FREE链表上
    xdes_set_segment_id(descr, seg_id, XDES_FSEG, mtr);
    flst_add_last(inode + FSEG_FREE, descr + XDES_FLST_NODE, mtr);

    fseg_fill_free_list(inode, space, page_size,
                        xdes_get_offset(descr) + FSP_EXTENT_SIZE, mtr);
  }

  return (descr);
}
```

