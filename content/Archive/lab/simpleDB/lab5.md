---
date created: 2023-02-28
date modified: 2024-01-27
---
[MIT 6.830 Lab5 SimpleDB BTree Index - OneStep (waruto.top)](https://waruto.top/posts/mit-6.830-lab5-simpledb-btree-index/)
dirtypages 另一个缓冲区？存在哪个类里？

与课本不同的是，叶结点直接存放page，而不是指针

getParentWithEmptySlots达成了递归的效果，子结点在申请有空位的父结点时，父结点可能也在分裂

分支结点的分裂的过程不发生在插入中，而是发生在子结点申请具有空位的父结点时，因此保证了当子结点拿到父结点插入时，父结点已经分裂完具有空位了

b+树没有什么固定的标准 大概要做到的就是自调整项的比例和树的高度 分支结点中的项是否允许重复，相等的项只允许放在一侧还是两侧都可以

指针指向的不是页中的entry，而是一整页，因此内部结点分裂出新的页后要更新其所有子结点的指向

internal结点中的entry值确实不重复，但并非类似教科书里把插入值直接推到根结点，而是把分裂之后的中间值推上去（删除在原层级中的entry）,接下来子结点就可以在这层插入新值了

stealFromRightInternalPage过程的几个关键问题：

父：	|ABC| （其中B左右指针指向两个子结点）

子：|D_ _| |GHI|

1. G的左指针和D的右指针指向不一致，不能直接把G往左移，怎么办？

   > 让B的左右指针分别指向G的左指针和D的右指针，然后把B移入左侧子结点，这样左右就连起来了，右侧结点可以一直往左移，这样做会导致任意移完后的左子结点的最右指针和右子结点的最左指针指向同一个结点，于是正好利用此，把右侧子结点的最左entry删掉，然后把原来B位置的entry改成它的值

2. merge分支结点的过程也要经过类似的旋转操作

参考资料：
[MIT 6.830 Lab6 Rollback and Recovery - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/505850419)
