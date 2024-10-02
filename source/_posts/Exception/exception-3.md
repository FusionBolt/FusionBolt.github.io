---
title: LLVM异常实现三 libunwind
typora-root-url: ../../source
date: 2024-10-02 11:28:08
category: Compiler
tags: Exception
---

上期提到的__cxa_xxx相关函数的实现离不开libunwind的相关接口，libunwind专门用于平台无关的堆栈展开和错误处理，内部做了很多平台相关的兼容工作，这期我们主要来介绍一下libunwind相关接口的具体实现。

# _Unwind_RaiseException

之前在throw的时候提到其中调用了_Unwind_RaiseException，这里则是libunwind的一个入口。

```cpp
void
__cxa_throw(void *thrown_object, std::type_info *tinfo, void (*dest)(void *)) {
    ...
    _Unwind_RaiseException(&exception_header->unwindHeader);
		...
}
```

Raise Exception的过程分为两步

1. 搜索exception所在的栈
2. clean up，本质是跳转到landingpad进行处理

UnwindLevel1.c

```cpp
/// Called by __cxa_throw.  Only returns if there is a fatal error.
_LIBUNWIND_EXPORT _Unwind_Reason_Code
_Unwind_RaiseException(_Unwind_Exception *exception_object) {
  _LIBUNWIND_TRACE_API("_Unwind_RaiseException(ex_obj=%p)",
                       (void *)exception_object);
  unw_context_t uc;
  unw_cursor_t cursor;
  __unw_getcontext(&uc);

  // Mark that this is a non-forced unwind, so _Unwind_Resume()
  // can do the right thing.
  exception_object->private_1 = 0;
  exception_object->private_2 = 0;

  // phase 1: the search phase
  _Unwind_Reason_Code phase1 = unwind_phase1(&uc, &cursor, exception_object);
  if (phase1 != _URC_NO_REASON)
    return phase1;

  // phase 2: the clean up phase
  return unwind_phase2(&uc, &cursor, exception_object);
}
```

# 出现的数据结构

libunwind.h

```cpp
struct unw_context_t {
  uint64_t data[_LIBUNWIND_CONTEXT_SIZE];
};
typedef struct unw_context_t unw_context_t;

struct unw_cursor_t {
  uint64_t data[_LIBUNWIND_CURSOR_SIZE];
} LIBUNWIND_CURSOR_ALIGNMENT_ATTR;
typedef struct unw_cursor_t unw_cursor_t;

#if defined(_WIN32) && defined(__SEH__)
  #define LIBUNWIND_CURSOR_ALIGNMENT_ATTR __attribute__((__aligned__(16)))
#else
  #define LIBUNWIND_CURSOR_ALIGNMENT_ATTR
#endif
```

context是用于程序运行中的上下文，包括各种寄存器的值。而cursor只是用于存放cursor对象的一个空间，后面会在这个空间上构造对应的cursor对象。_LIBUNWIND_CONTEXT_SIZE和_LIBUNWIND_CURSOR_SIZE则是定义在include/__libunwind_config.h中，不同平台不一致

比如i386的定义

```cpp
# if defined(__i386__)
#  define _LIBUNWIND_TARGET_I386
#  define _LIBUNWIND_CONTEXT_SIZE 8
#  define _LIBUNWIND_CURSOR_SIZE 15
#  define _LIBUNWIND_HIGHEST_DWARF_REGISTER _LIBUNWIND_HIGHEST_DWARF_REGISTER_X86
```

```cpp
struct _Unwind_Exception {
  _Unwind_Exception_Class exception_class;
  void (*exception_cleanup)(_Unwind_Reason_Code reason,
                            _Unwind_Exception *exc);
#if defined(__SEH__) && !defined(__USING_SJLJ_EXCEPTIONS__)
  uintptr_t private_[6];
#else
  uintptr_t private_1; // non-zero means forced unwind
  uintptr_t private_2; // holds sp that phase1 found for phase2 to use
#endif
#if __SIZEOF_POINTER__ == 4
  // The implementation of _Unwind_Exception uses an attribute mode on the
  // above fields which has the side effect of causing this whole struct to
  // round up to 32 bytes in size (48 with SEH). To be more explicit, we add
  // pad fields added for binary compatibility.
  uint32_t reserved[3];
#endif
  // The Itanium ABI requires that _Unwind_Exception objects are "double-word
  // aligned".  GCC has interpreted this to mean "use the maximum useful
  // alignment for the target"; so do we.
} __attribute__((__aligned__));

typedef uint64_t _Unwind_Exception_Class;
```

_Unwind_Exception主要是保存了exception_class，exception_clean_up的函数指针，private值。

# __unw_getcontext

实现在UnwindRegistersSave.S

根据不同的目标架构有着不同实现，但做的事情本质上是相同的，将当前程序执行的上下文（各种寄存器）临时保存起来。

i386的实现参考

```cpp
#
# extern int __unw_getcontext(unw_context_t* thread_state)
#
# On entry:
#   +                       +
#   +-----------------------+
#   + thread_state pointer  +
#   +-----------------------+
#   + return address        +
#   +-----------------------+   <-- SP
#   +                       +
#
```

```cpp
DEFINE_LIBUNWIND_FUNCTION(__unw_getcontext)

  _LIBUNWIND_CET_ENDBR
  push  %eax
  movl  8(%esp), %eax
  movl  %ebx,  4(%eax)
  movl  %ecx,  8(%eax)
  movl  %edx, 12(%eax)
  movl  %edi, 16(%eax)
  movl  %esi, 20(%eax)
  movl  %ebp, 24(%eax)
  movl  %esp, %edx
  addl  $8, %edx
  movl  %edx, 28(%eax)  # store what sp was at call site as esp
  # skip ss
  # skip eflags
  movl  4(%esp), %edx
  movl  %edx, 40(%eax)  # store return address as eip
  # skip cs
  # skip ds
  # skip es
  # skip fs
  # skip gs
  movl  (%esp), %edx
  movl  %edx, (%eax)  # store original eax
  popl  %eax
  xorl  %eax, %eax    # return UNW_ESUCCESS
  ret
```

# unwind_phase1

这个函数主要做了如下几件事情

1. 初始化cursor
2. 开始循环操作，根据cursor中的当前pc找到对应的eh_frame，从中获取return address（下次step的时候就会基于返回地址的调用者的栈）并更新cursor中的pc。
3. 获取proc_info
4. frame的handler不为空时，读取frame中personality，根据结果判断是否为所找的栈帧
   1. 如果是则将exception_object→private_2记录到sp中
   2. 否则继续查找
5. 如果全部找完了那么就是_URC_NO_REASON，**而不是返回错误**。

```cpp
static _Unwind_Reason_Code
unwind_phase1(unw_context_t *uc, unw_cursor_t *cursor, _Unwind_Exception *exception_object) {
  __unw_init_local(cursor, uc);

  // Walk each frame looking for a place to stop.
  while (true) {
    // Ask libunwind to get next frame (skip over first which is
    // _Unwind_RaiseException).
    int stepResult = __unw_step(cursor);
    if (stepResult == 0) {
          (void *)exception_object);
      return _URC_END_OF_STACK;
    } else if (stepResult < 0) {
          (void *)exception_object);
      return _URC_FATAL_PHASE1_ERROR;
    }

    // See if frame has code to run (has personality routine).
    unw_proc_info_t frameInfo;
    unw_word_t sp;
    if (__unw_get_proc_info(cursor, &frameInfo) != UNW_ESUCCESS) {
          (void *)exception_object);
      return _URC_FATAL_PHASE1_ERROR;
    }

    // If there is a personality routine, ask it if it will want to stop at
    // this frame.
    if (frameInfo.handler != 0) {
      _Unwind_Personality_Fn p =
          (_Unwind_Personality_Fn)(uintptr_t)(frameInfo.handler);
      _Unwind_Reason_Code personalityResult =
          (*p)(1, _UA_SEARCH_PHASE, exception_object->exception_class,
               exception_object, (struct _Unwind_Context *)(cursor));
      switch (personalityResult) {
      case _URC_HANDLER_FOUND:
        // found a catch clause or locals that need destructing in this frame
        // stop search and remember stack pointer at the frame
        __unw_get_reg(cursor, UNW_REG_SP, &sp);
        exception_object->private_2 = (uintptr_t)sp;
        return _URC_NO_REASON;

      case _URC_CONTINUE_UNWIND:
        // continue unwinding
        break;

      default:
        // something went wrong
        return _URC_FATAL_PHASE1_ERROR;
      }
    }
  }
  return _URC_NO_REASON;
}
```

第一阶段的personality，这个handler是从frameInfo中获取的，表明每个frame都可以有自己单独的personality，每个frame关联了一个函数，而在前面查看llvm ir的时候也是针对函数添加attribute标识personality函数。

以下是这个过程中出现的一些函数的实现

## __unw_init_local

```cpp
/// Create a cursor of a thread in this process given 'context' recorded by
/// __unw_getcontext().
_LIBUNWIND_HIDDEN int __unw_init_local(unw_cursor_t *cursor,
                                       unw_context_t *context) {
  _LIBUNWIND_TRACE_API("__unw_init_local(cursor=%p, context=%p)",
                       static_cast<void *>(cursor),
                       static_cast<void *>(context));
#if defined(__i386__)
# define REGISTER_KIND Registers_x86
...
#endif
  // Use "placement new" to allocate UnwindCursor in the cursor buffer.
  new (reinterpret_cast<UnwindCursor<LocalAddressSpace, REGISTER_KIND> *>(cursor))
      UnwindCursor<LocalAddressSpace, REGISTER_KIND>(
          context, LocalAddressSpace::sThisAddressSpace);
#undef REGISTER_KIND
  AbstractUnwindCursor *co = (AbstractUnwindCursor *)cursor;
  co->setInfoBasedOnIPRegister();

  return UNW_ESUCCESS;
}
```

在cursor的地址上通过placement new构造对象，所以cursor实际上是一个UnwindCursor

```cpp
// libunwind does not and should not depend on C++ library which means that we
// need our own definition of inline placement new.
static void *operator new(size_t, UnwindCursor<A, R> *p) { return p; }
```

类型为UnwindCursor<LocalAddressSpace, Registers_x86>

### UnwindCursor

这个类主要用于指向unwind过程中的各个栈帧，通过这个类提取出相关的信息。

libunwind/src/UnwindCursor.hpp

```cpp
/// UnwindCursor contains all state (including all register values) during
/// an unwind.  This is normally stack allocated inside a unw_cursor_t.
template <typename A, typename R>
class UnwindCursor : public AbstractUnwindCursor{
  typedef typename A::pint_t pint_t;
public:
                      UnwindCursor(unw_context_t *context, A &as);
                      UnwindCursor(A &as, void *threadArg);
  ...
  A               &_addressSpace;
  R                _registers;
  unw_proc_info_t  _info;
  bool             _unwindInfoMissing; // 是否有调试信息
  bool             _isSignalFrame;
};

template <typename A, typename R>
UnwindCursor<A, R>::UnwindCursor(unw_context_t *context, A &as)
    : _addressSpace(as), _registers(context), _unwindInfoMissing(false),
      _isSignalFrame(false) {
  static_assert((check_fit<UnwindCursor<A, R>, unw_cursor_t>::does_fit),
                "UnwindCursor<> does not fit in unw_cursor_t");
  static_assert((alignof(UnwindCursor<A, R>) <= alignof(unw_cursor_t)),
                "UnwindCursor<> requires more alignment than unw_cursor_t");
  memset(&_info, 0, sizeof(_info));
}
```

先前传入的context作为这里的registers，在registers的构造函数中会拷贝context地址中的值到对应保存registers值的成员变量中。

```cpp
struct unw_proc_info_t {
  unw_word_t  start_ip;         /* start address of function */
  unw_word_t  end_ip;           /* address after end of function */
  unw_word_t  lsda;             /* address of language specific data area, */
                                /*  or zero if not used */
  unw_word_t  handler;          /* personality routine, or zero if not used */
  unw_word_t  gp;               /* not used */
  unw_word_t  flags;            /* not used */
  uint32_t    format;           /* compact unwind encoding, or zero if none */
  uint32_t    unwind_info_size; /* size of DWARF unwind info, or zero if none */
  unw_word_t  unwind_info;      /* address of DWARF unwind info, or zero */
  unw_word_t  extra;            /* mach_header of mach-o image containing func */
};
typedef struct unw_proc_info_t unw_proc_info_t;
```

proc_info主要保存了针对某个proc的一个栈帧的一些异常处理信息。

### UnwindCursor::setInfoBasedOnIPRegister

最后设置setInfoBasedOnIPRegister。这里各种平台相关的宏定义太多，我们先暂且忽略掉大部分的，先来看drawf unwind的情况。

1. 获取了产生异常的pc地址
2. 针对最后一条命令是throw的情况修正对应的pc
3. 寻找unwind sections
4. 找到后则从对应的section中解析信息填写到_info中，之后返回（getInfoFromDwarfSection）
5. 如果没找到那么标记没有相关信息，最后返回

```cpp
template <typename A, typename R>
void UnwindCursor<A, R>::setInfoBasedOnIPRegister(bool isReturnAddress) {
  pint_t pc = static_cast<pint_t>(this->getReg(UNW_REG_IP));
  // Exit early if at the top of the stack.
  if (pc == 0) {
    _unwindInfoMissing = true;
    return;
  }
  
  // If the last line of a function is a "throw" the compiler sometimes
  // emits no instructions after the call to __cxa_throw.  This means
  // the return address is actually the start of the next function.
  // To disambiguate this, back up the pc when we know it is a return
  // address.
  if (isReturnAddress)
	  --pc;
	  
  // Ask address space object to find unwind sections for this pc.
  UnwindInfoSections sects;
  if (_addressSpace.findUnwindSections(pc, sects)) {
		 ....
#if defined(_LIBUNWIND_SUPPORT_DWARF_UNWIND)
    // If there is dwarf unwind info, look there next.
    if (sects.dwarf_section != 0) {
      if (this->getInfoFromDwarfSection(pc, sects)) {
        // found info in dwarf, done
        return;
      }
    }
#endif
		...
  }
  
  // no unwind info, flag that we can't reliably unwind
  _unwindInfoMissing = true;
}
```

```cpp
/// Used by findUnwindSections() to return info about needed sections.
struct UnwindInfoSections {
#if defined(_LIBUNWIND_SUPPORT_DWARF_UNWIND) ||                                \
    defined(_LIBUNWIND_SUPPORT_COMPACT_UNWIND) ||                              \
    defined(_LIBUNWIND_USE_DL_ITERATE_PHDR)
  // No dso_base for SEH.
  uintptr_t       dso_base;
#endif
#if defined(_LIBUNWIND_USE_DL_ITERATE_PHDR)
  size_t          text_segment_length;
#endif
#if defined(_LIBUNWIND_SUPPORT_DWARF_UNWIND)
  uintptr_t       dwarf_section;
  size_t          dwarf_section_length;
#endif
#if defined(_LIBUNWIND_SUPPORT_DWARF_INDEX)
  uintptr_t       dwarf_index_section;
  size_t          dwarf_index_section_length;
#endif
#if defined(_LIBUNWIND_SUPPORT_COMPACT_UNWIND)
  uintptr_t       compact_unwind_section;
  size_t          compact_unwind_section_length;
#endif
#if defined(_LIBUNWIND_ARM_EHABI)
  uintptr_t       arm_section;
  size_t          arm_section_length;
#endif
};
```

### findUnwindSections

这里各种平台的处理都是完全不同的，实现的本质都是寻找对应的段，比如ehframe，这里选择DWARF unwind以及baremetal的情况作为参考，直接使用链接器中定义的__eh_frame_start和__eh_frame_end来获得对应的eh_frame section的长度，根据是否为空判断对应的信息是否存在，并且更新对应的UnwindInfoSections

```cpp
inline bool LocalAddressSpace::findUnwindSections(pint_t targetAddr,
                                                  UnwindInfoSections &info) {
#elif defined(_LIBUNWIND_SUPPORT_DWARF_UNWIND) && defined(_LIBUNWIND_IS_BAREMETAL)
  info.dso_base = 0;
  // Bare metal is statically linked, so no need to ask the dynamic loader
  info.dwarf_section_length = (size_t)(&__eh_frame_end - &__eh_frame_start);
  info.dwarf_section =        (uintptr_t)(&__eh_frame_start);
  _LIBUNWIND_TRACE_UNWINDING("findUnwindSections: section %p length %p",
                             (void *)info.dwarf_section, (void *)info.dwarf_section_length);
#if defined(_LIBUNWIND_SUPPORT_DWARF_INDEX)
  info.dwarf_index_section =        (uintptr_t)(&__eh_frame_hdr_start);
  info.dwarf_index_section_length = (size_t)(&__eh_frame_hdr_end - &__eh_frame_hdr_start);
  _LIBUNWIND_TRACE_UNWINDING("findUnwindSections: index section %p length %p",
                             (void *)info.dwarf_index_section, (void *)info.dwarf_index_section_length);
#endif       
  if (info.dwarf_section_length)
    return true;   
#elif ...                                       
}
```

## __unw_step

调用cursor的step。

这里主要是解析eh_frame中的信息，之后更新寄存器信息，包括对应的返回值地址，用于下一次step的时候根据这个返回值找到调用者的栈帧。

```cpp
/// Move cursor to next frame.
_LIBUNWIND_HIDDEN int __unw_step(unw_cursor_t *cursor) {
  _LIBUNWIND_TRACE_API("__unw_step(cursor=%p)", static_cast<void *>(cursor));
  AbstractUnwindCursor *co = (AbstractUnwindCursor *)cursor;
  return co->step();
}

template <typename A, typename R> int UnwindCursor<A, R>::step(bool stage2) {
  (void)stage2;
  // Bottom of stack is defined is when unwind info cannot be found.
  if (_unwindInfoMissing)
    return UNW_STEP_END;

...
#elif defined(_LIBUNWIND_SUPPORT_DWARF_UNWIND)
    **result = this->stepWithDwarfFDE(stage2);**
...
#endif
  }

  // update info based on new PC
  if (result == UNW_STEP_SUCCESS) {
    this->setInfoBasedOnIPRegister(true);
    if (_unwindInfoMissing)
      return UNW_STEP_END;
  }

  return result;
}

int stepWithDwarfFDE(bool stage2) {
  return DwarfInstructions<A, R>::stepWithDwarf(
      _addressSpace, (pint_t)this->getReg(UNW_REG_IP),
      (pint_t)_info.unwind_info, _registers, _isSignalFrame, stage2);
}
```

```cpp
template <typename A, typename R>
int DwarfInstructions<A, R>::stepWithDwarf(A &addressSpace, pint_t pc,
                                           pint_t fdeStart, R &registers,
                                           bool &isSignalFrame) {
  FDE_Info fdeInfo;
  CIE_Info cieInfo;
  if (CFI_Parser<A>::decodeFDE(addressSpace, fdeStart, &fdeInfo,
                               &cieInfo) == NULL) {
    PrologInfo prolog;
    if (CFI_Parser<A>::parseFDEInstructions(addressSpace, fdeInfo, cieInfo, pc,
                                            R::getArch(), &prolog)) {
      // get pointer to cfa (architecture specific)
      pint_t cfa = getCFA(addressSpace, prolog, registers);

       // restore registers that DWARF says were saved
      R newRegisters = registers;

      // Typically, the CFA is the stack pointer at the call site in
      // the previous frame. However, there are scenarios in which this is not
      // true. For example, if we switched to a new stack. In that case, the
      // value of the previous SP might be indicated by a CFI directive.
      //
      // We set the SP here to the CFA, allowing for it to be overridden
      // by a CFI directive later on.
      newRegisters.setSP(cfa);

      pint_t returnAddress = 0;
      const int lastReg = R::lastDwarfRegNum();
      assert(static_cast<int>(CFI_Parser<A>::kMaxRegisterNumber) >= lastReg &&
             "register range too large");
      assert(lastReg >= (int)cieInfo.returnAddressRegister &&
             "register range does not contain return address register");
      for (int i = 0; i <= lastReg; ++i) {
        if (prolog.savedRegisters[i].location !=
            CFI_Parser<A>::kRegisterUnused) {
          if (registers.validFloatRegister(i))
            newRegisters.setFloatRegister(
                i, getSavedFloatRegister(addressSpace, registers, cfa,
                                         prolog.savedRegisters[i]));
          else if (registers.validVectorRegister(i))
            newRegisters.setVectorRegister(
                i, getSavedVectorRegister(addressSpace, registers, cfa,
                                          prolog.savedRegisters[i]));
          else if (i == (int)cieInfo.returnAddressRegister)
            returnAddress = getSavedRegister(addressSpace, registers, cfa,
                                             prolog.savedRegisters[i]);
          else if (registers.validRegister(i))
            newRegisters.setRegister(
                i, getSavedRegister(addressSpace, registers, cfa,
                                    prolog.savedRegisters[i]));
          else
            return UNW_EBADREG;
        } else if (i == (int)cieInfo.returnAddressRegister) {
            // Leaf function keeps the return address in register and there is no
            // explicit intructions how to restore it.
            returnAddress = registers.getRegister(cieInfo.returnAddressRegister);
        }
      }

      isSignalFrame = cieInfo.isSignalFrame;

			...

      // Return address is address after call site instruction, so setting IP to
      // that does simualates a return.
      newRegisters.setIP(returnAddress);

      // Simulate the step by replacing the register set with the new ones.
      registers = newRegisters;

      return UNW_STEP_SUCCESS;
    }
  }
  return UNW_EBADFRAME;
}
```

## __unw_get_proc_info

将前面step过程时cursor更新的proc_info信息写入到info指针中

```cpp
/// Get unwind info at cursor position in stack frame.
_LIBUNWIND_HIDDEN int __unw_get_proc_info(unw_cursor_t *cursor,
                                          unw_proc_info_t *info) {
  _LIBUNWIND_TRACE_API("__unw_get_proc_info(cursor=%p, &info=%p)",
                       static_cast<void *>(cursor), static_cast<void *>(info));
  AbstractUnwindCursor *co = (AbstractUnwindCursor *)cursor;
  co->getInfo(info);
  if (info->end_ip == 0)
    return UNW_ENOINFO;
  return UNW_ESUCCESS;
}
_LIBUNWIND_WEAK_ALIAS(__unw_get_proc_info, unw_get_proc_info)
```

```cpp
template <typename A, typename R>
void UnwindCursor<A, R>::getInfo(unw_proc_info_t *info) {
  if (_unwindInfoMissing)
    memset(info, 0, sizeof(*info));
  else
    *info = _info;
}
```

# context与cursor的联系

context和cusor两个变量的各种传递关系搞得我比较混乱，加上各种调用也比较复杂，因此我在这里整理一下这两个变量的关系。

context是一个buffer，用于保存上下文的寄存器

cursor是一个buffer，用于对象的实际构造

两个变量主要在_Unwind_RaiseException的过程中起到作用。

1. 首先在_Unwind_RaiseException的开始读取了context到context的buffer中
2. __unw_init_local:在每个phase的开始初始化local，即根据context的值。
   1. 将context所保存的寄存器拷贝到cursor中
   2. 内部调用setInfoBasedOnIPRegister
      1. getInfoFromDwarfSection
         1. getInfoFromFdeCie 更新info
3. 在定位对应栈帧的时候会调用cursor.step()
   1. 内部调用stepWithDwarfFDE
      1. 更新registers。registers = newRegisters;
4. __unw_get_proc_info: 读取cursor中的proc info，以及sp

register每次都会拷贝context中的值。也就是说第一次更新的第二次不会生效，第二次开始搜索的位置和第一次相同。

# unwind_phase2

```cpp
static _Unwind_Reason_Code
unwind_phase2(unw_context_t *uc, unw_cursor_t *cursor, _Unwind_Exception *exception_object) {
  __unw_init_local(cursor, uc);

  // uc is initialized by __unw_getcontext in the parent frame. The first stack
  // frame walked is unwind_phase2.
  unsigned framesWalked = 1;
  // Walk each frame until we reach where search phase said to stop.
  while (true) {

    // Ask libunwind to get next frame (skip over first which is
    // _Unwind_RaiseException).
    int stepResult = __unw_step(cursor);
    if (stepResult == 0) {
      return _URC_END_OF_STACK;
    } else if (stepResult < 0) {
      return _URC_FATAL_PHASE2_ERROR;
    }

    // Get info about this frame.
    unw_word_t sp;
    unw_proc_info_t frameInfo;
    __unw_get_reg(cursor, UNW_REG_SP, &sp);
    if (__unw_get_proc_info(cursor, &frameInfo) != UNW_ESUCCESS) {
      return _URC_FATAL_PHASE2_ERROR;
    }

    ++framesWalked;
    // If there is a personality routine, tell it we are unwinding.
    if (frameInfo.handler != 0) {
      _Unwind_Personality_Fn p =
          (_Unwind_Personality_Fn)(uintptr_t)(frameInfo.handler);
      _Unwind_Action action = _UA_CLEANUP_PHASE;
      if (sp == exception_object->private_2) {
        // Tell personality this was the frame it marked in phase 1.
        action = (_Unwind_Action)(_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME);
      }
       _Unwind_Reason_Code personalityResult =
          (*p)(1, action, exception_object->exception_class, exception_object,
               (struct _Unwind_Context *)(cursor));
      switch (personalityResult) {
      case _URC_CONTINUE_UNWIND:
        // Continue unwinding
        if (sp == exception_object->private_2) {
          // Phase 1 said we would stop at this frame, but we did not...
          _LIBUNWIND_ABORT("during phase1 personality function said it would "
                           "stop here, but now in phase2 it did not stop here");
        }
        break;
      case _URC_INSTALL_CONTEXT:
        // Personality routine says to transfer control to landing pad.
        // We may get control back if landing pad calls _Unwind_Resume().
        __unw_phase2_resume(cursor, framesWalked);
        // __unw_phase2_resume() only returns if there was an error.
        return _URC_FATAL_PHASE2_ERROR;
      default:
        // Personality routine returned an unknown result code.
        return _URC_FATAL_PHASE2_ERROR;
      }
    }
  }

  // Clean up phase did not resume at the frame that the search phase
  // said it would...
  return _URC_FATAL_PHASE2_ERROR;
}
```

1. 找到next_frame
2. 获取proc_info
3. frame的handler不为空的时候
   1. 根据private_2的情况设置不同的action，之后再执行personality操作
   2. 查看结果
      1. _URC_INSTALL_CONTEXT会执行resume，transfer control to landing pad
      2. _URC_CONTINUE_UNWIND在前一步search的时候设置了，表明应该在这个frame中停下，但实际没停，所以会炸
4. 否则继续查找，如果全部找完了那么就是_URC_FATAL_PHASE2_ERROR

本质上是找到第一个handler不为空的frame

注意两次执行handler时的action，第一次是_UA_SEARCH_PHASE，第二次是cleanup或者cleanup | handle

```cpp
typedef enum {
  _UA_SEARCH_PHASE = 1,
  _UA_CLEANUP_PHASE = 2,
  _UA_HANDLER_FRAME = 4,
  _UA_FORCE_UNWIND = 8,
  _UA_END_OF_STACK = 16 // gcc extension to C++ ABI
} _Unwind_Action;
```

# phase2_resume

```cpp
// When CET is enabled, each "call" instruction will push return address to
// CET shadow stack, each "ret" instruction will pop current CET shadow stack
// top and compare it with target address which program will return.
// In exception handing, **some stack frames will be skipped before jumping to
// landing pad and we must adjust CET shadow stack accordingly.**
// _LIBUNWIND_POP_CET_SSP is used to adjust CET shadow stack pointer and we
// directly jump to __libunwind_Registerts_x86/x86_64_jumpto instead of using
// a regular function call to avoid pushing to CET shadow stack again.
#if !defined(_LIBUNWIND_USE_CET)
#define __unw_phase2_resume(cursor, fn) __unw_resume((cursor))
#elif defined(_LIBUNWIND_TARGET_I386)
#define __unw_phase2_resume(cursor, fn)                                        \
  do {                                                                         \
    _LIBUNWIND_POP_CET_SSP((fn));                                              \
    void *cetRegContext = __libunwind_cet_get_registers((cursor));             \
    void *cetJumpAddress = __libunwind_cet_get_jump_target();                  \
    __asm__ volatile("push %%edi\n\t"                                          \
                     "sub $4, %%esp\n\t"                                       \
                     "jmp *%%edx\n\t" :: "D"(cetRegContext),                   \
                     "d"(cetJumpAddress));                                     \
  } while (0)
#elif defined(_LIBUNWIND_TARGET_X86_64)
#define __unw_phase2_resusme(cursor, fn)                                        \
  do {                                                                         \
    _LIBUNWIND_POP_CET_SSP((fn));                                            \
    void *cetRegContext = __libunwind_cet_get_registers((cursor));             \
    void *cetJumpAddress = __libunwind_cet_get_jump_target();                  \
    __asm__ volatile("jmpq *%%rdx\n\t" :: "D"(cetRegContext),                  \
                     "d"(cetJumpAddress));                                     \
  } while (0)
#endif
```

这里主要是跳转到对应的汇编处理代码。使用CET时会额外插入CET命令，并且调用__unw_resume。

CET：Control-flow Enforcement Technology。控制流保护

[Control-flow integrity](https://en.wikipedia.org/wiki/Control-flow_integrity)

> **some stack frames will be skipped before jumping to landing pad and we must adjust CET shadow stack accordingly.**

```cpp
/// Resume execution at cursor position (aka longjump).
_LIBUNWIND_HIDDEN int __unw_resume(unw_cursor_t *cursor) {
  _LIBUNWIND_TRACE_API("__unw_resume(cursor=%p)", static_cast<void *>(cursor));
#if __has_feature(address_sanitizer) || defined(__SANITIZE_ADDRESS__)
  // Inform the ASan runtime that now might be a good time to clean stuff up.
  __asan_handle_no_return();
#endif
  AbstractUnwindCursor *co = (AbstractUnwindCursor *)cursor;
  co->jumpto();
  return UNW_EUNSPEC;
}

template <typename A, typename R> void UnwindCursor<A, R>::jumpto() {
  _registers.jumpto();
}

void jumpto() { __libunwind_Registers_x86_jumpto(this); }
```

剩下的情况，根据i386和x86_64进行不同的处理

```cpp
#if defined(_LIBUNWIND_USE_CET)
extern "C" void *__libunwind_cet_get_registers(unw_cursor_t *cursor) {
  AbstractUnwindCursor *co = (AbstractUnwindCursor *)cursor;
  return co->get_registers();
}
#endif

#if defined(_LIBUNWIND_TARGET_I386)
class _LIBUNWIND_HIDDEN Registers_x86;
extern "C" void __libunwind_Registers_x86_jumpto(Registers_x86 *);

#if defined(_LIBUNWIND_USE_CET)
extern "C" void *__libunwind_cet_get_jump_target() {
  return reinterpret_cast<void *>(&__libunwind_Registers_x86_jumpto);
}
#endif
```

最后都会跳转到这个__libunwind_Registers_x86_jumpto，下面的内容是原始的汇编代码以及我加的一些注释。

```cpp
DEFINE_LIBUNWIND_FUNCTION(__libunwind_Registers_x86_jumpto)
#
# extern "C" void __libunwind_Registers_x86_jumpto(Registers_x86 *);
#
# On entry:
#  +                       +
#  +-----------------------+
#  + thread_state pointer  +
#  +-----------------------+
#  + return address        +
#  +-----------------------+   <-- SP
#  +                       +

  _LIBUNWIND_CET_ENDBR
  movl   4(%esp), %eax # esp的地址+4（即this指针的值）加载到eax
  # set up eax and ret on new stack location
  movl  28(%eax), %edx # edx holds new stack pointer
		  # 28(%eax) 是this的esp，这个sp是在寻找栈帧解析dwarf的时候记录的
  subl  $8,%edx # edx -= 8
  
  --- this.esp  -->  ---
								    	    8
							       --- edx = this.esp - 8
  
  movl  %edx, 28(%eax) # 保存edx到this的eip中
  # 相当于this的esp -= 8
  movl  0(%eax), %ebx # this的eax地址的值加载到ebx
  # 原来的eax保存的是指向unwind_exception的指针，这个是在personality中设置的 
  movl  %ebx, 0(%edx) # ebx的值写到edx所在地址，也就是
  movl  40(%eax), %ebx # eip写到ebx
  movl  %ebx, 4(%edx) # 4(edx) = eip
  
	--- ebx + 8 // this.esp
	this.eax
	--- ebx + 4
	新的 eip
	--- ebx
  新开辟了一块空间，保存了eip和registers
  
  # we now have ret and eax pushed onto where new stack will be
  # restore all registers
  movl   4(%eax), %ebx
  movl   8(%eax), %ecx
  movl  12(%eax), %edx
  movl  16(%eax), %edi
  movl  20(%eax), %esi
  movl  24(%eax), %ebp
  movl  28(%eax), %esp
  # skip ss
  # skip eflags
  pop    %eax  # eax was already pushed on new stack
  pop    %ecx  # pop esp上的值到eax和ecx，ecx保存了eip，即landingpad的值
  jmp    *%ecx
  # skip cs
  # skip ds
  # skip es
  # skip fs
  # skip gs
```

这里ecx是landingpad的地址，是在phase2的时候personality中设置的对应的值。
