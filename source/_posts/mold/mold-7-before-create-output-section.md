---
title: mold源码阅读七 创建输出段之前
typora-root-url: ../../source
date: 2023-05-20 16:06:32
category: Linker
tags: mold
---

![Untitled](/images/mold-7-before-create-output-section/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:101015341 p7</center> 

上期的内容主要是section size相关的优化，这期内容是创建输出段前的最后一些处理

# Compute Merged Section Size

```cpp
// Compute sizes of sections containing mergeable strings.
compute_merged_section_sizes(ctx);
```

```cpp
template <typename E>
void compute_merged_section_sizes(Context<E> &ctx) {
  Timer t(ctx, "compute_merged_section_sizes");

  // Mark section fragments referenced by live objects.
  if (!ctx.arg.gc_sections) {
    tbb::parallel_for_each(ctx.objs, [](ObjectFile<E> *file) {
      for (std::unique_ptr<MergeableSection<E>> &m : file->mergeable_sections)
        if (m)
          for (SectionFragment<E> *frag : m->fragments)
            frag->is_alive.store(true, std::memory_order_relaxed);
    });
  }

  // Add an identification string to .comment.
  if (!ctx.arg.oformat_binary)
    add_comment_string(ctx, mold_version);

  // Embed command line arguments for debugging.
  if (char *env = getenv("MOLD_DEBUG"); env && env[0])
    add_comment_string(ctx, "mold command line: " + get_cmdline_args(ctx));

  Timer t2(ctx, "MergedSection assign_offsets");
  tbb::parallel_for_each(ctx.merged_sections,
                         [&](std::unique_ptr<MergedSection<E>> &sec) {
    sec->assign_offsets(ctx);
  });
}
```

这个过程做了三件事情

1. 对非gc_sections的情况下标记fragment，在开启这个选项时会在之前的过程标记过
2. 之后是添加comment string。
3. 最后是针对每一个merged_section调用assign_offsets

关于上面的oformat，其命令行的描述如下

> -oformat=binary Omit ELF, section and program headers

add_comment_string的实现

```cpp
template <typename E>
void add_comment_string(Context<E> &ctx, std::string str) {
  MergedSection<E> *sec =
    MergedSection<E>::get_instance(ctx, ".comment", SHT_PROGBITS,
                                   SHF_MERGE | SHF_STRINGS);

  std::string_view buf = save_string(ctx, str);
  std::string_view data(buf.data(), buf.size() + 1);
  SectionFragment<E> *frag = sec->insert(data, hash_string(data), 0);
  frag->is_alive = true;
}
```

这个过程获取对应的MergeSecgtion的instance，之后插入comment str中一个新的fragment。

接下来看一下assign_offsets的实现

```cpp
template <typename E>
void MergedSection<E>::assign_offsets(Context<E> &ctx) {
  std::vector<i64> sizes(map.NUM_SHARDS);
  std::vector<i64> max_p2aligns(map.NUM_SHARDS);
  shard_offsets.resize(map.NUM_SHARDS + 1);

  i64 shard_size = map.nbuckets / map.NUM_SHARDS;

  tbb::parallel_for((i64)0, map.NUM_SHARDS, [&](i64 i) {
    struct KeyVal {
      std::string_view key;
      SectionFragment<E> *val;
    };

    std::vector<KeyVal> fragments;
    fragments.reserve(shard_size);

    for (i64 j = shard_size * i; j < shard_size * (i + 1); j++)
      if (SectionFragment<E> &frag = map.values[j]; frag.is_alive)
        fragments.push_back({{map.keys[j], map.key_sizes[j]}, &frag});

    // Sort fragments to make output deterministic.
    tbb::parallel_sort(fragments.begin(), fragments.end(),
                       [](const KeyVal &a, const KeyVal &b) {
      return std::tuple{(u32)a.val->p2align, a.key.size(), a.key} <
             std::tuple{(u32)b.val->p2align, b.key.size(), b.key};
    });

    // Assign offsets.
    i64 offset = 0;
    i64 p2align = 0;

    for (KeyVal &kv : fragments) {
      SectionFragment<E> &frag = *kv.val;
      offset = align_to(offset, 1 << frag.p2align);
      frag.offset = offset;
      offset += kv.key.size();
      p2align = std::max<i64>(p2align, frag.p2align);
    }

    sizes[i] = offset;
    max_p2aligns[i] = p2align;

    static Counter merged_strings("merged_strings");
    merged_strings += fragments.size();
  });

  i64 p2align = 0;
  for (i64 x : max_p2aligns)
    p2align = std::max(p2align, x);

  for (i64 i = 1; i < map.NUM_SHARDS + 1; i++)
    shard_offsets[i] =
      align_to(shard_offsets[i - 1] + sizes[i - 1], 1 << p2align);

  tbb::parallel_for((i64)1, map.NUM_SHARDS, [&](i64 i) {
    for (i64 j = shard_size * i; j < shard_size * (i + 1); j++)
      if (SectionFragment<E> &frag = map.values[j]; frag.is_alive)
        frag.offset += shard_offsets[i];
  });

  this->shdr.sh_size = shard_offsets[map.NUM_SHARDS];
  this->shdr.sh_addralign = 1 << p2align;
}
```

assign_offsets主要目的是设置对应MergedSection的section header中的sh_size和sh_addralign

这里的实现首先为了并行计算，将数据划分为了map.NUM_SHARDS个shard块。在每个并行的body中，先构建了对应的KeyVal，之后为了输出的确定性进行排序，最后计算其section fragment的p2aligns，以及将其长度设置为offset的初始值

在这之后算出一个最大的p2align用于设置MergedSection的section header的sh_addralign，以及计算出每一个shard块中fragment的shared_offset，最后将最后一个shard的offset(下标为n的元素，类似于vector的end的位置）作为整个MergedSection的size

# Create Synthetic Sections

这里主要创建一些特殊的段

```cpp
// Create linker-synthesized sections such as .got or .plt.
  create_synthetic_sections(ctx);
```

```cpp
template <typename E>
void create_synthetic_sections(Context<E> &ctx) {
  auto push = [&]<typename T>(T *x) {
    ctx.chunks.push_back(x);
    ctx.chunk_pool.emplace_back(x);
    return x;
  };

  if (!ctx.arg.oformat_binary) {
    auto find = [&](std::string_view name) {
      for (SectionOrder &ord : ctx.arg.section_order)
        if (ord.type == SectionOrder::SECTION && ord.name == name)
          return true;
      return false;
    };

    if (ctx.arg.section_order.empty() || find("EHDR"))
      ctx.ehdr = push(new OutputEhdr<E>(SHF_ALLOC));
    else
      ctx.ehdr = push(new OutputEhdr<E>(0));

    if (ctx.arg.section_order.empty() || find("PHDR"))
      ctx.phdr = push(new OutputPhdr<E>(SHF_ALLOC));
    else
      ctx.phdr = push(new OutputPhdr<E>(0));

    ctx.shdr = push(new OutputShdr<E>);
  }

  ctx.got = push(new GotSection<E>);

  if constexpr (!is_sparc<E>)
    ctx.gotplt = push(new GotPltSection<E>);

  ctx.reldyn = push(new RelDynSection<E>);
  ctx.relplt = push(new RelPltSection<E>);

  if (ctx.arg.pack_dyn_relocs_relr)
    ctx.relrdyn = push(new RelrDynSection<E>);

  ctx.strtab = push(new StrtabSection<E>);
  ctx.plt = push(new PltSection<E>);
  ctx.pltgot = push(new PltGotSection<E>);
  ctx.symtab = push(new SymtabSection<E>);
  ctx.dynsym = push(new DynsymSection<E>);
  ctx.dynstr = push(new DynstrSection<E>);
  ctx.eh_frame = push(new EhFrameSection<E>);
  ctx.copyrel = push(new CopyrelSection<E>(false));
  ctx.copyrel_relro = push(new CopyrelSection<E>(true));

  if (!ctx.arg.oformat_binary)
    ctx.shstrtab = push(new ShstrtabSection<E>);

  if (!ctx.arg.dynamic_linker.empty())
    ctx.interp = push(new InterpSection<E>);
  if (ctx.arg.build_id.kind != BuildId::NONE)
    ctx.buildid = push(new BuildIdSection<E>);
  if (ctx.arg.eh_frame_hdr)
    ctx.eh_frame_hdr = push(new EhFrameHdrSection<E>);
  if (ctx.arg.gdb_index)
    ctx.gdb_index = push(new GdbIndexSection<E>);
  if (ctx.arg.z_relro && ctx.arg.section_order.empty() &&
      ctx.arg.z_separate_code != SEPARATE_LOADABLE_SEGMENTS)
    ctx.relro_padding = push(new RelroPaddingSection<E>);
  if (ctx.arg.hash_style_sysv)
    ctx.hash = push(new HashSection<E>);
  if (ctx.arg.hash_style_gnu)
    ctx.gnu_hash = push(new GnuHashSection<E>);
  if (!ctx.arg.version_definitions.empty())
    ctx.verdef = push(new VerdefSection<E>);
  if (ctx.arg.emit_relocs)
    ctx.eh_frame_reloc = push(new EhFrameRelocSection<E>);

  if (ctx.arg.shared || !ctx.dsos.empty() || ctx.arg.pie)
    ctx.dynamic = push(new DynamicSection<E>);

  ctx.versym = push(new VersymSection<E>);
  ctx.verneed = push(new VerneedSection<E>);
  ctx.note_package = push(new NotePackageSection<E>);
  ctx.note_property = push(new NotePropertySection<E>);

  if (ctx.arg.is_static) {
    if constexpr (is_s390x<E>)
      ctx.s390x_tls_get_offset = push(new S390XTlsGetOffsetSection);

    if constexpr (is_sparc<E>)
      ctx.sparc_tls_get_addr = push(new SparcTlsGetAddrSection);
  }

  if constexpr (std::is_same_v<E, PPC64V1>)
    ctx.ppc64_opd = push(new PPC64OpdSection);

  // If .dynamic exists, .dynsym and .dynstr must exist as well
  // since .dynamic refers them.
  if (ctx.dynamic) {
    ctx.dynstr->keep();
    ctx.dynsym->keep();
  }

  ctx.tls_get_addr = get_symbol(ctx, "__tls_get_addr");
  ctx.tls_get_offset = get_symbol(ctx, "__tls_get_offset");
}
```

在这里其实已经开始创建输出的内容了，因为是直接push到chunk中。在mold中chunk则是表示用于输出的一片区域，关于Chunk类源码中有这样的注释

> Chunk represents a contiguous region in an output file.

首先是oformat_binary选项控制的EHDR和PHDR。

EHDR和PHDR分别是ELF Header和Program Header

EHDR和PHDR在不指定section_order或者指定的情况下存在对应的section则作为一个ALLOC的chunk加入到chunks中。

之后是添加了一些常见的段，以及各种参数控制的段，不再一一赘述。

最后提一下如果dynamic section存在的话，那么保留dynstr和dynsym段，也就是设置其size为1

```cpp
void keep() { this->shdr.sh_size = 1; }
```

# Check Duplicate Symbol

```cpp
// Make sure that there's no duplicate symbol
if (!ctx.arg.allow_multiple_definition)
  check_duplicate_symbols(ctx);
```

```cpp
template <typename E>
void check_duplicate_symbols(Context<E> &ctx) {
  Timer t(ctx, "check_duplicate_symbols");

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (i64 i = file->first_global; i < file->elf_syms.size(); i++) {
      const ElfSym<E> &esym = file->elf_syms[i];
      Symbol<E> &sym = *file->symbols[i];

      // Skip if our symbol is undef or weak
      if (sym.file == file || sym.file == ctx.internal_obj ||
          esym.is_undef() || esym.is_common() || (esym.st_bind == STB_WEAK))
        continue;

      // Skip if our symbol is in a dead section. In most cases, the
      // section has been eliminated due to comdat deduplication.
      if (!esym.is_abs()) {
        InputSection<E> *isec = file->get_section(esym);
        if (!isec || !isec->is_alive)
          continue;
      }

      Error(ctx) << "duplicate symbol: " << *file << ": " << *sym.file
                 << ": " << sym;
    }
  });

  ctx.checkpoint();
}
```

针对所有的obj进行检查，遍历所有的global symbol。

首先通过sym.file ==file 检查符号owner是否为当前文件

```cpp
// A symbol is owned by a file. If two or more files define the
// same symbol, the one with the strongest definition owns the symbol.
// If `file` is null, the symbol is equivalent to nonexistent.
InputFile<E> *file = nullptr;
```

以及如果是internal_obj中的符号，也进行跳过。剩下的就是可能有冲突的情况，但undef、weak、common的符号冲突不会造成影响，只有重复定义会导致冲突，因此这些情况也进行跳过。

最后跳过在dead section的符号，未满足前面条件的符号则是重复符号

# Check Symbol Types

```cpp
// Warn if symbols with different types are defined under the same name.
  check_symbol_types(ctx);
```

```cpp
template <typename E>
void check_symbol_types(Context<E> &ctx) {
  Timer t(ctx, "check_symbol_types");

  auto normalize_type = [](u32 type) {
    if (type == STT_GNU_IFUNC)
      return STT_FUNC;
    return type;
  };

  auto check = [&](InputFile<E> *file) {
    for (i64 i = file->first_global; i < file->elf_syms.size(); i++) {
      const ElfSym<E> &esym = file->elf_syms[i];
      Symbol<E> &sym = *file->symbols[i];

      if (!sym.file)
        continue;

      u32 their_type = normalize_type(sym.esym().st_type);
      u32 our_type = normalize_type(esym.st_type);

      if (their_type != STT_NOTYPE && our_type != STT_NOTYPE &&
          their_type != our_type)
        Warn(ctx) << "symbol type mismatch: " << sym << '\n'
                  << ">>> defined in " << *sym.file << " as "
                  << stt_to_string(sym.esym().st_type) << '\n'
                  << ">>> defined in " << *file << " as "
                  << stt_to_string(esym.st_type);
    }
  };

  tbb::parallel_for_each(ctx.objs, check);
  tbb::parallel_for_each(ctx.dsos, check);
}
```

这里针对的是所有的obj和dso里的所有global_symbol进行检查。检查实际的Symbol和ElfSym中的type是否一致，但这里只是warning，而不像之前重复符号的检查一样直接报错。检查的方式是首先对两者的type进行normalize的操作，之后进行比较，都不为空NOTYPE的情况下判断相等性。我觉得这里更像是一种针对resolve的结果检查，因为一个esym是不会被修改的，只有Symbol引用的esym对象会发生改变。
