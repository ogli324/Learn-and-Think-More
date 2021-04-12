# 详解Android中OkHttp3的例子和在子线程更新UI线程的方法

url：https://www.jb51.net/article/114397.htm

okHttp用于android的http请求。据说很厉害，我们来一起尝尝鲜。但是使用okHttp也会有一些小坑，后面会讲到如何掉进坑里并爬出来。

首先需要了解一点，这里说的UI线程和主线程是一回事儿。就是唯一可以更新UI的线程。这个只是点会在给okHttp填坑的时候用到。而且，这个内容本身在日常的开发中也经常用到，值得好好学一学。

**okHttp发起同步请求**

第一个列子是一个同步请求的例子。

```
private void performSyncHttpRequest() {
  OkHttpClient client = new OkHttpClient();
 
  Request request = new Request.Builder()
    .url("http://www.baidu.com")
    .build();
  Call call = client.newCall(request);
  Response response = call.execute();
}
```

但是这样的直接在android的主线程里调用一个网络请求的方法是行不通的，直接抛出UI Thread 请求网络的异常。所以我们这里为了可以掩饰要做一点小小的改动。把请求写成同步请求的方式，但是放在一个worker线程里异步的做这个操作。

```
private Handler requestHandler = new Handler() {
  @Override
  public void handleMessage(Message msg) {
    switch (msg.what) {
      case REQUEST_SUCCESS:
        Toast.makeText(MainActivity.this, "SUCCESSFUL", Toast.LENGTH_SHORT).show();
        break;
      case REQUEST_FAIL:
        Toast.makeText(MainActivity.this, "request failed", Toast.LENGTH_SHORT).show();
        break;
      default:
        super.handleMessage(msg);
    }
  }
};
 
private void performSyncHttpRequest() {
  Runnable requestTask = new Runnable() {
    @Override
    public void run() {
      Message msg = requestHandler.obtainMessage();
      try {
        OkHttpClient client = new OkHttpClient();
 
        Request request = new Request.Builder()
            .url("http://www.baidu.com")
            .build();
        Call call = client.newCall(request);
        // 1
        Response response = call.execute();
 
        if (!response.isSuccessful()) {
          msg.what = REQUEST_FAIL;
        } else {
          msg.what = REQUEST_SUCCESS;
        }
      } catch (IOException ex) {
        msg.what = REQUEST_FAIL;
      } finally {
        // send the message
        // 2
        msg.sendToTarget();
      }
    }
  };
 
  Thread requestThread = new Thread(requestTask);
  requestThread.start();
} 
```

所以同步的请求都是这么做的Response response = call.execute();。

1.发起同步请求之前先新初始化一个OkHttpClient。然后是具体的请求，用请求builder来创建这个Request。我们这里为了简单url就是*http://www.baidu.com*了。接下来用前面初始化好的client发起一个call：Call call = client.newCall(request);。最后执行这个call：Response response = call.execute();并获得请求的response。

2.这一部分的可以暂时不要关注。因为这个例子只是为了能以运行起来的方式展示okHttp如何发起同步请求。

**okHttp发起异步请求**

既然android本身不支持发起同步请求，当然也没人要发起同步请求。这么做是能导致严重的用户体验问题。想象一下，如果你有一个瀑布流，然后瀑布流里全部显示的都是图片。现在用户要不断地往下翻看瀑布流的图片。如果这些图片都用同步请求的话，什么时候可以翻一页不说，系统的ANR早就跳出来了。

所以我们就探究一下如何发起一个异步的请求。

```
private void performAsyncHttpRequest() {
  OkHttpClient client = new OkHttpClient();
 
  Request request = new Request.Builder()
      .url("http://www.baidu.com")
      .build();
  Call call = client.newCall(request);
  // 1
  call.enqueue(new Callback() {
    // 2
    @Override
    public void onFailure(Call call, IOException e) {
      //Toast.makeText(MainActivity.this, e.getMessage(), Toast.LENGTH_SHORT).show();
       
      if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
        Log.d(TAG, "Main Thread");
      } else {
        Log.d(TAG, "Not Main Thread");
      }
    }
 
    @Override
    public void onResponse(Call call, final Response response) throws IOException {
      // 3
      if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
        Log.d(TAG, "Main Thread");
      } else {
        Log.d(TAG, "Not Main Thread");
      }
    }
  });
}  
```



1.同步请求用execute方法，异步就用call.enqueue(new Callback()方法。

2.这个Callback接口提供了两个方法，一个是onFailure，一个是onResponse。这两个方法分别在请求失败和成功的时候调用。

3.本来一切都似乎应该很简单。网络请求成功或者失败直接在界面更新了。但是木有想到这样会抛异常。然后看了看发现原来onFailure和onResponse两个方法不是在主线程执行。打印出来的log是：okhttp.demo.com.okhttpdemo D/###okHttp: Not Main Thread。

所以要在主线程中更新view只好想别的办法了。在worker线程里更新主线程会抛异常。一般来说有这么几个方法在子线程里更新view。

**在子线程更新UI线程**

**一、Activity的runOnUiThread方法**

在Activity中有这么一个方法runOnUiThread。这个方法需要一个Runnable实例作为参数。

```
MainActivity.this.runOnUiThread(new Runnable() {
  @Override
  public void run() {
    Log.d(TAG, "code: ");
    Toast.makeText(MainActivity.this, String.valueOf(response.code()), Toast.LENGTH_SHORT).show();
  }
});
```



**二、View的post方法**

View的post方法也是一样，扔一个Runnable的实例进去。然后就在主线程执行了。Toast肯定是没有这个方法的。

```
MainActivity.this.mView.post(new Runnable() {
  public void run() {
    Log.d("UI thread", "I am the UI thread");
  }
});
```



**三、其他**

1.用Handler，这个前面的okHttp同步请求的例子可以用。

2.AsyncTask, 有两个方法可以在主线程中执行：onProgressUpdate和onPostExecute。这里我们并不是要更新进度，所以考虑的是后一个方法。

```
private class BackgroundTask extends AsyncTask<String, Void, Bitmap> {
  protected void onPostExecute(Bitmap result) {
    Log.d("UI thread", "I am the UI thread");
  }
}
```



综合以上，更新UI线程的方法里最后说到的Handler方法和AsyncTask都太重。尤其是AsyncTask。还要继承实现一堆的方法之后才可以能达到目的，同时还和我们要用的okHttp的使用方法很多不兼容的地方。

所以我们只考虑前面的两种。但是两种方法其实是不一样的。当然，这里并不是说方法的名字不一样。我们来看看android的源代码，这两个方法是如何实现的。

```
public final void runOnUiThread(Runnable action) {
  if (Thread.currentThread() != mUiThread) {
    mHandler.post(action);
  } else {
    action.run();
  }
}
// mHandler.post(action); 之post方法的实现
public final boolean post(Runnable r) {
  return sendMessageDelayed(getPostMessage(r), 0);
}
```

方法runOnUiThread最后会调用Handler的sendMessageDelayed。但是这里只delay了0。也就是方法传到这里的时候会立即执行runOnUiThread的参数Runnable实例会立即执行。

下面看看View的post方法：

```
public boolean post(Runnable action) {
  final AttachInfo attachInfo = mAttachInfo;
  if (attachInfo != null) {
    return attachInfo.mHandler.post(action);
  }
  // Assume that post will succeed later
  ViewRootImpl.getRunQueue().post(action);
  return true;
}
```

一般会执行的是ViewRootImpl.getRunQueue().post(action);。也就是Runnable的实例只是添加到了事件队列中，按照顺序执行。并不一定会立即执行。

我们探讨了那么多，最后就使用runOnUiThread来更新界面，也就是方法一了。

```

call.enqueue(new Callback() {
  @Override
  public void onFailure(Call call, IOException e) {
    final String errorMMessage = e.getMessage();
 
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      Log.d(TAG, "Main Thread");
    } else {
      Log.d(TAG, "Not Main Thread");
    }
 
    MainActivity.this.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        Toast.makeText(MainActivity.this, errorMMessage, Toast.LENGTH_SHORT).show();
      }
    });
  }
 
  @Override
  public void onResponse(Call call, final Response response) throws IOException {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      Log.d(TAG, "Main Thread");
    } else {
      Log.d(TAG, "Not Main Thread");
    }
 
    MainActivity.this.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        Log.d(TAG, "code: ");
        Toast.makeText(MainActivity.this, String.valueOf(response.code()), Toast.LENGTH_SHORT).show();
      }
    });
  }
});
```

无论请求成功还是失败，都弹出一个Toast。用MainActivity.this.runOnUiThread在UI线程中弹出这个Toast。

以上就是本文的全部内容，希望对大家的学习有所帮助，也希望大家多多支持脚本之家。

**您可能感兴趣的文章:**

- [android使用handler ui线程和子线程通讯更新ui示例](https://www.jb51.net/article/45528.htm)
- [Android 在子线程中更新UI的几种方法示例](https://www.jb51.net/article/121892.htm)
- [Android编程实现使用handler在子线程中更新UI示例](https://www.jb51.net/article/124666.htm)
- [Android子线程与更新UI问题的深入讲解](https://www.jb51.net/article/157504.htm)