---
title: LLVM异常实现零 异常的多层结构与实现方式
typora-root-url: ../../source
date: 2024-10-02 11:27:36
category: Compiler
tags: Exception
---

本系列的博客的内容是LLVM异常实现的整个过程，从C++生成LLVM IR开始，到运行时实际调用的库函数，会从抛出异常的过程开始结合llvm相关的代码进行讲解。这一期主要内容是讲解异常相关的结构、底层实现方式等基础信息，之后从顶向下逐层分解其中的实现（这部分形式有些类似于笔记），在最后一期会将整个结构串起来，同时有一个流程图供读者进行参考，中间几期细节比较多，很容易迷失在其中，可以参照最后一期的图来阅读中间的内容。

# 多层结构

先说结论，异常主要由两部分组成

1. 语言相关的abi实现
2. 语言无关的部分（调用libunwind库）

其中语言相关的abi实现需要传递信息给libunwind，比如说一些情况要怎么处理，传递符合要求的文件头等

# 语言相关的实现

当我们编写编程语言的时候，不同的语言有不同的异常语法。比如说常见的对于一个块做try，捕获产生的不同Exception。假设这些语言都接入llvm进行代码生成，尽管编程语言有着不同的语法，但在用语法树生成llvm代码时都会生成类似的内容。

以下用C++举例，这是一段C++代码

```cpp
void f1()
{
    int a = 1;
}

void f2()
{
    throw "error";
}

void f3()
{
    f1();
    f2();
}
```

执行clang++ -S -emit-llvm main.cpp && cat main.ll查看对应的llvm ir

```cpp
@.str = private unnamed_addr constant [6 x i8] c"error\00", align 1
@_ZTIPKc = external constant i8*

; Function Attrs: noinline nounwind optnone ssp uwtable
define void @_Z2f1v() #0 {
  %1 = alloca i32, align 4
  store i32 1, i32* %1, align 4
  ret void
}

; Function Attrs: noinline optnone ssp uwtable
define void @_Z2f2v() #1 {
  %1 = call i8* @__cxa_allocate_exception(i64 8) #2
  %2 = bitcast i8* %1 to i8**
  store i8* getelementptr inbounds ([6 x i8], [6 x i8]* @.str, i64 0, i64 0), i8** %2, align 16
  call void @__cxa_throw(i8* %1, i8* bitcast (i8** @_ZTIPKc to i8*), i8* null) #3
  unreachable
}

declare i8* @__cxa_allocate_exception(i64)

declare void @__cxa_throw(i8*, i8*, i8*)

; Function Attrs: noinline optnone ssp uwtable
define void @_Z2f3v() #1 {
  call void @_Z2f1v()
  call void @_Z2f2v()
  ret void
}
```

我们来看编译出的三个函数，对于未throw的f1来说，相比f2多了一个nounwind这一个attr，并且多了两个函数调用。而调用了f1和f2的f3，因为调用了f2这个需要unwind的函数因此和f2同样没有nounwind的attr。

关于这个attr含义也很简单，用于标明函数是否会抛出异常。以下是LLVM reference中的原始文档

> This function attribute indicates that the function never raises an exception. If the function does raise an exception, its runtime behavior is undefined. However, functions marked nounwind may still trap or generate asynchronous exceptions. Exception handling schemes that are recognized by LLVM to handle asynchronous exceptions, such as SEH, will still provide their implementation defined semantics.

接着我们来看两个令人在意的函数调用：__cxa_allocate_exception和 __cxa_throw

这些是在libcxxabi中的函数，看名字我们能大概猜到其中的含义，一个是分配exception另一个则是抛出。

我们在这里先简单窥探一下__cxa_throw的实现

```cpp
void
__cxa_throw(void *thrown_object, std::type_info *tinfo, void (*dest)(void *)) {
    __cxa_eh_globals *globals = __cxa_get_globals();
    __cxa_exception* exception_header = cxa_exception_from_thrown_object(thrown_object);

    exception_header->unexpectedHandler = std::get_unexpected();
    exception_header->terminateHandler  = std::get_terminate();
    exception_header->exceptionType = tinfo;
    exception_header->exceptionDestructor = dest;
    setOurExceptionClass(&exception_header->unwindHeader);
    exception_header->referenceCount = 1;  // This is a newly allocated exception, no need for thread safety.
    globals->uncaughtExceptions += 1;   // Not atomically, since globals are thread-local

    exception_header->unwindHeader.exception_cleanup = exception_cleanup_func;

#if __has_feature(address_sanitizer)
    // Inform the ASan runtime that now might be a good time to clean stuff up.
    __asan_handle_no_return();
#endif

#ifdef __USING_SJLJ_EXCEPTIONS__
    _Unwind_SjLj_RaiseException(&exception_header->unwindHeader);
#else
    _Unwind_RaiseException(&exception_header->unwindHeader);
#endif
    //  This only happens when there is no handler, or some unexpected unwinding
    //     error happens.
    failed_throw(exception_header);
}
```

可以看到其中调用了_Unwind_RaiseException，这个函数是属于libunwind库的一个接口，而libunwind中则再无其他库的引用，这印证了前面提到的异常实现的两部分：语言相关的abi和libunwind。

# libunwind的实现

libunwind中主流的异常实现方式有三类

1. seh(structure exception handling)，在Windows上使用，但官方建议使用ISO标准C++异常处理
2. ehabi(exception handling application binary interface)，arm中定义的二进制接口，定义如何传递和处理异常的规则。
3. sjlj(setjmp / longjmp)，通过jmp指令跳转到异常处理代码。同时ehframe和dwaf经常与sjlj联合使用

[https://learn.microsoft.com/zh-cn/cpp/cpp/structured-exception-handling-c-cpp?view=msvc-170](https://learn.microsoft.com/zh-cn/cpp/cpp/structured-exception-handling-c-cpp?view=msvc-170)
