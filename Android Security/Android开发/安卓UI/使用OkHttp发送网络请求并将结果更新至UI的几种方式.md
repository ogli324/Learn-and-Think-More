# 使用OkHttp发送网络请求并将结果更新至UI的几种方式

url：https://www.jianshu.com/p/ed836b1413b9

发送网络请求时我们写大部分安卓项目时无法避免的一环，使用OkHttp库可以很好的帮助我们封装网络请求的底层处理细节，更专注的完成实际业务需求。但是安卓不允许我们直接在UI线程运行网络请求，因为网络请求可能会阻塞UI的响应，因此我们只能开辟新的线程来处理网络请求。那么怎么在网络请求完成后在子线程中更新UI呢？下面我们就来讨论一下几种简单的实现方式。

### 1. 使用AsyncTask + OkHttp同步请求

AsyncTask类是用来在子线程中异步处理一些耗时操作的一个工具类，我们先继承AsyncTask类编写一个处理自己业务需求的子类，然后在其中编写逻辑代码，最后在UI线程中启动AsyncTask即可，如下代码：



```java
private class UserLoginTask extends AsyncTask<String, Void, LoginResult> {

        /**
         * 登录参数
         * @param params params[0] ==> username,
         *               params[1] ==> password
         * @return
         */
        @Override
        protected LoginResult doInBackground(String... params) {
            String username = params[0];
            String password = params[1];
            NetworkHelper networkHelper = NetworkHelper.getInstance();
            try {
                User user = networkHelper.login(username, password);
                //login failed
                if (user == null) {
                    Log.i(TAG, "Username and password do not match");
                    result.setStatus(ResultStatus.AUTH_ERROR);
                } else {
                    Log.i(TAG, "Login success");
                    result.setStatus(ResultStatus.SUCCESS);
                    result.setUser(user);
                }
            } catch (IOException e) {
                Log.e(TAG, "Login failed due to network error", e);
                result.setStatus(ResultStatus.NETWORK_ERROR);
            }
            return result;
        }

        @Override
        protected void onPostExecute(LoginResult loginResult) {
            mProgressDialog.hide();

            if (loginResult.getStatus() == ResultStatus.SUCCESS) {
                // do login success task， update UI
                ...
            } else if (loginResult.getStatus() == ResultStatus.NETWORK_ERROR){
                Toast.makeText(getApplication(), R.string.error_network_fail, Toast.LENGTH_SHORT)
                        .show();
            } else if (loginResult.getStatus() == ResultStatus.AUTH_ERROR) {
                Toast.makeText(getApplication(), R.string.error_incorrect_password, Toast.LENGTH_SHORT)
                        .show();
            }
        }
    }
```

以上代码中UserLoginTask是LoginActivity的一个子类，其中最重要的有两个方法：`doInBackground(String... params)`和`onPostExecute(LoginResult loginResult)`，前者的返回值会传入后者的方法参数中。前者在子线程中执行，因此**不可以在doInBackground中更新UI**，否则会抛出异常，而后者是在主线程中执行的，所以相关UI操作可以在这里进行。
 然后在Activity中为LoginButton设置监听，在点击时创建并启动LoginTask：



```java
mLoginButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                login();
            }
        });
...
private void login() {
        String username = mUsernameEditText.getText().toString();
        String password = mPasswordEditText.getText().toString();
        if (username.isEmpty() || password.isEmpty()) {
            Toast.makeText(getApplication(), R.string.error_empty_username_or_password, Toast.LENGTH_SHORT)
                    .show();
            return;
        }
        //如果登录任务还未完成，防止创建重复的登录任务
        if (mLoginTask != null) {
            return;
        }

        mProgressDialog.show();

        mLoginTask = new UserLoginTask();
        mLoginTask.execute(username, password);
}
```

其中最后一行代码`mLoginTask.execute(username, password)`就是在子线程中执行LoginTask中方法。
 以上仅为创建子线程任务的部分，而其中`networkHelper.login(username, password)`调用中才是真正执行OkHttp请求的地方，因为AsyncTask已经是子线程了，所以在发送OkHttp请求时就不需要使用异步请求，发送同步请求就可以了，下面给出其简单代码（以post请求为例）：



```java
public User login(String username, String password) throws IOException {
        OkHttpClient client = new OkHttpClient();
        String requestBody = "{\"username\": \"" + username + "\", \"password\": \"" + password + "\"}";
        String res;
        RequestBody body = RequestBody.create(JSON, requestBody);
        Request request = new Request.Builder()
                .url(baseUrl + path)  //你的请求URL
                .post(body)
                .build();
        Response response = client.newCall(request).execute();
        if (response.isSuccessful()) {
            res = response.body().string();
        } else {
            throw new IOException("Unexpected code " + response);
        }
        //用户名或密码错误，返回空字符串
        if (res == null || res.trim().isEmpty()) {
            return null;
        }
        return gson.fromJson(res, User.class);
    }
```

OkHttp的使用非常简单，创建client，创建request，然后调用client.newCall(request).execute()就可以得到response，然后对其进行处理即可，详情可参考官方文档，这里不再赘述。

#### 2. 使用OkHttp的异步请求

异步OkHttp请求可以是我们不需要再编写自己的AsyncTask了，OkHttp会自动在子线程中执行网络请求，并在请求成功或失败后回来调用相应的回调方法，如下：



```java
public void asyncGetStudent(final Activity activity, int groupId, final NetworkCallback<User> callback) {
        String authToken = getAuthToken(activity);
        String path = "/group/" + groupId + "/students";
        Request request = buildGetRequest(path, authToken);
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                callback.onGetFail(e);
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if (!response.isSuccessful()) {
                    onFailure(call, new IOException("Unexpected Code: " + response));
                } else {
                    String responseJson = response.body().string();
                    User[] students = gson.fromJson(responseJson, User[].class);
                    callback.onGetSuccess(students);
                }
            }
        });
    }
```

异步请求与同步请求的主要区别在于调用`clinet.newCall(request).enqueue(Callback)`而非execute()方法，然后在`onFailure`和`onResponse`中处理结果。
 为了集中处理网络请求，我们依然将所有网络请求的代码放在了NetworkHelper类中，然后在Activity中调用其中的方法，以上代码中的NetworkCallback<T>回调接口如下：



```java
public interface NetworkCallback<T> {
    void onGetSuccess(T[] resultList);
    void onGetFail(Exception ex);
}
```

该回调接口在网络请求完成后调用，其实现定义在调用网络请求的Activity或Fragment中：



```java
protected void onCreate(Bundle savedInstance) {
    NetworkHelper.getInstance().asyncGetStudent(this, groupId, new NetworkCallback<User>() {
                @Override
                public void onGetSuccess(User[] resultList) {
                    mStudents = resultList;
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            updateUI();
                        }
                    });
                }

                @Override
                public void onGetFail(Exception ex) {
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            Toast.makeText(StudentListActivity.this, R.string.error_network_fail, Toast.LENGTH_SHORT)
                                    .show();
                        }
                    });
                }
}
```

在上面的代码片段中，我们在拿到结果resultList后，没有直接使用它来更新UI，而是调用了一个runOnUIThread(Runnable runable)方法，这是为什么呢？
 **因为回调方法的执行依然是在子线程中的，所以回调方法中依然不能更新UI！**，这里我们使用一个简单方便的调用`runOnUiThread`来更新UI，该方法接受一个Runnable对象，只要在Runnable中更新UI就可以了。



