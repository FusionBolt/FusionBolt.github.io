---
title: mold源码阅读十 段排序
typora-root-url: ../../source
date: 2023-06-24 20:39:10
category: Linker
tags: mold
---

![Untitled](/images/mold-10-sort-section/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:76218989</center> 

# 段排序

本篇文章提到的mold中出现的段排序，包含了一个chunk内的段与段的排序，还包含了chunk与chunk之间的排序。或者也可以说是对于输入角度来看待的排序，以及从输出角度看待的段进行排序。对于输入来讲，段的基本单位是InputSection，比如说一个输入文件中的一个text段，而对于输出来讲，也就是目标文件来讲，段的基本单位是一个chunk，而一个chunk是由多个输入的段组成的，比如说大的text段是由所有的输入文件中的text段组合而成。

首先要说明为什么需要进行排序

sort_init_fini，sort_ctor_dtor以及shuffle_sections属于chunk内的段与段之间的排序，在这里来说是为了满足mold的特殊需求。不过完全随机以及reverse的shuffle我还是不明白为什么需要这样来做。对于init这样的段来说，链接器需要将所有输入文件的同名段合并到同一个输出段中，因此必须要在chunk内的段与段之间的排序。

sort_output_sections属于chunk与chunk之间的排序，这里排序的目的很大一部分是为了满足特定规则的需要，不论是regular顺序的还是指定的顺序都会使得ehdr/phdr在最前，而shdr在最后。

对于中间灵活可变的部分，和对齐以及跳转指令都有关系。跳转指令这个问题我在实际遇到过，JAL指令的立即数字段的长度是固定的，而所要跳转的地址超出了JAL这个字段所能代表的长度，最后通过修改链接脚本中相关段的顺序使得地址控制在了立即数的范围之内。

# sort_init_fini

```cpp
template <typename E>
void sort_init_fini(Context<E> &ctx) {
  Timer t(ctx, "sort_init_fini");

  auto get_priority = [](InputSection<E> *isec) {
    static std::regex re(R"(\.(\d+)$)", std::regex_constants::optimize);
    std::string_view name = isec->name();
    std::cmatch m;
    if (std::regex_search(name.data(), name.data() + name.size(), m, re))
      return std::stoi(m[1]);
    return 65536;
  };

  for (Chunk<E> *chunk : ctx.chunks) {
    if (OutputSection<E> *osec = chunk->to_osec()) {
      if (osec->name == ".init_array" || osec->name == ".preinit_array" ||
          osec->name == ".fini_array") {
        if (ctx.arg.shuffle_sections == SHUFFLE_SECTIONS_REVERSE)
          std::reverse(osec->members.begin(), osec->members.end());

        sort(osec->members, [&](InputSection<E> *a, InputSection<E> *b) {
          return get_priority(a) < get_priority(b);
        });
      }
    }
  }
}
```

将所有的chunk转换为OutputSection，其中对init_array，preinit_array，fini_array段的所有成员按照priority进行排序。

> --shuffle-sections[=SEED]   Randomize the output by shuffling input sections

关于init以及后面ctor相关的段，参考maskray聚聚的博客

[.init, .ctors, and .init_array](https://maskray.me/blog/2021-11-07-init-ctors-init-array)

其中有这样一段示例汇编

```nasm
.section	.text.startup,"ax",@progbits
_GLOBAL__sub_I_a.cc:
  callq	_Znwm
  callq	_ZN1SC1Ev
  callq	getpid

.section	.init_array.101,"aw",@init_array
## legacy: .section .ctors.65434,"aw",@progbits
.p2align	3
.quad	_Z7init101v

.section	.init_array.102,"aw",@init_array
## legacy: .section .ctors.65433,"aw",@progbits
.p2align	3
.quad	_Z7init102v

.section	.init_array,"aw",@init_array
## legacy: .section .ctors,"aw",@progbits
.p2align	3
.quad	_Z4initv
.quad	_GLOBAL__sub_I_a.cc
```

这里面的init_array根据不同的priority增加不同的后缀数字，而这里的排序正是针对这些

# sort_ctor_dtor

```cpp
template <typename E>
void sort_ctor_dtor(Context<E> &ctx) {
  Timer t(ctx, "sort_ctor_dtor");

  auto get_priority = [](InputSection<E> *isec) {
    auto opts = std::regex_constants::optimize | std::regex_constants::ECMAScript;
    static std::regex re1(R"((?:clang_rt\.)?crtbegin)", opts);
    static std::regex re2(R"((?:clang_rt\.)?crtend)", opts);
    static std::regex re3(R"(\.(\d+)$)", opts);

    // crtbegin.o and crtend.o contain marker symbols such as
    // __CTOR_LIST__ or __DTOR_LIST__. So they have to be at the
    // beginning or end of the section.
    std::smatch m;
    if (std::regex_search(isec->file.filename, m, re1))
      return -2;
    if (std::regex_search(isec->file.filename, m, re2))
      return 65536;

    std::string name(isec->name());
    if (std::regex_search(name, m, re3))
      return std::stoi(m[1]);
    return -1;
  };

  for (Chunk<E> *chunk : ctx.chunks) {
    if (OutputSection<E> *osec = chunk->to_osec()) {
      if (osec->name == ".ctors" || osec->name == ".dtors") {
        if (ctx.arg.shuffle_sections != SHUFFLE_SECTIONS_REVERSE)
          std::reverse(osec->members.begin(), osec->members.end());

        sort(osec->members, [&](InputSection<E> *a, InputSection<E> *b) {
          return get_priority(a) < get_priority(b);
        });
      }
    }
  }
}
```

和sort init类似的逻辑，除了priority的计算方式不同。这里针对了clang_rt的crtbegin和crtend做了特殊的处理，最后再对其余的按照编号进行排序。

# shuffle_sections

```cpp
// Handle --shuffle-sections
if (ctx.arg.shuffle_sections != SHUFFLE_SECTIONS_NONE)
  shuffle_sections(ctx);
```

> -shuffle-sections[=SEED] Randomize the output by shuffling input sections

```cpp
template <typename E>
void shuffle_sections(Context<E> &ctx) {
  Timer t(ctx, "shuffle_sections");

  auto is_eligible = [](OutputSection<E> &osec) {
    return osec.name != ".init" && osec.name != ".fini" &&
           osec.name != ".ctors" && osec.name != ".dtors" &&
           osec.name != ".init_array" && osec.name != ".preinit_array" &&
           osec.name != ".fini_array";
  };

  switch (ctx.arg.shuffle_sections) {
  case SHUFFLE_SECTIONS_NONE:
    unreachable();
  case SHUFFLE_SECTIONS_SHUFFLE: {
    u64 seed;
    if (ctx.arg.shuffle_sections_seed)
      seed = *ctx.arg.shuffle_sections_seed;
    else
      seed = ((u64)std::random_device()() << 32) | std::random_device()();

    tbb::parallel_for_each(ctx.chunks, [&](Chunk<E> *chunk) {
      if (OutputSection<E> *osec = chunk->to_osec())
        if (is_eligible(*osec))
          shuffle(osec->members, seed + hash_string(osec->name));
    });
    break;
  }
  case SHUFFLE_SECTIONS_REVERSE:
    tbb::parallel_for_each(ctx.chunks, [&](Chunk<E> *chunk) {
      if (OutputSection<E> *osec = chunk->to_osec())
        if (is_eligible(*osec))
          std::reverse(osec->members.begin(), osec->members.end());
    });
    break;
  }
}
```

这里跳过了上面sort_init_fini以及sort_ctor_dtor的特殊段，对剩下段的members根据shuffle_sections选项进行shuffle或者reverse

# copy str

```cpp
// Copy string referred by .dynamic to .dynstr.
for (SharedFile<E> *file : ctx.dsos)
  ctx.dynstr->add_string(file->soname);
for (std::string_view str : ctx.arg.auxiliary)
  ctx.dynstr->add_string(str);
for (std::string_view str : ctx.arg.filter)
  ctx.dynstr->add_string(str);
if (!ctx.arg.rpaths.empty())
  ctx.dynstr->add_string(ctx.arg.rpaths);
if (!ctx.arg.soname.empty())
  ctx.dynstr->add_string(ctx.arg.soname);
```

拷贝所有动态库所需要用的字符串到dynstr中，主要是soname和一些路径之类的信息，而这些信息只是通过段合并是无法添加到dynstr中的，因为这些属于链接时的信息，无法通过链接输入的编译产物获取。

关于这几个链接选项的介绍

> -f SHLIB, --auxiliary SHLIB Set DT_AUXILIARY to the specified value

> -F LIBNAME, --filter LIBNAME
> Set DT_FILTER to the specified value

> --rpath DIR                 Add DIR to runtime search path

> h LIBNAME, --soname LIBNAME Set shared library name

# scan_relocations

```cpp
// Scan relocations to find symbols that need entries in .got, .plt,
// .got.plt, .dynsym, .dynstr, etc.
scan_relocations(ctx);
```

```cpp
template <typename E>
void scan_relocations(Context<E> &ctx) {
  Timer t(ctx, "scan_relocations");

  // Scan relocations to find dynamic symbols.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->scan_relocations(ctx);
  });

	...
}
```

为了保证输出文件中的got和plt等section包含对应rel的信息，递归所有所有的rel找到在got和plt中需要的符号，并且添加到got/plt中。而got和plt本身就是synthetic的段，无法从编译产物中获取，只能在链接的时候产生，为了确保rel段符号的正确查找，一定需要这一步骤。

这里主要做了三部分

1. 扫描每个obj里所有段中的符号，另外标记rel段中要处理的符合条件的符号为NEEDS_PLT
2. 将flag不为空，或者是imported/exported的符号保留，过滤掉其他符号
3. 对过滤后的符号，根据其flga添加到对应的chunk中，比如说got或者plt等，最后再清空其flag

## rel scan

```cpp
template <typename E>
void scan_relocations(Context<E> &ctx) {
  Timer t(ctx, "scan_relocations");

  // Scan relocations to find dynamic symbols.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->scan_relocations(ctx);
  });

  // Exit if there was a relocation that refers an undefined symbol.
  ctx.checkpoint();
  ...
}
```

针对每个obj进行scan_relocations

### object file

```cpp
template <typename E>
void ObjectFile<E>::scan_relocations(Context<E> &ctx) {
  // Scan relocations against seciton contents
  for (std::unique_ptr<InputSection<E>> &isec : sections)
    if (isec && isec->is_alive && (isec->shdr().sh_flags & SHF_ALLOC))
      isec->scan_relocations(ctx);

  // Scan relocations against exception frames
  for (CieRecord<E> &cie : cies) {
    for (ElfRel<E> &rel : cie.get_rels()) {
      Symbol<E> &sym = *this->symbols[rel.r_sym];

      if (sym.is_imported) {
        if (sym.get_type() != STT_FUNC)
          Fatal(ctx) << *this << ": " << sym
                     << ": .eh_frame CIE record with an external data reference"
                     << " is not supported";
        sym.flags |= NEEDS_PLT;
      }
    }
  }
}
```

在讲InputSection之前先看一下CIE的部分。CIE的代码中也会包含rel的部分。

### InputSection

isec->scan_relocations(ctx);

这里针对不同的arch有着不同的特化实现

![Untitled](/images/mold-10-sort-section/Untitled%201.png)

这里我们拿i386和riscv的实现做对比看一下差异

![Untitled](/images/mold-10-sort-section/Untitled%202.png)

可以看到前面的内容几乎一致，获取rels之后遍历，针对不同的rel type做出不同的处理

关于riscv的rel type，可以参考这两个链接

[https://github.com/riscv-non-isa/riscv-elf-psabi-doc](https://github.com/riscv-non-isa/riscv-elf-psabi-doc)

[RISC-V ELF规范和函数调用规范 - 国内芯片技术交流 - RISC-V单片机中文网——全球首家只专注于RISC-V单片机行业应用的中文网站](https://www.risc-v1.com/thread-906-1-1.html)

然后我们看针对不同rel type处理的部分

本质都是调用了InputSection的一些辅助函数，挑几个看一下。在此之前先补充两个缩写的全拼

absrel: absolute relocations

pcrel: PC-relative relocations

```cpp
switch (rel.r_type) {
  case R_RISCV_32:
    if constexpr (E::is_64)
      scan_absrel(ctx, sym, rel);
    else
      scan_dyn_absrel(ctx, sym, rel);
    break;
  case R_RISCV_HI20:
    scan_absrel(ctx, sym, rel);
    break;
  case R_RISCV_64:
    if constexpr (!E::is_64)
      Fatal(ctx) << *this << ": R_RISCV_64 cannot be used on RV32";
    scan_dyn_absrel(ctx, sym, rel);
    break;
  case R_RISCV_CALL:
  case R_RISCV_CALL_PLT:
    if (sym.is_imported)
      sym.flags.fetch_or(NEEDS_PLT, std::memory_order_relaxed);
    break;
  case R_RISCV_GOT_HI20:
    sym.flags.fetch_or(NEEDS_GOT, std::memory_order_relaxed);
    break;
  case R_RISCV_TLS_GOT_HI20:
    ctx.has_gottp_rel.store(true, std::memory_order_relaxed);
    sym.flags.fetch_or(NEEDS_GOTTP, std::memory_order_relaxed);
    break;
  case R_RISCV_TLS_GD_HI20:
    sym.flags.fetch_or(NEEDS_TLSGD, std::memory_order_relaxed);
    break;
  case R_RISCV_32_PCREL:
    scan_pcrel(ctx, sym, rel);
    break;
```

fetch_or是执行按位or运算，可以简单视为 flags |= NEEDS_XXX，相当于更新了flag，前面也说过在这个scan的过程本质就是要更新其flag

这里主要的内容就是scan_xxx，而这些scan的实现都是调用的scan_rel，区别是传入了不同的action

```cpp
template <typename E>
void InputSection<E>::scan_pcrel(Context<E> &ctx, Symbol<E> &sym,
                                 const ElfRel<E> &rel) {
  scan_rel(ctx, *this, sym, rel, get_pcrel_action(ctx, sym));
}

template <typename E>
void InputSection<E>::scan_absrel(Context<E> &ctx, Symbol<E> &sym,
                                  const ElfRel<E> &rel) {
  scan_rel(ctx, *this, sym, rel, get_absrel_action(ctx, sym));
}

template <typename E>
void InputSection<E>::scan_dyn_absrel(Context<E> &ctx, Symbol<E> &sym,
                                      const ElfRel<E> &rel) {
  scan_rel(ctx, *this, sym, rel, get_dyn_absrel_action(ctx, sym));
}

template <typename E>
void InputSection<E>::scan_toc_rel(Context<E> &ctx, Symbol<E> &sym,
                                   const ElfRel<E> &rel) {
  scan_rel(ctx, *this, sym, rel, get_ppc64_toc_action(ctx, sym));
}
```

而不同的action则是通过查表的方式来获取。每个不同的类别有着自己的表，将这个表传递给get_rel_action后进行获取。在get_rel_action中则是通过output type和sym type进行查表

```cpp
template <typename E>
static Action get_pcrel_action(Context<E> &ctx, Symbol<E> &sym) {
  // This is for PC-relative relocations (e.g. R_X86_64_PC32).
  // We cannot promote them to dynamic relocations because the dynamic
  // linker generally does not support PC-relative relocations.
  constexpr static Action table[3][4] = {
    // Absolute  Local    Imported data  Imported code
    {  ERROR,    NONE,    ERROR,         PLT    },  // Shared object
    {  ERROR,    NONE,    COPYREL,       PLT    },  // Position-independent exec
    {  NONE,     NONE,    COPYREL,       CPLT   },  // Position-dependent exec
  };

  return get_rel_action(ctx, sym, table);
}

template <typename E>
static Action get_absrel_action(Context<E> &ctx, Symbol<E> &sym) {
  // This is a decision table for absolute relocations that is smaller
  // than the word size (e.g. R_X86_64_32). Since the dynamic linker
  // generally does not support dynamic relocations smaller than the
  // word size, we need to report an error if a relocation cannot be
  // resolved at link-time.
  constexpr static Action table[3][4] = {
    // Absolute  Local    Imported data  Imported code
    {  NONE,     ERROR,   ERROR,         ERROR },  // Shared object
    {  NONE,     ERROR,   ERROR,         ERROR },  // Position-independent exec
    {  NONE,     NONE,    COPYREL,       CPLT  },  // Position-dependent exec
  };

  return get_rel_action(ctx, sym, table);
}

template <typename E>
static Action get_dyn_absrel_action(Context<E> &ctx, Symbol<E> &sym) {
  if (sym.is_ifunc())
    return IFUNC;

  // This is a decision table for absolute relocations for the word
  // size data (e.g. R_X86_64_64). Unlike the absrel_table, we can emit
  // a dynamic relocation if we cannot resolve an address at link-time.
  constexpr static Action table[3][4] = {
    // Absolute  Local    Imported data  Imported code
    {  NONE,     BASEREL, DYNREL,        DYNREL   },  // Shared object
    {  NONE,     BASEREL, DYNREL,        DYNREL   },  // Position-independent exec
    {  NONE,     NONE,    DYN_COPYREL,   DYN_CPLT },  // Position-dependent exec
  };

  return get_rel_action(ctx, sym, table);
}

template <typename E>
static Action get_ppc64_toc_action(Context<E> &ctx, Symbol<E> &sym) {
  if (sym.is_ifunc())
    return IFUNC;

  // As a special case, we do not create copy relocations nor canonical
  // PLTs for .toc sections. PPC64's .toc is a compiler-generated
  // GOT-like section, and no user-generated code directly uses values
  // in it.
  constexpr static Action table[3][4] = {
    // Absolute  Local    Imported data  Imported code
    {  NONE,     BASEREL, DYNREL,        DYNREL },  // Shared object
    {  NONE,     BASEREL, DYNREL,        DYNREL },  // Position-independent exec
    {  NONE,     NONE,    DYNREL,        DYNREL },  // Position-dependent exec
  };

  return get_rel_action(ctx, sym, table);
}
```

```cpp
template <typename E>
static Action get_rel_action(Context<E> &ctx, Symbol<E> &sym,
                             const Action table[3][4]) {
  auto get_output_type = [&] {
    if (ctx.arg.shared)
      return 0;
    if (ctx.arg.pie)
      return 1;
    return 2;
  };

  auto get_sym_type = [&] {
    if (sym.is_absolute())
      return 0;
    if (!sym.is_imported)
      return 1;
    if (sym.get_type() != STT_FUNC)
      return 2;
    return 3;
  };

  return table[get_output_type()][get_sym_type()];
}
```

### scan_rel

这里是对符号进行标记的地方，根据传入的不同action使用不同处理以及标记方式

```cpp
template <typename E>
static void scan_rel(Context<E> &ctx, InputSection<E> &isec, Symbol<E> &sym,
                     const ElfRel<E> &rel, Action action) {
  bool writable = (isec.shdr().sh_flags & SHF_WRITE);

  auto error = [&] {
    std::string msg = sym.is_absolute() ? "-fno-PIC" : "-fPIC";
    Error(ctx) << isec << ": " << rel << " relocation at offset 0x"
               << std::hex << rel.r_offset << " against symbol `"
               << sym << "' can not be used; recompile with " << msg;
  };

  auto check_textrel = [&] {
    if (!writable) {
      if (ctx.arg.z_text) {
        error();
      } else if (ctx.arg.warn_textrel) {
        Warn(ctx) << isec << ": relocation against symbol `" << sym
                  << "' in read-only section";
      }
      ctx.has_textrel = true;
    }
  };

  auto copyrel = [&] {
    assert(sym.is_imported);
    if (sym.esym().st_visibility == STV_PROTECTED) {
      Error(ctx) << isec
                 << ": cannot make copy relocation for protected symbol '" << sym
                 << "', defined in " << *sym.file << "; recompile with -fPIC";
    }
    sym.flags |= NEEDS_COPYREL;
  };

  auto dynrel = [&] {
    check_textrel();
    isec.file.num_dynrel++;
  };

  switch (action) {
  case NONE:
    break;
  case ERROR:
    error();
    break;
  case COPYREL:
    if (!ctx.arg.z_copyreloc)
      error();
    copyrel();
    break;
  case DYN_COPYREL:
    if (writable || !ctx.arg.z_copyreloc)
      dynrel();
    else
      copyrel();
    break;
  case PLT:
    sym.flags |= NEEDS_PLT;
    break;
  case CPLT:
    sym.flags |= NEEDS_CPLT;
    break;
  case DYN_CPLT:
    if (writable)
      dynrel();
    else
      sym.flags |= NEEDS_CPLT;
    break;
  case DYNREL:
  case IFUNC:
    dynrel();
    break;
  case BASEREL:
    check_textrel();
    if (!isec.is_relr_reloc(ctx, rel))
      isec.file.num_dynrel++;
    break;
  default:
    unreachable();
  }
}
```

## symbols to a vec

```cpp
template <typename E>
void scan_relocations(Context<E> &ctx) {
	...

	// Aggregate dynamic symbols to a single vector.
	std::vector<InputFile<E> *> files;
	append(files, ctx.objs);
	append(files, ctx.dsos);
	
	std::vector<std::vector<Symbol<E> *>> vec(files.size());
	
	tbb::parallel_for((i64)0, (i64)files.size(), [&](i64 i) {
	  for (Symbol<E> *sym : files[i]->symbols)
	    if (sym->file == files[i])
	      if (sym->flags || sym->is_imported || sym->is_exported)
	        vec[i].push_back(sym);
	});
	
	std::vector<Symbol<E> *> syms = flatten(vec);
	ctx.symbol_aux.reserve(syms.size());

	...
}
```

这里将所有的文件中所有符合条件的symbol收集起来，其中flag则是在前面的阶段进行标记的

关于这里的symbol_aux

```cpp
// Symbol auxiliary data
std::vector<SymbolAux> symbol_aux;
```

SymbolAux

```cpp
// Additional class members for dynamic symbols. Because most symbols
// don't need them and we allocate tens of millions of symbol objects
// for large programs, we separate them from `Symbol` class to save
// memory.
struct SymbolAux {
  i32 got_idx = -1;
  i32 gottp_idx = -1;
  i32 tlsgd_idx = -1;
  i32 tlsdesc_idx = -1;
  i32 plt_idx = -1;
  i32 pltgot_idx = -1;
  i32 opd_idx = -1;
  i32 dynsym_idx = -1;
  u32 djb_hash = 0;
};
```

## add to table

```cpp
template <typename E>
void scan_relocations(Context<E> &ctx) {
	...
	
	// Assign offsets in additional tables for each dynamic symbol.
  for (Symbol<E> *sym : syms) {
    add_aux(sym);

    if (sym->is_imported || sym->is_exported)
      ctx.dynsym->add_symbol(ctx, sym);

    if (sym->flags & NEEDS_GOT)
      ctx.got->add_got_symbol(ctx, sym);

    if (sym->flags & NEEDS_CPLT) {
      sym->is_canonical = true;

      // A canonical PLT needs to be visible from DSOs.
      sym->is_exported = true;

      // We can't use .plt.got for a canonical PLT because otherwise
      // .plt.got and .got would refer each other, resulting in an
      // infinite loop at runtime.
      ctx.plt->add_symbol(ctx, sym);
    } else if (sym->flags & NEEDS_PLT) {
      if (sym->flags & NEEDS_GOT)
        ctx.pltgot->add_symbol(ctx, sym);
      else
        ctx.plt->add_symbol(ctx, sym);
    }

    if (sym->flags & NEEDS_GOTTP)
      ctx.got->add_gottp_symbol(ctx, sym);

    if (sym->flags & NEEDS_TLSGD)
      ctx.got->add_tlsgd_symbol(ctx, sym);

    if (sym->flags & NEEDS_TLSDESC)
      ctx.got->add_tlsdesc_symbol(ctx, sym);

    if (sym->flags & NEEDS_COPYREL) {
      assert(sym->file->is_dso);
      SharedFile<E> *file = (SharedFile<E> *)sym->file;
      sym->copyrel_readonly = file->is_readonly(ctx, sym);

      if (sym->copyrel_readonly)
        ctx.copyrel_relro->add_symbol(ctx, sym);
      else
        ctx.copyrel->add_symbol(ctx, sym);

      // If a symbol needs copyrel, it is considered both imported
      // and exported.
      assert(sym->is_imported);
      sym->is_exported = true;

      // Aliases of this symbol are also copied so that they will be
      // resolved to the same address at runtime.
      for (Symbol<E> *alias : file->find_aliases(sym)) {
        add_aux(alias);
        alias->is_imported = true;
        alias->is_exported = true;
        alias->has_copyrel = true;
        alias->value = sym->value;
        alias->copyrel_readonly = sym->copyrel_readonly;
        ctx.dynsym->add_symbol(ctx, alias);
      }
    }

    if constexpr (std::is_same_v<E, PPC64V1>)
      if (sym->flags & NEEDS_OPD)
        ctx.ppc64_opd->add_symbol(ctx, sym);

    sym->flags = 0;
  }

  if (ctx.needs_tlsld)
    ctx.got->add_tlsld(ctx);

  if (ctx.has_textrel && ctx.arg.warn_textrel)
    Warn(ctx) << "creating a DT_TEXTREL in an output file";
}
```

将所有符合条件的符号添加到ctx.symbol_aux中，之后加入到对应的表中

# compute_section_sizes

```cpp
// Compute sizes of output sections while assigning offsets
// within an output section to input sections.
compute_section_sizes(ctx);
```

这里做了如下几件事情

1. chunks里面找到所有符合条件的osec进行处理
   1. 划分group并且处理每个group的size和p2align
   2. 计算与设置group的offset以及p2align
2. 处理ARM的特殊情况
3. 根据arg的section_align设定特定osec的sh_addralign

## chunks的处理

```cpp
template <typename E>
void compute_section_sizes(Context<E> &ctx) {
  Timer t(ctx, "compute_section_sizes");

  struct Group {
    i64 size = 0;
    i64 p2align = 0;
    i64 offset = 0;
    std::span<InputSection<E> *> members;
  };

	tbb::parallel_for_each(ctx.chunks, [&](Chunk<E> *chunk) {
    OutputSection<E> *osec = chunk->to_osec();
    if (!osec)
      return;

    // This pattern will be processed in the next loop.
    if constexpr (needs_thunk<E>)
      if ((osec->shdr.sh_flags & SHF_EXECINSTR) && !ctx.arg.relocatable)
        return;

    // Since one output section may contain millions of input sections,
    // we first split input sections into groups and assign offsets to
    // groups.
    std::vector<Group> groups;
    constexpr i64 group_size = 10000;

    for (std::span<InputSection<E> *> span : split(osec->members, group_size))
      groups.push_back(Group{.members = span});

    tbb::parallel_for_each(groups, [](Group &group) {
      for (InputSection<E> *isec : group.members) {
        group.size = align_to(group.size, 1 << isec->p2align) + isec->sh_size;
        group.p2align = std::max<i64>(group.p2align, isec->p2align);
      }
    });

    i64 offset = 0;
    i64 p2align = 0;

    for (i64 i = 0; i < groups.size(); i++) {
      offset = align_to(offset, 1 << groups[i].p2align);
      groups[i].offset = offset;
      offset += groups[i].size;
      p2align = std::max(p2align, groups[i].p2align);
    }

    osec->shdr.sh_size = offset;
    osec->shdr.sh_addralign = 1 << p2align;

    // Assign offsets to input sections.
    tbb::parallel_for_each(groups, [](Group &group) {
      i64 offset = group.offset;
      for (InputSection<E> *isec : group.members) {
        offset = align_to(offset, 1 << isec->p2align);
        isec->offset = offset;
        offset += isec->sh_size;
      }
    });
  });
```

1. 找到chunks中的osec
2. 将osec中的InputSections拆分为几个group并行计算
3. 针对每个group的每个InputSection
   1. 将group size根据isec的p2align计算出一个对齐的size，之后加上当前isec的size
   2. p2align更新为最大值
4. 更新所有group的offset以及p2align
5. 更新osec的size和addralign
6. 对所有group中的input section设置offset

## ARM的处理

```cpp
template <typename E>
void compute_section_sizes(Context<E> &ctx) {
	...

	// On ARM32 or ARM64, we may need to create so-called "range extension
  // thunks" to extend branch instructions reach, as they can jump only
  // to ±16 MiB or ±128 MiB, respecitvely.
  //
  // In the following loop, We compute the sizes of sections while
  // inserting thunks. This pass cannot be parallelized. That is,
  // create_range_extension_thunks is parallelized internally, but the
  // function itself is not thread-safe.
  if constexpr (needs_thunk<E>) {
    for (Chunk<E> *chunk : ctx.chunks) {
      OutputSection<E> *osec = chunk->to_osec();
      if (osec && (osec->shdr.sh_flags & SHF_EXECINSTR) && !ctx.arg.relocatable) {
        create_range_extension_thunks(ctx, *osec);

        for (InputSection<E> *isec : osec->members)
          osec->shdr.sh_addralign =
            std::max<u32>(osec->shdr.sh_addralign, 1 << isec->p2align);
      }
    }
  }

  ...
}
```

## 设定osec的sh_addralign

```cpp
template <typename E>
void compute_section_sizes(Context<E> &ctx) {
	...
	for (Chunk<E> *chunk : ctx.chunks)
    if (OutputSection<E> *osec = chunk->to_osec())
      if (u32 align = ctx.arg.section_align[osec->name])
        osec->shdr.sh_addralign = std::max<u32>(osec->shdr.sh_addralign, align);
}
```

# sort_output_sections

```cpp
// Sort sections by section attributes so that we'll have to
// create as few segments as possible.
sort_output_sections(ctx);
```

```cpp
template <typename E>
void sort_output_sections(Context<E> &ctx) {
  if (ctx.arg.section_order.empty())
    sort_output_sections_regular(ctx);
  else
    sort_output_sections_by_order(ctx);
}
```

这里对所有chunk进行了排序。

其中顺序为

```cpp
ELF Header
program header
normal memory allocated sections
non-memory-allocated sections
section header
```

normal memory allocated sections的顺序会根据用户指定的顺序，或者使用一套regular的规则

## regular

```cpp
template <typename E>
void sort_output_sections_regular(Context<E> &ctx) {
	...
	sort(ctx.chunks, [&](Chunk<E> *a, Chunk<E> *b) {
	    // Sort sections by segments
	    i64 x = get_rank1(a);
	    i64 y = get_rank1(b);
	    if (x != y)
	      return x < y;
	
	    // Ties are broken by additional rules
	    return get_rank2(a) < get_rank2(b);
	  });
}
```

注释中对排序的规则进行了说明

```cpp
// We want to sort output chunks in the following order.
//
//   <ELF header>
//   <program header>
//   .interp
//   .note
//   .hash
//   .gnu.hash
//   .dynsym
//   .dynstr
//   .gnu.version
//   .gnu.version_r
//   .rela.dyn
//   .rela.plt
//   <readonly data>
//   <readonly code>
//   <writable tdata>
//   <writable tbss>
//   <writable RELRO data>
//   .got
//   .toc
//   <writable RELRO bss>
//   .relro_padding
//   <writable non-RELRO data>
//   <writable non-RELRO bss>
//   <non-memory-allocated sections>
//   <section header>
```

代码实现

```cpp
auto get_rank1 = [&](Chunk<E> *chunk) {
  u64 type = chunk->shdr.sh_type;
  u64 flags = chunk->shdr.sh_flags;

  if (chunk == ctx.ehdr)
    return 0;
  if (chunk == ctx.phdr)
    return 1;
  if (chunk == ctx.interp)
    return 2;
  if (type == SHT_NOTE && (flags & SHF_ALLOC))
    return 3;
  if (chunk == ctx.hash)
    return 4;
  if (chunk == ctx.gnu_hash)
    return 5;
  if (chunk == ctx.dynsym)
    return 6;
  if (chunk == ctx.dynstr)
    return 7;
  if (chunk == ctx.versym)
    return 8;
  if (chunk == ctx.verneed)
    return 9;
  if (chunk == ctx.reldyn)
    return 10;
  if (chunk == ctx.relplt)
    return 11;
  if (chunk == ctx.shdr)
    return INT32_MAX;

  bool alloc = (flags & SHF_ALLOC);
  bool writable = (flags & SHF_WRITE);
  bool exec = (flags & SHF_EXECINSTR);
  bool tls = (flags & SHF_TLS);
  bool relro = is_relro(ctx, chunk);
  bool is_bss = (type == SHT_NOBITS);

  return (1 << 10) | (!alloc << 9) | (writable << 8) | (exec << 7) |
         (!tls << 6) | (!relro << 5) | (is_bss << 4);
};

auto get_rank2 = [&](Chunk<E> *chunk) -> i64 {
  if (chunk->shdr.sh_type == SHT_NOTE)
    return -chunk->shdr.sh_addralign;

  if (chunk == ctx.relro_padding)
    return INT_MAX;
  if (chunk->name == ".toc")
    return 2;
  if (chunk == ctx.got)
    return 1;
  return 0;
};
```

> tls: This section holds Thread-Local Storage, meaning that each separate execution flow has its own distinct instance of this data.

## order

```cpp
// Sort sections according to a --section-order argument.
template <typename E>
void sort_output_sections_by_order(Context<E> &ctx) {
	...
	// It is an error if a section order cannot be determined by a given
	// section order list.
	for (Chunk<E> *chunk : ctx.chunks)
	  chunk->sect_order = get_rank(chunk);
	
	// Sort output sections by --section-order
	sort(ctx.chunks, [&](Chunk<E> *a, Chunk<E> *b) {
	  return a->sect_order < b->sect_order;
	});
}
```

所有的chunk设置sect_order，之后根据这个排序。这个功能我觉得就是类似于在链接脚本中按顺序写下段的名字然后按照脚本的顺序来排序

```cpp
auto get_rank = [&](Chunk<E> *chunk) -> i64 {
  u64 flags = chunk->shdr.sh_flags;

  if (chunk == ctx.ehdr && !(chunk->shdr.sh_flags & SHF_ALLOC))
    return -2;
  if (chunk == ctx.phdr && !(chunk->shdr.sh_flags & SHF_ALLOC))
    return -1;

  if (chunk == ctx.shdr)
    return INT32_MAX;
  if (!(flags & SHF_ALLOC))
    return INT32_MAX - 1;

  for (i64 i = 0; const SectionOrder &arg : ctx.arg.section_order) {
    if (arg.type == SectionOrder::SECTION && arg.name == chunk->name)
      return i;
    i++;
  }

  std::string_view group = get_section_order_group(*chunk);

  for (i64 i = 0; i < ctx.arg.section_order.size(); i++) {
    SectionOrder arg = ctx.arg.section_order[i];
    if (arg.type == SectionOrder::GROUP && arg.name == group)
      return i;
  }

  Error(ctx) << "--section-order: missing section specification for "
             << chunk->name;
  return 0;
};
```

针对ehdr，phdr，以及shdr强制指定一个rank

之后根据arg的order查找优先级。如果没指定，则根据section_order_group再查优先级

```cpp
template <typename E>
static std::string_view get_section_order_group(Chunk<E> &chunk) {
  if (chunk.shdr.sh_type == SHT_NOBITS)
    return "BSS";
  if (chunk.shdr.sh_flags & SHF_EXECINSTR)
    return "TEXT";
  if (chunk.shdr.sh_flags & SHF_WRITE)
    return "DATA";
  return "RODATA";
};
```
