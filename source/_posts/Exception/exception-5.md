---
title: LLVM异常实现五 总结回顾
typora-root-url: ../../source
date: 2024-10-02 11:28:15
category: Compiler
tags: Exception
---

整个异常处理的过程中，涉及到各种层级之间相互调用，同时还有一些函数还会负责处理不同的功能（比如说personality会同时处理search以及clean），这都导致了整个过程相对复杂，因此在这里选择将异常抛出的整个过程再次用文字整理一遍，省略去庞大的代码细节的同时相对轻易的看到了整个过程是如何运转，都做了哪些事情，利于我们的理解。

首先一般的异常实现是由两级组成，一级由语言相关的abi，personality，以及语言相关的exception table（language specificatio data area）组成，用于实际在编译的过程中插入到代码以及生成产物中。

另一级则是libunwind的部分，主要是用于栈回溯，寻找异常栈，被用于在语言相关abi中调用，这部分封装了不同的体系结构以及异常实现方式。

然后我们根据编译到运行的流程来整理一遍

# 编译期间

1. 源码 → AST：将try catch等异常相关的源码为ast
2. AST → LLVM IR：将异常相关的ast转换为对于abi的调用以及特殊的指令（landing pad，resume等），clang中包含abi的处理，因此这里指定了personality
   1. 基础知识
      1. landing pad
      2. resume
      3. personality
3. Codegen：
   1. 收集特殊指令的信息，转换为对应的机器指令
   2. 生成eh_frame段
      1. exception table：lsda
      2. personality
4. Runtime：实际调用abi的实现

# 运行时

1. 函数入口分配异常对象 __cxa_allocation_exception
2. 执行函数体
3. 执行到抛出异常的位置调用 __cxa_throw
   1. _Unwind_RaiseException
      1. getcontext
         1. 保存当前的寄存器到context中
      2. unwind_phase1 # search
         1. 初始化 cursor
            1.  placement new在cursor的空间上构建对象
            2.  setInfoBasedOnIPRegister
                1. 获取并修正产生异常的pc
                2. 寻找unwind section，因平台和异常处理方式而异
                3. 找到后则从对应的section中读取信息填写到_info中，之后返回
                4. 如果没找到那么标记没有相关信息，最后返回
         2. 循环处理找到要处理的frame
            1. 通过cursor找到下一个frame，step
               1. step内部根据不同的处理方式找到对应的pc，更新到cursor中
            2. 获取proc的frame信息
            3. personality存在的情况下进行调用
               1. 如果找到对应的handler，那么记录sp指针并且正常返回
               2. 如果返回continue_unwind，寻找下一个frame
               3. 返回其他情况直接报错
      3. unwind_phase2 # clean up
         1. 初始化 cursor
         2. 循环处理找到要处理的frame
            1. 通过cursor找到下一个frame，step_stage2
            2. 获取proc的frame信息
            3. personality存在的情况下进行调用
               1. 如果返回continue unwind，如果sp和之前找到的相同，那么直接报错。因为前面确定了对应的栈帧，这里不应该继续unwind。
               2. 如果返回install_context，会进行resume，跳转到landingpad
               3. 返回其他情况直接报错
4. 产生异常后跳转到landingpad的位置
   1. 取出landingpad中的值信息
5. 捕获对象 __cxa_begin_catch
   1. 减少未处理对象计数
   2. 更新对象的信息
   3. 将对象push到栈上
6. 如果对象的类型匹配那么就处理，如果不匹配继续找下一个，直到找到并且处理，最后进行__cxa_end_catch
7. 找不到匹配到异常类型就resume到上一级

# 流程图

![Exception实现.png](/images/exception-5/summary.png)
