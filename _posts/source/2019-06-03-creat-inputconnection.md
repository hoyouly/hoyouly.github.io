---
layout: post
title: 源码分析 - Andrid 输入法框架 之  InputConnection 对象创建
category: 源码分析
tags: IMF
---
<!-- * content -->
<!-- {:toc} -->

我们在前面知道。在View的onTouchEvent()中，执行了requestFocus()使得TextView获得了焦点。

而requestFocus() -> requestFocusNoSearch()-> handleFocusGainInternal()->onFocusChanged(),所以我们就直接看 onFocusChanged()
### View # onFocusChanged()
```java
protected void onFocusChanged(boolean gainFocus, @FocusDirection int direction, @Nullable Rect previouslyFocusedRect) {
  ...
  InputMethodManager imm = InputMethodManager.peekInstance();
  if (!gainFocus) {
    if (isPressed()) {
      setPressed(false);
    }
    if (imm != null && mAttachInfo != null && mAttachInfo.mHasWindowFocus) {
      imm.focusOut(this);
    }
    onFocusLost();
  } else if (imm != null && mAttachInfo != null && mAttachInfo.mHasWindowFocus) {
    //通知IMMS该view获得了焦点，到此，这后面的逻辑就和上面的window获得焦点导致view和输入法绑定的逻辑一样了
    imm.focusIn(this);
  }
  ...
}
```
在这里面，我们看到了一些熟悉的东西。InputMethodManager.peekInstance()，得到IMM对象，然后执行到了imm.focusIn(this)方法。   
所以我们可知。焦点 view 请求绑定输入法是通过调用InputMethodManager.focusIn()实现的。    
### IMM # focusIn()

```java
public void focusIn(View view) {
  synchronized (mH) {
    focusInLocked(view);
  }
}

void focusInLocked(View view) {
  if (mCurRootView != view.getRootView()) {
    return;
  }
  //保存焦点view变量
  mNextServedView = view;
  scheduleCheckFocusLocked(view);
}

static void scheduleCheckFocusLocked(View view) {
  ViewRootImpl viewRootImpl = view.getViewRootImpl();
  if (viewRootImpl != null) {
    viewRootImpl.dispatchCheckFocus();
  }
}
```
focusIn(),经过一系列调用，最后执行到了viewRootImpl.dispatchCheckFocus()中。

而在 dispatchCheckFocus()中，由通过发消息，处理消息执行到了IMM的checkFocus()中。转了一圈，又到了IMM中，那就继续查看 IMM 的checkFocus()中

###  IMM # checkFocus()

```java
public void checkFocus() {
  if (checkFocusNoStartInput(false, true)) {
    startInputInner(null, 0, 0, 0);
  }
}

boolean startInputInner(IBinder windowGainingFocus, int controlFlags, int softInputMode, int windowFlags) {
    ...
    EditorInfo tba = new EditorInfo();
    tba.packageName = view.getContext().getPackageName();
    tba.fieldId = view.getId();
    //创建数据通信连接接口，这个会传送到 InputMethodService
    //InputMethodService后面就通过这个connection将输入法的字符传递给该view
    //由具体的view创建,比如TextView创建的
    InputConnection ic = view.onCreateInputConnection(tba);
    synchronized (mH) {
      ...
      mServedInputConnection = ic;
      ControlledInputConnectionWrapper servedContext;
      if (ic != null) {
        ...
        //将InputConnection封装为binder对象，这个是真正可以实现跨进程通信的封装类
        servedContext = new ControlledInputConnectionWrapper(vh.getLooper(), ic, this);
      } else {
        servedContext = null;
      }
      ...
      InputBindResult res;
      if (windowGainingFocus != null) {//focusIn这个不会走到这条分支
        res = mService.windowGainedFocus(mClient, windowGainingFocus, controlFlags, softInputMode, windowFlags, tba, servedContext);
      } else {
        //通知 InputMethodManagerService,该程序的view获得焦点，IMMS就会将这个view和输入法绑定
        res = mService.startInput(mClient, servedContext, tba, controlFlags);
      }
    }
    return true;
}
```

在这里面就会通过 view.onCreateInputConnection() 创建一个InputConnection

### TextView # onCreateInputConnection()
Edittext 中没有实现onCreateInputConnection(), TextView 中实现了这个方法

```java
public InputConnection onCreateInputConnection(EditorInfo outAttrs) {
  if (onCheckIsTextEditor() && isEnabled()) {
    ...
    if (mText instanceof Editable) {
      //露面了，是 EditableInputConnection, textView作为参数传入
      InputConnection ic = new EditableInputConnection(this);
      outAttrs.initialSelStart = getSelectionStart();
      outAttrs.initialSelEnd = getSelectionEnd();
      outAttrs.initialCapsMode = ic.getCursorCapsMode(getInputType());
      return ic;
    }
  }
  return null;
}
```
这样在 IMM 中保存了对应View的InputConnection对象，同时会存到servedContext 中，然后执行 mService.startInput()。

经过一系列调用，mService.startInput()->startInputLocked()->startInputUncheckedLocked()，执行到了 startInputUncheckedLocked()中。
### IMMS # startInputUncheckedLocked()

主要关注的就是 inputContext，因为这里面保存了我们通过TexView创建的 EditableInputConnection 对象
```java
//InputMethodManagerService
InputBindResult startInputUncheckedLocked(ClientState cs, IInputContext inputContext, EditorInfo attribute, int controlFlags) {
  //将新程序设置为当前活动的程序
  mCurClient = cs;
  mCurInputContext = inputContext;
  mCurAttribute = attribute;

  if (mCurId != null && mCurId.equals(mCurMethodId)) {
    if (cs.curSession != null) {
        //连接已经建立，直接开始绑定
        return attachNewInputLocked((controlFlags & InputMethodManager.CONTROL_START_INITIAL) != 0);
      }
      ...
    }
    //否则需要启动输入法，并建立连接
    return startInputInnerLocked();
}
```
inputContext会把值赋给 mCurInputContext，所以 mCurInputContext中就带有 EditableInputConnection对象。
在attachNewInputLocked()中会发消息MSG_START_INPUT 消息，同时把 mCurInputContext 发送出去。
```java
InputBindResult attachNewInputLocked(boolean initial) {
  ...
  executeOrSendMessage(session.method, mCaller.obtainMessageOOO(MSG_START_INPUT, session, mCurInputContext, mCurAttribute));
  ...
  return new InputBindResult(session.session, (session.channel != null ? session.channel.dup() : null), mCurId, mCurSeq, mCurUserActionNotificationSequenceNumber);
}
```
然后处理消息。
```java
case MSG_START_INPUT:
  args = (SomeArgs) msg.obj;
  try {
	   SessionState session = (SessionState) args.arg1;
	   setEnabledSessionInMainThread(session);
	   session.method.startInput((IInputContext) args.arg2, (EditorInfo) args.arg3);
    } catch (RemoteException e) {
    }
    args.recycle();
  return true;
```
args.arg2 就是 mCurInputContext    
session.method 就是 IInputMethodWrapper 对象    
在 startInput()中会继续发消息 DO_START_INPUT，同时把 mCurInputContext 值继续发出去。然后再次处理消息。

```java
//IInputMethodWrapper.java
case DO_START_INPUT: {
  SomeArgs args = (SomeArgs) msg.obj;
  // IInputContext就是输入法和文本输入view的通信接口 通过这个接口，输入法能够获取view的信息，也能够直接将文本传送给view
  IInputContext inputContext = (IInputContext) args.arg1;
  InputConnection ic = inputContext != null ? new InputConnectionWrapper(inputContext) : null;
  EditorInfo info = (EditorInfo) args.arg2;
  info.makeCompatible(mTargetSdkVersion);
  inputMethod.startInput(ic, info);
  args.recycle();
  return;
}
```
这里得到 InputConnection 对象，这个就是我们通过TextView传递过来的 EditableInputConnection对象，

因为 inputMethod 就是IMS中的内部类 InputMethodImpl 的对象，继续执行 startInput()->doStartInput()
### InputMethodImpl # doStartInput()
```java
//InputMethodImpl.java
void doStartInput(InputConnection ic, EditorInfo attribute, boolean restarting) {
    if (!restarting) {
        doFinishInput();
    }
    mInputStarted = true;
    mStartedInputConnection = ic;
    mInputEditorInfo = attribute;
    initialize();
    onStartInput(attribute, restarting);
    if (mWindowVisible) {
        if (mShowInputRequested) {
            mInputViewStarted = true;
            onStartInputView(mInputEditorInfo, restarting);
            startExtractingText(true);
        } else if (mCandidatesVisibility == View.VISIBLE) {
            mCandidatesViewStarted = true;
            onStartCandidatesView(mInputEditorInfo, restarting);
        }
    }
}
```
在这里面，会把这个传递过来的ic赋给 mStartedInputConnection,而我们在提交文本的时候，通过getCurrentInputConnection()其实就是这个

```java
public InputConnection getCurrentInputConnection() {
    InputConnection ic = mStartedInputConnection;
    if (ic != null) {
        return ic;
    }
    return mInputConnection;
}
```
因为 mStartedInputConnection 不为空，所以getCurrentInputConnection()就直接返回了 EditableInputConnection对象 。

---
搬运地址：    
