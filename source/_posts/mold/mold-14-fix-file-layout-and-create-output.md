---
title: mold源码阅读十四 固定文件layout以及创建输出
typora-root-url: ../../source
date: 2023-07-26 00:07:02
category: Linker
tags:
  - [mold] 
  - [rel]
---

# mold源码阅读十四 fix file layout and create output

![Untitled](/images/mold-14-fix-file-layout-and-create-output/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:92848682</center> 

上一期主要讲解了shdr计算更新的部分以及osec offset的设置，这期则是做链接最后的工作。上期在对段shrink的时候也提到部分synthetic的符号值还未固定，本期就会从这部分的值提起，之后则是对debug_section进行压缩，同时文件的大小也会产生变化，到了这里整个文件内部的layout以及文件的大小也就固定了。

接下来就是创建output file，将数据实际拷贝到对应的输出buffer中，实际apply relocate，以及一些其他的操作，此时链接的产物已经完成了。

# fix_synthetic_symbols

```cpp
// Set actual addresses to linker-synthesized symbols.
fix_synthetic_symbols(ctx);
```

这里主要的任务是设置synthetic符号的值以及对应的origin。设置值的过程大部分都是设置对应chunk的shdr，origin则是标识符号来源，其他细节暂且不进行介绍，后面会单独一期详细查看所有synthetic的符号以及synthetic的section在整个链接过程中的行为，符号的具体作用等。

```cpp
template <typename E>
void fix_synthetic_symbols(Context<E> &ctx) {
  auto start = [](Symbol<E> *sym, auto &chunk, i64 bias = 0) {
    if (sym && chunk) {
      sym->set_output_section(chunk);
      sym->value = chunk->shdr.sh_addr + bias;
    }
  };

  auto stop = [](Symbol<E> *sym, auto &chunk) {
    if (sym && chunk) {
      sym->set_output_section(chunk);
      sym->value = chunk->shdr.sh_addr + chunk->shdr.sh_size;
    }
  };

  std::vector<Chunk<E> *> sections;
  for (Chunk<E> *chunk : ctx.chunks)
    if (chunk->kind() != HEADER && (chunk->shdr.sh_flags & SHF_ALLOC))
      sections.push_back(chunk);

  auto find = [&](std::string name) -> Chunk<E> * {
    for (Chunk<E> *chunk : sections)
      if (chunk->name == name)
        return chunk;
    return nullptr;
  };

  // __bss_start
  if (Chunk<E> *chunk = find(".bss"))
    start(ctx.__bss_start, chunk);

  if (ctx.ehdr && (ctx.ehdr->shdr.sh_flags & SHF_ALLOC)) {
    ctx.__ehdr_start->set_output_section(sections[0]);
    ctx.__ehdr_start->value = ctx.ehdr->shdr.sh_addr;
    ctx.__executable_start->set_output_section(sections[0]);
    ctx.__executable_start->value = ctx.ehdr->shdr.sh_addr;
  }

  if (ctx.__dso_handle) {
    ctx.__dso_handle->set_output_section(sections[0]);
    ctx.__dso_handle->value = sections[0]->shdr.sh_addr;
  }

  // __rel_iplt_start and __rel_iplt_end. These symbols need to be
  // defined in a statically-linked non-relocatable executable because
  // such executable lacks the .dynamic section and thus there's no way
  // to find ifunc relocations other than these symbols.
  //
  // We don't want to set values to these symbols if we are creating a
  // static PIE due to a glibc bug. Static PIE has a dynamic section.
  // If we set values to these symbols in a static PIE, glibc attempts
  // to run ifunc initializers twice, with the second attempt with wrong
  // function addresses, causing a segmentation fault.
  if (ctx.reldyn && ctx.arg.is_static && !ctx.arg.pie) {
    stop(ctx.__rel_iplt_start, ctx.reldyn);
    stop(ctx.__rel_iplt_end, ctx.reldyn);

    ctx.__rel_iplt_start->value -=
      get_num_irelative_relocs(ctx) * sizeof(ElfRel<E>);
  }

  // __{init,fini}_array_{start,end}
  for (Chunk<E> *chunk : sections) {
    switch (chunk->shdr.sh_type) {
    case SHT_INIT_ARRAY:
      start(ctx.__init_array_start, chunk);
      stop(ctx.__init_array_end, chunk);
      break;
    case SHT_PREINIT_ARRAY:
      start(ctx.__preinit_array_start, chunk);
      stop(ctx.__preinit_array_end, chunk);
      break;
    case SHT_FINI_ARRAY:
      start(ctx.__fini_array_start, chunk);
      stop(ctx.__fini_array_end, chunk);
      break;
    }
  }

  // _end, _etext, _edata and the like
  for (Chunk<E> *chunk : sections) {
    if (chunk->shdr.sh_flags & SHF_ALLOC) {
      stop(ctx._end, chunk);
      stop(ctx.end, chunk);
    }

    if (chunk->shdr.sh_flags & SHF_EXECINSTR) {
      stop(ctx._etext, chunk);
      stop(ctx.etext, chunk);
    }

    if (chunk->shdr.sh_type != SHT_NOBITS &&
        (chunk->shdr.sh_flags & SHF_ALLOC)) {
      stop(ctx._edata, chunk);
      stop(ctx.edata, chunk);
    }
  }

  // _DYNAMIC
  start(ctx._DYNAMIC, ctx.dynamic);

  // _GLOBAL_OFFSET_TABLE_. I don't know why, but for the sake of
  // compatibility with existing code, it must be set to the beginning of
  // .got.plt instead of .got only on i386 and x86-64.
  if constexpr (is_x86<E>)
    start(ctx._GLOBAL_OFFSET_TABLE_, ctx.gotplt);
  else
    start(ctx._GLOBAL_OFFSET_TABLE_, ctx.got);

  // _PROCEDURE_LINKAGE_TABLE_. We need this on SPARC.
  start(ctx._PROCEDURE_LINKAGE_TABLE_, ctx.plt);

  // _TLS_MODULE_BASE_. This symbol is used to obtain the address of
  // the TLS block in the TLSDESC model. I believe GCC and Clang don't
  // create a reference to it, but Intel compiler seems to be using
  // this symbol.
  if (ctx._TLS_MODULE_BASE_) {
    ctx._TLS_MODULE_BASE_->set_output_section(sections[0]);
    ctx._TLS_MODULE_BASE_->value = ctx.tls_begin;
  }

  // __GNU_EH_FRAME_HDR
  start(ctx.__GNU_EH_FRAME_HDR, ctx.eh_frame_hdr);

  // RISC-V's __global_pointer$
  if (ctx.__global_pointer) {
    if (Chunk<E> *chunk = find(".sdata")) {
      start(ctx.__global_pointer, chunk, 0x800);
    } else {
      ctx.__global_pointer->set_output_section(sections[0]);
      ctx.__global_pointer->value = 0;
    }
  }

  // ARM32's __exidx_{start,end}
  if (ctx.__exidx_start) {
    if (Chunk<E> *chunk = find(".ARM.exidx")) {
      start(ctx.__exidx_start, chunk);
      stop(ctx.__exidx_end, chunk);
    }
  }

  // PPC64's ".TOC." symbol.
  if (ctx.TOC) {
    if (Chunk<E> *chunk = find(".got")) {
      start(ctx.TOC, chunk, 0x8000);
    } else if (Chunk<E> *chunk = find(".toc")) {
      start(ctx.TOC, chunk, 0x8000);
    } else {
      ctx.TOC->set_output_section(sections[0]);
      ctx.TOC->value = 0;
    }
  }

  // __start_ and __stop_ symbols
  for (Chunk<E> *chunk : sections) {
    if (std::optional<std::string> name = get_start_stop_name(ctx, *chunk)) {
      start(get_symbol(ctx, save_string(ctx, "__start_" + *name)), chunk);
      stop(get_symbol(ctx, save_string(ctx, "__stop_" + *name)), chunk);

      if (ctx.arg.physical_image_base) {
        u64 paddr = to_paddr(ctx, chunk->shdr.sh_addr);

        Symbol<E> *x = get_symbol(ctx, save_string(ctx, "__phys_start_" + *name));
        x->set_output_section(chunk);
        x->value = paddr;

        Symbol<E> *y = get_symbol(ctx, save_string(ctx, "__phys_stop_" + *name));
        y->set_output_section(chunk);
        y->value = paddr + chunk->shdr.sh_size;
      }
    }
  }

  // --defsym=sym=value symbols
  for (i64 i = 0; i < ctx.arg.defsyms.size(); i++) {
    Symbol<E> *sym = ctx.arg.defsyms[i].first;
    std::variant<Symbol<E> *, u64> val = ctx.arg.defsyms[i].second;

    if (u64 *addr = std::get_if<u64>(&val)) {
      sym->origin = 0;
      sym->value = *addr;
      continue;
    }

    Symbol<E> *sym2 = std::get<Symbol<E> *>(val);
    if (!sym2->file) {
      Error(ctx) << "--defsym: undefined symbol: " << *sym2;
      continue;
    }

    sym->value = sym2->value;
    sym->origin = sym2->origin;
    sym->visibility = sym2->visibility.load();
  }

  // --section-order symbols
  for (SectionOrder &ord : ctx.arg.section_order)
    if (ord.type == SectionOrder::SYMBOL)
      get_symbol(ctx, ord.name)->set_output_section(sections[0]);
}
```

# compress_debug_sections

```cpp
// If --compress-debug-sections is given, compress .debug_* sections
// using zlib.
if (ctx.arg.compress_debug_sections != COMPRESS_NONE)
  filesize = compress_debug_sections(ctx);
```

压缩了所有debug相关的section，由于压缩了section，段的size发生改变，offset也会随之改变，因此之后还需要更新相关表的shdr，最后还会返回新的file size。具体的压缩过程这里就不详细看了。

> --compress-debug-sections [none,zlib,zlib-gabi,zstd]
>                                         Compress .debug_* sections

```cpp
template <typename E>
i64 compress_debug_sections(Context<E> &ctx) {
  Timer t(ctx, "compress_debug_sections");

  tbb::parallel_for((i64)0, (i64)ctx.chunks.size(), [&](i64 i) {
    Chunk<E> &chunk = *ctx.chunks[i];

    if ((chunk.shdr.sh_flags & SHF_ALLOC) || chunk.shdr.sh_size == 0 ||
        !chunk.name.starts_with(".debug"))
      return;

    Chunk<E> *comp = new CompressedSection<E>(ctx, chunk);
    ctx.chunk_pool.emplace_back(comp);
    ctx.chunks[i] = comp;
  });

  ctx.shstrtab->update_shdr(ctx);

  if (ctx.ehdr)
    ctx.ehdr->update_shdr(ctx);
  if (ctx.shdr)
    ctx.shdr->update_shdr(ctx);

  return set_osec_offsets(ctx);
}
```

```cpp
template <typename E>
class CompressedSection : public Chunk<E> {
public:
  CompressedSection(Context<E> &ctx, Chunk<E> &chunk);
  void copy_buf(Context<E> &ctx) override;
  u8 *get_uncompressed_data() override { return uncompressed.get(); }

private:
  ElfChdr<E> chdr = {};
  std::unique_ptr<Compressor> compressed;
  std::unique_ptr<u8[]> uncompressed;
};
```

```cpp
template <typename E>
CompressedSection<E>::CompressedSection(Context<E> &ctx, Chunk<E> &chunk) {
  assert(chunk.name.starts_with(".debug"));
  this->name = chunk.name;

  uncompressed.reset(new u8[chunk.shdr.sh_size]);
  chunk.write_to(ctx, uncompressed.get());

  switch (ctx.arg.compress_debug_sections) {
  case COMPRESS_ZLIB:
    chdr.ch_type = ELFCOMPRESS_ZLIB;
    compressed.reset(new ZlibCompressor(uncompressed.get(), chunk.shdr.sh_size));
    break;
  case COMPRESS_ZSTD:
    chdr.ch_type = ELFCOMPRESS_ZSTD;
    compressed.reset(new ZstdCompressor(uncompressed.get(), chunk.shdr.sh_size));
    break;
  default:
    unreachable();
  }

  chdr.ch_size = chunk.shdr.sh_size;
  chdr.ch_addralign = chunk.shdr.sh_addralign;

  this->shdr = chunk.shdr;
  this->shdr.sh_flags |= SHF_COMPRESSED;
  this->shdr.sh_addralign = 1;
  this->shdr.sh_size = sizeof(chdr) + compressed->compressed_size;
  this->shndx = chunk.shndx;

  // We don't need to keep the original data unless --gdb-index is given.
  if (!ctx.arg.gdb_index)
    uncompressed.reset(nullptr);
}
```

# create output file

到这个位置，所有memory以及file中的layout都就固定了，因此开始准备创建输出文件并且将chunks拷贝到output file中。

```cpp
// Create an output file
ctx.output_file =
  OutputFile<Context<E>>::open(ctx, ctx.arg.output, filesize, 0777);
ctx.buf = ctx.output_file->buf;
```

这里的filesize是上一期的set_osec中最后得到的offset（如果经过压缩过debug_section那么就是上面压缩后的filesize），0777则是文件的权限

# copy chunks

```cpp
// Copy input sections to the output file and apply relocations.
copy_chunks(ctx);
```

这里遍历了所有chunk并且每个都拷贝到输出文件中。但是先拷贝了非rel的段，之后才拷贝所有rel段，因为在copy output section的时候会apply relocate，在rel_offset的位置写入数据，而在后面rel段copy_buf的时候还可能向同样的地址写入数据。

这里会介绍一下一些主要的copy_chunk的实现（RelSection，OutputSection），其他synthetic符号的细节等到之后的文章再看细节。

```cpp
// Copy chunks to an output file
template <typename E>
void copy_chunks(Context<E> &ctx) {
  Timer t(ctx, "copy_chunks");

  auto copy = [&](Chunk<E> &chunk) {
    std::string name = chunk.name.empty() ? "(header)" : std::string(chunk.name);
    Timer t2(ctx, name, &t);
    chunk.copy_buf(ctx);
  };

  // For --relocatable and --emit-relocs, we want to copy non-relocation
  // sections first. This is because REL-type relocation sections (as
  // opposed to RELA-type) stores relocation addends to target sections.
  tbb::parallel_for_each(ctx.chunks, [&](Chunk<E> *chunk) {
    if (chunk->shdr.sh_type != (is_rela<E> ? SHT_RELA : SHT_REL))
      copy(*chunk);
  });

  tbb::parallel_for_each(ctx.chunks, [&](Chunk<E> *chunk) {
    if (chunk->shdr.sh_type == (is_rela<E> ? SHT_RELA : SHT_REL))
      copy(*chunk);
  });

  report_undef_errors(ctx);

  if constexpr (std::is_same_v<E, ARM32>)
    fixup_arm_exidx_section(ctx);
}
```

## rel的查找过程

不论是否为rel的output section，都需要有一个定位rel具体位置的过程。首先会先找到所在的osec，一个osec由多个输入的isec组成，每个isec根据其offset在osec中定位，找到具体的isec后则是找到相关的所有rel段

## OutputSection

对nobits的output section写入数据

1. 拷贝InputSections的内容到output file中
   1. copy数据本身
   2. apply relocate
2. 清理掉trail padding（设置为0）
3. 处理thunk

```cpp
template <typename E>
void OutputSection<E>::copy_buf(Context<E> &ctx) {
  if (this->shdr.sh_type != SHT_NOBITS)
    write_to(ctx, ctx.buf + this->shdr.sh_offset);
}

template <typename E>
void OutputSection<E>::write_to(Context<E> &ctx, u8 *buf) {
  auto clear = [&](u8 *loc, i64 size) {
    // As a special case, .init and .fini are filled with NOPs because the
    // runtime executes the sections as if they were a single function.
    // .init and .fini are superceded by .init_array and .fini_array and
    // being actively used only on s390x though.
    if (is_s390x<E> && (this->name == ".init" || this->name == ".fini")) {
      for (i64 i = 0; i < size; i += 2)
        *(ub16 *)(loc + i) = 0x0700; // nop
    } else {
      memset(loc, 0, size);
    }
  };

  tbb::parallel_for((i64)0, (i64)members.size(), [&](i64 i) {
    // Copy section contents to an output file
    InputSection<E> &isec = *members[i];
    isec.write_to(ctx, buf + isec.offset);

    // Clear trailing padding
    u64 this_end = isec.offset + isec.sh_size;
    u64 next_start = (i == members.size() - 1) ?
      (u64)this->shdr.sh_size : members[i + 1]->offset;
    clear(buf + this_end, next_start - this_end);
  });

  if constexpr (needs_thunk<E>) {
    tbb::parallel_for_each(thunks,
                           [&](std::unique_ptr<RangeExtensionThunk<E>> &thunk) {
      thunk->copy_buf(ctx);
    });
  }
}
```

![Untitled](/images/mold-14-fix-file-layout-and-create-output/Untitled%201.png)

根据osec→shdr.sh_addr以及isec.offset定位到具体的isec，并对每一个isec进行write_to

```cpp
template <typename E>
void InputSection<E>::write_to(Context<E> &ctx, u8 *buf) {
  if (shdr().sh_type == SHT_NOBITS || sh_size == 0)
    return;

  // Copy data
  if constexpr (is_riscv<E>) {
    copy_contents_riscv(ctx, buf);
  } else {
    uncompress_to(ctx, buf);
  }

  // Apply relocations
  if (!ctx.arg.relocatable) {
    if (shdr().sh_flags & SHF_ALLOC)
      apply_reloc_alloc(ctx, buf);
    else
      apply_reloc_nonalloc(ctx, buf);
  }
}

template <typename E>
void InputSection<E>::uncompress_to(Context<E> &ctx, u8 *buf) {
  if (!(shdr().sh_flags & SHF_COMPRESSED) || uncompressed) {
    memcpy(buf, contents.data(), contents.size());
    return;
  }

  if (contents.size() < sizeof(ElfChdr<E>))
    Fatal(ctx) << *this << ": corrupted compressed section";

  ElfChdr<E> &hdr = *(ElfChdr<E> *)&contents[0];
  std::string_view data = contents.substr(sizeof(ElfChdr<E>));

  switch (hdr.ch_type) {
  case ELFCOMPRESS_ZLIB: {
    unsigned long size = sh_size;
    if (::uncompress(buf, &size, (u8 *)data.data(), data.size()) != Z_OK)
      Fatal(ctx) << *this << ": uncompress failed";
    assert(size == sh_size);
    break;
  }
  case ELFCOMPRESS_ZSTD:
    if (ZSTD_decompress(buf, sh_size, (u8 *)data.data(), data.size()) != sh_size)
      Fatal(ctx) << *this << ": ZSTD_decompress failed";
    break;
  default:
    Fatal(ctx) << *this << ": unsupported compression type: 0x"
               << std::hex << hdr.ch_type;
  }
}
```

### copy数据

针对非压缩的数据则直接copy，对于压缩后的数据则进行解压

```cpp
template <typename E>
void InputSection<E>::uncompress_to(Context<E> &ctx, u8 *buf) {
  if (!(shdr().sh_flags & SHF_COMPRESSED) || uncompressed) {
    memcpy(buf, contents.data(), contents.size());
    return;
  }

  if (contents.size() < sizeof(ElfChdr<E>))
    Fatal(ctx) << *this << ": corrupted compressed section";

  ElfChdr<E> &hdr = *(ElfChdr<E> *)&contents[0];
  std::string_view data = contents.substr(sizeof(ElfChdr<E>));

  switch (hdr.ch_type) {
  case ELFCOMPRESS_ZLIB: {
    unsigned long size = sh_size;
    if (::uncompress(buf, &size, (u8 *)data.data(), data.size()) != Z_OK)
      Fatal(ctx) << *this << ": uncompress failed";
    assert(size == sh_size);
    break;
  }
  case ELFCOMPRESS_ZSTD:
    if (ZSTD_decompress(buf, sh_size, (u8 *)data.data(), data.size()) != sh_size)
      Fatal(ctx) << *this << ": ZSTD_decompress failed";
    break;
  default:
    Fatal(ctx) << *this << ": unsupported compression type: 0x"
               << std::hex << hdr.ch_type;
  }
}
```

### apply reloc alloc

这个过程也是因架构而异的，下面的代码来自rv

针对每个rel段的位置填写对应符号的地址，因为ElfRel本身不携带这个信息，对应的参数只有r_offset, r_type, r_sym，rela还会多一个r_addend。但根据rel类型的不同计算的方式也有些许的差异。具体的不同rel的计算方式要参考官方的文档，比如说rv的

[https://github.com/riscv-non-isa/riscv-elf-psabi-doc/blob/master/riscv-elf.adoc](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/blob/master/riscv-elf.adoc)

针对每个rel写入的loc的位置如图所示为osec→shdr.sh_addr + isec.offset + r_offset，不过注意这里的r_offset根据架构不同，可能会进行特殊处理，比如说下面rv的实现中有一个rel.r_offset - get_r_delta(i)的过程（之前shrink过程导致这里需要再处理delta的值）

另外apply_reloc_noalloc的过程也是类似，不再重复展示

![Untitled](/images/mold-14-fix-file-layout-and-create-output/Untitled%202.png)

```cpp
template <typename E>
void InputSection<E>::apply_reloc_alloc(Context<E> &ctx, u8 *base) {
  std::span<const ElfRel<E>> rels = get_rels(ctx);

  ElfRel<E> *dynrel = nullptr;
  if (ctx.reldyn)
    dynrel = (ElfRel<E> *)(ctx.buf + ctx.reldyn->shdr.sh_offset +
                           file.reldyn_offset + this->reldyn_offset);

  auto get_r_delta = [&](i64 idx) {
    return extra.r_deltas.empty() ? 0 : extra.r_deltas[idx];
  };

  for (i64 i = 0; i < rels.size(); i++) {
    const ElfRel<E> &rel = rels[i];
    if (rel.r_type == R_NONE || rel.r_type == R_RISCV_RELAX)
      continue;

    Symbol<E> &sym = *file.symbols[rel.r_sym];
    i64 r_offset = rel.r_offset - get_r_delta(i);
    i64 removed_bytes = get_r_delta(i + 1) - get_r_delta(i);
    u8 *loc = base + r_offset;

    auto check = [&](i64 val, i64 lo, i64 hi) {
      if (val < lo || hi <= val)
        Error(ctx) << *this << ": relocation " << rel << " against "
                   << sym << " out of range: " << val << " is not in ["
                   << lo << ", " << hi << ")";
    };

#define S   sym.get_addr(ctx)
#define A   rel.r_addend
#define P   (get_addr() + r_offset)
#define G   (sym.get_got_idx(ctx) * sizeof(Word<E>))
#define GOT ctx.got->shdr.sh_addr

    switch (rel.r_type) {
    case R_RISCV_32:
      if constexpr (E::is_64)
        *(U32<E> *)loc = S + A;
      else
        apply_dyn_absrel(ctx, sym, rel, loc, S, A, P, dynrel);
      break;
    case R_RISCV_64:
      assert(E::is_64);
      apply_dyn_absrel(ctx, sym, rel, loc, S, A, P, dynrel);
      break;
    case R_RISCV_BRANCH: {
      i64 val = S + A - P;
      check(val, -(1 << 12), 1 << 12);
      write_btype(loc, val);
      break;
    }
    case R_RISCV_JAL: {
      i64 val = S + A - P;
      check(val, -(1 << 20), 1 << 20);
      write_jtype(loc, val);
      break;
    }
    case R_RISCV_CALL:
    case R_RISCV_CALL_PLT: {
      u32 rd = get_rd(*(ul32 *)(contents.data() + rel.r_offset + 4));

      if (removed_bytes == 4) {
        // auipc + jalr -> jal
        *(ul32 *)loc = (rd << 7) | 0b1101111;
        write_jtype(loc, S + A - P);
      } else if (removed_bytes == 6 && rd == 0) {
        // auipc + jalr -> c.j
        *(ul16 *)loc = 0b101'00000000000'01;
        write_cjtype(loc, S + A - P);
      } else if (removed_bytes == 6 && rd == 1) {
        // auipc + jalr -> c.jal
        assert(!E::is_64);
        *(ul16 *)loc = 0b001'00000000000'01;
        write_cjtype(loc, S + A - P);
      } else {
        assert(removed_bytes == 0);
        u64 val = sym.esym().is_undef_weak() ? 0 : S + A - P;
        check(val, -(1LL << 31), 1LL << 31);
        write_utype(loc, val);
        write_itype(loc + 4, val);
      }
      break;
    }
    case R_RISCV_GOT_HI20:
      *(ul32 *)loc = G + GOT + A - P;
      break;
    case R_RISCV_TLS_GOT_HI20:
      *(ul32 *)loc = sym.get_gottp_addr(ctx) + A - P;
      break;
    case R_RISCV_TLS_GD_HI20:
      *(ul32 *)loc = sym.get_tlsgd_addr(ctx) + A - P;
      break;
    case R_RISCV_PCREL_HI20:
      if (sym.esym().is_undef_weak()) {
        // Calling an undefined weak symbol does not make sense.
        // We make such call into an infinite loop. This should
        // help debugging of a faulty program.
        *(ul32 *)loc = 0;
      } else {
        *(ul32 *)loc = S + A - P;
      }
      break;
    case R_RISCV_HI20: {
      i64 val = S + A;
      if (removed_bytes == 0) {
        check(val, -(1LL << 31), 1LL << 31);
        write_utype(loc, val);
      } else {
        assert(removed_bytes == 4);
        assert(sign_extend(val, 11) == val);
      }
      break;
    }
    case R_RISCV_LO12_I:
    case R_RISCV_LO12_S: {
      i64 val = S + A;
      if (rel.r_type == R_RISCV_LO12_I)
        write_itype(loc, val);
      else
        write_stype(loc, val);

      // Rewrite `lw t1, 0(t0)` with `lw t1, 0(x0)` if the address is
      // accessible relative to the zero register. If the upper 20 bits
      // are all zero, the corresponding LUI might have been removed.
      if (sign_extend(val, 11) == val)
        set_rs1(loc, 0);
      break;
    }
    case R_RISCV_TPREL_HI20:
      assert(removed_bytes == 0 || removed_bytes == 4);
      if (removed_bytes == 0)
        write_utype(loc, S + A - ctx.tp_addr);
      break;
    case R_RISCV_TPREL_ADD:
      break;
    case R_RISCV_TPREL_LO12_I:
    case R_RISCV_TPREL_LO12_S: {
      i64 val = S + A - ctx.tp_addr;
      if (rel.r_type == R_RISCV_TPREL_LO12_I)
        write_itype(loc, val);
      else
        write_stype(loc, val);

      // Rewrite `lw t1, 0(t0)` with `lw t1, 0(tp)` if the address is
      // directly accessible using tp. tp is x4.
      if (sign_extend(val, 11) == val)
        set_rs1(loc, 4);
      break;
    }
    case R_RISCV_ADD8:
      loc += S + A;
      break;
    case R_RISCV_ADD16:
      *(U16<E> *)loc += S + A;
      break;
    case R_RISCV_ADD32:
      *(U32<E> *)loc += S + A;
      break;
    case R_RISCV_ADD64:
      *(U64<E> *)loc += S + A;
      break;
    case R_RISCV_SUB8:
      loc -= S + A;
      break;
    case R_RISCV_SUB16:
      *(U16<E> *)loc -= S + A;
      break;
    case R_RISCV_SUB32:
      *(U32<E> *)loc -= S + A;
      break;
    case R_RISCV_SUB64:
      *(U64<E> *)loc -= S + A;
      break;
    case R_RISCV_ALIGN: {
      // A R_RISCV_ALIGN is followed by a NOP sequence. We need to remove
      // zero or more bytes so that the instruction after R_RISCV_ALIGN is
      // aligned to a given alignment boundary.
      //
      // We need to guarantee that the NOP sequence is valid after byte
      // removal (e.g. we can't remove the first 2 bytes of a 4-byte NOP).
      // For the sake of simplicity, we always rewrite the entire NOP sequence.
      i64 padding_bytes = rel.r_addend - removed_bytes;
      assert((padding_bytes & 1) == 0);

      i64 i = 0;
      for (; i <= padding_bytes - 4; i += 4)
        *(ul32 *)(loc + i) = 0x0000'0013; // nop
      if (i < padding_bytes)
        *(ul16 *)(loc + i) = 0x0001;      // c.nop
      break;
    }
    case R_RISCV_RVC_BRANCH: {
      i64 val = S + A - P;
      check(val, -(1 << 8), 1 << 8);
      write_cbtype(loc, val);
      break;
    }
    case R_RISCV_RVC_JUMP: {
      i64 val = S + A - P;
      check(val, -(1 << 11), 1 << 11);
      write_cjtype(loc, val);
      break;
    }
    case R_RISCV_SUB6:
      *loc = (*loc & 0b1100'0000) | ((*loc - (S + A)) & 0b0011'1111);
      break;
    case R_RISCV_SET6:
      *loc = (*loc & 0b1100'0000) | ((S + A) & 0b0011'1111);
      break;
    case R_RISCV_SET8:
      *loc = S + A;
      break;
    case R_RISCV_SET16:
      *(U16<E> *)loc = S + A;
      break;
    case R_RISCV_SET32:
      *(U32<E> *)loc = S + A;
      break;
    case R_RISCV_32_PCREL:
      *(U32<E> *)loc = S + A - P;
      break;
    case R_RISCV_PCREL_LO12_I:
    case R_RISCV_PCREL_LO12_S:
      // These relocations are handled in the next loop.
      break;
    default:
      unreachable();
    }

#undef S
#undef A
#undef P
#undef G
#undef GOT
  }

  // Handle PC-relative LO12 relocations. In the above loop, pcrel HI20
  // relocations overwrote instructions with full 32-bit values to allow
  // their corresponding pcrel LO12 relocations to read their values.
  for (i64 i = 0; i < rels.size(); i++) {
    switch (rels[i].r_type) {
    case R_RISCV_PCREL_LO12_I:
    case R_RISCV_PCREL_LO12_S: {
      Symbol<E> &sym = *file.symbols[rels[i].r_sym];
      assert(sym.get_input_section() == this);

      u8 *loc = base + rels[i].r_offset - get_r_delta(i);
      u32 val = *(ul32 *)(base + sym.value);

      if (rels[i].r_type == R_RISCV_PCREL_LO12_I)
        write_itype(loc, val);
      else
        write_stype(loc, val);
    }
    }
  }

  // Restore the original instructions pcrel HI20 relocations overwrote.
  for (i64 i = 0; i < rels.size(); i++) {
    switch (rels[i].r_type) {
    case R_RISCV_GOT_HI20:
    case R_RISCV_PCREL_HI20:
    case R_RISCV_TLS_GOT_HI20:
    case R_RISCV_TLS_GD_HI20: {
      u8 *loc = base + rels[i].r_offset - get_r_delta(i);
      u32 val = *(ul32 *)loc;
      memcpy(loc, contents.data() + rels[i].r_offset, 4);
      write_utype(loc, val);
    }
    }
  }
}
```

## rel

![Untitled](/images/mold-14-fix-file-layout-and-create-output/Untitled%203.png)

rel会先计算r_offset，值为对应osec的地址 + isec.offset + r_offset（来自输入的elf文件），r_type则保留，这个计算方式和上面apply_reloc的过程完全一致

之后的处理过程如下

1. 针对section外的符号直接获取其index，以及addend的信息并且设置值

2. section的符号则获取到对应的osec的shndx，设置addend为对应section的offset + get_addend()。其中get_addend的过程因架构而异。

   1. 针对SectionFragment则符号更改为output_section.shndx，原始符号或许是指向合并为fragment之前，由于已经merge到了一起，因此只能指向fragment所在的osec
   2. 针对普通section则直接设置为对应osec的shndx

3. 设置r_addend

   1. rela直接设置前面计算的addend

   2. 如果是relocatable，那么会根据rel的type在base + rel.r_offset的位置写入addend的值。这个base与上面的r_offset不同，但实际上都是指向最初计算的r_offset的位置，只是这里要写入文件，因此要以文件的buf为起点，而不是0。关于write_addend也是类似于get_addend，

      relocatable的情况最后write_addend的位置，也就是之前apply_reloc_alloc写入信息的位置，针对没有addend的情况只能将信息覆盖到这里

```cpp
template <typename E>
void RelocSection<E>::copy_buf(Context<E> &ctx) {
  auto write = [&](ElfRel<E> &out, InputSection<E> &isec, const ElfRel<E> &rel) {
    memset(&out, 0, sizeof(out));
    out.r_offset = isec.output_section->shdr.sh_addr + isec.offset + rel.r_offset;
    out.r_type = rel.r_type;

    Symbol<E> &sym = *isec.file.symbols[rel.r_sym];

    if (sym.esym().st_type == STT_SECTION) {
      i64 addend;

      if (SectionFragment<E> *frag = sym.get_frag()) {
        out.r_sym = frag->output_section.shndx;
        addend = frag->offset + sym.value + get_addend(isec, rel);
      } else {
        InputSection<E> *target = sym.get_input_section();
        OutputSection<E> *osec = target->output_section;
        out.r_sym = osec->shndx;
        addend = get_addend(isec, rel) + target->offset;
      }

      if constexpr (is_rela<E>) {
        out.r_addend = addend;
      } else if (ctx.arg.relocatable) {
        u8 *base = ctx.buf + isec.output_section->shdr.sh_offset + isec.offset;
        write_addend(base + rel.r_offset, addend, rel);
      }
    } else {
      if (sym.sym_idx)
        out.r_sym = sym.get_output_sym_idx(ctx);
      if constexpr (is_rela<E>)
        out.r_addend = rel.r_addend;
    }
  };

  tbb::parallel_for((i64)0, (i64)output_section.members.size(), [&](i64 i) {
    ElfRel<E> *buf = (ElfRel<E> *)(ctx.buf + this->shdr.sh_offset) + offsets[i];
    InputSection<E> &isec = *output_section.members[i];
    std::span<const ElfRel<E>> rels = isec.get_rels(ctx);

    for (i64 j = 0; j < rels.size(); j++)
      write(buf[j], isec, rels[j]);
  });
}

template <typename E>
inline i64 Symbol<E>::get_output_sym_idx(Context<E> &ctx) const {
  i64 i = file->output_sym_indices[sym_idx];
  assert(i != -1);
  if (is_local(ctx))
    return file->local_symtab_idx + i;
  return file->global_symtab_idx + i;
}
```

# gdb_index

```cpp
// Some part of .gdb_index couldn't be computed until other debug
// sections are complete. We have complete debug sections now, so
// write the rest of .gdb_index.
if (ctx.gdb_index)
  ctx.gdb_index->write_address_areas(ctx);
```

这里主要是gdb_index写入实际地址，因为在这里符号的地址都已经确定。

```cpp
template <typename E>
void GdbIndexSection<E>::write_address_areas(Context<E> &ctx) {
  Timer t(ctx, "GdbIndexSection::write_address_areas");

  if (this->shdr.sh_size == 0)
    return;

  u8 *base = ctx.buf + this->shdr.sh_offset;

  for (Chunk<E> *chunk : ctx.chunks) {
    std::string_view name = chunk->name;
    if (name == ".debug_info")
      ctx.debug_info = chunk;
    if (name == ".debug_abbrev")
      ctx.debug_abbrev = chunk;
    if (name == ".debug_ranges")
      ctx.debug_ranges = chunk;
    if (name == ".debug_addr")
      ctx.debug_addr = chunk;
    if (name == ".debug_rnglists")
      ctx.debug_rnglists = chunk;
  }

  assert(ctx.debug_info);
  assert(ctx.debug_abbrev);

  struct Entry {
    ul64 start;
    ul64 end;
    ul32 attr;
  };

  // Read address ranges from debug sections and copy them to .gdb_index.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    if (!file->debug_info)
      return;

    Entry *begin = (Entry *)(base + header.areas_offset + file->area_offset);
    Entry *e = begin;
    u64 offset = file->debug_info->offset;

    for (i64 i = 0; i < file->compunits.size(); i++) {
      std::vector<u64> addrs = read_address_areas(ctx, *file, offset);

      for (i64 j = 0; j < addrs.size(); j += 2) {
        // Skip an empty range
        if (addrs[j] == addrs[j + 1])
          continue;

        // Gdb crashes if there are entries with address 0.
        if (addrs[j] == 0)
          continue;

        assert(e < begin + file->num_areas);
        e->start = addrs[j];
        e->end = addrs[j + 1];
        e->attr = file->compunits_idx + i;
        e++;
      }
      offset += file->compunits[i].size();
    }

    // Fill trailing null entries with dummy values because gdb
    // crashes if there are entries with address 0.
    u64 filler;
    if (e == begin)
      filler = ctx.etext->get_addr(ctx) - 1;
    else
      filler = e[-1].start;

    for (; e < begin + file->num_areas; e++) {
      e->start = filler;
      e->end = filler;
      e->attr = file->compunits_idx;
    }
  });
}
```

# sort reldyn

```cpp
// Dynamic linker works better with sorted .rela.dyn section,
// so we sort them.
ctx.reldyn->sort(ctx);
```

对rel段排序，这么做的原理如注释所描述

```cpp
// This is the reason why we sort dynamic relocations. Quote from
// https://www.airs.com/blog/archives/186:
//
//   The dynamic linker in glibc uses a one element cache when processing
//   relocs: if a relocation refers to the same symbol as the previous
//   relocation, then the dynamic linker reuses the value rather than
//   looking up the symbol again. Thus the dynamic linker gets the best
//   results if the dynamic relocations are sorted so that all dynamic
//   relocations for a given dynamic symbol are adjacent.
//
//   Other than that, the linker sorts together all relative relocations,
//   which don't have symbols. Two relative relocations, or two relocations
//   against the same symbol, are sorted by the address in the output
//   file. This tends to optimize paging and caching when there are two
//   references from the same page.
//
// We group IFUNC relocations at the end of .rel.dyn because we want to
// apply all the other relocations before running user-supplied ifunc
// resolver functions.
```

大意如下：

1. glibc的linker有一个cache，如果一个relocation和前面的relocation引用了相同符号，那么会直2接引用值，而不是重新查找。
2. linker会将所有没有符号的relative relocation排序，两个relative relocation或者两个针对同一个符号的relocation会按照文件地址排序。存在同一页面的两个引用时可以优化分页和缓存

对于一个符号有多个relocation的情况，比如说一个全局变量被不同代码段引用多次，那么每个引用都需要生成一个条目。另外没有符号的relative relocation，是指重定位的记录中不包含符号，只包含偏移，比如说基于pc的相对寻址。

mold在.rel.dyn的末尾对IFUNC重定位进行分组,因为希望在运行用户提供的ifunc解析函数之前应用所有其他重定位。

排序规则基于如下三个方面

1. 根据r_type计算的rank
2. r_sym：重定位的符号在符号表中的索引
3. r_offset：重定位的位置

```cpp
template <typename E>
void RelDynSection<E>::sort(Context<E> &ctx) {
  Timer t(ctx, "sort_dynamic_relocs");

  ElfRel<E> *begin = (ElfRel<E> *)(ctx.buf + this->shdr.sh_offset);
  ElfRel<E> *end = (ElfRel<E> *)((u8 *)begin + this->shdr.sh_size);

  auto get_rank = [](u32 r_type) {
    switch (r_type) {
    case E::R_RELATIVE: return 0;
    case E::R_IRELATIVE: return 2;
    default: return 1;
    }
  };

  tbb::parallel_sort(begin, end, [&](const ElfRel<E> &a, const ElfRel<E> &b) {
    return std::tuple(get_rank(a.r_type), a.r_sym, a.r_offset) <
           std::tuple(get_rank(b.r_type), b.r_sym, b.r_offset);
  });
}
```

# clear_padding

```cpp
// Zero-clear paddings between sections
clear_padding(ctx);
```

将bss外的段中所有padding的空间设置为0，上一期只是设置offset来保证padding，但是padding范围内的值是未定的，在osec写到文件后再来将这部分空间置零。

```cpp
   		      size          padding
  |                     |         |
offset                       next_offset
```

```cpp
template <typename E>
void clear_padding(Context<E> &ctx) {
  Timer t(ctx, "clear_padding");

  auto zero = [&](Chunk<E> *chunk, i64 next_start) {
    i64 pos = chunk->shdr.sh_offset + chunk->shdr.sh_size;
    memset(ctx.buf + pos, 0, next_start - pos);
  };

  std::vector<Chunk<E> *> chunks = ctx.chunks;

  std::erase_if(chunks, [](Chunk<E> *chunk) {
    return chunk->shdr.sh_type == SHT_NOBITS;
  });

  for (i64 i = 1; i < chunks.size(); i++)
    zero(chunks[i - 1], chunks[i]->shdr.sh_offset);
  zero(chunks.back(), ctx.output_file->filesize);
}
```

# buildid

```cpp
// .note.gnu.build-id section contains a cryptographic hash of the
// entire output file. Now that we wrote everything except build-id,
// we can compute it.
if (ctx.buildid)
  ctx.buildid->write_buildid(ctx);
```

计算文件哈希，这对于elf来说并非必要的部分，但是有哈希可以用于校验文件是否完整是否有问题等，无需重新计算。

实际写入到header后的位置，因此写入地址是shdr.sh_offset + HEADER_SIZE。对于几种实现算法这里不再讨论。

> --build-id [none,md5,sha1,sha256,uuid,HEXSTRING] 
>                                      Generate build ID
> --no-build-id

```cpp
template <typename E>
class BuildIdSection : public Chunk<E> {
public:
  BuildIdSection() {
    this->name = ".note.gnu.build-id";
    this->shdr.sh_type = SHT_NOTE;
    this->shdr.sh_flags = SHF_ALLOC;
    this->shdr.sh_addralign = 4;
    this->shdr.sh_size = 1;
  }

  void update_shdr(Context<E> &ctx) override;
  void copy_buf(Context<E> &ctx) override;
  void write_buildid(Context<E> &ctx);

  static constexpr i64 HEADER_SIZE = 16;
};
```

```cpp
template <typename E>
void BuildIdSection<E>::write_buildid(Context<E> &ctx) {
  Timer t(ctx, "build_id");

  switch (ctx.arg.build_id.kind) {
  case BuildId::HEX:
    write_vector(ctx.buf + this->shdr.sh_offset + HEADER_SIZE,
                 ctx.arg.build_id.value);
    return;
  case BuildId::HASH:
    // Modern x86 processors have purpose-built instructions to accelerate
    // SHA256 computation, and SHA256 outperforms MD5 on such computers.
    // So, we always compute SHA256 and truncate it if smaller digest was
    // requested.
    compute_sha256(ctx, this->shdr.sh_offset + HEADER_SIZE);
    return;
  case BuildId::UUID: {
    std::array<u8, 16> uuid = get_uuid_v4();
    memcpy(ctx.buf + this->shdr.sh_offset + HEADER_SIZE, uuid.data(), 16);
    return;
  }
  default:
    unreachable();
  }
}
```

# close file

```cpp
// Close the output file. This is the end of the linker's main job.
ctx.output_file->close(ctx);
```

至此文件已经成功输出，只剩下最后的一些收尾工作，就留到下期再讲。
