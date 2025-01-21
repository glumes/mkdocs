---
title: " Android 6.0 Launcher 启动 Activity 过程分析小结（四）"
date: 2017-12-22T10:40:49+08:00
categories: ["Android"]
tags: ["Android"]
toc: true
original: true
addwechat: true
 
slug: "android-start-activity-from-launcher-4"
image: "/banner/robotic-5714849.svg"
---


在如下三篇文章中过了一遍 Launcher 启动 Activity 的代码流程。

*	[ Android 6.0 Launcher 启动 Activity 过程源码分析（一）](https://glumes.com/android-start-activity-from-launcher-1)
*	[ Android 6.0 Launcher 启动 Activity 过程源码分析（二）](https://glumes.com/android-start-activity-from-launcher-2)
*	[ Android 6.0 Launcher 启动 Activity 过程源码分析（三）](https://glumes.com/android-start-activity-from-launcher-3)

然而， 即使看过了多遍代码流程依旧有点云里雾里的感觉。不从整体上来把握，光抓住细节代码会始终不得要领。

由于是从 Launcher 组件启动一个 Activity 组件，其中还需要与 ActivityManagerService 通信，而这三个部分都是位于不同的进程内，涉及进程间通信，因此可以将整个过程划分为三个不同的部分来分析，在 Launcher 进程内的操作，在 ActivityManagerService 进程内的操作，在创建的应用程序进程内的操作。

<!--more-->

## Launcher 进程内的操作

在 Launcher 进程内执行函数：

*	Launcher . `startActivitySafely`(View v, Intent intent, Object tag)

*	Launcher . `startActivity`(View v, Intent intent, Object tag)
*	Activity . `startActivity`(Intent intent, @Nullable Bundle options)
*	Activity . `startActivityForResult`(Intent intent, int requestCode, @Nullable Bundle options)
*	Instrumentation . `execStartActivity`( Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options)
*	ActivityManagerProxy . `startActivity`(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle options)

Launcher 进程这部分执行的操作主要就是 `委托`ActivityManagerService 去启动一个 Activity 组件，把这个 Activity 组件的 `Intent`信息传递过去，把自己的包名信息传递过去，还把自己的本地 Binder 对象 `ApplicationThread`传递过去，以便 ActivityManagerService 可以通过它来联系到 Launcher 组件。

毕竟，当 ActivityManagerService 要去创建新的 Activity 组件时，首先得让之前的 Activity 组件进入 `Paused`状态。

Launcher 组件的运行流程从`startActivitySafely`到`startActivity`，再从 Activity 类的`startActivity`到`startActivityForResult`，最后通过 `Instrumentation`类的`execStartActivity`方法得到 ActivityManagerService 的代理对象 `ActivityManagerProxy`来执行`startActivity`。成功地把 Activity 组件启动的任务交给了 ActivityManagerService 去完成。

## ActivityManagerService 进程

在 Launcher 进程内执行函数：

*	ActivityManagerService . `startActivity`(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle options)
*	ActivityManagerService . `startActivityAsUser`(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId)

*	ActivityStackSupervisor . `startActivityMayWait`(...)
*	ActivityStackSupervisor . `startActivityLocked`( ...)
*	ActivityStackSupervisor . `startActivityUncheckedLocked`( ...)
*	ActivityStack . `startActivityLocked`(ActivityRecord r, boolean newTask, boolean doResume, boolean keepCurTransition, Bundle options)
*	ActivityStackSupervisor . `resumeTopActivitiesLocked`(ActivityStack targetStack, ActivityRecord target, Bundle targetOptions)
*	ActivityStack . `resumeTopActivityLocked`(ActivityRecord prev, Bundle options)
*	ActivityStack . `resumeTopActivityInnerLocked`(ActivityRecord prev, Bundle options)
*	ActivityStackSupervisor . `startSpecificActivityLocked`(ActivityRecord r, boolean andResume, boolean checkConfig)

ActivityManagerService 响应跨进程调用执行`startActivity`方法，方法内执行`startActivityAsUser`方法。

接着执行`startActivityMayWait`方法，首先通过 `PackageManager`解析待启动 Activity 的 Intent 信息，得到了包含更多信息的 `ActivityInfo`对象。

接着开始执行`startActivityLocked`方法，创建一个待启动的 Activity 组件的 `ActivityRecord`对象，里面包含了相关信息。在接下来的方法`startActivityUncheckedLocked`中为待启动的 Activity 组件找到它的 `Task` 任务，由于是新建的 Activity，则直接创建了一个新的 `Task`。而这个 `Task`也是 `ActivityRecord`的成员变量`task`。由这个 `TaskRecord`得到它的成员变量`ActivityStack` ，管理 Activity 的堆栈。

接下来便执行 `ActivityStack`类的`startActivityLocked`方法， 首先将待启动的 ActivityRecord 位于栈顶，然后执行 `resumeTopActivitiesLocked`方法，将所有 ActivityStack 栈顶的 ActivityRecord 运行至 `Resumed` 状态，其中都是调用的`resumeTopActivityLocked`方法，而该方法内部又是调用的`resumeTopActivityInnerLocked`方法，该方法会做一系列的判断，判断当年能够将 ActivityStack 栈顶的 Activity 运行至 `Resumed`状态，比如：当有 Activity 没有进入 `Paused` 状态时，便会通过跨进程通信来通知它进入 `Paused` 状态。当所有的状态都满足了，准备就绪了，就会去检查应用程序进程是否启动了，如果没有则去创建进程，否则让 Activity 组件运行至 `Resumed` 状态。

## ActivityManagerService 创建应用程序进程

*	ActivityStackSupervisor . `startSpecificActivityLocked`(ActivityRecord r, boolean andResume, boolean checkConfig)
*	ActivityManagerService . `startProcessLocked`(...)

*	ActivityManagerService . `newProcessRecordLocked`(ApplicationInfo info, String customProcess, boolean isolated, int isolatedUid)
*	ActivityManagerService . `startProcessLocked`(...)


`startSpecificActivityLocked`方法会创建应用程序进程，在 `startProcessLocked`方法内部根据进程名称和用户 ID 检查应用程序是否存在，不存在，则调用了`newProcessRecordLocked`方法创建了一个`ProcessRecord`对象，它包含了进程的基本信息，根据这个 `ProcessRecord`对象，在`startProcessLocked`函数的另一形式中去创建了应用程序进程。

当应用程序进程创建成功后，会得到一个大于 0 的进程 ID ，而 ActivityManagerService 就以这个 ID 为关键字将这个进程对应的 ProcessRecord 保存在变量`mPidsSelfLocked`中，以便 ActivityManagerService 可以根据这个 ID 找到该进程的记录信息。

当进程创建成功后，便会开始运行了，而 Java 程序的入口就是 `main`函数了。

## Activity 应用程序进程启动


*	ActivityThread . `main`(...)

*	ActivityThread . `attach`(boolean system)
*	ActivityManagerService . `attachApplication`(IApplicationThread thread)
*	ActivityManagerService . `attachApplicationLocked`(IApplicationThread thread, int pid)
*	ApplicationThread . `bindApplication`(...)
*	ActivityThread . `handleBindApplication`(AppBindData data)
*	StackSupervisor . `attachApplicationLocked`(ProcessRecord app)
*	StackSupervisor .  `realStartActivityLocked`(ActivityRecord r, ProcessRecord app, boolean andResume, boolean checkConfig)
*	ApplicationThread . `scheduleLaunchActivity`(...)
*	ActivityThread . `handleLaunchActivity`(ActivityClientRecord r, Intent customIntent)
*	ActivityThread . `performLaunchActivity`(ActivityClientRecord r, Intent customIntent)
*	ActivityThread . `handleResumeActivity`(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume)
*	ActivityThread . `performResumeActivity`(IBinder token, boolean clearHide) 

由于 Process 类的 start 方法指定了 ActivityThread 类的 `main` 方法作为一个进程的开始点，跟进 main 方法，初始化线程消息循环，并且在 `attach`方法中向 ActivityManagerService 跨进程发送请求，要求 ActivityManagerService 将当前进程添加 Application ，毕竟这还是个单纯的进程，并不是 Android 进程。

于是 ActivityManagerService 执行了 `attachApplication`方法响应这个跨进程请求，接着又继续调用了 `attachApplicationLocked`方法，从`mPidsSelfLocked`变量中根据 PID 找到 Activity 应用程序进程对应的 `ProcessRecord` 。然后将`ProcessRecord`绑定到应用进程，将`ApplicationThread`赋值给`ProcessRecord`的成员变量`thread`，这样就能 ActivityManagerService 就能通过它来和 Activity 通信了。

接下来 ActivityManagerService 又发起跨进程调用，通过 `ApplicationThread`的`bindApplication`方法。ActivityThread 主线程消息循环收到 Handler 发送的消息，响应方法为`handleBindApplication`。在此方法内，创建了Application 运行环境上下文`Context`、监控程序与系统交互的`Instrumentation`、应用程序的`Application`、并且调用了`Application`的 onCreate 方法。

当 ActivityManagerService 将 Activity 的 Application 创建完成后，又开始调度 Activity 的生命周期了。通过 StackSupervisor 类的 `attachApplicationLocked`方法，找到位于栈顶的 `ActivityRecord`对象，检查这个 Activity 组件的用户 ID 和进程名是否与 ProcessRecord 对象所描述的一致，如果一致，则调用`realStartActivityLocked`真正的开启一个 Activity 组件了。


 开启一个 Activity 组件的生命周期当然是在应用程序进程内了，此时又是 ActivityManagerService 的跨进程通信，执行 ApplicationThread 的 `scheduleLaunchActivity`，ActivityThread 的消息循环响应方法为 `handleLaunchActivity`。在该方法内又继续执行 `performLaunchActivity`将 Activity 组件启动起来。通过反射创建了一个 Activity 对象，并且创建了 Activity 的运行环境上下文，通过 `Instrumentation` 执行了 Activity 的 onCreate 方法、onStart 方法。接着又执行了`handleResumeActivity`方法让 Activity 运行到 `Resumed`状态，真正执行操作的是`performResumeActivity`方法，而最终还是调用 `Instrumentation`类的`callActivityOnResume`方法。

至此，一个 Activity 组件就已经创建成功了，里面具体的代码细节暂不深入讨论了，比如 Activity 的加载之类的。










