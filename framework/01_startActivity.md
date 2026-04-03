> 基于 aosp 源码 android15-release  分支分析 startActivity 启动流程

<br/>

# 一、总体流程图

应用程序调用 startActivity
    ↓
ContextImpl.startActivity(Intent)
    ↓
ContextImpl.startActivity(Intent, Bundle)
    ↓
Instrumentation.execStartActivity()
    ↓
ActivityTaskManagerService.startActivity()
    ↓
ActivityTaskSupervisor.startActivityMayWait()
    ↓
ActivityRecord 创建与 Activity 启动
    ↓
ActivityThread.handleLaunchActivity()
    ↓
Activity.onCreate/onStart/onResume

<br/>

# 二、详细流程分析

## 2.1 第一阶段:Context 层 (应用侧)

### 2.1.1 ContextImpl.startActivity(Intent) - 入口方法

```java
@Override
public void startActivity(Intent intent) {
    warnIfCallingFromSystemProcess();  // 警告:如果从系统进程调用
    startActivity(intent, null);        // 委托给重载方法
}
```

<br/>

关键作用:

- 这是所有 startActivity 调用的入口点

- 检查是否从系统进程调用(安全警告)

- 将调用传递给带 Bundle 参数的重载版本

    <br/>

### 2.1.2 ContextImpl.startActivity(Intent, Bundle) - 核心验证与转发

```java
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // 关键验证:非 Activity Context 必须添加 FLAG_ACTIVITY_NEW_TASK
    final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

    if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && (targetSdkVersion < Build.VERSION_CODES.N
                    || targetSdkVersion >= Build.VERSION_CODES.P)
            && (options == null
                    || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity"
                        + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                        + " Is this really what you want?");
    }
    
    // 通过 Instrumentation 执行启动
    mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity) null, intent, -1, options);
}
```

<br/>

关键作用:

- 安全检查: 验证调用者身份

- Task Flag 验证: 如果从 Application/Service 等非 Activity Context 启动,必须添加 FLAG_ACTIVITY_NEW_TASK,否则抛出异常

- 委托给 Instrumentation: 通过 mMainThread.getInstrumentation() 获取 Instrumentation 实例并调用 execStartActivity

    <br/>

<br/>

## 2.2 第二阶段:Instrumentation 层 (监控与拦截)

### 2.2.1 Instrumentation.execStartActivity() - 监控拦截点

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    
    // 1. 检查 ActivityMonitor(用于测试/监控)
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                ActivityResult result = null;
                if (am.ignoreMatchingSpecificIntents()) {
                    result = am.onStartActivity(who, intent, options);
                }
                if (result != null) {
                    am.mHits++;
                    return result;  // 被监控拦截,返回模拟结果
                }
            }
        }
    }
    
    // 2. 调用 ATMS 启动 Activity
    try {
        intent.migrateExtraStreamToClipData(who);
        intent.prepareToLeaveProcess(who);
        int result = ActivityTaskManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);  // 检查启动结果
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

<br/>

- 测试支持: 允许通过 ActivityMonitor 拦截和模拟 Activity 启动(用于单元测试)
- Intent 准备: 迁移数据和准备跨进程传输
- 系统服务调用: 通过 Binder 调用 ActivityTaskManagerService
- 结果检查: 通过 checkStartActivityResult 验证启动是否成功

<br/>

## 2.3 第三阶段:ActivityTaskManagerService 层 (系统服务)

### 2.3.1 ActivityTaskManagerService (ATMS) - 系统服务核心

ATMS 是 Android 10+ 引入的活动任务管理服务,负责管理所有 Activity 的启动和任务栈。

<br/>

启动流程:

ActivityTaskManagerService.startActivity()
    ↓
startActivityAsUser()  // 处理多用户
    ↓
ActivityStarter.execute()  // Activity 启动器
    ↓
startActivityMayWait()  // 处理并发启动
    ↓
startActivity()  // 核心启动逻辑
    ↓
startActivityUnchecked()  // 不检查权限的启动
    ↓
resumeFocusedTasksTopActivities()  // 恢复目标 Activity

<br/>

关键职责:

- Intent 解析: 通过 PackageManager 解析 Intent,找到目标 Activity
- 权限检查: 验证调用者是否有权限启动目标 Activity
- Task 管理: 决定 Activity 应该在哪个 Task 中启动(新建/复用)
- 栈管理: 管理 Activity 返回栈的入栈和出栈
- 进程管理: 如果目标进程未启动,先启动进程

<br/>

#### 2.3.1.1 ActivityTaskManagerService.startActivityAsUser()

```java
@Override
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}

// 实际实现 (第1269-1313行)
private int startActivityAsUser(IApplicationThread caller, String callingPackage,
        @Nullable String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
    
    // ① 解析 ActivityOptions
    final SafeActivityOptions opts = SafeActivityOptions.fromBundle(bOptions);

    // ② 验证调用者包名与 UID 是否匹配
    assertPackageMatchesCallingUid(callingPackage);
    
    // ③ 禁止孤立进程调用
    enforceNotIsolatedCaller("startActivityAsUser");

    // ④ SDK Sandbox 相关安全检查
    if (isSdkSandboxActivityIntent(mContext, intent)) {
        SdkSandboxManagerLocal sdkSandboxManagerLocal = LocalManagerRegistry.getManager(
                SdkSandboxManagerLocal.class);
        sdkSandboxManagerLocal.enforceAllowedToHostSandboxedActivity(
                intent, Binder.getCallingUid(), callingPackage);
    }

    // ⑤ 检查目标用户权限
    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // ⑥ 获取 ActivityStarter 并执行 (链式调用)
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)              // 设置调用者线程
            .setCallingPackage(callingPackage)
            .setCallingFeatureId(callingFeatureId)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)          // 结果返回目标
            .setResultWho(resultWho)
            .setRequestCode(requestCode)    // 请求码
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(opts)
            .setUserId(userId)
            .execute();                      // 执行启动
}
```

- 使用建造者模式配置启动参数
- 通过 execute() 触发实际启动流程

<br/>

#### 2.3.1.2 ActivityStarter.execute()

```java
int execute() {
    try {
        // ① 执行开始回调
        onExecutionStarted();

        // ② 检查 Intent 是否包含文件描述符(安全限制)
        if (mRequest.intent != null) {
            if (mRequest.intent.hasFileDescriptors()) {
                throw new IllegalArgumentException("File descriptors passed in Intent");
            }
            // 移除不匹配的扩展标志
            mRequest.intent.removeExtendedFlags(Intent.EXTENDED_FLAG_FILTER_MISMATCH);
        }

        // ③ 记录启动指标
        final LaunchingState launchingState;
        synchronized (mService.mGlobalLock) {
            final ActivityRecord caller = ActivityRecord.forTokenLocked(mRequest.resultTo);
            final int callingUid = mRequest.realCallingUid == Request.DEFAULT_REAL_CALLING_UID
                    ?  Binder.getCallingUid() : mRequest.realCallingUid;
            launchingState = mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(
                    mRequest.intent, caller, callingUid);
        }

        // ④ 解析 Activity(如果还未解析)
        if (mRequest.activityInfo == null) {
            mRequest.resolveActivity(mSupervisor);  // 调用 PackageManager 解析
        }

        // ⑤ 在锁内执行实际启动
        int res = START_CANCELED;
        synchronized (mService.mGlobalLock) {
            final boolean globalConfigWillChange = mRequest.globalConfig != null
                    && mService.getGlobalConfiguration().diff(mRequest.globalConfig) != 0;
            final Task rootTask = mRootWindowContainer.getTopDisplayFocusedRootTask();
            if (rootTask != null) {
                rootTask.mConfigWillChange = globalConfigWillChange;
            }

            final long origId = Binder.clearCallingIdentity();
            try {
                // ⑥ 检查是否需要切换到重量级进程
                res = resolveToHeavyWeightSwitcherIfNeeded();
                if (res != START_SUCCESS) {
                    return res;
                }
                
                // ⑦ 执行启动请求 (核心方法)
                res = executeRequest(mRequest);
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
        }

        return res;
    } finally {
        onExecutionComplete();  // 清理和回收
    }
}
```

<br/>

#### 2.3.1.3 ActivityStarter.executeRequest : 权限检查与 ActivityRecord 创建

```java
private int executeRequest(Request request) {
    // ① 参数提取
    final IApplicationThread caller = request.caller;
    Intent intent = request.intent;
    ActivityInfo aInfo = request.activityInfo;
    // ... 其他参数

    int err = ActivityManager.START_SUCCESS;
    
    // ② 获取调用者进程
    WindowProcessController callerApp = null;
    if (caller != null) {
        callerApp = mService.getProcessController(caller);
        if (callerApp != null) {
            callingPid = callerApp.getPid();
            callingUid = callerApp.mInfo.uid;
        } else {
            err = START_PERMISSION_DENIED;
        }
    }

    // ③ 检查 Intent 是否可解析
    if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
        err = ActivityManager.START_INTENT_NOT_RESOLVED;
    }

    // ④ 检查 ActivityInfo 是否存在
    if (err == ActivityManager.START_SUCCESS && aInfo == null) {
        err = ActivityManager.START_CLASS_NOT_FOUND;
    }

    // ⑤ 权限检查 - 背景 Activity 启动限制
    boolean abort;
    try {
        abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
                requestCode, callingPid, callingUid, callingPackage, callingFeatureId,
                request.ignoreTargetSecurity, inTask != null, callerApp, resultRecord,
                resultRootTask);
    } catch (SecurityException e) {
        // 处理权限异常
    }
    
    // ⑥ 防火墙检查
    abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
            callingPid, resolvedType, aInfo.applicationInfo);
    
    // ⑦ 创建 ActivityRecord (系统侧的 Activity 记录)
    final ActivityRecord r = new ActivityRecord.Builder(mService)
            .setCaller(callerApp)
            .setLaunchedFromPid(callingPid)
            .setLaunchedFromUid(callingUid)
            .setLaunchedFromPackage(callingPackage)
            .setLaunchedFromFeature(callingFeatureId)
            .setIntent(intent)
            .setResolvedType(resolvedType)
            .setActivityInfo(aInfo)
            .setConfiguration(mService.getGlobalConfiguration())
            .setResultTo(resultRecord)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setComponentSpecified(request.componentSpecified)
            .setRootVoiceInteraction(voiceSession != null)
            .setActivityOptions(checkedOptions)
            .setSourceRecord(sourceRecord)
            .build();

    mLastStartActivityRecord = r;

    // ⑧ 启动 Activity (未检查版本)
    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
            request.voiceInteractor, startFlags, checkedOptions,
            inTask, inTaskFragment, balVerdict, intentGrants, realCallingUid);

    return mLastStartActivityResult;
}
```

<br/>

#### 2.3.1.4 ActivityStarter.startActivityUnchecked: Task 管理与窗口操作

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment,
        BalVerdict balVerdict,
        NeededUriGrants intentGrants, int realCallingUid) {
    
    int result = START_CANCELED;
    final Task startedActivityRootTask;

    // ① 创建过渡动画
    final TransitionController transitionController = r.mTransitionController;
    Transition newTransition = transitionController.isShellTransitionsEnabled()
            ? transitionController.createAndStartCollecting(TRANSIT_OPEN) : null;
    RemoteTransition remoteTransition = r.takeRemoteTransition();
    
    try {
        // ② 延迟窗口布局
        mService.deferWindowLayout();
        transitionController.collect(r);
        
        try {
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
            
            // ③ 核心启动逻辑
            result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, options, inTask, inTaskFragment, balVerdict,
                    intentGrants, realCallingUid);
        } catch (Exception ex) {
            Slog.e(TAG, "Exception on startActivityInner", ex);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
            // ④ 处理启动结果
            startedActivityRootTask = handleStartResult(r, options, result, newTransition,
                    remoteTransition);
        }
    } finally {
        mService.continueWindowLayout();  // 继续窗口布局
    }
    
    // ⑤ 启动后处理
    postStartActivityProcessing(r, result, startedActivityRootTask);

    return result;
}
```

<br/>

#### 2.3.1.5 startActivityInner - Task 分配与恢复

```java
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment, BalVerdict balVerdict,
        NeededUriGrants intentGrants, int realCallingUid) {
    
    // ① 设置初始状态
    setInitialState(r, options, inTask, inTaskFragment, startFlags, sourceRecord,
            voiceSession, voiceInteractor, balVerdict.getCode(), realCallingUid);

    // ② 计算 Launch Flags
    computeLaunchingTaskFlags();
    mIntent.setFlags(mLaunchFlags);

    // ③ 确定可复用的 Task
    final Task reusedTask = resolveReusableTask(includeLaunchedFromBubble);
    
    // ④ 计算目标 Task
    final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
    final boolean newTask = targetTask == null;  // 是否创建新 Task
    mTargetTask = targetTask;

    // ⑤ 计算启动参数
    computeLaunchParams(r, sourceRecord, targetTask);

    // ⑥ 检查是否允许启动
    int startResult = isAllowedToStart(r, newTask, targetTask);
    if (startResult != START_SUCCESS) {
        return startResult;
    }

    // ⑦ 创建新 Task 或添加到现有 Task
    if (targetTask != null) {
        // 检查 Task 权重限制
        if (targetTask.getTreeWeight() > MAX_TASK_WEIGHT_FOR_ADDING_ACTIVITY) {
            targetTask.removeImmediately("bulky-task");
            return START_ABORTED;
        }
    } else {
        mAddingToTask = true;
    }

    // ⑧ 检查是否需要传递给当前顶部 Activity
    final Task topRootTask = mPreferredTaskDisplayArea.getFocusedRootTask();
    if (topRootTask != null) {
        startResult = deliverToCurrentTopIfNeeded(topRootTask, intentGrants);
        if (startResult != START_SUCCESS) {
            return startResult;
        }
    }

    // ⑨ 创建或获取 Root Task
    if (mTargetRootTask == null) {
        mTargetRootTask = getOrCreateRootTask(mStartActivity, mLaunchFlags, targetTask,
                mOptions);
    }
    
    // ⑩ 创建新 Task 或添加到现有 Task
    if (newTask) {
        final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTask() : null;
        setNewTask(taskToAffiliate);  // 创建新 Task
    } else if (mAddingToTask) {
        addOrReparentStartingActivity(targetTask, "adding to task");  // 添加到现有 Task
    }

    // ⑪ 移动到前台
    if (mDoResume) {
        if (!avoidMoveToFront()) {
            mTargetRootTask.getRootTask().moveToFront("reuseOrNewTask", targetTask);
        }
    }

    // ⑫ 授予 URI 权限
    mService.mUgmInternal.grantUriPermissionUncheckedFromIntent(intentGrants,
            mStartActivity.getUriPermissionsLocked());

    // ⑬ 获取 Task 并准备启动
    final Task startedTask = mStartActivity.getTask();
    
    // ⑭ 在 Task 中锁定启动
    mTargetRootTask.startActivityLocked(mStartActivity, topRootTask, newTask, isTaskSwitch,
            mOptions, sourceRecord);
    
    // ⑮ 恢复焦点 Task 的顶部 Activity (关键!)
    if (mDoResume) {
        final ActivityRecord topTaskActivity = startedTask.topRunningActivityLocked();
        if (!mTargetRootTask.isTopActivityFocusable()
                || (topTaskActivity != null && topTaskActivity.isTaskOverlay()
                && mStartActivity != topTaskActivity)) {
            // 不可聚焦的 Activity(如 PIP)
            mTargetRootTask.ensureActivitiesVisible(null);
            mTargetRootTask.mDisplayContent.executeAppTransition();
        } else {
            // 恢复正常 Activity
            if (mTargetRootTask.isTopActivityFocusable()
                    && !mRootWindowContainer.isTopDisplayFocusedRootTask(mTargetRootTask)) {
                if (!avoidMoveToFront()) {
                    mTargetRootTask.moveToFront("startActivityInner");
                }
            }
            // ★★★ 这里会触发应用进程的 Activity 启动 ★★★
            mRootWindowContainer.resumeFocusedTasksTopActivities(
                    mTargetRootTask, mStartActivity, mOptions, mTransientLaunch);
        }
    }

    return START_SUCCESS;
}
```

<br/>

<br/>

<br/>

<br/>

## 2.4 第四阶段:ActivityThread 层 (应用进程)

### 2.4.1 ActivityThread.handleLaunchActivity() - 应用侧启动

#### 2.4.1.1 ClientTransactionHandler 事务调度

```java
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
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
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

mMainThread.sendActivityResult 将结果发送给应用主线程，应用侧开始启动

```java
    public void sendActivityResult(
            IBinder activityToken, String id, int requestCode,
            int resultCode, Intent data) {
        if (DEBUG_RESULTS) Slog.v(TAG, "sendActivityResult: id=" + id
                + " req=" + requestCode + " res=" + resultCode + " data=" + data);
        final ArrayList<ResultInfo> list = new ArrayList<>();
        list.add(new ResultInfo(id, requestCode, resultCode, data, activityToken));
        final ClientTransaction clientTransaction = ClientTransaction.obtain(mAppThread);
        final ActivityResultItem activityResultItem = ActivityResultItem.obtain(
                activityToken, list);
        clientTransaction.addTransactionItem(activityResultItem);
        try {
            mAppThread.scheduleTransaction(clientTransaction);
        } catch (RemoteException e) {
            // Local scheduling
        }
    }
```

mAppThread.scheduleTransaction 开始事物调度

```java
// ClientTransactionHandler.java (ActivityThread 的父类)
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);  // 预处理
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

<br/>

执行 executeTransactionItems

```java
    /**
     * Resolve transaction.
     * First all callbacks will be executed in the order they appear in the list. If a callback
     * requires a certain pre- or post-execution state, the client will be transitioned accordingly.
     * Then the client will cycle to the final lifecycle state if provided. Otherwise, it will
     * either remain in the initial state, or last state needed by a callback.
     */
    public void execute(@NonNull ClientTransaction transaction) {
        if (DEBUG_RESOLVER) {
            Slog.d(TAG, tId(transaction) + "Start resolving transaction");
            Slog.d(TAG, transactionToString(transaction, mTransactionHandler));
        }

        Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "clientTransactionExecuted");
        try {
            if (transaction.getTransactionItems() != null) {
                executeTransactionItems(transaction);
            } else {
                // TODO(b/260873529): cleanup after launch.
                executeCallbacks(transaction);
                executeLifecycleState(transaction);
            }
        } catch (Exception e) {
            Slog.e(TAG, "Failed to execute the transaction: "
                    + transactionToString(transaction, mTransactionHandler));
            throw e;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
        }

        mPendingActions.clear();
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
    }
```

item.execute(mTransactionHandler, mPendingActions);  item 为 LaunchActivityItem 类

<br/>

#### 2.4.1.2 LaunchActivityItem.execute 执行启动

```java
@Override
public void execute(@NonNull ClientTransactionHandler client,
        @NonNull PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    
    // 创建 ActivityClientRecord
    final ActivityClientRecord r = new ActivityClientRecord(mActivityToken, mIntent, mIdent,
            mInfo, mOverrideConfig, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mSceneTransitionInfo, mIsForward,
            mProfilerInfo, client, mAssistToken, mShareableActivityToken, mLaunchedFromBubble,
            mTaskFragmentToken, mInitialCallerInfoAccessToken, mActivityWindowInfo);
    
    // 调用 ActivityThread.handleLaunchActivity
    client.handleLaunchActivity(r, pendingActions, mDeviceId, null /* customIntent */);
    
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

<br/>

#### 2.4.1.3 ActivityThread.handleLaunchActivity - 应用侧核心

```java
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, int deviceId, Intent customIntent) {
    
    // ① 取消 GC 空闲处理器(防止启动过程被打断)
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    // ② 启动性能分析(如果需要)
    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
    }

    // ③ 确保配置和资源是最新的
    applyPendingApplicationInfoChanges(r.activityInfo.packageName);
    mConfigurationController.handleConfigurationChanged(null, null);
    updateDeviceIdForNonUIContexts(deviceId);

    // ④ 初始化硬件加速
    if (ThreadedRenderer.sRendererEnabled
            && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
        HardwareRenderer.preload();
    }
    WindowManagerGlobal.initialize();

    // ⑤ 通知 GraphicsEnvironment
    GraphicsEnvironment.hintActivityLaunch();

    // ⑥ 执行实际启动(核心方法)
    final Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfigurationController.getConfiguration());
        reportSizeConfigurations(r);
        
        // ⑦ 设置状态恢复标志
        if (!r.activity.mFinished && pendingActions != null) {
            pendingActions.setOldState(r.state);
            pendingActions.setRestoreInstanceState(true);
            pendingActions.setCallOnPostCreate(true);
        }

        // ⑧ 触发窗口信息回调
        handleActivityWindowInfoChanged(r);
    } else {
        // 启动失败,通知 ATMS 结束 Activity
        ActivityClient.getInstance().finishActivity(r.token, Activity.RESULT_CANCELED,
                null /* resultData */, Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
    }

    return a;
}
```

<br/>

#### 2.4.1.4 ActivityThread.performLaunchActivity - Activity 实例化

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ========== 第一步:获取 PackageInfo ==========
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, mCompatibilityInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }

    // ========== 第二步:解析 Component ==========
    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    // ========== 第三步:创建 ContextImpl ==========
    ContextImpl activityBaseContext = createBaseContextForActivity(r);

    // ========== 第四步:通过反射创建 Activity 实例 ==========
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = activityBaseContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        
        // 设置 ClassLoader
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess(...);
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component + ": " + e.toString(), e);
        }
    }

    // ========== 第五步:创建 Application ==========
    try {
        Application app = r.packageInfo.makeApplicationInner(false, mInstrumentation);

        // ========== 第六步:注册到 Activity 列表 ==========
        synchronized (mResourcesManager) {
            mActivities.put(r.token, r);  // 重要!后续查找用
        }

        if (activity != null) {
            // ========== 第七步:创建配置和窗口 ==========
            CharSequence title = r.activityInfo.loadLabel(
                    activityBaseContext.getPackageManager());
            Configuration config = new Configuration(
                    mConfigurationController.getCompatConfiguration());
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }

            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;  // 复用窗口
                r.mPendingRemoveWindow = null;
            }

            // 加载 Resources
            activityBaseContext.getResources().addLoaders(
                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

            // ========== 第八步:Activity.attach() ==========
            activityBaseContext.setOuterContext(activity);
            activity.attach(activityBaseContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.activityConfigCallback,
                    r.assistToken, r.shareableActivityToken, 
                    r.initialCallerInfoAccessToken);

            // 设置主题
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            // ========== 第九步:调用 onCreate() ==========
            r.activity = activity;  // 先赋值,便于回调中查找
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }

            // 验证 super.onCreate() 是否被调用
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
        }

        // 设置生命周期状态
        r.setState(ON_CREATE);

    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to start activity " + component + ": " + e.toString(), e);
        }
    }

    return activity;
}
```

<br/>

<br/>

当 ATMS 决定启动 Activity 后,会通过 Binder 回调到应用进程的 ApplicationThread:

```java
// ApplicationThread (ActivityThread 内部类)
@Override
public final void scheduleLaunchActivity(Intent intent, IBinder token, ...) {
    // 发送消息到主线程
    sendMessage(H.LAUNCH_ACTIVITY, r);
}

// ActivityThread.H (Handler)
case LAUNCH_ACTIVITY: {
    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
    handleLaunchActivity(r, ...);
}

private Activity handleLaunchActivity(ActivityClientRecord r, ...) {
    // 1. 创建 Activity 实例
    Activity a = performLaunchActivity(r, ...);
    
    if (a != null) {
        // 2. 调用 onCreate
        // 3. 调用 onStart
        // 4. 调用 onResume
        handleResumeActivity(...);
    }
    return a;
}
```

<br/>

## 完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│ 系统进程 (ATMS)                                                  │
├─────────────────────────────────────────────────────────────────┤
│ ATMS.startActivityAsUser()                                       │
│   ↓                                                              │
│ ActivityStartController.obtainStarter()                          │
│   ↓                                                              │
│ ActivityStarter.execute()                                        │
│   ↓                                                              │
│ ActivityStarter.executeRequest()                                 │
│   ├─ 权限检查                                                    │
│   ├─ 创建 ActivityRecord (系统侧记录)                             │
│   └─ startActivityUnchecked()                                    │
│       ↓                                                          │
│       startActivityInner()                                       │
│         ├─ 计算 Task                                             │
│         ├─ 创建/复用 Task                                        │
│         └─ resumeFocusedTasksTopActivities()                     │
└─────────────────────────────────────────────────────────────────┘
                          ↓ Binder IPC
┌─────────────────────────────────────────────────────────────────┐
│ 应用进程 (ActivityThread)                                         │
├─────────────────────────────────────────────────────────────────┤
│ ClientTransaction.scheduleTransaction()                          │
│   ↓                                                              │
│ ApplicationThread.scheduleTransaction()                          │
│   ↓                                                              │
│ H.EXECUTE_TRANSACTION                                            │
│   ↓                                                              │
│ TransactionExecutor.execute()                                    │
│   ↓                                                              │
│ LaunchActivityItem.execute()                                     │
│   ↓                                                              │
│ ActivityThread.handleLaunchActivity()                            │
│   ↓                                                              │
│ performLaunchActivity()                                          │
│   ├─ newActivity() 反射创建                                      │
│   ├─ createBaseContextForActivity() 创建 Context                  │
│   ├─ activity.attach() 绑定上下文                                 │
│   └─ callActivityOnCreate() 调用 onCreate                        │
└─────────────────────────────────────────────────────────────────┘
```

两层记录机制
系统侧: ActivityRecord - 管理窗口、Task、显示
应用侧: ActivityClientRecord - 管理生命周期、实例状态

<br/>

<br/>

```java
调用者              ContextImpl          Instrumentation              ATMS              ActivityThread
  |                     |                      |                        |                    |
  |--startActivity()--> |                      |                        |                    |
  |                     |--startActivity()-->  |                        |                    |
  |                     |                      |--execStartActivity()-->|                    |
  |                     |                      |                        |--startActivity()-> |
  |                     |                      |                        |                    |--scheduleLaunchActivity()
  |                     |                      |                        |                    |
  |                     |                      |                        |<--Binder Callback--|
  |                     |                      |                        |                    |
  |                     |                      |                        |               handleLaunchActivity()
  |                     |                      |                        |                    |
  |                     |                      |                        |              performLaunchActivity()
  |                     |                      |                        |                    |
  |                     |                      |                        |              Instrumentation.newActivity()
  |                     |                      |                        |                    |
  |                     |                      |                        |              activity.attach()
  |                     |                      |                        |                    |
  |                     |                      |                        |              callActivityOnCreate()
  |                     |                      |                        |                    |
  |                     |                      |                        |              activity.onStart()
  |                     |                      |                        |                    |
  |                     |                      |                        |              activity.onResume()
```

1. ContextImpl.startActivity 是入口,负责基础验证

2. Instrumentation 提供监控和测试能力

3. ATMS 是系统服务核心,负责任务栈管理和权限检查

4. ActivityThread 负责在应用进程中创建和初始化 Activity

5. 整个过程涉及多次跨进程通信(Binder)

6. 非 Activity Context 启动必须添加 FLAG_ACTIVITY_NEW_TASK

    这个流程展示了 Android 框架的精妙设计:通过分层架构、装饰器模式、Binder IPC 和生命周期管理,实现了安全、灵活、可测试的 Activity 启动机制。

<br/>

## 2.5 第五阶段:Activity 生命周期

### 第一阶段：从 onCreate 到 onResume 的流程

#### 2.5.1 performLaunchActivity 执行完毕，返回到 handleLaunchActivity

```java
LaunchActivityItem.execute()
  └─ handleLaunchActivity()
      └─ performLaunchActivity()
          └─ Activity.onCreate()  ← 执行到这里
  └─ execute 返回
  └─ 回到 TransactionExecutor.executeCallbacks()
  └─ cycleToPath(ON_CREATE → ON_RESUME)
  └─ performLifecycleSequence()
      ├─ ON_START: StartActivityItem.execute()
      │   └─ handleStartActivity()
      │       └─ Activity.onStart()  ← 触发 onStart
      │
      └─ ON_RESUME: ResumeActivityItem.execute()
          └─ handleResumeActivity()
              └─ performResumeActivity()
                  └─ Activity.onResume()  ← 触发 onResume
```
