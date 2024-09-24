---
标题: "MIT 6.830 Lab4 SimpleDB Transactions - OneStep"
笔记ID: H83DH22
笔记类型: page
星标: false
tags: 
域名: waruto.top
域名2: waruto.top
作者: ""
原文链接: "https://waruto.top/posts/mit-6.830-lab4-simpledb-transactions/"
五彩链接: "https://marker.dotalk.cn/#/?noteidx=H83DH22"
划线数量: 2
创建时间: 2023-05-05 00:20
更新时间: 2023-05-05 00:26
---

## Notes


## Highlights
> 而strict-2PL，则要求transaction在执行结束（commit/abort）时统一释放所有锁。这样避免了一个transaction abort后，其他transaction级联abort
> #notes 可恢复性：依赖的事务比当前事务先提交
> 无级联性：读之前确保依赖的事务已经提交
> 注意2PL指的是单个事务内要区分加锁段和解锁段，和其他事务无关，因此strict-2PL就是3级封锁协议

> Strict 2PL保证隔离性
> #notes 严格的两阶段锁是可串行的

