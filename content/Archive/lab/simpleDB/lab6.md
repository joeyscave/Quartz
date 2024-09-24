---
date created: 2023-03-02
date modified: 2024-01-27
---
[MIT 6.830 Lab6 Rollback And Recovery - OneStep (waruto.top)](https://waruto.top/posts/mit-6.830-lab6-rollback-and-recovery/)
+ 注意点

日志先于缓存写回磁盘

日志写缓存 和 缓存在flushPage之后写回日志可能造成死锁 因此要先拿到日志写缓存时需要先拿到缓冲区锁 再锁自己

```java
synchronized (Database.getBufferPool()) {
   synchronized (this) {

   ..

   }
}
```

+ 背景：在lab4中实现的缓冲区策略其实是 force + no steal，不需要日志，这个实验中为了测试日志的有效性，在测试代码中用提前调用flushpages（写回磁盘）等来 tricky 地实现 no force + steal 

+ 日志总体的恢复(recover)策略是

1. 从前往后扫描，且只扫描一次
2. checkpoint会记录当前活跃（在checkpoint之前begin但还没有commit/abort）的事务和脏页，而LogFile.logCheckpoint()会把当前所有脏页写回磁盘，这意味这从当前活跃事务begin到checkpoint这一段都已经被写入磁盘，因此可以放心地从checkpoint开始恢复
3. 所有的事务简单分成两种：有commt/abort的 和 没有的，对前者采取的策略就是redo（abort的redo就是中间所有的update都redo，一直碰到abort就undo整个事务，就像没有crash时abort的事务的整个流程一样；commit的redo就是简单的redo中间所有update）；对于后者的策略就是undo
4. **为什么只需要扫描一次？**维护一个事务集合，在扫描的过程中来收集上述没有commit/abort的事务，在扫描结束后，undo这些事务。具体的算法是先把checkpoint的所有当前活跃事务加入集合，再在扫描的过程中遇到begin把此事务加入集合，如果遇到commit/abort就把此事务移出集合，所有扫描完剩下的就是没有commit/abort的了