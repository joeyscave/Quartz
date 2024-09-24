
> 元凯文 200110213



## 一、内容分析

mmap想要做的事说起来很简单，就是把文件内容映射到虚拟内存上，让用户像操作内存空间那样便利地操作文件内容，无论是读还是写。但mmap实验对我来说是xv6系列实验中最难的一个，不同于pgtbl实验漫长的debug过程带来的难，mmap对我来说难在概念的理解上，我对此还是有一些悬而未解的问题：mmap究竟有多大意义？mmap如何应对并发？mmap如何与原本的用户空间和平共处，避免在虚拟地址中到处挖洞？先后去看了6.S081 lec17和《Virtual Memory Primitives for User Programs》，依旧未能解开这些问题，最终还是悲壮地转向了dead pool学长（至今还不知道是哪个助教！）实现此实验的思路，算是对前两个问题有了一点答案。而挖洞的问题，mmaptest以保证addr always be zero帮我们避免了，但在实际的比如linux系统中，addr有可能不是0时，又是怎么处理的呢？

#### mmap是否提供性能？
提供便利性是显而易见，mmap是否也提供了性能？mmap内部对文件的读写也是直接调用readi/writei，表面看来只是在原本的用户操作上多提供了一层封装，但试想这样一个场景，当你正在规规矩矩地读写一个正常大小的文件，但隔壁某个不知廉耻的进程正在读写一个大文件，于是你的文件在bcache中的缓存屁股还没坐热就被踢出去了，下一次你读文件时又得去硬盘走一遭了，而有了mmap，竞争从bcache转移到了整个物理空间，除非物理空间都被塞满了，不然你的文件会一直在内存中，因此隔壁的进程再不能给你使绊子了，性能得到提升。这是比较典型的用空间换时间的思路。

#### mmap如何应对并发？

一直很疑惑的一个点是，当两个进程同时mmap了一个文件，那么其中一个进程在munmap之前对文件做的任何改动，另外一个进程是无从知晓的，他始终在原先的文件内容上操作，我对此感到奇怪主要由于之前的文件操作是更加细粒度的，用户直接调用的read/write是带有锁的，进程a在单次write完成后进程b可以读取到文件的变化。后来我有点想通了，可以把原先的文件操作想象成带有自动保存功能，而mmap不带自动保存功能，munmap系统调用就是那个手动的保存键，这一次保存可能覆盖掉同时段其他进程对同一文件的保存。

## 二、设计方法

+ VMA的设计

  可以参考linux中VMA的设计，本实验中VMA必要的字段包括起始和结束地址、当前VMA是否在使用、flags（保存MAP_PRIVATE/MAP_SHARED）、访问权限（读/写）、关联文件和文件描述符。

+ 如何分配用户地址

  本实验保证addr恒为0，因此不必考虑用户指定映射到某个地址的情况。
  正确地给vma分配地址依赖于两个不冲突：

  + 不与用户正常的地址空间（p->size）冲突
  + 不与已有（in use）的 vma 空间冲突

  对于前者，可以效仿trampoline和trapframe的做法，把vma映射到非常高的地址区域（exactly，映射到trapframe之下）；对于后者，可以依赖两个条件来达成：

  + vma空间按照固定顺序增长
  + 在PCB中维护一个字段专门用于记录当前vma的边界

+ 如何管理所有vma

  可以效仿进程管理文件描述符的方法，在PCB中专门维护一个定长数组来管理所有VMA

+ 回收vma时回收（recollect）空间

  maxva表示下一次分配vma时mmap可用的最大虚拟地址，有两种回收策略：

  + 挤压所有现有的vma（使得任何vma之间不存在漏洞），maxva = lowest vma's start addr
  + 直接maxva = lowest vma's start addr

  显然，前者最大化地避免了空间浪费，但带来较多的额外性能损耗（memmove），因此本实验选择了后者，为实现此回收策略，需要在原先维护的maxva之外添加一个字段，用来维护 vmas 数组中当前使用的边界vma

## 三、算法分析

+ VMA设计
  ```c
  struct VMA
  {
    int valid;         // whether in use
    uint64 start_addr; // start addr
    uint64 end_addr;   // end addr
    int flags;         // flags
    int prot;          // protection mode (read/write)
    struct file *file; // mapped file
    int fd;            // mapped file's descriptor
  };
  ```

+ mmap
  ```c
  uint64
  sys_mmap(void)
  {
    uint64 addr;
    int length;
    int prot;
    int flags;
    int fd;
    struct file *f;
    int offset;
  
    if (argaddr(0, &addr) < 0 || argint(1, &length) < 0 ||
        argint(2, &prot) < 0 || argint(3, &flags) < 0 ||
        argfd(4, &fd, &f) < 0 || argint(5, &offset) < 0)
    {
      printf("mmap:obtain arg fail\n");
      return -1;
    }
  
    // check if prot conflict
    if (!f->writable && (prot & PROT_WRITE) && (flags & MAP_SHARED))
    {
      printf("mmap:prot conflict\n");
      return -1;
    }
  
    struct proc *p;
    p = myproc();
  
    // seek a vma that cloud be use
    struct VMA *vma = 0;
    for (int i = NVMA - 1; i >= 0; i--)
    {
      if (p->vmas[i].valid)
      {
        vma = &p->vmas[i];
        p->maxvma = i;
        break;
      }
    }
  
    // set up the vma
    if (vma)
    {
      // start should below the end because kalloc allocate mem follow an up order.
      uint64 end_addr = PGROUNDDOWN(p->maxva);
      uint64 start_addr = PGROUNDDOWN(p->maxva - length);
      vma->valid = 0;
      vma->fd = fd;
      vma->file = f;
      vma->flags = flags;
      vma->prot = prot;
      vma->end_addr = end_addr;
      vma->start_addr = start_addr;
      // increase file ref avoids file close while its map still in mem
      vma->file->ref++;
      p->maxva = start_addr;
    }
    else
    {
      printf("mmap:run out of vma\n");
      return -1;
    }
  
    return vma->start_addr;
  }
  ```

  + 主要流程分为三步：取回参数、获取可用 vma、设置 vma 字段
  + 需要检查mmap是否合法，不合法的mmap为需要写回文件但不具有写文件权限，但注意也许用户需要写会文件但可能根本没写，比如根本没有传入PROT_WRITE，这种情况不算入不合法
  + 记得自增相应映射文件的引用数，以避免文件还没写回之前就被关闭

+ munmap
  ```c
  uint64
  sys_munmap(void)
  {
    uint64 addr;
    int length;
  
    if (argaddr(0, &addr) < 0 || argint(1, &length) < 0)
    {
      printf("munmap:obtain arg fail\n");
      return -1;
    }
  
    struct proc *p;
    p = myproc();
  
    for (int i = NVMA - 1; i >= 0; i--)
    {
      if (p->vmas[i].start_addr <= addr && addr <= p->vmas[i].end_addr)
      {
        struct VMA *vma = &p->vmas[i];
  
        // assume unmap always at the start
        // check if the start address has a map
        if (walkaddr(p->pagetable, vma->start_addr))
        {
          // if shared, write back
          if (vma->flags == MAP_SHARED)
            filewrite(vma->file, vma->start_addr, length);
  
          uvmunmap(p->pagetable, vma->start_addr, length / PGSIZE, 1);
        }
  
        vma->start_addr += length;
        // a complete unmap
        if (vma->start_addr == vma->end_addr)
        {
          vma->file->ref--;
          vma->valid = 1;
        }
  
        // recollect making maxva = lowest vma that in use
        int j;
        for (j = p->maxvma; j < NVMA; j++)
        {
          if (!p->vmas[j].valid)
          {
            p->maxva = p->vmas[j].start_addr;
            p->maxvma = j;
            break;
          }
        }
        if (j == NVMA)
          p->maxva = VMASTART;
  
        return 0;
      }
    }
  
    return -1;
  }
  ```

  + 主要流程分为五步：获取参数、找到地址对应的vma、写回文件（若需要）、清除vma（若该vma中所有地址均被unmap）、回收空间
  + 注意回收空间时遍历的顺序要和mmap中寻找可用vma的顺序保持一致
  + exit()中的代码与之类似，只不过exit时一次性回收所有vma，不用再搞这个回收策略了

+ mmaptrap（usertrap中调用）
  ```c
  int mmaptrap()
  {
    struct proc *p = myproc();
    uint64 va = PGROUNDDOWN(r_stval());
  
    // seek the vma corresponding to va
    struct VMA *vma = 0;
    for (int i = NVMA; i >= 0; i--)
    {
      if (p->vmas[i].start_addr <= va && va <= p->vmas[i].end_addr)
      {
        vma = &p->vmas[i];
        break;
      }
    }
  
    if (vma == 0)
      return -1;
  
    // lazily kalloc mem
    char *pa = (char *)kalloc();
    if (pa == 0)
    {
      printf("mmaptrap(): run out of mem\n");
      return -1;
    }
    memset(pa, 0, PGSIZE);
    // PROT_READ = 0x01, PTE_R = 0x10, so prot should be shift
    if (mappages(p->pagetable, va, PGSIZE, (uint64)pa, (vma->prot << 1) | PTE_U | PTE_X) < 0)
    {
      printf("mmaptrap(): mappages fail\n");
      kfree(pa);
      return -1;
    }
  
    struct file *f = vma->file;
    // because the offset passed into mmap always be zero, so ignore it
    int offset = va - vma->start_addr;
  
    // read from file into mem
    ilock(f->ip);
    readi(f->ip, 1, va, offset, PGSIZE);
    iunlock(f->ip);
  
    return 0;
  }
  ```

  + 主要流程分为三步：找到地址对应的vma、分配物理空间并映射页表、从文件中读入数据
  + 注意到 PROT_READ = 0x01，PTE_R = 0x10，写标志位也是同样的关系，因此mappage时传入PROT标志位时应该左移一位
  + offset的计算实际上应该是文件内offset+地址偏移量，因为实验保证传入的文件内offset始终为0，因此这里在计算时直接忽略了

+ fork中

  ```c
    // Modify fork to ensure that the child has the same mapped regions as the parent.
    np->maxva = p->maxva;
    np->maxvma = p->maxvma;
    for (int i = NVMA - 1; i >= 0; i--)
    {
      if (p->vmas[i].file)
        p->vmas[i].file->ref++;
      np->vmas[i] = p->vmas[i];
    }
  ```

  + 就是子进程复制父进程中vma相关的所有字段，这里图省事，没有另外开辟物理地址，而是让父子进程同一个vma指向相同的地址了
