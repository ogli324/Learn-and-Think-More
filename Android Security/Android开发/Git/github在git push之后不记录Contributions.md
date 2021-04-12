# github在git push之后不记录Contributions

url：https://blog.csdn.net/hanchao5272/article/details/79224161

参考github官方帮助文档：Changing author info

1.产生原因分析
最近发现github在git push之后不记录Contributions[贡献]，这种情况只在单位的开发机上出现，在家里的开发机上完全正常。
通过查询资料，发现原来是因为单位开发机上git配置的email不正确。
1. 通过git config查看配置的邮箱

```
$ git config --global --list
user.name=zhangsan
user.email=wrong_email@163.com
```

这个邮箱地址应该与`github`中`Public profile`配置中的`Public email`一致。

2. 还可以通过`git log`进行确认 

```
$ git log
commit 1908f7703a96e4df3a85c2b6e8ea179eb6112f8a
Merge: 40d9ff0 2f96407
Author: zhangsan <wrong_email@163.com>
Date:   Wed Jan 31 15:39:21 2018 +0800
...
```

2.解决方案
假设前提：
- 假设我叫张三，github用户名是zhangsan，当前的email是wrong_email@163.com，正确的email是right_email@163.com。
- 假设这个项目的名字是mydemo，位于IdeaProjects目录下。解决流程如下：

1.克隆裸库：通过git clone --bare克隆裸库（bare repository：只记录版本库历史记录的版本库）

```
//返回上一级目录
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects/mydemo (master)
$ cd ..

//创建baredemo目录，用于存放裸库
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects
$ mkdir baredemo

//查看目录情况
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects
$ ls
mydemo  baredemo

//进入baredemo目录
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects
$ cd baredemo/

//克隆裸库
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects/baredemo
$ git clone --bare git@github.com:zhangsan/mydemo
Cloning into bare repository 'mydemo.git'...
remote: Counting objects: 129, done.
remote: Compressing objects: 100% (65/65), done.
Receiving objects:  32% (42/129)
Receiving objects: 100% (129/129), 17.77 KiB | 0 bytes/s, done.
Resolving deltas: 100% (21/21), done.

//进入裸库
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects/baredemo
$ cd mydemo.git/
```

**2.创建并执行脚本修改作者信息的文件**

```
//创建脚本文件
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects/baredemo/baredemo.git (BARE:master)
$ touch refactor_email.sh
```

```
然后将下列内容拷贝到`refactor_email.sh`中，注意修改`OLD_EMAIL[错误邮箱]`、`CORRECT_NAME[用户名]`和`CORRECT_EMAIL[正确邮箱]`

```

```
#!/bin/bash
git filter-branch --env-filter '
OLD_EMAIL="wrong_email@163.com"
CORRECT_NAME="zhangsan"
CORRECT_EMAIL="right_email@163.com"
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
export GIT_COMMITTER_NAME="$CORRECT_NAME"
export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
export GIT_AUTHOR_NAME="$CORRECT_NAME"
export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi' --tag-name-filter cat -- --branches --tags
```

保存并执行脚本文件

```
//执行脚本文件
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects/baredemo/baredemo.git (BARE:master)
./refactor_email.sh
```

**3.重新提交版本信息**

```
//确认是否已经修改正确
$ git log
commit 1908f7703a96e4df3a85c2b6e8ea179eb6112f8a
Merge: 40d9ff0 2f96407
Author: zhangsan <right_email@163.com>
Date:   Wed Jan 31 15:39:21 2018 +0800
...

//重新提交版本信息
git push --force --tags origin 'refs/heads/*'
```

**4.删除裸库**

```
//退出裸库
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects/baredemo/baredemo.git (BARE:master)
$ cd ..

//退出裸库目录
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects/baredemo
$ cd ..

//删除裸库目录
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects
$ rm -rf baredemo/

//查看目录情况
zhangsan@DESKTOP-FG4HTMB MINGW64 ~/IdeaProjects
$ ls
mydemo
```

**5.修改git配置信息**

```
//将邮箱修改正确
git config --global user.name "right_email@163.com"

//查看配置情况
$ git config --global --list
user.name=zhangsan
user.email=right_email@163.com
```

