# Android ART执行类方法的过程

url：https://www.jianshu.com/p/2ff1b63f686b

7.0之前ART只有AOT模式，7.0之后采用解释、JIT、AOT混合执行。方法调用有如下几种流程：
 机器码-->机器码
 机器码-->解释
 解释-->解释
 解释-->机器码

## 基本执行流程

java方法在ART虚拟机中以ArtMethod表示，其中entry_point_from_quick_compiled_code_保存的是方法的入口地址，在类加载的LinkCode()函数中设置入口地址。



```cpp
class ArtMethod {
  struct PtrSizedFields {
    void* data_;//与JNI有关
    void* entry_point_from_quick_compiled_code_;//java方法入口函数地址
  } ptr_sized_fields_;
};
```

从zygote启动创建虚拟机环境后，通过AndroidRuntime::start()开始进入java世界，如下代码调用的就是com.android.internal.os.ZygoteInit类的静态成员函数main。



```c
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{

    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
        "([Ljava/lang/String;)V");
    env->CallStaticVoidMethod(startClass, startMeth, strArray);
}
```

env->CallStaticVoidMethod是一个函数指针，经验证实际指向jni_internal.cc::CallStaticVoidMethodV()函数，如下会进一步调用到ArtMethod的Invoke()方法。



```c
1.static void CallStaticVoidMethodV(JNIEnv* env, jclass, jmethodID mid, va_list args)
    2.JValue InvokeWithVarArgs(const ScopedObjectAccessAlreadyRunnable& soa, jobject obj, jmethodID mid,
                         va_list args)
        3.void InvokeWithArgArray(const ScopedObjectAccessAlreadyRunnable& soa,
                               ArtMethod* method, ArgArray* arg_array, JValue* result,
                               const char* shorty)
            4.method->Invoke(soa.Self(), args, arg_array->GetNumBytes(), result, shorty);
```

Invoke()方法最终调用到汇编代码art_quick_invoke_stub_internal，汇编又进一步跳转到ART_METHOD_QUICK_CODE_OFFSET_32偏移，根据定义可知，该偏移正是ArtMethod对象中的成员变量entry_point_from_compiled_code_，由此真正开始进入java方法。



```c
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {
  if (!IsStatic()) {
    (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
  } else {
    (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
  }
}
```



```c
1.void art_quick_invoke_stub(ArtMethod* method, uint32_t* args, uint32_t args_size,
                            Thread* self, JValue* result, const char* shorty)

2.void art_quick_invoke_static_stub(ArtMethod* method, uint32_t* args,
                                    uint32_t args_size, Thread* self, JValue* result,
                                    const char* shorty)
      template <bool kIsStatic>
    3.static void quick_invoke_reg_setup(ArtMethod* method, uint32_t* args, uint32_t args_size,
                                       Thread* self, JValue* result, const char* shorty)
        4.void art_quick_invoke_stub_internal(ArtMethod*, uint32_t*, uint32_t,
                                            Thread* self, JValue* result, uint32_t, uint32_t*,
                                            uint32_t*);
```

art/runtime/arch/arm/quick_entrypoints_arm.S:



```ruby
ENTRY art_quick_invoke_stub_internal
    SPILL_ALL_CALLEE_SAVE_GPRS             @ spill regs (9)

    ldr    ip, [r0, #ART_METHOD_QUICK_CODE_OFFSET_32]  @ get pointer to the code
    blx    ip                              @ call the method

END art_quick_invoke_stub_internal
```

art/runtime/generated/asm_support_gen.h:



```cpp
#define ART_METHOD_QUICK_CODE_OFFSET_32 24
DEFINE_CHECK_EQ(static_cast<int32_t>(ART_METHOD_QUICK_CODE_OFFSET_32), (static_cast<int32_t>(art::ArtMethod::EntryPointFromQuickCompiledCodeOffset(art::PointerSize::k32).Int32Value())))
```

## LinkCode介绍

如上述介绍，通过com.android.internal.os.ZygoteInit::main类、方法字符串信息，最终查找到方法的ArtMethod对象，并通过entry_point_from_compiled_code_开始执行方法对应的指令。
 设置entry_point_from_compiled_code_是在Class_Linker.cc::LinkCode()完成的，由于虚拟机执行情况复杂，解释、static、JNI等执行情况，entry_point_from_compiled_code_并不直接指向AOT编译的机器码，而是指向一段Trampoline代码来间接执行。



```c
static void LinkCode(ClassLinker* class_linker,
                     ArtMethod* method,
                     const OatFile::OatClass* oat_class,
                     uint32_t class_def_method_index) REQUIRES_SHARED(Locks::mutator_lock_) {

  if (oat_class != nullptr) {
    //设置entry_point_from_compiled_code_为OAT编译生成的机器码地址
    const OatFile::OatMethod oat_method = oat_class->GetOatMethod(class_def_method_index);
    oat_method.LinkMethod(method);
  }

  const void* quick_code = method->GetEntryPointFromQuickCompiledCode();
  //机器码地址为空或者是调试状态等，需要解释模式
  bool enter_interpreter = class_linker->ShouldUseInterpreterEntrypoint(method, quick_code);

  if (method->IsStatic() && !method->IsConstructor()) {
    //静态方法且不是类初始化"<clinit>"方法，设置入口地址为art_quick_resolution_trampoline
    //跳转到artQuickResolutionTrampoline函数。该函数和类的解析有关
    //初始化完毕后会调用FixupStaticTrampolines()来更新入口地址为正确的机器码。
    method->SetEntryPointFromQuickCompiledCode(GetQuickResolutionStub());
  } else if (quick_code == nullptr && method->IsNative()) {
    //jni方法，设置入口地址为art_quick_generic_jni_trampoline，跳转到artQuickGenericJniTrampoline函数。
    method->SetEntryPointFromQuickCompiledCode(GetQuickGenericJniStub());
  } else if (enter_interpreter) {
    //解释执行，设置入口地址为art_quick_to_interpreter_bridge，跳转到artQuickToInterpreterBridge函数。
    method->SetEntryPointFromQuickCompiledCode(GetQuickToInterpreterBridge());
  }

  //jni方法，设置ArtMethod ptr_sized_fields_.data为art_jni_dlsym_lookup_stub，跳转到artFindNativeMethod函数。
  if (method->IsNative()) {
    method->UnregisterNative();
  }
}
```

**Trampoline函数：artQuickToInterpreterBridge**
 上述有4种Trampoline函数，我们以机器码-->解释过程为例介绍artQuickToInterpreterBridge。
 art/runtime/entrypoints/quick/quick_trampoline_entrypoints.cc：



```rust
uint64_t artQuickToInterpreterBridge(ArtMethod* method, Thread* self, ArtMethod** sp) {
  ManagedStack fragment;//栈管理相关

  //A->B，不是代理方法的话，non_proxy_method 就是被调用的ArtMethod* B本身。
  ArtMethod* non_proxy_method = method->GetInterfaceMethodIfProxy(kRuntimePointerSize);
  CodeItemDataAccessor accessor(non_proxy_method->DexInstructionData());
  const char* shorty = non_proxy_method->GetShorty(&shorty_len);

  JValue result;
  if (UNLIKELY(deopt_frame != nullptr)) {
    ...//HDeoptimize相关，暂不关注
  } else {
    //创建ArtMethod* B的栈帧shadow_frame
    ShadowFrameAllocaUniquePtr shadow_frame_unique_ptr =
        CREATE_SHADOW_FRAME(num_regs, /* link */ nullptr, method, /* dex pc */ 0);
    ShadowFrame* shadow_frame = shadow_frame_unique_ptr.get();
    size_t first_arg_reg = accessor.RegistersSize() - accessor.InsSize();
    //借助BuildQuickShadowFrameVisitor将调用参数放到shadow_frame对象中
    BuildQuickShadowFrameVisitor shadow_frame_builder(sp, method->IsStatic(), shorty, shorty_len,
                                                      shadow_frame, first_arg_reg);
    shadow_frame_builder.VisitArguments();

    //判断ArtMethod* B所属类是否初始化
    const bool needs_initialization =
        method->IsStatic() && !method->GetDeclaringClass()->IsInitialized();
    
    //栈管理相关
    self->PushManagedStackFragment(&fragment);
    self->PushShadowFrame(shadow_frame);
    //初始化，类初始化就是调用ClassLinker的EnsureInitialized函数。
    if (needs_initialization) {
      StackHandleScope<1> hs(self);
      Handle<mirror::Class> h_class(hs.NewHandle(shadow_frame->GetMethod()->GetDeclaringClass()));
      if (!Runtime::Current()->GetClassLinker()->EnsureInitialized(self, h_class, true, true)) {
        self->PopManagedStackFragment(fragment);
        return 0;
      }
    }
    //解释执行的入口函数
    result = interpreter::EnterInterpreterFromEntryPoint(self, accessor, shadow_frame);
  }
  //栈管理相关
  self->PopManagedStackFragment(fragment);

  return result.GetJ();
}
```

art/runtime/interpreter/interpreter.cc：



```c
JValue EnterInterpreterFromEntryPoint(Thread* self, const CodeItemDataAccessor& accessor,
                                      ShadowFrame* shadow_frame) {
  //JIT相关
  jit::Jit* jit = Runtime::Current()->GetJit();
  if (jit != nullptr) {
    jit->NotifyCompiledCodeToInterpreterTransition(self, shadow_frame->GetMethod());
  }
  //关键函数
  return Execute(self, accessor, *shadow_frame, JValue());
}
```

解释执行的实现方式有2种，由kInterpreterImplKind的值控制：
 kMterpImplKind：根据不同CPU平台，采用汇编语言实现。默认值。
 kSwitchImplKind ：由C++编写，基于switch/case实现。



```c
static inline JValue Execute(
    Thread* self,
    const CodeItemDataAccessor& accessor,
    ShadowFrame& shadow_frame,
    JValue result_register,
    bool stay_in_interpreter = false) REQUIRES_SHARED(Locks::mutator_lock_) {
  ArtMethod* method = shadow_frame.GetMethod();
  //transaction_active 和dex2oat有关，完整虚拟机返回false。
  bool transaction_active = Runtime::Current()->IsActiveTransaction();
  if (LIKELY(method->SkipAccessChecks())) {
    if (kInterpreterImplKind == kMterpImplKind) {
        //汇编实现
        ExecuteMterpImpl(self, accessor.Insns(), &shadow_frame, &result_register);
    } else {
        //C++ switch/case实现
        return ExecuteSwitchImpl<false, false>(self, accessor, shadow_frame, result_register, false);
    }
  }
}
```

ExecuteSwitchImplCpp处理逻辑非常工整，一种dex指令对应switch中的一种case。
 art/runtime/interpreter/interpreter_switch_impl.cc:



```c
void ExecuteSwitchImplCpp(SwitchImplContext* ctx) {
  //dex_pc指向要执行的dex指令
  uint32_t dex_pc = shadow_frame.GetDexPC();
  //insns代表方法的dex指令码数组
  const uint16_t* const insns = accessor.Insns();
  const Instruction* inst = Instruction::At(insns + dex_pc);
  uint16_t inst_data;
  do {
    //遍历dex指令码数组
    dex_pc = inst->GetDexPc(insns);
    shadow_frame.SetDexPC(dex_pc);
    TraceExecution(shadow_frame, inst, dex_pc);
    inst_data = inst->Fetch16(0);
    //针对每一种dex指令处理。每种dex之前，都有一个PREAMBLE宏，就是调用instrumentation的DexPcMovedEvent函数。
    switch (inst->Opcode(inst_data)) {
      case Instruction::NOP:
        PREAMBLE();
        inst = inst->Next_1xx();
        break;
      case Instruction::INVOKE_DIRECT: {
        PREAMBLE();
        //调用DoInvoke
        bool success = DoInvoke<kDirect, false, do_access_check>(
            self, shadow_frame, inst, inst_data, &result_register);
        POSSIBLY_HANDLE_PENDING_EXCEPTION(!success, Next_3xx);
        break;
      }
  } while (!interpret_one_instruction);
  //记录dex指令执行的位置，并更新到shadow_frame中
  shadow_frame.SetDexPC(inst->GetDexPc(insns));
  ctx->result = result_register;
  return;
}
```

DoInvoke是一个模板函数，能处理invoke-direct/static/super/virtual/interface等指令。
 现在我们处于方法B中，接下来看B如何调用C，对应ArtMethod* C。



```c
template<InvokeType type, bool is_range, bool do_access_check>
static inline bool DoInvoke(Thread* self,
                            ShadowFrame& shadow_frame,
                            const Instruction* inst,
                            uint16_t inst_data,
                            JValue* result) {
  //method_idx为方法c在dex文件里method_ids数组中的索引
  const uint32_t method_idx = (is_range) ? inst->VRegB_3rc() : inst->VRegB_35c();
  //找到方法c对应的对象。它作为参数存储在方法B的ShadowFrame对象中。
  const uint32_t vregC = (is_range) ? inst->VRegC_3rc() : inst->VRegC_35c();
  ObjPtr<mirror::Object> receiver =
      (type == kStatic) ? nullptr : shadow_frame.GetVRegReference(vregC);
  //sf_method代表ArtMethod* B
  ArtMethod* sf_method = shadow_frame.GetMethod();
  //FindMethodFromCode查找目标方法c对应的ArtMethod对象，即ArtMethod* C。
  ArtMethod* const called_method = FindMethodFromCode<type, do_access_check>(
      method_idx, &receiver, sf_method, self);

  if (UNLIKELY(called_method == nullptr)) {
  } else if (UNLIKELY(!called_method->IsInvokable())) {
  } else {
    ...
    return DoCall<is_range, do_access_check>(called_method, self, shadow_frame, inst, inst_data,
                                             result);
  }
}
```

DoCall又进一步调用DoCallCommon。
 art/runtime/interpreter/interpreter_common.cc:



```cpp
template <bool is_range,
          bool do_assignability_check>
static inline bool DoCallCommon(ArtMethod* called_method,
                                Thread* self,
                                ShadowFrame& shadow_frame,
                                JValue* result,
                                uint16_t number_of_inputs,
                                uint32_t (&arg)[Instruction::kMaxVarArgRegs],
                                uint32_t vregC) {
  //创建方法c所需的ShadowFrame对象
  ShadowFrameAllocaUniquePtr shadow_frame_unique_ptr =
      CREATE_SHADOW_FRAME(num_regs, &shadow_frame, called_method, /* dex pc */ 0);
  ShadowFrame* new_shadow_frame = shadow_frame_unique_ptr.get();

  if (do_assignability_check) {
    ...//不考虑此种情况
  } else {
    //从B的ShadowFrame中拷贝C所需参数到C的ShadowFrame对象中
    CopyRegisters<is_range>(shadow_frame,
                            new_shadow_frame,
                            arg,
                            vregC,
                            first_dest_reg,
                            number_of_inputs);
  }
  //跳转到方法C
  PerformCall(self,
              accessor,
              shadow_frame.GetMethod(),
              first_dest_reg,
              new_shadow_frame,
              result,
              use_interpreter_entrypoint);

  return !self->IsExceptionPending();
}
```

art/runtime/common_dex_operations.h:



```c
inline void PerformCall(Thread* self,
                        const CodeItemDataAccessor& accessor,
                        ArtMethod* caller_method,
                        const size_t first_dest_reg,
                        ShadowFrame* callee_frame,
                        JValue* result,
                        bool use_interpreter_entrypoint)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  if (LIKELY(Runtime::Current()->IsStarted())) {
    if (use_interpreter_entrypoint) {
      //解释模式，debug或者机器码不存在，调用ArtInterpreterToInterpreterBridge
      interpreter::ArtInterpreterToInterpreterBridge(self, accessor, callee_frame, result);
    } else {
      //以机器码执行方法C，调用ArtInterpreterToCompiledCodeBridge
      interpreter::ArtInterpreterToCompiledCodeBridge(
          self, caller_method, callee_frame, first_dest_reg, result);
    }
  } else {
    interpreter::UnstartedRuntime::Invoke(self, accessor, callee_frame, result, first_dest_reg);
  }
}
```

art/runtime/interpreter/interpreter.cc:



```c
void ArtInterpreterToInterpreterBridge(Thread* self,
                                       const CodeItemDataAccessor& accessor,
                                       ShadowFrame* shadow_frame,
                                       JValue* result) {
  self->PushShadowFrame(shadow_frame);//栈管理相关
  EnsureInitialized();//static的话，是否初始化
  if (LIKELY(!shadow_frame->GetMethod()->IsNative())) {
    //如果不是JNI方法，调用Execute执行。又回到上述描述调用流程。
    result->SetJ(Execute(self, accessor, *shadow_frame, JValue()).GetJ());
  } 
  self->PopShadowFrame();//栈管理相关
}
```

art/runtime/interpreter/interpreter_common.cc:



```c
void ArtInterpreterToCompiledCodeBridge(Thread* self,
                                        ArtMethod* caller,
                                        ShadowFrame* shadow_frame,
                                        uint16_t arg_offset,
                                        JValue* result)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  ArtMethod* method = shadow_frame->GetMethod();
  EnsureInitialized();//static的话，是否初始化

  //又回到上述调用流程，Invoke最终调用到entry_point_from_quick_compiled_code_
  method->Invoke(self, shadow_frame->GetVRegArgs(arg_offset),
                 (shadow_frame->NumberOfVRegs() - arg_offset) * sizeof(uint32_t),
                 result, method->GetInterfaceMethodIfProxy(kRuntimePointerSize)->GetShorty());
}
```

