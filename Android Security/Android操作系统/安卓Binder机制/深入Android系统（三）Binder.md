# 深入Android系统（三）Binder

url：https://juejin.cn/post/6868883691494440974

# 1-导读



# 导读(总结)

> 导读叕叕叕叕来了。

本来计划着8月底看完`binder`，而且是用一篇文章来记录完所学的知识，可是：

- 读着读着，感觉缺少一个能串联起整个`binder`的主线，所以决定增加`导读`

- 写着写着，突然发现一篇装不下，所以拆成了三篇

  - `导读`和`binder简介`为一篇

  - [`binder的使用`](https://juejin.im/post/6868882707809173517)为一篇

  - ```
    binder原理
    ```

    为一大篇（85KB....）

    - 掘进放不下拆分成：[原理篇](https://juejin.im/post/6868882596739153927)
    - 掘进放不下拆分成：[驱动篇](https://juejin.im/post/6868882533028511752)

## 简单的开始

我们先简单了解下`binder架构`涉及的几个概念：

- `客户进程`：发起远程方法调用的进程
- `服务进程`：真正服务所运行的进程
- `binder驱动`：`binder通信`的核心，远程调用都是通过它来处理和传递的
- `servicemanager`：一个管理`binder服务`的进程

一个简单版本的跨进程调用的流程：

- `客户进程`把调用信息发给`驱动`，等待`驱动`返回消息
- `驱动`处理一下调用信息，并通知`服务进程`
- `服务进程`接到`驱动`的通知，根据调用信息开始执行
- `服务进程`执行完毕，通知`驱动`
- `驱动`接收到`服务进程`的通知，对数据进行简单处理，通知`客户进程`读取数据
- `客户进程`接收到通知，读取`驱动`返回的数据

需要简单区分的两个名词：

- 实现一个`Binder服务`：利用`Binder`来实现一个自己的`IPC`的服务（怎么使用`Binder`）
- `Binder`的实现：`Binder`的实现原理，涉及到学习`binder驱动`部分了

## Binder的一些疑问

> 第一个问题，`客户进程`要调用`服务进程`的函数，怎么确认要调用哪一个`服务`呢？又怎么知道要调用哪个函数呢？

我们先看几个知识点：

- 在

  ```
  Binder服务
  ```

  的使用上，Android 定义了一些模板类

  - 在

    ```
    服务进程
    ```

    中，如果你写的服务想要支持

    ```
    Binder远程调用
    ```

    ，就必须按我这样写

    - 对于`C++`层，要手动实现`Bn...`、`Bp...`等类（不知道咋写？Android中那么多支持IPC的服务，借鉴呗。好消息是现在有个`HIDL`可以简化开发）
    - 对于`Java`层，因为`AIDL`的存在，会有脚本自动生成`Bn...`、`Bp...`等类，只需要在回调中实现业务逻辑就可以

  - 而对于

    ```
    客户进程
    ```

    ，如果想完成一个远程调用，那就在编程阶段必须

    - 对于`C++`层，要得到想要调用服务的`Bp.....`类型信息才可以，在编程阶段就要写进`客户进程`
    - 对于`Java`层，抄一份想要调用的`服务进程`中`AIDL`在`客户进程`，重新编译一下就可以获得对应服务的接口类了

  - 对于

    ```
    客户进程
    ```

    ，还有一种完成远程调用的方式，就是我们使用的各种

    ```
    ...Manager
    ```

    - 和`c++`层和`AIDL`一样，我们在客户进程也是通过对于`...Manager`类的接口来完成的调用

- 想象一个

  ```
  自由的通信状态
  ```

  应该是：给我个

  ```
  binder驱动
  ```

  我可以忍受，干嘛要集成你的

  ```
  服务接口类
  ```

  啊，这么多条条框框的。我想调用哪家的

  ```
  IPC
  ```

  服务，直接通过

  ```
  驱动
  ```

  发给他就可以了哇。

  - ```
    too young,too naive
    ```

    ，本人考虑并不周全，只是举例哈：

    - 如何取得要调用的服务对象？后台服务千千万，就算不取得，也得要通知这个服务吧
    - 如何传递要调用哪个方法呢？
    - 如何获得远程服务有多少方法呢？
    - 服务进程在编写时除了完成业务逻辑还需要单独和驱动交流工作。

  - 这些问题，

    ```
    Android
    ```

    通过自己设计出来的的Binder相关的

    ```
    模板类
    ```

    都解决了

    - 不知道服务对象在哪里？

      - 那就在`客户进程`加上我设计好的`Bp...`类，里面有提前定义好的接口`asInterface`，直接转换就好了

    - 不知道有哪些可以远程调用的方法？

      - 都给你`Bp...`类了，还要啥`自行车`？

    - 用和

      ```
      binder驱动
      ```

      通信么？

      - 通啥信啊，封装好了，直接调用方法就行了

- 在这里回答

  ```
  第一个问题
  ```

  ：

  - 在编程时，我们根据需求，把需要调用的

    ```
    IPC
    ```

    服务的

    ```
    Bp...
    ```

    类方到我们的客户进程

    - `Bp...`类有个`asInterface`方法，可以通过这个方法转化为我们需要的`代理对象`

  - 而关于调用方法的指定，binder 会在

    ```
    Bp...
    ```

    和

    ```
    Bn...
    ```

    模板类中定义出调用的函数号来

    - 客户进程有了`Bp...`类，也就有了调用的函数号

> 第二个问题，我们自己写的服务什么时候启动的呢？

从常理来讲，我们的`服务`要在`进程启动`的时候就要开始监听外界的消息了，不然万一错过消息怎么办。

`Binder`也是这样做的：

- 对于

  ```
  服务进程
  ```

  来说

  - 会在

    ```
    进程启动
    ```

    时执行

    ```
    joinThreadPool
    ```

    - 这个函数会进入`while循环`读取`binder驱动`的数据

- 对于

  ```
  客户进程
  ```

  来说

  - 在发送远程调用时，会执行

    ```
    transact
    ```

    函数时

    - 这个函数在发送完调用数据给`binder驱动`后，会继续读取，查看返回数据

> servicemanager 怎么没占多少篇幅呢？

首先这不是一个问题，哈哈哈哈！

- `servicemanager`是`Android`用来管理注册的`Binder服务`的，我们的`getService`大概率是找的它。（还没没看源码，不要轻信）
- `servicemanager`并没有按照Android定义的模板来，自己实现了一个简易版本的
- `servicemanager`在Android系统启动时就启动了，这部分在`init.rc`中配置
- 没有`servicemanager`不影响我们认识学习`binder`整体流程和结构
- 有了`servicemanager`，可以加深我们对`binder`的理解

## 导读结束

`binder导读`部分对一些知识点的描述比较简单。细节还是看正文

- 书本中源码是`Android 5.0`，但是`binder`这部分从`9.0`上看也没有太大变化，只是增加了一些`log`打印
- 这代码强悍的生命力不正是我们追求的么，哈哈哈
- 本来还想在导读中说一下`一次拷贝`的事情，算了，留在正文吧

`binder`的课程学习比预计完成时间晚了两周。。。。。

好在真滴可以学到不少东西，下面是正文，未能精通，请谨慎阅读😌

------

# Binder 简介

> ```
> Binder`是Android特有的一种进程间通信方式。`Android Binder`的前身是 `OpenBinder`，最早由 `Dianne Hackborn` 开发并用于`PalmOS`上，后来`Dianne Hackborn`加入了`Google`，在`OpenBinder`的基础上开发了`Android Binder
> ```

`Binder`和传统的`IPC机制`相比，融合了远程过程调用(`RPC`)的概念，而且这种远程调用不是传统的`面向过程`的远程调用，而是一种`面向对象`的远程调用。

从Unix发展而来的`IPC机制`，只能提供比较原始的进程间通信手段，通信的双方必须处理线程同步、内存管理等复杂问题，不但工作量大，而且容易出错。

除了`Socket`、匿名管道以外，传统的`IPC`，例如命名管道、信号量、消息队列等已经从Android中去掉了。

和其他`IPC`相比，`Socket`是一种比较成熟的通信手段，同步控制也很容易实现。`Socket`用于网络通信非常合适，但是用于进程间通信，效率就不太高了。

> Android在架构上一直希望模糊进程的概念，取而代之以`组件`的概念。应用不需要关系`组件`的存放位置、运行在哪个进程、生命周期等问题。随时随地，只要拥有`Binder`对象，就能使用`组件`功能。`Binder`就像一张网，将整个系统的组件，跨越进程和线程，组织在了一起。

`Binder`很强大也很复杂。但我们在使用时却是一件很轻松的事情，不需要考虑线程同步、内存分配等问题。这些复杂的事情都在`Binder`框架中完成了。

正因为如此，Android系统的服务都是利用`Binder`构建的。Android系统服务有几十种之多，这是任何一个其它嵌入式平台所不具备的。

> Android在进程间传递数据使用的是共享内存的方式，这样数据只需要复制一次就能从一个进程到达另一个进程（一般的IPC都需要两步，从用户进程复制到内核，再从内核复制到服务进程），大大提高了数据传输效率。

在安全性方面Android也做了考虑，`Binder`调用时会传递进程的`euid`到服务端，因此，服务端可以通过检查调用进程的权限来决定是否允许其使用所调用的服务。

> 大家可以阅读下知乎大神的 [为什么 Android 要采用 Binder 作为 IPC 机制？](https://www.zhihu.com/question/39440766?sort=created) 来加深下了解，文章有惊喜

## `Binder`对象定义

为了理解方便，把`Binder`相关的对象进行如下定义：

- `Binder实体对象`：`Binder实体对象`就是`Binder`服务的提供者。一个提供`Binder`服务的类必须继承`BBinder`类。因此，有时为了强调对象的类型，也用`BBinder对象`来代替`Binder 实体对象`。
- `Binder引用对象`：`Binder引用对象`是`Binder实体对象`在客户进程的代表，每个引用对象的类型都是`BpBinder`类。同样可以用名称`BpBinder对象`来代替`Binder引用对象`。
- `Binder代理对象`：`Binder代理对象`也称为`Binder接口对象`，它主要为客户端的上层应用提供接口服务，从`IInterface`类派生，它实现了`Binder`服务的函数接口，当然只是一个转调的空壳。通过`Binder代理对象`应用能像使用本地对象一样使用远端的`Binder实体对象`提供的服务。
- `IBinder对象`：`BBinder`和`BpBinder`类都是从`IBinder`对象继承而来的。在很多场合，不需要刻意地去区分`Binder实体对象`和`Binder引用对象`，这是刻意使用`IBinder对象`来统一称呼它们。

`Binder接口对象`主要和应用程序打交道，将`Binder接口对象`和`Binder引用对象`分开的好处**就是`Binder接口对象`可以有很多实例，但是它们包含的是同一个`Binder引用对象`这样方便了用户层的使用**。

关系图如下：
 ![image](images/Binder%E5%AF%B9%E8%B1%A1%E5%85%B3%E7%B3%BB.png)

应用完全可以抛开`Binder接口对象`直接使用`Binder引用对象`，但是这样开发的程序兼容性不好。也正是因为在客户端将`Binder引用对象`和`Binder接口对象`分离，Android才能用一套架构来同时为`Java层`和`Native层`提供`Binder`服务。**隔离后`Binder`底层不需要关心上层的实现细节，只需要和`Binder实体对象`和`Binder引用对象`进行交互。**

## `Binder`的架构

`Binder`通信的参与者由4部分组成：

- `Binder驱动`：`Binder`的核心，实现各种`Binder`的底层操作。
- `ServiceManager`：提供`Binder 实体对象`的名称到`Binder 引用对象`的转换服务
- `服务端`：`Binder`服务的提供者
- `客户端`：`Binder`服务的使用者

关系图如下：
 ![image](images/Binder%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

`Binder 驱动`的简单解释：

- `Binder 驱动`属于`Binder`架构的核心，通过文件系统内的标准接口，如`open`、`ioctl`、`mmap`等，向用户层提供服务。
- 应用层和`Binder 驱动`之间的数据通过`ioctl`接口来完成，并没有使用`read`和`write`系统调用。使用`ioctl`的好处就是一次系统调用就可以完成用户系统和驱动之间的双向数据交换，不但简化了控制逻辑，也提高了传输效率。
- `Binder 驱动`的主要功能就是提供`Binder`通信的通道，维护`Binder`对象的引用计数，转换传输中的`Binder实体对象`和`Binder引用对象`以及管理数据缓存区。

`ServiceManager`的简单解释：

- `ServiceManager`是一个守护进程，它的作用是提供`Binder`服务的查询功能，返回被查询服务的引用。

- 对于一个

  ```
  Binder
  ```

  服务，通常会有一个唯一的字符串标识，只要它向

  ```
  ServiceManager
  ```

  注册了这个标识，应用就可以通过标识来获得

  ```
  Binder
  ```

  服务的引用对象。

  - `ServiceManager`是一个独立的进程，对于其他想要获得`Binder`服务的进程来说，首要任务是获得`ServiceManager`。
  - `ServiceManager`也是通过Binder框架来提供服务的，感觉遇到了鸡生蛋蛋生鸡的情况
  - 为了保证其他进程能获得`ServiceManager`，Android默认注册绑定了`ServiceManager`：`Binder引用对象`的核心数据是一个`Binder实体对象`的引用号，它是在驱动内部分配的一个值。`Binder框架`硬性规定了`0`代表`ServiceManager`。这样用户进程使用参数`0`可以直接构造出`ServiceManager`的`Binder引用对象`。

- 为什么查找

  ```
  Binder
  ```

  服务功能不直接放到

  ```
  Binder驱动
  ```

  中完成呢？

  - 是Android的安全管理要求，不容许任意进程都能注册`Binder`服务，**虽然任意进程都能创建`Binder`服务**，但只有`root进程`或`system进程`才可以不受限制地向`ServiceManager`注册服务。
  - `ServiceManager`中有一张表，里面规定了能够注册服务的进程的用户ID，以及能够注册的服务名称。

> `Binder`服务可以分成两种：`实名服务`和`匿名服务`。除`实名服务`能够通过`ServiceManager`查询到外，它们从开发到使用没有任何其他区别。

对于普通应用开发的`Binder`服务，都是`匿名服务`。对于`匿名服务`无法通过`ServiceManager`获取，那么客户端通过什么方式才能获得`Binder引用对象`呢？

- 首先，`匿名Binder服务`的传输必须基于已经建立好的Binder通信连接，也就是基于实名注册的`Binder`

- `Server`将`匿名Binder对象`写入`Parcel`，接着通过`ioctl`和`binder驱动`通信，`bidner驱动`会对传入的对象进行检查。

- 如果发现传入的

  ```
  type
  ```

  是

  ```
  binder
  ```

  ，则会在当前进程（

  ```
  Server
  ```

  对应的

  ```
  proc
  ```

  ）查找是否有这个节点

  - 如果没有，就用传入的`匿名Binder对象`在用户空间的`指针`和`cookie`（通常是`BBinder`本身）构造一个新的节点，接着得到`匿名Binder对象`的引用，存放在`transaction_data`中等待`Client`去获取。

- `Client`得到这个`匿名Binder`的引用，通过这个引用向位于`Server`中的实体发送请求。

> 匿名Binder这部分暂时收集到的资料就是这些，等学习完后面的再来补充吧

## 组件`Service`和匿名Binder服务

> Android组件`Service`包含了一种启动Java匿名Binder服务的方法，我们来简单看下

- 首先，需要应用调用`bindService`方法发出一个`Intent`
- `Framework`根据`Intent`找到对应的组件`Service`并启动它，此时组件`Service`中`匿名Binder服务`也将同时创建出来。
- 随后，`Framework`会把服务的`IBinder`对象通过`ConnectivityManager`的回调方法`onServiceConnected()`传回到应用
- 这样应用就得到了`匿名Binder服务`的引用对象

在这里Android的`Framework`用`Intent`代替了`Binder服务`的名称来查找对应的服务，同时也承担了`ServiceManager`的工作，解析`Intent`并传回服务的引用对象。如图：
 ![image](images/%E7%BB%84%E4%BB%B6Service%E5%92%8C%E5%8C%BF%E5%90%8DBinder%E6%9C%8D%E5%8A%A1.png)

Android的`Framework`并没有使用新的技术来传递`Binder对象`，因为`Framework`中担任这个角色的`ActivityManagerService`本身就是一个`Binder服务`，最终还是通过`Binder框架`在服务端和客户端之间传递了`IBinder对象`。

## Binder的层次

从代码实现上划分，Binde设计的类可以分为4个层次，如图： ![image](images/Binder%E5%B1%82%E6%AC%A1%E5%85%B3%E7%B3%BB.png)

- 最上层是位于`Framework`中的各种Binder服务类和他们的接口类。这一层的类非常多，例如：`ActivityManagerService`、`WindowManagerService`、`PackageManagerService`等，它们为应用程序提供了各种各样的服务。
- 第二层是用于给最上层的`服务类和接口类`的开发提供基础，如IBinder、BBinder、BpBinder等。
- 第三层是和`Binder驱动`交互的`IPCThreadState`和`ProcessState`类
- 最底层是`Binder驱动`。

从图上看，`libbinder`被拆分成了两个层次，主要原因是：

- 第一层和第二层联系很紧密，第二层中的各种Binder类用来支撑服务类和代理类的开发
- 但是第三层的`IPCThread`和第四层的驱动耦合的很厉害

这些正是`Binder架构`被人诟病的地方，驱动和应用层之间过于耦合了，违反了`Linux驱动设计`的原则，因此，主流的Linux并不愿意接纳`Binder`。



# 2-使用

# 如何使用 Binder

就开发语言而言，`Binder服务`可以用`Java`实现，也可以用`C++`实现，通常，我们在`Java`代码中调用`Java`语言实现的服务，在`C++`代码中调用`C++`编写的服务。但是从原理上讲，`Binder`并没有这种语言平台的限制，混合调用也是可以的。

应用可以在任意的进程和线程中使用`Binder服务`的功能，非常方便、灵活。也不用关心同步、死锁等问题

## 使用`Binder服务`

`C++`层和`Java`层使用`Binder服务`的方式基本一样，包括函数的接口类型都相同，这里以`C++`为例：

- 使用`Binder服务`前要先得到它的引用对象，比如：

  > `sp<IServiceManager> sm = defaultServiceManager();`
  >  `sp<IBinder> binder = sm->getService("serviceName");`

  - `defaultServiceManager()`用来获得`ServiceManager`服务的`Binder引用对象`。前面已经说过`ServiceManager`的`Binder引用对象`是直接用`数值0`作为引用号构造出来的
  - `sm->getService("media.camera")`是查找注册的`Binder服务`，找到后会返回服务的`IBinder对象`，否则返回`NULL`

- 返回的`IBinder对象`，其实是`Binder引用对象`，但是应用需要使用的是`Binder代理对象`，因此需要进行如下转化：

  > `ICameraService service = ICameraService.Stub.asInterface(binder)`

  - 话说这部分应该是Java的转换方式，换成`C++`的版本应该是

  > `sp<ICameraService> service = ICameraService::asInterface(binder)`

- `Binder`中提供了`asInterface`方法来完成`Binder代理对象`的转换，`asInterface`会检查参数中的`IBinder对象`的服务类型是否相符，如果不符将返回`NULL`。

正常情况下，`ServiceManager`进程会在所有应用启动前启动，而且不会停止服务，因此使用`defaultServiceManager()`不必检查返回为`NULL`的情况，但是对于方法`getService()`的返回值需要检查是否为`NULL`，因为服务可能不存在或者未启动。

## `Binder`的混合调用

`Binder`是可以混合调用的，在`C++`中可以调用`Java`实现的`Binder服务`，在`Java`中也可以调用`C++`实现的`Binder服务`。

以`ActivityManagerService`服务为例，其中定义的方法为

```java
public int checkPermission(String permission, int pid, int uid, int mode);
复制代码
```

，我们看下`C++`调用的大体流程（有注释哈）：

```cpp
//获取Binder引用对象
sp<Binder> binder = defaultServiceManager()->getService("activity");
//创建Parcel data 放置方法调用是的参数信息
Parcel data = Parcel.obtain();
//创建Parcel reply 接收返回信息
Parcel reply = Parcel.obtain();
//添加接口识别Token
data.writeInterfaceToken("android.app.IActivityManager");
//添加请求的权限、pid、uid、mode等方法调用信息
data.writeString16(permission);
data.writeInt32(pid);
data.writeInt32(uid);
data.writeInt32(mode);
//调用远程函数号为54的远程服务
binder->transact(54, data, &reply, 0);
复制代码
```

示例代码中最重要的是`54`这个参数，叫做`远程函数号`。这个号码在服务端代码中定义，而且随着系统版本不同可能会有所改变，所以示例方法并不是一个很科学和常用的方法，不要学坏哈。只是想说明`Binder`很强大。。。

## Java层的Binder服务

我们先用`组件Service`来回顾一下流程：

首先，编写一个`AIDL`文件

```
interface ICommandService {
    void setResult_bool(String cmdid, boolean result);
    void setResult_byte(String cmdid, in byte[] resultMsg);
    void setResult_string(String cmdid, String resultMsg);
    void finishCommand(String cmdid, String param);
}
复制代码
```

然后，我们创建一个自定义的Service，然后这样写：

```java
public class CommandService extends Service{
    protected final ICommandService.Stub mBinder = new ICommandService.Stub() {
        @Override
        public void setResult_bool(String cmdid, boolean result) throws RemoteException {
            //do something
        }
        @Override
        public void setResult_byte(String cmdid, byte[] resultMsg) throws RemoteException {
            //do something
        }
        @Override
        public void setResult_string(String cmdid, String resultMsg) throws RemoteException {
            //do something
        }
        @Override
        public void finishCommand(String cmdid, String param) throws RemoteException {
            //do something
        }
    };
    @Override
    public IBinder onBind(Intent intent) {
        //do something
        return mBinder;
    }
}
复制代码
```

`ICommandService.Stub`类就是真正的`Binder服务类`，它是通过我们前面定义的`AIDL`文件自动生成的，不过它是`抽象`的，所以我们需要继承并重写自己的业务逻辑。

## `C++`层的Binder服务

> `C++`层实现`Bidner服务`比较麻烦，主要是没有了`AIDL`的辅助，需要我们手动完成中间层的代码。

书中的源码部分看的有些没太看懂，本人从9.0项目源码中找了一份，我们来看下：

> 温馨提示：下面代码中会出现继承`BnInterface`、`IInterface`、`BpInterface`的操作，以及`DECLARE_META_INTERFACE`和`IMPLEMENT_META_INTERFACE`两个宏，这里暂不做详细解释，后面会有[Binder应用层的核心类详解](#应用层的核心类)。我们暂先理解为：继承`BnInterface`为实现服务端功能、继承`BpInterface`为实现客户端、继承`IInterface`表示定义要提供的服务接口

首先，是一个`IImagePlayerService.h`的头文件，定义了：

- `IImagePlayerService`：通过`纯虚函数`函数定义计划提供的Binder服务接口
- `BnImagePlayerService`：根据`IImagePlayerService`中提供的接口，定义对应的枚举对象；重载`onTransact()`虚拟函数

我们看下代码：

```cpp
    class IImagePlayerService: public IInterface {
      public:
        DECLARE_META_INTERFACE(ImagePlayerService);
        
        virtual int init() = 0;
        virtual int start() = 0;
        virtual int release() = 0;
        virtual int setTranslate(float tx, float ty) = 0;
    };
    class BnImagePlayerService: public BnInterface<IImagePlayerService> {
      public:
        enum {
            IMAGE_INIT = IBinder::FIRST_CALL_TRANSACTION,
            IMAGE_START,
            IMAGE_RELEASE,
            IMAGE_SET_TRANSLATE,
        };
        virtual status_t onTransact(uint32_t code, const Parcel& data,
                                    Parcel* reply, uint32_t flags = 0);
    };
复制代码
```

`IImagePlayerService.h`这部分的意思大概是：我们自己的Binder服务要提供哪些功能，先把这部分定义出来。

我们再来看下`IImagePlayerService.h`的实现`IImagePlayerService.cpp`的内容：

```cpp
    class BpImagePlayerService : public BpInterface<IImagePlayerService> {
      public:
        BpImagePlayerService(const sp<IBinder>& impl)
            : BpInterface<IImagePlayerService>(impl) {
        }
        virtual int init() {
            Parcel data, reply;
            data.writeInterfaceToken(IImagePlayerService::getInterfaceDescriptor());
            remote()->transact(BnImagePlayerService::IMAGE_INIT, data, &reply);
            return NO_ERROR;
        }
        virtual int start() {
            Parcel data, reply;
            data.writeInterfaceToken(IImagePlayerService::getInterfaceDescriptor());
            remote()->transact(BnImagePlayerService::IMAGE_START, data, &reply);
            return NO_ERROR;
        }
        virtual int release() {
            Parcel data, reply;
            data.writeInterfaceToken(IImagePlayerService::getInterfaceDescriptor());
            remote()->transact(BnImagePlayerService::IMAGE_RELEASE, data, &reply);
            return NO_ERROR;
        }
        virtual int setTranslate(float tx, float ty) {
            Parcel data, reply;
            data.writeInterfaceToken(IImagePlayerService::getInterfaceDescriptor());

            data.writeFloat(tx);
            data.writeFloat(ty);
            remote()->transact(BnImagePlayerService::IMAGE_SET_TRANSLATE, data, &reply);
            return NO_ERROR;
        }
    };
    
    IMPLEMENT_META_INTERFACE(ImagePlayerService, "droidlogic.IImagePlayerService");
    
    status_t BnImagePlayerService::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) {
        switch (code) {
            case IMAGE_INIT: {
                CHECK_INTERFACE(IImagePlayerService, data, reply);
                int result = init();
                reply->writeInt32(result);
                return NO_ERROR;
            }
            case IMAGE_START: {
                CHECK_INTERFACE(IImagePlayerService, data, reply);
                int result = start();
                reply->writeInt32(result);
                return NO_ERROR;
            }
            case IMAGE_RELEASE: {
                CHECK_INTERFACE(IImagePlayerService, data, reply);
                int result = release();
                reply->writeInt32(result);
                return NO_ERROR;
            }
            case IMAGE_SET_TRANSLATE: {
                CHECK_INTERFACE(IImagePlayerService, data, reply);
                int result = setTranslate(data.readFloat(), data.readFloat());
                reply->writeInt32(result);
                return NO_ERROR;
            }
            default: {
                return BBinder::onTransact(code, data, reply, flags);
            }
        }
        // should be unreachable
        return NO_ERROR;
    }
复制代码
```

`IImagePlayerService.cpp` 做了这几件事情：

- 实现

  ```
  BnImagePlayerService
  ```

  类的

  ```
  onTransact
  ```

  函数。

  - `BnImagePlayerService`类的对象实例会运行在服务进程中

  - `onTransact`函数在运行时会被底层的IPCThreadState类调用。

  - 函数实现比较公式化，根据参数中的函数号来调用对应的函数。

  - 函数中的

    ```
    CHECK_INTERFACE
    ```

    用来检查

    ```
    Parcel
    ```

    对象中的数据是否正确

    - 检查原理是先取出`Parcel`对象中客户端传来的类描述字符串
    - 与当前类中定义的字符串比较，相同则正确

  - 然后按照函数参数的排列顺序取出参数，并使用这些参数调用服务端的函数

    - 可以参考`case IMAGE_SET_TRANSLATE`

- 定义

  ```
  BpImagePlayerService
  ```

  类。

  - `Binder服务`在客户端的接口类

  - `BpImagePlayerService`类的对象实例会运行在客户端的进程中

  - 继承`BpInterface<IImagePlayerService>`

  - 实现

    ```
    IImagePlayerService
    ```

    中定义的

    ```
    纯虚函数
    ```

    ，把函数调用的信息发送给远端

    - `data.writeInterfaceToken`把类的描述字符串放到`Parcel`中
    - 通过`remote()`函数得到`Binder引用对象`的指针
    - 调用`Binder引用对象`的`transact()`函数把数据传给底层
    - `IBinder`的`transact()`函数其实有4个参数，由于第四个参数flags定义了默认值0，因此调用`transact()`常常只使用3个参数。
    - 如果客户端调用`Binder服务`是不打算等待对方运行结束就继续执行，可以使用`TF_ONE_WAY`作为第四个参数以异步的方式调用`Binder服务`。这种方式也就意味着无法得到返回值了。

会不会有个疑惑`IImagePlayerService.cpp`为什么把这两个类定义在一起来？

- 主要原因是两个类中都是用了相同的函数号码和类的描述字符串。如果分开，需要维护两份，容易失误

到这里和`Binde服务`相关的定义就结束了，接下来就是最后一步了，实现我们`Binder服务`的业务逻辑了，我们看下`ImagePlayerService.h`的头文件定义

```cpp
    class ImagePlayerService :  public BnImagePlayerService {
      public:
        ImagePlayerService();
        virtual ~ImagePlayerService();

        virtual int init();
        virtual int start();
        virtual int release();
        virtual int setTranslate(float tx, float ty);
    };
复制代码
```

这里我们需要注意的是继承`BnImagePlayerService`。具体函数的实现其实在`ImagePlayerService.cpp`完成：

```cpp
int ImagePlayerService::init() {
    //.....
}
//.....省略部分方法
int ImagePlayerService::setTranslate(float tx, float ty) {
    //.....
}
复制代码
```

书中并没有定义头文件，直接做了服务类的实现。

到这里`C++`版本的`Binder服务`就完成了，我们看下整体代码结构：
 ![image](http://cdn.hualee.top/C%2B%2BBinder%E6%9C%8D%E5%8A%A1.png)

## Google的`HIDL`

不过，上面的**这种操作已经过`OUT`了**，Android 从8.0开始做了架构调整（一个称为`Treble`的模式），增加了`HIDL`接口定义语言进行`HAL层`和应用层的解耦合优化。

> HAL 接口定义语言（简称 HIDL，发音为“hide-l”）是用于指定 HAL 和其用户之间的接口的一种接口描述语言 (IDL)。HIDL 允许指定类型和方法调用（会汇集到接口和软件包中）。从更广泛的意义上来说，HIDL 是用于在可以独立编译的代码库之间进行通信的系统。

> HIDL 旨在用于进程间通信 (IPC)。进程之间的通信采用 Binder 机制。对于必须与进程相关联的代码库，还可以使用直通模式（在 Java 中不受支持）。

从介绍来看，`HIDL`在`Binder机制`的基础上增加了一个`直通模式`，这部分暂不深入深究哈，当前紧要任务还是看完`Binder的核心实现`，官方传送门：[HIDL](https://source.android.google.cn/devices/architecture/hidl?hl=zh_cn)

暂且把它当成和`AIDL`一样的接口定义语言就好，只是`AIDL`用与`Java层`的服务实现，而`HIDL`用于`C++层`的服务实现。

# 应用层的核心类

> 前面我们了解了Binder的使用，都是基于`libbinder`库来实现的，我们看下`libbinder`库中的一些核心类，源码位置：`frameworks/native/libs/binder`

`libbinder`库中的`IInterface`类、`BpInterface`类、`BnInterface`类、`BBinder`类、`BpBinder`类、`IBinder`类共同构成了Binder应用层的核心类。

## `IInterface`类中的两个宏

> 在`C++层Binder服务实现`中，还有一些知识没有细说，包括`IInterface`接口类、`BpInterface`模板类、`BnInterface`模板类，以及宏`DECLARE_META_INTERFACE`和宏`IMPLEMENT_META_INTERFACE`。

我们先来看下宏`DECLARE_META_INTERFACE`的定义：

```h
#define DECLARE_META_INTERFACE(INTERFACE)                               \
    static const ::android::String16 descriptor;                        \
    static ::android::sp<I##INTERFACE> asInterface(                     \
            const ::android::sp<::android::IBinder>& obj);              \
    virtual const ::android::String16& getInterfaceDescriptor() const;  \
    I##INTERFACE();                                                     \
    virtual ~I##INTERFACE();            
复制代码
```

这个宏定义了一些成员变量和方法，在类中使用这个宏就意味着将这些成员变量和方法加入到了这个类中。转换逻辑也比较简单，**把`##INTERFACE`替换成传入的`参数`即可**，对于前面的例子，相当于添加了如下代码：

```cpp
    static const ::android::String16 descriptor;
    static ::android::sp<IImagePlayerService> asInterface(
            const ::android::sp<::android::IBinder>& obj);
    virtual const ::android::String16& getInterfaceDescriptor() const;
    IImagePlayerService();
    virtual ~IImagePlayerService();  
复制代码
```

这里添加了静态成员变量`descriptor`、静态函数`asInterface`，以及类`构造函数`和`析构函数`。可能这些变量和函数是每个`Binder服务`必须实现的，为了方便开发，Android把它们定义成了宏。

我们再来看下宏`IMPLEMENT_META_INTERFACE`的定义：

```h
#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    const ::android::String16 I##INTERFACE::descriptor(NAME);           \
    const ::android::String16&                                          \
            I##INTERFACE::getInterfaceDescriptor() const {              \
        return I##INTERFACE::descriptor;                                \
    }                                                                   \
    ::android::sp<I##INTERFACE> I##INTERFACE::asInterface(              \
            const ::android::sp<::android::IBinder>& obj)               \
    {                                                                   \
        ::android::sp<I##INTERFACE> intr;                               \
        if (obj != NULL) {                                              \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == NULL) {                                         \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    I##INTERFACE::I##INTERFACE() { }                                    \
    I##INTERFACE::~I##INTERFACE() { }                                   \
复制代码
```

对于实例代码中的

```cpp
IMPLEMENT_META_INTERFACE(ImagePlayerService, "droidlogic.IImagePlayerService");
复制代码
```

则会转换为：

```cpp
    const ::android::String16 IImagePlayerService::descriptor("droidlogic.IImagePlayerService");
    const ::android::String16&
            IImagePlayerService::getInterfaceDescriptor() const {
        return IImagePlayerService::descriptor;
    }
    ::android::sp<IImagePlayerService> IImagePlayerService::asInterface(
            const ::android::sp<::android::IBinder>& obj)
    {
        ::android::sp<IImagePlayerService> intr;
        if (obj != NULL) {
            intr = static_cast<IImagePlayerService*>(
                obj->queryLocalInterface(
                        IImagePlayerService::descriptor).get());
            if (intr == NULL) {
                intr = new BpImagePlayerService(obj);
            }
        }
        return intr;
    }
    IImagePlayerService::IImagePlayerService() { }
    IImagePlayerService::~IImagePlayerService() { }    
复制代码
```

- 变量`descriptor`是一个字符串，用来描述`Bidner服务`。在检查客户端传来的参数是否合法时会用到。
- 函数`asInterface`主要用来把`Binder引用对象`转换为`Binder代理对象`。这个函数很关键，但我们目前还不能看明白，先记住这个方法，我们继续学习核心类。

## Binder核心类的关系

我们先来看下核心类的继承关系图：
 ![image](images/libbinder%E4%B8%AD%E7%B1%BB%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

### 继承关系中虚线代表了什么？

> 上图中实线表示的继承关系可以从源码中找到，关键是**虚线的继承部分**，没有找到明显的继承关系，我们根据示例代码来分析下。

我们看下`BnInterface`的定义先：

```cpp
template<typename INTERFACE>
class BnInterface : public INTERFACE, public BBinder
{
public:
    virtual sp<IInterface>      queryLocalInterface(const String16& _descriptor);
    virtual const String16&     getInterfaceDescriptor() const;

protected:
    typedef INTERFACE           BaseInterface;
    virtual IBinder*            onAsBinder();
};
复制代码
```

这是一个模板类，而在`C++`版本Binder服务中，我们的类是这样定义的：

```cpp
class IImagePlayerService: public IInterface{};
class BnImagePlayerService: public BnInterface<IImagePlayerService>{};
复制代码
```

对于模板类`BnInterface`，我们使用的时候要给一个`INTERFACE`，也就是`IImagePlayerService`。

> 其实很像Java的泛型不是？

我们看到，模板类`BnInterface`的定义中，类继承部分是这样写的：

```cpp
class BnInterface : public INTERFACE, public BBinder
复制代码
```

而对于示例代码中的`BnInterface<IImagePlayerService>`则意味`BnInterface`会把`IImagePlayerService`当做`INTERFACE`继承，也就转化为

```cpp
class BnInterface : public IImagePlayerService, public BBinder
复制代码
```

而`IImagePlayerService`又是继承自`IInterface`。

所以模板类`BnInterface`最后还是会继承`IInterface`，可能这层关系有些隐晦，所以就用虚线表示了。同样`BpInterface`也是如此。

### 一张更容易理解的核心类图

![image](images/Binder%E8%BF%9B%E7%A8%8B%E9%97%B4%E6%A0%B8%E5%BF%83%E7%B1%BB%E5%9B%BE.png)

#### 图解一：`BpInterface` 和 `BpBinder`的聚合关系

这张图中，多了一个`BpInterface`和`BpBinder`的聚合关系，我们通过源码来认识下：

首先是`BpInterface`模板类的定义：

```cpp
template<typename INTERFACE>
class BpInterface : public INTERFACE, public BpRefBase
{
public:
    explicit                    BpInterface(const sp<IBinder>& remote);

protected:
    typedef INTERFACE           BaseInterface;
    virtual IBinder*            onAsBinder();
};
复制代码
```

我们这次重点关注的是：

- 显式构造函数

  ```
  BpInterface(const sp<IBinder>& remote)
  ```

  - 传入了一个`IBinder`类型的指针，其实就是`BpBinder`
  - 这个指针会在父类`BpRefBase`的构造函数中做一系列的处理
  - 很多远程方法调用都是使用这个`BpBinder`

- 继承类

  ```
  BpRefBase
  ```

  - 真正关联`BpBinder`的操作
  - 主要成员`mRemote`，一个指向`IBinder`的指针
  - 定义了`remote()`方法

我们看下`BpRefBase`类的部分信息：

```cpp
class BpRefBase : public virtual RefBase
{
protected:
    explicit                BpRefBase(const sp<IBinder>& o);
    virtual                 ~BpRefBase();
    //......
    inline  IBinder*        remote()                { return mRemote; }
private:
    //......
    IBinder* const          mRemote;
    //......
};
复制代码
```

#### 图解二：服务端的调用逻辑

> `BnInterface`类是开发Binder服务必须继承的一个模板类，它继承`BBinder`的同时也继承了`IInterface`类（这部分的继承逻辑前面已经细讲了哈）。

`BBinder`是服务端的核心，实际的函数实现都在`BBinder`的继承类中。先看下`BBinder`的部分函数定义：

```cpp
class BBinder : public IBinder
{
public:
    //......
    virtual const String16& getInterfaceDescriptor() const;
    virtual bool        isBinderAlive() const;
    virtual status_t    pingBinder();
    virtual status_t    dump(int fd, const Vector<String16>& args);

    // NOLINTNEXTLINE(google-default-arguments)
    virtual status_t    transact(   uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0) final;
    // NOLINTNEXTLINE(google-default-arguments)
    virtual status_t    linkToDeath(const sp<DeathRecipient>& recipient,
                                    void* cookie = nullptr,
                                    uint32_t flags = 0);
    // NOLINTNEXTLINE(google-default-arguments)
    virtual status_t    unlinkToDeath(  const wp<DeathRecipient>& recipient,
                                        void* cookie = nullptr,
                                        uint32_t flags = 0,
                                        wp<DeathRecipient>* outRecipient = nullptr);
//......
};
复制代码
```

`BBinder`负责和更底层的`IPCThread`打交道。但是，当`Binder调用`从`Binder驱`动传递过来时，`IPCThread`会调用`BBinder`的`transact()`函数，然后`transact()`函数会调用继承类的`onTransact()`函数。

这部分其实要和`IPCThread`结合起来看，先跟着书本走吧，等下返个场，哈哈

#### 图解三：客户端的调用逻辑

```
BpBinder`类是客户端Binder的核心，也是继承自`IBinder`类。我们在`图解一`已经说过：`BpInterface`类并不是从`BpBinder`类继承来的，而是继承自`BpRefBase`。`BpBinder`和`BpInterface`是`聚合关系
```

这么做是主要是为了：

- 使得上层开发更加灵活
- `Binder`的内部实现和上层应用完全隔离开，用户类完全不知道`BpBinder`的存在
- `BpBinder`改动了，用户类不用重新编译也能运行

`BpBinder`负责和更底层的`IPCThread`打交道。客户端通过`remote()->transact(...)`调用`IBinder`的`transact`虚函数，实际调用的是`BpBinder`的`transact`函数。`transact`函数会调用`IPCThread`类的函数来向驱动传递数据。

### Binder类继承关系的总结

- 从继承关系上看`IBinder`是`BBinder`和`BpBinder`的基类。

- 在应用层看见的只是

  ```
  IBinder
  ```

  ，

  ```
  BBinder
  ```

  和

  ```
  BpBinder
  ```

  都藏在后面

  - 这样在概念上就统一了
  - 从`Binder`的使用角度，不用刻意地区分当前对象是`实体对象`还是`引用对象`
  - 只要获得`IBinder`对象，就可以使用服务

- ```
  IBinder
  ```

  中有两个接口，

  ```
  localBinder
  ```

  和

  ```
  remoteBinder
  ```

  ，从返回值我们就可以知道他们代表的

  ```
  Binder服务对象
  ```

  了：

  - `virtual BBinder*  localBinder();`
  - `virtual BpBinder* remoteBinder();`
  - 对于服务端，`Binder服务`就在本进程中，可以用`localBinder`获取IBinder对象，而`remoteBinder`则会返回`NULL`
  - 对于客户端，`Binder服务`在其他进程中，可以用`remoteBinder`获取IBinder对象，而`localBinder`则会返回`NULL`
  - 如果想**确定一个IBInder对象到底是`BpBinder`还是`BBinder`对象时，通过这两个接口就可以判断**

## 函数asInterface的奥秘

> 理解了`Binder`核心类关系后，我们再来看下`asInterface()`函数

`asInterface()`函数就是把Binder引用对象转换成代理对象的关键，`asInterface()`函数的宏定义如下：

```cpp
    ::android::sp<I##INTERFACE> I##INTERFACE::asInterface(              \
            const ::android::sp<::android::IBinder>& obj)               \
    {                                                                   \
        ::android::sp<I##INTERFACE> intr;                               \
        if (obj != nullptr) {                                           \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == nullptr) {                                      \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    } 
复制代码
```

上面代码中变量`obj`的类型是`IBinder`，但是`obj`是参数，它既可能是`BpBinder对象`，也可能是`BBinder对象`，判断逻辑如下：

- 如果

  ```
  obj
  ```

  传入的是一个

  ```
  BpBinder对象
  ```

  - 在`BpBinder类`中没有重载`queryLocalInterface`函数，因此，调用的是其基类`IBinder`的函数，直接返回`sp<NULL>`
  - 通过`sp<NULL>.get`结果还是`NULL`
  - 所以`intr`为`NULL`
  - 这样接下来就会new一个Bp##INTERFACE对象

- 如果

  ```
  obj
  ```

  传入的是一个

  ```
  BBinder对象
  ```

  ，在这种情形下，因为继承关系，

  ```
  BBinder对象
  ```

  同时也会是一个

  ```
  BnInterface对象
  ```

  ，在

  ```
  BnInterface类
  ```

  中重载了

  ```
  queryLocalInterface
  ```

  函数，代码如下：

  > ```cpp
  > template<typename INTERFACE>
  > inline sp<IInterface> BnInterface<INTERFACE>::queryLocalInterface(
  >         const String16& _descriptor)
  > {
  >     if (_descriptor == INTERFACE::descriptor) return this;
  >     return nullptr;
  > }
  > 复制代码
  > ```

  - 如果参数传递的描述字符串和类的描述字符串相同，则返回`this`指针，也就是`obj`本身
  - 否则，返回`NULL`

`asIntreface`函数的作用总结如下：

- 在客户端调用`asInterface`，会创建一个`Binder代理对象`，代理对象中包含了`Binder引用对象`。
- 在服务端调用`asInterface`，直接返回参数的指针。因为在服务进程中可以直接调用服务类，无需创建`Binder代理对象`。

## Binder的`死亡通知`

作为`Binder服务`的提供者，`Binder实体对象`也会死亡，可能是正常退出，也可能是意外崩溃。一旦实体对象死亡，`Binder引用对象`需要有机制来得到通知。

### 注册接收Binde的`死亡通知`

IBinder中提供了一个纯虚函数`linkToDeath`，定义如下：

```cpp
    virtual status_t        linkToDeath(const sp<DeathRecipient>& recipient,
                                        void* cookie = nullptr,
                                        uint32_t flags = 0) = 0;
复制代码
```

函数具体实现暂不深究，我们看下使用方式：

```cpp
class HeapCache : public IBinder::DeathRecipient
{
public:
    HeapCache(sp<IBinder> binder){
        mService=binder
    };
    virtual void binderDied(const wp<IBinder>& who);

private:
    wp<IBinder> mService;
};
void HeapCache::binderDied(const wp<IBinder>& binder)
{
    //死亡会调用此方法
}
复制代码
```

在继承`DeathRecipient`类中需要重载`binderDied`函数，当Binder服务死亡时，这个函数会被回调。

`linkToDeath`函数会调用IPCThreadState的函数来告诉驱动需要接收某个Binder服务死亡的通知消息，我们看下BpBinder的相关代码：

```cpp
status_t BpBinder::linkToDeath(
    const sp<DeathRecipient>& recipient, void* cookie, uint32_t flags)
{
    //......
    IPCThreadState* self = IPCThreadState::self();
    self->requestDeathNotification(mHandle, this);
    self->flushCommands();
    //......
}
复制代码
```

### 发送Binder`死亡通知`

一旦驱动检测到某个`Binder服务`已经死亡，会根据设置向应用层发送消息，`IPCThreadState`中检测到死亡消息后，利用保存在消息中的`BpBinder`的指针，调用其`sendObituary`函数，`IPCThreadState.cpp`中代码定义如下：

```cpp
    case BR_DEAD_BINDER:
        {
            BpBinder *proxy = (BpBinder*)mIn.readPointer();
            proxy->sendObituary();
            mOut.writeInt32(BC_DEAD_BINDER_DONE);
            mOut.writePointer((uintptr_t)proxy);
        } break;
复制代码
```

`sendObituary`函数的主要工作是调用`reportOneDeath`函数通知上次应用，代码定义如下：

```cpp
void BpBinder::sendObituary()
{
    //......
    //这部的代码没看懂是干啥用的
    Vector<Obituary>* obits = mObituaries;
    if(obits != nullptr) {
        ALOGV("Clearing sent death notification: %p handle %d\n", this, mHandle);
        IPCThreadState* self = IPCThreadState::self();
        self->clearDeathNotification(mHandle, this);
        self->flushCommands();
        mObituaries = nullptr;
    }
    //这部的代码没看懂是干啥用的
    //......
    
    const size_t N = obits->size();
    for (size_t i=0; i<N; i++) {
        reportOneDeath(obits->itemAt(i));
    }
    //......
}
void BpBinder::reportOneDeath(const Obituary& obit)
{
    sp<DeathRecipient> recipient = obit.recipient.promote();
    ALOGV("Reporting death to recipient: %p\n", recipient.get());
    if (recipient == nullptr) return;

    recipient->binderDied(this);
}
复制代码
```

监听死亡通知对象都被保存在了`Vector类型`的成员变量`mObituaries`中，然后通过循环来调用`reportOneDeath`通知所有对象。

而在`reportOneDeath`函数中把弱引用转变成强引用后再调用`binderDied`。

到这里就和**注册接收Binde的`死亡通知`的操作**接上拉。

## Java 层的 Binder 类

了解了`C++层`的`Binder`类体系后，我们再来看下`Java层`提供的`Binder`体系。 ![image](images/Java%E5%B1%82Binder%E7%B1%BB%E5%9B%BE.png) `Java层`的`Binder`虽然设计了自己的框架，但最后还是通过`native层`的`BpBinder`和`BBinder`。`Java层`的`Binder框架`和`C++层`的原理是一样的，而且`Java层`可以通过`AIDL`来生成相关的细节代码，所以我们这次重点关注：**`Java框架`是通过什么方式和`C++`中的`BpBinder`和`BBinder`关联起来的?**

### Java 层的 Binder 服务类

开发一个Java服务，必须继承`android.os.Binder`类。`Binder`类中最重要的成员变量如下：

```java
public class Binder implements IBinder {
    //......
    private final long mObject;
    private IInterface mOwner;
    private String mDescriptor;
    //......
    
    public Binder(@Nullable String descriptor)  {
        //获取 JavaBBinderHolder 类的实例
        mObject = getNativeBBinderHolder();
        //registerNativeAllocation 是在 Android8.0（API27）引入的一种辅助回收native内存的机制，本身并不复杂，抽个时间好好看看它
        NoImagePreloadHolder.sRegistry.registerNativeAllocation(this, mObject);
        //......
        mDescriptor = descriptor;
    }
}

static jlong android_os_Binder_getNativeBBinderHolder(JNIEnv* env, jobject clazz)
{
    JavaBBinderHolder* jbh = new JavaBBinderHolder();
    return (jlong) jbh;
}
复制代码
```

- `mObject` 存放的是native层关联对象额指针
- `mOwner` 存放的是Binder继承类的引用，实际上就是 Java Binder对象本身
- `mDescriptor` 存放的是类的描述字符串

`android.os.Binder`类在构造方法中会通过`getNativeBBinderHolder()`本地方法获取`JavaBBinderHolder`类的实例，并把指针保存在`Java层`的`mObject`变量中。我们看下`JavaBBinderHolder`是干啥的：

```cpp
class JavaBBinderHolder
{
public:
    sp<JavaBBinder> get(JNIEnv* env, jobject obj)
    {
        AutoMutex _l(mLock);
        sp<JavaBBinder> b = mBinder.promote();
        if (b == NULL) {
            b = new JavaBBinder(env, obj);
            //......
            mBinder = b;
            //......
        }

        return b;
    }
    //......
private:
    Mutex           mLock;
    wp<JavaBBinder> mBinder;
    //......
};
复制代码
```

`JavaBBinderHolder`类在第一次调用`get`函数时创建一个`JavaBBinder`对象，并通过`mBinder`保存`JavaBBinder`对象的弱引用。而`JavaBBinder`是从`BBinder`类继承过来的，我们看下`JavaBBinder`的核心定义：

```cpp
class JavaBBinder : public BBinder
{
    //......
    status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0) override
    {
        JNIEnv* env = javavm_to_jnienv(mVM);

        IPCThreadState* thread_state = IPCThreadState::self();
        const int32_t strict_policy_before = thread_state->getStrictModePolicy();

        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
    }
    
private:
    JavaVM* const   mVM;
    jobject const   mObject;  // GlobalRef to Java Binder
    //......
};
复制代码
```

- `mVM` 保存的是Java虚拟机的指针
- `mObject` 保存的是`android.os.Binder`在`native层`的`JNI对象`，就是调用`JavaBBinderHolder`的`get`函数是参入的`jobject obj`参数

到这里，我们就可以梳理出`Java层`和`native层`的`Binder服务`关系：

- 首先，`Java层`的`Binder`对象在构造方法中，通过JNI调用，是的`mObject`属性，持有了`native`层`JavaBBinderHolder`对象的指针

- 而在`JavaBBinderHolder`中，`mBinder`变量持有一个`JavaBBinder`对象的弱引用

- ```
  JavaBBinder
  ```

  继承自

  ```
  BBinder
  ```

  ，并重写了

  ```
  onTransact
  ```

  - `onTransact`方法中会通过`JNI`调用`Java层`的`android.os.Binder`的`execTransact`方法

### Java层的Binder引用类

应用在`Java层`调用`Binder`的操作同样也是通过`Binder代理对象`来完成的，和`C++`中一样，`代理对象`的生成也是通过接口类中的`asIntreface`方法来完成。`Java`中的接口类一般都可以通过`AIDL`自动生成。格式如下：

```java
    public static com.example.factory.ICommandService asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.fengmi.factory.ICommandService))) {
        return ((com.example.factory.ICommandService)iin);
      }
      return new com.example.factory.ICommandService.Stub.Proxy(obj);
    }
复制代码
```

这个`asIntreface`方法的逻辑其实和`C++`中的`asIntreface`一样：

- 先通过`queryLocalInterface`方法的返回值判断传入的参数是否是在服务端调用传入的`Binder服务对象`，如果是直接返回该对象

- 否就就认为

  ```
  obj
  ```

  是

  ```
  Binder引用对象
  ```

  ，这时需要新创建一个

  ```
  Binder代理对象
  ```

  ，也就是

  ```
  return new com.example.factory.ICommandService.Stub.Proxy(obj);
  ```

  - `com.example.factory.ICommandService.Stub.Proxy`就是`AIDL`自动生成的代理对象。这个类实现了`ICommandService`接口

  - 这样对于应用开发，只需要关系

    ```
    ICommandService
    ```

    接口包含了哪些方法即可，真正实现细节无需关心。

    - 只能说`Binder`设计的很是巧妙，整个体系在功能实现上基本形式化了

- `Java层`的`Binder代理对象`实现的思路和`C++层`如出一辙，可以阅读源码`frameworks/base/core/java/android/os/*Binder*`来加深印象

我们再来看下`asInterface`方法的参数`android.os.IBinder obj`，通常情况我们都是通过`ServiceManager`的`getService`方法来得到这个`IBinder对象`，我们看下`getService`的部分代码（由于和Android 5.0不太一样，加注释了哟）：

```
//Android 9 的这部分源码和书中有些不同，不过原理性的东西还是一样的
    //file::ServiceManager.java
    public static IBinder getService(String name) {
        try {
            // 查看缓存
            IBinder service = sCache.get(name);
            if (service != null) {
                // 有缓存，直接返回缓存对象
                return service;
            } else {
                // allowBlocking是在设置mWarnOnBlocking属性，看样子是将属性设置为可接受不受信任代码的调用
                // 真正获取Binder对象的是rawGetService
                // rawGetService 最后调用了 ServiceManagerNative的getService()
                //真正触发binder通信并获取到binder对象的，就是在ServiceManagerNative中
                return Binder.allowBlocking(rawGetService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }
    //file::Binder.java
    //建议看看英文注释吧，木有完全理解。
    public static IBinder allowBlocking(IBinder binder) {
        try {
            if (binder instanceof BinderProxy) {
                //仅仅在binder对象是客户端时进行设置
                ((BinderProxy) binder).mWarnOnBlocking = false;
            } else if (binder != null && binder.getInterfaceDescriptor() != null
                    && binder.queryLocalInterface(binder.getInterfaceDescriptor()) == null) {
                Log.w(TAG, "Unable to allow blocking on interface " + binder);
            }
        } catch (RemoteException ignored) {
        }
        return binder;
    }
    
    //file::ServiceManager.java
    private static IBinder rawGetService(String name) throws RemoteException {
        //......
        final IBinder binder = getIServiceManager().getService(name);
        //忽略很多代码
        return binder;
    }
    //file::ServiceManager.java
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        // 最后执行到了ServiceManagerNative
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }
复制代码
```

再看下`ServiceManagerNative`的部分代码，已添加注释：

```
    //file::ServiceManagerNative.java
    public IBinder getService(String name) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        //设置要获取的Binder服务的描述符
        data.writeInterfaceToken(IServiceManager.descriptor);
        //设置要获取的Binder服务的名称
        data.writeString(name);
        //通过 transact 发送Binder请求，怎么发送的据说后面原理会讲到
        // GET_SERVICE_TRANSACTION 就是获取特定binder服务请求号
        mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
        
        //通过Parcel类的readStrongBinder来获得IBinder对象
        //readStrongBinder是一个本地方法，它的实现是nativeReadStrongBinder
        //然后调用到了android_os_Parcel_readStrongBinder
        IBinder binder = reply.readStrongBinder();
        reply.recycle();
        data.recycle();
        return binder;
    }
    
    //file::android_os_Parcel.cpp
    //方法中的 parcel->readStrongBinder() 会返回一个C++层的IBinder对象
    //这个C++层的IBinder对象怎么创建的据说后面原理部分会讲
    //javaObjectForIBinder则根据C++层的IBinder对象创建一个Java层的BinderProxy对象
    static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        return javaObjectForIBinder(env, parcel->readStrongBinder());
    }
    return NULL;
}
复制代码
```

`javaObjectForIBinder`实现了在`native层`创建`Java类`实例，我们仔细看下这个神奇的方法：

```cpp
// android_util_binder.cpp
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    if (val == NULL) return NULL;

    if (val->checkSubclass(&gBinderOffsets)) {
        // 如果是JavaBBinder 说明在服务端，直接返回该对象
        // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object.
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        LOGDEATH("objectForBinder %p: it's our own %p!\n", val.get(), object);
        return object;
    }

    // For the rest of the function we will hold this lock, to serialize
    // looking/creation/destruction of Java proxies for native Binder proxies.
    //加锁
    AutoMutex _l(gProxyLock);

    // 查看缓存中是否存在 BinderProxyNativeData，没有就新建
    // BinderProxyNativeData就是一结构体，存放了BpBinder的指针和对应注册Binder服务死亡通知的对象List的指针
    BinderProxyNativeData* nativeData = gNativeDataCache;
    if (nativeData == nullptr) {
        nativeData = new BinderProxyNativeData();
    }
    // gNativeDataCache is now logically empty.
    // 通过JNI调用CallStaticObjectMethod创建BinderProxy对象
    // 这里的JNI调用其实就是在调用Java层BinderProxy类的getInstance方法
    // getInstance 需要传入long nativeData, long iBinder两个指针
    // 所以，在创建完Java层的BinderProxy对象后，它已经持有了BpBinder和BinderProxyNativeData的指针
    jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
            gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
            
    if (env->ExceptionCheck()) {
        //异常检查
        // In the exception case, getInstance still took ownership of nativeData.
        gNativeDataCache = nullptr;
        return NULL;
    }
    //通过 getBPNativeData 获取object对象中的mNativeData字段的数据
    //并转化为BinderProxyNativeData指针
    BinderProxyNativeData* actualNativeData = getBPNativeData(env, object);
    if (actualNativeData == nativeData) {
        // 两个指针相等
        // New BinderProxy; we still have exclusive access.
        // 新建死亡通知注册表
        nativeData->mOrgue = new DeathRecipientList;
        // 把 mObject 指针指向 传入的IBiner，也就是BpBinder
        nativeData->mObject = val;
        gNativeDataCache = nullptr;
        //.....
    } else {
        // nativeData wasn't used. Reuse it the next time.
        gNativeDataCache = nativeData;
    }

    return object;
}
复制代码
```

经过这个方法的操作，`Java层`获得了一个在`native层`创建的`BinderProxy`实例，同时也关联上了`BpBinder`对象。

如果从Java调用Binder服务，会使用`BinderProxy`类的`transact`方法，这最后也是调用了一个本地方法，我们看下实现：

```cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    //......
    //省略参数合法性判断
    //......
    
    // 获取 BpBinder 指针
    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    
    //......
    
    // 执行 BpBinder 的 transact
    status_t err = target->transact(code, *data, reply, flags);
    
    //......
}
复制代码
```

### 中期总结

学习到这里，我们了解了`如何使用Binder`以及`Binder的一些基本知识`：

- 数据通信通过`Parcel`来传递
- 对于客户端
  - 客户端可以通过`linkToDeath`来注册监听服务端的死亡通知
  - 不管是从`C++`还是`Java`的客户端调用，都会走到`BpBinder`这里
  - 客户端触发`Binder调用`的操作是`BpBinder`的`transact`
  - `BpBinder`的`transact`调用的是`IPCThreadState::self()->transact()`
- 对于服务端
  - 有`AIDL`支持的`Java层`，在服务端的业务实现上很简单
  - 支持`HIDL`的`C++`，在实现上也变得比较容易
  - 不管是`C++`还是`Java`，最后都是通过继承`BBinder`来实现
  - 在`onTransact`中实现不同方法调用的`case`

但是，我还有这么几个疑问：

- 客户端`transact`后是怎么通知到服务端的？
- 服务端`reply.write*`怎么把数据给到客户端的？
- `onTransact`在什么时候被调用的？

那我们就带着疑问，继续学习`Binder的实现原理`





# 3-原理

# Binder的实现原理

涉及到原理源码肯定是少不了的，9.0 `binder` 相关的源码分为三部分：

- Java：`frameworks/base/core/java/android/os/Binder.java`
- native：`frameworks/native/libs/binder/`
- driver：`common/drivers/android/binder.c`

还有一点需要明确的是：

- `用户进程`：针对`内核`空间或者`binder驱动`来说的，这里指的是向`binder驱动`发送消息的进程
- `客户进程`：针对`binder通信`的`服务进程`来说的，这里指的是发起`binder调用`的进程

书中这两个名词没有做详细说明，一开始看的有点晕

源码在手，干啥都有

## Binder设计相关的问题

`Binder`实现的远程调用是一种`面向对象`的远程调用，那么它和`面向过程`的远程调用区别在什么地方呢？

- ```
  面向过程
  ```

  的远程调用实现起来比较容易：

  - 只需要通过某种方式把需要执行的`函数号`和`参数`传递到`服务进程`
  - 然后`服务进程`根据`函数号`和`参数`执行对应的函数就完成了

- ```
  面向对象
  ```

  的调用则比较复杂：

  - 同一个服务类可以创建对个对象

    - 调用时不但要通过`函数号`和`参数`来识别要执行函数
    - 同时还要指定具体的对象

  - 对象是有生命周期的，需要监听管理

    - 服务中的`实体对象`死亡后，客户进程的`引用对象`也需要删除

    - 这个过程需要自动完成而不能由上层的客户程序协助完成

    - 因此为了管理方便，客户进程中需要（

      ```
      ProcessState
      ```

      类的作用）：

      - 集中管理本进程中所有的`Binder引用对象`
      - 并负责它们的创建和释放

设计复杂也带来了功能的强大，正因为`Binder`是`面向对象`的，我们可以创建多个`Binder实体对象`来服务不同的客户，每个对象有自己的数据。相互之间不会干扰。

为了系统中所有`引用对象`和`实体对象`能相互关联：

- `Binder`在驱动中建立了一张所有进程的`引用对象`和`实体对象`的`关联表`。

- 有了关联，

  ```
  Binder
  ```

  又必须保证用户进程中

  ```
  实体对象
  ```

  和

  ```
  引用对象
  ```

  跟驱动中的数据一致

  - 为了达到这个目标，`Binder`定义了自己的引用计数规则，而且这种规则是跨进程的。

参数的传递问题：

- 一般的对象作为参数传递没有太大问题，只需要`序列化`和`反序列化`就能实现。

- 但是

  ```
  Binder对象
  ```

  作为参数传递的时候，就会面临

  ```
  实体对象
  ```

  和

  ```
  引用对象
  ```

  相互转换的问题。

  - 为了让上层应用使用方便，这种转换在驱动中自动完成

- 普通的

  ```
  IPC
  ```

  传递参数数据时，要经历两次数据复制的过程：

  - 一次是从调用者的数据缓冲区复制到内核的缓冲区
  - 一次是从内核的缓冲区复制到接受进程的读缓冲区

- ```
  Binder
  ```

  为了提高效率：

  - 为每一个进程创建了一块缓存区，这块缓存区在内核和用户进程间共享
  - 传输数据到驱动，需要：
    - 从发送进程的用户空间缓存区复制到目标进程在驱动的缓存区
    - 目标进程从驱动中读取数据就不需要从内核空间复制到用户空间了，而是直接从内核共享的缓存区中读取
  - 这样减少了一次数据复制的过程

对于服务进程中`Binder调用`的执行：

- 每次执行必须在一个线程中完成

- 如果线程不停地创建和释放，会带来很大的系统开销。

- 使用

  ```
  线程池
  ```

  来管理

  ```
  Binder调用
  ```

  的执行

  - 在`Binder`的设计中，除了第一个线程是应用层主动创建的
  - 线程池中的其他线程都是在驱动的请求下才创建的
  - 这样将线程数量降到最低，并保证从驱动到来的`Binder调用`有线程可以使用。

## Binder的线程模型

### Binder 的线程池

在`Zygote进程`启动时，会调用`AppRuntime`的`onZygoteInit`函数（书中第8章，还没看到），代码如下：

```cpp
    virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }
复制代码
```

所有Android应用都是从`Zygote进程`中`fork`出来的。因此，这段代码对所有应用进程都有效。

- ```
  onZygoteInit
  ```

  函数首先调用

  ```
  self
  ```

  函数来得到

  ```
  ProcessState
  ```

  类的实例，每个进程都只会有一个实例。

  ```
  ProcessState
  ```

  构造函数如下：

  > ```
  > ProcessState::ProcessState(const char *driver)
  > : mDriverName(String8(driver))
  > , mDriverFD(open_driver(driver))
  > //......
  > {
  >   if (mDriverFD >= 0) {
  >      // mmap the binder, providing a chunk of virtual >   address space to receive transactions.
  >     mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
  >     if (mVMStart == MAP_FAILED) {
  >         // *sigh*
  >         ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
  >         close(mDriverFD);
  >         mDriverFD = -1;
  >         mDriverName.clear();
  >     }
  >   }
  > }
  > 复制代码
  > ```

- ```
  ProcessState
  ```

  类做了两件事：

  - 调用`open_driver()`函数打开`Binder设备`

  - 调用

    ```
    mmap()
    ```

    函数在驱动中分配了一块内存空间

    - 这块内存空间大小略小于1MB，定义如下：

    > ```cpp
    > #define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)
    > 复制代码
    > ```

    - 这块内存并不是给上层应用使用
    - 这块内存用于Binder驱动中接受传递给本进程的Binder数据
    - 这块内存在内核和本应用中共享

- 得到

  ```
  ProcessState
  ```

  类的实例后，调用它的

  ```
  startThreadPool()
  ```

  函数启动

  ```
  线程池
  ```

  ，看下代码：

  > ```cpp
  > void ProcessState::startThreadPool(){
  >    AutoMutex _l(mLock);
  >    if (!mThreadPoolStarted) {
  >        mThreadPoolStarted = true;
  >         spawnPooledThread(true);
  >     }
  > }
  > 复制代码
  > ```

  - 先判断`mThreadPoolStarted`是否为`true`，不为`true`才继续执行

  - 然后把`mThreadPoolStarted`设置为`true`，这说明`startThreadPool()`在进程中只会运行一次

  - 接着调用

    ```
    spawnPooledThread
    ```

    函数，参数为

    ```
    true
    ```

    > ```cpp
    > void ProcessState::spawnPooledThread(bool isMain)
    > {
    >   if (mThreadPoolStarted) {
    >       String8 name = makeBinderThreadName();
    >       ALOGV("Spawning new pooled thread, name=%s\n", name.string());
    >       sp<Thread> t = new PoolThread(isMain);
    >       t->run(name.string());
    >   }
    > }
    > 复制代码
    > ```

  - `spawnPooledThread`函数的作用是创建加入线程池的线程

  - 函数中创建了一个`PoolThread`类，类的`run`函数会创建线程

  - 传入的参数为`true`，说明这个线程是`线程池`的第一个线程

  - 以后再创建的线程都是接到驱动通知后创建的，传入的参数为

    ```
    false
    ```

    ，像这样

    > ```cpp
    > status_t IPCThreadState::executeCommand(int32_t cmd){
    >   //......
    >    case BR_SPAWN_LOOPER:
    >       mProcess->spawnPooledThread(false);
    >       break;
    >   //......
    > }
    > 复制代码
    > ```

  - 我们再看下

    ```
    PoolThread
    ```

    类的业务实现

    ```
    threadLoop
    ```

    函数

    > ```cpp
    > protected:
    > virtual bool threadLoop()
    > {
    >   IPCThreadState::self()->joinThreadPool(mIsMain);
    >   return false;
    > }
    > 复制代码
    > ```

    - 返回`false`代表执行一次，为什么是在`threadLoop()`中执行业务逻辑，可以看下[Android 中的`threadLoop`](https://blog.csdn.net/f2006116/article/details/89058397)
    - 具体调用细节大家阅读`frameworks`源码吧，路径应该是在：`frameworks/av/services/audioflinger/Threads.h`
    - `threadLoop`函数只是调用了`IPCThreadState`的`joinThreadPool`函数，这个函数后面单练它

好的，我们先来梳理下线程池这部分内容：

- 首先，应用启动时会执行`onZygoteInit`函数，这部分会`打开Binder设备`并`申请共享内存空间`
- 然后，执行`ProcessState`的`startThreadPool`创建`线程池`
- 然后，通过创建`PoolThread`的实例，创建`线程池`中的第一个线程
- 最后，`PoolThread`只是简单调用了`IPCThreadState`的`joinThreadPool`函数

关于`IPCThreadState`，稍后详谈

### 调用Binder服务的线程

客户端Binder服务的调用是通过IBinder的transact函数完成的。这里的IBInder实际上是BpBinder对象，代码如下：

```cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }
    return DEAD_OBJECT;
}
复制代码
```

这部分代码做了：

- 通过

  ```
  IPCThreadState
  ```

  的

  ```
  self
  ```

  静态函数获取

  ```
  IPCThreadState
  ```

  的指针

  - `IPCThreadState`对象会和每个线程关联
  - `self`函数会判断本线程是否有关联的`IPCThreadState`对象
  - 没有关联的对象则新创建一个`IPCThreadState`对象，并保存到当前线程中

- 得到`IPCThreadState`对象后，接着调用了`IPCThreadState`的`transact`，我们看下代码：

```cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err;
    flags |= TF_ACCEPT_FDS;
    //......
    //把要发送的数据放到类的成员变量mOut中
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }
    if ((flags & TF_ONE_WAY) == 0) {// 同步调用方式
        //......
        if (reply) {
            // 调用者要求返回结果，此时向底层发送数据并等待返回值
            err = waitForResponse(reply);
        } else {
            // 调用者不需要返回值，但是还要等待远程执行完毕
            // 这里用fakeRely来接收返回的Parcel对象
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
       //......
    } else {// 异步调用方式，函数会立即返回
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
复制代码
```

- `IPCThreadState`的`transact`首先执行`writeTransactionData`包要传输的数据保存到`mOut`变量中
- 然后通过`waitForResponse`把`mOut`中的数据通过`ioctl`发送给底层驱动

对于一个进程而言，只有一个`Binder驱动`的`文件描述符`，所有`ioctl`调用都是使用这个描述符。如果`客户端`有好几个线程同时执行远程调用，它们都将在同一个描述符的`ioctl`函数上等待。那么当数据到来时，哪个线程会受到回复呢？

- 通常的`IO模型`，在同一个描述符上等待的多个线程会被随机唤醒一个

- 但是Android在

  ```
  Binder驱动
  ```

  中记录了每次Binder调用的信息，其中就包括

  ```
  线程ID
  ```

  ，因此

  ```
  Binder驱动
  ```

  知道返回值应该交给哪个线程

  - 由驱动来处理线程的唤醒比在应用层做同样的事情要更简单，效率也更高
  - 但是，从架构的角度来看，这种设计比较糟糕，应用层和驱动产生了耦合。

好的，我们在对这部分做个小结：

- `客户端`从某个线程中发起调用，将参数打包后，通过`ioctl`函数传递给驱动
- `客户端`挂起并等待`ioctl`返回函数的结果
- `binder驱动`记录下调用线程的信息，然后根据调用的`binder对象`寻找`Binder服务`所在的进程，也就是`服务端`
- `binder驱动`找到`服务端`后先查看是否有空闲线程，没有则通知`服务端`创建
- `服务端`得到空闲线程后，根据`binder驱动`中保存的`BBinder对象`的指针调用相应的函数
- `服务端`在函数返回后在通过`ioctl`把结果打包传递给`binder驱动`
- `binder驱动`根据返回信息查找调用者线程
- `binder驱动`找到调用的线程后并唤醒它，并通过`ioctl`函数把结果传递回去
- `客户端`的线程得到返回结果后继续运行

## Binder对象的传递

当`Binder对象`作为参数传递是，会有两种情形：

- `Binder实体对象`作为参数传递：`非Binder对象`的传递是通过在接收端复制对象完成的，但是`Binder实体对象`是无法复制的，因此需要在客户进程中创建一个`Binder引用对象`来代替实体对象。

- ```
  Binder引用
  ```

  对象作为参数传递：

  - 如果传递的目的地是该引用对象对应实体对象所在的进程，那么：

    - `Binder框架`必须把这个`引用对象`转换成`Binder实体对象`。
    - 不能再创建一个新的`实体对象`
    - 必须找到并使用原来的`实体对象`

  - 如果传递的目的地是另一个

    ```
    客户端
    ```

    进程，那么：

    - 不能简单的复制`引用对象`
    - 需要建立目的进程中的`引用对象`和`实体对象`的关系
    - 这个关系的建立是在`Binder驱动`中完成的

### Binder对象传递流程简介

`Binder调用`的参数传递是通过`Parcel类`来完成的。先来简单看下`Binder实体对象`转换成`Binder引用对象`的过程：

- 在

  ```
  服务进程
  ```

  中将

  ```
  IBinder（BBinder）对象
  ```

  加入到

  ```
  Parcel对象
  ```

  后，

  ```
  Parcel对象
  ```

  会：

  - 打包数据，并把数据类型标记为`BINDER_TYPE_BINDER`
  - 把`BpBinder`的指针放进`cookie`字段
  - 通过`ioctl`把`Parcel对象`的数据传递到`Binder驱动`中

- ```
  Binder驱动
  ```

  会检查传递进来的数据，如果发现了标记为

  ```
  BINDER_TYPE_BINDER
  ```

  的数据：

  - 会先查找和服务进程相关的

    ```
    Binder实体对象表
    ```

    ：

    - 如果表中还没有这个实体对象的记录，则创建新的节点，并保存信息。

  - 然后驱动会查看客户进程的

    ```
    Binder对象引用表
    ```

    ：

    - 如果没有引用对象的记录，同样会创建新的节点
    - 并让这个节点中某个字段指向服务进程的`Binder实体对象表`中的节点

  - 接下来驱动对

    ```
    Parcel对象
    ```

    中的数据进行改动：

    - 把数据从`BINDER_TYPE_BINDER`改为`BINDER_TYPE_HANEL`
    - 同时把`handle`的值设为`Binder对象引用表`中的节点

  - 最后，把改动的数据传到`客户进程`

- ```
  客户端
  ```

  接收到数据，发现数据中的

  ```
  Binder类型
  ```

  为

  ```
  BINDER_TYPE_HANEL
  ```

  后

  - 使用

    ```
    handle
    ```

    值作为参数，调用

    ```
    ProcessState
    ```

    类中的函数

    ```
    getStrongProxyFoHandle
    ```

    来得到

    ```
    BpBinder对象
    ```

    - 如果对象不存在则创建一个新的对象
    - 这个`BpBinder对象`会一直保存在`ProcessState`的`mHandleToObject`表中

  - 这样，客户端就得到了`Binder引用对象`

### 写入Binder对象的过程

有了上面的整体流程，我们来看下`Binder对象`的写入细节：

- Parcel类中：

  - 写入

    ```
    Binder对象
    ```

    的函数是：

    - `writeStrongBinder`：写入强引用`Binder对象`
    - `writeWeakBinder`：写入弱引用`Binder对象`

  - 读取

    ```
    Binder对象
    ```

    的函数是：

    - ```
      readStrongBinder
      ```

      ：获取强引用

      ```
      Binder对象
      ```

      - 强引用的Binder对象可以分为`实体对象`和`引用对象`

    - ```
      readWeakBinder
      ```

      ：获取弱引用

      ```
      Binder对象
      ```

      - 弱引用则没有区分`实体对象`和`引用对象`

看下`writeStrongBinder`的代码：

```cpp
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
    // flatten_binder 整个方法其实是在向obj这个结构体存放数据
    flat_binder_object obj;

    if (IPCThreadState::self()->backgroundSchedulingDisabled()) {
        /* minimum priority for all nodes is nice 0 */
        obj.flags = FLAT_BINDER_FLAG_ACCEPTS_FDS;
    } else {
        /* minimum priority for all nodes is MAX_NICE(19) */
        obj.flags = 0x13 | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    }
    
    if (binder != NULL) {
        // 调用localBinder函数开区分是实体对象还是引用对象
        IBinder *local = binder->localBinder();
        if (!local) { //binder引用对象
            BpBinder *proxy = binder->remoteBinder();
            if (proxy == NULL) {
                ALOGE("null proxy");
            }
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.hdr.type = BINDER_TYPE_HANDLE;
            obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
            obj.handle = handle;
            obj.cookie = 0;
        } else { // binder实体对象
            obj.hdr.type = BINDER_TYPE_BINDER;
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
        obj.hdr.type = BINDER_TYPE_BINDER;
        obj.binder = 0;
        obj.cookie = 0;
    }

    return finish_flatten_binder(binder, obj, out);
}
复制代码
```

`flatten_binder`整个方法其实是在向`flat_binder_object`这个结构体存放数据。我们看下`flat_binder_object`的结构：

```cpp
struct flat_binder_object {
	/* 8 bytes for large_flat_header. */
	__u32		type;
	__u32		flags;
	/* 8 bytes of data. */
	union {
		binder_uintptr_t	binder;	/* local object */
		__u32			handle;	/* remote object */
	};
	/* extra data associated with local object */
	binder_uintptr_t	cookie;
};
复制代码
```

我们看下`flat_binder_object`中的属性：

- ```
  type
  ```

  的类型：

  - `BINDER_TYPE_BINDER`：用来表示Binder实体对象
  - `BINDER_TYPE_WEAK_BINDER`：用来表示Bindr实体对象的弱引用
  - `BINDER_TYPE_HANDLE`：用来表示Binder引用对象
  - `BINDER_TYPE_WEAK_HANDLE`：用来表示Binder引用对象的弱引用
  - `BINDER_TYPE_FD`：用来表示一个文件描述符

- `flag`字段用来保存向驱动传递的标志

- `union.binder`在打包`实体对象`时存放的是对象的`弱引用指针`

- `union.handle`在打包`引用对象`时存放的是对象中的`handle值`

- `cookie`字段只用在打包`实体对象`时，存放的是`BBinder指针`

### 解析`强引用`Binder对象数据的过程

Parcel 类中解析数据的函数是`unflatten_binder`，代码如下：

```cpp
status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);

    if (flat) {
        switch (flat->hdr.type) {
            case BINDER_TYPE_BINDER:
                *out = reinterpret_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                *out = proc->getStrongProxyForHandle(flat->handle);
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
复制代码
```

`unflatten_binder`的逻辑是：

- 如果是`BINDER_TYPE_BINDER`类型的数据，说明接收到的数据类型是`Binder实体对象`，此时`cookie字段`存放的是本进程的`Binder实体对象`的指针，可直接转化成`IBinder`的指针

- 如果是

  ```
  BINDER_TYPE_HANDLE
  ```

  类型的数据，则调用

  ```
  ProcessState类
  ```

  的

  ```
  getStrongProxyForHandle
  ```

  函数来得到

  ```
  BpBinder
  ```

  对象，函数代码如下：

  > ```cpp
  > sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
  > {
  >    sp<IBinder> result;
  >    AutoMutex _l(mLock);
  >    // 根据handle查看进程中是否已经创建了引用对象
  >    // 如果进程中不存在handle对应的引用对象，在表中插入新的元素并返回
  >    handle_entry* e = lookupHandleLocked(handle);
  >    if (e != NULL) {
  >        IBinder* b = e->binder; 
  >        //根据返回元素中的binder值来判断是否有引用对象
  >        if (b == NULL || !e->refs->attemptIncWeak(this)) {
  >            if (handle == 0) { // handle为0 表示是ServiceManager的引用对象
  >                Parcel data;
  >                //发送PING_TRANSACTION检查ServiceManager是否存在
  >                status_t status = IPCThreadState::self()->transact(
  >                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
  >                if (status == DEAD_OBJECT)
  >                   return NULL;
  >            }
  >            b = BpBinder::create(handle); // 创建一个新的引用对象
  >            e->binder = b;// 放入元素中的binder属性
  >            if (b) e->refs = b->getWeakRefs();
  >            result = b;
  >        } else {
  >            result.force_set(b);// 如果引用对象已经存在，放入到返回的对象result中
  >            e->refs->decWeak(this);
  >        }
  >    }
  >    return result;
  > }
  > 复制代码
  > ```

`getStrongProxyForHandle()`函数会调用`lookupHandleLocked()`来查找`handle`在进程中对应的`引用对象`。所有进程的`引用对象`都保存在`ProcessState`的`mHandleToObject`变量中。`mHandleToObject`变量定义如下：

```cpp
Vector<handle_entry> mHandleToObject;
复制代码
```

- ```
  mHandleToObject
  ```

  是一个

  ```
  Vector
  ```

  集合类，元素类型

  ```
  handle_entry
  ```

  - `handle_entry`结构很简单：

  > ```cpp
  > struct handle_entry {
  >   IBinder* binder;
  >   RefBase::weakref_type* refs;
  > };
  > 复制代码
  > ```

- `lookupHandleLocked()`函数就是使用`handle`作为`关键项`来查找对应的`handle_entry`，没有则创建新的`handle_entry`，并添加到集合中

- 当获得

  ```
  handle_entry
  ```

  后，如果

  ```
  handle
  ```

  值为0，表明要创建的是

  ```
  ServiceManager
  ```

  的

  ```
  引用对象
  ```

  - 并发送`PING_TRANSACTION`消息来检查`ServiceManager`是否已经创建

## IPCThreadState类

每个`Binder线程`都会有一个关联的`IPCThreadState类`的对象。`IPCThreadState类`主要的作用是和`Binder驱动`交互，发送接收`Binder数据`，处理和`Binder驱动`之间来往的消息。

我们在`Binder线程模型`中已经知道：

- `Binder服务`启动时，服务线程调用了`joinThreadPool()`函数
- 远程调用`Binder服务`时，客户线程调用了`waitForResponse()`函数

这两个函数都是定义在`IPCThreadState类`中，我们分别来看下这两个函数。

### `waitForResponse()`函数

函数定义如下：

```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;//和驱动通信
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;//没有数据，重新开始循环

        cmd = (uint32_t)mIn.readInt32();//读取数据
        //......
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;
        //......省略部分case语句
        case BR_REPLY:
        // Binder调用返回的消息
        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    //...... 错误处理
    return err;
}
复制代码
```

`waitForResponse()`函数中是一个无限`while循环`，在循环中，重复下面的工作：

- 调用`talkWithDriver`函数`发送/接收`数据

- 如果有消息从驱动返回，会通过

  ```
  switch语句
  ```

  处理消息。

  - 如果收到错误消息或者调用返回的消息，将通过`goto`语句跳出`while循环`
  - 如果还有未处理的消息，则交给`executeCommand`函数处理

我们再仔细看下`Binder调用`收到的返回类型为`BR_REPLY`的代码：

```cpp
        case BR_REPLY:
            {
                binder_transaction_data tr;
                //按照 binder_transaction_data 结构大小读取数据
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    //reply 不为null，表示调用者需要返回结果
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        //binder 调用成功，把从驱动来的数据设置到reply对象中
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        //binder调用失败，使用freeBuffer函数释放驱动中分配的缓冲区
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t), this);
                    }
                } else {
                    //调用者不需要返回结果，使用freeBuffer函数释放驱动中分配的缓冲区
                    freeBuffer(NULL,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t), this);
                    continue;
                }
            }
复制代码
```

针对`BR_REPLY`类型的处理流程是：

- 如果

  ```
  Binder调用
  ```

  成功返回，并且

  ```
  调用者
  ```

  也需要返回值

  - 把接收到的数据放在`Parcel`对象`reply`中返回

- 如果

  ```
  Binder调用
  ```

  不成功，或者

  ```
  调用者
  ```

  不需要返回数据

  - 通过`freeBuffer`释放驱动中分配的缓冲区

因为`Binder`提供了一块驱动和应用层`共享的内存空间`，所以在接收`Binder数据`时**不需要**`额外创建缓冲区`并`再进行一次拷贝`了，但是如果不及时通知驱动释放缓冲区中占用的无用内存，会很快会耗光这部分共享空间。

上面代码中的`reply->ipcSetDataReference`方法，在设置`Parcel对象`的同时，同样也把`freeBuffer`的指针作为参数传入到对象中，这样`reply对象`删除时，也会调用`freeBuffer`函数来释放驱动中的缓冲区。

`waitForResponse()`函数的作用是`发送Binder调用的数据并等待返回值`。为什么还需要使用循环的方式反复和驱动交互？原因有两点：

- 一是消息协议中要求应用层通过

  ```
  BC_TRANSACTION
  ```

  发送

  ```
  Binder调用数据
  ```

  后：

  - 驱动要先给应用层回复`BC_TRANSACTION_COMPLETE`消息，表示已经说到并且认可本次`Binder调用数据`。
  - 然后上层应用再次调用`talkWithDriver`来等待驱动返回调用结果
  - 如果调用结果返回了，会收到`BR_REPLY`消息

- 二是等待调用返回期间，驱动可能会给线程发送消息，利用这个线程帮忙干点活。。。。

### `joinThreadPool`函数

> 在`Binder线程池`部分已经知道：应用启动时会伴随着启动`Binder服务`，而最后执行到的方法就是`joinThreadPool`函数。

我们看下函数定义：

```cpp
void IPCThreadState::joinThreadPool(bool isMain)
{
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        processPendingDerefs();
        //now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();//读取并处理驱动发送的消息
        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            abort();
        }
        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    mOut.writeInt32(BC_EXIT_LOOPER); //退出前向驱动发送线程退出消息
    talkWithDriver(false);
}
复制代码
```

`joinThreadPool`函数的结构是一个while循环。

- 传入的参数

  ```
  isMain
  ```

  - 如果为

    ```
    true
    ```

    （通常是线程池第一个线程发起的调用），则向驱动发送

    ```
    BC_ENTER_LOOPER
    ```

    消息

    - 发送`BC_ENTER_LOOPER`的线程会被驱动标记为“主”线程
    - 不会在空闲时间被驱动要求退出

  - 否则，发送`BC_REGISTER_LOOPER`。

  - 这两条消息都是告诉驱动：本线程已经做好准备接收驱动来的Binder调用了

- 进入循环，调用了

  ```
  processPendingDerefs()
  ```

  函数

  - 用来处理

    ```
    IPCThreadState
    ```

    对象中

    ```
    mPendingWeakDerefs
    ```

    和

    ```
    mPendingStrongDerefs
    ```

    的

    ```
    Binder对象
    ```

    的引用计数

    - `mPendingWeakDerefs`和`mPendingStrongDerefs`都是`Vector`集合

  - 当接收到驱动发来的`BR_RELEASE`消息时，就会把其中的`Binder对象`放到`mPendingStrongDerefs`中

  - 并在`processPendingDerefs()`函数中介绍对象的引用计数

- 调用

  ```
  getAndExecuteCommand
  ```

  函数

  - 函数中调用`talkWithDriver`读取驱动传递的数据
  - 然后调用`executeCommand`来执行

到这里，我们再来看下`talkWithDriver`和`executeCommand`两个函数

### `talkWithDriver`函数

> `talkWithDriver`函数的作用是把`IPCThreadState`类中的`mOut变量`保存的数据通过`ioctl`函数发送到驱动，同时把驱动返回的数据放到类的`mIn变量`中。

`talkWithDriver`函数的代码如下：

```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }
    //ioctl 传输时所使用的的数据结构
    binder_write_read bwr;

    // Is the read buffer empty?
    // 判断 mIn 中的数据是否已经读取完毕，没有的话还需要继续读取
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    // 英文描述的很详细了哟
    // 如果不需要读取数据（doReceive=false，needRead=true），那么就可以准备写数据了
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    bwr.write_size = outAvail;//表示要写入的长度
    bwr.write_buffer = (uintptr_t)mOut.data();//要写入的数据的指针

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;//这个字段后面会被填上驱动读取到的数据长度（写入到驱动的数据长度）
    bwr.read_consumed = 0;// 这个字段后面会被填上从驱动返回的数据长度
    status_t err;
    do {
// 9.0增加了一些平台判断，可能以后要多平台去支持了吧
#if defined(__ANDROID__)
        // 用ioctl和驱动交换数据
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD <= 0) {
            // 这个情况应该是设备节点不可用
            err = -EBADF;
        }
    } while (err == -EINTR);

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                // 如果已经写入驱动的数据长度小于mOut中的数据长度
                // 说明还没发送完，把已经写入驱动的数据移除掉
                // 剩下的数据等待下次发送
                mOut.remove(0, bwr.write_consumed);
            else {
                // 数据已经全部写入驱动，复位mOut
                mOut.setDataSize(0);
                // 做一些指针的清理工作
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) {
            // 说明从驱动中读到了数据，设置好mInt对象
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        return NO_ERROR;
    }

    return err;
}
复制代码
```

- 准备发送到驱动中的数据保存在成员变量`mOut`中

- 从驱动中读取到的数据保存在成员变量`mInt`中

- 调用

  ```
  talkWithDriver
  ```

  时，如果

  ```
  mInt
  ```

  还有数据

  - 表示还没有处理完驱动发来的消息
  - 本次函数调用将不会从驱动中读取数据

- ```
  ioctl
  ```

  函数

  - 使用的命令是`BINDER_WRITE_READ`
  - 需要`binder_write_read`结构体作为参数
  - 驱动篇再看

### `executeCommand` 函数

> `executeCommand` 函数是一个大的switch语句，处理从驱动传递过来的消息。

我们前面遇到了一些消息，大概包括：

- `BR_SPAWN_LOOP`：驱动通知启动新线程的消息
- `BR_DEAD_BINDER`：驱动通知`Binder服务`死亡的消息
- `BR_FINISHED`：驱动通知线程退出的消息
- `BR_ERROR`,`BR_OK`,`BR_NOOP`：驱动简单的回复消息
- `BR_RELEASE`,`BR_INCREFS`,`BR_DECREFS`：驱动通知增加和减少`Binder对象`跨进程的引用计数
- `BR_TRANSACTION`，：驱动通知进行`Binder调用`的消息

重点是`BR_TRANSACTION`，代码定义如下：

```cpp
case BR_TRANSACTION:
        {
            binder_transaction_data tr;
            result = mIn.read(&tr, sizeof(tr));
            if (result != NO_ERROR) break; // 数据异常直接退出

            Parcel buffer;
            //用从驱动接收的数据设置Parcel对象Buffer
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);

            const pid_t origPid = mCallingPid;
            const uid_t origUid = mCallingUid;
            const int32_t origStrictModePolicy = mStrictModePolicy;
            const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;
            // 从消息中取出调用者的进程ID和euid
            mCallingPid = tr.sender_pid;
            mCallingUid = tr.sender_euid;
            mLastTransactionBinderFlags = tr.flags;

            Parcel reply;
            status_t error;

            if (tr.target.ptr) {
                // We only have a weak reference on the target object, so we must first try to
                // safely acquire a strong reference before doing anything else with it.
                if (reinterpret_cast<RefBase::weakref_type*>(
                        tr.target.ptr)->attemptIncStrong(this)) {
                    // 如果ptr指针不为空，cookie保存的是BBinder的指针
                    // 调用cookie的transact函数
                    error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
                            &reply, tr.flags);
                    // 及时清除指针
                    reinterpret_cast<BBinder*>(tr.cookie)->decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }

            } else {
                //如果tr.target.ptr为0 表示是ServiceManager
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }
            // 如果是同步调用，则把reply对象发送回去，否则什么也不做
            if ((tr.flags & TF_ONE_WAY) == 0) {
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                if (error < NO_ERROR) reply.setError(error);
                sendReply(reply, 0);
            } else {
                LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
            }

            mCallingPid = origPid;
            mCallingUid = origUid;
            mStrictModePolicy = origStrictModePolicy;
            mLastTransactionBinderFlags = origTransactionBinderFlags;
        }
        break;
复制代码
```

`BR_TRANSACTION`消息的处理过程是：

- 把消息解析出来后放置到`Parcel`类型的`buffer`对象中

- 使用消息参数里的

  ```
  BBinder指针
  ```

  来调用

  ```
  transact
  ```

  函数

  - `transact`函数会传入`函数号`、`buffer`、`reply`
  - `transact`函数根据`函数号`来调用对应的`服务函数`
  - `服务函数`执行完后，结果保存在`reply`对象中

- 如果是同步调用，使用`sendReply`函数返回`reply`对象

- 如果是异步调用，则什么也不做，结束调用

# 4-驱动

# Binder驱动

> Binder 驱动是整个Binder框架的核心，这部分就会详细介绍`消息协议`、`内存共享机制`、`对象传递`的具体细节了

## 应用层和驱动的消息协议

`Binder应用层`的`IPCThreadState`和`Binder驱动`之间通过`ioctl`来传递数据，因此，定义了一些`ioctl命令`：

| 命令                       | 说明                                                         | 数据格式                   |
| -------------------------- | ------------------------------------------------------------ | -------------------------- |
| `BIDNER_WRITE_READ`        | 向驱动读取和写入数据，既可以单独读写，也可以同时读写。通过传入的数据中有无读写数据来控制 | `struct binder_write_read` |
| `BINDER_SET_IDLE_TIMEOUT`  | 有定义未使用                                                 | `int64_t`                  |
| `BINDER_SET_MAX_THREADS`   | 设置线程池的最大线程数，达到上限后驱动将不会通知应用层启动新线程 | `size_t`                   |
| `BINDER_SET_IDLE_PRIORITY` | 有定义未使用                                                 | `int`                      |
| `BINDER_SET_CONTEXT_MGR`   | 将本进程设置为`Binder系统`的`管理进程`，只有`ServiceManager`进程才会使用这个命令 | `int`                      |
| `BINDER_THREAD_EXIT`       | 通知驱动当前线程要退出了，以便驱动清理和该线程相关的数据     | `int`                      |
| `BINDER_VERSION`           | 获取`Binder`的版本号                                         | `struct binder_verison`    |

表中的命令中。最常用的命令是`BIDNER_WRITE_READ`，其余都是一些辅助性命令。`BIDNER_WRITE_READ`使用的数据格式定义如下：

```cpp
struct binder_write_read {
	binder_size_t		write_size;	//计划向驱动写入的字节长度
	binder_size_t		write_consumed;	//实际写入驱动的长度，书中称驱动实际读取的字节长度
	binder_uintptr_t	write_buffer;//传递给驱动数据的buffer指针，也就是真正要写入的数据
	binder_size_t		read_size;	//计划从驱动中读取的字节长度
	binder_size_t		read_consumed;	//实际从驱动中读取到的字节长度
	binder_uintptr_t	read_buffer; //接收存放读取到的数据的指针
};
复制代码
```

存放在`read_buffer`和`write_buffer`中的数据也是有格式的，格式是消息`ID`加上`binder_transaction_data`，`binder_transaction_data`定义如下：

```cpp
struct binder_transaction_data {
	union {
		__u32	handle;
		binder_uintptr_t ptr;
	} target;               // BpBinder对象使用handle，BBinder使用ptr
	binder_uintptr_t	cookie;	// 对于BBinder对象，这里是BBinder的指针
	__u32		code;		// Binder服务的函数号码
	__u32	        flags;
	pid_t		sender_pid; // 发送方的进程ID
	uid_t		sender_euid; // 发送方的euid
	binder_size_t	data_size; // 整个数据区的大小
	binder_size_t	offsets_size; // IPC对象区域的大小	
	union {
		struct {
			binder_uintptr_t	buffer; // 指向数据区开头的指针
			binder_uintptr_t	offsets;// 指向数据区中IPC对对象区的指针
		} ptr;
		__u8	buf[8];
	} data;
};
复制代码
```

结构`binder_transaction_data`中：

- ```
  data.ptr
  ```

  指向

  ```
  传递给驱动的数据区
  ```

  的起始地址

  - `传递给驱动的数据区`就是从应用层传递下来的`Parcel对象`的数据区

- 在

  ```
  Parcel对象
  ```

  中如果打包了

  ```
  IPC对象
  ```

  （也就是

  ```
  Binder对象
  ```

  ），或者

  ```
  文件描述符
  ```

  ，数据区中会有一段空间专门用来保存这部分数据。

  - 因此使用`data.ptr.offsets`表示数据区`IPC对象`的起始位置
  - 用`offsets_size`表示数据区`IPC对象`的大小

`Binder消息`则根据`发送`和`接收`分为两套，从命名上也比较容易区分：

- `发送类型`的指的是消息从`应用层`到`驱动`的过程，通常以`BC_`开头
- `接收类型`的指的是消息从`驱动`到`应用层`的过程，通常以`BR_`开头

### 发送给驱动的Binder命令列表

| 命令                            | 说明                                       |
| ------------------------------- | ------------------------------------------ |
| `BC_TRANSACTION`                | 发送`Binder调用`的数据                     |
| `BC_REPLY`                      | 返回`Binder调用`的返回值                   |
| `BC_ACQUIRE_RESULT`             | 回应`BR_ATTEMPT_ACQUIRE`命令               |
| `BC_FREE_BUFFER`                | 释放通过`mmap`分配的内存块                 |
| `BC_INCREFS`                    | 增加`Binder对象`的弱引用计数               |
| `BC_ACQUIRE`                    | 增加`Binder对象`的强引用计数               |
| `BC_DECREFS`                    | 减少`Binder对象`的弱引用计数               |
| `BC_RELEASE`                    | 减少`Binder对象`的强引用计数               |
| `BC_INCREFS_DONE`               | 回应`BC_INCREFS`指令                       |
| `BC_ACQUIRE_DONE`               | 回应`BC_ACQUIRE`指令                       |
| `BC_ATTEMPT_ACQUIRE`            | 将Binder对象的弱引用升级为强引用           |
| `BC_REGISTER_LOOPER`            | 把当前线程注册为线程池的`主线程`           |
| `BC_ENTER_LOOPER`               | 通知驱动，线程可以进入数据的发送和接收     |
| `BC_EXIT_LOOPER`                | 通知驱动，当前线程退出数据的发送和接收     |
| `BC_REQUEST_DEATH_NOTYFICATION` | 通知驱动接收某个`Binder服务`的死亡通知     |
| `BC_CLEAR_DEATH_NOTYFICATION`   | 通知驱动不再接收某个`Binder服务`的死亡通知 |
| `BC_DEAD_BINDER_DONE`           | 回应`BR_DEAD_BINDER`命令                   |

### 驱动返回的命令列表

| 命令                               | 说明                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `BR_ERROR`                         | 驱动内部出错了                                               |
| `BR_OK`                            | 命令成功                                                     |
| `BR_TRANSACTION`                   | Binder调用命令                                               |
| `BR_REPLY`                         | 返回Binder调用的结果                                         |
| `BR_ACQUIRE_RESULT`                | 未使用                                                       |
| `BR_DEAD_REPLY`                    | 向驱动发送`Binder调用`时，如果对方已经死亡，则驱动回应此命令 |
| `BR_TRANSACTION_COMPLETE`          | 回应`BR_TRANSACTION`命令                                     |
| `BR_INCREFS`                       | 要求增加`Binder对象`的弱引用计数                             |
| `BR_ACQUIRE`                       | 要求增加`Binder对象`的强引用计数                             |
| `BR_DECREFS`                       | 要求减少`Binder对象`的弱引用计数                             |
| `BR_RELEASE`                       | 要求减少`Binder对象`的强引用计数                             |
| `BR_ATTEMPT_ACQUIRE`               | 要求将`Binder对象`的弱引用变成强引用                         |
| `BR_NOOP`                          | 命令成功                                                     |
| `BR_SPAWN_LOOPER`                  | 通知创建`Binder线程`                                         |
| `BR_FINISHED`                      | 未使用                                                       |
| `BR_DEAD_BINDER`                   | 通知关注的`Binder`已经死亡                                   |
| `BR_CLEAR_DEATH_NOTYFICATION_DONE` | 回应`BC_CLEAR_DEATH_NOTYFICATION`                            |
| `BR_FAILED_REPLY`                  | 如果`Binder调用`的`函数号`不正确，回复本消息                 |

### Binder消息序列图

前面列出的两个命令列表看上去指令蛮多的，其实可以分为四类：

- `Binder线程`管理相关
- `Binder方法调用`相关
- `Binder对象引用计数管理`相关
- `Binder对象死亡通知`相关

看下面两个序列图:

`Binder调用`的消息序列图： ![image](images/binder%E8%B0%83%E7%94%A8%E6%B6%88%E6%81%AF%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

`Binder对象引用计数管理`的消息序列图： ![image](images/binder%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E6%B6%88%E6%81%AF%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

## `Binder驱动`分析

`Binder驱动`中定义的操作函数如下：

```cpp
static const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = binder_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};
复制代码
```

`Binder驱动`中并没有实现常用的`read`、`write`操作。数据的传输都是通过`ioctl`操作来完成的。

`Binder驱动`中定义了很多的数据结构，一些主要的数据结构如下：

| 数据结构        | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| `binder_proc`   | 每个使用`open`打开`Binder设备文件`的进程都会在驱动中创建一个`biner_proc的`结构，用来记录该进程的各种信息和状态。例如：`Binder节点表`、`节点引用表`等 |
| `binder_thread` | 每个`Binder线程`在`Binder驱动`中都有一个`binder_thread`结构，记录线程相关的信息，例如要完成的任务等 |
| `binder_node`   | `binder_proc`中有一张`Binder节点对象表`，表项是`binder_node`结构。代表进程中的BBinder对象，记录BBinder对象的指针、引用计数等数据 |
| `binder_ref`    | `binder_proc`中还有一张`节点引用表`，表项是`binder_ref`结构。代表进程中的`BpBinder对象`，保存所引用的对象`binder_node`指针。`BpBinder`中的`mHandler`值，就是它在索引表中的位置 |
| `binder_buffer` | 驱动通过`mmap`的方式创建了一块大的缓存区，每次`Binder`传输数据，会在缓存区分配一个`binder_buffer`的结构来保存数据 |

在Binder驱动中，大量使用`红黑树`来管理数据。`Binder驱动`中是由内核提供的实现，具体表现是生命的一个全局变量`binder_procs`：

```cpp
static HLIST_HEAD(binder_procs);
复制代码
```

所有进程的`binder_proc`结构体都将插入到这个变量代表的`红黑树`中。以`binder_open`的代码为例：

```cpp
static int binder_open(struct inode *nodp, struct file *filp)
{
	struct binder_proc *proc;
	struct binder_device *binder_dev;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE, "%s: %d:%d\n", __func__,
		     current->group_leader->pid, current->pid);

	proc = kzalloc(sizeof(*proc), GFP_KERNEL);//分配空间给binder_proc结构体
	//......
	get_task_struct(current->group_leader);// 获取当前进程的task结构
	proc->tsk = current->group_leader;
	//......
	INIT_LIST_HEAD(&proc->todo);// 初始化binder_porc中的todo队列
	//......
	filp->private_data = proc;// 将proc保存到文件结构中，供下次调用使用
	mutex_lock(&binder_procs_lock);
	// 将proc插入到全局变量binder_procs中
	hlist_add_head(&proc->proc_node, &binder_procs);
	mutex_unlock(&binder_procs_lock);
	//......
	return 0;
}
复制代码
```

`binder_open`函数主要功能是：

- 打开`Binder驱动`的设备文件

- 为当前进程创建和初始化`binder_proc`结构体`proc`

- 将`proc`插入到红黑树`binder_procs`中

- 将

  ```
  proc
  ```

  放入到

  ```
  file
  ```

  结构的

  ```
  private_data
  ```

  字段中

  - 调用驱动的其他操作时可以从`file`结构中取出代表当前进程的`binder_proc`

## Binder的内存共享机制

前面介绍了，用户进程打开`Binder设备`后，会调用`mmap`在驱动中创建一块内存空间用于接收传递给本进程的`Binder数据`。我们看下`mmap`的代码(这部分是9.0的源码，和书中的结构有些不同)：

```cpp
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	const char *failure_string;
	if (proc->tsk != current->group_leader)
		return -EINVAL;
	// 检查要求分配的空间是否大于SZ_4M的定义
	if ((vma->vm_end - vma->vm_start) > SZ_4M)
		vma->vm_end = vma->vm_start + SZ_4M;
    // 检查mmap的标记，不能带有FORBIDDEN_MMAP_FLAGS
	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
		ret = -EPERM;
		failure_string = "bad vm_flags";
		goto err_bad_arg;
	}
	//fork 的子进程无法复制映射空间，并且不允许修改属性
	vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
	vma->vm_flags &= ~VM_MAYWRITE;

	vma->vm_ops = &binder_vm_ops;
	vma->vm_private_data = proc;
	// 真正内存分配的逻辑在 binder_alloc_mmap_handler中
	ret = binder_alloc_mmap_handler(&proc->alloc, vma);
	//......
}
int binder_alloc_mmap_handler(struct binder_alloc *alloc,
			      struct vm_area_struct *vma)
{
	int ret;
	struct vm_struct *area;
	const char *failure_string;
	struct binder_buffer *buffer;

	mutex_lock(&binder_alloc_mmap_lock);
	//如果已分配缓存区，goto 错误
	if (alloc->buffer) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}
	//在用户进程分配一块内存作为缓冲区
	area = get_vm_area(vma->vm_end - vma->vm_start, VM_ALLOC);
    //如果内存分配失败，goto 对应错误
	if (area == NULL) {
		ret = -ENOMEM;
		failure_string = "get_vm_area";
		goto err_get_vm_area_failed;
	}
	//把分配的缓冲区指针放在binder_proc的buffer字段
	alloc->buffer = area->addr;
	//重新配置下内存起始地址和偏移量
	alloc->user_buffer_offset =
		vma->vm_start - (uintptr_t)alloc->buffer;
	mutex_unlock(&binder_alloc_mmap_lock);
	//创建物理页结构体
	alloc->pages = kzalloc(sizeof(alloc->pages[0]) *
				   ((vma->vm_end - vma->vm_start) / PAGE_SIZE),
			       GFP_KERNEL);
			       
	alloc->buffer_size = vma->vm_end - vma->vm_start;
	//在内核分配struct buffer的内存空间
	buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);

	//这部分和书中着实不太一样，看的晕晕的
	//目前只看出内存地址关联部分
	buffer->data = alloc->buffer;
	list_add(&buffer->entry, &alloc->buffers);
	buffer->free = 1;
	binder_insert_free_buffer(alloc, buffer);
	// 异步传输将可用空间大小设置为映射大小的一半
	alloc->free_async_space = alloc->buffer_size / 2;
	barrier();
	alloc->vma = vma;
	alloc->vma_vm_mm = vma->vm_mm;
	/* Same as mmgrab() in later kernel versions */
	atomic_inc(&alloc->vma_vm_mm->mm_count);

	return 0;
// ......对应的错误信息
}
复制代码
```

内存分配时：

- 首先调用`get_vm_area`在`用户进程`分配一块地址空间
- 接着在`内核`中也分配同样页数大小的空间
- 然后把它们的`物理内存`地址绑定在一起
- 这样`应用`和`内核`间就能共享一块空间了

当发生Binder调用时：

- 数据会从`调用进程`复制到`内核空间`中

- 驱动会在

  ```
  服务进程
  ```

  的

  ```
  缓冲区
  ```

  中寻找一块合适大小的空间来存放数据

  - 因为`服务进程`的`用户空间`的`缓冲区`和`内核空间`的`缓冲区`是共享的
  - 所以`服务进程`不需要将数据再从`内核空间`复制到`用户空间`
  - 节省了一次复制过程
  - 提高了`Binder通信`效率

## 驱动的`ioctl`操作

`ioctl`在驱动中的实现是`binder_ioctl`函数，用来处理`ioctl`操作的命令，这些命令最常见的就是`BINDER_WRITE_READ`，用于`Binder调用`，先看下这个命令的处理过程：

```cpp
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	
	// 检查binder_stop_on_user_error的值是否小于2，这个值表示Binder中的错误数
	// 大于2则挂起当前进程到binder_user_error_wait的等待队列上
	ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret)
		goto err_unlocked;
		
	// 获取当前线程的数据结构
	thread = binder_get_thread(proc);
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	switch (cmd) {
	case BINDER_WRITE_READ:
	    //调用 binder_ioctl_write_read 方法
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		break;
}
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
	// 这里应该就是就是一次拷贝，把数据复制到内核中
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
	if (bwr.write_size > 0) {
	    // 如果命令时传递给驱动的数据，调用 binder_thread_write 处理
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		//省略异常处理和log打印
	}
	if (bwr.read_size > 0) {
	    // 如果命令时读取驱动的数据，调用 binder_thread_read 处理
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		binder_inner_proc_lock(proc);
		// 如果进程的todo队列不为空，唤醒用户进程处理
		if (!binder_worklist_empty_ilocked(&proc->todo))
			binder_wakeup_proc_ilocked(proc);
		binder_inner_proc_unlock(proc);
		//省略异常处理和log打印
	}
// 异常处理
}
复制代码
```

`BINDER_WRITE_READ`命令有可能是向驱动写数据，也可能是从驱动读数据

- 写数据，通过`binder_thread_write`方法
- 读数据，通过`binder_thread_read`方法
- 在下一节`调用过程`会细讲这两个方法

除了`BINDER_WRITE_READ`命令外，还有像`BINDER_SET_MAX_THREADS`、`BINDER_SET_CONTEXT_MGR`等辅助通信的命令，大家可以阅读下源码。

学习的重点还是看`调用过程`的数据`读写部分`

## `Binder调用`过程

`Binder调用`是`Binder驱动`中最核心的活动，整个过程几乎涉及了`Binder驱动`所有的数据结构。我们先从调用的整体流程上来看下：

- ```
  Binder调用
  ```

  从

  ```
  客户进程
  ```

  中发起

  - 通过`ioctl`操作来运行`驱动`的`binder_ioctl`

- ```
  驱动
  ```

  接收到命令

  ```
  BC_TRANSACTION
  ```

  后

  - 找到代表`服务进程`的`binder_proc`结构体
  - 在`服务进程`的缓冲区分配一块内存来保存从`客户进程`传递的数据
  - `ioctl`回复消息`BR_TRANSACTION_COMPLETED`

- ---------到这里，本次`ioctl`调用结束---------

- ```
  客户进程
  ```

  收到回复消息后

  - 再次调用`ioctl`进入等待，准备读取数据，直到调用结果返回

`服务进程`无法知道何时会有`Binder调用`到达，因此它至少由一个线程在`ioctl`上等待。

- `驱动`会在有调用到达时唤醒等待的线程，并将数据传给`服务进程`

- 同一时刻可能有多个

  ```
  Binder调用
  ```

  到达

  - `驱动`为每个调用创建一个`binder_transaction`的结构体

  - 将

    ```
    结构体
    ```

    中的

    ```
    work字段
    ```

    插入到

    ```
    服务进程
    ```

    的

    ```
    todo队列
    ```

    中

    - `todo队列`是一个`binder_work`结构的列表

    - 每个进程都会有一个`todo队列`来接收需要完成的工作

    - 当

      ```
      服务端
      ```

      的

      ```
      用户进程
      ```

      对

      ```
      ioctl
      ```

      执行读操作时

      - 会循环执行`todo队列`中需要完成的任务
      - 直到队列为空才挂起等待

### 客户进程的调用流程

上面刚刚讲到，`客户进程`发送的`Binder调用`消息会执行到函数`binder_thread_write`，这个函数的功能就是处理所有`用户进程`发送的消息，我们看下和`Binder调用`有关的部分：

```cpp
static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
//......
        case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;
			// 包结构体tr的数据复制到内核
			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			// 调用 binder_transaction 函数处理
			binder_transaction(proc, thread, &tr,
					   cmd == BC_REPLY, 0);
			break;
		}
//......
}
复制代码
```

对`BC_TRANSACTION`命令的处理就是调用了`binder_transaction`函数，`binder_transaction`函数既要处理`BC_TRANSACTION`也会处理`BC_REPLY`消息，我们先看读取的情况，也就是`reply=false`的情况：

1. 找到代表目标进程的节点：

```cpp
        if (tr->target.handle) {
			struct binder_ref *ref;
			//查找handle对应的引用项
			ref = binder_get_ref_olocked(proc, tr->target.handle,
						     true);
			if (ref) {
			    //引用项的node字段保存了Binder对象节点的指针
			    //复制给target_node
				target_node = binder_get_node_refs_for_txn(
						ref->node, &target_proc,
						&return_error);
			} else {
                //......
			}
		//......
		} else {
		    //把管理进程的节点名放到target_node中
			target_node = context->binder_context_mgr_node;
			//......
		}
复制代码
```

1. 搜寻目标线程

```cpp
		if (!(tr->flags & TF_ONE_WAY) && thread->transaction_stack) {
			struct binder_transaction *tmp;
			tmp = thread->transaction_stack;
			//......
			while (tmp) {
				struct binder_thread *from;
				spin_lock(&tmp->lock);
				from = tmp->from;
				if (from && from->proc == target_proc) {
					atomic_inc(&from->tmp_ref);
					target_thread = from;
					spin_unlock(&tmp->lock);
					break;
				}
				spin_unlock(&tmp->lock);
				tmp = tmp->from_parent;
			}
		}
复制代码
```

上面这段代码的逻辑是：

- 如果本次调用不是异步调用，并且调用者线程中的`transaction_stack`不为`NULL`，则在其中查找和本次调用具有相同目标进程的`transaction_stack`，如果找到，则把它的目标线程设置为本次调用的目标线程。
- `transaction_stack`是一个列表，保存了本线程所有正在执行的Binder调用的`binder_transaction`结构体。
- `Binder驱动`中用`binder_transaction`来保存一次`Binder调用`的所有数据，包括传递的数据、通信双发的进程、线程信息等。因为Binder调用涉及两个进程，还要向调用端传递返回值。
- 所以，驱动中用结构`binder_transaction`保存还没结束的`Binder调用`。
- 通常情况下执行`Binder调用`时不会存在相同进程的`Binder调用`，因此`target_thread`的值大多为`NULL`

1. 如果有目标线程，则使用`目标线程`中的`todo队列`，否则使用`目标进程`的`todo队列`

```cpp
	if (target_thread)
		e->to_thread = target_thread->pid;
	e->to_proc = target_proc->pid;
复制代码
```

1. 为当前的`Binder调用`创建`binder_transaction`结构，并用调用的数据填充它。

```cpp
	t = kzalloc(sizeof(*t), GFP_KERNEL);
	//......
	tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);
	//......
	if (!reply && !(tr->flags & TF_ONE_WAY))
		t->from = thread;
	else
		t->from = NULL;
	t->sender_euid = task_euid(proc->tsk);
	t->to_proc = target_proc;
	t->to_thread = target_thread;
	t->code = tr->code;
	t->flags = tr->flags;
	if (!(t->flags & TF_ONE_WAY) &&
	    binder_supported_policy(current->policy)) {
		/* Inherit supported policies for synchronous transactions */
		t->priority.sched_policy = current->policy;
		t->priority.prio = current->normal_prio;
	} else {
		/* Otherwise, fall back to the default priority */
		t->priority = target_proc->default_priority;
	}
复制代码
```

1. 在`目标进程`(也就是`服务端`)的缓冲区分配空间，复制`用户进程`的数据到`内核`

```cpp
    t->buffer = binder_alloc_new_buf(&target_proc->alloc, tr->data_size,
		tr->offsets_size, extra_buffers_size,
		!reply && (t->flags & TF_ONE_WAY));
	//......
	if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
			   tr->data.ptr.buffer, tr->data_size)) {
		//......
	}
	if (copy_from_user(offp, (const void __user *)(uintptr_t)
			   tr->data.ptr.offsets, tr->offsets_size)) {
		//......
	}
复制代码
```

- 这里复制的数据是`Binder调用`的参数数据，也是需要传递给`服务进程`的数据，因此需要缓冲区。这个缓冲区是在`目标进程`的大的缓冲区中分配的

1. 处理传输中的`Binder对象`

```cpp
off_end = (void *)off_start + tr->offsets_size;
	sg_bufp = (u8 *)(PTR_ALIGN(off_end, sizeof(void *)));
	sg_buf_end = sg_bufp + extra_buffers_size;
	off_min = 0;
	for (; offp < off_end; offp++) {
		//......
	}
复制代码
```

- 通过参数传递的`Binder对象`需要进行`转换`，这里的`for循环`就是进行转换操作，内容较多，会在下面`处理传递的Binder对象`章节进行讲解

1. 将本次调用的`binder_transaction`结构体链接到线程的`binder_stack`列表中

```cpp
if (!(t->flags & TF_ONE_WAY)) {
		//......
		t->need_reply = 1;
		t->from_parent = thread->transaction_stack;
		thread->transaction_stack = t;
		// 书中下面注释部分的代码已经整合到 binder_proc_transaction 函数中了
		// 首先，将结构 binder_transaction中的binder_work放到目标目标进程或线程的todo队列
		// 然后，创建一个新的binder_work的结构体，并将它放到发送线程的todo队列
		if (!binder_proc_transaction(t, target_proc, target_thread)) {
		    //...... 失败处理
		}
	}
复制代码
```

- `binder_transaction`结构中包含了一个`binder_work`的结构体，因此它可以被放到`todo队列`中
- 本地调用时`binder_work`的`type`被设置为`BINDER_WORK_TRANSACTION`后，插入了`目标进程`或`目标线程`的`todo队列`中
- 为了使`客户进程`能收到回复消息，这里也会创建一个新的`binder_work`的结构体，并把它的`type`设置成了`BINDER_WORK_TRANSAXTION_COMPLETET`，并将它插入到当前线程的`todo队列`中

前面介绍过，在`binder_thread_write`函数执行完后，还会去判断是否需要执行`binder_thread_read`函数：

- 由于调用端执行完`BC_TRANSACTION`后，会立刻执行`ioctl`的`读`指令
- 所以，在调用操作上，`binder_thread_read`函数算是紧接着`binder_thread_write`函数后执行的

刚才新建的`binder_work`的结构体已经插入`todo队列`了，我们看下`binder_thread_read`函数会进行哪些和`todo队列`相关的操作：

```cpp
while (1) {
        //......
        // 获得todo队列，获取失败则goto retry
		if (!binder_worklist_empty_ilocked(&thread->todo))
			list = &thread->todo;
		else if (!binder_worklist_empty_ilocked(&proc->todo) &&
			   wait_for_proc_work)
			list = &proc->todo;
		else {
			binder_inner_proc_unlock(proc);
			/* no data added */
			if (ptr - buffer == 4 && !thread->looper_need_return)
				goto retry;
			break;
		}
		//取出todo队列中的元素
		w = binder_dequeue_work_head_ilocked(list);
		//......
		switch (w->type) {
		    //......省略大量case语句
		    case BINDER_WORK_TRANSACTION_COMPLETE: {
			    binder_inner_proc_unlock(proc);
			    cmd = BR_TRANSACTION_COMPLETE;
			    //把返回消息通过put_user放到用户空间的指针中
			    if (put_user(cmd, (uint32_t __user *)ptr)) 
				    return -EFAULT;
			    ptr += sizeof(uint32_t);

			    binder_stat_br(proc, thread, cmd);

			    kfree(w);
			    binder_stats_deleted(BINDER_STAT_TRANSACTION_COMPLETE);
		    } break;
		    //......省略大量case语句
	    }
		//......
}
复制代码
```

- `binder_thread_read`函数所做的处理就是把回复消息`BR_TRANSACTION_COMPLETE`复制到用户空间
- 这样`客户进程`就可以收到回复消息了

到这里，`客户进程`的调用结束了。但是，`Binder调用`才完成一半，接下来，看看`服务进程`是如何调用数据的。

### `服务进程`的调用流程

`服务进程`至少有一个线程会在ioctl上等待调用的到来。`服务进程`调用ioctl时传递的是读数据的请求，所以最后调用的也是`binder_thread_read`函数，我们看下`binder_thread_read`函数完整的处理流程：

1. 如果保存返回结果的缓冲区中还没有数据，先写入`BR_NOOP`消息：

```cpp
    if (*consumed == 0) {
		if (put_user(BR_NOOP, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
	}
复制代码
```

1. 进入循环处理所有`todo队列`中的工作

```cpp
while (1) {
    //......
}
复制代码
```

1. 读取线程或进程`todo队列`中需要完成的`工作`

```cpp
        struct binder_work *w = NULL;
        //......
		if (!binder_worklist_empty_ilocked(&thread->todo))
			list = &thread->todo;
		else if (!binder_worklist_empty_ilocked(&proc->todo) &&
			   wait_for_proc_work)
			list = &proc->todo;
		//......
		w = binder_dequeue_work_head_ilocked(list);
复制代码
```

1. 用`switch语句`处理所有类型的`工作`

```cpp
    switch (w->type) {
		case BINDER_WORK_TRANSACTION: {
			binder_inner_proc_unlock(proc);
			t = container_of(w, struct binder_transaction, work);
		} break;
    }
复制代码
```

- 经过`客户进程的调用流程`后，此时的`服务进程`中已经存在一个类型为`BINDER_WORK_TRANSACTION`的工作需要处理
- 这里只是取出了和`binder_work`关联的`binder_transaction`结构体指针
- 并保存到变量`t`中

1. 调整线程的优先级

```cpp
        if (!t)
			continue;

		if (t->buffer->target_node) {
			struct binder_node *target_node = t->buffer->target_node;
			struct binder_priority node_prio;

			tr.target.ptr = target_node->ptr;
			tr.cookie =  target_node->cookie;
			node_prio.sched_policy = target_node->sched_policy;
			node_prio.prio = target_node->min_priority;
			binder_transaction_priority(current, t, node_prio,
						    target_node->inherit_rt);
			cmd = BR_TRANSACTION;
		}
复制代码
```

- 如果t为NULL，继续循环
- 否则，开始准备返回的消息`BR_TRANSACTION`
- 同时，设置线程的优先级
  - 如果`调用线程`的优先级低于`当前线程`指定的`最低优先级`，则把`当前线程`的优先级设为`调用线程`的优先级
  - 否则，把`当前线程`设为指定的`最低优先级`
  - 这意味着`Binder线程`会以尽量低的优先级运行

1. 准备返回的数据

```cpp
        tr.code = t->code;
		tr.flags = t->flags;
		tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);
		//......
		// 让tr中的data指针指向内核中保存的数据缓冲区
		tr.data_size = t->buffer->data_size;
		tr.offsets_size = t->buffer->offsets_size;
		tr.data.ptr.buffer = (binder_uintptr_t)
			((uintptr_t)t->buffer->data +
			binder_alloc_get_user_buffer_offset(&proc->alloc));
		tr.data.ptr.offsets = tr.data.ptr.buffer +
					ALIGN(t->buffer->data_size,
					    sizeof(void *));
		// 把 BR_TRANSACTION 消息复制到用户空间
		if (put_user(cmd, (uint32_t __user *)ptr)) {
			//......
			return -EFAULT;
		}
		ptr += sizeof(uint32_t);
		if (copy_to_user(ptr, &tr, sizeof(tr))) {
		    //把结构体tr数据复制到用户空间
			//......
			return -EFAULT;
		}
		ptr += sizeof(tr);

		//......
		break;//跳出while循环
复制代码
```

这一段代码都是为消息`BR_TRANSACTION`准备返回数据，要注意的是：

- 调用`copy_to_user`复制到`用户空间`的只是`结构体tr`的数据
- `服务进程`得到这个结构体之后，会直接读取它里面的`data指针`的数据
- 数据准备完毕后，使用`break语句`跳出`while循环`

1. 启动新线程

```cpp
    if (proc->requested_threads == 0 &&
	    list_empty(&thread->proc->waiting_threads) &&
	    proc->requested_threads_started < proc->max_threads &&
	    (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
	     BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
	     /*spawn a new thread if we leave this out */) {
		proc->requested_threads++;
		//......
		if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
			return -EFAULT;
	}
复制代码
```

当前线程要执行`Binder调用`，新来的调用也需要线程来处理，因此：

- 函数结束前会检查进程中可用的线程数
  - 如果需要创建新线程，则在返回的`buffer数据`中增加`BR_SPAWN_LOOPER`消息，`服务进程`收到这个消息会启动新的线程

完整的Binder调用过程还需要把回复消息传递给客户进程，这个过程使用的函数还是前面的这些，暂时不分析了。`消化一下先`

## 处理传递的Binder对象

前面介绍了`Binder对象`传递的原理和用户层的实现。（原理上整个人还是晕晕的），我们来看下`Binder驱动`如何实现`Binder对象`的传递的。

- ```
  Binder驱动
  ```

  中，代表每个进程的结构

  ```
  binde_proc
  ```

  中有两个字段：

  ```
  nodes
  ```

  和

  ```
  refs_by_node
  ```

  - 这两个字段各指向两颗红黑树的头
  - `nodes`：指向的是`Binder节点对象表`，储存本进程中`Binder实体对象`相关的数据
  - `refs_by_node`：指向的是`Binder引用对象表`，存储本进程的`Binder引用对象`的数据以及对应的`实体对象的节点指针`

来个抽象点的图： ![image](images/binder%E8%8A%82%E7%82%B9%E8%A1%A8%E4%B8%8E%E5%BC%95%E7%94%A8%E8%A1%A8.png)

`Binder驱动`中处理对象转换的代码位于函数`binder_transaction`中

```cpp
        case BINDER_TYPE_BINDER:
		case BINDER_TYPE_WEAK_BINDER: {
			struct flat_binder_object *fp;
			fp = to_flat_binder_object(hdr);
			ret = binder_translate_binder(fp, t, thread);
		} break;
static int binder_translate_binder(struct flat_binder_object *fp,
				   struct binder_transaction *t,
				   struct binder_thread *thread)
{
	struct binder_node *node;
	struct binder_proc *proc = thread->proc;
	struct binder_proc *target_proc = t->to_proc;
	struct binder_ref_data rdata;
	int ret = 0;
	//根据binder值查找Binder对象表中的节点
	node = binder_get_node(proc, fp->binder);
	if (!node) { //没有则新建一个节点
		node = binder_new_node(proc, fp);
		if (!node)
			return -ENOMEM;
	}
	//......
	if (security_binder_transfer_binder(proc->tsk, target_proc->tsk)) {
		ret = -EPERM;
		goto done;
	}
	// 在接收端进程中寻找节点的引用，找不到会创建一个新的引用
	ret = binder_inc_ref_for_node(target_proc, node,
			fp->hdr.type == BINDER_TYPE_BINDER,
			&thread->todo, &rdata);
	if (ret)
		goto done;
	// 将传递的binder数据结构的type的值改为 BINDER_TYPE_HANDLE
	if (fp->hdr.type == BINDER_TYPE_BINDER)
		fp->hdr.type = BINDER_TYPE_HANDLE;
	else
		fp->hdr.type = BINDER_TYPE_WEAK_HANDLE;
	fp->binder = 0;
	// rdata.desc存放的是节点引用表中的序号，赋值给handle
	fp->handle = rdata.desc;
	fp->cookie = 0;
	//......
}
复制代码
```

上面的流程是

- 通过

  ```
  binder_get_node
  ```

  函数在发送进程的

  ```
  Binder对象节点表
  ```

  中查找节点

  - 查找是通过比较数据中的`binder字段`和节点中对应的字段进行的
  - `binder字段`中存放的是`Binder对象`的弱引用指针
  - 如果没找到
    - 新建节点
    - 把`binder字段`、`cookie字段`的值保存到新节点

- 通过

  ```
  binder_inc_ref_for_node
  ```

  函数在目标进程查找

  ```
  Binder节点
  ```

  的

  ```
  引用
  ```

  。

  - 没找到会新建一个
  - `引用`指的是`节点引用表`中的`refs_by_node`节点，包含指向`Binder节点`的指针

- 把数据的`type`字段的值改为`BINDER_TYPE_HANDLE`或`BINDER_TYPE_WEAK_HANDLE`

- 把`handle`字段的值设为在`节点引用表`的序号，这也是`Binder引用对象`中`handle值`的来历

下面再看看如何处理类型`BINDER_TYPE_HANDLE`或`BINDER_TYPE_WEAK_HANDLE`的

```cpp
        case BINDER_TYPE_HANDLE:
		case BINDER_TYPE_WEAK_HANDLE: {
			struct flat_binder_object *fp;

			fp = to_flat_binder_object(hdr);
			ret = binder_translate_handle(fp, t, thread);
		} break;
static int binder_translate_handle(struct flat_binder_object *fp,
				   struct binder_transaction *t,
				   struct binder_thread *thread)
{
    //......
    // 通过handle值查找节点引用
	node = binder_get_node_from_ref(proc, fp->handle,
			fp->hdr.type == BINDER_TYPE_HANDLE, &src_rdata);
	if (!node) {
	    //.....
		return -EINVAL;
	}
	//......
	if (node->proc == target_proc) {
	    // 如果目标进程就是Binder对象的进程，开始转换
		if (fp->hdr.type == BINDER_TYPE_HANDLE)
			fp->hdr.type = BINDER_TYPE_BINDER;
		else
			fp->hdr.type = BINDER_TYPE_WEAK_BINDER;
		fp->binder = node->ptr;
		fp->cookie = node->cookie;
		if (node->proc)
			binder_inner_proc_lock(node->proc);
		binder_inc_node_nilocked(node,
					 fp->hdr.type == BINDER_TYPE_BINDER,
					 0, NULL);
		//......
	} else {
		//......
		// 如果不是，则在目标进程新建一个Binder节点的引用
		ret = binder_inc_ref_for_node(target_proc, node,
				fp->hdr.type == BINDER_TYPE_HANDLE,
				NULL, &dest_rdata);

		//......
		fp->handle = dest_rdata.desc;
		fp->cookie = 0;
		trace_binder_transaction_ref_to_ref(t, node, &src_rdata,
		//......
	}
}
复制代码
```

这部分代码的流程是：

- 通过传输数据的

  ```
  handle字段
  ```

  在发送进程的

  ```
  节点引用表
  ```

  中查找

  - 正常情况下是可以找到，不能就错误返回

- 如果目标进程就是Binder对象所在的进程

  - 开始进行转换，把数据的`type字段`转为`BINDER_TYPE_BINDER`或`BINDER_TYPE_WEAK_BINDER`BINDER_WORK_TRANSACTION
  - 把`binder`和`cookie`字段设置为节点中保存的值

- 如果目标进程不是Binder对象所在的进程

  - 在目标对象中建立一个节点对象的引用

到这里呢，Binder原理部分就差不多了，已经了解了包括：

- 客户端`调用`、`接收`消息的过程
- 服务端`监听`、`回复`消息的过
- 驱动`传输数据`、`调度线程`的操作

现在，我们再来看最后的一小部分：`ServiceManager`的作用

# `ServiceManager`的作用

关于`ServiceManager`，先简单描述：

- `ServiceManager`是`Binder架构`中用来解析`Binder名称`的模块

- `ServiceManager`本身就是一个`Binder服务`

- ```
  ServiceManager
  ```

  并没有使用

  ```
  libbinder
  ```

  来构建

  ```
  Binder服务
  ```

  - 2020-09-04 同步了下项目代码，发现也替换成`libbinder`那一套了。。。。。
  - 不确定是不是更换仓储了
  - 要是这样就没意思了，简易版本对于加深`binder`理解还是很有帮助的
  - 我们暂且按照老的版本来看下吧

- `ServiceManager`自己实现了一个简单的`binder框架`来直接和驱动通信。。。

`ServiceManager`源码路径在：`frameworks/native/cmds/servicemanager`，主要包含两个文件：

- `binder.c`：用来实现简单的Binder通信功能

  - 简单版本的`binder协议`实现
  - 直接的`ioctl`操作与`binder驱动`通信
  - 不是本节的重点，感兴趣可以参照源码来学习啦

- `service_manager.c`：用来实现`ServiceManager`的业务逻辑

  - 重点是了解`ServiceManager`如何响应`Binder服务`的注册和查询的

- 这么重要的服务要在什么时间启动呢？

  - 这部分是在 `system/core/rootdir/init.rc`中配置

  > ```rc
  > on post-fs
  >     ......
  >     # start essential services
  >     start logd
  >     start servicemanager
  >     start hwservicemanager
  >     start vndservicemanager
  > 复制代码
  > ```

  - `hwservicemanager` 用来支持`HIDL`
  - `vndservicemanager` 第三方厂商使用，应该是从`Treble架构`中出现的

## `ServiceManager`的架构

从`ServiceManager`的`main`函数开始：

```cpp
int main(int argc, char** argv)
{
    struct binder_state *bs;
    union selinux_callback cb;
    char *driver;

    if (argc > 1) {
        driver = argv[1];
    } else {
        driver = "/dev/binder";
    }

    bs = binder_open(driver, 128*1024);
    if (!bs) {
        //......
        // 省略一些宏判断
        return -1;
    }

    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }
    
    // 设置selinux callback
    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
    
    //......
    // 省略一些宏判断，都是为了获取sehandle（检查SELinux权限）
    sehandle = selinux_android_service_context_handle();
    
    selinux_status_open(true);
    if (sehandle == NULL) {
        ALOGE("SELinux: Failed to acquire sehandle. Aborting.\n");
        abort();
    }
    if (getcon(&service_manager_context) != 0) {
        ALOGE("SELinux: Failed to acquire service_manager context. Aborting.\n");
        abort();
    }
    binder_loop(bs, svcmgr_handler);

    return 0;
}
复制代码
```

`main`函数的流程如下：

- 首先调用

  ```
  binder_open
  ```

  来打开

  ```
  binder设备
  ```

  和初始化系统

  - 同时创建一块`128x1024`大小的内存空间

- 接着调用

  ```
  binder_become_context_manager
  ```

  把本进程设置为

  ```
  Binder框架
  ```

  的管理进程

  - `binder_become_context_manager`函数代码如下：

  > ```cpp
  > int binder_become_context_manager(struct binder_state *bs)
  > {
  >     return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
  > }
  > 复制代码
  > ```

  - 函数灰常简单，直接通过`ioctl`把控制命令`BINDER_SET_CONTEXT_MGR`发到了驱动

- 最后执行`binder_loop(bs, svcmgr_handler)`，简单处理后，循环等待消息

我们要说一说`binder_loop(bs, svcmgr_handler)`，小弟`C语言`不熟，感觉这个操作很`SAO`:

1. `binder_loop`的函数定义是：

```cpp
void binder_loop(struct binder_state *bs, binder_handler func)
复制代码
```

- 第二个参数传入的是一个`binder_handler`类型

1. 再来看下`binder_handler`类型的定义：

```cpp
typedef int (*binder_handler)(struct binder_state *bs,
                              struct binder_transaction_data *txn,
                              struct binder_io *msg,
                              struct binder_io *reply);
复制代码
```

- 这是定义了一个`函数指针类型`
- 返回值是`int`

1. 再来看下`binder_loop(bs, svcmgr_handler)`中的`svcmgr_handler`函数的声明

```cpp
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    //......
    // 省略一些switch语句，稍后详解
    return 0;
}
复制代码
```

- 传入的指针是`svcmgr_handler`函数指针
- 应该会自动转型为`binder_handler`指针类型
- 毕竟函数参数、返回值都是一样的
- 这部分是我觉得最神奇的地方。。。。。
  - 特意写了个C代码试了下，
  - 如果定义的函数的`参数`、`返回值`与`typedef`定义的`函数指针`不一致的话，强转编译会失败
  - 不过这种操作感觉还是很不正经。。（还是Java更严谨一些）

1. 参数理解的差不多了，我们仔细看下

   ```
   binder_loop
   ```

   的实现，这两个参数传进去干了啥。

   > PS：简化版就是更容易理解些。。。。。。

```cpp
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];
    
    // 老样子，用来记录写入到驱动的一些相关参数
    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;
    // 该属性应该是通知驱动已经准备好接收数据了
    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));

    for (;;) {
        //无限循环
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;
        // 开始读取数据
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            // 异常通信，退出循环
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }
        // 读取到数据后
        // 调用 binder_parse 进行数据解析
        // 顺便把 binder_handler 函数指针也传递进去
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        
        // 异常情况，退出循环
        if (res == 0 || res < 0) {
            //......
            break;
        }
    }
}
复制代码
```

- 有注释，很很简洁的逻辑
- 最后执行到了`binder_parse`函数

1. 我们继续跟进`binder_parse`函数

```cpp
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        //取指令
        uint32_t cmd = *(uint32_t *) ptr;
        switch(cmd) {
        //......
        // 由于binder_handler 在这里会被执行到
        // 所以我们先重点看这个，当Binder调用过来时
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            // 数据检查
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: txn too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            // 如果函数指针存在
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;
                // 一些初始化操作
                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn);
                //执行 binder_handler 函数指针指向的函数
                res = func(bs, txn, &msg, &reply);
                
                if (txn->flags & TF_ONE_WAY) {
                    // 这种情况不会返回结果
                    binder_free_buffer(bs, txn->data.ptr.buffer);
                } else {
                    // 返回结果数据
                    binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
                }
            }
            ptr += sizeof(*txn);
            break;
        }
        //......
    }
    return r;
}
复制代码
```

- 还是看注释哈，代码很简洁，注释很详细，哈哈哈

- 到这里，我们可以看出来，真正处理`调用服务`的是`binder_handler`这个函数指针啊

- 也就是说

  ```
  远程调用
  ```

  的处理逻辑在

  ```
  svcmgr_handler
  ```

  这个函数呀

  - 666！

1. 我们来看下`svcmgr_handler`函数

```cpp
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    //......
    // 如果请求的目标服务不是ServiceManager，直接返回
    if (txn->target.ptr != BINDER_SERVICE_MANAGER)
        return -1;
        
    // 如果请求消息内容只是简单的测试通路，不需要继续执行，直接返回 0
    if (txn->code == PING_TRANSACTION)
        return 0;

    // 检查收到的消息 id 串
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }
    //......
    // 检查SELinux 的权限
    if (sehandle && selinux_status_updated() > 0) {
        struct selabel_handle *tmp_sehandle = selinux_android_service_context_handle();
        if (tmp_sehandle) {
            selabel_close(sehandle);
            sehandle = tmp_sehandle;
        }
    }

    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        // 处理查询或者获取服务的指令
        //......
        break;
    case SVC_MGR_ADD_SERVICE:
        // 处理注册服务的的指令
        //......
        break;

    case SVC_MGR_LIST_SERVICES: {
        //......
        // 处理获取服务列表的指令
    }
    default:
        ALOGE("unknown code %d\n", txn->code);
        return -1;
    }
    //发送返回消息
    bio_put_uint32(reply, 0);
    return 0;
}
复制代码
```

- `ServiceManager`的架构非常简单高效，只有一个循环来和`binder驱动`进行通信

## `ServiceManager`提供的服务

从`svcmgr_handler`函数中可以看到，`ServiceManager`提供了三种服务功能：

- 注册`Binder服务`
- 查询`Binder服务`
- 获取`Binder服务`列表

### 注册`Binder服务`

在`case SVC_MGR_ADD_SERVICE`中实现的注册`Binder服务`功能，具体的实现函数是`do_add_service`，代码如下：

```cpp
int do_add_service(struct binder_state *bs, const uint16_t *s, size_t len, uint32_t handle,
                   uid_t uid, int allow_isolated, uint32_t dumpsys_priority, pid_t spid) {
    struct svcinfo *si;

    // 一些基础的信息判断
    if (!handle || (len == 0) || (len > 127))
        return -1;
    // 检查调用进程是否有权限注册服务
    if (!svc_can_register(s, len, spid, uid)) {
        //......省略log打印
        return -1;
    }
    // 查看要注册的服务是否已经存在
    si = find_svc(s, len);
    if (si) {
    // 如果存在，先把以前的Binder对象的引用计数减一
        if (si->handle) {
            svcinfo_death(bs, si);
        }
        // 把原先节点中的handle替换成新的handle
        si->handle = handle;
    } else {
        // 服务不存在，则生成新的列表项，初始化后加入列表
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {
            // 内存申请失败，直接退出
            return -1;
        }
        // 一些初始化操作
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->dumpsys_priority = dumpsys_priority;
        si->next = svclist;
        svclist = si;
    }
    // 增加Binder服务的引用计数
    binder_acquire(bs, handle);
    // 注册该Binder服务的死亡通知
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
复制代码
```

`do_add_service`函数的流程是：

- 首先检查调用的进程是否有注册服务的权限
  - 这部分是通过`SELinux`来控制的，后面会学习到
- 接着检查需要注册的服务是否已经存在
  - 存在，把原来`Binder服务`在驱动的引用计数`减一`
  - 不存在
    - 新创建一个`scvinfo`结构
    - 填充需要待注册服务的相关信息到结构中
    - 把结构加入到服务列表`svclist`
- 通知内核把`Binder服务`的引用计数`加一`
- 注册该服务的死亡通知

有木有发现对于服务已经存在的情况：

- 进行了先`减一`再`加一`的操作
- 是不是可以不用操作计数也可以呢？

可以思考下

### 查询`Binder服务`

在`case SVC_MGR_CHECK_SERVICE`中处理查询服务的功能，具体的实现接口是`do_find_service`函数，代码如下：

```cpp
uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    struct svcinfo *si = find_svc(s, len);
    if (!si || !si->handle) {
        return 0;
    }
    if (!si->allow_isolated) {
        // If this service doesn't allow access from isolated processes,
        // then check the uid to see if it is isolated.
        uid_t appid = uid % AID_USER;
        if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) {
            return 0;
        }
    }
    if (!svc_can_find(s, len, spid, uid)) {
        return 0;
    }
    return si->handle;
}
复制代码
```

`do_find_service`函数的主要工作是搜索列表、返回查找到的服务。请注意有一段判断uid的代码：

- 在调用`ServiceManager`的`addBinder`服务时有个参数`allowed_isolated`，用来指定服务是否允许从沙箱中访问

- 这里的代码应该是判断调用进程是否是一个

  ```
  隔离进程
  ```

  - 如果`appid`在`AID_ISOLATED_START(99000)`和`AID_ISOLATED_END(99999)`之间
  - 表明这个服务可以通过`allowed_isolated`来控制是否允许一般的用户进程来使用其服务

# 结语

终于、终于看完了。

书中后面其实还有`ashmem 匿名共享内存`的内容（Android自己实现的，在`mmap`基础上开发的，基于`binder`通信的内存共享）

想了想暂时不作为`binder`的笔记内容了，`binder`涉及的内容已经很复杂了，不过看完之后也是收获颇丰哈。

看来很必须要写导读(总结)了，哈哈


