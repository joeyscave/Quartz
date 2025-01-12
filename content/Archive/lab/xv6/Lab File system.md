---
date created: 2022-12-17
date modified: 2024-01-27
---

> 元凯文 200110213



## 一、内容分析

### Large files

inode中的addr数组存储了该文件存储数据的块号，addr数组一共有12个直接映射和1个不直接映射，前12个直接存储存储数据的块号，而最后1个则存储了一个特殊的块号，在这个块中放满了直接映射，因此一个inode可以持有的数据块总数为12+（1024/4）=268块，每块可以放1024byte数据，因此每个inode（文件）可以存储268\*1KB=268KB数据，这甚至放不下一首高质量的歌曲，因此想到要扩大这个数据量上限。具体方法是将一个原本的直接映射改造成双次映射，即一次映射得到的块为一个装满了间接映射的块，这样数据上限可以扩张到：(11+256+256\*256)\*1KB=64MB

![image-20221128150004190](http://img.yuankaiwen.site/202303111152589.png)



### Symbolic links

软链接是为了解决硬链接无法跨设备等问题而提出，硬链接是不产生真正的文件，只是在文件夹下添加一个指向同一个inode的entry，而软链接产生真正的文件，该文件具有特殊的文件类型，专门用来表示这是一个软链接文件，软链接在自己的数据块中写入目标文件（原文件）的路径，当需要从软链接溯源至原文件时，直接从软链接文件的数据块中读取原文件的路径即可，因此软链接文件和原文件实质上是两个完全不同的文件，即使原文件被删除，软链接文件依旧可以存在，只不过这时无法再成功导向原文件了而已。



## 二、设计方法

### Large files

首先要修改相应define，包括NDIRECT和MAXSIZE，否则mkfs构建文件系统时会出现问题。然后修改bmap函数，bmap的主要作用是获取一个inode和文件内逻辑块号（类似虚拟地址）作为参数，返回该块的实际块号（类似物理地址），写法大致仿照对singe-direct的处理就行，无非就是多了一层寻找块。最后是修改itrunc函数，此函数的主要作用是将相应inode（文件）中的数据清空，改写思路也大致仿照之前对single-indiret的处理，不过要特别注意下获取和释放锁的时机（因为这里需要依次bread两个间接块，可能造成死锁），以及需要另外的bp和a局部变量来接受第二层间接块的信息。

### Symbolic links

首先增加相应define，包括新的文件类型T_SYMLINK，新的create的tag：O_NOFOLLOW，表示不对软链接文件进行溯源找到原文件，就打开软链接文件自身，然后做好添加系统调用的一系列准备工作。实现该系用调用主要就两步，一是创建一个新的软链接文件（inode），二是把原文件的路径写入软链接文件的数据块，这两步分别依靠sysfile中的create和writei函数完成。最后需要修改open系统调用，以应对SYMLINK这一新的文件类型和O_NOFOLLOW这一新的tag，主要思路是将从路径获取软链接inode，再从软链接inode（文件）数据块获取下一个文件的路径，把这两步操作包在for循环中，当循环达到一定次数直接退出报错（循环链接了），或者当某次获取的inode发现已经不是软链接文件时就直接退出，或者传入了O_NOFOLLOW的tag时就直接获取一次inode就退出。

## 三、算法分析

### Large files

bmap中：

![image-20221128150144094](http://img.yuankaiwen.site/202303111153207.png)

itrunc中：

![image-20221128150207537](http://img.yuankaiwen.site/202303111153704.png)

### Symbolic links

Symlink:

![image-20221128150228870](http://img.yuankaiwen.site/202303111153142.png)

open中：

![image-20221128150241370](http://img.yuankaiwen.site/202303111154560.png)
