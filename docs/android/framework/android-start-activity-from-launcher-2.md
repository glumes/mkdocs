---
title: "Android 6.0 Launcher 启动 Activity 过程源码分析（二）"
date: 2017-12-22T10:40:42+08:00
categories: ["Android"]
tags: ["Android"]
toc: true
original: true
addwechat: true
 
slug: "android-start-activity-from-launcher-2"
image: "/banner/robotic-5714849.svg"
---


在 [Android 6.0 Launcher 启动 Activity 过程源码分析（一）](https://glumes.com/android-start-activity-from-launcher-1) 分析完了 Launcher 组件中启动的步骤，接下来的环节是该 ActivityManagerService 出场了。

通过 ActivityManagerNative.getDefault() 方法得到 ActivityManagerService 的代理对象后执行的 startActivity 方法，最终会发起进程间通信请求，通过 Binder 驱动，再调用 ActivityManagerService 中对应的方法。

<!--more-->

### ActivityManagerService 类的 startActivity()方法
``` java
 @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }
```

显然，最后调用了 `startActivityAsUser` 方法：

### ActivityManagerService 类的 startActivityAsUser()方法

``` java
@Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        // 相关验证过程
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        // 
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
```
在 `startActivityAsUser`先是执行了相关的验证过程，然后调用了类型为 `ActivityStackSupervisor`的`mStackSupervisor`的`startActivityMayWait()`方法。

### ActivityStackSupervisor 类的 startActivityMayWait() 方法

Android 中的 Activity 组件堆栈信息，也就是 Task，是用 `ActivityStack` 类管理的，而`ActivityStackSupervisor`则是一个管理所有的`ActivityStack`的类。

``` java
    final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
	    // 省略部分代码
     
        // Collect information about the target of the Intent.
        // 通过 PackageManagerService 解析 Intent 参数内容，获得更多信息，保存到 ActivityInfo 类中
        ActivityInfo aInfo =
                resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);
		
		// 省略部分关于 aInfo 代码

            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);

            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }

// resolveActivity 函数
    ActivityInfo resolveActivity(Intent intent, String resolvedType, int startFlags,
            ProfilerInfo profilerInfo, int userId) {
        // Collect information about the target of the Intent.
        ActivityInfo aInfo;
        try {
            ResolveInfo rInfo =
                AppGlobals.getPackageManager().resolveIntent(
                        intent, resolvedType,
                        PackageManager.MATCH_DEFAULT_ONLY
                                    | ActivityManagerService.STOCK_PM_FLAGS, userId);
            aInfo = rInfo != null ? rInfo.activityInfo : null;
        } catch (RemoteException e) {
            aInfo = null;
        }
        // 省略部分代码
        
    }
```


首先会先调用`resolveActivity`函数，通过 PackageManagerService 解析 Intent 得到更多信息，返回`ActivityInfo`对象之后，便执行 `startActivityLocked`方法。

### ActivityStackSupervisor 类的 startActivityLocked() 方法

``` java
final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;
		ProcessRecord callerApp = null;
        if (caller != null) {
            callerApp = mService.getRecordForAppLocked(caller); // 得到调用者的 ProcessRecord
            if (callerApp != null) {
                callingPid = callerApp.pid;  // 调用者的 PID
                callingUid = callerApp.info.uid; // 调用者的 UID 
            } else {
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }
		
		// 省略部分代码 ，得到描述 Launcher 组件的 ActivityRecord 对象，保存在变量 sourceRecord 中
		ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            sourceRecord = isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                    "Will send result to " + resultTo + " " + sourceRecord);
            if (sourceRecord != null) { 
            // 从 Launcher 启动，requestCode 为 -1 ，下面的if不成立，resultRecord 为 null 。
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }
		
		// 创建用来描述被启动的 Activity 组件的 ActivityRecord 。
		ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage, intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null, this, container, options);
                	
		err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);
}

// 根据 token 找到对应的 ActivityRecord 变量，相当于一个数组变量里面的每个元素又持有一个数组。
ActivityRecord isInAnyStackLocked(IBinder token) {
		// mActivityDisplays 变量的类型为 SparseArray<ActivityDisplay>
        int numDisplays = mActivityDisplays.size();
        for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
        // 内部持有一个 ArrayList<ActivityStack> 类型的变量 mStacks
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityRecord r = stacks.get(stackNdx).isInStackLocked(token);
                if (r != null) {
                    return r;
                }
            }
        }
        return null;
    }
```

每一个应用程序进程都使用一个 `ProcessRecord`对象来描述，并且保存在 ActivityManagerService 内部。ActivityStackSupervisor 的 mService 变量指向了ActivityManagerService 。通过它的成员函数`getRecordForAppLocked`来获得参数`caller`对应的一个`ProcessRecord`对象`callerApp`。而参数`caller`指向的是 Launcher 组件所运行的应用程序进程的一个 ApplicationThread 对象，因此，`ProcessRecord`对象 `callerApp`实际上就指向了 Launcher 组件所运行的应用程序进程，接着得到这个应用程序进程的 PID  和 UID ，保存在参数`callingPid`和`callingUid`中。

ActivityStackSupervisor 变量内部有一个变量 mActivityDisplays，类型为`SparseArray<ActivityDisplay>`,而`ActivityDisplay`变量内部又持有一个`mStack`变量，类型为`ArrayList<ActivityStack>`，通过找到Activity 组件堆栈 `ActivityStack`，从而得到用来描述`Launcher`组件的`ActivityRecord`对象，保存在变量`sourceRecord`中，而由于 requestCode 为 -1 ，则 `resultRecord`继续为 null 。


最后，创建了一个 ActivityRecord 对象`r`用来描述即将启动的 Activity 组件。

现在，就已经得到请求启动 Activity 组件的 Launcher ActivityRecord 信息`sourceRecord`以及需要启动的 Activity 的组件信息`r`，下一步就执行`startActivityUncheckedLocked`操作。

### ActivityStackSupervisor 类的 startActivityUncheckedLocked() 方法

``` java
final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags, boolean doResume, Bundle options, TaskRecord inTask) {
		final Intent intent = r.intent;
        final int callingUid = r.launchedFromUid;
        // 得到目标 Activity 组件的启动标志位
		int launchFlags = intent.getFlags(); 
		
        // We'll invoke onUserLeaving before onPause only if the launching
        // activity did not explicitly state that this is an automated launch.
        mUserLeaving = (launchFlags & Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;

		boolean addingToTask = false; // 是否将需要启动的 Activity 添加到给定的 task 中
        TaskRecord reuseTask = null; // 

        // If the caller is not coming from another activity, but has given us an
        // explicit task into which they would like us to launch the new activity,
        // then let's see about doing that.
        // 如果调用者
        if (sourceRecord == null && inTask != null && inTask.stack != null) {
		 // 省略部分代码
         // If task is empty, then adopt the interesting intent launch flags in to the
         // activity being started.
            if (root == null) {
                final int flagsOfInterest = Intent.FLAG_ACTIVITY_NEW_TASK
                        | Intent.FLAG_ACTIVITY_MULTIPLE_TASK | Intent.FLAG_ACTIVITY_NEW_DOCUMENT
                        | Intent.FLAG_ACTIVITY_RETAIN_IN_RECENTS;
                launchFlags = (launchFlags&~flagsOfInterest)
                        | (baseIntent.getFlags()&flagsOfInterest);
                intent.setFlags(launchFlags);
                inTask.setIntent(r);
                addingToTask = true;
        }
		
		// 省略启动 Activity 已经启动过的情况，主要是将 Activity 移植 Task 栈顶
		// 启动一个从未启动过的 Activity 
		boolean newTask = false; // 表示在一个新的 Task 中启动 Activity
        boolean keepCurTransition = false;
        
        // Should this be considered a new task?
        // 是否要创建一个新的 Task ，当然是要的
        if (r.resultTo == null && inTask == null && !addingToTask
                && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
            targetStack = computeStackFocus(r, newTask);
            targetStack.moveToFront("startingNewTask");

            if (reuseTask == null) {  // reuseTask 为 null，创建一个新的 Task 
                r.setTask(targetStack.createTaskRecord(getNextTaskId(),
                        newTaskInfo != null ? newTaskInfo : r.info,
                        newTaskIntent != null ? newTaskIntent : intent,
                        voiceSession, voiceInteractor, !launchTaskBehind /* toTop */),
                        taskToAffiliate);
                if (DEBUG_TASKS) Slog.v(TAG_TASKS,
                        "Starting new activity " + r + " in new task " + r.task);
            } else {
                r.setTask(reuseTask, taskToAffiliate);
            }
            if (isLockTaskModeViolation(r.task)) {
                Slog.e(TAG, "Attempted Lock Task Mode violation r=" + r);
                return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
            if (!movedHome) {
                if ((launchFlags &
                        (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                    // Caller wants to appear on home activity, so before starting
                    // their own activity we will bring home to the front.
                    r.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE);
                }
            }
        } else if (sourceRecord != null) {
			// 上面的 if 判断成立，省略代码
			// 在调用者的 Task 中做操作 ，resumeTopActivityLocked 方法
		} else if (inTask != null) {
			// 上面的 if 判断成立，省略代码
			// 在指定的 Task 中做操作，
		} else {
			// 上面的 if 判断成立，省略代码
		}
			
		mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,
                intent, r.getUriPermissionsLocked(), r.userId);

        if (sourceRecord != null && sourceRecord.isRecentsActivity()) {
            r.task.setTaskToReturnTo(RECENTS_ACTIVITY_TYPE);
        }
        if (newTask) {
            EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, r.userId, r.task.taskId);
        }
        ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
        targetStack.mLastPausedActivity = null;
        // 启动 Activity 组件的下一步
        targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
        if (!launchTaskBehind) {
            // Don't set focus on an activity that's going to the back.
            mService.setFocusedActivityLocked(r, "startedActivity");
        }
        return ActivityManager.START_SUCCESS;
}
```
	
检查 launchFlags 的 `Intent.Flag_ACTIVITY_NO_USER_ACTION` 是否等于 1 。如果等于 1，则表示目标 Activity 组件不是由用户手动启动的。如果目标 Activity 组件是由用户手动启动的，那么用来启动它的源 Activity 就会获得一个用户离开事件通知。

由于是从 Launcher 启动 Activity 组件，则 `Flag_ACTIVITY_NO_USER_ACTION`等于 0 ，mUserLeaving 为 True ，表示后面要向 Launcher 组件发送一个用户离开事件通知。

由于从 Launcher 启动的 Activity 运行在另一个 Task 中，则 addingToTask 为 `false` ，同时 `reuseTask`也是为 `null` 的，`inTask`为 `null`，并且 `r.resultTo`是一个 ActivityRecord 类型，由于 Activity 组件还没启动，也是为 `null`。所以，最上面的 if 判断成立，直接创建一个新的 Task 了。


### ActivityStack 类的 startActivityLocked() 方法


``` java
    final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
		TaskRecord rTask = r.task;
        final int taskId = rTask.taskId;
        // mLaunchTaskBehind tasks get placed at the back of the task stack.
        if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
            // Last activity in task had been removed or ActivityManagerService is reusing task.
            // Insert or replace.
            // Might not even be in.
            insertTaskAtTop(rTask, r);
            mWindowManager.moveTaskToTop(taskId);
        }
		
		 if (!newTask) {
			 // newTask 为 true ，省略部分代码
		 }
		 
		// Place a new activity at top of stack, so it is next to interact
        // with the user.

        // If we are not placing the new activity frontmost, we do not want
        // to deliver the onUserLeaving callback to the actual frontmost activity 
        // task 变量为 null，尚未赋值， IF 判断不成立
        if (task == r.task && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
            mStackSupervisor.mUserLeaving = false;
            if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                    "startActivity() behind front, mUserLeaving=false");
        }

		task = r.task;
		// Slot the activity into the history stack and proceed
        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
                new RuntimeException("here").fillInStackTrace());
        task.addActivityToTop(r);
        task.setFrontOfTask();
		 r.putInHistory();

		// 省略 Window 添加部分代码
        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
// 将 ActivityRecord 添加到栈的方法
void addActivityToTop(ActivityRecord r) {
        addActivityAtIndex(mActivities.size(), r);
}
```

在上一步中，已经通过 `r.setTask()`方法创建了一个新的 Task，并且 `newTask`变量为 `true` 。当 Activity 是在一个新的 Task 中启动时，需要将它放到 TaskRecord 的中，并且位于堆栈的最上方。

当添加完之后，便继续执行 `resumeTopActivitiesLocked` 方法。

### ActivityStackSupervisor 类的 resumeTopActivitiesLocked() 方法

``` java
boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
	        // 调用 ActivityStack 类的 resumeTopActivityLocked 方法
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    // Already started above.
                    continue;
                }
                if (isFrontStack(stack)) {
	                // 调用 ActivityStack 类的 resumeTopActivityLocked 方法，参数为 null
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }
```
ActivityStackSupervisor 类重载了两种形式的 `resumeTopActivitiesLocked`方法，主要就是将所有ActivityStack栈顶的ActivityRecord迁移到显示状态，都是调用 `ActivityStack`类的`resumeTopActivityLocked`方法，只不过参数略有不同了。


###  ActivityStack 类的 resumeTopActivityLocked() 方法

``` java
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
```

在该方法内部最后调用了`resumeTopActivityInnerLocked`方法。

### ActivityStack 类的 resumeTopActivityInnerLocked() 方法
``` java
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
		 // Find the first activity that is not finishing.
		 // 找到当前 ActivityRecord 的栈顶，指向了要启动的 Activity 组件。
        final ActivityRecord next = topRunningActivityLocked(null);
		final TaskRecord prevTask = prev != null ? prev.task : null;
		
		if (next == null) {
			进入该分支表示没有要启动的 Activity 。
		}
		// 省略部分代码
		// If the top activity is the resumed one, nothing to do.
		// 检查要启动的 Activity 组件是否等于当前被激活的 Activity 组件，如果等于，并且处于 RESUMED 状态，直接返回
        if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                    mStackSupervisor.allResumedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            return false;
        }
		
		// If we are sleeping, and there is no resumed activity, and the top
        // activity is paused, well that is the state we want.
        // 检查要启动的 Activity 组件是否等于上一次被中止了的 Activity 组件，如果等于，
        // 并且这时候系统正要进入关机或睡眠状态，则直接退出，启动毫无意义
        if (mService.isSleepingOrShuttingDown()
                && mLastPausedActivity == next
                && mStackSupervisor.allPausedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            return false;
        }

		// If we are currently pausing an activity, then don't do anything until that is done.
        // 检查系统中止 Activity 组件是否完成，如果没有，则直接返回了，等待所有的 Activity 进入中止状态
        if (!mStackSupervisor.allPausedActivitiesComplete()) {
            return false;
        }

		// We need to start pausing the current activity so the top one can be resumed...
        // Launcher 组件进入 onPause 状态
        boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
        if (mResumedActivity != null) {
        // mResumedActivity 指向了 Launcher 组件，不为 null ，则中止 Launcher 组件
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }
		// 省略部分代码
		
		if (next.app != null && next.app.thread != null) {
			// 待启动的 Activity 在新的进程中，app 变量为 null 
		} else {
			// 创建一个新的应用程序进程
			mStackSupervisor.startSpecificActivityLocked(next, true, true);
		}
		
    }

```

ActivityStack 类有三个成员变量：`mResumedActivity`、`mLastPausedActivity`、`mPausingActivity`，它们的类型均为 `ActivityRecord`，分别用来描述系统当前激活的 Activity 组件、上一次被中止的 Activity 组件，以及正在被中止的 Activity 组件。

而在`resumeTopActivityInnerLocked`方法中，待其的 Activity 的 ActivityRecord 已经位于栈顶了，需要将它运行到 `Resumed` 状态，而在这之前需要判断满足很多条件，比如当前所有的 Activity 组件要处于 `onPaused`状态，Launcher 组件要处于`onPaused`状态，否则会直接 return 退出了。

当满足上面的条件时，最后就是判断待启动的 Activity 组件的应用程序进程是否创建，如果还没有，则通过`startSpecificActivityLocked`创建一个应用程序进程来启动 Activity 组件。

	
## 涉及到的其他类

*	ActivityStack
*	ActivityStackSupervisor
*	ActivityDisplay
*	ActivityRecord
*	ActivityInfo
*	TaskRecord


## 参考
1、 Android 6.0 源码 
2、《Android 系统源代码情景分析》
3、http://duanqz.github.io/2016-07-29-Activity-LaunchProcess-Part1

## 疑问

*	Activity 的 堆栈 ActivityStack 