---
title: "开源库之 OkHttp 源码分析"
date: 2018-09-11T22:13:45+08:00
subtitle: ""
tags: ["OkHttp"]
categories: ["Android"]
toc: true
draft: false
original: true
addwechat: true
notoc: true
 
slug: "okhttp-analysis"
author: "glumes"
---

分析一波 OkHttp 的源码实现。


<!--more-->

## 简单使用

官方给出了使用例子，具体详情参考 [官网](http://square.github.io/okhttp/)。

```java
// 创建 OkHttp 请求客户端
OkHttpClient client = new OkHttpClient();
// 构建一个请求
Request request = new Request.Builder()
      .url(url)
      .build();
// 执行网络请求并返回结果      
Response response = client.newCall(request).execute();      
// 得到结果内容
response.body().string();
```

可以看到，使用过程还是很容易理解的：

> 首先创建请求客户端 -> 接着创建网络请求 -> 然后发出网络请求 -> 最后得到请求结果。

把网络请求过程抽象化之后就是上面的流水线步骤了，其中最重要的步骤就是发出网络请求了，在 OkHttp 中用到了责任链模式，对发出的网络请求会按照责任链表的顺序依次进行处理，比如请求重试、连接处理、缓存处理、日志处理等，这个地方再写一篇文章详细分析。


下面针对 OkHttp 的调用流程和结构作分析。

## 创建请求

首先是创建请求客户端 OkHttpClient 和网络请求 Request 。

对于这种要创建某某对象的，上来就是一个建造者模式，建造者模式可以说是在开源项目中最常见的设计模式了。

在 OkHttpClient 和 Request 内部都有一个 `Builder` 的内部类用来执行具体的构建操作，一般 `Builder` 类有很多方法，它们都是用来设置具体构建参数的，顺着哪些方法就可以找到都有哪些配置选项，这些选项也就是对外暴露的接口，这算是阅读源码的一个小技巧了。

OkHttpClient 的 Builder 内部配置选项：

```java
      dispatcher = new Dispatcher();
      // 使用默认的选项
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
```

重点分析 `Dispatcher` 类用来管理和分发请求的，其他的如果没有指定的话都是默认选项，等用到时再具体分析。

在 Request 内部的 Builder 也有一些配置选项：

```java
    HttpUrl url;
    String method;
    Headers.Builder headers;
    RequestBody body;
    /** A mutable map of tags, or an immutable empty map if we don't have any. */
    Map<Class<?>, Object> tags = Collections.emptyMap();
```

像 `url`、`body`、`method` 这些都是 HTTP 请求相关的内容，用一个 `Request` 类将它们进行封装，到具体执行请求时，都会将它们取出来使用。

## 同步执行

```kotlin
val response = client.newCall(request).execute()
```

同步执行的代码如上，首先是 `newCall` 方法将 request 转换成 `RealCall` 对象，它实现了 `Call` 的接口。

`Request` 类只是封装了请求的信息，比如 `GET`、`POST` 方法之类的，但具体的执行通过 `Call` 这一层来实现了。

```java
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```

然后是 Call 的 `execute` 方法。

```java
  @Override public Response execute() throws IOException {
    // 保证线程安全
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      // 把请求添加到队列中
      client.dispatcher().executed(this);
      // 具体执行网络请求
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      // 请求失败的回调
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      // 将请求从队列中移除
      client.dispatcher().finished(this);
    }
  }
```

首先是 `synchronized` 加锁保证线程安全，然后是通过 `Dispatcher` 类的 `executed` 方法将请求 `RealCall` 添加到请求的同步队列中去。

```java
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

synchronized void executed(RealCall call) {
  runningSyncCalls.add(call);
}
```

`runningSyncCalls` 是一个 `Deque` 类型的队列。`Deque` 是一个双端队列，它具有队列和栈的特性，队列中的元素可以从两端弹出，而插入和删除操作只能在队列的两端进行。

`executed` 方法只是将请求添加到了队列中，具体的执行在 `getResponseWithInterceptorChain` 方法中，在这里就会将请求通过OkHttp 的各种拦截器，按照责任链模式进行调用，并得到请求返回的结果，用 `Response` 类封装。

当请求结束后，会调用 `Dispatcher` 类的 `finished` 方法。

```java
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }
  
  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      // 将请求从队列中移除
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }  
```

`finished` 方法主要是将请求 `RealCall` 从队列中 `runningSyncCalls` 移除。

这里还有个小知识点就是 `try`、`catch`、`finally` 三者的执行顺序，在 `try` 语句里面执行了 `return` 语句，但是返回的结果会保存在一个临时区域里面，然后执行 `finally` 语句，移除队列请求后再继续返回。

## 网络请求

下面重点看一下 `getResponseWithInterceptorChain` 方法。

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    // OkHttpClient 提供的默认拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    // 重点参数是 0 ，表示责任链的开始
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

有一个拦截器 `Interceptor` 的集合，在创建 OkHttpClient 的时候可以通过 `Builder` 去添加我们自定义的拦截器，另外 OkHttp 也提供了几个默认的拦截器。

把拦截器添加到集合中，作为 `RealInterceptorChain` 类的参数，它实现了 `Chain` 接口，表示为一条`链`。在这条`链`上，每个拦截器都是它的一个节点，并且以链上任何一个拦截器为起点，又可以开始一条新的链。

具体来看看 `proceed` 方法：

```java
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }
  
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;
    // 删除一些判断和抛出异常的方法
    
    // Call the next interceptor in the chain.
    // 以拦截器队列中的下一个拦截器为起点，构建新的请求链
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    // 取出当前请求链中的第一个
    Interceptor interceptor = interceptors.get(index);
    // 执行当前拦截器的功能，并且开始子链的请求流程
    Response response = interceptor.intercept(next);

    // 删除一些判断和抛出异常的方法
    return response;
  }
```

整个拦截器的集合 `interceptors` 是不会变的，而 `index` 就对应从拦截器集合中取出拦截器的索引，为 0 表示取出第一个来作为整条 `链` 的起点，从而构建一条请求链。

当一个拦截器执行具体功能时，也就是 `intercept` 方法，会把下一个`链`作为参数传递过去，这样就又会以这条 `链` 为起始继续下一步的执行。

比如，看 `HttpLoggingInterceptor` 内部的实现:

```java
    Request request = chain.request();
    if (level == Level.NONE) {
      return chain.proceed(request);
    }
```

如果要打印的日志即为为 `NONE`，也就是不打印，直接就开始 `链` 的下一个执行了。

简单地看，这里就是一个递归的调用流程。

## 异步执行

```kotlin
client.newCall(request).enqueue(object : okhttp3.Callback {

    override fun onFailure(call: okhttp3.Call?, e: IOException?) {
    }
    
    override fun onResponse(call: okhttp3.Call?, response: okhttp3.Response?) {
    }
})
```

异步请求的代码如上，和同步请求的区别在于后面是 `enqueue` 方法。

```java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    // 通过 Dispatcher 管理请求
    // 
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

`enqueue` 需要传入一个 `Callback` 的回调接口，用来处理请求成功和失败。另外，在 `enqueue` 的内部还是通过 `Dispatcher` 进行请求的管理。


这里传入的参数是 `AsyncCall` 类，不再是 `RealCall` 类了，`AsyncCall` 类继承自 `NamedRunnable` 类，`NamedRunnable` 实现了 `Runnable` 接口。

`NamedRunnable` 意思就是有名字的线程，会在线程执行时临时改变线程的名字，执行结束后再改回来。

```java
  @Override public final void run() {
    // 保存当前线程的名字
    String oldName = Thread.currentThread().getName();
    // 设置新的名字
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      // 执行后再改为原来的名字
      Thread.currentThread().setName(oldName);
    }
  }
```

另外，`AsyncCall` 实现了 `NamedRunnable` 的抽象方法 `execute` ，该方法也是网络请求的具体执行部分，问题在于这些异步请求是如何分发和管理的，还是回到 `Dispatcher` 类中来。

```java
  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```

首先是判断异步请求队列 `runningAsyncCalls` 的个数是否超过最大请求数了，另外是判断同一主机的网络请求个数是否超过限制。

如果都不超过，就将请求添加到异步请求队列中，并在线程池中去执行。

如果超过了，就把请求添加到准备就绪的队列 `readyAsyncCalls` 中，等待后续再去执行。

`executorService()` 方法的执行就是初始化线程池的。

```java
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```

`ThreadPoolExecutor` 用来提供一个线程池，它的方法原型如下：

```java
    public ThreadPoolExecutor(int corePoolSize,             // 核心线程池的大小
                              int maximumPoolSize,        // 线程池的最大大小
                              long keepAliveTime,         // 线程池空闲时，线程的存活时间
                              TimeUnit unit,             // 存活时间的时间单位
                              BlockingQueue<Runnable> workQueue, // 存放线程任务的队列
                              ThreadFactory threadFactory,      // 创建线程的工厂
                              RejectedExecutionHandler handler) {     // 任务拒绝策略
            // 省略
    }
```

使用线程池的好处是显而易见的，统一管理网络请求，减少线程频繁创建、销毁带来的开销。

通过线程池的 `execute` 方法去执行请求，具体就是 `Runnable` 接口中的 `run` 方法。

```java
    // 异步请求中的方法
    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        // 具体执行的请求还是在这里
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
          // 请求失败的回调
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
```

异步请求执行的线程变了，肯定就得有相应的回调通知，另外，具体是网络请求部分还是在 `getResponseWithInterceptorChain` 方法中。

在 `finally` 中还是会将异步的请求从队列中移除，不同的是内部最后一个参数为 false 。

```java
  void finished(AsyncCall call) {
    // 与同步请求不同的是，最后一个参数为 True, 同步请求为 false
    finished(runningAsyncCalls, call, true);
  }
```

因为 `True`，所以 `promoteCalls` 就会执行。

```java
  // 异步请求才会执行该方法
  private void promoteCalls() {
    // 优先把 readyAsyncCalls 队列中的请求执行完
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }
```

之前提到，如果异步请求队列已经满了，就会把请求放到准备就绪的队列 `readyAsyncCalls` 中去，那么 `promoteCalls` 方法就是来处理 `readyAsyncCalls` 队列中的请求的。

每次从队列中移除异步请求后 size 都会减一，然后再判断队列的数量是否还是超过了最大请求数量，如果是就返回，表示优先把异步队列中的请求消耗完，如果不是，表示有空位，那就从准备就绪的队列中取出请求来执行。

把 `readyAsyncCalls` 队列中的请求放到 `runningAsyncCalls` 队列中去，再通过线程池去执行，并且还是会判断 `runningAsyncCalls` 的数量是否超过了最大请求，超过了就返回，表示 OkHttp 中整个异步请求的数量不能超过 `maxRequests` 表示的数量。

就这样实现了 OkHttp 的异步请求管理流程，并且还有一个类似请求排队的机制，


## 小结

OkHttp 是用来执行网络请求，包括同步和异步的请求。

对于同步请求，直接就执行了，同步阻塞直到返回取得请求结果。

对于异步请求，对请求进行管理，限制请求最大数量，如果超出数量就排队候选，每次执行完一次异步请求，有空位就去处理排队的请求。

对于具体的请求过程，通过责任链的模式，把请求分成多个过程，递归地对请求进行处理。


