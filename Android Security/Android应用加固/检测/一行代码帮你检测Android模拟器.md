# 一行码帮你检测Android模拟器

url：https://juejin.cn/post/6844903637840297998

## 目录

- 简介
- 初代常规手段
- 进阶手段
- 改良手段和新思路
- 最终方案
- 测试结果
- Demo地址

## 简介

最近有业务上的要求，要求app在本地进行诸如软件多开、hook框架、模拟器等安全检测，防止作弊行为。

防作弊一直是老生常谈的问题，而模拟器的检测往往是防作弊中的重要一环，但在查找资料的过程中发现，网上的模拟器检测方案已经有些过时了，只能自己再跟进学习，本文对这次学习内容进行总结： **模拟器的检测秉持一句话：抓取特征值与真机比较。**

## 初代常规手段

早期模拟器没那么多套路，特征值非常明显，某些值甚至是一长串的0，检测起来很方便，常规的方案如

- 检查手机IMEI等一系列编号

```
TelephonyManager tm = (TelephonyManager)this.getSystemService(Context.TELEPHONY_SERVICE);  
  String deviceid = tm.getDeviceId();//获取IMEI号  
  String te1  = tm.getLine1Number();//获取本机号码  
  String imei = tm.getSimSerialNumber();//获得SIM卡的序号  
  String imsi = tm.getSubscriberId();//得到用户Id 
 
复制代码
```

- 读取手机品牌信息

```
	android.os.Build.BRAND,
	android.os.Build.MANUFACTURER,
	android.os.Build.MODEL
	...
复制代码
```

- 检查cpu信息

```
		String value = null;
        Object roSecureObj;
        try {
            roSecureObj = Class.forName("android.os.SystemProperties")
                    .getMethod("get", String.class)
                    .invoke(null, "ro.product.board");
            if (roSecureObj != null) value = (String) roSecureObj;
        } catch (Exception e) {
            value = null;
        } finally {
            return value;
        }

复制代码
```

优点：通过检查真机上最直白的几个特征，就可以完成模拟器的检测

缺点：

1.现在的模拟器基本可以做到模拟手机号码，手机品牌，cpu信息等，比如逍遥/夜神模拟器读取ro.product.board进行了处理，能得到预先设置的cpu信息；

2.真机的手机号码也不一定就能拿到（比如电信卡）;

3.拿手机号码这个需要权限，用户不一定喜欢。

所以决定弃用以上方案。

## 进阶手段

再思考真机上的特征，进一步我们有通过检查硬件信息的思路，形如蓝牙，语音输入设备，wlan，相机等

- 检查mac地址

```
		Enumeration networkInterfaces;
        String str = null;
        networkInterfaces = NetworkInterface.getNetworkInterfaces();
        while (networkInterfaces.hasMoreElements()) {
            NetworkInterface networkInterface = (NetworkInterface) networkInterfaces.nextElement();
            if (networkInterface != null) {
                byte[] hardwareAddress;
                byte[] bArr = new byte[0];
                hardwareAddress = networkInterface.getHardwareAddress();
                if (!(hardwareAddress == null || hardwareAddress.length == 0)) {
                    String str2;
                    StringBuilder stringBuilder = new StringBuilder();
                 ...//方法太长
                }
            }
        }
}
复制代码
```

- 检查电池温度，轮询检查电量，充电状态

```
		IntentFilter filter = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
        Intent batteryStatus = context.registerReceiver(null, filter);
        if (batteryStatus == null) return false;
        int chargePlug = batteryStatus.getIntExtra(BatteryManager.EXTRA_PLUGGED, -1);
        return chargePlug == BatteryManager.BATTERY_PLUGGED_USB;//检测usb充电
复制代码
```

优点：比初代方案有深入；

缺点： 1.mac地址现在可以被模拟，且获取mac地址的代码有点长（M以下版本还要传context）写起来不不优雅；

2.通过电池信息来准确检测，需要一定的时间间隔，属于非实时方案；

3.蓝牙和相机需要添加相应权限。

所以不推荐集成。

## 改进方案和新的研究

在研究各个模拟器的过程中，尤其是在研究[build.prop](http://www.miui.com/article-239-1.html)文件时，发现以下(但不限于)问题 1.基带信息几乎没有；

2.处理器信息ro.product.board和ro.board.platform有冲突或者不一致；

3.部分模拟器在读控制组信息时读取不到；

4.连上wifi但会出现 Link encap:UNSPEC未指定网卡类型的情况。

借着问题依次进行解析。

- 基带信息

基带是手机上的一块电路板，刷基带实际上就是刷这个电路的控制软件。

我是这样去理解模拟器没有基带信息的情况"因为模拟器没有真实的电路板（基带电路），所以没法刷基带软件进去，所以没办法得到基带信息"，不知道这样理解对不对，欢迎拍砖。

当然了，部分真机在刷机失败的时候也会出现丢失基带的情况，这部分机器我们不多讨论。

```
		try {
            roSecureObj = Class.forName("android.os.SystemProperties")
                    .getMethod("get", String.class)
                    .invoke(null, "gsm.version.baseband");
            if (roSecureObj != null) value = (String) roSecureObj;
        } catch (Exception e) {
            value = null;
        }
复制代码
```

- 处理器信息

最简单的方法就是直接拿android.os.Build.BOARD，实际上也是去读取ro.product.board值，

这个值代表cpu型号，比如msm8998是骁龙835，hi3650是麒麟950。

这个值真机几乎不为空，AS模拟器会有如gphone的特征值，部分模拟器上是可以随时变更的（因为拿模拟器来玩高帧率模式的手游）。

可是还有一个ro.board.platform值，这个值代表主板平台，极少的模拟器会去更改这个值，甚至有的模拟器没有这个值，一般来说真机的两值相等。

当然真机也有例外，测试机一加5T两者都是msm8998，而华为P9 board值EVA-AL10,platform值hi3650。



![两者不一致](images/1649260b0685766b)



根据处理器信息做一个检测指标。

```
String productBoard = CommandUtil.getSingleInstance().getProperty("ro.product.board");
if (productBoard == null | "".equals(productBoard)) ++suspectCount;

String boardPlatform = CommandUtil.getSingleInstance().getProperty("ro.board.platform");
if (boardPlatform == null | "".equals(boardPlatform)) ++suspectCount;

if (productBoard != null && boardPlatform != null && !productBoard.equals(boardPlatform))
    ++suspectCount;
复制代码
```

- 渠道信息

渠道信息是ro.build.flavor值，在有限的真机和模拟机器的测试情况下，有以下推测 『真机基本上都有这个值，部分模拟器没有这个值，基于vbox的模拟器上有特征值：vbox』



![vbox特征值，平台信息空](images/1649260b06b51f31)



根据渠道信息做一个检测指标

```
String buildFlavor = CommandUtil.getSingleInstance().getProperty("ro.build.flavor");
if (buildFlavor == null | "".equals(buildFlavor) | (buildFlavor != null && buildFlavor.contains("vbox")))
    ++suspectCount;
复制代码
```

- 进程组信息

利用读取maps文件检测软件多开的时候，在部分模拟器上却遇到了runtimeException异常。 原因是读取/proc/self/cgroup进程组信息的时候，部分模拟器没有这个值，因为个人水平有限，暂时不知道原因是什么，不过却刚好拿这个做检测方案。

```
关键代码
process = Runtime.getRuntime().exec("sh");
            bufferedOutputStream = new BufferedOutputStream(process.getOutputStream());
            bufferedInputStream = new BufferedInputStream(process.getInputStream());
            bufferedOutputStream.write("cat /proc/self/cgroup");
            bufferedOutputStream.write('\n');
            bufferedOutputStream.flush();
复制代码
```

- wlan驱动未指定异常

Android离不开unix，所以尝试了adb shell 运行指令。运行ifconfig时，发现在连接wifi的情况下，AS模拟器显示 『wlan0  Link encap:UNSPEC』 未指定网卡类型，而真机情况下是『wlan0  Link encap:Ethernet』以太网。



![AS模拟器的wlan情况](images/1649260b0743f59f)



不过接着测试非wifi情况下，该值都拿不到，所以不推荐使用。

## 最终方案

结合以上研究，得出一个嫌疑指数，综合判断是否运行在模拟器中。

*EasyProtectorLib.checkIsRunningInEmulator()*的代码实现如下

```
    public boolean readSysProperty() {
        int suspectCount = 0;
        //读基带信息
        String basebandVersion = CommandUtil.getSingleInstance().getProperty("gsm.version.baseband");
        if (TextUtils.isEmpty(baseBandVersion))
           ++suspectCount;
        //读渠道信息，针对一些基于vbox的模拟器
        String buildFlavor = CommandUtil.getSingleInstance().getProperty("ro.build.flavor");
        if (TextUtils.isEmpty(buildFlavor) | (buildFlavor != null && buildFlavor.contains("vbox")))
            ++suspectCount;
        //读处理器信息，这里经常会被处理
        String productBoard = CommandUtil.getSingleInstance().getProperty("ro.product.board");
        if (TextUtils.isEmpty(productBoard) | (productBoard != null && productBoard.contains("android")))
            ++suspectCount;
        //读处理器平台，这里不常会处理
        String boardPlatform = CommandUtil.getSingleInstance().getProperty("ro.board.platform");
        if (TextUtils.isEmpty(boardPlatform) | (boardPlatform != null && boardPlatform.contains("android"))) 
            ++suspectCount;
        //高通的cpu两者信息一般是一致的
       if (!TextUtils.isEmpty(productBoard)
                && !TextUtils.isEmpty(boardPlatform)
                && !productBoard.equals(boardPlatform))
            ++suspectCount;
        //一些模拟器读取不到进程租信息
        String filter = CommandUtil.getSingleInstance().exec("cat /proc/self/cgroup");
        if (filter == null || filter.length() == 0) ++suspectCount;

        return suspectCount > 2;
    }
复制代码
```

以下是测试情况*

| 机器/测试方案     | 基带信息 | 渠道信息 | 处理器信息 | 进程组 | 检测结果 |
| ----------------- | -------- | -------- | ---------- | ------ | -------- |
| AS自带模拟器      | O        | O        | O          | X      | 模拟器   |
| Genymotion2.12.1  | O        | O        | O          | X      | 模拟器   |
| 逍遥模拟器5.3.2   | O        | X        | X          | O      | 模拟器   |
| Appetize          | O        | X        | O          | X      | 模拟器   |
| 夜神模拟器6.1.1   | O        | O        | O          | O      | 模拟器   |
| 腾讯手游助手2.0.5 | O        | O        | O          | X      | 模拟器   |
| 雷电模拟器3.27    | O        | X        | X          | X      | 真机*    |
| 一加5T            | X        | X        | X          | X      | 真机     |
| 华为P9            | X        | X        | O          | X      | 真机     |

*O代表该方案检测为模拟器，X代表检测正常

*Xamarin/Manymo因为网络原因暂未进行测试

*雷电模拟器后续修复

*因安卓机型太广，真机覆盖测试不完全，有空大家去git提issue

## Demo地址

本文方案已经集成到[EasyProtectorLib](https://jcenter.bintray.com/com/lahm/library/easy-protector-release/)

github地址： [github.com/lamster2018…](https://github.com/lamster2018/EasyProtector)

中文文档见：[www.jianshu.com/p/c37b1bdb4…](https://www.jianshu.com/p/c37b1bdb4757)

## Todo

1.检测到多开应该提供回调给开发者自行处理；


