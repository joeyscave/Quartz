---
publish: false
date created: 2022-12-17
date modified: 2024-01-13
---
### 一、 资源

+ oh my git 游戏
+ 廖雪峰博客

### 二、 心得

#### 1.分支

1. 分支类似于一个指针，指向某个提交点，如果没有这个指针（git branch -d xxx 删除分支），那么会从分支当前指向的提交点开始删除，一直向前删到某个被包含在其他分支的提交路径的提交点为止，删除分支类似一个对树剪枝的过程
2. HEAD一般指向分支，也可以直接指向提交点，前者在commit时分支会自动跟着走，后者则不会

#### 2.合并冲突

1. 并不是两个文件某个部分不同就会冲突，而是两个文件对某个部分都做过修改才会冲突

#### 3.远程

1. fetch其实就是拉取远程的分支，分支名称为remoteName/branchName，其他与本地分支其实并无区别
