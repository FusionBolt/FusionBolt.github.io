---
title: mold源码阅读六 section size优化
typora-root-url: ../../source
date: 2023-05-07 23:59:24
category: Linker
tags: mold
---

![Untitled](/images/mold-6-section-size-reduce/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:101015341 p2</center> 

上一期我们讲解了一些符号相关的处理，这一期我们来讲一些对于section size的优化处理。

# mark_addrsig

```cpp
// Read address-significant section information.
if (ctx.arg.icf && !ctx.arg.icf_all)
  mark_addrsig(ctx);

template <typename E>
void mark_addrsig(Context<E> &ctx) {
  Timer t(ctx, "mark_addrsig");

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->mark_addrsig(ctx);
  });
}
```

相关的命令行参数

> -icf=[all,safe,none] Fold identical code 
> --no-icf

针对所有的obj处理，因为dso的地址相关信息是在运行时加载进行处理

```cpp
template <typename E>
void ObjectFile<E>::mark_addrsig(Context<E> &ctx) {
  // Parse a .llvm_addrsig section.
  if (llvm_addrsig) {
    u8 *cur = (u8 *)llvm_addrsig->contents.data();
    u8 *end = cur + llvm_addrsig->contents.size();

    while (cur != end) {
      Symbol<E> &sym = *this->symbols[read_uleb(cur)];
      if (sym.file == this)
        if (InputSection<E> *isec = sym.get_input_section())
          isec->address_significant = true;
    }
  }

  // We treat a symbol's address as significant if
  //
  // 1. we have no address significance information for the symbol, or
  // 2. the symbol can be referenced from the outside in an address-
  //    significant manner.
  for (Symbol<E> *sym : this->symbols)
    if (sym->file == this)
      if (InputSection<E> *isec = sym->get_input_section())
        if (!llvm_addrsig || sym->is_exported)
          isec->address_significant = true;
}
```

关于llvm_addrsig，处理的是一个范围的Symbol，将这个范围的Symbol的address_significant设置为True

```cpp
// in ObjectFile<E>::initialize_sections
std::unique_ptr<InputSection<E>> llvm_addrsig;
// Save .llvm_addrsig for --icf=safe.
if (shdr.sh_type == SHT_LLVM_ADDRSIG && !ctx.arg.relocatable) {
  llvm_addrsig = std::make_unique<InputSection<E>>(ctx, *this, name, i);
  continue;
}
```

普通的symbol address，针对非llvm_addrsig或者exported的symbol将address_significant为True

那么address_significant是什么呢

[https://llvm.org/docs/Extensions.html#sht-llvm-addrsig-section-address-significance-table](https://llvm.org/docs/Extensions.html#sht-llvm-addrsig-section-address-significance-table)

> the address of the symbol is used in a comparison or leaks outside the translation unit

简单来说就是这个地址会被用于比较或者用于翻译单元之外，这个变量的具体含义到后面使用的时候会结合场景进一步讲述。

# gc_sections

```cpp
// Garbage-collect unreachable sections.
if (ctx.arg.gc_sections)
  gc_sections(ctx);

template <typename E>
void gc_sections(Context<E> &ctx) {
  Timer t(ctx, "gc");

  mark_nonalloc_fragments(ctx);

  tbb::concurrent_vector<InputSection<E> *> rootset;
  collect_root_set(ctx, rootset);
  mark(ctx, rootset);
  sweep(ctx);
}
```

gc_sections主要是对section像GC一样进行mark and sweep，清理掉未被使用的段，关于gc_sections的选项

> --gc-sections               Remove unreferenced sections

## mark_nonalloc_fragments

```cpp
// Non-alloc section fragments are not subject of garbage collection.
// This function marks such fragments.
template <typename E>
static void mark_nonalloc_fragments(Context<E> &ctx) {
  Timer t(ctx, "mark_nonalloc_fragments");

  tbb::parallel_for_each(ctx.objs, [](ObjectFile<E> *file) {
    for (std::unique_ptr<MergeableSection<E>> &m : file->mergeable_sections)
      if (m)
        for (SectionFragment<E> *frag : m->fragments)
          if (!(frag->output_section.shdr.sh_flags & SHF_ALLOC))
            frag->is_alive.store(true, std::memory_order_relaxed);
  });
}
```

Non-alloc的fragment不是垃圾回收的对象，因此这里只是标记，避免后续被sweep

## collect_root_set

```cpp
template <typename E>
static void collect_root_set(Context<E> &ctx,
                             tbb::concurrent_vector<InputSection<E> *> &rootset) {
  Timer t(ctx, "collect_root_set");

  auto enqueue_section = [&](InputSection<E> *isec) {
    if (mark_section(isec))
      rootset.push_back(isec);
  };

  auto enqueue_symbol = [&](Symbol<E> *sym) {
    if (sym) {
      if (SectionFragment<E> *frag = sym->get_frag())
        frag->is_alive.store(true, std::memory_order_relaxed);
      else
        enqueue_section(sym->get_input_section());
    }
  };

  // Add sections that are not subject to garbage collection.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (std::unique_ptr<InputSection<E>> &isec : file->sections) {
      if (!isec || !isec->is_alive)
        continue;

      // -gc-sections discards only SHF_ALLOC sections. If you want to
      // reduce the amount of non-memory-mapped segments, you should
      // use `strip` command, compile without debug info or use
      // -strip-all linker option.
      u32 flags = isec->shdr().sh_flags;
      if (!(flags & SHF_ALLOC))
        isec->is_visited = true;

      if (should_keep(*isec))
        enqueue_section(isec.get());
    }
  });

  // Add sections containing exported symbols
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (Symbol<E> *sym : file->symbols)
      if (sym->file == file && sym->is_exported)
        enqueue_symbol(sym);
  });

  // Add sections referenced by root symbols.
  enqueue_symbol(get_symbol(ctx, ctx.arg.entry));

  for (std::string_view name : ctx.arg.undefined)
    enqueue_symbol(get_symbol(ctx, name));

  for (std::string_view name : ctx.arg.require_defined)
    enqueue_symbol(get_symbol(ctx, name));

  // .eh_frame consists of variable-length records called CIE and FDE
  // records, and they are a unit of inclusion or exclusion.
  // We just keep all CIEs and everything that are referenced by them.
  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (CieRecord<E> &cie : file->cies)
      for (const ElfRel<E> &rel : cie.get_rels())
        enqueue_symbol(file->symbols[rel.r_sym]);
  });
}
```

这里主要是进行收集root，以便之后进行mark and sweep。

主要收集的方向有两个

1. 对section直接添加，这里主要是针对一些不受垃圾回收影响的段。具体条件参考should_keep的实现。

2. 针对符号进行处理，如果是在fragment中则会设置其为alive，因为fragment并非扫描的root。如果是在普通的段中则将符号引用的section添加到root中。

   而符号的来源分为这么几种

   1. is_exported
   2. undefined
   3. require_defined
   4. cie中的rel符号

```cpp
template <typename E>
static bool should_keep(const InputSection<E> &isec) {
  u32 type = isec.shdr().sh_type;
  u32 flags = isec.shdr().sh_flags;
  std::string_view name = isec.name();

  return (flags & SHF_GNU_RETAIN) ||
         type == SHT_NOTE ||
         type == SHT_INIT_ARRAY ||
         type == SHT_FINI_ARRAY ||
         type == SHT_PREINIT_ARRAY ||
         (std::is_same_v<E, ARM32> && type == SHT_ARM_EXIDX) ||
         name.starts_with(".ctors") ||
         name.starts_with(".dtors") ||
         name.starts_with(".init") ||
         name.starts_with(".fini") ||
         is_c_identifier(name);
}
```

## mark

```cpp
// Mark all reachable sections
template <typename E>
static void mark(Context<E> &ctx,
                 tbb::concurrent_vector<InputSection<E> *> &rootset) {
  Timer t(ctx, "mark");

  tbb::parallel_for_each(rootset, [&](InputSection<E> *isec,
                                    tbb::feeder<InputSection<E> *> &feeder) {
    visit(ctx, isec, feeder, 0);
  });
}
```

```cpp
template <typename E>
static void visit(Context<E> &ctx, InputSection<E> *isec,
                  tbb::feeder<InputSection<E> *> &feeder, i64 depth) {
  assert(isec->is_visited);

  // If this is a text section, .eh_frame may contain records
  // describing how to handle exceptions for that function.
  // We want to keep associated .eh_frame records.
  for (FdeRecord<E> &fde : isec->get_fdes())
    for (const ElfRel<E> &rel : fde.get_rels(isec->file).subspan(1))
      if (Symbol<E> *sym = isec->file.symbols[rel.r_sym])
        if (mark_section(sym->get_input_section()))
          feeder.add(sym->get_input_section());

  for (const ElfRel<E> &rel : isec->get_rels(ctx)) {
    Symbol<E> &sym = *isec->file.symbols[rel.r_sym];

    // Symbol can refer either a section fragment or an input section.
    // Mark a fragment as alive.
    if (SectionFragment<E> *frag = sym.get_frag()) {
      frag->is_alive.store(true, std::memory_order_relaxed);
      continue;
    }

    if (!mark_section(sym.get_input_section()))
      continue;

    // Mark a section alive. For better performacne, we don't call
    // `feeder.add` too often.
    if (depth < 3)
      visit(ctx, sym.get_input_section(), feeder, depth + 1);
    else
      feeder.add(sym.get_input_section());
  }
}
```

从rootset出发

1. 针对fde record中的rel符号所在的section进行标记，并且添加到feeder中（本质是加到了rootset中，后续会继续从这些节点开始遍历）
2. 针对rel段中的符号进行遍历，如果是fragment则设置其alive，之后对sym的input section进行标记，标记成功的话则继续递归执行。

## sweep

```cpp
// Remove unreachable sections
template <typename E>
static void sweep(Context<E> &ctx) {
  Timer t(ctx, "sweep");
  static Counter counter("garbage_sections");

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    for (std::unique_ptr<InputSection<E>> &isec : file->sections) {
      if (isec && isec->is_alive && !isec->is_visited) {
        if (ctx.arg.print_gc_sections)
          SyncOut(ctx) << "removing unused section " << *isec;
        isec->kill();
        counter++;
      }
    }
  });
}
```

```cpp
template <typename E>
inline void InputSection<E>::kill() {
  if (is_alive.exchange(false))
    for (FdeRecord<E> &fde : get_fdes())
      fde.is_alive = false;
}
```

在ehframe那一期提到会清理未用到的record，而在这里实际执行了fde的清理工作。

# icf_sections

这段内容比较长，建议单独查看源码对应位置进行对照，相关实现在elf/icf.cc中

icf的全拼推测是identical code folding

```cpp
// Merge identical read-only sections.
if (ctx.arg.icf)
	icf_sections(ctx);
```

```cpp
template <typename E>
void icf_sections(Context<E> &ctx) {
  Timer t(ctx, "icf");
  if (ctx.objs.empty())
    return;

  uniquify_cies(ctx);
  merge_leaf_nodes(ctx);
	...
}
```

## uniquify_cies

```cpp
template <typename E>
static void uniquify_cies(Context<E> &ctx) {
  Timer t(ctx, "uniquify_cies");
  std::vector<CieRecord<E> *> cies;

  for (ObjectFile<E> *file : ctx.objs) {
    for (CieRecord<E> &cie : file->cies) {
      for (i64 i = 0; i < cies.size(); i++) {
        if (cie.equals(*cies[i])) {
          cie.icf_idx = i;
          goto found;
        }
      }
      cie.icf_idx = cies.size();
      cies.push_back(&cie);
    found:;
    }
  }
}
```

针对所有obj中的所有cie，如果cie和cies中的任何一个相同，也就是出现了重复，则继续查看下一个cie是否重复，没有重复则将cie加进去。

这里我不太明白，为什么不保存一个CieRecord的Set，避免了再写一个循环的麻烦？如果有读者能解答我的疑惑欢迎邮件联系我。

## merge_leaf_nodes

```cpp
// Early merge of leaf nodes, which can be processed without constructing the
// entire graph. This reduces the vertex count and improves memory efficiency.
template <typename E>
static void merge_leaf_nodes(Context<E> &ctx) {
  Timer t(ctx, "merge_leaf_nodes");

  static Counter eligible("icf_eligibles");
  static Counter non_eligible("icf_non_eligibles");
  static Counter leaf("icf_leaf_nodes");

  tbb::concurrent_unordered_map<InputSection<E> *, InputSection<E> *,
                                LeafHasher<E>, LeafEq<E>> map;

  tbb::parallel_for((i64)0, (i64)ctx.objs.size(), [&](i64 i) {
    for (std::unique_ptr<InputSection<E>> &isec : ctx.objs[i]->sections) {
      if (!isec || !isec->is_alive)
        continue;

      if (!is_eligible(ctx, *isec)) {
        non_eligible++;
        continue;
      }

      if (is_leaf(ctx, *isec)) {
        leaf++;
        isec->icf_leaf = true;
        auto [it, inserted] = map.insert({isec.get(), isec.get()});
        if (!inserted && isec->get_priority() < it->second->get_priority())
          it->second = isec.get();
      } else {
        eligible++;
        isec->icf_eligible = true;
      }
    }
  });

  tbb::parallel_for((i64)0, (i64)ctx.objs.size(), [&](i64 i) {
    for (std::unique_ptr<InputSection<E>> &isec : ctx.objs[i]->sections) {
      if (isec && isec->is_alive && isec->icf_leaf) {
        auto it = map.find(isec.get());
        assert(it != map.end());
        isec->leader = it->second;
      }
    }
  });
}
```

针对所有obj中eligible的sections来处理。

是leaf则设置leaf并且插入到map中，但是如果insert失败，且priority更高，那么就更新对应的section

非leaf的情况下只设置eligible，留到后面进行处理。

之后针对所有obj的sections，如果是icf_leaf，那么更新其leader为map中对应的值

### 关于其中出现的InputSection的字段

```cpp
// in InputSection

// For ICF
//
// `leader` is the section that this section has been merged with.
// Three kind of values are possible:
// - `leader == nullptr`: This section was not eligible for ICF.
// - `leader == this`: This section was retained.
// - `leader != this`: This section was merged with another identical section.
InputSection<E> *leader = nullptr;
u32 icf_idx = -1;
bool icf_eligible = false;
bool icf_leaf = false;
```

简单来说这个leader实际上是用于指向当前section的一个唯一实现。

如果leader存在且为自己，那么对应内容的段只访问过一次，如果不为自己的话，那么代表这不是第一次访问对应内容的段了。

用实际实现结合注释来说明leader这个字段。

1. ==nullptr：这种情况表明这个section是not eligible的，也就是说会在上面的循环被忽略掉
2. ==this：这种情况表明这个section是对应内容的段第一次出现，在后面更新leader的过程中是找到的section和自身相同。
3. ≠this：这种情况表明后面更新leader的查找过程中，找到的section其实是其对应内容在前面第一次出现的段，也就是指向了对应的leader

举个例子，假设有s1, s2, s3三个section，s1是not eligible的，s2和s3是相同的，按照s1-s3的顺序进行扫描

s1 = nullptr

s2 = s2 # leader

s3 = s2

### is_eligible

```cpp
template <typename E>
static bool is_eligible(Context<E> &ctx, InputSection<E> &isec) {
  const ElfShdr<E> &shdr = isec.shdr();
  std::string_view name = isec.name();

  bool is_alloc = (shdr.sh_flags & SHF_ALLOC);
  bool is_exec = (shdr.sh_flags & SHF_EXECINSTR) ||
                 ctx.arg.ignore_data_address_equality;
  bool is_relro = (name == ".data.rel.ro" ||
                   name.starts_with(".data.rel.ro."));
  bool is_readonly = !(shdr.sh_flags & SHF_WRITE) || is_relro;
  bool is_bss = (shdr.sh_type == SHT_NOBITS);
  bool is_empty = (shdr.sh_size == 0);
  bool is_init = (shdr.sh_type == SHT_INIT_ARRAY || name == ".init");
  bool is_fini = (shdr.sh_type == SHT_FINI_ARRAY || name == ".fini");
  bool is_enumerable = is_c_identifier(name);
  bool is_addr_taken = !ctx.arg.icf_all && isec.address_significant;

  return is_alloc && is_exec && is_readonly && !is_bss && !is_empty &&
         !is_init && !is_fini && !is_enumerable && !is_addr_taken;
}
```

如果不满足这些情况的话无法被fold，具体条件以及判断方式无需再多讲解，纯粹是对应的规则。

注意这里出现了上面说的address_significant，需要为false才能满足，也就是说需要用地址比较的情况是无法被fold的。

## gather_sections

```cpp
// Prepare for the propagation rounds.
std::vector<InputSection<E> *> sections = gather_sections(ctx);
```

```cpp
template <typename E>
static std::vector<InputSection<E> *> gather_sections(Context<E> &ctx) {
  Timer t(ctx, "gather_sections");

  // Count the number of input sections for each input file.
  std::vector<i64> num_sections(ctx.objs.size());

  tbb::parallel_for((i64)0, (i64)ctx.objs.size(), [&](i64 i) {
    for (std::unique_ptr<InputSection<E>> &isec : ctx.objs[i]->sections)
      if (isec && isec->is_alive && isec->icf_eligible)
        num_sections[i]++;
  });

  std::vector<i64> section_indices(ctx.objs.size());
  for (i64 i = 0; i < ctx.objs.size() - 1; i++)
    section_indices[i + 1] = section_indices[i] + num_sections[i];

  std::vector<InputSection<E> *> sections(
    section_indices.back() + num_sections.back());

  // Fill `sections` contents.
  tbb::parallel_for((i64)0, (i64)ctx.objs.size(), [&](i64 i) {
    i64 idx = section_indices[i];
    for (std::unique_ptr<InputSection<E>> &isec : ctx.objs[i]->sections)
      if (isec && isec->is_alive && isec->icf_eligible)
        sections[idx++] = isec.get();
  });

  tbb::parallel_for((i64)0, (i64)sections.size(), [&](i64 i) {
    sections[i]->icf_idx = i;
  });

  return sections;
}
```

这里出现了三个vector，先来理清对应的作用

1. num_sections：每个obj中icf_eligible的input sections数量
2. section_indices：由前一个section_indices和num_sections的值决定，其实是用于标记每个位置的objs的input section的起始在最终的sections中的坐标
3. sections：初始化的容量是其实是section_indices[ctx.objs.size()]的值

这样说可能比较抽象，举个例子

num_sections: 2, 3, 4, 5

section_indices: 0, 2+0, 3+2+0, 4+3+2+0

5+4+3+2+0

算出来的其实是所有obj中icf_eligible的input sections的数量

之后是fill content的部分，并行的获取每个obj中的所有icf_eligible的input section的指针

## Digest

接下来的部分都是在计算digest，具体算法有兴趣的可以去实现中自行查看细节。

什么是digest，这个链接中的一个回答说的比较明白了，我选取了关键内容放出来

[https://crypto.stackexchange.com/questions/51243/what-is-the-difference-between-a-digest-and-a-hash-function](https://crypto.stackexchange.com/questions/51243/what-is-the-difference-between-a-digest-and-a-hash-function)

> The digest is the output of the hash function.
> For example, sha256 has a digest of 256 bits, i.e. its digest has a length of 32 bytes.

```cpp
typedef std::array<uint8_t, HASH_SIZE> Digest;
...
// We allocate 3 arrays to store hashes for each vertex.
//
// Index 0 and 1 are used for tree hashes from the previous
// iteration and the current iteration. They switch roles every
// iteration. See `slot` below.
//
// Index 2 stores the initial, single-vertex hash. This is combined
// with hashes from the connected vertices to form the tree hash
// described above.
std::vector<std::vector<Digest>> digests(3);
digests[0] = compute_digests<E>(ctx, sections);
digests[1].resize(digests[0].size());
digests[2] = digests[0];

std::vector<u32> edges;
std::vector<u32> edge_indices;
gather_edges<E>(ctx, sections, edges, edge_indices);

BitVector converged(digests[0].size());
bool slot = 0;
```

```cpp
// Execute the propagation rounds until convergence is obtained.
{
  Timer t(ctx, "propagate");
  tbb::affinity_partitioner ap;

  // A cheap test that the graph hasn't converged yet.
  // The loop after this one uses a strict condition, but it's expensive
  // as it requires sorting the entire hash collection.
  //
  // For nodes that have a cycle in downstream (i.e. recursive
  // functions and functions that calls recursive functions) will always
  // change with the iterations. Nodes that doesn't (i.e. non-recursive
  // functions) will stop changing as soon as the propagation depth reaches
  // the call tree depth.
  // Here, we test whether we have reached sufficient depth for the latter,
  // which is a necessary (but not sufficient) condition for convergence.
  i64 num_changed = -1;
  for (;;) {
    i64 n = propagate<E>(digests, edges, edge_indices, slot, converged, ap);
    if (n == num_changed)
      break;
    num_changed = n;
  }
	// Run the pass until the unique number of hashes stop increasing, at which
  // point we have achieved convergence (proof omitted for brevity).
  i64 num_classes = -1;
  for (;;) {
    // count_num_classes requires sorting which is O(n log n), so do a little
    // more work beforehand to amortize that log factor.
    for (i64 i = 0; i < 10; i++)
      propagate<E>(digests, edges, edge_indices, slot, converged, ap);

    i64 n = count_num_classes<E>(digests[slot], ap);
    if (n == num_classes)
      break;
    num_classes = n;
  }
}
```

## group sections

```cpp
// Group sections by SHA digest.
{
  Timer t(ctx, "group");

  auto *map = new tbb::concurrent_unordered_map<Digest, InputSection<E> *>;
  std::span<Digest> digest = digests[slot];

  tbb::parallel_for((i64)0, (i64)sections.size(), [&](i64 i) {
    InputSection<E> *isec = sections[i];
    auto [it, inserted] = map->insert({digest[i], isec});
    if (!inserted && isec->get_priority() < it->second->get_priority())
      it->second = isec;
  });

  tbb::parallel_for((i64)0, (i64)sections.size(), [&](i64 i) {
    auto it = map->find(digest[i]);
    assert(it != map->end());
    sections[i]->leader = it->second;
  });

  // Since free'ing the map is slow, postpone it.
  ctx.on_exit.push_back([=] { delete map; });
}

if (ctx.arg.print_icf_sections)
  print_icf_sections(ctx);
```

这里我们暂时忽略digest是怎么来的细节，直接看这里使用的过程。将digest关联一个input section，这里的逻辑很像merge_leaf_nodes，只是key换成了Digest，本质更换了一种hash方式，另外不再是只针对leaf的了

## sweep sections

```cpp
// Eliminate duplicate sections.
// Symbols pointing to eliminated sections will be redirected on the fly when
// exporting to the symtab.
{
  Timer t(ctx, "sweep");
  static Counter eliminated("icf_eliminated");
  tbb::parallel_for_each(ctx.objs, [](ObjectFile<E> *file) {
    for (std::unique_ptr<InputSection<E>> &isec : file->sections) {
      if (isec && isec->is_alive && isec->is_killed_by_icf()) {
        isec->kill();
        eliminated++;
      }
    }
  });
}

template<typename E>
inline bool InputSection<E>::is_killed_by_icf() const {
  return this->leader && this->leader != this;
}
```

最后消除掉重复的section。判断重复的依据是leader不等于自身。

## icf_sections 总结

1. CieRecord去重
2. merge leaf node
3. 取出所有需要处理的section
4. 计算digest
5. 根据digest处理所有需要处理的section
6. 消除重复的section
