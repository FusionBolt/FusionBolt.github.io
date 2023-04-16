---
title: mold源码阅读 其一 读取输入文件
typora-root-url: ../../source
date: 2023-02-26 17:40:55
category: Linker
tags: mold
---

![图像](/images/mold-1-read-input-files/FhW-kVdagAI7itc.jpeg)上一期主要讲了链接前的一些准备流程以及在mold中链接过程的简单介绍。这期开始我们从链接过程中的功能开始介绍。在开始之前，提前说明一下里面各种缩写有很多，我会在第一次出现时提及缩写具体含义是什么，如果后期更的期数比较多会考虑专门写一页缩写的参考，方便查阅。

首先是解析输入，命令行参数解析的细节略过，但是这里不能略过elf文件的解析。我们从代码的实现去看elf的结构，再和文档中的图进行对比，同时尽可能从代码中去捋清不同结构之间的联系。

我们从elf_main函数中的read_input_files开始

```cpp
read_input_files(ctx, file_args);
```

# read_input_files

```cpp
template <typename E>
static void read_input_files(Context<E> &ctx, std::span<std::string> args) {
  Timer t(ctx, "read_input_files");

  std::vector<std::tuple<bool, bool, bool, bool>> state;
  ctx.is_static = ctx.arg.is_static;

  while (!args.empty()) {
    std::string_view arg = args[0];
    args = args.subspan(1);

    if (arg == "--as-needed") {
      ctx.as_needed = true;
    } else if (arg == "--no-as-needed") {
      ctx.as_needed = false;
    } else if (arg == "--whole-archive") {
      ctx.whole_archive = true;
    } else if (arg == "--no-whole-archive") {
      ctx.whole_archive = false;
    } else if (arg == "--Bstatic") {
      ctx.is_static = true;
    } else if (arg == "--Bdynamic") {
      ctx.is_static = false;
    } else if (arg == "--start-lib") {
      ctx.in_lib = true;
    } else if (arg == "--end-lib") {
      ctx.in_lib = false;
    } else if (remove_prefix(arg, "--version-script=")) {
      MappedFile<Context<E>> *mf = find_from_search_paths(ctx, std::string(arg));
      if (!mf)
        Fatal(ctx) << "--version-script: file not found: " << arg;
      parse_version_script(ctx, mf);
    } else if (remove_prefix(arg, "--dynamic-list=")) {
      MappedFile<Context<E>> *mf = find_from_search_paths(ctx, std::string(arg));
      if (!mf)
        Fatal(ctx) << "--dynamic-list: file not found: " << arg;
      parse_dynamic_list(ctx, mf);
    } else if (remove_prefix(arg, "--export-dynamic-symbol=")) {
      if (arg == "*")
        ctx.default_version = VER_NDX_GLOBAL;
      else
        ctx.version_patterns.push_back({arg, "--export-dynamic-symbol",
                                        "global", VER_NDX_GLOBAL, false});
    } else if (remove_prefix(arg, "--export-dynamic-symbol-list=")) {
      MappedFile<Context<E>> *mf = find_from_search_paths(ctx, std::string(arg));
      if (!mf)
        Fatal(ctx) << "--export-dynamic-symbol-list: file not found: " << arg;
      parse_dynamic_list(ctx, mf);
    } else if (arg == "--push-state") {
      state.push_back({ctx.as_needed, ctx.whole_archive, ctx.is_static,
                       ctx.in_lib});
    } else if (arg == "--pop-state") {
      if (state.empty())
        Fatal(ctx) << "no state pushed before popping";
      std::tie(ctx.as_needed, ctx.whole_archive, ctx.is_static, ctx.in_lib) =
        state.back();
      state.pop_back();
    } else if (remove_prefix(arg, "-l")) {
      MappedFile<Context<E>> *mf = find_library(ctx, std::string(arg));
      mf->given_fullpath = false;
      read_file(ctx, mf);
    } else {
      read_file(ctx, MappedFile<Context<E>>::must_open(ctx, std::string(arg)));
    }
  }

  if (ctx.objs.empty())
    Fatal(ctx) << "no input files";

  ctx.tg.wait();
}
```

首先是根据命令行参数确定要读取的输入文件，这里大部分的分支是为了读取符号version信息相关的，主要是看read_file的实现。在看实现之前可以看到传入了一个MappedFile，而这个类的实现其实就是在打开文件的时候使用了mmap进行映射，而must_open则是进行判断，失败了直接报错，这里也不贴具体细节代码了。

# read_file

```cpp
template <typename E>
void read_file(Context<E> &ctx, MappedFile<Context<E>> *mf) {
  if (ctx.visited.contains(mf->name))
    return;

  FileType type = get_file_type(mf);
	... 省略对不同type的处理
}
```

首先是get_file_type，这个是通过文件开头的值确定文件的类型，我们这里以ELF的代码为例。

```cpp
template <typename C>
FileType get_file_type(MappedFile<C> *mf) {
  std::string_view data = mf->get_contents();

  if (data.empty())
    return FileType::EMPTY;

  if (data.starts_with("\177ELF")) {
    u8 byte_order = ((elf::EL32Ehdr *)data.data())->e_ident[elf::EI_DATA];

    if (byte_order == elf::ELFDATA2LSB) {
      elf::EL32Ehdr &ehdr = *(elf::EL32Ehdr *)data.data();

      if (ehdr.e_type == elf::ET_REL) {
        if (ehdr.e_ident[elf::EI_CLASS] == elf::ELFCLASS32) {
          if (is_gcc_lto_obj<elf::I386>(mf))
            return FileType::GCC_LTO_OBJ;
        } else {
          if (is_gcc_lto_obj<elf::X86_64>(mf))
            return FileType::GCC_LTO_OBJ;
        }
        return FileType::ELF_OBJ;
      }

      if (ehdr.e_type == elf::ET_DYN)
        return FileType::ELF_DSO;
    } else {
      elf::EB32Ehdr &ehdr = *(elf::EB32Ehdr *)data.data();

      if (ehdr.e_type == elf::ET_REL) {
        if (ehdr.e_ident[elf::EI_CLASS] == elf::ELFCLASS32) {
          if (is_gcc_lto_obj<elf::M68K>(mf))
            return FileType::GCC_LTO_OBJ;
        } else {
          if (is_gcc_lto_obj<elf::SPARC64>(mf))
            return FileType::GCC_LTO_OBJ;
        }
        return FileType::ELF_OBJ;
      }

      if (ehdr.e_type == elf::ET_DYN)
        return FileType::ELF_DSO;
    }
    return FileType::UNKNOWN;
  }
  ... 省略其他格式的判断
}
```

先从数据开头的“\177ELF”确定为ELF文件，之后根据ELFHeader里面的内容读取更多的信息。

```cpp
enum class FileType {
  UNKNOWN,
  EMPTY,
  ELF_OBJ,
  ELF_DSO,
  MACH_OBJ,
  MACH_EXE,
  MACH_DYLIB,
  MACH_BUNDLE,
  MACH_UNIVERSAL,
  AR,
  THIN_AR,
  TAPI,
  TEXT,
  GCC_LTO_OBJ,
  LLVM_BITCODE,
};
```

mold当前所支持的FileType就是这些，但是注意，GitHub中mold项目下只存在elf文件的支持，mach的格式则是在sold这个项目中处理。除此之外的文件格式都在以下的switch中进行处理

[https://github.com/bluewhalesystems/sold](https://github.com/bluewhalesystems/sold)

```cpp
switch (type) {
case FileType::ELF_OBJ:
  ctx.objs.push_back(new_object_file(ctx, mf, ""));
  return;
case FileType::ELF_DSO:
  ctx.dsos.push_back(new_shared_file(ctx, mf));
  ctx.visited.insert(mf->name);
  return;
case FileType::AR:
case FileType::THIN_AR:
  for (MappedFile<Context<E>> *child : read_archive_members(ctx, mf)) {
    switch (get_file_type(child)) {
    case FileType::ELF_OBJ:
      ctx.objs.push_back(new_object_file(ctx, child, mf->name));
      break;
    case FileType::GCC_LTO_OBJ:
    case FileType::LLVM_BITCODE:
      if (ObjectFile<E> *file = new_lto_obj(ctx, child, mf->name))
        ctx.objs.push_back(file);
      break;
    default:
      break;
    }
  }
  ctx.visited.insert(mf->name);
  return;
case FileType::TEXT:
  parse_linker_script(ctx, mf);
  return;
case FileType::GCC_LTO_OBJ:
case FileType::LLVM_BITCODE:
  if (ObjectFile<E> *file = new_lto_obj(ctx, mf, ""))
    ctx.objs.push_back(file);
  return;
default:
  Fatal(ctx) << mf->name << ": unknown file type";
}
```

简化下来这里主要分为这么几类文件

1. archive file
2. lto
3. linker_script
4. object_file
5. shared_file

## archive file

archive file，也就是俗称的.a文件，其实就是许多个object文件塞到一起只需要解析其中所有member，之后将每个member进行读取即可。

```cpp
template <typename C>
std::vector<MappedFile<C> *>
read_archive_members(C &ctx, MappedFile<C> *mf) {
  switch (get_file_type(mf)) {
  case FileType::AR:
    return read_fat_archive_members(ctx, mf);
  case FileType::THIN_AR:
    return read_thin_archive_members(ctx, mf);
  default:
    unreachable();
  }
}
```

关于ar和thin ar

[ar (GNU Binary Utilities)](https://sourceware.org/binutils/docs/binutils/ar.html)

> An archive can either be *thin* or it can be normal. It cannot be both at the same time. Once an archive is created its format cannot be changed without first deleting it and then creating a new archive in its place.

这里的具体细节暂且略过，如感兴趣可自行查看源码

## lto

lto是用于link time optimization的文件，而本质上还是一个object文件，

```cpp
template <typename E>
static ObjectFile<E> *new_lto_obj(Context<E> &ctx, MappedFile<Context<E>> *mf,
                                  std::string archive_name) {
  static Counter count("parsed_lto_objs");
  count++;

  if (ctx.arg.ignore_ir_file.count(mf->get_identifier()))
    return nullptr;

  ObjectFile<E> *file = read_lto_object(ctx, mf);
  file->priority = ctx.file_priority++;
  file->archive_name = archive_name;
  file->is_in_lib = ctx.in_lib || (!archive_name.empty() && !ctx.whole_archive);
  file->is_alive = !file->is_in_lib;
  ctx.has_lto_object = true;
  if (ctx.arg.trace)
    SyncOut(ctx) << "trace: " << *file;
  return file;
}
```

在mold中解析lto的方式是通过指定plugin，加载对应的so来进行处理

```cpp
template <typename E>
ObjectFile<E> *read_lto_object(Context<E> &ctx, MappedFile<Context<E>> *mf) {
  // V0 API's claim_file is not thread-safe.
  static std::mutex mu;
  std::unique_lock lock(mu, std::defer_lock);
  if (!is_gcc_linker_api_v1)
    lock.lock();

  if (ctx.arg.plugin.empty())
    Fatal(ctx) << mf->name << ": don't know how to handle this LTO object file "
               << "because no -plugin option was given. Please make sure you "
               << "added -flto not only for creating object files but also for "
               << "creating the final executable.";

  // dlopen the linker plugin file
  static std::once_flag flag;
  std::call_once(flag, [&] { load_plugin(ctx); });
```

学习解析文件的实现主要是要进一步了解ELF的格式，所以这里具体细节就不进行考据了。

## linker script

mold的linker script根据解析的过程来看比较简单，没有在ld的脚本中的指定SECTION地址之类的内容，主要是对format以及符号version的一些控制。

```cpp
template <typename E>
void parse_linker_script(Context<E> &ctx, MappedFile<Context<E>> *mf) {
  current_file<E> = mf;

  std::vector<std::string_view> vec = tokenize(ctx, mf->get_contents());
  std::span<std::string_view> tok = vec;

  while (!tok.empty()) {
    if (tok[0] == "OUTPUT_FORMAT") {
      tok = read_output_format(ctx, tok.subspan(1));
    } else if (tok[0] == "INPUT" || tok[0] == "GROUP") {
      tok = read_group(ctx, tok.subspan(1));
    } else if (tok[0] == "VERSION") {
      tok = tok.subspan(1);
      tok = skip(ctx, tok, "{");
      read_version_script(ctx, tok);
      tok = skip(ctx, tok, "}");
    } else if (tok.size() > 3 && tok[1] == "=" && tok[3] == ";") {
      ctx.arg.defsyms.emplace_back(get_symbol(ctx, unquote(tok[0])),
                                   get_symbol(ctx, unquote(tok[2])));
      tok = tok.subspan(4);
    } else if (tok[0] == ";") {
      tok = tok.subspan(1);
    } else {
      SyntaxError(ctx, tok[0]) << "unknown linker script token";
    }
  }
}
```

## object file

object file是解析过程的重点之一。

```cpp
template <typename E>
static ObjectFile<E> *new_object_file(Context<E> &ctx, MappedFile<Context<E>> *mf,
                                      std::string archive_name) {
  static Counter count("parsed_objs");
  count++;

  check_file_compatibility(ctx, mf);

  bool in_lib = ctx.in_lib || (!archive_name.empty() && !ctx.whole_archive);
  ObjectFile<E> *file = ObjectFile<E>::create(ctx, mf, archive_name, in_lib);
  file->priority = ctx.file_priority++;
  ctx.tg.run([file, &ctx] { file->parse(ctx); });
  if (ctx.arg.trace)
    SyncOut(ctx) << "trace: " << *file;
  return file;
}
```

mold以链接速度快出名，而其快的原因之一就是充分利用了多线程，实际进行多线程操作的地方是在这里，ctx.tg.run，tg则是一个tbb::task_group，简而言之就是在这里开启了多线程的解析input file。

做了一些简单的in_lib参数处理，因为archive的链接机制默认是按需链接，而不是像shared file一样全部链接，之后在这里创建了object file并且开始parse。关于创建和parse的细节在后面再说。

## shared file

shared file同样是解析过程的重点之一。

```cpp
template <typename E>
static SharedFile<E> *
new_shared_file(Context<E> &ctx, MappedFile<Context<E>> *mf) {
  check_file_compatibility(ctx, mf);

  SharedFile<E> *file = SharedFile<E>::create(ctx, mf);
  file->priority = ctx.file_priority++;
  ctx.tg.run([file, &ctx] { file->parse(ctx); });
  if (ctx.arg.trace)
    SyncOut(ctx) << "trace: " << *file;
  return file;
}
```

在这里做了和object file类似的事情。

# InputFile

在详细讲解object file和shared file创建以及解析之前先介绍一下他们和InputFile类

![Untitled](/images/mold-1-read-input-files/Untitled.png)

ObjectFile和SharedFile都是简单的从InputFile中继承下来的。而这里的InputFile更像是代表了一个输入的ELF文件，构造的过程中做了一些ELF的基础解析，同时还提供了一些通用的接口，交由ObjectFile和SharedFile各自实现。

我们来看一下InputFile的构造函数部分

```cpp
template <typename E>
InputFile<E>::InputFile(Context<E> &ctx, MappedFile<Context<E>> *mf)
  : mf(mf), filename(mf->name) {
  if (mf->size < sizeof(ElfEhdr<E>))
    Fatal(ctx) << *this << ": file too small";
  if (memcmp(mf->data, "\177ELF", 4))
    Fatal(ctx) << *this << ": not an ELF file";

  ElfEhdr<E> &ehdr = *(ElfEhdr<E> *)mf->data;
  is_dso = (ehdr.e_type == ET_DYN);

  ElfShdr<E> *sh_begin = (ElfShdr<E> *)(mf->data + ehdr.e_shoff);

  // e_shnum contains the total number of sections in an object file.
  // Since it is a 16-bit integer field, it's not large enough to
  // represent >65535 sections. If an object file contains more than 65535
  // sections, the actual number is stored to sh_size field.
  i64 num_sections = (ehdr.e_shnum == 0) ? sh_begin->sh_size : ehdr.e_shnum;

  if (mf->data + mf->size < (u8 *)(sh_begin + num_sections))
    Fatal(ctx) << *this << ": e_shoff or e_shnum corrupted: "
               << mf->size << " " << num_sections;
  elf_sections = {sh_begin, sh_begin + num_sections};

  // e_shstrndx is a 16-bit field. If .shstrtab's section index is
  // too large, the actual number is stored to sh_link field.
  i64 shstrtab_idx = (ehdr.e_shstrndx == SHN_XINDEX)
    ? sh_begin->sh_link : ehdr.e_shstrndx;

  shstrtab = this->get_string(ctx, shstrtab_idx);
}
```

首先是从文件大小和文件头部标识信息进行ELF的校验，其次是做一些简单的解析。根据代码中可知，整个文件最开始的部分即可作为一个ElfEhdr（Ehdr：Elf Header）

根据header的信息可以解析出是否为dso文件，ElfShdr（Shdr：Section Header）的起始地址和长度，以及shstrtab（Section Header String Table）的位置。

大多数的参数直接可以获取，但是对于e_shnum和e_shstrndx来说，由于长度只有16bit的限制，因此如果值过大，则会分别存到第一个Shdr的sh_size以及sh_link中。

那么根据这段代码我们可以看出ELF的文件信息是这样的

# ObjectFile

## create

首先是ObjectFile的创建

```cpp
template <typename E>
ObjectFile<E> *
ObjectFile<E>::create(Context<E> &ctx, MappedFile<Context<E>> *mf,
                      std::string archive_name, bool is_in_lib) {
  ObjectFile<E> *obj = new ObjectFile<E>(ctx, mf, archive_name, is_in_lib);
  ctx.obj_pool.emplace_back(obj);
  return obj;
}

template <typename E>
ObjectFile<E>::ObjectFile(Context<E> &ctx, MappedFile<Context<E>> *mf,
                          std::string archive_name, bool is_in_lib)
  : InputFile<E>(ctx, mf), archive_name(archive_name), is_in_lib(is_in_lib) {
  this->is_alive = !is_in_lib;
}
```

ObjectFile的构造函数被放入了private中，因此必须通过静态的create方法来创建实例。在每次创建的时候会将对应的obj对象放入到全局的ctx.obj_pool中，mold中的内存与生命周期的管理方式则是全部交由ctx保有，到最后一起释放。而对应的obj_pool为了多线程的设计也都使用了并发的数据结构。

```cpp
 tbb::concurrent_vector<std::unique_ptr<ObjectFile<E>>>
```

ObjectFile的构造函数只是传递了参数，大部分的解析还是在InputFile的构造函数中执行。

## parse过程开始

```cpp
template <typename E>
void ObjectFile<E>::parse(Context<E> &ctx) {
  sections.resize(this->elf_sections.size());
  symtab_sec = this->find_section(SHT_SYMTAB);

  if (symtab_sec) {
    // In ELF, all local symbols precede global symbols in the symbol table.
    // sh_info has an index of the first global symbol.
    this->first_global = symtab_sec->sh_info;
    this->elf_syms = this->template get_data<ElfSym<E>>(ctx, *symtab_sec);
    this->symbol_strtab = this->get_string(ctx, symtab_sec->sh_link);
  }

  initialize_sections(ctx);
  initialize_symbols(ctx);
  sort_relocations(ctx);
  initialize_ehframe_sections(ctx);
}
```

## symtab_sec

首先是寻找symtab_sec的过程，寻找段的过程非常简单

```cpp
template <typename E>
ElfShdr<E> *InputFile<E>::find_section(i64 type) {
  for (ElfShdr<E> &sec : elf_sections)
    if (sec.sh_type == type)
      return &sec;
  return nullptr;
}
```

symtab_sec不存在的情况多半是strip了，直接在elf中搜索symtab是能搜到的，但是如果strip以后就无法找到这个段了，也就是为空的情况

![Untitled](/images/mold-1-read-input-files/Untitled%201.png)

sh_link和sh_info对于不同的section有不同的含义，对于这里的symtab来说sh_info就是保存了第一个global symbol的index，而sh_link就是保存了symbol_strtab的地址

![Untitled](/images/mold-1-read-input-files/Untitled%202.png)

## initialize_sections

```cpp
template <typename E>
void ObjectFile<E>::initialize_sections(Context<E> &ctx) {
  // Read sections
  for (i64 i = 0; i < this->elf_sections.size(); i++) {
    const ElfShdr<E> &shdr = this->elf_sections[i];
```

针对所有的sections开始处理，以下内容都在个循环体之中

### 特殊SHT的处理

SHT（Section Header Type）

```cpp
if ((shdr.sh_flags & SHF_EXCLUDE) && !(shdr.sh_flags & SHF_ALLOC) &&
    shdr.sh_type != SHT_LLVM_ADDRSIG && !ctx.arg.relocatable)
  continue;
```

这几个段无法在ELF标准中查到，后来查到了这么一段介绍

SHF_EXCLUDE：This section is excluded from input to the link-edit of an executable or shared object. This flag is ignored if the SHF_ALLOC flag is also set, or if relocations exist against the section.

1. 如果alloc被set则失效，因此这里要SHF_EXCLUDE以及SHF_ALLOC都满足条件
2. 同时sh_type为SHF_LLVM_ADDRSIG且不是relocatable

关于SHF_LLVM_ADDRSIG

[LLVM Extensions — LLVM 17.0.0git documentation](https://llvm.org/docs/Extensions.html#id20)

### groups

首先是关于Group的介绍

> This section defines a section group. A section group is a set of sections that are related and that must be treated specially by the linker (see [below](https://refspecs.linuxbase.org/elf/gabi4+/ch4.sheader.html#section_groups) for further details). Sections of type `SHT_GROUP` may appear only in relocatable objects (objects with the ELF header `e_type` member set to `ET_REL`). The section header table entry for a group section must appear in the section header table before the entries for any of the sections that are members of the group.

[https://refspecs.linuxbase.org/elf/gabi4+/ch4.sheader.html](https://refspecs.linuxbase.org/elf/gabi4+/ch4.sheader.html)

在实现中首先是寻找对应group的签名，签名是关联到了一个esym上，而这个符号的索引则是记录在sh_info中

```cpp
// Get the signature of this section group.
if (shdr.sh_info >= this->elf_syms.size())
  Fatal(ctx) << *this << ": invalid symbol index";
const ElfSym<E> &esym = this->elf_syms[shdr.sh_info];

std::string_view signature;
if (esym.st_type == STT_SECTION) {
  signature = this->shstrtab.data() +
              this->elf_sections[esym.st_shndx].sh_name;
} else {
  signature = this->symbol_strtab.data() + esym.st_name;
}
```

接下来就是一些特殊情况的处理。

1. 跳过wm4
2. 跳过entries[0]为0的情况
3. 如果[0]不是GRP_COMDAT则是错误

之后获取comdat group members，并使用signature来关联一个ComdatGroup

```cpp
// Ignore a broken comdat group GCC emits for .debug_macros.
// https://github.com/rui314/mold/issues/438
if (signature.starts_with("wm4."))
  continue;

// Get comdat group members.
std::span<U32<E>> entries = this->template get_data<U32<E>>(ctx, shdr);

if (entries.empty())
  Fatal(ctx) << *this << ": empty SHT_GROUP";
if (entries[0] == 0)
  continue;
if (entries[0] != GRP_COMDAT)
  Fatal(ctx) << *this << ": unsupported SHT_GROUP format";

typename decltype(ctx.comdat_groups)::const_accessor acc;
ctx.comdat_groups.insert(acc, {signature, ComdatGroup()});
ComdatGroup *group = const_cast<ComdatGroup *>(&acc->second);
comdat_groups.push_back({group, (u32)i, entries.subspan(1)});
break;
```

关于上面处理过程中出现的成员的定义

```cpp
// in context
tbb::concurrent_hash_map<std::string_view, ComdatGroup, HashCmp> comdat_groups;

// in ObjectFile
std::vector<ComdatGroupRef<E>> comdat_groups;

template <typename E>
struct ComdatGroupRef {
  ComdatGroup *group;
  u32 sect_idx;
  std::span<U32<E>> members;
};
```

首先是根据签名关联一个group空，之后将对应group的引用传递给ObjectFile中的comdat_groups

里面的i就是section的index

来看一下这个group段的排布

```cpp
| SectionSize | Group1SectionIndex | Group2SectionIndex | … |
```

关于GRP_COMDAT文档中也有提到

> This is a COMDAT group. It may duplicate another COMDAT group in another object file, where duplication is defined as having the same group signature. In such cases, only one of the duplicate groups may be retained by the linker, and the members of the remaining groups must be discarded.

### 常规SHT处理

此处还有很长的特殊段以及开启—gdb-index后需要处理的内容，并非重点，此处先跳过。

常规处理就是简单创建了一个InputSection

```cpp
this->sections[i] = std::make_unique<InputSection<E>>(ctx, *this, name, i);
```

### Attach relocation sections to their target sections.

到这里，所有的section已经执行过了一遍，最后再进行关联

```cpp
// Attach relocation sections to their target sections.
for (i64 i = 0; i < this->elf_sections.size(); i++) {
  const ElfShdr<E> &shdr = this->elf_sections[i];
  if (shdr.sh_type != (is_rela<E> ? SHT_RELA : SHT_REL))
    continue;

  if (shdr.sh_info >= sections.size())
    Fatal(ctx) << *this << ": invalid relocated section index: "
               << (u32)shdr.sh_info;

  if (std::unique_ptr<InputSection<E>> &target = sections[shdr.sh_info]) {
    assert(target->relsec_idx == -1);
    target->relsec_idx = i;
  }
}
```

针对RELA和REL处理，设置上对应的relsec_idx

```cpp
template <typename E>
static constexpr bool is_rela = requires(ElfRel<E> r) { r.r_addend; };
```

## initialize_symbols

这部分的过程主要是将esym转换为Symbol。esym则是ElfSym的缩写，也就是Elf文件中的Symbol定义，而Symbol则是mold中自己定义的，相当于转换为自己想要的格式。

这里的symtab_sec是parse刚开始的时候寻找的section，对应的符号表不存在则不进行这个过程。首先初始化了local_syms以及第0个符号

```cpp
template <typename E>
void ObjectFile<E>::initialize_symbols(Context<E> &ctx) {
  if (!symtab_sec)
    return;

  static Counter counter("all_syms");
  counter += this->elf_syms.size();

  // Initialize local symbols
  this->local_syms.resize(this->first_global);
  this->local_syms[0].file = this;
  this->local_syms[0].sym_idx = 0;
```

### local symbol

```cpp
for (i64 i = 1; i < this->first_global; i++) {
    const ElfSym<E> &esym = this->elf_syms[i];
    if (esym.is_common())
      Fatal(ctx) << *this << ": common local symbol?";

    std::string_view name;
    if (esym.st_type == STT_SECTION)
      name = this->shstrtab.data() + this->elf_sections[get_shndx(esym)].sh_name;
    else
      name = this->symbol_strtab.data() + esym.st_name;

    Symbol<E> &sym = this->local_syms[i];
    sym.set_name(name);
    sym.file = this;
    sym.value = esym.st_value;
    sym.sym_idx = i;

    if (!esym.is_abs())
      sym.set_input_section(sections[get_shndx(esym)].get());
  }
```

首先是对于common符号的判断

```cpp
bool is_common() const { return st_shndx == SHN_COMMON; }
```

关于这个SHN_COMMON

> SHN_COMMON Symbols defined relative to this section are common symbols, such as FORTRAN COMMON or unallocated C external variables.

大意是common的话不能是local，比如这里说的unallocated C external variables，external和local就是冲突的。

除了报错的common符号之外，其他符号在后面获取对应的名字，如果是section name则去shstrtab中寻找，否则就是常规的符号名，去symbol_strtab中寻找。这里的名字本质上是一个距离对应字符串段的offset，因为字符串相关的数据都统一保存在这shstrtab和symbol_strtab中了。

之后就是获取local_syms的引用，开始设置对应的信息。

在最后，对非abs符号的处理。

```cpp
bool is_abs() const { return st_shndx == SHN_ABS; }
```

> SHN_ABS This value specifies absolute values for the corresponding reference. For example, symbols defined relative to section number SHN_ABS have **absolute values and are not affected by relocation.**

非abs符号，也就是说都是相对地址，会affected by relocation。

而实际set_input_section则是设置其mask位，用于区分什么性质的符号。

```cpp
template <typename E>
inline void Symbol<E>::set_input_section(InputSection<E> *isec) {
  uintptr_t addr = (uintptr_t)isec;
  assert((addr & TAG_MASK) == 0);
  origin = addr | TAG_ISEC;
}
```

用2bit区分不同情况

```cpp
// A symbol usually belongs to an input section, but it can belong
// to a section fragment, an output section or nothing
// (i.e. absolute symbol). `origin` holds one of them. We use the
// least significant two bits to distinguish type.
enum : uintptr_t {
  TAG_ABS  = 0b00,
  TAG_ISEC = 0b01,
  TAG_OSEC = 0b10,
  TAG_FRAG = 0b11,
  TAG_MASK = 0b11,
};
```

### global symbol

```cpp
this->symbols.resize(this->elf_syms.size());

i64 num_globals = this->elf_syms.size() - this->first_global;
symvers.resize(num_globals);

for (i64 i = 0; i < this->first_global; i++)
  this->symbols[i] = &this->local_syms[i];
```

在开始处理之前可以看到这里又有两个resize容器的位置，目前为止有三处，这里写明了对应的容器以及所处的类，用于区分这个信息是否为ObjectFile only的

1. local symbols(InputFile)
2. symbols(InputFile)
3. symvers (ObjectFile)

之后将local_sym绑定到symbols中

之后是详细的处理过程

```cpp
// Initialize global symbols
for (i64 i = this->first_global; i < this->elf_syms.size(); i++) {
  const ElfSym<E> &esym = this->elf_syms[i];

  // Get a symbol name
  std::string_view key = this->symbol_strtab.data() + esym.st_name;
  std::string_view name = key;

  // Parse symbol version after atsign
  if (i64 pos = name.find('@'); pos != name.npos) {
    std::string_view ver = name.substr(pos + 1);
    name = name.substr(0, pos);

    if (!ver.empty() && ver != "@") {
      if (ver.starts_with('@'))
        key = name;
      if (!esym.is_undef())
        symvers[i - this->first_global] = ver.data();
    }
  }

  this->symbols[i] = insert_symbol(ctx, esym, key, name);
  if (esym.is_common())
    has_common_symbol = true;
}

std::vector<const char *> symvers;
```

这里不需要再区分是否为Section的符号，因为global符号不包含section符号。

这里最主要的是需要解析symbol version，因为有的符号会依赖于版本号。要注意的是这个东西并非ELF的官方定义，而是GNU的一个扩展，因此去看elf specification是找不到的。关于名称规范也很简单，常规符号名后接@加符号版本

解析符号版本完成后设置到symvers中，关于这个版本号，最常见的就是GLIBC，以下是本机helloworld代码的示范

```cpp
~/tmp > nm ./a.out | grep "@"
 w __cxa_finalize@GLIBC_2.2.5
 U __libc_start_main@GLIBC_2.34
 U puts@GLIBC_2.2.5
```

之后是insert symbol，并且设置其common属性。要注意除了这些解析方式外，global symbol和local symbol相比还有一个比较隐藏的不同，global symbol没有设置对应的file，后面很多符号的处理会进行判断file。

接下来是insert symbol的实现

```cpp
// Returns a symbol object for a given key. This function handles
// the -wrap option.
template <typename E>
static Symbol<E> *insert_symbol(Context<E> &ctx, const ElfSym<E> &esym,
                                std::string_view key, std::string_view name) {
  if (esym.is_undef() && name.starts_with("__real_") &&
      ctx.arg.wrap.contains(name.substr(7))) {
    return get_symbol(ctx, key.substr(7), name.substr(7));
  }

  Symbol<E> *sym = get_symbol(ctx, key, name);

  if (esym.is_undef() && sym->wrap) {
    key = save_string(ctx, "__wrap_" + std::string(key));
    name = save_string(ctx, "__wrap_" + std::string(name));
    return get_symbol(ctx, key, name);
  }
  return sym;
}

template <typename C>
std::string_view save_string(C &ctx, const std::string &str) {
  u8 *buf = new u8[str.size() + 1];
  memcpy(buf, str.data(), str.size());
  buf[str.size()] = '\0';
  ctx.string_pool.push_back(std::unique_ptr<u8[]>(buf));
  return {(char *)buf, str.size()};
}
```

这里不只是不存在key就创建key并返回那么简单。

1. 关于save_string的问题，这里也是和之前一样，创建了string后由ctx来管理生命周期，返回一个string_view提供使用。
2. 除此之外get_symbol的部分是实际执行了符号不存在则创建新符号并且返回的工作

```cpp
// If we haven't seen the same `key` before, create a new instance
// of Symbol and returns it. Otherwise, returns the previously-
// instantiated object. `key` is usually the same as `name`.
template <typename E>
Symbol<E> *get_symbol(Context<E> &ctx, std::string_view key,
                      std::string_view name) {
  typename decltype(ctx.symbol_map)::const_accessor acc;
  ctx.symbol_map.insert(acc, {key, Symbol<E>(name)});
  return const_cast<Symbol<E> *>(&acc->second);
}
```

1. 最后提一下-wrap option选项

这个-wrap是在main中read_input_files之前的地方设置的

```cpp
// Handle --wrap options if any.
for (std::string_view name : ctx.arg.wrap)
  get_symbol(ctx, name)->wrap = true;
```

关于这个选项我参考了这个回答里的内容，虽然是gcc的介绍，但是本质是相同的

[How to wrap functions with the `--wrap` option correctly?](https://stackoverflow.com/questions/46444052/how-to-wrap-functions-with-the-wrap-option-correctly)

我摘选了一些关键的段落

> -wrap=symbol
> Use a wrapper function for symbol. Any undefined reference to symbol will be resolved to "__wrap_symbol". Any undefined reference to "__real_symbol" will be resolved to symbol.
> ...
> If you link other code with this file using --wrap malloc, then all calls to "malloc" will call the function "__wrap_malloc" instead. The call to "__real_malloc" in "__wrap_malloc" will call the real "malloc" function.

> ... Any **undefined reference** to symbol will be resolved to "__wrap_symbol". Any **undefined reference**  to "__real_symbol" will be resolved to symbol.

至此，initialize_symbols就结束了

## sort_relocations

```cpp
// Relocations are usually sorted by r_offset in relocation tables,
// but for some reason only RISC-V does not follow that convention.
// We expect them to be sorted, so sort them if necessary.
template <typename E>
void ObjectFile<E>::sort_relocations(Context<E> &ctx) {
  if constexpr (is_riscv<E>) {
    auto less = [&](const ElfRel<E> &a, const ElfRel<E> &b) {
      return a.r_offset < b.r_offset;
    };

    for (i64 i = 1; i < sections.size(); i++) {
      std::unique_ptr<InputSection<E>> &isec = sections[i];
      if (!isec || !isec->is_alive || !(isec->shdr().sh_flags & SHF_ALLOC))
        continue;

      std::span<ElfRel<E>> rels = isec->get_rels(ctx);
      if (!std::is_sorted(rels.begin(), rels.end(), less))
        sort(rels, less);
    }
  }
}
```

根据注释，这里的sort是为了将不遵守约定没按照r_offset排序的rv的relocations转换为遵循约定的格式

## initialize_ehframe_sections

关于这里的内容比较长，不仅要包含解析本身，还有ehframe本身的内容，因此留到下期再继续讲。

# 图解总结

画了一些比较粗糙的图示将今天的内容串联起来（未标记长度信息，部分大小不标准，没精力画了）

![Untitled](/images/mold-1-read-input-files/Untitled%203.png)

首先是读取InputFile时的流程，主要是ElfHeader指向ELF文件的哪一部分

![Untitled](/images/mold-1-read-input-files/Untitled%204.png)

其次是读取Section的时候符号表相关的查找流程，这里还没来得及画具体取名字的部分

从Section Header Table中找到对应sh_type为SHT_SYMTAB的段，之后根据offset和size找到具体存放symbol的位置，同时通过sh_info确定第一个global symbol的index

# 参考资料汇总

[Sections](https://refspecs.linuxbase.org/elf/gabi4+/ch4.sheader.html)

[Elf Specification 1.2](https://refspecs.linuxfoundation.org/elf/elf.pdf)
