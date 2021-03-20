### InnoDB REDO LOG BUFFER管理

redo log buffer是用来缓存数据库运行过程中产生的WAL日志，WAL是决定数据库性能最重要的模块，用户事务的提交也是以WAL落盘作为标准，因此，如何

数据库运行时产生的WAL会首先被缓存在REDO LOG BUFFER中，在特定的时机会被从缓存写入至磁盘WAL文件，本文重点阐述InnoDB存储引擎的WAL缓存实现。

### **初始化**

```cpp
bool log_sys_init(uint32_t n_files, uint64_t file_size, space_id_t space_id)
{
    ...
    log_allocate_buffer(log);
    ...
}

void log_allocate_buffer(log_t &log)
{
  // srv_log_buffer_size默认大小为16MB
  log.buf.create(srv_log_buffer_size);
}
```

### **从LOG BUFFER分配空间**

MySQL 8.0中优化了WAL写入log buffer流程，将其分解为两个步骤，以减小锁冲突。

步骤1：为mtr的WAL在log buffer中预留空间

步骤2：将mtr的WAL拷贝至步骤1中预留的缓冲区

我们接下来重点关注下预留空间。

```cpp
Log_handle log_buffer_reserve(log_t &log, size_t len) {
    ...
    const sn_t start_sn = log.sn.fetch_add(len);
    const sn_t end_sn = start_sn + len;
    ...
    handle.start_lsn = log_translate_sn_to_lsn(start_sn);
    handle.end_lsn = log_translate_sn_to_lsn(end_sn);

    if (unlikely(end_sn > log.sn_limit_for_end.load())) {
        log_wait_for_space_after_reserving(log, handle);
    }
}
```

所谓的预留空间其实就是为WAL日志分配lsn：<handle.start_lsn, handle.end_lsn>，这决定了WAL最终写入至W文件的位置。通过引入原子变量log_sys.sn从而减小了临界区的范围，以提升高并发时数据库系统性能。

在分配完lsn后用户线程就可以并行地向缓冲区中对应位置开始拷贝数据(简单地将lsn %srv_log_buffer_size)，因而分配的时候必须克制：要保证接下来一定能正确地将日志写进去。因此，分配时作有限制，如果end_sn超过了当前的sn_limit_for_end，就必须等待，这也是上面*log_wait_for_space_after_reserving*的目的。

我们先看看sn_limit_for_end是如何设置：

```cpp
void log_writer_write_buffer(log_t &log, lsn_t next_write_lsn)
{
  log_files_write_buffer(
      log, buf_begin, buf_end - buf_begin,
      ut_uint64_align_down(last_write_lsn, OS_FILE_LOG_BLOCK_SIZE));

  log_update_limits(log);
}

void log_update_limits(log_t &log) {
    const lsn_t write_lsn = log.write_lsn.load();
    // 将当前write_lsn + log buffer size可以计算出当前可供分配的lsn上限
    // 如果超过该上限，那么必须等该值下一次推进
    // 小于该上限，意味着分配的空间可以立即写入redo log
    const sn_t limit_for_end = log_translate_lsn_to_sn(write_lsn) +
                             log.buf_size_sn.load() -
                             2 * OS_FILE_LOG_BLOCK_SIZE;
    log.sn_limit_for_end.store(limit_for_end);
    ...
}
```

我们看*log_wait_for_space_after_reserving*是如何来控制等待的。为此，我们举个例子说明可能更直观一点。

> 假如redo log buffer大小为100，当前write_lsn为200（即lsn位于200以前的WAL已经全部写入，这之前的redo占据的buffer空间可以被复用），因此可算得limit_for_end为300。假如有如下几个线程来申请lsn：
> 线程A：为其分配的lsn范围为200 ~ 250
> 线程B：为其分配的lsn范围为250 ~ 300
> 线程C：为其分配的lsn范围为300 ~ 450
> 线程D：为其分配的lsn范围为500 ~ 600

- 线程A，由于其end_lsn为250，小于limit_for_end的300，因此，它无需等待，可以立即将它产生的redo log拷贝到log buffer。
- 线程B，同线程A
- 线程C，由于其end_lsn为450，大于limit_for_end的300，因此，它需要等待。而且由于write_lsn最大只能到300（300以后的内容被C占据了还没拷贝呢也就谈不上写了），而且其内容总长度为150，超过了log buffer的容量100，因此，必须将log_buffer进行扩容，假设扩容后的大小为150，在扩容后，我们只需要等write_lsn达到300我们就可以返回了，因为接下来的空间就可以完全容纳本次要写入的redo log
- 线程D，经历过上面的扩容后，只需要write_lsn推进到450就可以安全地返回，因为可以保证本线程的redo log安全写入log buffer了。

```cpp
void log_wait_for_space_after_reserving(log_t &log,
                                        Log_handle &handle) 
{
    const sn_t start_sn = log_translate_lsn_to_sn(handle.start_lsn);
    const sn_t end_sn = log_translate_lsn_to_sn(handle.end_lsn);
    const sn_t len = end_sn - start_sn;

    log_wait_for_space_in_log_buf(log, start_sn);

    // 扩容处理
    if (len > log.buf_size_sn.load()) {
        ...
    }
    log_wait_for_space_in_log_buf(log, end_sn);
}

// 等待end_sn可以被安全写入
// 也即此时write_sn + buf_size >= end_sn
void log_wait_for_space_in_log_buf(log_t &log, sn_t end_sn) {
  const sn_t write_sn = log_translate_lsn_to_sn(log.write_lsn.load());

  const sn_t buf_size_sn = log.buf_size_sn.load();

  lsn = log_translate_sn_to_lsn(end_sn + OS_FILE_LOG_BLOCK_SIZE - buf_size_sn);

  log_write_up_to(log, lsn, false);
}
```