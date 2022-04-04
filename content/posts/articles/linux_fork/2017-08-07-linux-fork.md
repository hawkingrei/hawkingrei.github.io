---
title:      "Linux系统编程笔记：Linux中的fork"
date:       "2017-08-08 10:00:00"
tags: ["Linux","C", "note"]
---


> “fork是一种创建自身进程副本的操作。 ”

## Background

最近正在阅读[Twemproxy](https://github.com/twitter/twemproxy)的源代码，从中发现涉及到大量《操作系统原理》和Linux系统编程的知识，由此我这些知识记录下来，做一个系列的笔记。

## 概论

在多任务操作系统中，运行中的程序需要一种方法来创建新进程，例如运行其他程序。Fork及其变种在类Unix系统中通常是这样做的唯一方式。如果进程需要启动另一个程序的可执行文件，它需要先Fork来创建一个自身的副本。然后由该副本即“子进程”调用exec系统调用，用其他程序覆盖自身：停止执行自己之前的程序并执行其他程序。


## 正文

### 进程复制

Fork操作会为子进程创建一个单独的定址空间。子进程拥有父进程所有内存段的精确副本，其中包括：进程上下文、进程堆栈、内存信息、打开的文件描述符、信号控制设置、进程优先级、进程组号、当前工作目录、根目录、资源限制、控制终端等。

当一个进程调用fork时，它被认为是父进程，新创建的进程是它的孩子（子进程）。在fork之后，两个进程还运行着相同的程序，都像是调用了该系统调用一般恢复执行。然后它们可以检查调用的返回值确定其状态：是父进程还是子进程，以及据此行事。

子进程与父进程的区别在于：

* 父进程设置的锁，子进程不继承（因为如果是排它锁，被继承的话，矛盾了）。

* 各自的进程ID和父进程ID不同,且与存在的进程PID都不同。

* 子进程的未决告警被清除。

* 子进程的未决信号集设置为空集。

* 子进程不会从父进程继承`dnotify`(directory change notifications)。

* 在子进程中的 resource utilizations (`getrusage`)和CPU time counters (`times`)会被归零。

* 子进程不能从父进程继承计时器(`setitimer`, `alarm`, `timer_create`)。

* 子进程不能从父进程继承异步I/0操作（`aio_write`,`aio_read`）,也不能从父进程继承asynchronous I/O contexts.(`io_setup`)。

上面列出是在POSIX.1-2001下的不同点,下面是linux系统特用的不通点。

* 子进程不能继承由`ioperm`设置的I/O 端口访问权限，子进程必须使用`ioperm`来开启端口访问权限。

* 子进程的中断信号是`SIGCHLD`（`clone`）。

* Memory mappings 是被`madvise`设置的话，MADV_DONTFORK标识不会继承给子进程的

* 默认的timer slack值由父进程中的当前timer slack值来设置。
（可以看一下`prctl`中的PR_SET_TIMERSLACK的描述）

* prctl的PRSETPDEATHSIG设置会被重置，因此当父进程终止后，子进程不会收到信号。

注意以下几点:

* 子进程只包含一个线程，并且有fork产生。父进程的整个虚拟地址空间被复制给子进程，包括互斥器、条件变量和其他的pthread对象；这种行为带来的问题pthread_atfork可能有帮助。

* 子进程继承了父进程打开文件描述符。父进程和子进程指向相同的文件描述符。也就是说，两个进程的文件描述符共享相同的打开文件状态标志、当前文件偏移和信号驱动IO属性。

* 子进程继承了父进程打开消息队列描述符。父进程和子进程指向相同的消息队列描述符。也就是说，两个描述符共享相同的标志。

* 子进程继承了父进程打开目录流的集合。POSIX.1-2001说父子进程中的目录流可能共享目录流位置，但Linux/glibc没这么做。

### 实践

```c++
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv)
{
    printf("--beginning of program\n");

    int counter = 0;
    pid_t pid = fork();

    if (pid == 0)
    {
        // child process
        int i = 0;
        for (; i < 5; ++i)
        {
            printf("child process: counter=%d\n", ++counter);
        }
    }
    else if (pid > 0)
    {
        // parent process
        int j = 0;
        for (; j < 5; ++j)
        {
            printf("parent process: counter=%d\n", ++counter);
        }
    }
    else
    {
        // fork failed
        printf("fork() failed!\n");
        return 1;
    }

    printf("--end of program--\n");

    return 0;
}
```

运行结果

```
--beginning of program
parent process: counter=1
parent process: counter=2
parent process: counter=3
parent process: counter=4
parent process: counter=5
child process: counter=1
--end of program--
child process: counter=2
child process: counter=3
child process: counter=4
child process: counter=5
--end of program--
```


