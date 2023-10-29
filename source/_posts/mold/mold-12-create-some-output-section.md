---
title: mold源码阅读十二 创建一些输出段
typora-root-url: ../../source
date: 2023-07-09 16:35:19
category: Linker
tags:
  - [mold] 
  - [eh_frame]
  - [symtab]
---

![Untitled](/images/mold-12-create-some-output-section/105296500_p0_master1200.jpg)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:105296500_p0</center> 

# Fill gnu.version section contents

```cpp
// Fill .gnu.version_d section contents.
if (ctx.verdef)
  ctx.verdef->construct(ctx);

// Fill .gnu.version_r section contents.
ctx.verneed->construct(ctx);
```

这里对verdef和verneed段进行构造，实际写入内容。其中包含了字符串信息，因此还会将字符串写入dynstr中。

## verdef

对于VerdefSection中的contents是多组ElfVerDef + ElfVerdaux。前者是verdef的信息，后者则是指向对应字符串在dynstr中的offset。

需要将ctx.arg.version_definitions以及output自身的信息写入到verdef段中，因此这样的数据实际有ctx.arg.version_definitions.size() + 1组。

```cpp
| verdef | verdaux | verdef | verdaux |
						/                  /	
|     dynstr0    |    dynstr1    | ... | dynstrn |
```

```cpp
template <typename E>
class VerdefSection : public Chunk<E> {
public:
  VerdefSection() {
    this->name = ".gnu.version_d";
    this->shdr.sh_type = SHT_GNU_VERDEF;
    this->shdr.sh_flags = SHF_ALLOC;
    this->shdr.sh_addralign = 8;
  }

  void construct(Context<E> &ctx);
  void update_shdr(Context<E> &ctx) override;
  void copy_buf(Context<E> &ctx) override;

  std::vector<u8> contents;
};
```

每次写入的时候会先在当前位置写入ElfVerDef的信息，之后写入ElfVerdaux的信息，同时在这个过程中更新当前位置的指针。传入的verstr实际保存在ctx.dynstr中，而Verdaux中保存的是则是verstr在dynstr中的offset，而VerDef仅保存索引，hash等信息。

```cpp
template <typename E>
void VerdefSection<E>::construct(Context<E> &ctx) {
  Timer t(ctx, "fill_verdef");

  if (ctx.arg.version_definitions.empty())
    return;

  // Resize .gnu.version
  ctx.versym->contents.resize(ctx.dynsym->symbols.size(), 1);
  ctx.versym->contents[0] = 0;

  // Allocate a buffer for .gnu.version_d.
  contents.resize((sizeof(ElfVerdef<E>) + sizeof(ElfVerdaux<E>)) *
                  (ctx.arg.version_definitions.size() + 1));

  u8 *buf = (u8 *)&contents[0];
  u8 *ptr = buf;
  ElfVerdef<E> *verdef = nullptr;

  auto write = [&](std::string_view verstr, i64 idx, i64 flags) {
    this->shdr.sh_info++;
    if (verdef)
      verdef->vd_next = ptr - (u8 *)verdef;

    verdef = (ElfVerdef<E> *)ptr;
    ptr += sizeof(ElfVerdef<E>);

    verdef->vd_version = 1;
    verdef->vd_flags = flags;
    verdef->vd_ndx = idx;
    verdef->vd_cnt = 1;
    verdef->vd_hash = elf_hash(verstr);
    verdef->vd_aux = sizeof(ElfVerdef<E>);

    ElfVerdaux<E> *aux = (ElfVerdaux<E> *)ptr;
    ptr += sizeof(ElfVerdaux<E>);
    aux->vda_name = ctx.dynstr->add_string(verstr);
  };

  std::string_view basename = ctx.arg.soname.empty() ?
    ctx.arg.output : ctx.arg.soname;
  write(basename, 1, VER_FLG_BASE);

  i64 idx = 2;
  for (std::string_view verstr : ctx.arg.version_definitions)
    write(verstr, idx++, 0);

  for (Symbol<E> *sym : std::span<Symbol<E> *>(ctx.dynsym->symbols).subspan(1))
    ctx.versym->contents[sym->get_dynsym_idx(ctx)] = sym->ver_idx;
}
```

ver_idx的值是

```tsx
static constexpr u32 VER_NDX_LOCAL = 0;
static constexpr u32 VER_NDX_GLOBAL = 1;
static constexpr u32 VER_NDX_LAST_RESERVED = 1;
```

## verneed

这里的数据格式和vardef不太一样，content是一个Verneed接着多个Vednaux构成。每个Verneed表示一个文件的开始。由于这里是针对dynsym处理，因此实际Vednaux的数量和dynsym的数量相同。在分配空间的时候注释也有写到allocate large enought buffer，避免了每个文件一个dynsym的极端场景。

```tsx
|verneed|vednaux|vednaux|...|verneed|vednaux|vednaux|vednaux|
```

另外不在dso或者sym->ver_idx <= VER_NDX_LAST_RESERVED的sym，这些符号并不需要填充verneed字段，因此会先被过滤掉。之后由于content是以一个文件为一个小组，为了后面添加信息方便会根据soname以及ver_idx进行排序。

```cpp
template <typename E>
void VerneedSection<E>::construct(Context<E> &ctx) {
  Timer t(ctx, "fill_verneed");

  if (ctx.dynsym->symbols.empty())
    return;

  // Create a list of versioned symbols and sort by file and version.
  std::vector<Symbol<E> *> syms(ctx.dynsym->symbols.begin() + 1,
                                ctx.dynsym->symbols.end());

  std::erase_if(syms, [](Symbol<E> *sym) {
    return !sym->file->is_dso || sym->ver_idx <= VER_NDX_LAST_RESERVED;
  });

  if (syms.empty())
    return;

  sort(syms, [](Symbol<E> *a, Symbol<E> *b) {
    return std::tuple(((SharedFile<E> *)a->file)->soname, a->ver_idx) <
           std::tuple(((SharedFile<E> *)b->file)->soname, b->ver_idx);
  });

  // Resize of .gnu.version
  ctx.versym->contents.resize(ctx.dynsym->symbols.size(), 1);
  ctx.versym->contents[0] = 0;

  // Allocate a large enough buffer for .gnu.version_r.
  contents.resize((sizeof(ElfVerneed<E>) + sizeof(ElfVernaux<E>)) * syms.size());

  // Fill .gnu.version_r.
  u8 *buf = (u8 *)&contents[0];
  u8 *ptr = buf;
  ElfVerneed<E> *verneed = nullptr;
  ElfVernaux<E> *aux = nullptr;

  u16 veridx = VER_NDX_LAST_RESERVED + ctx.arg.version_definitions.size();

  for (i64 i = 0; i < syms.size(); i++) {
    if (i == 0 || syms[i - 1]->file != syms[i]->file) {
      start_group(syms[i]->file);
      add_entry(syms[i]);
    } else if (syms[i - 1]->ver_idx != syms[i]->ver_idx) {
      add_entry(syms[i]);
    }

    ctx.versym->contents[syms[i]->get_dynsym_idx(ctx)] = veridx;
  }

  // Resize .gnu.version_r to fit to its contents.
  contents.resize(ptr - buf);
}
```

处理过程中根据如果是第一个符号或者连续两个符号不是相同的file就start_group。

要注意ctx.versym->contents又重新resize了一次，在后面遍历符号的时候又会再次写入，或许是因为verdef是根据选项来决定是否执行的。两次resize实际上size是相同的，而verneed中并非所有符号都会写入versym→content，部分被过滤的符号是没有再次写入的，也就是说被过滤的符号会保留verneed的部分。

接着来看一下start_group的部分。这个函数中会sh_info递增，处理verneed（关联一个file），并且aux置空。也就是说VerneedSection的sh_info存放的是ElfVerneed的数量。每个ElfVerneed关联了一个文件，以及aux的size。

```cpp
auto start_group = [&](InputFile<E> *file) {
  this->shdr.sh_info++;
  if (verneed)
    verneed->vn_next = ptr - (u8 *)verneed;

  verneed = (ElfVerneed<E> *)ptr;
  ptr += sizeof(*verneed);
  verneed->vn_version = 1;
  verneed->vn_file = ctx.dynstr->find_string(((SharedFile<E> *)file)->soname);
  verneed->vn_aux = sizeof(ElfVerneed<E>);
  aux = nullptr;
};
```

在add_entry中会递增当前的verneed的计数，将信息填写到ElfVernaux中，并且更新当前aux的指针

```cpp
auto add_entry = [&](Symbol<E> *sym) {
  verneed->vn_cnt++;

  if (aux)
    aux->vna_next = sizeof(ElfVernaux<E>);
  aux = (ElfVernaux<E> *)ptr;
  ptr += sizeof(*aux);

  std::string_view verstr = sym->get_version();
  aux->vna_hash = elf_hash(verstr);
  aux->vna_other = ++veridx;
  aux->vna_name = ctx.dynstr->add_string(verstr);
};
```

# create_output_symtab

这个过程是用于创建symtab和strtab，创建的时候会实际选择哪些符号要写到文件中。我们熟悉的strip，如果添加了链接选项那么就是在这里开始生效的。

相关的链接选项在mold中有如下几个

> -s, --strip-all             Strip .symtab section

> --retain-symbols-file FILE  Keep only symbols listed in FILE

> discard_all

strip大家都很熟悉了，就是去掉生成文件中的symtab段

retain-symbols-file则是会产生一个符号文件，包含程序的调试信息，也就是说生成的文件说不包含符号信息，所有符号都在符号文件中。

discard_all是丢弃目标程序中未直接使用的信息，其中就包含符号表和调试信息。

```cpp
// Compute .symtab and .strtab sizes for each file.
create_output_symtab(ctx);
```

```cpp
template <typename E>
void create_output_symtab(Context<E> &ctx) {
  Timer t(ctx, "compute_symtab_size");

  tbb::parallel_for_each(ctx.chunks, [&](Chunk<E> *chunk) {
    chunk->compute_symtab_size(ctx);
  });

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->compute_symtab_size(ctx);
  });

  tbb::parallel_for_each(ctx.dsos, [&](SharedFile<E> *file) {
    file->compute_symtab_size(ctx);
  });
}
```

## chunk

```cpp
// Chunk::compute_symtab_size

// Some synethetic sections add local symbols to the output.
// For example, range extension thunks adds function_name@thunk
// symbol for each thunk entry. The following members are used
// for such synthesizing symbols.
virtual void compute_symtab_size(Context<E> &ctx) {};
```

对于chunk来说，不是所有的都需要做这一步操作的。在mold中仅针对OutputSection，Got，Plt，PltGot这几个chunk来处理。

实际要做的就是遍历所有符号更新其strtab_size以及num_local_symtab（用于标记local符号的数量，也就是这个阶段要计算的symtab size），不论是哪一种chunk都是如此，下面就不再赘述，只贴代码了。

### OutputSection

```cpp
// Compute spaces needed for thunk symbols
template <typename E>
void OutputSection<E>::compute_symtab_size(Context<E> &ctx) {
  if (ctx.arg.strip_all || ctx.arg.retain_symbols_file || ctx.arg.relocatable)
    return;

  if constexpr (needs_thunk<E>) {
    this->strtab_size = 0;
    this->num_local_symtab = 0;

    if constexpr (std::is_same_v<E, ARM32>)
      this->strtab_size = 9; // for "$t", "$a" and "$d" symbols

    for (std::unique_ptr<RangeExtensionThunk<E>> &thunk : thunks) {
      // For ARM32, we emit additional symbol "$t", "$a" and "$d" for
      // each thunk to mark the beginning of ARM code.
      if constexpr (std::is_same_v<E, ARM32>)
        this->num_local_symtab += thunk->symbols.size() * 4;
      else
        this->num_local_symtab += thunk->symbols.size();

      for (Symbol<E> *sym : thunk->symbols)
        this->strtab_size += sym->name().size() + sizeof("$thunk");
    }
  }
}
```

注意这里relocatable的段也不会算入symtab size中，因为地址并非固定，需要加载时重定位，如果把符号放入输出文件中，会使得重定位更加困难，并且加载时会失效。

need_thunk：

1. 输出段中代码间隔比较大，直接跳转无法到达的时候需要thunk来中专
2. 跳转的src和dest指令集不兼容需要thunk翻译
3. 地址随机化(ASLR: Address space layout randomization)时需要thunk动态计算目标地址
4. 地址运行时才能确定时需要thunk计算地址

基本上都是一些无法直接跳转的情况，也因此会引入新的符号。而thunk本质上是一个新的代码段，需要符号进行表示，用以被其他代码识别。

```cpp
template <typename E>
static constexpr bool needs_thunk = requires { E::thunk_size; };
```

根据mold代码中的实现，目前需要thunk的是ARM32，ARM64，PPC64V1，PPC64V2

### got

```cpp
template <typename E>
void GotSection<E>::compute_symtab_size(Context<E> &ctx) {
  if (ctx.arg.strip_all || ctx.arg.retain_symbols_file)
    return;

  this->strtab_size = 0;
  this->num_local_symtab = 0;

  for (Symbol<E> *sym : got_syms) {
    this->strtab_size += sym->name().size() + sizeof("$got");
    this->num_local_symtab++;
  }

  for (Symbol<E> *sym : gottp_syms) {
    this->strtab_size += sym->name().size() + sizeof("$gottp");
    this->num_local_symtab++;
  }

  for (Symbol<E> *sym : tlsgd_syms) {
    this->strtab_size += sym->name().size() + sizeof("$tlsgd");
    this->num_local_symtab++;
  }

  for (Symbol<E> *sym : tlsdesc_syms) {
    this->strtab_size += sym->name().size() + sizeof("$tlsdesc");
    this->num_local_symtab++;
  }

  if (tlsld_idx != -1) {
    this->strtab_size += sizeof("$tlsld");
    this->num_local_symtab++;
  }
}
```

### PLT

```cpp
template <typename E>
void PltSection<E>::compute_symtab_size(Context<E> &ctx) {
  if (ctx.arg.strip_all || ctx.arg.retain_symbols_file)
    return;

  this->num_local_symtab = symbols.size();
  this->strtab_size = 0;

  for (Symbol<E> *sym : symbols)
    this->strtab_size += sym->name().size() + sizeof("$plt");
}
```

### PLTGOT

```cpp
template <typename E>
void PltGotSection<E>::compute_symtab_size(Context<E> &ctx) {
  if (ctx.arg.strip_all || ctx.arg.retain_symbols_file)
    return;

  this->num_local_symtab = symbols.size();
  this->strtab_size = 0;

  for (Symbol<E> *sym : symbols)
    this->strtab_size += sym->name().size() + sizeof("$pltgot");
}
```

## obj

在obj中，主要计算了local和global符号的名字占用的空间，用于更新strtable_size，另外还会更新对应的output_sym_indices

要注意的是计算名字空间的时候，这里的名字需要使用null结尾，因此size还需要加一。

```cpp
template <typename E>
void ObjectFile<E>::compute_symtab_size(Context<E> &ctx) {
  if (ctx.arg.strip_all)
    return;

  this->output_sym_indices.resize(this->elf_syms.size(), -1);

  auto is_alive = [&](Symbol<E> &sym) -> bool {
    if (!ctx.arg.gc_sections)
      return true;

    if (SectionFragment<E> *frag = sym.get_frag())
      return frag->is_alive;
    if (InputSection<E> *isec = sym.get_input_section())
      return isec->is_alive;
    return true;
  };

  // Compute the size of local symbols
  if (!ctx.arg.discard_all && !ctx.arg.strip_all && !ctx.arg.retain_symbols_file) {
    for (i64 i = 1; i < this->first_global; i++) {
      Symbol<E> &sym = *this->symbols[i];

      if (is_alive(sym) && should_write_to_local_symtab(ctx, sym)) {
        this->strtab_size += sym.name().size() + 1;
        this->output_sym_indices[i] = this->num_local_symtab++;
        sym.write_to_symtab = true;
      }
    }
  }

  // Compute the size of global symbols.
  for (i64 i = this->first_global; i < this->elf_syms.size(); i++) {
    Symbol<E> &sym = *this->symbols[i];

    if (sym.file == this && is_alive(sym) &&
        (!ctx.arg.retain_symbols_file || sym.write_to_symtab)) {
      this->strtab_size += sym.name().size() + 1;
      // Global symbols can be demoted to local symbols based on visibility,
      // version scripts etc.
      if (sym.is_local(ctx))
        this->output_sym_indices[i] = this->num_local_symtab++;
      else
        this->output_sym_indices[i] = this->num_global_symtab++;
      sym.write_to_symtab = true;
    }
  }
}
```

对于local symbol除了要判断alive之外，还有一个should_write_to_local_symtab的判断，除了更新size外还会更新write_to_symtab

```cpp
template <typename E>
static bool should_write_to_local_symtab(Context<E> &ctx, Symbol<E> &sym) {
  if (sym.get_type() == STT_SECTION)
    return false;

  // Local symbols are discarded if --discard-local is given or they
  // are in a mergeable section. I *believe* we exclude symbols in
  // mergeable sections because (1) there are too many and (2) they are
  // merged, so their origins shouldn't matter, but I don't really
  // know the rationale. Anyway, this is the behavior of the
  // traditional linkers.
  if (sym.name().starts_with(".L")) {
    if (ctx.arg.discard_locals)
      return false;

    if (InputSection<E> *isec = sym.get_input_section())
      if (isec->shdr().sh_flags & SHF_MERGE)
        return false;
  }

  return true;
}
```

> -X, --discard-locals        Discard temporary local symbols

本地符号以本地标签为前缀开头，这个标签通常为.L，这里主要是对discard_locals进行处理，另外属于SHF_MERGE的段也不会写到local，根据这里注释的意思是SHF_MERGE段段符号太多了，并且是merge以后的，所以其来源不重要，并且传统的链接器都是这么做的。（我对这块也不了解，只能按照注释所说的来看了）

还有一个sym.is_local的判断看起来比较疑惑。根据注释所描述，global sym会基于visibility和version scripts等因素变成local sym，比如说设置某个global sym的可见性为特定范围，或者对应的脚本。当全局符号降级为local的时候则不再对外可见，因此不再占用全局符号表的空间。

代码里的判断是这样的

```cpp
template <typename E>
inline bool Symbol<E>::is_local(Context<E> &ctx) const {
  if (ctx.arg.relocatable)
    return esym().st_bind == STB_LOCAL;
  return !is_imported && !is_exported;
}
```

对于relocatable来说，如果st_bind为STB_LOCAL，那么这个符号一定是local的

对于非imported以及非exported的全局符号，通常是模块内部实现细节使用，不能外部访问。比如说有如下几种情况

1. 静态全局符号，只能模块内部可见（因为静态符号的作用域限定在模块内，因此会被认为是local符号，对全局静态变量的访问只需要通过内存地址，而不需要符号名进行绑定）
2. 匿名全局符号，没有被显示的使用export或者extern等进行标记，并且对外部是不可见的。比如说在.c中定义了一个全局变量，但是外部无法访问到。
3. 未使用的全局符号，不会被访问，同时会被优化掉

因此这些情况属于local，记入num_local_symtab

关于imported和exported的计算过程，可以参考之前第五期的文章，其中有根据可见性来设置exported和imported的部分

[https://homura.live/2023/04/29/mold/mold-5-symbol/](https://homura.live/2023/04/29/mold/mold-5-symbol/)

## dso

```cpp
template <typename E>
void SharedFile<E>::compute_symtab_size(Context<E> &ctx) {
  if (ctx.arg.strip_all)
    return;

  this->output_sym_indices.resize(this->elf_syms.size(), -1);

  // Compute the size of global symbols.
  for (i64 i = this->first_global; i < this->symbols.size(); i++) {
    Symbol<E> &sym = *this->symbols[i];

    if (sym.file == this && (sym.is_imported || sym.is_exported) &&
        (!ctx.arg.retain_symbols_file || sym.write_to_symtab)) {
      this->strtab_size += sym.name().size() + 1;
      this->output_sym_indices[i] = this->num_global_symtab++;
      sym.write_to_symtab = true;
    }
  }
}
```

这里的要点就是imported或者exported才需要计入num_global_symtab

# eh_frame_construct

```cpp
// .eh_frame is a special section from the linker's point of view,
// as its contents are parsed and reconstructed by the linker,
// unlike other sections that are regarded as opaque bytes.
// Here, we construct output .eh_frame contents.
ctx.eh_frame->construct(ctx);
```

由于eh_frame在mold中自行做了parse的，因此需要再手动构造output中eh_frame的部分，在构造的过程中主要是消除重复的部分，另外各个段是由offset以及idx关联起来的，更新这些信息也是必要的工作。

关于mold自行parse eh_frame的部分可以参考第二期的内容[https://homura.live/2023/04/05/mold/mold-2-read-shared-files/](https://homura.live/2023/04/05/mold/mold-2-read-shared-files/)

在构造的过程主要由如下几部分组成

1. 确保输入存在eh_frame，不存在则无需构造
2. 删除dead fed，重新设置offset
3. uniquify cie，重新设置offset
4. fde idx的更新
5. 重新设置文件中存储的fde的offset
6. 填充最后的null word

```cpp
template <typename E>
class EhFrameSection : public Chunk<E> {
public:
  EhFrameSection() {
    this->name = ".eh_frame";
    this->shdr.sh_type = SHT_PROGBITS;
    this->shdr.sh_flags = SHF_ALLOC;
    this->shdr.sh_addralign = sizeof(Word<E>);
  }

  void construct(Context<E> &ctx);
  void apply_reloc(Context<E> &ctx, const ElfRel<E> &rel, u64 offset, u64 val);
  void copy_buf(Context<E> &ctx) override;
};
```

```cpp
template <typename E>
void EhFrameSection<E>::construct(Context<E> &ctx) {
  Timer t(ctx, "eh_frame");

  // If .eh_frame is missing in all input files, we don't want to
  // create an output .eh_frame section.
  if (std::all_of(ctx.objs.begin(), ctx.objs.end(),
                  [](ObjectFile<E> *file) { return file->cies.empty(); })) {
    this->shdr.sh_size = 0;
    return;
  }

  // Remove dead FDEs and assign them offsets within their corresponding
  // CIE group.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    std::erase_if(file->fdes, [](FdeRecord<E> &fde) { return !fde.is_alive; });

    i64 offset = 0;
    for (FdeRecord<E> &fde : file->fdes) {
      fde.output_offset = offset;
      offset += fde.size(*file);
    }
    file->fde_size = offset;
  });

  // Uniquify CIEs and assign offsets to them.
  std::vector<CieRecord<E> *> leaders;
  auto find_leader = [&](CieRecord<E> &cie) -> CieRecord<E> * {
    for (CieRecord<E> *leader : leaders)
      if (cie.equals(*leader))
        return leader;
    return nullptr;
  };

  i64 offset = 0;
  for (ObjectFile<E> *file : ctx.objs) {
    for (CieRecord<E> &cie : file->cies) {
      if (CieRecord<E> *leader = find_leader(cie)) {
        cie.output_offset = leader->output_offset;
      } else {
        cie.output_offset = offset;
        cie.is_leader = true;
        offset += cie.size();
        leaders.push_back(&cie);
      }
    }
  }

  // Assign FDE offsets to files.
  i64 idx = 0;
  for (ObjectFile<E> *file : ctx.objs) {
    file->fde_idx = idx;
    idx += file->fdes.size();

    file->fde_offset = offset;
    offset += file->fde_size;
  }

  // .eh_frame must end with a null word.
  this->shdr.sh_size = offset + 4;
}
```

# gdb index

gdb-index是用于加速gdb的段，对应的链接选项

> --gdb-index                 Create .gdb_index for faster gdb startup

这边就不具体详细介绍了，有兴趣的可以自行看一下资料

[https://sourceware.org/gdb/onlinedocs/gdb/Index-Files.html](https://sourceware.org/gdb/onlinedocs/gdb/Index-Files.html)

```cpp
// Handle --gdb-index.
if (ctx.arg.gdb_index)
  ctx.gdb_index->construct(ctx);
```

```cpp
// This page explains the format of .gdb_index:
// https://sourceware.org/gdb/onlinedocs/gdb/Index-Section-Format.html
template <typename E>
void GdbIndexSection<E>::construct(Context<E> &ctx) {
  Timer t(ctx, "GdbIndexSection::construct");

  std::atomic_bool has_debug_info = false;

  // Read debug sections
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    if (file->debug_info) {
      // Read compilation units from .debug_info.
      file->compunits = read_compunits(ctx, *file);

      // Count the number of address areas contained in this file.
      file->num_areas = estimate_address_areas(ctx, *file);
      has_debug_info = true;
    }
  });

  if (!has_debug_info)
    return;

  // Initialize `area_offset` and `compunits_idx`.
  for (i64 i = 0; i < ctx.objs.size() - 1; i++) {
    ctx.objs[i + 1]->area_offset =
      ctx.objs[i]->area_offset + ctx.objs[i]->num_areas * 20;
    ctx.objs[i + 1]->compunits_idx =
      ctx.objs[i]->compunits_idx + ctx.objs[i]->compunits.size();
  }

  // Read .debug_gnu_pubnames and .debug_gnu_pubtypes.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->gdb_names = read_pubnames(ctx, *file);
  });

  // Estimate the unique number of pubnames.
  HyperLogLog estimator;
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    HyperLogLog e;
    for (GdbIndexName &name : file->gdb_names)
      e.insert(name.hash);
    estimator.merge(e);
  });

  // Uniquify pubnames by inserting all name strings into a concurrent
  // hashmap.
  map.resize(estimator.get_cardinality() * 2);
  tbb::enumerable_thread_specific<i64> num_names;

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (GdbIndexName &name : file->gdb_names) {
      MapEntry *ent;
      bool inserted;
      std::tie(ent, inserted) = map.insert(name.name, name.hash, {file, name.hash});
      if (inserted)
        num_names.local()++;

      ObjectFile<E> *old_val = ent->owner;
      while (file->priority < old_val->priority &&
             !ent->owner.compare_exchange_weak(old_val, file));

      ent->num_attrs++;
      name.entry_idx = ent - map.values;
    }
  });

  // Assign offsets for names and attributes within each file.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (GdbIndexName &name : file->gdb_names) {
      MapEntry &ent = map.values[name.entry_idx];
      if (ent.owner == file) {
        ent.attr_offset = file->attrs_size;
        file->attrs_size += (ent.num_attrs + 1) * 4;
        ent.name_offset = file->names_size;
        file->names_size += name.name.size() + 1;
      }
    }
  });

  // Compute per-file name and attributes offsets.
  for (i64 i = 0; i < ctx.objs.size() - 1; i++)
    ctx.objs[i + 1]->attrs_offset =
      ctx.objs[i]->attrs_offset + ctx.objs[i]->attrs_size;

  ctx.objs[0]->names_offset =
    ctx.objs.back()->attrs_offset + ctx.objs.back()->attrs_size;

  for (i64 i = 0; i < ctx.objs.size() - 1; i++)
    ctx.objs[i + 1]->names_offset =
      ctx.objs[i]->names_offset + ctx.objs[i]->names_size;

  // .gdb_index contains an on-disk hash table for pubnames and
  // pubtypes. We aim 75% utilization. As per the format specification,
  // It must be a power of two.
  i64 num_symtab_entries =
    std::max<i64>(bit_ceil(num_names.combine(std::plus()) * 4 / 3), 16);

  // Now that we can compute the size of this section.
  ObjectFile<E> &last = *ctx.objs.back();
  i64 compunits_size = (last.compunits_idx + last.compunits.size()) * 16;
  i64 areas_size = last.area_offset + last.num_areas * 20;
  i64 offset = sizeof(header);

  header.cu_list_offset = offset;
  offset += compunits_size;

  header.cu_types_offset = offset;
  header.areas_offset = offset;
  offset += areas_size;

  header.symtab_offset = offset;
  offset += num_symtab_entries * 8;

  header.const_pool_offset = offset;
  offset += last.names_offset + last.names_size;

  this->shdr.sh_size = offset;
}
```
