# Git pull 强制拉取并覆盖本地代码

url：https://blog.csdn.net/dpengwang/article/details/82821203

两个电脑同时对git上的项目进行跟新时，不免要用到将git上的代码拉取到本地更新本地代码的操作，鉴于自己对git使用的还不是很熟练，所以就直接采取暴力的方法，直接拉取并覆盖本地的所有代码，命令如下

```
git fech --all
git reset --hard origin/master
git pull
```



