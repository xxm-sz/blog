对于Android手机APP普通开发者来说，KeyEvent接触相对较少，相反接触较多的应该是TouchEvent。而Android TV开发者对KeyEvent的接触就非常频繁。这也是手机应用和TV应用的主要区别：一个主要响应手指触摸事件，一个响应遥控器按键事件。

本文主要基于Android 9.0的源码，踏着巨人的肩膀，进行分析的，个人能力有限，有误请多多指正。篇幅也比较长，对于流程不感兴趣的，可以直接看文末总结。

带着疑问学习本文：

1. KeyEvent在Activity、View层次结构是如何分发的？
2. 我们设置View的OnKeyListener和重写onKeyDown()函数，能否同时生效？
3. OnKeyListener与onKeyDown()的调用顺序？
4. 如果所有View都不消费事件，最后KeyEvent如何处理？


## ViewRootImpl
由于`TouchEvent`和`KeyEvent`等等相关Event的源头是`ViewRootImpl`，所以需要先了解什么是`ViewRootImpl`。在Android中，View之间的关系是以View树的形式组织的，也是说，可以通过根View查找到所有的 View，进而可以判断view树是否有view消费相关事件。在Activity中，根View就是 DecorView。ViewRootImpl 本身并不是一个View,而是View树的管理者，ViewRootImpl可以对View进行布局（`layout`），测量(`measure`)和绘制(`draw`)，以及分发事件。同时也是View和WindowManager的桥梁。

[关于ViewRootImpl的更多信息](https://juejin.im/entry/5a2e603bf265da432c23cdde)
### ViewPostImeInputStage
ViewRootImpl以不同的`InputStage`来管理不同的事件(`Event`)，界面相关的事件（触摸事件、按键事件、轨迹事件和手势事件）由ViewRootImpl的内部类`ViewPostImeInputStage`来完成。`ViewPostImeInputStage`对象内的`onProcess()`函数是触摸事件和焦点事件的源头。
```
@Override
protected int onProcess(QueuedInputEvent q) {
    if (q.mEvent instanceof KeyEvent) {
        //按键事件（焦点事件）
        return processKeyEvent(q);
    } else {
        final int source = q.mEvent.getSource();
        if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
            //触摸事件
            return processPointerEvent(q);
        } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
            //轨迹事件，现在基本不用
            return processTrackballEvent(q);
        } else {
            //通用触摸事件
            return processGenericMotionEvent(q);
        }
    }
}
```
这里只关心`processKeyEvent()`函数对按键（焦点）事件做了什么处理。
```
private int processKeyEvent(QueuedInputEvent q) {
    final KeyEvent event = (KeyEvent)q.mEvent;

    if (mUnhandledKeyManager.preViewDispatch(event)) {
        return FINISH_HANDLED;
    }

    //在View层次结构分发按键事件
    //如果View树中有View消费事件dispatchKeyEvent（）函数返回true，
    后续步骤不再执行
    if (mView.dispatchKeyEvent(event)) {
        return FINISH_HANDLED;
    }
    //一些保护措施
    //在View层次结构不消费事件，判断窗口是否有输入事件或者已经停止和销毁
    if (shouldDropInputEvent(q)) {
         return FINISH_NOT_HANDLED;
    }


    if (mUnhandledKeyManager.dispatch(mView, event)) {
        return FINISH_HANDLED;
    }
    //用来保存焦点事件方向
    int groupNavigationDirection = 0;
    //处理tab键，判断焦点的方向
    if (event.getAction() == KeyEvent.ACTION_DOWN
            && event.getKeyCode() == KeyEvent.KEYCODE_TAB) {
        //metaStateHasModifiers()根据指定的meta状态按下特定时返回true
         //如果按的是组合键则返回false
        if (KeyEvent.metaStateHasModifiers(event.getMetaState(), KeyEvent.META_META_ON)) {
            groupNavigationDirection = View.FOCUS_FORWARD;
        } else if (KeyEvent.metaStateHasModifiers(event.getMetaState(),
                KeyEvent.META_META_ON | KeyEvent.META_SHIFT_ON)) {
            groupNavigationDirection = View.FOCUS_BACKWARD;
        }
    }

    // If a modifier is held, try to interpret the key as a shortcut.
    if (event.getAction() == KeyEvent.ACTION_DOWN
            && !KeyEvent.metaStateHasNoModifiers(event.getMetaState())
            && event.getRepeatCount() == 0
            && !KeyEvent.isModifierKey(event.getKeyCode())
            && groupNavigationDirection == 0) {
        if (mView.dispatchKeyShortcutEvent(event)) {
            return FINISH_HANDLED;
        }
        if (shouldDropInputEvent(q)) {
            return FINISH_NOT_HANDLED;
        }
    }

    // 应用 fallback 策略
    // 具体实现见PhoneFallbackEventHandler中dispatchKeyEvent()方法
    // 主要是对媒体键，音量键，通话键等做处理
    if (mFallbackEventHandler.dispatchKeyEvent(event)) {
        return FINISH_HANDLED;
    }
    if (shouldDropInputEvent(q)) {
        return FINISH_NOT_HANDLED;
    }

    //View层次结构不处理KeyEvent,那么变成了寻找焦点的过程
    if (event.getAction() == KeyEvent.ACTION_DOWN) {
        if (groupNavigationDirection != 0) {
            if (performKeyboardGroupNavigation(groupNavigationDirection)) {
                return FINISH_HANDLED;
            }
        } else {
            if (performFocusNavigation(event)) {
                return FINISH_HANDLED;
            }
        }
    }
     return FORWARD;
}
```
在`processKeyEvent()`函数中，我们主要关心按键事件在View层次结构的分发（`mView.dispatchKeyEvent(event)`）和函数最后自动处理焦点事件两大过程。

## KeyEvent在View层次结构的分发
在上一小节中，会调用ViewPostImeInputStage对象内`processKeyEvent()`函数，使KeyEvent在View的层次结构中分发。
其中`mView`是DecorView对象，并调用其`dispatchKeyEvent()`函数。
```
// DecorView 
@Override
public boolean dispatchKeyEvent(KeyEvent event) {
    final int keyCode = event.getKeyCode();
    final int action = event.getAction();
    final boolean isDown = action == KeyEvent.ACTION_DOWN;
    //第一次按下，处理panel快捷键
    if (isDown && (event.getRepeatCount() == 0)) {
        if ((mWindow.mPanelChordingKey > 0) && (mWindow.mPanelChordingKey != keyCode)) {
            boolean handled = dispatchKeyShortcutEvent(event);
            if (handled) {
                return true;
            }
        }

        if ((mWindow.mPreparedPanel != null) && mWindow.mPreparedPanel.isOpen) {
            if (mWindow.performPanelShortcut(mWindow.mPreparedPanel, keyCode, event, 0)) {
                return true;
            }
        }
    }
    //重点分析的地方
    //当Window未destoryed且callback非null,
    //交给Window对象的callback处理
    if (!mWindow.isDestroyed()) {
        final Window.Callback cb = mWindow.getCallback();
        // mFeatureId < 0 表示是application的DecorView，比如Activity、Dialog
        final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)//事件派发给callback对象
                : super.dispatchKeyEvent(event);//派发给ViewGroup（View层次结构）
        if (handled) {
            return true;
        }
    }
    //只有View不消费事件，才将事件交给Window对象
    return isDown ? mWindow.onKeyDown(mFeatureId, event.getKeyCode(), event)
            : mWindow.onKeyUp(mFeatureId, event.getKeyCode(), event);
}
```
我们知道Activity和Dialog都是实现了CallBack接口，因此这里先分析KeyEvent在Activity中分发，再看看View层次结构分发。因为PhoneWindow是Window的唯一实现类，所以最后需要看看PhoneWindow的`onKeyDown()`和`onKeyUp()`函数。


```
//Activity.java

public boolean dispatchKeyEvent(KeyEvent event) {
    onUserInteraction();

    final int keyCode = event.getKeyCode();
    //处理KEYCODE_MENU键，Activity有ActionBar且消费该按键
    if (keyCode == KeyEvent.KEYCODE_MENU &&
            mActionBar != null && mActionBar.onMenuKeyEvent(event)) {
        return true;
    }

    Window win = getWindow();
    //这里KeyEvent交给Window对象分发
    if (win.superDispatchKeyEvent(event)) {
        return true;
    }
    View decor = mDecor;
    if (decor == null) decor = win.getDecorView();
    
    //Window对象win不消费Key事件,则将事件交给KeyEvent自身
    return event.dispatch(this, decor != null
            ? decor.getKeyDispatcherState() : null, this);
}
```
通过后面的学习，可以KeyEvent在Window对象的分发其实是在View的层次结构分发。在View层次结构中不消费KeyEvent事件，那么会交给KeyEvent自身处理，会调用Activity相关方法，如`onKeyDown()`。

我们知道PhoneWindow是Window的具体实现，所以看看PhoneWindow对象的`superDispatchKeyEvent()`函数。
```
//Window

/**
 * Used by custom windows, such as Dialog, to pass the key press event
 * further down the view hierarchy. Application developers should
 * not need to implement or call this.
 *
 */
public abstract boolean superDispatchKeyEvent(KeyEvent event);

//PhoneWindow

@Override
public boolean superDispatchKeyEvent(KeyEvent event) {
    return mDecor.superDispatchKeyEvent(event);
}
```
可以看到，又回到DecorView对象的`superDispatchKeyEvent()`函数。
```
// DecorView

public boolean superDispatchKeyEvent(KeyEvent event) {
    //优先处理KEYCODE_BACK的事件
    if (event.getKeyCode() == KeyEvent.KEYCODE_BACK) {
        final int action = event.getAction();
        // Back cancels action modes first.
        if (mPrimaryActionMode != null) {
            if (action == KeyEvent.ACTION_UP) {
                mPrimaryActionMode.finish();
            }
            return true;
        }
    }
    //在View层次结构进行派发
    if (super.dispatchKeyEvent(event)) {
        return true;
    }
    
    //如果View层次结构不消费，且ViewRootImpl不为空，
    //则在ViewRootImpl对象处理
    return (getViewRootImpl() != null) && getViewRootImpl().dispatchUnhandledKeyEvent(event);
}
```
View层次结构大概如下图所示，这里先看看KeyEvent在View层次结构的分发。

![](https://user-gold-cdn.xitu.io/2019/8/22/16cb71ac53a6725e?w=526&h=385&f=png&s=11954)
我们知道，DecorView继承至FrameLayout,而FrameLayout是ViewGroup的子类。
ViewGroup类的`dispatchKeyEvent()`函数如下：
```
//ViewGroup

 @Override
public boolean dispatchKeyEvent(KeyEvent event) {
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onKeyEvent(event, 1);
    }
    //ViewGroup是focused或者设置了具体的大小，则交给它实现
    if ((mPrivateFlags & (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS))
            == (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS)) {
        //在View中的实现
        if (super.dispatchKeyEvent(event)) {
            return true;
        }
    } else if (mFocused != null && (mFocused.mPrivateFlags & PFLAG_HAS_BOUNDS)
            == PFLAG_HAS_BOUNDS) {
        //mFocused表示当前ViewGroup中获得焦点或者包含焦点的View（子View）
        if (mFocused.dispatchKeyEvent(event)) {
            return true;
        }
    }

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 1);
    }
    return false;
}
```
从上面代码可以看出，子View想要获得焦点,处理KeyEvent，需要设置focusable属性为true。KeyEvent会优先派发给符合条件的ViewGroup处理，而不是子View。`mFocused.dispatchKeyEvent(event)`中的`dispatchKeyEvent()`可能会迭代调用，因为子View也可能是ViewGroup。

这里看看View中事件分发。
```
//View

    /**
     * Dispatch a key event to the next view on the focus path. This path runs
     * from the top of the view tree down to the currently focused view. If this
     * view has focus, it will dispatch to itself. Otherwise it will dispatch
     * the next node down the focus path. This method also fires any key
     * listeners.
     *
     * @param event The key event to be dispatched.
     * @return True if the event was handled, false otherwise.
     */
    public boolean dispatchKeyEvent(KeyEvent event) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onKeyEvent(event, 0);
        }

        // Give any attached key listener a first crack at the event.
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        //如果我们给View设置了OnKeyListener且View是ENABLED状态,
        //则会回调我们的了OnKeyListener
        if (li != null && li.mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnKeyListener.onKey(this, event.getKeyCode(), event)) {
            return true;
        }
        //调用KeyEvent.dispatch方法，并将view对象本身作为参数传递进去，view的各种callback方法在这里被触发
        if (event.dispatch(this, mAttachInfo != null
                ? mAttachInfo.mKeyDispatchState : null, this)) {
            return true;
        }

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }
        return false;
    }
```
ViewGroup和View的`dispatchKeyEvent()`就构成了View层次结构的KeyEvent分发，且总是从树根DecorView开始到具体的View。注意到此处在View不消费KeyEvent会调用KeyEvent.dispatch方法，在Activity也会调用该方法。
```
//KeyEvent
public final boolean dispatch(Callback receiver, DispatcherState state,
        Object target) {
    switch (mAction) {
        case ACTION_DOWN: {
            mFlags &= ~FLAG_START_TRACKING;
            //回调Callback对象receiver的onKeyDown函数，上文知道Activity和View都实现Callback
            boolean res = receiver.onKeyDown(mKeyCode, this);
            if (state != null) {//通常成立
                if (res && mRepeatCount == 0 && (mFlags&FLAG_START_TRACKING) != 0) {//判断是否轨迹事件
                    state.startTracking(this, target);
                } else if (isLongPress() && state.isTracking(this)) {
                    try {
                        //处理长按事件
                        if (receiver.onKeyLongPress(mKeyCode, this)) {
                            state.performedLongPress(this);
                            res = true;//消费该事件
                        }
                    } catch (AbstractMethodError e) {
                    }
                }
            }
            return res;
        }
        case ACTION_UP:
            if (state != null) {
                //reset state的内部状态，也改变了KeyEvent的某些状态
                state.handleUpEvent(this);
            }
            //回调Callback对象receiver的onKeyUp函数
            return receiver.onKeyUp(mKeyCode, this);
        case ACTION_MULTIPLE:
            final int count = mRepeatCount;
            final int code = mKeyCode;
            if (receiver.onKeyMultiple(code, count, this)) {
                return true;
            }
            if (code != KeyEvent.KEYCODE_UNKNOWN) {
                mAction = ACTION_DOWN;
                mRepeatCount = 0;
                boolean handled = receiver.onKeyDown(code, this);
                if (handled) {
                    mAction = ACTION_UP;
                    receiver.onKeyUp(code, this);
                }
                mAction = ACTION_MULTIPLE;
                mRepeatCount = count;
                return handled;
            }
            return false;
    }
    return false;
}
```
在上面代码中，可以看到Callback对象的`onKeyDown()`,`onKeyUp()`,`onKeyLongPress()`函数被回调。而Activity和View是Callback接口的实现，因此调用Activity和View对应的方法。

先看看Activity对几个方法的实现：
```
//Activity

public boolean onKeyDown(int keyCode, KeyEvent event)  {
    //处理返回键
    if (keyCode == KeyEvent.KEYCODE_BACK) {
        if (getApplicationInfo().targetSdkVersion
                >= Build.VERSION_CODES.ECLAIR) {
            // 标记追踪这个key event    
            event.startTracking();
            } else {
            //手机APP常在Activity重写该方法，
            //要求用户双击两次来退出APP,而不是一次就退出APP
            onBackPressed();
        }
        return true;
    }

    if (mDefaultKeyMode == DEFAULT_KEYS_DISABLE) {
        return false;
    } else if (mDefaultKeyMode == DEFAULT_KEYS_SHORTCUT) {
        Window w = getWindow();
        if (w.hasFeature(Window.FEATURE_OPTIONS_PANEL) &&
                w.performPanelShortcut(Window.FEATURE_OPTIONS_PANEL, keyCode, event,
                        Menu.FLAG_ALWAYS_PERFORM_CLOSE)) {
            return true;
        }
        return false;
    } else if (keyCode == KeyEvent.KEYCODE_TAB) {
        return false;
    } else {
        boolean clearSpannable = false;
        boolean handled;
        if ((event.getRepeatCount() != 0) || event.isSystem()) {
            clearSpannable = true;
            handled = false;
        } else {
            handled = TextKeyListener.getInstance().onKeyDown(
                    null, mDefaultKeySsb, keyCode, event);
            if (handled && mDefaultKeySsb.length() > 0) {
                final String str = mDefaultKeySsb.toString();
                clearSpannable = true;
                switch (mDefaultKeyMode) {
                case DEFAULT_KEYS_DIALER:
                    Intent intent = new Intent(Intent.ACTION_DIAL,  Uri.parse("tel:" + str));
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    startActivity(intent);
                    break;
                case DEFAULT_KEYS_SEARCH_LOCAL:
                    startSearch(str, false, null, false);
                    break;
                case DEFAULT_KEYS_SEARCH_GLOBAL:
                    startSearch(str, false, null, true);
                    break;
                }
            }
        }
        if (clearSpannable) {
            mDefaultKeySsb.clear();
            mDefaultKeySsb.clearSpans();
            Selection.setSelection(mDefaultKeySsb,0);
        }
        return handled;
    }
}

public boolean onKeyLongPress(int keyCode, KeyEvent event) {
    return false;
}

public boolean onKeyUp(int keyCode, KeyEvent event) {
    if (getApplicationInfo().targetSdkVersion
            >= Build.VERSION_CODES.ECLAIR) {
        if (keyCode == KeyEvent.KEYCODE_BACK && event.isTracking()
                && !event.isCanceled()) {
            onBackPressed();
            return true;
        }
    }
    return false;
}
```
而View中实现：
```
//View
public boolean onKeyDown(int keyCode, KeyEvent event) {
   //KEYCODE_DPAD_CENTER、KEYCODE_ENTER、
   //KEYCODE_SPACE、KEYCODE_NUMPAD_ENTER都返回ture,其他返回false
    if (KeyEvent.isConfirmKey(keyCode)) {
        //当前View是DISABLED状态直接返回false
        if ((mViewFlags & ENABLED_MASK) == DISABLED) {
            return true;
        }

      if (event.getRepeatCount() == 0) {
            final boolean clickable = (mViewFlags & CLICKABLE) == CLICKABLE
                    || (mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE;
            if (clickable || (mViewFlags & TOOLTIP) == TOOLTIP) {
                final float x = getWidth() / 2f;
                final float y = getHeight() / 2f;
                if (clickable) {
                    //标记为Pressed，例如根据状态改变背景
                    setPressed(true, x, y);
                }
                //长按检测
                checkForLongClick(0, x, y);
                return true;
            }
        }
    }

    return false;
}

public boolean onKeyUp(int keyCode, KeyEvent event) {
    if (KeyEvent.isConfirmKey(keyCode)) {
        if ((mViewFlags & ENABLED_MASK) == DISABLED) {
            return true;
        }
        if ((mViewFlags & CLICKABLE) == CLICKABLE && isPressed()) {
            setPressed(false);

            if (!mHasPerformedLongPress) {
                removeLongPressCallback();
                if (!event.isCanceled()) {
                    return performClickInternal();
                }
            }
        }
    }
    return false;
}
```
在DecorView的`superDispatchKeyEvent()`函数最后一行，如果View层次结构不消费事件，那么会调用ViewRootImpl对象的`dispatchUnhandledKeyEvent()`函数，这里主要是将未被消费的KeyEvent分发给注册了处理所有KeyEvent的处理者（监听器）
```
//ViewRootImpl.java

private final UnhandledKeyManager mUnhandledKeyManager
                     =new UnhandledKeyManager();

public boolean dispatchUnhandledKeyEvent(KeyEvent event) {
    return mUnhandledKeyManager.dispatch(mView, event);
}

boolean dispatch(View root, KeyEvent event) {
        if (mDispatched) {
            return false;
        }
        View consumer;
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "UnhandledKeyEvent dispatch");
            mDispatched = true;
            //将未处理的KeyEvent进行分发,如果有View消费该事件，则返回该
            //View
            consumer = root.dispatchUnhandledKeyEvent(event);
            //用于追踪该事件
            if (event.getAction() == KeyEvent.ACTION_DOWN) {
                    int keycode = event.getKeyCode();
                    if (consumer != null && !KeyEvent.isModifierKey(keycode)) {
                        mCapturedKeys.put(keycode, new WeakReference<>(consumer));
                    }
                }
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
            return consumer != null;
        }
```
这里的root是DecorView,但从参数的类型来说，这里`root.dispatchUnhandledKeyEvent(event)`应该是在View中实现的。
```
//View.java

View dispatchUnhandledKeyEvent(KeyEvent evt) {
    if (onUnhandledKeyEvent(evt)) {
        return this;
    }
    return null;
}

boolean onUnhandledKeyEvent(@NonNull KeyEvent event) {
    if (mListenerInfo != null && mListenerInfo.mUnhandledKeyListeners != null) {
        //mListenerInfo通过栈的方式保存是否有View设置UnhandledKeyListener，如果有且消费事件，则该DecorView消费该事件
        for (int i = mListenerInfo.mUnhandledKeyListeners.size() - 1; i >= 0; --i) {
            if (mListenerInfo.mUnhandledKeyListeners.get(i).onUnhandledKeyEvent(this, event)) {
                return true;
            }
        }
    }
    return false;
}
```
通过上文代码可以知道，我们可以在View中添加`addOnUnhandledKeyEventListener(OnUnhandledKeyEventListener)`来监听所有未被处理的KeyEvent。会在KeyEvent正常分发未被消费前，且早于Window的Callback被回调。

在DecorView的dispatchKeyEvent()函数，如果View层次结构不消费事件，那么会交给PhoneWindow的`onKeyDown()`或`onKeyUp()`函数。
```
//PhoneWindow
protected boolean onKeyDown(int featureId, int keyCode, KeyEvent event) {

    final KeyEvent.DispatcherState dispatcher =
            mDecor != null ? mDecor.getKeyDispatcherState() : null;
 Integer.toHexString(event.getFlags()));

    switch (keyCode) {
        case KeyEvent.KEYCODE_VOLUME_UP:
        case KeyEvent.KEYCODE_VOLUME_DOWN:
        case KeyEvent.KEYCODE_VOLUME_MUTE: {
            if (mMediaController != null) {
                mMediaController.dispatchVolumeButtonEventAsSystemService(event);
            } else {
                getMediaSessionManager().dispatchVolumeKeyEventAsSystemService(event,
                        mVolumeControlStreamType);
            }
            return true;
        }

        case KeyEvent.KEYCODE_MEDIA_PLAY:
        case KeyEvent.KEYCODE_MEDIA_PAUSE:
        case KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE:
        case KeyEvent.KEYCODE_MUTE:
        case KeyEvent.KEYCODE_HEADSETHOOK:
        case KeyEvent.KEYCODE_MEDIA_STOP:
        case KeyEvent.KEYCODE_MEDIA_NEXT:
        case KeyEvent.KEYCODE_MEDIA_PREVIOUS:
        case KeyEvent.KEYCODE_MEDIA_REWIND:
        case KeyEvent.KEYCODE_MEDIA_RECORD:
        case KeyEvent.KEYCODE_MEDIA_FAST_FORWARD: {
            if (mMediaController != null) {
                if (mMediaController.dispatchMediaButtonEventAsSystemService(event)) {
                    return true;
                }
            }
            return false;
        }

        case KeyEvent.KEYCODE_MENU: {
            onKeyDownPanel((featureId < 0) ? FEATURE_OPTIONS_PANEL : featureId, event);
            return true;
        }

        case KeyEvent.KEYCODE_BACK: {
            if (event.getRepeatCount() > 0) break;
            if (featureId < 0) break;
            if (dispatcher != null) {
                dispatcher.startTracking(event, this);
            }
            return true;
        }

    }

    return false;
}

protected boolean onKeyUp(int featureId, int keyCode, KeyEvent event) {
    final KeyEvent.DispatcherState dispatcher =
            mDecor != null ? mDecor.getKeyDispatcherState() : null;
    if (dispatcher != null) {
        dispatcher.handleUpEvent(event);
    }

    switch (keyCode) {
        case KeyEvent.KEYCODE_VOLUME_UP:
        case KeyEvent.KEYCODE_VOLUME_DOWN: {
            if (mMediaController != null) {
                mMediaController.dispatchVolumeButtonEventAsSystemService(event);
            } else {
                getMediaSessionManager().dispatchVolumeKeyEventAsSystemService(
                        event, mVolumeControlStreamType);
            }
            return true;
        }
        case KeyEvent.KEYCODE_VOLUME_MUTE: {
            getMediaSessionManager().dispatchVolumeKeyEventAsSystemService(
                    event, AudioManager.USE_DEFAULT_STREAM_TYPE);
            return true;
        }

        case KeyEvent.KEYCODE_MEDIA_PLAY:
        case KeyEvent.KEYCODE_MEDIA_PAUSE:
        case KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE:
        case KeyEvent.KEYCODE_MUTE:
        case KeyEvent.KEYCODE_HEADSETHOOK:
        case KeyEvent.KEYCODE_MEDIA_STOP:
        case KeyEvent.KEYCODE_MEDIA_NEXT:
        case KeyEvent.KEYCODE_MEDIA_PREVIOUS:
        case KeyEvent.KEYCODE_MEDIA_REWIND:
        case KeyEvent.KEYCODE_MEDIA_RECORD:
        case KeyEvent.KEYCODE_MEDIA_FAST_FORWARD: {
            if (mMediaController != null) {
                if (mMediaController.dispatchMediaButtonEventAsSystemService(event)) {
                        return true;
                    }
                }
                return false;
            }

        case KeyEvent.KEYCODE_MENU: {
            onKeyUpPanel(featureId < 0 ? FEATURE_OPTIONS_PANEL : featureId,
                    event);
            return true;
        }

        case KeyEvent.KEYCODE_BACK: {
            if (featureId < 0) break;
            if (event.isTracking() && !event.isCanceled()) {
                if (featureId == FEATURE_OPTIONS_PANEL) {
                    PanelFeatureState st = getPanelState(featureId, false);
                    if (st != null && st.isInExpandedMode) {
                        reopenMenu(true);
                        return true;
                    }
                }
                closePanel(featureId);
                return true;
            }
            break;
        }

        case KeyEvent.KEYCODE_SEARCH: {                /*

            if (isNotInstantAppAndKeyguardRestricted()) {
                break;
            }
            if ((getContext().getResources().getConfiguration().uiMode
                    & Configuration.UI_MODE_TYPE_MASK) == Configuration.UI_MODE_TYPE_WATCH) {
                break;
            }
            if (event.isTracking() && !event.isCanceled()) {
                launchDefaultSearch(event);
            }
            return true;
        }

        case KeyEvent.KEYCODE_WINDOW: {
            if (mSupportsPictureInPicture && !event.isCanceled()) {
                getWindowControllerCallback().enterPictureInPictureModeIfPossible();
            }
            return true;
        }
    }

    return false;
}
```
从上面可以看到，PhoneWindow也只对一些物理按键做了处理，如果PhoneWindow和View、Activity都没有消费事件，那么ViewPostImeInputStage对象通过系统算法自动寻找焦点了。

#### 总结一下：
1. 监听器的优先级高于Callback的回调，也就是说OnKeyListener的函数优先Callback的onKeyDown等等函数的回调。
2. View的Callback回调要早于Activity，Activity的回调早于PhoneWindow。
3. 优先级高的消费KeyEvent,优先级低的不再受理该事件。

下面通过时序图对上文KeyEvent做一个整体流程的阐释（虽然不能准备表达意思）
![](https://user-gold-cdn.xitu.io/2019/8/26/16cce5e520e1a1eb?w=1055&h=924&f=png&s=33305)
在图表示在KeyEvent在DecorView开始不拦截最终在View的OnKeyListener或Callback对象被消费的过程。

[参考文章](https://www.cnblogs.com/xiaoweiz/p/3803301.html)
## 系统自动寻找焦点
回到梦开始的地方ViewPostImeInputStage对象的`processKeyEvent()`函数的末尾，在自动处理焦点的地方，会调用`performFocusNavigation()`函数来寻找下个获得焦点的View。
```
//ViewRootImpl.java

private boolean performFocusNavigation(KeyEvent event) {

    int direction = 0;
    //从下面代码可以看出，switch语句在此的主要作用是判断焦点的方向
    switch (event.getKeyCode()) {
        case KeyEvent.KEYCODE_DPAD_LEFT:
            if (event.hasNoModifiers()) {
                direction = View.FOCUS_LEFT;
                }
            break;
        case KeyEvent.KEYCODE_DPAD_RIGHT:
            if (event.hasNoModifiers()) {
                direction = View.FOCUS_RIGHT;
            }
            break;
        case KeyEvent.KEYCODE_DPAD_UP:
            if (event.hasNoModifiers()) {
                direction = View.FOCUS_UP;
            }
            break;
        case KeyEvent.KEYCODE_DPAD_DOWN:
            if (event.hasNoModifiers()) {
                direction = View.FOCUS_DOWN;
            }
            break;
        case KeyEvent.KEYCODE_TAB:
            if (event.hasNoModifiers()) {
                direction = View.FOCUS_FORWARD;
            } else if (event.hasModifiers(KeyEvent.META_SHIFT_ON)) {
                direction = View.FOCUS_BACKWARD;
            }
            break;
    }
    
    if (direction != 0) {
        //分析一
        //mView在这里是DecorView对象，查找出当前获得焦点的View
        View focused = mView.findFocus();
        if (focused != null) {
            //分析二
            //在当前获得焦点View通过指定方向搜索下一个获得焦点的View
            View v = focused.focusSearch(direction);
            if (v != null && v != focused) {
                focused.getFocusedRect(mTempRect);
                if (mView instanceof ViewGroup) {
                    ((ViewGroup) mView).offsetDescendantRectToMyCoords(
                            focused, mTempRect);
                    ((ViewGroup) mView).offsetRectIntoDescendantCoords(
                            v, mTempRect);
                }
                if (v.requestFocus(direction, mTempRect)) {
                    playSoundEffect(SoundEffectConstants
                            .getContantForFocusDirection(direction));
                    return true;
                }
            }

            //给DecorView最后一次处理焦点的机会
            if (mView.dispatchUnhandledMove(focused, direction)) {
                return true;
            }
        } else {
            //如果没有View获得焦点
            if (mView.restoreDefaultFocus()) {
                return true;
            }
        }
    }
    return false;
}
```
在分析一处，通过DecorView对象`mView`来查找已获得焦点的View，`findFocus()`函数在ViewGroup和View都有实现，而DecorView继承自ViewGroup。这里其实进入了View的层次结果查找已获得焦点的View.
```
//ViewGroup.java
@Override
public View findFocus() {
    
    if (isFocused()) {
        return this;
    }

    if (mFocused != null) {
        //mFocused表示获得焦点的View,有可能是ViewGroup,也有可能是具体的View
        return mFocused.findFocus();
    }
    return null;
}

//View.java
public View findFocus() {
    return (mPrivateFlags & PFLAG_FOCUSED) != 0 ? this : null;
}

public boolean isFocused() {
    return (mPrivateFlags & PFLAG_FOCUSED) != 0;
}
```
对于ViewGroup来说，如果本身获得焦点则直接返回自身即可，否则继续通过`mFocused.findFocus()`函数继续查找已获得焦点的View。具体的View是否获得焦点与ViewGroup的判断条件是一致的，判断`PFLAG_FOCUSED`标志位，也就是我们调用View的focuesabe=true，如果设置，则返回该View，否则返回null，表示没有View获得焦点。

假设寻找到了已获得焦点的View，那么下面就是寻找下一个获得焦点的View。也就是ViewPostImeInputStage对象的`performFocusNavigation()`函数分析二的代码。由于`focused`对象有可能是ViewGroup，也有可能是具体的View。一起看看它两的实现。
```
//View
//具体View的实现非常的简单，如果有父视图ViewGroup，则在俯视图在寻找，
//否则返回null
public View focusSearch(@FocusRealDirection int direction) {
    if (mParent != null) {
        return mParent.focusSearch(this, direction);
    } else {
        return null;
    }
}
//ViewGroup.java
@Override
public View focusSearch(View focused, int direction) {
    if (isRootNamespace()) { //判断当前ViewGroup是否顶层View，即DecorView
        //调用FocusFinder实例findNextFocus进行查找
        return FocusFinder.getInstance().findNextFocus(this, focused, direction);
    } else if (mParent != null) {
        //递归到顶层View
        return mParent.focusSearch(focused, direction);
    }
    return null;
}
```
通过递归方式，从内到外，到布局最外层View，准确说是ViewGroup。然后调用FocusFinder的实例方法findNextFocus()来寻找下一个获得焦点的View。
FocusFinder类通过算法，根据当前获得焦点的View和按键方向来寻找下一个获得焦点的View。
```
//FocusFinder.java
public final View findNextFocus(ViewGroup root, View focused, int direction) {
    return findNextFocus(root, focused, null, direction);
}

private View findNextFocus(ViewGroup root, View focused, Rect focusedRect, int direction) {
    View next = null;
    ViewGroup effectiveRoot = getEffectiveRoot(root, focused);
    if (focused != null) {
        //寻找当前获得焦点view是否设置了指定方向的下一个获得焦点的View
        next = findNextUserSpecifiedFocus(effectiveRoot, focused, direction);
    }
    if (next != null) {
        return next;
    }
    ArrayList<View> focusables = mTempList;
    try {
        focusables.clear();
        effectiveRoot.addFocusables(focusables, direction);
        if (!focusables.isEmpty()) {
            //通过算法寻找最近可获得焦点的View
            next = findNextFocus(effectiveRoot, focused, focusedRect, direction, focusables);
        }
    } finally {
        focusables.clear();
    }
    return next;
}
```
`findNextFocus()`函数主要通过两种方式来寻找指定方向的下一个获得焦。的View：我们在XML布局或者代码设置特定方向获得焦点的View;通过算法来寻找特定方向可以获得焦点最近的View。

对于方式一，`findNextUserSpecifiedFocus()`函数的实现如下：
```
//View.java
//主要作用是判断当前获得焦点的View有没有指定下一个获得焦点View的ID
View findUserSetNextFocus(View root, @FocusDirection int direction) {
    switch (direction) {
        case FOCUS_LEFT:
            if (mNextFocusLeftId == View.NO_ID) return null;
        return findViewInsideOutShouldExist(root, mNextFocusLeftId);
        case FOCUS_RIGHT:
            if (mNextFocusRightId == View.NO_ID) return null;
            return findViewInsideOutShouldExist(root, mNextFocusRightId);
        case FOCUS_UP:
            if (mNextFocusUpId == View.NO_ID) return null;
           return findViewInsideOutShouldExist(root, mNextFocusUpId);
        case FOCUS_DOWN:
            if (mNextFocusDownId == View.NO_ID) return null;
            return findViewInsideOutShouldExist(root, mNextFocusDownId);
        case FOCUS_FORWARD:
            if (mNextFocusForwardId == View.NO_ID) return null;
            return findViewInsideOutShouldExist(root, mNextFocusForwardId);
        case FOCUS_BACKWARD: {
            if (mID == View.NO_ID) return null;
            final int id = mID;
            return root.findViewByPredicateInsideOut(this, new Predicate<View>() {
                @Override
                public boolean test(View t) {
                    return t.mNextFocusForwardId == id;
                }
            });
        }
    }
    return null;
}
//MatchIdPredicate匹配到相同ID会返回true
private View findViewInsideOutShouldExist(View root, int id) {
    if (mMatchIdPredicate == null) {
        mMatchIdPredicate = new MatchIdPredicate();
    }
    mMatchIdPredicate.mId = id;
    View result = root.findViewByPredicateInsideOut(this, mMatchIdPredicate);
    if (result == null) {
        Log.w(VIEW_LOG_TAG, "couldn't find view with id " + id);
    }
    return result;
}
//从当前View开始，遍历View的层次结构来的下一个获得焦点的View
public final <T extends View> T findViewByPredicateInsideOut(
    View start, Predicate<View> predicate) {
    View childToSkip = null;
    for (;;) {
        T view = start.findViewByPredicateTraversal(predicate, childToSkip);
        if (view != null || start == this) {
            return view;
        }

        ViewParent parent = start.getParent();
        if (parent == null || !(parent instanceof View)) {
            return null;
        }

        childToSkip = start;
        start = (View) parent;
    }
}
//匹配相同ID，返回该View
protected <T extends View> T findViewByPredicateTraversal(Predicate<View> predicate,
        View childToSkip) {
    if (predicate.test(this)) {
        return (T) this;
    }
    return null;
}
```
在上面的相关方法，主要是通过在View的层次结构中去寻找到和指定id匹配的View。

那么方式二，通过算法来寻找下一个焦点又是如何的呢？
```
//FocusFinder.java
private View findNextFocus(ViewGroup root, View focused, Rect focusedRect,
    int direction, ArrayList<View> focusables) {
    if (focused != null) {
        if (focusedRect == null) {
            focusedRect = mFocusedRect;
        }
        // fill in interesting rect from focused
        focused.getFocusedRect(focusedRect);
        root.offsetDescendantRectToMyCoords(focused, focusedRect);
    } else {
    if (focusedRect == null) {
        focusedRect = mFocusedRect;
        // make up a rect at top left or bottom right of root
        switch (direction) {
            case View.FOCUS_RIGHT:
            case View.FOCUS_DOWN:
                setFocusTopLeft(root, focusedRect);
                break;
            case View.FOCUS_FORWARD:
                if (root.isLayoutRtl()) {
                    setFocusBottomRight(root, focusedRect);
                } else {
                    setFocusTopLeft(root, focusedRect);
                }
                break;

            case View.FOCUS_LEFT:
            case View.FOCUS_UP:
                setFocusBottomRight(root, focusedRect);
                break;
            case View.FOCUS_BACKWARD:
                if (root.isLayoutRtl()) {
                    setFocusTopLeft(root, focusedRect);
                } else {
                    setFocusBottomRight(root, focusedRect);
                break;
            }
        }
    }
}
```
对FocusFinder就不作进一步分析了。感兴趣同学可以自己看看源码。

看到这里，估计都很累了，我们想了解也都知道了。
## 总结
所有的KeyEvent都会优先在View的层次结构分发，然后再通过自动寻找焦点来查找下一个获得焦点的View。这就是为什么在OnKeyListener或Callback相关回调方法返回true消费KeyEvent,下一个View无法获得焦点。

KeyEvent在View的层次结构分发总是从外到里，外层ViewGroup消费KeyEvent,内层的View是无法获得焦点的。这就是为什么我们不想要EditText弹出软键盘，在根布局设置focusable为true的原因。

OnKeyListener的调用要早于KeyEvent.Callback的调用，如果设置了OnKeyListener并消费了KeyEvent,那么Callback相关函数不会再被调用。

另外，想要在KeyEvent分发前处理KeyEvent,例如TV开发处理特殊的按键，可以修改`PhoneWindowManager`的`interceptKeyBeforeDispatching()`函数。

**最后的最后，能回答开头的问题么？**
