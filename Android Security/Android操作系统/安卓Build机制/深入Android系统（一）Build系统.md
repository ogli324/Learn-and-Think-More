# 深入Android系统（一）Build系统

url：https://juejin.cn/post/6854573211045068814



> `深入Android系统`这本书是以Android5.0为基础讲解，但本人使用的是Android9.0的源码，所以和原书内容会有些出入。

> 对于Android的构建系统，在`Android7.0`之后Google就已经使用Soong构建系统，旨在取代 Make。它利用 Kati GNU Make 克隆工具和 Ninja 构建系统组件来加速 Android 的构建。[这里是官方构建传送门](https://source.android.google.cn/setup/build)

**读书是一个引导学习的过程，读此书的目的是全面了解Android系统，当有一个全面了解后再来看新版特性吧。5.0 确实老了点。哈哈哈（PS：公司的项目都是9.0的，两个版本对比学习吧）**

Android的`Build系统`是基于`GNU Make`和`Shell`构建的一套编译环境。由于Android是一个庞大的系统，包含了很多模块，为了管理整套源码的编译，Android专门开发了自己的`Build系统`。

Android的`Build系统`可以做：

- 系统二进制文件或apk应用程序的编译、链接、打包工作
- 生成目标文件系统的镜像以及各种配置文件
- 维护模块间的依赖关系
- 确保某个模块的修改能引起依赖文件的重新编译

从大的方面讲，Android的`Build系统`可以分成三部分：

- Android`Build系统`的框架和核心，位于build/core目录下
- 各个产品的配置文件，位于device目录下
- 各个模块的配置文件:`Android.mk`，位于各个模块的源文件目录下

# Android `Build系统` 核心

编译源码时常用的三条指令是：

```
source build/envsetup.sh
lunch eva_userdebug
make
复制代码
```

我们就顺着这三条指令，一步步分析整个编译过程。

## 编译环境的建立

### `envsetup.sh`文件的作用

执行Android系统的编译，必须先运行`envsetup.sh`脚本，这个脚本会建立Android的编译环境。我们看下`envsetup.sh`的部分代码：

```
function add_lunch_combo() # 忽略实现
function print_lunch_menu() # 忽略实现
function lunch() # 忽略实现
function m() # 忽略实现
function findmakefile() # 忽略实现
function mm() # 忽略实现
function mmm() # 忽略实现
function mma() # 忽略实现
function mmma() # 忽略实现
function croot() # 忽略实现
function cproj() # 忽略实现
复制代码
```

我们看到`envsetup.sh`文中定义了很多shell命令，上面列出的是一部分常用指令的定义，在执行完`envsetup.sh`脚本后，这些指令就可以调用了。

而`envsetup.sh`文件中会执行的命令是：

```
# add the default one here
add_lunch_combo aosp_arm-eng
add_lunch_combo aosp_arm64-eng
add_lunch_combo aosp_mips-eng
add_lunch_combo aosp_mips64-eng
add_lunch_combo aosp_x86-eng
add_lunch_combo aosp_x86_64-eng

# Execute the contents of any vendorsetup.sh files we can find.
for f in `test -d device && find -L device -maxdepth 4 -name 'vendorsetup.sh' 2> /dev/null | sort` \
         `test -d vendor && find -L vendor -maxdepth 4 -name 'vendorsetup.sh' 2> /dev/null | sort` \
         `test -d product && find -L product -maxdepth 4 -name 'vendorsetup.sh' 2> /dev/null | sort`
do
    echo "including $f"
    . $f
done
unset f
复制代码
```

这部分代码中：

- `add_lunch_combo`增加了6条编译选项
- for循环查找`device`、`product`、`vendor`目录下的`vendorsetup.sh`文件，并执行它们

以公司的业务为例，在`vendor/formovie/goblin/vendorsetup.sh`中的内容如下：

```
add_lunch_combo goblin-eng
add_lunch_combo goblin-user
add_lunch_combo goblin-userdebug
add_lunch_combo goblin_fact-userdebug
复制代码
```

这样，`envsetup.sh`做了三件事情：

- 创建`shell`命令
- 查找第三方定义的`vendorsetup.sh`
- 执行`add_lunch_combo`

那么`add_lunch_combo`是在干啥呢？
 我们看下实现：

```
unset LUNCH_MENU_CHOICES
function add_lunch_combo()
{
    local new_combo=$1
    local c
    # for循环查找数组中是否已存在新元素
    for c in ${LUNCH_MENU_CHOICES[@]} ; do
        if [ "$new_combo" = "$c" ] ; then
            return
        fi
    done
    # 将新元素加入 LUNCH_MENU_CHOICES 中
    LUNCH_MENU_CHOICES=(${LUNCH_MENU_CHOICES[@]} $new_combo)
}
复制代码
```

`add_lunch_combo`的功能是将调用该命令时传入的参数存放到一个全局变量数组`LUNCH_MENU_CHOICES`中。代码中有注释哈。==shell 的语法规则看着很随意啊，没有俺们Java严禁呢==

**`lunch`指令打印的就是这个数组的内容**

既然`envsetup.sh`定义了那么编译指令，我们看下都是干啥用的吧：

| 命令    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| lunch   | 指定当前编译的产品                                           |
| tapas   | 设置Build环境变量                                            |
| croot   | 快速切换到源码的根目录                                       |
| m       | 编译整个源码，无需切换到根目录                               |
| mm      | 编译当前目录下的所有模块，但是不编译它们的依赖               |
| mma     | 编译当前目录下的所有模块，同时编译它们的依赖                 |
| mmm     | 编译指定目录下的所有模块，但是不编译它们的依赖               |
| mmma    | 编译指定目录下的所有模块，同时编译它们的依赖                 |
| cgrep   | 对系统所有的c/c++文件执行grep命令                            |
| ggrep   | 对系统所有本地的Gradle文件执行grep命令                       |
| jgrep   | 对系统所有的Java文件执行grep命令                             |
| resgrep | 对系统中所有res目录下的xml文件执行grep命令                   |
| sgrep   | 对系统中的所有源文件执行grep命令                             |
| godir   | 根据godir后的参数文件名在整个源码目录中查找，然后切换到该目录 |

### `lunch`命令的功能

我们看下`lunch`的源码（加注释的那种）：

```
function lunch()
{
    local answer
    # 判断lunch指令后是否有跟随参数
    if [ "$1" ] ; then
        answer=$1
    else
        # 无参数打印可用项目列表
        print_lunch_menu
        # 强行推荐一波
        echo -n "Which would you like? [aosp_arm-eng] "
        # 等待用户输入并赋值到 answer
        read answer
    fi

    local selection=
    
    # 如果answer 是空，给个默认值aosp_arm-eng
    if [ -z "$answer" ]
    then
        selection=aosp_arm-eng
    # 判断是否是数字，作者这是把 grep 用到了极致了啊
    # grep -q 表示quite模式，有匹配立即返回 0
    # grep -e 后面跟随一个匹配表达式，多个表达式需要多个-e
    elif (echo -n $answer | grep -q -e "^[0-9][0-9]*$")
    then
        # answer 在数组范围中，就给selection赋值
        if [ $answer -le ${#LUNCH_MENU_CHOICES[@]} ]
        then
            selection=${LUNCH_MENU_CHOICES[$(($answer-1))]}
        fi
    else
        # 不是数字，不为空，直接使用传入的参数
        selection=$answer
    fi

    export TARGET_BUILD_APPS=

    local product variant_and_version variant version
    #从左向右剔除出现第一个`-`及其后面的字符，赋值给 product
    product=${selection%%-*} # Trim everything after first dash
    #从左向右剔除出现第一个`-`及其前面的字符，赋值给 variant_and_version
    variant_and_version=${selection#*-} # Trim everything up to first dash
    
    # 这里的逻辑是要取出variant 的类型是eng、user、userdebug
    if [ "$variant_and_version" != "$selection" ]; then
        variant=${variant_and_version%%-*}
        # version解析，没有解析到的话也不会报错
        if [ "$variant" != "$variant_and_version" ]; then
            version=${variant_and_version#*-}
        fi
    fi

    # 如果 product 为空的话，返回错误
    if [ -z "$product" ]
    then
        echo
        echo "Invalid lunch combo: $selection"
        return 1
    fi
    
    TARGET_PRODUCT=$product \
    TARGET_BUILD_VARIANT=$variant \
    TARGET_PLATFORM_VERSION=$version \
    # 设置build变量
    build_build_var_cache
    # 判断 build_build_var_cache 的返回值，如果不为0，退出
    if [ $? -ne 0 ]
    then
        return 1
    fi
    
    export TARGET_PRODUCT=$(get_build_var TARGET_PRODUCT)
    export TARGET_BUILD_VARIANT=$(get_build_var TARGET_BUILD_VARIANT)
    if [ -n "$version" ]; then
      # 字符串长度不为0，设置 TARGET_PLATFORM_VERSION
      export TARGET_PLATFORM_VERSION=$(get_build_var TARGET_PLATFORM_VERSION)
    else
      # 字符串长度为0，取消设置 TARGET_PLATFORM_VERSION
      unset TARGET_PLATFORM_VERSION
    fi
    export TARGET_BUILD_TYPE=release

    echo
    # 设置一些编译时的环境，比如输出路径、输出镜像的序列号等
    set_stuff_for_environment
    #打印配置信息
    printconfig
    # 清除本地缓存
    destroy_build_var_cache
}
复制代码
```

lunch 代码主要作用是根据用户输入或者选择的产品名来设置与具体产品相关的环境变量。

lunch中赋值的相关变量主要是（以`goblin_fact-userdebug`为例）：

- `TARGET_PRODUCT`：对应`goblin_fact`
- `TARGET_BUILD_VARIANT`：对应`userdebug`
- `TARGET_BUILD_TYPE`：对应`release`

## Build相关的环境变量

执行完lunch命令后，输出如下：

```
...
此处显示add_lunch_combo添加的编译项目，并标有序列号
...
Which would you like? [aosp_arm-eng] 
复制代码
```

等待确认使用哪个项目进行编译，我们使用65号`goblin_fact-userdebug`项目，环境变量信息输出如下：

```
Which would you like? [aosp_arm-eng]  65

============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=9
TARGET_PRODUCT=goblin_fact
TARGET_BUILD_VARIANT=userdebug
TARGET_BUILD_TYPE=release
TARGET_ARCH=arm
TARGET_ARCH_VARIANT=armv7-a-neon
TARGET_CPU_VARIANT=cortex-a9
HOST_ARCH=x86_64
HOST_OS=linux
HOST_OS_EXTRA=Linux-4.15.0-70-generic-x86_64-Ubuntu-16.04.6-LTS
BUILD_ID=PQ3B.190705.003
OUT_DIR=out
复制代码
```

这些环境变量将影响编译过程。我们简单看下变量代表的含义：

- `PLATFORM_VERSION_CODENAME`：表示平台版本的名称，通常是`AOSP（Android Open Source Project）`
- `PLATFORM_VERSION`：Android平台的版本号
- `TARGET_PRODUCT`：所编译产品的名称
- `TARGET_BUILD_VARIANT`：编译产品的类型，可能有`eng`、`user`、`userdebug`
- `TARGET_BUILD_TYPE`：表示编译的类型，包括`release`和`debug`。当选择`debug`时会加入一些调试信息。
- `TARGET_ARCH`：编译目标的CPU架构
- `TARGET_ARCH_VARIANT`：编译目标的CPU架构版本
- `TARGET_CPU_VARIANT`：编译目标的CPU架构代号
- `HOST_ARCH`：编译平台的架构
- `HOST_OS`：编译平台的操作系统
- `HOST_OS_EXTRA`：编译平台操作系统的额外信息
- `BUILD_ID`：`BUILD_ID`会出现在编译的版本信息中，可以利用这个环境变量来定义公司特有标识
- `OUT_DIR`：指定编译结果的输出目录

## `Build系统`的层次关系

执行完`lunch`命令后，就可以使用`make`命令来执行编译`Build`。

我们先从编译的角度描述下Build系统的功能：

- 生成用于

  ```
  刷机
  ```

  的各种

  ```
  image
  ```

  文件。过程中可以进行：

  - 文件拷贝
  - 模块编译
  - 生成配置文件

- 能够根据产品需求灵活配置编译模块

- 能够提供单个模块的编译功能

知道了功能，那我们再来看下`Build系统`是怎么实现这些功能的。

执行`make`命令后的调用流程：

```
graph LR
build/envsetup.sh-make-->build/soong/soong_ui.bash
build/soong/soong_ui.bash-->build/soong/ui/build/kati.go
build/soong/ui/build/kati.go --> build/make/core/main.mk
复制代码
这部分其实和书本上差别挺大的，9.0多了一些东西
```

- 增加kati框架把`Android.mk`转换成ninja file。
- 增加soong框架把`Android.bp`转换成ninja file。
- 增加`Android.bp`是jason形式组织的文件，将来逐步取代`Android.mk`

我们这里明确一点就好，**从`build/make/core/main.mk`开始，通过`include`命令将其它的`.mk`文件包含进来，最终形成一个包括所有编译脚本的集合**。这个集合相当于一个巨大的Makefile文件

虽然Makefile看上去很庞大，其实主要由三种内容构成：

- 变量定义
- 函数定义
- 目标依赖规则

Android的Makefile简要体系如下：

![image](images/17366d83c1b21642)



针对上图橘色标识的三个部分，Build系统会在这里分别引入具体产品的配置文件`AndroidProduct.mk`和`BoardConfig.mk`。以及各模块的编译文件`Android.mk`。

`build/make/core/combo`目录下的mk文件定义了`GCC编译器`的版本和参数。
 `build/make/core/clang`目录下的mk文件定义了`LLVM编译器`clang的路径和参数。

Build系统主要`.mk`文件简介：

| 文件名              | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| `main.mk`           | Android Build系统的主控文件。该文件的主要作用是include其他mk文件，以及定义几个重要的编译目标，如droid、sdk、ndk等。同时检查编译工具的版本，如make、gcc、javac等。 |
| `config.mk`         | Android Build系统的配置文件，主要定义了许多常量来负责不同类型模块的编译，定义编译器参数并引入产品的`BoardConfig.mk`文件来配置产品参数，同时也定义了一些编译工具的路径，如aapt、mkbooting、javajar等。 |
| `envsetup.mk`       | 包含进`product_config.mk`文件，并根据其内容设置编译产品所需要的环境变量，如`TARGET_PRODUCT`、`TARGET_BUILD_VARIANT`、`HOST_OS`等。并检查变量值的合法性，指定编译结果的输出路径。 |
| `product_config.mk` | 包含进了系统中所有的`AndroidProduct.mk`文件，并根据当前产品的配置文件来设置产品编译相关的变量 |
| `product.mk`        | 定义`product_config.mk`文件中使用的各种函数                  |
| `definitions.mk`    | 定义了大量Build系统中使用的函数。                            |
| `dexpreopt.mk`      | 定义与dex优化有关的路径和参数                                |
| `Makefile`          | 定义了系统最终完成编译完成所需要的各种目标和规则             |

==书中5.0的编译脚本和9.0比变化太大。。。。。不深究细节实现了，复杂的事情交给脚本吧，先简单了解主要`.mk`文件的include关系==

# Android的产品配置文件

产品配置文件的作用是按照Build系统的要求，将生成产品的各种image文件所需要的配置信息（如版本号、各种参数等）、资源（图片、字体等）、二进制文件（apk、jar包、so库等）组织起来，同时可以灵活增加或删除一些模块。

通常产品配置文件放在device目录下，而vendor目录下放一些硬件的HAL库和放第三方厂商的定制代码。

## 以goblin为例的配置文件

通常device目录中分为以下几个子目录：

- common：用来存放各个产品通用的配置脚本、文件等。
- sample：一个产品配置的例子。
- generic：存放用于模拟器的产品，包括x86、arm、mips架构
- 厂商产品目录（samsung、amlogic、formovie等）

针对goblin项目，配置文件集中在`device/formovie/goblin`路径下，核心文件包括：`vendorsetup.sh`、`AndroidProduct.mk`、`BoardConfig.mk`。我们逐个看下。

### `vendorsetup.sh`

前面提到过，`vendorsetup.sh`会在初始化编译环境时被`envsetup.sh`文件包含进去。它主要的作用是调用`add_lunch_combo`来添加产品名称。

以goblin为例，它的`vendorsetup.sh`内容如下：

```
add_lunch_combo goblin-eng
add_lunch_combo goblin-user
add_lunch_combo goblin-userdebug
add_lunch_combo goblin_fact-userdebug
复制代码
```

### `AndroidProduct.mk`

`AndroidProduct.mk`会在Build系统的`product_config.mk`文件中被包含进去，这个文件最重要的作用是**定义一个变量`PRODUCT_MAKEFILES`**，这个变量定义的是本配置目录中所有编译的入口文件。

以goblin为例，它的`AndroidProduct.mk`内容如下：

```
ifeq ($(TARGET_PRODUCT),goblin)
PRODUCT_MAKEFILES := $(LOCAL_DIR)/goblin.mk
else ifeq ($(TARGET_PRODUCT),goblin_fact)
PRODUCT_MAKEFILES := $(LOCAL_DIR)/goblin_fact.mk
复制代码
```

根据`lunch`的不同产品执行不同的`.mk`文件。

### `BoardConfig.mk`

`BoardConfig.mk`会被Build系统的`envsetup.sh`包含进去。这个文件主要定义了和设备硬件（包括CPU、WIFI、CPS等）相关的一些参数。我们看下部分编译变量的作用：

- `TARGET_CPU_ABI`：表示CPU的编程接口。比如`armeabi-v7a`。==`ABI`是啥？下面详解==
- `TARGET_CPU_ABI2`：表示CPU的编程接口。
- `TARGET_CPU_SMP`：表示CPU是否为多核CPU。
- `TARGET_ARCH`：定义CPU的架构。比如`arm`。
- `TARGET_ARCH_VARIANT`：定义CPU的架构版本。比如：`armv7-a-neon`
- `TARGET_CPU_VARIANT`：定义CPU的代号。比如：`contex-a9`
- `TARGET_NO_BOOTLOADER`：如果为true，编译出的image不包含bootloader。
- `BOARD_KERNEL_BASE`：装载kernel镜像时的基地址。比如：`0x0`
- `BOARD_KERNEL_OFFSET`：kernel镜像的偏移地址。比如：`0x1080000`
- `BOARD_KERNEL_CMDLINE`：装载kernel时传给kernel的命令行参数。比如：`--cmdline "root=/dev/mmcblc0p20"`
- `BOARD_MKBOOTIMG_ARGS`：使用mkbooting工具生成boot.img时的参数。比如：`--kernel_offset 0x1080000`
- `BOARD_USES_ALSA_AUDIO`：为true时，表示主板声音系统使用的是ALSA架构。
- `BOARD_HAVE_BLUETOOTH`：为true时，表示主板支持蓝牙。
- `BOARD_WLAN_DEVICE`：定义WiFi设备名称。
- `TARGET_BOARD_PLATFORM`：表示主板平台的型号。比如`tl1`
- `TARGET_USERIMAGES_USE_EXT4`：为true时，表示目标文件系统采用EXT4格式。
- `TARGET_NO_RADIOIMAGE`：为true时，表示编译的镜像中没有射频部分。
- `WPA_SPPLICANT_VERSION`：定义 WIFI WPA 的版本。
- `BOARD_WPA_SUPPLICANT_DRIVER`：指定一种`WPA_SUPPLICANT`的驱动。
- `BOARD_HOSTAPD_DRIVER`：指定WiFi热点的驱动。
- `WIFI_DRIVER_FW_PATH_PARAM`：指定WiFi驱动的参数路径。
- `WIFI_DRIVER_FW_PATH_AP`：定义WiFi热点firmware文件的路径。

以上是相对核心的变量，Build系统中的变量实在是太多了。。。。
 我们解释下

#### `ABI`是啥？

> 大部分来自百度百科

首先，我们看下`API`的解释：应用程序接口（Application Programming Interface，API），又称为应用编程接口，就是软件系统不同组成部分衔接的约定。由于软件的规模日益庞大，常常需要把复杂的系统划分成小的组成部分，编程接口的设计十分重要。程序设计的实践中，编程接口的设计首先要使软件系统的职责得到合理划分。良好的接口设计可以降低系统各部分的相互依赖，提高组成单元的内聚性，降低组成单元间的耦合程度，从而提高系统的维护性和扩展性。

`ABI`是Application Binary Interface的缩写，叫做`应用程序二进制接口`。不同于`API` ，`API`定义了源代码和库之间的接口，因此同样的代码可以在支持这个`API`的任何系统中编译 ，然而`ABI`允许编译好的目标代码在使用兼容`ABI`的系统中无需改动就能运行。同时 `ABI`掩盖了各种细节，例如:调用约定控制着函数的参数如何传送以及如何接受返回值；系统调用的编码和一个应用如何向操作系统进行系统调用；以及在一个完整的操作系统`ABI`中，对象文件的二进制格式、程序库等等。

它描述了应用程序与OS之间的底层接口。`ABI`涉及了程序的各个方面，比如：目标文件格式、数据类型、数据对齐、函数调用约定以及函数如何传递参数、如何返回值、系统调用号、如何实现系统调用等。

一套完整的`ABI`（比如：Intel Binary Compatibility Standard (iBCS)），可以让程序在所有支持该`ABI`的系统上运行，而无需对程序进行修改。

### 另外几个重要的`.mk`

在`device/formovie/goblin`路径下还有几个和Build相关的`.mk`

```
goblin_fact.mk    goblin.mk    device.mk
复制代码
```

`goblin_fact.mk`和`goblin.mk`就是我们在`AndroidProduct.mk`中定义的两个入口文件。而`device.mk`主要做下面几个事情：

- 将 kernel 的镜像复制到目标系统里
- 将 Linux 系统的初始化文件和分区表等复制到目标系统里
- 定义系统支持的分辨率
- 指定系统的overlay目录
- 添加一些模块
- 设置系统属性值
- include一些其他配置文件

我们看下部分内容：

```
PRODUCT_DIR := goblin
PRODUCT_NAME := goblin_fact #产品名称
PRODUCT_DEVICE := goblin #设备名称，非常关键
PRODUCT_BRAND := 产品品牌
PRODUCT_MODEL := 产品型号
PRODUCT_MANUFACTURER := 制造商

# 增加编译模块 Settings 和 Launcher
PRODUCT_PACKAGES += \
    Settings \
    Launcher

# 拷贝一个c++库到 vendor/lib/ 下
PRODUCT_COPY_FILES += \
    device/formovie/goblin/files/libstdc++.so:vendor/lib/libstdc++.so

# include 一个 device.mk 脚本
$(call inherit-product, device/formovie/$(PRODUCT_DIR)/device.mk)

########################################################
#                   device.mk                          #
########################################################

# 指定系统的overlay目录
DEVICE_PACKAGE_OVERLAYS := \
    device/formovie/$(PRODUCT_DIR)/overlay

# 设置系统属性值
PRODUCT_PROPERTY_OVERRIDES += \
    persist.sys.usb.config=mtp

# 定义系统支持的分辨率
PRODUCT_AAPT_CONFIG := xlarge hdpi xhdpi tvdpi
PRODUCT_AAPT_PREF_CONFIG := hdpi
复制代码
```

上面一些重要的编译变量解释如下：

- `PRODUCT_COPY_FILES`：将编译目录下的一个文件复制到目标文件系统，如果文件夹不存在，会自动创建。
- `PRODUCT_PACKAGES`：用来定义产品的模块列表，所有在模块列表中定义的模块都会被编译进去。
- `PRODUCT_AAPT_CONFIG`：指定系统支持的屏幕密度类型（dip）。所谓支持，是指在编译时，会将相应的资源文件添加到`framework_res.apk`文件中。
- `PRODUCT_AAPT_PREF_CONFIG`：指定系统实际的屏幕密度类型。
- `DEVICE_PACKAGE_OVERLAYS`：很重要的变量，指定了系统的`overlay`目录。系统编译时会使用`overlay`目录下存放的资源文件替换系统或者末班原有的资源文件。这样在不改动原生资源文件的情况下，就能实现产品的个性化。而且`overlay`的目录可以有多个，它们会按照在变量中的先后顺序来替换资源文件，利用这个特性可以定义公共的`overlay`目录，以及各个产品专属的`overlay`目录，最大限度的重用资源文件。
- `PRODUCT_PROPERTY_OVERRIDES`：定义系统的属性值。如果属性以`ro.`开头，那么这个属性是只读属性。一旦设置，属性将不能改变。如果属性以`persisit.`开头，当设置这个属性时，它的值将写入`/data/property`中。

## 编译类型`eng`、`user`和`userdebug`

### `eng`类型

默认的编译类型。执行`make`相当于执行`make eng`。

编译时会将下列模块编译进系统：

- 在`Android.mk`中用`LOCAL_MODULE_TAGS`定义了标签：`eng`、`debug`、`shell_$(TARGET_SHELL)`、`user`和`development`的模块
- 非APK模块并且不带任何标签的模块
- 所有产品配置文件中指定的APK模块

编译的系统带有下列属性：

- `ro.secure=0`
- `ro.debuggable=1`
- `ro.kernel.android.checkjni=1`

编译的系统中默认情况下`adb`是打开的

### `user`类型

编译时会将下面这些模块编译进系统：

- 在`Android.mk`中用`LOCAL_MODULE_TAGS`定义了标签：`shell_$(TARGET_SHELL)`和`user`的模块
- 非APK模块并且不带任何标签的模块
- 所有产品配置文件中指定的APK模块，同时忽略其标签属性

编译后的系统属性：

- `ro.secure=1`
- `ro.debuggable=0`

编译的系统缺省情况下`adb`是不可用的，需要在设置中手动打开

### `userdebug`类型

编译时会将下列模块编译进系统：

- 在`Android.mk`中用`LOCAL_MODULE_TAGS`定义了标签：`debug`、`shell_$(TARGET_SHELL)`和`user`的模块
- 非APK模块并且不带任何标签的模块
- 所有产品配置文件中指定的APK模块，同时忽略其标签属性

编译后的系统属性：

- `ro.secure=1`
- `ro.debuggable=1`

编译的系统缺省情况下`adb`是不可用的，需要在设置中手动打开

## 产品的`image`文件

Android 编译完成后会生成一个`image`文件，包括：

- `boot.img`：包含通过 `mkbootimg` 组合在一起的内核映像和 RAM 磁盘
- `system,img`：主要包含 Android 框架
- `recovery.img`：用于存储在 OTA 过程中启动的恢复映像。
- ==`cache.img`==：用于存储临时数据，如果设备使用 A/B 更新，则可以不要此分区。
- ==`misc.img`==：供恢复映像使用，存储空间不能小于 4KB。
- `userdata.img`：包含用户安装的应用和数据，包括自定义数据。
- ==`metadata.img`==：如果设备被加密，则需要使用 metadata 分区，该分区的存储空间不能小于 16MB。
- ==`vendor.img`==：包含所有不可分发给 Android 开源项目 (AOSP) 的二进制文件。如果没有专有信息，则可以省略此分区。（应该是第三方厂商的相关数据）
- ==`radio.img`==：包含无线装置映像。只有包含无线装置且在专用分区中包含无线装置专用软件的设备才需要此分区。
- ==`tos.img`==：用于存储 Trusty 操作系统的二进制映像文件，仅在设备包含 Trusty 时使用。

黄色标识是书中未提及的，简单记录下。

### `boot.img`

`boot.img`是Android自定义的文件格式。该格式包括了一个`2*1024`的文件头，头文件后面使用gzip压缩过的kernel映像，后面是一个ramdisk映像，最后是一个载入器程序（可选部分）。

`boot.img`结构如下：

```
- kernel.img
- ramdisk.img
  -/
    - init.rc
    - init
    - etc -> /system/etc
    - system/ (mount point)
    - vendor/ (mount point)
    - odm/ (mount point)
    ...
复制代码
```

`boot.img`以`page`为单位存放数据，编译时通过`BOARD_KERNEL_PAGESIZE` 指定。通常值为2048。

`ramdisk.img`是一个小型文件系统，它包括了初始化Linux系统所需要的全部核心文件。

### `recovery.img`

`recovery.img`相当于一个小型文本界面的Linux系统，它有自己的内核和文件系统。

`recovery.img`的作用是恢复或升级系统。因此，在sbin目录下会有一个recovery程序；同时也包括`adbd`和系统配置文件`init.rc`。不过这几个文件和boot.img中的不相同的。

### `system.img`

`system.img`就是设备中system目录的镜像，里面包含了Android系统的主要目录和文件。目录包括：

```
- system.img
  -/
    - app           #存放一般的apk文件
    - bin           #存放一些Linux的工具，大部分是toolbox的链接
    - etc           #存放系统的配置文件
    - fonts         #存放系统的字体文件
    - framework     #存放系统平台的所有jar包和资源文件包
    - lib           #存放系统的共享库
    - media         #存放系统的多媒体资源，主要是铃声。
    - priv-app      #存放系统核心的apk文件
    - tts           #存放系统的语音合成文件
    - usr           #存放各种键盘布局、时间区域文件
    - vendor        #存放一些第三方厂商的配置文件（现有版本已经拎出来作为一个单独的分区了）
    - xbin          #存放系统管理工具，相当于标准Linux文件系统中的sbin
    - build.prop    #系统属性的定义文件
复制代码
```

### `userdata.img`

`userdata.img`是设备中`/data`目录的镜像，初始时一般不包含任何文件。在Android系统初始化时会在`/data`目录下创建一些子目录和文件。

## 提升编译速度

### 内存

尽可能大的内存

### SSD

SSD提升文件读写速度

### CCCache

编译优化，增加缓存。

```
export USE_CACHE = 1
export CCACHE_DIR=/<path_of_your_choice>/.ccache
prebuilts/misc/linux-x86/ccache/ccache -M 50G
复制代码
```

需要注意的是CCCache并不能提高第一次编译的速度。原理是将一些系统库的编译结果缓存起来，下次编译时如果检测到库没有变化，就直接使用缓存的文件（印象中坑比较多）。

## 编译Android模拟器

操作如下：

```
. build/envsetup.sh
lunch sdk-eng
make
复制代码
```

编译完成后

```
emulator
复制代码
```

对于x86的PC，启动模拟器之前，可以先执行位于SDK的`extras/intel/Hardware_Accelerated_Execution_Manager`下的`intelhaxm`来提升运行速度。

# 编译Android的模块

> **和Android相关的编译变量到底有哪些，都是干啥的，官方讲解在`build/make/core/build-system.html`中。**   迷茫的时候就看官方解释吧。

Android 中的各种模块，无论是apk应用、可执行程序还是jar包，都可以通过Build系统编译生成。在每个模块的源码目录下，都有一个`Android.mk`文件，里面包含了模块代码的位置、模块的名称、需要链接的动态库等一系列的定义。

以Settings为例：

```
# 设置LOCAL_PATH为当前目录
LOCAL_PATH:= $(call my-dir)
# 清除"LOCAL_PATH"外，所有的"LOCAL_"变量
include $(CLEAR_VARS)
######上面这部分每个module直接抄过来就好######

# 指定模块的名称
LOCAL_PACKAGE_NAME := Settings
# 为true时，会使用sdk的hide的api來编译
LOCAL_PRIVATE_PLATFORM_APIS := true
# 指定模块签名使用platform签名
LOCAL_CERTIFICATE := platform
# 为true时表示apk将安装到priv-app下
LOCAL_PRIVILEGED_MODULE := true
# 定义模块的标签为optional
LOCAL_MODULE_TAGS := optional
# 使用AAPT2优化版本
LOCAL_USE_AAPT2 := true

# 指定依赖的共享 Java 类库
LOCAL_JAVA_LIBRARIES := \
    bouncycastle \
    telephony-common \
    ims-common

# 指定依赖的静态 Java 类库
LOCAL_STATIC_JAVA_LIBRARIES := \
    android-arch-lifecycle-runtime \
    android-arch-lifecycle-extensions \
    guava \
    jsr305 \
    settings-logtags \

# 声明调用android的包
LOCAL_STATIC_ANDROID_LIBRARIES := \
    android-slices-builders \
    android-slices-core \
    android-slices-view \
    android-support-compat \
    android-support-v4 \
    android-support-v13 \
    android-support-v7-appcompat \
    android-support-v7-cardview \
    android-support-v7-preference \
    android-support-v7-recyclerview \
    android-support-v14-preference \

# 定义源文件列表
LOCAL_SRC_FILES := \
        $(call all-logtags-files-under, src)

# 指定混淆的标志
LOCAL_PROGUARD_FLAG_FILES := proguard.flags
# 指定编译模块类型为 APK
include $(BUILD_PACKAGE)
# 将源码目录下其余的Android.mk都包含进来
include $(call all-makefiles-under,$(LOCAL_PATH))
复制代码
```

## 模块编译变量

`Android.mk`文件能编译出不同的模块，是通过`include`某个模块编译文件来实现的。比如编译apk时的指令：

```
include $(BUILD_PACKAGE)
复制代码
```

经过一系列的`grep`，发现这些模块编译变量都是在`build/make/core/config.mk`文件中定义的。我们看下部分变量的声明（很多都没见过。。。用到的时候再查吧）：

```
# ###############################################################
# Build system internal files
# ###############################################################

# 清除Local_变量
CLEAR_VARS:= $(BUILD_SYSTEM)/clear_vars.mk
# 产生编译平台使用的本地静态库
BUILD_HOST_STATIC_LIBRARY:= $(BUILD_SYSTEM)/host_static_library.mk
# 产生编译平台使用的本地共享库
BUILD_HOST_SHARED_LIBRARY:= $(BUILD_SYSTEM)/host_shared_library.mk
# 产生目标平台使用的本地静态库
BUILD_STATIC_LIBRARY:= $(BUILD_SYSTEM)/static_library.mk
BUILD_HEADER_LIBRARY:= $(BUILD_SYSTEM)/header_library.mk
BUILD_AUX_STATIC_LIBRARY:= $(BUILD_SYSTEM)/aux_static_library.mk
BUILD_AUX_EXECUTABLE:= $(BUILD_SYSTEM)/aux_executable.mk
# 产生编译平台使用的共享库
BUILD_SHARED_LIBRARY:= $(BUILD_SYSTEM)/shared_library.mk
# 产生目标系统使用的Linux可执行程序
BUILD_EXECUTABLE:= $(BUILD_SYSTEM)/executable.mk
# 产生编译平台下可使用的可执行程序
BUILD_HOST_EXECUTABLE:= $(BUILD_SYSTEM)/host_executable.mk
# 产生apk文件
BUILD_PACKAGE:= $(BUILD_SYSTEM)/package.mk
BUILD_PHONY_PACKAGE:= $(BUILD_SYSTEM)/phony_package.mk
BUILD_RRO_PACKAGE:= $(BUILD_SYSTEM)/build_rro_package.mk
BUILD_HOST_PREBUILT:= $(BUILD_SYSTEM)/host_prebuilt.mk
# 定义预编译模块目标，目的是将预编译模块引入系统
BUILD_PREBUILT:= $(BUILD_SYSTEM)/prebuilt.mk
# 定义多个预编译模块目标
BUILD_MULTI_PREBUILT:= $(BUILD_SYSTEM)/multi_prebuilt.mk
# 产生Java共享库
BUILD_JAVA_LIBRARY:= $(BUILD_SYSTEM)/java_library.mk
# 产生Java静态库
BUILD_STATIC_JAVA_LIBRARY:= $(BUILD_SYSTEM)/static_java_library.mk
# 产生用于编译平台下的Java共享库
BUILD_HOST_JAVA_LIBRARY:= $(BUILD_SYSTEM)/host_java_library.mk
BUILD_DROIDDOC:= $(BUILD_SYSTEM)/droiddoc.mk
BUILD_APIDIFF:= $(BUILD_SYSTEM)/apidiff.mk
# 用来将 LOCAL_COPY_HEADERS 变量定义的文件拷贝到 LOCAL_COPY_HEADERS_TO 定义的路径中
BUILD_COPY_HEADERS := $(BUILD_SYSTEM)/copy_headers.mk
BUILD_NATIVE_TEST := $(BUILD_SYSTEM)/native_test.mk
BUILD_NATIVE_BENCHMARK := $(BUILD_SYSTEM)/native_benchmark.mk
BUILD_HOST_NATIVE_TEST := $(BUILD_SYSTEM)/host_native_test.mk
BUILD_FUZZ_TEST := $(BUILD_SYSTEM)/fuzz_test.mk
BUILD_HOST_FUZZ_TEST := $(BUILD_SYSTEM)/host_fuzz_test.mk
复制代码
```

所以 `include $(BUILD_PACKAGE)`其实就是

```
include build/make/core/package.mk
复制代码
```

Google爸爸很贴心哈。

感兴趣的同学可以深入研究下这些`.mk`文件哟。

## 常用模块定义实例

### 编译一个 apk 文件

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
# 指定依赖的共享Java库
LOCAL_JAVA_LIBRARIES := 
# 指定依赖的静态Java库
LOCAL_STATIC_JAVA_LIBRARIES := 

# 指定模块的标签
LOCAL_MODULE_TAGS := optional/user/userdebug/eng
# 指定模块的名称
LOCAL_PACKAGE_NAME := apkName
# 指定模块的签名方式
LOCAL_CERTIFICATE := platform

# 不使用代码混淆的工具进行代码混淆
LOCAL_PROGUARD_ENABLED:= disabled
# 使用hide api编译
LOCAL_PRIVATE_PLATFORM_APIS := true
# 安装在priv-app目录下
LOCAL_PRIVILEGED_MODULE := true

# 指定 AndroidManifest.xml 文件路径
LOCAL_MANIFEST_FILE := app/src/main/AndroidManifest.xml
# 指定资源路径
LOCAL_RESOURCE_DIR := $(LOCAL_PATH)/app/src/main/res
# 指定源码路径
LOCAL_SRC_FILES := \
        $(call all-java-files-under, src)

include $(BUILD_PACKAGE)
复制代码
```

### 编译一个Java共享库

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

# 编译模块的标签
LOCAL_MODULE_TAGS := optional/user/userdebug/eng
# 编译模块的名称
LOCAL_MODULE := java_lib_name
# 源码编译路径
LOCAL_SRC_FILES := $(call all-java-files-under,java)

include $(BUILD_JAVA_LIBRARY)
复制代码
```

### 编译一个Java静态库

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

# 编译模块的名称
LOCAL_MODULE := java_static_lib_name
# 源码编译路径
LOCAL_SRC_FILES := $(call all-java-files-under,java)

include $(BUILD_STATIC_JAVA_LIBRARY)
复制代码
```

### 编译一个Java资源包文件

资源部也是一个 apk 文件，但是没有代码。

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

# 使用 AAPT2 打包资源
LOCAL_USE_AAPT2 := true
# 值为true的话，说明手动写依赖的jar包了。无论是系统的还是用户自己的
LOCAL_NO_STANDARD_LIBRARIES := true
# 指定签名类型
LOCAL_CERTIFICATE := platform
# 指定 AAPT 工具参数（现在使用AAPT2优化版本）
LOCAL_AAPT_FLAGS := -x
# 定义模块名字
LOCAL_PACKAGE_NAME := java_res_name_lib
# 定义模块标签为 user
LOCAL_MODULE_PATH := user
# 定义模块编译后的输出路径
LOCAL_MODULE_PATH := $(TARGET_OUT_JAVA_LIBRARIES)
# 值为true时，其他的apk可以引用本模块的资源
LOCAL_EXPORT_PACKAGE_RESOURCES := true
include $(BUILD_PACKAGE)
复制代码
```

### 编译一个可执行文件

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
# 定义源码路径
LOCAL_SRC_FIFLES := Executor.cpp
# 指定模块需要链接的动态库
LOCAL_SHARE_LIBRARIES := libutils libbinder
# 定义编译标志
ifeq ($(TARGET_OS),linux)
    LOCAL_CFLAGS += -DXP_UNIX
endif
# 指定模块的名称
LOCAL_MODULE := executor_name
include $(BUILD_EXECUTABLE)
复制代码
```

### 编译一个native共享库

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional/user/userdebug/eng
LOCAL_MODULE := lib_native_share_name
# 指定需要编译的源文件
LOCAL_SRC_FIFLES := Native.cpp
# 指定模块需要链接的静态库
LOCAL_SHARE_LIBRARIES := \
    libutils \
    libbinder
# 指定模块依赖的静态库
LOCAL_STATIC_LIBRARIES := lib_native_static_name
#指定头文件查找路径
LOCAL_C_INCLUDES += \
    $(JNI_H_INCLUDE) \
    $(LOCAL_PATH)/../include
# 定义编译标志
LOCAL_CFLAGS += -O
include $(BUILD_SHARED_LIBRARY)
复制代码
```

### 编译一个native的静态库

```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional/user/userdebug/eng
LOCAL_MODULE := lib_native_static_name
# 指定模块的源文件
LOCAL_SRC_FIFLES := \
    NativeStatic.cpp
# 指定头文件查找路径
LOCAL_C_INCLUDES:=  \
# 定义编译标志
LOCAL_CFLAGS :=-D_ANDROID_
include $(BUILD_STATIC_LIBRARY)
复制代码
```

## 预编译模块的定义

在实际开发中，有很多apk、文件、jar包等都是预先编译好的。编译系统时需要将这些二进制文件复制到生成的image文件中。

常用的方法是通过`PRODUCT_COPY_FILES`变量将文件复制到生成的image文件中，像这样：

```
PRODUCT_COPY_FILES += \
	$(LOCAL_PATH)/media/autovideo_265.mkv:/system/factory/autovideo_265.mkv 
复制代码
```

但是对于apk、或jar包，需要使用系统的签名才能正常运行；另外一些动态库文件可能是源码中某些模块依赖的。这样用上述的方式就行不通了。

Android 提供了预编译模块的方法来对策这个问题，大体流程如下：

- 通过`LOCAL_SRC_FILES`指定二进制文件（apk或jar包）的路径
- 通过`LOCAL_MODULE_CLASS`指定编译模块的类型
- 通过`include $(BUILD_PREBUILT)`执行预编译

### `LOCAL_MODULE_CLASS` 可以指定哪些类型呢？

首先，Google爸爸说：

```
LOCAL_MODULE_CLASS
Which kind of module this is. This variable is used to construct other variable names used to locate the modules. See base_rules.mk and envsetup.mk.
复制代码
```

那我们就从这两个文件看下

base_rules.mk:

```
EXECUTABLES
SHARED_LIBRARIES
STATIC_LIBRARIES
NATIVE_BENCHMARK
HEADER_LIBRARIES
NATIVE_TESTS
JAVA_LIBRARIES
APPS
复制代码
```

不过`envsetup.mk` 并没有明确的声明。

### 以预编译 apk 文件为例

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE := Factory
LOCAL_SRC_FILES := release/factory_test-release.apk
LOCAL_PRIVILEGED_MODULE := true
LOCAL_MODULE_CLASS := APPS
LOCAL_CERTIFICATE := platform
LOCAL_MODULE_TAGS := optional

include $(BUILD_PREBUILT)
复制代码
```

## 常用的`LOCAL_`变量

> 再次说明，变量含义这部分在官方`build/make/core/build-system.html`都有说明哈，那里的更专业和详细。

上面的模块编译中用到了那么多`LOCAL_`变量，方便查找先把常用的几个记录一下：

| 变量名                          | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| `LOCAL_ASSET_FILES`             | 编译 APK 文件时用于指定资源列表，通常写成`LOCAL_ASSET_FILES += $(call find-subdir-assets)` |
| `LOCAL_CC`                      | 自定义C编译器来替换默认的编译器                              |
| `LOCAL_CXX`                     | 自定义C++编译器来替换默认的编译器                            |
| `LOCAL_CFLAGS`                  | 定义额外的C/C++编译器参数                                    |
| `LOCAL_CPPFLAGS`                | 定义额外的C++编译器参数，不用在C编译器中                     |
| `LOCAL_CPP_EXTENSION`           | 自定义C++源文件的后缀。例如：`LOCAL_CPP_EXTENSION := .cc`。一旦定义，模块中的所有源文件都必须使用改后缀，目前不支持混合使用。 |
| `LOCAL_C_INCLUDES`              | 指定头文件的搜索路径。                                       |
| `LOCAL_FORCE_STATIC_EXECUTABLE` | 如果编译时需要链接的库有共享和静态两种。此变量为true时会优先链接静态库。通常这种情况只会在编译`root/sbin`目录下的应用才会用到，因为它们执行时间比较早，文件系统的其他部分还没有加载 |
| `LOCAL_GENERATED_SOURCES`       | Files that you add to `LOCAL_GENERATED_SOURCES` will be automatically generated and then linked in when your module is built.(书中写的不时很理解，直接上原文) |
| `LOCAL_MODULE_TAGS`             | 定义模块标签，Build系统根据标签来决定哪些模块需要编译。      |
| `LOCAL_REQUIRED_MODULES`        | 指定依赖的模块，一旦当前模块被安装，通过此变量指定的模块也会被安装 |
| `LOCAL_JAVACFLAGS`              | 定义额外的javac编译器的参数                                  |
| `LOCAL_JAVA_LIBRARIES`          | 指定模块依赖的Java共享库                                     |
| `LOCAL_LDFLAGS`                 | 定义连接器ld的参数                                           |
| `LOCAL_LDLIBS`                  | 指定模块连接时依赖的库。如果库文件不存在，并不会引发它们的编译。（`LOCAL_LDLIBS` allows you to specify additional libraries that are not part of the build for your executable or library. Specify the libraries you want in -lxxx format; they're passed directly to the link line. However, keep in mind that there will be no dependency generated for these libraries. It's most useful in simulator builds where you want to use a library preinstalled on the host. The linker (ld) is a particularly fussy beast, so it's sometimes necessary to pass other flags here if you're doing something sneaky. Some examples:`LOCAL_LDLIBS += -lcurses -lpthread`，`LOCAL_LDLIBS += -Wl,-z,origin` |
| `LOCAL_NO_MANIFEST`             | 为true时，表示软件包中没有AndroidManifest.xml。（资源包中常用） |
| `LOCAL_PACKAGE_NAME`            | 指定APP应用名称                                              |
| `LOCAL_PATH`                    | 指定`Android.mk`文件所在目录                                 |
| `LOCAL_POST_PROCESS_COMMAND`    | 在编译host相关模块时，可以用此变量定义一条命令在link完成后执行 |
| `LOCAL_PREBUILT_LIBS`           | 指定预编译C/C++动态和静态库列表，用于预编译模块              |
| `LOCAL_PREBUILT_JAVA_LIBRARIES` | 指定预编译Java库列表，用于预编译模块                         |
| `LOCAL_SHARED_LIBRARIES`        | 指定模块依赖的C/C++共享库列表                                |
| `LOCAL_SRC_FILES`               | 指定源文件列表                                               |
| `LOCAL_STATIC_LIBRARIES`        | 指定模块依赖的C/C++静态库列表                                |
| `LOCAL_MODULE`                  | 除apk模块使用`LOCAL_PACKAGE_NAME`指定模块名外，其余的模块都使用`LOCAL_MODULE`指定模块名 |
| `LOCAL_MODULE_PATH`             | 指定模块在目标系统的安装路径。可执行文件和动态库模块一般会搭配`LOCAL_UNSTRIPPED_PATH`使用 |
| `LOCAL_UNSTRIPPED_PATH`         | 指定模块的unstrpped版本在out目录下保存的路径                 |
| `LOCAL_ADDITIONAL_DEPENDENCIES` | 模块需要依赖其他任何实际未内置的模块，则可以将这些make目标添加到 `LOCAL_ADDITIONAL_DEPENDENCIES`。通常，这是针对其他一些不会自动创建的依赖项的解决方法。 |
| `LOCAL_MODULE_CLASS`            | 定义模块的分类。根据分类，生成的模块会被安装到对应的目录下。例如：APPS，安装到/system/app；`SHARED_LIBRARIES`，安装到/system/lib；EXECUTABLE，安装到/system/bin下；ETC，安装到/system/etc；但是如果同时使用`LOCAL_MODULE_PATH`定义了路径，则安装到该路径。 |
| `LOCAL_MODULE_NAME`             | 指定模块名                                                   |
| `LOCAL_MODULE_SUFFIX`           | 指定当前模块的后缀。一旦指定，系统在产生目标文件时，会以模块名+后缀来创建 |
| `LOCAL_STRIP_MODULE`            | 指定模块是否需要strip，一般在编译可执行文件和动态库时才使用该变量 |

# Android中的签名

在Android系统中，所有安装到系统中的apk都需要签名，所谓签名就是给应用附加一个数字证书。虽然数字证书有很多用途，但是在Android中，它唯一的作用就是表明制作者的身份。

## Android应用签名方法

### 创建用于签名的证书文件

Android 使用Java工具keytool生成数字证书，命令如下：

```
keytool -genkey -v -keystore lee.keystore -alias lee.keystore -keyalg RSA -validity 30000
复制代码
```

参数解释：

- `-keystore lee.keystore` 表示证书文件名
- `-alias lee.keystore` 表示证书的别名
- `-keyalg RSA` 表示采用的加密算法
- `-validity 30000` 表示证书的有效期

执行命令后，终端提示需要设置输入的信息如下：

- 首先，需要设置一个`keystore`的密码，并重复输入确认。
- 然后输入各种个人信息，输入`yes`确认信息。
- 确认完个人信息后需要设置一个`key`的密码，并重复输入确认。

#### **为什么有两个密码？**

- 第一个设置的密码是存放数字证书的容器(`keystore`)的密码。
- 第二个设置的密码是新创建的数字证书的密码，在签名时使用。

用指令查看下刚才创建的证书容器：

```
keytool -list -keystore lee.keystore
复制代码
```

此时，就需要输入第一个设置的密码了，输入后打印如下：

```
lijie@leeMBP:~$ keytool -list -keystore lee.keystore
输入密钥库口令:  
密钥库类型: jks
密钥库提供方: SUN

您的密钥库包含 1 个条目

lee.keystore, 2020-7-19, PrivateKeyEntry, 
证书指纹 (SHA1): BD:6A:85:18:22:34:91:25:F9:38:1F:CB:A2:BA:EC:91:78:3C:67:9F
复制代码
```

### 应用签名步骤

签名命令

```
jarsigner -verbose -keystore lee.keystore -signedjar android-release-signed.apk android-release-unsigned.apk lee.keystore
复制代码
```

- `-verbose` 表示要打印签名过程
- `-keystore` 用来指定 keystore 文件
- `-signedjar` 指定签名后的apk名称
- `android-release-unsigned.apk` 待签名文件
- 最后`lee.keystore`是证书的别名

执行过程遇到了如下错误：

```
lijie@leeMBP:~/Desktop$ jarsigner -verbose -keystore ~/lee.keystore -signedjar Feng-Factory-signed.apk Feng-Factory.apk lee.keystore
输入密钥库的密码短语: 
输入lee.keystore的密钥口令: 
jarsigner: 无法对 jar 进行签名: java.util.zip.ZipException: invalid entry compressed size (expected 26171 but got 26705 bytes)
复制代码
```

因为Feng-Factory.apk是被签过名的，此时需要把`META-INF`文件夹删掉（去掉旧的签名文件），再次执行签名，OK了。输出如下：

```
....
>>> 签名者
    X.509, CN=lee, OU=hualee, O=hualee.top, L=PK, ST=PK_HD, C=CN
    [可信证书]

jar 已签名。
警告: 
签名者证书为自签名证书
复制代码
```

签名完成后，我们来查看下签名信息：

```
jarsigner -verify -verbose -certs Feng-Factory-signed.apk 
复制代码
```

会得到如下返回：

```
....
- 由 "CN=lee, OU=hualee, O=hualee.top, L=PK, ST=PK_HD, C=CN" 签名
    摘要算法: SHA-256
    签名算法: SHA256withRSA, 2048 位密钥

jar 已验证。
复制代码
```

经过签名修改后，在应用发布前还需要进行`APK对齐`，命令如下：

```
zipalign -f -v infile.apk outfile.apk
复制代码
```

`APK对齐`第一次接触。。。。。根据书中的描述，是为了提升资源访问效率。

对齐后的文件可以使用如下指令检查：

```
zipalign -c -v 4 test.apk 
复制代码
```

## Android 系统签名

Android 系统签名的方法和应用签名不大一样。在`build/make/target/product/security`目录中存放着4组后缀名为`.x509.pem`和`.pk8`的文件，如下：

```
- security
    - Android.mk
    - README
    - media.pk8
    - media.x509.pem
    - platform.pk8
    - platform.x509.pem
    - shared.pk8
    - shared.x509.pem
    - testkey.pk8
    - testkey.x509.pem
    - verity.pk8
    - verity.x509.pem
    - verity_key
复制代码
```

- `*.pk8`文件是私钥
- `*.x509.pem` 文件是公钥
- testkey 用于普通 APK
- platform 用于系统核心的 APK
- shared 用于Launcher、contacts等重要 APK
- media 用于系统的多媒体和下载类 APK

### 生成系统的签名文件

需要使用源码中的`development/tools/make_key`命令。

通过`make_key`生成系统签名指令如下：

```
make_key testkey '/C=CN/ST=PK/L=OK_HD/O=hualee/OU=hualee.top/CN=lee/emailAddress=lee@mail.com'
复制代码
```

- 第一个参数是key的名字，包括`media`、`platform`、`shared`、`testkey`
- 第二个参数是key的相关信息，C=国家、ST=省、L=地区、O=组织、OU=组织单元、CN=姓名

通过`make_key`生成的签名文件，
 如果所有产品都想使用同一种签名，可以直接用这些文件覆盖掉`build/make/target/product/security`目录下的文件。
 如果不同产品还需要不同的签名，可以新建一个目录，通过编译变量`PRODUCT_DEFAULT_DEV_CERTIFICATE`来指定。例如：

```
PRODUCT_DEFAULT_DEV_CERTIFICATE := # your security files path
复制代码
```

而在Build系统编译时，签名是自动完成的。Build中的签名是通过`signapk.jar`完成的，指令如下：

```
java -jar signapk.jar platform.x509.pem platform.pk8 test.apk test_signed.apk
复制代码
```

### 生成系统签名的 keystore 文件

> 当我们开发系统应用时，为了方便调试，可能需要在AS中添加系统签名文件。

下面记录下如何生成系统签名的`keystore`文件：

- 取源码目录中`build/make/target/product/security` 取`platform.pk8` `platform.x509.pem`放到一个目录下
- 生成`shared.priv.pem`，指令： `openssl pkcs8 -in platform.pk8 -inform DER -outform PEM -out shared.priv.pem -nocrypt`
- 生成`pkcs12`，指令：`openssl pkcs12 -export -in platform.x509.pem -inkey shared.priv.pem -out shared.pk12 -name androiddebugkey`
  - 输入密码，android
  - 确认密码，android
- 生成`debug.keystore`，指令：`keytool -importkeystore -deststorepass android -destkeypass android -destkeystore debug.keystore -srckeystore shared.pk12 -srcstoretype PKCS12 -srcstorepass android -alias androiddebugkey`

# 结语

Android Build系统终于看到了一个大概，虽然Google早就已经使用`Soong`来进行系统构建了，但是Android的历史遗留问题多滴很那。一步一步来吧，原理啥的应该都还是共同的。

写完本篇，了解了Android系统的庞大，以前看不懂的`include`倒也了然了一些，哈哈哈，加油！

奥奥，还有个事情，想知道源码怎么下载么？在图片下面哟：

![image](images/17366d83cec82db1)



- [中科大-AOSP(Android) 镜像使用帮助](https://lug.ustc.edu.cn/wiki/mirrors/help/aosp)
- [清华大学-Android 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)

不谢 (￣_,￣ )


作者：apigfly
链接：https://juejin.cn/post/6854573211045068814
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。