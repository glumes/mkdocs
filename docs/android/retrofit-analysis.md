---
title: "开源库之 Retrofit 源码分析"
date: 2018-09-09T14:54:57+08:00
subtitle: ""
tags: ["Retrofit"]
categories: ["Android"]
toc: true
draft: false
original: true
addwechat: true
 
slug: "retrofit-analysis"
author: "glumes"
---

分析一波 Retrofit 的源码实现。

<!--more-->

## 简单使用

官方给出了使用例子，具体详情参看 [官网](https://square.github.io/retrofit/)。

如果加入 RxJava 和 Gson ,就需要添加对应的适配和转换工厂类。

```java
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

围绕 Retrofit 的调用流程分析它的内部实现。

## Retrofit 类的构建

首先是 Retrofit 类的构建，通过建造者模式来实现。

在 `Retrofit.Builder` 类的调用方法中指定一些参数，就比如上面的 RxJava 和 Gson 对应的工厂类。

在 `build` 方法里面可以看到全部的所需要的内容：

```java
    public Retrofit build() {
      // HTTP 请求的 host 路径
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }
      // 用来执行请求的工厂，若没有设置就是使用 OKHttpClient
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }
      // 异步请求时回调线程切换的执行器
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }
      // 网络请求适配器的工厂集合
      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
      // 数据转换器的工厂
      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories =
          new ArrayList<>(1 + this.converterFactories.size());
      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);
      // 创建 Retrofit 实例
      return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }

```

在 Build 创建 Retrofit 时有如下要注意的东西：

* callFactory : 网络请求工厂
* adapterFactories : 网络请求适配器工厂的集合
* converterFactories : 数据转换器工厂的集合
* callbackExecutor : 回调方法执行器


这四个东西可以说是 Retrofit 中很重要的内容了，它们在 Retrofit 类中都是接口或抽象类类型，将定义与实现进行分离，可以做到面向接口编程。

Retrofit 是对网络请求的封装库，一个网络请求的过程大致可以抽象如下：

```
创建网络请求 --> 发出网络请求 --> 收到请求内容 --> 对内容进行处理
```


而 Retrofit 可以说是对上面的流程都进行了抽象，抽象出一些类和接口，这些抽象类或者接口的调用就是实现上面的请求流程。


`adapterFactories` 就是对创建的网络请求适配的集合，比如默认请求是 `Call`，但是可以通过 `RxJavaCallAdapterFactory` 将网络请求适配到 RxJava 的 `Observable` 。

`callFactory` 就是用来执行网络请求的，将请求发出去，如果没有指定该变量，则默认使用 `OkHttpClient` 来完成该功能。

`converterFactories` 就是将请求后的内容转换成指定的格式的集合，如果服务器返回的是 GSON 格式，那么就可以指定 `GsonConverterFactory` 将 HTTP 的 Response 内容转换 GSON 对象。

`callbackExecutor` 就是在网络请求结束后，从子线程切换到对数据操作的线程，主要就是切换到主线程。


依照网络请求流程，上面四个重点变量在请求中出现的先后顺序也是如下：

```
adapterFactories --> callFactory --> converterFactories --> callbackExecutor
```

另外，Retrofit 还是支持多平台的，除了 Android 平台之外，还有 Java 等平台，在 `Retrofit.Builder` 的构造方法中就对当前平台做了一个判断，这里就只看 Android Platform 就好了。


弄懂了大致的脉络，接着再来看实际代码。


如果没有指定请求后的线程切换操作，那么就会使用 `defaultCallbackExecutor` 。 Android 下默认的就是 `MainThreadExecutor` ，通过主线程的 Handler 来实现线程切换。

```java
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());
      // 向主线程 post 一个 runnable 实现切换
      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
```


如果没有添加网络请求适配器对象，那么就会使用默认的 `defaultCallAdapterFactory` 。Android 下默认的就是 `ExecutorCallAdapterFactory` ，


它的主要代码如下：

```java
  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    // 返回 CallAdapter 对象进行适配
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }
      // 具体适配函数
      @Override public Call<Object> adapt(Call<Object> call) {
        // 返回 ExecutorCallbackCall 函数，因为就是对 call 的一个委托
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
```

重点在于返回 `CallAdapter` 对象，它将网络请求进行适配，如果没有使用 RxJava 的话，那么返回的就是 `ExecutorCallbackCall` 。

如果没有指定请求结果转换对象，那么就会使用默认的 `BuiltInConverters` 类进行转换，`BuiltInConverters` 与请求适配和线程切换不同，它是和平台无关的。

在 `adapterFactories` 和 `converterFactories` 中还用到了工厂模式，通过工厂模式对具体执行操作的 `CallAdapter` 和 `Converter` 进行封装。

以上就完成了 Retrofit 对象的创建。

## 动态代理创建网络请求对象

接下来就是使用 Retrofit 类通过动态代理创建一个代理对象实例。

```java
// 定义请求接口
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
// 动态代理创建实现了请求接口的类
GitHubService service = retrofit.create(GitHubService.class);
```

动态代理的作用与实现，可以参考 [这篇文章](https://glumes.com/post/android/java-proxy-design-pattern/)。

请求接口定义的所有方法，都会通过 `InvocationHandler` 中的 `invoke` 方法去分发处理。

也就是说，请求接口定义的返回类型是 `Call` ，那么 `invoke` 方法返回的也是 `Call` 类型。

主要代码如下：

```java
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.adapt(okHttpCall);
          }
        });
```

其中，`method` 就是对应接口中声明的方法，对每一个方法都会通过 `loadServiceMethod` 方法将它转换成 `ServiceMethod` 对象。

serviceMethod 会先从缓存 `ConcurrentHashMap` 中查找，如果没有在通过 `ServiceMethod.Builder` 创建一个。

创建时首先是如下两个方法：

```java
    // 通过 method 的返回类型和 注解 找到对应的 callAdapter
    callAdapter = createCallAdapter();
    // 通过 method 的 请求返回类型 和 注解 找到对应的 responseConverter
    responseConverter = createResponseConverter();
```


这两个方法有点类似，在前面提到的 `adapterFactories` 适配器集合和 `converterFactories` 转换器集合都是列表。

要通过方法的返回类型、请求结果的返回类型、方法注解，为当前方法找到合适的适配器和转换器。

在这里可以看到，Retrofit 其实支持在请求接口中定义不同的返回类型，可以 `Call` 和 `Observable` 同时使用，也可以 `GSON` 和 `Response` 同时使用。

确定了请求适配和结果转换类之后，就是处理接口方法中定义的那些注解，分别是为该方法定义的注解和为该方法参数定义的注解。


```java
      // 处理方法对应的注解
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }
      // 处理方法参数对应的注解
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
          parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }
```

关于注解处理这块比较冗长，文章篇幅有限，就不细写了，其实也主要是对那些 `GET`、`POST`、`HEAD` 等进行处理。

对这些注解的处理在最后创建 `Call` 用来发起网络请求时会用到。

当注解也处理完了之后，就创建了 serviceMethod 对象，接着创建 `OkHttpCall` 对象，这个对象就不是缓存的了，因为它就是用来执行请求的，每一次都要新建一个。

OkHttpCall 实现了 `Call` 接口，可以看做是对它的封装。`Call` 接口在 OkHttp 中就是用来发起网络请求的，它主要方法就是 `execute` 和 `enqueue` ，

然后 serviceMethod 对象通过 `adapt` 方法对 OkHttpCall 进行适配转换，并且返回了转换结果，如果在创建 Retrofit 时添加了 RxJava 的适配，那么返回的就是 Observable 类型。

这个 `adapt` 方法的适配其实就是由 serviceMethod 的 `callAdapter` 来实现的。

上面提到，默认是使用 `ExecutorCallAdapterFactory` 工厂来实现的。

最后，`invoke` 函数返回的就是把 `OkHttpCall` 经过 `adapt` 适配后的对象，默认情况下返回 `ExecutorCallbackCall` 对象。


## 执行网络请求

有了 `ExecutorCallbackCall` 对象之后，就可以像 OkHttp 那样去执行请求了。

```java
        // 异步请求
        repos.enqueue(object : Callback<MutableList<Repo>> {
            override fun onFailure(call: Call<MutableList<Repo>>?, t: Throwable?) {
            }

            override fun onResponse(call: Call<MutableList<Repo>>?, response: Response<MutableList<Repo>>?) {
            }
        })
        // 同步请求
        repos.execute()
```


`ExecutorCallbackCall` 对象其实是对 `OkHttpCall` 的委托，具体的 `enqueue` 和 `execute` 方法都是由 OkHttpCall 来实现的。

先看一下同步调用

```java
   private @Nullable okhttp3.Call rawCall;
   @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;
      call = rawCall;
      if (call == null) {
        try {
        // createRawCall 方法创建 call 
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException | Error e) {
        }
      }
    }
    // 取消请求
    if (canceled) {
      call.cancel();
    }
    // 解析请求内容
    return parseResponse(call.execute());
  }
```

实际是通过 `createRawCall` 方法来创建 `Call` 的，在 `createRawCall` 里面又主要是通过 `serviceMethod.toCall` 方法来完成的。

```java
  okhttp3.Call toCall(@Nullable Object... args) throws IOException {
    // 用上了之前解析注解得到的结果
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);
    // 处理请求的参数
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }
    // 把注解和参数对应上
    for (int p = 0; p < argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }
    // 最后还是通过 callFactory 来创建 Call 
    return callFactory.newCall(requestBuilder.build());
  }
```

在 `toCall` 方法里就用到了之前解析的注解内容，依然通过建造者模式创建请求，并且把请求的参数和对应的注解内容对应上，也就是 `ParameterHandler` 的 `apply` 方法。

最后是通过 callFactory 网络请求工厂来创建请求，默认就是使用 OkHttp 来创建。


`parseResponse` 方法会处理 `Call` 执行的结果，最终也是通过 `serviceMethod.toResponse` 方法来处理。

```java
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```

在该方法里，用到之前提到的数据转换器 `responseConverter` 来实现数据的转换,最终就得到了请求的结果 `Response` 。

异步请求和同步请求大致相同，不同点在于异步请求要切换到主线程，这也是用之前提到的 `callbackExecutor` 来实现线程切换。

```java
      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          // 切换线程
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }
        //...
      }
```

`enqueue` 请求的回调方法通过 `callbackExecutor` 实现了线程切换。

以上，就分析了 Retrofit 从创建到发起网络请求的过程。

## 小结

Retrofit 的设计思想非常值得学习，事实上直接使用 OkHttp 同样可以实现功能，但是 Retrofit 对 OkHttp 的封装却更加方便了。

首先，是把网络请求的类型和参数，通过注解来处理了，用注解标注，可以简化不少代码，在 Retrofit 内部去实现注解的解析，并且通过动态代理去创建实现了注解方法接口的对象实例。

其次，大量设计模式的运用让 Retrofit 更容易去拓展，添加不同的适配和转换，把网络请求的每一个环节都进行拆分，面向接口的编程，网络请求适配和数据转换，每一个部分都可以独立开。

接着，基于对 OkHttp 的封装，能够利用 OkHttp 去实现最重要的网络请求功能。

最后，学习 Retrofit 的设计思想，并把它们用到我们的代码中去。