
> 元凯文 200110213



## 一、内容分析

### Print a page table

获得根page table的地址作为参数，打印出整个page table，需要呈现以下内容：

1. 各级page table中的所有有效的pte和由此推演出的物理地址
2. 层级结构（通过".."的数量表示）

### A kernel page table per process & Simplify

xv6原先的页表设计是维护一个全局的kernel page table， 当进程trap into kernel时便使用这张kernel page table，当进程处于user mode时使用自己独有的存储用户态数据的page table，这种分离的设计引起了两个问题：

1. 频繁的页表切换（向satp寄存器存入页表地址）
2. 当进程处于kernel态，无法直接访问到用户态数据，需要通过user page table索引

介于此，实验提出为每一个进程维护一个通用的页表，此页表中既包括kernel态所需的数据，也包含user态的数据，使得进程处于kernel态时一方面可以正常工作，另一方面可以直接访问用户态数据。

实验通过将原先向user page table索引的两个copy函数替换成直接访问的两个copy_new函数，来检验通用页表是否正常工作。其中实验二的目标是使这个通用页表先能复刻kernel page table，能够在使用通用页表时不影响kernel态的正常工作，实验三的目标是将用户态的数据加入通用页表的映射，能够在kernel态访问user态数据，两个实验合起来完成了整个通用页表的改造。

## 二、设计方法

### Print a page table

类似于tree结构的遍历，核心思想是递归，通过以子层page table作为参数，循环调用自身来逐层向下遍历，但这里有一个问题，就是在**循环调用的过程中tree的层级信息流失了**，而打印时需要知道层级信息，采用的办法是另设一个_vmprint函数，不但接受page table作为参数，也接受一个int值作为层级信息参数，而原先的vmprint中只需要以顶层调用\_vmprint就可以了。

### A kernel page table per process & Simplify

构造这样一个通用页表需要关注整个页表的生命周期，即分配内存->添加映射->删除映射->回收内存中的每一环，

以下几个注意点：

+ 内核栈相关

  + 每个进程都有独属的内核栈，通用页表可以不用映射其他进程的内核栈，但需要映射自己的内核栈
  + 映射内核栈的时机选择在allocproc中，即完成通用页表的创建、映射内核后立刻映射进程内核栈

+ 通用页表的创建

  + 秉承创建>修改的原则，没有选择实验hint提示的修改kvminit的建议，而是另设一个函数vminit，专门用以创建通用页表
  + vminit完成的工作有：为通用页表分配内存，向通用页表添加PLIC、UART0等内核映射

+ TRAMPOLINE和TRAPFRAME的处理

  + TRAMPOLINE在物理地址中应该仅有一份，内核态页表、用户态页表的最后一个page都映射到同一处物理地址，即存放TRAMPOLINE的地址，因此通用页表只需将最后一个page映射到同一处，就同时完成了内核态和用户态页表部分的TRAMPOLINE映射。
  + 因为内核态页表在用户态可用内存空间段是直接映射，因此虽然用户态页表添加了trapframe映射，通用页表并不需要添加复制用户态页表的trapframe映射，另外因为通用页表的TRAMPOLINE page之下就是内核栈的page，也无法添加trapframe映射

+ 通用页表的回收

  ```c
  // Free a process's page table, and free the
  // physical memory it refers to.
  void proc_freepagetable(pagetable_t pagetable, uint64 sz)
  {
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmfree(pagetable, sz);
  }
  ```

  先来看看用户态页表的回收，它主要完成了以下几个任务：

  + 删除页表中叶子对TRAMPOLINE的映射
  + 删除页表中叶子对TRAPFRAME的映射
  + 删除页表中叶子对其他物理地址空间的映射（同时free掉物理空间）
  + 删除整个页表的层级结构

  注意到用户态页表的回收将TRAMPOLINE和TRAPFRAME拎出来单独处理了，这是因为这两张page的物理空间是不需要free的，而通用页表的所有空间都不需要free（在用户态页表收回中已经free掉），因此不必单独对此进行处理，只需要完成以下两个任务：

  + 删除所有有效映射的叶子
  + 删除整个页表的层级结构

+ 切换页表

  + 之前切换进程时因为切换的过程始终处于内核态，所以无需切换页表，而当使用通用页表后，进程切换时需要同时切换到进程相应的通用页表（即向satp寄存器存入根页表的地址）
  + 因为进程的切换由scheduler控制，因此应该在此处同时完成页表的切换
  + 注意当无进程运行时应当切换至内核页表

+ PTE_U权限

  + 当处于内核态时，无法访问具有PTE_U权限的物理地址，因此在复制用户态页表的映射至通用页表时，应该特别注意取消PTE_U权限的设置

## 三、算法分析

### Print a page table

```c
// print the whole pagetable upon the address of the pagetable and the level of PGTL tree
void _vmprint(pagetable_t pagetable, int level)
{

  // there are 2^9 = 512 PTEs in a page table.
  for (int i = 0; i < 512; i++)
  {
    pte_t pte = pagetable[i];
    if (pte & PTE_V)
    {
      for (int j = 0; j < level + 1; j++)
      {
        printf(".. ");
      }
      pagetable_t child = (pagetable_t)PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      if (level < 2)
        _vmprint(child, level + 1);
    }
  }
}

// call _vmprint
void vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  _vmprint(pagetable, 0);
}
```

+ ```c
  for (int i = 0; i < 512; i++)
  ```

  遍历当前page table中的每一个pte

+ ```c
  if (pte & PTE_V)
  ```

  只打印已经mapping（即有效）的pte

+ ```c
  if (level < 2)
      _vmprint(child, level + 1);
  ```

  递归调用子层pagetable，若level==2，说明当前pte已经为叶子

### A kernel page table per process & Simplify

+ exec.c
  ```c
  // Commit to the user image.
    oldpagetable = p->pagetable;
    p->pagetable = pagetable;
    oldkpagetable = p->kpagetable;
    p->kpagetable = kpagetable;
    // switch to kernel & user pagetable
    w_satp(MAKE_SATP(p->kpagetable));
    sfence_vma();
    p->sz = sz;
    p->trapframe->epc = elf.entry; // initial program counter = main
    p->trapframe->sp = sp;         // initial stack pointer
    proc_freepagetable(oldpagetable, oldsz);
    proc_freekpagetable(oldkpagetable);
  ```

  + ```c
      oldkpagetable = p->kpagetable;
      p->kpagetable = kpagetable;
    ```

    oldkpagetable为只有内核态映射的页表，kpagetable为复制了用户态映射的页表

  + ```c
      // switch to kernel & user pagetable
      w_satp(MAKE_SATP(p->kpagetable));
      sfence_vma();
    ```

    如果复制内核态映射的实现方式是直接在原先的通用页表上修改，则不需要这一步，然而这里采用的实现是创建一张新的页表，因此在完成后需要切换寄存器中的页表，这是一个隐藏的注意点，在这里debug了非常久

  + ```c
    proc_freekpagetable(oldkpagetable);
    ```

    别忘记最后将原来的页表的空间释放

+ proc.c

  + ```c
    static struct proc *
    allocproc(void)
    {
      ******;
      ******;
      ******;
    
      // Allocate kernel & user page table.
      if ((p->kpagetable = allokpgtbl(p)) == 0)
      {
        freeproc(p);
        release(&p->lock);
        return 0;
      }
    
      // Set up new context to start executing at forkret,
      // which returns to user space.
      memset(&p->context, 0, sizeof(p->context));
      p->context.ra = (uint64)forkret;
      p->context.sp = p->kstack + PGSIZE;
    
      return p;
    }
    ```

    在初始化进程中加入了初始化通用页表的步骤

  + ```c
    pagetable_t allokpgtbl(struct proc *p)
    {
      pagetable_t kpagetable;
    
      // Allocate kernel & user page table.
      if ((kpagetable = vminit()) == 0)
      {
        return 0;
      }
    
      // Mapping the pa of kernel_stack to kernel & user pagetable
      uint64 va = KSTACK((int)(p - proc));
      uint64 pa = kwalkaddr(va);
      mappages(kpagetable, va, PGSIZE, pa, PTE_R | PTE_W);
      return kpagetable;
    }
    ```

    初始化进程的页表，分为两步：

    + vminit：为通用页表分配空间，复制内核态页表
    + 为内核栈添加映射

  + ```c
    // Free a process's user & kernel page table, don't free the
    // physical memory it refers to.
    void proc_freekpagetable(pagetable_t pagetable)
    {
      kvmunmap(pagetable, 0);
      freewalk(pagetable);
    }
    ```

    回收通用页表，分为两步：

    + kvmunmap：删除所有有效叶子映射
    + freewalk：删除页表层级结构

  + ```c
    for (p = proc; p < &proc[NPROC]; p++)
        {
          // use kernel page table if no process running
          kvminithart();
          acquire(&p->lock);
          if (p->state == RUNNABLE)
          {
            // Switch to chosen process.  It is the process's job
            // to release its lock and then reacquire it
            // before jumping back to us.
            p->state = RUNNING;
            c->proc = p;
    
            // switch to kernel & user pagetable
            w_satp(MAKE_SATP(p->kpagetable));
            sfence_vma();
    
            swtch(&c->context, &p->context);
    
            // Process is done running for now.
            // It should have changed its p->state before coming back.
            c->proc = 0;
    
            found = 1;
          }
          release(&p->lock);
        }
    ```

    此段代码位于scheduler中，完成两个任务：

    + 无进程运行时使用内核页表
    + 进程运行前切换至该进程的通用页表

+ vm.c

  + ```c
    /*
     * create a direct-map page table for the process's kernel & user pagetable
     */
    pagetable_t vminit()
    {
      pagetable_t pagetable = (pagetable_t)kalloc();
    
      if (pagetable == 0)
        return 0;
    
      memset(pagetable, 0, PGSIZE);
    
      // uart registers
      mappages(pagetable, UART0, PGSIZE, UART0, PTE_R | PTE_W);
    
      // virtio mmio disk interface
      mappages(pagetable, VIRTIO0, PGSIZE, VIRTIO0, PTE_R | PTE_W);
    
      // CLINT
      // mappages(pagetable, CLINT, 0x10000, CLINT, PTE_R | PTE_W);
    
      // PLIC
      mappages(pagetable, PLIC, 0x400000, PLIC, PTE_R | PTE_W);
    
      // map kernel text executable and read-only.
      mappages(pagetable, KERNBASE, (uint64)etext - KERNBASE, KERNBASE, PTE_R | PTE_X);
    
      // map kernel data and the physical RAM we'll make use of.
      mappages(pagetable, (uint64)etext, PHYSTOP - (uint64)etext, (uint64)etext, PTE_R | PTE_W);
    
      // map the trampoline for trap entry/exit to
      // the highest virtual address in the kernel.
      mappages(pagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X);
      return pagetable;
    }
    ```

    用于通用页表的创建，包括分配物理空间和复制内核态映射，这里有一个疑问：**为什么不添加时钟相关（CLINT）映射不影响系统运行？**希望做完后面的实验可以回来回答这个问题

  + ```c
    void kvmunmap(pagetable_t pagetable, int level)
    {
      for (int i = 0; i < 512; i++)
      {
        pte_t pte = pagetable[i];
        if (pte & PTE_V)
        {
          if (level < 2)
          {
            pagetable_t child = (pagetable_t)PTE2PA(pte);
            kvmunmap(child, level + 1);
          }
          else
          {
            pagetable[i] = 0;
          }
        }
      }
    }
    ```

    用于删除通用页表所以有效的叶子映射，类似vmprint的设计，依旧利用了层级信息level和递归调用

  + ```c
    // Load the user initcode into address 0 of pagetable,
    // for the very first process.
    // sz must be less than a page.
    void uvminit(pagetable_t pagetable, pagetable_t kpagetable, uchar *src, uint sz)
    {
      char *mem;
    
      if (sz >= PGSIZE)
        panic("inituvm: more than a page");
      mem = kalloc();
      memset(mem, 0, PGSIZE);
      mappages(pagetable, 0, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_X | PTE_U);
      mappages(kpagetable, 0, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_X);
      memmove(mem, src, sz);
    }
    ```

    增加了参数kpagetable，传入进程的通用页表，使得能够添加一份映射到通用页表中，特别注意向通用页表中添加映射时无需设置PTE_U权限，另外uvmalloc和uvmdealloc两函数也做了相似处理