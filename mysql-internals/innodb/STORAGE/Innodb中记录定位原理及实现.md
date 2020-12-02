### Innodb中记录定位原理及实现

在innodb中，记录定位是最基本的操作，无论是普通查询、插入或者是删除，都需要先找到目标行记录的位置。

因此高效的定位方法是数据库性能的基本保证。Innodb中使用一个叫做page_cur_t结构体进行存储，称之为

cursor游标。

```c++
struct page_cur_t {
  const dict_index_t* index;
  rec_t*    rec;    /*!< pointer to a record on page */
  ulint    offsets;
  buf_block_t* block;   /*!< pointer to the block containing rec */
};
```

其中包含了本index的数据字典类容、实际的数据、记录所在block等信息，本文结合源代码探讨定位方法。

我们先来明确一下概念：

> * 记录(rec)：通常为存储在内存中行记录的完全拷贝，通常用一个unsigned char* 指针指向整个记录
> * 元组(dtuple)：物理记录的逻辑体现，他就复杂得多，但是一个记录(rec)对应一个元组(dtuple),由dtuple_t结构体表示，其中每一个field由一个dfield_t结构体表示，数据存储在dfied_t的一个叫做void* data的指针中

**查询模式**

在innodb中的常用的search mode有如下几个

```c++
enum page_cur_mode_t {
  PAGE_CUR_UNSUPP = 0,
  // 大于
  PAGE_CUR_G = 1,
  // 大于等于,点查询时会走该模式
  PAGE_CUR_GE = 2,
  // 小于
  PAGE_CUR_L = 3,
  // 小于等于,插入新记录时会走该模式
  PAGE_CUR_LE = 4,
};
```

**matched_fields和matched_bytes**

大家在源码中能看到matched_fields和matched_bytes两个值，那么他们代表什么意思呢？

以int类型为例，因为在函数cmp_dtuple_rec_with_match_bytes是逐个字段逐个字节进行比较的，关键代码如下

```c++
while (cur_field < n_cmp) {
  rec_byte = *rec_b_ptr++;
  dtuple_byte = *dtuple_b_ptr++;
}
```

比如int 2、int 3在innodb中内部表示为0x80000002和0x80000003，如果他们进行比较那么最终此field的比较为

不相等(-1)，那么matched_fields=0 但是matched_bytes=3，因为
• 0x 800000 02
• 0x 800000 03

我们能够发现其中有3个字节是相同的及[0x80、0x00、0x00] 。简单来说matched_fields为相同field数量。如果

field不相同则会返回相同的字节数matched_bytes。当然cmp_dtuple_rec_with_match_bytes对不同数据类型的

比较方式也不相同，具体可以参考一下源码，这里不再过多解释。

#### 页面内定位行记录

在页面内查找的原理是先二分查找确定记录所在的slot，然后在slot内部使用类似二分查找的方法定位记录。
定位directory slot

```c++
void page_cur_search_with_match(...)
{
  low = 0;
  up = page_dir_get_n_slots(page) - 1;

  // 结束条件为up slot - low slot小于等于1
  // 即目标rec一定位于low和up slot之间
  while (up - low > 1) {
    mid = (low + up) / 2;
    slot = page_dir_get_nth_slot(page, mid);
    // mid_rec是mid slot内的第一个rec(也是值最小的rec)
    mid_rec = page_dir_slot_get_rec(slot);
    ...
    cmp = cmp_dtuple_rec_with_match(tuple, mid_rec, index, offsets,
                    &cur_matched_fields);
    // 如果tuple > mid_rec,提升low
    if (cmp > 0) {
low_slot_match:
      low = mid;
      low_matched_fields = cur_matched_fields;
    } else if (cmp) {
      // tuple < mid_rec,降低up
up_slot_match:
      up = mid;
      up_matched_fields = cur_matched_fields;
    } else if (mode == PAGE_CUR_G || mode == PAGE_CUR_LE) {
      // 如果查找模式是大于或者小于等于
      goto low_slot_match;
    } else {
      // 如果查找模式是小于或者大于等于
      goto up_slot_match;
    }
}
```

我们以一个简单例子说明slot定位过程，假如表中有以下记录：

```sql
mysql> select * from t4;
+----+
| a |
+----+
| 1 |
| 2 |
| 3 |
| 4 |
| 5 |
| 6 |
| 7 |
| 8 |
| 9 |
| 10 |
| 11 |
| 12 |
| 13 |
| 14 |
| 15 |
| 16 |
| 17 |
| 18 |
| 19 |
| 20 |
+----+
20 rows in set (0.00 sec)
```

我们现在要执行如下查询：

```sql
select * from t4 where a = 6;
```

根据我们的解析工具分析当前page的记录情况：

```c++
$./innodb_page_parser -file=data/test/t4.ibd -page 4
**** FILE_PAGE_HEADER ****
  0:  page_crc   : 0x7a4957e2
  4:  page_no    : 4
  8:  page_version : 4294967295
 12:  space_version : 4294967295
 16:  page_lsn   : 1373190210
 24:  page_type   : FIL_PAGE_INDEX
 26:  page_flush_lsn: 0
 34:  space_id   : 6
** ** FILE_PAGE_HEADER ****
 42:  page_format : COMPACT
 38:  n_dir_slots : 6
 40:  heap_top   : 560
 42:  heap_number : 0x8016
 44:  free     : 0
 46:  deleted_bytes: 0
 48:  last_insert : 543
 50:  direction  : PAGE_NO_DIRECTION
 52:  n_direction : 0
 54:  n_records  : 20
 56:  max_trx_id  : 0
 64:  level    : 0
 66:  index_id   : 150
 74:  btr_seg_leaf : Space: 6, Page: 2, Offset: 626
 84:  btr_seg_top : Space: 6, Page: 2, Offset: 434
slots[0]: 112
slots[1]: 455
slots[2]: 367
slots[3]: 279
slots[4]: 191
slots[5]: 99

records[0]: n_owned: 1, heap_no: 0, rec_type: REC_INFIMUM, next_record: 26, value:
records[1]: n_owned: 0, heap_no: 16, rec_type: REC_DATA, next_record: 22, value:
records[2]: n_owned: 0, heap_no: 16, rec_type: REC_DATA, next_record: 22, value:
records[3]: n_owned: 0, heap_no: 32, rec_type: REC_DATA, next_record: 22, value:
records[4]: n_owned: 4, heap_no: 32, rec_type: REC_DATA, next_record: 22, value:
records[5]: n_owned: 0, heap_no: 48, rec_type: REC_DATA, next_record: 22, value:
records[6]: n_owned: 0, heap_no: 48, rec_type: REC_DATA, next_record: 22, value:
records[7]: n_owned: 0, heap_no: 64, rec_type: REC_DATA, next_record: 22, value:
records[8]: n_owned: 4, heap_no: 64, rec_type: REC_DATA, next_record: 22, value:
records[9]: n_owned: 0, heap_no: 80, rec_type: REC_DATA, next_record: 22, value:
records[10]: n_owned: 0, heap_no: 80, rec_type: REC_DATA, next_record: 22, value:
records[11]: n_owned: 0, heap_no: 96, rec_type: REC_DATA, next_record: 22, value:
records[12]: n_owned: 4, heap_no: 96, rec_type: REC_DATA, next_record: 22, value:
records[13]: n_owned: 0, heap_no: 112, rec_type: REC_DATA, next_record: 22, value:
records[14]: n_owned: 0, heap_no: 112, rec_type: REC_DATA, next_record: 22, value:
records[15]: n_owned: 0, heap_no: 128, rec_type: REC_DATA, next_record: 22, value:
records[16]: n_owned: 4, heap_no: 128, rec_type: REC_DATA, next_record: 22, value:
records[17]: n_owned: 0, heap_no: 144, rec_type: REC_DATA, next_record: 22, value:
records[18]: n_owned: 0, heap_no: 144, rec_type: REC_DATA, next_record: 44, value:
records[19]: n_owned: 0, heap_no: 160, rec_type: REC_DATA, next_record: 65514, value:
records[20]: n_owned: 0, heap_no: 160, rec_type: REC_DATA, next_record: 65127, value:
records[21]: n_owned: 5, heap_no: 1, rec_type: REC_SUPREMUM, next_record: 0, value:

```

可以看到，当前页面有6个slot，22条记录（包括infimum和supremum以及20条用户行记录）。进入该函数时：

```
(gdb) p/x (int32_t)tuple->fields[0].data
$49 = 0x6000080

(gdb) p mode
$53 = PAGE_CUR_GE

(gdb) p low
50 = 0
(gdb) p up
51 = 5
```

tuple内的数据确实是6，且对于点查，其模式为PAGE_CUR_GE。

第一次，计算得到的mid slot为2，该slot管辖的第一个record值为8，如下：

```
(gdb) p/x (int32_t)mid_rec
$54 = 0x8000080
```

因而下调up slot的值为2，继续下一轮计算，得到mid slot管辖的第一个record值为4：

```
(gdb) p mid
59 = 1
(gdb) p/x *(int32_t*)mid_rec
60 = 0x4000080
```

由于mid_rec值小于tuple内容，于是提升low slot值为1，此时low = 1，up = 2。于是便定位出了目标rec所属的slot范围，跳出循环。本例中low_rec和up_rec分别为:

```
(gdb) p/x (int32_t)up_rec
14 = 0x8000080

(gdb) p/x *(int32_t*)low_rec
15 = 0x4000080
```

接下来便是在low_rec和up_rec之间查找目标record。

#### 定位目标record

```c++
void page_cur_search_with_match(...)
{
  ...
  // 在low_rec和up_rec之间寻找目标record
 	slot = page_dir_get_nth_slot(page, low);
 	low_rec = page_dir_slot_get_rec(slot);
 	slot = page_dir_get_nth_slot(page, up);
 	up_rec = page_dir_slot_get_rec(slot);

 	// 其实是从low_rec开始做了个顺序搜索
 	while (page_rec_get_next_const(low_rec) != up_rec) {
    mid_rec = page_rec_get_next_const(low_rec);
 	  ...
    cmp = cmp_dtuple_rec_with_match(tuple, mid_rec, index, offsets,
                  &cur_matched_fields);
	  // 如果tuple比mid_rec更大
    // 继续往前下一个搜索
    if (cmp > 0) {
    low_rec_match:
   	  low_rec = mid_rec;
   	  low_matched_fields = cur_matched_fields;
 	  } else if (cmp) {
   	  // 如果tuple比mid_rec更小,那就缩小上界
up_rec_match:
   	  up_rec = mid_rec;
   	  up_matched_fields = cur_matched_fields;
 	  } else if (mode == PAGE_CUR_G || mode == PAGE_CUR_LE) {
  	  ...
   	  // 如果相等且模式是小于等于或者大于
   	  goto low_rec_match;
 	  } else {
   	  // 如果tuple和mid_rec相等且模式是小于或者大于等于
   	  goto up_rec_match;
 	  }
  }

 	// 到这里
 	// 如果查找模式是大于或者大于等于,那么设置游标cursor为up_rec
 	if (mode <= PAGE_CUR_GE) {
    page_cur_position(up_rec, block, cursor);
  } else {
    // 查找模式是小于或者小于等于,设置游标cursor为low_rec
    page_cur_position(low_rec, block, cursor);
  }
  ...
}
```

1. mid_rec第一次为5，小于目标值6，于是提升low_rec为mid_rec
2. mid_rec第二次为6，与目标值相等，而且此时查找模式是PAGE_CUR_GE（要查询的是大于等于6的记录），我们虽然找到了6，但不能确定其就是第一个key为6的记录，还需要再向下找。设置up_rec为mid_rec，继续向下寻找（因为是要找大于等于6，mid_rec已经为6了，说明第一个等于6的记录只可能小于等于mid_rec位置）。此时low_rec为5，up_rec为6，满足了跳出循环的条件：next(low_rec) == up_rec。

跳出循环，由于模式是大于等于，于是设置游标在up_rec处，即上面步骤2中的记录6。至于其他的情况，感兴趣

的朋友可以按照这里的思路自行推演，难度不大。

#### 查询优化

由于每次插入前，都需要调用上述函数确定插入位置，为了提高效率，InnoDB针对按照主键顺序插入的场景做了

一个小优化。针对该场景，只需要直接定位在数据页的最后(PAGE_LAST_INSERT)就可以了。至于怎么判断当前是

否按照主键顺序插入，就依赖页面头部的PAGE_N_DIRECTION，PAGE_LAST_INSERT，PAGE_DIRECTION这几个

信息了，目前的代码中要求满足5个条件：

> * 当前的数据页是叶子节点
> * 位置查询模式为PAGE_CUR_LE(插入新记录时走该查询模式)
> * 相同方向的插入已经大于3了(page_header_get_field(page, PAGE_N_DIRECTION) > 3)
> * 最后插入的记录的偏移量为空(page_header_get_ptr(page, PAGE_LAST_INSERT) != 0)
> * 从右边插入的(page_header_get_field(page, PAGE_DIRECTION) == PAGE_RIGHT)

```c++
page_cur_try_search_shortcut
void page_cur_search_with_match(...)
{
  ...
 	if (page_is_leaf(page) && (mode == PAGE_CUR_LE) &&
   	  !dict_index_is_spatial(index) &&
  	  (page_header_get_field(page, PAGE_N_DIRECTION) > 3) &&
  	  (page_header_get_ptr(page, PAGE_LAST_INSERT)) &&
  		(page_header_get_field(page, PAGE_DIRECTION) == PAGE_RIGHT))
  {
    // 优化路径
    if (page_cur_try_search_shortcut(block, index, tuple, iup_matched_fields,
                    ilow_matched_fields, cursor)) {
      return;
	  }
  }
  ...
}

bool page_cur_try_search_shortcut(...)
{
 	ibool success = FALSE;
  ...

 	// 找到PAGE_LAST_INSERT记录
  rec = page_header_get_ptr(page, PAGE_LAST_INSERT);
 	offsets =
    rec_get_offsets(rec, index, offsets, dtuple_get_n_fields(tuple), &heap);

 	low_match = up_match = std::min(*ilow_matched_fields, *iup_matched_fields);

 	// 如果tuplePAGE_LAST_INSERT记录值,说明此次非按主键顺序插入
 	// 返回false,老老实实执行原有查找
 	if (cmp_dtuple_rec_with_match(tuple, rec, index, offsets, &low_match) < 0) {
    goto exit_func;
  }

 	// 找到PAGE_LAST_INSERT在page中的next record
 	// 如果next record并非是supremum记录且tuple比它更大
 	// 那说明在PAGE_LAST_INSERT之后发生了非主键顺序插入
 	// 此时也无法执行快速定位
 	next_rec = page_rec_get_next_const(rec);
 	if (!page_rec_is_supremum(next_rec)) {
 	  ...
    if (cmp_dtuple_rec_with_match(tuple, next_rec, index, offsets, &up_match) >=
      0) {
   	  goto exit_func;
 	  }
  }
 	// 到这里说明快速定位是成功的,设置游标cursor
 	page_cur_position(rec, block, cursor);
 	success = TRUE;

exit_func:
  return (success);
}
```



