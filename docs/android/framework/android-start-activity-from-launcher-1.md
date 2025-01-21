---
title: "Android 6.0 Launcher 启动 Activity 过程源码分析（一）"
date: 2017-12-22T10:40:38+08:00
categories: ["Android"]
tags: ["Android"]
toc: true
original: true
addwechat: true
 
slug: "android-start-activity-from-launcher-1"
image: "/banner/robotic-5714849.svg"
---


当 Android 系统在启动时，会扫描系统特定目录，然后自动安装里面的 Android 应用程序。当系统启动完成之后，会启动一个 Home 应用程序来显示安装在系统中的 Android 应用程序。

这个应用程序就是 Launcher 应用，也就是手机屏幕上显示的各种应用图标，Launcher 是 Android 系统启动的第一个应用程序。

而当我们点击应用程序图标时，也就开启了从 Launcher 启动 Activity 的过程。

<!--more-->

## 过程概述

Activity 其实也还是 Java 里的一个类，但是在 Android 开发中可不能把它当做一个普通类来处理，它可是 Android 四大组件之一。

Activity 的启动是由 ActivityManagerService 来完成的，由于 Launcher 组件、Activity 组件、ActivityManagerService 组件分别是运行在三个不同的进程中的，这三个进程的通信是通过 Binder 进程间通信机制来完成。

Launcher 组件通过 ServiceManager 得到 ActivityManagerService 的代理对象 Binder ，通过它向 ActivityManagerService 发起进程间通信请求，同时把自己应用程序进程（ActivityThread）的 Binder 对象（ApplicationThread）传递给 ActivityManagerService，以便 ActivityManagerService 调用 Launcher 组件的方法。这样就形成了一个有来有去的双方通信机制。

Launcher 组件启动 Activity 的过程大致如下所示：

1.	Launcher 组件向 ActivityManagerService 组件发送一个启动 Activity 组件的进程间通信请求。
2ActivityManagerService 首先将要启动的 Activity 组件的信息保存下来，然后再向 Launcher 组件发送一个进入中止状态（`onPause 方法`）的进程间通信请求。
2.	Launcher 组件进入到中止状态（`onPause 方法`）之后，就会向 ActivityManagerService 发送一个已进入中止状态的进程间通信请求，以便 ActivityManagerService 可以继续执行启动 Activity 组件的操作。
3.	ActivityManagerService 会检查当前 Activity 组件的应用程序进程是否存在，如果不存在，则会通过 Binder 进程间通信机制来请求 Zygote 进程将当前应用程序进程启动起来。
4.	新的应用程序进程启动完毕后，就会向 ActivityManagerService 发送一个启动完成的进程间通信请求，以便 ActivityManagerService 可以继续执行启动 Activity 组件的操作。
5.	ActivityManagerService 将第二步保留的 Activity 组件的信息发送给第四步创建的应用程序进程，以便它可以将 Activity 组件启动起来。

## 代码过程分析

### Launcher 类的 startActivitySafely() 方法

Launcher 毕竟是一个系统应用，它的源代码位于 `packages/apps/Launcher2/src/com/android/launcher2/Launcher.java` 。

``` java
/**
 * Default launcher application.
 */
public final class Launcher extends Activity
        implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, View.OnTouchListener {
```

从代码中可以看到 Launcher 继承自 Activity，并且实现了点击、长按、触摸等接口。


所以在`onClick`方法中就能找到启动 Activity 组件的调用方法 `startActivitySafely`。其中`Intent`参数包含了相应的信息。

``` java
 boolean startActivitySafely(View v, Intent intent, Object tag) {
        boolean success = false;
        try {
	        // 调用 Launcher 类的 startActivity 函数，不是 Activity 里面的，参数形式不对。
            success = startActivity(v, intent, tag);
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
        }
        return success;
    }
```

`startActivitySafely`函数又调用了 Launcher 里面封装的 `startActivity(View view, Intent intent, Object tag)`。

### Launcher 类的 startActivity() 方法
``` java
 boolean startActivity(View v, Intent intent, Object tag) {
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            // Only launch using the new animation if the shortcut has not opted out (this is a
            // private contract between launcher and may be ignored in the future).
            boolean useLaunchAnimation = (v != null) &&  
                    !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);
            // 安卓多用户相关 UserHandle 
            UserHandle user = (UserHandle) intent.getParcelableExtra(ApplicationInfo.EXTRA_PROFILE);
            LauncherApps launcherApps = (LauncherApps) // 
                    this.getSystemService(Context.LAUNCHER_APPS_SERVICE);
            if (useLaunchAnimation) {  // 是否使用启动动画
                ActivityOptions opts = ActivityOptions.makeScaleUpAnimation(v, 0, 0,
                        v.getMeasuredWidth(), v.getMeasuredHeight());
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    // Could be launching some bookkeeping activity
                    startActivity(intent, opts.toBundle()); // 使用动画，启动Activity
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(),
                            opts.toBundle());
                }
            } else {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent);
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(), null);
                }
            }
            return true;
        } catch (SecurityException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
        }
        return false;
    }
```

由于启动的 Activity 和 Launcher 组件不在同一进程，则在 Intent 上添加了 `FLAG_ACTIVITY_NEW_TASK`的 Flag 启动标识，以便可以在新的 `TASK`中启动。

启动时会判断当前 Activity 组件是否使用动画效果。并且，Android 中还有  `LAUNCHER_APPS_SERVICE` 这样一个 Service 用来启动 Activity 组件。

这里，假设没有动画，并且 user == null 的 if 判断成立，则直接调用父类 Activity 的 `startActivity(Intent intent)`方法。
``` java
@Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }
```
最终调用了父类 Activity 的`startActivity(Intent intent, @Nullable Bundle options)`方法。

``` java
 @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            startActivityForResult(intent, -1); // 方法内调用的也是上一种方法形式，只是 options 为 null 
        }
    }
```

### Activity 类的 startActivityForResult() 方法
由于之前假设的 options 参数为 null，则直接调用`startActivityForResult(intent, -1)`，方法原型如下：
``` java
public void startActivityForResult(Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }
```

这里面调用的也是`startActivityForResult`，只不过第三个参数为 null 了，也就是上面的 options 参数。
只有当请求码 `requestCode >=0`，才会执行调用者的 `onActivityResult()`方法，这里为 -1 ，不会。
``` java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                mStartedActivity = true;
            }
            cancelInputsAndStartExitTransition(options);
        } else {
			// 由于第一次从 Launcher 启动 Activity ，mParent 为 null 。
        }
    }
```
在 if 判断里面执行了 `Instrumentation` 类的`execStartActivity`方法。

`Instrumentation` 类会在应用的任何代码执行前被实列化，是用来监控应用程序和系统之间的交互操作。

方法原型为：
``` java
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
```

这里比较重要的就是里面传入的参数了，之前的参数都是 Intent 附带的 Activity 组件的信息。

一一对应如下：

|参数类型|实际参数|含义|
|---|----|---|
|Context who|this|Context 类|
|IBinder contextThread|mMainThread.getApplicationThread()|自己的Binder本地对象|
|IBinder token|mToken|ActivityManagerService 中的ActivityRecord 代理对象|
|Activity target|this|目标|
|Intent intent|intent|组件信息|
|int requestCode|requestCode|请求码|
|Bundle options|options|附件信息|

其中，options、requestCode、intent、who 参数都比较好理解。

而 contextThread 参数是一个 `IBinder` 类型，实际参数为`mMainThread.getApplicationThread()`。

其实 `mMainThread` 的类型为 `ActivityThread`，用来描述一个应用程序进程，它并不是一个线程，是一个 `final` 类型的类。

系统每当启动一个应用程序进程时，都会在它里面加载一个 `ActivityThread`类实例，并且将这个 `ActivityThread`类实例保存在每一个在该进程启动的 Activity 组件的父类 Activity 的成员变量 `mMainThread`中。

而`ActivityThread`类的成员函数`getApplicationThread()`用来获取它内部的一个类型为`ApplicationThread`的`Binder本地对象`。

``` java
private class ApplicationThread extends ApplicationThreadNative {}
public abstract class ApplicationThreadNative extends Binder implements IApplicationThread {}
```
`ApplicationThread`类是一个 Binder 类，那么它的作用是用来与 ActivityManagerService 通信的。

之前在 [Binder 学习](http://www.glumes.com/android-binder-note-1/)的时候，bindService 操作会在 onServiceConnected 返回 Service 的一个 Binder 对象，通过该 Binder 对象便可以执行 Service 中的函数，此时便完成的了 Activity 与 Service 的通信。而现在，我们在 execStartActivity() 方法中把自己的 Binder 对象传递给 ActivityManagerService，不就是相当于 onServiceConnected 中回调 Binder 了嘛。有了这个 Binder 对象，ActivityManagerService 才能够通知 Launcher 组件进入 `Paused` 状态。

`mToken`类也是一个 Binder 类型，它是一个 `Binder代理对象`，它指向了 ActivityManagerService 中一个类型为 `ActivityRecord`的 `Binder本地对象`。每一个已经启动的 Activity 组件在 ActivityManagerService 中都有一个对应的`ActivityRecord`对象，用来维护对应的 Activity 组件的运行状态。将它传递给 ActivityManagerService，以便 ActivityManagerService 获得 Launcher 组件的详细信息了。


### Instrumentation 类的 execStartActivity() 方法

``` java
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
		// 省略部分代码
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

Intent 执行了一些准备工作后，便调用 ActivityManagerService 的代理对象的 `startActivity`方法了。

ActivityManagerNative.getDefault() 方法返回的就是 ActivityManagerService 的代理对象，代理对象的获取在之前 [Binder 学习](http://www.glumes.com/android-binder-note-1/)的文章中已经了解过了。

``` java
public abstract class ActivityManagerNative extends Binder implements IActivityManager{

	static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }

        return new ActivityManagerProxy(obj);
    }
    
	static public IActivityManager getDefault() {
        return gDefault.get();
    }
	
	private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
        // 通过 ServiceManager 来获得 ActivityServiceManager 的代理对象。
        // 再通过 asInterface 方法判断是否为同一进程，不是，则封装成 ActivityManagerProxy 代理对象。
        // ActivityManagerProxy 代理对象的操作都是由 ActivityServiceManager 的代理对象 Binder 来完成的。
        // ActivityManagerProxy 只是在之上对数据封装了一层。
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
}
```

在 ActivityManagerNative 中还看到了一个有意思的单例写法：
``` java
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


### ActivityManagerProxy 类的 startActivity() 方法

Instrumentation 类的 execStartActivity() 方法最终调用了ActivityManagerProxy 类的 startActivity() 方法，由 ActivityManagerService 的代理对象 Binder 去发起类型为 START_ACTIVITY_TRANSACTION 的进程间通信请求。参数信息如注释所示：
``` java
    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        // caller 参数为 Launcher 组件所运行的应用程序进程的 ApplicationThread Binder 本地对象。
        data.writeStrongBinder(caller != null ? caller.asBinder() : null); 
        // Launcher 组件的包名
        data.writeString(callingPackage);
        // 需要启动的 Activity 组件的信息
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        // ActivityManagerService 内部的 ActivityRecord 对象，保存了 Launcher 组件的详细信息。
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```

以上五步，就完成了启动一个 Activity ，在 Launcher 组件中的步骤，这五步都在 Launcher 应用程序进程内执行的。

剩下的步骤将会在 ActivityManagerService 进程去执行。




## 涉及到的其他类

*	UserHandle
*	LauncherApps
*	ActivityOptions
*	Instrumentation
	*	监控应用程序和系统之间的交互操作
*	ActivityRecord 
	*	维护 Activity 组件的运行状态
*	ActivityThread
	*	表示 Android 应用程序进程
*	ApplicationThread
	*	Binder 类型变量，用来和 ActivityManagerService 通信



## 参考
1.  Android 6.0 源码
2. 《Android 系统源代码情景分析》



