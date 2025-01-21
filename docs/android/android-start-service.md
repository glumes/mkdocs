---
title: "Android 6.0 Service 启动过程源码分析（一）"
date: 2017-12-22T15:40:19+08:00
categories: ["Android"]
tags: ["Android"]
toc: true
original: true
addwechat: true
slug: "android-start-service"
 
---


Service 组件也是 Android 四大组件之一，它的启动过程分为显示和隐式两种。对于隐式启动的 Service 组件来说，我们只需要它的组件名称；对于显示启动的 Service 组件来说，我们需要知道它的类名称。

Service 组件可以被 Activity 组件启动，也可以被其他的 Service 组件启动。同时，它既可以在启动它的 Activity 组件或者 Service 组件所在的应用程序中启动，也可以在一个新的应用程序进程中启动。

<!--more-->

## Service 组件在新进程中的启动过程

不管 Service 组件是在 Activity 组件中启动的还是在 Service 组件中启动的，它们都是继承自`ContextWrapper`类的，最后调用的还是`ContextImpl`的`startService`方法。

所以直接从`ContextImpl`方法开始分析即可。

### ContextImpl 类的 startService() 方法
``` java
@Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, mUser);
    }

```

显然最后还是调用的`startServiceCommon`方法了。

### ContextImpl 类的 startServiceCommon() 方法
``` java
private ComponentName startServiceCommon(Intent service, UserHandle user) {
        try {
            validateServiceIntent(service);
            service.prepareToLeaveProcess();
            // 向 ActivityManagerService 发送请求启动 Service 组件
            ComponentName cn = ActivityManagerNative.getDefault().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), getOpPackageName(), user.getIdentifier());
            if (cn != null) {
                if (cn.getPackageName().equals("!")) {
                    throw new SecurityException( "Not allowed to start service " + service
                            + " without permission " + cn.getClassName());
                } else if (cn.getPackageName().equals("!!")) {
                    throw new SecurityException( "Unable to start service " + service
                            + ": " + cn.getClassName());
                }
            }
            return cn;
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
    }
```
在`startServiceCommon`方法内又是 IPC 进程间通信了，通过`ActivityManagerProxy`向 Binder 驱动发送类型为`START_SERVICE_TRANSACTION`的消息，在`ActivityManagerService`响应对应消息。


### ActivityManagerService 类的 startService() 方法

``` java
public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId)
            throws TransactionTooLargeException {
        enforceNotIsolatedCaller("startService");
        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        if (callingPackage == null) {
            throw new IllegalArgumentException("callingPackage cannot be null");
        }
        synchronized(this) {
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            ComponentName res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid, callingPackage, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
```

在 ActivityManagerService 内又是调用 ActivityServices 类的 `startServiceLocked()` 方法，将调用者的`pid`和`uid`传入。

### ActiveServices 类的 startServiceLocked() 方法

``` java
    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, String callingPackage, int userId)
            throws TransactionTooLargeException {
        final boolean callerFg;
        if (caller != null) {
            final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
            if (callerApp == null) {
                throw new SecurityException("");// 省略字符串内容
            }
            callerFg = callerApp.setSchedGroup != Process.THREAD_GROUP_BG_NONINTERACTIVE;
        } else {
            callerFg = true;
        }
        // 查找是否存在于 service 对应的 ServiceRecord 对象，没有则封装一个
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg);
        if (res == null) {
            return null;
        }
        if (res.record == null) {
            return new ComponentName("!", res.permission != null
                    ? res.permission : "private to package");
        }
        ServiceRecord r = res.record;
        if (!mAm.getUserManagerLocked().exists(r.userId)) { // 用户是否存在检查
            Slog.d(TAG, "Trying to start service with non-existent user! " + r.userId);
            return null;
        }

        NeededUriGrants neededGrants = mAm.checkGrantUriPermissionFromIntentLocked(
                callingUid, r.packageName, service, service.getFlags(), null, r.userId);
        if (unscheduleServiceRestartLocked(r, callingUid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
        }
        r.lastActivity = SystemClock.uptimeMillis();
        r.startRequested = true;
        r.delayedStop = false;
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                service, neededGrants));

        final ServiceMap smap = getServiceMap(r.userId);
        boolean addToStarting = false;
        if (!callerFg && r.app == null && mAm.mStartedUsers.get(r.userId) != null) {
            ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
            if (proc == null || proc.curProcState > ActivityManager.PROCESS_STATE_RECEIVER) {
                if (r.delayed) {
                    // This service is already scheduled for a delayed start; just leave
                    // it still waiting.
                    return r.name;
                }
                if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                    // Something else is starting, delay!
                    smap.mDelayedStartList.add(r);
                    r.delayed = true;
                    return r.name;
                }
                addToStarting = true;
            } else if (proc.curProcState >= ActivityManager.PROCESS_STATE_SERVICE) {
                addToStarting = true;
            } 
        } 

        return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    }
```

在 `startServiceLocked` 方法内的调用 `retrieveServiceLocked`方法在 ActivityManagerService 检查是否存在与参数 `service`对应的一个`ServiceRecord`对象。如果不存在，那么 ActivityManagerService 就会到 PackageManagerService 中去获取与参数`service`对应的一个 Service 组件的信息，然后将这些信息封装成一个`ServiceRecord`对象，最后将这个`ServiceRecord`对象封装成一个`ServiceLookupResult`对象返回给调用者。

接着就是对用户是否存在进行检查，最后调用`startServiceInnerLocked`方法。

###  ActiveServices 类的 startServiceInnerLocked() 方法

``` java
 ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
        ProcessStats.ServiceState stracker = r.getTracker();
        if (stracker != null) {
            stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
        }
        r.callStart = false;
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startRunningLocked();
        }
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false);
        if (error != null) {
            return new ComponentName("!!", error);
        }

        if (r.startRequested && addToStarting) {
            boolean first = smap.mStartingBackground.size() == 0;
            smap.mStartingBackground.add(r);
            r.startingBgTimeout = SystemClock.uptimeMillis() + BG_START_TIMEOUT;
            if (first) {
                smap.rescheduleDelayedStarts();
            }
        } else if (callerFg) {
            smap.ensureNotStartingBackground(r);
        }
        return r.name;
    }
```
该方法内调用`bringUpServiceLocked`方法。


###   ActiveServices 类的 bringUpServiceLocked() 方法
``` java
private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting) throws TransactionTooLargeException {

        if (r.app != null && r.app.thread != null) {// 满足条件，执行 service 的 onStartCommand 方法
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        if (!whileRestarting && r.restartDelay > 0) {
            // If waiting for a restart, then do nothing.
            return null;
        }
        // We are now bringing the service up, so no longer in the
        // restarting state.
        if (mRestartingServices.remove(r)) {
            r.resetRestartCounter();
            clearRestartingIfNeededLocked(r);
        }

        // Make sure this service is no longer considered delayed, we are starting it now.
        if (r.delayed) {
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        // Make sure that the user who owns this service is started.  If not,
        // we don't want to allow it to run.
        if (mAm.mStartedUsers.get(r.userId) == null) {
            bringDownServiceLocked(r);
            return msg;
        }

        // Service is now being launched, its package can't be stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
        }

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;

        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }
            }
        } else {
            app = r.isolatedProc;
        }

        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        if (app == null) {
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }

        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                stopServiceLocked(r);
            }
        }

        return null;
    }

```

在`bringUpServiceLocked`方法内先判断`ServiceRecord`的变量`app`和`thread`变量是否为`null`，也就是检查`service`对应的进程`ProcessRecord`和`ApplicationThread`对象是否存在，若存在，则直接执行`service`的`onStartCommand`方法。

若不存在，则将`service`从重启和延时启动队列中移开，因为它正在启动中了，最后再确保拥有该`service`的用户已经启动了。

接着 ActivityManagerService 根据进程名`procName`和`uid`调用`getProcessRecordLocked`方法查看对应的进程`ProcessRecord`是否存在。

*	若存在，则调用`realStartServiceLocked`方法。
*	若不存在，则调用`startProcessLocked`方法，先启动进程。

ActivityManagerService 通过`startProcessLocked`方法启动一个新的进程，在分析`Launcher`启动 Activity 组件时已经了解过了。

ActivityManagerService 创建一个新的进程，而一个新进程的入口就是`ActivityThread`类的`main`方法。在`ActivityThread`内会创建一个`ActivityThread`对象和一个`ApplicationThread`对象，而在`main`方法内又会调用`ActivityThread`的`attach`方法，用来向 ActivityManagerService 发送一个 类型为 `ATTACH_APPLICATION_TRANSACTION`的进程间通信，把`ApplicationThread`对象传递给 ActivityManagerService，以便能够 ActivityManagerService 可以和新进程进行 Binder 进程间通信。


ActivityManagerService 响应类型为`ATTACH_APPLICATION_TRANSACTION`的进程间通信，执行`attachApplication`方法，进而执行`attachApplicationLocked`方法。

在`attachApplicationLocked`方法内，首先会进行 Binder 跨进程调用，执行 `ApplicationThread`的`bindApplication`方法，然后再依次调度应用的`Activity`、`Service`、`Broadcast`组件。

其中，调度`Service`组件的代码如下：
``` java
    // Find any services that should be running in this process...
        if (!badApp) {
            try {
	            // mServices 的类型为 ActivityServices
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
                badApp = true;
            }
        }
```

### ActivityServices 类的 attachApplicationLocked() 方法

``` java
    boolean attachApplicationLocked(ProcessRecord proc, String processName)
            throws RemoteException {
        boolean didSomething = false;
        // Collect any services that are waiting for this process to come up.
        if (mPendingServices.size() > 0) {
            ServiceRecord sr = null;
            try {
                for (int i=0; i<mPendingServices.size(); i++) {
                    sr = mPendingServices.get(i);
                    if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                            || !processName.equals(sr.processName))) {
                        continue;
                    }
                    mPendingServices.remove(i);
                    i--;
                    proc.addPackage(sr.appInfo.packageName, sr.appInfo.versionCode,
                            mAm.mProcessStats);
                    realStartServiceLocked(sr, proc, sr.createdFromFg);
                    didSomething = true;
                    if (!isServiceNeeded(sr, false, false)) {// 不需要的服务就丢弃
                        bringDownServiceLocked(sr);
                    }
                }
            } catch (RemoteException e) {
                throw e;
            }
        }
        // Also, if there are any services that are waiting to restart and
        // would run in this process, now is a good time to start them.  It would
        // be weird to bring up the process but arbitrarily not let the services
        // run at this point just because their restart time hasn't come up.
        if (mRestartingServices.size() > 0) {
            ServiceRecord sr;
            for (int i=0; i<mRestartingServices.size(); i++) {
                sr = mRestartingServices.get(i);
                if (proc != sr.isolatedProc && (proc.uid != sr.appInfo.uid
                        || !processName.equals(sr.processName))) {
                    continue;
                }
                mAm.mHandler.removeCallbacks(sr.restarter);
                mAm.mHandler.post(sr.restarter);
            }
        }
        return didSomething;
    }
```

`attachApplicationLocked`方法内启动`mPendingServices`队列中的服务和`mRestartingServices`中的服务。

真正启动服务的方法就是`realStartServiceLocked`。

### ActivityServices 类的 realStartServiceLocked() 方法

``` java
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);
        // 发送一个延时消息 SERVICE_TIMEOUT_MSG，Service 若没有启动超时，则回告诉 AMS 取消该消息
        bumpServiceExecutingLocked(r, execInFg, "create");
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();

        boolean created = false;
        try {
            if (LOG_SERVICE_START_STOP) {
                String nameTerm;
                int lastPeriod = r.shortName.lastIndexOf('.');
                nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
                EventLogTags.writeAmCreateService(
                        r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
            }
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.ensurePackageDexOpt(r.serviceInfo.packageName);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            // 进入 Service 的 onCreate 
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app); // 创建服务时，应用挂了
            throw e;
        } finally {
            if (!created) {
                // Keep the executeNesting count accurate.
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);

                // Cleanup.
                if (newService) {
                    app.services.remove(r);
                    r.app = null;
                }

                // Retry.
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }

        requestServiceBindingsLocked(r, execInFg);
        updateServiceClientActivitiesLocked(app, null, true);

        // If the service is in the started state, and there are no
        // pending arguments, then fake up one so its onStartCommand() will
        // be called.
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }
		// 进入 Service 的 onStartCommand 
        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                stopServiceLocked(r);
            }
        }
    }
```

在`realStartServiceLocked`方法内，首先会通过`bumpServiceExecutingLocked`方法，然后再调用`scheduleServiceTimeoutLocked`方法，向 ActivityManagerService 的 `MainHandler`类型的 `mHandler` 发送一个消息 `SERVICE_TIMEOUT_MSG`。若 Service 在启动过程中没有及时向 ActivityManagerService 返回消息，把`SERVICE_TIMEOUT_MSG`消息给取消掉，那么则认为 Service 启动超时了。


标记完了时间后，就通过`scheduleCreateService`方法来启动 Service ，让 Service 进去`onCreate`状态。

新创建的进程把自己的 Binder 本地对象 ApplicationThread 传递给了 ActivityManagerService ，ActivityManagerService 就通过它来与新进程通信了，`scheduleCreateService`方法向 Binder 驱动发送了一个类型为 `SCHEDULE_CREATE_SERVICE_TRANSACTION`的消息，而在 `ApplicationThread`响应了该消息。

``` java
public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            sendMessage(H.CREATE_SERVICE, s);
        }
```
ApplicationThread 的 `scheduleCreateService`方法向 ActivityThread 的主线程 `mH`发送了消息`CREATE_SERVICE`。ActivityThread 最终响应该消息。

### ActivityThread 类的 handleCreateService() 方法
``` java
 private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            // 通过反射创建目标服务对象
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
        }

        try {
            // 创建 ContextImpl
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
			// 创建 Application 对象
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            // Service 进入 onCreate 状态
            service.onCreate();
            mServices.put(data.token, service);
            try {
	            // 向 ActivityManagerService 发送消息取消 Service 启动延时消息
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                // nothing to do.
            }
        } catch (Exception e) {
        }
    }
```

ActivityThread 类的`handleCreateService`主要是创建了 Service 对象，并且创建了`ContextImpl`对象和`Application`对象。同时调用了 Service 的`onCreate`方法。我们继承的 Service 的`onCreate`方法也是在这里被系统回调的。

随后向 ActivityManagerService 发送消息，取消 Service 延时启动的消息。

Service 进入 `onCreate`状态后，接下来该进入`onStartCommand`状态。

### ActivityServices 类的 sendServiceArgsLocked() 方法
``` java
    private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) {
            return;
        }

        while (r.pendingStarts.size() > 0) {
            Exception caughtException = null;
            ServiceRecord.StartItem si;
            try {
                si = r.pendingStarts.remove(0);
                if (si.intent == null && N > 1) {
                    // If somehow we got a dummy null intent in the middle,
                    // then skip it.  DO NOT skip a null intent when it is
                    // the only one in the list -- this is to support the
                    // onStartCommand(null) case.
                    continue;
                }
                si.deliveredTime = SystemClock.uptimeMillis();
                r.deliveredStarts.add(si);
                si.deliveryCount++;
                if (si.neededGrants != null) {
                    mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                            si.getUriPermissionsLocked());
                }
                // 添加延时处理，类似于 onCreate 
                bumpServiceExecutingLocked(r, execInFg, "start");
                if (!oomAdjusted) {
                    oomAdjusted = true;
                    mAm.updateOomAdjLocked(r.app);
                }
                int flags = 0;
                if (si.deliveryCount > 1) {
                    flags |= Service.START_FLAG_RETRY;
                }
                if (si.doneExecutingCount > 0) {
                    flags |= Service.START_FLAG_REDELIVERY;
                }
	            // 跨进程 Service 进入 onStartCommand 状态
                r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
            } catch (TransactionTooLargeException e) {
            } catch (RemoteException e) {
            } catch (Exception e) {
            }
        }
    }
```

在`sendServiceArgsLocked`方法内通过跨进程`scheduleServiceArgs`方法，向 ActivityThread 发送消息。


### ActivityThread 类的 handleServiceArgs() 方法
``` java
    private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                if (data.args != null) {
                    data.args.setExtrasClassLoader(s.getClassLoader());
                    data.args.prepareToEnterProcess();
                }
                int res;
                if (!data.taskRemoved) {
	                // Service 进入 onStartCommand 状态
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                } else {
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }
                QueuedWork.waitToFinish();
                try {
                // 向 ActivityManagerService 发送消息取消延时
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                } catch (RemoteException e) {
                    // nothing to do.
                }
                ensureJitEnabled();
            } catch (Exception e) {
            }
        }
    }
```
ActivityThread 响应 Binder 驱动发送来的消息，仍然通过类型为`H`的`mH`处理消息，处理消息的方法就是`handleServiceArgs`。在方法内，回调了 Service 的`onStartCommand`方法，并且向 ActivityManagerService 发送消息取消了延时操作。


至此，Service 在新进程中的启动过程就分析结束了，从调用者的`startService`方法，再到 ActivityManagerService 的处理，再到 Service 进程的`onCreate`和`onStartCommand`方法。


## 参考
1. Android 6.0 源码
2. 《Android 系统源代码情景分析》
3. http://gityuan.com/2016/03/06/start-service/


