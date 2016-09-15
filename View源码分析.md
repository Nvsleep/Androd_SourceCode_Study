## View源码分析

* Android 6.0 & API Level  23
* Github: [Nvsleep](https://github.com/Nvsleep)
* 邮箱: lizhenqiao@126.com

##简述
* 主要分析从XML资源文件中生成View对象过程;
* 以及View的构造函数,measure(),layout()方法分析;
* invalidate()请求刷新重绘视图过程分析;
* View自身touch事件处理onTouchEvent()方法分析

##View对象的生成

一般View对象的生成有两种,从XML文件中通过LayoutInflater(子类PhoneLayoutInflater)的inflate()经反射生成对象,一种是直接new View(Context)直接通过构造函数生成;

在Activity.onCreate()中的setContentView(@LayoutRes int layoutResID),最终调用的是其window(PhoneWindow)的setContentView(int layoutResID):


    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
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
			// 填充资源文件中视图到activity的window的顶层视图内容部分
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
PhoneWindow是在Activity.attach()中生成的,且传入的Context是当前Activity对象本身,并由window的final Context mContext成员变量引用;

在为window初始化顶层视图DecorView后,会调用LayoutInflater的inflate(@LayoutRes int resource, @Nullable ViewGroup root)方法,下面具体看下该过程:

	// root参数为由资源文件生成的view的父view
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }

	// attachToRoot参数表示是否将生成的view 添加add到root父view中
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }
		// 从资源文件中解析出"layout"即与布局相关的Parser对象
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }

	// 主要方法
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
            final Context inflaterContext = mContext;
			// 转换为AttributeSet属性集合
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }
				
                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }
				// 首先看首节点是否属于"merge"标签,如果inflate时候没有给予父view,则会抛出异常
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
					// 真正生成view的地方
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
						// 传入给予的父view不为空,则根据属性集合在父view中为当前生成的view生成一个LayoutParams布局参数对象
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
							// 如果传入父view不为空,一般attachToRoot==true,就不会在这里直接为生成的view赋值LayoutParams
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
					// 这里去为当前生成的view填充其节点下面的子view
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
						// 这里将由xml生成的view树addview到传入的父view中
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
						// 如果调用inflate()没有传入root父view,或者attachToRoot==false,则当前方法返回的对象是反射生成的view本身,否则返回的其实是父view即传入的root参数
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (Exception e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                                + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }

            Trace.traceEnd(Trace.TRACE_TAG_VIEW);

            return result;
        }
    }

看下createViewFromTag()的实现

   	private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
        return createViewFromTag(parent, name, context, attrs, false);
    }

    View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        // Apply a theme wrapper, if allowed and one is specified.
        if (!ignoreThemeAttr) {
            final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
            final int themeResId = ta.getResourceId(0, 0);
            if (themeResId != 0) {
                context = new ContextThemeWrapper(context, themeResId);
            }
            ta.recycle();
        }

        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

        try {
            View view;
			// 一般情况这里的mFactory2==null,mFactory==null
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }
			// 在acitivity.attach()中为其window的LayoutInflater mLayoutInflater设置了mPrivateFactory = acitivity本身.
			// 这里主要用于在xml中直接使用Fragment标签,会去acitivity中生成该Fragment对象并返回其Fragment.onCreatView()中返回的该fragment封装的view
            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
						// 通过反射生成Android系统view
                        view = onCreateView(parent, name, attrs);
                    } else {
						// 通过反射生成我们自定义的view
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } catch (InflateException e) {
            throw e;

        } catch (ClassNotFoundException e) {
            final InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;

        } catch (Exception e) {
            final InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;
        }
    }

看下为当前节点view填充子节点view所调用的rInflateChildren()方法过程:

    final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }

    void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;

        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
				// 解析"tag"标签节点
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
				// 解析"include"标签节点
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
				// 由此可见"merge"节点只能是首节点
                throw new InflateException("<merge /> must be the root element");
            } else {
				// 这里同之前一样,先生成该节点view对象,然后循环填充该节点view的子view,最后会将该节点addview到该节点view的上一层级节点view即传入父节点view中.
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }
		// 通知该父节点View其内部所有子view已经由xml打入填充完毕
        if (finishInflate) {
            parent.onFinishInflate();
        }
    }

* 可以看出,在调用Acitivity.setContentView()方法完成后,已经将视图树建立完毕,即将资源文件各个view节点分别通过反射生成对象然后addview到父view中,且将xml资源文件的根view填充依附到window的顶层视图容器DecorView的内容部分mContentParent中.这一过程中,所有view都在构造函数中做了初始化,且都设置了由父view容器调用generateLayoutParams()生成对应的LayoutParams对象.
* 手动调用LayoutInflater.inflate()资源文件的时候,最大的区别就在inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) 中的root参数和attachToRoot参数设置,如果root为null则返回无设置LayoutParams()的xml根节点view对象,若root!=null,但是attachToRoot==false则返回是设置LayoutParams()的xml根节点view对象,若root!=null同时attachToRoot==true则将xml根view生成LayoutParams()且通过父view即root依附addview到root上

##View的构造函数
view的构造函数有5个,除了无参数的构造函数一般最后都调用如下2个:

    public View(Context context) {
        mContext = context;
        mResources = context != null ? context.getResources() : null;
        mViewFlags = SOUND_EFFECTS_ENABLED | HAPTIC_FEEDBACK_ENABLED;
        // Set some flags defaults
        mPrivateFlags2 =
                (LAYOUT_DIRECTION_DEFAULT << PFLAG2_LAYOUT_DIRECTION_MASK_SHIFT) |
                (TEXT_DIRECTION_DEFAULT << PFLAG2_TEXT_DIRECTION_MASK_SHIFT) |
                (PFLAG2_TEXT_DIRECTION_RESOLVED_DEFAULT) |
                (TEXT_ALIGNMENT_DEFAULT << PFLAG2_TEXT_ALIGNMENT_MASK_SHIFT) |
                (PFLAG2_TEXT_ALIGNMENT_RESOLVED_DEFAULT) |
                (IMPORTANT_FOR_ACCESSIBILITY_DEFAULT << PFLAG2_IMPORTANT_FOR_ACCESSIBILITY_SHIFT);
        mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
        setOverScrollMode(OVER_SCROLL_IF_CONTENT_SCROLLS);
        mUserPaddingStart = UNDEFINED_PADDING;
        mUserPaddingEnd = UNDEFINED_PADDING;
        mRenderNode = RenderNode.create(getClass().getName(), this);

        if (!sCompatibilityDone && context != null) {
            final int targetSdkVersion = context.getApplicationInfo().targetSdkVersion;

            // Older apps may need this compatibility hack for measurement.
            sUseBrokenMakeMeasureSpec = targetSdkVersion <= JELLY_BEAN_MR1;

            // Older apps expect onMeasure() to always be called on a layout pass, regardless
            // of whether a layout was requested on that View.
            sIgnoreMeasureCache = targetSdkVersion < KITKAT;

            Canvas.sCompatibilityRestore = targetSdkVersion < M;

            // In M and newer, our widgets can pass a "hint" value in the size
            // for UNSPECIFIED MeasureSpecs. This lets child views of scrolling containers
            // know what the expected parent size is going to be, so e.g. list items can size
            // themselves at 1/3 the size of their container. It breaks older apps though,
            // specifically apps that use some popular open source libraries.
            sUseZeroUnspecifiedMeasureSpec = targetSdkVersion < M;

            sCompatibilityDone = true;
        }
    }



    public View(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        this(context);

        final TypedArray a = context.obtainStyledAttributes(
                attrs, com.android.internal.R.styleable.View, defStyleAttr, defStyleRes);

        if (mDebugViewAttributes) {
            saveAttributeData(attrs, a);
        }

        Drawable background = null;

        int leftPadding = -1;
        int topPadding = -1;
        int rightPadding = -1;
        int bottomPadding = -1;
        int startPadding = UNDEFINED_PADDING;
        int endPadding = UNDEFINED_PADDING;

        int padding = -1;

        int viewFlagValues = 0;
        int viewFlagMasks = 0;

        boolean setScrollContainer = false;

        int x = 0;
        int y = 0;

        float tx = 0;
        float ty = 0;
        float tz = 0;
        float elevation = 0;
        float rotation = 0;
        float rotationX = 0;
        float rotationY = 0;
        float sx = 1f;
        float sy = 1f;
        boolean transformSet = false;

        int scrollbarStyle = SCROLLBARS_INSIDE_OVERLAY;
        int overScrollMode = mOverScrollMode;
        boolean initializeScrollbars = false;
        boolean initializeScrollIndicators = false;

        boolean startPaddingDefined = false;
        boolean endPaddingDefined = false;
        boolean leftPaddingDefined = false;
        boolean rightPaddingDefined = false;

        final int targetSdkVersion = context.getApplicationInfo().targetSdkVersion;

        final int N = a.getIndexCount();
		for (int i = 0; i < N; i++) {
			int attr = a.getIndex(i);
            switch (attr) {
			....

			}

		}
		....
		setOverScrollMode(overScrollMode);
        if (background != null) {
            setBackground(background);
        }
        internalSetPadding(
                mUserPaddingLeftInitial,
                topPadding >= 0 ? topPadding : mPaddingTop,
                mUserPaddingRightInitial,
                bottomPadding >= 0 ? bottomPadding : mPaddingBottom);
        if (x != 0 || y != 0) {
            scrollTo(x, y);
        }

        if (transformSet) {
            setTranslationX(tx);
            setTranslationY(ty);
            setTranslationZ(tz);
            setElevation(elevation);
            setRotation(rotation);
            setRotationX(rotationX);
            setRotationY(rotationY);
            setScaleX(sx);
            setScaleY(sy);
        }
		....

	}

可以看出,view的构造函数设置了初始化了一些基本属性,比如Context,mRenderNode等,主要还是解析并获取了对应属性集合中一些参数并赋值给成员变量,比如mBackground,mPaddingTop,mPaddingLeft,mPaddingRight,mPaddingBottom等等,为mRenderNode赋值设置translationX,translationY,rotationX等相关显示参数等;

##measure()测量方法

view的measure()是由父view分发调用(最开始是从ViewRootImpl的performMeasure()测量DecroView开始的)的,传入的参数widthMeasureSpec和heightMeasureSpec,是由32位int值来表示,高2位代表该view的测量模式,剩余30位代表了实际了父view根据自身尺寸情况和子view布局参数生成的一个期望值.


    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
		// 通常没有设置layoutMode,这里为false
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
		// 生成mMeasureCache Long-long稀疏数组中保存测量值的key
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
		// 这里看flag的PFLAG_FORCE_LAYOUT位是否被强制置位,或者测量参数与上次是否有不一样
        if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {

            // first clears the measured dimension flag
			// 清除flag的PFLAG_MEASURED_DIMENSION_SET位,表明view当前没有设置尺寸
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();
			// 查询保存在缓存中是否有当前测量值的key,有的话同时flag的PFLAG_FORCE_LAYOUT位没有强制设置的话则不需要重新onMeasure()测量
            int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                    mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
				// 这里去具体设置该view的测量宽mMeasuredWidth和高mMeasuredHeight
                onMeasure(widthMeasureSpec, heightMeasureSpec);
				// 清除flag3的PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT位
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
				// 如果是直接拿缓存没有走onMeasure()则在这里置该位,在layout()中再去onMeasure()
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
			// 这里检查一下,防止子类重新没有真正为该view赋值设置测量宽高
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }
			// 置flag的PFLAG_LAYOUT_REQUIRED位表明需要走layout()
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
		// 保存当前由父view下发的测量值
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
	
	// View默认的onMeasure()
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

	// 这里可以看出,view默认的测量中layoutParams宽高设置为match_parent和wrap_content效果一样,均为父view计算给出的size
	// 所以自定义view一般要重写onMeasure()区分设置layoutParams为wrap_content情况
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }


    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
		// 一般情况没有设置layoutmode时候,为false
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }

	// 这里真正为该view赋值mMeasuredWidth和mMeasuredHeight,同时标记flag的PFLAG_MEASURED_DIMENSION_SET位,表明已经设置好测量宽高
    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;
        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }

view的measure方法是final修饰的,通常在派生类中重新onMeasure()方法可以根据layoutParams的情况(match_parent或是wrap_content)重新设置赋值测量的宽和高.对于自定义ViewGroup,还要在onMeasure()中去遍历测量其子view.

##layout()布局方法

View的layout()方法也是由父view分发调用的,最开始是从ViewRootImpl的performLayout()开始的


    /**
     * @param l Left position, relative to parent 该view左边界距离父容器view左边界距离
     * @param t Top position, relative to parent 该view上边界距离父容器view上边界距离
     * @param r Right position, relative to parent 该view右边界距离父容器view左边界距离
     * @param b Bottom position, relative to parent 该view下边界距离父容器view上边界距离
     */
	// 很明显,传入的参数分别是当前view相对父view的左边界,上边界,由边界,下边界的距离
    public void layout(int l, int t, int r, int b) {
		// 这里如果在上面的measeure()中是直接从缓存中拿的没有走onMeasure()的话要去走一次onMeasure()
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
		// 首先记录上次该view四边与其父view左边界,上边界相应的距离,仅仅用于通知位置改变回调
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

		// ture 表示该view在父view中位置发生变化,即该view有一边与其父view左or上边界相应的距离发生改变
		// 真正改变位置的一般是在setFrame()中
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
		// 如果该view位置改变,或者flag的PFLAG_LAYOUT_REQUIRED被强制置位,则去重新onLayout()布局子view
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
			// 当前该view位置在其父view中位置变化了,则要重新去布局其子view,当然如果有子view的话
			// 一般想系统的LinearLayout,FrameLayout,RelativeLayout以及自定义viewGroup进要重新该方法
			// 去分配设置子view的相对该view的位置,View.java默认为空实现,ViewGroup一般要去重写该方法
            onLayout(changed, l, t, r, b);
			// 自己和子view都已经分配好位置后,清除flag的PFLAG_LAYOUT_REQUIRED位
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
			// 通知位置变化的监听回调
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }
		// 清除flag的PFLAG_FORCE_LAYOUT位
        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }

	// 为当前view指定其在父容器view中具体位置的是在该方法中
    protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;

        if (DBG) {
            Log.d("View", this + " View.setFrame(" + left + "," + top + ","
                    + right + "," + bottom + ")");
        }

        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;

            // Remember our drawn bit
			// 首先记录当前的flag的PFLAG_DRAWN位
            int drawn = mPrivateFlags & PFLAG_DRAWN;

			// 每个view都是一个矩形,实际宽度为对应边界
            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
			// 该view的位置即将改变,先去刷新一遍该view的当前视图区域
            invalidate(sizeChanged);
			// 分配赋值新位置,同时更新RenderNode mRenderNode的参数
            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
			// 置位flag的PFLAG_HAS_BOUNDS位,表示该view有边界束缚范围了
            mPrivateFlags |= PFLAG_HAS_BOUNDS;


			// view宽度or高度变化的方法通知
            if (sizeChanged) {
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }

            if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {
                // If we are visible, force the DRAWN bit to on so that
                // this invalidate will go through (at least to our parent).
                // This is because someone may have invalidated this view
                // before this call to setFrame came in, thereby clearing
                // the DRAWN bit.
				// 因为一个invalidate()过程开始进入后会清除flag的PFLAG_DRAWN位,直到当次UI绘制结束会重新置该位表示当次UI绘制完成
				// 所以这里为了保证一定走该方法去刷新视图,故先强制设置
                mPrivateFlags |= PFLAG_DRAWN;
				// 这里是真正去刷新位置变化后该view的视图了
                invalidate(sizeChanged);
                // parent display list may need to be recreated based on a change in the bounds
                // of any child
                invalidateParentCaches();
            }

            // Reset drawn bit to original value (invalidate turns it off)
			// 在invalidate()方法调用之后,恢复之前 的flag的PFLAG_DRAWN位
            mPrivateFlags |= drawn;
			// 该view位置变化的同时告知其背景size改变了
            mBackgroundSizeChanged = true;
            if (mForegroundInfo != null) {
                mForegroundInfo.mBoundsChanged = true;
            }

            notifySubtreeAccessibilityStateChangedIfNeeded();
        }
        return changed;
    }

从上面看出,view的layout()主要为该view指定其在父容器view中位置,且会两次调用invalidate()方法去刷新UI,之后会走onLayout()方法,如果该view本身是一个容器ViewGroup子类的话,则需要在该方法中调用子view的layout()方法为其子view分配指定相对其左边界和上边界的相对位置

##invalidate()

这个方法通常用来刷新某个view的UI视图,主要是会调用onDraw()方法;

该方法内部十分复杂,下面简要分析其大概过程

    /**
     * Invalidate the whole view. If the view is visible,
     * {@link #onDraw(android.graphics.Canvas)} will be called at some point in
     * the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     */
	// 由注释可知,调用该方法必须在主线程且会在随后调用onDraw()方法
    public void invalidate() {
        invalidate(true);
    }

    /**
     * This is where the invalidate() work actually happens. A full invalidate()
     * causes the drawing cache to be invalidated, but this function can be
     * called with invalidateCache set to false to skip that invalidation step
     * for cases that do not need it (for example, a component that remains at
     * the same dimensions with the same content).
     *
     * @param invalidateCache Whether the drawing cache for this view should be
     *            invalidated as well. This is usually true for a full
     *            invalidate, but may be set to false if the View's contents or
     *            dimensions have not changed.
     */
	// invalidateCache为ture会使view的flag的置位PFLAG_INVALIDATED,清除PFLAG_DRAWING_CACHE_VALID位
    void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }

    void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        if (mGhostView != null) {
            mGhostView.invalidate(true);
            return;
        }
		// 一般如果该view不是VISIBLE状态且不处于动画状态则中断刷新UI
        if (skipInvalidate()) {
            return;
        }

		// 首先判断PFLAG_DRAWN位上传UI绘制是否完成,PFLAG_HAS_BOUNDS位是否已经指定该view四边界位置,在layout()的setFrame()方法调用中置位了
        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            if (fullInvalidate) {
                mLastIsOpaque = isOpaque();
				// 这里清楚该位表明当次UI绘制已经开始没有完成
                mPrivateFlags &= ~PFLAG_DRAWN;
            }

            mPrivateFlags |= PFLAG_DIRTY;
			// PFLAG_INVALIDATED表示需要重新创建View的DisplayListCanvas
			// PFLAG_DRAWING_CACHE_VALID表示view的Bitmap drawing cache缓存有效可用,清除该位表明需要
			// 去创建该view的Bitmap drawing cache
            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            // Propagate the damage rectangle to the parent view.
			// mAttachInfo这个成员变量是在绘制显示窗口顶层视图时候创建ViewRootImpl这个顶级ViewParent对象的时候生成的
			// 并在performTraversals()中调用host.dispatchAttachedToWindow(mAttachInfo, 0)方法沿着view树体系向下赋值引用
            final AttachInfo ai = mAttachInfo;
			// 每个view的容器父view都赋值引用到子View的mParent成员变量,最顶层的容器View即DecorView的mParent指向了顶级ViewParent即ViewRootImpl对象
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }

            // Damage the entire projection receiver, if necessary.
            if (mBackground != null && mBackground.isProjected()) {
                final View receiver = getProjectionReceiver();
                if (receiver != null) {
                    receiver.damageInParent();
                }
            }

            // Damage the entire IsolatedZVolume receiving this view's shadow.
            if (isHardwareAccelerated() && getZ() != 0) {
                damageShadowReceiver();
            }
        }
    }

由上面可以看出,view.invalidate()最后是把其自身边界尺寸传递到上一层父view中,让父view调用invalidateChild()方法,一般父容器view是ViewGroup这个实现了ViewParent的抽象类的派生类,最顶层的容器View即DecorView的mParent指向了顶级ViewParent即ViewRootImpl,下面看下ViewGroup的invalidateChild()方法:

	public final void invalidateChild(View child, final Rect dirty) {

		ViewParent parent = this;
		....
		final int[] location = attachInfo.mInvalidateChildLocation;
        location[CHILD_LEFT_INDEX] = child.mLeft;
        location[CHILD_TOP_INDEX] = child.mTop;

		....

		do {
                View view = null;
                if (parent instanceof View) {
                    view = (View) parent;
                }

                if (drawAnimation) {
                    if (view != null) {
                        view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                    } else if (parent instanceof ViewRootImpl) {
                        ((ViewRootImpl) parent).mIsAnimating = true;
                    }
                }

                // If the parent is dirty opaque or not dirty, mark it dirty with the opaque
                // flag coming from the child that initiated the invalidate
                if (view != null) {
                    if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
                        opaqueFlag = PFLAG_DIRTY;
                    }
                    if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                        view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                    }
                }
				// 重点方法,这里parent先就是当前this对象
                parent = parent.invalidateChildInParent(location, dirty);
                if (view != null) {
                    // Account for transform on current parent
                    Matrix m = view.getMatrix();
                    if (!m.isIdentity()) {
                        RectF boundingRect = attachInfo.mTmpTransformRect;
                        boundingRect.set(dirty);
                        m.mapRect(boundingRect);
                        dirty.set((int) (boundingRect.left - 0.5f),
                                (int) (boundingRect.top - 0.5f),
                                (int) (boundingRect.right + 0.5f),
                                (int) (boundingRect.bottom + 0.5f));
                    }
                }
            } while (parent != null);

	}


    /**
     * Don't call or override this method. It is used for the implementation of
     * the view hierarchy.
     *
     * This implementation returns null if this ViewGroup does not have a parent,
     * if this ViewGroup is already fully invalidated or if the dirty rectangle
     * does not intersect with this ViewGroup's bounds.
     */
	// 联合设置统一当前需要重绘的UI视图矩形与该view本身的四边界
    public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
        if ((mPrivateFlags & PFLAG_DRAWN) == PFLAG_DRAWN ||
                (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID) {
            if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) !=
                        FLAG_OPTIMIZE_INVALIDATE) {
                dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                        location[CHILD_TOP_INDEX] - mScrollY);
                if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                    dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
                }

                final int left = mLeft;
                final int top = mTop;

                if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                    if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                        dirty.setEmpty();
                    }
                }
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;

                location[CHILD_LEFT_INDEX] = left;
                location[CHILD_TOP_INDEX] = top;

                if (mLayerType != LAYER_TYPE_NONE) {
                    mPrivateFlags |= PFLAG_INVALIDATED;
                }

                return mParent;

            } else {
                mPrivateFlags &= ~PFLAG_DRAWN & ~PFLAG_DRAWING_CACHE_VALID;

                location[CHILD_LEFT_INDEX] = mLeft;
                location[CHILD_TOP_INDEX] = mTop;
                if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                    dirty.set(0, 0, mRight - mLeft, mBottom - mTop);
                } else {
                    // in case the dirty rect extends outside the bounds of this container
                    dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
                }

                if (mLayerType != LAYER_TYPE_NONE) {
                    mPrivateFlags |= PFLAG_INVALIDATED;
                }

                return mParent;
            }
        }

        return null;
    }

invalidateChild()里面的do..while()循环一直不端的调用invalidateChildInParent(),invalidateChildInParent()返回父ViewParent并联合设置统一当前需要重绘的UI矩形与view本身四边界,直到view树向上到达ViewRootImpl的invalidateChildInParent()方法返回null中断循环,看下顶层ViewParent实现类的ViewRootImpl的invalidateChildInParent()方法;

    @Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
		// 检查方法调用的线程是否就是UI线程即APP主线程,不是则抛出异常,这也是为何invaldate()要在主线程调用的直接原因
        checkThread();
        if (DEBUG_DRAW) Log.v(TAG, "Invalidate child: " + dirty);

        if (dirty == null) {
            invalidate();
            return null;
        } else if (dirty.isEmpty() && !mIsAnimating) {
            return null;
        }

        if (mCurScrollY != 0 || mTranslator != null) {
            mTempRect.set(dirty);
            dirty = mTempRect;
            if (mCurScrollY != 0) {
                dirty.offset(0, -mCurScrollY);
            }
            if (mTranslator != null) {
                mTranslator.translateRectInAppWindowToScreen(dirty);
            }
            if (mAttachInfo.mScalingRequired) {
                dirty.inset(-1, -1);
            }
        }

        invalidateRectOnScreen(dirty);

        return null;
    }

	// mThread即当前APP进程主线程,在ViewRootImpl构造函数中赋值,因为ViewRootImpl必然是在主线程创建的
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }

	// 合并由view树体系向上传来的需要重绘的区域与成员变量mDirty区域,并保存在成员变量mDirty中
	// 之后调用scheduleTraversals()向主线程Looper的MessageQueue消息队列发送一个Message去请求刷新视图了
    private void invalidateRectOnScreen(Rect dirty) {
        final Rect localDirty = mDirty;
        if (!localDirty.isEmpty() && !localDirty.contains(dirty)) {
            mAttachInfo.mSetIgnoreDirtyState = true;
            mAttachInfo.mIgnoreDirtyState = true;
        }

        // Add the new dirty rect to the current one
        localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
        // Intersect with the bounds of the window to skip
        // updates that lie outside of the visible region
        final float appScale = mAttachInfo.mApplicationScale;
        final boolean intersected = localDirty.intersect(0, 0,
                (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
        if (!intersected) {
            localDirty.setEmpty();
        }
        if (!mWillDrawSoon && (intersected || mIsAnimating)) {
            scheduleTraversals();
        }
    }

如果当前已经有一个UI刷新过程在进行中的时候,mWillDrawSoon为true,则不再去刷新视图了;



##onTouchEvent()

不管是ViewGroup容器还是View本身,首先都是一个View,作为View都有自身的基本的touch事件处理逻辑,即View类中的onTouchEvent()方法;
如果是ViewGroup的话,一般是没有子view处理分发的事件或者自己拦截了事件强制分发到自身处理,然后调用其onTouchEvent()方法

PS:写到这里,不想在这篇文章分析view的touch事件处理了,今天中秋节,想想很有空,还是写一篇View和ViewGroup的事件分发,拦截,自身处理的源码分析文章.

* Android 6.0 & API Level  23
* Github: [Nvsleep](https://github.com/Nvsleep)
* 邮箱: lizhenqiao@126.com


> 寒蝉凄切，对长亭晚，骤雨初歇。都门帐饮无绪，留恋处，兰舟催发。执手相看泪眼，竟无语凝噎。念去去，千里烟波，暮霭沉沉楚天阔。
> 
> 多情自古伤离别，更那堪，冷落清秋节！今宵酒醒何处？杨柳岸，晓风残月。此去经年，应是良辰好景虚设。便纵有千种风情，更与何人说？
