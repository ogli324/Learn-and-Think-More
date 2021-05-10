# 深入Android系统（九）Android系统的核心-SystemServer进程

url：https://juejin.cn/post/6910805095294697479



`SystemServer`是`Android`系统的核心之一，大部分`Android`提供的服务都运行在这个进程里。

为了防止应用进程对系统造成破坏，`Android`的应用进程没有权限直接访问设备的底层资源，只能通过`SystemServer`中的服务代理访问。

本篇重点是了解`SystemServer`的启动过程以及它的`Watchdog`模块

# `SystemServer`的创建过程

`SystemServer`的创建可以分为两部分：

- 在`Zygote`进程中`fork`并初始化`SystemServer`进程的过程
- 执行`SystemServer`类的`main`方法来启动系统服务的过程

## 创建`SystemServer`进程

在`Zygote进程`一篇中我们已经知道`init.rc`文件中定义了`Zygote进程`的启动参数，如下：

```rc
service zygote /system/bin/app_process32 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    ......
复制代码
```

其中定义了`--start-system-server`参数，因此，在`ZygoteInit`类的`main`方法中，会执行到`forkSystemServer()`函数：

```java
if (startSystemServer) {
    Runnable r = forkSystemServer(abiList, socketName, zygoteServer);
    // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
    // child (system_server) process.
    if (r != null) {
        r.run();
        return;
    }
}
复制代码
```

`forkSystemServer()`函数如下：

```java
    private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
        ......
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1024,1032,1065,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
            "com.android.server.SystemServer",
        };
        ......
        int pid;
        try {
            ......
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.runtimeFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }
        /* For child process */
        if (pid == 0) {
            ......
            zygoteServer.closeServerSocket();
            return handleSystemServerProcess(parsedArgs);
        }
        return null;
    }
复制代码
```

在上面的方法中，主要做了如下事项：

- 准备

  ```
  Systemserver
  ```

  的启动参数：

  - `进程ID` 和 `组ID`设置为 `1000`
  - 设定进程名称为`system_server`
  - 指定`Systemserver`的执行类`com.android.server.SystemServer`

- 调用

  ```
  Zygote
  ```

  类的

  ```
  forkSystemServer()
  ```

  函数

  ```
  fork
  ```

  出

  ```
  SystemServer
  ```

  进程

  - `Zygote`类也是通过`native层`的函数来完成实际的工作，函数如下

  ```c
  static jint com_android_internal_os_Zygote_nativeForkSystemServer(...) {
      pid_t pid = ForkAndSpecializeCommon(...);
      if (pid > 0) {
          // The zygote process checks whether the child process has died or not.
          ALOGI("System server process %d has been created", pid);
          gSystemServerPid = pid;
          // There is a slight window that the system server process has crashed
          // but it went unnoticed because we haven't published its pid yet. So
          // we recheck here just to make sure that all is well.
          int status;
          if (waitpid(pid, &status, WNOHANG) == pid) {
              ALOGE("System server process %d has died. Restarting Zygote!", pid);
              RuntimeAbort(env, __LINE__, "System server process has died. Restarting Zygote!");
          }
          ......
      }
      return pid;
  }
  复制代码
  ```

  - 从`native层`的函数来看，对于`SystemServer`进程的`fork`，`Zygote`进程会通过`waitpid()`函数来检查`SystemServer`进程是否启动成功，如果不成功 ，`Zygote`进程会退出重启
  - 相关的细节在`native层`的`ForkAndSpecializeCommon()`中：

  ```c
  static pid_t ForkAndSpecializeCommon(......) {
      SetSignalHandlers();
      ......
  }
  static void SetSignalHandlers() {
      struct sigaction sig_chld = {};
      sig_chld.sa_handler = SigChldHandler;
      ......
  }
  // This signal handler is for zygote mode, since the zygote must reap its children
  static void SigChldHandler(int /*signal_number*/) {
      pid_t pid;
      ......
      while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
          ......
          if (pid == gSystemServerPid) {
              ALOGE("Exit zygote because system server (%d) has terminated", pid);
              kill(getpid(), SIGKILL);
          }
      }
      ......
  }
  复制代码
  ```

- ```
  SystemServer
  ```

  进程

  ```
  fork
  ```

  后，在

  ```
  fork
  ```

  出的子进程中：

  - 先关闭从`Zygote进程`继承来的`socket`
  - 接着调用`handleSystemServerProcess()`来初始化`SystemServer`进程，流程如下：

  ```java
  /**
   * Finish remaining work for the newly forked system server process.
   */
  private static Runnable handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
      // set umask to 0077 so new files and directories will default to owner-only permissions.
      // 设置umask为0077（权限的补码）
      // 这样SystemServer创建的文件属性就是0700，只有进程本身可以访问
      Os.umask(S_IRWXG | S_IRWXO);
      ......
      if (parsedArgs.invokeWith != null) {
          // 一般情况不会走到这里，invokeWith基本上都为null
          ......
          // 通过 app_process 来启动，进程不会return
          WrapperInit.execApplication(parsedArgs.invokeWith,
                  parsedArgs.niceName, parsedArgs.targetSdkVersion,
                  VMRuntime.getCurrentInstructionSet(), null, args);
  
          throw new IllegalStateException("Unexpected return from WrapperInit.execApplication");
      } else {
          ......
          // 通过查找启动类的main方法，然后打包成Runnable对象返回
          return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
      }
      /* should never reach here */
  }
  复制代码
  ```

  - `invokeWith`通常为`null`，所以基本都是通过`ZygoteInit.zygoteInit()`函数来处理，返回打包好的`Runnable`对象。后面就是执行`r.run()`来调用`SystemServer`的`main()`方法了

我们看看`SystemServer`的`main()`方法干了啥

## `SystemServer`初始化

`SystemServer`的`main()`方法如下：

```java
    public static void main(String[] args) {
        new SystemServer().run();
    }
复制代码
```

`main()`方法中创建了一个`SystemServer`的对象并调用`run()`函数，函数代码如下，删减版：

```java
    private void run() {
        // System Server 启动前的准备阶段
        try {
            // 设置系统时间
            if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
                Slog.w(TAG, "System clock is before 1970; setting to 1970.");
                SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
            }
            ...... // 省略时区，语言、国家的设置
            ...... // 省略 Binder、Sqlite相关属性设置
            // Here we go!
            Slog.i(TAG, "Entered the Android system server!");
            int uptimeMillis = (int) SystemClock.elapsedRealtime();
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, uptimeMillis);
            ......
            // 设置当前虚拟机的运行库路径
            SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());
            // 调整内存
            VMRuntime.getRuntime().clearGrowthLimit();
            VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
            ...... //省略一些系统属性的设置
            // 设置进程相关属性
            android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
            android.os.Process.setCanSelfBackground(false);
            // 初始化Looper相关
            Looper.prepareMainLooper();
            Looper.getMainLooper().setSlowLogThresholdMs(SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);
            // 加载libandroid_services.so
            System.loadLibrary("android_servers");
            .....
            // Initialize the system context.
            createSystemContext();
            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,mRuntimeStartElapsedTime, mRuntimeStartUptime);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
            ......
        } finally {
            traceEnd();  // InitBeforeStartServices
        }
        // System Server 的启动阶段
        try {
            traceBeginAndSlog("StartServices");
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
            ......
        } catch (Throwable ex) {
            ......
            throw ex;
        } finally {
            traceEnd();
        }
        ......
        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
复制代码
```

`run()`方法主要分为两个阶段：

- 一个是`SystemServer`启动前的准备阶段，主要进行了：
  - 系统时间、时区、语言的设置
  - 虚拟机运行库、内存参数、内存利用率的调整
  - 设置当前线程的优先级
  - 加载`libandroid_services.so`库
  - 初始化`Looper`
  - `SystemContext`与`SystemServiceManager`的初始化
- 另一个是`SystemServer`的启动阶段：
  - 执行`startBootstrapServices`、`startCoreServices`、`startOtherServices`启动所有的系统服务
  - 调用`Looper.loop()`循环处理消息

我们先来看下准备阶段`SystemContext`的初始化过程

### `createSystemContext()`初始化

函数如下：

```java
    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
        
        final Context systemUiContext = activityThread.getSystemUiContext();
        systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
    }
复制代码
```

`createSystemContext()`方法的流程比较简单：

- 通过`ActivityThread`的静态方法`systemMain()`来得到一个`ActivityThread`对象
- 然后调用`ActivityThread`对象的`getSystemContext()/getSystemUiContext()`方法来得到对应的`Context`对象
- 通过`Context`对象设置一个默认的`theme`

所以重点还是`ActivityThread.systemMain()`方法：

```java
    public static ActivityThread systemMain() {
        // The system process on low-memory devices do not get to use hardware
        // accelerated drawing, since this can add too much overhead to the
        // process.
        if (!ActivityManager.isHighEndGfx()) {
            ThreadedRenderer.disable(true);
        } else {
            ThreadedRenderer.enableForegroundTrimming();
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(true, 0);
        return thread;
    }
复制代码
```

方法内容并不复杂，首先判断是否需要启动硬件渲染，然后创建了一个`ActivityThread`对象。

关于`ActivityThread`，在`Zygote进程`学习的时候我们已经知道：

- `ActivityThread`是应用程序的主线程类
- 启动应用最后执行到的就是`ActivityThread`的`main()`方法

**关键是`SystemServer`为什么要创建`ActivityThread`对象呢?**
 实际上`SystemServer`不仅仅是一个单纯的后台进程，它也是一个运行着组件`Service`的进程，很多系统对话框就是从`SystemServer`中显示出来的，因此`SystemServer`本身也需要一个和`APK`应用类似的上下文环境，创建`ActivityThread`是获取这个环境的第一步

但是`ActivityThread`在`SystemServer`进程中与普通进程还是有区别的，这里主要是通过`attach(boolen system,int seq)`方法的参数`system`来标识，函数如下：

```java
    // system = true 表示这是在 SystemServer 中创建的
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            // 正常应用的启动才会走到这里
            ......
        } else {
            // Don't set application object here -- if the system crashes,
            // we can't display an alert, we just want to die die die.
            // 设置在DDMS中的应用名称
            android.ddm.DdmHandleAppName.setAppName("system_process",UserHandle.myUserId());
            try {
                // 创建Instrumentation对象
                // 一个ActivityThread对应一个，可以监视应用的生命周期
                mInstrumentation = new Instrumentation();
                mInstrumentation.basicInit(this);
                // 创建Context对象
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                mInitialApplication.onCreate();
            } catch (Exception e) {
                throw new RuntimeException(
                        "Unable to instantiate Application():" + e.toString(), e);
            }
        }
复制代码
```

`attach()`方法在`system`为`true`的情况下：

- 设置进程在`DDMS`中的名称
- 创建了`ContextImpl`和`Application`对象
- 最后调用了`Application`对象的`onCreate`方法

**这里在模仿应用启动有木有，是哪一个`APK`呢？**
 在`ContextImpl`对象创建时，使用的是`getSystemContext().mPackageInfo`来获取的`Apk`信息，我们看看：

```java
    public ContextImpl getSystemContext() {
        synchronized (this) {
            if (mSystemContext == null) {
                mSystemContext = ContextImpl.createSystemContext(this);
            }
            return mSystemContext;
        }
    }
复制代码
```

再看下`createSystemContext()`方法：

```java
    static ContextImpl createSystemContext(ActivityThread mainThread) {
        LoadedApk packageInfo = new LoadedApk(mainThread);
        ......
        return context;
    }
复制代码
```

`createSystemContext()`方法创建了一个`LoadedApk`的对象，也就是`mPackageInfo`，我们来看看它只有一个参数的构造方法：

```java
    LoadedApk(ActivityThread activityThread) {
        mActivityThread = activityThread;
        mApplicationInfo = new ApplicationInfo();
        mApplicationInfo.packageName = "android";
        mPackageName = "android"; // 设置包名为 android
        ......
    }
复制代码
```

`LoadedApk`对象用来保存一个已经加载了的`apk`文件的信息，上面的构造方法将使用的包名指定为`android`。

**还记得`Android资源管理篇`的`framewok-res.apk`么，它的包名就是`android`。。。**

`getSystemContext().mPackageInfo`指明的是`framwork-res.apk`的信息

> 还要记得，`framwork-res.apk`的相关资源 在`Zygote进程`中的`preloading`环节就已经加载完成了哟

所以，`createSystemContext()`方法的初始化相当于创建了一个`framwork-res.apk`的上下文环境，然后再进行相应的`Theme`设置

而对于`getSystemUiContext()`和`getSystemContext()`这两个方法，我们查看`ActivityThread`的成员变量就会发现分别对应了`mSystemUiContext`和`mSystemContext`。这部分在`5.0`的源码中还未出现，应该是`Android`后面进行了优化拆分，暂不追究，哈哈哈

到这里，`SystemServer`准备阶段的工作已经完成，后面就是启动一系列的`Service`了

## `SystemServer`启动阶段

启动阶段分成了三步：

```java
startBootstrapServices();
startCoreServices();
startOtherServices();
复制代码
```

### `startBootstrapServices()`

`startBootstrapServices()`启动的都是一些很基础关键的服务，这些服务都有很复杂的相互依赖性，我们来简单看下：

```java
    private void startBootstrapServices() {
        ......
        // Installer 会通过binder关联 installd 服务
        // installd 在Init进程中就会被启动，很重要，所以放在第一位
        // installd 支持很多指令，像ping、rename都是在其中定义的
        // 后面在APK安装篇幅单独介绍
        Installer installer = mSystemServiceManager.startService(Installer.class);
        traceEnd();
        ......
        // 添加设备标识符访问策略服务
        mSystemServiceManager.startService(DeviceIdentifiersPolicyService.class);
        ......
        // 启动 ActivityManagerService，并进行一些关联操作
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        
        // 启动 PowerManagerService
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
        ......
        // 初始化mActivityManagerServicede的电源管理相关功能
        mActivityManagerService.initPowerManagement();
        ......
        // 启动 RecoverySystemService
        // recovery system也是很重要的一个服务
        // 可以触发ota，设置或清除bootloader相关的数据
        mSystemServiceManager.startService(RecoverySystemService.class);
        ......
        // 标记启动事件
        RescueParty.noteBoot(mSystemContext);
        // 启动LightsService，管理LED、背光显示等
        mSystemServiceManager.startService(LightsService.class);
        // 通知所有服务当前状态
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
        ......// 省略一些看上去不重要的service
        // 启动PackageManagerService
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();
        ......
        // 启动 OverlayManagerService
        OverlayManagerService overlayManagerService = new OverlayManagerService(
                mSystemContext, installer);
        mSystemServiceManager.startService(overlayManagerService);
        ......
        // 启动 SensorService 这是一个native方法
        // SensorService 提供的是各种传感器的服务
        startSensorService();
    }
复制代码
```

单单`startBootstrapServices`启动的服务就不止`10`种，我们先总结上面的代码规律。

可以发现，服务的启动基本上都是通过`SystemServiceManager`的`startService()`方法。跟踪代码我们会找到`SystemServiceManager`的一个成员变量`ArrayList<SystemService> mServices`。

`SystemServiceManager`管理的其实就是`SystemService`的集合，`SystemService`是一个抽象类

**如果我们想定义一个系统服务，就需要实现`SystemService`这个抽象类**

`PackageManagerService`有点特殊，不过这并不影响这种设计模式，后面的章节会单独学习到它的独特之处

### `startCoreServices()`

我们接着看下`startCoreServices()`，相对简洁了很多：

```java
    private void startCoreServices() {
        // Tracks the battery level.  Requires LightService.
        // 启动电池管理service，依赖bootstrap中的LightService
        // 该服务会定期广播电池的相关状态
        mSystemServiceManager.startService(BatteryService.class);

        // 启动 应用使用情况数据收集服务
        mSystemServiceManager.startService(UsageStatsService.class);
        // ActivityManagerService 又来横插一脚，无处不在的家伙
        mActivityManagerService setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));
        ......
        // 启动WebView service
        mWebViewUpdateService = mSystemServiceManager.startService(WebViewUpdateService.class);
        ......
        // 启动 binder 调用的耗时统计服务
        BinderCallsStatsService.start();
    }
复制代码
```

`startCoreServices()`启动的服务看上去也没那么`核心`不是。

### `startOtherServices()`

再看下`startOtherServices()`足足有`1000`多行，我们找下重点

```java
    /**
     * Starts a miscellaneous grab bag of stuff that has yet to be refactored
     * and organized.
     */
    // google 形容这部分是混乱的，有待整理重构，确实恶心啊
    // 很多服务会依赖其他的服务，比如NotificationManager会依赖StorageManager......
    // 让我们来精简下
    private void startOtherServices() {
        ......// 初始化一些比较基础的服务
        // WindowManagerService、NetworkManagementService
        // WatchDog、NetworkPolicyManagerService等
        
        .....// 初始化 UI 相关的服务
        // InputMethodManagerService、AccessibilityManagerService
        // StorageManagerService、NotificationManagerService
        // UiModeManagerService等
        
        ......// 此处省略了约800行的各式各样的服务的初始化
        // 有点心疼当时写这个方法的前辈，预祝圣诞快乐。。。。

        // These are needed to propagate to the runnable below.
        // 将上面初始化好的一些必要的服务在ActivityManagerService的systemReady中进行进一步的处理
        // 需要进一步处理的服务就是下面这些
        final NetworkManagementService networkManagementF = networkManagement;
        .....
        final IpSecService ipSecServiceF = ipSecService;
        final WindowManagerService windowManagerF = wm;
        // 执行 ActivityManagerService 的 systemReady 方法
        mActivityManagerService.systemReady(() -> {
            ......
            // 标记状态
            mSystemServiceManager.startBootPhase(SystemService.PHASE_ACTIVITY_MANAGER_READY);
            ...... 
            startSystemUi(context, windowManagerF);
            ......// 对前面创建好的服务做进一步的配置，大多是执行一些systemReady操作，如：
            // networkManagementF.systemReady()、ipSecServiceF.systemReady()、networkStatsF.systemReady()
            // connectivityF.systemReady()、networkPolicyF.systemReady(networkPolicyInitReadySignal)
            Watchdog.getInstance().start();
            // Wait for all packages to be prepared
            mPackageManagerService.waitForAppDataPrepared();
            ......
            // 通知所有服务当前状态
            mSystemServiceManager.startBootPhase(SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
            ......// 尝试初次执行一些服务，像定位、地区检测等
            // locationF.systemRunning()、countryDetectorF.systemRunning()、networkTimeUpdaterF.systemRunning()
            ......
        }, BOOT_TIMINGS_TRACE_LOG);
    }
复制代码
```

注释很简洁详细了哈！

### 启动阶段的总结

从`SystemServer`的启动阶段我们可以了解：

- `SystemServer`启动的服务是真的多，而且大多数服务之间存在较强的依赖，有着相对严格的启动顺序

- 系统服务需要实现

  ```
  com.android.server.SystemService
  ```

  （直接继承或者提供相应的内部类），

  ```
  Installer
  ```

  和

  ```
  AppWidgetService
  ```

  分别代表了两种实现方式

  - `grep`查看相应的继承类，足足有`86`种之多

- 系统服务的启动通过

  ```
  SystemServiceManager
  ```

  的

  ```
  startService(systemservice)
  ```

  函数

  - `startService(systemservice)`函数会把要启动的`systemservice`添加到`mServices`集合中
  - 然后会回调`SystemService`的`onStart`抽象方法启动服务

- ```
  SystemServer
  ```

  的启动阶段以

  ```
  mActivityManagerService.systemReady()
  ```

  方法为终点，这个函数比较吸引人的是：

  - 传入了一个

    ```
    Runnable
    ```

    对象，对一些服务进行了额外的处理，这个对象主要是用来：

    - 启动`SystemUI`、`Watchdog`等服务

    - 执行一些服务的

      ```
      systemReady()
      ```

      和

      ```
      systemRunning()
      ```

      方法

      - 再吐槽一下，这两个函数完全没有抽象出来，都是在各个服务中单独声明，很难去阅读

  - 执行一个很著名的函数`startHomeActivityLocked()`来广播通知`launcher`启动

好了，适可而止了，本篇是为了梳理`SystemServer`的启动流程，`Launcher`启动什么的留在后面详细学习哈！

接下来看看`WatchDog`相关的知识

# `SystemServer`中的`Watchdog`

> 在`Init进程`的学习中我们介绍了`watchdogd`守护进程，这个守护进程会在规定的时间内向`硬件看门狗`发送消息表示自己没出故障；超时看门狗就会重启设备。

对于`SystemServer`进程来说，运行的服务数量超过80种，是最有可能出现问题的进程，因此，有必要对`SystemServer`中运行的各种线程实施监控。

但是，如果仿照`硬件看门狗`的方式每个线程定时去`喂狗`，不但非常浪费资源，而且会导致程序设计更加复杂。因此，`Android`开发了`Watchdog`类作为`软件看门狗`来监控`SystemServer`中的线程。一旦发现问题，`Watchdog`会杀死`SystemServer`进程。

前面在`Zygote`进程部分的学习我们知道：

- `SystemServer`的父进程`Zygote`进程接收到`SystemServer`的死亡信号后会杀死自己。
- 然后`Zygote`进程的死亡信号会传递给`Init`进程，`Init`进程会杀死`Zygote`进程的所有子进程并重启`Zygote`进程。

这样，整个系统的基础服务都会重启一遍。这种`软启动`的方式能解决大部分问题，并且速度更快。那么让我们来学习下怎么实现的吧

## 启动`Watchdog`

`Watchdog`是继承自`Thread`，在`SystemServer`中创建`Watchdog`对象是在`startOtherServices()`中，代码如下：

```java
        final Watchdog watchdog = Watchdog.getInstance();
        watchdog.init(context, mActivityManagerService);
复制代码
```

`Watchdog`是单例的运行模式，第一次调用`getInstance()`会创建`Watchdog`对象并保存到全局变量`sWatchdog`中，我们看下构造方法：

```java
    private Watchdog() {
        // 用来监听服务的
        mMonitorChecker = new HandlerChecker(FgThread.getHandler(),
        "foreground thread", DEFAULT_TIMEOUT);
        mHandlerCheckers.add(mMonitorChecker);
        mHandlerCheckers.add(new HandlerChecker(new Handler(Looper.getMainLooper()),
        "main thread", DEFAULT_TIMEOUT));
        mHandlerCheckers.add(new HandlerChecker(UiThread.getHandler(),
                "ui thread", DEFAULT_TIMEOUT));
        mHandlerCheckers.add(new HandlerChecker(IoThread.getHandler(),
                "i/o thread", DEFAULT_TIMEOUT));
        mHandlerCheckers.add(new HandlerChecker(DisplayThread.getHandler(),
                "display thread", DEFAULT_TIMEOUT));
        addMonitor(new BinderThreadMonitor());
        mOpenFdMonitor = OpenFdMonitor.create();
    }
复制代码
```

- ```
  Watchdog
  ```

  构造方法的主要工作是创建几个

  ```
  HandlerChecker
  ```

  对象，并添加到

  ```
  mHandlerCheckers
  ```

  集合中。

  - 每个`HandlerChecker`对象对应一个被监控的`HandlerThread`线程，通过获取到对应线程的`Handler`来与其通信。
  - `HandlerChecker`类的简要结构如下：

  ```java
  public final class HandlerChecker implements Runnable {
      private final Handler mHandler;
      private final String mName;
      private final long mWaitMax;
      private final ArrayList<Monitor> mMonitors = new ArrayList<Monitor>();
      ......
      HandlerChecker(Handler handler, String name, long waitMaxMillis) {
          mHandler = handler;
          mName = name;
          mWaitMax = waitMaxMillis;
          ......
      }
      public void addMonitor(Monitor monitor) {
          mMonitors.add(monitor);
      }
  }
  public interface Monitor {
      void monitor();
  }
  复制代码
  ```

  - 请留意`Monitor`接口，如果一个服务需要通过`Watchdog`来监控，它必须实现这个`Monitor`接口

- 构造方法中的

  ```
  FgThread
  ```

  、

  ```
  UiThread
  ```

  、

  ```
  IoThread
  ```

  、

  ```
  DisplayThread
  ```

  都是继承自

  ```
  HandlerThread
  ```

  。

  - `Android`用它们分别执行不同类型的任务，`Io相关`、`UI相关`等
  - 实现上都是通过`Handler`相关的那一套，大家看看`HandlerThread`类就知道了。

`Watchdog`对象创建后，接下来会调用`init()`方法进行初始化，代码如下：

```java
    public void init(Context context, ActivityManagerService activity) {
        mResolver = context.getContentResolver();
        mActivity = activity;
        context.registerReceiver(new RebootRequestReceiver(),
                new IntentFilter(Intent.ACTION_REBOOT),
                android.Manifest.permission.REBOOT, null);
    }
复制代码
```

`init()`注册了广播接收器，当收到`Intent.ACTION_REBOOT`的广播后，会执行重启操作。

最后，在`startOtherServices()`的`mActivityManagerService.systemReady()`阶段，执行`Watchdog`的启动：

```
 mActivityManagerService.systemReady(() -> {
        ...
        Watchdog.getInstance().start();
        ...
},...);
复制代码
```

## `Watchdog`监控的服务和线程

`Watchdog`主要监控线程，在`SystemServer`进程中运行着很多线程，它们负责处理一些重要模块的消息。如果一个线程陷入死循环或者和其他线程相互死锁了，`Watchdog`需要有办法识别出它们。

`Watchdog`中提供了两个方法`addThread()`和`addMonitor()`分别用来增加需要监控的线程和服务：

- `addThread()`用来添加线程监听：

  ```java
  public void addThread(Handler thread, long timeoutMillis) {
      synchronized (this) {
          if (isAlive()) {
              throw new RuntimeException("Threads can't be added once the Watchdog is running");
          }
          final String name = thread.getLooper().getThread().getName();
          mHandlerCheckers.add(new HandlerChecker(thread, name, timeoutMillis));
      }
  }
  复制代码
  ```

  - 创建了一个`HandlerChecker`对象并添加到`mHandlerCheckers`集合中

- `addMonitor()`用来添加服务的监听：

  ```java
  public void addMonitor(Monitor monitor) {
      synchronized (this) {
          if (isAlive()) {
              throw new RuntimeException("Monitors can't be added once the Watchdog is running");
          }
          mMonitorChecker.addMonitor(monitor);
      }
  }
  复制代码
  ```

  - 对服务的监控`Android`使用`mMonitorChecker`一个`HandlerChecker`对象来完成
  - `mMonitorChecker`在`Watchdog`对象初始化时就创建完成

通过`addThread()`添加监控的线程有：

- `主线程`
- `FgThread`
- `UiThread`
- `IoThread`
- `DisplayThread`
- `PowerManagerService`的线程
- `PackageManagerService`的线程
- `PermissionManagerService`的线程
- `ActivityManagerService`的线程

通过`addMonitor()`添加监控的服务有：

```shell
NetworkManagementService.java
StorageManagerService.java
ActivityManagerService.java
InputManagerService.java
MediaRouterService.java
MediaSessionService.java
MediaProjectionManagerService.java
PowerManagerService.java
TvRemoteService.java
WindowManagerService.java
复制代码
```

## `Watchdog`监控的原理

> 前面讲过，`Binder`调用是在某个`Binder`线程中执行的，但是执行的线程并不固定，因此，`Watchdog`不能用监控一个普通线程的方法来判断某个`Binder`服务是否正常

如果运行在`Binder`线程中的方法使用了全局的资源，就必须建立`临界区`来实施保护。通常的做法是使用`synchronized`关键字。如：

```java
synchronized (mLock){
    ......
}
复制代码
```

这种情况下，我们可以使用锁`mLock`持有的时间是否超时来判断服务是否正常。

而`Watchdog`的思想就是：给线程发送消息，如果发送的消息不能在规定的时间内得到处理，即表明线程被不正常的占用了。

我们看下`Watchdog`整体流程的实现：

```java
    static final long DEFAULT_TIMEOUT = DB ? 10*1000 : 60*1000;
    static final long CHECK_INTERVAL = DEFAULT_TIMEOUT / 2;
    @Override
    public void run() {
        boolean waitedHalf = false;
        while (true) {
            final List<HandlerChecker> blockedCheckers;
            final String subject;
            final boolean allowRestart;
            int debuggerWasConnected = 0;
            synchronized (this) {
                long timeout = CHECK_INTERVAL;
                // 通过scheduleCheckLocked给监控的线程发送消息
                for (int i=0; i<mHandlerCheckers.size(); i++) {
                    HandlerChecker hc = mHandlerCheckers.get(i);
                    hc.scheduleCheckLocked();
                }
                ......
                // 休眠特定的时间，默认为30s
                long start = SystemClock.uptimeMillis();
                while (timeout > 0) {
                    ......
                    try {
                        wait(timeout);
                    } catch (InterruptedException e) {
                        Log.wtf(TAG, e);
                    }
                    ......
                    timeout = CHECK_INTERVAL - (SystemClock.uptimeMillis() - start);
                }
                ......
                // 检测是否有线程或服务出问题了
                final int waitState = evaluateCheckerCompletionLocked();
                    if (waitState == COMPLETED) {
                        // The monitors have returned; reset
                        waitedHalf = false;
                        continue;
                    } else if (waitState == WAITING) {
                        // still waiting but within their configured intervals; back off and recheck
                        continue;
                    } else if (waitState == WAITED_HALF) {
                        if (!waitedHalf) {
                            // We've waited half the deadlock-detection interval.  Pull a stack
                            // trace and wait another half.
                            ArrayList<Integer> pids = new ArrayList<Integer>();
                            pids.add(Process.myPid());
                            ActivityManagerService.dumpStackTraces(true, pids, null, null,
                                getInterestingNativePids());
                            waitedHalf = true;
                        }
                        continue;
                    }
                ......

            // If we got here, that means that the system is most likely hung.
            // First collect stack traces from all threads of the system process.
            // Then kill this process so that the system will restart.
            ......
            {
                Process.killProcess(Process.myPid());
                System.exit(10);
            }
            waitedHalf = false;
        }
    }
复制代码
```

`run()`方法中是一个无限循环，每次循环中主要进行：

- 调用

  ```
  HandlerChecker
  ```

  的

  ```
  scheduleCheckLocked
  ```

  方法给所有受监控的线程发送消息，代码如下：

  ```java
      public void scheduleCheckLocked() {
          if (mMonitors.size() == 0 && mHandler.getLooper().getQueue().isPolling()) {
              mCompleted = true;
              return;
          }
          if (!mCompleted) {
              // we already have a check in flight, so no need
              return;
          }
          mCompleted = false;
          mCurrentMonitor = null;
          mStartTime = SystemClock.uptimeMillis();
          mHandler.postAtFrontOfQueue(this);
      }
  复制代码
  ```

  - ```
    HandlerChecker
    ```

    对象先判断

    ```
    mMonitors
    ```

    的

    ```
    size
    ```

    是否为

    ```
    0
    ```

    - `0`说明当前`HandlerChecker`对象没有监控服务
    - 如果此时被监控线程的消息队列处于空闲状态，说明线程运行良好，直接返回

  - 不为

    ```
    0
    ```

    :

    - 则先把`mCompleted`设为`false`，然后记录消息发送时间`mStartTime`
    - 然后调用`postAtFrontOfQueue`给被监控线程发送一个`Runnable`消息

- ```
  postAtFrontOfQueue()
  ```

  发送的消息的处理方法就是

  ```
  HandlerChecker
  ```

  的

  ```
  run()
  ```

  方法：

  ```java
      @Override
      public void run() {
          final int size = mMonitors.size();
          for (int i = 0 ; i < size ; i++) {
              synchronized (Watchdog.this) {
                  mCurrentMonitor = mMonitors.get(i);
              }
              mCurrentMonitor.monitor();
          }
          synchronized (Watchdog.this) {
              mCompleted = true;
              mCurrentMonitor = null;
          }
      }
  复制代码
  ```

  - 如果消息处理方法`run()`能执行，说明受监控的线程本身没有问题

  - 然后通过调用服务中实现的

    ```
    monitor()
    ```

    方法检查被监控服务的状态

    - 通常`monitor()`方法的实现是获取服务中的锁，如果不能得到，线程就会挂起，`monitor()`通常实现如下：

    ```java
    public void monitor(){
        sychronized(mLock){}
    }
    复制代码
    ```

    - 这样`mCompleted`无法被设置为`true`了

  - ```
    mCompleted
    ```

    设置为

    ```
    true
    ```

    ，说明

    ```
    HandlerChecker
    ```

    对象监控的线程或者服务正常。否则就可能有问题。

    - 是否真的有问题，需要根据等待的时间是否超过规定时间来判断

- 给受监控的线程发送完消息后，调用`wait`方法让`Watchdog`休眠特定的一段时间，默认为`30S`

- 休眠结束后，通过

  ```
  evaluateCheckerCompletionLocked()
  ```

  逐个检查是否有线程或服务出现问题，一旦发现问题，马上杀死进程。方法代码如下：

  ```java
  private int evaluateCheckerCompletionLocked() {
      int state = COMPLETED;
      for (int i=0; i<mHandlerCheckers.size(); i++) {
          HandlerChecker hc = mHandlerCheckers.get(i);
          state = Math.max(state, hc.getCompletionStateLocked());
      }
      return state;
  }
  复制代码
  ```

  - `evaluateCheckerCompletionLocked()`调用每个`HandlerChecker`的`getCompletionStateLocked()`方法来获取对象的状态值，`getCompletionStateLocked`代码如下：

    ```java
    public int getCompletionStateLocked() {
        if (mCompleted) {
            return COMPLETED;
        } else {
            long latency = SystemClock.uptimeMillis() - mStartTime;
            if (latency < mWaitMax/2) {
                return WAITING;
            } else if (latency < mWaitMax) {
                return WAITED_HALF;
            }
        }
        return OVERDUE;
    }
    复制代码
    ```

  - 状态值分为4种：

    - `COMPLETED`：值为0，表示状态良好
    - `WAITING`：值为1，表示正在等待消息处理结果
    - `WAITED_HALF`：值为2，表示正在等待并且等待的时间已经大于规定时间的一半，但是还未超过规定时间
    - `OVERDUE`：值为3，表示等待时间已经超过了规定的时间

  - `evaluateCheckerCompletionLocked()`希望获取到最坏的情况，所以使用`Math.max()`来进行比对过滤

# 结语

`SystemServer`本身没有特别难理解的地方，比较费神的是它维持了太多的服务，`start*Services()`阅读起来太痛苦了，好在：

- 在服务列表中看到了很多眼熟的服务，像`ActivityManagerService`、`PackageManagerService`等服务，不至于特别枯燥，让后续的学习有了些盼头
- 关于`Watchdog`的学习上，基于`HandlerThread`构建的监听模式很值得学习

下一篇开始学习`PackageManagerService`相关的知识，go go go!


作者：apigfly
链接：https://juejin.cn/post/6910805095294697479
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。