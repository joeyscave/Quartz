
> 元凯文 200110213



## 一、内容分析

### Lab Backtrace

这个实验的核心是**stack frame**的概念，一个frame可以看作是保存一个函数信息的单元，其中囊括了返回地址、参数信息等，因此可以通过依次walk栈中的frame来获取调用的路径（trace)，另外尤为关键的概念是两个指针：**sp**和**fp**，sp始终指向栈的顶部，fp指向当前frame中的某一个字段，该字段中存放了上一个frame（fp）的位置，正是通过fp的这种类似链表的结构，我们得以从当前的frame向前溯源，实现backtrace()

<img src="C:\Users\86186\Downloads\pic\temp\image-20220927234511440.png" alt="image-20220927234511440" style="zoom: 50%;" />

### Lab Alarm

alarm类似于一个软件层面的time interrupt，用户可以自定义interrupt的interval以及interrrupt后的向量函数。在用户层面可见的是两个系统调用：sigalarm和sigreturn，前者设置alarm，后者使得进程从向量函数中退出回到原本的指令中

## 二、设计方法

### Lab Backtrace

两个关键问题：

+ 如何获取当前fp？

  通过内联汇编代码：

  ```c
  static inline uint64
  r_fp()
  {
    uint64 x;
    asm volatile("mv %0, s0"
                 : "=r"(x));
    return x;
  }
  ```

+ 如何终止遍历？

  每个kernel stack单独享有一个page，因此当前指针是否还在同一page里可以表示指针是否离开了kernel stack

### Lab Alarm

五个关键问题：

+ 在哪里设置interval、handler和已经经过的ticks？

  在proc.h中的struct proc中

+ 如何相对进程“异步”地发起interrupt？

  自然的想法是fork一个子进程来专门做这件事，然而这却带来很多麻烦，一方面是额外的开销，一方面子进程需要监视父进程的情况，当父进程结束，这个alarm子进程也就没有存在的必要，种种原因使得此方法成为下策。
  注意到interval是以tick为基本单位，而每个tick都会有一次来自硬件的time interrupt，而time interrupt会导致进程trap into kernel，执行usertrap等函数，这意味着每个tick过去时进程总会到同一段routine里去，这对于我们是个好机会，可以在这段routine中加入alarm的判断、生效和计时等逻辑

+ 如何执行handler？

  trapframe中的epc字段存储回到用户态后进程开始执行的pc，因此可以通过改写epc为handler的地址来使进程回到用户态后开始执行handler

+ 如何在执行完毕handler后回到进程之前执行的指令？

  因为我们通过简单地改写了epc来执行handler，使得原先的指令位置被覆写，因此在覆写之前需要将trapframe中必要的环境保存下来，这里不仅仅包括epc的值，也包括所有寄存器值，在执行完毕handler之后，handler内部调用sigreturn，sigreturn将所有保存的环境填回寄存器，使得进程回到之前指令。

+ 如何避免在上一个handler还没退出时就因再次到期而进入下一个handler？

  在struct proc中设置一个字段专门指示上一个handler的退出情况

## 三、算法分析

### Lab Backtrace

```c
void backtrace(void)
{
  uint64 s0 = r_fp();
  printf("backtrace:\n");
  for (; s0 < PGROUNDUP(s0); s0 = *(uint64 *)(s0 - 16))
  {
    // print return address (offset -8)
    printf("%p\n", *(uint64 *)(s0 - 8));
  }
}
```

+ ```c
  uint64 s0 = r_fp();
  ```

  获取当前frame位置

+ ```c
  s0 = *(uint64 *)(s0 - 16);
  ```

  s0-16指向当前frame的frame pointer，造型后取出值，即得上一个frame的位置

+ ```c
  #define PGROUNDUP(sz) (((sz) + PGSIZE - 1) & ~(PGSIZE - 1))
  s0 < PGROUNDUP(s0);
  ```

  当s0到了page的边缘，即s0的值为PGSIZE的倍数时，PGROUNDUP中的+PGSIZE-1不会起到效果，此时PGROUNDUP计算出的值将正好等于s0，以此作为循环结束的条件

+ ```c
  *(uint64 *)(s0 - 8)
  ```

  s0-8指向当前frame中存储的return address，造型后取出值输出，即该frame代表的函数的返回地址

### Lab Alarm

```c
struct alarmframe
{
  uint64 interval; // sigalarm interval
  uint64 handler;  // sigalarm handle function
  uint64 ticks;    // passed ticks since last handle alram
  uint64 epc;      // saved user program counter
  uint64 returned; // indicate if returned from handler
  uint64 ra;
  uint64 sp;
  uint64 gp;
  uint64 tp;
  uint64 t0;
  uint64 t1;
  uint64 t2;
  uint64 s0;
  uint64 s1;
  uint64 a0;
  uint64 a1;
  uint64 a2;
  uint64 a3;
  uint64 a4;
  uint64 a5;
  uint64 a6;
  uint64 a7;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
  uint64 t3;
  uint64 t4;
  uint64 t5;
  uint64 t6;
};
```

+ 效仿trapframe，将alarm需要的字段聚合成一个struct alarmframe写在proc.h中，这里特别需要注意的是，在proc.c中初始化进程函数中对alarmframe初始化时要显式地为它分配内存空间，同理，要为分配的空间mapping，收回进程的时候要记得free分配的空间
+ 设置epc为第4个字段是为了和trapframe对齐，便于赋值

```c
uint64
sys_sigalarm()
{
  struct proc *p = myproc();
  int ticks;
  uint64 handler;

  if (argint(0, &ticks) < 0)
    return -1;
  if (argaddr(1, &handler) < 0)
    return -1;

  p->alarmframe->interval = ticks;
  p->alarmframe->handler = handler;

  return 0;
}
```

+ 两个arg函数从trapframe中分别取回interval和handler两个参数
+ 得到两个参数后对alarmframe中相应字段初始化

```c
if (which_dev == 2)
  {
    if (p->alarmframe->interval != 0)
    {
      p->alarmframe->ticks++;
      if (p->alarmframe->interval == p->alarmframe->ticks)
      {
        p->alarmframe->ticks = 0;
        if (p->alarmframe->returned == 0)
        {
          alarmsave();
          p->trapframe->epc = p->alarmframe->handler;
          p->alarmframe->returned = 1;
        }
      }
    }
    yield();
  }
```

+ 这段逻辑位于trap.c: usertrap()中

+ ```c
  if (which_dev == 2)
  ```

  确保只有是time interrupt时才计时以及生效，忽略systemcall等其他trap的影响

+ ```c
  if (p->alarmframe->interval != 0)
  ```

  因为interval被初始化为0，因此指示了alarm已经被设置

+ ```c
  if (p->alarmframe->returned == 0)
  ```

  指示上一个handler已经退出

+ ```c
  alarmsave();
  ```

  保存此时trapframe环境，具体见下

+ ```c
  p->trapframe->epc = p->alarmframe->handler;
  ```

  将执行位置转向handler

```c
// save the user register in alarmframe
void alarmsave(void)
{
  struct proc *p = myproc();
  uint64 *ap = (uint64 *)p->alarmframe;
  uint64 *tp = (uint64 *)p->trapframe;

  ap += 3;
  tp += 3;
  *ap = *tp; // alarmframe->epc = trapframe->epc
  tp++;
  ap++;

  // save other registers
  for (int i = 0; i < 31; i++)
  {
    *(++ap) = *(++tp);
  }
}
```

+ ap和tp分别为指向alarmframe和trapframe中某一个字段的指针
+ 先保存epc，后保存其他所有寄存器

```c
uint64
sys_sigreturn()
{
  struct proc *p = myproc();
  uint64 *ap = (uint64 *)p->alarmframe;
  uint64 *tp = (uint64 *)p->trapframe;

  ap += 3;
  tp += 3;
  *tp = *ap; // trapframe->epc = alarmframe->epc
  tp++;
  ap++;

  // resume other registers
  for (int i = 0; i < 31; i++)
  {
    *(++tp) = *(++ap);
  }

  p->alarmframe->returned = 0;

  return 0;
}
```

+ 恢复寄存器现场，基本是alarmsave的逆过程

+ ```c
  p->alarmframe->returned = 0;
  ```

  表示退出此handler