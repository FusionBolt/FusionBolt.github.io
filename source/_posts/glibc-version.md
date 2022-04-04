---
title: 关于glibc与GLIBC_XX
typora-root-url: ../../source
date: 2022-03-29 23:33:44
category: C
tags: Link
---

![67650124_p0.jpg](/images/glibc-version/67650124_p0.jpg)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">是GLIBC的版本多还是miku的版本多? pixiv:67650124</center> 

# glibc位置

这个不同系统不一致，linux中比较多的存在于/lib/libc.so.6

想要查找libc的位置可以通过ldd(linux)/otool(mac)查看依赖于libc.so的库（有的库会静态塞进去，这种的是看不了）

有的时候ldd看到的错误信息也会包含glibc的路径，这些还是根据不同的情况来查找

# 确认当前环境glibc版本信息

```bash
ldd --version
```

```c
#include <gnu/libc-version.h>
#include <stdio.h>
int main()
{
  printf("%s", gnu_get_libc_version());
}
```

两者都可以

# GLIBC Version兼容性

本质上这是一个so的不同版本兼容性问题。通常我们看到的so的版本号是 主版本号.次版本号，比如说2.6。**链接的时候只会进行主版本号的判断**，不同主版本号可能是不兼容的（不管实际如何，我们都应该视为不兼容，链接器也会报错的）。而次版本号保证新版本会兼容旧版本，比如说2.6兼容2.4

# 关于自己编译的库

## 查看GLIBC的依赖

简单的命令查看

```bash
strings libxxx.so | grep "^GLIBC"
```

你会看到多个版本号，由于新版本兼容旧版本，因此其中最新的一个GLIBC版本号是我们所需要的。这时你可能有很多小问号，让我们一个一个的来解决

## 自己的库的GLIBC Version怎么来的？

上面也提及了次版本号会高版本兼容低版本，但是如果依赖高版本的却运行于低版本时可能会出现找不到符号的情况，因此引入了**基于符号的版本机制**。即对应符号可以依赖于某个特定的次版本号

我们从一个例子来将这些串联起来。以下以上面提到过的确认当前环境GLIBC信息的示例代码为例，实际GLIBC版本大概率不会相同，与你的系统环境有关

首先使用strings查看，可以看到搜到了两个版本

```scala
GLIBC_2.2.5
GLIBC_2.34
```

当然我想你可能已经尝试过前面确认当前版本GLIBC Version的命令，发现这里的符号和当前版本的符号并不相同。我们先讲解这些版本的来源，之后就会明白原因了

那么为什么会有两个版本呢？两个版本又是怎么来的呢？让我们用nm查看一下其中的符号

```scala
000000000000039c r __abi_tag
0000000000004038 B __bss_start
0000000000004038 b completed.0
                 w __cxa_finalize@GLIBC_2.2.5
0000000000004028 D __data_start
0000000000004028 W data_start
0000000000001080 t deregister_tm_clones
00000000000010f0 t __do_global_dtors_aux
0000000000003df0 d __do_global_dtors_aux_fini_array_entry
0000000000004030 D __dso_handle
0000000000003df8 d _DYNAMIC
0000000000004038 D _edata
0000000000004040 B _end
0000000000001170 T _fini
0000000000001140 t frame_dummy
0000000000003de8 d __frame_dummy_init_array_entry
00000000000020a8 r __FRAME_END__
0000000000004000 d _GLOBAL_OFFSET_TABLE_
                 w __gmon_start__
0000000000002008 r __GNU_EH_FRAME_HDR
                 U gnu_get_libc_version@GLIBC_2.2.5
0000000000001000 T _init
0000000000002000 R _IO_stdin_used
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 U __libc_start_main@GLIBC_2.34
0000000000001149 T main
                 U printf@GLIBC_2.2.5
00000000000010b0 t register_tm_clones
0000000000001050 T _start
0000000000004038 D __TMC_END__
```

可以看到 __cxa_finalize, gnu_get_libc_version, printf是基于2.2.5，而__libc_start_main是基于2.34，这正好与我们前面看到的符号相关联。

看到这里你应该已经明白了，自己的库中GLIBC版本是来源于所使用的符号所标明的版本，因此我们在当前环境编出来的库的依赖版本实际上是当前环境的库中对应符号所依赖的版本号

# libc.so与libc.so.6

libc.so虽然长得像so，但它并不是，甚至不是一个软链接。内容大致是这样的

```
/* GNU ld script
   Use the shared library, but some functions are only in
   the static library, so try that secondarily.  */
OUTPUT_FORMAT(elf64-x86-64)
GROUP ( /usr/lib/libc.so.6 /usr/lib/libc_nonshared.a  AS_NEEDED ( /usr/lib/ld-linux-x86-64.so.2 ) )
```

# 参考资料

程序员的自我修养：链接、装载与库
