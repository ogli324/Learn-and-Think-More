# Android安全防护/检查root/检查Xposed/反调试/应用多开/模拟器检测（持续更新1.1.0）

url：https://www.jianshu.com/p/c37b1bdb4757

# 文章目录

- 食用方法
- root权限检查
- Xposed框架检查
- 应用多开检查
- 反调试方案
- 模拟器检测
- TODO

------

# 使用方法

implementation 'com.lahm.library:easy-protector-release:latest.release'

**https://github.com/lamster2018/EasyProtector**

![img](https:////upload-images.jianshu.io/upload_images/2554175-0bc24eb8a302c766.png?imageMogr2/auto-orient/strip|imageView2/2/w/608/format/webp)

demo



------

# root权限检查

开发者会使用诸如xposed，cydiasubstrate的框架进行hook操作，前提是拥有root权限。

关于root的原理，请参考《Android Root原理分析及防Root新思路》
 https://blog.csdn.net/hsluoyc/article/details/50560782

简单来说就是去拿『ro.secure』的值做判断，
 ro.secure值为1，adb权限降为shell，则认为没有root权限。
 ~~但是单纯的判断该值是没法检查userdebug版本的root权限~~

~~结合《Android判断设备是User版本还是Eng版本》~~
 ~~https://www.jianshu.com/p/7407cf6c34bd~~
 ~~其实还有一个值ro.debuggable~~

|                 |  ro.secure=0   | ro.secure=1 |
| --------------- | :------------: | :---------: |
| ro.debuggable=0 |       /        |    user     |
| ro.debuggable=1 | eng/userdebug* |      /      |

*~~暂无userdebug的机器，不知道ro.secure是否为1，埋坑~~
 userdebug 的debuggable值未知，secure为0.

~~实际上通过『ro.debuggable』值判断更准确~~
 直接读取ro.secure值足够了

下一步再检验是否存在su文件
 方案来自《Android检查手机是否被root》
 https://www.jianshu.com/p/f9f39704e30c

通过检查su是否存在，su是否可执行，综合判断root权限。

*EasyProtectorLib.checkIsRoot()*的内部实现



```java
    public boolean isRoot() {
        int secureProp = getroSecureProp();
        if (secureProp == 0)//eng/userdebug版本，自带root权限
            return true;
        else return isSUExist();//user版本，继续查su文件
    }

    private int getroSecureProp() {
        int secureProp;
        String roSecureObj = CommandUtil.getSingleInstance().getProperty("ro.secure");
        if (roSecureObj == null) secureProp = 1;
        else {
            if ("0".equals(roSecureObj)) secureProp = 0;
            else secureProp = 1;
        }
        return secureProp;
    }

    private boolean isSUExist() {
        File file = null;
        String[] paths = {"/sbin/su",
                "/system/bin/su",
                "/system/xbin/su",
                "/data/local/xbin/su",
                "/data/local/bin/su",
                "/system/sd/xbin/su",
                "/system/bin/failsafe/su",
                "/data/local/su"};
        for (String path : paths) {
            file = new File(path);
            if (file.exists()) return true;//可以继续做可执行判断
        }
        return false;
    }
```

------

# Xposed框架检查

原理请参考我的《反Xposed方案学习笔记》
 https://www.jianshu.com/p/ee0062468251

所有的方案回归到一点：判断xposed的包是否存在。
 1.是通过主动抛出异常查栈信息;
 2.是主动反射调用。

当检测到xp框架存在时，我们先行调用xp方法，关闭xp框架达到反制的目的。

EasyProtectorLib.checkIsXposedExist()_内部实现



```kotlin
    private static final String XPOSED_HELPERS = "de.robv.android.xposed.XposedHelpers";
    private static final String XPOSED_BRIDGE = "de.robv.android.xposed.XposedBridge";

    //手动抛出异常，检查堆栈信息是否有xp框架包
    public boolean isEposedExistByThrow() {
        try {
            throw new Exception("gg");
        } catch (Exception e) {
            for (StackTraceElement stackTraceElement : e.getStackTrace()) {
                if (stackTraceElement.getClassName().contains(XPOSED_BRIDGE)) return true;
            }
            return false;
        }
    }

    //检查xposed包是否存在
    public boolean isXposedExists() {
        try {
            Object xpHelperObj = ClassLoader
                    .getSystemClassLoader()
                    .loadClass(XPOSED_HELPERS)
                    .newInstance();
        } catch (InstantiationException e) {
            e.printStackTrace();
            return true;
        } catch (IllegalAccessException e) {
            //实测debug跑到这里报异常
            e.printStackTrace();
            return true;
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
            return false;
        }

        try {
            Object xpBridgeObj = ClassLoader
                    .getSystemClassLoader()
                    .loadClass(XPOSED_BRIDGE)
                    .newInstance();
        } catch (InstantiationException e) {
            e.printStackTrace();
            return true;
        } catch (IllegalAccessException e) {
            //实测debug跑到这里报异常
            e.printStackTrace();
            return true;
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }

    //尝试关闭xp的全局开关，亲测可用
    public boolean tryShutdownXposed() {
        if (isEposedExistByThrow()) {
            Field xpdisabledHooks = null;
            try {
                xpdisabledHooks = ClassLoader.getSystemClassLoader()
                        .loadClass(XPOSED_BRIDGE)
                        .getDeclaredField("disableHooks");
                xpdisabledHooks.setAccessible(true);
                xpdisabledHooks.set(null, Boolean.TRUE);
                return true;
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
                return false;
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
                return false;
            } catch (IllegalAccessException e) {
                e.printStackTrace();
                return false;
            }
        } else return true;
    }
```

------

# 多开软件检测

多开软件的检测方案这里提供5种，首先4种来自
 《Android多开/分身检测》
 https://blog.darkness463.top/2018/05/04/Android-Virtual-Check/

《Android虚拟机多开检测》
 https://www.jianshu.com/p/216d65d9971e

这里提供代码整理，一键调用，



```css
        VirtualApkCheckUtil.getSingleInstance().checkByPrivateFilePath(this);
        VirtualApkCheckUtil.getSingleInstance().checkByOriginApkPackageName(this);
        VirtualApkCheckUtil.getSingleInstance().checkByHasSameUid();
        VirtualApkCheckUtil.getSingleInstance().checkByMultiApkPackageName();
        VirtualApkCheckUtil.getSingleInstance().checkByPortListening(getPackageName(),CallBack);
```

第5种方案来自我同事的启发，我起名叫端口检测法，具体思路已经单独成文见
 《一行代码帮你检测Android多开软件》
 https://www.jianshu.com/p/65c841749dd6

测试情况

| 测试机器/多开软件*                | 多开分身6.9 | 平行空间4.0.8389 | 双开助手3.8.4 | 分身大师2.5.1 | VirtualXP0.11.2 | Virtual App * |
| --------------------------------- | ----------- | ---------------- | ------------- | ------------- | --------------- | ------------- |
| 红米3S/Android6.0/原生eng         | XXXOO       | OXOOO            | OXOOO         | XOOOO         | XXXOO           | XXXOO         |
| 华为P9/Android7.0/EUI 5.0 root    | XXXXO       | OXOXO            | OXOXO         | XOOXO         | XXXXO           | XXXOO         |
| 小米MIX2/Android8.0/MIUI稳定版9.5 | XXXXO       | OXOXO            | OXOXO         | XOOXO         | XXXXO           | XXXOO         |
| 一加5T/Android8.1/氢OS 5.1 稳定版 | XXXXO       | OXOXO            | OXOXO         | XOOXO         | XXXXO           | XXXOO         |

*测试方案顺序如下12345，测试结果X代表未能检测O成功检测多开
 *virtual app测试版本是git开源版，商用版已经修复uid的问题
 *现在方案1234检测狭义多开已经失效，建议使用方案5做广义多开检测

#### 1.文件路径检测

![img](https:////upload-images.jianshu.io/upload_images/2554175-d54c9e5f5fba518f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



```tsx
public boolean checkByPrivateFilePath(Context context) {
        String path = context.getFilesDir().getPath();
        for (String virtualPkg : virtualPkgs) {
            if (path.contains(virtualPkg)) return true;
        }
        return false;
    }
```

#### 2.应用列表检测

![img](https:////upload-images.jianshu.io/upload_images/2554175-f53ac565a296c42d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



简单来说，多开app把原始app克隆了，并让自己的包名跟原始app一样，当使用克隆app时，会检测到原始app的包名会和多开app包名一样（就是有两个一样的包名）



```csharp
    public boolean checkByOriginApkPackageName(Context context) {
        try {
            if (context == null)  return false;
            int count = 0;
            String packageName = context.getPackageName();
            PackageManager pm = context.getPackageManager();
            List<PackageInfo> pkgs = pm.getInstalledPackages(0);
            for (PackageInfo info : pkgs) {
                if (packageName.equals(info.packageName)) {
                    count++;
                }
            }
            return count > 1;
        } catch (Exception ignore) {
        }
        return false;
    }
```

##### 3.maps检测

![img](https:////upload-images.jianshu.io/upload_images/2554175-cf0005968d448331.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

需要维护多款分身包名



```tsx
    public boolean checkByMultiApkPackageName() {
        BufferedReader bufr = null;
        try {
            bufr = new BufferedReader(new FileReader("/proc/self/maps"));
            String line;
            while ((line = bufr.readLine()) != null) {
                for (String pkg : virtualPkgs) {
                    if (line.contains(pkg)) {
                        return true;
                    }
                }
            }
        } catch (Exception ignore) {

        } finally {
            if (bufr != null) {
                try {
                    bufr.close();
                } catch (IOException e) {

                }
            }
        }
        return false;
    }
```

#### 4.ps检测

![img](https:////upload-images.jianshu.io/upload_images/2554175-86078e4a840b4ddb.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



简单来说，检测自身进程，如果该进程下的包名有不同多个私有文件目录，则认为被多开



```dart
    public boolean checkByHasSameUid() {
        String filter = getUidStrFormat();//拿uid
        String result = CommandUtil.getSingleInstance().exec("ps");
        if (result == null || result.isEmpty()) return false;

        String[] lines = result.split("\n");
        if (lines == null || lines.length <= 0) return false;

        int exitDirCount = 0;
        for (int i = 0; i < lines.length; i++) {
            if (lines[i].contains(filter)) {
                int pkgStartIndex = lines[i].lastIndexOf(" ");
                String processName = lines[i].substring(pkgStartIndex <= 0
                        ? 0 : pkgStartIndex + 1, lines[i].length());
                File dataFile = new File(String.format("/data/data/%s", processName, Locale.CHINA));
                if (dataFile.exists()) {
                    exitDirCount++;
                }
            }
        }

        return exitDirCount > 1;
    }
```

#### 5.端口检测

前4种方案，有一种直接对抗的意思，不希望我们的app运行在多开软件中，第5种方案，我们不直接对抗，只要不是在同一机器上同时运行同一app，我们都认为该app没有被多开。
 假如同时运行着两个app（无论先开始运行），两个app进行一个通信，如果通信成功，我们则认为其中有一个是克隆体。



```java
        //遍历查找已开启的端口
        String tcp6 = CommandUtil.getSingleInstance().exec("cat /proc/net/tcp6");
        if (TextUtils.isEmpty(tcp6)) return;
        String[] lines = tcp6.split("\n");
        ArrayList<Integer> portList = new ArrayList<>();
        for (int i = 0, len = lines.length; i < len; i++) {
            int localHost = lines[i].indexOf("0100007F:");//127.0.0.1:
            if (localHost < 0) continue;
            String singlePort = lines[i].substring(localHost + 9, localHost + 13);
            Integer port = Integer.parseInt(singlePort, 16);
            portList.add(port);
        }
```

对每个端口开启线程尝试连接,并且发送一段自定义的消息，作为钥匙，这里一般发送包名就行(刚好多开软件会把包名处理）



```java
            Socket socket = new Socket("127.0.0.1", port);
            socket.setSoTimeout(2000);
            OutputStream outputStream = socket.getOutputStream();
            outputStream.write((secret + "\n").getBytes("utf-8"));
            outputStream.flush();
            socket.shutdownOutput();
```

之后自己再开启端口监听作为服务器，等待连接，如果被连接上之后且消息匹配，则认为有一个克隆体在同时运行。



```java
private void startServer(String secret) {
        Random random = new Random();
        ServerSocket serverSocket = null;
        try {
            serverSocket = new ServerSocket();
            serverSocket.bind(new InetSocketAddress("127.0.0.1",
                    random.nextInt(55534) + 10000));
            while (true) {
                Socket socket = serverSocket.accept();
                ReadThread readThread = new ReadThread(secret, socket);
                readThread.start();
//                serverSocket.close();
            }
        } catch (BindException e) {
            startServer(secret);//may be loop forever
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

*因为端口通信需要Internet权限，本库不会通过网络上传任何隐私

#### 6.LocalServerSocket端口占用

基本思路与前面一致，但是会更简单
 借助LocalServerSocket的构造方法完成端口占用 v1.1.0支持



```dart
/**
     * Creates a new server socket listening at specified name.
     * On the Android platform, the name is created in the Linux
     * abstract namespace (instead of on the filesystem).
     * 
     * @param name address for socket
     * @throws IOException
     */
    public LocalServerSocket(String name) throws IOException {
        impl = new LocalSocketImpl();
        impl.create(LocalSocket.SOCKET_STREAM);
        localAddress = new LocalSocketAddress(name);
        impl.bind(localAddress);
        impl.listen(LISTEN_BACKLOG);
    }
```

------

# 反调试方案

我们不希望自己的app被反编译/动态调试，那首先应该了解如何反编译/动态调试，此处可以参考我的《动态调试笔记--调试smali》
 https://www.jianshu.com/p/90f495191a6a

然后从调试的步骤来分析学习检测。

1.修改清单更改apk版本为debug版，我们发出去的包为release包，进行调试的话，要求为debug版（如果是已root的机器则没有这个要求），所以首先可检查当前版本是否为debug，或者签名信息有没有被更改。



```java
public boolean checkIsDebugVersion(Context context) {
        return (context.getApplicationInfo().flags
                & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
    }
```

该方法提供了C++实现,见
 https://github.com/lamster2018/learnNDK/blob/master/app/src/main/jni/ctest.cpp的checkDebug方法

2.等待调试器附加，直接用api检查debugger是否被附加



```java
public boolean checkIsDebuggerConnected() {
        return android.os.Debug.isDebuggerConnected();
    }
```

实测效果，可以结合电量变化的广播监听来做usb插拔监听，如果是usb充电，此时来检查debugger是否被插入，但是debugger attach到app需要一定时间，所以并不是实时的，还有我们常用的waiting for attach，建议监听到usb插上，开启一个子线程轮训检查，30s后关闭这个子线程。



```java
    //检查usb充电状态
    public boolean checkIsUsbCharging(Context context) {
        IntentFilter filter = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
        Intent batteryStatus = context.registerReceiver(null, filter);
        if (batteryStatus == null) return false;
        int chargePlug = batteryStatus.getIntExtra(BatteryManager.EXTRA_PLUGGED, -1);
        return chargePlug == BatteryManager.BATTERY_PLUGGED_USB;
    }
```

3.检查端口占用



```java
public boolean isPortUsing(String host, int port) throws UnknownHostException {
        boolean flag = false;
        InetAddress theAddress = InetAddress.getByName(host);
        try {
            Socket socket = new Socket(theAddress, port);
            flag = true;
        } catch (IOException e) {
        }
        return flag;
    }
```

4.当app被调试的时候，进程中会有traceid被记录，该原理可参考
 《jni动态注册/轮询traceid/反调试学习笔记》
 https://www.jianshu.com/p/082456acf89c

检查traceid提供java和c++实现
 原理都是轮询读取/proc/Pid/status的TracerPid值
 当debugger attach到app时，tracerId不为0，如ida附加调试时，tracerId为23946.
 *测试机华为P9，会自己给自己附加一个tracer，该值小于1000

鉴于篇幅，此处不贴c++代码。
 *EasyProtectorLib.checkIsBeingTracedByC()*使用c++方案



```kotlin
public boolean readProcStatus() {
        try {
            BufferedReader localBufferedReader =
                    new BufferedReader(new FileReader("/proc/" + Process.myPid() + "/status"));
            String tracerPid = "";
            for (; ; ) {
                String str = localBufferedReader.readLine();
                if (str.contains("TracerPid")) {
                    tracerPid = str.substring(str.indexOf(":") + 1, str.length()).trim();
                    break;
                }
                if (str == null) {
                    break;
                }
            }
            localBufferedReader.close();
            if ("0".equals(tracerPid)) return false;
            else return true;
        } catch (Exception fuck) {
            return false;
        }
    }
```

# 模拟器检测

具体研究单独成文见《一行代码帮你检测Android模拟器》
 https://www.jianshu.com/p/434b3075b5dd

现在的模拟器基本可以做到模拟手机号码，手机品牌，cpu信息等，常规的java方案也可能被hook掉，比如逍遥模拟器读取ro.product.board进行了处理，能得到设置的cpu信息。

在研究各个模拟器的过程中，尤其是在研究[build.prop](http://www.miui.com/article-239-1.html)文件时，发现各个模拟器的处理方式不一样，比如以下但不限于
 1.基带信息几乎没有；
 2.处理器信息ro.product.board和ro.board.platform异常；
 3.部分模拟器在读控制组信息时读取不到；
 4.连上wifi但会出现 Link encap:UNSPEC未指定网卡类型的情况

新增
 增加特定值的检测，比如天天模拟器的hardware是ttVM
 传感器的检测，部分模拟器的传感器个数只有1个
 安装包的检测，模拟器的用户预装app很少
 结合以上信息，综合判断是否运行在模拟器中。

*EasyProtectorLib.checkIsRunningInEmulator()*的代码实现如下



```java
    @Deprecated
    public boolean readSysProperty() {
        return readSysProperty(null, null);
    }

    public boolean readSysProperty(Context context, EmulatorCheckCallback callback) {
        this.emulatorCheckCallback = callback;
        int suspectCount = 0;

        String baseBandVersion = getProperty("gsm.version.baseband");
        if (null == baseBandVersion || baseBandVersion.contains("1.0.0.0"))
            ++suspectCount;//基带信息

        String buildFlavor = getProperty("ro.build.flavor");
        if (null == buildFlavor || buildFlavor.contains("vbox") || buildFlavor.contains("sdk_gphone"))
            ++suspectCount;//渠道

        String productBoard = getProperty("ro.product.board");
        if (null == productBoard || productBoard.contains("android") | productBoard.contains("goldfish"))
            ++suspectCount;//芯片

        String boardPlatform = getProperty("ro.board.platform");
        if (null == boardPlatform || boardPlatform.contains("android"))
            ++suspectCount;//芯片平台

        String hardWare = getProperty("ro.hardware");
        if (null == hardWare) ++suspectCount;
        else if (hardWare.toLowerCase().contains("ttvm")) suspectCount += 10;//天天
        else if (hardWare.toLowerCase().contains("nox")) suspectCount += 10;//夜神

        String cameraFlash = "";
        String sensorNum = "sensorNum";
        if (context != null) {
            boolean isSupportCameraFlash = context.getPackageManager().hasSystemFeature("android.hardware.camera.flash");//是否支持闪光灯
            if (!isSupportCameraFlash) ++suspectCount;
            cameraFlash = isSupportCameraFlash ? "support CameraFlash" : "unsupport CameraFlash";

            SensorManager sm = (SensorManager) context.getSystemService(Context.SENSOR_SERVICE);
            int sensorSize = sm.getSensorList(Sensor.TYPE_ALL).size();
            if (sensorSize < 7) ++suspectCount;//传感器个数
            sensorNum = sensorNum + sensorSize;
        }

        String userApps = CommandUtil.getSingleInstance().exec("pm list package -3");
        String userAppNum = "userAppNum";
        int userAppSize = getUserAppNums(userApps);
        if (userAppSize < 5) ++suspectCount;//用户安装的app个数
        userAppNum = userAppNum + userAppSize;

        String filter = CommandUtil.getSingleInstance().exec("cat /proc/self/cgroup");
        if (null == filter) ++suspectCount;//进程租

        if (emulatorCheckCallback != null) {
            StringBuffer stringBuffer = new StringBuffer("ceshi start|")
                    .append(baseBandVersion).append("|")
                    .append(buildFlavor).append("|")
                    .append(productBoard).append("|")
                    .append(boardPlatform).append("|")
                    .append(hardWare).append("|")
                    .append(cameraFlash).append("|")
                    .append(sensorNum).append("|")
                    .append(userAppNum).append("|")
                    .append(filter).append("|end");
            emulatorCheckCallback.findEmulator(stringBuffer.toString());
            emulatorCheckCallback = null;
        }
        return suspectCount > 3;
    }
```

以下是测试情况*

|    机器/测试方案     | 检测结果 |
| :------------------: | -------- |
|   AS自带模拟器 9.0   | 模拟器   |
|   Genymotion2.12.1   | 模拟器   |
|   逍遥模拟器6.0.0    | 模拟器   |
|       Appetize       | 模拟器   |
| 夜神模拟器6.2.5.3010 | 模拟器   |
| 腾讯手游助手2.0.6.8  | 模拟器   |
|    雷电模拟器3.41    | 模拟器   |
|   木木模拟器2.0.25   | 模拟器   |
|        一加5T        | 真机     |
|        华为P9        | 真机     |

*因安卓机型太广，真机覆盖测试不完全，有空大家去git提issue

# TODO

1.Accessibility检查（反自动抢红包/接单）；
 2.模拟器的光感，陀螺仪检测；-v1.0.5support
 3.检测到模拟器/多开应该给回调给开发者自行处理，而不是直接FC；--v1.0.4 support


