---
title: LLVM异常实现四 personality
typora-root-url: ../../source
date: 2024-10-02 11:28:11
category: Compiler
tags: Exception
---

前面libunwind的过程中多次和personality进行交互，这部分是由语言提供和语言相关的内容。libunwind的两个阶段对应到这一个函数之中，personality部分根据libunwind扫描到的平台相关的信息，在ehframe中扫描到当前栈帧对应的异常处理信息。

personality的第一阶段主要任务找到对应栈的异常信息，将这些信息写入一个异常对象中。第二阶段则是将异常相关的信息实际写入到context相关的寄存器中，待返回后供libunwind跳转到异常处理的位置.

# 不同平台下的personality

不同平台下有些许差距，为了减少代码提及在这里使用宏定义进行了区分

```cpp
#if !defined(_LIBCXXABI_ARM_EHABI)
		#if defined(__SEH__) && !defined(__USING_SJLJ_EXCEPTIONS__)
		static _Unwind_Reason_Code __gxx_personality_imp
		#else
		_LIBCXXABI_FUNC_VIS _Unwind_Reason_Code
				#ifdef __USING_SJLJ_EXCEPTIONS__
				__gxx_personality_sj0
				#else
				__gxx_personality_v0
				#endif
		#endif
		
		#if defined(__SEH__) && !defined(__USING_SJLJ_EXCEPTIONS__)
		extern "C" _LIBCXXABI_FUNC_VIS EXCEPTION_DISPOSITION
		__gxx_personality_seh0(PEXCEPTION_RECORD ms_exc, void *this_frame,
		                       PCONTEXT ms_orig_context, PDISPATCHER_CONTEXT ms_disp)
		{
		  return _GCC_specific_handler(ms_exc, this_frame, ms_orig_context, ms_disp,
		                               __gxx_personality_imp);
		}
		#endif
#else
extern "C" _Unwind_Reason_Code __gnu_unwind_frame(_Unwind_Exception*,
                                                  _Unwind_Context*);
static _Unwind_Reason_Code continue_unwind(_Unwind_Exception* unwind_exception,...
// ARM register names
#if !defined(_LIBUNWIND_VERSION)
static const uint32_t REG_UCB = 12;  // Register to save _Unwind_Control_Block
#endif
static const uint32_t REG_SP = 13;
extern "C" _LIBCXXABI_FUNC_VIS _Unwind_Reason_Code
**__gxx_personality_v0**(_Unwind_State state,
                     _Unwind_Exception* unwind_exception,
                     _Unwind_Context* context)
{...}
#endif
```

personality主要是用于在libunwind中提到的两个phase。

```cpp
__gxx_personality_sj0(int version, _Unwind_Action actions, uint64_t exceptionClass,
                     _Unwind_Exception* unwind_exception, _Unwind_Context* context)
{
    if (version != 1 || unwind_exception == 0 || context == 0)
        return _URC_FATAL_PHASE1_ERROR;

    bool native_exception = (exceptionClass     & get_vendor_and_language) ==
                            (kOurExceptionClass & get_vendor_and_language);
    scan_results results;
    
    // Process a catch handler for a native exception first.
    if (actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME) &&
        native_exception) {
	     ... // 在phase2中使用   
    }
    // In other cases we need to scan LSDA.
    scan_eh_tab(results, actions, native_exception, unwind_exception, context);
    if (results.reason == _URC_CONTINUE_UNWIND ||
        results.reason == _URC_FATAL_PHASE1_ERROR)
        return results.reason;
    
    if (actions & _UA_SEARCH_PHASE) {
	    ... // phase1中使用
    }
    
		
		... // phase2
    return _URC_INSTALL_CONTEXT;
}
```

# phase1

search phase，主要用于找到对应栈的异常处理信息，因此在此之前需要先扫描eh_frame来获取lsda。

如果找不到对应处理信息那么会返回continue unwind，在libunwind中继续step到下一个栈，直到找到对应的栈帧或者scan出错为止。

如果找到异常处理信息，那么会保存在__cxa_exception中，供phase2读取使用。

```cpp
scan_eh_tab(results, actions, native_exception, unwind_exception, context);
if (results.reason == _URC_CONTINUE_UNWIND ||
    results.reason == _URC_FATAL_PHASE1_ERROR)
    return results.reason;

if (actions & _UA_SEARCH_PHASE)
{
    // Phase 1 search:  All we're looking for in phase 1 is a handler that
    //   halts unwinding
    assert(results.reason == _URC_HANDLER_FOUND);
    if (native_exception) {
        // For a native exception, cache the LSDA result.
        __cxa_exception* exc = (__cxa_exception*)(unwind_exception + 1) - 1;
        exc->handlerSwitchValue = static_cast<int>(results.ttypeIndex);
        exc->actionRecord = results.actionRecord;
        exc->languageSpecificData = results.languageSpecificData;
        exc->catchTemp = reinterpret_cast<void*>(results.landingPad);
        exc->adjustedPtr = results.adjustedPtr;
    }
    return _URC_HANDLER_FOUND;
}
```

# phase2

clean up phase，此时会有两类情况。

一类是在phase1中已经找好对应的异常处理信息，可以直接读取信息并且设置context的寄存器，此时读取的就是phase1中保存的结果，通过set_registers设置对应的值，之后会返回libunwind并且跳转到libunwind。

另一类则是没有可用处理信息的情况，会再次scan_eh_tab，使用搜索到的结果更新context。两次进行scan的原因是第一阶段允许

```cpp
// Process a catch handler for a native exception first.
if (actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME) &&
    native_exception) {
    // Reload the results from the phase 1 cache.
    __cxa_exception* exception_header =
        (__cxa_exception*)(unwind_exception + 1) - 1;
    results.ttypeIndex = exception_header->handlerSwitchValue;
    results.actionRecord = exception_header->actionRecord;
    results.languageSpecificData = exception_header->languageSpecificData;
    results.landingPad =
        reinterpret_cast<uintptr_t>(exception_header->catchTemp);
    results.adjustedPtr = exception_header->adjustedPtr;

    // Jump to the handler.
    set_registers(unwind_exception, context, results);
    // Cache base for calculating the address of ttype in
    // __cxa_call_unexpected.
    return _URC_INSTALL_CONTEXT;
}
...
scan_eh_tab(results, actions, native_exception, unwind_exception, context);
...
// if phase1
...
assert(actions & _UA_CLEANUP_PHASE);
assert(results.reason == _URC_HANDLER_FOUND);
set_registers(unwind_exception, context, results);
// Cache base for calculating the address of ttype in __cxa_call_unexpected.
if (results.ttypeIndex < 0) {
  __cxa_exception* exception_header =
        (__cxa_exception*)(unwind_exception + 1) - 1;
#if defined(_AIX)
  exception_header->catchTemp = (void *)_Unwind_GetDataRelBase(context);
#else
  exception_header->catchTemp = 0;
#endif
}
return _URC_INSTALL_CONTEXT;
```

# lsda的结构

整个流程中少不了的是scan_eh_tab这个过程，而这个函数主要的用途就是去解析lsda的结构读取想要的信息。

图中Exception Table所对应的数据是lsda的结构，这个结构根据平台不同也有所变化，具体详情参考libcxxabi/src/cxa_personality.cpp中的注释

![Untitled](/images/exception-4/Untitled.png)

主要的信息都在callSiteTable中，之前的部分主要是一些元信息

# scan_eh_tab

```cpp
static void scan_eh_tab(scan_results &results, _Unwind_Action actions,
                        bool native_exception,
                        _Unwind_Exception *unwind_exception,
                        _Unwind_Context *context);
```

三种scan的类型

1. Scan for handler with native or foreign exception.
2. Scan for handler with foreign exception.
3. Scan for cleanups.

scan的结果

```cpp
struct scan_results
{
    int64_t        ttypeIndex;   // > 0 catch handler, < 0 exception spec handler, == 0 a cleanup
    const uint8_t* actionRecord;         // Currently unused.  Retained to ease future maintenance.
    const uint8_t* languageSpecificData;  // Needed only for __cxa_call_unexpected
    uintptr_t      landingPad;   // null -> nothing found, else something found
    void*          adjustedPtr;  // Used in cxa_exception.cpp
    _Unwind_Reason_Code reason;  // One of _URC_FATAL_PHASE1_ERROR,
                                 //        _URC_FATAL_PHASE2_ERROR,
                                 //        _URC_CONTINUE_UNWIND,
                                 //        _URC_HANDLER_FOUND
};
```

整个scan都过程代码太长，主要的流程就是找到需要查询的数据（LSDA），之后解码数据，一个个检查是否为对应栈帧的数据。

使用文字流程代替

1. 检查对应的actions
2. 获取LSDA（LanguageSpecificData）
3. 设置base，获取ip，func start frame
4. LSDA的数据解码，获取callSiteTable以及actionTable
5. 遍历所有的callSite
   1. 根据当前callSite的位置解码start，length，landingpad，actionEntry
   2. 根据不同架构计算出实际的landingpad
   3. actionEntry为0，那么返回reason
      1. 如果是search phase那么是continue unwind
      2. 否则是handler found
   4. 根据action表以及actionEntry得到实际的action
   5. 循环处理
      1. 根据action得到actionRecord和ttypeindex
      2. ttypeindex > 0
         1. 找到一个catch，get_shim_type_info获取type检查catch
         2. 如果catch为0，那么这里catch everything
            1. 包含foreign exception。这里必须是一个search phase。clean up phase with foreign exception，或者force unwinding
            2. 记录下ttypeIndex, actionRecord, adjustedPtr（get_thrown_object_ptr）, reason并且返回
         3. 如果catch不为0且是native exception
            1. 那么此时是一个catch(T)，将不会catch一个foreign exception
            2. 获取exception header
            3. 获取adjustedPtr和excpType
            4. 全为空则terminate
            5. 如果catchType can_catch，那么handler_found，记录信息并且返回
         4. 否则scan下一个action
      3. ttypeindex < 0
         1. _UA_FORCE_UNWIND，意味着找到一个exception specification，直接跳过
         2. 如果不是force并且native_exception
            1. 获取exception header
            2. 获取adjustedPtr和excpType
            3. 全为空则terminate
            4. 如果exception_spec_can_catch，那么handler_found，记录信息并且返回
         3. 否则foreign exception caught by exception spec，记录信息并且返回
      4. ttypeindex == 0， 表明cleanup为true
      5. action中读取actionOffset
      6. actionOffset为0，代表是action list的结尾
         1. hasCleaup 并且是处于cleanup phase的话表示找到handler_found
         2. 否则是continue_unwind
      7. 前进到下一个action
6. 找不到任何eh table entry来指定如何处理exception的情况下terminate

## _Unwind_GetLanguageSpecificData

```cpp
/// Called by personality handler during phase 2 to get LSDA for current frame.
_LIBUNWIND_EXPORT uintptr_t
_Unwind_GetLanguageSpecificData(struct _Unwind_Context *context) {
  unw_cursor_t *cursor = (unw_cursor_t *)context;
  unw_proc_info_t frameInfo;
  uintptr_t result = 0;
  if (__unw_get_proc_info(cursor, &frameInfo) == UNW_ESUCCESS)
    result = (uintptr_t)frameInfo.lsda;
  _LIBUNWIND_TRACE_API(
      "_Unwind_GetLanguageSpecificData(context=%p) => 0x%llx",
      static_cast<void *>(context), (long long)result);
  return result;
}

```

unw_proc_info_t中包含了lsda的地址，直接读取并且返回

## set_registers

```cpp
static
void
set_registers(_Unwind_Exception* unwind_exception, _Unwind_Context* context,
              const scan_results& results)
{
#if defined(__USING_SJLJ_EXCEPTIONS__) || defined(__USING_WASM_EXCEPTIONS__)
#define __builtin_eh_return_data_regno(regno) regno
#elif defined(__ibmxl__)
// IBM xlclang++ compiler does not support __builtin_eh_return_data_regno.
#define __builtin_eh_return_data_regno(regno) regno + 3
#endif
  _Unwind_SetGR(context, __builtin_eh_return_data_regno(0),
                reinterpret_cast<uintptr_t>(unwind_exception));
  _Unwind_SetGR(context, __builtin_eh_return_data_regno(1),
                static_cast<uintptr_t>(results.ttypeIndex));
  _Unwind_SetIP(context, results.landingPad);
}
```

这里将exception对象地址和typeindex放到了context中，可以联系上一篇文章中的phase2_resume部分的汇编查看，那里用到的值都是在这里设置的。

后面的具体实现大概看一下就好

```cpp
_LIBUNWIND_EXPORT_UNWIND_LEVEL1
void _Unwind_SetGR(struct _Unwind_Context *context, int index,
                   uintptr_t value) {
  _Unwind_VRS_Set(context, _UVRSC_CORE, (uint32_t)index, _UVRSD_UINT32, &value);
}

_LIBUNWIND_EXPORT_UNWIND_LEVEL1
void _Unwind_SetIP(struct _Unwind_Context *context, uintptr_t value) {
  uintptr_t thumb_bit = _Unwind_GetGR(context, 15) & ((uintptr_t)0x1);
  _Unwind_SetGR(context, 15, value | thumb_bit);
}
```

```cpp
_LIBUNWIND_EXPORT void _Unwind_SetGR(struct _Unwind_Context *context, int index,
                                     uintptr_t value) {
  _LIBUNWIND_TRACE_API("_Unwind_SetGR(context=%p, reg=%d, value=0x%0" PRIxPTR
                       ")",
                       (void *)context, index, value);
  unw_cursor_t *cursor = (unw_cursor_t *)context;
  __unw_set_reg(cursor, index, value);
}
```

```cpp
/// Set value of specified register at cursor position in stack frame.
_LIBUNWIND_HIDDEN int __unw_set_reg(unw_cursor_t *cursor, unw_regnum_t regNum,
                                    unw_word_t value) {
  _LIBUNWIND_TRACE_API("__unw_set_reg(cursor=%p, regNum=%d, value=0x%" PRIxPTR
                       ")",
                       static_cast<void *>(cursor), regNum, value);
  typedef LocalAddressSpace::pint_t pint_t;
  AbstractUnwindCursor *co = (AbstractUnwindCursor *)cursor;
  if (co->validReg(regNum)) {
    co->setReg(regNum, (pint_t)value);
    // special case altering IP to re-find info (being called by personality
    // function)
    if (regNum == UNW_REG_IP) {
      unw_proc_info_t info;
      // First, get the FDE for the old location and then update it.
      co->getInfo(&info);
      co->setInfoBasedOnIPRegister(false);
      // If the original call expects stack adjustment, perform this now.
      // Normal frame unwinding would have included the offset already in the
      // CFA computation.
      // Note: for PA-RISC and other platforms where the stack grows up,
      // this should actually be - info.gp. LLVM doesn't currently support
      // any such platforms and Clang doesn't export a macro for them.
      if (info.gp)
        co->setReg(UNW_REG_SP, co->getReg(UNW_REG_SP) + info.gp);
    }
    return UNW_ESUCCESS;
  }
  return UNW_EBADREG;
}

template <typename A, typename R>
void UnwindCursor<A, R>::setReg(int regNum, unw_word_t value) {
  _registers.setRegister(regNum, (typename A::pint_t)value);
}
```

```cpp
inline void Registers_x86::setRegister(int regNum, uint32_t value) {
  switch (regNum) {
  case UNW_REG_IP:
    _registers.__eip = value;
    return;
  case UNW_REG_SP:
    _registers.__esp = value;
    return;
  case UNW_X86_EAX:
    _registers.__eax = value;
    return;
  case UNW_X86_ECX:
    _registers.__ecx = value;
    return;
  case UNW_X86_EDX:
    _registers.__edx = value;
    return;
  case UNW_X86_EBX:
    _registers.__ebx = value;
    return;
#if !defined(__APPLE__)
  case UNW_X86_ESP:
#else
  case UNW_X86_EBP:
#endif
    _registers.__ebp = value;
    return;
#if !defined(__APPLE__)
  case UNW_X86_EBP:
#else
  case UNW_X86_ESP:
#endif
    _registers.__esp = value;
    return;
  case UNW_X86_ESI:
    _registers.__esi = value;
    return;
  case UNW_X86_EDI:
    _registers.__edi = value;
    return;
  }
  _LIBUNWIND_ABORT("unsupported x86 register");
}
```

# phase 2仍然要从头开始进行search的原因

libunwind中两次搜索的起始位置都是相同的，结合personality和libunwind的代码我们可以来分析一下原因。

在正常搜寻到我们所需要的数据的情况下，exception中已经提前保存了第一次搜寻的结果，这里就会直接取出，不会真正的再次搜索。当第二次search到的sp和第一次保存的sp不同时第二次才会真正从头开始搜寻。

根据代码中的条件分析，search phase时在scan_eh_tab内，如果找到对应条目的actionEntry为0，则返回失败，但是clean phase actionEntry的时候可以为0。

actionEntry为0的时候表示为non-catching handler，也就是仅做clean up，意味着当没有用户的异常处理代码，只需要正常清理的时候，并不需要有action entry来清理。
