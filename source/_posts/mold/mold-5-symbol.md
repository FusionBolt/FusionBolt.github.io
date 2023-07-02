---
title: mold源码阅读五 符号相关
typora-root-url: ../../source
date: 2023-04-29 17:09:46
category: Linker
tags: mold
---

![Untitled](/images/mold-5-symbol/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:73005507</center> 

上期讲完了resolve_section_pieces，在这之后本应是combine_object，但是combine_object几乎包含了后面的所有过程，因此等到整个流程讲完后或许会再回来讲，这一期的内容以符号版本的处理为主。

# 为common symbol创建bss段

```cpp
// Create .bss sections for common symbols.
convert_common_symbols(ctx);
```

```cpp
template <typename E>
void convert_common_symbols(Context<E> &ctx) {
  Timer t(ctx, "convert_common_symbols");

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->convert_common_symbols(ctx);
  });
}
```

## common symbol

首先解释一下common symbol。根据mold的注释所讲，common symbols被用于C的tentative definition，tentative definition是指C语言在一个头文件中允许全局变量定义省略extern。header会存在于多个翻译单元中，但这个符号不会导致duplicate symbol error，相反linker会将他们merge到一个单一的实例中。

还给出了一个例子，比如说一个头文件中有一个tentative definition是int foo，在一个C文件中包含了其包含初始值定义，比如说int foo = 5（real definition），那么这个tentative definition的符号会被resolve到real definition上。如果没有real definition，那么tentative definition会得到默认值。

[About Tentative definition](https://stackoverflow.com/questions/3095861/about-tentative-definition)

参考这个stackoverflow的回答，C语言中纯变量声明会被处理为extern的，我想这就是允许省略extern的原因，编译器帮你做了这件事情，尽管这或许与你的预期不符。

简单总结来说就是头文件中一个全局的声明在不同编译单元有不同定义的时候需要进行resolve一个单一实现，声明的symbol其实是属于多个文件的，因此是common的。

## 实现

```cpp
template <typename E>
void ObjectFile<E>::convert_common_symbols(Context<E> &ctx) {
  if (!has_common_symbol)
    return;
```

has_common_symbol初始化的地方是在input-files.cc中的void ObjectFile<E>::initialize_symbols

```cpp
// initialize_symbols
this->symbols[i] = insert_symbol(ctx, esym, key, name);
    if (esym.is_common())
      has_common_symbol = true;
  }
}
```

实际的处理是针对所有global的common符号

```cpp
for (i64 i = this->first_global; i < this->elf_syms.size(); i++) {
  if (!this->elf_syms[i].is_common())
    continue;

  Symbol<E> &sym = *this->symbols[i];
  std::scoped_lock lock(sym.mu);

  if (sym.file != this) {
    if (ctx.arg.warn_common)
      Warn(ctx) << *this << ": multiple common symbols: " << sym;
    continue;
  }
...
```

提示warning

```cpp
// in for
elf_sections2.push_back({});
ElfShdr<E> &shdr = elf_sections2.back();
memset(&shdr, 0, sizeof(shdr));

std::string_view name;

if (sym.get_type() == STT_TLS) {
  name = ".tls_common";
  shdr.sh_flags = SHF_ALLOC | SHF_WRITE | SHF_TLS;
} else {
  name = ".common";
  shdr.sh_flags = SHF_ALLOC | SHF_WRITE;
}

shdr.sh_type = SHT_NOBITS;
shdr.sh_size = this->elf_syms[i].st_size;
shdr.sh_addralign = this->elf_syms[i].st_value;
```

关于SHF_TLS

> SHF_TLS: This section holds Thread-Local Storage, meaning that each separate execution flow has its own distinct instance of this data. Implementations need not support this flag.

[https://refspecs.linuxbase.org/elf/gabi4+/ch4.sheader.html](https://refspecs.linuxbase.org/elf/gabi4+/ch4.sheader.html)

对每个global common符号创建了一个ElfShdr后开始设置其基本信息

```cpp
i64 idx = this->elf_sections.size() + elf_sections2.size() - 1;
std::unique_ptr<InputSection<E>> isec =
  std::make_unique<InputSection<E>>(ctx, *this, name, idx);

sym.file = this;
sym.set_input_section(isec.get());
sym.value = 0;
sym.sym_idx = i;
sym.ver_idx = ctx.default_version;
sym.is_weak = false;
sym.is_imported = false;
sym.is_exported = false;

sections.push_back(std::move(isec));
// for end
```

创建了一个指向elf_sections的InputSecion，之后添加到sections中。

# apply_version_script

```cpp
// Apply version scripts.
apply_version_script(ctx);
```

## simple

```cpp
template <typename E>
void apply_version_script(Context<E> &ctx) {
  Timer t(ctx, "apply_version_script");

  // If all patterns are simple (i.e. not containing any meta-
  // characters and is not a C++ name), we can simply look up
  // symbols.
  auto is_simple = [&] {
    for (VersionPattern &v : ctx.version_patterns)
      if (v.is_cpp || v.pattern.find_first_of("*?[") != v.pattern.npos)
        return false;
    return true;
  };
```

首先定义了is_simple。

simple的定义

1. 非cpp name。VersionPattern是根据链接器参数创建与添加的，is_cpp也同样是在那时指定的。
2. 不包含meta字符的名字。根据代码中，meta字符即是*[?中的char

```cpp
if (is_simple()) {
    for (VersionPattern &v : ctx.version_patterns) {
      Symbol<E> *sym = get_symbol(ctx, v.pattern);

      if (!sym->file && !ctx.arg.undefined_version)
        Warn(ctx) << v.source << ": cannot assign version `" << v.ver_str
                  << "` to symbol `" << *sym << "`: symbol not found";

      if (sym->file && !sym->file->is_dso)
        sym->ver_idx = v.ver_idx;
    }
    return;
  }
```

如果是simple的那么直接针对所有的version_pattern找到对应的符号，并且将其通过设置ver_idx的方式与VersionPattern进行关联

## otherwise

```cpp
// Otherwise, use glob pattern matchers.
MultiGlob matcher;
MultiGlob cpp_matcher;

for (i64 i = 0; i < ctx.version_patterns.size(); i++) {
  VersionPattern &v = ctx.version_patterns[i];
  if (v.is_cpp) {
    if (!cpp_matcher.add(v.pattern, i))
      Fatal(ctx) << "invalid version pattern: " << v.pattern;
  } else {
    if (!matcher.add(v.pattern, i))
      Fatal(ctx) << "invalid version pattern: " << v.pattern;
  }
}
```

首先将cpp和其他复杂的符号区分开

```cpp
tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
  for (Symbol<E> *sym : file->get_global_syms()) {
    if (sym->file != file)
      continue;

    std::string_view name = sym->name();
    i64 match = INT64_MAX;

    if (std::optional<u32> idx = matcher.find(name))
      match = std::min<i64>(match, *idx);

    // Match non-mangled symbols against the C++ pattern as well.
    // Weird, but required to match other linkers' behavior.
    if (!cpp_matcher.empty()) {
      if (std::optional<std::string_view> s = cpp_demangle(name))
        name = *s;
      if (std::optional<u32> idx = cpp_matcher.find(name))
        match = std::min<i64>(match, *idx);
    }

    if (match != INT64_MAX)
      sym->ver_idx = ctx.version_patterns[match].ver_idx;
  }
});
```

在所有obj文件中找到所有global symbol

如果普通matcher中能找到其名字，那么更新match的index

如果cpp matcher非空，那么对其进行demangle操作，之后在cpp matcher中寻找其名字并且更新match的index

最后将符号与对应VersionPattern进行关联。

而这里的demangle操作是直接调用对应平台的abi。starts_with(”_Z”)是代表这是一个mangling的名字。关于mangling的规则参考

[https://github.com/gchatelet/gcc_cpp_mangling_documentation](https://github.com/gchatelet/gcc_cpp_mangling_documentation)

```cpp
std::optional<std::string_view> cpp_demangle(std::string_view name) {
  static thread_local char *buf;
  static thread_local size_t buflen;

  // TODO(cwasser): Actually demangle Symbols on Windows using e.g.
  // `UnDecorateSymbolName` from Dbghelp, maybe even Itanium symbols?
#ifndef _WIN32
  if (name.starts_with("_Z")) {
    int status;
    char *p = abi::__cxa_demangle(std::string(name).c_str(), buf, &buflen, &status);
    if (status == 0) {
      buf = p;
      return p;
    }
  }
#endif

  return {};
}
```

## obj only

我在读到这里，很好奇为什么只针对的是obj而不考虑dso，看了下相关的命令行参数的介绍才明白过来

> -E, --export-dynamic Put symbols in the dynamic symbol table --no-export-dynamic

# parse_symbol_version

```cpp
// Parse symbol version suffixes (e.g. "foo@ver1").
parse_symbol_version(ctx);
```

```cpp
template <typename E>
void parse_symbol_version(Context<E> &ctx) {
  if (!ctx.arg.shared)
    return;
```

这是针对生成shared库的操作，因为只有动态链接才需要考虑加载符号版本的问题，符号版本是为了加载动态库的时候确保更新后符号的实现一致，如果和预想的实现不一致可能引起其他问题。

```cpp
std::unordered_map<std::string_view, u16> verdefs;
for (i64 i = 0; i < ctx.arg.version_definitions.size(); i++)
  verdefs[ctx.arg.version_definitions[i]] = i + VER_NDX_LAST_RESERVED + 1;
```

实现中首先是将每一个version信息绑定到一个i + VER_NDX_LAST_RESERVED + 1的值（其中的version_definitions则是在read_version_script中读取的）。

接下来是针对了所有的global object进行操作。

```cpp
tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (i64 i = 0; i < file->elf_syms.size() - file->first_global; i++) {
      // Match VERSION part of symbol foo@VERSION with version definitions.
      // The symbols' VERSION parts are in file->symvers.
      if (!file->symvers[i])
        continue;

      Symbol<E> *sym = file->symbols[i + file->first_global];
      if (sym->file != file)
        continue;

      std::string_view ver = file->symvers[i];
			...
  });
```

首先找到了对应文件的global符号

```cpp
bool is_default = false;
if (ver.starts_with('@')) {
  is_default = true;
  ver = ver.substr(1);
}
```

获取符号的版本具体的值

```cpp
auto it = verdefs.find(ver);
if (it == verdefs.end()) {
  Error(ctx) << *file << ": symbol " << *sym <<  " has undefined version "
             << ver;
  continue;
}
```

根据具体的值查找到上面保存的index

```cpp
sym->ver_idx = it->second;
if (!is_default)
  sym->ver_idx |= VERSYM_HIDDEN;
```

非default行为的版本，也就是非@开头的版本则ver_idx设置为HIDDEN。关于非default的情况，这种符号version则是在default_symver选项中添加的。

该选项对应的介绍以及代码实现处

> Use soname as a symbol version and append that version to all symbols.

```cpp
if (ctx.arg.default_symver) {
    std::string ver = ctx.arg.soname.empty() ?
      filepath(ctx.arg.output).filename().string() : std::string(ctx.arg.soname);
    ctx.arg.version_definitions.push_back(ver);
    ctx.default_version = VER_NDX_LAST_RESERVED + 1;
  }
```

```cpp
// If both symbol `foo` and `foo@VERSION` are defined, `foo@VERSION`
// hides `foo` so that all references to `foo` are resolved to a
// versioned symbol. Likewise, if `foo@VERSION` and `foo@@VERSION` are
// defined, the default one takes precedence.
Symbol<E> *sym2 = get_symbol(ctx, sym->name());
if (sym2->file == file && !file->symvers[sym2->sym_idx - file->first_global])
  if (sym2->ver_idx == ctx.default_version ||
      (sym2->ver_idx & ~VERSYM_HIDDEN) == (sym->ver_idx & ~VERSYM_HIDDEN))
    sym2->ver_idx = VER_NDX_LOCAL;
```

最后是一个符号名同时存在带有version和没有version的两种定义，那么带有version信息的则会对外隐藏不带有特殊version信息的实现（设置为local）。通过当前sym的名字在ctx中查找同名符号，之后进行处理操作。

# compute_import_export

```cpp
// Set is_imported and is_exported bits for each symbol.
compute_import_export(ctx);
```

```cpp
template <typename E>
void compute_import_export(Context<E> &ctx) {
  Timer t(ctx, "compute_import_export");

  // If we are creating an executable, we want to export symbols referenced
  // by DSOs unless they are explicitly marked as local by a version script.
  if (!ctx.arg.shared) {
    tbb::parallel_for_each(ctx.dsos, [&](SharedFile<E> *file) {
      for (Symbol<E> *sym : file->symbols) {
        if (sym->file && !sym->file->is_dso && sym->visibility != STV_HIDDEN) {
          if (sym->ver_idx != VER_NDX_LOCAL || !ctx.default_version_from_version_script) {
            std::scoped_lock lock(sym->mu);
            sym->is_exported = true;
          }
        }
      }
    });
  }
```

在不生成shared库的情况下，针对所有的dso进行处理，在创建可执行文件的时候，导出被dso引用的且不被标记为local的符号。

```cpp
// Export symbols that are not hidden or marked as local.
// We also want to mark imported symbols as such.
tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
  for (Symbol<E> *sym : file->get_global_syms()) {
    if (!sym->file || sym->visibility == STV_HIDDEN ||
        sym->ver_idx == VER_NDX_LOCAL)
      continue;

    if (sym->file != file && sym->file->is_dso && !sym->is_absolute()) {
      std::scoped_lock lock(sym->mu);
      sym->is_imported = true;
      continue;
    }

    // If we are creating a DSO, all global symbols are exported by default.
    if (sym->file == file) {
      std::scoped_lock lock(sym->mu);
      sym->is_exported = true;

      if (ctx.arg.shared && sym->visibility != STV_PROTECTED &&
          !ctx.arg.Bsymbolic &&
          !(ctx.arg.Bsymbolic_functions && sym->get_type() == STT_FUNC))
        sym->is_imported = true;
    }
  }
});
```

objs中找到global符号，对于HIDDEN或者NDX_LOCAL的符号都跳过。

如果使用一个在dso中的符号，就需要运行时import它，因此需要设置对应符号为imported

如果创建dso，那么所有的global符号默认都要export。

# 总结

convert_common_symbols：给所有global的common符号创建一个对应的InputSection段

apply_version_script：将解析命令行参数产生的VersionPatten关联到对应的obj文件中的symbol

parse_symbol_version：读取symbol version信息，处理对应的ver_idx，以及针对不同版本符号的处理

compute_import_export：对所有符号计算对应的import和export信息
