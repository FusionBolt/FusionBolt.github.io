---
title: Rc-lang开发周记7 GC也没有那么可怕 其一
date: 2022-02-06 12:36:30
category: GC
tags: Rc-lang
---

本周的内容主要是写了一点点GC，同时做了一些对接GC的改动，之后接入了gtest开始测试。

由于GC基本的功能还没写完（你这也太慢了），本周将着重介绍一下GC的原理 ，让读者对GC对一些概念之类有个大概的了解，实现的细节以及我在实现中遇到思考的问题留到下周再说，~~可以等到下周养肥再一起看~~

本周从质和量来说都无法令人满意，状态比较差要写不下去了，但是起码比咕了强

# GC的对象表示

对象被保存在内存中，而对象则分为**头**和**域**两部分。

其中头被用于标识对象信息，比如说类型，以及gc的tag信息，利用tag信息来判断当前对象的状态

域则是能够被编程语言访问到的部分。域很显然可能是一个值，也可能是一个指向对象的指针

## Ruby

让我们看一下Ruby的RObject的定义

```cpp
struct RObject {

    /** Basic part, including flags and class. */
    struct RBasic basic;
    /** Object's specific fields. */
    union {
        struct {
            uint32_t numiv;
            VALUE *ivptr;
            struct st_table *iv_index_tbl;
        } heap;
        VALUE ary[ROBJECT_EMBED_LEN_MAX];
    } as;
};
```

不需要关心过多的细节，可以看到很明显是分为了头和域两部分。

让我们再来看一下头部 RBasic

```cpp
struct
RUBY_ALIGNAS(SIZEOF_VALUE)
RBasic {
    VALUE flags;
    const VALUE klass;
}
```

很显然，一个标记和一个类信息。Ruby采用的也是标记算法，这里有flags保存标记信息

## Python

再来看一下Python的实现。这次我们从头部开始看起

```cpp
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
} PyObject;
```

Object本质上是对象的头部信息。python是通过引用计数实现的GC，可以看到有一个ob_refcnt，同时还有一个保存Type的对象，

第一行的_PyObject_HEAD_EXTRA

```cpp
#ifdef Py_TRACE_REFS
/* Define pointers to support a doubly-linked list of all live heap objects. */
#define _PyObject_HEAD_EXTRA            \
    struct _object *_ob_next;           \
    struct _object *_ob_prev;

#define _PyObject_EXTRA_INIT 0, 0,

#else
#  define _PyObject_HEAD_EXTRA
#  define _PyObject_EXTRA_INIT
#endif
```

可以看到这是为了方便测试以及跟踪执行情况而添加的内容

看一下Python的对象

```cpp
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;
```

其中的ob_size是用于可变长对象使用的，例如List

## 对比

Python是每个对象的头部有一个PyObject的指针，不同的类型是基于这个扩展的

而Ruby是每个对象是一个RObject，对象内部也有一个相同的头部RBasic，而不同的类型都是RObject本身

虽然实现的方式略有不同，但是本质上还是一样的。而对于GC的实现也是一样，所以我们之后只是大概提一下实现方式的本质

# 实现算法

在这里只简单谈及标记清除、引用计数以及复制，这三者是最基本的算法，改进版本暂且也不会提及，本周的内容的目的只是希望读者能够对GC有一些了解。其他算法都是从它们衍生出来的本质并没有发生变化（~~其实主要是因为我只看了这三个~~）

## 标记清除

标记清除，我个人觉得用追溯更形象一些，因为需要从一些节点开始遍历访问所有的对象，对这些对象设置上tag，之后再对没有打上tag的对象进行回收

## 引用计数

在对象的头部设置一个字段用于标记有几个对象正在应用当前对象，在被创建的时候会设置标记为1，而被一个新的对象引用的时候计数就加1

当然这个做法存在一个很明显的问题，就是如果两个对象互相保存了对方的引用，那么就会造成循环引用的情况。C++的智能指针也是使用循环计数，因此依然会遇到这样的问题，而在C++中的解决方案是需要使用一个不获取对象所有权的weak_ptr来解决这个问题。

## 复制

对于复制算法来讲，实际上将堆等分为两部分。一部分是正在使用的空间，另一部分是作为复制的临时空间。

复制算法将所有的活动对象从当前正在使用的空间复制到临时空间，之后直接将两块空间交换，也就是说没被复制的对象直接被销毁了

# 参考书籍

垃圾回收的算法与实现

Python源码剖析
