### UNDO LOG综述

innodb事务运行时产生的日志包括redo和undo。redo log是重做日志，提供前滚操作，undo log是回滚日志，提供回滚操作。undo log不是redo log的逆向过程，其实它们都算是用来恢复的日志：

- redo log是物理日志，记录的是数据页的修改，该记录具备幂等性：即多次将同一个物理日志应用到特定物理页上其状态依然正确。而UNDO LOG则是记录逻辑日志，记录的是行的某个版本，类似SNAPSHOT。
- REDO LOG按照RECORD存储在WAL文件中，采用顺序追加写以实现最佳性能。以后我们会知道，只要事务的REDO LOG落盘，事务便可以视为提交成功并返回。而UNDO LOG存储与普通数据页一致，在INNODB中也使用段页式存储方式，因而UNDO LOG的写入也是按照PAGE为单位进行，无法做到顺序写入。
- UNDO LOG也作为普通页面，对于UNDO LOG PAGE的修改也同样会记录REDO LOG。
- UNDO LOG主要用来实现事务回滚以及MVCC功能

‌

本章节会按照如下目录组织：‌

- **UNDO LOG物理格式**：在该章节中我们会详细描述UNDO LOG SEGMENT、UNDO LOG PAGE、UNDO LOG RECORD的具体组织
- **UNDO LOG创建**：在该章节中我会从mysql initialize阶段开始，梳理UNDO LOG文件是如何被创建、mysql启动时是如何被加载等流程。同时，结合源代码厘清UNDO LOG在内存中的映像
- **UNDO LOG与事务**：该章节会将事务与UNDO LOG结合，重点描述事务在运行过程中如何记录、修改UNDO LOG。按照事务运行过程分阶段描述，包括PREPARE、COMMIT、ROLLBACK等
- **UNDO LOG与MVCC**：细致描述UNDO LOG如何实现MVCC
- **UNDO LOG清理**：描述INNODB中如何实现无用UNDO LOG记录的清理
- **UNDO LOG与奔溃恢复**：描述UNDO LOG在奔溃恢复中扮演的作用