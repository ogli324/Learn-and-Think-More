# `使用frida来hook artmethod的RegisterNative

url：http://missking.cc/2020/09/02/Frida-Native-Register/



继续学习记录

之前找到了最终有调用到ArtMethod的RegisterNative来进行绑定。现在使用frida来hook打印一下注册的函数名和地址。先用ida找到我们之前找到的ArtMethod下的RegisterNative

找到了导出函数符号`_ZN3art9ArtMethod14RegisterNativeEPKvb`

然后这里只有ArtMethod的指针，要获取完整的函数名。要使用ArtMethod的函数PrettyMethod。

导出函数符号为 `_ZN3art9ArtMethod12PrettyMethodEb`

然后frida的代码就是先hook到RegisterNative，然后调用PrettyMethod来获取函数名



```
function hook_native_register(){
    var module= Process.getModuleByName("libart.so");
    var ArtMethodRegisterNative=module.getExportByName("_ZN3art9ArtMethod14RegisterNativeEPKvb");
    Interceptor.attach(ArtMethodRegisterNative,{
        onEnter:function(args){
            var curArtMethod=ptr(args[0]);
            var native_method=args[1];
            var method_idx=ptr(curArtMethod).add(12).readU32();
            var prettyMethodPtr=Module.getExportByName("libart.so","_ZN3art9ArtMethod12PrettyMethodEb");
            var result=Memory.alloc(0x100);
            var prettyMethod=new NativeFunction(prettyMethodPtr,'pointer',['pointer','pointer','bool']);
            prettyMethod(ptr(result),ptr(curArtMethod),1);
            var methodName=result.add(0x8).readPointer().readCString()
            console.log(",method_idx:",method_idx,"native_ptr:",native_method,",methodName:",methodName);
        },
        onLeave:function(retval){

        }
    })
}

function main(){
    hook_native_register();
}

setImmediate(main);
```

关键要注意的一个地方是prettyMethod的调用，它是一个特殊的调用，类和结构的返回值都会进行特殊的方式返回。这里返回的是一个string，所以使用了特殊方式进行返回结果。默认新加一个参数在最前面，然后将返回值填充到第一个参数里面。而不是直接返回。下面放一下frida文档的原文

```
As for structs or classes passed by value, instead of a string provide an array containing the struct’s field types following each other. You may nest these as deep as desired for representing structs inside structs. Note that the returned object is also a NativePointer, and can thus be passed to Interceptor#attach.This must match the struct/class exactly, so if you have a struct with three ints, you must pass ['int', 'int', 'int']For a class that has virtual methods, the first parameter will be a pointer to the vtable.For C++ scenarios involving a return value that is larger than Process.pointerSize, a NativePointer to preallocated space must be passed in as the first parameter. (This scenario is common in WebKit, for example.)
```

这里原本执行是

```
var prettyMethod=new NativeFunction(prettyMethodPtr,'pointer',['pointer','bool']);
```

特殊处理后变成了

```
var prettyMethod=new NativeFunction(prettyMethodPtr,'pointer',['pointer','pointer','bool']);
```

最后frida启动测试

```
frida -U --no-pause -f com.example.classloadertest -l mydemo1.js
```

下面是运行结果

```
method_idx: 131 native_ptr: 0xcb4ddea5 ,methodName: java.lang.String com.stub.StubApp.interface14(int)
method_idx: 141 native_ptr: 0xcb516d41 ,methodName: void com.stub.StubApp.mark()
method_idx: 142 native_ptr: 0xcb516a75 ,methodName: void com.stub.StubApp.mark(android.location.Location)
method_idx: 140 native_ptr: 0xcb516979 ,methodName: android.location.Location com.stub.StubApp.mark(android.location.LocationManager, java.lang.String)
method_idx: 136 native_ptr: 0xcb4ddf05 ,methodName: void com.stub.StubApp.interface5(android.app.Application)
method_idx: 128 native_ptr: 0xcb4eb3d5 ,methodName: void com.stub.StubApp.interface11(int)
method_idx: 129 native_ptr: 0xcb4de28d ,methodName: java.util.Enumeration com.stub.StubApp.interface12(dalvik.system.DexFile)
method_idx: 138 native_ptr: 0xcb4e0b19 ,methodName: boolean com.stub.StubApp.interface7(android.app.Application, android.content.Context)
method_idx: 139 native_ptr: 0xcb4de721 ,methodName: boolean com.stub.StubApp.interface8(android.app.Application, android.content.Context)
method_idx: 5228 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.String com.jg.bh.BH.getCloundTag(android.content.Context)
method_idx: 5229 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.String com.jg.bh.BH.getMsg(android.content.Context)
method_idx: 5230 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.BH.install(android.content.Context)
method_idx: 5233 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.BH.updateConfig(android.content.Context, java.lang.String)
method_idx: 5358 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.String com.jg.bh.proxy.MyProxy.getEnable(android.content.Context)
method_idx: 5359 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.String com.jg.bh.proxy.MyProxy.getMetaRow(android.content.Context)
method_idx: 5360 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.Object com.jg.bh.proxy.MyProxy.getProxyObject(java.lang.Object, java.lang.reflect.InvocationHandler)
method_idx: 5361 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.String com.jg.bh.proxy.MyProxy.getVersion(android.content.Context)
method_idx: 5362 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.proxy.MyProxy.installAll(android.content.Context)
method_idx: 5363 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.Object com.jg.bh.proxy.MyProxy.newProxyInstance(java.lang.ClassLoader, java.lang.Class[], java.lang.reflect.InvocationHandler)
method_idx: 5364 native_ptr: 0xcb4ec3f1 ,methodName: org.json.JSONObject com.jg.bh.proxy.MyProxy.parseConfig(java.lang.String)
method_idx: 5365 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.proxy.MyProxy.putBHString(android.content.Context, java.lang.String)
method_idx: 5366 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.proxy.MyProxy.putConfig(android.content.Context, java.lang.String)
method_idx: 5399 native_ptr: 0xcb4ec3f1 ,methodName: boolean com.jg.bh.util.MetaRow.access$000(com.jg.bh.util.MetaRow)
method_idx: 5400 native_ptr: 0xcb4ec3f1 ,methodName: boolean com.jg.bh.util.MetaRow.access$002(com.jg.bh.util.MetaRow, boolean)
method_idx: 5401 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.access$100(com.jg.bh.util.MetaRow, org.json.JSONObject)
method_idx: 5402 native_ptr: 0xcb4ec3f1 ,methodName: org.json.JSONObject com.jg.bh.util.MetaRow.getCellList()
method_idx: 5403 native_ptr: 0xcb4ec3f1 ,methodName: org.json.JSONObject com.jg.bh.util.MetaRow.getCelllData(android.telephony.CellInfo)
method_idx: 5405 native_ptr: 0xcb4ec3f1 ,methodName: org.json.JSONObject com.jg.bh.util.MetaRow.getWiFiInfo()
method_idx: 5406 native_ptr: 0xcb4ec3f1 ,methodName: org.json.JSONObject com.jg.bh.util.MetaRow.getWifiScanList()
method_idx: 5417 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.syncJsonData(org.json.JSONObject)
method_idx: 5404 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.String com.jg.bh.util.MetaRow.getNetConnType()
method_idx: 5407 native_ptr: 0xcb4ec3f1 ,methodName: boolean com.jg.bh.util.MetaRow.isGlobalVpn()
method_idx: 5408 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setCellInfoList(java.util.List)
method_idx: 5409 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setConnWifi(android.net.wifi.WifiInfo)
method_idx: 5410 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setData(java.lang.String)
method_idx: 5411 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setDhcpInfo(android.net.DhcpInfo)
method_idx: 5412 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setLocation(android.location.Location)
method_idx: 5413 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setWifiConf(java.lang.Object)
method_idx: 5414 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setWifiScanner(java.util.List)
method_idx: 5415 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setmNISubtype(int)
method_idx: 5416 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.util.MetaRow.setmNIType(int)
method_idx: 5418 native_ptr: 0xcb4ec3f1 ,methodName: org.json.JSONObject com.jg.bh.util.MetaRow.toJSON()
method_idx: 5257 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.hook.binder.BaseServiceHook.onInstall()
method_idx: 5301 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.hook.binder.TelephonyHook.access$000(com.jg.bh.hook.binder.TelephonyHook, java.lang.Object)
method_idx: 5302 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.hook.binder.TelephonyHook.access$100(com.jg.bh.hook.binder.TelephonyHook, java.lang.Object)
method_idx: 5303 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.hook.binder.TelephonyHook.callGetAllCellInfo(java.lang.Object)
method_idx: 5304 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.hook.binder.TelephonyHook.callGetCellLocation(java.lang.Object)
method_idx: 5305 native_ptr: 0xcb4ec3f1 ,methodName: com.jg.bh.hook.binder.BinderProxyHookHandler$IInterfaceHookHandler com.jg.bh.hook.binder.TelephonyHook.createBinderProxyHookHandler(android.os.IBinder, java.lang.Class)
method_idx: 5265 native_ptr: 0xcb4ec3f1 ,methodName: com.jg.bh.hook.binder.BinderProxyHookHandler$IInterfaceHookHandler com.jg.bh.hook.binder.BinderProxyHookHandler.getmInterfacehandler()
method_idx: 5266 native_ptr: 0xcb4ec3f1 ,methodName: java.lang.Object com.jg.bh.hook.binder.BinderProxyHookHandler.invoke(java.lang.Object, java.lang.reflect.Method, java.lang.Object[])
method_idx: 5267 native_ptr: 0xcb4ec3f1 ,methodName: void com.jg.bh.hook.binder.BinderProxyHookHandler.setmInterfacehandler(com.jg.bh.hook.binder.BinderProxyHookHandler$IInterfaceHookHandler)
method_idx: 5273 native_ptr: 0xcb4ec3f1 ,methodName: com.jg.bh.hook.binder.BinderProxyHookHandler$IInterfaceHookHandler com.jg.bh.hook.binder.IConnectivityManagerHook.createBinderProxyHookHandler(android.os.IBinder, java.lang.Class)
method_idx: 5280 native_ptr: 0xcb4ec3f1 ,methodName: com.jg.bh.hook.binder.BinderProxyHookHandler$IInterfaceHookHandler com.jg.bh.hook.binder.ILocationManagerHook.createBinderProxyHookHandler(android.os.IBinder, java.lang.Class)
method_idx: 5287 native_ptr: 0xcb4ec3f1 ,methodName: com.jg.bh.hook.binder.BinderProxyHookHandler$IInterfaceHookHandler com.jg.bh.hook.binder.IWifiManagerHook.createBinderProxyHookHandler(android.os.IBinder, java.lang.Class)
method_idx: 5294 native_ptr: 0xcb4ec3f1 ,methodName: com.jg.bh.hook.binder.BinderProxyHookHandler$IInterfaceHookHandler com.jg.bh.hook.binder.IWifiScannerHook.createBinderProxyHookHandler(android.os.IBinder, java.lang.Class)
method_idx: 11474 native_ptr: 0xcb4defb9 ,methodName: java.lang.String[] dalvik.system.DexFile.getClassNameList(java.lang.Object)
method_idx: 5172 native_ptr: 0xcb4dd59d ,methodName: java.lang.String com.common.busi.CustomView.getAppkey()
method_idx: 5173 native_ptr: 0xcb4dd78d ,methodName: long com.common.busi.CustomView.getPT()
method_idx: 5451 native_ptr: 0xcb4dd8a9 ,methodName: int com.jg.ce.Interface2.interface1(android.content.Context)
method_idx: 5452 native_ptr: 0xcb4dd8b7 ,methodName: int com.jg.ce.Interface2.interface2(android.content.Context)
method_idx: 5453 native_ptr: 0xcb4dd8bb ,methodName: int com.jg.ce.Interface2.interface3(android.content.Context)
method_idx: 5454 native_ptr: 0xcb4dd8c1 ,methodName: int com.jg.ce.Interface2.interface4(android.content.Context)
method_idx: 5455 native_ptr: 0xcb4dd8c5 ,methodName: int com.jg.ce.Interface2.interface5(android.content.Context)
method_idx: 5643 native_ptr: 0xcb4db755 ,methodName: void com.stub.stub07.Stub01.mark1(java.lang.String)
method_idx: 5231 native_ptr: 0xcb4db8d9 ,methodName: void com.jg.bh.BH.mark2(java.lang.String, double, double)
method_idx: 5232 native_ptr: 0xcb52d269 ,methodName: java.lang.String com.jg.bh.BH.mark3()
method_idx: 5196 native_ptr: 0xcb4ec3f1 ,methodName: void com.example.classloadertest.MainActivity.onCreate(android.os.Bundle)
```