
> 元凯文 200110213



## 一、内容分析

#### Uthread

在xv6中，内核具有多个线程，而一个用户进程只持有一个线程，此实验的目标就是实现用户态的多线程。有a,b,c三个线程，每个线程在遍历一次后就立刻出让cpu，调度其他线程继续工作，实验的核心在于实现线程之间的调度，主要任务在于上下文和栈的保存与切换

#### Using threads

本实验为多线程向哈希表中添加对象，添加对象的put函数是一个原子操作，需要添加锁来保证不会有多个线程同时进入该临界区，实验的核心在于找出互斥的操作，保证这些操作原子化的同时尽可能提高性能

#### Barrier

本实验需要构造一个barrier，使得除非所有线程在barrier的某轮次中全部抵达，否则线程必须在barrier中等待其他线程到达，实验的核心在于不同线程之间传递（是否全部抵达）的信号

## 二、设计方法

#### Uthread

Uthread_switch.S的实现参考内核的switch.S，只需要保存callee_registers，因为调用者认为被调用者可能会修改其他寄存器，因此已将其他寄存器保存在栈中，所以没必要重复保存

在Uthread.c中的thread数据结构中添加context字段，类似process.h中的context，用于保存线程对应的寄存器上下文，在构造线程和切换线程时对此字段中的相应值进行修改

#### Using threads

主要使用互斥锁，但是若只有一把锁，其他线程等待时间过长，对程序的性能影响很大，考虑到哈希表中一共有5个bucket，即使同样是对哈希表写操作，若写的对象是不同的bucket，线程之间其实不会相互影响，具体put执行顺序也不会对结果造成影响，于是可以为每个bucket设置一把互斥锁，只有当其他线程要求写入同一个bucket时才会被互斥锁挡住，这样在保证了程序正常运行的前提下同时提高了性能

#### Barrier

使用条件变量来传递信号

## 三、算法分析

#### Uthread

+ uthread.c

  ```c
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)(t->stack + STACK_SIZE); // notice the stack's start location
  ```

  + 位于thread_create()，初始化线程时需要先初始化ra和sp两个寄存器，因为switch返回后从r寄存器中的地址开始执行，因此ra寄存器应该指向线程代码开始的位置
  + 特别注意栈的增长顺序是从高位置到低位置，因此栈的初始地址应为栈的最高位置

  ```c
  /* YOUR CODE HERE
   * Invoke thread_switch to switch from t to next_thread:
   * thread_switch(??, ??);
   */
  thread_switch((uint64)&t->context, (uint64)&current_thread->context);
  ```

  + 位于thread_scheduler()，当线程切换真的发生时，调用switch来将当前寄存器内容保存至上一个线程的数据结构中，并从当前线程的数据结构中恢复寄存器内容至cpu

+ uthread_switch.S
  ```c
  thread_switch:
  	/* YOUR CODE HERE */
  	    sd ra, 0(a0)
          sd sp, 8(a0)
          sd s0, 16(a0)
          sd s1, 24(a0)
          sd s2, 32(a0)
          sd s3, 40(a0)
          sd s4, 48(a0)
          sd s5, 56(a0)
          sd s6, 64(a0)
          sd s7, 72(a0)
          sd s8, 80(a0)
          sd s9, 88(a0)
          sd s10, 96(a0)
          sd s11, 104(a0)
  
          ld ra, 0(a1)
          ld sp, 8(a1)
          ld s0, 16(a1)
          ld s1, 24(a1)
          ld s2, 32(a1)
          ld s3, 40(a1)
          ld s4, 48(a1)
          ld s5, 56(a1)
          ld s6, 64(a1)
          ld s7, 72(a1)
          ld s8, 80(a1)
          ld s9, 88(a1)
          ld s10, 96(a1)
          ld s11, 104(a1)
  		
  		ret    /* return to ra */
  ```

#### Using threads

```c
static void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  pthread_mutex_lock(&lock[i]);
  for (e = table[i]; e != 0; e = e->next)
  {
    if (e->key == key)
      break;
  }
  if (e)
  {
    // update the existing key.
    e->value = value;
  }
  else
  {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock[i]);
}
```

#### Barrier

```c
static void
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread == nthread)
  {
    bstate.nthread = 0;
    bstate.round++;
    for (int i = 0; i < nthread - 1; i++)
    {
      pthread_cond_signal(&bstate.barrier_cond);
    }
  }
  else
  {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

+ 用bstate.nthread指示已到达的线程的数量
+ 用bstate.round指示barrier的轮次
+ 因为对bstate.nthread自增为原子操作，所以应先获取锁
+ 用条件 （bstate.nthread == nthread）来判断是否所有线程全部到达
+ 若未全部到达，调用cond_wait，放弃锁的同时使得线程进入等待队列
+ 若已全部达到，重置bstate.nthread，并自增bstate.round，唤醒其他全部线程
+ 离开前释放锁
