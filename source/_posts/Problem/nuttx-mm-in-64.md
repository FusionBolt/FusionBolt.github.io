---
title: NuttX mm模块在64位环境下的问题
typora-root-url: ../../source
date: 2022-10-02 16:03:03
category:
  - [Debug]
tags: 
  - [Memory]
  - [Baremetal]
  - [NuttX]
---

随手记录一下最近折磨了我很久的一个问题。最近在基于某一套裸机工具链做交叉编译并且在某个模拟器上执行代码，模拟器上几乎没法断点，没法用调试器，只能手工加log的方式。加上打log本身非常拖累运行速度，几乎一秒一个字符，所以这个问题来来回回拖了好几天才解决。

提供的工具链中内存分配和释放相关的代码是基于开源的nuttx做了一点点修改，不涉及代码隐私问题，因此这里也会直接贴对应的代码。nuttx是为32位设计的系统，直接拿来64位的环境自然会有不少问题。

nuttx源码

[https://github.com/projectara/nuttx/tree/master/nuttx/include/nuttx/mm](https://github.com/projectara/nuttx/tree/master/nuttx/include/nuttx/mm)

[https://github.com/projectara/nuttx/tree/master/nuttx/mm/mm_heap](https://github.com/projectara/nuttx/tree/master/nuttx/mm/mm_heap)

# 最小可复现代码与初定位

模拟器上执行代码的时候遇到vector的第三次push_back就会死循环在某个地方，写了一个vector push_back的用例来测试，依然会死循环卡住。

```cpp
void test_vector_pushback()
{
  std::vector<int> v;
  printf("p 1\n");
  v.push_back(1);
  printf("p 2\n");
  v.push_back(2);
  printf("p 3\n");
  v.push_back(3);
}
```

这是最简单的用例，自然不可能是我代码写错了。后来想到模拟器或许能dump pc，拿到pc后再去反汇编代码中看（全部都是静态链接塞进去），发现在这里死循环了

mm_mallinfo.c

```cpp
for (node = heap->mm_heapstart[region];
           node < heap->mm_heapend[region];
     node = (struct mm_allocnode_s *)((char *)node + node->size))
{
	//  printf("node=%p size=%d pre=%d (%c)\n", node,
	//         node->size, (node->preceding & ~MM_ALLOC_BIT),
	//         (node->preceding & MM_ALLOC_BIT) ? 'A' : 'F');
  if ((node->preceding & MM_ALLOC_BIT) != 0)
  {
    uordblks += node->size;
  }
  else
  {
    ordblks++;
    fordblks += node->size;
    if (node->size > mxordblk)
    {
      mxordblk = node->size;
    }
  }

```

看到这个for循环的更新和判断条件，第一反应想到的就是size在某个地方为0了，导致不断在原地打转，因此我打印了heap的start和end，以及开启了循环内的打印

```cpp
heapstart:00000000001C9B30
heapend:000000001EFFFFE8
...
node=00000000001CAF08 size=0000000000000410 pre=00000000000011D0 (A)
node=00000000001CB318 size=0000000000000010 pre=0000000000000410 (F)
node=00000000001CB328 size=00000000001C75C8 pre=0000000000000000 (F)
node=00000000003928F0 size=0000000000000000 pre=0000000000000000 (F)
```

可以看到遍历到某个node的时候size就变成了空。但我这个时候注意力全都放在了size为空这件事情上，因为这个工程同事之前接触到free出错的情况，就让同事来帮忙看，这才意识到原来0 size node之前的node的size和pre也都不对劲。

之后通过打各种log，将直接产生问题的地方定位到了free中，同时也就能在出错之前打印出原本正确的node信息。

```cpp
node=00000000001CAF08 size=0000000000000410 pre=00000000000011D0 (A)
node=00000000001CB318 size=0000000000000010 pre=0000000000000410 (A)
node=00000000001CB328 size=0000000000000010 pre=0000000000000010 (A)
node=00000000001CB338 size=000000001EE34CB0 pre=0000000000000010 (F)
```

注意这里坏掉的是1CB328，也就是倒数第二个结点

再看一下关于free的主要逻辑。源代码比较长，由于在这个例子中未进行merge，因此省略了对应的逻辑。

```cpp
void mm_free(struct mm_heap_s *heap, void *mem, void *caller)
{
  struct mm_freenode_s *node;
  struct mm_freenode_s *prev;
  struct mm_freenode_s *next;

  (void)caller;
  //mvdbg("Freeing %p\n", mem);

  /* Protect against attempts to free a NULL reference */

  if (!mem)
    {
      return;
    }
  ...

  /* Map the memory chunk into a free node */

  node = (struct mm_freenode_s *)((uint64_t)mem - SIZEOF_MM_ALLOCNODE);
  node->preceding &= ~MM_ALLOC_BIT;
  ...

  /* Add the merged node to the nodelist */

  mm_addfreechunk(heap, node);
}
```

这里很明显关键在于mm_addfreechunk。但是在看这个函数之前，我们先看一下heap和各种node是怎样的。

# heap与node

heap的成员很多，我们在这里只放出我们这里需要关注的几个。

```cpp
struct mm_heap_s
{
  struct mm_allocnode_s *mm_heapstart[CONFIG_MM_REGIONS];
  struct mm_allocnode_s *mm_heapend[CONFIG_MM_REGIONS];
  struct mm_freenode_s mm_nodelist[MM_NNODES];
};
```

之后我们先来看一下初始化全局堆的地方

```cpp
void mm_heap_initialize(void)
{
    mm_initialize(&g_mmheap, &__heap_start, (uint64_t)(&__heap_end) - (uint64_t)(&__heap_start));
}
```

```cpp
void mm_initialize(struct mm_heap_s *heap, void *heapstart,
                   size_t heapsize)
{
  int i;

  //mlldbg("Heap: start=%p size=%u\n", heapstart, heapsize);

  /* The following two lines have cause problems for some older ZiLog
   * compilers in the past (but not the more recent).  Life is easier if we
   * just the suppress them altogther for those tools.
   */

#ifndef __ZILOG__
  //CHECK_ALLOCNODE_SIZE;
  //CHECK_FREENODE_SIZE;
#endif

  /* Set up global variables */

  heap->mm_heapsize = 0;

#if CONFIG_MM_REGIONS > 1
  heap->mm_nregions = 0;
#endif

  /* Initialize the node array */

  memset(heap->mm_nodelist, 0, sizeof(struct mm_freenode_s) * MM_NNODES);
  for (i = 1; i < MM_NNODES; i++)
    {
      heap->mm_nodelist[i-1].flink = &heap->mm_nodelist[i];
      heap->mm_nodelist[i].blink   = &heap->mm_nodelist[i-1];
    }

  /* Initialize the malloc semaphore to one (to support one-at-
   * a-time access to private data sets).
   */

  mm_seminitialize(heap);

  /* Add the initial region of memory to the heap */

  mm_addregion(heap, heapstart, heapsize);
}
```

初始化nodelist，添加一个region。（目前的代码中只有一个region

```cpp
void mm_addregion(struct mm_heap_s *heap, void *heapstart,
                  size_t heapsize)
{
  struct mm_freenode_s *node;
  uintptr_t heapbase;
  uintptr_t heapend;
#if CONFIG_MM_REGIONS > 1
  int IDX = heap->mm_nregions;
#else
# define IDX 0
#endif

  /* If the MCU handles wide addresses but the memory manager is configured
   * for a small heap, then verify that the caller is  not doing something
   * crazy.
   */

#if defined(CONFIG_MM_SMALL) && !defined(CONFIG_SMALL_MEMORY)
  //DEBUGASSERT(heapsize <= MMSIZE_MAX+1);
#endif

  /* Adjust the provide heap start and size so that they are both aligned
   * with the MM_MIN_CHUNK size.
   */

  heapbase = MM_ALIGN_UP((uintptr_t)heapstart);
  heapend  = MM_ALIGN_DOWN((uintptr_t)heapstart + (uintptr_t)heapsize);
  heapsize = heapend - heapbase;

  //mlldbg("Region %d: base=%p size=%u\n", IDX+1, heapstart, heapsize);

  /* Add the size of this region to the total size of the heap */

  heap->mm_heapsize += heapsize;

  /* Create two "allocated" guard nodes at the beginning and end of
   * the heap.  These only serve to keep us from allocating outside
   * of the heap.
   *
   * And create one free node between the guard nodes that contains
   * all available memory.
   */

  heap->mm_heapstart[IDX]            = (struct mm_allocnode_s *)heapbase;
  heap->mm_heapstart[IDX]->size      = SIZEOF_MM_ALLOCNODE;
  heap->mm_heapstart[IDX]->preceding = MM_ALLOC_BIT;

  node                        = (struct mm_freenode_s *)(heapbase + SIZEOF_MM_ALLOCNODE);
  node->size                  = heapsize - 2*SIZEOF_MM_ALLOCNODE;
  node->preceding             = SIZEOF_MM_ALLOCNODE;

  heap->mm_heapend[IDX]              = (struct mm_allocnode_s *)(heapend - SIZEOF_MM_ALLOCNODE);
  heap->mm_heapend[IDX]->size        = SIZEOF_MM_ALLOCNODE;
  heap->mm_heapend[IDX]->preceding   = node->size | MM_ALLOC_BIT;

#undef IDX

#if CONFIG_MM_REGIONS > 1
  heap->mm_nregions++;
#endif

  /* Add the single, large free node to the nodelist */

  mm_addfreechunk(heap, node);
}
```

heapstart和heapend分别保存了一个指向heap开始和结尾的allocnode的地址，初始化的时候中间有一个非常大的空闲的freenode，而随着之后内存的分配，中间会有越来越多的node。

注意allocnode和freenode的异同

```cpp
struct mm_allocnode_s
{
  mmsize_t size;           /* Size of this chunk */
  mmsize_t preceding;      /* Size of the preceding chunk */
};
```

```cpp
struct mm_freenode_s
{
  mmsize_t size;                   /* Size of this chunk */
  mmsize_t preceding;              /* Size of the preceding chunk */
  struct mm_freenode_s *flink; /* Supports a doubly linked list */
  struct mm_freenode_s *blink;
};
```

显而易见，allocnode和freenode存储的时候都是以一个size和preceding开始，只是free的后面还会跟两个指针。

其中的preceding保存了前一个chunk的size，同时也标记了当前的块是被分配的状态还是被释放的状态，allocnode和freenode的处理方式都是不相同的。

我们再回到初始化的部分，可以看到start和end的size是SIZEOF_MM_ALLOCNODE，中间空闲的node size为heapsize - 2 * SIZEOF_MM_ALLOCNODE，也就是说**这个size是算入了保存内存信息的空间**。

# mm_addfreechunk

我们再回来看mm_addfreechunk。我在这个函的前后从heapstart开始出发采用size递增的方式遍历，经过addfreechunk之后就开始死循环了。

```cpp
void mm_addfreechunk(struct mm_heap_s *heap, struct mm_freenode_s *node)
{
  struct mm_freenode_s *next;
  struct mm_freenode_s *prev;

  /* Convert the size to a nodelist index */

  int ndx = mm_size2ndx(node->size);

  /* Now put the new node int the next */

  for (prev = &heap->mm_nodelist[ndx], next = heap->mm_nodelist[ndx].flink;
       next && next->size && next->size < node->size;
       prev = next, next = next->flink);

  /* Does it go in mid next or at the end? */

  prev->flink = node;
  node->blink = prev;
  node->flink = next;

  if (next)
    {
      /* The new node goes between prev and next */

      next->blink = node;
    }
}
```

这个函数逻辑也比较简单，找到对应的节点，修改flink和blink，只是看着这段逻辑很难想到为什么会引起那么奇怪的问题。

不过我一开始以错误的思路打下了一个log反而利于我想明白问题。最初理解node排布之后，我手动采用了node + size的方式访问到了这种方式访问到的最后一个node。我在mm_addfreechunk之前获取了最后一个node，并在前后打印该node的信息，发现并没有什么异常。后来晚上回家的路上突然意识到这样打印是有问题的，mm_addfreechunk会改变连接关系。但是这后来给了我一个提示，原来end node所在的地址没有被写掉。

# 内存排布与解决方案

最后我开始画了内存图，想明白了原因。

回看最早出现死循环的地方，每次循环的递增是通过node = (struct mm_allocnode_s *)((char *)node + node->size))来做的，也就是说所有的node是排布在heapstart和heapend中间

```cpp
start     328            338            free                 end
|size|prec|size|prec|data|size|prec|data|size|prec|data      |size|prec|
```

倒数第二个结点(338)坏掉，是因为倒数第三个结点(328)数据写越界了。这块空间被释放掉以后那么起始地址就会被视为一个freenode，在后面mm_addfreechunk修改对应的flink和blink的时候，由于除了size和preceding的数据大小小于了两个指针的大小，因此覆写了下一个内存块开头的部分。

那么我们实际上需要保证每次分配给数据的大小需要大于等于两个指针的大小。

mm_malloc.c

```cpp
void *mm_malloc(struct mm_heap_s *heap, size_t size, void *caller)
{
  struct mm_freenode_s *node;
  void *ret = NULL;
  int ndx;
#if defined(CONFIG_MM_DETECT_ERROR)
  size_t real_size;
#endif

  /* Handle bad sizes */

  if (size < 1)
    {
      return NULL;
    }

#if defined(CONFIG_MM_DETECT_ERROR)
  size = (size + 3) & ~3;
  real_size = size;
  size += MDBG_SZ_HEAD + MDBG_SZ_TAIL;
#endif

  /* Adjust the size to account for (1) the size of the allocated node and
   * (2) to make sure that it is an even multiple of our granule size.
   */

  size = MM_ALIGN_UP(size + SIZEOF_MM_ALLOCNODE);
```

这里最后实际alloc的size是MM_ALIGN_UP以后的大小

mm.h

```cpp
#if defined(CONFIG_MM_SMALL) && UINTPTR_MAX <= UINT32_MAX
/* Two byte offsets; Pointers may be 2 or 4 bytes;
 * sizeof(struct mm_freenode_s) is 8 or 12 bytes.
 * REVISIT: We could do better on machines with 16-bit addressing.
 */

#  define MM_MIN_SHIFT    4  /* 16 bytes */
#  define MM_MAX_SHIFT   15  /* 32 Kb */

#elif defined(CONFIG_HAVE_LONG_LONG)
/* Four byte offsets; Pointers may be 4 or 8 bytes
 * sizeof(struct mm_freenode_s) is 16 or 24 bytes.
 */

#  if UINTPTR_MAX <= UINT32_MAX
#    define MM_MIN_SHIFT  4  /* 16 bytes */
#  elif UINTPTR_MAX <= UINT64_MAX
#    define MM_MIN_SHIFT  5  /* 32 bytes */
#  endif
#  define MM_MAX_SHIFT   22  /*  4 Mb */

#else
/* Four byte offsets; Pointers must be 4 bytes.
 * sizeof(struct mm_freenode_s) is 16 bytes.
 */

#  define MM_MIN_SHIFT    4  /* 16 bytes */
#  define MM_MAX_SHIFT   22  /*  4 Mb */
#endif

/* All other definitions derive from these two */

#define MM_MIN_CHUNK     (1 << MM_MIN_SHIFT)
#define MM_MAX_CHUNK     (1 << MM_MAX_SHIFT)
#define MM_NNODES        (MM_MAX_SHIFT - MM_MIN_SHIFT + 1)

#define MM_GRAN_MASK     (MM_MIN_CHUNK-1)
#define MM_ALIGN_UP(a)   (((a) + MM_GRAN_MASK) & ~MM_GRAN_MASK)
#define MM_ALIGN_DOWN(a) ((a) & ~MM_GRAN_MASK)
```

根据这里的代码可以得知我们只需要修改对应的MM_MIN_SHIFT即可解决问题

解决问题以后发现在这段代码的正上方也有相关的注释

```cpp
/* Chunk Header Definitions *************************************************/
/* These definitions define the characteristics of allocator
 *
 * MM_MIN_SHIFT is used to define MM_MIN_CHUNK.
 * MM_MIN_CHUNK - is the smallest physical chunk that can
 *   be allocated.  It must be at least a large as
 *   sizeof(struct mm_freenode_s).  Larger values may
 *   improve performance slightly, but will waste memory
 *   due to quantization losses.
 *
 * MM_MAX_SHIFT is used to define MM_MAX_CHUNK
 * MM_MAX_CHUNK is the largest, contiguous chunk of memory
 *   that can be allocated.  It can range from 16-bytes to
 *   4Gb.  Larger values of MM_MAX_SHIFT can cause larger
 *   data structure sizes and, perhaps, minor performance
 *   losses.
 */
```

这个文件访问了很多次，但是每次都是为了访问特定的声明和定义，没有在意到其他地方的注释。不过自己潜入代码中去了解，自己去思考原因也算是一个增加经验的机会。就算提早看到了这个注释可能因为缺少很多信息也不会想到
