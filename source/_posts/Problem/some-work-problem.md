---
title: 工作踩坑小结
typora-root-url: ../../source
date: 2022-10-02 17:26:12
category: Debug
tags: 
  - [CrossCompiling]
  - [Link]
  - [Conda]
---

前些时间工作中踩到的坑做个简单小总结，第一次搞裸机与交叉编译，本次内容也以此为主。

# 编译

一开始闹了一个小乌龙，工具链支持到c++17的标准，但是同事之前指定了14的标准，差点就要把filesystem相关的代码全改掉了。但是后来依然编译不过，在需要系统调用的标准库处报了错误，这才想到裸机并没有这种东西，最后还是加条件判断宏全部处理掉了…

# 链接

## 修复问题

裸机的启动代码中有一些汇编，其中JAL跳转指令在链接的时候报了错

startup.S:120:(.text+0xbe): relocation truncated to fit: R_RISCV_JAL against symbol `SystemInit' defined in .text.SystemInit section in

[https://stackoverflow.com/questions/10486116/what-does-this-gcc-error-relocation-truncated-to-fit-mean](https://stackoverflow.com/questions/10486116/what-does-this-gcc-error-relocation-truncated-to-fit-mean)

先说结论，JAL指令的立即数字段的长度是固定的，而所要跳转的地址超出了JAL这个字段所能代表的长度。

最初猜想是否和我的lib大小有关系，尝试删掉了部分代码缩小了接近一半的体积后果然可行。但是依靠这种方法解决是不可行的，代码体积无法再简化了，而且以后lib体积只会增大。参考链接中设置mcmodel，然而依然报错。

接着尝试修改链接顺序，因为符号的顺序是和链接的顺序相关的，想要将对应的符号放到链接的最前面，但是需要跳转到我的lib中的符号，又不方便再去调整lib中的顺序。

最后在同事的提醒下修改了链接脚本，将这些报错的text section放到了最前面。

```nasm
.text : {
  . = ALIGN(0x8) ;
  __stext = . ;
  KEEP(*startup.o(*.text*))
  KEEP(*startup.o(*.vectors*))
  /* avoid link failed when lib too large */
  *(.text.SystemInit)
  *(.text.trap_c)
  *(.text.vTaskSwitchContext)
  *(.text.startup.main)
	*(.text)
  *(.text*)
  *(.text.*)
  ...
}
```

# conda的环境问题

在使用某个python库的时候提示了Could not find a suitable hostfxr library，一直以为hostfxr相关的库版本错了，直到我点进这个源码看

```python
def load_hostfxr(dotnet_root: str):
    hostfxr_name = _get_dll_name("hostfxr")
    hostfxr_path = os.path.join(dotnet_root, "host", "fxr", "?.*", hostfxr_name)
    for hostfxr_path in reversed(sorted(glob.glob(hostfxr_path))):
        print(hostfxr_path)
        try:
            return ffi.dlopen(hostfxr_path)
        except Exception as error:
            pass
    raise RuntimeError(f"Could not find a suitable hostfxr library in {dotnet_root}")
```

一看血压直接拉满，抛了异常一律视为没找到。手动改成打印错误信息才发现是dlopen的时候所加载的glibcxx版本不对，由于是在conda环境下因此去修改conda的链接。不是第一次被conda坑了…

# 优化与调试

这算是我第一次实际遇到因为优化产生的问题。由于最近在调试内存分配相关模块的问题，我想要手动malloc/new一块内存复现问题。此处为代码

```cpp
void test_malloc() {
	int *a = malloc(16);
	printf("na\n");
	int *b = malloc(32);
	printf("nb\n");
	free(a);
	int *c = malloc(16);
	printf("nc\n");
	free(c);
	int *d = malloc(32);
	printf("nd\n");
}
```

由于用的是裸机专用的工具链，因此内存的分配和释放都会调用工具链中的代码，我在其中打了log，但是发现new的时候并没有打印log。

没有调试器，想了半天怎么也想不明白，最后查看反汇编发现画风是这样的

![Untitled](/images/some-work-problem/Untitled.png)

指定编译选项的部分都是其他同事编写的，我一开始也没往这里想。看了半天最后发现原来malloc被优化掉了。b和d很直接，是unused的代码，但是a和c都被free了却依然被优化掉。

关于这个问题好奇搜了一下，搜到这个回答

[https://stackoverflow.com/questions/17899497/malloc-and-gcc-optimization-2](https://stackoverflow.com/questions/17899497/malloc-and-gcc-optimization-2)

the optimizer knows malloc and considers it is a function with no side-effects，多半是编译器内部针对特定符号编写的优化
