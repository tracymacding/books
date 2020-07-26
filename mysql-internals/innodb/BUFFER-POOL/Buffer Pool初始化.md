### Buffer Pool初始化



#### 创建内存池

```c++
dberr_t srv_start(...) {
    err = buf_pool_init(srv_buf_pool_size, srv_buf_pool_instances);
    ...
}

dberr_t buf_pool_init(ulint total_size, ulint n_instances) {
    ...
    // 并行地创建buffer pool实例
    for (i = 0; i < n_instances; ) {
        for (ulint id = i; id < n; ++id) {
            threads.emplace_back(std::thread(buf_pool_create, &buf_pool_ptr[id], size,
                                       id, &m, std::ref(errs[id])));
        }
    }
}
```

‌innodb的内存池是由多个内存池实例（*buf_pool_t*）所构成。内存池实例数由参数*buffer_pool_instances*控制，而每个实例使用的内存大小则受*buffer_pool_size*控制。

‌buffer pool的内存又被按照chunk为单位进行分割。chunk是一块连续内存，存储多个buffer block。由mem成员指向该连续内存地址，size则是该chunk的总内存大小。chunk大小由参数*buffer_pool_chunk_size*控制，默认大小是128MB。

**chunk创建与初始化**

每个buffer pool实例按照chunk为单位划分，chunk是一段连续内存，接下来介绍chunk初始化：

```c++
void buf_pool_create(buf_pool_t *buf_pool, ulint buf_pool_size,
                     ulint instance_no, std::mutex *mutex,
                     dberr_t &err)
{
    do {
        if (!buf_chunk_init(buf_pool, chunk, chunk_size, mutex)) {
            ...
        } 
        buf_pool->curr_size += chunk->size;
    } while (++chunk < buf_pool->chunks + buf_pool->n_chunks);
    ...
}

buf_chunk_t *buf_chunk_init(...)
{
  buf_block_t *block;
  byte *frame;
  ulint i;

  mem_size = ut_2pow_round(mem_size, UNIV_PAGE_SIZE);
  // 需要加上block管理结构block_t占用的内存大小
  mem_size += ut_2pow_round(
      (mem_size / UNIV_PAGE_SIZE) * (sizeof *block) + (UNIV_PAGE_SIZE - 1),
      UNIV_PAGE_SIZE);

  chunk->mem = buf_pool->allocator.allocate_large(mem_size, &chunk->mem_pfx);

  chunk->blocks = (buf_block_t *)chunk->mem;

  frame = (byte *)ut_align(chunk->mem, UNIV_PAGE_SIZE);
  chunk->size = chunk->mem_pfx.m_size / UNIV_PAGE_SIZE - (frame != chunk->mem);
  {
    ulint size = chunk->size;
    while (frame < (byte *)(chunk->blocks + size)) {
      frame += UNIV_PAGE_SIZE;
      size--;
    }
    chunk->size = size;
  }

  block = chunk->blocks;

  for (i = chunk->size; i--;) {
    buf_block_init(buf_pool, block, frame);
    UT_LIST_ADD_LAST(buf_pool->free, &block->page);
    block++;
    frame += UNIV_PAGE_SIZE;
  }
  // 不知道这里在干什么
  buf_pool_register_chunk(chunk);
  return (chunk);
}
```

‌chunk初始化主要做了一下几件事：

1. 为chunk创建一段连续内存空间，注意分配空间时还需要考虑到每个block的描述对象（*block_t*）占用的内存大小
2. chunk内的内存空间按照block为单位管理，block大小为*UNIV_PAGE_SIZE*，默认大小为16KB。

![img](https://gblobscdn.gitbook.com/assets%2F-LeuGf4juyuq9zjuATOD%2F-LimT2KtcICtch7qhmH5%2F-LimTOr9py-UI2exVqaM%2Fimage.png?alt=media&token=5e55754f-b6a5-4ec7-9cd7-85f86a25e5b6)

最终形成的整个结构：

![img](https://gblobscdn.gitbook.com/assets%2F-LeuGf4juyuq9zjuATOD%2F-LimT2KtcICtch7qhmH5%2F-LimTYzuNcxbke5bPS31%2Fimage.png?alt=media&token=eadfe881-369f-437f-a069-ad1c93431ec4)