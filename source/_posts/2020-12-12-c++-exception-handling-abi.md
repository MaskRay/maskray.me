layout: post
title: C++ exception handling ABI
author: MaskRay
tags: [c++,gcc,llvm]
---

Updated in 2024-11.

[中文版](#中文版)

I wrote [an article](https://maskray.me/blog/2020-11-08-stack-unwinding) a few weeks ago to introduce stack unwinding in detail.
Today I will introduce C++ exception handling, an application of stack unwinding.
Exception handling has a variety of ABI (interoperability of C++ implementations), the most widely used of which is [_Itanium C++ ABI: Exception Handling_](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)

<!-- more -->

## Itanium C++ ABI: Exception Handling

Simplified exception handling process (from throw to catch):

* Call `__cxa_allocate_exception` to allocate space to store the exception object and the exception header `__cxa_exception`
* Jump to `__cxa_throw`, set the `__cxa_exception` fields and then jump to `_Unwind_RaiseException`
* In `_Unwind_RaiseException`, execute the search phase, call personality routines to find matching try catch (type matching)
* In `_Unwind_RaiseException`, execute the cleanup phase: call personality routines to find stack frames containing out-of-scope variables, and for each stack frame, jump to its landing pad to execute the constructors. The landing pad uses `_Unwind_Resume` to resume the cleanup phase
* The cleanup phase executed by `_Unwind_RaiseException` jumps to the landing pad corresponding to the matching try catch
* The landing pad calls `__cxa_begin_catch`, executes the catch code, and then calls `__cxa_end_catch`
* `__cxa_end_catch` decreases the handler count of the exception object, and if it reaches zero, it also destroys the exception object

Note: each stack frame may use a different personality routine. It is common that all frames share the same routine, though.

Among these steps, `_Unwind_RaiseException` is responsible for stack unwinding and is language independent.
The language-related concepts (catch block, out-of-scope variable) in stack unwinding are interpreted/encapsulated by the personality.
This is a key idea that makes the ABI applicable to other languages and allows other languages to be mixed with C++.

Therefore, Itanium C++ ABI: Exception Handling is divided into Level 1 Base ABI and Level 2 C++ ABI. Base ABI describes the language-independent stack unwinding part and defines the `_Unwind_*` API. Common implementations are:

* libgcc: `libgcc_s.so.1` and `libgcc_eh.a`
* Multiple libraries named libunwind (`libunwind.so` or `libunwind.a`). If you use Clang, you can use `--rtlib=compiler-rt --unwindlib=libunwind` to choose to link to libunwind, you can use llvm-project/libunwind or nongnu.org/libunwind

The C++ ABI is related to the C++ language and defines the `__cxa_*` API (`__cxa_allocate_exception`, `__cxa_throw`, `__cxa_begin_catch`, etc.). Common implementations are:

* libsupc++, part of libstdc++
* libc++abi in llvm-project

The C++ standard library implementation in llvm-project, libc++, can leverage libc++abi, libcxxrt or libsupc++, but libc++abi is recommended.

## Level 1 Base ABI

### Data structures

The main data structure is:

```cpp
// Level 1
struct _Unwind_Exception {
  _Unwind_Exception_Class exception_class; // an identifier, used to tell whether the exception is native
  _Unwind_Exception_Cleanup_Fn exception_cleanup;
  _Unwind_Word private_1; // zero: normal unwind; non-zero: forced unwind, the _Unwind_Stop_Fn function
  _Unwind_Word private_2; // saved stack pointer
} __attribute__((aligned));
```

```cpp
int main() {
  try {
    throw 1;
  } catch (...) {
    try {
      throw 2;
    } catch (...) {
      // The global exception stack has two exceptions here.
    }
  }
}
```

`exception_class` and `exception_cleanup` are set by the API that throws exceptions in Level 2. The Level 1 API does not process `exception_class`, but passes it to the personality routine.
Personality routines use this value to distinguish native and foreign exceptions.

> libc++abi `__cxa_throw` will set `exception_class` to uint64_t representing `"CLNGC++\0"`. libsupc++ uses uint64_t which means `"GNUCC++\0"`. The ABI requires that the lower bits contain `"C++\0"`.
> The exceptions thrown by libstdc++ will be treated as foreign exceptions by libc++abi. Only `catch (...)` can catch foreign exceptions.

> Exception propagation implementation mechanism will use another `exception_class` identifier to represent dependent exceptions.

`exception_cleanup` stores the destroying delete function of this exception object, which is used by `__cxa_end_catch` to destroy a foreign exception.

The private unwinder state (`private_1` and `private_2`) in an exception object should be neither read by nor written to by personality routines or other parts of the language-specific runtime.

The information required for the Unwind operation (for a given IP/SP, how to obtain the register information such as the IP/SP of the upper stack frame) is implementation-dependent, and Level 1 ABI does not define it.
In the ELF system, `.eh_frame` and `.eh_frame_hdr` (`PT_EH_FRAME` program header) store unwind information.
See [Stack unwinding](https://maskray.me/blog/2020-11-08-stack-unwinding).

### Level 1 API

`_Unwind_Reason_Code _Unwind_RaiseException(_Unwind_Exception *obj);` Perform stack unwinding for exceptions.
It is noreturn under normal circumstances, and will give control to matched catch handlers (catch block) or non-catch handlers (code blocks that need to execute destructors) like longjmp.
It is a two-phase process, divided into phase 1 (search phase) and phase 2 (cleanup phase).

* In the search phase, find matched catch handler and record the stack pointer in `private_2`
  + Trace the call chain based on IP/SP and other saved registers
  + For each stack frame, skip if there is no personality routine; call if there is (actions set to `_UA_SEARCH_PHASE`)
  + If personality returns `_URC_CONTINUE_UNWIND`, continue searching
  + If personality returns `_URC_HANDLER_FOUND`, it means that a matched catch handler or unmatched exception specification is found, and the search stops
* In the cleanup phase, jump to non-catch handlers (usually local variable destructors), and then transfer the control to the matched catch handler located in the search phase
  + Trace the call chain based on IP/SP and other saved registers
  + For each stack frame, skip if there is no personality routine; call if there is one (actions are set to `_UA_CLEANUP_PHASE`, and the stack frame marked by search phase will also set `_UA_HANDLER_FRAME`)
  + If personality returns `_URC_CONTINUE_UNWIND`, it means there is no landing pad, continue to unwind
  + If personality returns `_URC_INSTALL_CONTEXT`, it means there is a landing pad, jump to the landing pad
  + For intermediate stack frames that are not marked in the search phase, the landing pad performs cleanup work (usually destructors of out-of-scope variables), and calls `_Unwind_Resume` to jump back to the cleanup phase
  + For the stack frame marked by the search phase, the landing pad calls `__cxa_begin_catch`, then executes the code in the catch block, and finally calls `__cxa_end_catch` to destroy the exception object

The point of the two-phase process is to avoid any actual stack unwinding if there is no handler.
If there are just cleanup frames, an abort function can be called.
Cleanup frames are also less expensive than matching a handler.
However, parsing `.gcc_except_table` is probably not much less expensive than additionally matching a handler:)

```cpp
static _Unwind_Reason_Code unwind_phase1(unw_context_t *uc, _Unwind_Context *ctx,
                                         _Unwind_Exception *obj) {
  // Search phase: unwind and call personality with _UA_SEARCH_PHASE for each frame
  // until a handler (catch block) is found.
  unw_init_local(uc, ctx);
  for(;;) {
    if (ctx->fdeMissing) return _URC_END_OF_STACK;
    if (!step(ctx)) return _URC_FATAL_PHASE1_ERROR;
    ctx->getFdeAndCieFromIP();
    if (!ctx->personality) continue;
    switch (ctx->personality(1, _UA_SEARCH_PHASE, obj->exception_class, obj, ctx)) {
    case _URC_CONTINUE_UNWIND: break;
    case _URC_HANDLER_FOUND:
      unw_get_reg(ctx, UNW_REG_SP, &obj->private_2);
      return _URC_NO_REASON;
    default: return _URC_FATAL_PHASE1_ERROR; // e.g. stack corruption
    }
  }
  return _URC_NO_REASON;
}

static _Unwind_Reason_Code unwind_phase2(unw_context_t *uc, _Unwind_Context *ctx,
                                         _Unwind_Exception *obj) {
  // Cleanup phase: unwind and call personality with _UA_CLEANUP_PHASE for each frame
  // until reaching the handler. Restore the register state and transfer control.
  unw_init_local(uc, ctx);
  for(;;) {
    if (ctx->fdeMissing) return _URC_END_OF_STACK;
    if (!step(ctx)) return _URC_FATAL_PHASE2_ERROR;
    ctx->getFdeAndCieFromIP();
    if (!ctx->personality) continue;
    _Unwind_Action actions = _UA_CLEANUP_PHASE;
    size_t sp;
    unw_get_reg(ctx, UNW_REG_SP, &sp);
    if (sp == obj->private_2) actions |= _UA_HANDLER_FRAME;
    switch (ctx->personality(1, actions, obj->exception_class, obj, ctx)) {
    case _URC_CONTINUE_UNWIND:
      break;
    case _URC_INSTALL_CONTEXT:
      unw_resume(ctx); // Return if there is an error
      return _URC_FATAL_PHASE2_ERROR;
    default: return _URC_FATAL_PHASE2_ERROR; // Unknown result code
    }
  }
  return _URC_FATAL_PHASE2_ERROR;
}

_Unwind_Reason_Code _Unwind_RaiseException(_Unwind_Exception *obj) {
  unw_context_t uc;
  _Unwind_Context ctx;
  __unw_getcontext(&uc);
  _Unwind_Reason_Code phase1 = unwind_phase1(&uc, &ctx, obj);
  if (phase1 != _URC_NO_REASON) return phase1;
  return unwind_phase2(&uc, &ctx, obj);
}
```

> C++ does not support resumptive exception handling (correcting the exceptional condition and resuming execution at the point where it was raised), so the two-phase process is not necessary, but two-phase allows C++ and other languages to coexist on the call stack.

`_Unwind_Reason_Code _Unwind_ForcedUnwind(_Unwind_Exception *obj, _Unwind_Stop_Fn stop, void *stop_parameter);` Execute forced unwinding:
Skip the search phase and perform a slightly different cleanup phase. `private_2` is used as the parameter of the stop function.
It is similar to a foreign exception but is rarely used.

`void _Unwind_Resume(_Unwind_Exception *obj);` Continue the unwind process of phase 2. It is similar to longjmp, is noreturn, and is the only Level 1 API that is directly called by the compiler. The compiler usually calls this function at the end of non-catch handlers.

`void _Unwind_DeleteException(_Unwind_Exception *obj);` Destroy the specified exception object. It is the only Level 1 API that handles `exception_cleanup` and is called by `__cxa_end_catch`.

Many implementations provide extensions. Notably `_Unwind_Reason_Code _Unwind_Backtrace(_Unwind_Trace_Fn callback, void *ref);` is another special unwind process: it ignores personality and notifies an external callback of stack frame information.

## Level 2 C++ ABI

This part deals with language-related concepts such as throw, catch blocks, and out-of-scope variable destructors in C++.

### Data structures

Each thread has a global stack of currently caught exceptions, linked through the `nextException` field of the exception header.
`caughtExceptions` stores the most recent exception on the stack, and `__cxa_exception::nextException` points to the next exception in the stack.

```cpp
struct __cxa_eh_globals {
  __cxa_exception *caughtExceptions;
  unsigned uncaughtExceptions;
};
```

```cpp
int main() {
  try {
    throw 1;
  } catch (...) {
    try {
      throw 2;
    } catch (...) {
      // The global exception stack has two exceptions here.
    }
  }
}
```

The definition of `__cxa_exception` is as follows, and the end of it stores the `_Unwind_Exception` defined by Base ABI. `__cxa_exception` adds C++ semantic information on the basis of `_Unwind_Exception`.

```cpp
// Level 2
struct __cxa_exception {
  void *reserve; // here on 64-bit platforms
  size_t referenceCount; // here on 64-bit platforms
  std::type_info *exceptionType;
  void (*exceptionDestructor)(void *);
  unexpected_handler unexpectedHandler; // by default std::get_unexpected()
  terminate_handler terminateHandler; // by default std::get_terminate()
  __cxa_exception *nextException; // linked to the next exception on the thread stack
  int handlerCount; // incremented in __cxa_begin_catch, decremented in __cxa_end_catch, negated in __cxa_rethrow; last non-dependent performs the clean

  // The following fields cache information the catch handler found in phase 1.
  int handlerSwitchValue; // ttypeIndex in libc++abi
  const char *actionRecord;
  const char *languageSpecificData;
  void *catchTemp; // landingPad
  void *adjustedPtr; // adjusted pointer of the exception object

  _Unwind_Exception unwindHeader;
};
```

The information needed to process the exception (for a given IP, whether it is in a try catch, whether there are out-of-scope variable destructors that need to be executed, whether there is a dynamic exception specification) is called language-specific data area (LSDA), which is the implementation detail nor defined by Level 2 ABI.

### Landing pad

A landing pad is a section of code related to exceptions in the text section which performs one of the three tasks:

* In the cleanup clause, call destructors of out-of-scope variables or callbacks registered by `__attribute__((cleanup(...)))`, and then use `_Unwind_Resume` to resume cleanup phase
* A catch clause which captures the exception: call the destructors of out-of-scope variables, then call `__cxa_begin_catch`, execute the catch code, and finally call `__cxa_end_catch`
* rethrow: call destructors of out-of-scope variables in the catch clause, then call `__cxa_end_catch`, and then use `_Unwind_Resume` to resume cleanup phase

If a try block has multiple catch clauses, there will be multiple action table entries in series in the language-specific data area, but the landing pad includes all (conceptually merged) catch clauses.
Before the personality transfers control to the landing pad, it will call `_Unwind_SetGP` to set `__buitin_eh_return_data_regno(1)` to store switchValue and inform the landing pad which type matches.

A rethrow is triggered by `__cxa_rethrow` in the middle of the execution of the catch code. It needs to destruct the local variables defined by the catch clause and call `__cxa_end_catch` to offset the `__cxa_begin_catch` called at the beginning of the catch clause.

### `.gcc_except_table`

The language-specific data area on the ELF platforms is usually stored in the `.gcc_except_table` section. This section is parsed by `__gxx_personality_v0` and `__gcc_personality_v0`. Its structure is very simple:

* header (@LPStart, @TType and call sites coding, the starting offset of action records)
* call site table: Describe the landing pad offset (0 if not exists) and action record offset (biased by 1, 0 for no action) that should be executed for each call site (an address range)
* action table
* type table (referennced by postive switch values)
* dynamic exception specification (deprecated in C++, so rarely used) (referenced by negative switch values)

Here is an example:

```asm
  .section        .gcc_except_table,"a",@progbits
  .p2align        2
GCC_except_table0:
.Lexception0:
  .byte   255                      # @LPStart Encoding = omit
  .byte   3                        # @TType Encoding = udata4
  .uleb128 .Lttbase0-.Lttbaseref0  # The start of action records
.Lttbaseref0:
  .byte   1                        # Call site Encoding = uleb128
  .uleb128 .Lcst_end0-.Lcst_begin0
.Lcst_begin0:                      # 2 call site code ranges
  .uleb128 .Ltmp0-.Lfunc_begin0    # >> Call Site 1 <<
  .uleb128 .Ltmp1-.Ltmp0           #   Call between .Ltmp0 and .Ltmp1
  .uleb128 .Ltmp2-.Lfunc_begin0    #     jumps to .Ltmp2
  .byte   1                        #   On action: 1
  .uleb128 .Ltmp1-.Lfunc_begin0    # >> Call Site 2 <<
  .uleb128 .Lfunc_end0-.Ltmp1      #   Call between .Ltmp1 and .Lfunc_end0
  .byte   0                        #     has no landing pad
  .byte   0                        #   On action: cleanup
.Lcst_end0:
  .byte   1                        # >> Action Record 1 <<
                                   #   Catch TypeInfo 1
  .byte   0                                #   No further actions
  .p2align        2
                                   # >> Catch TypeInfos <<
  .long   _ZTIi                           # TypeInfo 1
```

Each call site record has two values besides call site offset and length: landing pad offset and action record offset.

* The landing pad offset is 0. The action record offset should also be 0. No landing pad
* The landing pad offset is not 0. With landing pad
  + The action record offset is 0, also called cleanup (the description of "cleanup" is somewhat ambiguous, because Level 1 has the term clean phase), usually describing local variable destructors and `__attribute__((cleanup(...)))`
  + The action record offset is not 0. The action record offset points to an action record in the action table. catch or noexcept specifier or exception specification

Each action record has two values:

* switch value (SLEB128): a positive index indicates the TypeInfo of the catch type in the type table; a negative number indicates the offset of the exception specification; 0 indicates a cleanup action which is similar to an action record offset of 0 in the call site record
* offset to the next action record: 0 indicates there is no next action record. This singly linked list form can describe multiple catches or an exception specification list

> The offset to next action record can be used not only as a singly linked list, but also as a trie, but it is rare such compression can find its usage in the wild.

The values of landing pad offset/action record offset corresponding to different areas in the program:

* A non-try block without local variable destructor: `landing_pad_offset==0 && action_record_offset==0`
* A non-try block with local variable destructors: `landing_pad_offset!=0 && action_record_offset==0`. phase 2 should stop and call cleanup
* A non-try block with `__attribute__((cleanup(...)))`: `landing_pad_offset!=0 && action_record_offset==0`. Same as above
* A try block: `landing_pad_offset!=0 && action_record_offset!=0`. The landing pad points to the code block obtained by catch splicing. Action record describes a catch for a switch value greater than 0
* A try block with `catch (...)`: Same as above. The action record is a switch value greater than 0 pointing to an entry with a value of 0 in the type table (indicating catch any)
* In a function with noexcept specifier, it is possible to propagate the exception to the caller area: `landing_pad_offset!=0 && action_record_offset!=0`. The landing pad points to the code block that calls `std::terminate`. The action record is a switch value greater than 0 pointing to an entry with a value of 0 in the type table (indicating catch any)
* In a function with an exception specifier, it may propagate the exception to the caller area: `landing_pad_offset!=0 && action_record_offset!=0`. The landing pad points to the code block that calls `__cxa_call_unexpected`. Action record is a switch value less than 0 describing an exception specifier list

### Level 2 API

`void *__cxa_allocate_exception(size_t thrown_size);`. The compiler generates a call to this function for `throw A();` and allocates a section of memory to store `__cxa_exception` and A object. `__cxa_exception` is immediately to the left of A object.
The following function illustrates the relationship between the address of the exception object operated by the program and `__cxa_exception`:
```cpp
static void *thrown_object_from_cxa_exception(__cxa_exception *exception_header) {
  return static_cast<void *>(exception_header + 1);
}
```

`void __cxa_throw(void *thrown, std::type_info *tinfo, void (*destructor)(void *));` Call the above function to find the `__cxa_exception` header, and fill in each field (`referenceCount, exception_class, unexpectedHandler, terminateHandler, exceptionType , exceptionDestructor, unwindHeader.exception_cleanup`) and then call `_Unwind_RaiseException`. This function is noreturn.

`void *__cxa_begin_catch(void *obj);` The compiler generates a call to this function at the beginning of the catch block. For a native exception,

* Add `handlerCount`
* Push the global exception stack of the thread to decrease `uncaught_exception`
* Return the adjusted pointer of the exception object

For a foreign exception (there is not necessarily a `__cxa_exception` header),

* Push if the global exception stack of the thread is empty, otherwise execute `std::terminate` (I don’t know if there is a field similar to `__cxa_exception::nextException`)
* Return `static_cast<_Unwind_Exception *>(obj) + 1` (assuming `_Unwind_Exception` is next to the thrown object)

Simplified implementation:
```cpp
void __cxa_throw(void *thrown, std::type_info *tinfo, void (*destructor)(void *)) {
  __cxa_exception *hdr = (__cxa_exception *)thrown - 1;
  hdr->exceptionType = tinfo; hdr->destructor = destructor;
  hdr->unexpectedHandler = std::get_unexpected();
  hdr->terminateHandler = std::get_terminate();
  hdr->unwindHeader.exception_class = ...;
  __cxa_get_globals()->uncaughtExceptions++;
  _Unwind_RaiseException(&hdr->unwindHeader);
  // Failed to unwind, e.g. the .eh_frame FDE is absent.
  __cxa_begin_catch(&hdr->unwindHeader);
  std::terminate();
}
```

`void __cxa_end_catch();` is called at the end of the catch block or when rethrow. For native exception:

* Get the current exception from the global exception stack of the thread, reduce `handlerCount`
* When `handlerCount` reaches 0, pop the global exception stack of the thread
* If this is a native exception: call `__cxa_free_exception` when `handlerCount` is decreased to 0 (if this is a dependent exception, decrease `referenceCount` and call `__cxa_free_exception` when it reaches 0)

For a foreign exception,

* Call `_Unwind_DeleteException`
* Execute `__cxa_eh_globals::uncaughtExceptions = nullptr;` (due to the nature of `__cxa_begin_catch`, there is exactly one exception in the stack)

`void __cxa_rethrow();` will mark the exception object, so that when `handlerCount` is reduced to 0 by `__cxa_end_catch`, it will not be destroyed, because this object will be reused by the cleanup phase restored by `_Unwind_Resume`.

Note that, except for `__cxa_begin_catch` and `__cxa_end_catch`, most `__cxa_*` functions cannot handle foreign exceptions (they do not have the `__cxa_exception` header).

### Examples

For the following code:
```cpp
#include <stdio.h>
struct A { ~A(); };
struct B { ~B(); };
void foo() { throw 0xB612; }
void bar() { B b; foo(); }
void qux() { try { A a; bar(); } catch (int x) { puts(""); } }
```

The compiled assembly conceptually looks like this:

```cpp
void foo() {
  __cxa_exception *thrown = __cxa_allocate_exception(4);
  *thrown = 42;
  __cxa_throw(thrown, &typeid(int), /*destructor=*/nullptr);
}
void bar() {
  B b; foo(); return;
  landing_pad: b.~B(); _Unwind_Resume();
}
void qux() {
  A a; bar(); return;
  landing_pad: __cxa_begin_catch(obj); puts(""); __cxa_end_catch(obj);
}
```

Control flow:

* `qux` calls `bar`. `bar` calls `foo`. `foo` throws an exception
* foo dynamically allocates a memory block, stores the thrown int and `__cxa_exception` header, and then executes `__cxa_throw`
* `__cxa_throw` fills in other fields of `__cxa_exception` and calls `_Unwind_RaiseException`

Next, `_Unwind_RaiseException` drives the two-phase process of Level 1.

* `_Unwind_RaiseException` executes phase 1: search phase
  + For `bar`, call personality with `_UA_SEARCH_PHASE` as the actions parameter and return `_URC_CONTINUE_UNWIND` (no catch handler)
  + For `qux`, call personality with `_UA_SEARCH_PHASE` as the actions parameter and return `_URC_HANDLER_FOUND` (with catch handler)
  + The stack pointer of the stack frame that marked qux will be marked (stored in `private_2`) and the search will stop
* `_Unwind_RaiseException` executes phase 2: cleanup phase
  + `bar`'s stack frame is not marked by search phase, call personality with `_UA_CLEANUP_PHASE` as actions parameter, return `_URC_INSTALL_CONTEXT`
  + Jump to the landing pad of the bar's stack frame
  + After cleaning the landing pad, use `_Unwind_Resume` to return to the cleanup phase
  + The stack frame of qux is marked by search phase, call personality with `_UA_CLEANUP_PHASE|_UA_HANDLER_FRAME` as the actions parameter, and return `_UA_INSTALL_CONTEXT`
  + Jump to the landing pad of the qux stack frame
  + The landing pad calls `__cxa_begin_catch`, executes the catch code, and then calls `__cxa_end_catch`

### `__gxx_personality_v0`

A personality routine is called by Level 1 ABI (both phase 1 and phase 2) to provide language-related processing.
Different languages, implementations or architectures may use different personality routines. Common personalities are as follows:

* `__gxx_personality_v0`: C++
* `__gxx_personality_sj0`: sjlj
* `__gcc_personality_v0`: C `-fexceptions` for `__attribute__((cleanup(...)))`
* `__CxxFrameHandler3`: Windows MSVC
* `__gxx_personality_seh0`: MinGW-w64 `-fseh-exceptions`
* `__objc_personality_v0`: ObjC in the macOS environment

The most common C++ implementation on ELF systems is `__gxx_personality_v0`. It is implemented by:

* GCC: `libstdc++-v3/libsupc++/eh_personality.cc`
* libc++abi: `src/cxa_personality.cpp`

`_Unwind_Reason_Code (*__personality_routine)(int version, _Unwind_Action action, uint64 exceptionClass, _Unwind_Exception *exceptionObject, _Unwind_Context *context);`

In the absence of errors:

* For `_UA_SEARCH_PHASE`, returns
  + `_URC_CONTINUE_UNWIND`: no lsda, or there is no landing pad, there is a non-catch handler or a matched exception specification
  + `_URC_HANDLER_FOUND`: there is a matched catch handler or an unmatched exception specification
* For `_UA_CLEANUP_PHASE`, returns
  + `_URC_CONTINUE_UNWIND`: no lsda, or there is no landing pad, or (not produced by a compiler) there is no cleanup action
  + `_URC_INSTALL_CONTEXT`: the other cases

Before transferring control to the landing pad, the personality will call `_Unwind_SetGP` to set two registers (architecture related, `__buitin_eh_return_data_regno(0)` and `__buitin_eh_return_data_regno(1)`) to store `_Unwind_Exception *` and `switchValue`.

Code:

```cpp
_unwind_Reason_Code __gxx_personality_v0(int version, _Unwind_Action actions, uint64_t exceptionClass, _Unwind_Exception *exc, _Unwind_Context *ctx) {
  scan_results results;
  if (actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME) && is_native) {
    auto *hdr = (__cxa_exception *)(exc+1) - 1;
    // Load cached results from phase 1.
    results.switchValue = hdr->handlerSwitchValue;
    results.actionRecord = hdr->actionRecord;
    results.languageSpecificData = hdr->languageSpecificData;
    results.landingPad = reinterpret_cast<uintptr_t>(hdr->catchTemp);
    results.adjustedPtr = hdr->adjustedPtr;

    _Unwind_SetGR(...);
    _Unwind_SetGR(...);
    _Unwind_SetIP(ctx, results.landingPad);
    return _URC_INSTALL_CONTEXT;
  }
  scan_eh_tab(results, actions, native_exception, unwind_exception, context);
  if (results.reason == _URC_CONTINUE_UNWIND ||
      results.reason == _URC_FATAL_PHASE1_ERROR)
    return results.reason;
  if (actions & _UA_SEARCH_PHASE) {
    auto *hdr = (__cxa_exception *)(exc+1) - 1;
    // Cache LSDA results in hdr.
    hdr->handlerSwitchValue = results.switchValue;
    hdr->actionRecord = results.actionRecord;
    hdr->languageSpecificData = results.languageSpecificData;
    hdr->catchTemp = reinterpret_cast<void *>(results.landingPad);
    hdr->adjustedPtr = results.adjustedPtr;
    return _URC_HANDLER_FOUND;
  }
  // _UA_CLEANUP_PHASE
  _Unwind_SetGR(...);
  _Unwind_SetGR(...);
  _Unwind_SetIP(ctx, results.landingPad);
  return _URC_INSTALL_CONTEXT;
}
```

For a native exception, when the personality returns `_URC_HANDLER_FOUND` in the search phase, the LSDA related information of the stack frame will be cached.
When the personality is called again in the cleanup phase with the argument `actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME)`, the personality loads the cache and there is no need to parse `.gcc_except_table`.

In the remaining three cases, the personality has to parse `.gcc_except_table`:

* `actions & _UA_SEARCH_PHASE`
* `actions & _UA_CLEANUP_PHASE && actions & _UA_HANDLER_FRAME && !is_native`: `catch (...)` can catch a foreign exception. An exception specification terminates upon a foreign exception.
* `actions & _UA_CLEANUP_PHASE && !(actions & _UA_HANDLER_FRAME)`: non-catch handlers and unmatched catch handlers, matched exception specification. Another case is `_Unwind_ForcedUnwind`.

```cpp
static void scan_eh_tab(...) {
  ...
  const uint8_t *lsda = (const uint8_t *)_Unwind_GetLanguageSpecificData(context);
  if (lsda == nullptr) { res.reason = _URC_CONTINUE_UNWIND; return; }
  res.languageSpecificData = lsda;
  uintptr_t ipOffset = _Unwind_GetIP(context) - 1 - _Unwind_GetRegionStart(context);
  for each call site entry {
    if (!(start <= ipOffset && ipOffset < start + length))
      continue;
    res.landingPad = landingPad;
    if (landingPad == 0) { res.reason = _URC_CONTINUE_UNWIND; return; }
    if (actionRecord == 0) { // cleanup
      res.reason = actions & _UA_SEARCH_PHASE ? _URC_CONTINUE_UNWIND : _URC_HANDLER_FOUND;
      return;
    }
    // A catch or a dynamic exception specification.
    const uint8_t *action = actionTableStart + (actionRecord - 1);
    bool hasCleanup = false;
    for(;;) {
      res.actionRecord = action;
      int64_t switchValue = readSLEB128(&action);
      if (switchValue > 0) { // catch
        auto *catchType = ...;
        if (catchType == nullptr) { // catch (...)
          res.switchValue = switchValue; res.actionRecord = action;
          res.adjustedPtr = getThrownObjectPtr(exc); res.reason = _URC_HANDLER_FOUND;
          return;
        } else if (is_native) { // catch (T ...)
          auto *hdr = (__cxa_exception *)(exc+1) - 1;
          if (catchType->can_catch(hdr->exceptionType, adjustedPtr)) {
            res.switchValue = switchValue; res.actionRecord = action;
            res.adjustedPtr = adjustedPtr; res.reason = _URC_HANDLER_FOUND;
            return;
          }
        }
      } else if (switchValue < 0) { // dynamic exception specification
        if (actions & _UA_FORCE_UNWIND) {
          // Skip if forced unwinding.
        } else if (is_native) {
          if (!exception_spec_can_catch) {
            // The landing pad will call __cxa_call_unexpected.
            assert(actions & _UA_SEARCH_PHASE);
            res.switchValue = switchValue; res.actionRecord = action;
            res.adjustedPtr = adjustedPtr; res.reason = _URC_HANDLER_FOUND;
            return;
          }
        } else {
          // A foreign exception cannot be matched by the exception specification. The landing pad will call __cxa_call_unexpected.
          res.switchValue = switchValue; res.actionRecord = action;
          res.adjustedPtr = getThrownObjectPtr(exc); res.reason = _URC_HANDLER_FOUND;
          return;
        }
      } else { // switchValue == 0: cleanup
        hasCleanup = true;
      }
      const uint8_t *temp = action;
      int64_t actionOffset = readSLEB128(&temp);
      if (actionOffset == 0) { // End of action list
        res.reason = hasCleanup && actions & _UA_CLEANUP_PHASE
          ? _URC_HANDLER_FOUND : _URC_CONTINUE_UNWIND;
        return;
      }
      action += actionOffset;
    }
  }
  call_terminate();
}
```

### `__gcc_personality_v0`

libgcc and compiler-rt/lib/builtins implement this function to handle `__attribute__((cleanup(...)))`.
The implementation does not return `_URC_HANDLER_FOUND` in the search phase, so the cleanup handler cannot serve as a catch handler.
However, we can supply our own implementation to return `_URC_HANDLER_FOUND` in the search phase...
On x86-64, `__buitin_eh_return_data_regno(0)` is RAX. We can let the cleanup handler pass RAX to the landing pad.

```cpp
// a.cc
#include <exception>
#include <stdio.h>

extern "C" void my_catch();
extern "C" void throw_exception() { throw 42; }

int main() {
  fprintf(stderr, "uncaught exceptions: %d\n", std::uncaught_exceptions());
  my_catch();
  fprintf(stderr, "uncaught exceptions: %d\n", std::uncaught_exceptions());
}

// b.c
#include <setjmp.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <unwind.h>

void throw_exception();

struct __cxa_eh_globals {
  struct __cxa_exception *caughtExceptions;
  unsigned uncaughtExceptions;
};

struct __cxa_eh_globals *__cxa_get_globals();

static uintptr_t readULEB128(const uint8_t **a) {
  uintptr_t res = 0, shift = 0;
  const uint8_t *p = *a;
  uint8_t b;
  do {
    b = *p++;
    res |= (b & 0x7f) << shift;
    shift += 7;
  } while (b & 0x80);
  *a = p;
  return res;
}

_Unwind_Reason_Code __gcc_personality_v0(int version, _Unwind_Action actions,
                                         uint64_t exception_class,
                                         struct _Unwind_Exception *obj,
                                         struct _Unwind_Context *ctx) {
  const uint8_t *lsda = _Unwind_GetLanguageSpecificData(ctx);
  if (lsda == 0)
    return _URC_CONTINUE_UNWIND;
  uintptr_t func = _Unwind_GetRegionStart(ctx);
  uintptr_t pc = _Unwind_GetIP(ctx) - 1 - func;
  if (*lsda++ != 255) // Skip LPStart
    readULEB128(&lsda);
  if (*lsda++ != 255) // Skip TType
    readULEB128(&lsda);
  uintptr_t call_site_table_len = 0;
  if (*lsda++ == 1)
    call_site_table_len = readULEB128(&lsda);
  const uint8_t *end = lsda + call_site_table_len;
  while (lsda < end) {
    uintptr_t start = readULEB128(&lsda), len = readULEB128(&lsda),
              lpad = readULEB128(&lsda);
    if (!(start <= pc && pc < start + len))
      continue;
    if (lpad == 0)
      return _URC_CONTINUE_UNWIND;
    if (actions & _UA_SEARCH_PHASE)
      return _URC_HANDLER_FOUND;
    _Unwind_SetGR(ctx, __builtin_eh_return_data_regno(0), (uintptr_t)obj);
    _Unwind_SetGR(ctx, __builtin_eh_return_data_regno(1), 0); // switchValue==0
    _Unwind_SetIP(ctx, func + lpad);
    return _URC_INSTALL_CONTEXT;
  }
  return _URC_FATAL_PHASE2_ERROR;
}

struct Catch {
  struct _Unwind_Exception *obj;
  jmp_buf env;
  bool do_catch;
};

__attribute__((used))
static void my_jump(struct Catch *c) {
  if (c->do_catch) {
    struct __cxa_eh_globals *globals = __cxa_get_globals();
    globals->uncaughtExceptions--;
    longjmp(c->env, 1);
  }
}

__attribute__((naked)) static void my_cleanup(struct Catch *c) {
  asm("movq %rax, (%rdi); jmp my_jump");
}

void my_catch() {
  __attribute__((cleanup(my_cleanup))) struct Catch c;
  if (setjmp(c.env) == 0) {
    c.do_catch = 1;
    throw_exception();
  } else {
    fprintf(stderr, "caught exception: %p\n", c.obj);
    fprintf(stderr, "value: %d\n", *(int *)(c.obj + 1));
    c.do_catch = 0;
  }
}
```

```text
% clang -c -fexceptions a.cc b.c
% clang++ a.o b.o
% ./a.out
uncaught exceptions: 0
caught exception: 0x10f7f10
value: 42
uncaught exceptions: 0
```

### Rethrow

The landing pad section briefly described the code executed by rethrow. Usually caught exception will be destroyed in `__cxa_end_catch`, so `__cxa_rethrow` will mark the exception object and increase `handlerCount`.

C++11 introduced Exception Propagation (N2179; `std::rethrow_exception` etc), and libstdc++ uses `__cxa_dependent_exception` to achieve.
For design see <https://gcc.gnu.org/legacy-ml/libstdc++/2008-05/msg00079.html>

```cpp
struct __cxa_dependent_exception {
  void *reserve;
  void *primaryException;
};
```

`std::current_exception` and `std::rethrow_exception` will increase the reference count.

In libstdc++, `__cxa_rethrow` calls GCC extension `_Unwind_Resume_or_Rethrow` which can resume forced unwinding.

### LLVM IR

In construction.

* nounwind: cannot unwind
* unwtables: force generation of the unwind table regardless of nounwind

```
if uwtables
  if nounwind
    CantUnwind
  else
    Unwind Table
else
  do nothing
```

## Compiler behavior

* `-fno-exceptions -fno-asynchronous-unwind-tables`: neither `.eh_frame` nor `.gcc_except_table` exists
* `-fno-exceptions -fasynchronous-unwind-tables`: `.eh_frame` exists, `.gcc_except_table` doesn't
* `-fexceptions`: both `.eh_frame` and `.gcc_except_table` exist
  + In GCC, for a `noexcept` function, a possibly-throwing call site unhandled by a try block does not get an entry in the `.gcc_except_table` call site table. If the function has no try block, it gets a header-only `.gcc_except_table` (4 bytes)
  + In Clang, there is a call site entry calling `__clang_call_terminate`. The size overhead is larger than GCC's scheme. Improving this requires LLVM IR work

When an exception propagates from a function to its caller (libgcc_s/libunwind & libsupc++/libc++abi):

* no `.eh_frame`: `_Unwind_RaiseException` returns `_URC_END_OF_STACK`. `__cxa_throw` calls `std::terminate`
* `.eh_frame` without `.gcc_except_table`: pass-through (local variable destructors are not called). This is the case of `-fno-exceptions -fasynchronous-unwind-tables`.
* `.eh_frame` with `.gcc_except_table` not covering the throwing call site: `__gxx_personality_v0` calls `std::terminate` since no call site code range matches
* `.eh_frame` with `.gcc_except_table` covering the throwing call site: do possible cleanup and unwind to the parent frame

Combined with the above description, when an exception will propagate to a caller of a noexcept function:

* `-fno-exceptions -fno-asynchronous-unwind-tables`: propagating through a function calls `std::terminate`
* `-fno-exceptions -fasynchronous-unwind-tables`: pass-through. Local variable destructors are not called. This behavior is unexpected.
* `-fexceptions`: propagating through a `noexcept` function calls `std::terminate`

When `std::terminate` is called, there is a diagnostic looking like `terminate called after throwing an instance of 'int'` (libstdc++; libc++ has a smiliar one).
There is no stack trace.
If the process installs a `SIGABRT` signal handler, the handler may get a stack trace and symbolize the addresses.

[Catching exceptions while unwinding through -fno-exceptions code](https://discourse.llvm.org/t/catching-exceptions-while-unwinding-through-fno-exceptions-code/57151) is a proposal to improve the diagnostics.

### Personality and typeinfo encoding

`.eh_frame` contains information about the unwind operation.
See [Stack unwinding](https://maskray.me/blog/2020-11-08-stack-unwinding) for its format.

In `-fpie/-fpic` mode, the personality and type info encodings have the `DW_EH_PE_indirect|DW_EH_PE_pcrel` bits on most targets.
```cc
void raise() { throw 42; }
bool foo() {
  try { raise(); } catch (int) { return true; }
  return false;
}
int main() { foo(); }
```
```asm
_Z3foov:
    .cfi_startproc
    .cfi_personality 155, DW.ref.__gxx_personality_v0
    .cfi_lsda 27, .Lexception0
...

	.section	.gcc_except_table,"a",@progbits
...
                                        # >> Catch TypeInfos <<
.Ltmp3:                                 # TypeInfo 1
	.long	.L_ZTIi.DW.stub-.Ltmp3
.Lttbase0:

	.data
	.p2align	3, 0x0
.L_ZTIi.DW.stub:
	.quad	_ZTIi
	.hidden	DW.ref.__gxx_personality_v0
	.weak	DW.ref.__gxx_personality_v0
	.section	.data.DW.ref.__gxx_personality_v0,"aGw",@progbits,DW.ref.__gxx_personality_v0,comdat
	.p2align	3, 0x0
	.type	DW.ref.__gxx_personality_v0,@object
	.size	DW.ref.__gxx_personality_v0, 8
DW.ref.__gxx_personality_v0:
	.quad	__gxx_personality_v0
```

In the example, `.eh_frame` contains a PC-relative relocations referencing `DW.ref.__gxx_personality_v0` 
`.gcc_except_table` contains a PC-relative relocation referencing `.L_ZTIi.DW.stub`.
The relocations are link-time constants, so `.eh_frame` can remain readonly.

`DW.ref.__gxx_personality_v0` and `.L_ZTIi.DW.stub` reside in writable sections which will contain dynamic relocations if `__gxx_personality_v0` and `_ZTIi` are defined in a shared object - which is often the case.

For `-fno-pic` code, different targets have different ideas.
AArch64 and RISC-V use `DW_EH_PE_indirect|DW_EH_PE_pcrel` as well.
On x86, `.cfi_personality` refers to `__gxx_personality_v0`. This will lead to a canonical PLT if `__gxx_personality_v0` is defined in a shared object (e.g. `libstdc++.so.6`).
I sent a patch <https://gcc.gnu.org/PR108622> to use `DW_EH_PE_indirect|DW_EH_PE_pcrel`.

### `R_MIPS_32` and `R_MIPS_64` personality encoding

<https://github.com/llvm/llvm-project/issues/58377>

```cpp
void foo() { try { throw 1; } catch (...) {} }
```

`mips64el-linux-gnuabi64-g++ -fpic` and `clang++ --target=mips64el-unknown-linux-gnuabi64 -fpic` use `DW_EH_PE_absptr | DW_EH_PE_indirect` to encode personality routine pointers.
Using `DW_EH_PE_absptr` instead of `DW_EH_PE_pcrel` is wrong.
GNU ld works around the compiler design problem by converting `DW_EH_PE_absptr` to `DW_EH_PE_pcrel`.
ld.lld does not support this and will report an error:
```sh
% clang++ --target=mips64el-linux-gnuabi -fpic -fuse-ld=lld -shared ex.cc
ld.lld: error: relocation R_MIPS_64 cannot be used against symbol 'DW.ref.__gxx_personality_v0'; recompile with -fPIC
>>> defined in /tmp/ex-40a996.o
>>> referenced by ex.cc
>>>               /tmp/ex-40a996.o:(.eh_frame+0x13)
...
```

`R_MIPS_32` for 32-bit builds is similar.

### Potentially-throwing `__cxa_end_catch`

`__cxa_end_catch` is potentially-throwing because it may destroy an exception object with a potentially-throwing destructor (e.g. `~C() noexcept(false) { ... }`).
```cpp
struct A { ~A(); };
void opaque();
void foo() {
  A a;
  // The exception object has an unknown type and may throw. The landing pad
  // then needs to call A::~A for `a` before jumping to _Unwind_Resume.
  try { opaque(); } catch (...) { }
}
```

To support an exception object with a potentially-throwing destructor, Clang generates conservative code for a catch-all clause or a catch clause matching a record type:

* assume that the exception object may have a throwing destructor
* emit `invoke void @__cxa_end_catch` (as the call is not marked as the `nounwind` attribute).
* emit a landing pad to destroy local variables and call `_Unwind_Resume`

Per C++ [dcl.fct.def.coroutine], a coroutine's function body implies a `catch (...)`.
Clang's code generation pessimizes even simple code, like:
```
UserFacing foo() {
  A a;
  opaque();
  co_return;
  // For `invoke void @__cxa_end_catch()`, the landing pad destroys the
  // promise_type and deletes the coro frame.
}
```

Throwing destructors are typically discouraged. In many environments, the destructors of exception objects are guaranteed to never throw, making our conservative code generation approach seem wasteful.

Furthermore, throwing destructors tend not to work well in practice:

* GCC does not emit call site records for the region containing `__cxa_end_catch`. This has been a long time, since 2000.
* If a catch-all clause catches an exception object that throws, both GCC and Clang using libstdc++ leak the allocated exception object.

To avoid code generation pessimization, I added [`-fassume-nothrow-exception-dtor`](https://reviews.llvm.org/D108905) for Clang 18 to assume that `__cxa_end_catch` calls have the `nounwind` attribute.
This requires that thrown exception objects' destructors will never throw.

To detect misuses, diagnose throw expressions with a potentially-throwing destructor. Technically, it is possible that a potentially-throwing destructor never throws when called transitively by `__cxa_end_catch`, but these cases seem rare enough to justify a relaxed mode.

## Misc

### Use libc++ and libc++abi

On Linux, compared with `clang`, `clang++` additionally links against libstdc++/libc++ and libm.

Dynamically link against libc++.so (which depends on libc++abi.so) (additionally specify `-pthread` if threads are used):

```sh
clang++ -stdlib=libc++ -nostdlib++ a.cc -lc++ -lc++abi
# clang -stdlib=libc++ a.cc -lc++ -lc++abi does not pass -lm to the linker.
```

If compile actions and link actions are separate (`-stdlib=libc++` passes `-lc++` but its position is undesired, so just don't use it):

```sh
clang++ -nostdlib++ a.cc -lc++ -lc++abi
```

Statically link in libc++.a (which includes the members of libc++abi.a). This requires a `-DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=on` build:

```sh
clang++ -stdlib=libc++ -static-libstdc++ -nostdlib++ a.cc -pthread
```

Statically link in libc++.a and libc++abi.a. This is a bit inferior because there is a duplicate -lc++ passed by the driver.

```sh
clang++ -stdlib=libc++ -static-libstdc++ -nostdlib++ a.cc -Wl,--push-state,-Bstatic -lc++ -lc++abi -Wl,--pop-state -pthread
```

### libc++abi and libsupc++

It is worth noting that the `<exception> <stdexcept>` type layout provided by libc++abi (such as `logic_error`, `runtime_error`, etc.) are specifically compatible with libsupc++.
After GCC 5 libstdc++ abandoned ref-counted `std::string`, libsupc++ still uses `__cow_string` for `logic_error` and other exception classes. libc++abi uses a similar ref-counted string.

libsupc++ and libc++abi do not use inline namespace and have conflicting symbol names. Therefore, usually a libc++/libc++abi application cannot use a shared object (ODR violation) of a dynamically linked `libstdc++.so`.

If you make some efforts, you can still solve this problem: compile the non-libsupc++ part of libstdc++ to get self-made `libstdc++.so.6`. The executable file link libc++abi provides the C++ ABI symbols required by `libstdc++.so.6`.

### Monolithic `.gcc_except_table`

Prior to Clang 12, a monolithic `.gcc_except_table` was used. Like many other metadata sections, the main problem with the monolithic sections is that they cannot be garbage collected by the linker.
For RISC-V `-mrelax` and basic block sections, there is a bigger problem: `.gcc_except_table` has relocations pointing to text sections local symbols.
If the pointed text sections are discarded in the COMDAT group, these relocations will be rejected by the linker (`error: relocation refers to a symbol in a discarded section`).
![.eh_frame with monolithic .gcc_except_table](/static/2020-12-12-c++-exception-handling/eh_frame_and_monolithic_gcc_except_table.svg)
![monolithic .gcc_except_table](/static/2020-12-12-c++-exception-handling/monolithic_gcc_except_table.svg){ width=70%}

The solution is to use fragmented `.gcc_except_table`(<https://reviews.llvm.org/D83655>).
![fragmented .gcc_except_table](/static/2020-12-12-c++-exception-handling/fragmented_gcc_except_table.svg){ width=70%}

But the actual deployment is not that simple:) ld.lld processes `--gc-sections` first (it is not clear which `.eh_frame` pieces are live), and then processes (and garbage collects) `.eh_frame`.

During `--gc-sections`, all `.eh_frame` pieces are live. They will mark all `.gcc_except_table.*` live.
According to the GC rules of the section group, a `.gcc_except_table.*` will mark other sections (including `.text.*`) live in the same section group.
The result is that `.text.*` in all section groups cannot be GC, resulting in increased input size.
![bad GC with .gcc_except_table.*](/static/2020-12-12-c++-exception-handling/lsda_gc.svg){ width=70%}

<https://reviews.llvm.org/D91579> fixed this problem: For `.eh_frame`, do not mark `.gcc_except_table` in section group.
![good GC with .gcc_except_table.*](/static/2020-12-12-c++-exception-handling/lsda_gc_new.svg){ width=70%}

### `clang -fbasic-block-sections=`

This option produces one section for each basic block (more aggressive than `-ffunction-sections`) for aggressive machine basic block optimizations.
There are some challenges integrating LSDA into this framework.

You can either allocate a `.gcc_except_table` for each basic block section needing LSDA, or let all basic block sections use the same `.gcc_except_table`.
The LLVM implementation chose the latter, which has several advantages:

* No duplicate headers
* Sharable type table
* Sharable action table (this only matters for the deprecated exception specification)

There is only one LPStart when using the same `.gcc_except_table`, and it is necessary to ensure that all offsets from landing pads to LPStart can be represented by relocations.
Because most architectures do not have a difference relocation type (`R_RISCV_SUB*`), placing landing pads in the same section is the choice.

## Exception handling ABI for the ARM architecture

The overall structure is the same as Itanium C++ ABI: Exception Handling, with some differences in data structure, `_Unwind_*`, etc.

<https://maskray.me/blog/2020-11-08-stack-unwinding> contains a few notes.

## Compact Exception Tables for MIPS ABIs

In construction.

Use `.eh_frame_entry` and `.gnu_extab` to describe.

Design thoughts:

* Exception code ranges are sorted and must be linearly searched. Therefore it would be more compact to specify each relative to the previous one, rather than relative to a fixed base.
* The landing pad is often close to the exception region that uses it. Therefore it is better to use the end of the exception region as the reference point, than use the function base address.
* The action table can be integrated directly with the exception region definition itself. This removes one indirection. The threading of actions can still occur, by providing an offset to the next exception encoding of interest.
* Often the action threading is to the next exception region, so optimizing that case is important.
* Catch types and exception specification type lists cannot easily be encoded inline with the exception regions themselves. It is necessary to preserve the unique indices that are automatically created by the DWARF scheme.

It uses compact unwind descriptors similar to ARM EH. Builtin PR1 means there is no language-dependent data, Builtin PR2 is used for C/C++

## Misc

Khalil Estell's CppCon 2024 talk _C++ Exceptions for Smaller Firmware_ mentions that a custom exception implementation that drops some rare functionality can make the library code size mush smaller, suitable for firmware development.



# 中文版

幾周前寫了[一篇文章](https://maskray.me/blog/2020-11-08-stack-unwinding)詳細介紹stack unwinding。
今天介紹C++ exception handling，stack unwinding的一個應用。Exception handling有多種ABI(interoperability of C++ implementations)，其中應用最廣泛的是[_Itanium C++ ABI: Exception Handling_](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)

<!-- more -->

## Itanium C++ ABI: Exception Handling

簡化的exception處理流程(從throw到catch)：

* 調用`__cxa_allocate_exception`分配空間存放exception object和exception header `__cxa_exception`
* 跳轉到`__cxa_throw`，設置`__cxa_exception`字段後跳轉到`_Unwind_RaiseException`
* `_Unwind_RaiseException`執行search phase，調用personality查找匹配的try catch(類型匹配)
* `_Unwind_RaiseException`執行cleanup phase：調用personality查找包含out-of-scope變量的stack frames，對於每個stack frame，跳轉到其landing pad執行destructors。該landing pad用`_Unwind_Resume`跳轉回cleanup phase
* `_Unwind_RaiseException`執行的cleanup phase跳轉到匹配的try catch對應的landing pad
* 該landing pad調用`__cxa_begin_catch`，執行catch代碼，然後調用`__cxa_end_catch`
* `__cxa_end_catch`銷毀exception object

注意：每個棧幀的personality routine可以不同。實踐中多個棧幀使用同一個personality routine是很常見的。

其中`_Unwind_RaiseException`負責stack unwinding，是語言無關的。而stack unwinding中的語言相關概念(catch block、out-of-scope variable)用personality解釋/封裝。
這是一個核心思想，使得該ABI可以應用與其他語言並允許其他語言和C++混用。

因此，Itanium C++ ABI: Exception Handling分成Level 1 Base ABI and Level 2 C++ ABI兩部分。Base ABI描述了語言無關的stack unwinding部分，定義了`_Unwind_*` API。常見實現是：

* libgcc: `libgcc_s.so.1` and `libgcc_eh.a`
* 多個名稱爲libunwind的庫(`libunwind.so`或`libunwind.a`)。使用Clang的話可以用`--rtlib=compiler-rt --unwindlib=libunwind`選擇鏈接libunwind，可以用llvm-project/libunwind或nongnu.org/libunwind

C++ ABI則和C++語言相關，定義了`__cxa_*` API(`__cxa_allocate_exception`, `__cxa_throw`, `__cxa_begin_catch`等)。常見實現是：

* libsupc++，libstdc++的一部分
* llvm-project中的libc++abi

llvm-project中的C++標準庫實現libc++可以接入libc++abi、libcxxrt或libsupc++，推薦使用libc++abi。

## Level 1 Base ABI

### Data structures

主要數據結構是：

```cpp
// Level 1
struct _Unwind_Exception {
  _Unwind_Exception_Class exception_class; // an identifier, used to tell whether the exception is native
  _Unwind_Exception_Cleanup_Fn exception_cleanup;
  _Unwind_Word private_1; // zero: normal unwind; non-zero: forced unwind, the _Unwind_Stop_Fn function
  _Unwind_Word private_2; // saved stack pointer
} __attribute__((aligned));
```

```cpp
int main() {
  try {
    throw 1;
  } catch (...) {
    try {
      throw 2;
    } catch (...) {
      // The global exception stack has two exceptions here.
    }
  }
}
```

`exception_class`和`exception_cleanup`是Level 2拋出exception的API設置的。Level 1 API不處理`exception_class`，只是把它傳遞給personality routine。Personality routine用該值區分native和foreign exceptions。

> libc++abi `__cxa_throw`會設置`exception_class`爲表示`"CLNGC++\0"`的uint64_t。libsupc++則使用表示`"GNUCC++\0"`的uint64_t。ABI要求低位包含`"C++\0"`。
> libstdc++拋出的exceptions會被libc++abi當作foreign exceptions。只有`catch (...)`可以捕獲foreign exceptions。

> Exception propagation實現機制會用另一個`exception_class`標識符來表示dependent exceptions。

`exception_cleanup`存放這個exception object的destroying delete函數，被`__cxa_end_catch`用來銷毀一個foreign exception。

`private_1`和`private_2`是Level 1私有的，不應被personality使用。

Unwind操作需要的信息(對於給定的IP/SP，如何獲取上一層棧幀的IP/SP等寄存器信息)是實現相關的，Level 1 ABI沒有定義。
在ELF系統裏，`.eh_frame`和`.eh_frame_hdr`(`PT_EH_FRAME` program header)存儲unwind信息。
參見[Stack unwinding](https://maskray.me/blog/2020-11-08-stack-unwinding)。

### Level 1 API

`_Unwind_Reason_Code _Unwind_RaiseException(_Unwind_Exception *obj);`執行用於exception的stack unwinding。
它正常情況下是noreturn的，會像longjmp那樣把控制權交給matched catch handler(catch block)或non-catch handlers(需要執行destructors的代碼塊)。
它是個two-phase process，分爲phase 1 (search phase)和phase 2 (cleanup phase)。

* search phase查找matched catch handler，把stack pointer記錄在`private_2`中
  + 根據IP/SP及其他保存的寄存器追溯調用鏈
  + 對於每個棧幀，如果沒有personality routine則跳過；有則調用(actions設置爲`_UA_SEARCH_PHASE`)
  + 若personality返回`_URC_CONTINUE_UNWIND`，繼續搜索
  + 若personality返回`_URC_HANDLER_FOUND`，表示找到了一個matched catch handler or unmatched exception specification，停止搜索
* cleanup phase跳轉到non-catch handlers(通常是local variable destructors)，再把控制權交給phase 1定位的matched catch handler
  + 根據IP/SP及其他保存的寄存器追溯調用鏈
  + 對於每個棧幀，如果沒有personality routine則跳過；有則調用(actions設置爲`_UA_CLEANUP_PHASE`，search phase標記的棧幀還會設置`_UA_HANDLER_FRAME`)
  + 若personality返回`_URC_CONTINUE_UNWIND`，表示沒有landing pad，繼續unwind
  + 若personality返回`_URC_INSTALL_CONTEXT`，表示有landing pad，跳轉到landing pad
  + 對於search phase沒有標記的中間棧幀，landing pad執行清理工作(一般是destructors of out-of-scope variables)，會調用`_Unwind_Resume`跳轉回cleanup phase
  + 對於被search phase標記的棧幀，landing pad調用`__cxa_begin_catch`，然後執行catch block中的代碼，最後調用`__cxa_end_catch`銷毀exception object

```cpp
static _Unwind_Reason_Code unwind_phase1(unw_context_t *uc, _Unwind_Context *ctx,
                                         _Unwind_Exception *obj) {
  // Search phase: unwind and call personality with _UA_SEARCH_PHASE for each frame
  // until a handler (catch block) is found.
  unw_init_local(uc, ctx);
  for(;;) {
    if (ctx->fdeMissing) return _URC_END_OF_STACK;
    if (!step(ctx)) return _URC_FATAL_PHASE1_ERROR;
    ctx->getFdeAndCieFromIP();
    if (!ctx->personality) continue;
    switch (ctx->personality(1, _UA_SEARCH_PHASE, obj->exception_class, obj, ctx)) {
    case _URC_CONTINUE_UNWIND: break;
    case _URC_HANDLER_FOUND:
      unw_get_reg(ctx, UNW_REG_SP, &obj->private_2);
      return _URC_NO_REASON;
    default: return _URC_FATAL_PHASE1_ERROR; // e.g. stack corruption
    }
  }
  return _URC_NO_REASON;
}

static _Unwind_Reason_Code unwind_phase2(unw_context_t *uc, _Unwind_Context *ctx,
                                         _Unwind_Exception *obj) {
  // Cleanup phase: unwind and call personality with _UA_CLEANUP_PHASE for each frame
  // until reaching the handler. Restore the register state and transfer control.
  unw_init_local(uc, ctx);
  for(;;) {
    if (ctx->fdeMissing) return _URC_END_OF_STACK;
    if (!step(ctx)) return _URC_FATAL_PHASE2_ERROR;
    ctx->getFdeAndCieFromIP();
    if (!ctx->personality) continue;
    _Unwind_Action actions = _UA_CLEANUP_PHASE;
    size_t sp;
    unw_get_reg(ctx, UNW_REG_SP, &sp);
    if (sp == obj->private_2) actions |= _UA_HANDLER_FRAME;
    switch (ctx->personality(1, actions, obj->exception_class, obj, ctx)) {
    case _URC_CONTINUE_UNWIND:
      break;
    case _URC_INSTALL_CONTEXT:
      unw_resume(ctx); // Return if there is an error
      return _URC_FATAL_PHASE2_ERROR;
    default: return _URC_FATAL_PHASE2_ERROR; // Unknown result code
    }
  }
  return _URC_FATAL_PHASE2_ERROR;
}

_Unwind_Reason_Code _Unwind_RaiseException(_Unwind_Exception *obj) {
  unw_context_t uc;
  _Unwind_Context ctx;
  __unw_getcontext(&uc);
  _Unwind_Reason_Code phase1 = unwind_phase1(&uc, &ctx, obj);
  if (phase1 != _URC_NO_REASON) return phase1;
  return unwind_phase2(&uc, &ctx, obj);
}
```

> C++不支持resumptive exception handling (correcting the exceptional condition and resuming execution at the point where it was raised)，所以two-phase process不是必需的，但two-phase允許C++和其他語言共存於call stack上。

`_Unwind_Reason_Code _Unwind_ForcedUnwind(_Unwind_Exception *obj, _Unwind_Stop_Fn stop, void *stop_parameter);`執行forced unwinding：
跳過search phase，執行稍微不同的cleanup phase。`private_2`被用作stop function的參數。
這個函數很少用到。

`void _Unwind_Resume(_Unwind_Exception *obj);`繼續phase 2的unwind過程。它類似longjmp，是noreturn的，是唯一被編譯器直接調用的Level 1 API。編譯器通常在non-catch handlers末尾調用該函數。

`void _Unwind_DeleteException(_Unwind_Exception *obj);`銷毀指定的exception object。它是唯一處理`exception_cleanup`的Level 1 API，被`__cxa_end_catch`調用。

很多實現提供擴展：`_Unwind_Reason_Code _Unwind_Backtrace(_Unwind_Trace_Fn callback, void *ref);`是另一種特殊的unwind過程：忽略personality，將棧幀信息通知一個外部callback。

## Level 2 C++ ABI

這一部分處理C++的throw、catch block、out-of-scope variable destructors等語言相關概念。

### Data structures

每個thread有一個全局exception棧，`caughtExceptions`存儲棧頂(最新)的exception，`__cxa_exception::nextException`指向棧中下一個exception。
```cpp
struct __cxa_eh_globals {
  __cxa_exception *caughtExceptions;
  unsigned uncaughtExceptions;
};
```

```cpp
int main() {
  try {
    throw 1;
  } catch (...) {
    try {
      throw 2;
    } catch (...) {
      // The global exception stack has two exceptions here.
    }
  }
}
```

`__cxa_exception`的定義如下，其末尾存放Base ABI定義的`_Unwind_Exception`。`__cxa_exception`在`_Unwind_Exception`基礎上添加了C++語義信息。

```cpp
// Level 2
struct __cxa_exception {
  void *reserve; // here on 64-bit platforms
  size_t referenceCount; // here on 64-bit platforms
  std::type_info *exceptionType;
  void (*exceptionDestructor)(void *);
  unexpected_handler unexpectedHandler; // by default std::get_unexpected()
  terminate_handler terminateHandler; // by default std::get_terminate()
  __cxa_exception *nextException; // linked to the next exception on the thread stack
  int handlerCount; // incremented in __cxa_begin_catch, decremented in __cxa_end_catch, negated in __cxa_rethrow; last non-dependent performs the clean

  // The following fields cache information the catch handler found in phase 1.
  int handlerSwitchValue; // ttypeIndex in libc++abi
  const char *actionRecord;
  const char *languageSpecificData;
  void *catchTemp; // landingPad
  void *adjustedPtr; // adjusted pointer of the exception object

  _Unwind_Exception unwindHeader;
};
```

處理exception需要的信息(對於給定的IP，是否在try catch中、是否有需要執行的out-of-scope variable destructors、是否有dynamic exception specification)叫作language-specific data area (LSDA)，是實現相關的，Level 2 ABI沒有定義。

### Landing pad

Landing pad是text section中的一段和exception相關的代碼，它有三種：

* cleanup clause：通常調用destructors of out-of-scope variables或`__attribute__((cleanup(...)))`註冊的callbacks，然後用`_Unwind_Resume`跳轉回cleanup phase
* 捕獲exception的catch clause：調用destructors of out-of-scope variables，然後調用`__cxa_begin_catch`，執行catch代碼，最後調用`__cxa_end_catch`
* rethrow：調用destructors of out-of-scope variables in the catch clause，然後調用`__cxa_end_catch`，接着用`_Unwind_Resume`跳轉回cleanup phase

如果一個try有多個catch，那麼language-specific data area裏會有多個串聯的action table entries，但landing pad描述合併的catch clauses。
Personality在轉移控制權給landing pad前，會調用`_Unwind_SetGP`設置`__buitin_eh_return_data_regno(1)`存放switchValue，告知landing pad哪一個類型匹配了。

Rethrow是在執行catch代碼中間被`__cxa_rethrow`觸發的，需要destruct catch clause定義的局部變量，調用`__cxa_end_catch`抵消catch clause開頭調用的`__cxa_begin_catch`。

### `.gcc_except_table`

ELF系統裏language-specific data area通常存儲在`.gcc_except_table` section中。該section被`__gxx_personality_v0`和`__gcc_personality_v0`解析。它的結構很簡單：

* header(@LPStart、@TType和call sites的編碼，action records的起始偏移)
* call site table: 描述每個call site(一個地址區間)應執行的landing pad offset (0 if not exists)和action record offset (biased by 1, 0 for no action)
* action table
* type table (referennced by postive switch values)
* dynamic exception specification (deprecated in C++, so rarely used) (referenced by negative switch values)

下面是一個例子：

```asm
  .section        .gcc_except_table,"a",@progbits
  .p2align        2
GCC_except_table0:
.Lexception0:
  .byte   255                      # @LPStart Encoding = omit
  .byte   3                        # @TType Encoding = udata4
  .uleb128 .Lttbase0-.Lttbaseref0  # The start of action records
.Lttbaseref0:
  .byte   1                        # Call site Encoding = uleb128
  .uleb128 .Lcst_end0-.Lcst_begin0
.Lcst_begin0:                      # 2 call site code ranges
  .uleb128 .Ltmp0-.Lfunc_begin0    # >> Call Site 1 <<
  .uleb128 .Ltmp1-.Ltmp0           #   Call between .Ltmp0 and .Ltmp1
  .uleb128 .Ltmp2-.Lfunc_begin0    #     jumps to .Ltmp2
  .byte   1                        #   On action: 1
  .uleb128 .Ltmp1-.Lfunc_begin0    # >> Call Site 2 <<
  .uleb128 .Lfunc_end0-.Ltmp1      #   Call between .Ltmp1 and .Lfunc_end0
  .byte   0                        #     has no landing pad
  .byte   0                        #   On action: cleanup
.Lcst_end0:
  .byte   1                        # >> Action Record 1 <<
                                   #   Catch TypeInfo 1
  .byte   0                                #   No further actions
  .p2align        2
                                   # >> Catch TypeInfos <<
  .long   _ZTIi                           # TypeInfo 1
```

每個call site record除了call site offset和length外還有兩個值landing pad offset和action record offset。

* landing pad offset爲0。action record offset也應爲0。沒有landing pad
* landing pad offset非0。有landing pad
  + action record offset爲0，也叫做cleanup("cleanup"這個描述有些歧義，因爲Level 1有clean phase的術語)，通常描述local variable destructors和`__attribute__((cleanup(...)))`
  + action record offset非0。action record offset指向action table中一條action record。catch or noexcept specifier or exception specification

每個action record有兩個值：

* switch value (SLEB128): 正數表示catch的類型的TypeInfo在type table中的下標；負數表示type table中一個exception specification的offset；0表示cleanup action，效果類似於call site record中action record offset爲0
* offset to next action record: 須要處理的下一個action record，0表示結束。這種單鏈表形式可以描述串聯的多個catch，或exception specification list

> offset to next action record不僅可以用作單鏈表，也可用作trie，但幾乎碰不到可以用上trie性質的場景。

程序中不同區域對應的landing pad offset/action record offset取值：

* 無local variable destructor的非try區域：`landing_pad_offset==0 && action_record_offset==0`
* 有local variable destructor的非try區域：`landing_pad_offset!=0 && action_record_offset==0`。phase 2應停下調用cleanup
* 有`__attribute__((cleanup(...)))`的變量的非try區域：`landing_pad_offset!=0 && action_record_offset==0`。同上
* try區域：`landing_pad_offset!=0 && action_record_offset!=0`。landing pad指向catch拼接得到的代碼塊。action record爲大於0的type filter描述一個catch
* try區域，含`catch (...)`：同上。action record爲大於0的type filter指向type table中一個值0的項(表示catch any)
* 在一個含noexcept specifier的函數可能propagate exception到caller的區域：`landing_pad_offset!=0 && action_record_offset!=0`。landing pad指向調用`std::terminate`的代碼塊。action record爲大於0的type filter指向type table中一個值0的項(表示catch any)
* 在一個含exception specifier的函數可能propagate exception到caller的區域：`landing_pad_offset!=0 && action_record_offset!=0`。landing pad指向調用`__cxa_call_unexpected`的代碼塊。action record爲小於0的type filter描述一個exception specifier list

### Level 2 API

`void *__cxa_allocate_exception(size_t thrown_size);`。編譯器爲`throw A();`生成該函數的調用，分配一段內存存放`__cxa_exception`和A object。`__cxa_exception`緊挨在A object左側。
下面這個函數說明了程序操作的exception object的地址和`__cxa_exception`的關係：
```cpp
static void *thrown_object_from_cxa_exception(__cxa_exception *exception_header) {
  return static_cast<void *>(exception_header + 1);
}
```

`void __cxa_throw(void *thrown, std::type_info *tinfo, void (*destructor)(void *));`調用上述函數找到`__cxa_exception` header，填充各個字段(`referenceCount, exception_class, unexpectedHandler, terminateHandler, exceptionType, exceptionDestructor, unwindHeader.exception_cleanup`)後調用`_Unwind_RaiseException`。這個函數是noreturn的。

`void *__cxa_begin_catch(void *obj);`。編譯器在catch block的開頭生成該函數的調用。對於native exception：

* 加`handlerCount`
* 壓入該thread的全局exception棧，減少`uncaught_exception`值
* 返回adjusted pointer of the exception object

對於foreign exception(不一定有`__cxa_exception` header)：

* 該thread的全局exception棧爲空的話則push，否則執行`std::terminate`(不知道是否有類似`__cxa_exception::nextException`的字段)
* 返回`static_cast<_Unwind_Exception *>(obj) + 1`(假設`_Unwind_Exception`緊挨着thrown object)

簡化實現：
```cpp
void __cxa_throw(void *thrown, std::type_info *tinfo, void (*destructor)(void *)) {
  __cxa_exception *hdr = (__cxa_exception *)thrown - 1;
  hdr->exceptionType = tinfo; hdr->destructor = destructor;
  hdr->unexpectedHandler = std::get_unexpected();
  hdr->terminateHandler = std::get_terminate();
  hdr->unwindHeader.exception_class = ...;
  __cxa_get_globals()->uncaughtExceptions++;
  _Unwind_RaiseException(&hdr->unwindHeader);
  // Failed to unwind, e.g. the .eh_frame FDE is absent.
  __cxa_begin_catch(&hdr->unwindHeader);
  std::terminate();
}
```

`void __cxa_end_catch();`在catch block末尾或rethrow時被調用。對於native exception：

* 從該thread的全局exception棧上獲取當前exception，減少`handlerCount`
* `handlerCount`到〇則pop該thread的全局exception棧
* 如果是native exception：`handlerCount`減少到0時調用`__cxa_free_exception`(有dependent exception時得減少`referenceCount`，到0時調用`__cxa_free_exception`)

對於foreign exception：

* 調用`_Unwind_DeleteException`
* 執行`__cxa_eh_globals::uncaughtExceptions = nullptr;`(由於`__cxa_begin_catch`性質，棧中有恰好一個exception)

`void __cxa_rethrow();`會標註exception object，使`handlerCount`被`__cxa_end_catch`減低到0時不會被銷毀，因爲這個object會被`_Unwind_Resume`恢復的cleanup phase復用。

注意，除了`__cxa_begin_catch`和`__cxa_end_catch`，多數`__cxa_*`函數無法處理foreign exceptions(沒有`__cxa_exception` header)。

### 實例

對於如下代碼：
```cpp
#include <stdio.h>
struct A { ~A(); };
struct B { ~B(); };
void foo() { throw 0xB612; }
void bar() { B b; foo(); }
void qux() { try { A a; bar(); } catch (int x) { puts(""); } }
```

編譯得到的彙編概念上長這樣：

```cpp
void foo() {
  __cxa_exception *thrown = __cxa_allocate_exception(4);
  *thrown = 42;
  __cxa_throw(thrown, &typeid(int), /*destructor=*/nullptr);
}
void bar() {
  B b; foo(); return;
  landing_pad: b.~B(); _Unwind_Resume();
}
void qux() {
  A a; bar(); return;
  landing_pad: __cxa_begin_catch(obj); puts(""); __cxa_end_catch(obj);
}
```

運行流程：

* qux調用bar，bar調用foo，foo拋出exception
* foo動態分配內存塊，存放拋出的int和`__cxa_exception` header，然後執行`__cxa_throw`
* `__cxa_throw`填充`__cxa_exception`的其他字段，調用`_Unwind_RaiseException`

接下來`_Unwind_RaiseException`驅動Level 1的two-phase process。

* `_Unwind_RaiseException`執行phase 1: search phase
  + 對於bar，以`_UA_SEARCH_PHASE`爲actions參數調用personality，返回`_URC_CONTINUE_UNWIND`(沒有catch handler)
  + 對於qux，以`_UA_SEARCH_PHASE`爲actions參數調用personality，返回`_URC_HANDLER_FOUND`(有catch handler)
  + 標記qux的棧幀的stack pointer會被標記(保存在`private_2`中)，並停止搜索
* `_Unwind_RaiseException`執行phase 2: cleanup phase
  + bar的棧幀不是search phase標記的，以`_UA_CLEANUP_PHASE`爲actions參數調用personality，返回`_URC_INSTALL_CONTEXT`
  + 跳轉到bar的棧幀的landing pad
  + landing pad清理b之後用`_Unwind_Resume`回到cleanup phase
  + qux的棧幀是search phase標記的，以`_UA_CLEANUP_PHASE|_UA_HANDLER_FRAME`爲actions參數調用personality，返回`_UA_INSTALL_CONTEXT`
  + 跳轉到qux棧幀的landing pad
  + landing pad調用`__cxa_begin_catch`，執行catch代碼，然後調用`__cxa_end_catch`

### `__gxx_personality_v0`

Personality routine被Level 1 phase 1和phase 2調用，用於提供語言相關處理。不同的語言、實現或架構可能使用不同的personality routines。常見的personality如下：

* `__gxx_personality_v0`: C++
* `__gxx_personality_sj0`: sjlj
* `__gcc_personality_v0`: C `-fexceptions`，用於`__attribute__((cleanup(...)))`
* `__CxxFrameHandler3`: Windows MSVC
* `__gxx_personality_seh0`: MinGW-w64 `-fseh-exceptions`
* `__objc_personality_v0`: MacOSX環境ObjC

C++在ELF系統上的實現最常用的是`__gxx_personality_v0`，其實現在：

* GCC: `libstdc++-v3/libsupc++/eh_personality.cc`
* libc++abi: `src/cxa_personality.cpp`

`_Unwind_Reason_Code (*__personality_routine)(int version, _Unwind_Action action, uint64 exceptionClass, _Unwind_Exception *exceptionObject, _Unwind_Context *context);`

沒有錯誤的情況下：

* For `_UA_SEARCH_PHASE`, returns
  + `_URC_CONTINUE_UNWIND`: no lsda, or there is no landing pad, there is a non-catch handler or a matched exception specification
  + `_URC_HANDLER_FOUND`: there is a matched catch handler or an unmatched exception specification
* For `_UA_CLEANUP_PHASE`, returns
  + `_URC_CONTINUE_UNWIND`: no lsda, or there is no landing pad, or (not produced by a compiler) there is no cleanup action
  + `_URC_INSTALL_CONTEXT`: the other cases

Personality轉移控制權給landing pad前，會調用`_Unwind_SetGP`設置兩個寄存器(架構相關，`__buitin_eh_return_data_regno(0)`和`__buitin_eh_return_data_regno(1)`)存放`_Unwind_Exception *`和`switchValue`。

代碼：

```cpp
_unwind_Reason_Code __gxx_personality_v0(int version, _Unwind_Action actions, uint64_t exceptionClass, _Unwind_Exception *exc, _Unwind_Context *ctx) {
  if (actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME) && is_native) {
    auto *hdr = (__cxa_exception *)(exc+1) - 1;
    // Load cached results from phase 1.
    results.switchValue = hdr->handlerSwitchValue;
    results.actionRecord = hdr->actionRecord;
    results.languageSpecificData = hdr->languageSpecificData;
    results.landingPad = reinterpret_cast<uintptr_t>(hdr->catchTemp);
    results.adjustedPtr = hdr->adjustedPtr;

    _Unwind_SetGR(...);
    _Unwind_SetGR(...);
    _Unwind_SetIP(ctx, res.landingPad);
    return _URC_INSTALL_CONTEXT;
  }
  scan_eh_tab(results, actions, native_exception, unwind_exception, context);
  if (results.reason == _URC_CONTINUE_UNWIND ||
      results.reason == _URC_FATAL_PHASE1_ERROR)
    return results.reason;
  if (actions & _UA_SEARCH_PHASE) {
    auto *hdr = (__cxa_exception *)(exc+1) - 1;
    // Cache LSDA results in hdr.
    hdr->handlerSwitchValue = results.switchValue;
    hdr->actionRecord = results.actionRecord;
    hdr->languageSpecificData = results.languageSpecificData;
    hdr->catchTemp = reinterpret_cast<void *>(results.landingPad);
    hdr->adjustedPtr = results.adjustedPtr;
    return _URC_HANDLER_FOUND;
  }
  // _UA_CLEANUP_PHASE
  _Unwind_SetGR(...);
  _Unwind_SetGR(...);
  _Unwind_SetIP(ctx, res.landingPad);
  return _URC_INSTALL_CONTEXT;
}
```

對於native exception，search phase personality返回`_URC_HANDLER_FOUND`時會緩存該棧幀的LSDA相關信息。在cleanup phase再度調用personality時`actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME)`，personality知道可以讀取緩存，不需要解析`.gcc_except_table`。

在剩下三種情況下會調用`scan_eh_tab`解析`.gcc_except_table`：

* `actions & _UA_SEARCH_PHASE`
* `actions & _UA_CLEANUP_PHASE && actions & _UA_HANDLER_FRAME && !is_native`: foreign exception可以被`catch (...)`捕獲，但遇到exception specification則應terminate
* `actions & _UA_CLEANUP_PHASE && !(actions & _UA_HANDLER_FRAME)`: non-catch handlers and unmatched catch handlers, matched exception specification。還有一種可能是`_Unwind_ForcedUnwind`的phase 2

```cpp
static void scan_eh_tab(...) {
  ...
  const uint8_t *lsda = (const uint8_t *)_Unwind_GetLanguageSpecificData(context);
  if (lsda == nullptr) { res.reason = _URC_CONTINUE_UNWIND; return; }
  res.languageSpecificData = lsda;
  uintptr_t ipOffset = _Unwind_GetIP(context) - 1 - _Unwind_GetRegionStart(context);
  for each call site entry {
    if (!(start <= ipOffset && ipOffset < start + length))
      continue;
    res.landingPad = landingPad;
    if (landingPad == 0) { res.reason = _URC_CONTINUE_UNWIND; return; }
    if (actionRecord == 0) { // cleanup
      res.reason = actions & _UA_SEARCH_PHASE ? _URC_CONTINUE_UNWIND : _URC_HANDLER_FOUND;
      return;
    }
    // A catch or a dynamic exception specification.
    const uint8_t *action = actionTableStart + (actionRecord - 1);
    bool hasCleanup = false;
    for(;;) {
      res.actionRecord = action;
      int64_t switchValue = readSLEB128(&action);
      if (switchValue > 0) { // catch
        auto *catchType = ...;
        if (catchType == nullptr) { // catch (...)
          res.switchValue = switchValue; res.actionRecord = action;
          res.adjustedPtr = getThrownObjectPtr(exc); res.reason = _URC_HANDLER_FOUND;
          return;
        } else if (is_native) { // catch (T ...)
          auto *hdr = (__cxa_exception *)(exc+1) - 1;
          if (catchType->can_catch(hdr->exceptionType, adjustedPtr)) {
            res.switchValue = switchValue; res.actionRecord = action;
            res.adjustedPtr = adjustedPtr; res.reason = _URC_HANDLER_FOUND;
            return;
          }
        }
      } else if (switchValue < 0) { // dynamic exception specification
        if (actions & _UA_FORCE_UNWIND) {
          // Skip if forced unwinding.
        } else if (is_native) {
          if (!exception_spec_can_catch) {
            // The landing pad will call __cxa_call_unexpected.
            assert(actions & _UA_SEARCH_PHASE);
            res.switchValue = switchValue; res.actionRecord = action;
            res.adjustedPtr = adjustedPtr; res.reason = _URC_HANDLER_FOUND;
            return;
          }
        } else {
          // A foreign exception cannot be matched by the exception specification. The landing pad will call __cxa_call_unexpected.
          res.switchValue = switchValue; res.actionRecord = action;
          res.adjustedPtr = getThrownObjectPtr(exc); res.reason = _URC_HANDLER_FOUND;
          return;
        }
      } else { // switchValue == 0: cleanup
        hasCleanup = true;
      }
      const uint8_t *temp = action;
      int64_t actionOffset = readSLEB128(&temp);
      if (actionOffset == 0) { // End of action list
        res.reason = hasCleanup && actions & _UA_CLEANUP_PHASE
          ? _URC_HANDLER_FOUND : _URC_CONTINUE_UNWIND;
        return;
      }
      action += actionOffset;
    }
  }
  call_terminate();
}
```

### `__gcc_personality_v0`

libgcc and compiler-rt/lib/builtins實現了這個函數來處理`__attribute__((cleanup(...)))`。
默認的實現在search phase沒有返回`_URC_HANDLER_FOUND`，所以cleanup handler不能用作catch handler。
然而，我們可以提供自己的實現在search phase返回`_URC_HANDLER_FOUND`...
在x86-64上，`__buitin_eh_return_data_regno(0)`是RAX。我們可以讓cleanup handler傳遞RAX給landing pad。

```cpp
// a.cc
#include <exception>
#include <stdio.h>

extern "C" void my_catch();
extern "C" void throw_exception() { throw 42; }

int main() {
  fprintf(stderr, "uncaught exceptions: %d\n", std::uncaught_exceptions());
  my_catch();
  fprintf(stderr, "uncaught exceptions: %d\n", std::uncaught_exceptions());
}

// b.c
#include <setjmp.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <unwind.h>

void throw_exception();

struct __cxa_eh_globals {
  struct __cxa_exception *caughtExceptions;
  unsigned uncaughtExceptions;
};

struct __cxa_eh_globals *__cxa_get_globals();

static uintptr_t readULEB128(const uint8_t **a) {
  uintptr_t res = 0, shift = 0;
  const uint8_t *p = *a;
  uint8_t b;
  do {
    b = *p++;
    res |= (b & 0x7f) << shift;
    shift += 7;
  } while (b & 0x80);
  *a = p;
  return res;
}

_Unwind_Reason_Code __gcc_personality_v0(int version, _Unwind_Action actions,
                                         uint64_t exception_class,
                                         struct _Unwind_Exception *obj,
                                         struct _Unwind_Context *ctx) {
  const uint8_t *lsda = _Unwind_GetLanguageSpecificData(ctx);
  if (lsda == 0)
    return _URC_CONTINUE_UNWIND;
  uintptr_t func = _Unwind_GetRegionStart(ctx);
  uintptr_t pc = _Unwind_GetIP(ctx) - 1 - func;
  if (*lsda++ != 255) // Skip LPStart
    readULEB128(&lsda);
  if (*lsda++ != 255) // Skip TType
    readULEB128(&lsda);
  uintptr_t call_site_table_len = 0;
  if (*lsda++ == 1)
    call_site_table_len = readULEB128(&lsda);
  const uint8_t *end = lsda + call_site_table_len;
  while (lsda < end) {
    uintptr_t start = readULEB128(&lsda), len = readULEB128(&lsda),
              lpad = readULEB128(&lsda);
    if (!(start <= pc && pc < start + len))
      continue;
    if (lpad == 0)
      return _URC_CONTINUE_UNWIND;
    if (actions & _UA_SEARCH_PHASE)
      return _URC_HANDLER_FOUND;
    _Unwind_SetGR(ctx, __builtin_eh_return_data_regno(0), (uintptr_t)obj);
    _Unwind_SetGR(ctx, __builtin_eh_return_data_regno(1), 0); // switchValue==0
    _Unwind_SetIP(ctx, func + lpad);
    return _URC_INSTALL_CONTEXT;
  }
  return _URC_FATAL_PHASE2_ERROR;
}

struct Catch {
  struct _Unwind_Exception *obj;
  jmp_buf env;
  bool do_catch;
};

__attribute__((used))
static void my_jump(struct Catch *c) {
  if (c->do_catch) {
    struct __cxa_eh_globals *globals = __cxa_get_globals();
    globals->uncaughtExceptions--;
    longjmp(c->env, 1);
  }
}

__attribute__((naked)) static void my_cleanup(struct Catch *c) {
  asm("movq %rax, (%rdi); jmp my_jump");
}

void my_catch() {
  __attribute__((cleanup(my_cleanup))) struct Catch c;
  if (setjmp(c.env) == 0) {
    c.do_catch = 1;
    throw_exception();
  } else {
    fprintf(stderr, "caught exception: %p\n", c.obj);
    fprintf(stderr, "value: %d\n", *(int *)(c.obj + 1));
    c.do_catch = 0;
  }
}
```

```text
% clang -c -fexceptions a.cc b.c
% clang++ a.o b.o
% ./a.out
uncaught exceptions: 0
caught exception: 0x10f7f10
value: 42
uncaught exceptions: 0
```

### Rethrow

前面Landing pad節簡述了rethrow執行的代碼。通常caught exception會在`__cxa_end_catch`銷毀，因此`__cxa_rethrow`會標記exception object並增加`handlerCount`。

C++11 引入了Exception Propagation (N2179; `std::rethrow_exception` etc)，libstdc++中使用`__cxa_dependent_exception`實現。
設計參見<https://gcc.gnu.org/legacy-ml/libstdc++/2008-05/msg00079.html>

```cpp
struct __cxa_dependent_exception {
  void *reserve;
  void *primaryException;
};
```

`std::current_exception`和`std::rethrow_exception`會增加引用計數。

在libstdc++裏, `__cxa_rethrow`調用GCC擴展`_Unwind_Resume_or_Rethrow`(能resume forced unwinding)。

### LLVM IR

待補充

* nounwind: cannot unwind
* unwtables: force generation of the unwind table regardless of nounwind

```
if uwtables
  if nounwind
    CantUnwind
  else
    Unwind Table
else
  do nothing
```

## 編譯器行爲

* `-fno-exceptions -fno-asynchronous-unwind-tables`: neither `.eh_frame` nor `.gcc_except_table` exists
* `-fno-exceptions -fasynchronous-unwind-tables`: `.eh_frame` exists, `.gcc_except_table` doesn't
* `-fexceptions`: both `.eh_frame` and `.gcc_except_table` exist
  + In GCC, for a `noexcept` function, a possibly-throwing call site unhandled by a try block does not get an entry in the `.gcc_except_table` call site table. If the function has no try block, it gets a header-only `.gcc_except_table` (4 bytes)
  + In Clang, there is a call site entry calling `__clang_call_terminate`. The size overhead is larger than GCC's scheme. Improving this requires LLVM IR work

如果某個exception將要propagate到一個function的caller時：

* no `.eh_frame`: `_Unwind_RaiseException` returns `_URC_END_OF_STACK`. `__cxa_throw` calls `std::terminate`
* `.eh_frame` without `.gcc_except_table`: pass-through (local variable destructors are not called). This is the case of `-fno-exceptions -fasynchronous-unwind-tables`.
* `.eh_frame` with empty `.gcc_except_table`: `__gxx_personality_v0` calls `std::terminate` since no call site code range matches
* `.eh_frame` with proper `.gcc_except_table`: unwind

結合上述描述，某個exception將要propagate到一個noexcept function的caller時：

* `-fno-exceptions -fno-asynchronous-unwind-tables`: propagating through a function calls `std::terminate`
* `-fno-exceptions -fasynchronous-unwind-tables`: pass-through. Local variable destructors are not called. This behavior is unexpected.
* `-fexceptions`: propagating through a `noexcept` function calls `std::terminate`

When `std::terminate` is called, there is a diagnostic looking like `terminate called after throwing an instance of 'int'` (libstdc++; libc++ has a smiliar one).
There is no stack trace.
If the process installs a `SIGABRT` signal handler, the handler may get a stack trace and symbolize the addresses.

[Catching exceptions while unwinding through -fno-exceptions code](https://discourse.llvm.org/t/catching-exceptions-while-unwinding-through-fno-exceptions-code/57151) is a proposal to improve the diagnostics.

### Personality and typeinfo encoding

`.eh_frame` contains information about the unwind operation.
See [Stack unwinding](https://maskray.me/blog/2020-11-08-stack-unwinding) for its format.

In `-fpie/-fpic` mode, the personality and type info encodings have the `DW_EH_PE_indirect|DW_EH_PE_pcrel` bits on most targets.
```cc
void raise() { throw 42; }
bool foo() {
  try { raise(); } catch (int) { return true; }
  return false;
}
int main() { foo(); }
```
```asm
_Z3foov:
    .cfi_startproc
    .cfi_personality 155, DW.ref.__gxx_personality_v0
    .cfi_lsda 27, .Lexception0
...

	.section	.gcc_except_table,"a",@progbits
...
                                        # >> Catch TypeInfos <<
.Ltmp3:                                 # TypeInfo 1
	.long	.L_ZTIi.DW.stub-.Ltmp3
.Lttbase0:

	.data
	.p2align	3, 0x0
.L_ZTIi.DW.stub:
	.quad	_ZTIi
	.hidden	DW.ref.__gxx_personality_v0
	.weak	DW.ref.__gxx_personality_v0
	.section	.data.DW.ref.__gxx_personality_v0,"aGw",@progbits,DW.ref.__gxx_personality_v0,comdat
	.p2align	3, 0x0
	.type	DW.ref.__gxx_personality_v0,@object
	.size	DW.ref.__gxx_personality_v0, 8
DW.ref.__gxx_personality_v0:
	.quad	__gxx_personality_v0
```

In the example, `.eh_frame` contains a PC-relative relocations referencing `DW.ref.__gxx_personality_v0` 
`.gcc_except_table` contains a PC-relative relocation referencing `.L_ZTIi.DW.stub`.
The relocations are link-time constants, so `.eh_frame` can remain readonly.

`DW.ref.__gxx_personality_v0` and `.L_ZTIi.DW.stub` reside in writable sections which will contain dynamic relocations if `__gxx_personality_v0` and `_ZTIi` are defined in a shared object - which is often the case.

For `-fno-pic` code, different targets have different ideas.
AArch64 and RISC-V use `DW_EH_PE_indirect|DW_EH_PE_pcrel` as well.
On x86, `.cfi_personality` refers to `__gxx_personality_v0`. This will lead to a canonical PLT if `__gxx_personality_v0` is defined in a shared object (e.g. `libstdc++.so.6`).
I sent a patch <https://gcc.gnu.org/PR108622> to use `DW_EH_PE_indirect|DW_EH_PE_pcrel`.

### `R_MIPS_32` and `R_MIPS_64` personality encoding

<https://github.com/llvm/llvm-project/issues/58377>

```cpp
void foo() { try { throw 1; } catch (...) {} }
```

`mips64el-linux-gnuabi64-g++ -fpic` and `clang++ --target=mips64el-unknown-linux-gnuabi64 -fpic` use `DW_EH_PE_absptr | DW_EH_PE_indirect` to encode personality routine pointers.
Using `DW_EH_PE_absptr` instead of `DW_EH_PE_pcrel` is wrong.
GNU ld works around the compiler design problem by converting `DW_EH_PE_absptr` to `DW_EH_PE_pcrel`.
ld.lld does not support this and will report an error:
```sh
% clang++ --target=mips64el-linux-gnuabi -fpic -fuse-ld=lld -shared ex.cc
ld.lld: error: relocation R_MIPS_64 cannot be used against symbol 'DW.ref.__gxx_personality_v0'; recompile with -fPIC
>>> defined in /tmp/ex-40a996.o
>>> referenced by ex.cc
>>>               /tmp/ex-40a996.o:(.eh_frame+0x13)
...
```

`R_MIPS_32` for 32-bit builds is similar.

### Potentially-throwing `__cxa_end_catch`

`__cxa_end_catch` is potentially-throwing because it may destroy an exception object with a potentially-throwing destructor (e.g. `~C() noexcept(false) { ... }`).
```cpp
struct A { ~A(); };
void opaque();
void foo() {
  A a;
  // The exception object has an unknown type and may throw. The landing pad
  // then needs to call A::~A for `a` before jumping to _Unwind_Resume.
  try { opaque(); } catch (...) { }
}
```

To support an exception object with a potentially-throwing destructor, Clang generates conservative code for a catch-all clause or a catch clause matching a record type:

* assume that the exception object may have a throwing destructor
* emit `invoke void @__cxa_end_catch` (as the call is not marked as the `nounwind` attribute).
* emit a landing pad to destroy local variables and call `_Unwind_Resume`

Per C++ [dcl.fct.def.coroutine], a coroutine's function body implies a `catch (...)`.
Clang's code generation pessimizes even simple code, like:
```
UserFacing foo() {
  A a;
  opaque();
  co_return;
  // For `invoke void @__cxa_end_catch()`, the landing pad destroys the
  // promise_type and deletes the coro frame.
}
```

Throwing destructors are typically discouraged. In many environments, the destructors of exception objects are guaranteed to never throw, making our conservative code generation approach seem wasteful.

Furthermore, throwing destructors tend not to work well in practice:

* GCC does not emit call site records for the region containing `__cxa_end_catch`. This has been a long time, since 2000.
* If a catch-all clause catches an exception object that throws, both GCC and Clang using libstdc++ leak the allocated exception object.

To avoid code generation pessimization, I added [`-fassume-nothrow-exception-dtor`](https://reviews.llvm.org/D108905) for Clang 18 to assume that `__cxa_end_catch` calls have the `nounwind` attribute.
This requires that thrown exception objects' destructors will never throw.

To detect misuses, diagnose throw expressions with a potentially-throwing destructor. Technically, it is possible that a potentially-throwing destructor never throws when called transitively by `__cxa_end_catch`, but these cases seem rare enough to justify a relaxed mode.

## 其他

### 使用libc++和libc++abi

On Linux, compared with `clang`, `clang++` additionally links against libstdc++/libc++ and libm.

Dynamically link against libc++.so (which depends on libc++abi.so) (additionally specify `-pthread` if threads are used):

```sh
clang++ -stdlib=libc++ -nostdlib++ a.cc -lc++ -lc++abi
# clang -stdlib=libc++ a.cc -lc++ -lc++abi does not pass -lm to the linker.
```

If compile actions and link actions are separate (`-stdlib=libc++` passes `-lc++` but its position is undesired, so just don't use it):

```sh
clang++ -nostdlib++ a.cc -lc++ -lc++abi
```

Statically link in libc++.a (which includes the members of libc++abi.a). This requires a `-DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=on` build:

```sh
clang++ -stdlib=libc++ -static-libstdc++ -nostdlib++ a.cc -pthread
```

Statically link in libc++.a and libc++abi.a. This is a bit inferior because there is a duplicate -lc++ passed by the driver.

```sh
clang++ -stdlib=libc++ -static-libstdc++ -nostdlib++ a.cc -Wl,--push-state,-Bstatic -lc++ -lc++abi -Wl,--pop-state -pthread
```

### libc++abi和libsupc++

值得注意的是，libc++abi提供的`<exception> <stdexcept>`類型佈局(如`logic_error` `runtime_error`等)都是特意和libsupc++兼容的。
GCC 5的libstdc++拋棄ref-counted `std::string`後libsupc++仍使用`__cow_string`用於`logic_error`等。libc++abi也使用了類似的ref-counted string。

libsupc++和libc++abi不使用inline namespace，有衝突的符號名，因此通常一個libc++/libc++abi應用無法使用某個動態鏈接`libstdc++.so`的shared object(ODR violation)。

如果花一些工夫，還是能解決這個問題的：編譯libstdc++中非libsupc++的部分得到自製`libstdc++.so.6`。可執行檔鏈接libc++abi提供`libstdc++.so.6`需要的C++ ABI符號。

### Monolithic `.gcc_except_table`

Clang 12之前採用monolithic `.gcc_except_table`。和其他很多metadata sections一樣，monolithic設計的主要問題是無法被linker garbage collect。
對於RISC-V `-mrelax`和basic block sections則會有更大的問題：`.gcc_except_table`有指向text sections local symbols的relocations。
如果指向的text sections在COMDAT group中被丟棄，則這些relocations會被linker拒絕(`error: relocation refers to a symbol in a discarded section`)。
![.eh_frame with monolithic .gcc_except_table](/static/2020-12-12-c++-exception-handling/eh_frame_and_monolithic_gcc_except_table.svg)
![monolithic .gcc_except_table](/static/2020-12-12-c++-exception-handling/monolithic_gcc_except_table.svg){ width=70% }

解決方案就是採用fragmented `.gcc_except_table`(<https://reviews.llvm.org/D83655>)。
![fragmented .gcc_except_table](/static/2020-12-12-c++-exception-handling/fragmented_gcc_except_table.svg){ width=70% }

但實際部署沒有那麼簡單:)LLD先處理`--gc-sections`(尚不明確哪些`.eh_frame` pieces是live的)，後處理(包括GC)`.eh_frame`。

`--gc-sections`時，所有`.eh_frame` pieces是live的。它們會標記所有`.gcc_except_table.*` live。
根據section group的GC規則，一個`.gcc_except_table.*`會標註同一section group的其他sections(包含`.text.*`) live。
結果就是所有section groups中的`.text.*`無法被GC，導致輸入大小增大。
![bad GC with .gcc_except_table.*](/static/2020-12-12-c++-exception-handling/lsda_gc.svg){ width=70% }

<https://reviews.llvm.org/D91579>修復了這個問題：對於`.eh_frame`，不要標註section group中的`.gcc_except_table`。
![good GC with .gcc_except_table.*](/static/2020-12-12-c++-exception-handling/lsda_gc_new.svg){ width=70% }

### `-fbasic-block-sections=`

使用basic block sections時，可以選擇每個basic block section獲得其專屬的`.gcc_except_table`，或者讓一個函數的所有basic block sections使用同一個`.gcc_except_table`。LLVM實現選擇了後者，有幾個好處：

* No duplicate headers
* Sharable type table
* Sharable action table (this only matters for the deprecated exception specification)

使用同一個`.gcc_except_table`就只有一個LPStart，得保證所有landing pads到LPStart的offsets均可以用relocations表示。
因爲多數架構沒有表示差的relocation type，因此把landing pads放在同一個section是最合適的表示方式。

## Exception handling ABI for the ARM architecture

整體結構和Itanium C++ ABI: Exception Handling相同，數據結構、`_Unwind_*`等有些許差異。

<https://maskray.me/blog/2020-11-08-stack-unwinding>含有少量註記。

## Compact Exception Tables for MIPS ABIs

用`.eh_frame_entry`和`.gnu_extab`描述。

設計理念：

* Exception code ranges are sorted and must be linearly searched. Therefore it would be more compact to specify each relative to the previous one, rather than relative to a fixed base.
* The landing pad is often close to the exception region that uses it. Therefore it is better to use the end of the exception region as the reference point, than use the function base address.
* The action table can be integrated directly with the exception region definition itself. This removes one indirection. The threading of actions can still occur, by providing an offset to the next exception encoding of interest.
* Often the action threading is to the next exception region, so optimizing that case is important.
* Catch types and exception specification type lists cannot easily be encoded inline with the exception regions themselves. It is necessary to preserve the unique indices that are automatically created by the DWARF scheme.

使用和ARM EH類似的compact unwind descriptors。Builtin PR1表示沒有language-dependent data，Builtin PR2用於C/C++
