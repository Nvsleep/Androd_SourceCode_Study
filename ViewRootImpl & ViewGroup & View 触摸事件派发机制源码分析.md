##ViewRootImpl & ViewGroup & View 触摸事件派发机制源码分析

* Android 6.0 & API Level  23
* Github: [Nvsleep](https://github.com/Nvsleep)
* 邮箱: lizhenqiao@126.com

##简述
* Activity顶层窗口接受屏幕触摸事件的准备以及对输入事件到来时候的预处理;
* ViewGroup的事件派发机制dispatchTouchEvent()分析;
* View自身的事件派发机制dispatchTouchEvent()分析;
* View自身onTouchEvent()方法分析;

## Activity窗口接受屏幕触摸事件的准备

一个Activity有一个PhoneWindow窗口,对应一个顶层ViewParent的实现类ViewRootImpl,窗口接受屏幕事件的准备工作是在ViewRootImpl.setView()中进行的

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
		synchronized (this) {
			if (mView == null) {
                mView = view;
				.....
				// Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
				// 这里先去向主线程发个消息稍后就去发起视图树的测量,布局,绘制显示工作,这样下面的操作完成后,视图窗口显示出来就可以马上接受各种输入事件了
                requestLayout();
				// 一般没有特别设置该窗口不能接受输入事件设置,这里if==true
				if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
					// 初始化设置一个当前窗口可以联通接受系统输入事件的的通道
                    mInputChannel = new InputChannel();
                }
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
					// 建立当前视图窗口与系统WindowManagerService服务的关联,并传入刚才创建的mInputChannel
					// 会在WindowManagerService服务进程为该APP窗口生成两个InputQueue,其中一个会调用InputQueue.transferTo()返回到当前APP进程窗口;另外一个保留在WindowManagerService为当前APP窗口创建的WindowState中
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
				.....
				// 很明显,view是PhoneWindow的内部类DecorView对象,而DecorView extends FrameLayout implements RootViewSurfaceTaker 
				// 所以这里if==ture
				if (view instanceof RootViewSurfaceTaker) {
					// 创建InputQueue的create和destroy的通知对象,这里DecroView.willYouTakeTheInputQueue()一般为null
                    mInputQueueCallback =
                        ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
                }
				// 从上面知道,一般没有特别设置该窗口不能接受输入事件设置,这里mInputChannel!=null已经生成
				if (mInputChannel != null) {
					// 一般情况DecroView.willYouTakeTheInputQueue()为null,所以这里if==false
                    if (mInputQueueCallback != null) {
                        mInputQueue = new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
					// 创建一个与当前窗口已经生成的InputChannel相关的接受输入事件的处理对象
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }
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
				// 这里设置当前各种不同类别输入事件到来时候按对应类型依次分别调用的处理对象
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

                mFirstInputStage = nativePreImeStage;
                mFirstPostImeInputStage = earlyPostImeStage;
                mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;

			}

		}

	}

从上面看出,在窗口初始化即ViewRootImpl.setView()中,会建立当前视图窗口体系与系WindowManagerService服务的关联,并在系统WindowManagerService服务为该窗口生成两个InputChannel输入事件通道,一个转移到当前顶层ViewParent即ViewRootImpl中,并在ViewRootImpl生成一个与输入事件通道关联的事件处理WindowInputEventReceiver内部类对象mInputEventReceiver:



    final class WindowInputEventReceiver extends InputEventReceiver {
        public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
            super(inputChannel, looper);
        }

		// 重写了父类onInputEvent,调用enqueueInputEvent实际处理native层返回的InputEvent输入事件
        @Override
        public void onInputEvent(InputEvent event) {
            enqueueInputEvent(event, this, 0, true);
        }

        @Override
        public void onBatchedInputEventPending() {
            if (mUnbufferedInputDispatch) {
                super.onBatchedInputEventPending();
            } else {
                scheduleConsumeBatchedInput();
            }
        }

        @Override
        public void dispose() {
            unscheduleConsumeBatchedInput();
            super.dispose();
        }
    }
	
	// 看下WindowInputEventReceiver父类
	public abstract class InputEventReceiver {

		....

	    // Called from native code.
	    @SuppressWarnings("unused")
		// 当输入事件到来时该方法由native层代码发起调用
	    private void dispatchInputEvent(int seq, InputEvent event) {
	        mSeqMap.put(event.getSequenceNumber(), seq);
	        onInputEvent(event);
	    }
		/**
	     * Called when an input event is received.
	     * The recipient should process the input event and then call {@link #finishInputEvent}
	     * to indicate whether the event was handled.  No new input events will be received
	     * until {@link #finishInputEvent} is called.
	     *
	     * @param event The input event that was received.
	     */
	    public void onInputEvent(InputEvent event) {
	        finishInputEvent(event, false);
	    }
		....

	}

至此可以看到,一个屏幕输入事件返回处理在当前视图窗口WindowInputEventReceiver内部类的onInputEvent(InputEvent event)中,随即调用了外部类ViewRootImpl.enqueueInputEvent(event, this, 0, true),需要注意的是这里的参数flags==0

    void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        adjustInputEventForCompatibility(event);
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);

        // Always enqueue the input event in order, regardless of its time stamp.
        // We do this because the application or the IME may inject key events
        // in response to touch events and we want to ensure that the injected keys
        // are processed in the order they were received and we cannot trust that
        // the time stamp of injected events are monotonic.
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        mPendingInputEventCount += 1;
        Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                mPendingInputEventCount);

        if (processImmediately) {
            doProcessInputEvents();
        } else {
            scheduleProcessInputEvents();
        }
    }

先是获取一个指向当前事件的输入事件队列QueuedInputEvent对象,然后根据情况赋值ViewRootImpl成员变量mPendingInputEventHead或者追加到mPendingInputEventTail的mNext尾部,随即调用doProcessInputEvents()

    void doProcessInputEvents() {
        // Deliver all pending input events in the queue.
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            if (mPendingInputEventHead == null) {
                mPendingInputEventTail = null;
            }
            q.mNext = null;

            mPendingInputEventCount -= 1;
            Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                    mPendingInputEventCount);

            long eventTime = q.mEvent.getEventTimeNano();
            long oldestEventTime = eventTime;
            if (q.mEvent instanceof MotionEvent) {
                MotionEvent me = (MotionEvent)q.mEvent;
                if (me.getHistorySize() > 0) {
                    oldestEventTime = me.getHistoricalEventTimeNano(0);
                }
            }
            mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);

            deliverInputEvent(q);
        }

        // We are done processing all input events that we can process right now
        // so we can clear the pending flag immediately.
        if (mProcessInputEventsScheduled) {
            mProcessInputEventsScheduled = false;
            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
        }
    }

只要mPendingInputEventHead!=null即当前待处理事件队列还有事件需要去被处理掉,就一直循环调用deliverInputEvent()

    private void deliverInputEvent(QueuedInputEvent q) {
        Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
                q.mEvent.getSequenceNumber());
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
        }

        InputStage stage;
        if (q.shouldSendToSynthesizer()) {
            stage = mSyntheticInputStage;
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }

        if (stage != null) {
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }

这里分别有两个判断,q.shouldSendToSynthesizer()和q.shouldSkipIme(),上面WindowInputEventReceiver拿到native返回的一个输入事件对象时候,调用的ViewRootImpl.enqueueInputEvent(event, this, 0, true),标记了输入事件的QueuedInputEvent对象至此为止falg==0,所以,这里一般情况调用在setView()中生成的NativePreImeInputStage mFirstInputStage对象接着去处理.从setView()方法中可知,一共生成了7个InputStage的子类对象依次接龙按事件类型对应去处理,入口是NativePreImeInputStage该子类对象,NativePreImeInputStage的顶层父类当然也是InputStage:


    abstract class InputStage {
        private final InputStage mNext;

        protected static final int FORWARD = 0;
        protected static final int FINISH_HANDLED = 1;
        protected static final int FINISH_NOT_HANDLED = 2;

        /**
         * Creates an input stage.
         * @param next The next stage to which events should be forwarded.
         */
        public InputStage(InputStage next) {
            mNext = next;
        }

        /**
         * Delivers an event to be processed.
         */
        public final void deliver(QueuedInputEvent q) {
            if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
                forward(q);
            } else if (shouldDropInputEvent(q)) {
                finish(q, false);
            } else {
                apply(q, onProcess(q));
            }
        }

        /**
         * Marks the the input event as finished then forwards it to the next stage.
         */
        protected void finish(QueuedInputEvent q, boolean handled) {
            q.mFlags |= QueuedInputEvent.FLAG_FINISHED;
            if (handled) {
                q.mFlags |= QueuedInputEvent.FLAG_FINISHED_HANDLED;
            }
            forward(q);
        }

        /**
         * Forwards the event to the next stage.
         */
        protected void forward(QueuedInputEvent q) {
            onDeliverToNext(q);
        }

        /**
         * Applies a result code from {@link #onProcess} to the specified event.
         */
        protected void apply(QueuedInputEvent q, int result) {
            if (result == FORWARD) {
                forward(q);
            } else if (result == FINISH_HANDLED) {
                finish(q, true);
            } else if (result == FINISH_NOT_HANDLED) {
                finish(q, false);
            } else {
                throw new IllegalArgumentException("Invalid result: " + result);
            }
        }

        /**
         * Called when an event is ready to be processed.
         * @return A result code indicating how the event was handled.
         */
        protected int onProcess(QueuedInputEvent q) {
            return FORWARD;
        }

        /**
         * Called when an event is being delivered to the next stage.
         */
        protected void onDeliverToNext(QueuedInputEvent q) {
            if (DEBUG_INPUT_STAGES) {
                Log.v(TAG, "Done with " + getClass().getSimpleName() + ". " + q);
            }
            if (mNext != null) {
                mNext.deliver(q);
            } else {
                finishInputEvent(q);
            }
        }	
		....
    }

* 从构造就可以看出,每个生成的InputStage对象都会有一个成员变量mNext指向下一个处理事件的InputStage对象;
* deliver()方法先判断该事件对象是否已经处理完成或者需要抛弃掉,都不满足则调用onProcess()处理该事件对象,处理完成后返回处理结果给apply()方法后续工作,根据onProcess()返回处理结果是否把事件传递给其mNext指向的下一个InputStage去处理;
* 当然具体处理是在子类的onProcess()中实现的了



    final class NativePreImeInputStage extends AsyncInputStage
            implements InputQueue.FinishedInputEventCallback {
        public NativePreImeInputStage(InputStage next, String traceCounter) {
            super(next, traceCounter);
        }

        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (mInputQueue != null && q.mEvent instanceof KeyEvent) {
                mInputQueue.sendInputEvent(q.mEvent, q, true, this);
                return DEFER;
            }
            return FORWARD;
        }

        @Override
        public void onFinishedInputEvent(Object token, boolean handled) {
            QueuedInputEvent q = (QueuedInputEvent)token;
            if (handled) {
                finish(q, true);
                return;
            }
            forward(q);
        }
    }
对于屏幕触摸事件,这里NativePreImeInputStage的onProcess()返回FORWARD,即交给其mNext即ViewPreImeInputStage去接龙处理

    final class ViewPreImeInputStage extends InputStage {
        public ViewPreImeInputStage(InputStage next) {
            super(next);
        }

        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            }
            return FORWARD;
        }

        private int processKeyEvent(QueuedInputEvent q) {
            final KeyEvent event = (KeyEvent)q.mEvent;
            if (mView.dispatchKeyEventPreIme(event)) {
                return FINISH_HANDLED;
            }
            return FORWARD;
        }
    
对于触摸事件ViewPreImeInputStage.onProcess()同样返回FORWARD,交给其其mNext即ViewPreImeInputStage去接龙处理ImeInputStage去处理

 	/**
     * Delivers input events to the ime.
     * Does not support pointer events.
     */
    final class ImeInputStage extends AsyncInputStage
            implements InputMethodManager.FinishedInputEventCallback {
        public ImeInputStage(InputStage next, String traceCounter) {
            super(next, traceCounter);
        }

        @Override
        protected int onProcess(QueuedInputEvent q) {
            if (mLastWasImTarget && !isInLocalFocusMode()) {
                InputMethodManager imm = InputMethodManager.peekInstance();
                if (imm != null) {
                    final InputEvent event = q.mEvent;
                    if (DEBUG_IMF) Log.v(TAG, "Sending input event to IME: " + event);
                    int result = imm.dispatchInputEvent(event, q, this, mHandler);
                    if (result == InputMethodManager.DISPATCH_HANDLED) {
                        return FINISH_HANDLED;
                    } else if (result == InputMethodManager.DISPATCH_NOT_HANDLED) {
                        // The IME could not handle it, so skip along to the next InputStage
                        return FORWARD;
                    } else {
                        return DEFER; // callback will be invoked later
                    }
                }
            }
            return FORWARD;
        }

        @Override
        public void onFinishedInputEvent(Object token, boolean handled) {
            QueuedInputEvent q = (QueuedInputEvent)token;
            if (handled) {
                finish(q, true);
                return;
            }
            forward(q);
        }
    }
很明显这里是处理输入法事件的,对于一般触摸事件同样返回FORWARD交给其mNext 即EarlyPostImeInputStage去处理,查看EarlyPostImeInputStage源码可知其对于触摸事件onProcess()返回FORWARD交给其mNext 即NativePostImeInputStage去处理,而NativePostImeInputStage.onProcess()同样返回FORWARD交给其mNext 即ViewPostImeInputStage去处理,看出ViewPostImeInputStage.onProcess():

    final class ViewPostImeInputStage extends InputStage {

		....

	    @Override
        protected int onProcess(QueuedInputEvent q) {
            if (q.mEvent instanceof KeyEvent) {
                return processKeyEvent(q);
            } else {
                // If delivering a new non-key event, make sure the window is
                // now allowed to start updating.
                handleDispatchWindowAnimationStopped();
                final int source = q.mEvent.getSource();
				// Whoops here!!
                if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                    return processPointerEvent(q);
                } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
                    return processTrackballEvent(q);
                } else {
                    return processGenericMotionEvent(q);
                }
            }
        }

		private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;

            mAttachInfo.mUnbufferedDispatchRequested = false;
            boolean handled = mView.dispatchPointerEvent(event);
            if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
                mUnbufferedInputDispatch = true;
                if (mConsumeBatchedInputScheduled) {
                    scheduleConsumeBatchedInputImmediately();
                }
            }
            return handled ? FINISH_HANDLED : FORWARD;
        }

		....

	}

对于触摸事件,这里的if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0)为ture,所以调用processPointerEvent()去处理,在processPointerEvent()中直接调用mView.dispatchPointerEvent(event),而mView即窗口的顶层视图DecroView;


####至此,终于看到ViewRootImpl对输入事件的准备工作以及经过一系列处理把触摸事件交由顶层视图DecroView的过程,DecroView和其父类FrameLayout,ViewGroup均没有重写此方法,故在View.dispatchPointerEvent()中:

    public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }

很明显,这里调用了dispatchTouchEvent()去处理,而DecorView重新了该方法:


        @Override
        public boolean dispatchTouchEvent(MotionEvent ev) {
			// 在Activity.attach()中已经把自己设置赋值到DecroView的外部类Window的Callback mCallback成员变量
			// 且在PhoneWindow生成DecroView对象的时候传入的mFeatureId=-1
            final Callback cb = getCallback();
            return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                    : super.dispatchTouchEvent(ev);
        }

所以这里,调用了Acitivity.dispatchTouchEvent()去处理

    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }

又让PhoneWindow.superDispatchTouchEvent()处理:

    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }

又让DecroView.superDispatchTouchEvent()处理:

        public boolean superDispatchTouchEvent(MotionEvent event) {
            return super.dispatchTouchEvent(event);
        }

DecroView继承之FrameLayout,FrameLayout继承之ViewGroup,所以事件终于到了ViewGroup的dispatchTouchEvent()中去处理了!

####到这里,事件终于到了ViewGroup.dispatchTouchEvent()了.


## ViewGroup的事件派发机制dispatchTouchEvent()分析

首先要明确,不管是onInterceptTouchEvent()拦截事件的方法,还是onTouchEvent()消耗处理事件的方法,对于一个触摸事件对象的派发,当事件到达在某一层View(广义,包括ViewGroup)上时候,其入口调用都是在一个dispatchTouchEvent()方法中进行中的.下面仔细分析下当某一层ViewGroup接受到一个触摸事件时候事件在该层中的派发机制

    /**
     * {@inheritDoc}
     */
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }
		// 首先定义事件是否被消耗处理结果,默认开始当然是false
        boolean handled = false;
		// 过滤没必要处理的事件
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
			// 首先判断是否是down事件,如果是down事件到达该层View则
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
				// 如果该层ViewGroup有当前处理接受触摸事件的子view对象mFirstTouchTarget,清除其关联的子view的
				// PFLAG_CANCEL_NEXT_UP_EVENT相关标记且生成一个cancle事件让其去处理,重置
				// TouchTarget mFirstTouchTarget触摸处理对象链
                cancelAndClearTouchTargets(ev);
				// 清除自身mGroupFlags的PFLAG_CANCEL_NEXT_UP_EVENT相关标记
				// 清除自身mGroupFlags的FLAG_DISALLOW_INTERCEPT标记
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
			// 这里如果是一个down事件,或者当前层ViewGroup已经有一个触摸处理对象链即该层有子view有能力处理触摸事件
			// 则if==true进入
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
				// 符合上面两种情况后,先读取其FLAG_DISALLOW_INTERCEPT标记即是否[不可拦截]事件
				// 这个标记在其子view调用其requestDisallowInterceptTouchEvent()方法会设置
				// 默认disallowIntercept为fasle
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
					// 这里表明当前层viewGroup是正常的有能力去走onInterceptTouchEvent()看是否正在拦截
					// ViewGroup默认onInterceptTouchEvent()返回false
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
					// 表明子view请求的该层viewGroup[不可拦截]事件有效
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
				// 如果当前事件是非down事件,同时该层ViewGroup还没有有能力接受处理事件的子view
				// 即其[触摸事件处理对象链]mFirstTouchTarget==null,则赋值intercepted=true
                intercepted = true;
            }

			// -----------------------------------
			// 到这里,可以看出intercepted=true只有两种情况:
			// 1.事件是非down事件同时该层ViewGroup还没有有能力接受处理事件的子view;
			// 2.该viewGroup自身onInterceptTouchEvent()返回了ture;
			// -----------------------------------

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
			// 检查当前ViewGoup自身的PFLAG_CANCEL_NEXT_UP_EVENT标记,如果该位有效则置为0同时返回true
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

			// -----------------------------------		
			// 这里canceled==ture有两种情况:
			// 1.该viewGroup自身PFLAG_CANCEL_NEXT_UP_EVENT标记有效;
			// 2.当前事件类型就是个cancle事件;
			// -----------------------------------

            // Update list of touch targets for pointer down, if needed.
			// 没有特别设置的情况下,在ViewGroup构造函数中调用initViewGroup()会置该位,即split=true
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {

				// -----------------------------------		
				// 能进入这里说明:
				// 1.该viewGroup自身PFLAG_CANCEL_NEXT_UP_EVENT标记无效;
				// 2.当前事件类型不是个cancle事件;
				// 3.该ViewGroup没有能力去拦截事件()
				// 4.move,down事件都可能进入这里
				// -----------------------------------

                // If the event is targeting accessiiblity focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

					// 过滤可能的move事件,下面只有在down事件到来时候才在该层ViewGroup去派发寻找可能的处理从down事件开始的这一段连续的触摸操作

                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
					// 很明显newTouchTarget==null,同时有子view的话则if==ture
                    if (newTouchTarget == null && childrenCount != 0) {
						// 当前触摸点到该viewGroup左边界距离
                        final float x = ev.getX(actionIndex);
						// // 当前触摸点到该viewGroup上边界距离
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
						// 按Z顺序由小到大放到list中,即Z小的角标小
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

							// for循环先取角标大的,即z大的子view

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

							// 这里先去看该子view是否可见,是否有动画,当前事件点是否在该view的边界内
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
								// 如果该view不可见同时没有动画,或者当前事件点不在该子view边界内,这去检查下一个子view;
                                continue;
                            }
								
							// ---------------------------------------
							// 进入这里表明当前的子view符合派发事件的基本条件!
							// 该事件点在该子view边界内,同时该子view可见或者有动画;
							// ---------------------------------------

							// 遍历当前的该层ViewGroup的[触摸处理对象链],找到与该子view关联的TouchTarget;
							// 一般情况应是返回null,因为只有down事件才会到这里,而该viewGroup首先
							// 就检查是否是down事件是的话就重置[触摸处理对象链]了
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }
							// 现在还没有派发事件给子view,所以先清楚其PFLAG_CANCEL_NEXT_UP_EVENT标记以防万一
                            resetCancelNextUpFlag(child);
							// 这里是真正的把down事件派发交给给该子view处理了;
							// 同时如果该子view实际是个ViewGroup的话,同样走的还是现在写的这个
							// [ViewGroup的事件派发机制dispatchTouchEvent()分析] O_O
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
								// 进入这里,表明该子view处理消耗了该down事件!
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
								// 既然该子view已经处理消耗了该事件,那么将该层viewGroup对象的成员变量
								// mFirstTouchTarget指向了一个关联了该子view的TouchTarget对象
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
								// 这里改变alreadyDispatchedToNewTouchTarget使其==true,后面用到
								// 注意只有是down事件而且子view消耗了该down事件才会使其==true同时跳出for循环
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

							// ------------------------------------
							// 走到这里表明虽然上述子view符合派发down事件,但是其没有消耗掉;
							// 即其dispatchTouchEvent()方法返回了false;
							// 那么继续for循环遍历查看下一个子view;
							// ------------------------------------

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }
					// 到这里如果newTouchTarget==null的话,表明该down事件没有被该层viewGroup的子view任何所消耗;
					// 既然如此,那么一般来说mFirstTouchTarget==null,所以不会走下面
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

			//---------------至此, 如果是down事件的话,该层ViewGroup的down事件可能的派发结束

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
				// 表明对于该层ViewGroup来说,没有任何与子view相关的触摸事件接受处理链;
				// 那么既然事件传到了该层ViewGroup,则就让自身作为一个View去处理!即会调用
				// View.dispatchTouchEvent(),等会下一个部分小段会分析该方法;
				// 同时,记录自身处理该事件的结果,true表示消耗处理了
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
				// 进入这里,表明对于该层的ViewGroup来说,是有子view有能力来处理该事件的;
				// 即其有子view接受处理并消耗这一事件系列开始的down事件,也有可能:当前就是down事件刚刚被其消耗
				// 即该层ViewGroup的mFirstTouchTarget不为null且与消耗了事件的子view相关联
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
				// 很明显,一开始target!=null
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
						// 这里表明:当前就是down事件且刚刚被子view处理消耗掉了
                        handled = true;
                    } else {
						// 这里表明:当前可能是move,up,cancle 事件,不可能是down事件了
						// 先检查该层ViewGroup是否拦截事件,虽然该层ViewGroup已经有事件处理能力的子view了
						// 如果检查到该子view的PFLAG_CANCEL_NEXT_UP_EVENT位不为0或者该层ViewGroup
						// 这个时候突然决定要去拦截事件了,则cancelChild==ture,随即调用dispatchTransformedTouchEvent()进一步处理
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
							 // 返回true,表明该子view消耗了事件,这里不是down事件,是否消耗没什么关系吧?!
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
				// 如果前面判断的该层ViewGroup的PFLAG_CANCEL_NEXT_UP_EVENT位不为0或者就是当前就是cancle事件
				// 重置TouchTarget mFirstTouchTarget触摸处理对象链
				// 清除自身mGroupFlags的PFLAG_CANCEL_NEXT_UP_EVENT相关标记
				// 清除自身mGroupFlags的FLAG_DISALLOW_INTERCEPT标记
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }

该过程有个实际处理事件的关键方法dispatchTransformedTouchEvent(),下面看看


  	 /**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {
            return false;
        }

        // If the number of pointers is the same and we don't need to perform any fancy
        // irreversible transformations, then we can reuse the motion event for this
        // dispatch as long as we are careful to revert any changes we make.
        // Otherwise we need to make a copy.
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }

* 该方法的第二个参数boolean cancel 为true的话,表示会强制将事件设置为cancel事件;
* 该方法的第三个参数View child,如果为null的话表示事件由该层ViewGroup自身调用View.dispatchTouchEvent()处理,不为null的话表示事件由该子view去处理;


## View的事件派发机制dispatchTouchEvent()

    public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }

        boolean result = false;

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }

        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
			// 这里首先检查是否设置了OnTouchListener事件触摸监听回调,然后看该View是否是ENABLED状态
			// 如果是的话,调用OnTouchListener.onTouch()去处理,如果onTouch()返回true,这标记result=true,表示事件已经被处理
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
			// 上面if不满足的情况下,会去走自己的onTouchEvent()方法去处理,如果自身的onTouchEvent()
			// 返回true,则表示该view消耗了事件,即dispatchTouchEvent()返回true
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
View的事件派发机制dispatchTouchEvent()就简单很多了,先看是否设置了OnTouchListener监听回调,同时看该view是否enable的,是的话,调用OnTouchListener.onTouch()方法让回调去处理,如果该方法执行且返回ture,这表示事件被消耗了,就不会走自身的onTouchEvent()处理,否则,就是走View自身的onTouchEvent()方法默认处理


##View自身对事件的处理onTouchEvent()


    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

		// 首先检查该view的flag的DISABLED位是否有效
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
			// 该view处于DISABLED状态,如果当前是up状态,就会将flag的PFLAG_PRESSED清除掉
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
			// 同时直接返回了该view的flag的CLICKABLE,LONG_CLICKABLE,CONTEXT_CLICKABLE位是否有效
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }

		// 这里表明该view是enable的

		// 如果该view设置了代理触摸对象,则交由代理对象处理,如果代理对象消耗掉了,则直接返回true
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

		// 下面直接判断了该view的flag的CLICKABLE,LONG_CLICKABLE,CONTEXT_CLICKABLE位是否有效
		// 如果有一个有效即进入处理,且一定返回true表示事件消耗掉
		// 如果flag的上述3个标记为全都无效,则直接返回false表示该view没有消耗处理掉该事件
        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch (action) {
                case MotionEvent.ACTION_UP:
					// up事件

					// 检查flag的PFLAG_PREPRESSED位先
					// down时候已经置flag的PFLAG_PREPRESSED位,同时如果down时候100ms延迟操作CheckForTap还没有执行,则prepressed==true
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

						// 如果长按事件触发并没有到,则一般if==true
                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {	
                            // This is a tap, so remove the longpress check
							// 取消长安触发检查
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
								// 生成一个PerformClick点击事件处理对象
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
								// 这里是up时候触发点击事件执行的地方
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }
						// 如果prepressed==true,生成延迟一定时间执行setPressed(false);
                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }
						// 试图取消tap检查操作
                        removeTapCallback();
                    }
					// 恢复默认mIgnoreNextUpEvent为false
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
					// down事件到来

					// 恢复默认的mHasPerformedLongPress是否执行长按为false
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
					// 很重要,表示该view是否在一个ViewGroup容器中,同时父view.shouldDelayChildPressedState()返回true
					// ViewGroup默认返回true
					// 一般只有顶层的DecroView不在一个ViewGroup容器中
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
					// 一般都是true
                    if (isInScrollingContainer) {
						// down事件,置flag的PFLAG_PREPRESSED表示将要设置到pressed状态
                        mPrivateFlags |= PFLAG_PREPRESSED;
						// 生成mPendingCheckForTap检查是否是Tap轻拍动作
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
						// 延迟100ms执行CheckForTap.run(),因为如果100ms内该view收到up,cancel
						// 或者move事件不在其边界范围内的话,会取消该延迟操作的执行
						// 如果时间流逝100ms同时该操作没有被取消的话,则会清除PFLAG_PREPRESSED位,置PFLAG_PRESSED位,同时发起检查长按操作
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(0);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
					// cancel事件到来,由ViewGroup.dispatchTouchEvent()可知,cancel事件触发一般是有可能父view强制派发的
					
					// 这里当然是置flag的PFLAG_PRESSED为0,同时试图取消tap和长按检查操作以防万一
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
					// 恢复默认mIgnoreNextUpEvent为false
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_MOVE:

					// move事件

					// 告知触摸事件的点的变化位置
                    drawableHotspotChanged(x, y);

                    // Be lenient about moving outside of buttons
					// 检查当前的触摸事件点是否在view边界+mTouchSlop的范围内
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
						// 如果不在:
						// 则试图取消down事件时候的延迟CheckForTap mPendingCheckForTap检查操作
                        removeTapCallback();
						// 同时如果view出于pressed状态则让其恢复,同时试图取消mPendingCheckForLongPress长安延迟操作
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }

            return true;
        }

        return false;
    }

看下down事件时候的CheckForTap的tap轻拍的检查触发,可以看到只有其执行了才会进行CheckForLongPress的长按触发检查

    private final class CheckForTap implements Runnable {
        public float x;
        public float y;

        @Override
        public void run() {
            mPrivateFlags &= ~PFLAG_PREPRESSED;
            setPressed(true, x, y);
			// 发起长按触发检查
            checkForLongClick(ViewConfiguration.getTapTimeout());
        }
    }

    private void checkForLongClick(int delayOffset) {
        if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) {
            mHasPerformedLongPress = false;

            if (mPendingCheckForLongPress == null) {
                mPendingCheckForLongPress = new CheckForLongPress();
            }
            mPendingCheckForLongPress.rememberWindowAttachCount();
            postDelayed(mPendingCheckForLongPress,
                    ViewConfiguration.getLongPressTimeout() - delayOffset);
        }
    }

    private final class CheckForLongPress implements Runnable {
        private int mOriginalWindowAttachCount;

        @Override
        public void run() {
            if (isPressed() && (mParent != null)
                    && mOriginalWindowAttachCount == mWindowAttachCount) {
				// 实际执行长按回调的地方,并且如果返回ture,则会标记mHasPerformedLongPress=true,会影响到点击事件的触发
                if (performLongClick()) {
                    mHasPerformedLongPress = true;
                }
            }
        }

        public void rememberWindowAttachCount() {
            mOriginalWindowAttachCount = mWindowAttachCount;
        }
    }


再看下up事件时候点击事件的触发

    private final class PerformClick implements Runnable {
        @Override
        public void run() {
            performClick();
        }
    }

    public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
			// 执行点击事件的回调
            li.mOnClickListener.onClick(this)
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }

####至此,已经基本上从源码层次分析完了ViewGroup和View对触摸事件派发机制,后续在ScrollView ViewPager的源码分析文章中再分析不同ViewGroup继承类他们各自特有的事件派发,拦截,处理情况.


* Android 6.0 & API Level  23
* Github: [Nvsleep](https://github.com/Nvsleep)
* 邮箱: lizhenqiao@126.com


	

