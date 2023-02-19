---
title: mold源码阅读 其零 main
typora-root-url: ../../source
date: 2023-02-12 22:06:32
category: Linker
tags: mold
---

![图像](/images/mold-0/FWlv3FsaMAAdSs2-20230212221048408.jpeg)

我们从main函数的开始，大致讲一下都做了哪些事情。之后再从每个流程中的具体实现开始阅读（如果我记得的话会回头在这里补上对应的链接），或者会以解决某些问题为线索写一篇，比如说某一些常见的参数具体在mold中怎么生效的，比如说whole_archive这种。为保证两部分文章内容的连贯性，内容不可避免会有一定重叠。

这个系列的一些约定

1. 只考虑elf的支持，其他平台相关的不再考虑
2. 文件路径都是项目根目录的相对路径

# 文件结构

由于代码比较少，项目的结构非常简单

```cpp
├── docs
├── elf
├── test
│   └── elf
└── third-party
    ├── mimalloc
    ├── rust-demangle
    ├── tbb
    ├── xxhash
    ├── zlib
    └── zstd
```

根目录下有一些共用的文件以及一些项目的常规文件

启动的main函数也是在根目录下

在elf目录下是我们需要看的主要核心代码

![Untitled](/images/mold-0/Untitled.png)

在这之中有两个作为主线的文件: main.cc和passes.cc

实际执行链接的主要流程都存放在main.cc的elf_main中，而这个过程执行的代码大多会指向passes.cc中的函数。不同目标arch的文件都用相应的文件名区分开了，以及其他的文件看名字也相对比较易懂。

# 链接前的准备流程

main.cc

```cpp
int main(int argc, char **argv) {
  mold::mold_version = mold::get_mold_version();

#if MOLD_IS_SOLD
  std::string cmd = mold::filepath(argv[0]).filename().string();
  if (cmd == "ld64" || cmd == "ld64.mold")
    return mold::macho::main(argc, argv);
#endif

  return mold::elf::main(argc, argv);
}
```

elf/main.cc

默认采用了X86_64

```cpp
int main(int argc, char **argv) {
  return elf_main<X86_64>(argc, argv);
}
```

对于不同的Machine Type是通过模板类型来区分的。在elf_main里面创建了全局的Context对象（并非是代码实现层面上的全局对象，只是所有的流程都需要传递ctx）并且解析命令行参数（命令行参数的具体实现就不再细看了）

```cpp
template <typename E>
int elf_main(int argc, char **argv) {
  Context<E> ctx;

  // Process -run option first. process_run_subcommand() does not return.
  if (argc >= 2 && (argv[1] == "-run"sv || argv[1] == "--run"sv)) {
#if defined(_WIN32) || defined(__APPLE__)
    Fatal(ctx) << "-run is supported only on Unix";
#endif
    process_run_subcommand(ctx, argc, argv);
  }

  // Parse non-positional command line options
  ctx.cmdline_args = expand_response_files(ctx, argv);
  std::vector<std::string> file_args = parse_nonpositional_args(ctx);
```

获取具体的machine_type

```cpp
// If no -m option is given, deduce it from input files.
if (ctx.arg.emulation == MachineType::NONE)
  ctx.arg.emulation = deduce_machine_type(ctx, file_args);

// Redo if -m is not x86-64.
if constexpr (std::is_same_v<E, X86_64>)
  if (ctx.arg.emulation != MachineType::X86_64)
    return redo_main<E>(argc, argv, ctx.arg.emulation);
```

redo_main就是简单的根据命令行参数指定的target来选择对应的模板类型进行特化

```cpp
template <typename E>
static int redo_main(int argc, char **argv, MachineType ty) {
  switch (ty) {
  case MachineType::I386:
    return elf_main<I386>(argc, argv);
  case MachineType::ARM64:
    return elf_main<ARM64>(argc, argv);
  case MachineType::ARM32:
    return elf_main<ARM32>(argc, argv);
  case MachineType::RV64LE:
    return elf_main<RV64LE>(argc, argv);
  case MachineType::RV64BE:
    return elf_main<RV64BE>(argc, argv);
  case MachineType::RV32LE:
    return elf_main<RV32LE>(argc, argv);
  case MachineType::RV32BE:
    return elf_main<RV32BE>(argc, argv);
  case MachineType::PPC64V1:
    return elf_main<PPC64V1>(argc, argv);
  case MachineType::PPC64V2:
    return elf_main<PPC64V2>(argc, argv);
  case MachineType::S390X:
    return elf_main<S390X>(argc, argv);
  case MachineType::SPARC64:
    return elf_main<SPARC64>(argc, argv);
  case MachineType::M68K:
    return elf_main<M68K>(argc, argv);
  default:
    unreachable();
  }
}
```

# 链接大体流程

根据注释和我个人的理解，分为如下这么几大部分

1. 解析所有的输入，包含命令行参数，输入的各种文件
2. 对于输入做链接器最基本的处理，包含符号解析，段合并，符号检查之类的
3. 创建一些synthetic的内容，包括一些段和符号
4. 将所有段、符号进行扫描以及按照需求进行排序，添加到全局的ctxt中
5. 计算与修正一些具体的信息，固定生成产物的memory layout
6. 修正某些地址，确保固定file layout
7. 将所有文件拷贝到输出文件中
8. 结束的清理操作

其中有些地方可以根据Timer来协助划分链接的流程。比如说拷贝到输出之前有这样一行

```cpp
Timer t_copy(ctx, "copy");
```

而到了后面的部分有这么一行对应，中间的部分很自然就是这一个步骤做的事情了

```cpp
t_copy.stop();
```

而main函数中的内容比较简洁，几乎每个小功能都划分为了一个函数，而且附加了大量的注释，比如说这样

```cpp
// Create .bss sections for common symbols.
convert_common_symbols(ctx);

// Apply version scripts.
apply_version_script(ctx);

// Parse symbol version suffixes (e.g. "foo@ver1").
parse_symbol_version(ctx);

// Set is_imported and is_exported bits for each symbol.
compute_import_export(ctx);
```

再加上代码比较长，这里就不放后续完整代码了。
