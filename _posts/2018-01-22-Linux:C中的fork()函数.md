---
layout:     post
title:      Linux/C中的fork()函数
date:       2018-01-22
author:     sg
catalog: true
tags:
    - Linux C
---

fork是用来创建和修改进程的。
### 1. 一个简单的例子
我们从这段代码说起
```c
#include <unistd.h>
#include <stdio.h>
int main(void)
{
  int pid = fork();
  if (pid < 0){
    printf("error when fork\n");
  }
  if (pid == 0){
    printf("I am parent\n");
  }
  if (pid > 0){
    printf("I am child\n");
  }
  return 0;
}
```
逻辑很简单，他的输出如下：
```bash
SG-Mac:clang sg$ ./a.out 
I am child
I am parent
```
fork()方法的返回值是新进程的pid，pid只有一个，但是可以看到pid等于0和大于0的情况都执行了。这就是它的神奇之处，给人一种它执行一次却返回两次的假象。
### 2.解释
我们把原来的进程叫做父进程，新出现的进程叫做子进程。 当父进程之行到fork方法体内部时，完成了新进程的构造。然后，父进程和子进程就从这个fork方法的内部继续向下执行，只不过附进程的fork方法返回0，子进程的fork方法返回他的进程编号。所以对于上面的代码，当跳出fork方法时，父进程会执行pid == 0的代码块， 子进程会执行pid > 0的代码块。这就给人一种fork返回两次的错觉。
### 3.实现
进程在内存中体现为代码区，常量区，静态区，堆区，栈区。 fork出的子进程在逻辑上使用了父进程的拷贝。包括操作系统相关的文件表，PC等。物理上采取了copyOnWrite的策略，也就是当子进程对内存区域做了改变之后，就复制一份相应的块出去。


