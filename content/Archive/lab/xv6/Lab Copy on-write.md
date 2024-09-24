---
date created: 2022-12-17
date modified: 2024-01-27
---

> 元凯文 200110213



## 一、内容分析

当进程fork子进程时，子进程需要另辟一块内存空间，完全复制父进程的用户态内存空间，然而子进程并不都需要使用父进程的数据，尤其是对于fork之后马上执行exec的子进程来说，这样做几乎既浪费物理空间，又不利于性能，本实验的目的就是解决这一痛点，思路类似lazy-allocation，**利用page fault来延迟实际的对物理内存的操作**。具体做法是当进程向一块被多个进程映射的物理空间写入时，抛出page fault，并在此时真正另辟一块内存空间，使得写操作写入新辟的内存空间。

## 二、设计方法

+ page fault的抛出与捕获
  + fork之后抹除所有相关PTE的PTE_W权限，则发起写请求时会引发page fault
  + 在usertrap中根据scause的值识别page fault并捕获
+ 物理空间的free政策
  + Copy on-write使得同一块物理空间被不同进程用户页表映射的情况出现，因此需要在所有映射都移除后才能真正释放物理空间
  + 定义一个专门用于记录reference count情况的rc数组，按照页表序号索引
  + 当kinit初始化内存和kalloc分配内存时初始化rc相应索引的值
  + 当kfree要求释放空间时先自减相应索引的值，再判断满足释放条件
  + 当fork时自增相应索引的值
+ PTE tags的使用
  + PTE的8位和9位为预留位，可以拿来使用
  + 8位用来记录该page是否为COW页
  + 9位用来记录该COW page在fork之前是否具有写权限
+ 注意修改copyout
  + copyout中若引发page fault，则usertrap无法捕捉，因此需要同样在copyout中捕捉page fault

## 三、算法分析

+ usertrap.c
  ```c
  else if (r_scause() == 13 || r_scause() == 15)
    {
      uint64 pa, stval = r_stval();
      pte_t *pte;
      uint flags;
      char *mem;
  
      if (stval >= p->sz)
      {
        goto err;
      }
      if ((pte = walk(p->pagetable, stval, 0)) == 0)
        panic("usertrap: pte should exist");
      if ((*pte & PTE_W) != 0)
      {
        goto err;
      }
      if ((*pte & PTE_C) == 0)
      {
        goto err;
      }
      if ((*pte & PTE_CW) == 0)
      {
        goto err;
      }
      if ((mem = kalloc()) == 0)
      {
        goto err;
      }
      pa = PTE2PA(*pte);
      flags = PTE_FLAGS(*pte);
      memmove(mem, (char *)pa, PGSIZE);
      if (mappages(p->pagetable, PGROUNDDOWN(stval), PGSIZE, (uint64)mem, ((flags | PTE_W) & ~PTE_C) & ~PTE_CW) != 0)
      {
        kfree(mem);
        goto err;
      }
      kfree((char *)pa);
    }
  ```

  + ```c
    else if (r_scause() == 13 || r_scause() == 15)
    ```

    捕获page fault

  + ```c
    if ((*pte & PTE_W) != 0)
        {
          goto err;
        }
        if ((*pte & PTE_C) == 0)
        {
          goto err;
        }
        if ((*pte & PTE_CW) == 0)
        {
          goto err;
        }
    ```

    需同时满足目标page不能写且是COW page且在fork之前具有写权限，才进行之后的操作

  + ```c
    memmove(mem, (char *)pa, PGSIZE);
    ```

    真正复制物理内存空间

  + ```c
    mappages(p->pagetable, PGROUNDDOWN(stval), PGSIZE, (uint64)mem, ((flags | PTE_W) & ~PTE_C) & ~PTE_CW)
    ```

    覆写映射到新分配的内存空间，注意对tags的修改，恢复了写权限，抹除了COW page相关标记

  + ```c
    kfree((char *)pa);
    ```

    请求释放原先的物理空间，究竟原先的物理空间还有没有映射，能不能直接释放，交给kfree去判断

  + copyout中的代码与之类似

+ kalloc.c
  ```c
  int rc[33000];
  
  void freerange(void *pa_start, void *pa_end)
  {
    char *p;
    p = (char *)PGROUNDUP((uint64)pa_start);
    for (; p + PGSIZE <= (char *)pa_end; p += PGSIZE)
    {
      rc[PA2INDEX((uint64)p)] = 1; // init the reference count
      kfree(p);
    }
  }
  
  void kfree(void *pa)
  {
    rc[PA2INDEX((uint64)pa)]--;
    if (rc[PA2INDEX((uint64)pa)] == 0)
    {
      struct run *r;
  
      if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP)
        panic("kfree");
  
      // Fill with junk to catch dangling refs.
      memset(pa, 1, PGSIZE);
  
      r = (struct run *)pa;
  
      acquire(&kmem.lock);
      r->next = kmem.freelist;
      kmem.freelist = r;
      release(&kmem.lock);
    }
  }
  
  void *
  kalloc(void)
  {
    struct run *r;
  
    acquire(&kmem.lock);
    r = kmem.freelist;
    if (r)
    {
      kmem.freelist = r->next;
      rc[PA2INDEX((uint64)r)] = 1; // init the reference count
    }
    release(&kmem.lock);
  
    if (r)
      memset((char *)r, 5, PGSIZE); // fill with junk
    return (void *)r;
  }
  ```

  + 其中PA2INDEX将物理地址转换成相应在rc数组中的索引，定义在riscv.h中：

    ```c
    #define PA2INDEX(x) ((x >> 12) - 0x80046)
    ```

    0x80046000为用户RAM的起始地址

  + 为使初始化时能够顺利free，需要在freerange中预先初始化rc相应索引的值

+ vm.c

  + ```c
    int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
    {
      pte_t *pte;
      uint64 pa, i;
      uint flags;
      // char *mem;
    
      for (i = 0; i < sz; i += PGSIZE)
      {
        if ((pte = walk(old, i, 0)) == 0)
          panic("uvmcopy: pte should exist");
        if ((*pte & PTE_V) == 0)
          panic("uvmcopy: page not present");
        if ((*pte & PTE_W) != 0)
          *pte = *pte | PTE_CW;
        pa = PTE2PA(*pte);
        flags = PTE_FLAGS(*pte);
        // if((mem = kalloc()) == 0)
        //   goto err;
        // memmove(mem, (char*)pa, PGSIZE);
        if (mappages(new, i, PGSIZE, (uint64)pa, (flags & (~PTE_W)) | PTE_C) != 0)
        {
          // kfree(mem);
          goto err;
        }
        rc[PA2INDEX((uint64)pa)]++;
        *pte = (*pte & ~PTE_W) | PTE_C;
      }
      return 0;
    
    err:
      uvmunmap(new, 0, i / PGSIZE, 1);
      return -1;
    }
    ```

    + 注意注释掉的几行，使得fork只是将新的pagetable映射到旧有的物理空间上
    + 注意在成功添加映射后需要对rc中相应索引进行自增

  + ```c
    int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
    {
      uint64 n, va0, pa0;
      pte_t *pte;
      uint flags;
      char *mem;
    
      while (len > 0)
      {
        va0 = PGROUNDDOWN(dstva);
        pa0 = walkaddr(pagetable, va0);
        if (pa0 == 0)
          return -1;
        n = PGSIZE - (dstva - va0);
        if (n > len)
          n = len;
        if ((pte = walk(pagetable, va0, 0)) == 0)
          panic("copyout: pte should exist");
        if ((*pte & PTE_W) == 0)
        {
          if ((*pte & PTE_C) == 0)
            goto err;
          if ((*pte & PTE_CW) == 0)
            goto err;
          if ((mem = kalloc()) == 0)
            goto err;
          flags = PTE_FLAGS(*pte);
          memmove(mem, (char *)pa0, PGSIZE);
          if (mappages(pagetable, PGROUNDDOWN(va0), PGSIZE, (uint64)mem, ((flags | PTE_W) & ~PTE_C) & ~PTE_CW) != 0)
          {
            kfree(mem);
            goto err;
          }
          memmove((void *)(mem + (dstva - va0)), src, n);
          kfree((char *)pa0);
        }
        else
        {
          memmove((void *)(pa0 + (dstva - va0)), src, n);
        }
    
        len -= n;
        src += n;
        dstva = va0 + PGSIZE;
      }
      return 0;
    
    err:
      printf("copyout fail\n");
      return -1;
    }
    ```

    基本与usertrap中的处理类似

