---
title: mold源码阅读 其三 符号决议
typora-root-url: ../../source
date: 2023-04-09 16:06:20
category: Linker
tags: 
  - [mold]
  - [symbol_resolve]
---

![Untitled](/images/mold-3-symbol-resolve/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:72272420</center> 

前面两期将读取输入的部分全部讲完了，本期开始涉及链接过程中的处理。在讲主要的符号决议之前，先讲一下mold在符号决议执行之前做的一些其他处理。

# dso uniquely

在读取完输入后首先做的是将shared object根据soname进行去重，因此我们可以在链接的过程中链接多个相同soname的库而不会产生冲突。

elf/main.cc

```cpp
{
    std::unordered_set<std::string_view> seen;
    std::erase_if(ctx.dsos, [&](SharedFile<E> *file) {
      return !seen.insert(file->soname).second;
    });
}
```

# apply_exclude_libs

elf/passes.cc

```cpp
template <typename E>
void apply_exclude_libs(Context<E> &ctx) {
  Timer t(ctx, "apply_exclude_libs");

  if (ctx.arg.exclude_libs.empty())
    return;

  std::unordered_set<std::string_view> set(ctx.arg.exclude_libs.begin(),
                                           ctx.arg.exclude_libs.end());

  for (ObjectFile<E> *file : ctx.objs) {
    if (!file->archive_name.empty())
      if (set.contains("ALL") ||
          set.contains(filepath(file->archive_name).filename().string()))
        file->exclude_libs = true;
  }
}
```

这里就是简单将所有包含在exclude-libs里的lib名字对应的exclude_libs设置为true，而这个设置在后面符号决议的过程会用到。

exclude_libs是命令行中获取的

```cpp
else if (read_arg("exclude-libs")) {
      append(ctx.arg.exclude_libs, split_by_comma_or_colon(arg));
}
```

# create_internal_file

## internal file是什么

内部的文件，用来保存linker-synthesized符号。linker-synthesized符号或许也可以理解为编译产物中不存在的符号。作为一个并不实际存在的文件，依然会作为一个普通的ObjFile加入到obj_pool中，主要用途是在create_output_sections以后来add_synthetic_symbol，与之相关联的有一个internal_esyms，里面都是具体相关的符号。

## 实现

在main函数中调用是这样的

```cpp
if (!ctx.arg.relocatable)
  create_internal_file(ctx);
```

elf/passes.cc

```cpp
template <typename E>
void create_internal_file(Context<E> &ctx) {
  ObjectFile<E> *obj = new ObjectFile<E>;
  ctx.obj_pool.emplace_back(obj);
  ctx.internal_obj = obj;
  ctx.objs.push_back(obj);

  ctx.internal_esyms.resize(1);

  obj->symbols.push_back(new Symbol<E>);
  obj->first_global = 1;
  obj->is_alive = true;
  obj->priority = 1;
```

首先创建了基本的ObjectFile对象并且进行了一些初始化的处理。

之后添加从命令行参数中读取的--defsym里的所有的defsym

```cpp
	for (i64 i = 0; i < ctx.arg.defsyms.size(); i++) {
	  std::pair<Symbol<E> *, std::variant<Symbol<E> *, u64>> &defsym = ctx.arg.defsyms[i];
	  add(defsym.first);
	
	  if (std::holds_alternative<Symbol<E> *>(defsym.second)) {
	    // Add an undefined symbol to keep a reference to the defsym target.
	    // This prevents elimination by e.g. LTO or gc-sections.
	    // The undefined symbol will never make to the final object file; we
	    // double-check that the defsym target is not undefined in
	    // fix_synthetic_symbols.
	    auto sym = std::get<Symbol<E> *>(defsym.second);
	    add_undef(sym);
	  }
	}
```

处理完defsym后再从命令行参数中读取的SectionOrder的符号

```cpp
for (SectionOrder &ord : ctx.arg.section_order)
  if (ord.type == SectionOrder::SYMBOL)
    add(get_symbol(ctx, ord.name));
```

关于SectionOrder的定义

```cpp
struct SectionOrder {
  enum { NONE, SECTION, GROUP, ADDR, ALIGN, SYMBOL } type = NONE;
  std::string name;
  u64 value = 0;
};
```

最后设置obj类的一些参数

```cpp
obj->elf_syms = ctx.internal_esyms;
obj->symvers.resize(ctx.internal_esyms.size() - 1);
```

## defsym

关于前面提到的defsym，我们来看一下mold的测试代码一部分来理解其作用。

```cpp
cat <<EOF | $CC -fPIC -o $t/a.o -c -xc -
#include <stdio.h>
extern char foo;
extern char bar;
void baz();

void print() {
  printf("Hello %p %p\n", &foo, &bar);
}

int main() {
  baz();
}
EOF

$CC -B. -o $t/exe $t/a.o -pie -Wl,-defsym=foo=16 \
  -Wl,-defsym=bar=0x2000 -Wl,-defsym=baz=print

$QEMU $t/exe | grep -q '^Hello 0x10 0x2000$'
```

通过defsym指定了符号名以及其实现的位置，尽管对应的符号在代码中并没有实现。

关于这里的add和add_undef的实现

```cpp
auto add = [&](Symbol<E> *sym) {
  obj->symbols.push_back(sym);

  // An actual value will be set to a linker-synthesized symbol by
  // fix_synthetic_symbols(). Until then, `value` doesn't have a valid
  // value. 0xdeadbeef is a unique dummy value to make debugging easier
  // if the field is accidentally used before it gets a valid one.
  sym->value = 0xdeadbeef;

  ElfSym<E> esym;
  memset(&esym, 0, sizeof(esym));
  esym.st_type = STT_NOTYPE;
  esym.st_shndx = SHN_ABS;
  esym.st_bind = STB_GLOBAL;
  esym.st_visibility = STV_DEFAULT;
  ctx.internal_esyms.push_back(esym);
};

auto add_undef = [&](Symbol<E> *sym) {
  obj->symbols.push_back(sym);
  sym->value = 0xdeadbeef;

  ElfSym<E> esym;
  memset(&esym, 0, sizeof(esym));
  esym.st_type = STT_NOTYPE;
  esym.st_shndx = SHN_UNDEF;
  esym.st_bind = STB_GLOBAL;
  esym.st_visibility = STV_DEFAULT;
  ctx.internal_esyms.push_back(esym);
};
```

通过add和add_undef函数把defsym指定的符号添加到symbols中，并且设定为了特殊值，关联到了一个esym里。主要的差别就在于st_shndx被设置为了不同的值。

# 符号决议

接下来是链接过程中比较重要的一个环节，符号决议（symbol resolve）

在mold中，这个部分做了四件事情

1. 检测所有需要使用的objet files
2. 移除重复的COMDAT段
3. 进行符号决议的过程。在多个不同的esym中选择出一个更高priority的关联到sym中
4. LTO的处理，处理后再次执行决议

## resolve_symbols

```cpp
template <typename E>
void resolve_symbols(Context<E> &ctx) {
  Timer t(ctx, "resolve_symbols");

  std::vector<ObjectFile<E> *> objs = ctx.objs;
  std::vector<SharedFile<E> *> dsos = ctx.dsos;

  do_resolve_symbols(ctx);

  if (ctx.has_lto_object) {
		...
		do_resolve_symbols(ctx);
  }
}
```

这里先省略lto相关的具体处理，很多处理和do_resolve_symbols中是类似的，因此放到后面再说。

```cpp
template <typename E>
void do_resolve_symbols(Context<E> &ctx) {
  auto for_each_file = [&](std::function<void(InputFile<E> *)> fn) {
    tbb::parallel_for_each(ctx.objs, fn);
    tbb::parallel_for_each(ctx.dsos, fn);
  };
	// 1. 检测object files
	// 2. 消除重复COMDAT
	// 3. 符号决议
}
```

## 1. 检测object files

archive extraction: .a成员只会在满足非archive object文件未定义符号之一的情况下才会被包含在最终的二进制文件中

链接时为了满足archive extraction的规则，mold采取的策略是：

1. 初步resolve：假设全部include，match undef符号
2. mark sweep消除无需使用的archive成员
3. 删除掉非archive成员

下面是具体的实现

```cpp
{
    Timer t(ctx, "extract_archive_members");

    // Register symbols
    for_each_file([&](InputFile<E> *file) { file->resolve_symbols(ctx); });

    // Mark reachable objects to decide which files to include into an output.
    // This also merges symbol visibility.
    mark_live_objects(ctx);

    // Cleanup. The rule used for archive extraction isn't accurate for the
    // general case of symbol extraction, so reset the resolution to be redone
    // later.
    for_each_file([](InputFile<E> *file) { file->clear_symbols(); });

    // Now that the symbol references are gone, remove the eliminated files from
    // the file list.
    std::erase_if(ctx.objs, [](InputFile<E> *file) { return !file->is_alive; });
    std::erase_if(ctx.dsos, [](InputFile<E> *file) { return !file->is_alive; });
  }
```

1. for_each_file针对objs和dsos处理resolve，mark live，clear，erase file
2. 标记所有可访问的输出到文件的object，之后合并可见性
3. 清除file的symbols
4. 最后清除掉objs和dsos中非alive的file

```cpp
template <typename E>
__attribute__((no_sanitize("thread")))
void InputFile<E>::clear_symbols() {
  for (Symbol<E> *sym : get_global_syms())
    if (sym->file == this)
      sym->clear();
}
```

### mark_live_objects

```cpp
template <typename E>
static void mark_live_objects(Context<E> &ctx) {
  auto mark_symbol = [&](std::string_view name) {
    if (InputFile<E> *file = get_symbol(ctx, name)->file)
      file->is_alive = true;
  };

  for (std::string_view name : ctx.arg.undefined)
    mark_symbol(name);
  for (std::string_view name : ctx.arg.require_defined)
    mark_symbol(name);

  std::vector<InputFile<E> *> roots;

  for (InputFile<E> *file : ctx.objs)
    if (file->is_alive)
      roots.push_back(file);

  for (InputFile<E> *file : ctx.dsos)
    if (file->is_alive)
      roots.push_back(file);

  tbb::parallel_for_each(roots, [&](InputFile<E> *file,
                                    tbb::feeder<InputFile<E> *> &feeder) {
    if (file->is_alive)
      file->mark_live_objects(ctx, [&](InputFile<E> *obj) { feeder.add(obj); });
  });
}
```

首先对所有参数中传入的undef以及require_define的符号所关联的文件进行mark，之后遍历所有alive的obj和dso，加入到root中，之后再进行mark_live_objects。部分文件会因为特殊的链接选项，比如说whole-archive会影响是否设置为is_alive，这部分会之后再以这个为主题单独讲一篇。

对于ObjectFile和SharedFile的mark方式也是不同的。

### ObjectFile::mark_live_objects

```cpp
template <typename E>
void
ObjectFile<E>::mark_live_objects(Context<E> &ctx,
                                 std::function<void(InputFile<E> *)> feeder) {
  assert(this->is_alive);

  for (i64 i = this->first_global; i < this->elf_syms.size(); i++) {
    const ElfSym<E> &esym = this->elf_syms[i];
    Symbol<E> &sym = *this->symbols[i];

    if (!esym.is_undef() && exclude_libs)
      merge_visibility(ctx, sym, STV_HIDDEN);
    else
      merge_visibility(ctx, sym, esym.st_visibility);

    if (sym.traced)
      print_trace_symbol(ctx, *this, esym, sym);

    if (esym.is_weak())
      continue;

    if (!sym.file)
      continue;

    bool keep = esym.is_undef() || (esym.is_common() && !sym.esym().is_common());
    if (keep && fast_mark(sym.file->is_alive)) {
      feeder(sym.file);

      if (sym.traced)
        SyncOut(ctx) << "trace-symbol: " << *this << " keeps " << *sym.file
                     << " for " << sym;
    }
  }
}
```

针对所有global的符号进行处理。

1. 首先对esym进行merge_visibility，对于存在定义的exclude_libs的符号来说是HIDDEN的，关于这一点在命令行参数处有说明。

   > -exclude-libs LIB,LIB,.. Mark all symbols in given libraries hidden

2. 跳过弱符号以及文件不存在的符号。

3. keep并且fast_mark成功的符号加入到root中。

keep这里需要是undef的情况，我认为是因为如果esym存在定义，那么定义就是存在于当前的ObjectFile中，也就不需要再重复加入到root中了。common我想同样是因为这种情况下所在的文件已经加入到root中了。

关于merge_visibility

```cpp
template <typename E>
void ObjectFile<E>::merge_visibility(Context<E> &ctx, Symbol<E> &sym,
                                     u8 visibility) {
  // Canonicalize visibility
  if (visibility == STV_INTERNAL)
    visibility = STV_HIDDEN;

  auto priority = [&](u8 visibility) {
    switch (visibility) {
    case STV_HIDDEN:
      return 1;
    case STV_PROTECTED:
      return 2;
    case STV_DEFAULT:
      return 3;
    }
    Fatal(ctx) << *this << ": unknown symbol visibility: " << sym;
  };

  update_minimum(sym.visibility, visibility, [&](u8 a, u8 b) {
    return priority(a) < priority(b);
  });
}
```

这里首先将INTERNAL转换为HIDDEN，之后按照最小的priority来更新visibility。

update_minimum只是一个针对多线程的封装，本质上是一个compare and exchange操作。

```cpp
template <typename T, typename Compare = std::less<T>>
void update_minimum(std::atomic<T> &atomic, u64 new_val, Compare cmp = {}) {
  T old_val = atomic.load(std::memory_order_relaxed);
  while (cmp(new_val, old_val) &&
         !atomic.compare_exchange_weak(old_val, new_val,
                                       std::memory_order_relaxed));
}
```

fast_mark也是针对多线程的一个封装，如果是false则更新为true。

```cpp
// An optimized "mark" operation for parallel mark-and-sweep algorithms.
// Returns true if `visited` was false and updated to true.
inline bool fast_mark(std::atomic<bool> &visited) {
  // A relaxed load + branch (assuming miss) takes only around 20 cycles,
  // while an atomic RMW can easily take hundreds on x86. We note that it's
  // common that another thread beat us in marking, so doing an optimistic
  // early test tends to improve performance in the ~20% ballpark.
  return !visited.load(std::memory_order_relaxed) &&
         !visited.exchange(true, std::memory_order_relaxed);
}
```

### SharedFile::mark_live_objects

```cpp
template <typename E>
void
SharedFile<E>::mark_live_objects(Context<E> &ctx,
                                 std::function<void(InputFile<E> *)> feeder) {
  for (i64 i = 0; i < this->elf_syms.size(); i++) {
    const ElfSym<E> &esym = this->elf_syms[i];
    Symbol<E> &sym = *this->symbols[i];

    if (sym.traced)
      print_trace_symbol(ctx, *this, esym, sym);

    if (esym.is_undef() && sym.file && !sym.file->is_dso &&
        fast_mark(sym.file->is_alive)) {
      feeder(sym.file);

      if (sym.traced)
        SyncOut(ctx) << "trace-symbol: " << *this << " keeps " << *sym.file
                     << " for " << sym;
    }
  }
}
```

这里要说的唯一一点是因为mark的是object而不是shared file，因此dso的情况下不会进行mark

## 2. 移除重复的COMDAT段

```cpp
{
  Timer t(ctx, "eliminate_comdats");

  tbb::parallel_for_each(ctx.objs, [](ObjectFile<E> *file) {
    file->resolve_comdat_groups();
  });

  tbb::parallel_for_each(ctx.objs, [](ObjectFile<E> *file) {
    file->eliminate_duplicate_comdat_groups();
  });
}
```

```cpp
template <typename E>
void ObjectFile<E>::resolve_comdat_groups() {
  for (ComdatGroupRef<E> &ref : comdat_groups)
    update_minimum(ref.group->owner, this->priority);
}
```

更新所有comdat_groups的priority

```cpp
template <typename E>
void ObjectFile<E>::eliminate_duplicate_comdat_groups() {
  for (ComdatGroupRef<E> &ref : comdat_groups)
    if (ref.group->owner != this->priority)
      for (u32 i : ref.members)
        if (sections[i])
          sections[i]->kill();
}

template <typename E>
inline void InputSection<E>::kill() {
  if (is_alive.exchange(false))
    for (FdeRecord<E> &fde : get_fdes())
      fde.is_alive = false;
}
```

eliminate重复的section。将section和fde的is_alive都设置为false。

这里将fde设置为false正对应了上期提到单独解析eh_frame的原因之一：消除重复的fde。

## 3. 实际进行符号决议的过程

```cpp
for_each_file([&](InputFile<E> *file) { file->resolve_symbols(ctx); });
```

resolve_symbols的实现对于ObjectFile和SharedFile是不同的

### ObjectFile

```cpp
template <typename E>
void ObjectFile<E>::resolve_symbols(Context<E> &ctx) {
  for (i64 i = this->first_global; i < this->elf_syms.size(); i++) {
    Symbol<E> &sym = *this->symbols[i];
    const ElfSym<E> &esym = this->elf_syms[i];

    if (esym.is_undef())
      continue;

    InputSection<E> *isec = nullptr;
    if (!esym.is_abs() && !esym.is_common()) {
      isec = get_section(esym);
      if (!isec || !isec->is_alive)
        continue;
    }

    std::scoped_lock lock(sym.mu);
    if (get_rank(this, esym, !this->is_alive) < get_rank(sym)) {
      sym.file = this;
      sym.set_input_section(isec);
      sym.value = esym.st_value;
      sym.sym_idx = i;
      sym.ver_idx = ctx.default_version;
      sym.is_weak = esym.is_weak();
      sym.is_imported = false;
      sym.is_exported = false;
    }
  }
}
```

符号决议是针对global symbol的elf_sym的。未定义的esym都跳过了，它们都不需要参与resolve的过程，因为resolve本质是找到需要加入到生成产物的符号实现，但是注意在前面mark的时候还是需要的。

也就是说Symbol类的sym其实是保留的最终唯一定义。而在决议的过程，不断的将esym和对应的sym进行比较。如果esym的rank小，也就是更加优先，那么就将sym中的信息更新为对应的esym的信息，这就是实际决议过程中做的事情。而这里也是实际初始化symbols成员里global的值的地方，local的部分初始化在parse的阶段就做好了，因为local的符号并不需要进行resolve。

### get_rank

```cpp
template <typename E>
static u64 get_rank(const Symbol<E> &sym) {
  if (!sym.file)
    return 7 << 24;
  return get_rank(sym.file, sym.esym(), !sym.file->is_alive);
}

// Symbols with higher priorities overwrites symbols with lower priorities.
// Here is the list of priorities, from the highest to the lowest.
//
//  1. Strong defined symbol
//  2. Weak defined symbol
//  3. Strong defined symbol in a DSO/archive
//  4. Weak Defined symbol in a DSO/archive
//  5. Common symbol
//  6. Common symbol in an archive
//  7. Unclaimed (nonexistent) symbol
//
// Ties are broken by file priority.
template <typename E>
static u64 get_rank(InputFile<E> *file, const ElfSym<E> &esym, bool is_lazy) {
  if (esym.is_common()) {
    assert(!file->is_dso);
    if (is_lazy)
      return (6 << 24) + file->priority;
    return (5 << 24) + file->priority;
  }

  if (file->is_dso || is_lazy) {
    if (esym.st_bind == STB_WEAK)
      return (4 << 24) + file->priority;
    return (3 << 24) + file->priority;
  }
  if (esym.st_bind == STB_WEAK)
    return (2 << 24) + file->priority;
  return (1 << 24) + file->priority;
}
```

get_rank的实现，根据注释我们可以看到不同类别符号的优先级。

### SharedFile

```cpp
template <typename E>
void SharedFile<E>::resolve_symbols(Context<E> &ctx) {
  for (i64 i = 0; i < this->symbols.size(); i++) {
    Symbol<E> &sym = *this->symbols[i];
    const ElfSym<E> &esym = this->elf_syms[i];
    if (esym.is_undef())
      continue;

    std::scoped_lock lock(sym.mu);

    if (get_rank(this, esym, false) < get_rank(sym)) {
      sym.file = this;
      sym.origin = 0;
      sym.value = esym.st_value;
      sym.sym_idx = i;
      sym.ver_idx = versyms[i];
      sym.is_weak = false;
      sym.is_imported = false;
      sym.is_exported = false;
    }
  }
}
```

对于SharedFile中的符号中不需要考虑是否是global的问题，上期解析SharedFile的部分也有提到，对应的first_global就是0。相对于

## 4. LTO的处理

```cpp
if (ctx.has_lto_object) {
  // Do link-time optimization. We pass all IR object files to the
  // compiler backend to compile them into a few ELF object files.
  //
  // The compiler backend needs to know how symbols are resolved,
  // so compute symbol visibility, import/export bits, etc early.
  mark_live_objects(ctx);
  apply_version_script(ctx);
  parse_symbol_version(ctx);
  compute_import_export(ctx);

  // Do LTO. It compiles IR object files into a few big ELF files.
  std::vector<ObjectFile<E> *> lto_objs = do_lto(ctx);

  // do_resolve_symbols() have removed unreferenced files. Restore the
  // original files here because some of them may have to be resurrected
  // because they are referenced by the ELF files returned from do_lto().
  ctx.objs = objs;
  ctx.dsos = dsos;

  append(ctx.objs, lto_objs);

  // Redo name resolution from scratch.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->clear_symbols();
    file->is_alive = !file->is_in_lib;
  });

  tbb::parallel_for_each(ctx.dsos, [&](SharedFile<E> *file) {
    file->clear_symbols();
    file->is_alive = !file->is_needed;
  });

  // Remove IR object files.
  for (ObjectFile<E> *file : ctx.objs)
    if (file->is_lto_obj)
      file->is_alive = false;

  std::erase_if(ctx.objs, [](ObjectFile<E> *file) { return file->is_lto_obj; });

  do_resolve_symbols(ctx);
}
```

关于apply_version_script, parse_symbol_version, compute_import_export这三个过程，会在之后的过程中讲解。这里简单来讲就是获取符号对应的版本信息以及对应import/export的属性。

这个部分做了这么几件事情

1. 计算出符号所需的基本信息
2. 实际执行lto
3. 将lto结果的object加入到全局
4. 清理旧的lto文件
5. 再次执行do_resolve_symbols整个过程
