---
title: "Android 6.0 Launcher 启动 Activity 过程源码分析（三）"
date: 2017-12-22T10:40:45+08:00
categories: ["Android"]
tags: ["Android"]
toc: true
original: true
addwechat: true
 
slug: "android-start-activity-from-launcher-3"
image: "/banner/robotic-5714849.svg"
---


在 [ Android 6.0 Launcher 启动 Activity 过程源码分析（二）](https://glumes.com/android-start-activity-from-launcher-2) 分析完了对待启动 Activity 组件的验证过程，获得组件信息，以及 ActivityRecord 添加至栈顶，将其他 Activity 进入中止状态，最后将待启动的 Activity 组件进入 `Resumed`状态，然而，由于待启动的 Activity 组件的应用程序进程尚未启动，最后执行 `startSpecificActivityLocked`方法创建进程。


<!--more-->

### ActivityStackSupervisor 类的 startSpecificActivityLocked() 方法

``` java
 void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);
		
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
            }
        }
		// 进程尚未启动，app 为 null 。
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```
在 ActivityManagerService 中，每一个 Activity 组件都有一个用户 ID 和一个进程名称，其中，用户 ID 是在安装该 Activity 组件时由 PackageManagerService 分配的，而进程名称则是由该 Activity 组件的 `android:process`属性来决定的。ActivityManagerService 在启动一个 Activity 组件时，首先会以它的用户 ID 和进程名称来检查系统中是否存在一个对应的应用程序进程。如果存在，就会直接通知这个应用程序进程将 Activity 组件启动起来；否则，就会先以这个用户 ID 和进程名称来创建一个应用程序进程，然后在通知这个应用程序进程将该 Activity 组件启动起来。

由于应用程序进程尚未启动，则 app 为 null ，mService 变量为 ActivityManagerService 。执行 `startProcessLocked`方法。

### ActivityManagerService类的 startProcessLocked() 方法
``` java
 final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
        long startTime = SystemClock.elapsedRealtime();
        ProcessRecord app;
        if (!isolated) { // 传入的 isolated 参数为 false ，if 成立，并不是隔离的进程
	        // 根据进程名称和用户 ID 得到应用程序进程，由于不存在，则为 null 。
            app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
            checkTime(startTime, "startProcess: after getProcessRecord");
			// 省略部分代码
        } else {
            // If this is an isolated process, it can't re-use an existing process.
            app = null;
        }
		// 当进程已经被分配的 PID 时，
		if (app != null && app.pid > 0) {
		}
		// 应用程序进程不存在，创建新的进程
		if (app == null) {
            checkTime(startTime, "startProcess: creating new process record");
            // 创建应用程序进程
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
            if (app == null) {
            }
            app.crashHandler = crashHandler;
            checkTime(startTime, "startProcess: done creating new process record");
        } else {
            // If this is a new package in the process, add the package to the list
            app.addPackage(info.packageName, info.versionCode, mProcessStats);
            checkTime(startTime, "startProcess: added package to existing proc");
        }
		// 创建应用程序进程后，最终调用 startProcessLocked 方法
		startProcessLocked(
                app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
```

ActivityManagerService 类的 `startProcessLocked`方法重载了多个形式，最终执行了上述的函数。

由于并不是隔离的进程，首先会根据进程名称和用户 ID 检查应用程序是否存在，由于不存在，app 为 null ，则创建了新的应用程序进程，通过`newProcessRecordLocked`方法。最后还是调用了 `startProcessLocked` 方法。

### ActivityManagerService类的 newProcessRecordLocked() 方法

``` java
final ProcessRecord newProcessRecordLocked(ApplicationInfo info, String customProcess,
            boolean isolated, int isolatedUid) {
        String proc = customProcess != null ? customProcess : info.processName;
        BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
        final int userId = UserHandle.getUserId(info.uid);
        int uid = info.uid;
        if (isolated) {
          // 省略与隔离进程相关代码
        }
        final ProcessRecord r = new ProcessRecord(stats, info, proc, uid);
        if (!mBooted && !mBooting
                && userId == UserHandle.USER_OWNER
                && (info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
            r.persistent = true;
        }
        addProcessNameLocked(r);
        return r;
    }
```
通过 ApplicationInfo 创建了一个 ProcessRecord 。


### ActivityManagerService类的 startProcessLocked() 方法

当创建完 ProcessRecord 后，最后还是调用了 `startProcessLocked`方法来创建一个进程 `Process` 。

``` java
private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
		// 省略部分代码
	    // Start the process.  It will either succeed and return a result containing
        // the PID of the new process, or else throw a RuntimeException.
        // 在之前的函数调用中，entryPoint 参数为 null，则赋值为 android.app.ActivityThread 。
        boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
        // 创建一个进程
        Process.ProcessStartResult startResult = Process.start(entryPoint,
                app.processName, uid, uid, gids, debugFlags, mountExternal,
                app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                app.info.dataDir, entryPointArgs);

		// 省略部分代码
		 app.setPid(startResult.pid);
            app.usingWrapper = startResult.usingWrapper;
            app.removed = false;
            app.killed = false;
            app.killedByAm = false;
            checkTime(startTime, "startProcess: starting to update pids map");
            synchronized (mPidsSelfLocked) {
            // app 类型为 ProcessRecord
            // 将 ProcessRecord 对象保存在 ActivityManagerService 类的成员变量 mPidsSelfLocked 中
                this.mPidsSelfLocked.put(startResult.pid, app);
                if (isActivityProcess) {
                // 向 ActivityManagerService 所运行的线程的消息队列发送 PROC_START_TIMEOUT_MSG 类型的消息
                // 并指定这个消息在 PROC_START_TIMEOUT 毫秒后处理
                    Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                    msg.obj = app;
                    mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                            ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
                }
            }
}
```

调用 Process 类的静态成员函数`start`来启动一个新的应用程序进程。

新的应用程序进程创建完成后，当前进程就会得到一个大于 0 的进程 ID ，保存在变量 `pid`中，接着就以变量 `pid`为关键字将参数 app 所指向的一个 ProcessRecord 对象保存在 ActivityManagerService 类的成员变量 `mPidsSelfLocked`中。

最后，当新的进程启动完后，还需要向 ActivityManagerService 发送一个通知，以便 ActivityManagerService 可以在它里面启动一个 Service 组件，否则，ActivityManagerService 会认为它超时了，因此，不能将 Activity 组件启动起来。


### ActivityThread 类的 main() 方法

由于 Process 类的静态方法 start 的第一个参数指明了进程进入点，则 ActivityThread 的 main 方法为一个进程的开始点。

``` java
public final class ActivityThread {
   final ApplicationThread mAppThread = new ApplicationThread();
   public static void main(String[] args) {
	   // 初始化主线程的消息队列
	   Looper.prepareMainLooper();
       ActivityThread thread = new ActivityThread();
       thread.attach(false);
	
	   if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
       }
       // 开启消息循环
       Looper.loop();
   }
   
   private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) { // 是否为系统进程
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            // 获得 ActivityManagerService 的代理对象
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
     
        } else {
		   // 省略系统进程代码
        } 
		// 省略 ViewRootImpl 相关代码
    }
}

// ActivityManagerProxy 的 attachApplication 方法
    public void attachApplication(IApplicationThread app) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(app.asBinder());
        mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
```

新的应用程序启动时，在 main 方法里主要做了两件事情：
*	在新的进程里面创建一个 ActivityThread 对象，并且调用它的成员函数 attach 向 ActivityManagerService 发送一个启动完成通知。
*	调用 Looper 类的静态成员函数`prepareMainLooper`创建一个消息循环，并且在向 ActivityManagerService 发送启动完成通知之后，使得当前进程进入到这个消息循环中。

在 ActivityThread 对象内还创建了一个 ApplicationThread 对象，在之前提到过 ApplicationThread 是一个 Binder 本地对象，ActivityManagerService 就是通过它来和应用程序通信的。

在 `attach`方法内部通过 `ActivityManagerNative.getDefault()`得到了 ActivityManagerService 的代理对象 `ActivityManagerProxy`。`ActivityManagerProxy`通过 Binder 驱动发起一个类型为 `ATTACH_APPLICATION_TRANSACTION`的进程间通信。接下来就是 ActivityManagerService 响应这个通信。

### ActivityManagerService类的 attachApplication() 方法
``` java
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```
接下来执行 `attachApplicationLocked`方法。

### ActivityManagerService类的 attachApplicationLocked() 方法

``` java
private final boolean attachApplicationLocked(IApplicationThread thread, int pid) {
		ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
        } else {
            app = null;
        }

	   final String processName = app.processName;
	   try {
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            app.resetPackageList(mProcessStats);
            startProcessLocked(app, "link fail", processName);
            return false;
        }
		app.makeActive(thread, mProcessStats);
        app.curAdj = app.setAdj = -100;
        app.curSchedGroup = app.setSchedGroup = Process.THREAD_GROUP_DEFAULT;
        app.forcingToForeground = null;
        updateProcessForegroundLocked(app, false, false);
        app.hasShownUi = false;
        app.debugging = false;
        app.cached = false;
        app.killedByAm = false;
		
		// 移除超时消息，应用程序在规定时间内完成了启动。
        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

		try{
		// 省略部分代码，跨进程调用 ActivityThread 的方法
			thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    enableTrackAllocation, isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
		}catch(Exception e){
			// 
		}
		boolean badApp = false;
        boolean didSomething = false;
		
		 // See if the top visible activity is waiting to run in this process...
		 // 调度 Activity
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                badApp = true;
            }
        }
		// Find any services that should be running in this process...
		// 调度 Service
        if (!badApp) {
            try {
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                badApp = true;
            }
        }
        // Check if a next-broadcast receiver is in this process...
        // 调度 Broadcast 
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {
                didSomething |= sendPendingBroadcastsLocked(app);
            } catch (Exception e) {
                // If the app died trying to launch the receiver we declare it 'bad'
                badApp = true;
            }
        }


}

// ApplicationThread 的 bindApplication 方法
public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean trackAllocation, boolean isRestrictedBackupMode,
                boolean persistent, Configuration config, CompatibilityInfo compatInfo,
                Map<String, IBinder> services, Bundle coreSettings) {

            if (services != null) {
                // Setup the service cache in the ServiceManager
                ServiceManager.initServiceCache(services);
            }
            setCoreSettings(coreSettings);
            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableOpenGlTrace = enableOpenGlTrace;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            // 向 ActivityThread 的应用程序主进程发送消息
            sendMessage(H.BIND_APPLICATION, data);
        }

```
方法的参数 `pid`指向了前面所创建的应用程序进程的 PID ，在之前的步骤中，ActivityManagerService 以 PID 为关键字将一个 ProcessRecord 对象保存在了成员变量 `mPidsSelfLocked` 中，因此又通过该 PID 将 ProcessRecord 对象取回，保存在 app 变量中。

然后就对 app 变量进程赋值初始化，最重要的就是将应用程序进程的 ApplicationThread 对象赋值给 ProcessThread 的 `thread` 成员变量，这样 ActivityManagerService 就可以通过这个 ApplicationThread 代理对象和新创建的应用程序进程进行通信了。

接下来会执行 thread 的 `bindApplication`方法，该方法是一个跨进程通信了，因为 thread 是 ApplicationThread 类型。ActivityManagerService 正是通过它与 Activity 组件通信的。该方法的作用主要是将应用程序的一些信息发送给 Activity ，例如：`ProfilerInfo`之类的。

而`ApplicationThread`内也响应了该方法，通过解析封装相应的数据，向主线程发送一个类型为`BIND_APPLICATION`的消息。

而在`ActivityThread`中也有一个类型为 `H`的 Handler ，继承自 Handler，用来处理主线程的消息循环。在 `H`类的`handleMessage`中通过`handleBindApplication`方法处理了类型为`BIND_APPLICATION`的消息。


当 `ActivityManagerService` 的信息传递给 ActivityThread 后，就可以开始调度了 Activity 组件了。通过 `attachApplicationLocked`方法调度 Activity 。



### ActivityThread 类的 handleBindApplication() 方法

``` java
private void handleBindApplication(AppBindData data) {

	// send up app name; do this *before* waiting for debugger
	// 设置进程名
        Process.setArgV0(data.processName);
    // 创建 Android 运行环境 Context .
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);

	// 初始化 Intrumentation 对象
	if (data.instrumentationName != null) {
		try {
	       java.lang.ClassLoader cl = instrContext.getClassLoader();
	       mInstrumentation = (Instrumentation)
	                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
	       } catch (Exception e) {
	       }
	       mInstrumentation.init(this, instrContext, appContext,
                   new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
                   data.instrumentationUiAutomationConnection);
	 } else {
            mInstrumentation = new Instrumentation();
     }

	 // Allow disk access during application and provider setup. This could
     // block processing ordered broadcasts, but later processing would
     // probably end up doing the same disk access.
 
        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            // 创建 Application 对象
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

            // don't bring up providers in restricted mode; they may depend on the
            // app's custom Application class
            if (!data.restrictedBackupMode) {
                List<ProviderInfo> providers = data.providers;
                if (providers != null) {
                    installContentProviders(app, providers);
                    // For process that contains content providers, we want to
                    // ensure that the JIT is enabled "at some point".
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {         
            }
            try {
            // 执行 Application 的 onCreate 方法
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
            }
        } finally {
        }
}
```

系统进程将很多与应用程序进程相关的信息传递至此进行绑定，就是为了初始化一个 Android 应用程序的运行信息。

*	初始化了运行环境上下文 Context 。
*	初始化了 Instrumentation 。
*	初始化了 Application 类。
*	调用了 Application 的 onCreate 方法。

当把这些信息传递给 Activity 组件，并且初始化结束后，新的一个进程就蜕变成了 Android 进程了。


###  StackSupervisor 类的 attachApplicationLocked() 方法

``` java
 boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFrontStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0);
        }
        return didSomething;
    }
```

遍历 ActivityStack 和 TaskRecord，找到位于 Activity 堆栈顶端的一个 ActivityRecord 对象 `hr`，接着检查这个 Activity 组件的用户 ID 和 进程名是否与 ProcessRecord 对象 app 所描述的应用程序的用户 ID 和进程名一致，如果一致，则调用 `realStartActivityLocked`方法来请求该应用程序进程启动一个 Activity 组件。


###  StackSupervisor 类的 realStartActivityLocked() 方法

``` java
    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

		r.app = app;
        app.waitingToKill = null;
        r.launchCount++;
        r.lastLaunchTime = SystemClock.uptimeMillis();
        
        int idx = app.activities.indexOf(r);
        if (idx < 0) {
            app.activities.add(r);
        }
        
	    try {
		    app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
        } catch (RemoteException e) {
        }
        return true;
  }
// ApplicationThread 的 scheduleRelaunchActivity 方法
 public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);
			// 向主线程发送消息
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```

首先将参数 `r`的成员变量`app`的值设置为参数`app`，表示它描述的 Activity 组件是在参数 app 所描述的应用程序进程中启动的，接着将该 Activity 组件添加到参数 `app`所描述的应用程序进程的 Activity 组件列表中。

接下来调用`thread`的`scheduleLaunchActivity`方法来通知前面创建的应用程序进程启动由参数`r`所描述的一个 Activity 组件。

由于 `thread`的类型是 ApplicationThread，又是一个 Binder 对象，则又是跨进程的通信，此时的执行还是在 ActivityManagerService 进程内的。通过 ApplicationThread 跨进程向应用程序进程发送请求。

在主线程的消息循环中对应的响应方法为`handleLaunchActivity`。

### ActivityThread 类的 handleLaunchActivity() 方法

``` java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {

		 // Initialize before creating the activity
        WindowManagerGlobal.initialize();
        Activity a = performLaunchActivity(r, customIntent);

		if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);
		} else {
            // If there was an error, for any reason, tell the activity
            // manager to stop us.
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
                // Ignore
            }
        }
```

首先调用了`performLaunchActivity`将 Activity 组件启动起来，然后在调用 `handleResumeActivity`方法将 Activity 组件的状态设置为 `Resumed`。


### ActivityThread 类的 performLaunchActivity() 方法

``` java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

	ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }
    // 通过反射 新建一个 Activity 对象
	Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            if (activity != null) {
	            // 创建 Context ，作为 Activity 的运行上下文环境
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);
                        
                if (r.isPersistable()) { // 回调 Activity 的 onCreate 函数
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
              
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) { // 回调 Activity 的 onStart 函数
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) { // Activity 的 onRestoreInstanceState 函数
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
              
            }
            r.paused = true;
            mActivities.put(r.token, r);
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
        }
}
```

首先获得要启动的 Activity 组件的包名和类名，它们使用一个`ComponentName`对象 component 来描述。

接着通过反射创建一个 Activity 类实例。然后，初始化了一个 Context 对象，作为前面创建的 Android 类实例的运行环境上下文，通过它就能访问特定的应用程序资源，再调用 `attach`方法，初始化 Activity 相关信息。

再调用`Instrumentation`类的`callActivityOnCreate`方法将 Activity 启动起来，此时，Activity 的 onCreate 方法就会被调用了，接下来就是调用 `performStart`方法，方法内部调用`Instrumentation`类的`callActivityOnStart`方法调用 Activity 的 onStart 方法。此时，Activity 的 onCreate 和 onStart 两大方法都调用了。


接下来就是执行`handleResumeActivity`方法将 Activity 组件的状态设置为 `Resumed`。

### ActivityThread 类的 handleResumeActivity() 方法

``` java
 final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {

	 // TODO Push resumeArgs into the activity for consideration
	 // 真正执行的是 performResumeActivity 方法
        ActivityClientRecord r = performResumeActivity(token, clearHide);
        if (r != null) {
	        // 省略和 Window 相关代码
        } else {
	        try {
                ActivityManagerNative.getDefault()
                    .finishActivity(token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
            }
        }
}
```
真正执行的是 `performResumeActivity`方法。

### ActivityThread 类的 performResumeActivity() 方法
``` java
    public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide) {
             ActivityClientRecord r = mActivities.get(token);
             if (r != null && !r.activity.mFinished) {
	             r.activity.performResume();
	         }
	         return r ;
	}
```

最后，`performResume`方法还是调用的`Instrumentation`类的 `callActivityOnResume`方法，让当前 Activity 组件进入了 `Resumed`状态。

至此，从 Launcher 组件启动 Activity 过程源码分析就算是过了一遍了，里面还有许多细节之处，等待日后挖掘。


## 参考

1. Android 6.0 源码 
2. 《Android 系统源代码情景分析》 
3. http://duanqz.github.io/2016-07-29-Activity-LaunchProcess-Part1

