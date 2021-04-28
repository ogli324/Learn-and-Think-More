# vmp入门

url：http://missking.cc/2020/11/16/vmp/



记录学习的过程

vmp壳的应用。从效果上面看，是把java的函数，加密成了native的函数，在实现函数加密保护时，主要是对相同参数并且相同返回值的函数加密成native的。所以会一般是两种情况，一个是直接对onCreate函数进行vmp保护，或者是抽象出一个统一参数和返回值的函数。然后对这个函数进行vmp保护。

native的函数都是需要动态注册的。所以我们先改造aosp源码。打印下注册了函数，以及函数的地址，方便我们后续进行调试跟踪，下面修改art_method.cc

```
const void* ArtMethod::RegisterNative(const void* native_method, bool is_fast) {
  CHECK(IsNative()) << PrettyMethod();
  CHECK(!IsFastNative()) << PrettyMethod();
  CHECK(native_method != nullptr) << PrettyMethod();
  if (is_fast) {
    AddAccessFlags(kAccFastNative);
  }
  LOG(ERROR)<<"krom ArtMethod::RegisterNative:"<<this->PrettyMethod()<<"----addr:"<<native_method;

  void* new_native_method = nullptr;
  Runtime::Current()->GetRuntimeCallbacks()->RegisterNativeMethod(this,
                                                                  native_method,
                                                                  /*out*/&new_native_method);
  SetEntryPointFromJni(new_native_method);
  return new_native_method;
}
```

由于vmp中将java函数转成了native函数，所以函数内肯定有大量jni调用处理。所以为了方便分析，再改造下rom，在jni函数调用的时候将触发调用的函数以及被调用的函数，都打印出来。这样就知道加密的一部分jni调用，InvokeWithArgArray这个函数就满足我们的需求，jni函数都会调用的地方。我们在这里插入日志记录即可。



```
static void InvokeWithArgArray(const ScopedObjectAccessAlreadyRunnable& soa,
                               ArtMethod* method, ArgArray* arg_array, JValue* result,
                               const char* shorty)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  uint32_t* args = arg_array->GetArray();
  if (UNLIKELY(soa.Env()->check_jni)) {
    CheckMethodArguments(soa.Vm(), method->GetInterfaceMethodIfProxy(kRuntimePointerSize), args);
  }
  ArtMethod* artmethod=nullptr;
  Thread* thread=Thread::Current;
  ManagedStack* managerStack=thread->GetManagedStack();
  if(managerStack!=nullptr){
      ArtMethod** methodPP=managerStack->GetTopQuickFrame();
      if(methodPP!=nullptr){
          artmethod=*methodPP;
      }
  }

  if(artmethod!=nullptr){
      LOG(ERROR)<<"krom start InvokeWithArgArray call:"<<artmethod->PrettyMethod()<<"----goal:"<<method->PrettyMethod();
  }
  method->Invoke(soa.Self(), args, arg_array->GetNumBytes(), result, shorty);
  if(artmethod!=nullptr){
      LOG(ERROR)<<"krom end InvokeWithArgArray call:"<<artmethod->PrettyMethod()<<"----goal:"<<method->PrettyMethod();
  }

}
```

然后为了方便调试，过掉各种检测，所以先修改aosp源码，能够在想要调试的jni函数执行前进行sleep等待。然后我们再附加进程。这样不在开始的时候直接附加进程，就可以直接跳过很多种动态调试检测。

所有jni函数在执行前后，art会进行一些准备和清场的步骤，比如JniMethodStart和JniMethodEnd。我们可以在JniMethodStart的时候判断一下。符合条件就sleep等待一下。下面贴下修改后的代码，文件路径是/art/runtime/entrypoints/quick/quick_jni_entrypoints.cc

```
// Called on entry to JNI, transition out of Runnable and release share of mutator_lock_.
extern uint32_t JniMethodStart(Thread* self) {
  JNIEnvExt* env = self->GetJniEnv();
  DCHECK(env != nullptr);
  uint32_t saved_local_ref_cookie = bit_cast<uint32_t>(env->local_ref_cookie);
  env->local_ref_cookie = env->locals.GetSegmentState();
  ArtMethod* native_method = *self->GetManagedStack()->GetTopQuickFrame();
  
  const char* methodname=native_method->PrettyMethod();
  //这里虽然是拿函数名判断。但是我们使用的时候，是用frida来hook这个strstr函数。然后再根据第一个参数判断。是想要断点的函数。就直接返回数据
  if(strstr(methodname,"JniMethodStart")!=nullptr){
      sleep(60);
  }

  if (!native_method->IsFastNative()) {
    // When not fast JNI we transition out of runnable.
    self->TransitionFromRunnableToSuspended(kNative);
  }
  return saved_local_ref_cookie;
}
```

最后我们使用frida配合rom来控制函数是否需要进入等待。从而达到跳过各种反调试。在最后jni函数执行前才进行附加调试的效果。

下面贴上配合的frida代码

```
function hook_start(){
    var libcModule=Process.getModuleByName("libc.so");
    var strstr=libcModule.getExportByName("strstr");
    Interceptor.attach(strstr,{
        onEnter:function(args){
            this.arg0=args[0];
            this.arg1=args[1];
            var method_name=ptr(this.arg0).readUtf8String();
            var call_name=ptr(this.arg1).readUtf8String();
            if(call_name.indexOf("JniMethodStart")!=-1 && method_name.indexOf("MainActivity.onCreate")!=-1){
                console.log("JniMethodStart onCreate enter");
                this.retval="1";
            }
        },onLeave:function(retval){
            if (this.retval){
                retval.replace(this.retval);
            }
        }
    })
}

function main(){
    hook_start();
}

setImmediate(main)
```

然后启动frida。我这里环境使用的xadb。所以可以简单的直接打开frida