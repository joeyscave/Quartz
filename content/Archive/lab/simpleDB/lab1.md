---
publish: false
date created: 2022-12-17
date modified: 2023-12-30
---
### 一、概述
[实验说明](material/lab1.md)
#### 1. 实验目的
这个实验的主要目的是熟悉 simpleDB 的整个存储结构以及完整测试流程。
#### 2. 代码心得
熟悉了迭代器的构造与使用。
### 二、 simpleDB 的存储结构

SimpleDB 主要包含下列 class：

- 表示 field，tuple，和 tuple schema 的类
- 将谓词和筛选条件应用到 tuples 上的类
- 一个或多个将 relation 保存在 disk 上的方法，并提供迭代 relation 包含的 tuples 的方法
- 一系列处理 tuples 的 operator（如 select，join，insert，delete 等）
- buffer pool，用于将活跃的 page 换存在内存中，并负责处理并发控制和事务
- catalog，保存 tables 和它们 schema 信息
![](../../z-source/simpledb-structure.png)

1. catalog 相当于数据库的元信息（数据字典），每个数据库持有一个，可以通过 Database.getCatalog 静态方法获取当前数据库的 catalog

2. 在 simpleDB 里一个 table 对应一个 file，表和文件是一对一的，但这不是必须的，完全可以多表存在一个文件里，或者一个文件存多表

3. Seqscan（一个 operator）在使用数据库时不直接与 bufferPool 打交道，而是与 file 打交道，从 file 中拿出表的迭代器，而 file 内部在构造迭代器时取 page 优先从 bufferPool 中，而不是从 disk 中，通过这种方法来使用 bufferPool 这个缓冲区
   ```java
   private void nextTupleIter() throws TransactionAbortedException, DbException {
       tupleIter= ((HeapPage)Database.getBufferPool().getPage(transactionId,new HeapPageId(tableId,pageCursor), Permissions.READ_ONLY)).iterator();
   }
   ```

4. table、page、record（tuple）都有属于自己唯一的标识，其中 tableID 的唯一性是依靠 table 对应文件互不相同的绝对路径的 hashcode 构造的，而 pageID 包含 tableID，recordID 包含 pageID，因此即便每一个 tuple 也有唯一的 id

### 三、完整测试流程

 [完整测试流程](material/lab1.md#^4dad80)
