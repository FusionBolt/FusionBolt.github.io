---
title: mold源码阅读十一  relr and dynsym
typora-root-url: ../../source
date: 2023-07-02 16:04:13
category: Linker
tags: 
  - [mold]
  - [got]
  - [rel]
---

![Untitled](/images/mold-11-rel-and-dynsym/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">推特画师Lyytoaoitori</center> 

# construct_relr

```cpp
// If --packed_dyn_relocs=relr was given, base relocations are stored
// to a .relr.dyn section in a compressed form. Construct a compressed
// relocations now so that we can fix section sizes and file layout.
if (ctx.arg.pack_dyn_relocs_relr)
  construct_relr(ctx);
```

> -z pack-relative-relocs Alias for --pack-dyn-relocs=relr
> -z nopack-relative-relocs

将OutputSection以及Got中的relocations以压缩的形式存储到relr.dyn，在这之后rel段的大小和layout就固定了。

```cpp
template <typename E>
void construct_relr(Context<E> &ctx) {
  Timer t(ctx, "construct_relr");

  tbb::parallel_for_each(ctx.chunks, [&](Chunk<E> *chunk) {
    if (OutputSection<E> *osec = chunk->to_osec())
      osec->construct_relr(ctx);
  });

  ctx.got->construct_relr(ctx);
}
```

## output section

将output section中所有符合条件的rel段收集起来，最后再压缩。

```cpp
template <typename E>
void OutputSection<E>::construct_relr(Context<E> &ctx) {
  if (!ctx.arg.pic)
    return;
  if (!(this->shdr.sh_flags & SHF_ALLOC))
    return;
  if (this->shdr.sh_addralign % sizeof(Word<E>))
    return;

  // Skip it if it is a text section because .text doesn't usually
  // contain any dynamic relocations.
  if (this->shdr.sh_flags & SHF_EXECINSTR)
    return;

  // Collect base relocations
  std::vector<std::vector<u64>> shards(members.size());

  tbb::parallel_for((i64)0, (i64)members.size(), [&](i64 i) {
    InputSection<E> &isec = *members[i];
    if ((1 << isec.p2align) < sizeof(Word<E>))
      return;

    for (const ElfRel<E> &r : isec.get_rels(ctx))
      if (r.r_type == E::R_ABS && (r.r_offset % sizeof(Word<E>)) == 0)
        if (Symbol<E> &sym = *isec.file.symbols[r.r_sym];
            !sym.is_absolute() && !sym.is_imported)
          shards[i].push_back(isec.offset + r.r_offset);
  });

  // Compress them
  std::vector<u64> pos = flatten(shards);
  relr = encode_relr(pos, sizeof(Word<E>));
}
```

这里要讲的是开头是否为pic的判断

> --pie, --pic-executable Create a position independent executable
> --no-pie, --no-pic-executable

```cpp
else if (read_flag("pie") || read_flag("pic-executable")) {
  ctx.arg.pic = true;
  ctx.arg.pie = true;
} else if (read_flag("no-pie") || read_flag("no-pic-executable")) {
  ctx.arg.pic = false;
  ctx.arg.pie = false;
```

有两个概念，pic和pie

pic：position-independent code

pie：position-independent executable

pic和pie是看似类似却是完全冲突的两个选项。

pie是生成位置无关的可执行程序，所有变量（静态和全局变量，或者说局部变量外的变量）的地址在executable中已经确定，由于这个位置确定因此不需要got表，尽管地址确定但是executable可以加载到任意地址，因为确定的是executable的内部偏移。

而pic通常是一个动态库，在运行时可以加载到任意位置，也就是说相对于加载这个pic库的executable的地址也是未知的，可能加载到前面，也可能加载到后面，确定的地址只有相对于这个pic库内部起始地址的偏移，因此需要利用got中的信息再计算具体加载后的地址。

看到这些内容也就明白为什么不是pic的话就返回了，因为pie的话并不需要进行重定位来支持动态加载

## got

got(global offset table)，保存了global符号的内存地址，比如说function或者全局变量，用于运行时重定位来解析这些地址。程序首次运行时got被初始化为未解析的地址，调用函数的时候通过rel.plt/rela.plt解析对应符号的地址，之后地址会被保存到got，供下次解析使用。

```cpp
template <typename E>
void GotSection<E>::construct_relr(Context<E> &ctx) {
  assert(ctx.arg.pack_dyn_relocs_relr);

  std::vector<u64> pos;
  for (GotEntry<E> &ent : get_got_entries(ctx))
    if (ent.is_relr(ctx))
      pos.push_back(ent.idx * sizeof(Word<E>));

  relr = encode_relr(pos, sizeof(Word<E>));
}
```

### get_got_entries

这个过程主要是从各个位置获取GotEntry

```cpp
template <typename E>
struct GotEntry {
  bool is_relr(Context<E> &ctx) const {
    return r_type == E::R_RELATIVE && ctx.arg.pack_dyn_relocs_relr;
  }

  i64 idx = 0;
  u64 val = 0;
  i64 r_type = R_NONE;
  Symbol<E> *sym = nullptr;
};
```

关于rel_type

> Relocation entries describe how to alter the following instruction and data fields (bit numbers appear in the lower box corners).

这里提前引用部分elf spec中提到的在i386中下面会用到的几种rel type的含义

R*_386_GLOB_DAT*

> *This relocation type is used to set a global offset table entry to the address of the specified symbol. The special relocation type allows one to determine the correspondence between symbols and global offset table entries.*

*R_386_RELATIVE*

> *The link editor creates this relocation type for dynamic linking. Its offset member gives a location within a shared object that contains a value representing a relative address. The dynamic linker computes the corresponding virtual address by adding the virtual address at which the shared object was loaded to the relative address. Relocation entries for this type must specify 0 for the symbol table index.*

关于其他的rel type参考信息

[https://docs.oracle.com/cd/E19683-01/817-3677/x-j1h4h/index.html](https://docs.oracle.com/cd/E19683-01/817-3677/x-j1h4h/index.html)

> A R_386_TLS_TPOFF relocation is left outstanding against the GOT table for the runtime linker to fill in with the static TLS offset for symbol x.

1. ordinary symbols

   1. 针对imported的符号：需要dynamic linker resolve，因此rel_type设置为GLOB_DAT。同时链接时地址未知，因此地址为0

   2. ifunc的符号，通常需要dynamic linker fix up，因此rel_type为R_IRELATIVE

      ifun: 间接函数。支持对一个函数创建多个实现，通过自己编写的resolver在运行时选择实现

      [http://sourceware.org/glibc/wiki/GNU_IFUNC](http://sourceware.org/glibc/wiki/GNU_IFUNC)

   3. pic且relative的情况，需要rel_type为R_IRELATIVE，否则不需要rel_type

   针对ordinary symbols获取地址时都是NO_PLT的，因为都是已知实现和地址，不需要动态链接。

2. TLVs

   TLV: thread local variarble

   根据是否为static的情况做不同的处理，是否为static由这两个编译选项所控制

   > --Bdynamic, --dy Link against shared libraries (default)
   > --Bstatic, --dn, --static Do not link against shared libraries

3. tls

   1. 针对符号是否为_TLS_MODULE_BASE_进行处理，唯一的区别是是否将符号关联进去，但是两者都需要设置rel_type为TLS_DESC

4. gottp_syms

   tp: thread pointer

   1. imported，这种符号所有信息未知，需要dynamic linker填充got entry，rel_type为R_TPOFF
   2. shared，知道offset，需要dynamic linker调整，rel_type为R_TPOFF
   3. other，链接时知道相对于tp的offset，所以能直接填写got entry

5. tlsld_idx

   1. 是否为static。static的情况下不需要rel，同时设置地址为1（表示main executable)否则需要设置rel为R_DTPMOD

总结一下

不需要设置rel_type的情况如下

1. ordinary symbol，pic且非relative符号的情况下，也就是说非pic或者pic但是没有relative符号（即不需要重定位）的情况下），不需要设置rel_type
2. TLVS为static的情况下不需要设置rel_type
3. 非shared以及imported的gottp symbol
4. tlsld_idx不为1且是static的情况

不过这里我有一个不明白的地方，为什么不需要rel_type的符号会在got中。查到的答案是

1. 作为函数的间接跳转入口:
   所有函数,包括不需要重定位的函数,在第一次调用时都需要通过.got表来间接跳转。即使函数在链接时就已经获得了绝对地址,但仍需要通过.got表调用。
2. 访问全局变量:
   程序中所有全局变量,包括不需要重定位的变量,都需要通过基址寄存器加上.got中的偏移量来访问。
   即使变量的值在链接时就已经确定,但程序仍需要通过.got表访问。
3. 作为函数指针:
   函数的地址可以被用作函数指针。而所有的函数指针,包括指向不需要重定位的函数的函数指针,都需要通过.got表来存取。
4. 链接器的要求:
   链接器要求所有函数和变量,无论是否需要重定位,都需要一个.got表项。这样它才能在程序加载时准确构建.got表。
5. 兼容性考虑:
   加入所有符号大大提高程序的兼容性。如果后续添加了需要重定位的符号,程序无需任何改动。
   所以,总之,.got表中的所有符号都是程序加载时解析的。
   即使符号不需要重定位,但仍需要通过.got表间接存取。主要是作为函数入口和变量、函数指针的访问入口。
   另外链接器及兼容性的要求也促使符号加入.got表。

```cpp
// Get .got and .rel.dyn contents.
//
// .got is a linker-synthesized constant pool whose entry is of pointer
// size. If we know a correct value for an entry, we'll just set that value
// to the entry. Otherwise, we'll create a dynamic relocation and let the
// dynamic linker to fill the entry at load-time.
//
// Most GOT entries contain addresses of global variable. If a global
// variable is an imported symbol, we don't know its address until runtime.
// GOT contains the addresses of such variables at runtime so that we can
// access imported global variables via GOT.
//
// Thread-local variables (TLVs) also use GOT entries. We need them because
// TLVs are accessed in a different way than the ordinary global variables.
// Their addresses are not unique; each thread has its own copy of TLVs.
template <typename E>
static std::vector<GotEntry<E>> get_got_entries(Context<E> &ctx) {
  std::vector<GotEntry<E>> entries;

  // Create GOT entries for ordinary symbols
  for (Symbol<E> *sym : ctx.got->got_syms) {
    i64 idx = sym->get_got_idx(ctx);

    // If a symbol is imported, let the dynamic linker to resolve it.
    if (sym->is_imported) {
      entries.push_back({idx, 0, E::R_GLOB_DAT, sym});
      continue;
    }

    // IFUNC always needs to be fixed up by the dynamic linker.
    if (sym->is_ifunc()) {
      entries.push_back({idx, sym->get_addr(ctx, NO_PLT), E::R_IRELATIVE});
      continue;
    }

    // If we know an address at link-time, fill that GOT entry now.
    // It may need a base relocation, though.
    if (ctx.arg.pic && sym->is_relative())
      entries.push_back({idx, sym->get_addr(ctx, NO_PLT), E::R_RELATIVE});
    else
      entries.push_back({idx, sym->get_addr(ctx, NO_PLT)});
  }

  // Create GOT entries for TLVs.
  for (Symbol<E> *sym : ctx.got->tlsgd_syms) {
    i64 idx = sym->get_tlsgd_idx(ctx);

    if (ctx.arg.is_static) {
      entries.push_back({idx, 1}); // One indicates the main executable file
      entries.push_back({idx + 1, sym->get_addr(ctx) - ctx.dtp_addr});
    } else {
      entries.push_back({idx, 0, E::R_DTPMOD, sym});
      entries.push_back({idx + 1, 0, E::R_DTPOFF, sym});
    }
  }

  if constexpr (supports_tlsdesc<E>) {
    for (Symbol<E> *sym : ctx.got->tlsdesc_syms) {
      // _TLS_MODULE_BASE_ is a linker-synthesized virtual symbol that
      // refers the begining of the TLS block.
      if (sym == ctx._TLS_MODULE_BASE_)
        entries.push_back({sym->get_tlsdesc_idx(ctx), 0, E::R_TLSDESC});
      else
        entries.push_back({sym->get_tlsdesc_idx(ctx), 0, E::R_TLSDESC, sym});
    }
  }

  for (Symbol<E> *sym : ctx.got->gottp_syms) {
    i64 idx = sym->get_gottp_idx(ctx);

    // If we know nothing about the symbol, let the dynamic linker
    // to fill the GOT entry.
    if (sym->is_imported) {
      entries.push_back({idx, 0, E::R_TPOFF, sym});
      continue;
    }

    // If we know the offset within the current thread vector,
    // let the dynamic linker to adjust it.
    if (ctx.arg.shared) {
      entries.push_back({idx, sym->get_addr(ctx) - ctx.tls_begin, E::R_TPOFF});
      continue;
    }

    // Otherwise, we know the offset from the thread pointer (TP) at
    // link-time, so we can fill the GOT entry directly.
    entries.push_back({idx, sym->get_addr(ctx) - ctx.tp_addr});
  }

  if (ctx.got->tlsld_idx != -1) {
    if (ctx.arg.is_static)
      entries.push_back({ctx.got->tlsld_idx, 1}); // 1 means the main executable
    else
      entries.push_back({ctx.got->tlsld_idx, 0, E::R_DTPMOD});
  }

  return entries;
}
```

## get_addr

这个函数是确定地址的过程。

首先说明PLT（Procedure Linkage Table），用于存放函数调用的跳转指令。主要用于提供函数入口点，实现间接调用。第一次调用对应函数时plt段被链接器处理，链接到函数的真实地址，也就是GOT中存放的具体值。

比如说某些符号在链接的时候是

```cpp
call fun@PLT
```

当调用fun后，这里的代码就会变成

```cpp
call *foo@GOT
```

另外是absolute符号，简单来说就是有一个固定的绝对地址的符号，因此可以直接获得其地址

[https://stackoverflow.com/questions/33324076/what-is-absolute-symbol-and-how-to-define-it-in-c](https://stackoverflow.com/questions/33324076/what-is-absolute-symbol-and-how-to-define-it-in-c)

1. 针对frag，非alive则是0，否则从frag中获取地址，
2. has copy rel，去从ctx中的copy_rel获取基地址
3. PPC64
4. plt，直接get_plt_addr
5. input section为空，absolute符号直接返回value的地址
6. input section非alive的情况
   1. killed by icf，从leader中获取地址
   2. eh_frame，根据符号名获取eh_frame中对应位置的地址
   3. 否则返回0
7. 普通的input section，直接isec→get_addr + value

下面代码中出现的value的含义如下，属于Symbol的成员

```cpp
// `value` contains symbol value. If it's an absolute symbol, it is
// equivalent to its address. If it belongs to an input section or a
// section fragment, value is added to the base of the input section
// to yield an address.
// u64 value = 0;
```

```cpp
template <typename E>
inline u64 SectionFragment<E>::get_addr(Context<E> &ctx) const {
  return output_section.shdr.sh_addr + offset;
}

// `value` contains symbol value. If it's an absolute symbol, it is
// equivalent to its address. If it belongs to an input section or a
// section fragment, value is added to the base of the input section
// to yield an address.
// u64 value = 0;

template <typename E>
inline u64 Symbol<E>::get_plt_addr(Context<E> &ctx) const {
  if (i32 idx = get_plt_idx(ctx); idx != -1)
    return ctx.plt->shdr.sh_addr + E::plt_hdr_size + idx * E::plt_size;
  return ctx.pltgot->shdr.sh_addr + get_pltgot_idx(ctx) * E::pltgot_size;
}

template <typename E>
inline i32 Symbol<E>::get_pltgot_idx(Context<E> &ctx) const {
  return (aux_idx == -1) ? -1 : ctx.symbol_aux[aux_idx].pltgot_idx;
}

template<typename E>
inline bool InputSection<E>::is_killed_by_icf() const {
  return this->leader && this->leader != this;
}

template <typename E>
inline u64 Symbol<E>::get_addr(Context<E> &ctx, i64 flags) const {
  if (SectionFragment<E> *frag = get_frag()) {
    if (!frag->is_alive) {
      // This condition is met if a non-alloc section refers an
      // alloc section and if the referenced piece of data is
      // garbage-collected. Typically, this condition occurs if a
      // debug info section refers a string constant in .rodata.
      return 0;
    }

    return frag->get_addr(ctx) + value;
  }

  if (has_copyrel) {
    return copyrel_readonly
      ? ctx.copyrel_relro->shdr.sh_addr + value
      : ctx.copyrel->shdr.sh_addr + value;
  }

  if constexpr (std::is_same_v<E, PPC64V1>)
    if (!(flags & NO_OPD) && has_opd(ctx))
      return get_opd_addr(ctx);

  if (!(flags & NO_PLT) && has_plt(ctx)) {
    assert(is_imported || is_ifunc());
    return get_plt_addr(ctx);
  }

  InputSection<E> *isec = get_input_section();
  if (!isec)
    return value; // absolute symbol

  if (!isec->is_alive) {
    if (isec->is_killed_by_icf())
      return isec->leader->get_addr() + value;

    if (isec->name() == ".eh_frame") {
      // .eh_frame contents are parsed and reconstructed by the linker,
      // so pointing to a specific location in a source .eh_frame
      // section doesn't make much sense. However, CRT files contain
      // symbols pointing to the very beginning and ending of the section.
      if (name() == "__EH_FRAME_BEGIN__" || name() == "__EH_FRAME_LIST__" ||
          name() == ".eh_frame_seg" || esym().st_type == STT_SECTION)
        return ctx.eh_frame->shdr.sh_addr;

      if (name() == "__FRAME_END__" || name() == "__EH_FRAME_LIST_END__")
        return ctx.eh_frame->shdr.sh_addr + ctx.eh_frame->shdr.sh_size;

      // ARM object files contain "$d" local symbol at the beginning
      // of data sections. Their values are not significant for .eh_frame,
      // so we just treat them as offset 0.
      if (name() == "$d" || name().starts_with("$d."))
        return ctx.eh_frame->shdr.sh_addr;

      Fatal(ctx) << "symbol referring .eh_frame is not supported: "
                 << *this << " " << *file;
    }

    // The control can reach here if there's a relocation that refers
    // a local symbol belonging to a comdat group section. This is a
    // violation of the spec, as all relocations should use only global
    // symbols of comdat members. However, .eh_frame tends to have such
    // relocations.
    return 0;
  }

  return isec->get_addr() + value;
}
```

# dynsym finalize

```cpp
// Reserve a space for dynamic symbol strings in .dynstr and sort
// .dynsym contents if necessary. Beyond this point, no symbol will
// be added to .dynsym.
ctx.dynsym->finalize(ctx);
```

为dynamic symbol的字符串在dynstr中留出空间，并且排序dynsym的内容。在这之后不会有符号被加入到dynsym，因此这里dynstr section的大小以及排布确定下来了。

具体的处理过程如下

1. symbols排序，local在前global在后，和elf中的格式一样。
2. 处理gnu_hash的情况
3. 设置dynsym_offset后计算dynstr的size
4. 更新DynsymSection的shdr的信息

```cpp
template <typename E>
void DynsymSection<E>::finalize(Context<E> &ctx) {
  Timer t(ctx, "DynsymSection::finalize");
  if (symbols.empty())
    return;

  // Sort symbols. In any symtab, local symbols must precede global symbols.
  auto first_global = std::stable_partition(symbols.begin() + 1, symbols.end(),
                                            [&](Symbol<E> *sym) {
    return sym->is_local(ctx);
  });

  // We also place undefined symbols before defined symbols for .gnu.hash.
  // Defined symbols are sorted by their hashes for .gnu.hash.
  if (ctx.gnu_hash) {
    // Count the number of exported symbols to compute the size of .gnu.hash.
    i64 num_exported = 0;
    for (i64 i = 1; i < symbols.size(); i++)
      if (symbols[i]->is_exported)
        num_exported++;

    u32 num_buckets = num_exported / ctx.gnu_hash->LOAD_FACTOR + 1;
    ctx.gnu_hash->num_buckets = num_buckets;

    tbb::parallel_for((i64)(first_global - symbols.begin()), (i64)symbols.size(),
                      [&](i64 i) {
      Symbol<E> &sym = *symbols[i];
      sym.set_dynsym_idx(ctx, i);
      sym.set_djb_hash(ctx, djb_hash(sym.name()));
    });

    tbb::parallel_sort(first_global, symbols.end(),
                       [&](Symbol<E> *a, Symbol<E> *b) {
      if (a->is_exported != b->is_exported)
        return b->is_exported;

      u32 h1 = a->get_djb_hash(ctx) % num_buckets;
      u32 h2 = b->get_djb_hash(ctx) % num_buckets;
      return std::tuple(h1, a->get_dynsym_idx(ctx)) <
             std::tuple(h2, b->get_dynsym_idx(ctx));
    });
  }

  // Compute .dynstr size
  ctx.dynstr->dynsym_offset = ctx.dynstr->shdr.sh_size;

  for (i64 i = 1; i < symbols.size(); i++) {
    symbols[i]->set_dynsym_idx(ctx, i);
    ctx.dynstr->shdr.sh_size += symbols[i]->name().size() + 1;
  }

  // ELF's symbol table sh_info holds the offset of the first global symbol.
  this->shdr.sh_info = first_global - symbols.begin();
}
```

```cpp
template <typename E>
inline void Symbol<E>::set_dynsym_idx(Context<E> &ctx, i32 idx) {
  assert(aux_idx != -1);
  ctx.symbol_aux[aux_idx].dynsym_idx = idx;
}
```

这里的ctx.symbol_aux[aux_idx].dynsym_idx是在之前的scan_relocations的过程中设置的，对应的dynsym_idx默认为-1

# report_undef_error

```cpp
// Print reports about undefined symbols, if needed.
if (ctx.arg.unresolved_symbols == UNRESOLVED_ERROR)
  report_undef_errors(ctx);
```

```cpp
// Report all undefined symbols, grouped by symbol.
template <typename E>
void report_undef_errors(Context<E> &ctx) {
  constexpr i64 max_errors = 3;

  for (auto &pair : ctx.undef_errors) {
    std::string_view sym_name = pair.first;
    std::span<std::string> errors = pair.second;

    if (ctx.arg.demangle)
      sym_name = demangle(sym_name);

    std::stringstream ss;
    ss << "undefined symbol: " << sym_name << "\n";

    for (i64 i = 0; i < errors.size() && i < max_errors; i++)
      ss << errors[i];

    if (errors.size() > max_errors)
      ss << ">>> referenced " << (errors.size() - max_errors) << " more times\n";

    if (ctx.arg.unresolved_symbols == UNRESOLVED_ERROR)
      Error(ctx) << ss.str();
    else if (ctx.arg.unresolved_symbols == UNRESOLVED_WARN)
      Warn(ctx) << ss.str();
  }

  ctx.checkpoint();
}
```

报告之前在claim_unresolved_symbols中收集的undef的错误信息
