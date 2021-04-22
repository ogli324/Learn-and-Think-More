# Mac配置Maven以及IntelliJ IDEA Maven配置

url：https://www.jianshu.com/p/7c9a5df967d4



## **准备工作**

- maven下载地址：http://maven.apache.org/download.cgi

![img](https:////upload-images.jianshu.io/upload_images/14020633-699947011397f4da.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Maven.png

- **以及咱们的 IntelliJ IDEA**

------

## **安装Maven**

1、下载完以后其实就是一个压缩包，解压即可，解压出来的文件如下图所示，图片里面画线的就是咱们下载的压缩包。



![img](https:////upload-images.jianshu.io/upload_images/14020633-e4b1c0142e846208.png?imageMogr2/auto-orient/strip|imageView2/2/w/838/format/webp)

Maven2.png



2、然后咱们去/usr/local目录下面(可以打开Finder command+shift+G)，把解压出来的文件拷贝到这个目录下面。



![img](https:////upload-images.jianshu.io/upload_images/14020633-08d9069bd075fbca.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Maven3.png

3、敲黑板，重点来了，接下来就是要去配置环境变量，然后去看看 ~/ 目录下面有没有 .bash_profile文件

- 要是没有 咱们就去创建一个



```shell
touch ~/.bash_profile
```

- ok如果有的话 咱们就直接进入



```shell
vi ~/.bash_profile
```

- 之后咱们配置一下环境变量



```shell
export M=$PATH:/usr/local/apache-maven-3.3.9/bin
```

- ok，ESC一下 然后 ： wq保存一下，然后在终端让命令生效一下



```shell
source ~/.bash_profile
```

4、这样我们来测试一下环境变量是否已经配置完成。在终端输入一个 mvn -v。如果出现下图则配置成功，如果没有的话，可以去检查一下 路径是否正确，或者版本号名称之类的。



![img](https:////upload-images.jianshu.io/upload_images/14020633-5d9dae901e8fedf9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

配置信息.png

5、Maven很像一个很大仓库，装了很多的jar包，我们需要的时候就去拿，这就涉及到一个“本地仓库”的问题，默认情况，会在/user/name/.m2下，所以我们需要去更改一下。打开刚刚在/usr/local/apache-maven-3.3.9/conf/settings.xml文件，加入



```xml
/usr/local/apache-maven3.3.9/repository
```

6、然后输入



```shell
mvn help:system
```

你就会发现哗啦啦一堆东西开始下载了



![img](https:////upload-images.jianshu.io/upload_images/14020633-99ddfe76eadf5759.png?imageMogr2/auto-orient/strip|imageView2/2/w/1176/format/webp)

ok.png

如果出现这个信息呢说明你已经下载成功，你可以在**/usr/local/apache-maven3.3.9**这个目录下面发现多了一个文件夹**repository** 里面会有很多的文件 这些文件就是我们从中央仓库下载到本地仓库的文件

------

## **配置IntelliJ IDEA**

1. 打开IntelliJ IDEA->configure->preference->Build->Build Tools->Maven

   ![img](https:////upload-images.jianshu.io/upload_images/14020633-849ed9c858a3f24d.png?imageMogr2/auto-orient/strip|imageView2/2/w/956/format/webp)

   IntelliJ IDEA.png

![img](https:////upload-images.jianshu.io/upload_images/14020633-b49a632eaf632156.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

IntelliJ IDEA2.png

1. 主要配置我上图圈的这三个内容，开始的时候IDEA会用自己的一个Maven，点击后面的按钮开始选择路径，这时候你发现你找不到usr了，你按一下comman+shift+.即可显示隐藏内容，或者直接手动输入 /usr/local/apache-maven3.3.9即可。

1. 在开始之前，将后面两个打钩，第二项内容这边选择setting.xml 即 选择你在上一个路径下面的/usr/local/apache-maven3.3.9/conf下面的settings文件

2. 第三个文件也就是本地仓库文件，在上一步配置Maven的时候我们已经在这个路径下面配置并且下载了一些中央库里面的包，所以选择该路径下面的repository即可

   ![img](https:////upload-images.jianshu.io/upload_images/14020633-b6fbba9c057fdf1c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

   IDea.png

如图所示 这样我们就配置完了IDEA，然后就开始我们的Spring Boot之旅吧。



