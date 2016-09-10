##Activity窗口视图创建显示过程源码分析

* Android 6.0 & API Level 23
* Github: [Nvsleep](https://github.com/Nvsleep)
* 邮箱: lizhenqiao@126.com


##简述
* 一般情况, 我们在Activity继承类重写onCrete()方法中会调用Activity.setContentView()方法传入需要显示的视图资源id或是View.
* 本文主要分析了从调用setContentView()起到Activity的视图窗口测量,布局,绘制的中间过程.

##S1: ActivityThread.handleLaunchActivity()
* 由上文[Activity启动过程源码分析]可知,acitivity的实际生成和启动的入口是APP当前进程的ActivityThread的内部类ApplicationThread.scheduleLaunchActivity()方法;
* 该方法通过Handler发送消息到当前进程主线程并调用ActivityThread.handleLaunchActivity()方法;




		private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
			....
			Activity a = performLaunchActivity(r, customIntent);
			if (a != null) {
				r.createdConfig = new Configuration(mConfiguration);
				Bundle oldState = r.state;
           		handleResumeActivity(r.token, false, r.isForward,!r.activity.mFinished && !r.startsNotResumed);
				....
			}
			....
			
		}



##S2: ActivityThread.performLaunchActivity()

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
				// 为了当前acitivity生成一个ContextImpl对象
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

在通过反射生成acitivity对象后,即调用了Activity.attach()方法初始化设置,这个是其生命周期方法onCreate()之前调用的;

##S3: Activity.attach()

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
		// 赋值上一步为当前acitivity生成的Context对象,实际是ContextImpl对象
        attachBaseContext(context);
		// acitivity的FragmentController mFragments成员变量初始化设置
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
		// 描述当前acitivity的一个特征对象,实际最初是ActivityStackSupervisor.startActivityLocked()
		// 中生成ActivityRecord对象时候生成的
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
* 可以看到,在这里 mWindow = new PhoneWindow(this) 为当前acitivity生成了一个Window窗口;
* mWindow.setCallback(this);为Window对象设置一个Callback回调对象,这里是acitivity对象本身,因为其实现Window.Callback接口;
* 最后为mWindow对象设置了一个WindowManager成员对象,同时引用赋值给当前acitivity对象;

##S4: PhoneWindow的生成相关

PhoneWindow extends Window 
	
	public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }	

	public Window(Context context) {
        mContext = context;
        mFeatures = mLocalFeatures = getDefaultFeatures(context);
    }
* PhoneWindow构造函数,主要是得到当前Context的引用,并通过LayoutInflater.from(context)得到一个PhoneLayoutInflater对象,用于后续通过资源id生成view对象.
* 这里看到LayoutInflater.from(context)和 mWindow.setWindowManager(...)中都调用了context.getSystemService(),实际在ActivityThread.performLaunchActivity()中生成acitivity对象后创建了一个ContextImpl对象并且在Activity.attach()中赋值给Activity的Context mBase成员;

##S5: ContextImpl.getSystemService()

	@Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }

	// 看下具体去拿服务的类
	final class SystemServiceRegistry {
	
		/**
	     * Gets a system service from a given context.
	     */
	    public static Object getSystemService(ContextImpl ctx, String name) {
	        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
	        return fetcher != null ? fetcher.getService(ctx) : null;
	    }

			/**
		     * Statically registers a system service with the context.
		     * This method must be called during static initialization only.
		     */
		    private static <T> void registerService(String serviceName, Class<T> serviceClass,
		            ServiceFetcher<T> serviceFetcher) {
		        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
		        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
		    }

		
		private static final HashMap<Class<?>, String> SYSTEM_SERVICE_NAMES =
		            new HashMap<Class<?>, String>();
		    private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
		            new HashMap<String, ServiceFetcher<?>>();
		    private static int sServiceCacheSize;
		
		    // Not instantiable.
		    private SystemServiceRegistry() { }


			static {
			
				....

				registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
               		new CachedServiceFetcher<LayoutInflater>() {
			            @Override
			            public LayoutInflater createService(ContextImpl ctx) {
							// 具体生成的对象是个PhoneLayoutInflater
			                return new PhoneLayoutInflater(ctx.getOuterContext());
			            }});

				registerService(Context.WINDOW_SERVICE, WindowManager.class,
	                new CachedServiceFetcher<WindowManager>() {
			            @Override
			            public WindowManager createService(ContextImpl ctx) {
							// 具体生成的对象是个WindowManagerImpl
			                return new WindowManagerImpl(ctx.getDisplay());
			            }});
				....

			}

	}

	    static abstract interface ServiceFetcher<T> {
        T getService(ContextImpl ctx);
    }

    /**
     * Override this class when the system service constructor needs a
     * ContextImpl and should be cached and retained by that context.
     */
    static abstract class CachedServiceFetcher<T> implements ServiceFetcher<T> {
        private final int mCacheIndex;

        public CachedServiceFetcher() {
            mCacheIndex = sServiceCacheSize++;
        }

        @Override
        @SuppressWarnings("unchecked")
        public final T getService(ContextImpl ctx) {
            final Object[] cache = ctx.mServiceCache;
            synchronized (cache) {
                // Fetch or create the service.
                Object service = cache[mCacheIndex];
                if (service == null) {
                    service = createService(ctx);
                    cache[mCacheIndex] = service;
                }
                return (T)service;
            }
        }

        public abstract T createService(ContextImpl ctx);
    }

很明显,通过LayoutInflater.from(context)得到的是一个PhoneLayoutInflater对象,通过context.getSystemService(Context.WINDOW_SERVICE)是一个WindowManagerImpl对象


##S6: Window.setWindowManager()

    /**
     * Set the window manager for use by this Window to, for example,
     * display panels.  This is <em>not</em> used for displaying the
     * Window itself -- that must be done by the client.
     *
     * @param wm The window manager for adding new windows.
     */
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName) {
        setWindowManager(wm, appToken, appName, false);
    }

    /**
     * Set the window manager for use by this Window to, for example,
     * display panels.  This is <em>not</em> used for displaying the
     * Window itself -- that must be done by the client.
     *
     * @param wm The window manager for adding new windows.
     */
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
很明显,参数wm是刚才获取的WindowManagerImpl对象,appToken是当前acitivity的一个标记特征对象,实际最初是ActivityStackSupervisor.startActivityLocked()中生成ActivityRecord对象时候生成的一个IApplicationToken.Stub对象,看下WindowManagerImpl.createLocalWindowManager()

	public final class WindowManagerImpl implements WindowManager {

		....

		private final Display mDisplay;
		private final Window mParentWindow;

		public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        	return new WindowManagerImpl(mDisplay, parentWindow);
    	}

	}
生成一个新的当前WindowManagerImpl对象,成员变量Display mDisplay指向当前的WindowManagerImpl对象,成员变量Window mParentWindow指向了当前window;

####至此,已经为当前acitivity生成了一个window,同时为这个window设置了当前acitivity作为CallBack回调,同时为window设置了一个WindowManagerImpl管理对象,且acitivity也拿到了这个管理对象;接下来在activity.attach()之后就会调用生命周期的onCreate()方法,一般情况在其中调用setContentView()


##S7: Activity.setContentView()

	public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
直接进入activity关联的PhoneWindow中为之设置view视图

##S8: PhoneWindow.setContentView()

    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
		// 第一次进来mContentParent默认为null,先去构造顶层的容器View
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
			// 在顶层容器View中通过inflate()为之生成视图树体系
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }

	// 初试化window的直接View容器DecorView mDecor,实际是一个FrameLayout,之后获取到
	// id==Window.D_ANDROID_CONTENT的视图内容容器ViewGroup mContentParent
	private void installDecor() {		
		if (mDecor == null) {
			// 生成一个DecorView对象
            mDecor = generateDecor();
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        }
		if (mContentParent == null) {
			mContentParent = generateLayout(mDecor);
		}
		....
	
	}

	// DecorView extends FrameLayout
    protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }

	protected ViewGroup generateLayout(DecorView decor) {
	
		....
		int layoutResource;

		int features = getLocalFeatures();
        // System.out.println("Features: 0x" + Integer.toHexString(features));
		// 根据window特性获取将要生成的view的系统资源id
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_title_icons;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println("Title Icons!");
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            // Special case for a window with only a progress bar (and title).
            // XXX Need to have a no-title version of embedded windows.
            layoutResource = R.layout.screen_progress;
            // System.out.println("Progress!");
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_custom_title;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                layoutResource = a.getResourceId(
                        R.styleable.Window_windowActionBarFullscreenDecorLayout,
                        R.layout.screen_action_bar);
            } else {
                layoutResource = R.layout.screen_title;
            }
            // System.out.println("Title!");
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            // Embedded, so no decoration is needed.
            layoutResource = R.layout.screen_simple;
            // System.out.println("Simple!");
        }
		// 将上面得到的系统View资源id生成view,把addView()添加到顶层容器DecorView中
		View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;
		// 在前面生成的顶层容器DecorView mDecor中遍历找到ID==com.android.internal.R.id.content的View作为视图内容容器并返回
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
		....
		return contentParent;

	}

* 至此,已经在activity的onCreate()中为window设置了视图,并将setContentView()中的view通过ViewGroup.addView()关联到window的顶层视图容器DecorView中.
* 接下来回看ActivityThread.handleLaunchActivity()中,会调用handleResumeActivity()方法做进一步操作

##S9: ActivityThread.handleResumeActivity()

    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {

		....
		// 主要为当前acitivity关联的ActivityClientRecord对象做一些变量赋值操作,同时也
		// 调用了Activity生命周期的onResume()方法
		// 当前acitivity关联的ActivityClientRecord对象是在ActivityThread.performLaunchActivity()中
		// 保存在ActivityThread的final ArrayMap<IBinder, ActivityClientRecord> mActivities变量中.
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
			// 这里主要判断当前acitivity是否去启动别的acitivity了,没有的话表明此acitivity即将
			// 要显示出来,willBeVisible == true
            boolean willBeVisible = !a.mStartedActivity;
            if (!willBeVisible) {
                try {
					// 通过Binder机制跳到ActivityManagerService中看当前acitivity所在的
					// ActivityStack中是否有位于当前acitivity之上的activity是否全屏的,是的话返回false
                    willBeVisible = ActivityManagerNative.getDefault().willActivityBeVisible(
                            a.getActivityToken());
                } catch (RemoteException e) {
                }
            }
			// 这里r.window没有赋值当然为null,activity.mFinished==fale,并且要显示的话进入if
            if (r.window == null && !a.mFinished && willBeVisible) {
				// 已经在acitivity.attach()中生成管理window,且在setContentView()中为window设置好了视图
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
				// 由S6中可知道,acitivity和window的WindowManager对象实际是WindowManagerImpl
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
					// 关联视图View和WindowManager
                    wm.addView(decor, l);
                }
           ....
		
	}

	// 上面r.window.getAttributes()获取的是window默认的mWindowAttributes = new WindowManager.LayoutParams(),
	public LayoutParams() {
			// 默认无参构造函数LayoutParams.width和LayoutParams.height均为LayoutParams.MATCH_PARENT
            super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
            type = TYPE_APPLICATION;
            format = PixelFormat.OPAQUE;
    }

至此,window的视图容器和WindowManager还未关联起来,下面看如何调用WindowManagerImpl.addView()关联的.

##S10: WindowManagerImpl.addView()

	public final class WindowManagerImpl implements WindowManager {

		private final Display mDisplay;
    	private final Window mParentWindow;
		private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

		....

		 @Override
   		 public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
	        applyDefaultToken(params);
	        mGlobal.addView(view, params, mDisplay, mParentWindow);
   		 }
		....
	}
由S6可知,这里的mParentWindow即是当前acitivity关联的window对象,由S5可知,mDisplay是与acitivity关联的ContextImpl对象获取的Display对象.

##S11: WindowManagerGlobal.addView()

    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent, then hardware acceleration for this view is
            // set from the application's hardware acceleration setting.
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }
			
			// 判断当前的activity关联的window中的顶层视图DecorView是否已经设置并关联过一个ViewRootImpl了
            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }
			// 为当前acitivity的关联window的顶层视图DecorView生成一个最顶层视图控制器ViewParent
			// 在Activity.attach()中生成window对象的时候传入的Context是acitivity本身,之后
			// window中生成顶层视图DecorView的时候用的Context也是这个acitivity本身
            root = new ViewRootImpl(view.getContext(), display);
			
            view.setLayoutParams(wparams);
			// 分别保存到WindowManagerGlobal的3个ArrayList中
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {
			// 进入最顶层ViewParent视图控制器去做最后的关联设置
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }

看下这里的view.setLayoutParams(wparams);即给当前acitivity的关联window的顶层视图DecorView设置LayoutParams参数


    public void setLayoutParams(ViewGroup.LayoutParams params) {
        if (params == null) {
            throw new NullPointerException("Layout parameters cannot be null");
        }
        mLayoutParams = params;
        resolveLayoutParams();
        if (mParent instanceof ViewGroup) {
            ((ViewGroup) mParent).onSetLayoutParams(this, params);
        }
        requestLayout();
    }

    @CallSuper
    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();
		// 这个时候该View还没有与ViewRootImpl最顶层控制器关联,故mAttachInfo==null
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
            ViewRootImpl viewRoot = getViewRootImpl();
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }
		// 设置标记要强制走layout和draw
        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;
		// 同mAttachInfo,这个时候mParent==null
        if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
    }

##S12: ViewRootImpl.setView()

	// 先看下ViewRootImpl的构造函数初始化
    public ViewRootImpl(Context context, Display display) {
		// 由S11可知,这个context就是当前的acitivity对象本身
        mContext = context;
		// 与WindowManagerService关联的一个重要对象,主要用于关联当前视图和系统WindowManagerService的通讯
        mWindowSession = WindowManagerGlobal.getWindowSession();
        mDisplay = display;
        mBasePackageName = context.getBasePackageName();

        mDisplayAdjustments = display.getDisplayAdjustments();

        mThread = Thread.currentThread();
        mLocation = new WindowLeaked(null);
        mLocation.fillInStackTrace();
        mWidth = -1;
        mHeight = -1;
        mDirty = new Rect();
        mTempRect = new Rect();
        mVisRect = new Rect();
        mWinFrame = new Rect();
		// 当前视图的一个IWindow.Stub对象,等下面与系统WindowManagerService关联时候用到,表征了当前的视图窗体
        mWindow = new W(this);
        mTargetSdkVersion = context.getApplicationInfo().targetSdkVersion;
        mViewVisibility = View.GONE;
        mTransparentRegion = new Region();
        mPreviousTransparentRegion = new Region();
        mFirst = true; // true for the first time the view is added
        mAdded = false;
		// 这里生成View视图树的AttachInfo对象
        mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this);
        mAccessibilityManager = AccessibilityManager.getInstance(context);
        mAccessibilityInteractionConnectionManager =
            new AccessibilityInteractionConnectionManager();
        mAccessibilityManager.addAccessibilityStateChangeListener(
                mAccessibilityInteractionConnectionManager);
        mHighContrastTextManager = new HighContrastTextManager();
        mAccessibilityManager.addHighTextContrastStateChangeListener(
                mHighContrastTextManager);
        mViewConfiguration = ViewConfiguration.get(context);
        mDensity = context.getResources().getDisplayMetrics().densityDpi;
        mNoncompatDensity = context.getResources().getDisplayMetrics().noncompatDensityDpi;
        mFallbackEventHandler = new PhoneFallbackEventHandler(context);
		// 重要的显示刷新帧处理对象,在调用requestLayout()刷新视图的时候用到了
        mChoreographer = Choreographer.getInstance();
        mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
        loadSystemProperties();
    }


    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                mAttachInfo.mDisplayState = mDisplay.getState();
                mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

                mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
                mFallbackEventHandler.setView(view);
                mWindowAttributes.copyFrom(attrs);
                if (mWindowAttributes.packageName == null) {
                    mWindowAttributes.packageName = mBasePackageName;
                }
                attrs = mWindowAttributes;
                // Keep track of the actual window flags supplied by the client.
                mClientWindowLayoutFlags = attrs.flags;

                setAccessibilityFocus(null, null);

                if (view instanceof RootViewSurfaceTaker) {
                    mSurfaceHolderCallback =
                            ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                    if (mSurfaceHolderCallback != null) {
                        mSurfaceHolder = new TakenSurfaceHolder();
                        mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                    }
                }

                // Compute surface insets required to draw at specified Z value.
                // TODO: Use real shadow insets for a constant max Z.
                if (!attrs.hasManualSurfaceInsets) {
                    final int surfaceInset = (int) Math.ceil(view.getZ() * 2);
                    attrs.surfaceInsets.set(surfaceInset, surfaceInset, surfaceInset, surfaceInset);
                }

                CompatibilityInfo compatibilityInfo = mDisplayAdjustments.getCompatibilityInfo();
                mTranslator = compatibilityInfo.getTranslator();

                // If the application owns the surface, don't enable hardware acceleration
                if (mSurfaceHolder == null) {
                    enableHardwareAcceleration(attrs);
                }

                boolean restore = false;
                if (mTranslator != null) {
                    mSurface.setCompatibilityTranslator(mTranslator);
                    restore = true;
                    attrs.backup();
                    mTranslator.translateWindowLayout(attrs);
                }
                if (DEBUG_LAYOUT) Log.d(TAG, "WindowLayout in setView:" + attrs);

                if (!compatibilityInfo.supportsScreen()) {
                    attrs.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
                    mLastInCompatMode = true;
                }

                mSoftInputMode = attrs.softInputMode;
                mWindowAttributesChanged = true;
                mWindowAttributesChangesFlag = WindowManager.LayoutParams.EVERYTHING_CHANGED;
				// 赋值根view
                mAttachInfo.mRootView = view;
                mAttachInfo.mScalingRequired = mTranslator != null;
                mAttachInfo.mApplicationScale =
                        mTranslator == null ? 1.0f : mTranslator.applicationScale;
                if (panelParentView != null) {
                    mAttachInfo.mPanelParentWindowToken
                            = panelParentView.getApplicationWindowToken();
                }
                mAdded = true;
                int res; /* = WindowManagerImpl.ADD_OKAY; */

                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
				// 去发起与该ViewRootImpl关联的视图树的测量,布局,绘制等操作
                requestLayout();
				// 这里一般前面没有设置特别设置过mWindowAttributes.inputFeatures,所以if()为true
				// 一般情况我们不会设置当前窗口不接受任何输入事件
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
					// 关联设置当前视图和WindowManagerService
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }

                if (mTranslator != null) {
                    mTranslator.translateRectInScreenToAppWindow(mAttachInfo.mContentInsets);
                }
                mPendingOverscanInsets.set(0, 0, 0, 0);
                mPendingContentInsets.set(mAttachInfo.mContentInsets);
                mPendingStableInsets.set(mAttachInfo.mStableInsets);
                mPendingVisibleInsets.set(0, 0, 0, 0);
                if (DEBUG_LAYOUT) Log.v(TAG, "Added window " + mWindow);
				// 前面关联当前视图和WindowManagerService失败了,抛出异常,中断程序
                if (res < WindowManagerGlobal.ADD_OKAY) {
                    mAttachInfo.mRootView = null;
                    mAdded = false;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    switch (res) {
                        case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                        case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not valid; is your activity running?");
                        case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- token " + attrs.token
                                    + " is not for an application");
                        case WindowManagerGlobal.ADD_APP_EXITING:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- app for token " + attrs.token
                                    + " is exiting");
                        case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window -- window " + mWindow
                                    + " has already been added");
                        case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                            // Silently ignore -- we would have just removed it
                            // right away, anyway.
                            return;
                        case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window " + mWindow +
                                    " -- another window of this type already exists");
                        case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                            throw new WindowManager.BadTokenException(
                                    "Unable to add window " + mWindow +
                                    " -- permission denied for this window type");
                        case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                            throw new WindowManager.InvalidDisplayException(
                                    "Unable to add window " + mWindow +
                                    " -- the specified display can not be found");
                        case WindowManagerGlobal.ADD_INVALID_TYPE:
                            throw new WindowManager.InvalidDisplayException(
                                    "Unable to add window " + mWindow
                                    + " -- the specified window type is not valid");
                    }
                    throw new RuntimeException(
                            "Unable to add window -- unknown error code " + res);
                }

                if (view instanceof RootViewSurfaceTaker) {
                    mInputQueueCallback =
                        ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
                }
				// 一般情况前面已经初始化了mInputChannel对象
                if (mInputChannel != null) {
                    if (mInputQueueCallback != null) {
                        mInputQueue = new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
					// 重要,这里为当前窗口生成了应该WindowInputEventReceiver即窗口事件的分发接受者
					// 一般各种屏幕触摸等事件最开始由本地代码调用java层WindowInputEventReceiver类的onInputEvent方法处理
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }
				// 为window的顶层View即其DecorView mDecor关联一个ViewParent即当前的ViewRootImpl对象
                view.assignParent(this);
                mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
                mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

                if (mAccessibilityManager.isEnabled()) {
                    mAccessibilityInteractionConnectionManager.ensureConnection();
                }

                if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
                    view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
                }

                // Set up the input pipeline.
                CharSequence counterSuffix = attrs.getTitle();
                mSyntheticInputStage = new SyntheticInputStage();
                InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
                InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                        "aq:native-post-ime:" + counterSuffix);
                InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
                InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                        "aq:ime:" + counterSuffix);
                InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
                InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                        "aq:native-pre-ime:" + counterSuffix);
				// 设置各种窗口事件的分发至当前window视图的处理对象,比如View的touch触摸事件一般分发至ViewPostImeInputStage处理
                mFirstInputStage = nativePreImeStage;
                mFirstPostImeInputStage = earlyPostImeStage;
                mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
            }
        }
    }

由上面可知,ViewRootImpl的setView()方法主要做了两件事:1是调用requestLayout()去测量,布局,绘制当前视图;2是为当前窗体关联到了WindowManagerService系统服务,并初始化了当前视图接受到屏幕一些事件的分发处理对象.


##S13: ViewRootImpl.requestLayout()

    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
			// 非主线程调用的话即抛出异常,所以平时调用View.requestLayout()一定要在主线程调用
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
			// 这里mHandler使用的当前程序主线程的Looper,这里在在当前程序的主线程的消息队列MessageQueue中插
			// 入一个同步栏栅,让异步消息优先调用
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
			// 这里最后是向主线程消息队列中放了一个异步消息,并最终在主线程Looper循环中调用mTraversalRunnable.run()方法
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
			// 移除主线程消息队列的前面插入的栏栅,让消息队列中的非异步的消息正常分发调用
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
			// 真正地去走视图树的测量,布局,绘制和显示去了
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }

一般情况,我们在Activity的生命周期的调用Handler.post()或者View.post()方法,都是向主线程的消息队列中放了一个同步消息,而同步消息是受到消息队列是否插入同步栏栅过滤影响的,而ViewRootImpe中让视图绘制布局显示的是向消息队列放了一个异步消息,且在之前插入了一个同步栏栅过滤非异步的消息对象,所以一般情况,都是ViewRootImpe.performTraversals()优先调用.这也是为何我们在Activity.onCreate()调用Handler.post()就可以拿到某个View的宽高的原因.


####至此,从源码中分析了Activity生成之后到其视图窗口的生成,测量,布局,绘制的中间过程


* Android 6.0 & API Level 23
* Github: [Nvsleep](https://github.com/Nvsleep)
* 邮箱: lizhenqiao@126.com

	





