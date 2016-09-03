# ListView源码分析
项目中使用ListView还是挺多的,之前看过几次,很是容易遗忘,今特做记录如下

API LEVEL 23
###主要从以下几点进行源码分析
* 构造函数初始化
* onMeasure()
* onLayout()
* onInterceptTouchEvent()和onTouchEvent()
* listview.setAdapter() 以及 adapter.notifyDataSetChanged()

##简要
ListView继承之AbsListView抽象类,所以大部分分析的源码都在这两个类中

##构造函数初始化过程
父类AbsListView的初始化:

	public AbsListView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
		//初始化设置一些额外属性值
        initAbsListView();

        mOwnerThread = Thread.currentThread();

		//初始化XML文件中设置的某些默认属性值
        final TypedArray a = context.obtainStyledAttributes(
                attrs, R.styleable.AbsListView, defStyleAttr, defStyleRes);

        final Drawable selector = a.getDrawable(R.styleable.AbsListView_listSelector);
        if (selector != null) {
            setSelector(selector);
        }

        mDrawSelectorOnTop = a.getBoolean(R.styleable.AbsListView_drawSelectorOnTop, false);

        setStackFromBottom(a.getBoolean(
                R.styleable.AbsListView_stackFromBottom, false));
        setScrollingCacheEnabled(a.getBoolean(
                R.styleable.AbsListView_scrollingCache, true));
        setTextFilterEnabled(a.getBoolean(
                R.styleable.AbsListView_textFilterEnabled, false));
        setTranscriptMode(a.getInt(
                R.styleable.AbsListView_transcriptMode, TRANSCRIPT_MODE_DISABLED));
        setCacheColorHint(a.getColor(
                R.styleable.AbsListView_cacheColorHint, 0));
        setSmoothScrollbarEnabled(a.getBoolean(
                R.styleable.AbsListView_smoothScrollbar, true));
        setChoiceMode(a.getInt(
                R.styleable.AbsListView_choiceMode, CHOICE_MODE_NONE));
        setFastScrollEnabled(a.getBoolean(
                R.styleable.AbsListView_fastScrollEnabled, false));
        setFastScrollStyle(a.getResourceId(
                R.styleable.AbsListView_fastScrollStyle, 0));
        setFastScrollAlwaysVisible(a.getBoolean(
                R.styleable.AbsListView_fastScrollAlwaysVisible, false));
        a.recycle();
    }

    private void initAbsListView() {
        // Setting focusable in touch mode will set the focusable property to true
		// 设置ListView本身可以点击即可以消耗父View分发的事件
        setClickable(true);
        setFocusableInTouchMode(true);
		// 因为向上父类还继承之ViewGroup,ViewGroup默认不需要重写draw()方法, 
		// 从而setWillNotDraw(true),但是AbsListView为了滚动效果,自身重写了View的
		// draw(),主要用于实现滚动到最底部或最顶部的非OVER_SCROLL_NEVER模式的效果
        setWillNotDraw(false);
        setAlwaysDrawnWithCacheEnabled(false);
        setScrollingCacheEnabled(true);

		// 事件处理相关变量初始化
        final ViewConfiguration configuration = ViewConfiguration.get(mContext);
        mTouchSlop = configuration.getScaledTouchSlop();
        mMinimumVelocity = configuration.getScaledMinimumFlingVelocity();
        mMaximumVelocity = configuration.getScaledMaximumFlingVelocity();
        mOverscrollDistance = configuration.getScaledOverscrollDistance();
        mOverflingDistance = configuration.getScaledOverflingDistance();

        mDensityScale = getContext().getResources().getDisplayMetrics().density;
    }

ListView的初始化:

	public ListView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        final TypedArray a = context.obtainStyledAttributes(
                attrs, R.styleable.ListView, defStyleAttr, defStyleRes);

        final CharSequence[] entries = a.getTextArray(R.styleable.ListView_entries);
        if (entries != null) {
            setAdapter(new ArrayAdapter<>(context, R.layout.simple_list_item_1, entries));
        }

		// 获取Item分割线的drawable对象
        final Drawable d = a.getDrawable(R.styleable.ListView_divider);
        if (d != null) {
            // Use an implicit divider height which may be explicitly
            // overridden by android:dividerHeight further down.
            setDivider(d);
        }

        final Drawable osHeader = a.getDrawable(R.styleable.ListView_overScrollHeader);
        if (osHeader != null) {
            setOverscrollHeader(osHeader);
        }

        final Drawable osFooter = a.getDrawable(R.styleable.ListView_overScrollFooter);
        if (osFooter != null) {
            setOverscrollFooter(osFooter);
        }

        // Use an explicit divider height, if specified.
		// item分割线的高度
        if (a.hasValueOrEmpty(R.styleable.ListView_dividerHeight)) {
            final int dividerHeight = a.getDimensionPixelSize(
                    R.styleable.ListView_dividerHeight, 0);
            if (dividerHeight != 0) {
                setDividerHeight(dividerHeight);
            }
        }

        mHeaderDividersEnabled = a.getBoolean(R.styleable.ListView_headerDividersEnabled, true);
        mFooterDividersEnabled = a.getBoolean(R.styleable.ListView_footerDividersEnabled, true);

        a.recycle();
    }
