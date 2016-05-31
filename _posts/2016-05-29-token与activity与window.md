---
layout:     post
title:      "token与activity与window"
subtitle:   ""
date:       2016-05-29
author:     "goIntoAction"
header-img: "img/post-bg.jpg"
tags:
    - android源码分析
---

Token是令牌的意思，我们访问网络的时候就有token的概念，其实Android系统中也有。Android中Token是ActivityRecord的静态内部类

    static class Token extends IApplicationToken.Stub {
            private final WeakReference<ActivityRecord> weakActivity;
            private final ActivityManagerService mService;
            ......
	}

从它现实了Stub，可以看出它其实也是Binder的一种。那么它的作用是什么呢？要说它的作用，需要先从ActivityRecord说起，我们知道在Android中AMS和应用是服务端与客户端的关系，在AMS帮应用维护Activity的生命周期、任务栈，所以在AMS中有一个List<ActivityRecord>来保存启动的Activity的记录，每个Activity就对应一个ActivityRecord。所以在Activity启动过程中，一个对应的ActivityRecord也会被AMS创建。
在ActivityStackSupervisor的startActivityLocked方法中会创建ActivityRecord，startActivityLocked最终调用者是AMS。

    final int startActivityLocked(IApplicationThread caller,
                Intent intent, String resolvedType, ActivityInfo aInfo,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                IBinder resultTo, String resultWho, int requestCode,
                int callingPid, int callingUid, String callingPackage,
                int realCallingPid, int realCallingUid, int startFlags, Bundle options,
                boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
                ActivityContainer container, TaskRecord inTask) {
            ......
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                    intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                    requestCode, componentSpecified, voiceSession != null, this, container, options);
      		......
	}
而在ActivityRecord的构造方法中又会创建Token。

	ActivityRecord(ActivityManagerService _service, ProcessRecord _caller,
            int _launchedFromUid, String _launchedFromPackage, Intent _intent, String _resolvedType,
            ActivityInfo aInfo, Configuration _configuration,
            ActivityRecord _resultTo, String _resultWho, int _reqCode,
            boolean _componentSpecified, boolean _rootVoiceInteraction,
            ActivityStackSupervisor supervisor,
            ActivityContainer container, Bundle options) {
        service = _service;
        appToken = new Token(this, service);
		......
        }

在启动Activity过程中，AMS需要与WMS通信，让WMS去创建窗口，为了能让Activity与Window关联起来，AMS会将Token传递给WMS。这一步骤调用顺序是
ActivityStackSupervisor.startActivityLocked() → ActivityStackSupervisor.startActivityUncheckedLocked() → ActivityStack.startActivityLocked()

	final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
            ......
             mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                                r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                                (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0,
                                r.userId, r.info.configChanges, task.voiceSession != null,
                                r.mLaunchTaskBehind);
			......
     }


WMS又会使用此Token构造AppWindowToken，并保存起来。

	AppWindowToken atoken = findAppWindowToken(token.asBinder());
            if (atoken != null) {
                Slog.w(TAG, "Attempted to add existing app token: " + token);
                return;
            }
    atoken = new AppWindowToken(this, token, voiceInteraction);


接着Activity启动流程会走到ActivityStackSupervisor.realStartActivityLocked。在此，AMS会通过Binder远程调用应用的ActivityThread.scheduleLaunchActivity，将Token传递到应用进程中。

final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
            	......
                app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
                ......
            }


至此AMS、WMS和应用进程中都有可以代表一个Activity的token，所以在三者中就可以做到activity的生命周期回调和窗口切换。例如我们在finish一个Activity的时候，ActivityThread会远程调用AMS的，其中一个参数就是token。

	public final boolean finishActivity(IBinder token, int resultCode, Intent resultData,
            boolean finishTask) {
            	......
            }

待AMS处理完后，才会回调Activity的生命周期方法。