### INNODB page heap管理

本文主要描述Innodb内物理空间管理算法。

INNODB页面内空间以HEAP描述，HEAP按照由低向高的方式增长，Index页面头部的PAGE_HEAP_TOP字段记录了heap当前的位置。

#### heap分配

```c++
// 计算page内除header/trailer/infimum rec/supremum rec/infimum slot/supremum slot之外的可用空间
ulint page_get_free_space_of_empty(ulint comp)
{
  if (comp) {
    // page总大小为UNIV_PAGE_SIZE
    // PAGE_NEW_SUPREMUM_END包含了header/infimum rec/supremum rec
    // PAGE_DIR为trailer大小(8B)
    // 2 * PAGE_DIR_SLOT_SIZE为infimum slot/supremum slot总大小
    return ((ulint)(UNIV_PAGE_SIZE - PAGE_NEW_SUPREMUM_END - PAGE_DIR -
                    2 * PAGE_DIR_SLOT_SIZE));
  }
  return ((ulint)(UNIV_PAGE_SIZE - PAGE_OLD_SUPREMUM_END - PAGE_DIR -
                  2 * PAGE_DIR_SLOT_SIZE));
}

ulint page_get_max_insert_size(const page_t *page, ulint n_recs)
{
  if (page_is_comp(page)) {
    // page内用户记录已使用的空间
    occupied =
        page_header_get_field(page, PAGE_HEAP_TOP) - PAGE_NEW_SUPREMUM_END +
        page_dir_calc_reserved_space(n_recs + page_dir_get_n_heap(page) - 2);
    free_space = page_get_free_space_of_empty(TRUE);
  } else {
    ...
  }

  if (occupied > free_space) {
    return (0);
  }
  return (free_space - occupied);
}
    
byte *page_mem_alloc_heap(...)
{
  avl_space = page_get_max_insert_size(page, 1);

  if (avl_space >= need) {
    block = page_header_get_ptr(page, PAGE_HEAP_TOP);
    // 更新PAGE_HEAP_TOP
    page_header_set_ptr(page, page_zip, PAGE_HEAP_TOP, block + need);
    *heap_no = page_dir_get_n_heap(page);
    // 更新page header中的heap_no,还是不大清楚heap_no的作用
    page_dir_set_n_heap(page, page_zip, 1 + *heap_no);
    return (block);
  }
  return (NULL);
}
```

Index page头部的PAGE_HEAP_TOP字段记录了heap的当前位置，下次从heap中分配空间时直接从该位置处开始。需要说明的是：PAGE_HEAP_TOP包含了被删除的用户记录。发现还有空间存储用户行记录时，将该位置返回并向前推进PAGE_HEAP_TOP位置。

### **PAGE_FREE record链表**

Innodb使用链表串联起所有被删除的行记录。例如，在前面的页面分裂实现文章中提到，分裂时会删除新/老页面的部分记录，此时会更新PAGE_FREE链表。：

```c++
// 从页面中向后批量删除行记录时会将其插入至PAGE_FREE链表
void page_delete_rec_list_end(...)
{
  ...
  prev_rec = page_rec_get_prev(rec);
  // 重设prev_rec的next record
  page_rec_set_next(prev_rec, page_get_supremum_rec(page));

  last_rec = page_rec_get_prev(page_get_supremum_rec(page));
  page_header_set_field(page, NULL, PAGE_GARBAGE,
                        size + page_header_get_field(page, PAGE_GARBAGE));
  ...
}

// 释放单个行记录时也会将其插入至PAGE_FREE链表
// 同时在调用者中会建立该rec的prev_rec和next_rec的关系
void page_mem_free(...)      
{
  free = page_header_get_ptr(page, PAGE_FREE);
  page_rec_set_next(rec, free);
  page_header_set_ptr(page, page_zip, PAGE_FREE, rec);

  garbage = page_header_get_field(page, PAGE_GARBAGE);

  page_header_set_field(page, page_zip, PAGE_GARBAGE,
                        garbage + rec_offs_size(offsets));
}
```

在这里删除单条或者一批行记录时会将这些记录从原有的页面记录链表中摘除，同时加入至PAGE_FREE链表。被加入至PAGE_FREE链表的行记录可以在下次分配时复用。

```c++
rec_t *page_cur_insert_rec_low(...)
{
  free_rec = page_header_get_ptr(page, PAGE_FREE);
  if (free_rec) {
    ...
    insert_buf = free_rec - rec_offs_extra_size(foffsets);
    if (page_is_comp(page)) {
      heap_no = rec_get_heap_no_new(free_rec);
      // 将被分配出去的rec从PAGE_FREE链表摘除
      page_mem_alloc_free(page, NULL, rec_get_next_ptr(free_rec, TRUE), rec_size);
    }
    ...
  }    
}
```

### **页面整理**

```c++
bool btr_page_reorganize_low(...)
{
  buf_block_t *block = page_cur_get_block(cursor);

  data_size1 = page_get_data_size(page);
  max_ins_size1 = page_get_max_insert_size_after_reorganize(page, 1);

  temp_block = buf_block_alloc(buf_pool);
  temp_page = temp_block->frame;

  // 首先将原page内容拷贝至临时page
  buf_frame_copy(temp_page, page);

  /* Save the cursor position. */
  pos = page_rec_get_n_recs_before(page_cur_get_rec(cursor));

  // 重新初始化原page
  page_create(block, mtr, dict_table_is_comp(index->table),
              fil_page_get_type(page));

  // 将临时page内的行记录逐个拷贝至重新初始化后的page
  page_copy_rec_list_end_no_locks(block, temp_block,
                                  page_get_infimum_rec(temp_page), index, mtr);
  ...
}
```

页面整理的目的是将那些被删除的行记录真正地从物理页中抹除，为此：

1. 首先分配一个临时page，并将原page的内容物理拷贝至其中
2. 重新初始化原page
3. 将临时page内的行记录逐个拷贝至2中重新初始化后的page

在不再3的拷贝中，页面中有效的行记录都被串联起来，而已被删除的行记录则是串联在PAGE_FREE链表上，因而直接通过有效行记录的遍历即可将所有有效行记录提取出来，这便是page reorganize。