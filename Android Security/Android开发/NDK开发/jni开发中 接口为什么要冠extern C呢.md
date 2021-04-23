# jni开发中 接口为什么要冠extern "C"呢

url：https://blog.csdn.net/waterseason/article/details/84471921

C++ 的代码里面：
extern "C"{
。。。
}
这是因为生成的二进制文件中，C和C++的符号表不相同造成的。Jni是按照C的生成规则去找函数的， 所以要加上extern C使编译器把函数按照C的规则编译 这样才能被JAVA调用



url：https://blog.csdn.net/z531575209/article/details/100586904

android studio jni开发默认是C++语言的,而且还都是静态注册



C++为了支持函数重载，函数在被C++编译后在符号库中的名字与C语言的不同。假如某个函数的原型为void f(int x, int y);该函数被C编译器编译后在符号库中的名字为_f，而C++编译器则会产生_f_int_int之类的名字。C++就是靠这种机制来实现函数重载的。 



被extern “C”修饰的函数或者变量是按照C语言方式编译和链接的，所以可以用一句话来概括extern “C”的真实目的是实现C++与C的混合编程。我想编译的时候用的编译器是C++编译器一个编译器吧,只是编译器支持多种不同的编译模式罢了,这个不是重点.

那么,jni通过这种方式支持重载的java native函数吗?特地做了个试验:

```
public class Config {
    static {
        System.loadLibrary("native-lib");
    }
    public static native void init(int code);
    public static native void init(int code,String name);
    public static native String getSign(String json);

}

extern "C" JNIEXPORT void JNICALL
Java_com_bbc_playnativec_Config_init(JNIEnv *env, jclass clazz, jint code) {
    std::string hello = "bbc config init, C++ pppp222";
    LOGE("code=%d,%s", code, hello.data());
}
```

发现C++中函数出现红线,提示错误不能编译了.说到底,jni入库还是C语言的,但本地的其他功能代码可以使用C++语言的特性来实现.
