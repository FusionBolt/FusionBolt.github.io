---
title: mold源码阅读十三 计算shdr以及osec offset
typora-root-url: ../../source
date: 2023-07-15 20:02:55
category: Linker
tags: 
  - [mold]
  - [shdr]
---

![Untitled](/images/mold-13-compute-shdr-and-set-osec-offsets/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:94763079</center> 

本期的内容主要是更新section header以及set output section offsets相关。当这些操作结束后，虚拟地址会固定，因此输出文件的memory layout就固定下来了。

# create_reloc_sections

```cpp
// If --emit-relocs is given, we'll copy relocation sections from input
// files to an output file.
if (ctx.arg.emit_relocs)
  create_reloc_sections(ctx);
```

这里主要做了2件事情

1. 这里将relocation段从input拷贝到output中，即设置所有OutputSection中reloc_sec
2. 加入到chunks中

```cpp
template <typename E>
void create_reloc_sections(Context<E> &ctx) {
  Timer t(ctx, "create_reloc_sections");

  // Create .rela.* sections
  tbb::parallel_for((i64)0, (i64)ctx.chunks.size(), [&](i64 i) {
    if (OutputSection<E> *osec = ctx.chunks[i]->to_osec())
      osec->reloc_sec.reset(new RelocSection<E>(ctx, *osec));
  });

  for (i64 i = 0, end = ctx.chunks.size(); i < end; i++)
    if (OutputSection<E> *osec = ctx.chunks[i]->to_osec())
      if (RelocSection<E> *x = osec->reloc_sec.get())
        ctx.chunks.push_back(x);
}
```

```cpp
template <typename E>
class RelocSection : public Chunk<E> {
public:
  RelocSection(Context<E> &ctx, OutputSection<E> &osec);
  void update_shdr(Context<E> &ctx) override;
  void copy_buf(Context<E> &ctx) override;

private:
  OutputSection<E> &output_section;
  std::vector<i64> offsets;
};
```

其中构造rel段的过程主要还是填写shdr的信息

```cpp
template <typename E>
RelocSection<E>::RelocSection(Context<E> &ctx, OutputSection<E> &osec)
  : output_section(osec) {
  if constexpr (is_rela<E>) {
    this->name = save_string(ctx, ".rela" + std::string(osec.name));
    this->shdr.sh_type = SHT_RELA;
  } else {
    this->name = save_string(ctx, ".rel" + std::string(osec.name));
    this->shdr.sh_type = SHT_REL;
  }

  this->shdr.sh_flags = SHF_INFO_LINK;
  this->shdr.sh_addralign = sizeof(Word<E>);
  this->shdr.sh_entsize = sizeof(ElfRel<E>);

  // Compute an offset for each input section
  offsets.resize(osec.members.size());

  auto scan = [&](const tbb::blocked_range<i64> &r, i64 sum, bool is_final) {
    for (i64 i = r.begin(); i < r.end(); i++) {
      InputSection<E> &isec = *osec.members[i];
      if (is_final)
        offsets[i] = sum;
      sum += isec.get_rels(ctx).size();
    }
    return sum;
  };

  i64 num_entries = tbb::parallel_scan(
    tbb::blocked_range<i64>(0, osec.members.size()), 0, scan, std::plus());

  this->shdr.sh_size = num_entries * sizeof(ElfRel<E>);
}
```

构造RelocSection的时候需要计算出rel的数量。每个OutputSection由多个InputSection组成，每个InputSection中又有多个rel段，这里遍历扫描计算出sum的数量。

这里构造的时候有rel和rela两种情况，它们有如下几种区别

1. rel只是简单的保存了需要被resolve的地址
2. rela保存了额外信息，其中的a是append。具体什么信息

> *SHT_RELA The section holds relocation entries with explicit addends, such as type*
> *Elf32_Rela for the 32-bit class of object files. An object file may have*
> *multiple relocation sections. See "Relocation'' below for details.*

另外RelocSection的shdr flag为SHF_INFO_LINK，意义如下

> This section headers sh_info field holds a section header table index.

设置sh_info的过程则是在后续compute_section_headers中

# compute_section_headers

```cpp
// Compute the section header values for all sections.
compute_section_headers(ctx);
```

这里主要做了这么几件事

1. 所有输出段更新shdr
2. 移除所有空的chunk
3. 重新设置所有chunk的index（因为上面移除了chunk，index发生了改变）
4. SymtabShndxSection的处理
5. 再次更新所有的shdr

```cpp
template <typename E>
void compute_section_headers(Context<E> &ctx) {
  // Update sh_size for each chunk.
  for (Chunk<E> *chunk : ctx.chunks)
    chunk->update_shdr(ctx);

  // Remove empty chunks.
  std::erase_if(ctx.chunks, [](Chunk<E> *chunk) {
    return chunk->kind() != OUTPUT_SECTION && chunk->shdr.sh_size == 0;
  });

  // Set section indices.
  i64 shndx = 1;
  for (i64 i = 0; i < ctx.chunks.size(); i++)
    if (ctx.chunks[i]->kind() != HEADER)
      ctx.chunks[i]->shndx = shndx++;

  if (ctx.symtab && SHN_LORESERVE <= shndx) {
    SymtabShndxSection<E> *sec = new SymtabShndxSection<E>;
    sec->shndx = shndx++;
    sec->shdr.sh_link = ctx.symtab->shndx;
    ctx.symtab_shndx = sec;
    ctx.chunks.push_back(sec);
    ctx.chunk_pool.emplace_back(sec);
  }

  if (ctx.shdr)
    ctx.shdr->shdr.sh_size = shndx * sizeof(ElfShdr<E>);

  // Some types of section header refer other section by index.
  // Recompute the section header to fill such fields with correct values.
  for (Chunk<E> *chunk : ctx.chunks)
    chunk->update_shdr(ctx);

  if (ctx.symtab_shndx) {
    i64 symtab_size = ctx.symtab->shdr.sh_size / sizeof(ElfSym<E>);
    ctx.symtab_shndx->shdr.sh_size = symtab_size * 4;
  }
}
```

## update_shdr

大多数synthetic的chunk都有自己的实现，一些类型的section header通过index引用了其他段，因此需要重新计算shdr中对应字段的值

```cpp
// Chunk represents a contiguous region in an output file.
template <typename E>
class Chunk {
  ...
  virtual void update_shdr(Context<E> &ctx) {}
	...
}
```

比如说OutputPhdr，在这里就需要update

```cpp
template <typename E>
void OutputPhdr<E>::update_shdr(Context<E> &ctx) {
  phdrs = create_phdr(ctx);
  this->shdr.sh_size = phdrs.size() * sizeof(ElfPhdr<E>);
}
```

## 移除空的chunk

这里选择了空的非OutputSection进行移除，判断是否为OutputSection则是根据ChunkKind

```cpp
typedef enum { HEADER, OUTPUT_SECTION, SYNTHETIC } ChunkKind;
```

其中HEADER是用于output的phdr，ehdr，shdr，chunk默认是SYNTHETIC，也就是说相当于最终只是删除一些空的synthetic的段

## 重新更新索引

在普通的根据chunk的序列设置索引后有一个SHN_LORESERVE的判断，和SHN_LORESERVE相关的信息有这些

[https://man7.org/linux/man-pages/man5/elf.5.html](https://man7.org/linux/man-pages/man5/elf.5.html)

```
e_shnum
              This member holds the number of entries in the section
              header table.  Thus the product of e_shent size and e_shnum
              gives the section header table's size in bytes.  If a file
              has no section header table,e_shnum holds the value of
              zero.

              If the number of entries in the section header table is
              larger than or equal to SHN_LORESERVE(0xff00), e_shnum
              holds the value zero and the real number of entries in the
              section header table is held in the sh_size member of the
              initial entry in section header table.  Otherwise, the
							sh_size member of the initial entry in the section header
              table holds the value zero.

e_shstrndx
              This member holds the section header table index of the
              entry associated with the section name string table.  If
              the file has no section name string table, this member
              holds the value SHN_UNDEF.

              If the index of section name string table section is
              larger than or equal to SHN_LORESERVE(0xff00), this
              member holds SHN_XINDEX(0xffff) and the real index of the
              section name string table section is held in thesh_link
              member of the initial entry in section header table.
              Otherwise, thesh_link member of the initial entry in
              section header table contains the value zero.
```

e_shnum和e_shstrndx是在EHDR中的信息。首先是shnum，当shdr table，也就是说section的数量超过SHN_LORESERVE的时候，e_shnum会设置为0，实际的数量会保存在shdr table的初始条目中的sh_size的字段里，其他情况这个条目的sh_size的字段是0。

这里创建了一个SymtabShndxSection，也就是”symtab_shndx”段，这个段保留了特殊的symbol table section index arrry，指向与符号表关联的shdr的索引。

> `.symtab_shndx`
>
>
> This section holds the special symbol table section index array, as described above. The section's attributes will include the `SHF_ALLOC` bit if the associated symbol table section does; otherwise that bit will be off.

这个section的sh_type为SHT_SYMTAB_SHNDX

> `SHT_SYMTAB_SHNDX`
>
>
> This section is associated with a section of type `SHT_SYMTAB` and is required if any of the section header indexes referenced by that symbol table contain the escape value `SHN_XINDEX`. The section is an array of `Elf32_Word` values. Each value corresponds one to one with a symbol table entry and appear in the same order as those entries. The values represent the section header indexes against which the symbol table entries are defined. Only if corresponding symbol table entry's `st_shndx` field contains the escape value `SHN_XINDEX` will the matching `Elf32_Word` hold the actual section header index; otherwise, the entry must be `SHN_UNDEF` (`0`).

关于e_shstrndx，这里也设置了ctx中的symtab_shndx。不论是e_shnum还是e_shstrndx都是在后续的过程中实际计算或者使用其信息，等到后面讲的时候再联系前面这些来看。

创建了SymtabShndxSection后则是设置其基本信息，至此compute_section_headers的过程就结束了。

# set_osec_offsets

```cpp
// Assign offsets to output sections
i64 filesize = set_osec_offsets(ctx);
```

设置所有output section的offset，根据是否有section_order会选择不同的排列方式，导致output的offset是不同的。

主要分为如下几部分

1. 设置段的虚拟地址
2. 设置段在文件中的offset
3. 处理phdr并且返回文件的长度

```cpp
// Assign virtual addresses and file offsets to output sections.
template <typename E>
i64 set_osec_offsets(Context<E> &ctx) {
  Timer t(ctx, "set_osec_offsets");

  for (;;) {
    if (ctx.arg.section_order.empty())
      set_virtual_addresses_regular(ctx);
    else
      set_virtual_addresses_by_order(ctx);

    i64 fileoff = set_file_offsets(ctx);

    // Assigning new offsets may change the contents and the length
    // of the program header, so repeat it until converge.
    if (!ctx.phdr)
      return fileoff;

    i64 sz = ctx.phdr->shdr.sh_size;
    ctx.phdr->update_shdr(ctx);
    if (sz == ctx.phdr->shdr.sh_size)
      return fileoff;
  }
}
```

## virtual address设置规则

```cpp
// This function assigns virtual addresses to output sections. Assigning
// addresses is a bit tricky because we want to pack sections as tightly
// as possible while not violating the constraints imposed by the hardware
// and the OS kernel. Specifically, we need to satisfy the following
// constraints:
//
// - Memory protection (readable, writable and executable) works at page
//   granularity. Therefore, if we want to set different memory attributes
//   to two sections, we need to place them into separate pages.
//
// - The ELF spec requires that a section's file offset is congruent to
//   its virtual address modulo the page size. For example, a section at
//   virtual address 0x401234 on x86-64 (4 KiB, or 0x1000 byte page
//   system) can be at file offset 0x3234 or 0x50234 but not at 0x1000.
//
// We need to insert paddings between sections if we can't satisfy the
// above constraints without them.
//
// We don't want to waste too much memory and disk space for paddings.
// There are a few tricks we can use to minimize paddings as below:
//
// - We want to place sections with the same memory attributes
//   contiguous as possible.
//
// - We can map the same file region to memory more than once. For
//   example, we can write code (with R and X bits) and read-only data
//   (with R bit) adjacent on file and map it twice as the last page of
//   the executable segment and the first page of the read-only data
//   segment. This doesn't save memory but saves disk space.
```

mold中不论是指定order还是不指定都是遵循基本的两条规则

1. memory protection: 不同attr的section分配到不同的页中，为了满足每个段只有一个attr的条件
2. ELF spec requires: vaddr的地址的模要是page size，这里会在设置地址的时候进行align，也就是insert padding。

这里提及的ticks:

1. place sections with the same memory attributes contiguous as possible. 相同attr尽可能连续。（因为不同attr需要放入不同的页）
2. map the same file region to memory more than once. map两次，因此不会节约内存空间，但是会节约磁盘空间

## set_virtual_addresses_regular

在设置地址之前，需要先对tls chunk计算一个align。因为tls chunk需要满足如下条件

1. tls块在vaddr中的起始地址地址需要对齐（普通内存块的要求是相同的）
2. 当被拷贝到新线程区域时tls_begin的offset也必须对齐。更具体的说，tls会有多个块，而每个块都有一个tls_begin，实际上是这个值要求对齐。

而mold的做法是选择其中最大的align值。

之后设置了起始地址image_base：默认0x200000

> -image-base ADDR Set the base address to a given value

regular设置地址的过程有如下几种情况

1. 跳过不需要alloc的段，因为并不需要加载到内存中
2. relro_padding，一定是满足memory protection的。
3. 指定section_start，不需要考虑memory protection，因为后面考虑这个的时候会和前一个的段进行比较。假设说连续两个都指定了start，即便不满足，那也是指定的行为。
4. 满足memory protection的条件。当前chunk和下一个chunk都不是relro_padding的情况，如果两个chunk的attr不同，那么处理。处理后继续执行后面的代码
5. tbss：SHF_TLS & SHT_NOBITS
6. 剩余的情况就是简单的进行align然后更新addr信息

```cpp
template <typename E>
static void set_virtual_addresses_regular(Context<E> &ctx) {
  constexpr i64 RELRO = 1LL << 32;

  auto get_flags = [&](Chunk<E> *chunk) {
    i64 flags = to_phdr_flags(ctx, chunk);
    if (is_relro(ctx, chunk))
      return flags | RELRO;
    return flags;
  };

  // Assign virtual addresses
  std::vector<Chunk<E> *> &chunks = ctx.chunks;
  u64 addr = ctx.arg.image_base;

  // TLS chunks alignments are special: in addition to having their virtual
  // addresses aligned, they also have to be aligned when the region of
  // tls_begin is copied to a new thread's storage area. In other words, their
  // offset against tls_begin also has to be aligned.
  //
  // A good way to achieve this is to take the largest alignment requirement
  // of all TLS sections and make tls_begin also aligned to that.
  Chunk<E> *first_tls_chunk = nullptr;
  u64 tls_alignment = 1;
  for (Chunk<E> *chunk : chunks) {
    if (chunk->shdr.sh_flags & SHF_TLS) {
      if (!first_tls_chunk)
        first_tls_chunk = chunk;
      tls_alignment = std::max(tls_alignment, (u64)chunk->shdr.sh_addralign);
    }
  }

  auto alignment = [&](Chunk<E> *chunk) {
    return chunk == first_tls_chunk ? tls_alignment : (u64)chunk->shdr.sh_addralign;
  };

  for (i64 i = 0; i < chunks.size(); i++) {
    if (!(chunks[i]->shdr.sh_flags & SHF_ALLOC))
      continue;

    // .relro_padding is a padding section to extend a PT_GNU_RELRO
    // segment to cover an entire page. Technically, we don't need a
    // .relro_padding section because we can leave a trailing part of a
    // segment an unused space. However, the `strip` command would delete
    // such an unused trailing part and make an executable invalid.
    // So we add a dummy section.
    if (chunks[i] == ctx.relro_padding) {
      chunks[i]->shdr.sh_addr = addr;
      chunks[i]->shdr.sh_size = align_to(addr, ctx.page_size) - addr;
      addr += ctx.page_size;
      continue;
    }

    // Handle --section-start first
    if (auto it = ctx.arg.section_start.find(chunks[i]->name);
        it != ctx.arg.section_start.end()) {
      addr = it->second;
      chunks[i]->shdr.sh_addr = addr;
      addr += chunks[i]->shdr.sh_size;
      continue;
    }

    // Memory protection works at page size granularity. We need to
    // put sections with different memory attributes into different
    // pages. We do it by inserting paddings here.
    if (i > 0 && chunks[i - 1] != ctx.relro_padding) {
      i64 flags1 = get_flags(chunks[i - 1]);
      i64 flags2 = get_flags(chunks[i]);

      if (flags1 != flags2) {
        switch (ctx.arg.z_separate_code) {
        case SEPARATE_LOADABLE_SEGMENTS:
          addr = align_to(addr, ctx.page_size);
          break;
        case SEPARATE_CODE:
          if ((flags1 & PF_X) != (flags2 & PF_X)) {
            addr = align_to(addr, ctx.page_size);
            break;
          }
          [[fallthrough]];
        case NOSEPARATE_CODE:
          if (addr % ctx.page_size != 0)
            addr += ctx.page_size;
          break;
        default:
          unreachable();
        }
      }
    }

    // TLS BSS sections are laid out so that they overlap with the
    // subsequent non-tbss sections. Overlapping is fine because a STT_TLS
    // segment contains an initialization image for newly-created threads,
    // and no one except the runtime reads its contents. Even the runtime
    // doesn't need a BSS part of a TLS initialization image; it just
    // leaves zero-initialized bytes as-is instead of copying zeros.
    // So no one really read tbss at runtime.
    //
    // We can instead allocate a dedicated virtual address space to tbss,
    // but that would be just a waste of the address and disk space.
    if (is_tbss(chunks[i])) {
      u64 addr2 = addr;
      for (;;) {
        addr2 = align_to(addr2, alignment(chunks[i]));
        chunks[i]->shdr.sh_addr = addr2;
        addr2 += chunks[i]->shdr.sh_size;
        if (i + 2 == chunks.size() || !is_tbss(chunks[i + 1]))
          break;
        i++;
      }
      continue;
    }

    addr = align_to(addr, alignment(chunks[i]));
    chunks[i]->shdr.sh_addr = addr;
    addr += chunks[i]->shdr.sh_size;
  }
}
```

### relro_padding

relro_padding用于确保RELRO页对齐，通常无需考虑，但是strip后会删除未使用的尾部空间，导致可执行文件无效。因此这里计算size的过程是用align的size减去当前的地址，之后addr直接递增一个page_size

关于PT_GNU_RELRO

> The array element specifies the location and size of a segment which may be made read-only after relocation shave been processed.

### section_start

根据命令行参数指定对应段的地址为指定的位置，更新当前段地址，并且递增对应段的size。其中提到的命令行参数的介绍

> -Tbss=ADDR Set address to .bss
> -Tdata Set address to .data
> -Ttext Set address to .text

### 满足memory protection

这里主要是针对相邻两个chunk的attr不同的情况。

根据链接选项可以分为三类

1. SEPARATE_LOADABLE_SEGMENTS
   1. 这里只是更新了当前的addr为根据page size进行align的值
2. SEPARATE_CODE
   1. 两个都不是PF_X（执行的权限）的情况下进行align
3. NOSEPARATE_CODE
   1. 如果当前的地址不是整除page size，那么直接增加一个page_size 

### to_phdr_flags

```cpp
template <typename E>
i64 to_phdr_flags(Context<E> &ctx, Chunk<E> *chunk) {
  // All sections are put into a single RWX segment if --omagic
  if (ctx.arg.omagic)
    return PF_R | PF_W | PF_X;

  bool write = (chunk->shdr.sh_flags & SHF_WRITE);
  bool exec = (chunk->shdr.sh_flags & SHF_EXECINSTR);

  // .text is not readable if --execute-only
  if (exec && ctx.arg.execute_only) {
    if (write)
      Error(ctx) << "--execute-only is not compatible with writable section: "
                 << chunk->name;
    return PF_X;
  }

  // .rodata is merged with .text if --no-rosegment
  if (!write && !ctx.arg.rosegment)
    exec = true;

  return PF_R | (write ? PF_W : 0) | (exec ? PF_X : 0);
}
```

对于页来说attr只有三种，读，写，执行。

如果指定omagic那么所有段都会塞到一个segment里面，因此其attr都是RWX的

> -N, --omagic Do not page align data, do not make text readonly 
> --no-omagic

之后则是根据shdr的flag判断是否可写可执行返回对应的flag

### tbss

这里是针对tbss(tls bss) section的处理，其判断逻辑为

```cpp
template <typename E>
static bool is_tbss(Chunk<E> *chunk) {
  return (chunk->shdr.sh_type == SHT_NOBITS) && (chunk->shdr.sh_flags & SHF_TLS);
}
```

.bss的SHT为SHT_NOBITS，并且其Flag为SHF_ALLOC & SHF_WRITE，因此这里只需要判断sh_type是否为SHT_NOBITS以及tls的判断条件就可以确定是否为tbss

> sh_offset: The byte offset from the beginning of the file to the first byte in the section. Section type SHT_NOBITS occupies no space in the file. Its sh_offset member locates the conceptual placement in the file.
>
>
> sh_size: The section's size in bytes. Unless the section type is SHT_NOBITS, the section occupies sh_size bytes in the file. A section of type SHT_NOBITS can have a nonzero size, but it occupies no space in the file.
>
> SHT_NOBITS: Identifies a section that occupies no space in the file but otherwise resembles SHT_PROGBITS. Although this section contains no bytes, the sh_offset member contains the conceptual file offset.

tbss会和后续的非tbss段重叠，因为STT_TLS segment包含了一个初始化image，运行时才会读取，运行时也不需要tbss的初始化image，会保留零初始的变化字节不变，所以运行时不会实际读取。尽管可以单独给tbss分配空间，但是会浪费地址和磁盘空间。

所以这里的代码单独创建了一个地址进行递增，不会影响到正常更新的地址。

## set_virtual_addresses_by_order

根据order的信息来设置地址

```cpp
enum { NONE, SECTION, GROUP, ADDR, ALIGN, SYMBOL } type = NONE;
```

根据SectionOrder信息的不同有如下几种情况处理

1. SECTION
   1. 计算一个地址赋给对应section的sh_addr以及更新当前的addr
2. GROUP
   1. 针对group段所有成员做和SECTION情况下相同的操作
3. ADDR
   1. 当前地址设置为SectionOrder中的值
4. ALIGN
   1. 当前地址设置为根据给定value对齐后的地址
5. SYMBOL
   1. 特定符号的地址设置为当前的地址

针对section的过程具体分为如下几步

1. 判断相邻段的attr，不同则进行将section根据page size进行padding
2. 根据对应段的align计算一个地址，赋给section并且更新shdr以及当前addr的信息
3. 找到下一个alloc的段

```cpp
template <typename E>
static void set_virtual_addresses_by_order(Context<E> &ctx) {
  std::vector<Chunk<E> *> &c = ctx.chunks;
  u64 addr = ctx.arg.image_base;
  i64 i = 0;

  while (i < c.size() && !(c[i]->shdr.sh_flags & SHF_ALLOC))
    i++;

  auto assign_addr = [&] {
    if (i != 0) {
      i64 flags1 = to_phdr_flags(ctx, c[i - 1]);
      i64 flags2 = to_phdr_flags(ctx, c[i]);

      // Memory protection works at page size granularity. We need to
      // put sections with different memory attributes into different
      // pages. We do it by inserting paddings here.
      if (flags1 != flags2) {
        switch (ctx.arg.z_separate_code) {
        case SEPARATE_LOADABLE_SEGMENTS:
          addr = align_to(addr, ctx.page_size);
          break;
        case SEPARATE_CODE:
          if ((flags1 & PF_X) != (flags2 & PF_X))
            addr = align_to(addr, ctx.page_size);
          break;
        default:
          break;
        }
      }
    }

    addr = align_to(addr, c[i]->shdr.sh_addralign);
    c[i]->shdr.sh_addr = addr;
    addr += c[i]->shdr.sh_size;

    do {
      i++;
    } while (i < c.size() && !(c[i]->shdr.sh_flags & SHF_ALLOC));
  };

  for (i64 j = 0; j < ctx.arg.section_order.size(); j++) {
    SectionOrder &ord = ctx.arg.section_order[j];
    switch (ord.type) {
    case SectionOrder::SECTION:
      if (i < c.size() && j == c[i]->sect_order)
        assign_addr();
      break;
    case SectionOrder::GROUP:
      while (i < c.size() && j == c[i]->sect_order)
        assign_addr();
      break;
    case SectionOrder::ADDR:
      addr = ord.value;
      break;
    case SectionOrder::ALIGN:
      addr = align_to(addr, ord.value);
      break;
    case SectionOrder::SYMBOL:
      get_symbol(ctx, ord.name)->value = addr;
      break;
    default:
      unreachable();
    }
  }
}
```

## set_file_offsets

在设置完段的虚拟地址后，需要设定段在文件中的offset

1. 不需要alloc的情况，仍然需要计算其align更新size。有一些段需要占用空间，但是不需要载入内存，因此前面设置虚拟地址的时候跳过了所有不需要alloc的段，这里计算offset的时候还是要考虑到的。
2. bss段不做处理直接跳过
3. 对offset做一次align。因为有的段可能并没有完整填充一个page size的空间，前面设置虚拟地址的过程并没有使得所有的size都满足page align。
4. 给alloc section设置尽可能连续的文件的offset，因此会从当前的chunk开始循环设置offset。这样连续加载提高了效率，并且减少碎片空间。不符合条件的情况如下
   1. 非alloc或者是bss
   2. 如果是给定了start_section的段，这样就无法设置连续的offset了，需要单独处理。包含了两种情况
      1. start的位置在开始循环的first chunk之前
      2. 相邻两个chunk的size过大，超过了page_size
5. 使用最后一个设置offset的chunk的信息更新offset
6. 找到下一个alloc的并且非SHT_NOBITS的chunk的index，之后处理下一个chunk

```cpp
// Assign file offsets to output sections.
template <typename E>
static i64 set_file_offsets(Context<E> &ctx) {
  std::vector<Chunk<E> *> &chunks = ctx.chunks;
  u64 fileoff = 0;
  i64 i = 0;

  while (i < chunks.size()) {
    Chunk<E> &first = *chunks[i];

    if (!(first.shdr.sh_flags & SHF_ALLOC)) {
      fileoff = align_to(fileoff, first.shdr.sh_addralign);
      first.shdr.sh_offset = fileoff;
      fileoff += first.shdr.sh_size;
      i++;
      continue;
    }

    if (first.shdr.sh_type == SHT_NOBITS) {
      i++;
      continue;
    }

    if (first.shdr.sh_addralign > ctx.page_size)
      fileoff = align_to(fileoff, first.shdr.sh_addralign);
    else
      fileoff = align_with_skew(fileoff, ctx.page_size, first.shdr.sh_addr);

    // Assign ALLOC sections contiguous file offsets as long as they
    // are contiguous in memory.
    for (;;) {
      chunks[i]->shdr.sh_offset =
        fileoff + chunks[i]->shdr.sh_addr - first.shdr.sh_addr;
      i++;

      if (i >= chunks.size() ||
          !(chunks[i]->shdr.sh_flags & SHF_ALLOC) ||
          chunks[i]->shdr.sh_type == SHT_NOBITS)
        break;

      // If --start-section is given, addresses may not increase
      // monotonically.
      if (chunks[i]->shdr.sh_addr < first.shdr.sh_addr)
        break;

      i64 gap_size = chunks[i]->shdr.sh_addr - chunks[i - 1]->shdr.sh_addr -
                     chunks[i - 1]->shdr.sh_size;

      // If --start-section is given, there may be a large gap between
      // sections. We don't want to allocate a disk space for a gap if
      // exists.
      if (gap_size >= ctx.page_size)
        break;
    }

    fileoff = chunks[i - 1]->shdr.sh_offset + chunks[i - 1]->shdr.sh_size;

    while (i < chunks.size() &&
           (chunks[i]->shdr.sh_flags & SHF_ALLOC) &&
           chunks[i]->shdr.sh_type == SHT_NOBITS)
      i++;
  }

  return fileoff;
}
```

# riscv_resize_sections

```cpp
// On RISC-V, branches are encode using multiple instructions so
// that they can jump to anywhere in ±2 GiB by default. They may
// be replaced with shorter instruction sequences if destinations
// are close enough. Do this optimization.
if constexpr (is_riscv<E>)
  filesize = riscv_resize_sections(ctx);
```

这里主要是针对riscv跳转在2GB范围的限制做的优化，优化后重新计算shdr

做了如下几个步骤

1. 获取eflag，unix通常假设RV64GC是baseline，所以这里flag要加上C扩展的flag（EF_RISCV_RVC），而C扩展则是允许压缩指令集，使用2字节编码指令。
2. 针对resizeable的section进行shrink
3. 修正符号值
4. 重新执行上一步的计算offset操作

```cpp
// Shrink sections by interpreting relocations.
//
// This operation seems to be optional, because by default longest
// instructions are being used. However, calling this function is actually
// mandatory because of R_RISCV_ALIGN. R_RISCV_ALIGN is a directive to the
// linker to align the location referred to by the relocation to a
// specified byte boundary. We at least have to interpret them to satisfy
// the alignment constraints.
template <typename E>
i64 riscv_resize_sections(Context<E> &ctx) {
  Timer t(ctx, "riscv_resize_sections");

  // True if we can use the 2-byte instructions. This is usually true on
  // Unix because RV64GC is generally considered the baseline hardware.
  bool use_rvc = get_eflags(ctx) & EF_RISCV_RVC;

  // Find all the relocations that can be relaxed.
  // This step should only shrink sections.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (std::unique_ptr<InputSection<E>> &isec : file->sections)
      if (is_resizable(ctx, isec.get()))
        shrink_section(ctx, *isec, use_rvc);
  });

  // Fix symbol values.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (Symbol<E> *sym : file->symbols) {
      if (sym->file != file)
        continue;

      InputSection<E> *isec = sym->get_input_section();
      if (!isec || isec->extra.r_deltas.empty())
        continue;

      std::span<const ElfRel<E>> rels = isec->get_rels(ctx);
      auto it = std::lower_bound(rels.begin(), rels.end(), sym->value,
                                 [&](const ElfRel<E> &r, u64 val) {
        return r.r_offset < val;
      });

      sym->value -= isec->extra.r_deltas[it - rels.begin()];
    }
  });

  // Re-compute section offset again to finalize them.
  compute_section_sizes(ctx);
  return set_osec_offsets(ctx);
}
```

```cpp
template <typename E>
u64 get_eflags(Context<E> &ctx) {
  if constexpr (std::is_same_v<E, ARM32>)
    return EF_ARM_EABI_VER5;

  if constexpr (is_riscv<E>) {
    std::vector<ObjectFile<E> *> objs = ctx.objs;
    std::erase(objs, ctx.internal_obj);

    if (objs.empty())
      return 0;

    u32 ret = objs[0]->get_ehdr().e_flags;
    for (i64 i = 1; i < objs.size(); i++)
      if (objs[i]->get_ehdr().e_flags & EF_RISCV_RVC)
        ret |= EF_RISCV_RVC;
    return ret;
  }

  if constexpr (std::is_same_v<E, PPC64V2>)
    return 2;
  return 0;
}
```

## resizable

```cpp
template <typename E>
static bool is_resizable(Context<E> &ctx, InputSection<E> *isec) {
  return isec && isec->is_alive && (isec->shdr().sh_flags & SHF_ALLOC) &&
         (isec->shdr().sh_flags & SHF_EXECINSTR);
}
```

SHF_EXECINSTR

> This section contains executable machine instructions.

需要分配空间且包含可执行的指令的段才能进行resize。

## shrink

针对所有的rel进行操作，最终为了更新delta值（shrink的size），用于后续fix symbol value

1. 更新delta的值

2. R_RISCV_ALIGN是用于align的部分，指向nop指令，因此需要删除nop指令使得对齐到指定alignment

3. 针对一些无法优化的情况进行跳过

   1. 未开启relax选项

      其中的relax选项是

      > --relax Optimize instructions (default) 
      > --no-relax

   2. 搜寻到了最后一个re l

   3. 另外下一个rel如果不是RELAX的也无法进行优化

4. 跳过internal_obj，因为synthetic符号还没有最终值，只有确定值的情况才能进行shrink

5. 针对不同rel type进行处理，添加不同的delta值。这里细节不再深究

6. 减少当前input section的size并且更新deltas的值

```cpp
// Scan relocations to shrink sections.
template <typename E>
static void shrink_section(Context<E> &ctx, InputSection<E> &isec, bool use_rvc) {
  std::span<const ElfRel<E>> rels = isec.get_rels(ctx);
  isec.extra.r_deltas.resize(rels.size() + 1);

  i64 delta = 0;

  for (i64 i = 0; i < rels.size(); i++) {
    const ElfRel<E> &r = rels[i];
    Symbol<E> &sym = *isec.file.symbols[r.r_sym];
    isec.extra.r_deltas[i] = delta;

    // Handling R_RISCV_ALIGN is mandatory.
    //
    // R_RISCV_ALIGN refers NOP instructions. We need to eliminate some
    // or all of the instructions so that the instruction that immediately
    // follows the NOPs is aligned to a specified alignment boundary.
    if (r.r_type == R_RISCV_ALIGN) {
      // The total bytes of NOPs is stored to r_addend, so the next
      // instruction is r_addend away.
      u64 loc = isec.get_addr() + r.r_offset - delta;
      u64 next_loc = loc + r.r_addend;
      u64 alignment = bit_ceil(r.r_addend + 1);
      assert(alignment <= (1 << isec.p2align));
      delta += next_loc - align_to(loc, alignment);
      continue;
    }

    // Handling other relocations is optional.
    if (!ctx.arg.relax || i == rels.size() - 1 ||
        rels[i + 1].r_type != R_RISCV_RELAX)
      continue;

    // Linker-synthesized symbols haven't been assigned their final
    // values when we are shrinking sections because actual values can
    // be computed only after we fix the file layout. Therefore, we
    // assume that relocations against such symbols are always
    // non-relaxable.
    if (sym.file == ctx.internal_obj)
      continue;

    switch (r.r_type) {
    case R_RISCV_CALL:
    case R_RISCV_CALL_PLT: {
      // These relocations refer an AUIPC + JALR instruction pair to
      // allow to jump to anywhere in PC ± 2 GiB. If the jump target is
      // close enough to PC, we can use C.J, C.JAL or JAL instead.
      i64 dist = compute_distance(ctx, sym, isec, r);
      if (dist & 1)
        break;

      std::string_view contents = isec.contents;
      i64 rd = get_rd(*(ul32 *)(contents.data() + r.r_offset + 4));

      if (rd == 0 && sign_extend(dist, 11) == dist && use_rvc) {
        // If rd is x0 and the jump target is within ±2 KiB, we can use
        // C.J, saving 6 bytes.
        delta += 6;
      } else if (rd == 1 && sign_extend(dist, 11) == dist && use_rvc && !E::is_64) {
        // If rd is x1 and the jump target is within ±2 KiB, we can use
        // C.JAL. This is RV32 only because C.JAL is RV32-only instruction.
        delta += 6;
      } else if (sign_extend(dist, 20) == dist) {
        // If the jump target is within ±1 MiB, we can use JAL.
        delta += 4;
      }
      break;
    }
    case R_RISCV_HI20: {
      // If the upper 20 bits are all zero, we can remove LUI.
      // The corresponding instructions referred by LO12_I/LO12_S
      // relocations will use the zero register instead.
      i64 val = sym.get_addr(ctx);
      if (sign_extend(val, 11) == val)
        delta += 4;
      break;
    }
    case R_RISCV_TPREL_HI20:
    case R_RISCV_TPREL_ADD: {
      // These relocations are used to materialize the upper 20 bits of
      // an address relative to the thread pointer as follows:
      //
      //  lui  a5,%tprel_hi(foo)         # R_RISCV_TPREL_HI20 (symbol)
      //  add  a5,a5,tp,%tprel_add(foo)  # R_RISCV_TPREL_ADD (symbol)
      //
      // Then thread-local variable `foo` is accessed with a 12-bit offset
      // like this:
      //
      //  sw   t0,%tprel_lo(foo)(a5)     # R_RISCV_TPREL_LO12_S (symbol)
      //
      // However, if the offset is ±2 KiB, we don't need to materialize
      // the upper 20 bits in a register. We can instead access the
      // thread-local variable directly with TP like this:
      //
      //  sw   t0,%tprel_lo(foo)(tp)
      //
      // Here, we remove `lui` and `add` if the offset is within ±2 KiB.
      i64 val = sym.get_addr(ctx) + r.r_addend - ctx.tp_addr;
      if (sign_extend(val, 11) == val)
        delta += 4;
      break;
    }
    }
  }

  isec.extra.r_deltas[rels.size()] = delta;
  isec.sh_size -= deltfa;
}

template <typename E> requires is_riscv<E>
struct InputSectionExtras<E> {
  std::vector<i32> r_deltas;
};
```
