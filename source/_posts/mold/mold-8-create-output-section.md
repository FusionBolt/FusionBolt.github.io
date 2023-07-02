---
title: mold源码阅读八 创建输出段
typora-root-url: ../../source
date: 2023-06-10 16:24:45
category: Linker
tags: mold
---

![Untitled](/images/mold-8-create-output-section/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:101015341_p18</center> 

上一期介绍了一些创建输出段之前的工作，本期主要是把创建输出相关的最后一些前置准备讲解完成。根据代码中的注释，add_synthetic_symbols以后，不会再有任何新的文件添加到ctx.objs和ctx.dsos中了。之后会再讲解简单的命令行参数处理，下一期再讲对于输出chunk中的一些处理

# create output sections

```cpp
// Create output sections for input sections.
template <typename E>
void create_output_sections(Context<E> &ctx) {
  Timer t(ctx, "create_output_sections");

  struct Cmp {
    size_t operator()(const OutputSectionKey &k) const {
      u64 h = hash_string(k.name);
      h = combine_hash(h, std::hash<u64>{}(k.type));
      h = combine_hash(h, std::hash<u64>{}(k.flags));
      return h;
    }
  };

  std::unordered_map<OutputSectionKey, OutputSection<E> *, Cmp> map;
  std::shared_mutex mu;
```

1. 首先针对所有的InputSection生成一个key，并且根据key创建所有的OutputSection
2. 将所有obj中的InputSection加入到对应OutputSection的members中
3. 对所有的output section和mergeable section加入到chunks
4. 将所有的chunk进行排序
5. 所有的chunk加入到ctx.chunks中（在加入之前chunks中有一些synthetic的chunk，在上一期中有提及）

以下是这五个过程的代码

```cpp
// Instantiate output sections
tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
  for (std::unique_ptr<InputSection<E>> &isec : file->sections) {
    if (!isec || !isec->is_alive)
      continue;

    OutputSectionKey key = get_output_section_key(ctx, *isec);

    {
      std::shared_lock lock(mu);
      auto it = map.find(key);
      if (it != map.end()) {
        isec->output_section = it->second;
        continue;
      }
    }

    std::unique_ptr<OutputSection<E>> osec =
      std::make_unique<OutputSection<E>>(key.name, key.type, key.flags);
    std::unique_lock lock(mu);

    auto [it, inserted] = map.insert({key, osec.get()});
    isec->output_section = it->second;
    if (inserted)
      ctx.osec_pool.emplace_back(std::move(osec));
  }
});
```

```cpp
// Add input sections to output sections
  for (ObjectFile<E> *file : ctx.objs)
    for (std::unique_ptr<InputSection<E>> &isec : file->sections)
      if (isec && isec->is_alive)
        isec->output_section->members.push_back(isec.get());
```

```cpp
// Add output sections and mergeable sections to ctx.chunks
  std::vector<Chunk<E> *> vec;
  for (std::pair<const OutputSectionKey, OutputSection<E> *> &kv : map)
    vec.push_back(kv.second);

  for (std::unique_ptr<MergedSection<E>> &osec : ctx.merged_sections)
    if (osec->shdr.sh_size)
      vec.push_back(osec.get());
```

```cpp
// Sections are added to the section lists in an arbitrary order
  // because they are created in parallel. Sort them to to make the
  // output deterministic.
  tbb::parallel_sort(vec.begin(), vec.end(), [](Chunk<E> *x, Chunk<E> *y) {
    return std::tuple(x->name, x->shdr.sh_type, x->shdr.sh_flags) <
           std::tuple(y->name, y->shdr.sh_type, y->shdr.sh_flags);
  });
```

```cpp
append(ctx.chunks, vec);
```

## get_output_section_key

这个函数的作用是从一个InputSection构造一个OutputSectionKey

```cpp
template <typename E>
static OutputSectionKey
get_output_section_key(Context<E> &ctx, InputSection<E> &isec) {
  const ElfShdr<E> &shdr = isec.shdr();
  std::string_view name = get_output_name(ctx, isec.name(), shdr.sh_flags);
  u64 type = canonicalize_type<E>(name, shdr.sh_type);
  u64 flags = shdr.sh_flags & ~(u64)SHF_COMPRESSED;

  if (!ctx.arg.relocatable)
    flags &= ~(u64)SHF_GROUP & ~(u64)SHF_GNU_RETAIN;

  // .init_array is usually writable. We don't want to create multiple
  // .init_array output sections, so make it always writable.
  // So is .fini_array.
  if (type == SHT_INIT_ARRAY || type == SHT_FINI_ARRAY)
    flags |= SHF_WRITE;
  return {name, type, flags};
}
```

```cpp
struct OutputSectionKey {
  std::string_view name;
  u64 type;
  u64 flags;

  bool operator==(const OutputSectionKey &other) const {
    return name == other.name && type == other.type && flags == other.flags;
  }
};
```

从InputSection获取的output name。这里有以下几种情况

1. 返回原始名字

2. 忽略段名字的后缀

   不知道这里应该用什么术语，还是举个例子，比如说里面的.ARM.exidx，如果有.ARM.exidx.f1以及.ARM.exidx.f2，那么这两个的名字都会归为.ARM.exidx

3. 将一些特殊的text段单独分开，而不是合并为一个text。这里涉及到了一个z_keep_text_section_prefix的编译选项，命令行的介绍是

   > z keep-text-section-prefix Keep .text.{hot,unknown,unlikely,startup,exit} as separate sections in the final binary

4. 对于text等特定段则是只保留原始前缀，比如说所有的.text.xxx最后都会合并到一个.text段。这个对于函数定义非常常见，查看编译产物的时候，经常会看到一些.text.function_name，最后都会合并为一个.text，这种合并其实就是这里实现的。

```cpp
template <typename E>
std::string_view
get_output_name(Context<E> &ctx, std::string_view name, u64 flags) {
  if (ctx.arg.relocatable && !ctx.arg.relocatable_merge_sections)
    return name;
  if (ctx.arg.unique && ctx.arg.unique->match(name))
    return name;
  if (flags & SHF_MERGE)
    return name;

  if (name.starts_with(".ARM.exidx"))
    return ".ARM.exidx";
  if (name.starts_with(".ARM.extab"))
    return ".ARM.extab";

  if (ctx.arg.z_keep_text_section_prefix) {
    static std::string_view prefixes[] = {
      ".text.hot.", ".text.unknown.", ".text.unlikely.", ".text.startup.",
      ".text.exit."
    };

    for (std::string_view prefix : prefixes) {
      std::string_view stem = prefix.substr(0, prefix.size() - 1);
      if (name == stem || name.starts_with(prefix))
        return stem;
    }
  }

  static std::string_view prefixes[] = {
    ".text.", ".data.rel.ro.", ".data.", ".rodata.", ".bss.rel.ro.", ".bss.",
    ".init_array.", ".fini_array.", ".tbss.", ".tdata.", ".gcc_except_table.",
    ".ctors.", ".dtors.", ".gnu.warning.",
  };

  for (std::string_view prefix : prefixes) {
    std::string_view stem = prefix.substr(0, prefix.size() - 1);
    if (name == stem || name.starts_with(prefix))
      return stem;
  }

  return name;
}
```

# add synthetic symbols

这里的功能如名字一样，就是添加一些synthetic的符号，添加后将这些符号关联到ctx.symtab中

```cpp
template <typename E>
void add_synthetic_symbols(Context<E> &ctx) {
  ObjectFile<E> &obj = *ctx.internal_obj;

  auto add = [&](std::string_view name) {
    ElfSym<E> esym;
    memset(&esym, 0, sizeof(esym));
    esym.st_type = STT_NOTYPE;
    esym.st_shndx = SHN_ABS;
    esym.st_bind = STB_GLOBAL;
    esym.st_visibility = STV_HIDDEN;
    ctx.internal_esyms.push_back(esym);

    Symbol<E> *sym = get_symbol(ctx, name);
    sym->value = 0xdeadbeef; // unique dummy value
    obj.symbols.push_back(sym);
    return sym;
  };
```

```cpp
ctx.__ehdr_start = add("__ehdr_start");
ctx.__init_array_start = add("__init_array_start");
ctx.__init_array_end = add("__init_array_end");
ctx.__fini_array_start = add("__fini_array_start");
ctx.__fini_array_end = add("__fini_array_end");
ctx.__preinit_array_start = add("__preinit_array_start");
ctx.__preinit_array_end = add("__preinit_array_end");
ctx._DYNAMIC = add("_DYNAMIC");
ctx._GLOBAL_OFFSET_TABLE_ = add("_GLOBAL_OFFSET_TABLE_");
ctx._PROCEDURE_LINKAGE_TABLE_ = add("_PROCEDURE_LINKAGE_TABLE_");
ctx.__bss_start = add("__bss_start");
ctx._end = add("_end");
ctx._etext = add("_etext");
ctx._edata = add("_edata");
ctx.__executable_start = add("__executable_start");

ctx.__rel_iplt_start =
  add(is_rela<E> ? "__rela_iplt_start" : "__rel_iplt_start");
ctx.__rel_iplt_end =
  add(is_rela<E> ? "__rela_iplt_end" : "__rel_iplt_end");

if (ctx.arg.eh_frame_hdr)
  ctx.__GNU_EH_FRAME_HDR = add("__GNU_EH_FRAME_HDR");

if (!get_symbol(ctx, "end")->file)
  ctx.end = add("end");
if (!get_symbol(ctx, "etext")->file)
  ctx.etext = add("etext");
if (!get_symbol(ctx, "edata")->file)
  ctx.edata = add("edata");
if (!get_symbol(ctx, "__dso_handle")->file)
  ctx.__dso_handle = add("__dso_handle");
```

添加通用的特殊符号

```cpp
if constexpr (supports_tlsdesc<E>)
  ctx._TLS_MODULE_BASE_ = add("_TLS_MODULE_BASE_");

if constexpr (is_riscv<E>)
  if (!ctx.arg.shared)
    ctx.__global_pointer = add("__global_pointer$");

if constexpr (std::is_same_v<E, ARM32>) {
  ctx.__exidx_start = add("__exidx_start");
  ctx.__exidx_end = add("__exidx_end");
}

if constexpr (is_ppc<E>)
  ctx.TOC = add(".TOC.");
```

针对特殊平台添加特定的符号

```cpp
for (Chunk<E> *chunk : ctx.chunks) {
    if (std::optional<std::string> name = get_start_stop_name(ctx, *chunk)) {
      add(save_string(ctx, "__start_" + *name));
      add(save_string(ctx, "__stop_" + *name));

      if (ctx.arg.physical_image_base) {
        add(save_string(ctx, "__phys_start_" + *name));
        add(save_string(ctx, "__phys_stop_" + *name));
      }
    }
  }
```

针对特殊名字的trunk的处理

```cpp
obj.elf_syms = ctx.internal_esyms;
obj.symvers.resize(ctx.internal_esyms.size() - 1);

obj.resolve_symbols(ctx);
```

对internal_obj进行symbol resolve

```cpp
// Make all synthetic symbols relative ones by associating them to
// a dummy output section.
for (Symbol<E> *sym : obj.symbols)
  if (sym->file == &obj)
    sym->set_output_section(ctx.symtab);
```

符号关联到symtab这个output section里

```cpp
// Handle --defsym symbols.
for (i64 i = 0; i < ctx.arg.defsyms.size(); i++) {
  Symbol<E> *sym = ctx.arg.defsyms[i].first;
  std::variant<Symbol<E> *, u64> val = ctx.arg.defsyms[i].second;

  Symbol<E> *target = nullptr;
  if (Symbol<E> **ref = std::get_if<Symbol<E> *>(&val))
    target = *ref;

  // If the alias refers another symobl, copy ELF symbol attributes.
  if (target) {
    ElfSym<E> &esym = obj.elf_syms[i + 1];
    esym.st_type = target->esym().st_type;
    if constexpr (requires { esym.ppc_local_entry; })
      esym.ppc_local_entry = target->esym().ppc_local_entry;
  }

  // Make the target absolute if necessary.
  if (!target || target->is_absolute())
    sym->origin = 0;
}
```

对于—defsym指定的符号进行处理

# check_cet_errors

```cpp
// Handle `-z cet-report`.
  if (ctx.arg.z_cet_report != CET_REPORT_NONE)
    check_cet_errors(ctx);
```

首先，cet是Control Flow Enforcement Technology的缩写。简单来说就是预防控制流攻击的一种技术

> Control-flow Enforcement Technology (CET) covers several related x86 processor features that provide protection against control flow hijacking attacks. CET can protect both applications and the kernel.
>
>
> CET introduces shadow stack and indirect branch tracking (IBT). A shadow stack is a secondary stack allocated from memory which cannot be directly modified by applications. When executing a CALL instruction, the processor pushes the return address to both the normal stack and the shadow stack. Upon function return, the processor pops the shadow stack copy and compares it to the normal stack copy. If the two differ, the processor raises a control-protection fault. IBT verifies indirect CALL/JMP targets are intended as marked by the compiler with 'ENDBR' opcodes. Not all CPU's have both Shadow Stack and Indirect Branch Tracking. Today in the 64-bit kernel, only userspace shadow stack and kernel IBT are supported.

[Control-flow Enforcement Technology (CET) Shadow Stack — The Linux Kernel  documentation](https://www.kernel.org/doc/html/next/x86/shstk.html)

cet_report有三类

```cpp
typedef enum {
  CET_REPORT_NONE,
  CET_REPORT_WARNING,
  CET_REPORT_ERROR,
} CetReportKind;
```

这个函数是用于进行针对每个file检查对应的gnu_properties，如果没有满足特定feature的话抛出warning或者error。

ELF中必须包含GNU_PROPERTY_X86_FEATURE_1_IBT和GNU_PROPERTY_X86_FEATURE_1_SHSTK属性才能支持cet。

```cpp
void check_cet_errors(Context<E> &ctx) {
  bool warning = (ctx.arg.z_cet_report == CET_REPORT_WARNING);
  assert(warning || (ctx.arg.z_cet_report == CET_REPORT_ERROR));

  auto has_feature = [](ObjectFile<E> *file, u32 feature) {
    return std::any_of(file->gnu_properties.begin(), file->gnu_properties.end(),
                       [&](auto kv) {
                         return kv.first == GNU_PROPERTY_X86_FEATURE_1_AND
                             && (kv.second & feature);
                       });
  };

  for (ObjectFile<E> *file : ctx.objs) {
    if (file == ctx.internal_obj)
      continue;
    if (!has_feature(file, GNU_PROPERTY_X86_FEATURE_1_IBT)) {
      if (warning)
        Warn(ctx) << *file << ": -cet-report=warning: "
                  << "missing GNU_PROPERTY_X86_FEATURE_1_IBT";
      else
        Error(ctx) << *file << ": -cet-report=error: "
                   << "missing GNU_PROPERTY_X86_FEATURE_1_IBT";
    }

    if (!has_feature(file, GNU_PROPERTY_X86_FEATURE_1_SHSTK)) {
      if (warning)
        Warn(ctx) << *file << ": -cet-report=warning: "
                  << "missing GNU_PROPERTY_X86_FEATURE_1_SHSTK";
      else
        Error(ctx) << *file << ": -cet-report=error: "
                   << "missing GNU_PROPERTY_X86_FEATURE_1_SHSTK";
    }
  }
}
```

# execstack-if-needed

```cpp
// Handle `-z execstack-if-needed`.
if (ctx.arg.z_execstack_if_needed)
  for (ObjectFile<E> *file : ctx.objs)
    if (file->needs_executable_stack)
      ctx.arg.z_execstack = true;
```

所有的obj中，如果有needs_executable_stack为true的情况，那么设置ctx中的arg。obj中的这个属性是在ObjectFile::initialize_sections中设置的。而全局的z_execstack会在后面被用到，此时先不过多提及。

```cpp
// .note.GNU-stack section controls executable-ness of the stack
// area in GNU linkers. We ignore that section because silently
// making the stack area executable is too dangerous. Tell our
// users about the difference if that matters.
if (name == ".note.GNU-stack" && !ctx.arg.relocatable) {
  if (shdr.sh_flags & SHF_EXECINSTR) {
    if (!ctx.arg.z_execstack && !ctx.arg.z_execstack_if_needed)
      Warn(ctx) << *this << ": this file may cause a segmentation"
        " fault because it requires an executable stack. See"
        " https://github.com/rui314/mold/tree/main/docs/execstack.md"
        " for more info.";
    needs_executable_stack = true;
  }
  continue;
}
```
