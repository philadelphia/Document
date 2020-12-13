OkHttp 源码分析

基于OkHttp3.11.0

```
implementation("com.squareup.okhttp3:okhttp:3.11.0")
```

网络框架的核心思想基本都是：

1. 构建基本的执行单元之后
2. 根据任务类型放入对应的任务队列里
3. 再由线程池去执行。

OKHttp也不例外。



# 初始化OKHttpClient

OKHttpclient采用了Builder模式创建

```java
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .readTimeout(5, TimeUnit.SECONDS)
        .writeTimeout(5, TimeUnit.SECONDS)
        .build();
```

# 构建执行单元

## 1：GET请求

```java
Request request = new Request.Builder().url("www.baidu.com")
        .get() // 默认就是get请求
        .build();
```



## 2: POST请求

POST与GET多了一个body。

```java
FormBody body = new FormBody.Builder()
        .add("key", "value")
        .add("key1", "value1")
        .build();

Request post = new Request.Builder().url("www.baidu.com")
        .post(body)
        .build();
```



# 执行

## 1：构建Call对象

```
Call call =	okHttpClient.newCall(request)
```

这里Request就是请求单元,不论是同步执行还是异步执行，都调用okHttpClient.newCall(request) 生成了一个RealCall对象，然后分别调用该对象的execute(同步执行)方法或者enqueue(异步执行)方法。

```Java
@Override public Call newCall(Request request) {
  return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

OKHttp的newCall方法调用RealCall的静态方法newRealCall().

```Java
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
  // Safely publish the Call instance to the EventListener.
  RealCall call = new RealCall(client, originalRequest, forWebSocket);
  call.eventListener = client.eventListenerFactory().create(call);
  return call;
}
```

```Java
private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
  this.client = client;
  this.originalRequest = originalRequest;
  this.forWebSocket = forWebSocket;
  this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
}
```

可以看到，newRealCall方法将client作为自己构造函数的参数传入,即每个Call对象都持有Client的引用。

## 2：执行

### 1：同步执行

```Java
try {
    Response response = okHttpClient.newCall(post).execute();
    if (response.code() ==  200){
     	 }
} catch (IOException e) {

}
```

下面这个方法是RealCall对象的execute方法

```
@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  eventListener.callStart(this);
  try {
    client.dispatcher().executed(this);
    Response result = getResponseWithInterceptorChain();
    if (result == null) throw new IOException("Canceled");
    return result;
  } catch (IOException e) {
    eventListener.callFailed(this, e);
    throw e;
  } finally {
    client.dispatcher().finished(this);
  }
}
```

可以看到调用了 client.dispatcher().executed(this);

最中调用 Response result = getResponseWithInterceptorChain();并返回Result对象

如果发生了异常则调用    eventListener.callFailed(this, e);

而最终这会调用    client.dispatcher().finished(this)，来结束这个call。

这里面有一个很重要的类Dispatcher，它是OKHttpclient的一个内部成员，同时OKHttpclient是request的内部成员

```java
public final class Dispatcher {
      private int maxRequests = 64;  //最大的请求数量
      private int maxRequestsPerHost = 5; //同义主机同时支持的最大请求数
      private @Nullable Runnable idleCallback;

      /** Executes calls. Created lazily. */
      private @Nullable ExecutorService executorService;

      /** Ready async calls in the order they'll be run. */
      private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();//准备执行的异步任务队列

      /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
      private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>(); //正在执行的异步任务队列

      /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
      private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>(); //正在运行的同步任务队列
  
  }
```

分别看到，这个类的同步执行方法是

```
synchronized void executed(RealCall call) {
 //将改call假如同步执行队列中
 runningSyncCalls.add(call);
}
```



### 2：异步执行

```java
okHttpClient.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {

    }
});
```

将callBack对象包装成一个AsyncCall对象加入到Dispatcher.

```
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  eventListener.callStart(this);
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

将AsyncCall对象交给Dispatcher对象执行。

```
synchronized void enqueue(AsyncCall call) {
	//首先判断当前的异步任务是否大于最大的异步任务数量（默认64） && 当前call对象的连接主机的访问数量是否大于主机支持的最大访问量（默认是5），如果同时满足以上两个条件，则将任务加到runningAsyncCalls
	如果不满足这两个条件中的任一个，则将异步任务加入到待执行队列里。
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    runningAsyncCalls.add(call);
    executorService().execute(call);
  } else {
    readyAsyncCalls.add(call);
  }
}
```

在RealCall执行enqueue方法时将一个回调对象作为构造参数生成了一个AsyncCall对象，传入了Dispatcher的enqueue方法。

```java
final class AsyncCall extends NamedRunnable {
      private final Callback responseCallback;

      AsyncCall(Callback responseCallback) {
        super("OkHttp %s", redactedUrl());
        this.responseCallback = responseCallback;
      }
  }
```

Async对象其实继承了NamedRunnable。而NamedRunnable实现了Runnable接口，可以看出其实Async其实就是一个命名后的runnable对象。

```
client.dispatcher().enqueue(new AsyncCall(responseCallback));
```

加入到Dispatcher中，最后交给Dispatcher中的executorService来执行该任务。

最终调用AsyncCall对象的execute方法

```
@Override protected void execute() {
    boolean signalledCallback = false;
    try {
      Response response = getResponseWithInterceptorChain();
      if (retryAndFollowUpInterceptor.isCanceled()) {
        signalledCallback = true;
        responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
      } else {
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      }
    } catch (IOException e) {
      if (signalledCallback) {
        // Do not signal the callback twice!
        Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
      } else {
        eventListener.callFailed(RealCall.this, e);
        responseCallback.onFailure(RealCall.this, e);
      }
    } finally {
      client.dispatcher().finished(this);
    }
  }
}
```

可以看出不管是同步执行还是异步执行，最终都是调用getResponseWithInterceptorChain();

方法得出Response

```Java
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(retryAndFollowUpInterceptor);
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(forWebSocket));

  Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
      originalRequest, this, eventListener, client.connectTimeoutMillis(),
      client.readTimeoutMillis(), client.writeTimeoutMillis());

  return chain.proceed(originalRequest);
}
```

在这个方法里首先将用户添加到OKHttpClient上的应用拦截器添加到拦截器列表里。然后一次添加重试重定向拦截器, 头部信息处理拦截器，缓存拦截器， 连接拦截器，然后再加上OKhttpclient配置的网络拦截器，最后添加最终发起请求的拦截器。

拦截器这里使用了不完整的责任链模式，依次处理返回的response，最后返回最终的response。