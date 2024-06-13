---
layout: post
title: "Journey of v8 call stack overflow with Maglev"
image: /img/202406/v8_00.png
tags: [v8, Maglev, internals]
---
* TOC
{:toc}

# Intro

I found an interesting [tweet](https://x.com/mistymntncop/status/1800485977049514474) on Twitter and analyzed it.  
This is about how to check if a particular function has completed Maglev compilation without using debug functions.  
To summarize, the number of recursive function calls may be different in the Interpreter version and the Maglev compiled version, so this can be used to distinguish them.    
  
# Stack Frame Overflow

```
./jit_test.js:3: RangeError: Maximum call stack size exceeded
    return 1 + getStackDepth();
```
This method uses RangeError, call stack overflow that appear like this.  
  
```js
function getStackDepth() {
  try {
    return 1 + getStackDepth();
  } catch (e) {                         // [1]
    return 1;
  }
}

function optimizeFunction(fn) {
  for (let i=0; i<10000; i++) {
    fn();
  }
}

function target() {
  x = 1.1 + 2.2;
  return getStackDepth();
}

optimizeFunction(getStackDepth);

const initialDepth = target();          // [2]
console.log(`Initial Stack Depth: ${initialDepth}`);

for(let i=0; i<412; i++) {
                // 408 => marking MAGLEV
                // 409 => compiling MAGLEV
                // 410 => completed compiling
                // 411 => execute compiled code
  if (i % 100 == 0) {
    console.log("===> "+i, target());   // [3]
  }
  else target();
}

const optimizedDepth = target();        // [4]
console.log(`Optimized Stack Depth: ${optimizedDepth}`);
```
In [1], an exception occurs when the Max stack frame is reached while calling a recursive function.  
[2] gets the number of recursive calls in the `target` function executed as bytecode.  
In [3], the `target` function is continuously called, and at some point it is compiled with Maglev. In my environment, preparation starts when `i` is 408 and the compiled code is executed when `i` is 411. Exact timing may vary depending on environment.  
[4] gets the number of recursive calls in the compiled `target` function.  
  
execution result:  
```
Initial Stack Depth: 17991
===> 0 17991
===> 100 17991
===> 200 17991
===> 300 17991
===> 400 17991
===> 406 17991
===> 407 17991
===> 408 17991
===> 409 17991
===> 410 17991
===> 411 17992
Optimized Stack Depth: 17992
```
  
Add `--trace-opt`:  
```
Initial Stack Depth: 17991
===> 0 17991
===> 100 17991
===> 200 17991
===> 300 17991
===> 400 17991
===> 406 17991
===> 407 17991
[marking 0x15ed0019ab0d <JSFunction target (sfi = 0x15ed0019a99d)> for optimization to MAGLEV, ConcurrencyMode::kConcurrent, reason: hot and stable]
===> 408 17991
[compiling method 0x15ed0019ab0d <JSFunction target (sfi = 0x15ed0019a99d)> (target MAGLEV), mode: ConcurrencyMode::kConcurrent]
===> 409 17991
[completed compiling 0x15ed0019ab0d <JSFunction target (sfi = 0x15ed0019a99d)> (target MAGLEV) - took 0.000, 0.593, 0.008 ms]
===> 410 17991
===> 411 17992
Optimized Stack Depth: 17992
```
Exactly after Maglev compilation completes, the depth changes.  
  

# Analysis

## Stack frame overflow

This is the interpreter processing flow for function calls.  
Please refer to this if you want to analyze it yourself, otherwise skip the code.  
  
src/execution/execution.cc
```cpp
V8_WARN_UNUSED_RESULT MaybeHandle<Object> Invoke(Isolate* isolate,
                                                 const InvokeParams& params) {
  RCS_SCOPE(isolate, RuntimeCallCounterId::kInvoke);
  DCHECK(!IsJSGlobalObject(*params.receiver));
  DCHECK_LE(params.argc, FixedArray::kMaxLength);
  DCHECK(!isolate->has_exception());
  ...
  // Placeholder for return value.
  Tagged<Object> value;
  Handle<Code> code =
      JSEntry(isolate, params.execution_target, params.is_construct);
  {
    // Save and restore context around invocation and block the
    // allocation of handles without explicit handle scopes.
    SaveContext save(isolate);
    SealHandleScope shs(isolate);

    if (v8_flags.clear_exceptions_on_js_entry) isolate->clear_exception();

    if (params.execution_target == Execution::Target::kCallable) {
      // clang-format off
      // {new_target}, {target}, {receiver}, return value: tagged pointers
      // {argv}: pointer to array of tagged pointers
      using JSEntryFunction = GeneratedCode<Address(
          Address root_register_value, Address new_target, Address target,
          Address receiver, intptr_t argc, Address** argv)>;
      // clang-format on
      JSEntryFunction stub_entry =
          JSEntryFunction::FromAddress(isolate, code->instruction_start());

      Address orig_func = (*params.new_target).ptr();
      Address func = (*params.target).ptr();
      Address recv = (*params.receiver).ptr();
      Address** argv = reinterpret_cast<Address**>(params.argv);
      RCS_SCOPE(isolate, RuntimeCallCounterId::kJS_Execution);
      value = Tagged<Object>(
          stub_entry.Call(isolate->isolate_data()->isolate_root(), orig_func,
                          func, recv, JSParameterCount(params.argc), argv));
	...
```
Find and call the function entry.  
  
```cpp
Handle<Code> JSEntry(Isolate* isolate, Execution::Target execution_target,
                     bool is_construct) {
  if (is_construct) {
    DCHECK_EQ(Execution::Target::kCallable, execution_target);
    return BUILTIN_CODE(isolate, JSConstructEntry);
  } else if (execution_target == Execution::Target::kCallable) {
    DCHECK(!is_construct);
    return BUILTIN_CODE(isolate, JSEntry);
  } else if (execution_target == Execution::Target::kRunMicrotasks) {
    DCHECK(!is_construct);
    return BUILTIN_CODE(isolate, JSRunMicrotasksEntry);
  }
  UNREACHABLE();
}
```
  
src/builtins/x64/builtins-x64.cc
```cpp
void Builtins::Generate_JSEntry(MacroAssembler* masm) {
  Generate_JSEntryVariant(masm, StackFrame::ENTRY, Builtin::kJSEntryTrampoline);
}
```
  
```cpp
void Builtins::Generate_JSEntryTrampoline(MacroAssembler* masm) {
  Generate_JSEntryTrampolineHelper(masm, false);
}
```
  
```cpp
static void Generate_JSEntryTrampolineHelper(MacroAssembler* masm,
                                             bool is_construct) {
  // Expects six C++ function parameters.
  // - Address root_register_value
  // - Address new_target (tagged Object pointer)
  // - Address function (tagged JSFunction pointer)
  // - Address receiver (tagged Object pointer)
  // - intptr_t argc
  // - Address** argv (pointer to array of tagged Object pointers)
  // (see Handle::Invoke in execution.cc).

  // Open a C++ scope for the FrameScope.
  {
    // Platform specific argument handling. After this, the stack contains
    // an internal frame and the pushed function and receiver, and
    // register rax and rbx holds the argument count and argument array,
    // while rdi holds the function pointer, rsi the context, and rdx the
    // new.target.

    // MSVC parameters in:
    // rcx        : root_register_value
    // rdx        : new_target
    // r8         : function
    // r9         : receiver
    // [rsp+0x20] : argc
    // [rsp+0x28] : argv
    //
    // GCC parameters in:
    // rdi : root_register_value
    // rsi : new_target
    // rdx : function
    // rcx : receiver
    // r8  : argc
    // r9  : argv

    __ movq(rdi, kCArgRegs[2]);
    __ Move(rdx, kCArgRegs[1]);
    // rdi : function
    // rdx : new_target
  ...
    // Invoke the builtin code.
    Builtin builtin = is_construct ? Builtin::kConstruct : Builtins::Call();
    __ CallBuiltin(builtin);
    ...
```
  
```cpp
void Builtins::Generate_Call_ReceiverIsAny(MacroAssembler* masm) {
  Generate_Call(masm, ConvertReceiverMode::kAny);
}
```
  
```cpp
void Builtins::Generate_CallFunction_ReceiverIsAny(MacroAssembler* masm) {
  Generate_CallFunction(masm, ConvertReceiverMode::kAny);
}
```
  
```cpp
// static
void Builtins::Generate_CallFunction(MacroAssembler* masm,
                                     ConvertReceiverMode mode) {
  // ----------- S t a t e -------------
  //  -- rax : the number of arguments
  //  -- rdi : the function to call (checked to be a JSFunction)
  // ------
  ...
  __ movzxwq(
      rbx, FieldOperand(rdx, SharedFunctionInfo::kFormalParameterCountOffset));
  __ InvokeFunctionCode(rdi, no_reg, rbx, rax, InvokeType::kJump);
}
```
  
src/codegen/x64/macro-assembler-x64.cc
```cpp
void MacroAssembler::InvokeFunctionCode(Register function, Register new_target,
                                        Register expected_parameter_count,
                                        Register actual_parameter_count,
                                        InvokeType type) {
  ...
  switch (type) {
    case InvokeType::kCall:
      CallJSFunction(function);
      break;
    case InvokeType::kJump:
      JumpJSFunction(function);
      break;
  }
  ...  
```
  
```cpp
void MacroAssembler::CallJSFunction(Register function_object) {
  static_assert(kJavaScriptCallCodeStartRegister == rcx, "ABI mismatch");
#ifdef V8_ENABLE_SANDBOX
  // When the sandbox is enabled, we can directly fetch the entrypoint pointer
  // from the code pointer table instead of going through the Code object. In
  // this way, we avoid one memory load on this code path.
  LoadCodeEntrypointViaCodePointer(
      rcx, FieldOperand(function_object, JSFunction::kCodeOffset));
  call(rcx);
#else
  LoadTaggedField(rcx, FieldOperand(function_object, JSFunction::kCodeOffset));
  CallCodeObject(rcx);
#endif
}
```
`InterpreterEntryTrampoline` is executed for `function_object`.  
  
src/builtins/x64/builtins-x64.cc
```cpp
void Builtins::Generate_InterpreterEntryTrampoline(
    MacroAssembler* masm, InterpreterEntryTrampolineMode mode) {
  Register closure = rdi;
  ...
  // Allocate the local and temporary register file on the stack.
  Label stack_overflow;
  {
    // Load frame size from the BytecodeArray object.
    __ movl(rcx, FieldOperand(kInterpreterBytecodeArrayRegister,
                              BytecodeArray::kFrameSizeOffset));

    // Do a stack check to ensure we don't go over the limit.
    __ movq(rax, rsp);
    __ subq(rax, rcx);
    __ cmpq(rax, __ StackLimitAsOperand(StackLimitKind::kRealStackLimit));
    __ j(below, &stack_overflow);

    // If ok, push undefined as the initial value for all register file entries.
    Label loop_header;
    Label loop_check;
    __ LoadRoot(kInterpreterAccumulatorRegister, RootIndex::kUndefinedValue);
    __ jmp(&loop_check, Label::kNear);
    __ bind(&loop_header);
    // TODO(rmcilroy): Consider doing more than one push per loop iteration.
    __ Push(kInterpreterAccumulatorRegister);
    // Continue loop if not done.
    __ bind(&loop_check);
    __ subq(rcx, Immediate(kSystemPointerSize));
    __ j(greater_equal, &loop_header, Label::kNear);
  }
  __ bind(&stack_overflow);
  __ CallRuntime(Runtime::kThrowStackOverflow);
  __ int3();  // Should not return.
}
```
  
This is the part related to checking the stack frame size in the above code.  
- `movl(rcx, FieldOperand(kInterpreterBytecodeArrayRegister, BytecodeArray::kFrameSizeOffset));`
- `movq(rax, rsp);`
- `subq(rax, rcx);`
- `cmpq(rax, __ StackLimitAsOperand(StackLimitKind::kRealStackLimit));`
- `j(below, &stack_overflow);`

This will further expand(=subq) the stack as much as `rcx`, and if `rsp` exceeds the limit when doing so, it will be processed as stack_overflow.  

## Stack Limit

This is an analysis of the size of the Stack Limit.  
  
src/codegen/x64/macro-assembler-x64.cc
```cpp
Operand MacroAssembler::StackLimitAsOperand(StackLimitKind kind) {
  DCHECK(root_array_available());
  intptr_t offset = kind == StackLimitKind::kRealStackLimit
                        ? IsolateData::real_jslimit_offset()
                        : IsolateData::jslimit_offset();

  CHECK(is_int32(offset));
  return Operand(kRootRegister, static_cast<int32_t>(offset));
}
```
  
src/execution/isolate-data.h
```cpp
  static constexpr int real_jslimit_offset() {
    return stack_guard_offset() + StackGuard::real_jslimit_offset();
  }
```
  
```cpp
#define V(Offset, Size, Name) \
  static constexpr int Name##_offset() { return Offset - kIsolateRootBias; }
  ISOLATE_DATA_FIELDS(V)
#undef V
```
 
src/execution/stack-guard.h
```cpp
  static constexpr int real_jslimit_offset() {
    return offsetof(StackGuard, thread_local_) +
           offsetof(ThreadLocal, real_jslimit_);
  }
```
That is, it returns the offset of `kRootRegister->StackGuard->ThreadLocal.real_jslimit_` from `kRootRegister`.  
  
```cpp
void StackGuard::ThreadLocal::Initialize(Isolate* isolate,
                                         const ExecutionAccess& lock) {
  const uintptr_t kLimitSize = v8_flags.stack_size * KB;
  DCHECK_GT(GetCurrentStackPosition(), kLimitSize);
  uintptr_t limit = GetCurrentStackPosition() - kLimitSize;
  real_jslimit_ = SimulatorStack::JsLimitFromCLimit(isolate, limit);
  set_jslimit(SimulatorStack::JsLimitFromCLimit(isolate, limit));
  real_climit_ = limit;
  set_climit(limit);
  interrupt_scopes_ = nullptr;
  interrupt_flags_ = 0;
}
```
As much as `kLimitSize` is available at the current stack position.  
  
src/common/globals.h
```c
#define V8_DEFAULT_STACK_SIZE_KB 984
```
kLimitSize = V8_DEFAULT_STACK_SIZE_KB * KB = 0x000f6000  
  
A stack frame of size 0x38 is used per `getStackDepth` call, and a rough calculation considering the speculative stack frames used in functions such as Trampoline that calls `getStackDepth` is (0x000f6000-(0x41~0x78))/ 0x38 = 17991.  

## Maglev

stack limit check before Maglev:  
```
(lldb) c
Process 15726 resuming
Process 15726 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
    frame #0: 0x00000001611c43df
->  0x1611c43df: cmpq   -0x60(%r13), %rsp
    0x1611c43e3: jbe    0x1611c447f
    0x1611c43e9: movabsq $0x51d0019aad1, %rdi      ; imm = 0x51D0019AAD1
    0x1611c43f3: movl   0x13(%rdi), %esi
Target 0: (d8) stopped.
(lldb) x/xg $r13-0x60
0x7fad30058020: 0x0000000306e25290  ; Stack Limit
(lldb) p/x $rsp
(unsigned long) 0x0000000306f1b218  ; Stack Pointer
(lldb) p/x 0x0000000306f1b218-0x0000000306e25290
(long) 0x00000000000f5f88
```
  
stack limit check before Maglev:  
```
(lldb) c
Process 15726 resuming
Process 15726 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
    frame #0: 0x00000001611c43df
->  0x1611c43df: cmpq   -0x60(%r13), %rsp
    0x1611c43e3: jbe    0x1611c447f
    0x1611c43e9: movabsq $0x51d0019aad1, %rdi      ; imm = 0x51D0019AAD1
    0x1611c43f3: movl   0x13(%rdi), %esi
Target 0: (d8) stopped.
(lldb) x/xg $r13-0x60
0x7fad30058020: 0x0000000306e25290  ; Stack Limit
(lldb) p/x $rsp
(unsigned long) 0x0000000306f1b238  ; Stack Pointer

(lldb) p/x 0x0000000306f1b238-0x0000000306e25290
(long) 0x00000000000f5fa8
```

I checked with a break in the `StackOverflow` check, and when the `target` function called `getStackDepth` and compared the Stack Limit, there was a difference in the stack pointer pointed to by the first call stack.  
After Maglev compilation, the size of the stack frame used by the `target` function has become as small as 0x20.  
  
Where did this get smaller?  

backtrace before Maglev:  
```
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 17.1
  * frame #0: 0x00000001611c43c0 ; getStackDepth
    frame #1: 0x00000001611c4426 ; getStackDepth
    frame #2: 0x00000001611c4426 ; getStackDepth
    frame #3: 0x00000001011e04aa d8`Builtins_InterpreterEntryTrampoline + 298
    frame #4: 0x00000001011e04aa d8`Builtins_InterpreterEntryTrampoline + 298
    frame #5: 0x00000001011ddf1c d8`Builtins_JSEntryTrampoline + 92
    frame #6: 0x00000001011ddc47 d8`Builtins_JSEntry + 135

; frame #3
(lldb) p/x $rsp
(unsigned long) 0x0000000306f1b208
(lldb) p/x $rbp
(unsigned long) 0x0000000306f1b248
```

backtrace after Maglev:  
```
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 16.1
  * frame #0: 0x00000001611c43c0 ; getStackDepth
    frame #1: 0x00000001611c4426 ; getStackDepth
    frame #2: 0x00000001611c4426 ; getStackDepth
    frame #3: 0x00000001611c4bbe ; target
    frame #4: 0x00000001011e04aa d8`Builtins_InterpreterEntryTrampoline + 298
    frame #5: 0x00000001011ddf1c d8`Builtins_JSEntryTrampoline + 92
    frame #6: 0x00000001011ddc47 d8`Builtins_JSEntry + 135

; frame #3
(lldb) p/x $rsp
(unsigned long) 0x0000000306f1b228
(lldb) p/x $rbp
(unsigned long) 0x0000000306f1b248
```

Before Maglev compilation, `Builtins_InterpreterEntryTrampoline` is called twice, to call the `target` function in the bytecode and the compiled `getStackDepth` that the `target` function calls.  
However, after Maglev compilation, the compiled `target` function and the compiled `getStackDepth` are called, so `Builtins_InterpreterEntryTrampoline` is reduced to once.  
In other words, the stack frame of the `Builtins_InterpreterEntryTrampoline` function uses 0x40 bytes, but the stack frame of the compiled `target` function uses 0x20 bytes, so the compiled version has 0x20 space before reaching the stack limit.  
This difference allowed the recursive function to be executed once more and the `target` function to return different results before and after Maglev compilation.  
