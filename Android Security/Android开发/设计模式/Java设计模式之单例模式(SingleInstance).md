# Java设计模式之单例模式(SingleInstance)

url：https://blog.csdn.net/u011897062/article/details/80700105



模式定义
单例：保证一个类仅有一个实例，并提供一个访问它的全局访问点。

需求背景
在App进程中保证类的实例唯一性，例如数据库访问入口等。
注意：构造函数可见性为private。这样使得外面的类不能通过引言来产生对象。
构造器（Constructor）声明为private的话，外面不能实例化，典型的单例模式和一些工具类（提供静态方法），都把构造器声明为private。所以要得到这个类的实例只能是类名.method()取的它实例。

具体实现
1 自己实现的单例模式

```
public class Singleton{  
//单个实例声明为私有
private static SingleInstance mInstance;
.......
//构造方法声明为私有
private SingleInstance(){
}
......
//获取单例的公共入口
    public static SingleInstance getInstance() {  
       if (mInstance == null) {  
           //保证只有创建实例的时候才加锁，线程安全模型
           synchronized (SingleInstance.class) {  
               if (mInstance == null) {  
                   mInstance = new SingleInstance();  
               }  
           }  
       }  
       return mInstance;  
    }  
}
```

**2 Android中提供的单例模式模板**

```
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```

使用，例如获取ActivityManager的代理对象

```
public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```

