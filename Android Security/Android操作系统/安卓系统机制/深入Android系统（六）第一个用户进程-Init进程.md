# 深入Android系统（六）第一个用户进程-Init进程

url：https://juejin.cn/post/6887456044760039438



> 十一假期有点堕落，无限火力有点上瘾，谨戒、谨戒

`Init进程`是`Linux 内核`启动后创建的第一个`用户进程`，地位非常重要。

`Init进程`在初始化过程中会启动很多重要的守护进程，因此，了解`Init进程`的启动过程有助于我们更好的理解`Android`系统。

在介绍`Init进程`前，我们先简单介绍下`Android`的启动过程。从系统角度看，`Android`的启动过程可分为3个大的阶段：

- `bootloader引导`

- `装载和启动Linux内核`

- ```
  启动Android系统
  ```

  ，可分为

  - 启动`Init进程`
  - 启动`Zygote`
  - 启动`SystemService`
  - 启动`SystemServer`
  - 启动`Home`
  - 等等......

我们看下启动过程图： ![image](https:////p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/887a6ea830a14d788a80e41308c12cc7~tplv-k3u1fbpfcp-zoom-1.image)

下面简单介绍下启动过程：

1. `Bootloader`引导

> 当按下电源键开机时，最先运行的就是`Bootloader`

- `Bootloader`的主要作用是初始化基本的硬件设备（如 CPU、内存、Flash等）并且建立内存空间映射，为装载`Linux内核`准备好合适的运行环境。

- 一旦`Linux内核`装载完毕，`Bootloader`将会从内存中清除掉

- 如果在

  ```
  Bootloader
  ```

  运行期间，按下预定义的的组合键，可以进入系统的更新模块。

  ```
  Android
  ```

  的下载更新可以选择进入

  ```
  Fastboot模式
  ```

  或者

  ```
  Recovery模式
  ```

  :

  - `Fastboot`是`Android`设计的一套通过`USB`来更新`Android`分区映像的协议，方便开发人员快速更新指定分区。
  - `Recovery`是`Android`特有的升级系统。利用`Recovery`模式可以进行恢复出厂设置，或者执行OTA、补丁和固件升级。进入`Recovery`模式实际上是启动了一个文本模式的`Linux`

1. 装载和启动`Linux`内核

> Android 的 `boot.img` 存放的就是`Linux`内核和一个`根文件系统`

- `Bootloader`会把`boot.img`映像装载进内存
- 然后`Linux`内核会执行整个系统的初始化
- 然后装载`根文件系统`
- 最后启动`Init`进程

1. 启动`Init`进程

> `Linux`内核加载完毕后，会首先启动`Init`进程，`Init`进程是系统的第一个进程

- ```
  Init
  ```

  进程启动过程中，会解析

  ```
  Linux
  ```

  的配置脚本

  ```
  init.rc
  ```

  文件。根据

  ```
  init.rc
  ```

  文件的内容，

  ```
  Init
  ```

  进程会：

  - 装载`Android`的文件系统
  - 创建系统目录
  - 初始化属性系统
  - 启动`Android`系统重要的守护进程，像`USB守护进程`、`adb守护进程`、`vold守护进程`、`rild守护进程`等

- 最后，`Init`进程也会作为守护进程来执行修改属性请求，重启崩溃的进程等操作

1. 启动`ServiceManager`

> `ServiceManager`由`Init`进程启动。在`Binder` 章节已经讲过，它的主要作用是管理`Binder服务`，负责`Binder服务`的注册与查找

1. 启动`Zygote`进程

> `Init`进程初始化结束时，会启动`Zygote`进程。`Zygote`进程负责`fork`出应用进程，是所有应用进程的父进程

- `Zygote`进程初始化时会创建`Android 虚拟机`、预装载系统的资源文件和`Java`类
- 所有从`Zygote`进程`fork`出的用户进程都将继承和共享这些预加载的资源，不用浪费时间重新加载，加快的应用程序的启动过程
- 启动结束后，`Zygote`进程也将变为守护进程，负责响应启动`APK`的请求

1. 启动`SystemServer`

> `SystemServer`是`Zygote`进程`fork`出的第一个进程，也是整个`Android`系统的核心进程

- `SystemServer`中运行着`Android`系统大部分的`Binder服务`
- `SystemServer`首先启动本地服务`SensorManager`
- 接着启动包括`ActivityManagerService`、`WindowsManagerService`、`PackageManagerService`在内的所有`Java服务`

1. 启动`MediaServer`

> ```
> MediaServer`由`Init`进程启动。它包含了一些多媒体相关的本地`Binder服务`，包括：`CameraService`、`AudioFlingerService`、`MediaPlayerService`、`AudioPolicyService
> ```

1. 启动`Launcher`

- `SystemServer`加载完所有的`Java服务`后，最后会调用`ActivityManagerService`的`SystemReady()`方法
- 在`SystemReady()`方法中，会发出`Intent<android.intent,category.HOME>`
- 凡是响应这个`Intent`的`apk`都会运行起来，一般`Launcher`应用才回去响应这个`Intent`

# `Init`进程的初始化过程

> `Init`进程的源码目录在`system/core/init`下。程序的入口函数`main()`位于文件`init.c`中

## `main()`函数的流程

> 书中使用的是`Android 5.0`源码，相比`Android 9.0`这部分已经有很多改动，不过大的方向是一致的，只能对比着学习了。。。

> `main()`函数比较长，整个`Init`进程的启动流程都在这个函数中。由于涉及的点比较多，这里我们先了解整体流程，细节后面补充，一点一点来哈

`Init`进程的`main()`函数的结构是这样的：

```c
int main(int argc, char** argv) {
    //启动参数判断部分
    
    if (is_first_stage) {
        //初始化第一阶段部分
    }
    
    //初始化第二阶段部分

    while (true) {
        //一个无限循环部分
    }
复制代码
```

### 启动程序参数判断

进入`main()`函数后，首先检查启动程序的文件名

函数源码：

```c
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }
复制代码
```

- 如果文件名是`ueventd`，执行`ueventd`守护进程的主函数`ueventd_main`
- 如果文件名是`watchdogd`，执行`watchdogd`守护进程的主函数`watchdogd_main`
- 都不是，则继续执行

才开始是不是就已经有些奇怪了，`Init`进程中还包含了另外两个守护进程的启动，这主要是因为这几个守护进程的代码重合度高，开发人员干脆都放在一起了。

我们看一下`Android.mk`中的片段：

```mk
# Create symlinks.
LOCAL_POST_INSTALL_CMD := $(hide) mkdir -p $(TARGET_ROOT_OUT)/sbin; \
    ln -sf ../init $(TARGET_ROOT_OUT)/sbin/ueventd; \
    ln -sf ../init $(TARGET_ROOT_OUT)/sbin/watchdogd
复制代码
```

- 在编译时，`Android`生成了两个指向`init`文件的符号链接`ueventd`和`watchdogd`
- 这样，启动时如果执行的是这两个符号链接，`main()`函数就可以根据名称判断到底启动哪一个

### 初始化的第一阶段

#### 设置文件属性掩码

函数源码：

```c
    // Clear the umask.
    umask(0);
复制代码
```

默认情况下一个进程创建出的文件合文件夹的属性是`022`，使用`umask(0)`意味着进程创建的属性是`0777`

#### `mount`相应的文件系统

函数源码：

```c
        // Get the basic filesystem setup we need put together in the initramdisk
        // on / and then we'll let the rc file figure out the rest.
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        //......
        mount("sysfs", "/sys", "sysfs", 0, NULL);
        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
        //......
        // Mount staging areas for devices managed by vold
        // See storage config details at http://source.android.com/devices/storage/
        mount("tmpfs", "/mnt", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
              "mode=0755,uid=0,gid=1000");
        // /mnt/vendor is used to mount vendor-specific partitions that can not be
        // part of the vendor partition, e.g. because they are mounted read-write.
        mkdir("/mnt/vendor", 0755);
        InitKernelLogging(argv);
复制代码
```

创建一些基本的目录，包括`/dev`、`/proc`、`/sys`等，同时把一些文件系统，如`tmpfs`、`devpt`、`proc`、`sysfs`等`mount`到相应的目录

- ```
  tmpfs
  ```

  是一种基于内存的文件系统，

  ```
  mount
  ```

  后就可以使用。

  - `tmpfs`文件系统下的文件都放在内存中，访问速度快，但是掉电丢失。因此适合存放一些临时性的文件
  - `tmpfs`文件系统的大小是动态变化的，刚开始占用空间很小，随着文件的增多会随之变大
  - `Android`将`tmpfs`文件系统`mount`到`/dev`目录。`/dev`目录用来存放系统创造的设备节点

- `devpts`是虚拟终端文件系统，通常`mount`到`/dev/pts`目录下

- ```
  proc
  ```

  也是一种基于内存的虚拟文件系统，它可以看作是内核内部数据结构的接口

  - 通过它可以获得系统的信息
  - 同时能够在运行时修改特定的内核参数

- `sysfs`文件系统和`proc`文件系统类似，它是在`Linux 2.6`内核引入的，作用是把系统设备和总线按层次组织起来，使得他们可以在用户空间存取

#### 初始化`kernel`的`Log`系统

通过`InitKernelLogging()`函数进行初始化，由于此时`Android`的`log系统`还没有启动，所以`Init`只能使用`kernel`的`log系统`

```c
        // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
        // talk to the outside world...
        InitKernelLogging(argv);
复制代码
```

#### 初始化`SELinux`

```c
        // Set up SELinux, loading the SELinux policy.
        SelinuxSetupKernelLogging();
        SelinuxInitialize();
复制代码
```

`SELinux`是在`Android 4.3`加入的安全内核，后面详细介绍

### 初始化的第二阶段

#### 创建`.booting`空文件

在`/dev`目录下创建一个空文件`.booting`表示初始化正在进行

```c
    // At this point we're in the second stage of init.
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";
    //......
    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
复制代码
```

> 大家留心注释，我们已经处于初始化的第二阶段了

- `is_booting()`函数会依靠空文件`.booting`来判断是否进程处于初始化中
- 初始化结束后这个文件将被删除

#### 初始化`Android`的属性系统

```c
    property_init();
复制代码
```

`property_init()`函数主要作用是创建一个共享区域来储存属性值，后面会详细介绍

#### 解析`kernel`参数并进行相关设置

```c
    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    process_kernel_dt();
    process_kernel_cmdline();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    export_kernel_boot_props();

    // Make the time that init started available for bootstat to log.
    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));
    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));

    // Set libavb version for Framework-only OTA match in Treble build.
    const char* avb_version = getenv("INIT_AVB_VERSION");
    if (avb_version) property_set("ro.boot.avb_version", avb_version);
复制代码
```

这部分进行的是属性的设置，我们看下几个重点方法：

- `process_kernel_dt()`函数：读取设备树(`DT`)上的属性设置信息，查找系统属性，然后通过`property_set`设置系统属性
- `process_kernel_cmdline()`函数：解析`kernel`的`cmdline`文件提取以 `androidboot.`字符串打头的字符串，通过`property_set`设置该系统属性
- `export_kernel_boot_props()`函数：额外设置一些属性，这个函数中定义了一个集合，集合中定义的属性都会从`kernel`中读取并记录下来

#### 进行第二阶段的`SELinux`设置

进行第二阶段的`SELinux`设置并恢复一些文件安全上下文

```c
    // Now set up SELinux for second stage.
    SelinuxSetupKernelLogging();
    SelabelInitialize();
    SelinuxRestoreContext();
复制代码
```

#### 初始化子进程终止信号处理函数

```c
    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        PLOG(FATAL) << "epoll_create1 failed";
    }

    sigchld_handler_init();
复制代码
```

> 在`linux`当中，父进程是通过捕捉`SIGCHLD`信号来得知子进程运行结束的情况，`SIGCHLD`信号会在子进程终止的时候发出。

- 为了防止`init`的子进程成为`僵尸进程`(zombie process)，`init` 在子进程结束时获取子进程的结束码
- 通过结束码将程序表中的子进程移除，防止成为僵尸进程的子进程占用程序表的空间
- 程序表的空间达到上限时，系统就不能再启动新的进程了，这样会引起严重的系统问题

#### 设置系统属性并开启属性服务

```c
    property_load_boot_defaults();
    export_oem_lock_status();
    start_property_service();
    set_usb_controller();
复制代码
```

- `property_load_boot_defaults()`、`export_oem_lock_status()`、`set_usb_controller()`这三个函数都是调用、设置一些系统属性
- `start_property_service()`：开启系统属性服务

#### 加载`init.rc`文件

> `init.rc`是一个可配置的初始化文件，在`Android`中被用作程序的启动脚本，它是`run commands`运行命令的缩写

> 通常第三方定制厂商可以配置额外的初始化配置：`init.%PRODUCT%.rc`。在`init`的初始化过程中会解析该配置文件，完成定制化的配置过程。

`init.rc`文件的规则和具体解析逻辑后面详解，先看下它在`main`函数中的相关流程。

函数代码：

```c
    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);

    subcontexts = InitializeSubcontexts();
    // 创建 action 相关对象
    ActionManager& am = ActionManager::GetInstance();
    // 创建 service 相关对象
    ServiceList& sm = ServiceList::GetInstance();
    // 加载并解析init.rc文件到对应的对象中
    LoadBootScripts(am, sm);
复制代码
```

解析完成后，会执行

```c
    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    // 等待冷插拔设备初始化完成
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");
    am.QueueBuiltinAction(SetMmapRndBitsAction, "SetMmapRndBits");
    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");
    // 初始化组合键监听模块
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    // 在屏幕上显示 Android 字样的Logo
    am.QueueBuiltinAction(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Starting the BoringSSL self test, for NIAP certification compliance.
    am.QueueBuiltinAction(StartBoringSslSelfTest, "StartBoringSslSelfTest");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");
复制代码
```

- ```
  am.QueueEventTrigger
  ```

  函数即表明到达了某个所需的某个时间条件

  - 如`am.QueueEventTrigger("early-init")`表明`early-init`条件触发，对应的动作可以开始执行
  - 需要注意的是这个函数只是将时间点(如：`early-init`)填充进`event_queue_`运行队列
  - 后面的`while(true)`循环才会真正的去按顺序取出，并触发相应的操作

到这里，

- `init.rc`相关的`action`和`service`已经解析完成
- 对应的列表也已经准备就绪
- 对应的`Trigger`也已经添加完成

接下来就是执行阶段了：

```c
    while (true) {
        // 执行命令列表中的 Action
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
        }
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            if (!shutting_down) {
                // 启动服务列表中的服务
                auto next_process_restart_time = RestartProcesses();
                //......
            }
            //......
        }
        
        // 监听子进程的死亡通知信号
        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }
复制代码
```

到这里，`main`函数的整体流程就分析完了。分析过程中我们舍弃了很多细节，接下来就是填补细节的时候了。

## 启动`Service`进程

在`main`函数的`while()`循环中会调用`RestartProcesses()`来启动服务列表中的服务进程，我们看下函数源码：

```c
static std::optional<boot_clock::time_point> RestartProcesses() {
    //......
    //循环检查每个服务
    for (const auto& s : ServiceList::GetInstance()) {
        // 判断标志位是否为 SVC_RESTARTING
        if (!(s->flags() & SVC_RESTARTING)) continue;
        // ......省略时间相关的判断
        // 启动服务进程
        auto result = s->Start();
    }
    //......
}
复制代码
RestartProcesses()`会检查每个服务：凡是带有`SVC_RESTARTING`标志的，才会执行服务的启动`s->Start();
```

其实重点在`s->Start();`方法，我们具体来看下（删减版）：

```c
Result<Success> Service::Start() {

    //......省略部分
    // 清空service相关标记
    // 判断service状态，如果已运行直接返回
    // 判断service二进制文件是否存在
    // 初始化console、scon(安全上下文)等
    //......省略部分

    pid_t pid = -1;
    if (namespace_flags_) {// 当service定义了namespace时会赋值为CLONE_NEWPID|CLONE_NEWNS
        pid = clone(nullptr, nullptr, namespace_flags_ | SIGCHLD, nullptr);
    } else {
        pid = fork();
    }

    if (pid == 0) {// 子进程创建成功
        //......省略部分
        // setenv、writepid、重定向标准IO
        //......省略部分

        // As requested, set our gid, supplemental gids, uid, context, and
        // priority. Aborts on failure.
        SetProcessAttributes();

        if (!ExpandArgsAndExecv(args_)) {// 解析参数并启动service
            PLOG(ERROR) << "cannot execve('" << args_[0] << "')";
        }
        // 不太懂这里退出的目的是干啥
        _exit(127);
    }

    if (pid < 0) {
        // 子进程创建失败
        pid_ = 0;
        return ErrnoError() << "Failed to fork";
    }

    //......省略部分
    // 执行service其他参数的设置，如oom_score_adj、创建并设置ProcessGroup相关的参数
    //......省略部分
    NotifyStateChange("running");
    return Success();
}
复制代码
```

`Service`进程的启动流程还有很多细节，这部分只是简单介绍下流程，涉及的`scon`、`PID`、`SID`、`PGID`等东西还很多。

偷懒啦！能力、时间有限，先往下学习，待用到时再来啃吧

# 解析启动脚本`init.rc`

`Init`进程启动时最重要的工作就是解析并执行启动文件`init.rc`。[官方说明文档下载链接](http://cdn.hualee.top/AIL.md)

## `init.rc`文件格式

`init.rc`文件是以`section`为单位组织的

- `section`分为两大类：
  - `action`：以关键字`on`开始，表示一堆命令的集合
  - `service`：以关键字`service`开始，表示启动某个进程的方式和参数
- `section`以关键字`on`或`service`开始，直到下一个`on`或`service`结束
- `section`中的注释以`#`开始

打个样儿：

```shell
import /init.usb.rc
import /init.${ro.hardware}.rc

on early-init
    mkdir /dev/memcg/system 0550 system system
    start ueventd

on init
    symlink /system/bin /bin
    symlink /system/etc /etc

on nonencrypted
    class_start main
    class_start late_start
    
on property:sys.boot_from_charger_mode=1
    class_stop charger
    trigger late-init

service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
    shutdown critical

service flash_recovery /system/bin/install-recovery.sh
    class main
    oneshot

service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
    shutdown critical
复制代码
```

无论是`action`还是`service`，并不是按照文件中的书写顺序执行的，执行与否以及何时执行要由`Init`进程在运行时决定。

对于`init.rc`的`action`：

- 关键字`on`后面跟的字符串称为`trigger`，如上面的`early-init`、`init`等。
- `trigger`后面是命令列表，命令列表中的每一行就是一条命令。

对于`init.rc`的`service`：

- 关键字`service`后面是`服务名称`，可以使用`start`加`服务名称`来启动一个服务。如`start ueventd`

- `服务名称`后面是进程的可执行文件的路径和启动参数

- ```
  service
  ```

  下面的行称为

  ```
  option
  ```

  ，每个

  ```
  option
  ```

  占一行

  - 例如：`class main`中的`class`表示服务所属的类别，可以通过`class_start`来启动一组服务，像`class_start main`

想要了解更多，可以参考源码中的`README`文档，路径是`system/core/init/README.md`

## `init.rc`的关键字

> 这部分是对`system/core/init/README.md`文档的整理，挑重点记录哈

```
Android`的`rc`脚本包含了4中类型的声明：`Action`、`Commands`、`Services`、`Options
```

- 所有的指令都以行为单位，各种符号则由空格隔开。

- `c语言`风格的反斜杠`\`可用于在符号间插入空格

- 双引号`""`可用于防止字符串被空格分割成多个记号

- 行末的反斜杠`\`可用于折行

- 注释以`#`开头

- ```
  Action
  ```

  和

  ```
  Services
  ```

  用来申明一个分组

  - 所有的`Commands`和`Options`都属于最近声明的分组
  - 位于第一个分组之前的`Commands`或`Options`将被忽略

### `Actions`

`Actions`是一组`Commands`的集合。每个`Action`都有一个`trigger`用来决定何时执行。当触发条件与`Action`的`trigger`匹配一致时，此`Action`会被加入到执行队列的尾部

每个`Action`都会依次从队列中取出，此`Action`的每个`Command`都将依次执行。

`Actions`格式如下：

```c
on  < trigger >
    < command >
    < command >
    < command >
复制代码
```

### `Services`

通过`Services`定义的程序，会在Init中启动，如果退出了会被重启。

`Services`的格式如下：

```c
service <name> <pathname> [ <argument> ]*
   <option>
   <option>
   ...
复制代码
```

### `Options`

`Options`是`Services`的修订项。它们决定了一个`Service`何时以及如何运行。

- `critical`：表示这是一个关键的`Service`，如果`Service`4分钟内重新启动超过4次，系统将自动重启并进入`recovery`模式

- ```
  console [<console>]
  ```

  ：表示该服务需要一个控制台

  - 第二个参数用来指定特定控制台名称，默认为`/dev/console`
  - 指定时需省略掉`/dev/`部分，如`/dev/tty0`需写成`console tty0`

- `disabled`：表示服务不会通过`class_start`启动。它必须以命令`start service_name`的方式指定来启动

- `setenv <name> <value>`：在`Service`启动时将环境变量`name`设置为`value`

- ```
  socket <name> <type> <perm> [ <user> [ <group> [ <seclabel> ] ] ]
  ```

  ：创建一个名为

  ```
  /dev/socket/<name>
  ```

  的套接字，并把文件描述符传递给要启动的进程

  - `type`的值必须是`dgram`、`stream`、`seqpacket`
  - `user`和`group`默认为0
  - `seclabel`是这个`socket`的`SElinux`安全上下文，默认为当前`service`的上下文

- `user <username>`：在执行此服务之前切换用户名，默认的是root。如果进程没有相应的权限，则不能使用该命令

- `oneshot`：`Service`退出后不再重启

- `class <name>`：给`Service`指定一个名字。所有同名字的服务可以同时启动和停止。如果不通过`class`显示指定，默认为`default`

- `onrestart`：当`Service`重启时，执行一条命令

还有很多哈，就不一一介绍了，像`shutdown <shutdown_behavior>`这种，参照官方说明就好啦

### `Triggers`

```
trigger`本质上是一个字符串，能够匹配某种包含该字符串的事件。`trigger`又被细分为`事件触发器(event trigger)`和`属性触发器(property trigger)
```

- `事件触发器`可由`trigger`命令或初始化过程中通过`QueueEventTrigger()`触发
  - 通常是一些事先定义的简单字符串，例如：`boot`,`late-init`等
- `属性触发器`是当指定属性的变量值变成指定值时触发
  - 其格式为`property:<name>=*`

请注意，一个`Action`可以有多个`属性触发器`，但是最多有一个`事件触发器`。看下官方的例子：

```c
on boot && property:a=b
复制代码
```

上面的`Action`只有在`boot`事件发生时，并且`属性a`和`数值b`相等的情况下才会被触发。

而对于下面的`Action`

```c
on property:a=b && property:c=d 
复制代码
```

存在三种触发情况：

- 在启动时,如果`属性a`的值等于`b`并且`属性c`的值等于`d`
- 在`属性c`的值已经是`d`的情况下,`属性a`的值被更新为`b`
- 在`属性a`的值已经是`b`的情况下,`属性c`的值被更新为`d`

对于`事件触发器`，大体包括：

| 类型                    | 说明                 |
| ----------------------- | -------------------- |
| `boot`                  | init.rc被装载后触发  |
| `device-added-<path>`   | 指定设备被添加时触发 |
| `device-removed-<path>` | 指定设备被移除时触发 |
| `service-exited-<name>` | 在特定服务退出时触发 |
| `early-init`            | 初始化之前触发       |
| `late-init`             | 初始化之后触发       |
| `init`                  | 初始化时触发         |

### `Commands`

`Command`是用于`Action`的命令列表或者`Service`的`Option<onrestart>`中，在源码中是这样的：

```c
static const Map builtin_functions = {
        {"chmod",                   {2,     2,    {true,   do_chmod}}},
        {"chown",                   {2,     3,    {true,   do_chown}}},
        {"class_start",             {1,     1,    {false,  do_class_start}}},
        {"class_stop",              {1,     1,    {false,  do_class_stop}}},
        ......
    };
复制代码
```

看几个常用的吧

1. `bootchart [start|stop]`：开启或关闭进程启动时间记录工具

```shell
    //init.rc file
    mkdir /data/bootchart 0755 shell shell
    bootchart start
复制代码
```

- 在`Init进程`中会启动`bootchart`，默认不会执行时间采集
- 当我们需要采集启动时间时，需创建一个`/data/bootchart/enabled`文件

1. `chmod <octal-mode> <path>`：更改文件权限

```shell
chmode 0755 /metadata/keystone
复制代码
```

1. `chown <owner> <group> <path>`：更改文件的所有者和组

```shell
chown system system /metadata/keystone
复制代码
```

1. `mkdir <path> [mode] [owner] [group]`：创建指定目录

```shell
mkdir /data/bootchart 0755 shell shell
复制代码
```

1. `trigger <event>`：触发某个`事件（Action）`，用于将该`事件`排在某个`事件`之后

```shell
on late-init
    trigger early-fs
    trigger fs
    trigger post-fs
    trigger late-fs
    trigger post-fs-data
    trigger zygote-start
    trigger load_persist_props_action
    trigger early-boot
    trigger boot
复制代码
```

1. `class_start <serviceclass>`：启动所有指定服务`class`下的未运行服务

```c
    class_start main
    class_start late_start
复制代码
```

1. `class_stop <serviceclass>`：停止所有指定服务`class`下的已运行服务

```c
    class_stop charger
复制代码
```

1. `exec [ <seclabel> [ <user> [ <group>\* ] ] ] -- <command> [ <argument>\* ]`：通过给定的参数`fork`和启动一个命令。

- 具体的命令在`--`后开始
- 参数包括`seclable`（默认的话使用`-`）、`user`、`group`
- `exec`为阻塞式，在当前命令完成前，不会运行其它命令，此时`Init`进程暂停执行。

```c
exec - system system -- /system/bin/tzdatacheck /system/usr/share/zoneinfo /data/misc/zoneinfo
复制代码
```

还有很多指令就不一一介绍了，参考官方文档和源码就好啦

## `init`脚本的解析

上面我们知道了，在`init.cpp`的`main`函数中通过`LoadBootScripts()`来加载`rc`脚本，我们简单看下解析流程（注释比较详细啦）

```c
/**
 * 7.0后，init.rc进行了拆分，每个服务都有自己的rc文件
 * 他们基本上都被加载到/system/etc/init，/vendor/etc/init, /odm/etc/init等目录
 * 等init.rc解析完成后，会来解析这些目录中的rc文件，用来执行相关的动作
 * ===============================================================================
 * 对于自定义的服务来说，我们只需要通过 LOCAL_INIT_RC 指定自己的rc文件即可
 * 编译时会根据分区标签将rc文件拷贝到指定的partition/etc/init目录
 */
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    // 创建解析器
    Parser parser = CreateParser(action_manager, service_list);
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        // 如果没有特殊配置ro.boot.init_rc，先解析 init.rc 文件
        parser.ParseConfig("/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
    } else {
        // 直接解析 ro.boot.init_rc 中的数据
        parser.ParseConfig(bootscript);
    }
}

/**
 * 创建解析器，目前只有三种section：service、on、import
 * 与之对应的，函数中也出现了三种解析器：ServiceParser、ActionParser、ImportParser
 */
Parser CreateParser(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser;
    parser.AddSectionParser("service", std::make_unique<ServiceParser>(&service_list, subcontexts));
    parser.AddSectionParser("on", std::make_unique<ActionParser>(&action_manager, subcontexts));
    parser.AddSectionParser("import", std::make_unique<ImportParser>(&parser));
    return parser;
}
复制代码
```

## `init`中启动的守护进程

在`init.rc`中定义了很多守护进程，9我们来看下相关内容：

```
# adb 守护进程放在了这里
import /init.usb.rc
# service adbd /system/bin/adbd --root_seclabel=u:r:su:s0
#    class core
#    ......

on boot
    # Start standard binderized HAL daemons
    class_start hal
    # 启动所有class core的服务
    # 像adb、console等
    class_start core

on eraly-init
    start ueventd

on post-fs
    # Load properties from
    #     /system/build.prop,
    #     /odm/build.prop,
    #     /vendor/build.prop and
    #     /factory/factory.prop
    load_system_props
    # start essential services
    # 这几个 service 的.rc文件都在对应项目中，通过LOCAL_INIT_RC来指定
    start logd
    # 启动三个Binder服务管理相关的service
    # servicemanager用于框架/应用进程之间的 IPC，使用 AIDL 接口
    start servicemanager
    # hwservicemanager用于框架/供应商进程之间的 IPC，使用 HIDL 接口
    # 也可用于供应商进程之间的 IPC，使用 HIDL 接口
    start hwservicemanager
    # vndservicemanager用于供应商/供应商进程之间的 IPC，使用 AIDL 接口
    start vndservicemanager

on post-fs-data
    # 启动 vold（Volume守护进程），负责系统扩展储存的自动挂载
    # 这个进程后面详解
    start vold

# 负责响应 uevent 事件，创建对应的设备节点
service ueventd /sbin/ueventd
    class core
    ......
    
# 包含常用的shell命令，如ls、cd等
service console /system/bin/sh
    class core
    ......

# Zygote 进程在这里导入，现在支持32位和64位
import /init.${ro.zygote}.rc
# zygote中会启动一些相关service，像media、netd、wificond等
# service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
#    onrestart restart audioserver
#    onrestart restart cameraserver
#    onrestart restart media
#    onrestart restart netd
#    onrestart restart wificond
# 启动 Zygote 进程，触发这个action的位置在 on late-init 中
on zygote-start && property:ro.crypto.state=unencrypted
    start netd
    start zygote
    start zygote_secondary
复制代码
```

启动流程和需要启动的`service`通过`init.rc`基本上就可以完成定制。

感觉`Init`进程通过解析`*.rc`的方式大大简化了开发，真的是6啊。设计这一套`AIL`的人是真滴猛。。。。。。。

# `Init`进程对信号的处理

`Init`进程是系统的一号进程，系统中的其他进程都是`Init`进程的后代.

按照`Linux`的设计，`Init`进程需要在这些后代`死亡`时负责`清理`它们，以防止它们变成`僵尸进程`。

## `僵尸进程`简介

> 关于`僵尸进程`可以参考[Wiki百科-僵尸进程](https://zh.wikipedia.org/wiki/僵尸进程)哈

在`类UNIX`系统中，`僵尸进程`是指完成执行（通过`exit`系统调用，或运行时发生致命错误或收到终止信号所致），但在操作系统的`进程表`中仍然存在其进程控制块，处于`终止状态`的进程。

这发生于`子进程`需要保留表项以允许其`父进程`读取子进程的退出状态：一旦`退出态`通过`wait系统调用`读取，`僵尸进程`条目就从进程表中删除，称之为`回收（reaped）`。

正常情况下，进程直接被其父进程`wait`并由系统回收。

僵尸进程的避免

- `父进程`通过`wait`和`waitpid`等函数等待`子进程`结束，这会导致`父进程`挂起
- 如果`父进程`很忙，那么可以用`signal`函数为`SIGCHLD`安装`handler`，因为`子进程`结束后， `父进程`会收到该信号，可以在`handler`中调用`wait`回收
- 如果`父进程`不关心`子进程`什么时候结束，那么可以用`signal（SIGCHLD,SIG_IGN）` 通知内核，自己对`子进程`的结束不感兴趣，那么`子进程`结束后，内核会回收，并不再给`父进程`发送信号
- 还有一些技巧，就是`fork`两次，`父进程`先`fork`一个`子进程`，然后继续工作，子进程`fork`一个`孙进程`后退出，那么`孙进程`被`init`接管，`孙进程`结束后，`Init`会回收。不过`子进程`的回收 还要自己做

我们下面来看下`Init`进程怎么处理的

## 初始化`SIGCHLD`信号

`Init`进程的`main`函数中，在初始化第二阶段，有这么一个方法`sigchld_handler_init()`：

```
// file : init.cpp
int main(int argc, char** argv) {
    ......
    sigchld_handler_init();
    ......
}
// file : sigchld_handler.cpp
void sigchld_handler_init() {
    // Create a signalling mechanism for SIGCHLD.
    // 创建一个socketpair，往一个socket中写，就可以从另外一个套接字中读取到数据
    int s[2];
    socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0, s);
    signal_write_fd = s[0];
    signal_read_fd = s[1];

    // Write to signal_write_fd if we catch SIGCHLD.
    // 信号初始化相关参数设置
    // 设置SIGCHLD_handler信号处理函数
    // 设置SA_NOCLDSTOP标志
    struct sigaction act;
    memset(&act, 0, sizeof(act));
    act.sa_handler = SIGCHLD_handler;
    act.sa_flags = SA_NOCLDSTOP;
    
    // 注册SIGCHLD信号
    sigaction(SIGCHLD, &act, 0);

    ReapAnyOutstandingChildren();
    
    // 注册signal_read_fd到epoll_fd
    register_epoll_handler(signal_read_fd, handle_signal);
}
// file : sigchld_handler.cpp
/**
 * SIGCHLD_handler的作用是当init进程接收到SIGCHLD信号时，往signal_write_fd中写入数据
 * 这个时候套接字对中的另外一个signal_read_fd就可读了。
 */
static void SIGCHLD_handler(int) {
    if (TEMP_FAILURE_RETRY(write(signal_write_fd, "1", 1)) == -1) {
        PLOG(ERROR) << "write(signal_write_fd) failed";
    }
}
// file : sigchld_handler.cpp
/**
 * register_epoll_handler函数主要的作用是注册属性socket文件描述符到轮询描述符epoll_fd
 */
void register_epoll_handler(int fd, void (*fn)()) {
    epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.ptr = reinterpret_cast<void*>(fn);
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        PLOG(ERROR) << "epoll_ctl failed";
    }
}
复制代码
```

信号的初始化是通过系统调用`sigaction()`来完成的

- 参数

  ```
  act
  ```

  中：

  - `sa_handler`用来指定信号的处理函数
  - `sa_flags`用来指定触发标志，`SA_NOCLDSTOP`标志意味着当子进程终止时才接收`SIGCHLD`信号

在`Linux`系统中，信号又称为`软中断`，信号的到来会中断进程正在处理的工作，因此在信号处理函数中不要去调用一些不可重入的函数。而且`Linux`不会对信号排队，不管是在信号的处理期间再来多少个信号，当前的信号处理函数执行完后，内核只会再发送一个信号给进程，因此，为了不丢失信号，我们的信号处理函数执行得越快越好

而对于`SIGCHLD`信号，父进程需要执行等待操作，这样的话时间就比较长了，因此需要有办法解决这个矛盾

- 上面的代码中创建了一对本地`socket`用于进程间通信
- 当信号到来时，`SIGCHLD_handler`处理函数只要向`socket`中的`signal_write_fd`写入数据就可以
- 这样，信号的处理就转变到了`socket`的处理上了

此时，我们需要监听`signal_read_fd`，并提供一个回调函数，这就是`register_epoll_handler()`函数的作用

- 函数中的`EPOLLIN`表示当文件描述符可读时才会触发
- `*fn`就是触发后的`回调函数指针`，赋值给了`ev.data.ptr`，请留意下这个指针变量，后面会用到
- 提供的回调函数就是`handle_signal()`

## 响应子进程的死亡事件

```
Init`进程启动完毕后，会监听创建的`socket`，如果有数据到来，主线程会唤醒并调用处理函数`handle_signal()
static void handle_signal() {
    // Clear outstanding requests.
    // 清空 signal_read_fd 中的数据
    char buf[32];
    read(signal_read_fd, buf, sizeof(buf));
    
    ReapAnyOutstandingChildren(); 
}
void ReapAnyOutstandingChildren() {
    while (ReapOneProcess()) {
    }
}
static bool ReapOneProcess() {
    siginfo_t siginfo = {};
    // This returns a zombie pid or informs us that there are no zombies left to be reaped.
    // It does NOT reap the pid; that is done below.
    // 查询是否存在僵尸进程
    if (TEMP_FAILURE_RETRY(waitid(P_ALL, 0, &siginfo, WEXITED | WNOHANG | WNOWAIT)) != 0) {
        PLOG(ERROR) << "waitid failed";
        return false;
    }
    // 没有僵尸进程直接返回
    auto pid = siginfo.si_pid;
    if (pid == 0) return false;

    // At this point we know we have a zombie pid, so we use this scopeguard to reap the pid
    // whenever the function returns from this point forward.
    // We do NOT want to reap the zombie earlier as in Service::Reap(), we kill(-pid, ...) and we
    // want the pid to remain valid throughout that (and potentially future) usages.
    // 等待子进程终止
    auto reaper = make_scope_guard([pid] { TEMP_FAILURE_RETRY(waitpid(pid, nullptr, WNOHANG)); });
    ......
    if (!service) return true;
    // 进行退出的其他操作
    service->Reap(siginfo);
    ......
}
复制代码
```

当接收到子进程的`SIGCHLD`信号后，会找出该进程对应的`Service`对象，然后调用`Reap`函数，我们看下函数内容：

```c
void Service::Reap(const siginfo_t& siginfo) {
    // 如果 不是oneshot 或者 是重启的子进程
    // 杀掉整个进程组，思考了下，先杀掉为重启做准备吧
    // 这样当重启的时候，就不会因为子进程已经存在而导致错误了
    if (!(flags_ & SVC_ONESHOT) || (flags_ & SVC_RESTART)) {
        KillProcessGroup(SIGKILL);
    }
    // 做一些当前进程的清理工作
    ......

    // Oneshot processes go into the disabled state on exit,
    // except when manually restarted.
    // 如果 是oneshot 或者 不是重启的子进程，设置为SVC_DISABLED
    if ((flags_ & SVC_ONESHOT) && !(flags_ & SVC_RESTART)) {
        flags_ |= SVC_DISABLED;
    }

    // Disabled and reset processes do not get restarted automatically.
    // 如果是SVC_DISABLED或者SVC_RESET
    // 设置进程状态为stopped，然后返回
    if (flags_ & (SVC_DISABLED | SVC_RESET))  {
        NotifyStateChange("stopped");
        return;
    }

    // If we crash > 4 times in 4 minutes, reboot into recovery.
    // 省略 crash 次数检测
    ......
    // 省略一些进程状态的设置，都是和重启相关的
    
    // Execute all onrestart commands for this service.
    // 执行onrestart的指令
    onrestart_.ExecuteAllCommands();
    
    // 设置状态为重启中
    NotifyStateChange("restarting");
    return;
}
复制代码
```

在`Reap()`函数中

- 会根据对应进程`Service`对象的`flags_`标志位来判断该进程能不能重启
- 如果需要重启，就给`flags_`标志位添加`SVC_RESTARTING`标志位

**到这里，我们清楚了`handle_signal()`函数的内部流程，那么它又是从哪里被调用的呢？**

我们再回到`init.cpp`的`main()`方法中看看：

```c
    while (true) {
        ......
        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }
复制代码
```

请注意`ev.data.ptr`，还记得`register_epoll_handler()`函数不

```java
void register_epoll_handler(int fd, void (*fn)()) {
    ......
    ev.data.ptr = reinterpret_cast<void*>(fn);
    ......
}
复制代码
```

当`epoll_wait`有数据接收到时，就会执行`((void (*)()) ev.data.ptr)();`，也就是我们的回调函数`handle_signal()`了

> 咳咳咳，到这里就把`Init`进程对子进程死亡通知的逻辑给梳理完了，源码一直在变，好在核心的逻辑没有变化，且看且珍惜吧，哈哈哈~

# 属性系统

## 简介

`属性`在`Android`系统中大量使用，用来保存系统设置或在进程间传递一些简单的信息

- 每个`属性`由`属性名`和`属性值`组成
- `属性名`通常一长串以`.`分割的字符串，这些名称的前缀有特定含义，不能随便改动
- `属性值`只能是字符串

`Java`层可以通过如下方法来获取和设置属性：

```java
//class android.os.SystemProperties
    @SystemApi
    public static String get(String key);
    @SystemApi
    public static String get(String key, String def);
    @hide
    public static void set(String key, String val);
复制代码
```

`native`层可以使用：

```c
android::base::GetProperty(key, "");
android::base::SetProperty(key, val);
复制代码
```

对于系统中的每个进程来说：

- `读取属性值`对任何进程都是没有限制的，直接由本进程从共享区域中读取
- `修改属性值`则必须通过`Init进程`完成，同时`Init进程`还需要检查发起请求的进程是否具有相应的权限

属性值修改成功后，`Init`进程会检查`init.rc`文件中是否已经定义了和该属性值匹配的`trigger`。如果有定义，则执行`trigger`下的命令。如：

```c
on property:ro.debuggable=1
    start console
复制代码
```

这个`trigger`的含义是：当属性`ro.debuggable`被设置为`1`，则执行命令`start console`，启动`console`

`Android`系统级应用和底层模块非常依赖`属性系统`，常常依靠`属性值`来决定它们的行为。

`Android`的系统设置程序中，很多功能的打开和关闭都是通过某个特定的`系统属性值`来控制。这也意味着随便改变`属性值`将会严重影响系统的运行，因此，对于`属性值`的修改必须要有特定的权限。对于权限的设定，现在统一由`SELinux`来控制。

## 属性服务的启动流程

我们先看下属性服务启动的整体流程：

```c
int main(int argc, char** argv) {
    ......
    // 属性服务初始化
    property_init();
    ......
    // 启动属性服务
    start_property_service();
    ......
}
复制代码
```

在`init.cpp`的`main()`函数中，通过`property_init();`来对属性服务进行初始化

```c
void property_init() {
    // 创建一个文件夹，权限711，只有owner才可以设置
    mkdir("/dev/__properties__", S_IRWXU | S_IXGRP | S_IXOTH);
    // 读取一些属性文件，将属性值存储在一个集合中
    CreateSerializedPropertyInfo();
    if (__system_property_area_init()) {// 创建属性共享内存空间(这个函数是libc库的部分)
        LOG(FATAL) << "Failed to initialize property area";
    }
    if (!property_info_area.LoadDefaultPath()) {// 加载默认路径上的属性到共享区域
        LOG(FATAL) << "Failed to load serialized property info file";
    }
}
复制代码
```

然后通过`start_property_service()`函数启动服务：

```c
void start_property_service() {
    // 省略SELinux相关操作
    ......
    property_set("ro.property_service.version", "2");
    // 创建prop service对应的socket，并返回socket fd
    property_set_fd = CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK, false, 0666, 0, 0, nullptr);
    // 省略创建失败的异常判断
    ......
    // 设置最大连接数量为8
    listen(property_set_fd, 8);
    // 注册epolls事件监听property_set_fd
    // 当监听到数据变化时，调用handle_property_set_fd函数
    register_epoll_handler(property_set_fd, handle_property_set_fd);
}
复制代码
```

- `socket`描述符`property_set_fd`被创建后，用`epoll`来监听`property_set_fd`
- 当`property_set_fd`有数据到来时，`init进程`将调用`handle_property_set_fd()`函数来进行处理

我们再来看下`handle_property_set_fd()`函数

```c
static void handle_property_set_fd() {
    static constexpr uint32_t kDefaultSocketTimeout = 2000; /* ms */
    // 省略一些异常判断
    ......
    uint32_t cmd = 0;
    // 省略cmd读取操作和一些异常判断
    ......
    switch (cmd) {
    case PROP_MSG_SETPROP: {
        char prop_name[PROP_NAME_MAX];
        char prop_value[PROP_VALUE_MAX];
        // 省略字符数据的读取组装操作
        ......
        uint32_t result =
            HandlePropertySet(prop_name, prop_value, socket.source_context(), cr, &error);
        // 省略异常情况处理
        ......
        break;
      }

    case PROP_MSG_SETPROP2: {
        std::string name;
        std::string value;
        // 省略字符串数据的读取操作
        ......
        uint32_t result = HandlePropertySet(name, value, socket.source_context(), cr, &error);
        // 省略异常情况处理
        ......
        break;
      }
    default:
        LOG(ERROR) << "sys_prop: invalid command " << cmd;
        socket.SendUint32(PROP_ERROR_INVALID_CMD);
        break;
    }
}
复制代码
Init进程`在接收到设置属性的`cmd`后，会执行处理函数`HandlePropertySet()
uint32_t HandlePropertySet(const std::string& name, const std::string& value,
                           const std::string& source_context, 
                           const ucred& cr, std::string* error) {
    // 判断要设置的属性名称是否合法
    // 相当于命名规则检查
    if (!IsLegalPropertyName(name)) {
        // 不合法直接返回
        return PROP_ERROR_INVALID_NAME;
    }
    // 如果是ctl开头，说明是控制类属性
    if (StartsWith(name, "ctl.")) {
        // 检查是否具有对应的控制权限
        ......
        // 权限通过后执行对应的控制指令
        // 其实控制指令就简单的几个start/stop等，大家可以在深入阅读下这个函数
        HandleControlMessage(name.c_str() + 4, value, cr.pid);
        return PROP_SUCCESS;
    }
    const char* target_context = nullptr;
    const char* type = nullptr;
    // 获取要设置的属性的上下文和数据类型
    // 后面会对target_context和type进行比较判断
    property_info_area->GetPropertyInfo(name.c_str(), &target_context, &type);
    // 检查是否具有当前属性的set权限
    if (!CheckMacPerms(name, target_context, source_context.c_str(), cr)) {
        // 没有直接返回
        return PROP_ERROR_PERMISSION_DENIED;
    }
    // 对属性的类型和要写入数据的类型进行判断
    // 大家看看CheckType函数就明白了，其实只有一个string类型。。。。。
    if (type == nullptr || !CheckType(type, value)) {
        // 不合法，直接返回
        return PROP_ERROR_INVALID_VALUE;
    }
    // 如果是sys.powerctl属性，需要做一些特殊处理
    if (name == "sys.powerctl") {
        // 增加一些额外打印
        ......
    }
    if (name == "selinux.restorecon_recursive") {
        // 特殊属性，特殊处理
        return PropertySetAsync(name, value, RestoreconRecursiveAsync, error);
    }
    return PropertySet(name, value, error);
}
复制代码
```

除了一些特殊的属性外，真正设置属性的函数是`PropertySet`

```c
static uint32_t PropertySet(const std::string& name, const std::string& value, std::string* error) {
    size_t valuelen = value.size();
    // 判断属性名是否合法
    if (!IsLegalPropertyName(name)) {
        *error = "Illegal property name";
        return PROP_ERROR_INVALID_NAME;
    }
    // 判断写入的数据长度是否合法
    // 判断属性是否为只读属性（ro）
    if (valuelen >= PROP_VALUE_MAX && !StartsWith(name, "ro.")) {
        *error = "Property value too long";
        return PROP_ERROR_INVALID_VALUE;
    }
    // 判断要写入数据的编码格式
    if (mbstowcs(nullptr, value.data(), 0) == static_cast<std::size_t>(-1)) {
        *error = "Value is not a UTF8 encoded string";
        return PROP_ERROR_INVALID_VALUE;
    }
    // 根据属性名获取系统中存放的属性对象
    prop_info* pi = (prop_info*) __system_property_find(name.c_str());
    if (pi != nullptr) {
        // ro.* properties are actually "write-once".
        // ro开头的属性只允许写入一次
        if (StartsWith(name, "ro.")) {
            *error = "Read-only property was already set";
            return PROP_ERROR_READ_ONLY_PROPERTY;
        }
        // 如果已经存在，并且不是只读属性，执行属性更新函数
        __system_property_update(pi, value.c_str(), valuelen);
    } else {
        // 如果系统中不存在属性，执行添加属性添加函数
        int rc = __system_property_add(name.c_str(), name.size(), value.c_str(), valuelen);
        if (rc < 0) {
            *error = "__system_property_add failed";
            return PROP_ERROR_SET_FAILED;
        }
    }
    // Don't write properties to disk until after we have read all default
    // properties to prevent them from being overwritten by default values.
    // 如果是持久化的属性，进行持久化处理
    if (persistent_properties_loaded && StartsWith(name, "persist.")) {
        WritePersistentProperty(name, value);
    }
    // 将属性的变更添加到Action队列中
    property_changed(name, value);
    return PROP_SUCCESS;
}
复制代码
```

`Init`进程`epoll`属性的`socket`，等待和处理属性请求。

- 如果有请求到来，则调用`handle_property_set_fd`来处理这个请求
- 在`handle_property_set_fd`函数里，首先检查请求者的`uid/gid`看看是否有权限，如果有权限则调`property_service.cpp`中的`PropertySet`函数。

在`PropertySet`函数中

- 它先查找就没有这个属性，如果找到，更改属性。如果找不到，则添加新属性。
- 更改时还会判断是不是`ro`属性，如果是，则不能更改。
- 如果是`persist`的话还会写到`/data/property/<name>`中。

最后它会调`property_changed`函数，把事件挂到队列里

- 如果有人注册这个属性的话（比如`init.rc`中`on property:ro.kernel.qemu=1`），最终会触发它

# `ueventd`和`watchdogd`简介

## `ueventd`进程

> 守护进程`ueventd`的主要作用是接收`ueventd`来创建和删除设备中`dev`目录下的设备节点

`ueventd`进程和`Init`进程并不是一个进程，但是它们的二进制文件是相同的，只不过启动时参数不一样导致程序的执行流程不一样。

在`init.rc`文件中

```
on early-init
    start ueventd

## Daemon processes to be run by init.
##
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
    shutdown critical
复制代码
```

这样`Init`进程在执行`action eraly-init`是就会启动`ueventd`进程。

## `watchdogd`进程

`watchdogd`和`ueventd`类型，都是独立于`Init`的进程，但是代码和`Init`进程在一起。`watchdogd`是用来配合硬件`看门狗`的。

当一个硬件系统开启了`watchdog`功能，那么运行在这个硬件系统之上的软件必须在规定的时间间隔内向`watchdog`发送一个信号。这个行为简称为`喂狗(feed dog)`，以免`watchdog`记时超时引发系统重起。

现在的系统中很少有看到`watchdogd`进程了，不过这种模式还是很不错、值得借鉴的


