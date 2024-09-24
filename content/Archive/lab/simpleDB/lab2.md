---
publish: false
date created: 2022-12-22
date modified: 2023-12-30
---
### 一、概述
[实验说明](material/lab2.md)

这个实验主要实现了insert/delete/join/group等常用sql操作



### 二、注意点

1. Join 中 fetchNext 原先的写法：

```java
        while (child1.hasNext()) {
            Tuple t1 = child1.next();
            while (child2.hasNext()) {
                Tuple t2 = child2.next();
                if (p.filter(t1, t2)) {
                    Tuple t = new Tuple(getTupleDesc());
                    int i = 0;
                    for (; i < child1.getTupleDesc().numFields(); i++) {
                        t.setField(i, t1.getField(i));
                    }
                    for (; i < getTupleDesc().numFields(); i++) {
                        t.setField(i, t2.getField(i - child1.getTupleDesc().numFields()));
                    }
                    return t;
                }
            }
            child2.rewind();
        }
        return null;
```

按照以上代码，在 return t 提前返回后，下一次进入该函数，则上次保存的 t1 信息已经丢失，因此需要将 t1,t2 保存为全局变量的形式，并对函数进行相应的改写

2. *TODO*：join 采用性能更好的算法

3. IntergerAggregator 之于 Aggregate 就像 Predicate 之于 filter，只不过后者无需在内部存储 tuple，而前者存储

4. *TODO*：evict page 探究其他算法（目前使用纯粹的LRU）
