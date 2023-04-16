---
title: mold源码阅读 其四 mergeable section
typora-root-url: ../../source
date: 2023-04-16 15:37:08
category: Linker
tags: 
  - [mold]
  - [Section]
---

![Untitled](/images/mold-4-mergeable-section/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:83834646</center> 

上一期的内容讲完了一些针对文件的简单处理以及符号决议，这一期的主要内容是在这之后针对mergeable section的决议与合并。

# resolve_section_pieces

这个过程是将mergeable的section split到更小的pieces中，并且将每一个piece和其他来自不同文件的pieces进行合并，最典型的例子是不同object file中string段的合并。mold中称mergeable section原子单元为section pieces。

所以这里的过程分为了两部分

1. 将普通的section转换为MegeableSection
2. resolve and merge

```cpp
template <typename E>
void resolve_section_pieces(Context<E> &ctx) {
  Timer t(ctx, "resolve_section_pieces");

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->initialize_mergeable_sections(ctx);
  });

  tbb::parallel_for_each(ctx.objs, [&](ObjectFile<E> *file) {
    file->resolve_section_pieces(ctx);
  });
}
```

# initialize_mergeable_sections

mold中attach section pieces symbols

```cpp
template <typename E>
void ObjectFile<E>::initialize_mergeable_sections(Context<E> &ctx) {
  mergeable_sections.resize(sections.size());

  for (i64 i = 0; i < sections.size(); i++) {
    if (std::unique_ptr<InputSection<E>> &isec = sections[i]) {
      if (std::unique_ptr<MergeableSection<E>> m = split_section(ctx, *isec)) {
        mergeable_sections[i] = std::move(m);
        isec->is_alive = false;
      }
    }
  }
}
```

针对每一个section进行split_section，转换为一个MergeableSection，之后将原始的section设置为非alive。

## MergeableSection

首先我们来看和MergeableSection相关的数据结构，有如下三个

1. MergeableSection
2. MergedSection
3. SectionFragment

其中每个MergeableSection中包含了多个SectionFragment，又关联了其对应的MergedSection。MergedSection是一个chunk，而chunk则是在链接后期要输出到文件的时候的一个基本单位，暂时先不进一步讲解。SectionFragment则是MergedSection根据MergeableSection传入的信息构造的，并且返回给MergeableSection保存的结构。

```cpp
template <typename E>
struct MergeableSection {
  std::pair<SectionFragment<E> *, i64> get_fragment(i64 offset);

  MergedSection<E> *parent;
  u8 p2align = 0;
  std::vector<std::string_view> strings;
  std::vector<u64> hashes;
  std::vector<u32> frag_offsets;
  std::vector<SectionFragment<E> *> fragments;
};
```

```cpp
template <typename E>
class MergedSection : public Chunk<E> {
public:
  static MergedSection<E> *
  get_instance(Context<E> &ctx, std::string_view name, u64 type, u64 flags);

  SectionFragment<E> *insert(std::string_view data, u64 hash, i64 p2align);
  void assign_offsets(Context<E> &ctx);
  void copy_buf(Context<E> &ctx) override;
  void write_to(Context<E> &ctx, u8 *buf) override;
  void print_stats(Context<E> &ctx);

  HyperLogLog estimator;

private:
  MergedSection(std::string_view name, u64 flags, u32 type);

  ConcurrentMap<SectionFragment<E>> map;
  std::vector<i64> shard_offsets;
  std::once_flag once_flag;
};
```

```cpp
template <typename E>
struct SectionFragment {
  SectionFragment(MergedSection<E> *sec) : output_section(*sec) {}

  SectionFragment(const SectionFragment &other)
    : output_section(other.output_section), offset(other.offset),
      p2align(other.p2align.load()), is_alive(other.is_alive.load()) {}

  u64 get_addr(Context<E> &ctx) const;

  MergedSection<E> &output_section;
  u32 offset = -1;
  std::atomic_uint8_t p2align = 0;
  std::atomic_bool is_alive = false;
};
```

MergedSection并不暴露对应的构造函数，而是通过对应的get_instance来获取实例

```cpp
MergedSection<E>::get_instance(Context<E> &ctx, std::string_view name,
                               u64 type, u64 flags) {
  name = get_merged_output_name(ctx, name, flags);
  flags = flags & ~(u64)SHF_GROUP & ~(u64)SHF_COMPRESSED;

  auto find = [&]() -> MergedSection * {
    for (std::unique_ptr<MergedSection<E>> &osec : ctx.merged_sections)
      if (std::tuple(name, flags, type) ==
          std::tuple(osec->name, osec->shdr.sh_flags, osec->shdr.sh_type))
        return osec.get();
    return nullptr;
  };

  // Search for an exiting output section.
  static std::shared_mutex mu;
  {
    std::shared_lock lock(mu);
    if (MergedSection *osec = find())
      return osec;
  }

  // Create a new output section.
  std::unique_lock lock(mu);
  if (MergedSection *osec = find())
    return osec;

  MergedSection *osec = new MergedSection(name, flags, type);
  ctx.merged_sections.emplace_back(osec);
  return osec;
}
```

每次获取的时候去ctx中寻找实例，不存在则创建新的并且返回。

## split_section

```cpp
template <typename E>
static std::unique_ptr<MergeableSection<E>>
split_section(Context<E> &ctx, InputSection<E> &sec) {
  if (!sec.is_alive || sec.sh_size == 0 || sec.relsec_idx != -1)
    return nullptr;

  const ElfShdr<E> &shdr = sec.shdr();
  if (!(shdr.sh_flags & SHF_MERGE))
    return nullptr;
```

由于是针对mergeable section，而判断标准则是根据section header中的sh_flgas的值，因此先通过检查flga来进行过滤。

```cpp
std::unique_ptr<MergeableSection<E>> rec(new MergeableSection<E>);
rec->parent = MergedSection<E>::get_instance(ctx, sec.name(), shdr.sh_type,
                                             shdr.sh_flags);
rec->p2align = sec.p2align;

// If thes section contents are compressed, uncompress them.
sec.uncompress(ctx);

std::string_view data = sec.contents;
const char *begin = data.data();
u64 entsize = shdr.sh_entsize;
HyperLogLog estimator;
```

做一些基本的初始化操作，包括创建了MergeableSection以及关联对应的MergedSection，取出数据等。

### split string

```cpp
// Split sections
if (shdr.sh_flags & SHF_STRINGS) {
  if (entsize == 0) {
    // GHC (Glasgow Haskell Compiler) sometimes creates a mergeable
    // string section with entsize of 0 instead of 1, though such
    // entsize is technically wrong. This is a workaround for the issue.
    entsize = 1;
  }

  while (!data.empty()) {
    size_t end = find_null(data, entsize);
    if (end == data.npos)
      Fatal(ctx) << sec << ": string is not null terminated";

    std::string_view substr = data.substr(0, end + entsize);
    data = data.substr(end + entsize);

    rec->strings.push_back(substr);
    rec->frag_offsets.push_back(substr.data() - begin);

    u64 hash = hash_string(substr);
    rec->hashes.push_back(hash);
    estimator.insert(hash);
  }
}

static size_t find_null(std::string_view data, u64 entsize) {
  if (entsize == 1)
    return data.find('\0');

  for (i64 i = 0; i <= data.size() - entsize; i += entsize)
    if (data.substr(i, entsize).find_first_not_of('\0') == data.npos)
      return i;

  return data.npos;
}
```

1. 找到terminator（’\0’）
2. 将对应的rec的strings添加找到的str
3. 添加对应的frag_offsets
4. 添加string的hash到estimator中

estimator是用于优化时间的方案，等到最后会提及，不影响合并的正确性。

### split other

```cpp
else {
    // OCaml compiler seems to create a mergeable non-string section with
    // entisze of 0. Such section is malformed. We do not split such section.
    if (entsize == 0)
      return nullptr;

    if (data.size() % entsize)
      Fatal(ctx) << sec << ": section size is not multiple of sh_entsize";

    while (!data.empty()) {
      std::string_view substr = data.substr(0, entsize);
      data = data.substr(entsize);

      rec->strings.push_back(substr);
      rec->frag_offsets.push_back(substr.data() - begin);

      u64 hash = hash_string(substr);
      rec->hashes.push_back(hash);
      estimator.insert(hash);
    }
  }
```

和split string的区别在于不是通过’\0’而是通过entsize判断一个piece的结束位置

```cpp
rec->parent->estimator.merge(estimator);
static Counter counter("string_fragments");
counter += rec->fragments.size();
return rec;
```

最后的收尾

# ObjectFile::resolve_section_pieces

- [ ] 如何判断是相同的字符串？？对应地址怎么办

```cpp
template <typename E>
void ObjectFile<E>::resolve_section_pieces(Context<E> &ctx) {
	for (std::unique_ptr<MergeableSection<E>> &m : mergeable_sections) {
    if (m) {
      m->fragments.reserve(m->strings.size());
      for (i64 i = 0; i < m->strings.size(); i++)
        m->fragments.push_back(m->parent->insert(m->strings[i], m->hashes[i],
                                                 m->p2align));

      // Shrink vectors that we will never use again to reclaim memory.
      m->strings.clear();
      m->hashes.clear();
    }
  }
```

将所有MergableSection的数据merge到对应的parent中。

```cpp
  for (i64 i = 1; i < this->elf_syms.size(); i++) {
    Symbol<E> &sym = *this->symbols[i];
    const ElfSym<E> &esym = this->elf_syms[i];

    if (esym.is_abs() || esym.is_common() || esym.is_undef())
      continue;

    std::unique_ptr<MergeableSection<E>> &m = mergeable_sections[get_shndx(esym)];
    if (!m)
      continue;

    SectionFragment<E> *frag;
    i64 frag_offset;
    std::tie(frag, frag_offset) = m->get_fragment(esym.st_value);

    if (!frag)
      Fatal(ctx) << *this << ": bad symbol value: " << esym.st_value;

    sym.set_frag(frag);
    sym.value = frag_offset;
  }
```

之后是attach section piece to symbols的过程。本质的操作是将对应的有定义的且非abs的符号关联到对应的fragment。

```cpp
// Compute the size of frag_syms.
  i64 nfrag_syms = 0;
  for (std::unique_ptr<InputSection<E>> &isec : sections)
    if (isec && isec->is_alive && (isec->shdr().sh_flags & SHF_ALLOC))
      for (ElfRel<E> &r : isec->get_rels(ctx))
        if (const ElfSym<E> &esym = this->elf_syms[r.r_sym];
            esym.st_type == STT_SECTION && mergeable_sections[get_shndx(esym)])
          nfrag_syms++;

  this->frag_syms.resize(nfrag_syms);
```

之后寻找满足条件的esym，统计对应的size。

注意寻找的是ElfRel中的esym，只有ElfRel中的esym才能被relocation，因为merge的过程中必然会修改各种地址信息。

这里根据sym得到的index获取对应的mergeable_section是在前一步init的过程中初始化的，也就是说这个index对于mergeable_section和原始的section是完全对应的，如果不是mergeable的section则返回的会是空指针。

接下来是引用mergeable section的relocation symbol，会针对每一个这样的symbol redirect rel sym到一个新创建的dummy到symbol上。

```cpp
// For each relocation referring a mergeable section symbol, we create
// a new dummy non-section symbol and redirect the relocation to the
// newly-created symbol.
i64 idx = 0;
for (std::unique_ptr<InputSection<E>> &isec : sections) {
  if (!isec || !isec->is_alive || !(isec->shdr().sh_flags & SHF_ALLOC))
    continue;

  for (ElfRel<E> &r : isec->get_rels(ctx)) {
    const ElfSym<E> &esym = this->elf_syms[r.r_sym];
    if (esym.st_type != STT_SECTION)
      continue;

    std::unique_ptr<MergeableSection<E>> &m = mergeable_sections[get_shndx(esym)];
    if (!m)
      continue;

    i64 r_addend = get_addend(*isec, r);

    SectionFragment<E> *frag;
    i64 in_frag_offset;
    std::tie(frag, in_frag_offset) = m->get_fragment(esym.st_value + r_addend);

    if (!frag)
      Fatal(ctx) << *this << ": bad relocation at " << r.r_sym;

    Symbol<E> &sym = this->frag_syms[idx];
    sym.file = this;
    sym.set_name("<fragment>");
    sym.sym_idx = r.r_sym;
    sym.visibility = STV_HIDDEN;
    sym.set_frag(frag);
    sym.value = in_frag_offset - r_addend;
    r.r_sym = this->elf_syms.size() + idx;
    idx++;
  }
}

template <typename E>
inline i64 get_addend(u8 *loc, const ElfRel<E> &rel) {
  return rel.r_addend;
}

template <typename E>
std::pair<SectionFragment<E> *, i64>
MergeableSection<E>::get_fragment(i64 offset) {
  std::vector<u32> &vec = frag_offsets;
  auto it = std::upper_bound(vec.begin(), vec.end(), offset);
  i64 idx = it - 1 - vec.begin();
  return {fragments[idx], offset - vec[idx]};
}
```

简单的对新的sym设置了基本信息，主要是进行双向的关联。

```cpp
sym.sym_idx = r.r_sym;
r.r_sym = this->elf_syms.size() + idx;

// Symbol:sym_idx
// Index into the symbol table of the owner file.
i32 sym_idx = -1;
```

这里将rel中的sym指向了elf_syms后面的位置，后面会将执行frag_syms逐一添加到elf_syms之后。

最后将frag_syms都添加到ObjectFile的symbols中，整个过程就全部结束了。

```cpp
assert(idx == this->frag_syms.size());

for (Symbol<E> &sym : this->frag_syms)
  this->symbols.push_back(&sym);
```

## MergedSection::insert

```cpp
template <typename E>
SectionFragment<E> *
MergedSection<E>::insert(std::string_view data, u64 hash, i64 p2align) {
  std::call_once(once_flag, [&] {
    // We aim 2/3 occupation ratio
    map.resize(estimator.get_cardinality() * 3 / 2);
  });

  SectionFragment<E> *frag;
  bool inserted;
  std::tie(frag, inserted) = map.insert(data, hash, SectionFragment(this));
  assert(frag);

  update_maximum(frag->p2align, p2align);
  return frag;
}

ConcurrentMap<SectionFragment<E>> map;
```

第一次insert的时候进行预估大小，之后进行insert。

在看到这里的实现我在想，在merge string的时候是要比较长度吗，在这里我得到了答案，是直接通过之前保存的hash保证unique。

另外这里用到了estimator，estimator的类型是hyperloglog，根据注释

> This file implements HyperLogLog algorithm, which estimates the number of unique items in a given multiset.

谷歌的结果是这样的

> HyperLogLog is **an algorithm for the count-distinct problem, approximating the number of distinct elements in a multiset**
> . Calculating the exact cardinality of the distinct elements of a multiset requires an amount of memory proportional to the cardinality, which is impractical for very large data sets.

有兴趣的可以去看wiki或者更多资料，这不在此系列博客的研究范围内。

# 整个过程的回顾

resolve_section_pieces由两部分操作组成

1. 针对所有mergeable的段进行split，将InputSection转换为对应的MergeableSection
2. 针对所有MergeableSection进行merge
   1. strings Merge到相关联的MergedSection中
   2. symbols attach to piece section
   3. 针对rel的symbol关联到一个新创建的dummy的symbol上
