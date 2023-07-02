---
title: mold源码阅读九 未解析符号的处理
typora-root-url: ../../source
date: 2023-06-19 21:54:17
category: Linker
tags: mold
---

![Untitled](/images/mold-9-unresolve-symbol/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:101015341_p16</center> 

本期内容主要是claim_unresolved_symbols的部分，其次是其他一些简单的处理

# claim_unresolved_symbols

```cpp
// If we are linking a .so file, remaining undefined symbols does
// not cause a linker error. Instead, they are treated as if they
// were imported symbols.
//
// If we are linking an executable, weak undefs are converted to
// weakly imported symbols so that they'll have another chance to be
// resolved.
claim_unresolved_symbols(ctx);
```

```cpp
template <typename E>
void claim_unresolved_symbols(Context<E> &ctx) {
  Timer t(ctx, "claim_unresolved_symbols");
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->claim_unresolved_symbols(ctx);
  });
}
```

这个函数主要还是针对需要在链接期就确定定义的符号进行检查，针对部分符号产生一些修改，在这个过程之后，不会再有符号发生新的变动了

对so来说undef是可以存在的，因此将避免报错，将undef的符号转换为imported，并且修改相关信息。

但是如果是protected或者hidden的符号即便链接了运行时也无法访问到，此时即便是undef也无法再在运行时找到定义，因此需要在链接时确定定义。也正因为这些条件，这里只需要对global符号做检查即可。

以下是具体处理过程

```cpp
template <typename E>
void ObjectFile<E>::claim_unresolved_symbols(Context<E> &ctx) {
  if (!this->is_alive)
    return;

	...
  for (i64 i = this->first_global; i < this->elf_syms.size(); i++) {
    const ElfSym<E> &esym = this->elf_syms[i];
    Symbol<E> &sym = *this->symbols[i];
    if (!esym.is_undef())
      continue;

    std::scoped_lock lock(sym.mu);

    // If a protected/hidden undefined symbol is resolved to an
    // imported symbol, it's handled as if no symbols were found.
    if (sym.file && sym.file->is_dso &&
        (sym.visibility == STV_PROTECTED || sym.visibility == STV_HIDDEN)) {
      report_undef(sym);
      continue;
    }

    if (sym.file &&
        (!sym.esym().is_undef() || sym.file->priority <= this->priority))
      continue;

    // If a symbol name is in the form of "foo@version", search for
    // symbol "foo" and check if the symbol has version "version".
    std::string_view key = this->symbol_strtab.data() + esym.st_name;
    if (i64 pos = key.find('@'); pos != key.npos) {
      Symbol<E> *sym2 = get_symbol(ctx, key.substr(0, pos));
      if (sym2->file && sym2->file->is_dso &&
          sym2->get_version() == key.substr(pos + 1)) {
        this->symbols[i] = sym2;
        continue;
      }
    }
		
		...
    if (esym.is_undef_weak()) {
      if (ctx.arg.shared && sym.visibility != STV_HIDDEN &&
          ctx.arg.z_dynamic_undefined_weak) {
        // Global weak undefined symbols are promoted to dynamic symbols
        // when when linking a DSO, unless `-z nodynamic_undefined_weak`
        // was given.
        claim(true);
      } else {
        // Otherwise, weak undefs are converted to absolute symbols with value 0.
        claim(false);
      }
      continue;
    }

    if (ctx.arg.unresolved_symbols == UNRESOLVED_WARN)
      report_undef(sym);

    // Traditionally, remaining undefined symbols cause a link failure
    // only when we are creating an executable. Undefined symbols in
    // shared objects are promoted to dynamic symbols, so that they'll
    // get another chance to be resolved at run-time. You can change the
    // behavior by passing `-z defs` to the linker.
    //
    // Even if `-z defs` is given, weak undefined symbols are still
    // promoted to dynamic symbols for compatibility with other linkers.
    // Some major programs, notably Firefox, depend on the behavior
    // (they use this loophole to export symbols from libxul.so).
    if (ctx.arg.shared && sym.visibility != STV_HIDDEN &&
        (!ctx.arg.z_defs || ctx.arg.unresolved_symbols != UNRESOLVED_ERROR)) {
      claim(true);
      continue;
    }

    // Convert remaining undefined symbols to absolute symbols with value 0.
    if (ctx.arg.unresolved_symbols != UNRESOLVED_ERROR || ctx.arg.noinhibit_exec)
      claim(false);
  }
}
```

如同上面所说，整个过程描述如下

1. 从全局符号开始，先跳过了已经有定义的esym

2. 将protected和hidden的符号进行报错

3. 对esym对应位置的sym进行判断，如果sym所对应的esym是有定义的也跳过。

   这种情况是esym实际的定义在其他位置，sym是esym resolve的结果

4. 解析符号名，如果带有版本信息则再次尝试进行重新将esym和sym进行关联。这个关联体现在esym对应index的symbols重新设置值

   ```cpp
   if (sym2->file && sym2->file->is_dso &&
       sym2->get_version() == key.substr(pos + 1)) {
     this->symbols[i] = sym2;
     continue;
   }
   ```

5. 针对undef_weak进行claim

6. 剩下的undef的符号在创建executable的时候导致链接失败，但在dso中会被提升为dynamic symbols

claim和report_undef的实现

```cpp
auto report_undef = [&](Symbol<E> &sym) {
  std::stringstream ss;
  if (std::string_view source = this->get_source_name(); !source.empty())
    ss << ">>> referenced by " << source << "\n";
  else
    ss << ">>> referenced by " << *this << "\n";

  typename decltype(ctx.undef_errors)::accessor acc;
  ctx.undef_errors.insert(acc, {sym.name(), {}});
  acc->second.push_back(ss.str());
};

// tbb::concurrent_hash_map<std::string_view, std::vector<std::string>> undef_errors;
```

```cpp
auto claim = [&](bool is_imported) {
  if (sym.traced)
    SyncOut(ctx) << "trace-symbol: " << *this << ": unresolved"
                 << (esym.is_weak() ? " weak" : "")
                 << " symbol " << sym;

  sym.file = this;
  sym.origin = 0;
  sym.value = 0;
  sym.sym_idx = i;
  sym.is_weak = false;
  sym.is_imported = is_imported;
  sym.is_exported = false;
  sym.ver_idx = is_imported ? 0 : ctx.default_version;
};
```

# print dependencies

```cpp
// Handle --print-dependencies
if (ctx.arg.print_dependencies == 1)
  print_dependencies(ctx);
else if (ctx.arg.print_dependencies == 2)
  print_dependencies_full(ctx);
```

针对所有的obj和dso打印其依赖，那么具体怎么样才算依赖呢？在一个obj a里面，有一个未定义的符号，链接的时候另一个obj b包含了这个符号的定义，那么这就算是a依赖b。

```cpp
template <typename E>
void print_dependencies(Context<E> &ctx) {
  SyncOut(ctx) <<
R"(# This is an output of the mold linker's --print-dependencies option.
#
# Each line consists of three fields, <file1>, <file2> and <symbol>
# separated by tab characters. It indicates that <file1> depends on
# <file2> to use <symbol>.)";

  auto print = [&](InputFile<E> *file) {
    for (i64 i = file->first_global; i < file->elf_syms.size(); i++) {
      ElfSym<E> &esym = file->elf_syms[i];
      Symbol<E> &sym = *file->symbols[i];
      if (esym.is_undef() && sym.file && sym.file != file)
        SyncOut(ctx) << *file << "\t" << *sym.file << "\t" << sym;
    }
  };

  for (InputFile<E> *file : ctx.objs)
    print(file);
  for (InputFile<E> *file : ctx.dsos)
    print(file);
}
```

这种是最简单的遍历所有文件打印其依赖，包含了obj a，obj b以及对应符号的名字

```cpp
template <typename E>
void print_dependencies_full(Context<E> &ctx) {
  SyncOut(ctx) <<
R"(# This is an output of the mold linker's --print-dependencies=full option.
#
# Each line consists of 4 fields, <section1>, <section2>, <symbol-type> and
# <symbol>, separated by tab characters. It indicates that <section1> depends
# on <section2> to use <symbol>. <symbol-type> is either "u" or "w" for
# regular undefined or weak undefined, respectively.
#
# If you want to obtain dependency information per function granularity,
# compile source files with the -ffunction-sections compiler flag.)";

  auto println = [&](auto &src, Symbol<E> &sym, ElfSym<E> &esym) {
    if (InputSection<E> *isec = sym.get_input_section())
      SyncOut(ctx) << src << "\t" << *isec
                   << "\t" << (esym.is_weak() ? 'w' : 'u')
                   << "\t" << sym;
    else
      SyncOut(ctx) << src << "\t" << *sym.file
                   << "\t" << (esym.is_weak() ? 'w' : 'u')
                   << "\t" << sym;
  };

  for (ObjectFile<E> *file : ctx.objs) {
    for (std::unique_ptr<InputSection<E>> &isec : file->sections) {
      if (!isec)
        continue;

      std::unordered_set<void *> visited;

      for (const ElfRel<E> &r : isec->get_rels(ctx)) {
        if (r.r_type == R_NONE)
          continue;

        ElfSym<E> &esym = file->elf_syms[r.r_sym];
        Symbol<E> &sym = *file->symbols[r.r_sym];

        if (esym.is_undef() && sym.file && sym.file != file &&
            visited.insert((void *)&sym).second)
          println(*isec, sym, esym);
      }
    }
  }

  for (SharedFile<E> *file : ctx.dsos) {
    for (i64 i = file->first_global; i < file->symbols.size(); i++) {
      ElfSym<E> &esym = file->elf_syms[i];
      Symbol<E> &sym = *file->symbols[i];
      if (esym.is_undef() && sym.file && sym.file != file)
        println(*file, sym, esym);
    }
  }
}
```

这种更复杂一些，不仅打印依赖，还包含了符号到底是undefined还是weak这一信息。

另外遍历objs的时候还针对每个obj遍历InputSection及其包含的rel，根据这些信息来进行打印。

遍历dsos的判断条件则是和上面最简单的打印是相同的。

# write_repro_file

```cpp
// Handle -repro
if (ctx.arg.repro)
  write_repro_file(ctx);
```

> --repro                     Embed input files to .repro section

repro file是Reproducible Example Routine file的简称，包含最小可复现用例，用于调试。具体写的过程并非这里关注的重点，有兴趣可以自行查看更多细节，这里只简单看一下由哪些部分组成。

```cpp
template <typename E>
void write_repro_file(Context<E> &ctx) {
  std::string path = ctx.arg.output + ".repro.tar";

  std::unique_ptr<TarWriter> tar =
    TarWriter::open(path, filepath(ctx.arg.output).filename().string() + ".repro");
  if (!tar)
    Fatal(ctx) << "cannot open " << path << ": " << errno_string();

  tar->append("response.txt", save_string(ctx, create_response_file(ctx)));
  tar->append("version.txt", save_string(ctx, mold_version + "\n"));

  std::unordered_set<std::string> seen;
  for (std::unique_ptr<MappedFile<Context<E>>> &mf : ctx.mf_pool) {
    if (!mf->parent) {
      std::string path = to_abs_path(mf->name).string();
      if (seen.insert(path).second) {
        // We reopen a file because we may have modified the contents of mf
        // in memory, which is mapped with PROT_WRITE and MAP_PRIVATE.
        MappedFile<Context<E>> *mf2 = MappedFile<Context<E>>::must_open(ctx, path);
        tar->append(path, mf2->get_contents());
        mf2->unmap();
      }
    }
  }
}

template <typename E>
static std::string create_response_file(Context<E> &ctx) {
  std::string buf;
  std::stringstream out;

  std::string cwd = std::filesystem::current_path().string();
  out << "-C " << cwd.substr(1) << "\n";

  if (cwd != "/") {
    out << "--chroot ..";
    i64 depth = std::count(cwd.begin(), cwd.end(), '/');
    for (i64 i = 1; i < depth; i++)
      out << "/..";
    out << "\n";
  }

  for (i64 i = 1; i < ctx.cmdline_args.size(); i++) {
    std::string_view arg = ctx.cmdline_args[i];
    if (arg != "-repro" && arg != "--repro")
      out << arg << "\n";
  }
  return out.str();
}
```

根据代码我们得知，主要分为三部分

1. response_file，本质上是编译命令以及参数
2. mold的version info
3. 所有的输入文件

也就表示这三者就是确定问题的必要条件，另外还可以认为执行到这里之后符号不会再发生什么改动，也不会产生新的用户引发的问题（比如说少链接文件，或者什么参数错了导致符号决议出问题等）

# required-defined

```cpp
// Handle --require-defined
for (std::string_view name : ctx.arg.require_defined)
  if (!get_symbol(ctx, name)->file)
    Error(ctx) << "--require-defined: undefined symbol: " << name;
```

强制要求某些符号是必须在链接时就包含定义的，对这些符号进行检查并且进行报错。
