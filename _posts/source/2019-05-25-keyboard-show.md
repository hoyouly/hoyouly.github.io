---
layout: post
title: 源码分析 - Andrid 输入法框架 之 键盘启动的流程
category: 源码分析
tags: IMF
---
<!-- * content -->
<!-- {:toc} -->

点击 EditText ， EditText 获得焦点，然后键盘显示，这是最常见的操作，可是里面的内部实现是怎么样的呢？带着这个疑问，我们来看看系统源码。   
我们知道， EditText 是获得了点击事件，才能显示键盘，如果 EditText 连点击事件都没收到，肯定不会显示键盘，所以， EditText 肯定会执行到 onTouchEvent() 方法。
由于 EditText 继承 Textview ，并且本身没有覆盖 onTouchEvent() ,所以我们需要查看 TextView 中的onTouchEvent()
``` java
//TextView.java
public boolean onTouchEvent(MotionEvent event) {
  ...
  final boolean superResult = super.onTouchEvent(event);
  ...
  // 显示 IME ，除非选择只读文本。
  final InputMethodManager imm = InputMethodManager.peekInstance();
  viewClicked(imm);
    //这个是真正显示输入法的调用
  if (!textIsSelectable && mEditor.mShowSoftInputOnFocus) {
    handled |= imm != null && imm.showSoftInput(this, 0);
  }
  ...
}
```
在 TextView 的 onTouchEvent() 中，我们发现了两点比较关键信息
1. 调用了父类的 onTouchEvent() 方法。
2. 找到了显示输入法地方 <font color="#ff000" >imm.showSoftInput()。</font>

首先我们查看父类的 onTouchEvent() 都做了什么
## View # onTouchEvent()
我们知道， onTouchEvent() 的返回值表示是否消耗该事件， true 表示消耗， false 表示不消耗。    
```java
public boolean onTouchEvent(MotionEvent event) {
  ...
if ((viewFlags & ENABLED_MASK) == DISABLED) {//View 处于不可点击状态下
    if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
      setPressed(false);
    }
    //处于不可点击状态下也是可以消耗事件的，只不过会会响应而已
    return (((viewFlags & CLICKABLE) == CLICKABLE || (viewFlags & LONG_CLICKABLE) ==LONG_CLICKABLE));
  }

if (mTouchDelegate != null) {//如果设置的 TouchDelegate ，则会执行 TouchDelegate 的 onTouchEvent() ，
      //onTouchEvent()的机制和 onTouchListener 类似
    if (mTouchDelegate.onTouchEvent(event)) {
      return true;
    }
}

if (((viewFlags & CLICKABLE) == CLICKABLE || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
     // 只要设置了 CLICKABLE 或者 LONG_CLICKABLE 任意一个为 true ，就会消耗掉事件
    switch (event.getAction()) {
      case MotionEvent.ACTION_UP:
        boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
        if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
          // take focus if we don't have it already and we should in
          // touch mode.
          boolean focusTaken = false;
          //让 view 获得焦点
          if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
            focusTaken = requestFocus();
          }
          ...
       }
    }
  }
}
```

代码里面带有注释，就不多解释了。只需要记得。在这里面通过 requestFocus()使得View获得了焦点。    
执行完View.onTouchEvent()后，就到了关键的得到InputMethodManager对象imm上了。

## 得到IMM对象
先看看 InputMethodManager imm = InputMethodManager.peekInstance()是怎么实现的吧。
```java
//InputMethodManager.java
public static InputMethodManager peekInstance() {
  return sInstance;
}
public static InputMethodManager getInstance() {
  synchronized (InputMethodManager.class) {
    if (sInstance == null) {
      // InputMethodManager其实就是一个 Binder service 的proxy
      IBinder b = ServiceManager.getService(Context.INPUT_METHOD_SERVICE);
      //这个 service 其实就是IMMS
      IInputMethodManager service = IInputMethodManager.Stub.asInterface(b);
      sInstance = new InputMethodManager(service, Looper.getMainLooper());
    }
    return sInstance;
  }
}
```
peekInstance()就是得到 IMM实例，而这个实例就是在 getInstance()中初始化的，关于IMM的初始化，在前面已经讲过了，详情可以查看[源码分析 - Andrid 输入法框架 之 IMM初始化](../../../../2019/05/20/IMF-start/#imm-初始化)   

InputMethodManager.getInstance()又是在哪里执行的呢，经过我们的搜索，发现在WindowManagerGloabl.java 中执行了。
```java
//WindowManagerGloabl.java
public static IWindowSession getWindowSession() {
  synchronized (WindowManagerGlobal.class) {
    if (sWindowSession == null) {
      try {
        //创建 InputMethodManager 实例
        InputMethodManager imm = InputMethodManager.getInstance();
        IWindowManager windowManager = getWindowManagerService();
        sWindowSession = windowManager.openSession(new IWindowSessionCallback.Stub() {
          @Override
          public void onAnimatorScaleChanged(float scale) {
            ValueAnimator.setDurationScale(scale);
          }
        }, imm.getClient(), imm.getInputContext());
      } catch (RemoteException e) {
        Log.e(TAG, "Failed to open window session", e);
      }
    }
    return sWindowSession;
  }
}
```
而getWindowSession()是在 ViewRootImp 初始化的时候执行的

```java
//ViewRootImpl.java
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    mWindowSession = WindowManagerGlobal.getWindowSession();
    ...
}
```

我们知道， 每一个 Window 对应一个 View 和 ViewRootImpl ， Window 和 View 通过 ViewRootImpl 建立联系 ，所以当我们在 addView() 的时候，就会创建一个ViewRootImpl，那么在这个时候。 InputMethodManager.getInstance()就会执行返回一个IMM对象sInstance，那么在我们点击EditText的时候，InputMethodManager.peekInstance() 就肯定不会为null，所以就可以执行到了imm. showSoftInput()中了。 那我们就看 showSoftInput() 的逻辑吧。

## IMM # showSoftInput()
```java
// IMM.java
public boolean showSoftInput(View view, int flags) {
    return showSoftInput(view, flags , null);
}
public boolean showSoftInput(View view, int flags , ResultReceiver resultReceiver) {
  checkFocus();
  synchronized (mH) {
    if (mServedView != view && (mServedView == null ||!mServedView.checkInputConnectionProxy(view))) {
      return false;
    }
    try {
      return mService.showSoftInput(mClient, flags , resultReceiver);
    } catch (RemoteException e) {
    }
    return false;
  }
}
```
看到了一个变量 mService ，在IMM初始化的时候，我们已经说过这个家伙了，是一个 IInputMethodManager 对象，真正实现对象是InputMethodManagerService（IMMS），这里有一步跨进程操作。那我们直接看 IMMS 的 showSoftInput()吧。

## IMMS # showSoftInput()

```java
@Override
public boolean showSoftInput(IInputMethodClient client, int flags , ResultReceiver resultReceiver) {
  if (!calledFromValidUser()) {
    return false;
  }
  int uid = Binder.getCallingUid();
  long ident = Binder.clearCallingIdentity();
  try {
    synchronized (mMethodMap) {
      if (mCurClient == null || client == null || mCurClient.client.asBinder() != client.asBinder()) {
        try {
          // 我们需要检查这是否是焦点在窗口管理器中的当前客户端，以便在输入启动之前进行此调用。
          if (!mIWindowManager.inputMethodClientHasFocus(client)) {
            return false;
          }
        } catch (RemoteException e) {
          return false;
        }
      }
      return showCurrentInputLocked(flags, resultReceiver);
    }
  } finally {
    Binder.restoreCallingIdentity(ident);
  }
}
```
执行到了 showCurrentInputLocked() 中
```java
boolean showCurrentInputLocked(int flags, ResultReceiver resultReceiver) {
  ...
  if (mCurMethod != null) {
    executeOrSendMessage(mCurMethod, mCaller.obtainMessageIOO(MSG_SHOW_SOFT_INPUT, getImeShowFlags() , mCurMethod , resultReceiver));
    mInputShown = true;
    if (mHaveConnection && !mVisibleBound) {
      bindCurrentInputMethodService(mCurIntent, mVisibleConnection , Context.BIND_AUTO_CREATE | Context.BIND_TREAT_LIKE_ACTIVITY);
      mVisibleBound = true;
    }
    res = true;
  } else if (mHaveConnection && SystemClock.uptimeMillis() >= (mLastBindTime + TIME_TO_RECONNECT)) {
    EventLog.writeEvent(EventLogTags.IMF_FORCE_RECONNECT_IME, mCurMethodId , SystemClock.uptimeMillis() - mLastBindTime, 1);
    mContext.unbindService(this);
    bindCurrentInputMethodService(mCurIntent, this , Context.BIND_AUTO_CREATE | Context.BIND_NOT_VISIBLE);
  } else {
  }
  return res;
}
```
在 showCurrentInputLocked中会通过executeOrSendMessage() 发送 MSG_SHOW_SOFT_INPUT 消息，最后就执行到了HandlerCaller.java 的 handleMessage() 的 MSG_SHOW_SOFT_INPUT 中

```java
executeOrSendMessage(mCurMethod, mCaller.obtainMessageIOO(MSG_SHOW_SOFT_INPUT,
 getImeShowFlags() , mCurMethod , resultReceiver));

//HandlerCaller.java
public Message obtainMessageIOO(int what, int arg1 , Object arg2 , Object arg3) {
    SomeArgs args = SomeArgs.obtain();
    args.arg1 = arg2;
    args.arg2 = arg3;
    return mH.obtainMessage(what, arg1 , 0 , args);
}

case MSG_SHOW_SOFT_INPUT:
    args = (SomeArgs) msg.obj;
    try {
      ((IInputMethod) args.arg1).showSoftInput(msg.arg1, (ResultReceiver) args.arg2);
    } catch (RemoteException e) {
    }
    args.recycle();
    return true;

```
而在 [源码分析 - Andrid 输入法框架 之 启动服务](../../../../2019/05/20/IMF-start/)也说了，在IMMS绑定成功，mCurMethod =IInputMethod.Stub.asInterface(service)，是IInputMethodWrapper的代理对象，而 args.arg1 就是传递过来的mCurMethod ，那我们就直接看看 IInputMethodWrapper的 showSoftInput()。
```java
//IInputMethodWrapper.java
public void showSoftInput(int flags, ResultReceiver resultReceiver) {
    mCaller.executeOrSendMessage(mCaller.obtainMessageIO(DO_SHOW_SOFT_INPUT, flags , resultReceiver));
}
//发送消息，然后处理DO_SHOW_SOFT_INPUT 这个消息
@Override
public void executeMessage(Message msg) {
		InputMethod inputMethod = mInputMethod.get();
    ...
    switch (msg.what) {
      case DO_SHOW_SOFT_INPUT:
        inputMethod.showSoftInput(msg.arg1, (ResultReceiver) msg.obj);
    }
  }
```
inputMethod 在 [源码分析 - Andrid 输入法框架 之 启动服务](../../../../2019/05/20/IMF-start/)也说了，就是InputMethodImpl对象。那么我们就直接看 inputMethod.showSoftInput()的流程吧。
## InputMethodImpl # showSoftInput()
```java
//InputMethodImpl.java
public void showSoftInput(int flags, ResultReceiver resultReceiver) {
    ...
    //这个是真正显示 UI 的函数
    showWindow(true);
    ...
}

public void showWindow(boolean showInput) {
  if (mInShowWindow) {
    return;
  }
  try {
    mWindowWasVisible = mWindowVisible;
    mInShowWindow = true;
    showWindowInner(showInput);
  } finally {
    mWindowWasVisible = true;
    mInShowWindow = false;
  }
}

void showWindowInner(boolean showInput) {
  ...
  initialize();
  updateFullscreenMode();
  //这个函数会创建输入法的键盘
  updateInputViewShown();

  if (!mWindowAdded || !mWindowCreated) {
    mWindowAdded = true;
    mWindowCreated = true;
    initialize();
    //创建输入法 dialog 里的候选词View
    View v = onCreateCandidatesView();
    if (v != null) {
      setCandidatesView(v);
    }
  }
  if (mShowInputRequested) {
    if (!mInputViewStarted) {
      mInputViewStarted = true;
      onStartInputView(mInputEditorInfo, false);
    }
  } else if (!mCandidatesViewStarted) {
    mCandidatesViewStarted = true;
    onStartCandidatesView(mInputEditorInfo, false);
  }
  if (doShowInput) {
    startExtractingText(false);
  }
  if (!wasVisible) {
    mImm.setImeWindowStatus(mToken, IME_ACTIVE , mBackDisposition);
    onWindowShown();
    //这个是 Dialog 的 window ,这里开始就显示 UI 了
    mWindow.show();
  }
}
```
关键也就是 showWindowInner()方法，这里面主要做了以下工作

1. 在 updateInputViewShown() 中会创建输入法键盘布局，并添加进去   
2. 通过 onCreateCandidatesView() 创建输入法 dialog 里的候选词View
3. onWindowShown()当输入法显示的时候，执行的方法，可以在这里面做一些操作
4. 执行 mWindow.show()从而显示输入法

接下来就说说 updateInputViewShown()

## InputMethodImpl # updateInputViewShown()

```java
public void updateInputViewShown() {
  boolean isShown = mShowInputRequested && onEvaluateInputViewShown();
  if (mIsInputViewShown != isShown && mWindowVisible) {
    mIsInputViewShown = isShown;
    mInputFrame.setVisibility(isShown ? View.VISIBLE : View.GONE);
    if (mInputView == null) {
      initialize();
        //这个是核心 view ，创建显示键盘的根view
      View v = onCreateInputView();
      if (v != null) {
        setInputView(v);
      }
    }
  }
}

public void setInputView(View view) {
  mInputFrame.removeAllViews();
  mInputFrame.addView(view, new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,ViewGroup.LayoutParams.WRAP_CONTENT));
  mInputView = view;
}
public View onCreateInputView() {
  return null;
}
```

onCreateInputView()在IMS中是空实现，这个是我们开发输入法的时候，设置输入法布局，创建这个View之后，就会通过 setInputView()把这个View添加到mInputFrame中。这个mInputFrame就是布局是R.layout.input_method中 id 为@android:id/inputArea的 FrameLayout。

例如
```java
public class LatinIME extends InputMethodService {

@Override
public View onCreateInputView() {
  if (mEnvironment.needDebug()) {
    Log.d(TAG, "onCreateInputView.");
   }
   rootView = getWindow().getWindow().findViewById(Window.ID_ANDROID_CONTENT);
   /**
   * 获取导航栏高度——方法1
   * */
   navigationBarHeight = -1;
   //获取status_bar_height资源的ID
   int resourceId = getResources().getIdentifier("navigation_bar_height", "dimen", "android");
   if (resourceId > 0) {
    //根据资源ID获取响应的尺寸值
     navigationBarHeight = getResources().getDimensionPixelSize(resourceId);
     }

     LayoutInflater inflater = getLayoutInflater();
     mSkbContainer = (SkbContainer) inflater.inflate(R.layout.skb_container, null);
    if (Utils.isShowKeyboardInEink()) {
      mSkbContainer.setBackgroundColor(getResources().getColor(R.color.white));
     }

    mSkbContainer.setService(this);
    mSkbContainer.setInputModeSwitcher(mInputModeSwitcher);
    mSkbContainer.setGestureDetector(mGestureDetectorSkb);

    return mSkbContainer;
  }

  @Override
	public View onCreateCandidatesView() {
    if (mEnvironment.needDebug()) {
      Log.d(TAG, "onCreateCandidatesView.");
    }

    LayoutInflater inflater = getLayoutInflater();
    mCandidatesContainer = (CandidatesContainer) inflater.inflate(R.layout.candidates_container, null);
    //将ComposingView直接显示到CandidatesContainer中
    mComposingView = (ComposingView) mCandidatesContainer.findViewById(R.id.composing_view);
    line_separator1 = (TextView) mCandidatesContainer.findViewById(R.id.line_separator1);
    line_separator2 = (TextView) mCandidatesContainer.findViewById(R.id.line_separator2);
    // Create balloon hint for candidates view.
    mCandidatesBalloon = new BalloonHint(this, mCandidatesContainer, MeasureSpec.UNSPECIFIED);
    mCandidatesBalloon.setBalloonBackground(getResources().getDrawable(R.drawable.candidate_balloon_bg));
    mCandidatesContainer.initialize(mChoiceNotifier, mCandidatesBalloon, mGestureDetectorCandidates);

    setCandidatesViewShown(true);
    return mCandidatesContainer;
  }
}
```

到这里，输入法已经显示出来了，可是怎么把文本提交到输入框中呢。

简单来说就是通过InputConnection 的 commitText()方法，在每个IMS中，都可以通过getCurrentInputConnection()得到对应的InputConnection，

```java
private void commitResultText(String resultText, String secondeText) {
  InputConnection ic = getCurrentInputConnection();
  ic.commitText(resultText, 1);
}
```
EditText 获得焦点，传递的InputConnection就是EditableInputConnection，它继承BaseInputConnection，至于这个是在什么时候传递过去的,请看 [源码分析 - Andrid 输入法 之  InputConnection 对象创建](../../../../2019/06/03/creat-inputconnection/)   

---
搬运地址：    
