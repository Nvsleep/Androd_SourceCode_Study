# ListView源码分析
项目中使用ListView还是挺多的,之前看过几次,很是容易遗忘,今特做记录如下

* Android 6.0 & API Level  23
* Github:[Nvsleep](https://github.com/Nvsleep)
* 邮箱:lizhenqiao@126.com
* QQ: 522910000

###主要从以下几点进行源码分析
* 构造函数初始化
* onMeasure()
* onLayout()
* listview.setAdapter() 以及 adapter.notifyDataSetChanged()
* onInterceptTouchEvent()和onTouchEvent()


###简要
ListView继承之AbsListView抽象类,所以大部分分析的源码都在这两个类中

##构造函数初始化过程
**父类AbsListView的初始化:**

	public AbsListView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
		// 初始化设置一些额外属性值
        initAbsListView();

        mOwnerThread = Thread.currentThread();

		// 初始化XML文件中设置的某些默认属性值
        final TypedArray a = context.obtainStyledAttributes(
                attrs, R.styleable.AbsListView, defStyleAttr, defStyleRes);

        final Drawable selector = a.getDrawable(R.styleable.AbsListView_listSelector);
        if (selector != null) {
            setSelector(selector);
        }

        mDrawSelectorOnTop = a.getBoolean(R.styleable.AbsListView_drawSelectorOnTop, false);
		
		// 初始化设置mStackFromBottom,这个影响到布局子view的顺序方式,默认为false
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

**ListView的初始化:**


	public ListView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        final TypedArray a = context.obtainStyledAttributes(
                attrs, R.styleable.ListView, defStyleAttr, defStyleRes);

        final CharSequence[] entries = a.getTextArray(R.styleable.ListView_entries);
        if (entries != null) {
            setAdapter(new ArrayAdapter<>(context, R.layout.simple_list_item_1, entries));
        }
		// 获取item分割线的drawable对象
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

##onMeasure()
**ListView的onMeasure()方法**

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // Sets up mListPadding
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int childWidth = 0;
        int childHeight = 0;
        int childState = 0;

        mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
        if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED
                || heightMode == MeasureSpec.UNSPECIFIED)) {
            final View child = obtainView(0, mIsScrap);

            // Lay out child directly against the parent measure spec so that
            // we can obtain exected minimum width and height.
            measureScrapChild(child, 0, widthMeasureSpec, heightSize);

            childWidth = child.getMeasuredWidth();
            childHeight = child.getMeasuredHeight();
            childState = combineMeasuredStates(childState, child.getMeasuredState());

            if (recycleOnMeasure() && mRecycler.shouldRecycleViewType(
                    ((LayoutParams) child.getLayoutParams()).viewType)) {
                mRecycler.addScrapView(child, 0);
            }
        }

        if (widthMode == MeasureSpec.UNSPECIFIED) {
            widthSize = mListPadding.left + mListPadding.right + childWidth +
                    getVerticalScrollbarWidth();
        } else {
            widthSize |= (childState & MEASURED_STATE_MASK);
        }

        if (heightMode == MeasureSpec.UNSPECIFIED) {
            heightSize = mListPadding.top + mListPadding.bottom + childHeight +
                    getVerticalFadingEdgeLength() * 2;
        }

		// 有时候需要使ListView的高度等于所有子item view 可以重写onMeasure()方法使其调用以
		// 下代码
        if (heightMode == MeasureSpec.AT_MOST) {
            // TODO: after first layout we should maybe start at the first visible position, not 0
            heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
        }

        setMeasuredDimension(widthSize, heightSize);

        mWidthMeasureSpec = widthMeasureSpec;
    }

整体上ListView的onMeasure方法比较简单,普通.

##onLayout()
ListView由adapter.getView()获取的子view的layout方式在此实现

**父类AbsListView的onLayout():**

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);

        mInLayout = true;

        final int childCount = getChildCount();
        if (changed) {
            for (int i = 0; i < childCount; i++) {
                getChildAt(i).forceLayout();
            }
            mRecycler.markChildrenDirty();
        }

		// 由子类ListView 和 GridView实现,是核心布局方法代码,也是listview与adapter交互数据
		// 的主要入口函数
        layoutChildren();
        mInLayout = false;

        mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;

        // TODO: Move somewhere sane. This doesn't belong in onLayout().
        if (mFastScroll != null) {
            mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
        }
    }

**ListView的layoutChildren():**

	@Override
    protected void layoutChildren() {
        final boolean blockLayoutRequests = mBlockLayoutRequests;
        if (blockLayoutRequests) {
            return;
        }

        mBlockLayoutRequests = true;

        try {
            super.layoutChildren();

            invalidate();

            if (mAdapter == null) {
                resetList();
                invokeOnItemScrollListener();
                return;
            }

            final int childrenTop = mListPadding.top;
            final int childrenBottom = mBottom - mTop - mListPadding.bottom;
			// 每次即将进行layout子item view的时候先记录当前listview已有的child view个数
            final int childCount = getChildCount();

            int index = 0;
            int delta = 0;

            View sel;
            View oldSel = null;
            View oldFirst = null;
            View newSel = null;

            // Remember stuff we will need down below
            switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                index = mNextSelectedPosition - mFirstPosition;
                if (index >= 0 && index < childCount) {
                    newSel = getChildAt(index);
                }
                break;
            case LAYOUT_FORCE_TOP:
            case LAYOUT_FORCE_BOTTOM:
            case LAYOUT_SPECIFIC:
            case LAYOUT_SYNC:
                break;
            case LAYOUT_MOVE_SELECTION:
            default:
                // Remember the previously selected view
                index = mSelectedPosition - mFirstPosition;
                if (index >= 0 && index < childCount) {
                    oldSel = getChildAt(index);
                }

                // Remember the previous first child
                oldFirst = getChildAt(0);

                if (mNextSelectedPosition >= 0) {
                    delta = mNextSelectedPosition - mSelectedPosition;
                }

                // Caution: newSel might be null
                newSel = getChildAt(index + delta);
            }


            boolean dataChanged = mDataChanged;
            if (dataChanged) {
                handleDataChanged();
            }

            // Handle the empty set by removing all views that are visible
            // and calling it a day
            if (mItemCount == 0) {
                resetList();
                invokeOnItemScrollListener();
                return;
            } else if (mItemCount != mAdapter.getCount()) {
                throw new IllegalStateException("The content of the adapter has changed but "
                        + "ListView did not receive a notification. Make sure the content of "
                        + "your adapter is not modified from a background thread, but only from "
                        + "the UI thread. Make sure your adapter calls notifyDataSetChanged() "
                        + "when its content changes. [in ListView(" + getId() + ", " + getClass()
                        + ") with Adapter(" + mAdapter.getClass() + ")]");
            }

            setSelectedPositionInt(mNextSelectedPosition);

            AccessibilityNodeInfo accessibilityFocusLayoutRestoreNode = null;
            View accessibilityFocusLayoutRestoreView = null;
            int accessibilityFocusPosition = INVALID_POSITION;

            // Remember which child, if any, had accessibility focus. This must
            // occur before recycling any views, since that will clear
            // accessibility focus.
            final ViewRootImpl viewRootImpl = getViewRootImpl();
            if (viewRootImpl != null) {
                final View focusHost = viewRootImpl.getAccessibilityFocusedHost();
                if (focusHost != null) {
                    final View focusChild = getAccessibilityFocusedChild(focusHost);
                    if (focusChild != null) {
                        if (!dataChanged || isDirectChildHeaderOrFooter(focusChild)
                                || focusChild.hasTransientState() || mAdapterHasStableIds) {
                            // The views won't be changing, so try to maintain
                            // focus on the current host and virtual view.
                            accessibilityFocusLayoutRestoreView = focusHost;
                            accessibilityFocusLayoutRestoreNode = viewRootImpl
                                    .getAccessibilityFocusedVirtualView();
                        }

                        // If all else fails, maintain focus at the same
                        // position.
                        accessibilityFocusPosition = getPositionForView(focusChild);
                    }
                }
            }

            View focusLayoutRestoreDirectChild = null;
            View focusLayoutRestoreView = null;

            // Take focus back to us temporarily to avoid the eventual call to
            // clear focus when removing the focused child below from messing
            // things up when ViewAncestor assigns focus back to someone else.
            final View focusedChild = getFocusedChild();
            if (focusedChild != null) {
                // TODO: in some cases focusedChild.getParent() == null

                // We can remember the focused view to restore after re-layout
                // if the data hasn't changed, or if the focused position is a
                // header or footer.
                if (!dataChanged || isDirectChildHeaderOrFooter(focusedChild)
                        || focusedChild.hasTransientState() || mAdapterHasStableIds) {
                    focusLayoutRestoreDirectChild = focusedChild;
                    // Remember the specific view that had focus.
                    focusLayoutRestoreView = findFocus();
                    if (focusLayoutRestoreView != null) {
                        // Tell it we are going to mess with it.
                        focusLayoutRestoreView.onStartTemporaryDetach();
                    }
                }
                requestFocus();
            }
			
			// Pull all children into the RecycleBin.
            // These views will be reused if possible
            final int firstPosition = mFirstPosition;
            final RecycleBin recycleBin = mRecycler;
			// 只有在调用adapter.notifyDatasetChanged()方法一直到layout()布局结束,
			// dataChanged为true,默认为false
            if (dataChanged) {
				// dataChanged为true,说明当前listview是有数据的了,把当前所有的item view
				// 存放到RecycleBin对象的mScrapViews中保存
                for (int i = 0; i < childCount; i++) {
                    recycleBin.addScrapView(getChildAt(i), firstPosition+i);
                }
            } else {
				// dataChanged默认为false,第一次执行此方法走这里
                recycleBin.fillActiveViews(childCount, firstPosition);
            }

            // Clear out old views
			// 清除当前listview所有的子view
            detachAllViewsFromParent();
            recycleBin.removeSkippedScrap();

			// 在我们调用listview.setAdapter()时候,已经将mLayoutMode = LAYOUT_NORMAL;
			// 所以通常情况下可认为mLayoutMode == LAYOUT_NORMAL
            switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                if (newSel != null) {
                    sel = fillFromSelection(newSel.getTop(), childrenTop, childrenBottom);
                } else {
                    sel = fillFromMiddle(childrenTop, childrenBottom);
                }
                break;
            case LAYOUT_SYNC:
                sel = fillSpecific(mSyncPosition, mSpecificTop);
                break;
            case LAYOUT_FORCE_BOTTOM:
                sel = fillUp(mItemCount - 1, childrenBottom);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_FORCE_TOP:
                mFirstPosition = 0;
                sel = fillFromTop(childrenTop);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_SPECIFIC:
                sel = fillSpecific(reconcileSelectedPosition(), mSpecificTop);
                break;
            case LAYOUT_MOVE_SELECTION:
                sel = moveSelection(oldSel, newSel, delta, childrenTop, childrenBottom);
                break;
            default:
				// 通常情况都走这里
                if (childCount == 0) {
					// listview第一次布局childCount必然为0走这里
                    if (!mStackFromBottom) {
						// 通常我们没有外部调用listview.setStackFromBottom()
						// 成员变量mStackFromBottom均为false都走这里
                        final int position = lookForSelectablePosition(0, true);
                        setSelectedPositionInt(position);
						// 从上到上布局listview能显示得下的子view
                        sel = fillFromTop(childrenTop);
                    } else {
                        final int position = lookForSelectablePosition(mItemCount - 1, false);
                        setSelectedPositionInt(position);
                        sel = fillUp(mItemCount - 1, childrenBottom);
                    }
                } else {
					// 非第一次layout,之前记录的存在的子view个数childCount不为0
					// 包括两种情况:1.listview首次布局中的第二次执行的onlayout();
					// 2.在后续listview已经显示存在子view然后数据改变时候调用
					// adapter.nitifyDatasetChanged()方法时候
                    if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
                        sel = fillSpecific(mSelectedPosition,
                                oldSel == null ? childrenTop : oldSel.getTop());
                    } else if (mFirstPosition < mItemCount) {
						// 通常情况走这里,fillSpecific()会调用fillUp()和fillDown()布局子view
                        sel = fillSpecific(mFirstPosition,
                                oldFirst == null ? childrenTop : oldFirst.getTop());
                    } else {
                        sel = fillSpecific(0, childrenTop);
                    }
                }
                break;
            }

            // Flush any cached views that did not get reused above
			// 至此,listview已经布局完成能够显示得下的子view,将recycleBin可能剩余的
			// mActiveViews中view移动到mScrapViews以便于listview滑动时候复用
            recycleBin.scrapActiveViews();

            if (sel != null) {
                // The current selected item should get focus if items are
                // focusable.
                if (mItemsCanFocus && hasFocus() && !sel.hasFocus()) {
                    final boolean focusWasTaken = (sel == focusLayoutRestoreDirectChild &&
                            focusLayoutRestoreView != null &&
                            focusLayoutRestoreView.requestFocus()) || sel.requestFocus();
                    if (!focusWasTaken) {
                        // Selected item didn't take focus, but we still want to
                        // make sure something else outside of the selected view
                        // has focus.
                        final View focused = getFocusedChild();
                        if (focused != null) {
                            focused.clearFocus();
                        }
                        positionSelector(INVALID_POSITION, sel);
                    } else {
                        sel.setSelected(false);
                        mSelectorRect.setEmpty();
                    }
                } else {
                    positionSelector(INVALID_POSITION, sel);
                }
                mSelectedTop = sel.getTop();
            } else {
                final boolean inTouchMode = mTouchMode == TOUCH_MODE_TAP
                        || mTouchMode == TOUCH_MODE_DONE_WAITING;
                if (inTouchMode) {
                    // If the user's finger is down, select the motion position.
                    final View child = getChildAt(mMotionPosition - mFirstPosition);
                    if (child != null) {
                        positionSelector(mMotionPosition, child);
                    }
                } else if (mSelectorPosition != INVALID_POSITION) {
                    // If we had previously positioned the selector somewhere,
                    // put it back there. It might not match up with the data,
                    // but it's transitioning out so it's not a big deal.
                    final View child = getChildAt(mSelectorPosition - mFirstPosition);
                    if (child != null) {
                        positionSelector(mSelectorPosition, child);
                    }
                } else {
                    // Otherwise, clear selection.
                    mSelectedTop = 0;
                    mSelectorRect.setEmpty();
                }

                // Even if there is not selected position, we may need to
                // restore focus (i.e. something focusable in touch mode).
                if (hasFocus() && focusLayoutRestoreView != null) {
                    focusLayoutRestoreView.requestFocus();
                }
            }

            // Attempt to restore accessibility focus, if necessary.
            if (viewRootImpl != null) {
                final View newAccessibilityFocusedView = viewRootImpl.getAccessibilityFocusedHost();
                if (newAccessibilityFocusedView == null) {
                    if (accessibilityFocusLayoutRestoreView != null
                            && accessibilityFocusLayoutRestoreView.isAttachedToWindow()) {
                        final AccessibilityNodeProvider provider =
                                accessibilityFocusLayoutRestoreView.getAccessibilityNodeProvider();
                        if (accessibilityFocusLayoutRestoreNode != null && provider != null) {
                            final int virtualViewId = AccessibilityNodeInfo.getVirtualDescendantId(
                                    accessibilityFocusLayoutRestoreNode.getSourceNodeId());
                            provider.performAction(virtualViewId,
                                    AccessibilityNodeInfo.ACTION_ACCESSIBILITY_FOCUS, null);
                        } else {
                            accessibilityFocusLayoutRestoreView.requestAccessibilityFocus();
                        }
                    } else if (accessibilityFocusPosition != INVALID_POSITION) {
                        // Bound the position within the visible children.
                        final int position = MathUtils.constrain(
                                accessibilityFocusPosition - mFirstPosition, 0,
                                getChildCount() - 1);
                        final View restoreView = getChildAt(position);
                        if (restoreView != null) {
                            restoreView.requestAccessibilityFocus();
                        }
                    }
                }
            }

            // Tell focus view we are done mucking with it, if it is still in
            // our view hierarchy.
            if (focusLayoutRestoreView != null
                    && focusLayoutRestoreView.getWindowToken() != null) {
                focusLayoutRestoreView.onFinishTemporaryDetach();
            }
            
			// 布局完成之后的操作
            mLayoutMode = LAYOUT_NORMAL;
            mDataChanged = false;
            if (mPositionScrollAfterLayout != null) {
                post(mPositionScrollAfterLayout);
                mPositionScrollAfterLayout = null;
            }
            mNeedSync = false;
            setNextSelectedPositionInt(mSelectedPosition);

            updateScrollIndicators();

            if (mItemCount > 0) {
                checkSelectionChanged();
            }

            invokeOnItemScrollListener();
        } finally {
            if (!blockLayoutRequests) {
                mBlockLayoutRequests = false;
            }
        }
    }

RecycleBin是AbsListView的一个非静态内部类,主要有一个数组成员变量View[] mActiveViews 和 ArrayList<View>[] mScrapViews.mActiveViews存放的是当前ListView可以使用的待激活的子item view,而mScrapViews存放的是在ListView滑动过程中滑出屏幕来回收以便下次利用的子item view

**fillFromTop():**

 	/**
     * Fills the list from top to bottom, starting with mFirstPosition
     *
     * @param nextTop The location where the top of the first item should be
     *        drawn
     *
     * @return The view that is currently selected
     */
    private View fillFromTop(int nextTop) {
        mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
        mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
        if (mFirstPosition < 0) {
            mFirstPosition = 0;
        }
        return fillDown(mFirstPosition, nextTop);
    }

其实就是保证mFirstPosition>=0的情况下调用fillDown()从上到下依次布局子item view

**fillDown():**

    /**
     * Fills the list from pos down to the end of the list view.
     *
     * @param pos The first position to put in the list
     *
     * @param nextTop The location where the top of the item associated with pos
     *        should be drawn
     *
     * @return The view that is currently selected, if it happens to be in the
     *         range that we draw.
     */
    private View fillDown(int pos, int nextTop) {
        View selectedView = null;

        int end = (mBottom - mTop);
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            end -= mListPadding.bottom;
        }
		// while循环在listview范围内布局可见数量的子item view
		// nextTop == mListPadding.top,可认为是listview的mPaddingTop
		// end == mListPadding.bottom,可认为是listview的mPaddingBottom
		// nextTop < end说明下一个要装载的item view的getTop()依然可见,那当然要布局到listview中
        while (nextTop < end && pos < mItemCount) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

            nextTop = child.getBottom() + mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos++;
        }

        setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
        return selectedView;
    }
 
第一次布局情况下,参数pos==0,即从0开始从上到下依次布局约定数量的从adapter.getView()获取的子view


**makeAndAddView():**

    /**
     * Obtain the view and add it to our list of children. The view can be made
     * fresh, converted from an unused view, or used as is if it was in the
     * recycle bin.
     *
     * @param position Logical position in the list
     * @param y Top or bottom edge of the view to add
     * @param flow If flow is true, align top edge to y. If false, align bottom
     *        edge to y.
     * @param childrenLeft Left edge where children should be positioned
     * @param selected Is this position selected?
     * @return View that was added
     */
    private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
        View child;

		// 默认情况mDataChanged==false,在调用adapter.notifyDatasetChanged()之后的
		// layout()阶段中mDataChanged == true
        if (!mDataChanged) {
            // Try to use an existing view for this position
			// 首先从mRecycler的mActiveViews数组中尝试获取可直接用的item view
			// 在前面layoutChildren()方法中首先如果mDataChanged==false会尝试把item view
			// 放到mRecycler的mActiveViews中保存,mRecycler.getActiveView()中对应positon
			// 的view被获取到之后本身就不再保存之
            child = mRecycler.getActiveView(position);
            if (child != null) {
                // Found it -- we're using an existing child
                // This just needs to be positioned
                setupChild(child, position, y, flow, childrenLeft, selected, true);

                return child;
            }
        }

        // Make a new view for this position, or convert an unused view if possible
        child = obtainView(position, mIsScrap);

        // This needs to be positioned and measured
        setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

        return child;
    }
主要用于从RecycleBin或者adapter.getView()中获取子view并在setupChild()中设置以及布局之

**setupChild():**


	/**
     * Add a view as a child and make sure it is measured (if necessary) and
     * positioned properly.
     *
     * @param child The view to add
     * @param position The position of this child
     * @param y The y position relative to which this view will be positioned
     * @param flowDown If true, align top edge to y. If false, align bottom
     *        edge to y.
     * @param childrenLeft Left edge where children should be positioned
     * @param selected Is this position selected?
     * @param recycled Has this view been pulled from the recycle bin? If so it
     *        does not need to be remeasured.
     */
    private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
            boolean selected, boolean recycled) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "setupListItem");

        final boolean isSelected = selected && shouldShowSelector();
        final boolean updateChildSelected = isSelected != child.isSelected();
        final int mode = mTouchMode;
        final boolean isPressed = mode > TOUCH_MODE_DOWN && mode < TOUCH_MODE_SCROLL &&
                mMotionPosition == position;
        final boolean updateChildPressed = isPressed != child.isPressed();
		// 如果child view是曾经使用过的,已经测量measure过了不需要再次measure之
        final boolean needToMeasure = !recycled || updateChildSelected || child.isLayoutRequested();

        // Respect layout params that are already in the view. Otherwise make some up...
        // noinspection unchecked
        AbsListView.LayoutParams p = (AbsListView.LayoutParams) child.getLayoutParams();
        if (p == null) {
            p = (AbsListView.LayoutParams) generateDefaultLayoutParams();
        }
        p.viewType = mAdapter.getItemViewType(position);

        if ((recycled && !p.forceAdd) || (p.recycledHeaderFooter
                && p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
			// 表明该child view是曾经使用过的,只需要attch一下就行了
            attachViewToParent(child, flowDown ? -1 : 0, p);
        } else {
            p.forceAdd = false;
            if (p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                p.recycledHeaderFooter = true;
            }
			// 表明不一定是曾经使用过的,需要addview到listview中
            addViewInLayout(child, flowDown ? -1 : 0, p, true);
        }

        if (updateChildSelected) {
            child.setSelected(isSelected);
        }

        if (updateChildPressed) {
            child.setPressed(isPressed);
        }

        if (mChoiceMode != CHOICE_MODE_NONE && mCheckStates != null) {
            if (child instanceof Checkable) {
                ((Checkable) child).setChecked(mCheckStates.get(position));
            } else if (getContext().getApplicationInfo().targetSdkVersion
                    >= android.os.Build.VERSION_CODES.HONEYCOMB) {
                child.setActivated(mCheckStates.get(position));
            }
        }

        if (needToMeasure) {
			// 测量item view,在viewgroup所有子view都需要测量执行view.measure()方法之后才能布局然后显示出来
            final int childWidthSpec = ViewGroup.getChildMeasureSpec(mWidthMeasureSpec,
                    mListPadding.left + mListPadding.right, p.width);
            final int lpHeight = p.height;
            final int childHeightSpec;
            if (lpHeight > 0) {
                childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
            } else {
                childHeightSpec = MeasureSpec.makeSafeMeasureSpec(getMeasuredHeight(),
                        MeasureSpec.UNSPECIFIED);
            }
            child.measure(childWidthSpec, childHeightSpec);
        } else {
            cleanupLayoutState(child);
        }

        final int w = child.getMeasuredWidth();
        final int h = child.getMeasuredHeight();
        final int childTop = flowDown ? y : y - h;

		// 放置该子view
        if (needToMeasure) {
            final int childRight = childrenLeft + w;
            final int childBottom = childTop + h;
            child.layout(childrenLeft, childTop, childRight, childBottom);
        } else {
            child.offsetLeftAndRight(childrenLeft - child.getLeft());
            child.offsetTopAndBottom(childTop - child.getTop());
        }

        if (mCachingStarted && !child.isDrawingCacheEnabled()) {
            child.setDrawingCacheEnabled(true);
        }

        if (recycled && (((AbsListView.LayoutParams)child.getLayoutParams()).scrappedFromPosition)
                != position) {
            child.jumpDrawablesToCurrentState();
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

**obtainView():**


    /**
     * Get a view and have it show the data associated with the specified
     * position. This is called when we have already discovered that the view is
     * not available for reuse in the recycle bin. The only choices left are
     * converting an old view or making a new one.
     *
     * @param position The position to display
     * @param isScrap Array of at least 1 boolean, the first entry will become true if
     *                the returned view was taken from the scrap heap, false if otherwise.
     *
     * @return A view displaying the data associated with the specified position
     */
    View obtainView(int position, boolean[] isScrap) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "obtainView");

        isScrap[0] = false;

		// 此部分忽略 
        // Check whether we have a transient state view. Attempt to re-bind the
        // data and discard the view if we fail.
        final View transientView = mRecycler.getTransientStateView(position);
        if (transientView != null) {
            final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

            // If the view type hasn't changed, attempt to re-bind the data.
            if (params.viewType == mAdapter.getItemViewType(position)) {
                final View updatedView = mAdapter.getView(position, transientView, this);

                // If we failed to re-bind the data, scrap the obtained view.
                if (updatedView != transientView) {
                    setItemViewLayoutParams(updatedView, position);
                    mRecycler.addScrapView(updatedView, position);
                }
            }

            isScrap[0] = true;

            // Finish the temporary detach started in addScrapView().
            transientView.dispatchFinishTemporaryDetach();
            return transientView;
        }

		// 关键部分
		// 先从mRecycler的mScrapViews数组中获取一个在滑动时候废弃保存的子view
        final View scrapView = mRecycler.getScrapView(position);
		// 平时写的adapter的getView()方法在此被调用了
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
				// 获取的不一样就把刚才的scrapView再次加入到mRecycler的mScrapViews数组中
                mRecycler.addScrapView(scrapView, position);
            } else {
				// 保存已经获取到废弃view且可以使用
                isScrap[0] = true;

                // Finish the temporary detach started in addScrapView().
                child.dispatchFinishTemporaryDetach();
            }
        }

        if (mCacheColorHint != 0) {
            child.setDrawingCacheBackgroundColor(mCacheColorHint);
        }

        if (child.getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
            child.setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
        }

        setItemViewLayoutParams(child, position);

        if (AccessibilityManager.getInstance(mContext).isEnabled()) {
            if (mAccessibilityDelegate == null) {
                mAccessibilityDelegate = new ListItemAccessibilityDelegate();
            }
            if (child.getAccessibilityDelegate() == null) {
                child.setAccessibilityDelegate(mAccessibilityDelegate);
            }
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);

        return child;
    }
返回的要么是adapter.getView()中调用inflate获取的view,要么是mRecycler的mScrapViews数组中获取的.

**fillSpecific():**

 	/**
     * Put a specific item at a specific location on the screen and then build
     * up and down from there.
     *
     * @param position The reference view to use as the starting point
     * @param top Pixel offset from the top of this view to the top of the
     *        reference view.
     *
     * @return The selected view, or null if the selected view is outside the
     *         visible area.
     */
    private View fillSpecific(int position, int top) {
        boolean tempIsSelected = position == mSelectedPosition;
		// 首先获取并设置布局当前positon的item view
        View temp = makeAndAddView(position, top, true, mListPadding.left, tempIsSelected);
        // Possibly changed again in fillUp if we add rows above this one.
        mFirstPosition = position;

        View above;
        View below;

        final int dividerHeight = mDividerHeight;

		// 然后分别调用fillUp()和fillDown()向上和向下获取并设置布局其他item view
        if (!mStackFromBottom) {
			// 通常情况走这里
            above = fillUp(position - 1, temp.getTop() - dividerHeight);
            // This will correct for the top of the first view not touching the top of the list
			// 保证listview的第一个或者最后一个item view在其paddingTop或是paddingBottom内
            adjustViewsUpOrDown();
            below = fillDown(position + 1, temp.getBottom() + dividerHeight);
            int childCount = getChildCount();
            if (childCount > 0) {
                correctTooHigh(childCount);
            }
        } else {
            below = fillDown(position + 1, temp.getBottom() + dividerHeight);
            // This will correct for the bottom of the last view not touching the bottom of the list
            adjustViewsUpOrDown();
            above = fillUp(position - 1, temp.getTop() - dividerHeight);
            int childCount = getChildCount();
            if (childCount > 0) {
                 correctTooLow(childCount);
            }
        }

        if (tempIsSelected) {
            return temp;
        } else if (above != null) {
            return above;
        } else {
            return below;
        }
    }


##ListView.setAdapter()


    /**
     * Sets the data behind this ListView.
     *
     * The adapter passed to this method may be wrapped by a {@link WrapperListAdapter},
     * depending on the ListView features currently in use. For instance, adding
     * headers and/or footers will cause the adapter to be wrapped.
     *
     * @param adapter The ListAdapter which is responsible for maintaining the
     *        data backing this list and for producing a view to represent an
     *        item in that data set.
     *
     * @see #getAdapter() 
     */
    @Override
    public void setAdapter(ListAdapter adapter) {
        if (mAdapter != null && mDataSetObserver != null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver);
        }
		// 将一些成员变量还原设置为初始默认值
        resetList();
		// mRecycler的mScrapViews清空并执行listview.removeDetachedView
        mRecycler.clear();

        if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
			// 如果listview有headerView或者FooterView则会生成包装adapter
            mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
        } else {
            mAdapter = adapter;
        }

        mOldSelectedPosition = INVALID_POSITION;
        mOldSelectedRowId = INVALID_ROW_ID;

        // AbsListView#setAdapter will update choice mode states.
        super.setAdapter(adapter);

        if (mAdapter != null) {
            mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();
            mOldItemCount = mItemCount;
			// listview的数据源的item个数
            mItemCount = mAdapter.getCount();
            checkFocus();
			
			// 重新生成一个内部类对象并将其注册到adapter中,用于通知回调数据源改变
            mDataSetObserver = new AdapterDataSetObserver();
            mAdapter.registerDataSetObserver(mDataSetObserver);
			
			// 设置listview的数据源类型,并在mRecycler中初始化对应个数的scrapViews list
            mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

            int position;
            if (mStackFromBottom) {
                position = lookForSelectablePosition(mItemCount - 1, false);
            } else {
                position = lookForSelectablePosition(0, true);
            }
            setSelectedPositionInt(position);
            setNextSelectedPositionInt(position);

            if (mItemCount == 0) {
                // Nothing selected
                checkSelectionChanged();
            }
        } else {
            mAreAllItemsSelectable = true;
            checkFocus();
            // Nothing selected
            checkSelectionChanged();
        }
		// 会调用顶层viewRootImpl.performTraversals(),导致视图重绘,listview重新
		// mesaure layout ...
        requestLayout();
    }

**BaseAdapter.registerDataSetObserver():**

**BaseAdapter.notifyDataSetChanged():**

	public abstract class BaseAdapter implements ListAdapter, SpinnerAdapter {
    private final DataSetObservable mDataSetObservable = new DataSetObservable();

    public boolean hasStableIds() {
        return false;
    }
    
	// 注册监听回调
    public void registerDataSetObserver(DataSetObserver observer) {
        mDataSetObservable.registerObserver(observer);
    }

    public void unregisterDataSetObserver(DataSetObserver observer) {
        mDataSetObservable.unregisterObserver(observer);
    }
    
    /**
     * Notifies the attached observers that the underlying data has been changed
     * and any View reflecting the data set should refresh itself.
     */
    public void notifyDataSetChanged() {
        mDataSetObservable.notifyChanged();
    }

    /**
     * Notifies the attached observers that the underlying data is no longer valid
     * or available. Once invoked this adapter is no longer valid and should
     * not report further data set changes.
     */
    public void notifyDataSetInvalidated() {
        mDataSetObservable.notifyInvalidated();
    }

    public boolean areAllItemsEnabled() {
        return true;
    }

    public boolean isEnabled(int position) {
        return true;
    }

    public View getDropDownView(int position, View convertView, ViewGroup parent) {
        return getView(position, convertView, parent);
    }

    public int getItemViewType(int position) {
        return 0;
    }

    public int getViewTypeCount() {
        return 1;
    }
    
    public boolean isEmpty() {
        return getCount() == 0;
    }
}

**Observable.registerObserver():**


	public abstract class Observable<T> {
    /**
     * The list of observers.  An observer can be in the list at most
     * once and will never be null.
     */
    protected final ArrayList<T> mObservers = new ArrayList<T>();

    /**
     * Adds an observer to the list. The observer cannot be null and it must not already
     * be registered.
     * @param observer the observer to register
     * @throws IllegalArgumentException the observer is null
     * @throws IllegalStateException the observer is already registered
     */
    public void registerObserver(T observer) {
        if (observer == null) {
            throw new IllegalArgumentException("The observer is null.");
        }
        synchronized(mObservers) {
            if (mObservers.contains(observer)) {
                throw new IllegalStateException("Observer " + observer + " is already registered.");
            }
            mObservers.add(observer);
        }
    }

    /**
     * Removes a previously registered observer. The observer must not be null and it
     * must already have been registered.
     * @param observer the observer to unregister
     * @throws IllegalArgumentException the observer is null
     * @throws IllegalStateException the observer is not yet registered
     */
    public void unregisterObserver(T observer) {
        if (observer == null) {
            throw new IllegalArgumentException("The observer is null.");
        }
        synchronized(mObservers) {
            int index = mObservers.indexOf(observer);
            if (index == -1) {
                throw new IllegalStateException("Observer " + observer + " was not registered.");
            }
            mObservers.remove(index);
        }
    }

    /**
     * Remove all registered observers.
     */
    public void unregisterAll() {
        synchronized(mObservers) {
            mObservers.clear();
        }
    }
	}

BaseAdapter.notifyDataSetChanged()最终会调用AdapterView中的内部类AdapterDataSetObserver.onChanged()

    class AdapterDataSetObserver extends DataSetObserver {

        private Parcelable mInstanceState = null;

        @Override
        public void onChanged() {
            mDataChanged = true;
            mOldItemCount = mItemCount;
            mItemCount = getAdapter().getCount();

            // Detect the case where a cursor that was previously invalidated has
            // been repopulated with new data.
            if (AdapterView.this.getAdapter().hasStableIds() && mInstanceState != null
                    && mOldItemCount == 0 && mItemCount > 0) {
                AdapterView.this.onRestoreInstanceState(mInstanceState);
                mInstanceState = null;
            } else {
                rememberSyncState();
            }
            checkFocus();
			// 同样,最终调用viewRootImpl.performTraversals(),导致视图重绘,执行listview的 
			// measure layout 方法等
            requestLayout();
        }

        @Override
        public void onInvalidated() {
            mDataChanged = true;

            if (AdapterView.this.getAdapter().hasStableIds()) {
                // Remember the current state for the case where our hosting activity is being
                // stopped and later restarted
                mInstanceState = AdapterView.this.onSaveInstanceState();
            }

            // Data is invalid so we should reset our state
            mOldItemCount = mItemCount;
            mItemCount = 0;
            mSelectedPosition = INVALID_POSITION;
            mSelectedRowId = INVALID_ROW_ID;
            mNextSelectedPosition = INVALID_POSITION;
            mNextSelectedRowId = INVALID_ROW_ID;
            mNeedSync = false;

            checkFocus();
            requestLayout();
        }


##onInterceptTouchEvent():

	@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        final int actionMasked = ev.getActionMasked();
        View v;

        if (mPositionScroller != null) {
            mPositionScroller.stop();
        }

        if (mIsDetaching || !isAttachedToWindow()) {
            // Something isn't right.
            // Since we rely on being attached to get data set change notifications,
            // don't risk doing anything where we might try to resync and find things
            // in a bogus state.
            return false;
        }

        if (mFastScroll != null && mFastScroll.onInterceptTouchEvent(ev)) {
            return true;
        }

        switch (actionMasked) {
        case MotionEvent.ACTION_DOWN: {
            int touchMode = mTouchMode;
			// 如果手指放下时候 listview正出于fling滚动状态或者OVERSCROLL,则马上拦截事件交
			// 由自身ontouchEvent()处理
            if (touchMode == TOUCH_MODE_OVERFLING || touchMode == TOUCH_MODE_OVERSCROLL) {
                mMotionCorrection = 0;
                return true;
            }

            final int x = (int) ev.getX();
            final int y = (int) ev.getY();
            mActivePointerId = ev.getPointerId(0);
			// 获取当前触摸事件在对应的item view的positon,在listview中实现了该方法
            int motionPosition = findMotionRow(y);
            if (touchMode != TOUCH_MODE_FLING && motionPosition >= 0) {
                // User clicked on an actual view (and was not stopping a fling).
                // Remember where the motion event started
                v = getChildAt(motionPosition - mFirstPosition);
                mMotionViewOriginalTop = v.getTop();
                mMotionX = x;
				// 记录down事件时候的mMotionY初始值
                mMotionY = y;
                mMotionPosition = motionPosition;
				// 设置触摸事件模式为TOUCH_MODE_DOWN
                mTouchMode = TOUCH_MODE_DOWN;
                clearScrollingCache();
            }
			// 初试设置down事件时候mLastY值
            mLastY = Integer.MIN_VALUE;
            initOrResetVelocityTracker();
            mVelocityTracker.addMovement(ev);
            mNestedYOffset = 0;
            startNestedScroll(SCROLL_AXIS_VERTICAL);
            if (touchMode == TOUCH_MODE_FLING) {
                return true;
            }
            break;
        }

        case MotionEvent.ACTION_MOVE: {
            switch (mTouchMode) {
            case TOUCH_MODE_DOWN:
                int pointerIndex = ev.findPointerIndex(mActivePointerId);
                if (pointerIndex == -1) {
                    pointerIndex = 0;
                    mActivePointerId = ev.getPointerId(pointerIndex);
                }
                final int y = (int) ev.getY(pointerIndex);
                initVelocityTrackerIfNotExists();
                mVelocityTracker.addMovement(ev);
				// 判断是否拦截事件自己处理,此为onInterceptTouchEvent()方法核心代码
                if (startScrollIfNeeded((int) ev.getX(pointerIndex), y, null)) {
                    return true;
                }
                break;
            }
            break;
        }

        case MotionEvent.ACTION_CANCEL:
        case MotionEvent.ACTION_UP: {
			// touchMode恢复默认状态
            mTouchMode = TOUCH_MODE_REST;
            mActivePointerId = INVALID_POINTER;
			// 回收速度VelocityTracker相关资源
            recycleVelocityTracker();
            reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
            stopNestedScroll();
            break;
        }

        case MotionEvent.ACTION_POINTER_UP: {
            onSecondaryPointerUp(ev);
            break;
        }
        }

        return false;
    }

	
	// 这个方法在onInterceptTouchEvent的move事件中调用,在onTouchEvent()的onTouchMove()方法
	// 中开始时候也会调用
 	private boolean startScrollIfNeeded(int x, int y, MotionEvent vtev) {
        // Check if we have moved far enough that it looks more like a
        // scroll than a tap
		// 得到当前事件的y值与down事件时候设置的值的差值
        final int deltaY = y - mMotionY;
        final int distance = Math.abs(deltaY);
		// mScrollY!=0即overscroll为true ,核心为distance > mTouchSlop即拦截事件自己处理
		// mTouchSlop在构造函数中初始化并赋值了
        final boolean overscroll = mScrollY != 0;
        if ((overscroll || distance > mTouchSlop) &&
                (getNestedScrollAxes() & SCROLL_AXIS_VERTICAL) == 0) {
            createScrollingCache();
            if (overscroll) {
                mTouchMode = TOUCH_MODE_OVERSCROLL;
                mMotionCorrection = 0;
            } else {
				// 设置触摸模式为TOUCH_MODE_SCROLL,在onTouchEvent()用到
                mTouchMode = TOUCH_MODE_SCROLL;
                mMotionCorrection = deltaY > 0 ? mTouchSlop : -mTouchSlop;
            }
			// 取消子view的长按监听触发
            removeCallbacks(mPendingCheckForLongPress);
            setPressed(false);
            final View motionView = getChildAt(mMotionPosition - mFirstPosition);
			// listview拦截了事件本身处理,所以恢复可能设置子view的press状态
            if (motionView != null) {
                motionView.setPressed(false);
            }
			// 通知ScrollState状态变化回调
            reportScrollStateChange(OnScrollListener.SCROLL_STATE_TOUCH_SCROLL);
            // Time to start stealing events! Once we've stolen them, don't let anyone
            // steal from us
            final ViewParent parent = getParent();
            if (parent != null) {
                parent.requestDisallowInterceptTouchEvent(true);
            }
			// 作用如名,如果满足条件,滚动listview
            scrollIfNeeded(x, y, vtev);
            return true;
        }

        return false;
    }

**onTouchEvent()之onTouchDown():**


    private void onTouchDown(MotionEvent ev) {
        mActivePointerId = ev.getPointerId(0);

        if (mTouchMode == TOUCH_MODE_OVERFLING) {
            // Stopped the fling. It is a scroll.
			// 如果已经正在出于TOUCH_MODE_OVERFLING则down事件瞬间中断fling
            mFlingRunnable.endFling();
            if (mPositionScroller != null) {
                mPositionScroller.stop();
            }
            mTouchMode = TOUCH_MODE_OVERSCROLL;
            mMotionX = (int) ev.getX();
            mMotionY = (int) ev.getY();
            mLastY = mMotionY;
            mMotionCorrection = 0;
            mDirection = 0;
        } else {
            final int x = (int) ev.getX();
            final int y = (int) ev.getY();
            int motionPosition = pointToPosition(x, y);

            if (!mDataChanged) {
                if (mTouchMode == TOUCH_MODE_FLING) {
                    // Stopped a fling. It is a scroll.
                    createScrollingCache();
                    mTouchMode = TOUCH_MODE_SCROLL;		
                    mMotionCorrection = 0;
                    motionPosition = findMotionRow(y);
					// 根据y速度判断是否马上中断fling
                    mFlingRunnable.flywheelTouch();
                } else if ((motionPosition >= 0) && getAdapter().isEnabled(motionPosition)) {
                    // User clicked on an actual view (and was not stopping a
                    // fling). It might be a click or a scroll. Assume it is a
                    // click until proven otherwise.
					// 设置touch模式为TOUCH_MODE_DOWN
                    mTouchMode = TOUCH_MODE_DOWN;

                    // FIXME Debounce
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }

                    mPendingCheckForTap.x = ev.getX();
                    mPendingCheckForTap.y = ev.getY();
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                }
            }

            if (motionPosition >= 0) {
                // Remember where the motion event started
                final View v = getChildAt(motionPosition - mFirstPosition);
                mMotionViewOriginalTop = v.getTop();
            }

            mMotionX = x;
			// 记录mMotionY
            mMotionY = y; 
            mMotionPosition = motionPosition;
			// 记录mLastY = Integer.MIN_VALUE
            mLastY = Integer.MIN_VALUE;
        }

        if (mTouchMode == TOUCH_MODE_DOWN && mMotionPosition != INVALID_POSITION
                && performButtonActionOnTouchDown(ev)) {
                removeCallbacks(mPendingCheckForTap);
        }
    }


**onTouchEvent()之onTouchMove():**



    private void onTouchMove(MotionEvent ev, MotionEvent vtev) {
        int pointerIndex = ev.findPointerIndex(mActivePointerId);
        if (pointerIndex == -1) {
            pointerIndex = 0;
            mActivePointerId = ev.getPointerId(pointerIndex);
        }

        if (mDataChanged) {
            // Re-sync everything if data has been changed
            // since the scroll operation can query the adapter.
            layoutChildren();
        }

        final int y = (int) ev.getY(pointerIndex);

        switch (mTouchMode) {
			// 刚开始进入这里 touchMode为TOUCH_MODE_DOWN
            case TOUCH_MODE_DOWN:
            case TOUCH_MODE_TAP:
            case TOUCH_MODE_DONE_WAITING:
                // Check if we have moved far enough that it looks more like a
                // scroll than a tap. If so, we'll enter scrolling mode.
				// 刚开始还么有处于可滚动状态,故进入判断是否可以滚动,核心判断调节为当然事件y
				// 与down事件mMotionY值的差值绝对值是否大于mTouchSlop
                if (startScrollIfNeeded((int) ev.getX(pointerIndex), y, vtev)) {
                    break;
                }
                // Otherwise, check containment within list bounds. If we're
                // outside bounds, cancel any active presses.
                final View motionView = getChildAt(mMotionPosition - mFirstPosition);
                final float x = ev.getX(pointerIndex);
                if (!pointInView(x, y, mTouchSlop)) {
                    setPressed(false);
                    if (motionView != null) {
                        motionView.setPressed(false);
                    }
                    removeCallbacks(mTouchMode == TOUCH_MODE_DOWN ?
                            mPendingCheckForTap : mPendingCheckForLongPress);
                    mTouchMode = TOUCH_MODE_DONE_WAITING;
                    updateSelectorState();
                } else if (motionView != null) {
                    // Still within bounds, update the hotspot.
                    final float[] point = mTmpPoint;
                    point[0] = x;
                    point[1] = y;
                    transformPointToViewLocal(point, motionView);
                    motionView.drawableHotspotChanged(point[0], point[1]);
                }
                break;
            case TOUCH_MODE_SCROLL:
            case TOUCH_MODE_OVERSCROLL:
				// 如果已经进入到listview的滚动状态,则直接执行scrollIfNeeded根据条件判断是否
				// 进行滚动
                scrollIfNeeded((int) ev.getX(pointerIndex), y, vtev);
                break;
        }
    }


 	 private void scrollIfNeeded(int x, int y, MotionEvent vtev) {
        int rawDeltaY = y - mMotionY;
        int scrollOffsetCorrection = 0;
        int scrollConsumedCorrection = 0;
		// mLastY==Integer.MIN_VALUE表明刚达到条件进入滚动状态
        if (mLastY == Integer.MIN_VALUE) {
			// 保证了状态过度时候平稳滚动
            rawDeltaY -= mMotionCorrection;
        }
        if (dispatchNestedPreScroll(0, mLastY != Integer.MIN_VALUE ? mLastY - y : -rawDeltaY,
                mScrollConsumed, mScrollOffset)) {
            rawDeltaY += mScrollConsumed[1];
            scrollOffsetCorrection = -mScrollOffset[1];
            scrollConsumedCorrection = mScrollConsumed[1];
            if (vtev != null) {
                vtev.offsetLocation(0, mScrollOffset[1]);
                mNestedYOffset += mScrollOffset[1];
            }
        }
        final int deltaY = rawDeltaY;
		// 两次连续下发事件的y差值.如果mLastY==Integer.MIN_VALUE表明刚达到条件进入滚动状态
		// 此时incrementalDeltaY = rawDeltaY,而rawDeltaY已经在上面进行了
	    // (rawDeltaY -= mMotionCorrection),保证了平稳过度
        int incrementalDeltaY =
                mLastY != Integer.MIN_VALUE ? y - mLastY + scrollConsumedCorrection : deltaY;
        int lastYCorrection = 0;
		if (mTouchMode == TOUCH_MODE_SCROLL) {
            if (PROFILE_SCROLLING) {
                if (!mScrollProfilingStarted) {
                    Debug.startMethodTracing("AbsListViewScroll");
                    mScrollProfilingStarted = true;
                }
            }

            if (mScrollStrictSpan == null) {
                // If it's non-null, we're already in a scroll.
                mScrollStrictSpan = StrictMode.enterCriticalSpan("AbsListView-scroll");
            }

            if (y != mLastY) {
                // We may be here after stopping a fling and continuing to scroll.
                // If so, we haven't disallowed intercepting touch events yet.
                // Make sure that we do so in case we're in a parent that can intercept.
                if ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) == 0 &&
                        Math.abs(rawDeltaY) > mTouchSlop) {
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }

                final int motionIndex;
                if (mMotionPosition >= 0) {
                    motionIndex = mMotionPosition - mFirstPosition;
                } else {
                    // If we don't have a motion position that we can reliably track,
                    // pick something in the middle to make a best guess at things below.
                    motionIndex = getChildCount() / 2;
                }

                int motionViewPrevTop = 0;
                View motionView = this.getChildAt(motionIndex);
                if (motionView != null) {
                    motionViewPrevTop = motionView.getTop();
                }

                // No need to do all this work if we're not going to move anyway
                boolean atEdge = false;
				// trackMotionScroll()方法真正进行滚动处理
                if (incrementalDeltaY != 0) {
                    atEdge = trackMotionScroll(deltaY, incrementalDeltaY);
                }

                // Check to see if we have bumped into the scroll limit
                motionView = this.getChildAt(motionIndex);
                if (motionView != null) {
                    // Check if the top of the motion view is where it is
                    // supposed to be
                    final int motionViewRealTop = motionView.getTop();
					// 滚动到最顶部或者最底部
                    if (atEdge) {
                        // Apply overscroll

                        int overscroll = -incrementalDeltaY -
                                (motionViewRealTop - motionViewPrevTop);
                        if (dispatchNestedScroll(0, overscroll - incrementalDeltaY, 0, overscroll,
                                mScrollOffset)) {
                            lastYCorrection -= mScrollOffset[1];
                            if (vtev != null) {
                                vtev.offsetLocation(0, mScrollOffset[1]);
                                mNestedYOffset += mScrollOffset[1];
                            }
                        } else {
                            final boolean atOverscrollEdge = overScrollBy(0, overscroll,
                                    0, mScrollY, 0, 0, 0, mOverscrollDistance, true);

                            if (atOverscrollEdge && mVelocityTracker != null) {
                                // Don't allow overfling if we're at the edge
                                mVelocityTracker.clear();
                            }

                            final int overscrollMode = getOverScrollMode();
                            if (overscrollMode == OVER_SCROLL_ALWAYS ||
                                    (overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS &&
                                            !contentFits())) {
                                if (!atOverscrollEdge) {
                                    mDirection = 0; // Reset when entering overscroll.
                                    mTouchMode = TOUCH_MODE_OVERSCROLL;
                                }
                                if (incrementalDeltaY > 0) {
									// 顶部 OVER_SCROLL效果,draw()方法中实现
                                    mEdgeGlowTop.onPull((float) -overscroll / getHeight(),
                                            (float) x / getWidth());
                                    if (!mEdgeGlowBottom.isFinished()) {
                                        mEdgeGlowBottom.onRelease();
                                    }
                                    invalidateTopGlow();
                                } else if (incrementalDeltaY < 0) {
									// 底部 OVER_SCROLL效果,draw()方法中实现
                                    mEdgeGlowBottom.onPull((float) overscroll / getHeight(),
                                            1.f - (float) x / getWidth());
                                    if (!mEdgeGlowTop.isFinished()) {
                                        mEdgeGlowTop.onRelease();
                                    }
                                    invalidateBottomGlow();
                                }
                            }
                        }
                    }
                    mMotionY = y + lastYCorrection + scrollOffsetCorrection;
                }
				// 记录当前事件的y
                mLastY = y + lastYCorrection + scrollOffsetCorrection;
            }
        }else if (mTouchMode == TOUCH_MODE_OVERSCROLL){.....}



**滚动实现核心代码trackMotionScroll(int deltaY, int incrementalDeltaY):**

关键变量在第二个参数incrementalDeltaY,即两次连续事件的y差值



 	/**
     * Track a motion scroll
     *
     * @param deltaY Amount to offset mMotionView. This is the accumulated delta since the motion
     *        began. Positive numbers mean the user's finger is moving down the screen.
     * @param incrementalDeltaY Change in deltaY from the previous event.
     * @return true if we're already at the beginning/end of the list and have nothing to do.
     */
 	boolean trackMotionScroll(int deltaY, int incrementalDeltaY) {
        final int childCount = getChildCount();
        if (childCount == 0) {
            return true;
        }
		// listview 第一个item view的mTop值
        final int firstTop = getChildAt(0).getTop();
		// // listview 最后一个item view的mBottom值
        final int lastBottom = getChildAt(childCount - 1).getBottom();

        final Rect listPadding = mListPadding;

        // "effective padding" In this case is the amount of padding that affects
        // how much space should not be filled by items. If we don't clip to padding
        // there is no effective padding.
        int effectivePaddingTop = 0;
        int effectivePaddingBottom = 0;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            effectivePaddingTop = listPadding.top;
            effectivePaddingBottom = listPadding.bottom;
        }

         // FIXME account for grid vertical spacing too?
        final int spaceAbove = effectivePaddingTop - firstTop;
        final int end = getHeight() - effectivePaddingBottom;
        final int spaceBelow = lastBottom - end;

        final int height = getHeight() - mPaddingBottom - mPaddingTop;
        if (deltaY < 0) {
            deltaY = Math.max(-(height - 1), deltaY);
        } else {
            deltaY = Math.min(height - 1, deltaY);
        }
		// 限定incrementalDeltaY在合理的最值范围内
        if (incrementalDeltaY < 0) {
            incrementalDeltaY = Math.max(-(height - 1), incrementalDeltaY);
        } else {
            incrementalDeltaY = Math.min(height - 1, incrementalDeltaY);
        }

        final int firstPosition = mFirstPosition;

        // Update our guesses for where the first and last views are
        if (firstPosition == 0) {
            mFirstPositionDistanceGuess = firstTop - listPadding.top;
        } else {
            mFirstPositionDistanceGuess += incrementalDeltaY;
        }
        if (firstPosition + childCount == mItemCount) {
            mLastPositionDistanceGuess = lastBottom + listPadding.bottom;
        } else {
            mLastPositionDistanceGuess += incrementalDeltaY;
        }
		// 判断是否在最顶部且手指向下滑动,是的话即不能向下滑动了
        final boolean cannotScrollDown = (firstPosition == 0 &&
                firstTop >= listPadding.top && incrementalDeltaY >= 0);
		// 判断是否在最底部且手指向上滑动,是的话即不能向上滑动了
        final boolean cannotScrollUp = (firstPosition + childCount == mItemCount &&
                lastBottom <= getHeight() - listPadding.bottom && incrementalDeltaY <= 0);
		// listview无法滚动即返回
        if (cannotScrollDown || cannotScrollUp) {
            return incrementalDeltaY != 0;
        }
		// incrementalDeltaY<0说明手指是向上滑动的,即listview内容视图是向下移动显示的
        final boolean down = incrementalDeltaY < 0;

        final boolean inTouchMode = isInTouchMode();
        if (inTouchMode) {
            hideSelector();
        }

        final int headerViewsCount = getHeaderViewsCount();
        final int footerViewsStart = mItemCount - getFooterViewsCount();

        int start = 0;
        int count = 0;

        if (down) {
			// 手指是向上滑动 incrementalDeltaY < 0
            int top = -incrementalDeltaY;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                top += listPadding.top;
            }
            for (int i = 0; i < childCount; i++) {
                final View child = getChildAt(i);
                if (child.getBottom() >= top) {
                    break;
                } else {
					// 最top的子view已经滑出listview
                    count++;
                    int position = firstPosition + i;
                    if (position >= headerViewsCount && position < footerViewsStart) {
                        // The view will be rebound to new data, clear any
                        // system-managed transient state.
                        child.clearAccessibilityFocus();
						// 将最顶部滑出的子view 加入到mRecycler的mScrapViews中保存
                        mRecycler.addScrapView(child, position);
                    }
                }
            }
        } else {
			// 手指是向下滑动 incrementalDeltaY > 0
            int bottom = getHeight() - incrementalDeltaY;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                bottom -= listPadding.bottom;
            }
            for (int i = childCount - 1; i >= 0; i--) {
                final View child = getChildAt(i);
                if (child.getTop() <= bottom) {
                    break;
                } else {
					// 最底部的子view已经滑出listview
                    start = i;
                    count++;
                    int position = firstPosition + i;
                    if (position >= headerViewsCount && position < footerViewsStart) {
                        // The view will be rebound to new data, clear any
                        // system-managed transient state.
                        child.clearAccessibilityFocus();
						// 将最底部滑出的子view 加入到mRecycler的mScrapViews中保存
                        mRecycler.addScrapView(child, position);
                    }
                }
            }
        }

        mMotionViewNewTop = mMotionViewOriginalTop + deltaY;

        mBlockLayoutRequests = true;

        if (count > 0) {
			// 将上面滑出的子view 从listview中detach掉
            detachViewsFromParent(start, count);
            mRecycler.removeSkippedScrap();
        }

        // invalidate before moving the children to avoid unnecessary invalidate
        // calls to bubble up from the children all the way to the top
        if (!awakenScrollBars()) {
           invalidate();
        }
		
		// 核心滚动便宜代码,根据incrementalDeltaY同步偏移所有的子view
        offsetChildrenTopAndBottom(incrementalDeltaY);

        if (down) {
            mFirstPosition += count;
        }
		
		// 根据条件判断是否填充滑动进入listview的子view
        final int absIncrementalDeltaY = Math.abs(incrementalDeltaY);
        if (spaceAbove < absIncrementalDeltaY || spaceBelow < absIncrementalDeltaY) {
            fillGap(down);
        }

        if (!inTouchMode && mSelectedPosition != INVALID_POSITION) {
            final int childIndex = mSelectedPosition - mFirstPosition;
            if (childIndex >= 0 && childIndex < getChildCount()) {
                positionSelector(mSelectedPosition, getChildAt(childIndex));
            }
        } else if (mSelectorPosition != INVALID_POSITION) {
            final int childIndex = mSelectorPosition - mFirstPosition;
            if (childIndex >= 0 && childIndex < getChildCount()) {
                positionSelector(INVALID_POSITION, getChildAt(childIndex));
            }
        } else {
            mSelectorRect.setEmpty();
        }

        mBlockLayoutRequests = false;

        invokeOnItemScrollListener();

        return false;
    }

**fillGap():**

滚动过程判断需要加载填充滑动进的子view的处理部分

    @Override
    void fillGap(boolean down) {
        final int count = getChildCount();
        if (down) {
			// 手指是向上滑动,需要填充最底部滑动进的子view
            int paddingTop = 0;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                paddingTop = getListPaddingTop();
            }
            final int startOffset = count > 0 ? getChildAt(count - 1).getBottom() + mDividerHeight :
                    paddingTop;
            fillDown(mFirstPosition + count, startOffset);
            correctTooHigh(getChildCount());
        } else {
			// 手指是向下滑动,需要填充最顶部滑动进的子view
            int paddingBottom = 0;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                paddingBottom = getListPaddingBottom();
            }
            final int startOffset = count > 0 ? getChildAt(0).getTop() - mDividerHeight :
                    getHeight() - paddingBottom;
            fillUp(mFirstPosition - 1, startOffset);
            correctTooLow(getChildCount());
        }
    }


**onTouchEvent()之onTouchUp():**


	private void onTouchUp(MotionEvent ev) {
		switch (mTouchMode) {
		
		....

		case TOUCH_MODE_SCROLL:
            final int childCount = getChildCount();
            if (childCount > 0) {
                final int firstChildTop = getChildAt(0).getTop();
                final int lastChildBottom = getChildAt(childCount - 1).getBottom();
                final int contentTop = mListPadding.top;
                final int contentBottom = getHeight() - mListPadding.bottom;
                if (mFirstPosition == 0 && firstChildTop >= contentTop &&
                        mFirstPosition + childCount < mItemCount &&
                        lastChildBottom <= getHeight() - contentBottom) {
                    mTouchMode = TOUCH_MODE_REST;
                    reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
                } else {
                    final VelocityTracker velocityTracker = mVelocityTracker;
                    velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);

                    final int initialVelocity = (int)
                            (velocityTracker.getYVelocity(mActivePointerId) * mVelocityScale);
                    // Fling if we have enough velocity and we aren't at a boundary.
                    // Since we can potentially overfling more than we can overscroll, don't
                    // allow the weird behavior where you can scroll to a boundary then
                    // fling further.
                    boolean flingVelocity = Math.abs(initialVelocity) > mMinimumVelocity;
					// 判断满足fling条件则进入
                    if (flingVelocity &&
                            !((mFirstPosition == 0 &&
                                    firstChildTop == contentTop - mOverscrollDistance) ||
                              (mFirstPosition + childCount == mItemCount &&
                                    lastChildBottom == contentBottom + mOverscrollDistance))) {
                        if (!dispatchNestedPreFling(0, -initialVelocity)) {
                            if (mFlingRunnable == null) {
                                mFlingRunnable = new FlingRunnable();
                            }
                            reportScrollStateChange(OnScrollListener.SCROLL_STATE_FLING);
							// fling走起
                            mFlingRunnable.start(-initialVelocity);
                            dispatchNestedFling(0, -initialVelocity, true);
                        } else {
                            mTouchMode = TOUCH_MODE_REST;
                            reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
                        }
                    } else {
						// 不满足fling条件 
                        mTouchMode = TOUCH_MODE_REST;
                        reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
                        if (mFlingRunnable != null) {
                            mFlingRunnable.endFling();
                        }
                        if (mPositionScroller != null) {
                            mPositionScroller.stop();
                        }
                        if (flingVelocity && !dispatchNestedPreFling(0, -initialVelocity)) {
                            dispatchNestedFling(0, -initialVelocity, false);
                        }
                    }
                }
            } else {
                mTouchMode = TOUCH_MODE_REST;
                reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
            }
            break;

		....
		
		}
		 setPressed(false);
		// over_scroll_mode 时候资源回收处理
        if (mEdgeGlowTop != null) {
            mEdgeGlowTop.onRelease();
            mEdgeGlowBottom.onRelease();
        }

        // Need to redraw since we probably aren't drawing the selector anymore
        invalidate();
        removeCallbacks(mPendingCheckForLongPress);
        recycleVelocityTracker();

        mActivePointerId = INVALID_POINTER;

        if (PROFILE_SCROLLING) {
            if (mScrollProfilingStarted) {
                Debug.stopMethodTracing();
                mScrollProfilingStarted = false;
            }
        }

        if (mScrollStrictSpan != null) {
            mScrollStrictSpan.finish();
            mScrollStrictSpan = null;
        }
	}

Up事件的fling处理主要在AbsListView的内部类FlingRunnable中:

FlingRunnable.start():

	void start(int initialVelocity) {
            int initialY = initialVelocity < 0 ? Integer.MAX_VALUE : 0;
            mLastFlingY = initialY;
            mScroller.setInterpolator(null);
			// 设置scroller做fling动作
            mScroller.fling(0, initialY, 0, initialVelocity,
                    0, Integer.MAX_VALUE, 0, Integer.MAX_VALUE);
            mTouchMode = TOUCH_MODE_FLING;
			// 在下一帧调用run()方法
            postOnAnimation(this);

            if (PROFILE_FLINGING) {
                if (!mFlingProfilingStarted) {
                    Debug.startMethodTracing("AbsListViewFling");
                    mFlingProfilingStarted = true;
                }
            }

            if (mFlingStrictSpan == null) {
                mFlingStrictSpan = StrictMode.enterCriticalSpan("AbsListView-fling");
            }
        }

FlingRunnable实现了Runnable,复写的run():


		@Override
        public void run() {
            switch (mTouchMode) {
            default:
                endFling();
                return;

            case TOUCH_MODE_SCROLL:
                if (mScroller.isFinished()) {
                    return;
                }
                // Fall through
            case TOUCH_MODE_FLING: {
                if (mDataChanged) {
                    layoutChildren();
                }

                if (mItemCount == 0 || getChildCount() == 0) {
                    endFling();
                    return;
                }

                final OverScroller scroller = mScroller;
                boolean more = scroller.computeScrollOffset();
                final int y = scroller.getCurrY();

                // Flip sign to convert finger direction to list items direction
                // (e.g. finger moving down means list is moving towards the top)
                int delta = mLastFlingY - y;

                // Pretend that each frame of a fling scroll is a touch scroll
                if (delta > 0) {
                    // List is moving towards the top. Use first view as mMotionPosition
                    mMotionPosition = mFirstPosition;
                    final View firstView = getChildAt(0);
                    mMotionViewOriginalTop = firstView.getTop();

                    // Don't fling more than 1 screen
                    delta = Math.min(getHeight() - mPaddingBottom - mPaddingTop - 1, delta);
                } else {
                    // List is moving towards the bottom. Use last view as mMotionPosition
                    int offsetToLast = getChildCount() - 1;
                    mMotionPosition = mFirstPosition + offsetToLast;

                    final View lastView = getChildAt(offsetToLast);
                    mMotionViewOriginalTop = lastView.getTop();

                    // Don't fling more than 1 screen
                    delta = Math.max(-(getHeight() - mPaddingBottom - mPaddingTop - 1), delta);
                }

                // Check to see if we have bumped into the scroll limit
                View motionView = getChildAt(mMotionPosition - mFirstPosition);
                int oldTop = 0;
                if (motionView != null) {
                    oldTop = motionView.getTop();
                }

                // Don't stop just because delta is zero (it could have been rounded)
				// fling滚动过程模拟的滑动处理
                final boolean atEdge = trackMotionScroll(delta, delta);
                final boolean atEnd = atEdge && (delta != 0);
                if (atEnd) {
                    if (motionView != null) {
                        // Tweak the scroll for how far we overshot
                        int overshoot = -(delta - (motionView.getTop() - oldTop));
                        overScrollBy(0, overshoot, 0, mScrollY, 0, 0,
                                0, mOverflingDistance, false);
                    }
                    if (more) {
                        edgeReached(delta);
                    }
                    break;
                }

                if (more && !atEnd) {
					// fling没有结束 继续递归调用run()方法
                    if (atEdge) invalidate();
                    mLastFlingY = y;
                    postOnAnimation(this);
                } else {	
					// fling已经结束
                    endFling();

                    if (PROFILE_FLINGING) {
                        if (mFlingProfilingStarted) {
                            Debug.stopMethodTracing();
                            mFlingProfilingStarted = false;
                        }

                        if (mFlingStrictSpan != null) {
                            mFlingStrictSpan.finish();
                            mFlingStrictSpan = null;
                        }
                    }
                }
                break;
            }

			....
           
            }
            }
        }

至此,ListView源码分析完...

* Github:[Nvsleep](https://github.com/Nvsleep)
* 邮箱:lizhenqiao@126.com
* QQ: 522910000
* SHARE LOVE UP !
