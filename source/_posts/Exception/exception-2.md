---
title: LLVM异常实现二 libcxxabi
typora-root-url: ../../source
date: 2024-10-02 11:28:05
category: Compiler
tags: Exception
---

在之前的博客提到带有异常相关的C++代码编译成llvm ir后，会插入libcxxabi的__cxa_xxx函数，这期则是来了解这些函数的实现。

# 接口

libcxxabi中的部分包含了exception，array的分配与释放，virtual相关，demangler等，目前我们在这里只关心异常相关的部分。

首先来看异常的接口，基本上都是之前内容中出现的函数声明，其中包含了分配释放等常见的异常操作。

libcxxabi/include/cxxabi.h

```cpp
// 2.4.2 Allocating the Exception Object
extern _LIBCXXABI_FUNC_VIS void *
__cxa_allocate_exception(size_t thrown_size) throw();
extern _LIBCXXABI_FUNC_VIS void
__cxa_free_exception(void *thrown_exception) throw();

// 2.4.3 Throwing the Exception Object
extern _LIBCXXABI_FUNC_VIS _LIBCXXABI_NORETURN void
__cxa_throw(void *thrown_exception, std::type_info *tinfo,
            void (*dest)(void *));

// 2.5.3 Exception Handlers
extern _LIBCXXABI_FUNC_VIS void *
__cxa_get_exception_ptr(void *exceptionObject) throw();
extern _LIBCXXABI_FUNC_VIS void *
__cxa_begin_catch(void *exceptionObject) throw();
extern _LIBCXXABI_FUNC_VIS void __cxa_end_catch();
#if defined(_LIBCXXABI_ARM_EHABI)
extern _LIBCXXABI_FUNC_VIS bool
__cxa_begin_cleanup(void *exceptionObject) throw();
extern _LIBCXXABI_FUNC_VIS void __cxa_end_cleanup();
#endif
extern _LIBCXXABI_FUNC_VIS std::type_info *__cxa_current_exception_type();

// 2.5.4 Rethrowing Exceptions
extern _LIBCXXABI_FUNC_VIS _LIBCXXABI_NORETURN void __cxa_rethrow();

// 2.6 Auxiliary Runtime APIs
extern _LIBCXXABI_FUNC_VIS _LIBCXXABI_NORETURN void __cxa_bad_cast(void);
extern _LIBCXXABI_FUNC_VIS _LIBCXXABI_NORETURN void __cxa_bad_typeid(void);
extern _LIBCXXABI_FUNC_VIS _LIBCXXABI_NORETURN void
__cxa_throw_bad_array_new_length(void);
```

# __cxa_allocate_exception

```cpp
//  Allocate a __cxa_exception object, and zero-fill it.
//  Reserve "thrown_size" bytes on the end for the user's exception
//  object. Zero-fill the object. If memory can't be allocated, call
//  std::terminate. Return a pointer to the memory to be used for the
//  user's exception object.
void *__cxa_allocate_exception(size_t thrown_size) throw() {
    size_t actual_size = cxa_exception_size_from_exception_thrown_size(thrown_size);

    // Allocate extra space before the __cxa_exception header to ensure the
    // start of the thrown object is sufficiently aligned.
    size_t header_offset = get_cxa_exception_offset();
    char *raw_buffer =
        (char *)__aligned_malloc_with_fallback(header_offset + actual_size);
    if (NULL == raw_buffer)
        std::terminate();

    __cxa_exception *exception_header =
        static_cast<__cxa_exception *>((void *)(raw_buffer + header_offset));
    ::memset(exception_header, 0, actual_size);
    return thrown_object_from_cxa_exception(exception_header);
}
```

这个函数主要分配了一块空间用于作为异常处理的对象，这块空间由两部分组成：throw对象的size以及__cxa_exception对象。最后返回的是throw对象开始位置的指针。

```cpp
| __cxa_exception |     size    |
								  ^
								  |
								 指针
```

接下来我们看一下更细致的代码实现。

## actual size

首先是计算实际分配的size，其中包含了前面的__cxa_exception以及throw对象的size。

```cpp
// Round s up to next multiple of a.
static inline
size_t aligned_allocation_size(size_t s, size_t a) {
    return (s + a - 1) & ~(a - 1);
}

static inline
size_t cxa_exception_size_from_exception_thrown_size(size_t size) {
    return aligned_allocation_size(size + sizeof (__cxa_exception),
                                   alignof(__cxa_exception));
}
```

## exception offset

另外需要计算header的起始地址，要确保对象的空间能满足align的计算，如果对象的align小于目标机器的align那么会在最前面填充。

```cpp
// Return the offset of the __cxa_exception header from the start of the
// allocated buffer. If __cxa_exception's alignment is smaller than the maximum
// useful alignment for the target machine, padding has to be inserted before
// the header to ensure the thrown object that follows the header is
// sufficiently aligned. This happens if _Unwind_exception isn't double-word
// aligned (on Darwin, for example).
static size_t get_cxa_exception_offset() {
  struct S {
  } __attribute__((aligned));

  // Compute the maximum alignment for the target machine.
  constexpr size_t alignment = alignof(S);
  constexpr size_t excp_size = sizeof(__cxa_exception);
  constexpr size_t aligned_size =
      (excp_size + alignment - 1) / alignment * alignment;
  constexpr size_t offset = aligned_size - excp_size;
  static_assert((offset == 0 || alignof(_Unwind_Exception) < alignment),
                "offset is non-zero only if _Unwind_Exception isn't aligned");
  return offset;
}
```

首先使用一个空结构体，按照struct的最小size进行align。如果__exa_exception的align小于目标机器的最大可用align，那么填充，返回offset，这个offset也就是实际上header开始的位置。

## malloc

malloc的size是包含了header_offset以及实际分配的整个size。

malloc的过程

```cpp
void* __aligned_malloc_with_fallback(size_t size) {
#if defined(_WIN32)
  if (void* dest = std::__libcpp_aligned_alloc(alignof(__aligned_type), size))
    return dest;
#elif defined(_LIBCPP_HAS_NO_LIBRARY_ALIGNED_ALLOCATION)
  if (void* dest = ::malloc(size))
    return dest;
#else
  if (size == 0)
    size = 1;
  if (void* dest = std::__libcpp_aligned_alloc(__alignof(__aligned_type), size))
    return dest;
#endif
  return fallback_malloc(size);
}
```

这里使用了标准库的align_alloc

关于fallback则是标准库分配失败的情况下的备选方法

```cpp
void* fallback_malloc(size_t len) {
  heap_node *p, *prev;
  const size_t nelems = alloc_size(len);
  mutexor mtx(&heap_mutex);

  if (NULL == freelist)
    init_heap();

  //  Walk the free list, looking for a "big enough" chunk
  for (p = freelist, prev = 0; p && p != list_end;
       prev = p, p = node_from_offset(p->next_node)) {

    if (p->len > nelems) { //  chunk is larger, shorten, and return the tail
      heap_node* q;

      p->len = static_cast<heap_size>(p->len - nelems);
      q = p + p->len;
      q->next_node = 0;
      q->len = static_cast<heap_size>(nelems);
      return (void*)(q + 1);
    }

    if (p->len == nelems) { // exact size match
      if (prev == 0)
        freelist = node_from_offset(p->next_node);
      else
        prev->next_node = p->next_node;
      p->next_node = 0;
      return (void*)(p + 1);
    }
  }
  return NULL; // couldn't find a spot big enough
}
```

## 返回异常对象指针

malloc后创建了一个__cxa_exception的指针，跳过header_offset指向了actual_size，并且将actual_size的部分置0

```cpp
				exception_header
								|
| header_offset | actual_size |
```

将actual size拆开的话实际的内存视图如下

```cpp
				exception_header
								|
| header_offset | __cxa_exception | thrown_size |
```

之后将exception_header这个指针递增，以便指向thrown_size，并且转换为void*返回

```cpp
// Note:  This is never called when exception_header is masquerading as a
//        __cxa_dependent_exception.
static
inline
void*
thrown_object_from_cxa_exception(__cxa_exception* exception_header)
{
    return static_cast<void*>(exception_header + 1);
}
```

下图中throw_object是最后实际返回的地址。

```cpp
					                   throw_object
					                  			|
| header_offset | __cxa_exception | thrown_size |
```

```cpp
------------------   <-- raw_buffer
	header offset
------------------

  __cxa_exception
  
------------------   <-- throw_object

   thrown_size

------------------
```

# __cxa_free

```cpp
//  Free a __cxa_exception object allocated with __cxa_allocate_exception.
void __cxa_free_exception(void *thrown_object) throw() {
    // Compute the size of the padding before the header.
    size_t header_offset = get_cxa_exception_offset();
    char *raw_buffer =
        ((char *)cxa_exception_from_thrown_object(thrown_object)) - header_offset;
    __aligned_free_with_fallback((void *)raw_buffer);
}
```

按照上面的内存排布，从object反向找到对应的raw_buffer然后再释放。需要将thrown_object的指针回退一个cxa_exception的位置，再回退header_offset，最终就能找到前面allocate的起始位置。

```cpp
static
inline
__cxa_exception*
cxa_exception_from_thrown_object(void* thrown_object)
{
    return static_cast<__cxa_exception*>(thrown_object) - 1;
}
```

```cpp
void __aligned_free_with_fallback(void* ptr) {
  if (is_fallback_ptr(ptr))
    fallback_free(ptr);
  else {
#if defined(_LIBCPP_HAS_NO_LIBRARY_ALIGNED_ALLOCATION)
    ::free(ptr);
#else
    std::__libcpp_aligned_free(ptr);
#endif
  }
}
```

# __cxa_throw

这里开始跳转到libunwind

libcxxabi/src/cxa_exception.cpp

初始化通过excepting object获取到的exception header

1. 通过exception object获取header

2. __cxa_exception设置基本信息

   1. 保存当前的unexpected_handler和terminate_handler
   2. 保存tinfo和dest argument

3. 在unwind header设置exception_class，64bit

4. 递增uncaught_exception flag

5. 调用_Unwind_RaiseException，参数是指向thrown exception的指针。

   这个函数开始执行unwinding

```cpp
// 2.4.3 Throwing the Exception Object
/*
After constructing the exception object with the throw argument value,
the generated code calls the __cxa_throw runtime library routine. This
routine never returns.

The __cxa_throw routine will do the following:

* Obtain the __cxa_exception header from the thrown exception object address,
which can be computed as follows:
 __cxa_exception *header = ((__cxa_exception *) thrown_exception - 1);
* Save the current unexpected_handler and terminate_handler in the __cxa_exception header.
* Save the tinfo and dest arguments in the __cxa_exception header.
* Set the exception_class field in the unwind header. This is a 64-bit value
representing the ASCII string "XXXXC++\0", where "XXXX" is a
vendor-dependent string. That is, for implementations conforming to this
ABI, the low-order 4 bytes of this 64-bit value will be "C++\0".
* Increment the uncaught_exception flag.
* Call _Unwind_RaiseException in the system unwind library, Its argument is the
pointer to the thrown exception, which __cxa_throw itself received as an argument.
__Unwind_RaiseException begins the process of stack unwinding, described
in Section 2.5. In special cases, such as an inability to find a
handler, _Unwind_RaiseException may return. In that case, __cxa_throw
will call terminate, assuming that there was no handler for the
exception.
*/
void
__cxa_throw(void *thrown_object, std::type_info *tinfo, void (*dest)(void *)) {
    __cxa_eh_globals *globals = __cxa_get_globals();
    __cxa_exception* exception_header = cxa_exception_from_thrown_object(thrown_object);

    exception_header->unexpectedHandler = std::get_unexpected();
    exception_header->terminateHandler  = std::get_terminate();
    exception_header->exceptionType = tinfo;
    exception_header->exceptionDestructor = dest;
    setOurExceptionClass(&exception_header->unwindHeader);
    exception_header->referenceCount = 1;  // This is a newly allocated exception, no need for thread safety.
    globals->uncaughtExceptions += 1;   // Not atomically, since globals are thread-local

    exception_header->unwindHeader.exception_cleanup = exception_cleanup_func;

#if __has_feature(address_sanitizer)
    // Inform the ASan runtime that now might be a good time to clean stuff up.
    __asan_handle_no_return();
#endif

#ifdef __USING_SJLJ_EXCEPTIONS__
    _Unwind_SjLj_RaiseException(&exception_header->unwindHeader);
#else
    _Unwind_RaiseException(&exception_header->unwindHeader);
#endif
    //  This only happens when there is no handler, or some unexpected unwinding
    //     error happens.
    failed_throw(exception_header);
}
```

首先是获取eh_globals和获取exception header的过程。获取header的时候由于header排布在thrown_object之前因此直接回退一个对象的位置即可，和allocate的时候返回的过程是相反的。

```cpp
//  Utility routines
static
inline
__cxa_exception*
cxa_exception_from_thrown_object(void* thrown_object)
{
    return static_cast<__cxa_exception*>(thrown_object) - 1;
}
```

# __cxa_eh_globals

cxa_exception.h

```cpp
struct _LIBCXXABI_HIDDEN __cxa_eh_globals {
    __cxa_exception *   caughtExceptions;
    unsigned int        uncaughtExceptions;
#if defined(_LIBCXXABI_ARM_EHABI)
    __cxa_exception* propagatingExceptions;
#endif
};

extern "C" _LIBCXXABI_FUNC_VIS __cxa_eh_globals * __cxa_get_globals      ();
extern "C" _LIBCXXABI_FUNC_VIS __cxa_eh_globals * __cxa_get_globals_fast ();
```

由于__cxa_exception里面保存了next，因此这里相当于保存了一个链表头

exa_exception_storage.cpp

```cpp
#if defined(_LIBCXXABI_HAS_NO_THREADS)

namespace __cxxabiv1 {
extern "C" {
    static __cxa_eh_globals eh_globals;
    __cxa_eh_globals *__cxa_get_globals() { return &eh_globals; }
    __cxa_eh_globals *__cxa_get_globals_fast() { return &eh_globals; }
} // extern "C"
} // namespace __cxxabiv1

#elif defined(HAS_THREAD_LOCAL)

namespace __cxxabiv1 {
namespace {
    __cxa_eh_globals *__globals() {
        static thread_local __cxa_eh_globals eh_globals;
        return &eh_globals;
    }
} // namespace

extern "C" {
    __cxa_eh_globals *__cxa_get_globals() { return __globals(); }
    __cxa_eh_globals *__cxa_get_globals_fast() { return __globals(); }
} // extern "C"
} // namespace __cxxabiv1
```

这里有一个thread_local的判断，不过本质是都是在这个cpp文件存了一个static的eh_global (exception handing global)

# 多个异常的管理

__cxa_eh_globals里面保存了栈顶的exception，以及未处理的exception数量。

每次有新的exception时，当前exception的header中的nextException会指向当前的caughtException，然后更新__cxa_eh_globals里面的exception header为这个新的exception的header。

```cpp
eh_globals
header = nullptr
```

产生了新的exception1

```cpp
eh_globals
header = nullptr
exception header1
next = nullptr
```

更新后

```cpp
eh_globals
header = exception header1
exception header1
next = nullptr（原始的eh_globals的nullptr）
```

产生了新的exception2

```cpp
eh_globals
header = exception header1
exception header1
next = nullptr（原始的eh_globals的nullptr）
exception header2
next = nullptr
```

更新后

```cpp
eh_globals
header = exception header2
exception header1
next = nullptr（原始的eh_globals的nullptr）
exception header2
next = exception header1（设置global之前global中的header）
```

异常处理过程优先处理最新的exception，处理完后会处理之前一个旧的exception，按照处理顺序来说也就是下一个。由于并不需要从旧向新的方向进行exception的查询，因此这里只需要支持单向即可。这里是添加，对应的删除也是类似。

简单总结下，begin_catch的时候减少计数并且把cxa_exception放到栈上，而在end_catch的时候将对象从globals中取出，而在catch以及throw的过程中通过增减uncaughtExceptions来管理当前未处理异常对象的数量。

![Untitled](/images/exception-2/Untitled.png)

# __cxa_catch

主要做的事情是globals以及exception_header的更新

## __cxa_begin_catch

这里才真正开始把exception放到__cxa_eh_globals里。

有两类exception

1. native：primary或者dependent。处理的时候并不关心具体是哪种。主要做了如下几件事情

   1. 增加handler count
   2. push到栈上
   3. 减少uncaught_exception count
   4. 返回adjusted pointer to the exception object

2. foreign：不包含__cxa_exception_header的异常。

   1. 不能增加handler count

   2. 只有stack为空才能push到stack，因为没办法连接到当前栈上

      栈不为空则terminate

   3. 不增加uncaught_exception

```cpp
void*
__cxa_begin_catch(void* unwind_arg) throw()
{
    _Unwind_Exception* unwind_exception = static_cast<_Unwind_Exception*>(unwind_arg);
    bool native_exception = __isOurExceptionClass(unwind_exception);
    __cxa_eh_globals* globals = __cxa_get_globals();
    // exception_header is a hackish offset from a foreign exception, but it
    //   works as long as we're careful not to try to access any __cxa_exception
    //   parts.
    __cxa_exception* exception_header =
            cxa_exception_from_exception_unwind_exception
            (
                static_cast<_Unwind_Exception*>(unwind_exception)
            );

#if defined(__MVS__)
    // Remove the exception object from the linked list of exceptions that the z/OS unwinder
    // maintains before adding it to the libc++abi list of caught exceptions.
    // The libc++abi will manage the lifetime of the exception from this point forward.
    _UnwindZOS_PopException();
#endif

    if (native_exception)
    {
        // Increment the handler count, removing the flag about being rethrown
        exception_header->handlerCount = exception_header->handlerCount < 0 ?
            -exception_header->handlerCount + 1 : exception_header->handlerCount + 1;
        //  place the exception on the top of the stack if it's not already
        //    there by a previous rethrow
        if (exception_header != globals->caughtExceptions)
        {
            exception_header->nextException = globals->caughtExceptions;
            globals->caughtExceptions = exception_header;
        }
        globals->uncaughtExceptions -= 1;   // Not atomically, since globals are thread-local
#if defined(_LIBCXXABI_ARM_EHABI)
        return reinterpret_cast<void*>(exception_header->unwindHeader.barrier_cache.bitpattern[0]);
#else
        return exception_header->adjustedPtr;
#endif
    }
    // Else this is a foreign exception
    // If the caughtExceptions stack is not empty, terminate
    if (globals->caughtExceptions != 0)
        std::terminate();
    // Push the foreign exception on to the stack
    globals->caughtExceptions = exception_header;
    return unwind_exception + 1;
}

```

```cpp
bool __isOurExceptionClass(const _Unwind_Exception* unwind_exception) {
    return (__getExceptionClass(unwind_exception) & get_vendor_and_language) ==
           (kOurExceptionClass                    & get_vendor_and_language);
}

//  Is it one of ours?
uint64_t __getExceptionClass(const _Unwind_Exception* unwind_exception) {
    // On x86 and some ARM unwinders, unwind_exception->exception_class is
    // a uint64_t. On other ARM unwinders, it is a char[8].
    // See: http://infocenter.arm.com/help/topic/com.arm.doc.ihi0038b/IHI0038B_ehabi.pdf
    // So we just copy it into a uint64_t to be sure.
    uint64_t exClass;
    ::memcpy(&exClass, &unwind_exception->exception_class, sizeof(exClass));
    return exClass;
}

static const uint64_t kOurExceptionClass          = 0x434C4E47432B2B00; // CLNGC++\0
static const uint64_t kOurDependentExceptionClass = 0x434C4E47432B2B01; // CLNGC++\1
static const uint64_t get_vendor_and_language     = 0xFFFFFFFFFFFFFF00; // mask for CLNGC++
```

## get ant set exception class

## __cxa_end_catch

1. 获取基本信息
2. native exception
   1. handlerCount小于0的情况
      1. 递增count为0，那么将当前处理的exception移除
   2. 大于0
      1. 递减count如果为0，那么将当前处理的exception移除
      2. isDependentException，那么释放
      3. 减少refcount
3. foreign exception
   1. 删除对应的exception

```cpp
void __cxa_end_catch() {
  static_assert(sizeof(__cxa_exception) == sizeof(__cxa_dependent_exception),
                "sizeof(__cxa_exception) must be equal to "
                "sizeof(__cxa_dependent_exception)");
  static_assert(__builtin_offsetof(__cxa_exception, referenceCount) ==
                    __builtin_offsetof(__cxa_dependent_exception,
                                       primaryException),
                "the layout of __cxa_exception must match the layout of "
                "__cxa_dependent_exception");
  static_assert(__builtin_offsetof(__cxa_exception, handlerCount) ==
                    __builtin_offsetof(__cxa_dependent_exception, handlerCount),
                "the layout of __cxa_exception must match the layout of "
                "__cxa_dependent_exception");
    __cxa_eh_globals* globals = __cxa_get_globals_fast(); // __cxa_get_globals called in __cxa_begin_catch
    __cxa_exception* exception_header = globals->caughtExceptions;
    // If we've rethrown a foreign exception, then globals->caughtExceptions
    //    will have been made an empty stack by __cxa_rethrow() and there is
    //    nothing more to be done.  Do nothing!
    if (NULL != exception_header)
    {
        bool native_exception = __isOurExceptionClass(&exception_header->unwindHeader);
        if (native_exception)
        {
            // This is a native exception
            if (exception_header->handlerCount < 0)
            {
                //  The exception has been rethrown by __cxa_rethrow, so don't delete it
                if (0 == incrementHandlerCount(exception_header))
                {
                    //  Remove from the chain of uncaught exceptions
                    globals->caughtExceptions = exception_header->nextException;
                    // but don't destroy
                }
                // Keep handlerCount negative in case there are nested catch's
                //   that need to be told that this exception is rethrown.  Don't
                //   erase this rethrow flag until the exception is recaught.
            }
            else
            {
                // The native exception has not been rethrown
                if (0 == decrementHandlerCount(exception_header))
                {
                    //  Remove from the chain of uncaught exceptions
                    globals->caughtExceptions = exception_header->nextException;
                    // Destroy this exception, being careful to distinguish
                    //    between dependent and primary exceptions
                    if (isDependentException(&exception_header->unwindHeader))
                    {
                        // Reset exception_header to primaryException and deallocate the dependent exception
                        __cxa_dependent_exception* dep_exception_header =
                            reinterpret_cast<__cxa_dependent_exception*>(exception_header);
                        exception_header =
                            cxa_exception_from_thrown_object(dep_exception_header->primaryException);
                        __cxa_free_dependent_exception(dep_exception_header);
                    }
                    // Destroy the primary exception only if its referenceCount goes to 0
                    //    (this decrement must be atomic)
                    __cxa_decrement_exception_refcount(thrown_object_from_cxa_exception(exception_header));
                }
            }
        }
        else
        {
            // The foreign exception has not been rethrown.  Pop the stack
            //    and delete it.  If there are nested catch's and they try
            //    to touch a foreign exception in any way, that is undefined
            //     behavior.  They likely can't since the only way to catch
            //     a foreign exception is with catch (...)!
            _Unwind_DeleteException(&globals->caughtExceptions->unwindHeader);
            globals->caughtExceptions = 0;
        }
    }
}
```
