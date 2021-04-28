# git error：invalid path问题解决（win下）

url：https://www.cnblogs.com/GyForever1004/p/13702643.html
**背景**

在 windows 上 clone 内核某分支源码报错

**1. 报错详情**

```
1 $ git reset --hard HEAD
2 error: invalid path 'drivers/gpu/drm/nouveau/nvkm/subdev/i2c/aux.c'
```

网络上查了一下大多是由于文件名格式不支持所至，但笔者尝试了一番无果。

**2. 最终解决办法**

```
git config core.protectNTFS false
```

查了下官方手册，官方原话： If set to true, do not allow checkout of paths that would cause problems with the NTFS filesystem 

大概意思是说NTFS有个路径保护机制，防止文件系统出错。