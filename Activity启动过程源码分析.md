#Activity启动过程源码分析

* Android 6.0 & API Level  23
* Github: [Nvsleep](https://github.com/Nvsleep)
* 邮箱: lizhenqiao@126.com
* QQ: 522910000


##简述
主要分析从Launch启动器点击APP图标到APP主Activity启动显示的过程

##S1: Launcher.onClick()
Android桌面本身也是一个启动器程序,系统默认启动器程序的核心activity实现类是Launcher:


	/**
	 * Default launcher application.
	 */
	public class Launcher extends Activity implements View.OnClickListener....{

		...
		
		public void onClick(View v) {
			....
	        Object tag = v.getTag();
	        if (tag instanceof ShortcutInfo) {
	            onClickAppShortcut(v);
	        } else if (tag instanceof FolderInfo) {
	            if (v instanceof FolderIcon) {
	                onClickFolderIcon(v);
	            }
	        } else if (v == mAllAppsButton) {
	            onClickAllAppsButton(v);
	        } else if (tag instanceof AppInfo) {
				// 点击某个APP图标
	            startAppShortcutOrInfoActivity(v);
	        } else if (tag instanceof LauncherAppWidgetInfo) {
	            if (v instanceof PendingAppWidgetHostView) {
	                onClickPendingWidget((PendingAppWidgetHostView) v);
	            }
	        }
   		 }
		....

		@Thunk 
		void startAppShortcutOrInfoActivity(View v) {
	        Object tag = v.getTag();
	        final ShortcutInfo shortcut;
	        final Intent intent;
	        if (tag instanceof ShortcutInfo) {
	            shortcut = (ShortcutInfo) tag;
	            intent = shortcut.intent;
	            int[] pos = new int[2];
	            v.getLocationOnScreen(pos);
	            intent.setSourceBounds(new Rect(pos[0], pos[1],
	                    pos[0] + v.getWidth(), pos[1] + v.getHeight()));
	
	        } else if (tag instanceof AppInfo) {
	            shortcut = null;
	            intent = ((AppInfo) tag).intent;
	        } else {
	            throw new IllegalArgumentException("Input must be a Shortcut or AppInfo");
	        }
			// 准备startActivity
	        boolean success = startActivitySafely(v, intent, tag);
	        mStats.recordLaunch(v, intent, shortcut);
	
	        if (success && v instanceof BubbleTextView) {
	            mWaitingForResume = (BubbleTextView) v;
	            mWaitingForResume.setStayPressed(true);
	        }
   		 }

		public boolean startActivitySafely(View v, Intent intent, Object tag) {
	        boolean success = false;
	        if (mIsSafeModeEnabled && !Utilities.isSystemApp(this, intent)) {
	            Toast.makeText(this, R.string.safemode_shortcut_error, Toast.LENGTH_SHORT).show();
	            return false;
	        }
	        try {
				// 准备startActivity
	            success = startActivity(v, intent, tag);
	        } catch (ActivityNotFoundException e) {
	            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
	            Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
	        }
	        return success;
	    }
		
	}

Launcher的点击APP图标经过一些处理之后在startActivity(v, intent, tag)中调用了父类Activity的startActivity(Intent intent, @Nullable Bundle options)

##S2: Activity.startActivityForResult()

	@Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }


	public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
			// 让mInstrumentation这个负责APP与系统交互的对象去执行startActivity具体请求
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
           	....
    }
* 每个Activity在启动之后都会初始化一个Instrumentation对象,用于桥接处理与系统交互的一些操作,比如说要启动其他的activity.
* mMainThread.getApplicationThread()获取的是Launcher类的ActivityThread对象的内部类ApplicationThread成员变量;ApplicationThread主要负责APP进程与系统进程之间的Binder通信,通俗的说将APP进程的ApplicationThread对象通过Binder传递给系统进程用于系统进程调用APP进程的某些方法让APP进程做出一些操作,比如生成,启动,恢复该进程的acitivity,启动该进程的Service等,后面会用到.


##S3: Instrumentation.execStartActivity()

	public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
		// Launcher启动器程序进程的用于跨进程通信的IApplicationThread对象
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
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
            intent.prepareToLeaveProcess();
			// 通过Binder跨进程到系统进程让ActivityManagerService去执行启动activity操作
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
乐器对象Instrumentation直接把要启动activity的操作交给ActivityManagerNative.getDefault()得到的IActivityManager接口实现对象了,下面看看具体的调用

##S4: ActivityManagerNative.getDefault()		


	public abstract class ActivityManagerNative extends Binder implements IActivityManager{


		/**
	     * Retrieve the system's default/global activity manager.
	     */
	    static public IActivityManager getDefault() {
			// 通过单例封装类Singleton去得到一个需要的单例对象
	        return gDefault.get();
	    }


		private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
			// 具体产生被包装的单例对象的方法,如果当前单例封装类内还没有需要的单例对象就会调用其create()方法初始化一个
	        protected IActivityManager create() {
				// 得到系统进程启动时候生成的ActivityManagerService对象在本地进程的代理Binder对象
	            IBinder b = ServiceManager.getService("activity");
	            if (false) {
	                Log.v("ActivityManager", "default service binder = " + b);
	            }
				// 生成ActivityManagerProxy对象
	            IActivityManager am = asInterface(b);
	            if (false) {
	                Log.v("ActivityManager", "default service = " + am);
	            }
	            return am;
	        }
   		};
	
		/**
	     * Cast a Binder object into an activity manager interface, generating
	     * a proxy if needed.
	     */
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

	}

可以看出ActivityManagerNative.getDefault()实际上是得到了一个ActivityManagerNative的内部类ActivityManagerProxy代理对象,接下来通过Binder进程间通讯机制进入ActivityManagerService进程执行启动activity操作

##S5: ActivityManagerService.startActivity()


	@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }

	@Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }

使用ActivityManagerService的一个成员变量ActivityStackSupervisor对象mStackSupervisor去执行接下来操作,mStackSupervisor在AMS对象构造函数中生成.

##S6: ActivityStackSupervisor.startActivityMayWait()

	final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
		....

		// 解析需要启动的activity信息,核心调用了PackageManagerService.resolveIntent()方法
		ActivityInfo aInfo =
                resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);
		
		....		

		// 下一步操作
		int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);
	
		....

	}


	final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
		
		....
		
		// 封装即将要启动的activity相关信息
		// options参数封装了包含启动动画的信息
		// resultRecord==null,后续r.resultTo==null
		ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, this, container, options);
		.....
		// 下一步操作
		err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);
		....
	
	}


	final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {

		....
		ActivityStack targetStack;
		....

		boolean newTask = false;
		
		// Should this be considered a new task?
        if (r.resultTo == null && inTask == null && !addingToTask
                && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
			// 获取要启动目标activity的ActivityStack,几乎Android里面的app的activity都用同一个
			// ActivityStack来维护的
            targetStack = computeStackFocus(r, newTask);
            targetStack.moveToFront("startingNewTask");

            if (reuseTask == null) {
				// 生成一个新的TaskRecord对象,并赋值给要启动的activityRecord的task变量
				// launchTaskBehind==false,所以会同时将目标activity的TaskRecord放到targetStack的成
				// 员变量ArrayList<TaskRecord> mTaskHistory最top处
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
           	....
        } 
		....
		// 进入ActivityStack去启动目标activity
		targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
		....
	
	}

上面的代码逻辑好复杂,省略的一些代码主要用于判断当前栈里面是否就有将要启动的activity,某些情况下,是不需要重新启动这个activity的实例,和activity启动模式以及前期传入intent中参数有关系.执行到这里,是要开辟一个新的activity任务栈来启动这个activity了.


##S7: ActivityStack.startActivityLocked()


	final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
		....
		TaskRecord task = null;
		task = r.task;	
		// 每个TaskRecord中都有一个ArrayList<ActivityRecord> mActivities用于记录当前任务栈中的历史activity
		// 设置将要启动activity相关信息ActivityRecord对象推到所在TaskRecord成员变量final 
		// ArrayList<ActivityRecord> mActivities的栈顶
		task.addActivityToTop(r);
        task.setFrontOfTask();
		....
		// 传过来的doResume为true,返回交由ActivityStackSupervisor对象继续处理
		if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
		....

	}
省略的代码主要做了目标activity与WindowManagerService相关的一些交互操作

##S8: ActivityStackSupervisor.resumeTopActivitiesLocked()

	boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
		// 上一步S7已经设置,这里if里面的判断为true
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    // Already started above.
					// 上面if中判断后已经执行,故跳过
                    continue;
                }
                if (isFrontStack(stack)) {
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }

##S9: ActivityStack.resumeTopActivityLocked()

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

	private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {

		....
		// 找到ActivityStack的ArrayList<TaskRecord> mTaskHistory顶部的TaskRecord的顶部的
		// ActivityRecord 信息,因为前面已经设置将要启动的ActivityRecord信息处于顶部之顶部
		final ActivityRecord next = topRunningActivityLocked(null);
		....
		// If the top activity is the resumed one, nothing to do.
		// 如果将要启动的activity已经出于resumed状态,则什么都不用做了
        if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                    mStackSupervisor.allResumedActivitiesComplete()) {
           ....
        }
		....
		// If we are sleeping, and there is no resumed activity, and the top
        // activity is paused, well that is the state we want.
		// 如果系统休眠同时将要启动的activity处于栈顶,则什么都不做
        if (mService.isSleepingOrShuttingDown()
                && mLastPausedActivity == next
                && mStackSupervisor.allPausedActivitiesComplete()) {
           ....
        }
		....
		// If we are currently pausing an activity, then don't do anything
        // until that is done.
		// 如果当前正有acitivity正在进入paused状态,则也什么都不做
        if (!mStackSupervisor.allPausedActivitiesComplete()) {
            ....
        }
		....
		// 上述几个情况都不满足时候,先把当前出于resumed状态的activity先推到paused状态,才可以启动下一个目标activity
		// 由于是从Launcher这个activity点击启动app的,所以接下来要把Launcher推入paused状态
		 if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }
	
	}

接下来要去将Launcher这个启动器activity推到paused状态

##S10: ActivityStack.startPausingLocked()

	final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {

		....
		ActivityRecord prev = mResumedActivity;
		....
		mResumedActivity = null;
	    mPausingActivity = prev;
	    mLastPausedActivity = prev;
		....
		if (prev.app != null && prev.app.thread != null) {
	            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
	            try {
	                ....
					// 实际执行代码
	                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
	                        userLeaving, prev.configChangeFlags, dontWait);
	            } catch (Exception e) {
	                // Ignore exception, if process died other code will cleanup.
	                Slog.w(TAG, "Exception thrown during pause", e);
	                mPausingActivity = null;
	                mLastPausedActivity = null;
	                mLastNoHistoryActivity = null;
	            }
	        } else {
	            mPausingActivity = null;
	            mLastPausedActivity = null;
	            mLastNoHistoryActivity = null;
	        }
		}
		....
	}

这里的mResumedActivity就是当前显示的activity,这里当然是launcher这个activity,这里把launcher这个app的进程的ActivityThread内部类ApplicationThread的一个远程接口拿来,通过binder进程间通讯跳转到Launcher进程

##S11: ApplicationThread.schedulePauseActivity()

	public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges, boolean dontReport) {
            sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                    configChanges);
    }
	
	private void sendMessage(int what, Object obj, int arg1, int arg2) {
        sendMessage(what, obj, arg1, arg2, false);
    }
	
	// 通过外部类ActivityThread的一个成员Handler对象向Launcher进程的主线程发送一个Message消息
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


##S11: ActivityThread.handlePauseActivity()
	
	private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
        ActivityClientRecord r = mActivities.get(token);
        if (r != null) {
            //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
            if (userLeaving) {
                performUserLeavingActivity(r);
            }

            r.activity.mConfigChangeFlags |= configChanges;
			// 去执行acitivity的pause
            performPauseActivity(token, finished, r.isPreHoneycomb());

            // Make sure any pending writes are now committed.
            if (r.isPreHoneycomb()) {
                QueuedWork.waitToFinish();
            }

            // Tell the activity manager we have paused.
            if (!dontReport) {
                try {
					// 进入AMS系统进程去告知当前acitivity已经推到paused状态
                    ActivityManagerNative.getDefault().activityPaused(token);
                } catch (RemoteException ex) {
                }
            }
            mSomeActivitiesChanged = true;
        }
    }

	final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState) {
        ActivityClientRecord r = mActivities.get(token);
        return r != null ? performPauseActivity(r, finished, saveState) : null;
    }

	final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState) {
		....
		mInstrumentation.callActivityOnPause(r.activity);
		....
	}

	// Instrumentation类的方法
	 public void callActivityOnPause(Activity activity) {
        activity.performPause();
    }
	//Activity类
	final void performPause() {
        mDoReportFullyDrawn = false;
        mFragments.dispatchPause();
        mCalled = false;
		// acitivity生命周期的pause状态回调
        onPause();
        mResumed = false;
        if (!mCalled && getApplicationInfo().targetSdkVersion
                >= android.os.Build.VERSION_CODES.GINGERBREAD) {
            throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onPause()");
        }
        mResumed = false;
    }


##S12:ActivityManagerService.activityPaused()

	@Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }

##S13:ActivityStack.activityPausedLocked()

	final void activityPausedLocked(IBinder token, boolean timeout) {
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE,
            "Activity paused: token=" + token + ", timeout=" + timeout);

        final ActivityRecord r = isInStackLocked(token);
        if (r != null) {
            mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
            if (mPausingActivity == r) {
                ....
				// 继续,参数resumeNext==true
                completePauseLocked(true);
            } else {
                ....
            }
        }
    }

	private void completePauseLocked(boolean resumeNext) {
		
		....
		if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!mService.isSleepingOrShuttingDown()) {
				// 当前系统不是休眠状态,继续恢复启动之前设置的栈顶要启动的activity
                mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
            } else {
                mStackSupervisor.checkReadyForSleepLocked();
                ActivityRecord top = topStack.topRunningActivityLocked(null);
                if (top == null || (prev != null && top != prev)) {
                    // If there are no more activities available to run,
                    // do resume anyway to start something.  Also if the top
                    // activity on the stack is not the just paused activity,
                    // we need to go ahead and resume it to ensure we complete
                    // an in-flight app switch.
                    mStackSupervisor.resumeTopActivitiesLocked(topStack, null, null);
                }
            }
        }
		....
	
	}


##S14: ActivityStackSupervisor.resumeTopActivitiesLocked()

	// 前面走过一次,继续调用当前ActivityStack去回复启动顶部将要启动的activity
    boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }
		....


##S15: ActivityStack.resumeTopActivityInnerLocked()
	
	// 前面走过一次
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
			// 依然走这里
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }

	// 再次走resumeTopActivityInnerLocked()方法,看这次跟上次有何不同
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
		
		.....
		
		// 跟前面一样获取之前设置的将要启动的activity的信息封装对象
		final ActivityRecord next = topRunningActivityLocked(null);
	
		// 前面一些判断和上一次一样
		......
		
		// 这是上次判断要去将Launcher推入paused状态的地方,注意在startPausingLocked()方法中已经把
			mResumedActivity=null,而且Launcher这个activity也已经推入到paused状态
		if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }
		....
		// 因为前面在生成目标activity的AcitivityRecord对象时候.app成员变量均没有赋值,故为false
		// next.app表示目标activity的ProcessRecord进程信息
		// next.app.thread表示目标activity进程的AcitivityThread对象的ApplicationThread成员变量信息
		// Android 每个应用程序进程都有一个ActivityThread对象对应,因为程序进程的入口函数是ActivityThread.main()方法
		if (next.app != null && next.app.thread != null) {
			.....
		} else {
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(
                                    next.info.applicationInfo),
                            next.nonLocalizedLabel,
                            next.labelRes, next.icon, next.logo, next.windowFlags,
                            null, true);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
            }
            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
			// 去启动目标acitivity
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
		....

	}


##S16: ActivityStackSupervisor.startSpecificActivityLocked()

	// 这里去判断是否需要开启一个进程去启动当前activity
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
		// 这里从ActivityManagerService根据进程名和uid去查询对应ProcessRecord对象
		// 如果是第一次启动目前程序,则app==null
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);
		// 这里判断为true表明,当前要启动的程序进程和进程里面的入口ActivityThread.main()已经调用,ActivityThread对象已经生成
		// 第一次从Launcher点击app图标启动的话,这里的app==null
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
		// 第一次从Launcher点击app图标,故要先为了该应用程序去启动一个进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }

##S17: ActivityManagerService.startProcessLocked()

    final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            String hostingType, ComponentName hostingName, boolean allowWhileBooting,
            boolean isolated, boolean keepIfLarge) {
        return startProcessLocked(processName, info, knownToBeDead, intentFlags, hostingType,
                hostingName, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
                null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
                null /* crashHandler */);
    }

	// 上面传过来的entryPoint==null
    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {

		....
		startProcessLocked(
                app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
		....
	}

	// 上面传过来的entryPoint==null
    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
		
		....
		
		// Start the process.  It will either succeed and return a result containing
        // the PID of the new process, or else throw a RuntimeException.
		// 传过来的entryPoint==null
        boolean isActivityProcess = (entryPoint == null);
		// 指定进程入口函数
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
        checkTime(startTime, "startProcess: asking zygote to start proc");
		// 去启动该app进程
        Process.ProcessStartResult startResult = Process.start(entryPoint,
                app.processName, uid, uid, gids, debugFlags, mountExternal,
                app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                app.info.dataDir, entryPointArgs);
		....
		// 设置进程的pid到当前ProcessRecord对象中
		app.setPid(startResult.pid);
		synchronized (mPidsSelfLocked) {
				// 这里存放到SparseArray<ProcessRecord> mPidsSelfLocked,后面会用到
                this.mPidsSelfLocked.put(startResult.pid, app);
                if (isActivityProcess) {
                    Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                    msg.obj = app;
                    mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                            ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
                }
            }
		....

	}

##S18: Process.start()

    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
			// 通过Zygote系统机制去启动一个APP进程
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }

    private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
            ArrayList<String> argsForZygote = new ArrayList<String>();

            // --runtime-args, --setuid=, --setgid=,
            // and --setgroups= must go first
            argsForZygote.add("--runtime-args");
            argsForZygote.add("--setuid=" + uid);
            argsForZygote.add("--setgid=" + gid);
            if ((debugFlags & Zygote.DEBUG_ENABLE_JNI_LOGGING) != 0) {
                argsForZygote.add("--enable-jni-logging");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_SAFEMODE) != 0) {
                argsForZygote.add("--enable-safemode");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {
                argsForZygote.add("--enable-debugger");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_CHECKJNI) != 0) {
                argsForZygote.add("--enable-checkjni");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_JIT) != 0) {
                argsForZygote.add("--enable-jit");
            }
            if ((debugFlags & Zygote.DEBUG_GENERATE_DEBUG_INFO) != 0) {
                argsForZygote.add("--generate-debug-info");
            }
            if ((debugFlags & Zygote.DEBUG_ENABLE_ASSERT) != 0) {
                argsForZygote.add("--enable-assert");
            }
            if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
                argsForZygote.add("--mount-external-default");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
                argsForZygote.add("--mount-external-read");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
                argsForZygote.add("--mount-external-write");
            }
            argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

            //TODO optionally enable debuger
            //argsForZygote.add("--enable-debugger");

            // --setgroups is a comma-separated list
            if (gids != null && gids.length > 0) {
                StringBuilder sb = new StringBuilder();
                sb.append("--setgroups=");

                int sz = gids.length;
                for (int i = 0; i < sz; i++) {
                    if (i != 0) {
                        sb.append(',');
                    }
                    sb.append(gids[i]);
                }

                argsForZygote.add(sb.toString());
            }

            if (niceName != null) {
                argsForZygote.add("--nice-name=" + niceName);
            }

            if (seInfo != null) {
                argsForZygote.add("--seinfo=" + seInfo);
            }

            if (instructionSet != null) {
                argsForZygote.add("--instruction-set=" + instructionSet);
            }

            if (appDataDir != null) {
                argsForZygote.add("--app-data-dir=" + appDataDir);
            }

            argsForZygote.add(processClass);

            if (extraArgs != null) {
                for (String arg : extraArgs) {
                    argsForZygote.add(arg);
                }
            }

            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
	
* 首先组装将要启动的进程参数到ArrayList<String> argsForZygote对象中
* 然后调用openZygoteSocketIfNeeded()判断是否需要启动当前进程ActivityManagerService所在的	SystemServer进程是否与Zygote通过Socket通讯连接,如果没有,则通过socket方式连接用于后续进程间	通讯.

    	private static ZygoteState openZygoteSocketIfNeeded(String abi) throws 	ZygoteStartFailedEx {
	        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
	            try {
					// 与Zygote进程建立socket连接
	                primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
	            } catch (IOException ioe) {
	                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
	            }
	        }
	
	        if (primaryZygoteState.matches(abi)) {
	            return primaryZygoteState;
	        }
	
	        // The primary zygote didn't match. Try the secondary.
	        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
	            try {
				// 与Zygote进程建立socket连接
	            secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
	            } catch (IOException ioe) {
	                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
	            }
	        }
	
	        if (secondaryZygoteState.matches(abi)) {
	            return secondaryZygoteState;
	        }
	
	        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
	    }
		
		// 通过socket流对象向通道中写入数据
		private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
	        try {
	            final BufferedWriter writer = zygoteState.writer;
	            final DataInputStream inputStream = zygoteState.inputStream;
	
	            writer.write(Integer.toString(args.size()));
	            writer.newLine();
	
	            int sz = args.size();
	            for (int i = 0; i < sz; i++) {
	                String arg = args.get(i);
	                if (arg.indexOf('\n') >= 0) {
	                    throw new ZygoteStartFailedEx(
	                            "embedded newlines not allowed");
	                }
	                writer.write(arg);
	                writer.newLine();
	            }
	
	            writer.flush();
	
	            // Should there be a timeout on this?
	            ProcessStartResult result = new ProcessStartResult();
	            result.pid = inputStream.readInt();
	            if (result.pid < 0) {
	                throw new ZygoteStartFailedEx("fork() failed");
	            }
	            result.usingWrapper = inputStream.readBoolean();
	            return result;
	        } catch (IOException ex) {
	            zygoteState.close();
	            throw new ZygoteStartFailedEx(ex);
	        }
	    }

接下来,Android系统启动时候的Zygote进程监听到ActivityManagerService所在进程通过socket发送过来的信息,就会调用每个进程的入口函数:ActivityThread.main().

##S19: ActivityThread.main()

    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        AndroidKeyStoreProvider.install();

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");
		// 首先准备每个APP进程(主线程)的Looper消息循环对象和MessageQueue消息队列
        Looper.prepareMainLooper();
		// 为每个APP进程创建一个ActivityThread对象,同时会初始化一个Binder内部类成员变量用于
		// Binder通讯 final ApplicationThread mAppThread = new ApplicationThread();
        ActivityThread thread = new ActivityThread();
		// 关键方法,调用attach()函数
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
		// 将当前进程(主线程)进入消息循环
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
}


##S20: ActivityThread.attach()

    private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
		// 当前是APP进程,非系统进程,所以上面传入的参数boolean system == false
        if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
			// 通过Binder进程间通讯进入ActivityManagerService所在进程
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
            ....
        } else {
          ....
        }
		....
    }

##S21: ActivityManagerService.attachApplication()

    @Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
			// 在S17中调用Process.start()之后已经将pid赋值到ProcessRecord中
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }

    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

		ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
				// S17中已经存放,所以这里可以取出来
                app = mPidsSelfLocked.get(pid);
            }
        } else {
            app = null;
        }
		....
		// 将thread赋值给ProcessRecord app的thread成员变量
		app.makeActive(thread, mProcessStats);
		// 为true
		boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
		....
		  try {
            ....
			// 夸进程通讯到刚才APP启动的进程去做一些操作
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            updateLruProcessLocked(app, false, null);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            app.resetPackageList(mProcessStats);
            app.unlinkDeathRecipient();
            startProcessLocked(app, "bind fail", processName);
            return false;
        }

		....
		boolean badApp = false;
		// See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
				// 去启动目标activity	
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }

	}
	
##S22: ActivityStackSupervisor.attachApplicationLocked()

    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
				// 前面已经设置,当前的ActivityStack已经置顶,如果不是,则跳过,前面S6已经设置
                if (!isFrontStack(stack)) {
                    continue;
                }
				// 从当前的ActivityStack栈中找到处于最顶的将要启动的目标acitivity,前面S6已经设置
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
							// Whoops! 真正去启动acitivity了!
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            Slog.w(TAG, "Exception in new application when starting activity "
                                  + hr.intent.getComponent().flattenToShortString(), e);
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

    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {
		....
		r.app = app;
		....
		try {
			app.forceProcessStateUpTo(mService.mTopProcessState);
			// 通过Binder夸进程到刚才APP启动的进程去执行启动目标acitivity的操作
			// 利用的是前面夸进程传过来的APP进程的ActivityThread内的ApplicationThread这个Binder对象
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
		
		} catch (RemoteException e) {
            ....
            app.activities.remove(r);
            throw e;
        }

	}

##S23: ApplicationThread.scheduleLaunchActivity()

        // we use token to identify this activity without having to send the
        // activity itself back to the activity manager. (matters more with ipc)
        @Override
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
			// 发个消息到app进程的主线程,前面在该进程启动入口函数的ActivityThread.main()调用的时候,已经准备好Looper消息循环
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }

	    private void sendMessage(int what, Object obj) {
	        sendMessage(what, obj, 0, 0, false);
	    }

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
			// 在生成该进程ActivityThread对象的时候已经初始化了一个Handler对象 mH
	        mH.sendMessage(msg);
   		 }
通过Handler发送消息对象切换到当前进程主线程的ActivityThread对象去执行具体启动acitivity操作

##S24: ActivityThread.handleLaunchActivity()

    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;
		
		....

        // Initialize before creating the activity
        WindowManagerGlobal.initialize();
		// 去启动acitivity,主要生成acitivity对象,调用其生命周期onCreate(),onStart()方法
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
			// 调用acitivity生命周期的onResume(),并将acitivity界面绘制,显示出来
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);	
			
			....
                
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
    }

##S25: ActivityThread.performLaunchActivity()

	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

		// 先是获取当前要启动的acitivity的一些信息,主要是packageInfo和component
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


        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
			// 通过反射生成acitivity对象先
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
			// 生成当前APP的Application对象,并调用其onCreate()
           	Application app = r.packageInfo.makeApplication(false, mInstrumentation);
			if (activity != null) {
				// 为了当前acitivity生成一个Context对象
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
				 // 为当前acitivity初始化设置一些必要的参数
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
				// acitivity生命周期的onCreate()在此调用
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
				// 如果没有调用父类的onCreate()方法则,activity.mCalled == false,抛出异常!
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
					// 看情况是否需要调用onRestoreInstanceState()方法做一些恢复数据操作
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
			// 还没有显示出来,所以这里先设置为true
            r.paused = true;
			// 保存当前acitivity信息到该进程的ActivityThread对象中,下面需要取出来用到
			mActivities.put(r.token, r);

		} catch ....

	}

##S25: Activity.attach()

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
		// 赋值上一步为当前acitivity生成的Context对象,实际是ContextIml对象
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
		// 为当前acitivity设置Window对象
        mWindow = new PhoneWindow(this);
		// 把当前acitivity对象设置到Window作为通知回调
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
		// 这个在XML中直接写Framgent时候返回调用acitivity中的生成view的方法
        mWindow.getLayoutInflater().setPrivateFactory(this);
		// 设置输入法模式
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
		// 这个当然是主线程啦
        mUiThread = Thread.currentThread();
		// 保存该应用程序进程的AcitivityThread进程入口对象信息
        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }
		// 为该acitivity的窗口Window设置WindowManager对象,实际是WindowManagerImpl对象
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
主要为acitivity,赋值Context,设置window等


##S26: ActivityThread.handleResumeActivity()

    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        // TODO Push resumeArgs into the activity for consideration
		// 主要调用了生命周期的onResume()方法
        ActivityClientRecord r = performResumeActivity(token, clearHide);

        if (r != null) {
            final Activity a = r.activity;

            if (localLOGV) Slog.v(
                TAG, "Resume " + r + " started activity: " +
                a.mStartedActivity + ", hideForNow: " + r.hideForNow
                + ", finished: " + a.mFinished);

            final int forwardBit = isForward ?
                    WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

            // If the window hasn't yet been added to the window manager,
            // and this guy didn't finish itself or start another activity,
            // then go ahead and add the window.
			// 注释很清楚了...这里为true
            boolean willBeVisible = !a.mStartedActivity;
            if (!willBeVisible) {
                try {
                    willBeVisible = ActivityManagerNative.getDefault().willActivityBeVisible(
                            a.getActivityToken());
                } catch (RemoteException e) {
                }
            }
            if (r.window == null && !a.mFinished && willBeVisible) {
				// 在上一布已经为activity设置了Widnow等
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
					// 核心方法,这里走下去就是acitivity视图顶层管理对象ViewRootImpl测量,绘制,显示了
                    wm.addView(decor, l);
                }
			}
			....

    }

至此,从Launcher点击APP图标然后在新进程中启动主acitivity的过程的源码已经分析完毕.其实还有acitivity中的视图View绘制和显示机制,以及事件屏幕事件分发,以后再分析.

* Android 6.0 & API Level  23
* Github: [Nvsleep](https://github.com/Nvsleep)
* 邮箱: lizhenqiao@126.com
* QQ: 522910000