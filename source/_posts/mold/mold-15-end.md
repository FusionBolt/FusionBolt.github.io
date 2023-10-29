---
title: mold源码阅读十五 最后的收尾工作
typora-root-url: ../../source
date: 2023-07-29 20:11:48
category: Linker
tags: mold
---

![Untitled](/images/mold-15-end/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:92983280</center> 

这一期没什么比较硬的重点知识，仅做为补全整个过程来补充，可以轻松愉快的食用。

# write dependency

```cpp
// Handle --dependency-file
if (!ctx.arg.dependency_file.empty())
  write_dependency_file(ctx);
```

将所有依赖，也就是链接过程中所有读取的文件，并且写入到文件中。可以用于确认某个文件是否被加入到链接过程中。

> --dependency-file=FILE      Write Makefile-style dependency rules to FILE

```cpp
// Write Makefile-style dependency rules to a file specified by
// --dependency-file. This is analogous to the compiler's -M flag.
template <typename E>
void write_dependency_file(Context<E> &ctx) {
  std::vector<std::string> deps;
  std::unordered_set<std::string> seen;

  for (std::unique_ptr<MappedFile<Context<E>>> &mf : ctx.mf_pool)
    if (!mf->parent)
      if (std::string path = path_clean(mf->name); seen.insert(path).second)
        deps.push_back(path);

  std::ofstream out;
  out.open(ctx.arg.dependency_file);
  if (out.fail())
    Fatal(ctx) << "--dependency-file: cannot open " << ctx.arg.dependency_file
               << ": " << errno_string();

  out << ctx.arg.output << ":";
  for (std::string &s : deps)
    out << " " << s;
  out << "\n";

  for (std::string &s : deps)
    out << "\n" << s << ":\n";
  out.close();
}
```

# clean lto object

```cpp
if (ctx.has_lto_object)
  lto_cleanup(ctx);
```

清理lto相关的文件，lto相关的操作都是类似于插件的形式执行的，以适配不同编译器产生的lto文件

```cpp
template <typename E>
void lto_cleanup(Context<E> &ctx) {
  Timer t(ctx, "lto_cleanup");

  if (cleanup_hook)
    cleanup_hook();
}
```

这个cleanup_hook也是在前面注册插件的时候要注册的

```cpp
template <typename E>
static PluginStatus register_cleanup_hook(CleanupHandler fn) {
  LOG << "register_cleanup_hook\n";
  cleanup_hook = fn;
  return LDPS_OK;
}
```

# print map

```cpp
if (ctx.arg.print_map)
    print_map(ctx);
```

> --Map FILE Write map file to a given file

收集信息并建立了section到symbol的map，之后遍历所有的chunk，进行打印。

1. 首先会打印一行chunk的信息
2. 如果不是osec那么会继续打印下一个chunk，否则之后会接着打印osec内部的所有members的信息

```cpp
template <typename E>
void print_map(Context<E> &ctx) {
  std::ostream *out = &std::cout;
  std::unique_ptr<std::ofstream> file;

  if (!ctx.arg.Map.empty()) {
    file = open_output_file(ctx);
    out = file.get();
  }

  // Construct a section-to-symbol map.
  Map<E> map = get_map(ctx);

  // Print a mapfile.
  *out << "               VMA       Size Align Out     In      Symbol\n";

  for (Chunk<E> *osec : ctx.chunks) {
    *out << std::showbase
         << std::setw(18) << std::hex << (u64)osec->shdr.sh_addr << std::dec
         << std::setw(11) << (u64)osec->shdr.sh_size
         << std::setw(6) << (u64)osec->shdr.sh_addralign
         << " " << osec->name << "\n";

    if (osec->kind() != OUTPUT_SECTION)
      continue;

    std::span<InputSection<E> *> members = ((OutputSection<E> *)osec)->members;
    std::vector<std::string> bufs(members.size());

    tbb::parallel_for((i64)0, (i64)members.size(), [&](i64 i) {
      InputSection<E> *mem = members[i];
      std::ostringstream ss;
      opt_demangle = ctx.arg.demangle;
      u64 addr = osec->shdr.sh_addr + mem->offset;

      ss << std::showbase
         << std::setw(18) << std::hex << addr << std::dec
         << std::setw(11) << (u64)mem->sh_size
         << std::setw(6) << (1 << (u64)mem->p2align)
         << "         " << *mem << "\n";

      typename Map<E>::const_accessor acc;
      if (map.find(acc, mem))
        for (Symbol<E> *sym : acc->second)
          ss << std::showbase
             << std::setw(18) << std::hex << sym->get_addr(ctx) << std::dec
             << "          0     0                 "
             << *sym << "\n";

      bufs[i] = ss.str();
    });

    for (std::string &str : bufs)
      *out << str;
  }
}
```

```cpp
template <typename E>
static Map<E> get_map(Context<E> &ctx) {
  Map<E> map;

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (Symbol<E> *sym : file->symbols) {
      if (sym->file != file || sym->get_type() == STT_SECTION)
        continue;

      if (InputSection<E> *isec = sym->get_input_section()) {
        assert(file == &isec->file);
        typename Map<E>::accessor acc;
        map.insert(acc, {isec, {}});
        acc->second.push_back(sym);
      }
    }
  });

  if (map.size() <= 1)
    return map;

  tbb::parallel_for(map.range(), [](const typename Map<E>::range_type &range) {
    for (auto it = range.begin(); it != range.end(); it++) {
      std::vector<Symbol<E> *> &vec = it->second;
      sort(vec, [](Symbol<E> *a, Symbol<E> *b) { return a->value < b->value; });
    }
  });
  return map;
}
```

# stats

```cpp
// Show stats numbers
if (ctx.arg.stats)
  show_stats(ctx);
```

在链接的过程中对于许多操作都会使用一个Counter记录数量，比如说符号的个数等，这里就是打印那些记录的信息

> --stats                     Print input statistics

```cpp
template <typename E>
void show_stats(Context<E> &ctx) {
  for (ObjectFile<E> *obj : ctx.objs) {
    static Counter defined("defined_syms");
    defined += obj->first_global - 1;

    static Counter undefined("undefined_syms");
    undefined += obj->symbols.size() - obj->first_global;

    for (std::unique_ptr<InputSection<E>> &sec : obj->sections) {
      if (!sec || !sec->is_alive)
        continue;

      static Counter alloc("reloc_alloc");
      static Counter nonalloc("reloc_nonalloc");

      if (sec->shdr().sh_flags & SHF_ALLOC)
        alloc += sec->get_rels(ctx).size();
      else
        nonalloc += sec->get_rels(ctx).size();
    }

    static Counter comdats("comdats");
    comdats += obj->comdat_groups.size();

    static Counter removed_comdats("removed_comdat_mem");
    for (ComdatGroupRef<E> &ref : obj->comdat_groups)
      if (ref.group->owner != obj->priority)
        removed_comdats += ref.members.size();

    static Counter num_cies("num_cies");
    num_cies += obj->cies.size();

    static Counter num_unique_cies("num_unique_cies");
    for (CieRecord<E> &cie : obj->cies)
      if (cie.is_leader)
        num_unique_cies++;

    static Counter num_fdes("num_fdes");
    num_fdes +=  obj->fdes.size();
  }

  static Counter num_bytes("total_input_bytes");
  for (std::unique_ptr<MappedFile<Context<E>>> &mf : ctx.mf_pool)
    num_bytes += mf->size;

  static Counter num_input_sections("input_sections");
  for (ObjectFile<E> *file : ctx.objs)
    num_input_sections += file->sections.size();

  static Counter num_output_chunks("output_chunks", ctx.chunks.size());
  static Counter num_objs("num_objs", ctx.objs.size());
  static Counter num_dsos("num_dsos", ctx.dsos.size());

  if constexpr (needs_thunk<E>) {
    static Counter thunk_bytes("thunk_bytes");
    for (Chunk<E> *chunk : ctx.chunks)
      if (OutputSection<E> *osec = chunk->to_osec())
        for (std::unique_ptr<RangeExtensionThunk<E>> &thunk : osec->thunks)
          thunk_bytes += thunk->size();
  }

  Counter::print();

  for (std::unique_ptr<MergedSection<E>> &sec : ctx.merged_sections)
    sec->print_stats(ctx);
}
```

```cpp
// Counter is used to collect statistics numbers.
class Counter {
public:
  Counter(std::string_view name, i64 value = 0) : name(name), values(value) {
    static std::mutex mu;
    std::scoped_lock lock(mu);
    instances.push_back(this);
  }

  Counter &operator++(int) {
    if (enabled)
      values.local()++;
    return *this;
  }

  Counter &operator+=(int delta) {
    if (enabled)
      values.local() += delta;
    return *this;
  }

  static void print();

  static inline bool enabled = false;

private:
  i64 get_value();

  std::string_view name;
  tbb::enumerable_thread_specific<i64> values;

  static inline std::vector<Counter *> instances;
};

void Counter::print() {
  sort(instances, [](Counter *a, Counter *b) {
    return a->get_value() > b->get_value();
  });

  for (Counter *c : instances)
    std::cout << std::setw(20) << std::right << c->name
              << "=" << c->get_value() << "\n";
}
```

```cpp
void MergedSection<E>::print_stats(Context<E> &ctx) {
  i64 used = 0;
  for (i64 i = 0; i < map.nbuckets; i++)
    if (map.keys[i])
      used++;

  SyncOut(ctx) << this->name
               << " estimation=" << estimator.get_cardinality()
               << " actual=" << used;
}
```

# perf

之前在各个过程中都会创建许多timer，在这个过程中把timer收集到的时间信息全部打印出来

```cpp
if (ctx.arg.perf)
  print_timer_records(ctx.timer_records);
```

```cpp
void print_timer_records(
    tbb::concurrent_vector<std::unique_ptr<TimerRecord>> &records) {
  for (i64 i = records.size() - 1; i >= 0; i--)
    records[i]->stop();

  for (i64 i = 0; i < records.size(); i++) {
    TimerRecord &inner = *records[i];
    if (inner.parent)
      continue;

    for (i64 j = i - 1; j >= 0; j--) {
      TimerRecord &outer = *records[j];
      if (outer.start <= inner.start && inner.end <= outer.end) {
        inner.parent = &outer;
        outer.children.push_back(&inner);
        break;
      }
    }
  }

  std::cout << "     User   System     Real  Name\n";

  for (std::unique_ptr<TimerRecord> &rec : records)
    if (!rec->parent)
      print_rec(*rec, 0);

  std::cout << std::flush;
}
```

# on_complete

```cpp
if (on_complete)
    on_complete();
```

```cpp
#if !defined(_WIN32) && !defined(__APPLE__)
  if (ctx.arg.fork)
    on_complete = fork_child();
#endif
```

因为退出一个大量内存占用的程序很慢，因此这里会fork一个子进程来进行实际的清理工作，主进程直接退出，能够提升结束的速度，让用户不可见的清理操作放到后台执行。

```cpp
#ifdef MOLD_X86_64
// Exiting from a program with large memory usage is slow --
// it may take a few hundred milliseconds. To hide the latency,
// we fork a child and let it do the actual linking work.
std::function<void()> fork_child() {
  int pipefd[2];
  if (pipe(pipefd) == -1) {
    perror("pipe");
    exit(1);
  }

  pid_t pid = fork();
  if (pid == -1) {
    perror("fork");
    exit(1);
  }

  if (pid > 0) {
    // Parent
    close(pipefd[1]);

    char buf[1];
    if (read(pipefd[0], buf, 1) == 1)
      _exit(0);

    int status;
    waitpid(pid, &status, 0);

    if (WIFEXITED(status))
      _exit(WEXITSTATUS(status));
    if (WIFSIGNALED(status))
      raise(WTERMSIG(status));
    _exit(1);
  }

  // Child
  close(pipefd[0]);

  return [=] {
    char buf[] = {1};
    [[maybe_unused]] int n = write(pipefd[1], buf, 1);
    assert(n == 1);
  };
}
#endif
```

# on_exit

```cpp
if (ctx.arg.quick_exit)
  _exit(0);

for (std::function<void()> &fn : ctx.on_exit)
  fn();

ctx.checkpoint();
```

直接exit或者调用exit的清理的函数

> --quick-exit Use quick_exit to exit (default) 
> --no-quick-exit

在mold中有的只有一处，在icf_sections中创建的map需要在这里销毁，但是也可能在lto的过程中注册了其他的exit函数。

```cpp
// Since free'ing the map is slow, postpone it.
ctx.on_exit.push_back([=] { delete map; });
```

最后在返回之前会再调用checkpoint检查是否有错误。

至此，整个mold的链接过程已经完全结束了。下一期会进行一个总结，并且记录一下一些自己的想法
