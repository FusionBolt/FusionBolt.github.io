---
title: 调试器之工作原理
typora-root-url: ../../source
date: 2023-01-09 23:27:58
category: Debugger
tags: ptrace
---

# 调试器之工作原理

之前对于调试器并没有什么了解，对于很多问题也没什么头脑，比如说attach是怎么做到的，怎么实现运行时断点的。今天来简单了解一下调试器部分功能的工作原理。

# 断点

对于调试来说第一步是要下断点。断点本质是到了指定位置后中断当前的进程，进入对应的中断处理程序。（信号的本质是软中断，这里、统一称发生了中断）

根据实现方式的不同分为如下三类。

## 软件断点

当cpu执行了特定调试指令后会发出一个中断，而软件断点要做的就是在对应的pc位置“插入”断点指令，说是插入，实际上是修改原指令，触发中断后再写回。

以x86的INT3指令为例，在一个位置设置断点后会保存该位置的原指令，之后在该位置写入INT3，当执行到这条指令的时候发生软中断，内核向子进程发送SIGTRAP信号，之后这个信号转发给父进程，此时再用保存的指令替换之前写入的INT3指令等待中断恢复。

## 硬件断点

某些cpu包含调试用的寄存器，通过设置对应的值来控制对应产生中断的pc位置以及一些其他信息。

[x86 debug register - Wikipedia](https://en.wikipedia.org/wiki/X86_debug_register)

cpu在执行代码之前会先确定要执行的地址是否保存在中断寄存器中，同时确认访问的地址是否处于设置了硬件断点的区域内，满足条件后会触发INT1中断。

## 内存断点

通过设置对应内存位置所在页为guard page，对保护页访问则会触发异常，之后页面恢复访问前的状态。

# ptrace

Linux中我们可以直接通过ptrace来打断点、读取信息或者是单步执行等。

关于ptrace的文档：[https://man7.org/linux/man-pages/man2/ptrace.2.html](https://man7.org/linux/man-pages/man2/ptrace.2.html)

## 直接调试

首先我们来看一下用法示例

```cpp
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h> 
int main()
{   pid_t child;
    long orig_eax;
    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else {
        wait(NULL);
        orig_eax = ptrace(PTRACE_PEEKUSER,child, 4 * ORIG_EAX,NULL);
        printf("The child made a ""system call %ld\n", orig_eax);
        ptrace(PTRACE_CONT, child, NULL, NULL);
    }
    return 0;
}
```

被调试的程序通过ptrace(PTRACE_TRACEME)来设定自身是被trace的对象，接着通过execl来执行对应的命令行程序，此时执行的程序作为调试器的子进程。

而调试器进程本身则是通过wait去等待子进程停下来，等wait返回后就可以查看子进程的信息或者对子进程进行操作。对于ptrace使用方面来说最重要的是选择合适的__ptrace_request，大多数调试器常见的功能都能通过设置这个参数来实现，比如说单步。

这个项目使用ptrace实现了许多debug的基础功能

[https://github.com/Kakaluoto/ptraceDebugger](https://github.com/Kakaluoto/ptraceDebugger)

## attach

通过设置__ptrace_request为PTRACE_ATTACH或者PTRACE_SEIZE还可以调试一个当前已经启动的进程。

对于常规的调试和attach的本质区别自然是进程间的关系，直接调试中调试器进程和被调试进程互为父子进程，而attach时两者是独立的，也因此有的时候attch会需要管理员权限。

# 其他系统

以上ptrace的实现都是基于Linux的api来讲的，macOS的ptrace的request缺少非常多基本功能，比如说读取寄存器的值。如果想要在mac下实现可以参考如下链接，如果是arm的Mac则这里很多接口仍然过时。（我反正不想折腾了，有这时间多看下Linux的不香吗）

[Uninformed - vol 4 article 3](http://uninformed.org/index.cgi?v=4&a=3&p=14)

[Using ptrace on OS X](https://www.spaceflint.com/?p=150)

而对于windows来说则是提供了另一套完全不同的api，有兴趣的可以自行了解。

[Debugger Programming Extension APIs - Windows drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-engine-and-extension-apis)

# 后续

这一期的内容都是一些非常容易搜到的基础知识，如果不鸽的话调试器后面会继续深入学习，造一个自己的debugger之类的。大概也会作为一个系列更新，可能深入的方向有如下几个

1. ptrace的具体实现细节代码
2. debug信息的格式以及源码级调试
3. lldb的学习
