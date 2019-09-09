虽然Android 10出了，但源码还在拉，不知道有啥新变化没，看了几天的Android 9.0的Activity启动流程。看源码真是枯燥乏味的，多看几次之后，总能发现点在应用层的东西，挺有味道的。阅读本文也需要很棒的耐心，毕竟我不会逗你开心。

这个过程主要分析，从代码调用了`startActivity（）`启动一个Activity的过程。虽然我们只是传递了intent参数，但源码需要经过精心的准备，才能通知ActivityManagerService(AMS)去启动Activity。

启动一个Activity的代码最后都会调用Activity的下面相关代码：
```
//Activity.java

@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
    
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
    
//我们在子Activity调用该方法，当requestCode>=0会返回结果，但如果Intent使用FLAG_ACTIVITY_NEW_TASK标志,
//会收到RESULT_CANCELED的响应码，即不会有结果返回
//同时，一些特殊的Intent，例如Intent带有ACTION_VIEW，也不会返回结果
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {//判断父Activity是否null，首次启动为null
        options = transferSpringboardActivityOptions(options);
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
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```
**ActivityMonitor**：Instrumention的静态内部类，用来监视Activity的启动过程，可通过Intent拦截Activity的启动或者寻找指定的Activity。Instrumention有一个持有ActivityMonitor元素的集合`mActivityMonitors`，一个Activity被启动后会创建一个新的ActivityMonitor加入到`mActivityMonitors`。
```
//Instrumention.Java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, String resultWho,
            Intent intent, int requestCode, Bundle options, UserHandle user) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        //遍历mActivityMonitors集合，判断是否拦截启动Activity的Intent
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    //默认为true,表示拦截Activity的启动
                    if (am.ignoreMatchingSpecificIntents()) {
                        //默认返回null,表示不拦截Activity的启动，子类可以返回非空来拦截
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) { //match默认为返回false
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            //调用AMS的startActivityAsUser（）
            int result = ActivityManager.getService()
                .startActivityAsUser(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, resultWho,
                        requestCode, 0, null, options, user.getIdentifier());
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```
通过ActivityManager.getService()来获取ActivityManagerService（AMS）在本地的代理对象，通过该对象进行[Binder跨进程通信](https://juejin.im/post/5d18ed91e51d45776031b03d)调用远程的ActivityManagerService（AMS）的`startActivityAsUser（）`函数。

**ActivityManager**：主要提供Activity和Service的相关信息，Activity之间和Service之间的交互过程。
```
//ActivityManager.java

public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                //通过ServiceManager获取远程AMS
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```
接着应该分析`AMS`相关操作流程，但此次我们假设`AMS`对象的`startActivityAsUser`能正常返回`result`,接着分析对`result`处理过程。
```
//Instrumentation.java
    public static void checkStartActivityResult(int res, Object intent) {
        //判断AMS启动Activity是否发生致命错误
        if (!ActivityManager.isStartResultFatalError(res)) {
            return;
        }

        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                            + ((Intent)intent).getComponent().toShortString()
                            + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);
            case ActivityManager.START_PERMISSION_DENIED:
                throw new SecurityException("Not allowed to start activity "
                        + intent);
            case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
                throw new AndroidRuntimeException(
                        "FORWARD_RESULT_FLAG used while also requesting a result");
            case ActivityManager.START_NOT_ACTIVITY:
                throw new IllegalArgumentException(
                        "PendingIntent is not an activity");
            case ActivityManager.START_NOT_VOICE_COMPATIBLE:
                throw new SecurityException(
                        "Starting under voice control not allowed for: " + intent);
            case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
                throw new IllegalStateException(
                        "Session calling startVoiceActivity does not match active session");
            case ActivityManager.START_VOICE_HIDDEN_SESSION:
                throw new IllegalStateException(
                        "Cannot start voice activity on a hidden session");
            case ActivityManager.START_ASSISTANT_NOT_ACTIVE_SESSION:
                throw new IllegalStateException(
                        "Session calling startAssistantActivity does not match active session");
            case ActivityManager.START_ASSISTANT_HIDDEN_SESSION:
                throw new IllegalStateException(
                        "Cannot start assistant activity on a hidden session");
            case ActivityManager.START_CANCELED:
                throw new AndroidRuntimeException("Activity could not be started for "
                        + intent);
            default:
                throw new AndroidRuntimeException("Unknown error code "
                        + res + " when starting " + intent);
        }
    }
    //  -100<result<-1为致命错误
    public static final boolean isStartResultFatalError(int result) {
        return FIRST_START_FATAL_ERROR_CODE <= result && result <= LAST_START_FATAL_ERROR_CODE;
    }
```
上面代码很简单，只是判断返回的result是否存在异常或错误。

**ActivityManagerService**：是Android中最核心的服务，主要负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作。

APP进程通过Binder跨进程通信机制调用了AMS对象的`startActivityAsUser()`函数。
```
//ActivitManagerService.java
    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }
    
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivity");

        userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();

    }
```
通过ActivityStartController对象获得一个ActivityStarter对象来执行Activity的启动。

**ActivityStartController**：被委托用来启动Activity，主要是启动外部（应用端）App的启动Activity的请求和以及将Activity交给ActivityStarter处理前的一些准备。

**ActivityStarter**：Activity启动的地方，根据启动Activity的Intent和flags,来决定如何启动Activity和关联到相应的任务栈。

通过ActivityStartController对象获得一个ActivityStarter对象:
```
//ActivityStartController.java
    //使用默认的工厂模式来获取ActivityStarter对象
    ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }
    
    ActivityStartController(ActivityManagerService service) {
        this(service, service.mStackSupervisor,
                new DefaultFactory(service, service.mStackSupervisor,
                    new ActivityStartInterceptor(service, service.mStackSupervisor)));
    }
//ActivityStarter.java ->ActivityStarter#DefaultFactory
        @Override
        public ActivityStarter obtain() {
            //通过启动池获取，max=3
            ActivityStarter starter = mStarterPool.acquire();
            if (starter == null) {
                starter = new ActivityStarter(mController, mService, mSupervisor, mInterceptor);
            }
            return starter;
        }
```
获得ActivityStarter对象之后，主要设置一些变量，例如`setMayWait(userId)`，然后调用`execute()`函数执行启动流程。
```
//ActivityStarter.java

    ActivityStarter setMayWait(int userId) {
        mRequest.mayWait = true;
        mRequest.userId = userId;
        return this;
    }
    
    int execute() {
        try {
            //在setMayWait()函数中将该条件设为true
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            }
        } finally {
            onExecutionComplete();
        }
    }
```
由于`mayWait`为`true`,会调用`startActivityMayWait()`函数。该函数的代码量非常大，如果看着累可以跳过。
```
    private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {
            ......
            int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                    ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                    allowPendingRemoteAnimationRegistryLookup);

            ......
            return res;
        }
    }
```
在上面代码中涉及到下面对象：

**ActivityStackSupervisor**：顾名思义，ActivityStack的监管。内部持有相关`ActivityStack`，相当于`ActivityStack`的辅助管理类。`mHomeStack`：Launcher的ActivityStatck;`mFocusedStack`：当前获得焦点（输入事件）的栈或启动下一个Activity的栈,可以理解为当前显示在界面Activity所在的栈；`mLastFocusedStack`：上次获得焦点的栈。

**ActivityStack**： 管理和描述一系列Activity的栈。至于Activity放在什么栈，就看到Android的启动模式和相关标志位了。

在`startActivityMayWait()`函数又调用`startActivity()`函数。
```
//ActivityStater.java

    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {

        if (TextUtils.isEmpty(reason)) {
            throw new IllegalArgumentException("Need to specify a reason.");
        }
        mLastStartReason = reason;
        mLastStartActivityTimeMs = System.currentTimeMillis();
        mLastStartActivityRecord[0] = null;

        mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                inTask, allowPendingRemoteAnimationRegistryLookup);

        if (outActivity != null) {
            // mLastStartActivityRecord[0] is set in the call to startActivity above.
            outActivity[0] = mLastStartActivityRecord[0];
        }

        return getExternalResult(mLastStartActivityResult);
    }
    
    
    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup) {
            ......
        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true /* doResume */, checkedOptions, inTask, outActivity);
    }

    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity) {
        int result = START_CANCELED;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            //流程的走向
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
        } finally {
            final ActivityStack stack = mStartActivity.getStack();
            if (!ActivityManager.isStartResultSuccessful(result) && stack != null) {
                stack.finishActivityLocked(mStartActivity, RESULT_CANCELED,
                        null /* intentResultData */, "startActivity", true /* oomAdj */);
            }
            mService.mWindowManager.continueSurfaceLayout();
        }

        postStartActivityProcessing(r, result, mTargetStack);

        return result;
    }
```
可以看到`startActivity()`函数调用了自身两次重载方法，接着会调用`startActivityUnchecked()`函数来处理启动模式和一些标志位。
```
//ActivityStarter.java

    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {

        .....
    mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
    .....    
     return START_SUCCESS;
}
```
处理完相关逻辑后，会调用`ActivityStatckSupervisor`对象的`resumeFocusedStackTopActivityLocked()`函数。
```
//ActivityStatckSupervisor.java

    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!readyToResume()) {
            return false;
        }

        //该条件为true
        if (targetStack != null && isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || !r.isState(RESUMED)) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.isState(RESUMED)) {
            mFocusedStack.executeAppTransition(targetOptions);
        }

        return false;
    }
```
这里主要调用了ActivityStatck对象的`resumeTopActivityUncheckedLocked()`函数。
```
//ActivityStatck.java

    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mStackSupervisor.inResumeTopActivity) {
            return false;
        }

        boolean result = false;
        try {
            mStackSupervisor.inResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }

        return result;
    }
```
该函数主要调用了`resumeTopActivityInnerLocked()`函数和`topRunningActivityLocked()`函数。
```
//ActivityStatck.java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    .....
        
    final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);

    ......
    
     mStackSupervisor.startSpecificActivityLocked(next, true, true);
}

```
改函数非常的长，主要判断是否有Activity处于Resume状态，如果有的话会先执行Pausing过程，然后通过`ActivityStatckSupervisor`对象的`startSpecificActivityLocked()`函数来启动目标Activity。
```
//ActivityStatckSupervisor.java
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
            
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        getLaunchTimeTracker().setLaunchTime(r);
        //判断APP进程是否存在，因为应用内启动是存在的
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                            mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
        }

        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
    
    
```
接着又调用了`realStartActivityLocked()`函数。
```
//ActivityStatckSupervisor.java
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
            ......
             //记住这里将app.thread进程传递进去了
             final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
            //添加callback添加LaunchActivityItem元素
            clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                    System.identityHashCode(r), r.info,
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                    r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                    profilerInfo));

            final ActivityLifecycleItem lifecycleItem;
            if (andResume) {
                lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
            } else {
                lifecycleItem = PauseActivityItem.obtain();
            }
            clientTransaction.setLifecycleStateRequest(lifecycleItem);

            mService.getLifecycleManager().scheduleTransaction(clientTransaction);
            ......
}
```
通过代码来看，Android 9.0通过事务机制来启动Activity。我们看一下`ClientLifecycleManager`对象的`scheduleTransaction()`函数。
```
//ClientLifecycleManager.java

    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            transaction.recycle();
        }
    }
    
//ClientTransaction.java
    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }
    
   private IApplicationThread mClient;
   
    public static ClientTransaction obtain(IApplicationThread client, IBinder activityToken) {
        ClientTransaction instance = ObjectPool.obtain(ClientTransaction.class);
        if (instance == null) {
            instance = new ClientTransaction();
        }
        instance.mClient = client;
        instance.mActivityToken = activityToken;

        return instance;
    }
```
`ClientTransaction`对象内的`mClient`变量在调`ActivityStatckSupervisor`对象的`realStartActivityLocked()`函数赋值为`app.thread`。`app.thread`，是`ActivityThread`的一个内部类`ApplicationThread`。

因此，我们看一下`ApplicationThread`对象的`scheduleTransaction()`函数。
```
//ActivityThread.java #ApplicationThread
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        ActivityThread.this.scheduleTransaction(transaction);
}

//ActivitThread继承自ClientTransactionHandler
//ClientTransactionHandler.java

void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```
`sendMessage()`函数在`ClientTransactionHandler`类是抽象方法，在子类`ActivityThread`实现。`ActivityThread`类中sendMessage()函数有很多重载函数。
```
//ActivityThread.java

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
    
    final H mH = new H();
    
public void handleMessage(Message msg) {
    switch (msg.what) {
        ......
        //msg.what=EXECUTE_TRANSACTION
        case EXECUTE_TRANSACTION:
            final ClientTransaction transaction = (ClientTransaction) msg.obj;
            mTransactionExecutor.execute(transaction);
            if (isSystem()) {
                transaction.recycle();
            }
            break;
       ......
    }
 .....
}
```
到这里调用了`TransactionExecutor`对象的`execute（）`函数。
```
//TransactionExecutor.java

    public void execute(ClientTransaction transaction) {
        final IBinder token = transaction.getActivityToken();
        
        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
    }
    
    public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        if (callbacks == null) {
            // No callbacks to execute, return early.
            return;
        }
        //当前目标Activity的应用的binder
        final IBinder token = transaction.getActivityToken();
        ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

        final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
        final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState()
                : UNDEFINED;

        final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);

        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            final ClientTransactionItem item = callbacks.get(i);
            final int postExecutionState = item.getPostExecutionState();
            final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                    item.getPostExecutionState());
            if (closestPreExecutionState != UNDEFINED) {
                cycleToPath(r, closestPreExecutionState);
            }
            //执行启动Activity
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            
            if (r == null) {
                r = mTransactionHandler.getActivityClient(token);
            }

            if (postExecutionState != UNDEFINED && r != null) {
                final boolean shouldExcludeLastTransition =
                        i == lastCallbackRequestingState && finalState == postExecutionState;
                cycleToPath(r, postExecutionState, shouldExcludeLastTransition);
            }
        }
    }
```
这里主要通过`ClientTransactionItem`对象的`execute（）`函数来执行启动目标`Activity`。`ClientTransactionItem`类是通过回调来让客户端（APP）来执行一些任务，例如Activity配置改变，多窗口模式改变等。`execute()`在其父接口`BaseClientRequest`定义，为此要找到实现该接口的子类。

在调用ActivityStackSupervisor对象的addCallback(),传递了这么个参数：`LaunchActivityItem`
```
//ActivityStatckSupervisor.java #realStartActivityLocked()
  
  clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                    System.identityHashCode(r), r.info,
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                    r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                    profilerInfo));
                    
//LaunchActivityItem.java

    public static LaunchActivityItem obtain(Intent intent, int ident, ActivityInfo info,
            Configuration curConfig, Configuration overrideConfig, CompatibilityInfo compatInfo,
            String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state,
            PersistableBundle persistentState, List<ResultInfo> pendingResults,
            List<ReferrerIntent> pendingNewIntents, boolean isForward, ProfilerInfo profilerInfo) {
        LaunchActivityItem instance = ObjectPool.obtain(LaunchActivityItem.class);
        if (instance == null) {
            instance = new LaunchActivityItem();
        }
        setValues(instance, intent, ident, info, curConfig, overrideConfig, compatInfo, referrer,
                voiceInteractor, procState, state, persistentState, pendingResults,
                pendingNewIntents, isForward, profilerInfo);

        return instance;
    }
    
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```
看到LaunchActivityItem类的execute（）函数，就看到了启动Activity的影子：`client.handleLaunchActivity()`。这里的`client`就是`ActivityThread`对象。
```
//ActivityThread.java

    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
            
        ......

        final Activity a = performLaunchActivity(r, customIntent);
        
        ......
        return a;
    }
    
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }
        //建立Activity的上下文
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //在AppComponentFactory类调用instantiateActivity()函数
            //通过类加载新建Activity实例
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }

                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                //将目标Activity关联到主线程，PhoneWindow创建等
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                //Activit启动的走向
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
            }
            r.setState(ON_CREATE);

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```
上面代码调用了`Instrumentation`对象的`callActivityOnCreate()`函数。
```
//Instrumentation.java

public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    //Activity,判断是否通过同步启动Activity
    prePerformCreate(activity);
    //Activit的启动
    activity.performCreate(icicle, persistentState);
    //Activity启动后，主要ActivityMonitor进行检测是否配对
    postPerformCreate(activity);
}

//Activity.java

    final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        mCanEnterPictureInPicture = true;
        restoreHasCurrentPermissionRequest(icicle);
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        writeEventLog(LOG_AM_ON_CREATE_CALLED, "performCreate");
        mActivityTransitionState.readState(icicle);

        mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
                com.android.internal.R.styleable.Window_windowNoDisplay, false);
        mFragments.dispatchActivityCreated();
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    }
    
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        if (DEBUG_LIFECYCLE) Slog.v(TAG, "onCreate " + this + ": " + savedInstanceState);

        if (mLastNonConfigurationInstances != null) {
            mFragments.restoreLoaderNonConfig(mLastNonConfigurationInstances.loaders);
        }
        if (mActivityInfo.parentActivityName != null) {
            if (mActionBar == null) {
                mEnableDefaultActionBarUp = true;
            } else {
                mActionBar.setDefaultDisplayHomeAsUpEnabled(true);
            }
        }
        if (savedInstanceState != null) {
            mAutoFillResetNeeded = savedInstanceState.getBoolean(AUTOFILL_RESET_NEEDED, false);
            mLastAutofillId = savedInstanceState.getInt(LAST_AUTOFILL_ID,
                    View.LAST_APP_AUTOFILL_ID);

            if (mAutoFillResetNeeded) {
                getAutofillManager().onCreate(savedInstanceState);
            }

            Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
            mFragments.restoreAllState(p, mLastNonConfigurationInstances != null
                    ? mLastNonConfigurationInstances.fragments : null);
        }
        mFragments.dispatchCreate();
        getApplication().dispatchActivityCreated(this, savedInstanceState);
        if (mVoiceInteractor != null) {
            mVoiceInteractor.attachActivity(this);
        }
        mRestoredFromBundle = savedInstanceState != null;
        mCalled = true;
    }
```
到这里，就能看我们老朋友`onCreate()`函数了。

虽然，了解Activity的启动流程，但是其中还有很多的知识点不清楚，触发到知识的盲区...